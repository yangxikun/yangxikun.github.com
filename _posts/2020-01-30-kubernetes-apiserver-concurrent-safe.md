---
layout: post
title: "Kubernetes ApiServer 并发安全机制"
description: "Kubernetes ApiServer 并发安全机制"
category: Kubernetes
tags: []
---
{% include JB/setup %}

#### 问题定义

笔者在这篇文章中想要探讨的问题是：当向 Kubernetes ApiServer 并发的发起多个请求，对同一个 API 资源对象进行更新时，Kubernetes ApiServer 是如何确保更新不丢失，以及如何做冲突检测的？

<!--more-->

#### 将 Kubectl 作为切入点

Kubectl 的命令有很多，其中笔者经常用于修改 API 资源对象的命令是 apply 和 edit。接下来以修改 Deployment 资源对象为例子，通过观察这2个命令发出的请求，发现它们都是使用 HTTP PATCH：

```text
PATCH /apis/extensions/v1beta1/namespaces/default/deployments/nginx HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: kubectl/v0.0.0 (linux/amd64) kubernetes/$Format
Content-Length: 246
Accept: application/json
Content-Type: application/strategic-merge-patch+json
Uber-Trace-Id: 5ca3fde0b9f9aaf1:0c0358897c8e0ef8:0f38135280523f87:1
Accept-Encoding: gzip

{"spec":{"template":{"spec":{"$setElementOrder/containers":[{"name":"nginx"}],"containers":[{"$setElementOrder/env":[{"name":"DEMO_GREETING"}],"env":[{"name":"DEMO_GREETING","value":"Hello from the environment#kubectl edit"}],"name":"nginx"}]}}}}
```

#### 跟踪 ApiServer 对 PATCH 请求的处理

ApiServer 处理 PATCH 请求的调用栈如下：

```text
restfulPatchResource
    // 参数
    r rest.Patcher // registry.Store
    scope handlers.RequestScope
    admit admission.Interface
    supportedTypes []string // 支持的资源对象 PATCH 类型
    // 返回值
    restful.RouteFunction // go-restful 的路由处理回调

handlers.PatchResource
    // 创建 handlers.patcher 对象
    // 参数
    r rest.Patcher // registry.Store
    scope *RequestScope
    admit admission.Interface
    patchTypes []string // 支持的资源对象 PATCH 类型
    // 返回值
    http.HandlerFunc

k8s.io/apiserver/pkg/endpoints/handlers.(*patcher).patchResource
    // 参数
    ctx context.Context // http.Request.Context
    scope *RequestScope
    // 返回值
    runtime.Object // 更新后的资源对象
    bool // 是否创建了资源对象（当更新一个不存在的资源对象时，可以直接创建）
    error

k8s.io/apiserver/pkg/registry/generic/registry.(*Store).Update
    // 参数
    ctx context.Context
    name string // 资源对象的名称，例如：/deployments/default/nginx
    objInfo rest.UpdatedObjectInfo // 更新资源对象所需要的信息
    createValidation rest.ValidateObjectFunc
    updateValidation rest.ValidateObjectUpdateFunc
    forceAllowCreate bool
    options *metav1.UpdateOptions
    // 返回值
    runtime.Object // 更新后的资源对象
    bool // 是否创建了资源对象（当更新一个不存在的资源对象时，可以直接创建）
    error

k8s.io/apiserver/pkg/registry/generic/registry.(*DryRunnableStorage).GuaranteedUpdate
    // 实现对资源对象进行模拟更新的逻辑
    // 如果 dryRun = true，则执行完后不会再调用持久化操作
    // 参数
    ctx context.Context
    key string // 资源对象的名称，例如：/deployments/default/nginx
    ptrToType runtime.Object, // 资源对象的指针类型，由 registry.Store.NewFunc 创建
    ignoreNotFound bool, // ？？？
    preconditions *storage.Preconditions, // 执行更新前的检查，检查 UID 和 ResourceVersion
    tryUpdate storage.UpdateFunc, // registry.Store.Update 方法中的闭包函数，执行资源对象的更新操作
    dryRun bool, // 是否空跑（即执行输入检查，在内存中尝试更新，但不持久化）
    suggestion ...runtime.Object
    // 返回值
    error

k8s.io/apiserver/pkg/storage/cacher.(*Cacher).GuaranteedUpdate
    // 从内存缓存 cacher.Cacher.watchCache 中查询 key 是否存在，若存在，作为 suggestion 传递下去
    // 参数
    ctx context.Context
    key string // 资源对象的名称，例如：/deployments/default/nginx
    ptrToType runtime.Object // 资源对象的指针类型，由 registry.Store.NewFunc 创建
    ignoreNotFound bool
    preconditions *storage.Preconditions // 执行更新前的检查，检查 UID 和 ResourceVersion
    tryUpdate storage.UpdateFunc // registry.Store.Update 方法中的闭包函数，执行资源对象的更新操作
    _ ...runtime.Object
    // 返回值
    error

k8s.io/apiserver/pkg/storage/etcd3.(*store).GuaranteedUpdate
    // 调用 tryUpdate 更新资源对象，并通过 Etcd 的事务更新操作，持久化到 Etcd
    // 参数
    ctx context.Context
    key string // 资源对象的名称，例如：/deployments/default/nginx
    out runtime.Object // 资源对象的指针类型，由 registry.Store.NewFunc 创建
    ignoreNotFound bool
    preconditions *storage.Preconditions // 执行更新前的检查，检查 UID 和 ResourceVersion
    tryUpdate storage.UpdateFunc // registry.Store.Update 方法中的闭包函数，执行资源对象的更新操作
    suggestion ...runtime.Object // 如果在内存缓存中能找到，则可以避免直接访问 Etcd
    // 返回值
    error
```

在整个调用栈中，有几个关键点需要展开分析下：

##### storage.Preconditions

执行 tryUpdate 前的检查。

```go
// Preconditions must be fulfilled before an operation (update, delete, etc.) is carried out.
type Preconditions struct {
    // Specifies the target UID.
    // +optional
    UID *types.UID `json:"uid,omitempty"`
    // Specifies the target ResourceVersion
    // +optional
    ResourceVersion *string `json:"resourceVersion,omitempty"`
}
```

storage.Preconditions 只有一个方法：func (p *Preconditions) Check(key string, obj runtime.Object) error，检查 obj 的 UID 和 ResourceVersion 是否与 Preconditions 一致。

在 PATCH 请求处理中，Preconditions 从 rest.UpdatedObjectInfo.Preconditions() 获取。

##### rest.UpdatedObjectInfo

tryUpdate 闭包中会调用 rest.UpdatedObjectInfo.UpdatedObject() 对资源对象执行更新操作。

```go
// UpdatedObjectInfo provides information about an updated object to an Updater.
// It requires access to the old object in order to return the newly updated object.
type UpdatedObjectInfo interface {
    // Returns preconditions built from the updated object, if applicable.
    // May return nil, or a preconditions object containing nil fields,
    // if no preconditions can be determined from the updated object.
    Preconditions() *metav1.Preconditions

    // UpdatedObject returns the updated object, given a context and old object.
    // The only time an empty oldObj should be passed in is if a "create on update" is occurring (there is no oldObj).
    UpdatedObject(ctx context.Context, oldObj runtime.Object) (newObj runtime.Object, err error)
}
```

在 PATCH 请求处理中，rest.UpdatedObjectInfo 的实现为 rest.defaultUpdatedObjectInfo：

```go
// defaultUpdatedObjectInfo implements UpdatedObjectInfo
type defaultUpdatedObjectInfo struct {
    // obj is the updated object
    obj runtime.Object

    // transformers is an optional list of transforming functions that modify or
    // replace obj using information from the context, old object, or other sources.
    transformers []TransformFunc
}

// DefaultUpdatedObjectInfo returns an UpdatedObjectInfo impl based on the specified object.
func DefaultUpdatedObjectInfo(obj runtime.Object, transformers ...TransformFunc) UpdatedObjectInfo {
    return &defaultUpdatedObjectInfo{obj, transformers}
}

// Preconditions satisfies the UpdatedObjectInfo interface.
// 注意这里只获取了 UID
func (i *defaultUpdatedObjectInfo) Preconditions() *metav1.Preconditions {
    // Attempt to get the UID out of the object
    accessor, err := meta.Accessor(i.obj)
    if err != nil {
        // If no UID can be read, no preconditions are possible
        return nil
    }

    // If empty, no preconditions needed
    uid := accessor.GetUID()
    if len(uid) == 0 {
        return nil
    }

    return &metav1.Preconditions{UID: &uid}
}

// UpdatedObject satisfies the UpdatedObjectInfo interface.
// It returns a copy of the held obj, passed through any configured transformers.
func (i *defaultUpdatedObjectInfo) UpdatedObject(ctx context.Context, oldObj runtime.Object) (runtime.Object, error) {
    var err error
    // Start with the configured object
    newObj := i.obj

    // If the original is non-nil (might be nil if the first transformer builds the object from the oldObj), make a copy,
    // so we don't return the original. BeforeUpdate can mutate the returned object, doing things like clearing ResourceVersion.
    // If we're re-called, we need to be able to return the pristine version.
    if newObj != nil {
        newObj = newObj.DeepCopyObject()
    }

    // Allow any configured transformers to update the new object
    for _, transformer := range i.transformers {
        newObj, err = transformer(ctx, newObj, oldObj) // if http.method == patch: patcher.applyPatch, patcher.applyAdmission
        if err != nil {
            return nil, err
        }
    }

    return newObj, nil
}
```

创建时 defaultUpdatedObjectInfo.obj 为 nil，则 defaultUpdatedObjectInfo.Preconditions() 返回 nil，即不会进行 Preconditions 检查。

```go
// patchResource divides PatchResource for easier unit testing
func (p *patcher) patchResource(ctx context.Context, scope *RequestScope) (runtime.Object, bool, error) {
    ......
    p.updatedObjectInfo = rest.DefaultUpdatedObjectInfo(nil, p.applyPatch, p.applyAdmission)
    ......
}
```

##### tryUpdate 闭包更新

```go
// Update performs an atomic update and set of the object. Returns the result of the update
// or an error. If the registry allows create-on-update, the create flow will be executed.
// A bool is returned along with the object and any errors, to indicate object creation.
func (e *Store) Update(ctx context.Context, name string, objInfo rest.UpdatedObjectInfo, createValidation rest.ValidateObjectFunc, updateValidation rest.ValidateObjectUpdateFunc, forceAllowCreate bool, options *metav1.UpdateOptions) (runtime.Object, bool, error) {
    ......
    err = e.Storage.GuaranteedUpdate(ctx, key, out, true, storagePreconditions, func(existing runtime.Object, res storage.ResponseMeta) (runtime.Object, *uint64, error) {
        // Given the existing object, get the new object
        obj, err := objInfo.UpdatedObject(ctx, existing) // defaultUpdatedObjectInfo.UpdatedObject
        if err != nil {
            return nil, nil, err
        }

        // If AllowUnconditionalUpdate() is true and the object specified by
        // the user does not have a resource version, then we populate it with
        // the latest version. Else, we check that the version specified by
        // the user matches the version of latest storage object.
        resourceVersion, err := e.Storage.Versioner().ObjectResourceVersion(obj)
        if err != nil {
            return nil, nil, err
        }
        doUnconditionalUpdate := resourceVersion == 0 && e.UpdateStrategy.AllowUnconditionalUpdate()

        version, err := e.Storage.Versioner().ObjectResourceVersion(existing)
        if err != nil {
            return nil, nil, err
        }
        // version == 0 说明 key 对应的资源对象不存在
        if version == 0 {
            if !e.UpdateStrategy.AllowCreateOnUpdate() && !forceAllowCreate {
                return nil, nil, kubeerr.NewNotFound(qualifiedResource, name)
            }
            creating = true
            creatingObj = obj
            if err := rest.BeforeCreate(e.CreateStrategy, ctx, obj); err != nil {
                return nil, nil, err
            }
            // at this point we have a fully formed object.  It is time to call the validators that the apiserver
            // handling chain wants to enforce.
            if createValidation != nil {
                if err := createValidation(obj.DeepCopyObject()); err != nil {
                    return nil, nil, err
                }
            }
            ttl, err := e.calculateTTL(obj, 0, false)
            if err != nil {
                return nil, nil, err
            }

            return obj, &ttl, nil
        }

        creating = false
        creatingObj = nil
        if doUnconditionalUpdate {
            // Update the object's resource version to match the latest
            // storage object's resource version.
            // 无条件更新
            err = e.Storage.Versioner().UpdateObject(obj, res.ResourceVersion)
            if err != nil {
                return nil, nil, err
            }
        } else {
            // Check if the object's resource version matches the latest
            // resource version.
            if resourceVersion == 0 {
                // TODO: The Invalid error should have a field for Resource.
                // After that field is added, we should fill the Resource and
                // leave the Kind field empty. See the discussion in #18526.
                qualifiedKind := schema.GroupKind{Group: qualifiedResource.Group, Kind: qualifiedResource.Resource}
                fieldErrList := field.ErrorList{field.Invalid(field.NewPath("metadata").Child("resourceVersion"), resourceVersion, "must be specified for an update")}
                return nil, nil, kubeerr.NewInvalid(qualifiedKind, name, fieldErrList)
            }
            // 这里是关键，更新前后的资源对象版本不一致，说明出现了并发更新冲突
            // PATCH 请求中 resourceVersion 始终是等于 version 的！！！
            // PUT 请求中才可能会出现 resourceVersion != version
            if resourceVersion != version {
                return nil, nil, kubeerr.NewConflict(qualifiedResource, name, fmt.Errorf(OptimisticLockErrorMsg))
            }
        }
        if err := rest.BeforeUpdate(e.UpdateStrategy, ctx, obj, existing); err != nil {
            return nil, nil, err
        }
        // at this point we have a fully formed object.  It is time to call the validators that the apiserver
        // handling chain wants to enforce.
        if updateValidation != nil {
            if err := updateValidation(obj.DeepCopyObject(), existing.DeepCopyObject()); err != nil {
                return nil, nil, err
            }
        }
        // Check the default delete-during-update conditions, and store-specific conditions if provided
        if ShouldDeleteDuringUpdate(ctx, key, obj, existing) &&
            (e.ShouldDeleteDuringUpdate == nil || e.ShouldDeleteDuringUpdate(ctx, key, obj, existing)) {
            deleteObj = obj
            return nil, nil, errEmptiedFinalizers
        }
        ttl, err := e.calculateTTL(obj, res.TTL, true)
        if err != nil {
            return nil, nil, err
        }
        if int64(ttl) != res.TTL {
            return obj, &ttl, nil
        }
        return obj, nil, nil
    }, dryrun.IsDryRun(options.DryRun))
    ......
}
```

##### 持久化到 Etcd

```go
// GuaranteedUpdate implements storage.Interface.GuaranteedUpdate.
func (s *store) GuaranteedUpdate(
    ctx context.Context, key string, out runtime.Object, ignoreNotFound bool,
    preconditions *storage.Preconditions, tryUpdate storage.UpdateFunc, suggestion ...runtime.Object) error {
    trace := utiltrace.New(fmt.Sprintf("GuaranteedUpdate etcd3: %s", getTypeName(out)))
    defer trace.LogIfLong(500 * time.Millisecond)

    v, err := conversion.EnforcePtr(out) // out 必须是非 nil 值的指针类型
    if err != nil {
        panic("unable to convert output object to pointer")
    }
    key = path.Join(s.pathPrefix, key)

    // 从 Etcd 存储中获取 key 对象的状态
    getCurrentState := func() (*objState, error) {
        startTime := time.Now()
        getResp, err := s.client.KV.Get(ctx, key, s.getOps...)
        metrics.RecordEtcdRequestLatency("get", getTypeName(out), startTime)
        if err != nil {
            return nil, err
        }
        return s.getState(getResp, key, v, ignoreNotFound)
    }

    var origState *objState
    var mustCheckData bool // = true 说明 origState 有可能不是最新的
    // 如果上层调用提供了 key 对象的值（从 k8s.io/apiserver/pkg/storage/cacher.Cacher.watchCache.GetByKey(key) 获取）
    // 则不需要访问 Etcd
    if len(suggestion) == 1 && suggestion[0] != nil {
        span.LogFields(log.Object("suggestion[0]", suggestion[0]))
        origState, err = s.getStateFromObject(suggestion[0])
        span.LogFields(log.Object("origState", origState))
        if err != nil {
            return err
        }
        mustCheckData = true
    } else {
        origState, err = getCurrentState()
        if err != nil {
            return err
        }
    }
    trace.Step("initial value restored")

    transformContext := authenticatedDataString(key)
    for {
        // 检查 origState.obj 的 UID、 ResourceVersion 是否与 preconditions 一致
        // PATCH 和 PUT 请求都不做检查！！！
        if err := preconditions.Check(key, origState.obj); err != nil {
            // If our data is already up to date, return the error
            // 如果 origState 已经是最新的状态了，则返回错误
            if !mustCheckData {
                return err
            }

            // 可能 origState 不是最新的状态，从 Etcd 获取最新的状态
            // It's possible we were working with stale data
            // Actually fetch
            origState, err = getCurrentState()
            if err != nil {
                return err
            }
            mustCheckData = false
            // Retry
            continue
        }

        // 在 origState.obj 的基础上进行修改
        ret, ttl, err := s.updateState(origState, tryUpdate)
        if err != nil {
            // origState 可能不是最新的状态，会从 Etcd 中获取最新的状态，再尝试更新一次
            // If our data is already up to date, return the error
            if !mustCheckData {
                return err
            }

            // It's possible we were working with stale data
            // Actually fetch
            origState, err = getCurrentState()
            if err != nil {
                return err
            }
            mustCheckData = false
            // Retry
            continue
        }

        data, err := runtime.Encode(s.codec, ret)
        if err != nil {
            return err
        }
        // 目前 origState.stale 始终为 false
        // 判断修改后的资源对象是否有变化
        if !origState.stale && bytes.Equal(data, origState.data) {
            // if we skipped the original Get in this loop, we must refresh from
            // etcd in order to be sure the data in the store is equivalent to
            // our desired serialization
            // mustCheckData = true 说明 origState 有可能不是最新的
            // 必须再确认一遍
            if mustCheckData {
                origState, err = getCurrentState()
                if err != nil {
                    return err
                }
                mustCheckData = false
                if !bytes.Equal(data, origState.data) {
                    // original data changed, restart loop
                    continue
                }
            }
            // recheck that the data from etcd is not stale before short-circuiting a write
            if !origState.stale {
                return decode(s.codec, s.versioner, origState.data, out, origState.rev)
            }
        }

        // 将对象序列化为二进制数据存储到 Etcd
        newData, err := s.transformer.TransformToStorage(data, transformContext)
        if err != nil {
            return storage.NewInternalError(err.Error())
        }

        // 设置 key 的过期时间
        opts, err := s.ttlOpts(ctx, int64(ttl))
        if err != nil {
            return err
        }
        trace.Step("Transaction prepared")

        span.LogFields(log.Uint64("ttl", ttl), log.Int64("origState.rev", origState.rev))

        // Etcd 事务
        // 注意这里会比较 Etcd 中的资源对象版本跟 origState.rev 是否一致
        // 如果一致，则更新
        // 否则，更新失败并获取当前最新的资源对象
        startTime := time.Now()
        txnResp, err := s.client.KV.Txn(ctx).If(
            clientv3.Compare(clientv3.ModRevision(key), "=", origState.rev),
        ).Then(
            clientv3.OpPut(key, string(newData), opts...),
        ).Else(
            clientv3.OpGet(key),
        ).Commit()
        metrics.RecordEtcdRequestLatency("update", getTypeName(out), startTime)
        if err != nil {
            return err
        }
        trace.Step("Transaction committed")
        if !txnResp.Succeeded {
            // 事务执行失败
            getResp := (*clientv3.GetResponse)(txnResp.Responses[0].GetResponseRange())
            klog.V(4).Infof("GuaranteedUpdate of %s failed because of a conflict, going to retry", key)
            // 获取最新的状态，并重试
            origState, err = s.getState(getResp, key, v, ignoreNotFound)
            if err != nil {
                return err
            }
            trace.Step("Retry value restored")
            mustCheckData = false
            continue
        }
        // 事务执行成功
        putResp := txnResp.Responses[0].GetResponsePut()

        return decode(s.codec, s.versioner, data, out, putResp.Header.Revision)
    }
}
```

#### 分析总结

##### 并发更新是如何确保更新不丢失的？

在将更新后的对象持久化到 Etcd 中时，通过事务保证的，事务的伪代码逻辑如下：

```text
if inEtcd(key).rev == inMemory(oldObj).rev:
    EtcdSet(key) = newObj
    transaction = success
else:
    EtcdGet(key)
    transaction = fail
```

##### 并发更新是如何做冲突检测的？

在上面的分析中，可以看到有两处冲突检测的判断：

1. Preconditions
1. tryUpdate 中的 resourceVersion != version

对于 kubectl apply 和 edit （发送的都是 PATCH 请求），创建的 Preconditions 是零值，所以不会通过 Preconditions 进行冲突检测，而在 tryUpdate 中调用 objInfo.UpdatedObject(ctx, existing) 得到的 newObj.rv 始终是等于 existing.rv 的，所以也不会进行冲突检测。

那什么时候会进行冲突检测？其实 kubectl 还有个 replace 的命令，通过抓包发现 replace 命令发送的是 PUT 请求，并且请求中会带有 resourceVersion：

```text
PUT /apis/extensions/v1beta1/namespaces/default/deployments/nginx HTTP/1.1
Host: localhost:8080
User-Agent: kubectl/v0.0.0 (linux/amd64) kubernetes/$Format
Content-Length: 866
Accept: application/json
Content-Type: application/json
Uber-Trace-Id: 6e685772cc06fc16:2514dc54a474fe88:4f488c05a7cef9c8:1
Accept-Encoding: gzip

{"apiVersion":"extensions/v1beta1","kind":"Deployment","metadata":{"labels":{"app":"nginx"},"name":"nginx","namespace":"default","resourceVersion":"744603"},"spec":{"progressDeadlineSeconds":600,"replicas":1,"revisionHistoryLimit":10,"selector":{"matchLabels":{"app":"nginx"}},"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"nginx"}},"spec":{"containers":[{"env":[{"name":"DEMO_GREETING","value":"Hello from the environment#kubectl replace"}],"image":"nginx","imagePullPolicy":"IfNotPresent","name":"nginx","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File"}],"dnsPolicy":"ClusterFirst","restartPolicy":"Always","schedulerName":"default-scheduler","securityContext":{},"terminationGracePeriodSeconds":30}}}}
```

PUT 请求在 ApiServer 中的处理与 PATCH 请求类似，都是调用 k8s.io/apiserver/pkg/registry/generic/registry.(*Store).Update，创建的 rest.UpdatedObjectInfo 为 rest.DefaultUpdatedObjectInfo(obj, transformers...)，注意这里传了 obj 参数值（通过 decode 请求 body 得到），而不是 nil。

在 PUT 请求处理中，创建的 Preconditions 也是零值，不会通过 Preconditions 做检查。但是在 tryUpdate 中，`resourceVersion, err := e.Storage.Versioner().ObjectResourceVersion(obj)` 得到的 resourceVersion 是请求 body 中的值，而不像 PATCH 请求一样，来自 existing（看下 defaultUpdatedObjectInfo.UpdatedObject 方法就明白了）。

所以在 PUT 请求处理中，会通过 tryUpdate 中的 resourceVersion != version，检测是否发生了并发写冲突。
