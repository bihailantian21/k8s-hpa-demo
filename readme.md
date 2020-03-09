# HPA 与 VPA

HPA：HPA 是 Kubernetes 里面 pod 弹性伸缩的实现，它能根据设置的监控指标阈值进行 pod 的弹性扩缩容，
目前默认 HPA 只能支持 cpu 和内存的阀值检测扩缩容机制，但也可以通过 custom metric api 调用 prometheus 
等监控系统实现自定义 metric 来更加灵活地监控指标实现弹性伸缩。但是 HPA 不能用于伸缩一些无法进行缩放的控制器如 DaemonSet。

VPA: Vertical Pod Autoscaler，Kubernetes 纵向扩容，即用户无需为其 pods 中的容器设置 request。
配置 VPA 后，它将根据使用情况自动设置 request，从而允许在节点上进行适当的调度，为每个 pod 提供适当的资源量。

# 介绍

自从 Kubernetes 1.8 开始，指标通过 Metrics API 从 Kubernetes 中获取，从 Kubernetes 1.11 开始 Heapster 被废弃不再推荐使用，metrics-server 替代了 Heapster。

Metrics server 是 Kubernetes 集群资源使用情况的聚合器，Kubernetes 中有些组件依赖资源指标 API(metric API)，
如 top、hpa 等。如果没有运行资源指标 API 接口，这些组件无法运行。之前使用的是 Heapster，目前推荐使用 metrics-server。

Metrics API 的 api 路径是 /apis/metrics.k8s.io/。

# 示例

这里先简单介绍与熟悉 HPA，运行 HPA Demo。

部署 [metrics-server](https://github.com/kubernetes-sigs/metrics-server)。在部署 metrics-server 之前，
请确认 Kubernetes 集群配置开启了 Aggregation Layer(聚合层)。参考[前提条件](https://github.com/kubernetes-sigs/metrics-server#requirements)。

kubernetes/deployment 目录下的 yaml 文件介绍：

这些 yaml 文件参考于 [metrics-server](https://github.com/kubernetes-sigs/metrics-server)。

```
aggregated-metrics-reader.yaml

auth-delegator.yaml

auth-reader.yaml

metrics-apiservice.yaml

metrics-server-deployment.yaml  
替换 k8s.gcr.io/metrics-server-amd64:v0.3.6 为 mirrorgooglecontainers/metrics-server-amd64:v0.3.6 
注意 imagePullPolicy: IfNotPresent
args 参数添加 
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP

metrics-server-service.yaml

resource-reader.yaml
```

以上 yaml 文件的作用，自行查找。

部署完 [metrics-server](https://github.com/kubernetes-sigs/metrics-server)后，执行 `kubectl api-versions` 操作，显示如下：

```
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
crd.projectcalico.org/v1
discovery.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
metrics.k8s.io/v1beta1   # 新增的
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

## 示例一

参照[官网 HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) 简单实现基于内存或者 CPU 的 HPA 案例。

这里简单的说下步骤，后面重点学习下基于 metric API 自定义指标的 HPA。

第一：打包镜像，运行 php-server 的 Deployment 与 Service

```
docker build -t  tanjunchen/hpa-example:test .
然后将镜像分发到各个 Node 节点上
kubectl apply -f php-apache.yaml
```
以上主要是使用自定义的镜像，使用官方镜像请科学上网。

或者直接运行命令 `kubectl apply -f https://k8s.io/examples/application/php-apache.yaml`

第二：创建 HPA

`kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5`

第三：查看 hpa 与 svc

`kubectl get hpa` 
`kubectl get svc -o wide`

第四：压力测试

```
kubectl run --generator=run-pod/v1 -i --tty load-generator --image=busybox /bin/sh
while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

`kubectl describe deploy Deployment/php-apache`

```
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  9m7s   deployment-controller  Scaled up replica set php-apache-6c5bfb4d65 to 1
  Normal  ScalingReplicaSet  6m4s   deployment-controller  Scaled up replica set php-apache-6c5bfb4d65 to 4
  Normal  ScalingReplicaSet  5m49s  deployment-controller  Scaled up replica set php-apache-6c5bfb4d65 to 5
```

kill 掉相应的 pod，几分钟后

```
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  12m    deployment-controller  Scaled up replica set php-apache-6c5bfb4d65 to 1
  Normal  ScalingReplicaSet  8m57s  deployment-controller  Scaled up replica set php-apache-6c5bfb4d65 to 4
  Normal  ScalingReplicaSet  8m42s  deployment-controller  Scaled up replica set php-apache-6c5bfb4d65 to 5
  Normal  ScalingReplicaSet  66s    deployment-controller  Scaled down replica set php-apache-6c5bfb4d65 to 1
```

# 示例二

基于自定义监控指标的弹性伸缩

实际生产环境中，通过 CPU 和内存的监控指标弹性伸缩不能很好的反映应用真实的状态，
所以需要根据应用本身自定义一些监控指标来进行弹性伸缩，如 web 应用，根据当前 QPS 来进行弹性，
Kubernetes HPA 本身也支持自定义监控指标。

自定义监控指标收集过程：

    pod 内置一个 metrics 或者挂一个 sidecar 当作 exporter 对外暴露
    Prometheus 收集对应的监控指标
    Prometheus-adapter 定期从 prometheus 收集指标对抓取的监控指标进行过滤和筛选，通过 custom-metrics-apiserver 将指标对外暴露
    HPA 控制器从 custom-metrics-apiserver 获取数据

参考 https://github.com/stefanprodan/k8s-prom-hpa

# 问题与总结

```
E0306 02:51:31.263602       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:k8s-node02: unable to fetch metrics from Kubelet k8s-node02 (192.168.17.132): Get https://192.168.17.132:10250/stats/summary?only_cpu_and_memory=true: dial tcp 192.168.17.132:10250: connect: no route to host, unable to fully scrape metrics from source kubelet_summary:k8s-node01: unable to fetch metrics from Kubelet k8s-node01 (192.168.17.131): Get https://192.168.17.131:10250/stats/summary?only_cpu_and_memory=true: dial tcp 192.168.17.131:10250: connect: no route to host]
```

```
E0306 02:41:43.384810       1 manager.go:111] unable to fully collect metrics: [unable to fully scrape metrics from source kubelet_summary:k8s-master: unable to fetch metrics from Kubelet k8s-master (k8s-master): Get https://k8s-master:10250/stats/summary?only_cpu_and_memory=true: x509: certificate signed by unknown authority, unable to fully scrape metrics from source kubelet_summary:k8s-node02: unable to fetch metrics from Kubelet k8s-node02 (k8s-node02): Get https://k8s-node02:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup k8s-node02 on 114.114.114.114:53: no such host, unable to fully scrape metrics from source kubelet_summary:k8s-node01: unable to fetch metrics from Kubelet k8s-node01 (k8s-node01): Get https://k8s-node01:10250/stats/summary?only_cpu_and_memory=true: dial tcp: lookup k8s-node01 on 8.8.8.8:53: no such host]
```
这就是为什么要在 args 中添加一下参数：
```
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```
经过以上修改，metrics-server 就会改为以 IP 形式来请求 metrics 数据，kubelet-insecure-tls 参数是因为改为 IP 后，
原来基于主机名的证书就不能用了（会提示x.509证书错误），只能使用非安全连接。当然这里只是针对测试或者本地环境，生产环境慎用。

另一种解决方案是使用 dnsmasq 构建一个上游的 dns 服务。

`kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.17.0  --pod-network-cidr=10.244.0.0/16`

kubectl top nodes 可能会出现以下问题：

![show-unknown](pic/top-node-show.jpg)

这种问题一般是由于网络导致的。可能是由于 `--pod-network-cidr` 参数指定的值跟物理网络段冲突，导致网络之间不通。可以参阅 [pod-network-cidr 有什么作用？](https://blog.csdn.net/shida_csdn/article/details/104334372)

我部署的网络插件是 [calico](kubernetes/network/calico.yaml)，注意我修改了 `CALICO_IPV4POOL_CIDR` 的值为 `172.20.0.0/16`。

`kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.17.0  --pod-network-cidr=172.20.0.0/16  --service-cidr=10.32.0.0/24`

kubectl top nodes

```
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master   141m         7%     2024Mi          53%       
k8s-node01   71m          7%     1776Mi          46%       
k8s-node02   180m         18%    1910Mi          50% 
```

常见排错手段：

telnet IP 10250

route -n

ip a

kubectl logs -n kube-system  calico-node-*

kubectl logs -n kube-system  metrics-server-**-*



# 参考

[官网 HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
[autoscaler](https://github.com/kubernetes/autoscaler)
[metrics-server](https://github.com/kubernetes-sigs/metrics-server)