# 09-3.部署 heapster 插件

Heapster是一个收集者，将每个Node上的cAdvisor的数据进行汇总，然后导到第三方工具(如InfluxDB)。

Heapster 是通过调用 kubelet 的 http API 来获取 cAdvisor 的 metrics 数据的。

由于 kublet 只在 10250 端口接收 https 请求，故需要修改 heapster 的 deployment 配置。同时，需要赋予 kube-system:heapster ServiceAccount 调用 kubelet API 的权限。

## 下载 heapster 文件

到 [heapster release 页面](https://github.com/kubernetes/heapster/releases) 下载最新版本的 heapster

``` bash
wget https://github.com/kubernetes/heapster/archive/v1.5.3.tar.gz
tar -xzvf v1.5.3.tar.gz
mv v1.5.3.tar.gz heapster-1.5.3.tar.gz
```

官方文件目录： `heapster-1.5.3/deploy/kube-config/influxdb`

## 修改配置
更改镜像地址的部分，如果网络状况允许，可以不替换镜像地址。

``` bash
$ cd heapster-1.5.3/deploy/kube-config/influxdb
$ cp grafana.yaml{,.orig}
$ diff grafana.yaml.orig grafana.yaml
16c16
<         image: gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
---
>         image: wanghkkk/heapster-grafana-amd64-v4.4.3:v4.4.3
67c67
<   # type: NodePort
---
>   type: NodePort
```
+ 开启 NodePort；

``` bash
$ cp heapster.yaml{,.orig}
$ diff heapster.yaml.orig heapster.yaml
23c23
<         image: gcr.io/google_containers/heapster-amd64:v1.5.3
---
>         image: fishchen/heapster-amd64:v1.5.3
27c27
<         - --source=kubernetes:https://kubernetes.default
---
>         - --source=kubernetes:https://kubernetes.default?kubeletHttps=true&kubeletPort=10250&insecure=true
```
+ 由于 kubelet 只在 10250 监听 https 请求，如果由于证书报错，也把insecure=true参数添加上， 故添加相关参数；

``` bash
$ cp influxdb.yaml{,.orig}
$ diff influxdb.yaml.orig influxdb.yaml
16c16
<         image: gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
---
>         image: fishchen/heapster-influxdb-amd64:v1.3.3
```

## 执行所有定义文件

``` bash
$ pwd
deploy/kube-config/influxdb
$ ls *.yam
grafana.yaml  heapster.yaml  influxdb.yaml
$ kubectl create -f  .
deployment.extensions/monitoring-grafana created
service/monitoring-grafana created
serviceaccount/heapster created
deployment.extensions/heapster created
service/heapster created
deployment.extensions/monitoring-influxdb created
service/monitoring-influxdb created
```
-----------
设置 rbac 访问
```bash
$ cd ../rbac/
$ pwd
deploy/kube-config/rbac
$ ls
heapster-rbac.yaml
$ cp heapster-rbac.yaml{,.orig}
$ diff heapster-rbac.yaml.orig heapster-rbac.yaml
12a13,26
> ---
> kind: ClusterRoleBinding
> apiVersion: rbac.authorization.k8s.io/v1beta1
> metadata:
>   name: heapster-kubelet-api
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: ClusterRole
>   name: system:kubelet-api-admin
> subjects:
> - kind: ServiceAccount
>   name: heapster
>   namespace: kube-system
>

$ kubectl create -f heapster-rbac.yaml
```

+ 将 serviceAccount kube-system:heapster 与 ClusterRole system:kubelet-api-admin 绑定，授予它调用 kubelet API 的权限；

## 检查执行结果

``` bash
[root@master01 rbac]# kubectl get pods -n kube-system | grep -E 'heapster|monitoring'
heapster-8f9559ddc-9dh4k                1/1     Running   0          3m33s
monitoring-grafana-56b668bccf-hlzz4     1/1     Running   0          3m34s
monitoring-influxdb-5c5bf4949d-jlw5j    1/1     Running   0          3m34s
```

## cAdvisor
由于新版本的k8s移除了 cAdvisor,需要在宿主上运行，或者使用daemonset方式运行，这里我们采用宿主上独立运行：
```bash
# cadvisor -housekeeping_interval 10s -port 4194 &
```

检查 kubernets dashboard 界面，可以正确显示各 Nodes、Pods 的 CPU、内存、负载等统计数据和图表：
![dashboard-heapster](/images/dashboard-heapster.png)

## 访问 grafana

1. 通过 kube-apiserver 访问：

获取 monitoring-grafana 服务 URL：

``` bash
[root@master01 rbac]# kubectl cluster-info
Kubernetes master is running at https://192.168.10.232:6443
Heapster is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/heapster/proxy
CoreDNS is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
monitoring-grafana is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
monitoring-influxdb is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy
```

浏览器访问 URL： `https://192.168.10.242:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy`
对于 virtuabox 做了端口映射： `http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy`

1. 通过 kubectl proxy 访问：
创建代理

``` bash
kubectl proxy --address='192.168.10.242' --port=8086 --accept-hosts='^*$'
Starting to serve on 192.168.10.242:8086
```

浏览器访问 URL：`http://192.168.10.242:8086/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy/?orgId=1`
对于 virtuabox 做了端口映射： `http://127.0.0.1:8086/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy/?orgId=1`

1. 通过 NodePort 访问：

``` bash
[root@master01 rbac]# kubectl get svc -n kube-system|grep -E 'monitoring|heapster'
heapster               ClusterIP   10.254.192.191   <none>        80/TCP          8m34s
monitoring-grafana     NodePort    10.254.132.56    <none>        80:39696/TCP    8m34s
monitoring-influxdb    ClusterIP   10.254.88.183    <none>        8086/TCP        8m34s
```

+ grafana 监听 NodePort 39696:

浏览器访问 URL：`http://192.168.10.242:39696/?orgId=1`

![grafana](/images/grafana.png)


参考：
1. 配置 heapster：https://github.com/kubernetes/heapster/blob/master/docs/source-configuration.md
