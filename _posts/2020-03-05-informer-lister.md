---
layout: post
title: "Informer 与 Lister 详解"
description: "Informer 与 Lister 详解"
category: Kubernetes
tags: []
---
{% include JB/setup %}

#### 简介

* Informer：
    * 同步本地缓存，把 API 资源对象缓存一份到本地
    * 根据发生的事件类型，触发事先注册好的控制器回调
* Lister：从本地缓存中获取 API 资源对象

<!--more-->

#### 创建

##### 工厂的创建

每一种 API 资源对象都会有对应的 Informer，由工厂 k8s.io/client-go/informers.sharedInformerFactory 负责创建，工厂的创建如下：

* client：与 kube-apiserver 交互的客户端实现
* defaultResync：[minResyncPeriod, 2*minResyncPeriod) 范围的一个随机值，用于设置 reflectors 重新同步的周期，默认 12h
* v1.NamespaceAll：空字符串
* sharedInformerFactory：
    * informers：存储创建好的 API 资源对象对应的 Informer
    * startedInformers：已启动的 Informer
    * customResync：为特定类型的 API 资源对象指定同步周期，目前未用到
    * tweakListOptions：设置请求参数

```go
// NewSharedInformerFactory constructs a new instance of sharedInformerFactory for all namespaces.
func NewSharedInformerFactory(client kubernetes.Interface, defaultResync time.Duration) SharedInformerFactory {
    return NewSharedInformerFactoryWithOptions(client, defaultResync)
}

// NewSharedInformerFactoryWithOptions constructs a new instance of a SharedInformerFactory with additional options.
func NewSharedInformerFactoryWithOptions(client kubernetes.Interface, defaultResync time.Duration, options ...SharedInformerOption) SharedInformerFactory {
    factory := &sharedInformerFactory{
        client:           client,
        namespace:        v1.NamespaceAll,
        defaultResync:    defaultResync,
        informers:        make(map[reflect.Type]cache.SharedIndexInformer),
        startedInformers: make(map[reflect.Type]bool),
        customResync:     make(map[reflect.Type]time.Duration),
    }

    // Apply all options
    for _, opt := range options {
        factory = opt(factory)
    }

    return factory
}
```

##### API 资源对象对应的 Informer 的创建

以 sharedInformerFactory.Apps().V1().Deployments() 为例，创建的是 k8s.io/client-go/informers/apps/v1.deploymentInformer，调用链路如下：

```go
// k8s.io/client-go/informers
func (f *sharedInformerFactory) Apps() apps.Interface {
    return apps.New(f, f.namespace, f.tweakListOptions)
}

// k8s.io/client-go/informers/apps
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
    return &group{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

func (g *group) V1() v1.Interface {
    return v1.New(g.factory, g.namespace, g.tweakListOptions)
}

// k8s.io/client-go/informers/apps/v1
func New(f internalinterfaces.SharedInformerFactory, namespace string, tweakListOptions internalinterfaces.TweakListOptionsFunc) Interface {
    return &version{factory: f, namespace: namespace, tweakListOptions: tweakListOptions}
}

func (v *version) Deployments() DeploymentInformer {
    return &deploymentInformer{factory: v.factory, namespace: v.namespace, tweakListOptions: v.tweakListOptions}
}
```

##### k8s.io/client-go/informers/apps/v1.deploymentInformer 解析

* deploymentInformer 实现了 DeploymentInformer
    * factory：工厂，即 k8s.io/client-go/informers.sharedInformerFactory
    * tweakListOptions：sharedInformerFactory.tweakListOptions
    * namespace：命名空间，与 sharedInformerFactory.namespace 保持一致

```go
// DeploymentInformer provides access to a shared informer and lister for
// Deployments.
type DeploymentInformer interface {
    Informer() cache.SharedIndexInformer
    Lister() v1.DeploymentLister
}

type deploymentInformer struct {
    factory          internalinterfaces.SharedInformerFactory
    tweakListOptions internalinterfaces.TweakListOptionsFunc
    namespace        string
}
```

Informer 的创建：

* Informer：调用 sharedInformerFactory.InformerFor
    * &appsv1.Deployment{}：声明使用的 API 资源对象类型
    * deploymentInformer.defaultInformer：创建方法
    * 在 sharedInformerFactory.informers 中判断是否已存在，如果存在则返回
    * 如果不存在，调用 deploymentInformer.defaultInformer(sharedInformerFactory.client, resyncPeriod) 创建并存储到 sharedInformerFactory.informers
        * resyncPeriod：如果 sharedInformerFactory.customResync\[reflect.TypeOf(&appsv1.Deployment{})\] 存在，则为该值，否则为 sharedInformerFactory.defaultResync
* defaultInformer：
    * cache.Indexers：underlying type 为 map\[string\]IndexFunc
        * cache.NamespaceIndex：字符串 namespace 字面量
        * cache.MetaNamespaceIndexFunc：返回 API 资源对象的命名空间
* cache.NewSharedIndexInformer：创建 k8s.ioo/client-go/tools/cache.sharedIndexInformer
    * cache.ListWatch：
        * ListFunc：获取对应的 API 资源对象列表
        * WatchFunc：监听对应的 API 资源对象变更

```go
func (f *deploymentInformer) Informer() cache.SharedIndexInformer {
    return f.factory.InformerFor(&appsv1.Deployment{}, f.defaultInformer)
}

func (f *deploymentInformer) defaultInformer(client kubernetes.Interface, resyncPeriod time.Duration) cache.SharedIndexInformer {
    return NewFilteredDeploymentInformer(client, f.namespace, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
}

// NewFilteredDeploymentInformer constructs a new informer for Deployment type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredDeploymentInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
    return cache.NewSharedIndexInformer(
        &cache.ListWatch{
            ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
                if tweakListOptions != nil {
                    tweakListOptions(&options)
                }
                return client.AppsV1().Deployments(namespace).List(options)
            },
            WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
                if tweakListOptions != nil {
                    tweakListOptions(&options)
                }
                return client.AppsV1().Deployments(namespace).Watch(options)
            },
        },
        &appsv1.Deployment{},
        resyncPeriod,
        indexers,
    )
}
```

Lister 的创建：

* k8s.io/client-go/listers/apps/v1.NewDeploymentLister：返回 deploymentLister
    * deploymentLister.indexer：赋值为 sharedIndexInformer.indexer，实现类型为 k8s.io/client-go/tools/cache.cache

```go
func (f *deploymentInformer) Lister() v1.DeploymentLister {
    return v1.NewDeploymentLister(f.Informer().GetIndexer())
}

// deploymentLister implements the DeploymentLister interface.
type deploymentLister struct {
    indexer cache.Indexer
}

// NewDeploymentLister returns a new DeploymentLister.
func NewDeploymentLister(indexer cache.Indexer) DeploymentLister {
    return &deploymentLister{indexer: indexer}
}
```

#### k8s.ioo/client-go/tools/cache.sharedIndexInformer 解析

* indexer：k8s.io/client-go/tools/cache.cache，通过应用 DeltaFIFO.Pop 出来的事件更新缓存，会在 sharedIndexInformer.HandleDeltas 中更新（Update/Add/Delete），用于 Lister 获取对象，作为持久化数据的一份缓存
    * cache.keyFunc：DeletionHandlingMetaNamespaceKeyFunc，检查 API 资源对象是否为 DeletedFinalStateUnknown
        * 如果是，返回 DeletedFinalStateUnknown.Key
        * 否则返回 MetaNamespaceKeyFunc(obj)，格式为 \<namespace\>/\<name\>，如果 \<namespace\> 为空，则返回 \<name\>
    * cache.cacheStorage：k8s.io/client-go/tools/cache.threadSafeMap
        * threadSafeMap.indexers：k8s.io/client-go/tools/cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}
        * threadSafeMap.indices：k8s.io/client-go/tools/cache.Indices，其 underlying type 为 map\[string\]Index
            * Index：map\[string\]sets.string，namespace => {name1, name2}
* controller：k8s.io/client-go/tools/cache.controller 负责与 kube-apiserver 交互
* processor：k8s.io/client-go/tools/cache.sharedProcessor 负责事件回调的注册和触发
* cacheMutationDetector：默认为 cache.dummyMutationDetector
    * k8s.io/client-go/tools/cache.dummyMutationDetector：空实现
    * k8s.io/client-go/tools/cache.defaultCacheMutationDetector：周期性检查缓存中的 API 资源对象是否发生了修改，如果发生修改，会触发 defaultCacheMutationDetector.failureFunc 或 panic
* listerWatcher：k8s.io/client-go/tools/cache.ListWatch
* objectType：API 资源类型，例如 &appsv1.Deployment{}
* resyncCheckPeriod：等于 defaultEventHandlerResyncPeriod
* defaultEventHandlerResyncPeriod：sharedInformerFactory.defaultResync
* clock：k8s.io/apimachinery/pkg/util/clock.RealClock
* started：是否启动了
* stopped：是否停止了
* startedLock：用于访问 started 和 stopped 时的互斥锁
* blockDeltas：停止事件分发，确保启动后新增的事件处理回调能够安全的加入

```go
type sharedIndexInformer struct {
    indexer    Indexer
    controller Controller

    processor             *sharedProcessor
    cacheMutationDetector CacheMutationDetector

    // This block is tracked to handle late initialization of the controller
    listerWatcher ListerWatcher
    objectType    runtime.Object

    // resyncCheckPeriod is how often we want the reflector's resync timer to fire so it can call
    // shouldResync to check if any of our listeners need a resync.
    resyncCheckPeriod time.Duration
    // defaultEventHandlerResyncPeriod is the default resync period for any handlers added via
    // AddEventHandler (i.e. they don't specify one and just want to use the shared informer's default
    // value).
    defaultEventHandlerResyncPeriod time.Duration
    // clock allows for testability
    clock clock.Clock

    started, stopped bool
    startedLock      sync.Mutex

    // blockDeltas gives a way to stop all event distribution so that a late event handler
    // can safely join the shared informer.
    blockDeltas sync.Mutex
}
```

##### k8s.io/client-go/tools/cache.controller

* controller
    * config：配置相关
        * Queue：k8s.io/client-go/tools/cache.DeltaFIFO 
        * ListerWatcher：k8s.io/client-go/tools/cache.ListWatch
        * Process：sharedIndexInformer.HandleDeltas
        * ObjectType：sharedIndexInformer.objectType
        * FullResyncPeriod：sharedIndexInformer.resyncCheckPeriod
        * ShouldResync：sharedProcessor.shouldResync 遍历每个 listener，调用 listener.shouldResync(now) 判断是否需要重新同步
            * processorListener.shouldResync：p.resyncPeriod 必须大于 0，判断是否满足 now.After(p.nextResync) \|\| now.Equal(p.nextResync)
        * RetryOnError：当 Process 返回错误的时候，是否重新入队 obj
    * reflector：k8s.io/client-go/tools/cache.Reflector
        * name：调用者的信息 shortpath/filename:line
        * metrics：目前未使用
        * expectedType：API 资源类型
        * store：k8s.io/client-go/tools/cache.DeltaFIFO
        * listerWatcher：k8s.io/client-go/tools/cache.ListWatch
        * period：time.Second Reflector.ListAndWatch 执行周期
        * resyncPeriod：sharedIndexInformer.resyncCheckPeriod
        * ShouldResync：sharedProcessor.shouldResync 遍历每个 listener，调用 listener.shouldResync(now) 判断是否需要重新同步
        * clock：k8s.io/apimachinery/pkg/util/clock.RealClock
        * lastSyncResourceVersion：同步到的 API 资源对象最新版本号
        * lastSyncResourceVersionMutex：保护 lastSyncResourceVersion 的访问
        * WatchListPageSize：初始化时分页获取对象列表的大小
    * reflectorMutex：用于对 reflector 进行赋值时的互斥访问（目前从代码看感觉不需要）
    * clock：k8s.io/apimachinery/pkg/util/clock.RealClock

```go
// Controller is a generic controller framework.
type controller struct {
    config         Config
    reflector      *Reflector
    reflectorMutex sync.RWMutex
    clock          clock.Clock
}

// Config contains all the settings for a Controller.
type Config struct {
    // The queue for your objects - has to be a DeltaFIFO due to
    // assumptions in the implementation. Your Process() function
    // should accept the output of this Queue's Pop() method.
    Queue

    // Something that can list and watch your objects.
    ListerWatcher

    // Something that can process your objects.
    Process ProcessFunc

    // The type of your objects.
    ObjectType runtime.Object

    // Reprocess everything at least this often.
    // Note that if it takes longer for you to clear the queue than this
    // period, you will end up processing items in the order determined
    // by FIFO.Replace(). Currently, this is random. If this is a
    // problem, we can change that replacement policy to append new
    // things to the end of the queue instead of replacing the entire
    // queue.
    FullResyncPeriod time.Duration

    // ShouldResync, if specified, is invoked when the controller's reflector determines the next
    // periodic sync should occur. If this returns true, it means the reflector should proceed with
    // the resync.
    ShouldResync ShouldResyncFunc

    // If true, when Process() returns an error, re-enqueue the object.
    // TODO: add interface to let you inject a delay/backoff or drop
    //       the object completely if desired. Pass the object in
    //       question to this interface as a parameter.
    RetryOnError bool
}
```

##### k8s.io/client-go/tools/cache.DeltaFIFO

* DeltaFIFO
    * lock/cond：保护对 items 和 queue 的访问
    * items：
        * string：DeltaFIFO.KeyOf(obj)，obj 可能为
            * Deltas：返回 obj.Newest().Object
            * DeletedFinalStateUnknown：返回 obj.Key
            * 返回 DeltaFIFO.keyFunc(obj)
        * Deltas：[]Delta
            * Delta：
                * Type：Added/Updated/Deleted/Sync
                * Object：发生变更后的对象状态
    * queue：如果 DeltaFIFO.KeyOf(obj) 不在 items 中，则入队 DeltaFIFO.KeyOf(obj)
    * populated：当调用过 Replace/Delete/Add/Update 方法时为 true
    * initialPopulationCount：调用 Replace 时，如果 populated = false，那么设置为 len(queue) 
    * keyFunc：MetaNamespaceKeyFunc，格式为 \<namespace\>/\<name\>，如果 \<namespace\> 是空，则为 \<name\>
    * knownObjects：sharedIndexInformer.indexer，记录 DeltaFIFO.Pop 出来的对象，会在 sharedIndexInformer.HandleDeltas 中更新（Update/Add/Delete），用于 Lister 获取对象，作为持久化数据的一份缓存
    * closed：队列是否关闭
    * closedLock：保护对 closed 的访问

```go
type DeltaFIFO struct {
    // lock/cond protects access to 'items' and 'queue'.
    lock sync.RWMutex
    cond sync.Cond

    // We depend on the property that items in the set are in
    // the queue and vice versa, and that all Deltas in this
    // map have at least one Delta.
    items map[string]Deltas
    queue []string

    // populated is true if the first batch of items inserted by Replace() has been populated
    // or Delete/Add/Update was called first.
    populated bool
    // initialPopulationCount is the number of items inserted by the first call of Replace()
    initialPopulationCount int

    // keyFunc is used to make the key used for queued item
    // insertion and retrieval, and should be deterministic.
    keyFunc KeyFunc

    // knownObjects list keys that are "known", for the
    // purpose of figuring out which items have been deleted
    // when Replace() or Delete() is called.
    knownObjects KeyListerGetter

    // Indication the queue is closed.
    // Used to indicate a queue is closed so a control loop can exit when a queue is empty.
    // Currently, not used to gate any of CRED operations.
    closed     bool
    closedLock sync.Mutex
}
```

##### k8s.io/client-go/tools/cache.sharedProcessor

* sharedProcessor：
    * listenersStarted：已启动的 listeners
    * listenersLock：保护 listeners 的访问
    * listeners：控制器注册的回调合集
        * processorListener：
            * nextCh：用于 processorListener.pop() 与 processorListener.handler 之间进行消息通信
            * addCh：用于 DeltaFIFO.pop() 与 processorListener.pop() 之间进行消息通信
            * handler：控制器回调的封装
            * pendingNotifications：存储所有未分发的消息
            * requestedResyncPeriod：sharedIndexInformer.defaultEventHandlerResyncPeriod
            * resyncPeriod：determineResyncPeriod(sharedIndexInformer.defaultEventHandlerResyncPeriod, sharedIndexInformer.resyncCheckPeriod)
            * nextResync：now.Add(p.resyncPeriod)
            * resyncLock：保护 resyncPeriod 和 nextResync 的访问
    * syncingListeners：需要进行同步的 processorListener
    * clock：k8s.io/apimachinery/pkg/util/clock.RealClock
    * wg：等待所有 processorListener 结束

```go
type sharedProcessor struct {
    listenersStarted bool
    listenersLock    sync.RWMutex
    listeners        []*processorListener
    syncingListeners []*processorListener
    clock            clock.Clock
    wg               wait.Group
}
```

#### 启动

##### 工作机制流程图

![](/assets/img/kubernetes-informer-lister.png)

##### k8s.ioo/client-go/tools/cache.sharedIndexInformer.Run

分别启动 k8s.io/client-go/tools/cache.controller.Run 和 k8s.io/client-go/tools/cache.sharedProcessor.Run。

```go
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()

    fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

    cfg := &Config{
        Queue:            fifo,
        ListerWatcher:    s.listerWatcher,
        ObjectType:       s.objectType,
        FullResyncPeriod: s.resyncCheckPeriod,
        RetryOnError:     false,
        ShouldResync:     s.processor.shouldResync,

        Process: s.HandleDeltas,
    }

    func() {
        s.startedLock.Lock()
        defer s.startedLock.Unlock()

        s.controller = New(cfg)
        s.controller.(*controller).clock = s.clock
        s.started = true
    }()

    // Separate stop channel because Processor should be stopped strictly after controller
    processorStopCh := make(chan struct{})
    var wg wait.Group
    defer wg.Wait()              // Wait for Processor to stop
    defer close(processorStopCh) // Tell Processor to stop
    wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
    // k8s.io/client-go/tools/cache.sharedProcessor.Run
    wg.StartWithChannel(processorStopCh, s.processor.run)

    defer func() {
        s.startedLock.Lock()
        defer s.startedLock.Unlock()
        s.stopped = true // Don't want any new listeners
    }()
    // k8s.io/client-go/tools/cache.controller.Run
    s.controller.Run(stopCh)
}
```

##### k8s.io/client-go/tools/cache.controller.Run

* Reflector.Run：
    * Reflector.ListAndWatch：每隔 Reflector.period 执行一次
        * items：通过 list 接口获取对象列表
        * Reflector.syncWith(items, resourceVersion)
            * Reflector.store.Replace：即调用 DeltaFIFO.Replace，入队 items 的 Sync Delta，如果 DeltaFIFO.knownObjects 的 key 不在 items 中，则会入队这些 key 的 Deleted Delta
        * Reflector.setLastSyncResourceVersion(resourceVersion)
        * 周期性执行 Reflector.store.Resync()
            * keys = DeltaFIFO.knownObjects.ListKeys()
            * for key in keys: DeltaFIFO.syncKeyLocked(k)
        * for
            * w = Reflector.listerWatcher.Watch(options)：调用对应 API 资源类型的 watch 接口，例如：client.AppsV1().Deployments(namespace).Watch(options)
                * options：
                    * ResourceVersion：同步到的最新版本号
                    * TimeoutSeconds：[5min, 10min)
                    * AllowWatchBookmarks：为了减轻重新 watch 时 kube-apiserver 的压力，alpha 阶段，未启用
            * Reflector.watchHandler(w, &resourceVersion, resyncerrc, stopCh)：
                * for range w.ResultChan()
                    * 根据发生的事件类型调用 Reflector.store.Add/Update/Delete（DeltaFIFO）
                    * r.setLastSyncResourceVersion(newResourceVersion)
* controller.processLoop：
    * for
        * obj = c.config.Queue.Pop(PopProcessFunc(c.config.Process)) // DeltaFIFO.Pop
            * sharedIndexInformer.HandleDeltas
                * 如果 Delta.Type in (Sync, Added, Updated)，那么执行 sharedIndexInformer.cacheMutationDetector.AddObject(d.Object)
                    * 如果 Delta.Object 在 sharedIndexInformer.indexer 中，则执行 sharedIndexInformer.indexer.Update(d.Object) 和 sharedIndexInformer.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
                    * 如果 Delta.Object 不在 sharedIndexInformer.indexer 中，则执行 sharedIndexInformer.indexer.Add(d.Object) 和 sharedIndexInformer.processor.distribute(addNotification{newObj: d.Object}, isSync)
                * 如果 Delta.Type =  Deleted，那么执行 sharedIndexInformer.indexer.Delete(d.Object) 和 sharedIndexInformer.processor.distribute(deleteNotification{oldObj: d.Object}, false)
        * 如果 c.config.Process 返回错误，并且 c.config.RetryOnError = true，则执行 c.config.Queue.AddIfNotPresent(obj)

```go
func (c *controller) Run(stopCh <-chan struct{}) {
    defer utilruntime.HandleCrash()
    go func() {
        <-stopCh
        c.config.Queue.Close() // DeltaFIFO.Close()
    }()
    r := NewReflector(
        c.config.ListerWatcher,
        c.config.ObjectType,
        c.config.Queue,
        c.config.FullResyncPeriod,
    )
    r.ShouldResync = c.config.ShouldResync // sharedProcessor.shouldResync
    r.clock = c.clock

    c.reflectorMutex.Lock()
    c.reflector = r
    c.reflectorMutex.Unlock()

    var wg wait.Group
    defer wg.Wait()

    wg.StartWithChannel(stopCh, r.Run)

    wait.Until(c.processLoop, time.Second, stopCh)
}

func (r *Reflector) Run(stopCh <-chan struct{}) {
    klog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
    wait.Until(func() {
        if err := r.ListAndWatch(stopCh); err != nil {
            utilruntime.HandleError(err)
        }
    }, r.period, stopCh)
}

func (c *controller) processLoop() {
    for {
        obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process)) // sharedIndexInformer.HandleDeltas
        if err != nil {
            if err == FIFOClosedError {
                return
            }
            if c.config.RetryOnError {
                // This is the safe way to re-enqueue.
                c.config.Queue.AddIfNotPresent(obj)
            }
        }
    }
}
```

##### k8s.io/client-go/tools/cache.sharedProcessor.Run

* listener.pop：从 processorListener.addCh 中接收消息，如果已有消息等待分发，则存入 processorListener.pendingNotifications，否则进行分发，写入到 processorListener.nextCh 中
    * 为何需要 processorListener.pendingNotifications？我的理解应该是为了避免某个 processorListener 阻塞，导致影响到其他 processorListener 的消息分发
* listener.run：从 processorListener.nextCh 中接收消息
    * 如果是 updateNotification，执行 ResourceEventHandlerFuncs.OnUpdate(notification.oldObj, notification.newObj)
    * 如果是 addNotification，执行 ResourceEventHandlerFuncs.OnAdd(notification.newObj)
    * 如果是 deleteNotification，执行 ResourceEventHandlerFuncs.OnDelete(notification.oldObj)

```go
func (p *sharedProcessor) run(stopCh <-chan struct{}) {
    func() {
        p.listenersLock.RLock()
        defer p.listenersLock.RUnlock()
        for _, listener := range p.listeners {
            p.wg.Start(listener.run)
            p.wg.Start(listener.pop)
        }
        p.listenersStarted = true
    }()
    <-stopCh
    p.listenersLock.RLock()
    defer p.listenersLock.RUnlock()
    for _, listener := range p.listeners {
        close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
    }
    p.wg.Wait() // Wait for all .pop() and .run() to stop
}
```
