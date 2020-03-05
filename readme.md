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
请确认 Kubernetes 集群配置开启了 Aggregation Layer(聚合层)。参考[前提](https://github.com/kubernetes-sigs/metrics-server#requirements)。

kubernetes/deployment 目录下的 yaml 文件介绍：

```
aggregated-metrics-reader.yaml
auth-delegator.yaml
auth-reader.yaml
metrics-apiservice.yaml
metrics-server-deployment.yaml  
替换 k8s.gcr.io/metrics-server-amd64:v0.3.6 为 mirrorgooglecontainers/metrics-server-amd64:v0.3.6 注意 imagePullPolicy: IfNotPresent
args 参数添加 
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
metrics-server-service.yaml
resource-reader.yaml
```



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

# 总结

# 参考

[官网 HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
[autoscaler](https://github.com/kubernetes/autoscaler)
[metrics-server](https://github.com/kubernetes-sigs/metrics-server)