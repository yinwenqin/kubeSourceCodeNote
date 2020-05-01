# P5-StatefulSet Controller

## 前言

在前面的几篇文章中，先对deployment controller进行了初步分析:

[Controller-P3-Deployment Controller](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/controller/Kubernetes源码学习-Controller-P3-Controller分类与Deployment%20Controller.md)

严格来讲deployment的管理pod的逻辑是基于replicaSet来实现的，因此接下来结合replicaSet controller进行了深入:

[Controller-P3-ReplicaSet Controller](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/controller/Kubernetes源码学习-Controller-P4-ReplicaSet%20Controller.md)

那么在本篇，来看看另一个最常用的承载在pod之上的管理单位的控制器实现: **StatefulSet Controller**



## StatefulSet 的基本特性

在看代码之前，先回顾一下sts的基本运行特性，代入地阅读代码会比较顺畅

**创建**

sts是有序的，pod副本有序串行地新建，pod名称为{sts_name}-{0..N}，从小序号的pod(名称为{sts_name}-0)创建，一直到第n个副本的pod(名称为{sts_name}-n)

**更新**

sts的更新策略有2种:

-  `RollingUpdateStatefulSetStrategyType`，默认的滚动更新策略，此策略下，更新时pod根据序号反顺序更新，从最大序号的pod开始删除重建，更新至序号最小的pod。更新过程中，始终保持pod数量等于指定副本数，即每删除一个pod，才会再创建一个。同时可以指定一个**partition**参数，指定这个参数后，只有序号大于等于partition的pod才会被更新，序号小于partition参数的pod不会被更新，例如有5个副本，partition设置为2，那么在更新sts时，0和1号pod不会更新，2 3 4号pod则会更新重建；此时继续将partition缩减为0，则0 1号pod也会更新重建。默认partition为0，即所有的pod都会更新。这个参数一般不会使用，但可用在发布时动态更新递减partition的值，来实现滚动灰度发布。

- `OnDeleteStatefulSetStrategyType`, 此策略下controller不会对pod做任何操作，由手动删除pod来触发新pod的创建

  

**删除**

删除sts时，可以指定级联模式的参数`--cascade=true`，默认为true，意思是删除sts会同时删除它所管理的pod。设置为false时，删除sts不会影响pod的运行，且sts重建后依然能与此前的pod关联起来(这种方式可能会产生孤儿pod)。

**关联关系**

先来看看sts和pod的关联方式:

```bash
# sts
[root@008019 ~]# kubectl get sts deptest11dev
NAME           READY   AGE
deptest11dev   2/2     99d

# pod
[root@008019 ~]# kubectl get pods  | grep deptest11dev
deptest11dev-0                                    1/1     Running                 1          99d
deptest11dev-1                                    1/1     Running                 0          3d17h

# edit pod
# 可以查看到pod的ownerReferences字段，与sts关联
ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: StatefulSet
    name: deptest11dev
    uid: 28ecf735-2ab4-11ea-afa8-1866daf0f324

# 可以查看到pod的labels标签，新增了一个controller-revision-hash标签，与controllerRevision关联
  labels:
    app: deptest11dev
    controller-revision-hash: deptest11dev-587f8bd845
    statefulset.kubernetes.io/pod-name: deptest11dev-1
```



再来看看sts和ControllerRevision关联方式:

```bash
[root@008019 ~]# kubectl get sts deptest11dev
NAME           READY   AGE
deptest11dev   2/2     99d



[root@008019 ~]# kubectl get ControllerRevisions | grep deptest11dev
deptest11dev-587f8bd845                                    statefulset.apps/deptest11dev                                    1          99d

[root@008019 ~]# kubectl get ControllerRevisions deptest11dev-587f8bd845
NAME                      CONTROLLER                      REVISION   AGE
deptest11dev-587f8bd845   statefulset.apps/deptest11dev   1          99d


# ControllerRevisions资源中的ownerReferences字段,可以看出sts与其通过这个字段关联
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: StatefulSet
    name: deptest11dev
    uid: 28ecf735-2ab4-11ea-afa8-1866daf0f324
    

# sts status字段,可以看出sts通过status下的currentRevision、updateRevision字段与ControllerRevision关联
status:
  collisionCount: 0
  currentReplicas: 2
  currentRevision: deptest11dev-587f8bd845
  observedGeneration: 3
  readyReplicas: 2
  replicas: 2
  updateRevision: deptest11dev-587f8bd845
  updatedReplicas: 2

# 对sts.spec字段里的内容更新后引起pod重建，sts开始滚动更新，此时sts的status字段内容如下:
status:
  collisionCount: 0
  currentReplicas: 1
  currentRevision: deptest11dev-587f8bd845
  observedGeneration: 4
  readyReplicas: 2
  replicas: 2
  # 这时可以发现updateRevision字段更新为了新的revision，即updateRevision是最近一次更新的Revision
  updateRevision: deptest11dev-7487498978
  
# 修改sts进行缩容/扩容 时的status字段：
status:
  collisionCount: 0
  currentReplicas: 3
  currentRevision: deptest11dev-7487498978
  observedGeneration: 5
  readyReplicas: 3
  replicas: 3
  # revision不会更新
  updateRevision: deptest11dev-7487498978
  updatedReplicas: 3
  

```

**记住这几者之间双向地关联方式，下面会提到。**



## StatefulSet Controller

### 初始化

`cmd/kube-controller-manager/app/apps.go:59`

```go
func startStatefulSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "statefulsets"}] {
		return nil, false, nil
	}
	go statefulset.NewStatefulSetController(
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Apps().V1().StatefulSets(),
		ctx.InformerFactory.Core().V1().PersistentVolumeClaims(),
		ctx.InformerFactory.Apps().V1().ControllerRevisions(),
		ctx.ClientBuilder.ClientOrDie("statefulset-controller"),
	).Run(1, ctx.Stop)
	return nil, true, nil
}
```

先来看看`NewStatefulSetController`做了什么：

==> `pkg/controller/statefulset/stateful_set.go:81`

```go
func NewStatefulSetController(
  // 1.StatefulSetController关注四种类型的资源: Pod/Sts/PVC/ControllerRevision
	podInformer coreinformers.PodInformer,
	setInformer appsinformers.StatefulSetInformer,
	pvcInformer coreinformers.PersistentVolumeClaimInformer,
	revInformer appsinformers.ControllerRevisionInformer,
	kubeClient clientset.Interface,
) *StatefulSetController {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	recorder := eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "statefulset-controller"})
  
  // 2.NewDefaultStatefulSetControl方法需要关注
	ssc := &StatefulSetController{
		kubeClient: kubeClient,
		control: NewDefaultStatefulSetControl(
			NewRealStatefulPodControl(
				kubeClient,
				setInformer.Lister(),
				podInformer.Lister(),
				pvcInformer.Lister(),
				recorder),
			NewRealStatefulSetStatusUpdater(kubeClient, setInformer.Lister()),
			history.NewHistory(kubeClient, revInformer.Lister()),
			recorder,
		),
		pvcListerSynced: pvcInformer.Informer().HasSynced,
		queue:           workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "statefulset"),
		podControl:      controller.RealPodControl{KubeClient: kubeClient, Recorder: recorder},

		revListerSynced: revInformer.Informer().HasSynced,
	}
  // 当sts管理的pod curd时对应的处理方法(按入workqueue/更新pod/删除pod)
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		// lookup the statefulset and enqueue
		AddFunc: ssc.addPod,
		// lookup current and old statefulset if labels changed
		UpdateFunc: ssc.updatePod,
		// lookup statefulset accounting for deletion tombstones
		DeleteFunc: ssc.deletePod,
	})
	ssc.podLister = podInformer.Lister()
	ssc.podListerSynced = podInformer.Informer().HasSynced
  
  // 当sts curd时对应的方法(sts压入workqueue)
	setInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc: ssc.enqueueStatefulSet,
			UpdateFunc: func(old, cur interface{}) {
				oldPS := old.(*apps.StatefulSet)
				curPS := cur.(*apps.StatefulSet)
				if oldPS.Status.Replicas != curPS.Status.Replicas {
					klog.V(4).Infof("Observed updated replica count for StatefulSet: %v, %d->%d", curPS.Name, oldPS.Status.Replicas, curPS.Status.Replicas)
				}
				ssc.enqueueStatefulSet(cur)
			},
			DeleteFunc: ssc.enqueueStatefulSet,
		},
		statefulSetResyncPeriod,
	)
	ssc.setLister = setInformer.Lister()
	ssc.setListerSynced = setInformer.Informer().HasSynced

	// TODO: Watch volumes
  // 返回ssc(StatefulSetController)
	return ssc
}
```

先看注释1，可以发现，StatefulSetController关注四种类型的资源: Pod/Sts/PVC/ControllerRevision，其中的**ControllerRevision**不太熟悉，先找出来看下它的结构，逐级跳转:

`cmd/kube-controller-manager/app/apps.go:63`

==> `vendor/k8s.io/client-go/informers/apps/v1/interface.go:28`

==>`vendor/k8s.io/client-go/informers/apps/v1/controllerrevision.go:38`

==> `vendor/k8s.io/client-go/listers/apps/v1/controllerrevision.go:29`

==> `vendor/k8s.io/api/apps/v1/types.go:800`

```go
type ControllerRevision struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Data is the serialized representation of the state.
	Data runtime.RawExtension `json:"data,omitempty" protobuf:"bytes,2,opt,name=data"`

	// Revision indicates the revision of the state represented by Data.
	Revision int64 `json:"revision" protobuf:"varint,3,opt,name=revision"`
}

```

阅读这个结构体上方的注释可以得知，ControllerRevision提供给DaemonSet和StatefulSet用作更新和回滚，ControllerRevision存放的是数据的快照，ControllerRevision生成之后内容是不可修改的，由调用端来负责序列化写入和反序列化读取。其中Revision(int64)字段相当于ControllerRevision的版本id号，Data字段则存放序列化后的数据。



**画外音：不难猜测，StatefulSet的更新以及回滚(也是一种特殊的更新)操作，是基于对新旧ControllerRevision的对比来进行的**



在来看下注释2，NewDefaultStatefulSetControl方法:

`pkg/controller/statefulset/stateful_set.go:95`

==> `pkg/controller/statefulset/stateful_set_control.go:54`

```go
func NewDefaultStatefulSetControl(
	podControl StatefulPodControlInterface,
	statusUpdater StatefulSetStatusUpdaterInterface,
	controllerHistory history.Interface,
	recorder record.EventRecorder) StatefulSetControlInterface {
	return &defaultStatefulSetControl{podControl, statusUpdater, controllerHistory, recorder}
}
```

**NewDefaultStatefulSetControl**返回的**defaultStatefulSetControl**结构体对象是sts管理控制逻辑的语义实现，**defaultStatefulSetControl**结构体里面包含了sts控制过程中的各种接口：

1. 管理sts对应的pod/pvc(podControl)的方法接口，有(**CreateStatefulPod/UpdateStatefulPod/DeleteStatefulPod**)这几个方法，通过**NewRealStatefulPodControl**函数返回的**realStatefulPodControl**结构体对象来实现
2. 管理sts status状态的更新(statusUpdater)的方法接口，有**UpdateStatefulSetStatus**这一个方法，通过**NewRealStatefulSetStatusUpdater**返回的**realStatefulSetStatusUpdater**结构体对象来实现。
3. 管理ControllerRevision版本(controllerHistory) 的方法接口，有(**ListControllerRevisions/CreateControllerRevision/DeleteControllerRevision/UpdateControllerRevision/AdoptControllerRevision/ReleaseControllerRevision**)这几个方法，通过***history.NewHistory***返回的**realHistory**结构体对象来实现。

现在接着往下，去看看ssc(StatefulSetController) 运行的Run函数。



### 工作过程

`*StatefulSetController.Run()`函数：

```go
func (ssc *StatefulSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer ssc.queue.ShutDown()

	klog.Infof("Starting stateful set controller")
	defer klog.Infof("Shutting down statefulset controller")

	if !controller.WaitForCacheSync("stateful set", stopCh, ssc.podListerSynced, ssc.setListerSynced, ssc.pvcListerSynced, ssc.revListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(ssc.worker, time.Second, stopCh)
	}

	<-stopCh
}
```

wait.Until定时器前面已经讲过，不再复述，重点在于ssc.worker函数，代码里有多次跳跃：



`pkg/controller/statefulset/stateful_set.go:159`

==>`pkg/controller/statefulset/stateful_set.go:410`

==> `pkg/controller/statefulset/stateful_set.go:399`

==>`pkg/controller/statefulset/stateful_set.go:415`

```go
// sync syncs the given statefulset.
func (ssc *StatefulSetController) sync(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing statefulset %q (%v)", key, time.Since(startTime))
	}()
  // key的样例: default/teststs，做个切割，拿到namespace和sts name
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
  // 获取到sts对象
	set, err := ssc.setLister.StatefulSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.Infof("StatefulSet has been deleted %v", key)
		return nil
	}
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("unable to retrieve StatefulSet %v from store: %v", key, err))
		return err
	}
  
  // labelSelector
	selector, err := metav1.LabelSelectorAsSelector(set.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("error converting StatefulSet %v selector: %v", key, err))
		// This is a non-transient error, so don't retry.
		return nil
	}

  // 孤儿Revisions修正托管
	if err := ssc.adoptOrphanRevisions(set); err != nil {
		return err
	}
  // 获取到sts管理的pod
	pods, err := ssc.getPodsForStatefulSet(set, selector)
	if err != nil {
		return err
	}
  // syncStatefulSet 执行sts sync
	return ssc.syncStatefulSet(set, pods)
}
```

来分步看下



#### 孤儿Revisions修正托管

上面指出sts和revision两者之间显示地双向指定字段来关联对方，明白这一点那么这个函数就好理解了。

出现孤儿ControllerRevisions的原因，很有可能是sts在此期间进行了反复的更新，更新时间差之中产生了脏数据.

`pkg/controller/statefulset/stateful_set.go:316`

```go
// adoptOrphanRevisions adopts any orphaned ControllerRevisions matched by set's Selector.
func (ssc *StatefulSetController) adoptOrphanRevisions(set *apps.StatefulSet) error {
  // 通过sts指定的revision相关字段找到对应的revisions
	revisions, err := ssc.control.ListRevisions(set)
	if err != nil {
		return err
	}
	hasOrphans := false
	for i := range revisions {
    // 通过revision指定的controller来源，来找sts。如果指定绑定的sts为空，那么说明此ControllerRevisions是孤儿状态(无托管)，需要回收
		if metav1.GetControllerOf(revisions[i]) == nil {
			hasOrphans = true
			break
		}
	}
  
  // 出现孤儿ControllerRevisions的原因，很有可能是sts在此期间进行了反复的更新，因此重新获取一次最新的sts
	if hasOrphans {
		fresh, err := ssc.kubeClient.AppsV1().StatefulSets(set.Namespace).Get(set.Name, metav1.GetOptions{})
		if err != nil {
			return err
		}
    // sts(old) 若与fresh sts uid不同，则说明期间sts可能经历了删除重建，本次逻辑的流程打破，抛错返回
		if fresh.UID != set.UID {
			return fmt.Errorf("original StatefulSet %v/%v is gone: got uid %v, wanted %v", set.Namespace, set.Name, fresh.UID, set.UID)
		}
    // 为这些controller sts指定为空的revision，若label匹配则加上ownerReferences sts指定，若label不匹配则gc
		return ssc.control.AdoptOrphanRevisions(set, revisions)
	}
	return nil
}
```



#### 获取到sts管理的pod

`pkg/controller/statefulset/stateful_set.go:285`

```go
func (ssc *StatefulSetController) getPodsForStatefulSet(set *apps.StatefulSet, selector labels.Selector) ([]*v1.Pod, error) {
	// List all pods to include the pods that don't match the selector anymore but
	// has a ControllerRef pointing to this StatefulSet.
	pods, err := ssc.podLister.Pods(set.Namespace).List(labels.Everything())
	if err != nil {
		return nil, err
	}
	
  // filter函数的作用是判断指定的pod和sts是否有所属关系，展开代码可以看到判断的方式很简单，对pod的名称做re字符串切割，最后一个"-"之前的字符串是parent，之后的数字是序号索引，判断parent与sts name是否一致，一致则为true，pod 属于 sts，不一致则为false
	filter := func(pod *v1.Pod) bool {
		// Only claim if it matches our StatefulSet name. Otherwise release/ignore.
		return isMemberOf(set, pod)
	}
  

	// 如同revision一样，若存在孤儿pod，也需要对孤儿pod进行收养，与sts label匹配则加上关联，label不匹配则解除关联。
	canAdoptFunc := controller.RecheckDeletionTimestamp(func() (metav1.Object, error) {
		fresh, err := ssc.kubeClient.AppsV1().StatefulSets(set.Namespace).Get(set.Name, metav1.GetOptions{})
		if err != nil {
			return nil, err
		}
		if fresh.UID != set.UID {
			return nil, fmt.Errorf("original StatefulSet %v/%v is gone: got uid %v, wanted %v", set.Namespace, set.Name, fresh.UID, set.UID)
		}
		return fresh, nil
	})

	cm := controller.NewPodControllerRefManager(ssc.podControl, set, selector, controllerKind, canAdoptFunc)
  // 执行筛选
	return cm.ClaimPods(pods, filter)
}
```



**ClaimPods**

`pkg/controller/controller_ref_manager.go:171`

```go
func (m *PodControllerRefManager) ClaimPods(pods []*v1.Pod, filters ...func(*v1.Pod) bool) ([]*v1.Pod, error) {
	var claimed []*v1.Pod
	var errlist []error
  
  
	match := func(obj metav1.Object) bool {
		pod := obj.(*v1.Pod)
    // 先根据标签匹配pod，仅当标签匹配通过后，再匹配下一步（sts调用则是按照上面说的取 pod name 字符串切割后与sts name对比）
		if !m.Selector.Matches(labels.Set(pod.Labels)) {
			return false
		}
		for _, filter := range filters {
			if !filter(pod) {
				return false
			}
		}
		return true
	}
	adopt := func(obj metav1.Object) error {
    // 收养pod(添加关联关系)，即为pod.metadata  patch ownerReferences字段。
		return m.AdoptPod(obj.(*v1.Pod))
	}
	release := func(obj metav1.Object) error {
    // 释放pod关联关系，即为pod.metadata  delete ownerReferences字段。
		return m.ReleasePod(obj.(*v1.Pod))
	}

	for _, pod := range pods {
    // 判断单个pod是否匹配，收养/释放孤儿pod的函数ClaimObject
		ok, err := m.ClaimObject(pod, match, adopt, release)
		if err != nil {
			errlist = append(errlist, err)
			continue
		}
		if ok {
			claimed = append(claimed, pod)
		}
	}
	return claimed, utilerrors.NewAggregate(errlist)
}
```

#### 

####ClaimObject

`pkg/controller/controller_ref_manager.go:66`

```go
func (m *BaseControllerRefManager) ClaimObject(obj metav1.Object, match func(metav1.Object) bool, adopt, release func(metav1.Object) error) (bool, error) {
  // 1 获取到pod.metadata中的ownerReferences字段
	controllerRef := metav1.GetControllerOf(obj)
  // 1-1 如果pod存在ownerReferences,则直接进入判断是否match
	if controllerRef != nil {
		if controllerRef.UID != m.Controller.GetUID() {
			// Owned by someone else. Ignore.
			return false, nil
		}
    // 1-2 匹配则返回true
		if match(obj) {
			return true, nil
		}
		
		if m.Controller.GetDeletionTimestamp() != nil {
			return false, nil
		}
    
    // 1-3 不匹配则pod释放关联字段,返回false
		if err := release(obj); err != nil {
			// If the pod no longer exists, ignore the error.
			if errors.IsNotFound(err) {
				return false, nil
			}
			return false, err
		}
		// Successfully released.
		return false, nil
	}
  
	// 2 孤儿pod，则要根据情况判断是否收养/释放
  // 2-1 已删除的sts或match规则不匹配，返回false
	if m.Controller.GetDeletionTimestamp() != nil || !match(obj) {
		// Ignore if we're being deleted or selector doesn't match.
		return false, nil
	}
	if obj.GetDeletionTimestamp() != nil {
		// Ignore if the object is being deleted
		return false, nil
	}
	// Selector matches. Try to adopt.
	if err := adopt(obj); err != nil {
		// If the pod no longer exists, ignore the error.
		if errors.IsNotFound(err) {
			return false, nil
		}
		// Either someone else claimed it first, or there was a transient error.
		// The controller should requeue and try again if it's still orphaned.
		return false, err
	}
	// 收养成功返回true
	return true, nil
}
```

到这里，所有应当被sts管理的pod(包括孤儿pod)就过滤完毕了，开始执行真正的sts sync。



#### syncStatefulSet

在找到了所有管理的pod后，就要开始sts 的sync，进行更新sts及更新pod的操作了，回到这里:

`pkg/controller/statefulset/stateful_set.go:451`

==> `pkg/controller/statefulset/stateful_set.go:458`

==> `pkg/controller/statefulset/stateful_set_control.go:75`

```go
func (ssc *defaultStatefulSetControl) UpdateStatefulSet(set *apps.StatefulSet, pods []*v1.Pod) error {

	// 取出sts所有的revision并排序
	revisions, err := ssc.ListRevisions(set)
	if err != nil {
		return err
	}
	history.SortControllerRevisions(revisions)

	// 获得当前revision，以及更新后最新的revision
	currentRevision, updateRevision, collisionCount, err := ssc.getStatefulSetRevisions(set, revisions)
	if err != nil {
		return err
	}

	// 核心方法，对pod进行操作
	status, err := ssc.updateStatefulSet(set, currentRevision, updateRevision, collisionCount, pods)
	if err != nil {
		return err
	}

	// 操作完成后记录修改sts.status
	err = ssc.updateStatefulSetStatus(set, status)
	if err != nil {
		return err
	}

	klog.V(4).Infof("StatefulSet %s/%s pod status replicas=%d ready=%d current=%d updated=%d",
		set.Namespace,
		set.Name,
		status.Replicas,
		status.ReadyReplicas,
		status.CurrentReplicas,
		status.UpdatedReplicas)

	klog.V(4).Infof("StatefulSet %s/%s revisions current=%s update=%s",
		set.Namespace,
		set.Name,
		status.CurrentRevision,
		status.UpdateRevision)

	// 对set的revision history进行维护
	return ssc.truncateHistory(set, pods, revisions, currentRevision, updateRevision)
}
```

这里面最核心的函数是`updateStatefulSetStatus`,接着往下



#### updateStatefulSet

这一个函数内容很多，200多行代码，需要说明的地方会在下面代码中注释。

```go
func (ssc *defaultStatefulSetControl) updateStatefulSet(
	set *apps.StatefulSet,
	currentRevision *apps.ControllerRevision,
	updateRevision *apps.ControllerRevision,
	collisionCount int32,
	pods []*v1.Pod) (*apps.StatefulSetStatus, error) {
  
  // 获取到当前sts currentSet，然后获取到需更新到的sts updateSet。要实现的更新效果是：
  
  // 1.滚动更新时，在未指定partition时，使当前sts的管理的pod缩减为0，updateSet的ready pod数 = spec.replicas 
  // 2.滚动更新时，在未指定partition后，大于等于partition的pod全部归于updateSet，小于partition值的pod还是归属于原currentSet
  // 3.OnDelete更新时，do nothing
	currentSet, err := ApplyRevision(set, currentRevision)
	if err != nil {
		return nil, err
	}
  
	updateSet, err := ApplyRevision(set, updateRevision)
	if err != nil {
		return nil, err
	}

	// set the generation, and revisions in the returned status
  // 重新计算sts的status
	status := apps.StatefulSetStatus{}
	status.ObservedGeneration = set.Generation
	status.CurrentRevision = currentRevision.Name
	status.UpdateRevision = updateRevision.Name
	status.CollisionCount = new(int32)
	*status.CollisionCount = collisionCount

	replicaCount := int(*set.Spec.Replicas)
	// replicas是合法副本，将满足 0 <= pod序号 < sts.spec.replicas的pod,放到这个slice里来。这里面的pod都是要保证ready的
	replicas := make([]*v1.Pod, replicaCount)
  // condemned是非法副本，将满足 pod序号 >= sts.spec.replicas的pod,放到这个slice里来,这些pod是要删除掉的(可能是被缩容掉的)
	condemned := make([]*v1.Pod, 0, len(pods))
	unhealthy := 0
	firstUnhealthyOrdinal := math.MaxInt32
	var firstUnhealthyPod *v1.Pod

	// First we partition pods into two lists valid replicas and condemned Pods
	for i := range pods {
		status.Replicas++

		// status.ReadyReplicas计数
		if isRunningAndReady(pods[i]) {
			status.ReadyReplicas++
		}

		if isCreated(pods[i]) && !isTerminating(pods[i]) {
      // 通过pod的controller-revision-hash label，判断pod属于currentSet还是UpdatedSet，分别计数
			if getPodRevision(pods[i]) == currentRevision.Name {
				status.CurrentReplicas++
			}
			if getPodRevision(pods[i]) == updateRevision.Name {
				status.UpdatedReplicas++
			}
		}

		if ord := getOrdinal(pods[i]); 0 <= ord && ord < replicaCount {
      // 将满足 0 <= pod序号 < sts.spec.replicas的pod,放到replicas这个slice里来
			replicas[ord] = pods[i]

		} else if ord >= replicaCount {
			// 将满足 pod序号 >= sts.spec.replicas的pod,放到condemned这个slice里来,这些pod是要删除掉的。
			condemned = append(condemned, pods[i])
		}
	}

 
  // replicas slice之中如果有索引位置为空，则需要填充相应的pod。
  // 根据currentSet.replicas/UpdatedSet.replicas/partition这三个值来判断pod是基于current revision还是基于update revision创建
	for ord := 0; ord < replicaCount; ord++ {
		if replicas[ord] == nil {
			replicas[ord] = newVersionedStatefulSetPod(
				currentSet,
				updateSet,
				currentRevision.Name,
				updateRevision.Name, ord)
		}
	}

	// 对需要删除的非法pod按照序号从大到小的顺序排序
	sort.Sort(ascendingOrdinal(condemned))

	// 如果有不健康的pod，也需要删除，但还是遵循串行的原则，优先删除非法pod中序号最大的，再到合法副本中的序号最小的。
	for i := range replicas {
		if !isHealthy(replicas[i]) {
			unhealthy++
			if ord := getOrdinal(replicas[i]); ord < firstUnhealthyOrdinal {
				firstUnhealthyOrdinal = ord
				firstUnhealthyPod = replicas[i]
			}
		}
	}

	for i := range condemned {
		if !isHealthy(condemned[i]) {
			unhealthy++
			if ord := getOrdinal(condemned[i]); ord < firstUnhealthyOrdinal {
				firstUnhealthyOrdinal = ord
				firstUnhealthyPod = condemned[i]
			}
		}
	}

	if unhealthy > 0 {
		klog.V(4).Infof("StatefulSet %s/%s has %d unhealthy Pods starting with %s",
			set.Namespace,
			set.Name,
			unhealthy,
			firstUnhealthyPod.Name)
	}

	// If the StatefulSet is being deleted, don't do anything other than updating
	// status.
	if set.DeletionTimestamp != nil {
		return &status, nil
	}

	monotonic := !allowsBurst(set)

	// 根据pod的序号，对它们依次进行检查并操作。
	for i := range replicas {
		// 错误状态的pod删除重建
		if isFailed(replicas[i]) {
			ssc.recorder.Eventf(set, v1.EventTypeWarning, "RecreatingFailedPod",
				"StatefulSet %s/%s is recreating failed Pod %s",
				set.Namespace,
				set.Name,
				replicas[i].Name)
			if err := ssc.podControl.DeleteStatefulPod(set, replicas[i]); err != nil {
				return &status, err
			}
			if getPodRevision(replicas[i]) == currentRevision.Name {
				status.CurrentReplicas--
			}
			if getPodRevision(replicas[i]) == updateRevision.Name {
				status.UpdatedReplicas--
			}
			status.Replicas--
			replicas[i] = newVersionedStatefulSetPod(
				currentSet,
				updateSet,
				currentRevision.Name,
				updateRevision.Name,
				i)
		}
    // pod没有被创建(可能是上面刚填充的)，就创建pod
		if !isCreated(replicas[i]) {
			if err := ssc.podControl.CreateStatefulPod(set, replicas[i]); err != nil {
				return &status, err
			}
			status.Replicas++
			if getPodRevision(replicas[i]) == currentRevision.Name {
				status.CurrentReplicas++
			}
			if getPodRevision(replicas[i]) == updateRevision.Name {
				status.UpdatedReplicas++
			}

			// 如果不允许burst，直接返回
			if monotonic {
				return &status, nil
			}
			// pod created, no more work possible for this round
			continue
		}
		// 如果不允许burst，对于终结中的pod不采取任何逻辑，等待它终结完毕后下一轮再操作。
		if isTerminating(replicas[i]) && monotonic {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to Terminate",
				set.Namespace,
				set.Name,
				replicas[i].Name)
			return &status, nil
		}
    // 如果是正在创建中的pod(还未达到ready状态)，同样不采取任何操作，因为需要保证创建操作依次有序
		if !isRunningAndReady(replicas[i]) && monotonic {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to be Running and Ready",
				set.Namespace,
				set.Name,
				replicas[i].Name)
			return &status, nil
		}
		
    // 如果此pod与sts已经匹配(ready),且存储满足sts、pod的要求，那么这个pod就是合格的pod，continue
		if identityMatches(set, replicas[i]) && storageMatches(set, replicas[i]) {
			continue
		}
    
		// 确保pod与sts的标签关联，以及为pod准备好它需要的pvc
		replica := replicas[i].DeepCopy()
		if err := ssc.podControl.UpdateStatefulPod(updateSet, replica); err != nil {
			return &status, err
		}
	}

  
  // 上面的合法副本得以保证之后，下面要开始按pod序号从大到小的顺序，删除非法pod了
	for target := len(condemned) - 1; target >= 0; target-- {
		// 终结中的pod不再处理，直接返回，等待下一轮检查
		if isTerminating(condemned[target]) {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to Terminate prior to scale down",
				set.Namespace,
				set.Name,
				condemned[target].Name)
			// block if we are in monotonic mode
			if monotonic {
				return &status, nil
			}
			continue
		}
		
    // 如果此非法pod不是ready状态，且不允许burst，且它不是优先级第一的非健康pod，不做任何操作。换而言之，即使是删除非健康的pod，也要按照序号从大到小的顺序串行执行。
		if !isRunningAndReady(condemned[target]) && monotonic && condemned[target] != firstUnhealthyPod {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to be Running and Ready prior to scale down",
				set.Namespace,
				set.Name,
				firstUnhealthyPod.Name)
			return &status, nil
		}
    // 开始删除此pod，更新status
		klog.V(2).Infof("StatefulSet %s/%s terminating Pod %s for scale down",
			set.Namespace,
			set.Name,
			condemned[target].Name)

		if err := ssc.podControl.DeleteStatefulPod(set, condemned[target]); err != nil {
			return &status, err
		}
		if getPodRevision(condemned[target]) == currentRevision.Name {
			status.CurrentReplicas--
		}
		if getPodRevision(condemned[target]) == updateRevision.Name {
			status.UpdatedReplicas--
		}
		if monotonic {
			return &status, nil
		}
	}

	// OnDelete更新模式下，不自动删除pod，需要手动删除pod来触发更新
	if set.Spec.UpdateStrategy.Type == apps.OnDeleteStatefulSetStrategyType {
		return &status, nil
	}

	// 经过上面那么多条件的过滤和准备，现在要开始对replicas里的合法pod进行检查了
	updateMin := 0
	if set.Spec.UpdateStrategy.RollingUpdate != nil {
		updateMin = int(*set.Spec.UpdateStrategy.RollingUpdate.Partition)
	}
	// 按pod的序号倒序检查
	for target := len(replicas) - 1; target >= updateMin; target-- {

		// 如果pod的revision不符合updateRevision，那么删除重建此pod
		if getPodRevision(replicas[target]) != updateRevision.Name && !isTerminating(replicas[target]) {
			klog.V(2).Infof("StatefulSet %s/%s terminating Pod %s for update",
				set.Namespace,
				set.Name,
				replicas[target].Name)
			err := ssc.podControl.DeleteStatefulPod(set, replicas[target])
			status.CurrentReplicas--
			return &status, err
		}

		// 合法pod更新过程中，还未到达ready状态的pod，等待它
		if !isHealthy(replicas[target]) {
			klog.V(4).Infof(
				"StatefulSet %s/%s is waiting for Pod %s to update",
				set.Namespace,
				set.Name,
				replicas[target].Name)
			return &status, nil
		}

	}
	return &status, nil
}
```

**updateStatefulSet函数总结**

1. 每个循环的周期中，最多操作一个pod
2. 根据sts.spec.replicas对比现有pod的序号，对pod进行划分，一部分划为合法(保留/重建)，一部分划为非法(删除)
3. 对pods进行划分，一部分划入current(old) set阵营，另一部分划入update(new) set阵营
4. 更新过程中，无论是删减、还是新建，都保持pod数量固定，有序地递增、递减
5. 最终保证所有的pod都归属于update revision



## 总结

statefulset  在设计上与 deployment 有许多不同的地方，例如：

- deployment通过rs管理pod，sts通过controllerRevision管理pod；

- deployment curd是无序的，sts强保证有序curd

- sts需要检查存储的匹配

在了解sts管理操作pod方式的基础上来看代码，会有许多的帮助。



