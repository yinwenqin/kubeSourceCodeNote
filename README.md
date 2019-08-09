## 前言
在熟悉kubernetes及常用组件、插件的管理使用后，总还觉得差了些什么，不够通透，是时候来读一读源码了，结合代码与实际使用场景来互相印证，有助于对kubernetes的理解更为透彻。这里将会分多篇介绍kubernetes各核心组件的工作模式、调度管理算法等。


## 版本

Kubernetes v1.14.3 ,最新部署的一套环境是此版本，代码版本保持一致，方便后续测试调试

## 核心组件
- [Scheduler](https://github.com/yinwenqin/kubeSourceCodeNote/tree/master/scheduler)
- Controller
- 待补充
- 

## 环境准备
golang(安装步骤略，网上很多),我的环境如下:

```
GOPATH="/Users/ywq/go"
go version go1.11.6 darwin/amd64
```
注意:本系列文档中,GOPATH路径`/Users/ywq/go/`,在自己的环境下请替换成相应本地的GOPATH，这一点后面不再说明

```
#拉取kubernetes项目源码:
cd /Users/ywq/go/
mkdir -p src/k8s.io
cd src/k8s.io
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
git checkout v1.14.3  # 切换到tag为v1.14.3的版本
```

源码拉取完成后，用自己的IDE打开，准备工作完毕.


## Tips
md文档已同步至github,welcome star:
https://github.com/yinwenqin/kubeSourceCodeNote

Kubernetes这一整个项目颇为庞大，一般情况下，如果熟悉kubernetes的应用，结合应用来理解源码的设计理念会容易许多，因此，对其应用不熟悉的朋友，不建议直接阅读源码。另外，目前处于边阅读代码边输出总结的阶段，如文中有误，请予以指正，非常感谢！

在此立一个flag，源码干货总结笔记，至少做到周不断更！



### 参考
1.官方开发者向导md文档: https://github.com/kubernetes/community/tree/master/contributors/devel

2.http://hutao.tech/k8s-source-code-analysis/

 
