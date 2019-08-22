---
title: "Kubernetes源码学习-Scheduler-P1-调度器入口篇"
date: 2019/08/05 16:27:53
tags: 
- Kubernetes
- Golang
- 读源码



---

## 

# 调度器入口

## 前言
本篇介绍scheduler的初始化相关逻辑

## 入口之前
入口函数是位于`cmd/kube-scheduler/scheduler.go`中的main()方法,调用的是app.NewSchedulerCommand()方法，跳转至此方法，可以看到函数上方的注释：

```
// NewSchedulerCommand creates a *cobra.Command object with default parameters
func NewSchedulerCommand() *cobra.Command {
    ...
}
```
NewSchedulerCommand创建的是一个cobra.Command对象，后续的命令行处理相关功能都是借助cobra来实现的，那么继续往下之前，为了避免从入口开始就一脸懵，有必要了解一下cobra这个工具

## cobra
#### 什么是cobra?
github主页: https://github.com/spf13/cobra
主页的介绍是: Cobra是一个强大的用于创建现代化CLI命令行程序的库，用于生成应用程序和命令文件。众多高知名度的项目采用了它，例如我们熟悉的kubernetes和docker
cobra创建的程序CLI遵循的模式是: `APPNAME COMMAND ARG --FLAG`，与常见的其他命令行程序一样，例如git: `git clone URL --bare`

#### 安装：

```
#最简单的安装方式，但毫无意外，事情并没有那么简单，我们的网络的问题，导致无法正常安装依赖，
go get -u github.com/spf13/cobra/cobra

#怎么办呢？先进入GOPATH中，手动安装报错缺失的两个依赖:
cd /Users/ywq/go/
mkdir -p src/golang.org/x
cd golang.org/x
git clone https://github.com/golang/text.git
git clone https://github.com/golang/sys.git

#然后执行:
go install github.com/spf13/cobra/cobra
matebook-x-pro:x ywq$ ls /Users/ywq/go/bin/cobra
/Users/ywq/go/bin/cobra
#安装完毕,记得把GOBIN加入PATH环境变量哦,否则无法直接运行cobra命令
```

#### 简单试用cobra:
```
matebook-x-pro:local ywq$ cd /Users/ywq/go/src/local/
matebook-x-pro:local ywq$ cobra init testapp --pkg-name=local/testapp
matebook-x-pro:local ywq$ ls
testapp
matebook-x-pro:local ywq$ ls testapp/
LICENSE  cmd/     main.go
matebook-x-pro:local ywq$ ls testapp/cmd/
root.go
matebook-x-pro:local ywq$ cd testapp
matebook-x-pro:local ywq$ go run main.go 
# 报错：subcommand is required，要求提供子命令
# 因需要多次测试，这里所有的测试步骤就把build的步骤跳过，直接使用go run main.go进行测试
```
**我们打开IDE来查看一下testapp的代码结构:**
![image](http://pwh8f9az4.bkt.clouddn.com/cobra1.jpg)
![image](http://pwh8f9az4.bkt.clouddn.com/cobra2.jpg)

```
# 现在还未创建子命令，那么来创建几个试试:
matebook-x-pro:testapp ywq$ cobra add get
get created at /Users/ywq/go/src/local/testapp
matebook-x-pro:testapp ywq$ cobra add delete
delete created at /Users/ywq/go/src/local/testapp
matebook-x-pro:testapp ywq$ cobra add add
add created at /Users/ywq/go/src/local/testapp
matebook-x-pro:testapp ywq$ cobra add update
matebook-x-pro:testapp ywq$ ls cmd/
add.go		delete.go	get.go		root.go		update.go

# 查看help，可以发现刚添加的子命令已经加入提示并可用了
matebook-x-pro:testapp ywq$ go run main.go -h
...

Available Commands:
  add         A brief description of your command
  delete      A brief description of your command
  get         A brief description of your command
  help        Help about any command
  update      A brief description of your command

# 调用子命令试试:
matebook-x-pro:testapp ywq$ go run main.go get
get called
matebook-x-pro:testapp ywq$ go run main.go add
add called
```

**来看看新增的子命令是怎么运行的呢？**
![image](http://pwh8f9az4.bkt.clouddn.com/cobra3.jpg)
截图圈中部分可以看出，子命令是在init()函数里为root级添加了一个子命令，先不去管底层实现，接着往下.

**测试cobra的强大简洁的flag处理**
我们在`cmd/delete.go`的init()函数中，定义一个flag处理配置:
```
var obj string
deleteCmd.PersistentFlags().StringVar(&obj,"object", "", "A function to delete an test object")
```
在`Run:func()`匿名函数中添加一行输出:
`fmt.Println("delete obj:",cmd.Flag("object").Value)`
![image](http://pwh8f9az4.bkt.clouddn.com/cobra4.jpg)

运行结果:

```
matebook-x-pro:testapp ywq$ go run main.go delete --object obj1
delete called
delete obj: obj1

```
如果觉得`--`flag符号太麻烦，cobra同样支持短符号`-`flag缩写:
![image](http://pwh8f9az4.bkt.clouddn.com/cobra5.jpg)

运行结果:

```
matebook-x-pro:testapp ywq$ go run main.go delete -o obj1
delete called
delete obj: obj1

```

这里只是两级命令加flag,但我们常见的，例如(kubectl delete pod xxx)，是有3级命令 + args的，怎么再多添加一级子命令呢？cobra帮你一条命令实现

```
matebook-x-pro:testapp ywq$ cobra add pods -p deleteCmd  # -p为父级命令，默认其名称格式为(parentCommandName)Cmd
matebook-x-pro:testapp ywq$ ls cmd
add.go          delete.go       get.go          pods.go         root.go         update.go

```
可以发现,cmd/目录下多了一个pods.go文件，我们来看看它是怎么关联上delete父级命令的,同时为它添加一行输出:
![image](http://pwh8f9az4.bkt.clouddn.com/cobra6.jpg)
执行命令:

```
matebook-x-pro:testapp ywq$ go run main.go delete pods pod1
pods called
delete pods: pod1

```

#### 看到这里，相信对cobra的强大简洁已经有了初步的认知，建议自行进入项目主页了解详情并进行安装测试

## 入口
通过对上方cobra的基本了解，我们不难知道，`cmd/kube-scheduler/scheduler.go`内的main()方法内部实际调用的是`cobra.Command.Run`内的匿名函数，我们可以进入`NewSchedulerCommand()`内部确认:
![image](http://pwh8f9az4.bkt.clouddn.com/main1.jpg)

可以看到，调用了`Run`内部`runCommand`方法，再来看看Run方法内部需要重点关注的几个点：
![image](http://pwh8f9az4.bkt.clouddn.com/runCommand.jpg)

其中，上方是对命令行的参数、选项校验的步骤，跳过，重点关注两个变量:`cc和stopCh`，这两个变量会作为最后调用`Run()`方法的参数，其中`stopCh`作用是作为主程序退出的信号通知其他各协程进行相关的退出操作的，另外一个cc变量非常重要，可以点击`c.Complete()`方法，查看该方法的详情：
![image](http://pwh8f9az4.bkt.clouddn.com/runCommand.jpg)
`Complete()`方法本质上返回的是一个Config结构体，该结构体内部的元素非常丰富，篇幅有限就不一一点开截图了，大家可以自行深入查看这些元素的作用，这里简单概括一下其中几个:

```
// scheduler 本身相关的配置都集中于此，例如名称、调度算法、pod亲和性权重、leader选举机制、metric绑定地址，健康检查绑定地址，绑定超时时间等等
ComponentConfig kubeschedulerconfig.KubeSchedulerConfiguration

// 这几个元素都是与apiserver认证授权相关的
InsecureServing        *apiserver.DeprecatedInsecureServingInfo // nil will disable serving on an insecure port
InsecureMetricsServing *apiserver.DeprecatedInsecureServingInfo // non-nil if metrics should be served independently
Authentication         apiserver.AuthenticationInfo
Authorization          apiserver.AuthorizationInfo
SecureServing          *apiserver.SecureServingInfo

// Clientset.Interface内部封装了向apiServer所支持的所有apiVersion(apps/v1beta2,extensions/v1beta1...)之下的resource(pod/deployment/service...)发起查询请求的功能
Client          clientset.Interface

// 这几个元素都是与Event资源相关的，实现rest api处理以及记录、通知等功能
EventClient     v1core.EventsGetter
Recorder        record.EventRecorder
Broadcaster     record.EventBroadcaster
```
这里层级非常深，不便展示，Config这一个结构体非常重要，可以认真读一读代码。回到`cmd/kube-scheduler/app/server.go`.`runCommand`这里来,接着往下，进入其最后return调用的`Run()`函数中，函数中的前部分都是启动scheduler相关的组件，如event broadcaster、informers、healthz server、metric server等，重点看图中红框圈出的`sched.Run()`,这才是scheduler主程序的调用运行函数:
![image](http://pwh8f9az4.bkt.clouddn.com/Run.jpg)

进入`sched.Run()`:
![image](http://pwh8f9az4.bkt.clouddn.com/scheRun.jpg)

`wait.Until`这个调用的逻辑是，直到收到stop信号才终止，在此之前循环运行`sched.scheduleOne`。代码走到这里，终于找到启动入口最内部的主体啦:
![image](http://pwh8f9az4.bkt.clouddn.com/scheduleOne.jpg)

`sched.scheduleOne`这个函数有代码点长，整体的功能可以概括为:获取需调度的pod、寻找匹配node、发起绑定到node请求、绑定检查等一系列操作.

#### 本篇入口篇到这里就先告一段落，下一篇开始阅读学习调度过程的逻辑！
