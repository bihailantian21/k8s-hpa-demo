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

设置 master 节点可以调度 Pod
kubectl taint node k8s-master node-role.kubernetes.io/master-

添加了 label metrics: "yes"
kubectl label nodes  k8s-master  metrics=yes


## 示例一

参照[官网 HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) 简单实现基于内存或者 CPU 的 HPA 案例。

这里简单的说下步骤，后面重点学习下基于 metric API 自定义指标的 HPA。

第一：打包镜像，运行 php-server 的 Deployment 与 Service
```
docker build -t  tanjunchen/hpa-example:test .
```
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
原来基于主机名的证书就不能用了（会提示x.509证书错误），只能使用非安全连接。

还有就是使用 dnsmasq 构建一个上游的 dns 服务

kubectl logs -n kube-system kube-apiserver-k8s-master

```
I0306 02:30:23.493919       1 log.go:172] http: TLS handshake error from 192.168.17.132:56160: remote error: tls: bad certificate
```

```
E0306 02:37:20.916909       1 controller.go:114] loading OpenAPI spec for "v1beta1.metrics.k8s.io" failed with: failed to retrieve openAPI spec, http error: ResponseCode: 503, Body: service unavailable
, Header: map[Content-Type:[text/plain; charset=utf-8] X-Content-Type-Options:[nosniff]]
```





# 参考

[官网 HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
[autoscaler](https://github.com/kubernetes/autoscaler)
[metrics-server](https://github.com/kubernetes-sigs/metrics-server)