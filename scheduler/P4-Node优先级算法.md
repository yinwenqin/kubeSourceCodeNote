# P4-Node优先级算法

## 前言

在上一篇文档中，我们过了一遍node筛选算法：

[p3-Node筛选算法](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/scheduler/P3-Node%E7%AD%9B%E9%80%89%E7%AE%97%E6%B3%95.md)

按调度规则设计，对筛选出的node，选择优先级最高的作为最终的fit node。那么本篇承接上一篇，进入下一步，node优先级算法。



## 正文

同上一篇，回到`pkg/scheduler/core/generic_scheduler.go`中的`Schedule()`函数，`pkg/scheduler/core/generic_scheduler.go:184`:

![](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/scheduler/image/p4/schedule.jpg)

截图中有几处标注，metric相关的几行，是收集metric信息，用以提供给prometheus使用的，kubernetes的几个核心组件都有这个功能，以后如果读prometheus的源码，这个单独拎出来再讲。



**PodAffinity**示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  namespace: default
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - podAffinityTerm:
        weight: 100
          labelSelector:
            matchExpressions:
            - key: like
              operator: In
              values:
              - pod-a
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: unlike
              operator: In
              values:
              - pod-a
          topologyKey: kubernetes.io/hostname          
  containers:
  - name: test
    image: gcr.io/google_containers/pause:2.0
```

