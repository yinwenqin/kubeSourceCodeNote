---
title: "Kubernetes源码学习-Controller-P1-多实例leader选举.md"
date: 2019/12/09 21:59:53
tags:
- Kubernetes
- Golang
- 读源码


---

# P1-多实例leader选举.md



## 前言

Kubernetes多master场景下，核心组件都是以一主多从的模式来运行的，在前面scheduler部分的文章中，并没有分析其主从选举及工作的流程，那么在本篇中，以controller为例，单独作一篇分析组件之间主从工作模式。

## 入口

如scheduler一样，controller的cmd启动也是借助的cobra，对cobra不了解可以回到前面的文章中查看，这里不再赘述，直接顺着入口找到启动函数：

==> `cmd/kube-controller-manager/controller-manager.go:38`

 `command := app.NewControllerManagerCommand()` 

==> `cmd/kube-controller-manager/app/controllermanager.go:109`

`Run(c.Complete(), wait.NeverStop)`

==> `cmd/kube-controller-manager/app/controllermanager.go:153`

`func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {}`

入口函数就在这里，代码块中已分段注释：

```go
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
  ...
	// 篇幅有限，省略部分代码
  
  // 启动kube-controller的http服务
	// Start the controller manager HTTP server
	// unsecuredMux is the handler for these controller *after* authn/authz filters have been applied
	var unsecuredMux *mux.PathRecorderMux
	if c.SecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, checks...)
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, &c.Authorization, &c.Authentication)
		// TODO: handle stoppedCh returned by c.SecureServing.Serve
		if _, err := c.SecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
	if c.InsecureServing != nil {
		unsecuredMux = genericcontrollermanager.NewBaseHandler(&c.ComponentConfig.Generic.Debugging, checks...)
		insecureSuperuserAuthn := server.AuthenticationInfo{Authenticator: &server.InsecureSuperuser{}}
		handler := genericcontrollermanager.BuildHandlerChain(unsecuredMux, nil, &insecureSuperuserAuthn)
		if err := c.InsecureServing.Serve(handler, 0, stopCh); err != nil {
			return err
		}
	}
  // 启动controller工作的run函数，特别标注，会作为回调函数在leader选举成功后执行
	run := func(ctx context.Context) {
		rootClientBuilder := controller.SimpleControllerClientBuilder{
			ClientConfig: c.Kubeconfig,
		}
		var clientBuilder controller.ControllerClientBuilder
		if c.ComponentConfig.KubeCloudShared.UseServiceAccountCredentials {
			if len(c.ComponentConfig.SAController.ServiceAccountKeyFile) == 0 {
				// It'c possible another controller process is creating the tokens for us.
				// If one isn't, we'll timeout and exit when our client builder is unable to create the tokens.
				klog.Warningf("--use-service-account-credentials was specified without providing a --service-account-private-key-file")
			}
			clientBuilder = controller.SAControllerClientBuilder{
				ClientConfig:         restclient.AnonymousClientConfig(c.Kubeconfig),
				CoreClient:           c.Client.CoreV1(),
				AuthenticationClient: c.Client.AuthenticationV1(),
				Namespace:            "kube-system",
			}
		} else {
			clientBuilder = rootClientBuilder
		}
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		if err != nil {
			klog.Fatalf("error building controller context: %v", err)
		}
		saTokenControllerInitFunc := serviceAccountTokenControllerStarter{rootClientBuilder: rootClientBuilder}.startServiceAccountTokenController
		
		if err := StartControllers(controllerContext, saTokenControllerInitFunc, NewControllerInitializers(controllerContext.LoopMode), unsecuredMux); err != nil {
			klog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(controllerContext.Stop)
		close(controllerContext.InformersStarted)

		select {}
	}

	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		run(context.TODO())
		panic("unreachable")
	}

	id, err := os.Hostname()
	if err != nil {
		return err
	}

	// add a uniquifier so that two processes on the same host don't accidentally both become active
	id = id + "_" + string(uuid.NewUUID())
	rl, err := resourcelock.New(c.ComponentConfig.Generic.LeaderElection.ResourceLock,
		"kube-system",
		"kube-controller-manager",
		c.LeaderElectionClient.CoreV1(),
		c.LeaderElectionClient.CoordinationV1(),
		resourcelock.ResourceLockConfig{
			Identity:      id,
			EventRecorder: c.EventRecorder,
		})
	if err != nil {
		klog.Fatalf("error creating lock: %v", err)
	}
  
	// 主从选举从这里开始
	leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: c.ComponentConfig.Generic.LeaderElection.LeaseDuration.Duration,
		RenewDeadline: c.ComponentConfig.Generic.LeaderElection.RenewDeadline.Duration,
		RetryPeriod:   c.ComponentConfig.Generic.LeaderElection.RetryPeriod.Duration,
		Callbacks: leaderelection.LeaderCallbacks{
      // 回调函数，选举成功后，主工作节点开始运行上方的工作run函数
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		},
		WatchDog: electionChecker,
		Name:     "kube-controller-manager",
	})
	panic("unreachable")
}
```

从这里可以看到，选举成为主领导节点后，才会进入工作流程，先跳过具体的工作流程，来看看leaderelection的选举过程

## 选举

#### 选举入口

==> `cmd/kube-controller-manager/app/controllermanager.go:252` 

`leaderelection.RunOrDie(context.TODO(), leaderelection.LeaderElectionConfig{}`

```go
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
   le, err := NewLeaderElector(lec)
   if err != nil {
      panic(err)
   }
   // 加载检查leader健康状态的http接口
   if lec.WatchDog != nil {
      lec.WatchDog.SetLeaderElection(le)
   }
   // 开始进入选举
   le.Run(ctx)
}
```

==> `vendor/k8s.io/client-go/tools/leaderelection/leaderelection.go:196`

`le.Run(ctx)`

```go
// Run starts the leader election loop
func (le *LeaderElector) Run(ctx context.Context) {
   defer func() {
      runtime.HandleCrash()
      le.config.Callbacks.OnStoppedLeading()
   }()
   // 1.acquire是竞选函数，如果选举执行失败直接返回
   if !le.acquire(ctx) {
      return // ctx signalled done
   }
   ctx, cancel := context.WithCancel(ctx)
   defer cancel()
   // 2.竞选成功则另起一个线程，执行上面特别标注的run工作函数，即controller的工作循环
   go le.config.Callbacks.OnStartedLeading(ctx)
   // 3.刷新leader状态函数
   le.renew(ctx)
}
```

 这个函数里包含多个defer和return，这里额外备注一下defer和return的执行先后顺序：

```
1.多个defer是以栈结构保存的，后入先出，下文的defer先执行
2.return在defer之后执行
3.触发return条件后，return上下文的所有defer中，下文的defer不会被执行
```

这个函数这里，大概可以看出选举执行的逻辑：

1.选举成功者，开始执行run()函数，即controller的工作函数。同时提供leader状态健康检查的api

2.选举失败者，会结束选举程序。但watchDog会持续运行，监测leader的健康状态

3.选举成功者，在之后会持续刷新自己的leader状态信息



#### 竞选函数：

`vendor/k8s.io/client-go/tools/leaderelection/leaderelection.go:212`

```go
// acquire loops calling tryAcquireOrRenew and returns true immediately when tryAcquireOrRenew succeeds.
// Returns false if ctx signals done.
// 选举者开始循环执行申请，若申请leader成功则返回true，若申请leader失败则进入循环状态，每间隔一段时间再申请一次

func (le *LeaderElector) acquire(ctx context.Context) bool {
   ctx, cancel := context.WithCancel(ctx)
   defer cancel()
   succeeded := false
   desc := le.config.Lock.Describe()
   klog.Infof("attempting to acquire leader lease  %v...", desc)
   // 进入循环申请leader状态，JitterUntil是一个定时循环功能的函数
   wait.JitterUntil(func() {
      // 申请或刷新leader函数
      succeeded = le.tryAcquireOrRenew()
      le.maybeReportTransition()
      if !succeeded {
         klog.V(4).Infof("failed to acquire lease %v", desc)
         return
      }
      le.config.Lock.RecordEvent("became leader")
      le.metrics.leaderOn(le.config.Name)
      klog.Infof("successfully acquired lease %v", desc)
     // 选举成功后，执行cancel()从定时循环函数中跳出来，返回成功结果
      cancel()
   }, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
   return succeeded
}
```



#### 定时执行函数

来看下定时循环函数JitterUntil的代码：

`vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:130`

```go
func JitterUntil(f func(), period time.Duration, jitterFactor float64, sliding bool, stopCh <-chan struct{}) {
	var t *time.Timer
	var sawTimeout bool

	for {
		select {
		case <-stopCh:
			return
		default:
		}

		jitteredPeriod := period
		if jitterFactor > 0.0 {
			jitteredPeriod = Jitter(period, jitterFactor)
		}
    // sliding代表是否将f()的执行时间计算在间隔之内
    // 若执行间隔将f()的执行时间包含在内，则在f()开始之前就启动计时器
		if !sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		func() {
			defer runtime.HandleCrash()
			f()
		}()
    
    // 若执行间隔不将f()的执行时间包含在内，则在f()执行完成之后再启动计时器
		if sliding {
			t = resetOrReuseTimer(t, jitteredPeriod, sawTimeout)
		}

		// 在这里，select的case没有优先级之分，因此，可能跳过stop判断，所以，在for loop的前面，也加入了一次stop判断，防止重复执行。
		select {
		case <-stopCh:
			return
    // 到达
		case <-t.C:
			sawTimeout = true
		}
	}
}

// resetOrReuseTimer avoids allocating a new timer if one is already in use.
// Not safe for multiple threads.
func resetOrReuseTimer(t *time.Timer, d time.Duration, sawTimeout bool) *time.Timer {
	if t == nil {
		return time.NewTimer(d)
	}
  // timer首次启动时，先将t.C channel内的值都取出来，避免channel消费方hang住
	if !t.Stop() && !sawTimeout {
		<-t.C
	}
  // 定时器重置
	t.Reset(d)
	return t
}

```

k8s定时任务用的是非常原生的time.timer()来实现的，t.C本质上还是一个channel struct {}，消费方运用select来触发到达指定计时间隔后，消费消息，进入下一次循环。

这里关于select结合channel的用法说明进行以下备注：

```
在select中，代码逻辑执行步骤如下：
1.检查每个case代码块
2.如果存在一个case代码块下有数据产生，执行对应case下的内容
3.如果多个case代码块下有数据产生，随机选取一个case并执行对应内容，无优先级之分
4.如果有default代码块，在没有任何case产生数据时，执行default代码块对应内容
5.如果default之后的代码为空，此时也没有任何case产生数据，则跳出select继续执行下文
6.如果任何一个case代码块都没有数据产生或代码上下文，同时也没有default，则select阻塞等待
```

关于go time.Timer，这里有一篇文章讲得很好：

https://tonybai.com/2016/12/21/how-to-use-timer-reset-in-golang-correctly/

#### 申请/刷新leader函数

`vendor/k8s.io/client-go/tools/leaderelection/leaderelection.go:293`

```go
// tryAcquireOrRenew tries to acquire a leader lease if it is not already acquired,
// else it tries to renew the lease if it has already been acquired. Returns true
// on success else returns false.
// 在初次选举、后续间隔刷新状态 这两处地方都会调用这个函数
// 如果参选者不是leader则尝试选举，如果已经是leader，则尝试续约租期，最后刷新信息
func (le *LeaderElector) tryAcquireOrRenew() bool {
   now := metav1.Now()
   leaderElectionRecord := rl.LeaderElectionRecord{
      HolderIdentity:       le.config.Lock.Identity(),
      LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
      RenewTime:            now,
      AcquireTime:          now,
   }

   // 1. obtain or create the ElectionRecord 
   // 第1步：获取当前的leader的竞选记录，如果当前还没有leader记录，则创建
   // 首先获取当前的leader记录
   oldLeaderElectionRecord, err := le.config.Lock.Get()
   if err != nil {
      if !errors.IsNotFound(err) {
         klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
         return false
      }
      if err = le.config.Lock.Create(leaderElectionRecord); err != nil {
         klog.Errorf("error initially creating leader election record: %v", err)
         return false
      }
      le.observedRecord = leaderElectionRecord
      le.observedTime = le.clock.Now()
      return true
   }
	 // 第2步，对比观察记录里的leader与当前实际的leader
   // 2. Record obtained, check the Identity & Time
   if !reflect.DeepEqual(le.observedRecord, *oldLeaderElectionRecord) {
     // 如果参选者的上一次观察记录中的leader，不是当前leader，则修改记录，以当前leader为准
      le.observedRecord = *oldLeaderElectionRecord
      le.observedTime = le.clock.Now()
   }
   if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
  		// 如果参选者不是当前的leader，且当前leader的任期尚未结束，则返回false，参选者选举失败
      le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
      !le.IsLeader() {
      klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
      return false
   }

   // 3. We're going to try to update. The leaderElectionRecord is set to it's default
   // here. Let's correct it before updating.
   if le.IsLeader() {
     // 如果参选者就是当前的leader本身，则修改记录里的当选时间变为它此前的当选时间，而不是本次时间，变更次数维持不变
      leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
      leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
   } else {
     // 如果参选者不是leader(则说明当前leader在任期已经结束，但并未续约)，则当前参选者变更为新的leader
      leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
   }

   // update the lock itself
   // 更新leader信息，更新leader锁，返回true选举过程顺利完成
   if err = le.config.Lock.Update(leaderElectionRecord); err != nil {
      klog.Errorf("Failed to update lock: %v", err)
      return false
   }
   le.observedRecord = leaderElectionRecord
   le.observedTime = le.clock.Now()
   return true
}
```

这一段代码中有多个leader记录信息相关的变量，很容易混淆，为了便于理解这里抽出来说明下：

```go
LeaderElector  # 参选者，每一个controller进程都会参与leader选举
oldLeaderElectionRecord  # 本次选举开始前，leader锁中记载的当前leader
leaderElectionRecord  # 本次选举的leader记录，最终会更新进入新的leader锁中
observedRecord  # 每个参选者都会定期观察当前的leader信息，记录在自身的这个字段中
```

先来看第1步中是怎么获取当前leader记录的：

`vendor/k8s.io/client-go/tools/leaderelection/resourcelock/leaselock.go:39`

```go
// Get returns the election record from a Lease spec
func (ll *LeaseLock) Get() (*LeaderElectionRecord, error) {
   var err error
  // 1.取得lease对象
   ll.lease, err = ll.Client.Leases(ll.LeaseMeta.Namespace).Get(ll.LeaseMeta.Name, metav1.GetOptions{})
   if err != nil {
      return nil, err
   }
  // 2.将lease.spec转为LeaderElectionRecord记录并返回
   return LeaseSpecToLeaderElectionRecord(&ll.lease.Spec), nil
}
```

取得lease对象的方法在这里：

`vendor/k8s.io/client-go/kubernetes/typed/coordination/v1/lease.go:66`

`func (c *leases) Get(name string, options metav1.GetOptions) (result *v1.Lease, err error) {}`

转换并返回的LeaderElectionRecord结构体是这样的：

```go
LeaderElectionRecord{
   HolderIdentity:       holderIdentity,   // leader持有标识
   LeaseDurationSeconds: leaseDurationSeconds,  // 选举间隔
   AcquireTime:          metav1.Time{spec.AcquireTime.Time},  // 选举成为leader的时间
   RenewTime:            metav1.Time{spec.RenewTime.Time},  // 续任时间
   LeaderTransitions:    leaseTransitions,  // leader位置的转接次数
}
```

对返回的LeaderElectionRecord进行对比，如果是自身，则续约，如果不是自身，则看leader是否过期，对leader lock信息相应处理。

#### 刷新选举状态函数

`vendor/k8s.io/client-go/tools/leaderelection/leaderelection.go:234`

```go
func (le *LeaderElector) renew(ctx context.Context) {
   ctx, cancel := context.WithCancel(ctx)
   defer cancel()
   wait.Until(func() {
      timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
      defer timeoutCancel()
      // 间隔刷新leader状态，成功则续约，不成功则释放
      err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
         done := make(chan bool, 1)
         go func() {
            defer close(done)
            done <- le.tryAcquireOrRenew()
         }()

         select {
         case <-timeoutCtx.Done():
            return false, fmt.Errorf("failed to tryAcquireOrRenew %s", timeoutCtx.Err())
         case result := <-done:
            return result, nil
         }
      }, timeoutCtx.Done())

      le.maybeReportTransition()
      desc := le.config.Lock.Describe()
      if err == nil {
         klog.V(5).Infof("successfully renewed lease %v", desc)
         return
      }
      le.config.Lock.RecordEvent("stopped leading")
      le.metrics.leaderOff(le.config.Name)
      klog.Infof("failed to renew lease %v: %v", desc, err)
      cancel()
   }, le.config.RetryPeriod, ctx.Done())

   // if we hold the lease, give it up
   if le.config.ReleaseOnCancel {
      le.release()
   }
}
```

tryAcquireOrRenew()和循环间隔执行函数同上面所讲基本一致，这里就不再说明了。

## 总结

组件选举大致可以概括为以下流程：

- 初始时，各实例均为LeaderElector，最先开始选举的，成为leader，成为工作实例。同时它会维护一份信息(leader lock)供各个LeaderElector探测，包括状态信息、健康监控接口等。

- 其余LeaderElector，进入热备状态，监控leader的运行状态，异常时会再次参与选举

- leader在运行中会间隔持续刷新自身的leader状态。

不止于controller，其余的几个组件，主从之间的工作关系也应当是如此。

**感谢阅读，欢迎指正**