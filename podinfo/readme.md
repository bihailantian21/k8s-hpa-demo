# HPA yaml

podinfo-dep.yaml

定义了一个 Deployment，用来控制 podinfo 副本数等状态。

在 spec 下添加了：

```
selector:
    matchLabels:
      app: podinfo
```

podinfo-hpa.yaml

定义了一个 HPA，当 podinfo 所有副本的 cpu 使用率的值超过 request 限制的 80% 或者 memory 的使用率超过 200 Mi 时会触发自动动态扩容行为，
扩容或缩容时必须满足一个约束条件是 Pod 的副本数要介于  2与 10 之间。

更改了：

```
apiVersion: autoscaling/v2beta1
```

podinfo-hpa-custom.yaml

HPA 自定义指标基于 Prometheus 的案例。

podinfo-ingress.yaml

通过其 Ingress，对外暴露 podinfo.weavedx.com，可以通过绑定主机名的方式来访问该应用。

podinfo-svc.yaml

定义了一个 Service，匹配 podinfo 标签，通过 NodePort 暴露 31198 端口。

具体详情可见[k8s-prom-hpa](https://github.com/stefanprodan/k8s-prom-hpa)