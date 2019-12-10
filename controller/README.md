---
title: "Kubernetes源码学习-Controller-总览篇"
date: 2019/12/05 19:15:49
tags: 
- Kubernetes
- Golang
- 读源码


---

##Controller源码分段阅读导航

- [多实例leader选举]()
- [Informer工作流程]()
- [Deployment Controller]()
- [StafulSet Controller]()
- 待补充

## 概述

### kube-controller的作用

**引述**

首先依照惯例，贴两篇官方对 　  于controller的设计逻辑和运行机制的说明文档：

https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md

https://github.com/kubernetes/community/blob/master/contributors/design-proposals/apps/controller_history.md



**Controller是做什么用的？**

Controller通过watch apiServer，循环地观察监控着某些特定的资源对象，获取它们当前的状态，对它们进行对比、修正、收敛，来使这些对象的状态不断靠近、直至达成在它们的声明语义中所期望的目标状态，这即是controller的作用。

**伪代码**

一个简单的controller工作模式的伪代码示例：

```go
for {
  // 期望状态
  desired := getDesiredState()
  // 当前状态
  current := getCurrentState()
  // 使当前状态达到期望状态
  makeChanges(desired, current)
}
```

### kube-controller的组成

kube-controller是一个控制组件，根据我们的使用经验，有多种经常使用的资源，都不是实际地直接进行任务计算的资源类型，而在申明之后由k8s自动发现并保证以达成申明语义状态的逻辑资源，例如deployment、statefulSet、pvc、endpoint等，这些资源都分别由对应的controller子组件，那么这样的controller子组件有多少呢？如下图：

![](http://mycloudn.kokoerp.com/20191206111312.jpg)

可见controller的组件数量是非常之多的，因此在本部分中计划只抽选其中的deploymentController和statefulSetController这两种常见的对pod管理类型资源对应的controller来进行源码分析。

### 代码结构

**启动入口**

`kubernetes/src/k8s.io/kubernetes/cmd/kube-controller-manager/controller-manager.go`

![](http://mycloudn.kokoerp.com/20191206153026.jpg)

**功能模块**

`kubernetes/src/k8s.io/kubernetes/pkg/controller`

![](http://mycloudn.kokoerp.com/20191206154051.jpg)

