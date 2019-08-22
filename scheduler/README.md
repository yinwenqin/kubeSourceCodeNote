# 调度器总体设计

## 调度器源码分段阅读目录
- [调度器入口](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/scheduler/Kubernetes%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-Scheduler-P1-%E8%B0%83%E5%BA%A6%E5%99%A8%E5%85%A5%E5%8F%A3%E7%AF%87.md)

- [调度器框架](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/scheduler/Kubernetes%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-Scheduler-P2-%E8%B0%83%E5%BA%A6%E5%99%A8%E6%A1%86%E6%9E%B6.md)

- [Node筛选算法](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/scheduler/Kubernetes%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-Scheduler-P3-Node%E7%AD%9B%E9%80%89%E7%AE%97%E6%B3%95.md)

- [Node优先级算法](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/scheduler/Kubernetes%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-Scheduler-P4-Node%E4%BC%98%E5%85%88%E7%BA%A7%E7%AE%97%E6%B3%95.md)

- 待补充

## 概览
首先列出官方md链接，讲解颇为生动：
https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md
这里用结合自己阅读代码的理解做一下翻译。

### 工作模式
Kubernetes scheduler独立运作与其他主要组件之外(例如API Server)，它连接API Server，watch观察，如果有PodSpec.NodeName为空的Pod出现，则开始工作，通过一定得筛选算法，筛选出合适的Node之后，向API Server发起一个绑定指示，申请将Pod与筛选出的Node进行绑定。

### 代码层级
回归到代码本身，scheduler的设计分为3个主要代码层级：
- `cmd/kube-scheduler/scheduler.go`: 这里的main()函数即是scheduler的入口，它会读取指定的命令行参数，初始化调度器框架，开始工作
- `pkg/scheduler/scheduler.go`: 调度器框架的整体代码，框架本身所有的运行、调度逻辑全部在这里
- `pkg/scheduler/core/generic_scheduler.go`: 上面是框架本身的所有调度逻辑，包括算法，而这一层，是调度器实际工作时使用的算法，默认情况下，并不是所有列举出的算法都在被实际使用，参考位于文件中的`Schedule()`函数

### 调度算法逻辑
逻辑图：
```
一个没有指定Spec.NodeName的:

    +---------------------------------------------+
    |               Schedulable nodes:            |
    |                                             |
    | +--------+    +--------+      +--------+    |
    | | node 1 |    | node 2 |      | node 3 |    |
    | +--------+    +--------+      +--------+    |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    断言(硬性指标)筛选: node 3 资源不足

    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+
    |             剩余可选nodes:                   |
    |   +--------+                 +--------+     |
    |   | node 1 |                 | node 2 |     |
    |   +--------+                 +--------+     |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    优先级判断:     node 1: priority=2
                   node 2: priority=5

    +-------------------+-------------------------+
                        |
                        |
                        v
            选择 max{node priority} = node 2
            node2则成为成功筛选出的与pod绑定的节点
```
为了给pod挑选出合适的node，调度器做出如下尝试步骤：
- 第一步，通过一系列的predicates(断言)指标，排除不合适的node，例如：pod.resources.requests.memory: 16Gi, node则计算：node.capacity 减去node上现有的所有pod的pod.resources.requests.memory的总和，如果差小于16Gi，那么则此项predicates结果为false，排除此节点
- 第二步，对通过了上一步筛选的node，执行一系列的优先级计算函数，计算的对象是node的负载情况，负载是即是node上现有的所有pod的pod.resources.requests的资源的总和除以node.capacity，值越高则负载越高，优先级越低
- 最终，挑选出了最高优先级的node，若有多个，则随机挑选其中一个

### Predicates and priorities policies
调度算法一共由Predicates和priorities这两部分组成，Predicates(断言)是用来过滤node的一系列策略集合，Priorities是用来优选node的一系列策略集合。默认情况下，kubernetes提供内建predicates/priorities策略，代码集中于`pkg/scheduler/algorithm/predicates/predicates.go` 和 `pkg/scheduler/algorithm/priorities`内.


### 调度策略扩展
管理员可以选择要应用的预定义调度策略中的哪一个，开发者也可以添加自定义的调度策略。

### 修改调度策略
默认调度策略是通过defaultPredicates() 和 defaultPriorities()这两个函数定义的，源码在 `pkg/scheduler/algorithmprovider/defaults/defaults.go`，我们可以通过命令行flag --policy-config-file CONFIG_FILE 来修改默认的调度策略。除此之外，也可以在`pkg/scheduler/algorithm/predicates/predicates.go` `pkg/scheduler/algorithm/priorities`源码中添加自定义的predicate和prioritie策略，然后注册到`defaultPredicates()`/`defaultPriorities()`中来实现自定义调度策略。

