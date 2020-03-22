---
layout: post
title: "Kubernetes API 资源对象的删除和 GarbageCollector Controller"
description: "Kubernetes API 资源对象的删除和 GarbageCollector Controller"
category: Kubernetes
tags: []
---
{% include JB/setup %}

参考文档：

* [termination-of-pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)
* [Garbage Collection](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/)
* 两篇设计文档，不过感觉已经有点过时了：[garbage-collection](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/garbage-collection.md) 和 [synchronous-garbage-collection](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/synchronous-garbage-collection.md)

#### Kubernetes API 资源对象的删除方式

##### Foreground cascading deletion

显示级联删除，根对象首先进入 deletion in progress 状态。在 deletion in progress 状态会有如下的情况：

* 对象仍然可以通过 REST API 可见。
* 会设置对象的 ObjectMeta.DeletionTimestamp 字段。
* 对象的 metadata.finalizers 字段包含了值 foregroundDeletion。
 
一旦对象被设置为 deletion in progress 状态，垃圾收集器在删除了所有 Blocking 状态的 dependents（ownerReference.blockOwnerDeletion=true）之后，对象才会被删除。
 
##### Background cascading deletion
 
隐式级联删除，kubernetes-apiserver 会立即删除对象，然后垃圾收集器会在后台删除 dependents。
 
##### Orphan
 
非级联删除，根对象首先进入 deletion in progress 状态。在 deletion in progress 状态会有如下的情况：
 
* 对象仍然可以通过 REST API 可见。
* 会设置对象的 deletionTimestamp 字段。
* 对象的 metadata.finalizers 字段包含了值 orphan。

一旦对象被设置为 deletion in progress 状态，垃圾收集器在解除所有 dependents 的关联之后，对象才会被删除。

##### Gracefully terminate

针对 Pod 特有的删除方式，允许优雅地终止容器。先发送 TERM 信号，过了宽限期还未终止，则发送 KILL 信号。

kube-apiserver 先设置 ObjectMeta.DeletionGracePeriodSeconds，默认为 30s，再由 kubelet 发送删除请求，请求参数中 DeleteOptions.GracePeriodSeconds = 0，kube-apiserver 判断到 lastGraceful = *options.GracePeriodSeconds = 0，就直接删除对象了。

<!--more-->
 
#### 如何控制删除方式

控制 Gracefully terminate 删除方式有两种，按优先级高低排序如下：

1. 在发起删除请求的参数中配置：DeleteOptions.GracePeriodSeconds
1. 在 pod.Spec.TerminationGracePeriodSeconds 配置

> 详情可查看 podStrategy.CheckGracefulDelete

控制 Foreground/Background/Orphan 删除方式有三种，按优先级由高到低排序如下：
 
 1\. 在发起删除请求的参数中配置
 
 * DeleteOptions
    * GracePeriodSeconds：优雅地终止容器允许的宽限期
    * Preconditions：并发更新时的冲突检测
    * OrphanDependents：Orphan 方式删除，标记为废弃字段
    * PropagationPolicy：
        * DeletePropagationOrphan：Orphan
        * DeletePropagationBackground：Background cascading deletion
        * DeletePropagationForeground：Foreground cascading deletion
 
 ```go
// DeleteOptions may be provided when deleting an API object.
type DeleteOptions struct {
	TypeMeta `json:",inline"`

	// The duration in seconds before the object should be deleted. Value must be non-negative integer.
	// The value zero indicates delete immediately. If this value is nil, the default grace period for the
	// specified type will be used.
	// Defaults to a per object value if not specified. zero means delete immediately.
	// +optional
	GracePeriodSeconds *int64 `json:"gracePeriodSeconds,omitempty" protobuf:"varint,1,opt,name=gracePeriodSeconds"`

	// Must be fulfilled before a deletion is carried out. If not possible, a 409 Conflict status will be
	// returned.
	// +k8s:conversion-gen=false
	// +optional
	Preconditions *Preconditions `json:"preconditions,omitempty" protobuf:"bytes,2,opt,name=preconditions"`

	// Deprecated: please use the PropagationPolicy, this field will be deprecated in 1.7.
	// Should the dependent objects be orphaned. If true/false, the "orphan"
	// finalizer will be added to/removed from the object's finalizers list.
	// Either this field or PropagationPolicy may be set, but not both.
	// +optional
	OrphanDependents *bool `json:"orphanDependents,omitempty" protobuf:"varint,3,opt,name=orphanDependents"`

	// Whether and how garbage collection will be performed.
	// Either this field or OrphanDependents may be set, but not both.
	// The default policy is decided by the existing finalizer set in the
	// metadata.finalizers and the resource-specific default policy.
	// Acceptable values are: 'Orphan' - orphan the dependents; 'Background' -
	// allow the garbage collector to delete the dependents in the background;
	// 'Foreground' - a cascading policy that deletes all dependents in the
	// foreground.
	// +optional
	PropagationPolicy *DeletionPropagation `json:"propagationPolicy,omitempty" protobuf:"varint,4,opt,name=propagationPolicy"`

	// When present, indicates that modifications should not be
	// persisted. An invalid or unrecognized dryRun directive will
	// result in an error response and no further processing of the
	// request. Valid values are:
	// - All: all dry run stages will be processed
	// +optional
	DryRun []string `json:"dryRun,omitempty" protobuf:"bytes,5,rep,name=dryRun"`
}
```

2\. 在 metadata.finalizers 中配置，可以设置的值如下：

* metav1.FinalizerOrphanDependents：Orphan
* metav1.FinalizerDeleteDependents：Foreground cascading deletion

> 注意只能设置其中一个

3\. 实现删除策略 rest.GarbageCollectionDeleteStrategy

* GarbageCollectionPolicy
    * DeleteDependents：目前未实际用到
    * OrphanDependents：Orphan
    * Unsupported：资源对象无法被 GC，目前只有 event

```go
// GarbageCollectionDeleteStrategy must be implemented by the registry that wants to
// orphan dependents by default.
type GarbageCollectionDeleteStrategy interface {
	// DefaultGarbageCollectionPolicy returns the default garbage collection behavior.
	DefaultGarbageCollectionPolicy(ctx context.Context) GarbageCollectionPolicy
}
```

例如 pkg/registry/apps/deployment.deploymentStrategy 的实现会根据 groupVersion 的情况：

* 如果为 extensionsv1beta1.SchemeGroupVersion/appsv1beta1.SchemeGroupVersion/appsv1beta2.SchemeGroupVersion，返回 rest.OrphanDependents，
* 否则返回 rest.DeleteDependents
 
#### 垃圾收集器

##### 控制循环流程图

![](/assets/img/kubernetes-garbage-collector-controller.png)

##### 对象状态机

当通过 kube-apiserver 删除对象时，如果是 orphan 或 foregroundDeletion 的方式，那么对象会进入对应的删除状态。然后 GarbageController 会执行对应的删除操作，删除条件满足后，会删除对应的 finalizer。kube-apiserver 判断到对象处于删除状态，并且 len(finalizers) = 0，则从 Etcd 中删除对象。

![](/assets/img/kubernetes-garbage-collector-obj-state-machine.png)

##### 事件处理流程

![](/assets/img/kubernetes-garbage-collector-process-obj.png)

##### GraphBuilder.processGraphChanges

GraphBuilder 维护着对象之间的依赖关系图，图中每个节点用 node 表示。GraphBuilder 会通过 Informer 机制，根据对象的增删改事件修改依赖关系图，然后根据对象变化情况，将对象的 node，以及相关的 owners/dependents 入队到 attemptToOrphan 或 attemptToDelete 中。

* gb.uidToNode：对象 UID 到 node 的映射
* gb.insertNode(newNode)：
    * gb.uidToNode.Write(newNode)
    * gb.addDependentToOwners(newNode, newNode.owners)：将 newNode 添加到 ownerNode.dependents 中，如果 ownerNode 不存在，则会创建虚拟节点，并且入队到 attemptToDelete
        * 因为子节点有可能比父节点先被观察到，而且父节点没有子节点信息，所以为了维护关系图，就需要为尚未观察到的父节点创建虚拟节点
        * 当父节点被观察到后，虚拟节点会被标记为普通节点
* gb.processTransitions(event.oldObj, accessor, node)：处理 obj 的状态变迁
    * startsWaitingForDependentsOrphaned(event.oldObj, accessor)：如果对象从非 orphan 状态进入到 orphan 状态，返回 true
        * deletionStartsWithFinalizer(event.oldObj, accessor, metav1.FinalizerOrphanDependents)
            * 如果 !beingDeleted(accessor) \|\| !hasFinalizer(accessor, metav1.FinalizerOrphanDependents)，返回 false
            * 如果 event.oldObj == nil \|\| !beingDeleted(oldAccessor) \|\| !hasFinalizer(oldAccessor, metav1.FinalizerOrphanDependents)，返回 true
                * gb.attemptToOrphan.Add(node)
    * startsWaitingForDependentsDeleted(event.oldObj, accessor)：如果对象从非 foregroundDeletion 状态进入到 foregroundDeletion 状态，返回true
        * deletionStartsWithFinalizer(event.oldObj, accessor, metav1.FinalizerDeleteDependents)
            * 如果 !beingDeleted(accessor) \|\| !hasFinalizer(accessor, metav1.FinalizerDeleteDependents)，返回 false
            * 如果 event.oldObj == nil \|\| !beingDeleted(oldAccessor) \|\| !hasFinalizer(oldAccessor, metav1.FinalizerDeleteDependents)，返回 true
                * node.markDeletingDependents()：标记正在删除 dependents
                * gb.attemptToDelete.Add(node.dependents)
                * gb.attemptToDelete.Add(node)
* referencesDiffs(existingNode.owners, accessor.GetOwnerReferences())：比较 ownerReferences 的变化，返回值如下
    * added：new - old
    * removed：old - new
    * changed：UID 相同，OwnerReference.Controller、OwnerReference.BlockOwnerDeletion 有不同
* gb.addUnblockedOwnersToDeleteQueue(removed, changed)：
    * for ref in removed
        * 如果 ref.BlockOwnerDeletion != nil && *ref.BlockOwnerDeletion，与父节点解除了关联，那么尝试删除父节点 gb.attemptToDelete.Add(gb.uidToNode.Read(ref.UID))
            * 这里是不是应该判断下父节点是否处于 foregroundDeletion 状态？
    * for pair in changed
        * wasBlocked = pair.oldRef.BlockOwnerDeletion != nil && *pair.oldRef.BlockOwnerDeletion
        * isUnblocked = pair.newRef.BlockOwnerDeletion == nil \|\| (pair.newRef.BlockOwnerDeletion != nil && !*pair.newRef.BlockOwnerDeletion)
        * 如果 wasBlocked && isUnblocked，那么 gb.attemptToDelete.Add(gb.uidToNode.Read(pair.newRef.UID))
            * 即 ownerReferences 的状态由 block 转移到 unblock，那么尝试删除 owner
            * 这里是不是应该判断下父节点是否处于 foregroundDeletion 状态？
* gb.removeDependentFromOwners(n *node, owners []metav1.OwnerReference)：
    * for owner in owners
        * gb.uidToNode.Read(owner.UID).deleteDependent(n)
* gb.removeNode(n *node)：
	* gb.uidToNode.Delete(n.identity.UID)
	* gb.removeDependentFromOwners(n, n.owners)

```go
// Dequeueing an event from graphChanges, updating graph, populating dirty_queue.
func (gb *GraphBuilder) processGraphChanges() bool {
	item, quit := gb.graphChanges.Get()
	if quit {
		return false
	}
	defer gb.graphChanges.Done(item)
	event, ok := item.(*event)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect a *event, got %v", item))
		return true
	}
	obj := event.obj
	accessor, err := meta.Accessor(obj)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("cannot access obj: %v", err))
		return true
	}
	klog.V(5).Infof("GraphBuilder process object: %s/%s, namespace %s, name %s, uid %s, event type %v", event.gvk.GroupVersion().String(), event.gvk.Kind, accessor.GetNamespace(), accessor.GetName(), string(accessor.GetUID()), event.eventType)
	// Check if the node already exists
	existingNode, found := gb.uidToNode.Read(accessor.GetUID())
	if found {
		// this marks the node as having been observed via an informer event
		// 1. this depends on graphChanges only containing add/update events from the actual informer
		// 2. this allows things tracking virtual nodes' existence to stop polling and rely on informer events
		existingNode.markObserved()
	}
	switch {
	case (event.eventType == addEvent \|\| event.eventType == updateEvent) && !found:
		newNode := &node{
			identity: objectReference{
				OwnerReference: metav1.OwnerReference{
					APIVersion: event.gvk.GroupVersion().String(),
					Kind:       event.gvk.Kind,
					UID:        accessor.GetUID(),
					Name:       accessor.GetName(),
				},
				Namespace: accessor.GetNamespace(),
			},
			dependents:         make(map[*node]struct{}),
			owners:             accessor.GetOwnerReferences(),
			deletingDependents: beingDeleted(accessor) && hasDeleteDependentsFinalizer(accessor),
			beingDeleted:       beingDeleted(accessor),
		}
		gb.insertNode(newNode)
		// the underlying delta_fifo may combine a creation and a deletion into
		// one event, so we need to further process the event.
		gb.processTransitions(event.oldObj, accessor, newNode)
	case (event.eventType == addEvent \|\| event.eventType == updateEvent) && found:
		// handle changes in ownerReferences
		// changed: UID 相同，其他不同
		added, removed, changed := referencesDiffs(existingNode.owners, accessor.GetOwnerReferences())
		if len(added) != 0 \|\| len(removed) != 0 \|\| len(changed) != 0 {
			// check if the changed dependency graph unblock owners that are
			// waiting for the deletion of their dependents.
			// 与父节点解除了关联，尝试删除父节点
            // 与父节点的关联关系由 block -> unblock，尝试删除父节点
			gb.addUnblockedOwnersToDeleteQueue(removed, changed)
			// update the node itself
			existingNode.owners = accessor.GetOwnerReferences()
			// Add the node to its new owners' dependent lists.
			gb.addDependentToOwners(existingNode, added)
			// remove the node from the dependent list of node that are no longer in
			// the node's owners list.
			gb.removeDependentFromOwners(existingNode, removed)
		}

		if beingDeleted(accessor) {
			existingNode.markBeingDeleted()
		}
		gb.processTransitions(event.oldObj, accessor, existingNode)
	case event.eventType == deleteEvent:
		if !found {
			klog.V(5).Infof("%v doesn't exist in the graph, this shouldn't happen", accessor.GetUID())
			return true
		}
		// removeNode updates the graph
		gb.removeNode(existingNode)
		existingNode.dependentsLock.RLock()
		defer existingNode.dependentsLock.RUnlock()
		if len(existingNode.dependents) > 0 {
			gb.absentOwnerCache.Add(accessor.GetUID())
		}
		for dep := range existingNode.dependents {
			gb.attemptToDelete.Add(dep)
		}
		for _, owner := range existingNode.owners {
			ownerNode, found := gb.uidToNode.Read(owner.UID)
			if !found \|\| !ownerNode.isDeletingDependents() {
				continue
			}
			// this is to let attempToDeleteItem check if all the owner's
			// dependents are deleted, if so, the owner will be deleted.
			gb.attemptToDelete.Add(ownerNode)
		}
	}
	return true
}
```

##### GarbageCollector.attemptToOrphanWorker

* gc.orphanDependents(owner objectReference, dependents []*node)：遍历 dependents，调用 deleteOwnerRef 接口删除 ownerReference
* gc.removeFinalizer(owner *node, targetFinalizer string)：删除 owner.Finalizers 中的 targetFinalizer

```go
// attemptToOrphanWorker dequeues a node from the attemptToOrphan, then finds its
// dependents based on the graph maintained by the GC, then removes it from the
// OwnerReferences of its dependents, and finally updates the owner to remove
// the "Orphan" finalizer. The node is added back into the attemptToOrphan if any of
// these steps fail.
func (gc *GarbageCollector) attemptToOrphanWorker() bool {
	item, quit := gc.attemptToOrphan.Get()
	gc.workerLock.RLock()
	defer gc.workerLock.RUnlock()
	if quit {
		return false
	}
	defer gc.attemptToOrphan.Done(item)
	owner, ok := item.(*node)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect *node, got %#v", item))
		return true
	}
	// we don't need to lock each element, because they never get updated
	owner.dependentsLock.RLock()
	dependents := make([]*node, 0, len(owner.dependents))
	for dependent := range owner.dependents {
		dependents = append(dependents, dependent)
	}
	owner.dependentsLock.RUnlock()

	err := gc.orphanDependents(owner.identity, dependents)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("orphanDependents for %s failed with %v", owner.identity, err))
		gc.attemptToOrphan.AddRateLimited(item)
		return true
	}
	// update the owner, remove "orphaningFinalizer" from its finalizers list
	err = gc.removeFinalizer(owner, metav1.FinalizerOrphanDependents)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("removeOrphanFinalizer for %s failed with %v", owner.identity, err))
		gc.attemptToOrphan.AddRateLimited(item)
	}
	return true
}
```

##### GarbageCollector.attemptToDeleteWorker

* gc.attemptToDeleteItem(item *node)：
    * 如果 item.isDeletingDependents() == true：显示级联删除，尝试删除所有 block 的子节点
        * gc.processDeletingDependentsItem(item *node)：
            * 如果 len(item.blockingDependents()) == 0，那么返回 gc.removeFinalizer(item, metav1.FinalizerDeleteDependents)
            * 否则遍历 item.blockingDependents()，如果 dep.isDeletingDependents() == false，那么 gc.attemptToDelete.Add(dep)
    * gc.classifyReferences(item, latest.GetOwnerReferences())：返回如下分类
        * solid：owner 不处于 "waitingForDependentsDeletion" 状态
        * dangling：owner 在 gc.absentOwnerCache 中，或者不在 kube-apiserver 中，即 owner 不存在了
            * gb 会把删除事件对应的节点缓存到 gb.absentOwnerCache( = gc.absentOwnerCache) 中
        * waitingForDependentsDeletion：ownerAccessor.GetDeletionTimestamp() != nil && hasDeleteDependentsFinalizer(ownerAccessor)
    * 根据 gc.classifyReferences 结果的不同，有如下三种处理情况：
        * len(solid) != 0：
            * 如果 len(dangling) == 0 && len(waitingForDependentsDeletion) == 0，那么直接返回
            * 否则，解除与 dangling、waitingForDependentsDeletion 的关联关系，但不能删除，因为还有 solid
        * len(waitingForDependentsDeletion) != 0 && item.dependentsLength() != 0：
            * 父节点全部处于 foregroundDeletion 状态，并且有子节点也处于 foregroundDeletion 状态，那么解除与父节点的关联，避免发生“死锁”
                * 考虑这种情况：obj1\[foregroundDeletion\]<-obj2<-obj3\[foregroundDeletion\]，同时 obj3\[foregroundDeletion\]<-obj1\[foregroundDeletion\]
                * 因为父节点采用 foregroundDeletion 显示级联删除的方式，那么其子节点也必须采用 foregroundDeletion 进行显示级联删除，则会出现如下情况：
                    * obj1\[foregroundDeletion\]<-obj2\[foregroundDeletion\]<-obj3\[foregroundDeletion\]，同时 obj3\[foregroundDeletion\]<-obj1\[foregroundDeletion\]
                    * 即出现了删除死锁，gc 假设这种情况是存在的，所以为了解决这个问题，就需要解除 obj1<-obj2 的关联
                    * 当然如果不存在循环依赖的话，那么就会出现显示级联删除的时候，父节点删除了，而子节点尚未删除
                    * 所以说，foregroundDeletion 在实现上只是尽最大努力去保证显示级联删除的效果
* n.isObserved：如果虚拟节点没有在 Informer 的事件中观察到，那么需要重新入队处理，防止因为 ListWatch 重建导致丢失虚拟节点对应的对象的 Add 和 Delete 事件，从而使得虚拟节点一直存在于关系图中，相关 issue：https://github.com/kubernetes/kubernetes/issues/56121#issuecomment-353265160

```go
func (gc *GarbageCollector) attemptToDeleteWorker() bool {
	item, quit := gc.attemptToDelete.Get()
	gc.workerLock.RLock()
	defer gc.workerLock.RUnlock()
	if quit {
		return false
	}
	defer gc.attemptToDelete.Done(item)
	n, ok := item.(*node)
	if !ok {
		utilruntime.HandleError(fmt.Errorf("expect *node, got %#v", item))
		return true
	}
	err := gc.attemptToDeleteItem(n)
	if err != nil {
		if _, ok := err.(*restMappingError); ok {
			// There are at least two ways this can happen:
			// 1. The reference is to an object of a custom type that has not yet been
			//    recognized by gc.restMapper (this is a transient error).
			// 2. The reference is to an invalid group/version. We don't currently
			//    have a way to distinguish this from a valid type we will recognize
			//    after the next discovery sync.
			// For now, record the error and retry.
			klog.V(5).Infof("error syncing item %s: %v", n, err)
		} else {
			utilruntime.HandleError(fmt.Errorf("error syncing item %s: %v", n, err))
		}
		// retry if garbage collection of an object failed.
		gc.attemptToDelete.AddRateLimited(item)
	} else if !n.isObserved() {
		// requeue if item hasn't been observed via an informer event yet.
		// otherwise a virtual node for an item added AND removed during watch reestablishment can get stuck in the graph and never removed.
		// see https://issue.k8s.io/56121
		klog.V(5).Infof("item %s hasn't been observed via informer yet", n.identity)
		gc.attemptToDelete.AddRateLimited(item)
	}
	return true
}

func (gc *GarbageCollector) attemptToDeleteItem(item *node) error {
	klog.V(2).Infof("processing item %s", item.identity)
	// "being deleted" is an one-way trip to the final deletion. We'll just wait for the final deletion, and then process the object's dependents.
	if item.isBeingDeleted() && !item.isDeletingDependents() {
		// Orphan
		//
		klog.V(5).Infof("processing item %s returned at once, because its DeletionTimestamp is non-nil", item.identity)
		return nil
	}
	// TODO: It's only necessary to talk to the API server if this is a
	// "virtual" node. The local graph could lag behind the real status, but in
	// practice, the difference is small.
	latest, err := gc.getObject(item.identity)
	switch {
	case errors.IsNotFound(err):
        // 创建的虚拟节点实际上不存在了，删除

		// the GraphBuilder can add "virtual" node for an owner that doesn't
		// exist yet, so we need to enqueue a virtual Delete event to remove
		// the virtual node from GraphBuilder.uidToNode.
		klog.V(5).Infof("item %v not found, generating a virtual delete event", item.identity)
		gc.dependencyGraphBuilder.enqueueVirtualDeleteEvent(item.identity)
		// since we're manually inserting a delete event to remove this node,
		// we don't need to keep tracking it as a virtual node and requeueing in attemptToDelete
		item.markObserved()
		return nil
	case err != nil:
		return err
	}

	if latest.GetUID() != item.identity.UID {
        // 创建的虚拟节点与实际节点不匹配，删除

		klog.V(5).Infof("UID doesn't match, item %v not found, generating a virtual delete event", item.identity)
		gc.dependencyGraphBuilder.enqueueVirtualDeleteEvent(item.identity)
		// since we're manually inserting a delete event to remove this node,
		// we don't need to keep tracking it as a virtual node and requeueing in attemptToDelete
		item.markObserved()
		return nil
	}

	// TODO: attemptToOrphanWorker() routine is similar. Consider merging
	// attemptToOrphanWorker() into attemptToDeleteItem() as well.
	if item.isDeletingDependents() {
        // 删除 block 的子节点
		return gc.processDeletingDependentsItem(item)
	}

	// compute if we should delete the item
	ownerReferences := latest.GetOwnerReferences()
	if len(ownerReferences) == 0 {
		klog.V(2).Infof("object %s's doesn't have an owner, continue on next item", item.identity)
		return nil
	}

	solid, dangling, waitingForDependentsDeletion, err := gc.classifyReferences(item, ownerReferences)
	if err != nil {
		return err
	}
	klog.V(5).Infof("classify references of %s.\nsolid: %#v\ndangling: %#v\nwaitingForDependentsDeletion: %#v\n", item.identity, solid, dangling, waitingForDependentsDeletion)

	switch {
	case len(solid) != 0:
		klog.V(2).Infof("object %#v has at least one existing owner: %#v, will not garbage collect", item.identity, solid)
		if len(dangling) == 0 && len(waitingForDependentsDeletion) == 0 {
			return nil
		}
		klog.V(2).Infof("remove dangling references %#v and waiting references %#v for object %s", dangling, waitingForDependentsDeletion, item.identity)
        // 解除与 dangling、waitingForDependentsDeletion 的关联关系，但不能删除，因为还有 solid

		// waitingForDependentsDeletion needs to be deleted from the
		// ownerReferences, otherwise the referenced objects will be stuck with
		// the FinalizerDeletingDependents and never get deleted.
		ownerUIDs := append(ownerRefsToUIDs(dangling), ownerRefsToUIDs(waitingForDependentsDeletion)...)
		patch := deleteOwnerRefStrategicMergePatch(item.identity.UID, ownerUIDs...)
		_, err = gc.patch(item, patch, func(n *node) ([]byte, error) {
			return gc.deleteOwnerRefJSONMergePatch(n, ownerUIDs...)
		})
		return err
	case len(waitingForDependentsDeletion) != 0 && item.dependentsLength() != 0:
        // 父节点全部处于 foregroundDeletion 状态，并且有子节点也处于 foregroundDeletion 状态，那么解除与父节点的关联，避免发生“死锁”
		deps := item.getDependents()
		for _, dep := range deps {
			if dep.isDeletingDependents() {
				// this circle detection has false positives, we need to
				// apply a more rigorous detection if this turns out to be a
				// problem.
				// there are multiple workers run attemptToDeleteItem in
				// parallel, the circle detection can fail in a race condition.
				klog.V(2).Infof("processing object %s, some of its owners and its dependent [%s] have FinalizerDeletingDependents, to prevent potential cycle, its ownerReferences are going to be modified to be non-blocking, then the object is going to be deleted with Foreground", item.identity, dep.identity)
				patch, err := item.unblockOwnerReferencesStrategicMergePatch()
				if err != nil {
					return err
				}
				if _, err := gc.patch(item, patch, gc.unblockOwnerReferencesJSONMergePatch); err != nil {
					return err
				}
				break
			}
		}
		klog.V(2).Infof("at least one owner of object %s has FinalizerDeletingDependents, and the object itself has dependents, so it is going to be deleted in Foreground", item.identity)
		// the deletion event will be observed by the graphBuilder, so the item
		// will be processed again in processDeletingDependentsItem. If it
		// doesn't have dependents, the function will remove the
		// FinalizerDeletingDependents from the item, resulting in the final
		// deletion of the item.
		policy := metav1.DeletePropagationForeground
		return gc.deleteObject(item.identity, &policy)
	default:
        // 节点没有父节点了，那么就可以删除了，具体采用哪种删除方式，根据 finalizers 的情况来

		// item doesn't have any solid owner, so it needs to be garbage
		// collected. Also, none of item's owners is waiting for the deletion of
		// the dependents, so set propagationPolicy based on existing finalizers.
		var policy metav1.DeletionPropagation
		switch {
		case hasOrphanFinalizer(latest):
			// if an existing orphan finalizer is already on the object, honor it.
			policy = metav1.DeletePropagationOrphan
		case hasDeleteDependentsFinalizer(latest):
			// if an existing foreground finalizer is already on the object, honor it.
			policy = metav1.DeletePropagationForeground
		default:
			// otherwise, default to background.
			policy = metav1.DeletePropagationBackground
		}
		klog.V(2).Infof("delete object %s with propagation policy %s", item.identity, policy)
		return gc.deleteObject(item.identity, &policy)
	}
}
```
