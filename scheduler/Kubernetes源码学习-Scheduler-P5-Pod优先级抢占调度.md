# P5-Pod优先级抢占调度

## 1. 前言

前面的两篇文章中，已经讲过了调度pod的算法(predicate/priority)，在kubernetes v1.8版本之后可以指定pod优先级(v1alpha1)，若资源不足导致高优先级pod匹配失败，高优先级pod会转而将其驱逐，抢占低优先级pod的资源，那么本篇就从代码的层面展开看一看pod抢占部分的逻辑。

## 2. 抢占调度入口

在P1-入口篇中我们找到了调度算法计算的入口，随后展开了调度算法的两篇解读，本篇我们再次回到此入口的位置，接着往下看:

`pkg/scheduler/scheduler.go:457`

```go
func (sched *Scheduler) scheduleOne() {
	... // 省略
  
  // 调度算法计算入口
	scheduleResult, err := sched.schedule(pod) 
  
	if err != nil {
		// schedule() may have failed because the pod would not fit on any host, so we try to
		// preempt, with the expectation that the next time the pod is tried for scheduling it
		// will fit due to the preemption. It is also possible that a different pod will schedule
		// into the resources that were preempted, but this is harmless.
		if fitError, ok := err.(*core.FitError); ok {
			if !util.PodPriorityEnabled() || sched.config.DisablePreemption {
				klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
					" No preemption is performed.")
			} else {
				preemptionStartTime := time.Now()
				sched.preempt(pod, fitError)  // 抢占调度逻辑入口
				metrics.PreemptionAttempts.Inc()
			  ... // 省略
			metrics.PodScheduleFailures.Inc()
		} else {
			klog.Errorf("error selecting node for pod: %v", err)
			metrics.PodScheduleErrors.Inc()
		}
		return
	}
```

注释中可看出，若在筛选算法中并未找到fitNode且返回了fitError，那么就会进入基于pod优先级的资源抢占的逻辑，入口是`sched.preempt(pod, fitError)`函数。在展开抢占逻辑之前，我们先来看一看pod优先级是怎么一回事吧。

### 2.1.  Pod优先级的定义

字面意义上来理解，pod优先级可以在调度的时候为高优先级的pod提供资源空间保障，若出现资源紧张的情况，则在其他约束规则允许的情况下，高优先级pod会抢占低优先级pod的资源。此功能在1.11版本以后默认开启，默认情况下pod的优先级是0，优先级值high is better，具体说明来看看官方文档的解释吧:

[**Pod Priority and Preemption**](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)

下面列举一个pod优先级使用的实例：

```yaml
# Example PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."

# Example Pod spec
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

了解了定义及如何使用，那我们来看看代码层面是如何实现的吧！

## 3. 抢占调度算法

从上面的入口跳转:

`pkg/scheduler/scheduler.go:469` --> `pkg/scheduler/scheduler.go:290`

```go
func (sched *Scheduler) preempt(preemptor *v1.Pod, scheduleErr error) (string, error) {
	preemptor, err := sched.config.PodPreemptor.GetUpdatedPod(preemptor)
	if err != nil {
		klog.Errorf("Error getting the updated preemptor pod object: %v", err)
		return "", err
	}
  // 通过默认注册的抢占算法，计算得出最终被执行抢占调度的node、node上需要驱逐的pod等信息
	node, victims, nominatedPodsToClear, err := sched.config.Algorithm.Preempt(preemptor, sched.config.NodeLister, scheduleErr)
	if err != nil {
		klog.Errorf("Error preempting victims to make room for %v/%v.", preemptor.Namespace, preemptor.Name)
		return "", err
	}
	var nodeName = ""
	if node != nil {
		nodeName = node.Name
		// Update the scheduling queue with the nominated pod information. Without
		// this, there would be a race condition between the next scheduling cycle
		// and the time the scheduler receives a Pod Update for the nominated pod.
    // 给调度队列内的preemptor pod加上提名node信息，避免下一个调度周期出现冲突
		sched.config.SchedulingQueue.UpdateNominatedPodForNode(preemptor, nodeName)

		// Make a call to update nominated node name of the pod on the API server.
    // 给待调度pod指定NominatedNodeName，pod.Status.NominatedNodeName = nodeName
		err = sched.config.PodPreemptor.SetNominatedNodeName(preemptor, nodeName)
		if err != nil {
			klog.Errorf("Error in preemption process. Cannot update pod %v/%v annotations: %v", preemptor.Namespace, preemptor.Name, err)
			sched.config.SchedulingQueue.DeleteNominatedPodIfExists(preemptor)
			return "", err
		}

		for _, victim := range victims {
      // 对node上需要驱逐的pod执行删除操作
			if err := sched.config.PodPreemptor.DeletePod(victim); err != nil {
				klog.Errorf("Error preempting pod %v/%v: %v", victim.Namespace, victim.Name, err)
				return "", err
			}
			sched.config.Recorder.Eventf(victim, v1.EventTypeNormal, "Preempted", "by %v/%v on node %v", preemptor.Namespace, preemptor.Name, nodeName)
		}
		metrics.PreemptionVictims.Set(float64(len(victims)))
	}
	// Clearing nominated pods should happen outside of "if node != nil". Node could
	// be nil when a pod with nominated node name is eligible to preempt again,
	// but preemption logic does not find any node for it. In that case Preempt()
	// function of generic_scheduler.go returns the pod itself for removal of the annotation.
  // 当找不到合适的抢占node时，可能是因为preemptor pod已经有了提名的node，但它又执行了一遍抢占逻辑，说明它
  // 在上一次调度周期中没有调度成功，因此，删除调度队列中比当前preemptor pod优先级更低的pod所指定的提名
  // node信息(pod.Status.NominatedNodeName)
	for _, p := range nominatedPodsToClear {
		rErr := sched.config.PodPreemptor.RemoveNominatedNodeName(p)
		if rErr != nil {
			klog.Errorf("Cannot remove nominated node annotation of pod: %v", rErr)
			// We do not return as this error is not critical.
		}
	}
	return nodeName, err
}
```



如优先级筛选算法一样，调度算法最终也是要挑选出一个供以实际运行抢占调度逻辑的node，那么一起来看看这个计算算法是怎么样的。如schedule()方法一样，preempt()的默认方法也在`generic_scheduler.go`这个文件中：

`pkg/scheduler/core/generic_scheduler.go:288`

将函数内拆成几个重要的部分，其余部分省略，逐个说明

```go
func (g *genericScheduler) Preempt(pod *v1.Pod, nodeLister algorithm.NodeLister, scheduleErr error) (*v1.Node, []*v1.Pod, []*v1.Pod, error) {
	// ... 省略
  
  // 每次开始抢占调度之前，检查一下当前pod是否已经有了提名抢占调度的节点，且该节点上当前不包含正在终结中的pod，若pod已有提名调度节点，且该节点上已经有pod正在终结中，则视为已经在执行抢占的动作了，所以不再往下重复执行。可以自行进去查看，比较简单，不拿出来讲了。
	if !podEligibleToPreemptOthers(pod, g.nodeInfoSnapshot.NodeInfoMap) {
		klog.V(5).Infof("Pod %v/%v is not eligible for more preemption.", pod.Namespace, pod.Name)
		return nil, nil, nil, nil
	}
  // ... 省略
  
  // potentialNodes，找出潜在的可能进行抢占调度的节点，下方详解
	potentialNodes := nodesWherePreemptionMightHelp(allNodes, fitError.FailedPredicates)

  // ... 省略
  
  // pdb,pod Disruption Budget,是用来保障可用副本的一种功能，下方详解
	pdbs, err := g.pdbLister.List(labels.Everything())
	if err != nil {
		return nil, nil, nil, err
	}
  
  // 最重要的抢占算法入口，下方详解
	nodeToVictims, err := selectNodesForPreemption(pod, g.nodeInfoSnapshot.NodeInfoMap, potentialNodes, g.predicates,
		g.predicateMetaProducer, g.schedulingQueue, pdbs)
	if err != nil {
		return nil, nil, nil, err
	}

	// ... 省略
  
  // candidateNode，从所有提名的node中挑选一个真正执行抢占步骤
	candidateNode := pickOneNodeForPreemption(nodeToVictims)
	if candidateNode == nil {
		return nil, nil, nil, nil
	}

	// 返回3个值，分别是选中的node、node上将要驱逐的pod、调度队列中比当前pod优先级更低的pod
	nominatedPods := g.getLowerPriorityNominatedPods(pod, candidateNode.Name)
	if nodeInfo, ok := g.nodeInfoSnapshot.NodeInfoMap[candidateNode.Name]; ok {
		return nodeInfo.Node(), nodeToVictims[candidateNode].Pods, nominatedPods, nil
	}

	return nil, nil, nil, fmt.Errorf(
		"preemption failed: the target node %s has been deleted from scheduler cache",
		candidateNode.Name)
}
```

### 3.1.  potentialNodes

第一步，先找出所有潜在的可能会参与抢占调度的node,何为潜在可能呢？意思是node调度此pod调度失败的原因并非"硬伤"类原因。所谓硬伤原因，指的是即使驱逐调几个pod，也无法改变此node无法运行这个pod的事实。这些硬伤包括哪些？来看看代码：

`pkg/scheduler/core/generic_scheduler.go:306 -> pkg/scheduler/core/generic_scheduler.go:1082`

```go
func nodesWherePreemptionMightHelp(nodes []*v1.Node, failedPredicatesMap FailedPredicateMap) []*v1.Node {
	potentialNodes := []*v1.Node{}
	for _, node := range nodes {
		unresolvableReasonExist := false
		failedPredicates, _ := failedPredicatesMap[node.Name]
		// If we assume that scheduler looks at all nodes and populates the failedPredicateMap
		// (which is the case today), the !found case should never happen, but we'd prefer
		// to rely less on such assumptions in the code when checking does not impose
		// significant overhead.
		// Also, we currently assume all failures returned by extender as resolvable.
		for _, failedPredicate := range failedPredicates {
			switch failedPredicate {
			case
        // 下面所有的failedPredicates，都视为"硬伤"，因此若相应的节点上若出现下面的失败原因之一，则视为该node不可参与抢占调度。
				predicates.ErrNodeSelectorNotMatch,
				predicates.ErrPodAffinityRulesNotMatch,
				predicates.ErrPodNotMatchHostName,
				predicates.ErrTaintsTolerationsNotMatch,
				predicates.ErrNodeLabelPresenceViolated,
				// Node conditions won't change when scheduler simulates removal of preemption victims.
				// So, it is pointless to try nodes that have not been able to host the pod due to node
				// conditions. These include ErrNodeNotReady, ErrNodeUnderPIDPressure, ErrNodeUnderMemoryPressure, ....
				predicates.ErrNodeNotReady,
				predicates.ErrNodeNetworkUnavailable,
				predicates.ErrNodeUnderDiskPressure,
				predicates.ErrNodeUnderPIDPressure,
				predicates.ErrNodeUnderMemoryPressure,
				predicates.ErrNodeUnschedulable,
				predicates.ErrNodeUnknownCondition,
				predicates.ErrVolumeZoneConflict,
				predicates.ErrVolumeNodeConflict,
				predicates.ErrVolumeBindConflict:
				unresolvableReasonExist = true
				break
			}
		}
		if !unresolvableReasonExist {
			klog.V(3).Infof("Node %v is a potential node for preemption.", node.Name)
			potentialNodes = append(potentialNodes, node)
		}
	}
	return potentialNodes
}
```

### 3.2.  Pod Disruption Budget(pdb)

这种资源类型本人没有实际应用过，查阅了一下官方的手册，实际上它也是kubernetes设计的一种抽象资源，主要用作面对主动中断时，保障副本可用数量的一种功能，与deployment的maxUnavailable不一样，maxUnavailable是在滚动更新(非主动中断)时用来保障，pdb通常是面对主动中断的场景，例如删除pod,drain node等主动操作，更多详细说明参考官方的手册:

[Specifying a Disruption Budget for your Application](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)

资源实例:

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper	
```

```bash
$ kubectl get poddisruptionbudgets
NAME      MIN-AVAILABLE   ALLOWED-DISRUPTIONS   AGE
zk-pdb    2               1                     7s


$ kubectl get poddisruptionbudgets zk-pdb -o yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  creationTimestamp: 2017-08-28T02:38:26Z
  generation: 1
  name: zk-pdb
...
status:
  currentHealthy: 3
  desiredHealthy: 3
  disruptedPods: null
  disruptionsAllowed: 1
  expectedPods: 3
  observedGeneration: 1
```

为什么这个资源相关的逻辑会出现在抢占调度里面呢？因为设计者将pod抢占造成的低优先级pod驱逐动作视为主动中断，有了这一层理解，我们接着往下。

### 3.3.  nodeToVictims 

selectNodesForPreemption()函数很重要，这个函数将会返回所有可行的node驱逐方案

`pkg/scheduler/core/generic_scheduler.go:316 ` ` selectNodesForPreemption` --> `pkg/scheduler/core/generic_scheduler.go:916`

```go
func selectNodesForPreemption(pod *v1.Pod,
	nodeNameToInfo map[string]*schedulernodeinfo.NodeInfo,
	potentialNodes []*v1.Node,
	fitPredicates map[string]predicates.FitPredicate,
	metadataProducer predicates.PredicateMetadataProducer,
	queue internalqueue.SchedulingQueue,
	pdbs []*policy.PodDisruptionBudget,
) (map[*v1.Node]*schedulerapi.Victims, error) {
  // 返回的结构体，类型是map，key是*v1.Node，value是一个结构体，包含两个元素:node上待驱逐的pod信息和将会违反PDB规则的次数
	nodeToVictims := map[*v1.Node]*schedulerapi.Victims{}
	var resultLock sync.Mutex

	// We can use the same metadata producer for all nodes.
	meta := metadataProducer(pod, nodeNameToInfo)
	checkNode := func(i int) {
		nodeName := potentialNodes[i].Name
		var metaCopy predicates.PredicateMetadata
		if meta != nil {
			metaCopy = meta.ShallowCopy()
		}
    // selectVictimsOnNode()是核心计算的函数
		pods, numPDBViolations, fits := selectVictimsOnNode(pod, metaCopy, nodeNameToInfo[nodeName], fitPredicates, queue, pdbs)
		if fits {
			resultLock.Lock()
			victims := schedulerapi.Victims{
				Pods:             pods,
				NumPDBViolations: numPDBViolations,
			}
			nodeToVictims[potentialNodes[i]] = &victims
			resultLock.Unlock()
		}
	}
  // 熟悉的并发控制模型
	workqueue.ParallelizeUntil(context.TODO(), 16, len(potentialNodes), checkNode)
	return nodeToVictims, nil
}
```

上面已在代码中对重要部分进行注释，不难发现，重要的计算函数是selectVictimsOnNode()函数，每个node所需要驱逐的pod，以及违反PDB规则次数信息，都由此函数来计算返回，最终组成nodeToVictims这个map，返回给上层调用函数。所以，接着来看selectVictimsOnNode()函数是怎么运行的。

**selectVictimsOnNode**

```go
func selectVictimsOnNode(
	pod *v1.Pod,
	meta predicates.PredicateMetadata,
	nodeInfo *schedulernodeinfo.NodeInfo,
	fitPredicates map[string]predicates.FitPredicate,
	queue internalqueue.SchedulingQueue,
	pdbs []*policy.PodDisruptionBudget,
) ([]*v1.Pod, int, bool) {
	if nodeInfo == nil {
		return nil, 0, false
	}
  
  // 潜在的受害者(pod)，按优先级排序的有序list，高优先级的排序靠前，低优先级的排序靠后
	potentialVictims := util.SortableList{CompFunc: util.HigherPriorityPod}
  // 在实际调度之前，所有的资源的考量计算都只能是预估，因此不能实际实施到node上，所以，基于node的元数据进行一个复制，将node信息的复制样本来参与计算，最终计算得到正确的结果后才会考虑实际往node上实施。
	nodeInfoCopy := nodeInfo.Clone()
  
  // 基于node复制样本，假设减去一个pod之后，复制样本重新计算得到的数据。例如node a上运行着有若干pod，假设减去了其上的pod1，pod1 request的内存是4Gi，那么可假设node可分配的内存就多了4Gi
	removePod := func(rp *v1.Pod) {
		nodeInfoCopy.RemovePod(rp)
		if meta != nil {
			meta.RemovePod(rp)
		}
	}
  
  // 基于node复制样本，假设加上一个pod之后，复制样本重新计算得到的数据。
	addPod := func(ap *v1.Pod) {
		nodeInfoCopy.AddPod(ap)
		if meta != nil {
			meta.AddPod(ap, nodeInfoCopy)
		}
	}

  // 首先，枚举出node上所有的低于待调度pod优先级的pod，并将它们加入潜在受害者potentialVictims，计算假设剔出它们后，node上现有的资源信息
	podPriority := util.GetPodPriority(pod)
	for _, p := range nodeInfoCopy.Pods() {
		if util.GetPodPriority(p) < podPriority {
			potentialVictims.Items = append(potentialVictims.Items, p)
			removePod(p)
		}
	}
	potentialVictims.Sort()

  // 第二步，判断待调度pod是否fit此node，主要是亲和性方面的考量，这个podFitsOnNode函数前面筛选算法已经讲过了，这里不再复述，这个函数通过后，会把待调度pod的request资源加入nodeInfoCopy内。
	if fits, _, err := podFitsOnNode(pod, meta, nodeInfoCopy, fitPredicates, queue, false); !fits {
		if err != nil {
			klog.Warningf("Encountered error while selecting victims on node %v: %v", nodeInfo.Node().Name, err)
		}
		return nil, 0, false
	}
	var victims []*v1.Pod
	numViolatingVictim := 0
  
  // 第三步，将前面枚举出的低优先级的pod有序list，拆分为两个有序list，一个是违反了PDB规则的(pdb.Status.PodDisruptionsAllowed <= 0,这个值等于0则代表理论上要求不能出现中断的pod副本)，一个是不违反PDB规则的。
	violatingVictims, nonViolatingVictims := filterPodsWithPDBViolation(potentialVictims.Items, pdbs)
  
  // 第四步，前面枚举假设把所有的低优先级pod都剔除了，但实际上可能不用剔除这么多，因此，保证了待调度pod计算进来之后，这里再用贪心法将低优先级的pod按优先级排序尽可能多地加入回来，最终无法调度的pod，才归为实际驱逐的pod。显而易见的是，优先保障有PDB约束的pod。
	reprievePod := func(p *v1.Pod) bool {
		addPod(p)
		fits, _, _ := podFitsOnNode(pod, meta, nodeInfoCopy, fitPredicates, queue, false)
		if !fits {
			removePod(p)
			victims = append(victims, p)
			klog.V(5).Infof("Pod %v/%v is a potential preemption victim on node %v.", p.Namespace, p.Name, nodeInfo.Node().Name)
		}
		return fits
	}
	for _, p := range violatingVictims {
		if !reprievePod(p) {
			numViolatingVictim++
		}
	}
	// Now we try to reprieve non-violating victims.
	for _, p := range nonViolatingVictims {
		reprievePod(p)
	}
  // 第五步，返回最终node的运算结果，分别是驱逐的pod list，以及驱逐的数量
	return victims, numViolatingVictim, true
}
```

这个函数分5步，先是枚举出所有的低优先级pod，再贪心保障尽量多的pod能正常运行，从而计算出最终需要被驱逐的pod及相关信息，详见代码内注释。

### 3.4.  candidateNode

上面函数返回每一个可抢占的node各自的抢占方案后，这里就需要筛选其中一个node来实际执行抢占调度操作。

`pkg/scheduler/core/generic_scheduler.go:330 pickOneNodeForPreemption()` --> `pkg/scheduler/core/generic_scheduler.go:809`

```go
func pickOneNodeForPreemption(nodesToVictims map[*v1.Node]*schedulerapi.Victims) *v1.Node {
	if len(nodesToVictims) == 0 {
		return nil
	}
	minNumPDBViolatingPods := math.MaxInt32
	var minNodes1 []*v1.Node
	lenNodes1 := 0
	for node, victims := range nodesToVictims {
		if len(victims.Pods) == 0 {
		  // 可能在调度的过程中，有极小的概率某个node上有pod终结了，使node上不再有需要驱逐的pod，那么pod可直接调度到该node上
			return node
		}
		// 按违反PDB约束的次数排序，越少的node优先级越高，若最大优先级的node只有一个，则直接返回违反次数最小的node，若有多个，则进入下一步筛选
		numPDBViolatingPods := victims.NumPDBViolations
		if numPDBViolatingPods < minNumPDBViolatingPods {
			minNumPDBViolatingPods = numPDBViolatingPods
			minNodes1 = nil
			lenNodes1 = 0
		}
		if numPDBViolatingPods == minNumPDBViolatingPods {
			minNodes1 = append(minNodes1, node)
			lenNodes1++
		}
	}
	if lenNodes1 == 1 {
		return minNodes1[0]
	}

	// 按node上需驱逐的第一个pod(即需驱逐的优先级最高的pod)的优先级大小排序，pod[0]优先级越小，则所属的node优先级越高，若最大优先级的node只有一个，则直接返回此node，若有多个，则进入下一步筛选
	minHighestPriority := int32(math.MaxInt32)
	var minNodes2 = make([]*v1.Node, lenNodes1)
	lenNodes2 := 0
	for i := 0; i < lenNodes1; i++ {
		node := minNodes1[i]
		victims := nodesToVictims[node]
		// highestPodPriority is the highest priority among the victims on this node.
		highestPodPriority := util.GetPodPriority(victims.Pods[0])
		if highestPodPriority < minHighestPriority {
			minHighestPriority = highestPodPriority
			lenNodes2 = 0
		}
		if highestPodPriority == minHighestPriority {
			minNodes2[lenNodes2] = node
			lenNodes2++
		}
	}
	if lenNodes2 == 1 {
		return minNodes2[0]
	}

	// 按node上需驱逐的所有的pod的优先级总和计算，总和越小，node优先级越高，若最大优先级的node只有一个，则直接返回此node，若有多个，则进入下一步筛选
	minSumPriorities := int64(math.MaxInt64)
	lenNodes1 = 0
	for i := 0; i < lenNodes2; i++ {
		var sumPriorities int64
		node := minNodes2[i]
		for _, pod := range nodesToVictims[node].Pods {
			// We add MaxInt32+1 to all priorities to make all of them >= 0. This is
			// needed so that a node with a few pods with negative priority is not
			// picked over a node with a smaller number of pods with the same negative
			// priority (and similar scenarios).
			sumPriorities += int64(util.GetPodPriority(pod)) + int64(math.MaxInt32+1)
		}
		if sumPriorities < minSumPriorities {
			minSumPriorities = sumPriorities
			lenNodes1 = 0
		}
		if sumPriorities == minSumPriorities {
			minNodes1[lenNodes1] = node
			lenNodes1++
		}
	}
	if lenNodes1 == 1 {
		return minNodes1[0]
	}

	// 按node上需驱逐的所有的pod数量计算，数量越少，node优先级越高，若最大优先级的node只有一个，则直接返回此node，若有多个，则进入下一步筛选
	minNumPods := math.MaxInt32
	lenNodes2 = 0
	for i := 0; i < lenNodes1; i++ {
		node := minNodes1[i]
		numPods := len(nodesToVictims[node].Pods)
		if numPods < minNumPods {
			minNumPods = numPods
			lenNodes2 = 0
		}
		if numPods == minNumPods {
			minNodes2[lenNodes2] = node
			lenNodes2++
		}
	}
	// 若经过上面四个步骤的筛选，筛选出的node还是不止一个，那么就挑选其中的第一个作为最后选中被执行抢占调度的node
	if lenNodes2 > 0 {
		return minNodes2[0]
	}
	klog.Errorf("Error in logic of node scoring for preemption. We should never reach here!")
	return nil
}
```

上面代码结合注释，可以归纳出，这个函数中做了非常细致地检查，最高分如下4个步骤来对node进行优先级排序，筛选出一个最终合适的node来被执行抢占调度pod的操作：

1.按违反PDB约束的次数排序

2.按node上需驱逐的第一个pod(即需驱逐的优先级最高的pod)的优先级大小排序

3.按node上需驱逐的所有的pod的优先级总和计算排序

4.按node上需驱逐的所有的pod数量计算排序

5.若经过上面四个步骤的筛选，筛选出的node还是不止一个，那么就挑选其中的第一个作为最后选中node

## 4. 总结

抢占调度的逻辑可以说是非常细致和精彩，例如

1.从资源计算的角度：

- 基于nodeInfo快照的计算，所有计算在最终确定实施之前都是预计算
- 先枚举出所有低优先级的pod，保障待调度pod能充分获取资源
- 在待调度pod能运行后，再尽力保障最多的低优先级pod能同时运行

2.从node选取的角度：

- 分4个步骤筛选以选出驱逐造成影响最小一个node

本章完，感谢阅读！













