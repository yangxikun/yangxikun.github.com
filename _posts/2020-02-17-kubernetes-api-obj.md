---
layout: post
title: "Kubernetes API 资源对象的版本控制"
description: "Kubernetes API 资源对象的版本控制"
category: Kubernetes
tags: []
---
{% include JB/setup %}


一个 API 资源对象的 Schema 的唯一标识由 apiVersion 和 kind 组成，其中 apiVersion 又分为 Group 和 Version。例如：

```text
Group    Version    Kind
apps     v1         Deployment
```

<!--more-->

kubernetes 的 apiVersion 有很多，可以通过 kubectl api-versions 命令查看当前支持的 API 版本：

```text
# kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1
coordination.k8s.io/v1beta1
crd.projectcalico.org/v1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
networking.k8s.io/v1beta1
node.k8s.io/v1beta1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

通过 kubectl api-resources 还可以查看当前支持的 API 资源对象：

```text
# kubectl api-resources | grep deploy
deployments                       deploy       apps                           true         Deployment
deployments                       deploy       extensions                     true         Deployment
```

一个 kind 可能出现在不同的 apiVersion 中，例如 Deployment 就出现在了如下多个 apiVersion 中：
 
 * k8s.io/api/apps/v1.Deployment
 * k8s.io/api/apps/v1beta1.Deployment
 * k8s.io/api/apps/v1beta2.Deployment
 * k8s.io/api/extensions/v1beta1.Deployment
 
 对于不同的 apiVersion，在 kube-apiserver 中，其对应的 HTTP Handler 是什么样子的呢？
 
在 kube-apiserver 的启动过程中，服务路由的安装是通过如下调用实现的：

```go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) {

    ......

    restStorageProviders := []RESTStorageProvider{
        auditregistrationrest.RESTStorageProvider{},
        authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
        authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
        autoscalingrest.RESTStorageProvider{},
        batchrest.RESTStorageProvider{},
        certificatesrest.RESTStorageProvider{},
        coordinationrest.RESTStorageProvider{},
        extensionsrest.RESTStorageProvider{},
        networkingrest.RESTStorageProvider{},
        noderest.RESTStorageProvider{},
        policyrest.RESTStorageProvider{},
        rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
        schedulingrest.RESTStorageProvider{},
        settingsrest.RESTStorageProvider{},
        storagerest.RESTStorageProvider{},
        // keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
        // See https://github.com/kubernetes/kubernetes/issues/42392
        appsrest.RESTStorageProvider{},
        admissionregistrationrest.RESTStorageProvider{},
        eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
    }
    m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)

    ......

}
``` 

以 appsrest.RESTStorageProvider{} 为例，从它的 NewRESTStorage 方法可以看出，它管理了 apps 这个 Group 下支持的 Version：

```go
func (p RESTStorageProvider) NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, bool) {
    apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apps.GroupName, legacyscheme.Scheme, legacyscheme.ParameterCodec, legacyscheme.Codecs)
    // If you add a version here, be sure to add an entry in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go with specific priorities.
    // TODO refactor the plumbing to provide the information in the APIGroupInfo

    if apiResourceConfigSource.VersionEnabled(appsapiv1beta1.SchemeGroupVersion) {
        apiGroupInfo.VersionedResourcesStorageMap[appsapiv1beta1.SchemeGroupVersion.Version] = p.v1beta1Storage(apiResourceConfigSource, restOptionsGetter)
    }
    if apiResourceConfigSource.VersionEnabled(appsapiv1beta2.SchemeGroupVersion) {
        apiGroupInfo.VersionedResourcesStorageMap[appsapiv1beta2.SchemeGroupVersion.Version] = p.v1beta2Storage(apiResourceConfigSource, restOptionsGetter)
    }
    if apiResourceConfigSource.VersionEnabled(appsapiv1.SchemeGroupVersion) {
        apiGroupInfo.VersionedResourcesStorageMap[appsapiv1.SchemeGroupVersion.Version] = p.v1Storage(apiResourceConfigSource, restOptionsGetter)
    }

    return apiGroupInfo, true
}
```

而每个 Version 下面管理着支持的 Kind：

```go
func (p RESTStorageProvider) v1Storage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) map[string]rest.Storage {
    storage := map[string]rest.Storage{}

    // deployments
    deploymentStorage := deploymentstore.NewStorage(restOptionsGetter)
    storage["deployments"] = deploymentStorage.Deployment
    storage["deployments/status"] = deploymentStorage.Status
    storage["deployments/scale"] = deploymentStorage.Scale

    ......

    return storage
}
```

对比不同 Version，发现对于同一个 Kind，使用的都是相同的 Storage 实现，例如 Deployment 的 Storage 实现就是 pkg/registry/apps/deployment/storage.DeploymentStorage。为何不同 apiVersion 的资源对象可以使用相同的 Storage 实现呢？

通过跟踪请求 PATCH /apis/apps/v1/namespaces/default/deployments/nginx，发现在处理过程中对资源对象版本进行了转换：

```text
patcher.patchResource
    ...
    schemaReferenceObj, err := p.unsafeConvertor.ConvertToVersion(p.restPatcher.New(), p.kind.GroupVersion())
    // *v1.Deployment                                               *apps.Deployment        apps/v1
    ...

smpPatcher.applyPatchToCurrentObject
    ...
    versionedObjToUpdate, err := p.creater.New(p.kind) // runtime.Scheme
    // *v1.Deployment
    if err != nil {
        return nil, err
    }
    if err := strategicPatchObject(p.defaulter, currentVersionedObject, p.patchBytes, versionedObjToUpdate, p.schemaReferenceObj); err != nil {
        return nil, err
    }
    newObj, err := p.unsafeConvertor.ConvertToVersion(versionedObjToUpdate, p.hubGroupVersion)
    // *apps.Deployment                                *v1.Deployment        apps/__internal
```

也就是 apps/v1/Deployment 会在处理的过程中被转换为内部对象 apps/Deployment。那么这个转换又是如何完成的呢？

通过调查 p.unsafeConvertor.ConvertToVersion，其实际上调用的是：

```text
// 该方法会将 in 转换为 target 指定的类型。
k8s.io/apimachinery/pkg/runtime.Scheme.ConvertToVersion(in Object, target GroupVersioner)
k8s.io/apimachinery/pkg/conversion.Converter.Convert(src, dest interface{}, flags FieldMatchingFlags, meta *Meta)
// 该方法会被注册到 conversion.Converter 中
pkg/apis/apps/v1.Convert_v1_Deployment_To_apps_Deployment(in *appsv1.Deployment, out *apps.Deployment, s conversion.Scope)
```

所以完成 apps/v1/Deployment 到 apps/Deployment 的转换就是由 pkg/apis/apps/v1.Convert_v1_Deployment_To_apps_Deployment 实现的，对应的 apps/Deployment 到 apps/v1/Deployment 的转换是由 pkg/apis/apps/v1.Convert_apps_Deployment_To_v1_Deployment 实现，用于从 Etcd 中读取资源对象后，解码出来的是 apps/Deployment，然后根据当前请求的 apiVersion 进行转换。