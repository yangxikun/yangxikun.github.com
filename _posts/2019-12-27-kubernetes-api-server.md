---
layout: post
title: "Kubernetes APIServer 源码学习"
description: "Kubernetes APIServer 源码学习"
category: Kubernetes
tags: []
---
{% include JB/setup %}

Kubernetes 版本：1.15.7

#### 服务的创建

入口文件：cmd/kube-apiserver/apiserver.go，主要函数 CreateServerChain 在 cmd/kube-apiserver/app/server.go 文件中，负责整个服务的构建，主要逻辑如下：

CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*genericapiserver.GenericAPIServer, error)
* CreateKubeAPIServerConfig
    * 创建核心服务配置
    * 参数：
    * completedOptions completedServerRunOptions
    * 返回值：
    * kubeAPIServerConfig *master.Config // genericapiserver.Config + master.ExtraConfig
    * insecureServingInfo *genericapiserver.DeprecatedInsecureServingInfo // 是否监听 localhost:8080，没有鉴权的接口地址
    * serviceResolver aggregatorapiserver.ServiceResolver // 扩展API的服务路由
    * pluginInitializers []admission.PluginInitializer // webhookPluginInitializer + kubePluginInitializer
    * admissionPostStartHook genericapiserver.PostStartHookFunc // discoveryRESTMapper.Reset()
    * lastErr error
* createAPIExtensionsConfig
    * 创建提供 Custom Resources API 服务的配置
    * 参数：
    * kubeAPIServerConfig genericapiserver.Config // master.Config.GenericConfig
    * externalInformers kubeexternalinformers.SharedInformerFactory // master.Config.ExtraConfig.VersionedInformers
    * pluginInitializers []admission.PluginInitializer // webhookPluginInitializer + kubePluginInitializer
    * commandOptions *options.ServerRunOptions // completedOptions.ServerRunOptions
    * masterCount int, // completedOptions.MasterCount 控制节点的数量
    * serviceResolver webhook.ServiceResolver // 扩展API的服务路由
    * authResolverWrapper webhook.AuthenticationInfoResolverWrapper // webhook.AuthenticationInfoResolverDelegator
* createAPIExtensionsServer  
    * 创建 Custom Resources API 服务
    * 参数：
    * apiextensionsConfig *apiextensionsapiserver.Config
    * delegateAPIServer genericapiserver.DelegationTarget // genericapiserver.NewEmptyDelegate()
    * 返回值：
    * *apiextensionsapiserver.CustomResourceDefinitions // apiExtensionsServer
    * error
* CreateKubeAPIServer
    * 创建核心服务
    * 参数：
    * kubeAPIServerConfig *master.Config
    * delegateAPIServer genericapiserver.DelegationTarget // apiExtensionsServer.GenericAPIServer
    * admissionPostStartHook genericapiserver.PostStartHookFunc // webhookPluginInitializer + kubePluginInitializer
    * 返回值：
    * *master.Master // kubeAPIServer
    * error
* kubeAPIServer.GenericAPIServer.PrepareRun()
* apiExtensionsServer.GenericAPIServer.PrepareRun()
    * 配置开启 swagger
    * 安装/healthz健康检查处理器
    * 设置在服务关闭之前调用 GenericAPIServer.AuditBackend.Shutdown()
* createAggregatorConfig
    * 创建聚合服务配置
    * 参数：
    * kubeAPIServerConfig genericapiserver.Config
    * commandOptions *options.ServerRunOptions
    * externalInformers kubeexternalinformers.SharedInformerFactory // master.Config.ExtraConfig.VersionedInformers
    * serviceResolver aggregatorapiserver.ServiceResolver // 扩展API的服务路由
    * proxyTransport *http.Transport
    * pluginInitializers []admission.PluginInitializer // webhookPluginInitializer + kubePluginInitializer
    * 返回值：
    * *aggregatorapiserver.Config // aggregatorConfig
    * error
* createAggregatorServer
    * 创建聚合服务
    * 参数：
    * aggregatorConfig *aggregatorapiserver.Config
    * delegateAPIServer genericapiserver.DelegationTarget // kubeAPIServer.GenericAPIServer
    * apiExtensionInformers apiextensionsinformers.SharedInformerFactory // apiExtensionsServe.Informers
    * 返回值：
    * *aggregatorapiserver.APIAggregator // aggregatorServer
    * error
* BuildInsecureHandlerChain
    * 监听不安全的localhost:8080地址
    * 参数：
    * apiHandler http.Handler, // aggregatorServer.GenericAPIServer.UnprotectedHandler()
    * c *server.Config // master.Config.GenericConfig
    * 返回值：
    * http.Handler // insecureHandlerChain
* insecureServingInfo.Serve
    * 启动监听localhost:8080的服务
* aggregatorServer.GenericAPIServer.PrepareRun().Run(stopCh)
* preparedGenericAPIServer.NonBlockingRun(stopCh <-chan struct{})
    * 聚合服务启动
    * preparedGenericAPIServer.AuditBackend.Run(auditStopCh)
    * preparedGenericAPIServer.SecureServingInfo.Serve(s.Handler, s.ShutdownTimeout, internalStopCh)
        * server.RunServer(server *http.Server, ln net.Listener, shutDownTimeout time.Duration, stopCh <-chan struct{}) (<-chan struct{}, error)
    * preparedGenericAPIServer.RunPostStartHooks(stopCh)

CreateServerChain 创建的3个 Server，都采用了类似 config->complete()->completedConfig->new()->server 的调用完成，如下图：

![](/assets/img/kubernetes-apiserver.png)

<!--more-->

#### 服务路由的安装

以 kubeAPIServer 为例，调用链路如下：

kubeAPIServerConfig.Complete().New(delegateAPIServer)
1. 创建 master.Master
1. master.Master.InstallAPIs 安装服务路由，传入参数：
    1. apiResourceConfigSource serverstorage.APIResourceConfigSource // 判断路由组版本号是否启用
    1. restOptionsGetter generic.RESTOptionsGetter
    1. restStorageProviders ...RESTStorageProvider // 路由提供者，在pkg/registry/下
1. 遍历每个路由提供者，调用其 RESTStorageProvider.NewRESTStorage 方法，获得所有的 API 路由组信息：apiGroupsInfo
1. 调用 master.Master.GenericAPIServer.InstallAPIGroups(apiGroupsInfo...) 安装路由组
1. 遍历每个路由组，调用 server.GenericAPIServer.InstallAPIResources
1. 遍历每个路由组下的版本，调用 endpoints.APIGroupVersion.InstallREST(container *restful.Container)
1. 调用 endpoints.APIInstaller.Install()，遍历每个请求路径，将其 handler 添加到 restful.WebService，最后 restful.WebService 会添加到 restful.Container 中

#### 资源对象 CRUD 请求的处理

以资源对象创建为例，将 handler 作为入口点，从 vendor/k8s.io/apiserver/pkg/endpoints/installer.go:APIInstaller.registerResourceHandlers 中可以知道 POST 方法对应的 handler 由 vendor/k8s.io/apiserver/pkg/endpoints/handlers/create.go:createHandler 创建，最终会调用 k8s.io/apiserver/pkg/registry/generic/registry/store.go:Store.Create。

在 pkg/registry 下的资源对象都会将 *genericregistry.Store 作为匿名/非匿名字段，因为 *genericregistry.Store 封装了对底层存储（ETCD3）的 CRUD 操作。以 Deployment 为例，其资源对象的 REST 封装有如下3个：

1. REST：Deployment对象本身
1. StatusREST：Deployment对象的状态
1. RollbackREST：Deployment对象的回滚

通过 NewREST 创建：

```go
// NewREST returns a RESTStorage object that will work against deployments.
func NewREST(optsGetter generic.RESTOptionsGetter) (*REST, *StatusREST, *RollbackREST) {
	store := &genericregistry.Store{
		NewFunc:                  func() runtime.Object { return &apps.Deployment{} },
		NewListFunc:              func() runtime.Object { return &apps.DeploymentList{} },
		DefaultQualifiedResource: apps.Resource("deployments"),

        // 各类资源对象可以设置对应的操作策略（操作前的准备、校验等）
		CreateStrategy: deployment.Strategy,
		UpdateStrategy: deployment.Strategy,
		DeleteStrategy: deployment.Strategy,

		TableConvertor: printerstorage.TableConvertor{TableGenerator: printers.NewTableGenerator().With(printersinternal.AddHandlers)},
	}
	options := &generic.StoreOptions{RESTOptions: optsGetter}
	if err := store.CompleteWithOptions(options); err != nil {
		panic(err) // TODO: Propagate error up
	}

	statusStore := *store
	statusStore.UpdateStrategy = deployment.StatusStrategy
	return &REST{store, []string{"all"}}, &StatusREST{store: &statusStore}, &RollbackREST{store: store}
}
```

![](/assets/img/kubernetes-api.png)

以 Deployment 的创建为例，其调用链路如下：

![](/assets/img/kubernetes-apiserver-create-deployment.png)
