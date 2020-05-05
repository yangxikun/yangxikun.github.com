---
layout: post
title: "Kubernetes Scheduler"
description: "Kubernetes Scheduler"
category: Kubernetes
tags: []
---
{% include JB/setup %}

#### schedulerCache：Pod 与 Node 的缓存

```go
type schedulerCache struct {
	stop   <-chan struct{}
	ttl    time.Duration
	period time.Duration

	// This mutex guards all fields within this cache struct.
	mu sync.RWMutex
	// a set of assumed pod keys.
	// The key could further be used to get an entry in podStates.
	assumedPods map[string]bool
	// a map from pod key to podState.
	podStates map[string]*podState
	nodes     map[string]*nodeInfoListItem
	// headNode points to the most recently updated NodeInfo in "nodes". It is the
	// head of the linked list.
	headNode *nodeInfoListItem
	nodeTree *nodeTree
	// A map from image name to its imageState.
	imageStates map[string]*imageState
}
```

* nodes：维护节点名称到节点信息的映射
* headNode：按照更新时间由大到小排序节点信息
* nodeTree：维护 zone 和 node 关系
    * 一个 node 只属于一个 zone
    * 一个 zone 可以包含多个 node
* imageStates：维护镜像名称到节点的关系

![](/assets/img/kubernetes-scheduler-node.png)

* podStates：
    * pod：Pod 的最新状态
    * deadline：在结束绑定之后，等待多久没有从 apiserver 接收到绑定完成的事件，就会由 cache.cleanupExpiredAssumedPods 从缓存中删除
    * bindingFinished：是否已经完成绑定
* assumedPods：值为 true 说明调度器为 Pod 分配了一个 Node

Pod 的调度状态变化：UnScheduled Pod -> (assume) -> Assumed Pod -> (bind) -> Scheduled Pod。

![](/assets/img/kubernetes-scheduler-pod.png)

#### PriorityQueue：调度队列

```go
type PriorityQueue struct {
	stop  <-chan struct{}
	clock util.Clock
	// podBackoff tracks backoff for pods attempting to be rescheduled
	podBackoff *PodBackoffMap

	lock sync.RWMutex
	cond sync.Cond

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	activeQ *heap.Heap
	// podBackoffQ is a heap ordered by backoff expiry. Pods which have completed backoff
	// are popped from this heap before the scheduler looks at activeQ
	podBackoffQ *heap.Heap
	// unschedulableQ holds pods that have been tried and determined unschedulable.
	unschedulableQ *UnschedulablePodsMap
	// nominatedPods is a structures that stores pods which are nominated to run
	// on nodes.
	nominatedPods *nominatedPodMap
	// schedulingCycle represents sequence number of scheduling cycle and is incremented
	// when a pod is popped.
	schedulingCycle int64
	// moveRequestCycle caches the sequence number of scheduling cycle when we
	// received a move request. Unscheduable pods in and before this scheduling
	// cycle will be put back to activeQueue if we were trying to schedule them
	// when we received move request.
	moveRequestCycle int64

	// closed indicates that the queue is closed.
	// It is mainly used to let Pop() exit its control loop while waiting for an item.
	closed bool
}
```

* activeQ：存放等待调度的 Pod 的优先级队列
* podBackoffQ：当 moveRequestCycle >= schedulingCycle 时，因调度失败而需要进行尝试重新调度的 Pod 会入队到该优先级队列，因为已经有事件（比如 pv/pvc 的创建，新增 node）发生了，所以会将这些 Pod 在其 backoffTime 到期后直接重试
* unschedulableQ：不可调度队列，当 moveRequestCycle < schedulingCycle 时，调度失败的 Pod 会入队到该“队列”，需要等待有事件（比如 pv/pvc 的创建，新增 node）发生才能重试
* nominatedPods：用于支持抢占调度，由于没有找到满足 Pod 调度的 Node，需要进行抢占，记录抢占 Pod 分配到了那个 Node，会导致 Node 上的一些牺牲者 Pods 被删除，但并不保证抢占 Pod 会被调度到 Node 上
* schedulingCycle：调度周期，activeQ 每次 Pop 出一个 Pod，递增 1
* moveRequestCycle：有事件（比如 pv/pvc 的创建，新增 node）发生了，需要把 unschedulableQ 中的 Pod 移动到 podBackoffQ 或 activeQ，然后赋值为 schedulingCycle

##### unschedulableQ

![](/assets/img/kubernetes-scheduler-queue-unschedulableQ.png)

##### podBackoffQ

![](/assets/img/kubernetes-scheduler-queue-podbackoffQ.png)

##### activeQ

![](/assets/img/kubernetes-scheduler-queue-activeQ.png)

##### nominatedPods

![](/assets/img/kubernetes-scheduler-queue-nominatedpods.png)

#### framework

在调度阶段和绑定阶段执行过程中提供 Hook，执行一些额外的插件。

#### 调度阶段

多个 Pod 的调度阶段是串行进行的。

检查 Node 是否合适：

* CheckNodeUnschedulablePredicate：检查 Tolerations 和 Taint，以及 Node.Spec.Unschedulable
* GeneralPredicates：
    * noncriticalPredicates：
        * PodFitsResources：
            * 检查 NodeInfo.allocatableResource.AllowedPodNumber
            * 检查 NodeInfo.allocatableResource 是否满足 podRequest
                * podRequest.MilliCPU：CPU
                * podRequest.Memory：内存
                * podRequest.EphemeralStorage：临时存储空间需求
                    * 非 tmpfs 的 emptyDir
                    * 容器日志
                    * 容器可写层
                * podRequest.ScalarResources：扩展资源
                    * Node-level extended resources：
                        * Device plugin managed resources
                        * Other resources
    * EssentialPredicates：
        * PodFitsHost：如果 pod.Spec.NodeName 不为空，检查 pod.Spec.NodeName == node.Name
        * PodFitsHostPorts：判断容器在宿主机上映射的端口是否有冲突
        * PodMatchNodeSelector：检查 pod.Spec.NodeSelector 和 pod.Spec.Affinity 是否匹配 node.Labels
* NoDiskConflict：检查 GCE、Amazon EBS、ISCSI 和 Ceph RBD 的挂载是否有冲突
* PodToleratesNodeTaints：检查 pod.Spec.Tolerations 是否容忍 nodeInfo.Taints() 类型为 v1.TaintEffectNoSchedule 和 v1.TaintEffectNoExecute 的 Taints
* checkServiceAffinity：尽量将 Pod 部署到一组相同 Labels 的节点上
* MaxPDVolumeCountChecker：检查 EBSVolumeFilterType、GCEPDVolumeFilterType、AzureDiskVolumeFilterType、CinderVolumeFilterType 在宿主机上的挂载是否超出了限制
* CSIMaxVolumeLimitChecker：检查挂载的 CSI 卷是否超过限制
* VolumeBindingChecker：检查 PVCs 是否有匹配的 PVs 或者能够动态配置；检查已绑定 PVC 的 PV NodeAffinity 是否满足
* VolumeZoneChecker：可用区挂载检查，检查 PVs Labels(LabelZoneFailureDomain, LabelZoneRegion) 是否满足 node.Labels
* EvenPodsSpreadPredicate：检查 Pod 在集群内故障域的分布是否满足限制
* InterPodAffinityMatches：先检查 Node 上已经存在的 Pod 的 PodAntiAffinity 是否允许，再检查待调度 Pod 的 PodAffinity 和 PodAntiAffinity 是否满足

为 Node 打分：

* BalancedAllocation：平衡 Node 不同资源的分配，避免出现 CPU 分配了 90%，而内存只分配了 10%
* ImageLocality：Node 上已经有 Pod 所需镜像的打分会越高
* InterPodAffinity：将 Pod 尽量调度到相同的拓扑区域
* LeastAllocated：将 Pod 尽量调度到有更多空闲资源的 Node
* NodeAffinity：根据 pod.Spec.Affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution 给节点打分
* NodePreferAvoidPods：如果 Pod 匹配 node.Annotations\["scheduler.alpha.kubernetes.io/preferAvoidPods"\]，则打 0 分，否则 100 分
* DefaultPodTopologySpread：确保 Pod 在区域内的 Nodes 中均匀分布
* 根据 pod.Spec.Tolerations TaintEffectPreferNoSchedule 打分，容忍度高的 Node 得分高

![](/assets/img/kubernetes-scheduler-syncloop.png)

#### 绑定阶段

多个 Pod 的绑定阶段是并发进行的。

![](/assets/img/kubernetes-scheduler-bind.png)

#### 抢占

抢占发生在调度阶段失败后，失败原因是没有合适的 Node，并且没有禁用抢占（默认没有禁用）。

![](/assets/img/kubernetes-scheduler-preempt.png)
