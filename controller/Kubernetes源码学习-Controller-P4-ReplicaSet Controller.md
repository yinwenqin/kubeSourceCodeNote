# P4-ReplicaSet Controller

## 前言

在上一篇文章中，对deployment controller的工作模式进行了详细地分析:

[Controller-P3-Controller](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/controller/Kubernetes源码学习-Controller-P3-Controller分类与Deployment%20Controller.md)

分析后得知，deployment controller更多的是对每个相应版本的replicaset副本数进行管理，而不涉及直接对pod的管理，因此，承接上节，本章来分析replicaSet Controller的源码.



## ReplicaSet Controller

### 初始化

参照上节一样，直接来到各类controller初始化的函数:

`cmd/kube-controller-manager/app/controllermanager.go:343`

```go
controllers["replicaset"] = startReplicaSetController
```

==> `cmd/kube-controller-manager/app/apps.go:69`

```go
	go replicaset.NewReplicaSetController(
    // replicaSet controller只关注ReplicaSets和Pod这两种资源。
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
		replicaset.BurstReplicas,
	).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
```



### 创建ReplicaSetController

先来看看NewReplicaSetController创建的过程:

==> `pkg/controller/replicaset/replica_set.go:109`

```go
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
  // NewBaseController方法往下看
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}

// NewBaseController is the implementation of NewReplicaSetController with additional injected
// parameters so that it can also serve as the implementation of NewReplicationController.
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		metrics.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}

	rsc := &ReplicaSetController{
		GroupVersionKind: gvk,
		kubeClient:       kubeClient,
		podControl:       podControl,
		burstReplicas:    burstReplicas,
		expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
	}

	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    rsc.enqueueReplicaSet,
		UpdateFunc: rsc.updateRS,
		DeleteFunc: rsc.enqueueReplicaSet,
	})
	rsc.rsLister = rsInformer.Lister()
  // informer会同步待操作的资源到本地的queue中，HasSynced方法就是用来判断判断queue是否已同步的
	rsc.rsListerSynced = rsInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: rsc.addPod,
		UpdateFunc: rsc.updatePod,
		DeleteFunc: rsc.deletePod,
	})
	rsc.podLister = podInformer.Lister()
  // informer会同步待操作的资源到本地的queue中，HasSynced方法就是用来判断判断queue是否已同步的
	rsc.podListerSynced = podInformer.Informer().HasSynced

	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}
```

NewBaseController这里主要关注AddEventHandler为资源的informer增加的curd方法，例如pod相关的addPod、updatePod、deletePod方法。



### ReplicaSetController Run方法

接着往下，创建好ReplicaSetController对象后，看它的运行过程，即Run方法。

==> `pkg/controller/replicaset/replica_set.go:177`

```go
// Run begins watching and syncing.
func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)
	klog.Infof("Starting %v controller", controllerName)
	defer klog.Infof("Shutting down %v controller", controllerName)
  
  // 判断各个informer的缓存是否已经同步完毕的函数
	if !controller.WaitForCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}
  // worker的数量默认是5个，开启5个worker，每个worker间隔1s运行一次rsc.worker函数，来检查并收敛rs的状态
	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

来到了这里，可发现ReplicaSetController.Run()函数和上一节的DeploymentController.Run()函数非常地相似。所以，从这里开始，各类controller之间代码相似的步骤可能会跳过，不再每个地方都重复详细说明。

往上溯源，可以找到，worker的数量配置默认为5个，参见这里:

`pkg/controller/apis/config/v1alpha1/defaults.go:219`

```go
func SetDefaults_ReplicaSetControllerConfiguration(obj *kubectrlmgrconfigv1alpha1.ReplicaSetControllerConfiguration) {
	if obj.ConcurrentRSSyncs == 0 {
		obj.ConcurrentRSSyncs = 5
	}
}
```

wait.Until()函数是很有意思的，上节也做过仔细分析，可以再回顾一下这里:

[waituntil循环计时器函数](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/controller/Kubernetes源码学习-Controller-P3-Controller分类与Deployment Controller.md#waituntil循环计时器函数)

好，直接进入主题，开始分析rsc.worker工作函数.



### 工作逻辑

`pkg/controller/replicaset/replica_set.go:190`

```go
	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}		
```

==> `pkg/controller/replicaset/replica_set.go:432`

```go
// processNextWorkItem()函数的作用是把informer work queue工作队列里的对象取出，按照申明的要求来处理它们，标记它们。
func (rsc *ReplicaSetController) worker() {
	for rsc.processNextWorkItem() {
	}
}
```

==> `pkg/controller/replicaset/replica_set.go:437`

```go
func (rsc *ReplicaSetController) processNextWorkItem() bool {
  // work queue中取出队首元素
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)
  // syncHandler每一个队列对象，强保证同一时间只会有一个go协程处理它(无并发竞争)。所谓sync，意思是将work queue中待操作的对象，同步实现到运行环境中。
	err := rsc.syncHandler(key.(string))
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("Sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}
```

主要函数是这个**syncHandler**，接着追溯，可以在这里找到这个结构体属性函数的赋值：

`pkg/controller/replicaset/replica_set.go:163`

```go
// NewBaseController is the implementation of NewReplicaSetController with additional injected
// parameters so that it can also serve as the implementation of NewReplicationController.
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
  // ... 省略
	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}
```

接着便可以找到**ReplicaSetController.syncReplicaSet**函数：

`pkg/controller/replicaset/replica_set.go:562`

```go

// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {

	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
	}()
  // key的字符串格式是这样的: ${NAMESPACE}/${NAME}
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
  // 获取到rs对象
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return err
	}
  
  // 判断rs是否实现所声明的期望状态，这里SatisfiedExpectations是使用expectations机制来判断这个rs是否满足期望状态。
	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Error converting pod selector to selector: %v", err))
		return nil
	}

	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
  // 取出所有的的pod，labels.Everything()取到的是空selector，即不使用label selector，取全部pod
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
  
	// Ignore inactive pods.
  // 去除 inactive状态的pod
	filteredPods := controller.FilterActivePods(allPods)

  
  // 根据rs和selector来选择受此rs版本管理的pod
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}
  
  
	var manageReplicasErr error
  // 如果rs未达到期望状态，则对副本进行管理，以使rs满足声明的期望状态
	if rsNeedsSync && rs.DeletionTimestamp == nil {
    // 最重要的函数manageReplicas，未达期望时，管理rs对应的pod(新增/删除)
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
  
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)
  
  // 只要有对应pod的更新，则需要更新rs的status字段
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	if err != nil {
		// Multiple things could lead to this update failing. Requeuing the replica set ensures
		// Returning an error causes a requeue without forcing a hotloop
		return err
	}

  // 当指定了MinReadySeconds时，即使pod 已经是ready状态了，但也不会视为Available，需要等待MinReadySeconds后再来刷新rs的状态。因此，enqueueReplicaSetAfter方法，异步等待MinReadySeconds后，把该rs重新压入work queue队列中
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.enqueueReplicaSetAfter(updatedRS, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

划重点，两个重要的函数：**SatisfiedExpectations**(判断是否满足sync条件) / **manageReplicas**(sync后续的副本pod新增、删除操作)。分别来看看



#### SatisfiedExpectations函数

在此之前，必须先了解一下rs controller(后面简称rsc)的Expectations机制。rsc会将每一个rs的期望状态(比如期望新增3个副本)保存在本地缓存中，在sync执行之前，会对期望状态进行条件判断，满足条件才会真正进行sync操作。

来看看SatisfiedExpectations函数的逻辑：

`pkg/controller/controller_utils.go:181`

```go
func (r *ControllerExpectations) SatisfiedExpectations(controllerKey string) bool {
  // 若此key存在Expectations期望状态
	if exp, exists, err := r.GetExpectations(controllerKey); exists {
    // Expectations期望状态达成或者过期，则需要sync
		if exp.Fulfilled() {
			klog.V(4).Infof("Controller expectations fulfilled %#v", exp)
			return true
		} else if exp.isExpired() {
			klog.V(4).Infof("Controller expectations expired %#v", exp)
			return true
		} else {
      // 存在期望状态但未达成，则无需sync。因为后面的handler在处理资源增删的时候会来新建和修改Expectations，说明当前正在接近期望状态中，所以本次无需再sync
			klog.V(4).Infof("Controller still waiting on expectations %#v", exp)
			return false
		}
	} 
  // 不存在Expectations(新增的资源对象)，或者获取Expectations出错，则视为需要执行sync
    else if err != nil {
		klog.V(2).Infof("Error encountered while checking expectations %#v, forcing sync", err)
	} else {
		klog.V(4).Infof("Controller %v either never recorded expectations, or the ttl expired.", controllerKey)
	}

	return true
}
```



#### manageReplicas函数

==> `pkg/controller/replicaset/replica_set.go:459`

```go
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
   // rs当前管理的pod数量 与 rs声明指定pod的数量 的差量
   diff := len(filteredPods) - int(*(rs.Spec.Replicas))
   rsKey, err := controller.KeyFunc(rs)
   if err != nil {
      utilruntime.HandleError(fmt.Errorf("Couldn't get key for %v %#v: %v", rsc.Kind, rs, err))
      return nil
   }
   // 当 rs当前管理的pod数量 小于 rs声明指定pod的数量 时，说明应该继续增加pod
   if diff < 0 {
      diff *= -1
      // 每次新增数量以突发增加数量burstReplicas为上限
      if diff > rsc.burstReplicas {
         diff = rsc.burstReplicas
      }
      // 创建ExpectCreations期望
      rsc.expectations.ExpectCreations(rsKey, diff)
      klog.V(2).Infof("Too few replicas for %v %s/%s, need %d, creating %d", rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff)
     
      // slowStartBatch用来以指数级批量启动pod, 其中controller.SlowStartInitialBatchSize默认值为1，作为底数。
      successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
         // 创建单个pod的函数 CreatePodsWithControllerRef
         err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
         if err != nil && errors.IsTimeout(err) {

            return nil
         }
         return err
      })


      if skippedPods := diff - successfulCreations; skippedPods > 0 {
         klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for %v %v/%v", skippedPods, rsc.Kind, rs.Namespace, rs.Name)
         for i := 0; i < skippedPods; i++ {
            // Decrement the expected number of creates because the informer won't observe this pod
            rsc.expectations.CreationObserved(rsKey)
         }
      }
      return err
     
     // 当 rs当前管理的pod数量 大于 rs声明指定pod的数量 时，说明应该减少pod
   } else if diff > 0 {
      if diff > rsc.burstReplicas {
         diff = rsc.burstReplicas
      }
      klog.V(2).Infof("Too many replicas for %v %s/%s, need %d, deleting %d", rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff)

      // 获取需要删除的pod
      podsToDelete := getPodsToDelete(filteredPods, diff)
      // 修改rs的期望状态，在期望中剔除将要删除的pod
      rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))

      errCh := make(chan error, diff)
      var wg sync.WaitGroup
      wg.Add(diff)
      // 并发删除目标pod
      for _, pod := range podsToDelete {
         go func(targetPod *v1.Pod) {
            defer wg.Done()
            if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
               // Decrement the expected number of deletes because the informer won't observe this deletion
               podKey := controller.PodKey(targetPod)
               klog.V(2).Infof("Failed to delete %v, decrementing expectations for %v %s/%s", podKey, rsc.Kind, rs.Namespace, rs.Name)
               rsc.expectations.DeletionObserved(rsKey, podKey)
               errCh <- err
            }
         }(pod)
      }
      wg.Wait()

      select {
      case err := <-errCh:
         // all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
         if err != nil {
            return err
         }
      default:
      }
   }

   return nil
}
```

这个函数即是实际操控管理pod副本数量的函数，其中的slowStartBatch批量启动pod的功能比较有意思，来看看。



#### 批量启动pod

`pkg/controller/replicaset/replica_set.go:658`

```go
func slowStartBatch(count int, initialBatchSize int, fn func() error) (int, error) {
   // 剩余要执行的数量
   remaining := count
   // 累计成功执行的数量
   successes := 0
  // batchSize是每次批量执行的数量，从initialBatchSize(1)和剩余数量中取最小值。每次循环执行成功后，batchSize乘以2，以指数级扩充。
   for batchSize := integer.IntMin(remaining, initialBatchSize); batchSize > 0; batchSize = integer.IntMin(2*batchSize, remaining) {
      errCh := make(chan error, batchSize)
      var wg sync.WaitGroup
      wg.Add(batchSize)
      for i := 0; i < batchSize; i++ {
         go func() {
            defer wg.Done()
            if err := fn(); err != nil {
               errCh <- err
            }
         }()
      }
      wg.Wait()
      curSuccesses := batchSize - len(errCh)
      successes += curSuccesses
      // 某一轮循环出错时，跳出循环，后续的不再执行。
      if len(errCh) > 0 {
         return successes, <-errCh
      }
      remaining -= batchSize
   }
   return successes, nil
}
```



### ReplicaSetController工作流程总结

总结一下，在出现新版本的rs后，rsc按照以下步骤进行工作：

1.通过SatisfiedExpectations函数，发现expectations期望状态本地缓存中不存在此rs key，因此返回true，需要sync

2.通过manageReplicas管理pod，新增或删除

3.判断pod副本数是多了还是少了，多则要删，少则要增

4.增删之前创建expectations对象并设置add / del值

5.slowStartBatch新增 / 并发删除 pod

6.更新expection

expections缓存机制，在运行的pod副本数在向声明指定的副本数收敛之时，很好地避免了频繁的informer数据查询，以及可能随之而来的数据更新不及时的问题，这个机制设计巧妙贯穿整个rsc工作过程，也是不太易于理解之处。