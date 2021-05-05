## Controller源码分段阅读导航

- [启动流程](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/kubelet/Kubernetes源码学习-Kubelet-P1-启动流程篇.md)

- (待补充)

  

## 概述

Kubelet作为k8s核心组件中的daemon端运行在集群中的每一个节点上，承接着控制平面的指令向数据平面传达。不像scheduler、controller组件只负责相对单一的功能，kubelet除了管理自身的运行时外，还需要和宿主系统(linux)、CRI、CNI、CSI等外部组件对接，无疑是一个复杂度很高的组件。



## kubelet的主要功能

- pod启停
- 容器网络管理
- Volume管理
- 探针检查
- 容器监控

