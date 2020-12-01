## APIServer源码分段阅读导航



- [开胃菜-基础结构信息](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/apiServer/Kubernetes源码学习-APIServer-P1-基础结构信息.md)

- [启动流程](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/apiServer/Kubernetes源码学习-APIServer-P2-启动流程.md)

- [APIServer的认证机制](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/apiServer/Kubernetes源码学习-APIServer-P3-APIServer的认证控制.md)

- [APIServer的授权机制](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/apiServer/Kubernetes源码学习-APIServer-P4-APIServer的授权控制.md)

  



## 概述

APIServer提供了 k8s各类资源对象的CURD/watch、认证授权、准入控制等众多核心功能，在k8s中定位类似于大脑和心脏，它的功能包括：

- 提供了集群管理的REST API接口(包括资源CURD、认证授权、数据校验以及集群状态变更)；
- 是所有模块的数据交互和通信的枢纽，各模块的运作都依赖于APIServer

- 提供丰富多样的集群安全管控机制
- 直连后端存储(Etcd)，是唯一与存储后端直接通信的模块



如图所示，这是创建一个资源(Pod)实例过程中，控制层面所经过的调用过程：

![](https://mycloudn.upweto.top/20201126165154.png)

在这个过程中，控制层面的每个组件、每一步骤，都需要与APIServer交互。

因此，APIServer无疑是各模块中 **最复杂、定位最核心、涉及面最广、代码量最大 **的模块。



**To be honest，看这个模块真的是压力山大，不求深解，不钻牛角尖，目标是摸清大体结构~**



