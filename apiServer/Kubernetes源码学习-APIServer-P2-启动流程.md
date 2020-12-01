## 前言

在前一篇[开胃菜-基础结构信息](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/apiServer/Kubernetes源码学习-APIServer-P1-基础结构信息.md)对APIServer各类基础信息、结构铺垫的基础上，开始进入APIServer源码的探讨环节。本篇拆分为 预启动、启动种两部分来分析。





## 预启动

### 资源注册

前面的基础篇有讲过，scheme是一种内存型的注册表，提供给各类gvk进行注册。在APIServer http服务启动前的第一步，就是将所支持的gvk注册到scheme中，后面的步骤会依赖scheme注册表信息。

值得注意的是，并没有函数方法来显示地注册scheme，而是通过go语言的包导入init机制来初始化注册的。

> Go语言在导入包时，被导入的包中的var/const变量以及init()函数会默认装载

`cmd/kube-apiserver/app/server.go:63`

```go
import (
		"k8s.io/kubernetes/pkg/api/legacyscheme"
  	...
  	"k8s.io/kubernetes/pkg/master"
 		...
   )

```

这里列举的两个包，legacyscheme包在导入时会初始化空的scheme注册表，master包初始化时会向scheme表install注册gvk。

#### legacyscheme包

`pkg/api/legacyscheme/scheme.go:29`

```go
// Scheme is the default instance of runtime.Scheme to which types in the Kubernetes API are already registered.
// NOTE: If you are copying this file to start a new api group, STOP! Copy the
// extensions group instead. This Scheme is special and should appear ONLY in
// the api group, unless you really know what you're doing.
// TODO(lavalamp): make the above error impossible.
var Scheme = runtime.NewScheme()

// Codecs provides access to encoding and decoding for the scheme
var Codecs = serializer.NewCodecFactory(Scheme)

// ParameterCodec handles versioning of objects that are converted to query parameters.
var ParameterCodec = runtime.NewParameterCodec(Scheme)
```



####master包

`pkg/master/import_known_versions.go:20`

```go
// These imports are the API groups the API server will support.
import (
	_ "k8s.io/kubernetes/pkg/apis/admission/install"
	_ "k8s.io/kubernetes/pkg/apis/admissionregistration/install"
	_ "k8s.io/kubernetes/pkg/apis/apps/install"
	_ "k8s.io/kubernetes/pkg/apis/auditregistration/install"
	_ "k8s.io/kubernetes/pkg/apis/authentication/install"
	_ "k8s.io/kubernetes/pkg/apis/authorization/install"
	_ "k8s.io/kubernetes/pkg/apis/autoscaling/install"
	_ "k8s.io/kubernetes/pkg/apis/batch/install"
	_ "k8s.io/kubernetes/pkg/apis/certificates/install"
	_ "k8s.io/kubernetes/pkg/apis/coordination/install"
	_ "k8s.io/kubernetes/pkg/apis/core/install"
	_ "k8s.io/kubernetes/pkg/apis/events/install"
	_ "k8s.io/kubernetes/pkg/apis/extensions/install"
	_ "k8s.io/kubernetes/pkg/apis/imagepolicy/install"
	_ "k8s.io/kubernetes/pkg/apis/networking/install"
	_ "k8s.io/kubernetes/pkg/apis/node/install"
	_ "k8s.io/kubernetes/pkg/apis/policy/install"
	_ "k8s.io/kubernetes/pkg/apis/rbac/install"
	_ "k8s.io/kubernetes/pkg/apis/scheduling/install"
	_ "k8s.io/kubernetes/pkg/apis/settings/install"
	_ "k8s.io/kubernetes/pkg/apis/storage/install"
)

```

这里会把所有内置group的install包导入，列举其中的`"k8s.io/kubernetes/pkg/apis/apps/install"` group为例，进入其中:

`pkg/apis/apps/install/install.go:31`

```go
func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(apps.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v1beta2.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta2.SchemeGroupVersion, v1beta1.SchemeGroupVersion))
}

```

这里可以看到，依次将group、version注册到了scheme中，并设置各version的优先级



### Cobra 命令参数解析

关于Cobra的参数解析，在源码系列第一篇中就有详细说明，这里略过。





### 初始化默认配置

从Run()开始进入，找到初始化默认配置的方法：

`cmd/kube-apiserver/app/server.go:144`

```go
// Run runs the specified APIServer.  This should never exit.
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())
  
  // 创建一组server链
	server, err := CreateServerChain(completeOptions, stopCh)
	if err != nil {
		return err
	}

	return server.PrepareRun().Run(stopCh)
}
```

--> `cmd/kube-apiserver/app/server.go:157` `CreateServerChain()`

--> `cmd/kube-apiserver/app/server.go:270` `CreateKubeAPIServerConfig()`

--> `cmd/kube-apiserver/app/server.go:285` `buildGenericConfig(s.ServerRunOptions, proxyTransport)`

```go
// BuildGenericConfig takes the master server options and produces the genericapiserver.Config associated with it
func buildGenericConfig(
	s *options.ServerRunOptions,
	proxyTransport *http.Transport,
) (
	genericConfig *genericapiserver.Config,
	versionedInformers clientgoinformers.SharedInformerFactory,
	insecureServingInfo *genericapiserver.DeprecatedInsecureServingInfo,
	serviceResolver aggregatorapiserver.ServiceResolver,
	pluginInitializers []admission.PluginInitializer,
	admissionPostStartHook genericapiserver.PostStartHookFunc,
	storageFactory *serverstorage.DefaultStorageFactory,
	lastErr error,
) {
	genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
	genericConfig.MergedResourceConfig = master.DefaultAPIResourceConfigSource()

 ...
}
```



APIServer因其模块众多，相应的配置参数也非常之多，分类一下：

- genericConfig 通用配置
- OpenAPI配置
- Storage (etcd)配置
- Authentication 认证配置
- Authorization 授权配置

#### genericConfig

`cmd/kube-apiserver/app/server.go:382`

```go
	genericConfig = genericapiserver.NewConfig(legacyscheme.Codecs)
	genericConfig.MergedResourceConfig = master.DefaultAPIResourceConfigSource()
```



`DefaultAPIResourceConfigSource()`方法用来启用、禁用默认的GV及其之下Resource：

`pkg/master/master.go:478`

```go
func DefaultAPIResourceConfigSource() *serverstorage.ResourceConfig {
	ret := serverstorage.NewResourceConfig()
	// NOTE: GroupVersions listed here will be enabled by default. Don't put alpha versions in the list.
	ret.EnableVersions(
		admissionregistrationv1beta1.SchemeGroupVersion,
		apiv1.SchemeGroupVersion,
		appsv1.SchemeGroupVersion,
		authenticationv1.SchemeGroupVersion,
		authenticationv1beta1.SchemeGroupVersion,
		authorizationapiv1.SchemeGroupVersion,
		authorizationapiv1beta1.SchemeGroupVersion,
		autoscalingapiv1.SchemeGroupVersion,
		autoscalingapiv2beta1.SchemeGroupVersion,
		autoscalingapiv2beta2.SchemeGroupVersion,
		batchapiv1.SchemeGroupVersion,
		batchapiv1beta1.SchemeGroupVersion,
		certificatesapiv1beta1.SchemeGroupVersion,
		coordinationapiv1.SchemeGroupVersion,
		coordinationapiv1beta1.SchemeGroupVersion,
		eventsv1beta1.SchemeGroupVersion,
		extensionsapiv1beta1.SchemeGroupVersion,
		networkingapiv1.SchemeGroupVersion,
		networkingapiv1beta1.SchemeGroupVersion,
		nodev1beta1.SchemeGroupVersion,
		policyapiv1beta1.SchemeGroupVersion,
		rbacv1.SchemeGroupVersion,
		rbacv1beta1.SchemeGroupVersion,
		storageapiv1.SchemeGroupVersion,
		storageapiv1beta1.SchemeGroupVersion,
		schedulingapiv1beta1.SchemeGroupVersion,
		schedulingapiv1.SchemeGroupVersion,
	)
	// enable non-deprecated beta resources in extensions/v1beta1 explicitly so we have a full list of what's possible to serve
	ret.EnableResources(
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("ingresses"),
	)
	// enable deprecated beta resources in extensions/v1beta1 explicitly so we have a full list of what's possible to serve
	ret.EnableResources(
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("daemonsets"),
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("deployments"),
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("networkpolicies"),
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("podsecuritypolicies"),
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("replicasets"),
		extensionsapiv1beta1.SchemeGroupVersion.WithResource("replicationcontrollers"),
	)
	// enable deprecated beta versions explicitly so we have a full list of what's possible to serve
	ret.EnableVersions(
		appsv1beta1.SchemeGroupVersion,
		appsv1beta2.SchemeGroupVersion,
	)
	// disable alpha versions explicitly so we have a full list of what's possible to serve
	ret.DisableVersions(
		auditregistrationv1alpha1.SchemeGroupVersion,
		batchapiv2alpha1.SchemeGroupVersion,
		nodev1alpha1.SchemeGroupVersion,
		rbacv1alpha1.SchemeGroupVersion,
		schedulingv1alpha1.SchemeGroupVersion,
		settingsv1alpha1.SchemeGroupVersion,
		storageapiv1alpha1.SchemeGroupVersion,
	)

	return ret
}

```



#### OpenAPI配置

`cmd/kube-apiserver/app/server.go:405`

```go
genericConfig.OpenAPIConfig = genericapiserver.DefaultOpenAPIConfig(generatedopenapi.GetOpenAPIDefinitions, openapinamer.NewDefinitionNamer(legacyscheme.Scheme, extensionsapiserver.Scheme, aggregatorscheme.Scheme))

```

根据OpenAPI规范的要求，定义OpenAPIDefinition文件，关于OpenAPI规范，可参考：

https://swagger.io/specification/#introduction



#### Storage (etcd)配置

`cmd/kube-apiserver/app/server.go:415`

```go
	storageFactoryConfig := kubeapiserver.NewStorageFactoryConfig()
	storageFactoryConfig.ApiResourceConfig = genericConfig.MergedResourceConfig
	completedStorageFactoryConfig, err := storageFactoryConfig.Complete(s.Etcd)
```

这里实例化了etcd storage对象，定义了etcd地址、认证、存储路径prefix等信息。



#### 认证配置

APIServer支持如下的认证策略：

- X509 Client Certs

- Static Token File

- Bootstrap Tokens

- Service Account Tokens

- OpenID Connect Tokens(OIDC)

- Webhook Token Authentication

- Authenticating Proxy

参考官方文档：

https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authentication-strategies

每一种认证策略都会被实例化为一个Authenticator认证器，被封装进http.Handler函数中来接收和处理客户端的认证请求。

`cmd/kube-apiserver/app/server.go:444`

```go
	genericConfig.Authentication.Authenticator, genericConfig.OpenAPIConfig.SecurityDefinitions, err = BuildAuthenticator(s, clientgoExternalClient, versionedInformers)
	if err != nil {
		lastErr = fmt.Errorf("invalid authentication config: %v", err)
		return
	}
```

--> `cmd/kube-apiserver/app/server.go:515`

```go
// BuildAuthenticator constructs the authenticator
func BuildAuthenticator(s *options.ServerRunOptions, extclient clientgoclientset.Interface, versionedInformer clientgoinformers.SharedInformerFactory) (authenticator.Request, *spec.SecurityDefinitions, error) {
  ...

	return authenticatorConfig.New()
}
```

--> `pkg/kubeapiserver/authenticator/config.go:85`

```go
func (config Config) New() (authenticator.Request, *spec.SecurityDefinitions, error) {
	var authenticators []authenticator.Request
	var tokenAuthenticators []authenticator.Token
	securityDefinitions := spec.SecurityDefinitions{}

	// front-proxy, BasicAuth methods, local first, then remote
	// Add the front proxy authenticator if requested
	if config.RequestHeaderConfig != nil {
		requestHeaderAuthenticator, err := headerrequest.NewSecure(
			config.RequestHeaderConfig.ClientCA,
			config.RequestHeaderConfig.AllowedClientNames,
			config.RequestHeaderConfig.UsernameHeaders,
			config.RequestHeaderConfig.GroupHeaders,
			config.RequestHeaderConfig.ExtraHeaderPrefixes,
		)
		if err != nil {
			return nil, nil, err
		}
		authenticators = append(authenticators, authenticator.WrapAudienceAgnosticRequest(config.APIAudiences, requestHeaderAuthenticator))
	}

	// basic auth
	if len(config.BasicAuthFile) > 0 {
		basicAuth, err := newAuthenticatorFromBasicAuthFile(config.BasicAuthFile)
		if err != nil {
			return nil, nil, err
		}
		authenticators = append(authenticators, authenticator.WrapAudienceAgnosticRequest(config.APIAudiences, basicAuth))

		securityDefinitions["HTTPBasic"] = &spec.SecurityScheme{
			SecuritySchemeProps: spec.SecuritySchemeProps{
				Type:        "basic",
				Description: "HTTP Basic authentication",
			},
		}
	}

	// X509 methods
	if len(config.ClientCAFile) > 0 {
		certAuth, err := newAuthenticatorFromClientCAFile(config.ClientCAFile)
		if err != nil {
			return nil, nil, err
		}
		authenticators = append(authenticators, certAuth)
	}

	// Bearer token methods, local first, then remote
	if len(config.TokenAuthFile) > 0 {
		tokenAuth, err := newAuthenticatorFromTokenFile(config.TokenAuthFile)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, tokenAuth))
	}
	if len(config.ServiceAccountKeyFiles) > 0 {
		serviceAccountAuth, err := newLegacyServiceAccountAuthenticator(config.ServiceAccountKeyFiles, config.ServiceAccountLookup, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.TokenRequest) && config.ServiceAccountIssuer != "" {
		serviceAccountAuth, err := newServiceAccountAuthenticator(config.ServiceAccountIssuer, config.ServiceAccountKeyFiles, config.APIAudiences, config.ServiceAccountTokenGetter)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, serviceAccountAuth)
	}
	if config.BootstrapToken {
		if config.BootstrapTokenAuthenticator != nil {
			// TODO: This can sometimes be nil because of
			tokenAuthenticators = append(tokenAuthenticators, authenticator.WrapAudienceAgnosticToken(config.APIAudiences, config.BootstrapTokenAuthenticator))
		}
	}
	// NOTE(ericchiang): Keep the OpenID Connect after Service Accounts.
	//
	// Because both plugins verify JWTs whichever comes first in the union experiences
	// cache misses for all requests using the other. While the service account plugin
	// simply returns an error, the OpenID Connect plugin may query the provider to
	// update the keys, causing performance hits.
	if len(config.OIDCIssuerURL) > 0 && len(config.OIDCClientID) > 0 {
		oidcAuth, err := newAuthenticatorFromOIDCIssuerURL(oidc.Options{
			IssuerURL:            config.OIDCIssuerURL,
			ClientID:             config.OIDCClientID,
			APIAudiences:         config.APIAudiences,
			CAFile:               config.OIDCCAFile,
			UsernameClaim:        config.OIDCUsernameClaim,
			UsernamePrefix:       config.OIDCUsernamePrefix,
			GroupsClaim:          config.OIDCGroupsClaim,
			GroupsPrefix:         config.OIDCGroupsPrefix,
			SupportedSigningAlgs: config.OIDCSigningAlgs,
			RequiredClaims:       config.OIDCRequiredClaims,
		})
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, oidcAuth)
	}
	if len(config.WebhookTokenAuthnConfigFile) > 0 {
		webhookTokenAuth, err := newWebhookTokenAuthenticator(config.WebhookTokenAuthnConfigFile, config.WebhookTokenAuthnCacheTTL, config.APIAudiences)
		if err != nil {
			return nil, nil, err
		}
		tokenAuthenticators = append(tokenAuthenticators, webhookTokenAuth)
	}

	if len(tokenAuthenticators) > 0 {
		// Union the token authenticators
		tokenAuth := tokenunion.New(tokenAuthenticators...)
		// Optionally cache authentication results
		if config.TokenSuccessCacheTTL > 0 || config.TokenFailureCacheTTL > 0 {
			tokenAuth = tokencache.New(tokenAuth, true, config.TokenSuccessCacheTTL, config.TokenFailureCacheTTL)
		}
		authenticators = append(authenticators, bearertoken.New(tokenAuth), websocket.NewProtocolAuthenticator(tokenAuth))
		securityDefinitions["BearerToken"] = &spec.SecurityScheme{
			SecuritySchemeProps: spec.SecuritySchemeProps{
				Type:        "apiKey",
				Name:        "authorization",
				In:          "header",
				Description: "Bearer Token authentication",
			},
		}
	}

	if len(authenticators) == 0 {
		if config.Anonymous {
			return anonymous.NewAuthenticator(), &securityDefinitions, nil
		}
		return nil, &securityDefinitions, nil
	}
  
  // 集合所有的认证器
	authenticator := union.New(authenticators...)

	authenticator = group.NewAuthenticatedGroupAdder(authenticator)

	if config.Anonymous {
		// If the authenticator chain returns an error, return an error (don't consider a bad bearer token
		// or invalid username/password combination anonymous).
		authenticator = union.NewFailOnError(authenticator, anonymous.NewAuthenticator())
	}

	return authenticator, &securityDefinitions, nil
}
```

这个New()方法依次生成每一种认证方式的Authenticator，New()方法的流程图如下：

<img src="https://mycloudn.upweto.top/20201130153857.png" style="zoom:50%;" />

`union.New(authenticators...)`方法集合所有的认证器

`vendor/k8s.io/apiserver/pkg/authentication/request/union/union.go:36`

```go
func New(authRequestHandlers ...authenticator.Request) authenticator.Request {
	if len(authRequestHandlers) == 1 {
		return authRequestHandlers[0]
	}
	return &unionAuthRequestHandler{Handlers: authRequestHandlers, FailOnError: false}
}
```

--> `vendor/k8s.io/apiserver/pkg/authentication/request/union/union.go:36`

```go
// New returns a request authenticator that validates credentials using a chain of authenticator.Request objects.
// The entire chain is tried until one succeeds. If all fail, an aggregate error is returned.
func New(authRequestHandlers ...authenticator.Request) authenticator.Request {
	if len(authRequestHandlers) == 1 {
		return authRequestHandlers[0]
	}
	return &unionAuthRequestHandler{Handlers: authRequestHandlers, FailOnError: false}
}

```

当认证请求进来时，遍历各个认证器，只要有一个认证器返回true则视为认证成功，全部为false则认证失败。



#### 授权配置

授权配置的流程与认证配置基本一致，快速略过。

APIServer支持如下的授权策略：

- AlwaysAllow

- AlwaysDeny

- webhook授权

- node授权

- ABAC授权

- RBAC授权



参考官方文档：

https://kubernetes.io/docs/reference/access-authn-authz/authorization/

`cmd/kube-apiserver/app/server.go:519`

```go
func BuildAuthorizer(s *options.ServerRunOptions, versionedInformers clientgoinformers.SharedInformerFactory) (authorizer.Authorizer, authorizer.RuleResolver, error) {
	authorizationConfig := s.Authorization.ToAuthorizationConfig(versionedInformers)
	return authorizationConfig.New()
}
```

--> `pkg/kubeapiserver/authorizer/config.go:59`

```go
func (config Config) New() (authorizer.Authorizer, authorizer.RuleResolver, error) {
	if len(config.AuthorizationModes) == 0 {
		return nil, nil, fmt.Errorf("at least one authorization mode must be passed")
	}

	var (
		authorizers   []authorizer.Authorizer
		ruleResolvers []authorizer.RuleResolver
	)

	for _, authorizationMode := range config.AuthorizationModes {
		// Keep cases in sync with constant list in k8s.io/kubernetes/pkg/kubeapiserver/authorizer/modes/modes.go.
		switch authorizationMode {
		case modes.ModeNode:
			graph := node.NewGraph()
			node.AddGraphEventHandlers(
				graph,
				config.VersionedInformerFactory.Core().V1().Nodes(),
				config.VersionedInformerFactory.Core().V1().Pods(),
				config.VersionedInformerFactory.Core().V1().PersistentVolumes(),
				config.VersionedInformerFactory.Storage().V1().VolumeAttachments(),
			)
			nodeAuthorizer := node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())
			authorizers = append(authorizers, nodeAuthorizer)

		case modes.ModeAlwaysAllow:
			alwaysAllowAuthorizer := authorizerfactory.NewAlwaysAllowAuthorizer()
			authorizers = append(authorizers, alwaysAllowAuthorizer)
			ruleResolvers = append(ruleResolvers, alwaysAllowAuthorizer)
		case modes.ModeAlwaysDeny:
			alwaysDenyAuthorizer := authorizerfactory.NewAlwaysDenyAuthorizer()
			authorizers = append(authorizers, alwaysDenyAuthorizer)
			ruleResolvers = append(ruleResolvers, alwaysDenyAuthorizer)
		case modes.ModeABAC:
			abacAuthorizer, err := abac.NewFromFile(config.PolicyFile)
			if err != nil {
				return nil, nil, err
			}
			authorizers = append(authorizers, abacAuthorizer)
			ruleResolvers = append(ruleResolvers, abacAuthorizer)
		case modes.ModeWebhook:
			webhookAuthorizer, err := webhook.New(config.WebhookConfigFile,
				config.WebhookCacheAuthorizedTTL,
				config.WebhookCacheUnauthorizedTTL)
			if err != nil {
				return nil, nil, err
			}
			authorizers = append(authorizers, webhookAuthorizer)
			ruleResolvers = append(ruleResolvers, webhookAuthorizer)
		case modes.ModeRBAC:
			rbacAuthorizer := rbac.New(
				&rbac.RoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().Roles().Lister()},
				&rbac.RoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().RoleBindings().Lister()},
				&rbac.ClusterRoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoles().Lister()},
				&rbac.ClusterRoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoleBindings().Lister()},
			)
			authorizers = append(authorizers, rbacAuthorizer)
			ruleResolvers = append(ruleResolvers, rbacAuthorizer)
		default:
			return nil, nil, fmt.Errorf("unknown authorization mode %s specified", authorizationMode)
		}
	}

	return union.New(authorizers...), union.NewRuleResolvers(ruleResolvers...), nil
}
```



### 创建APIExtensionsServer

上面在初始化完默认的配置后，并没有立即装载创建APIServer，K8s专门设计了一个扩展型的APIExtensionsServer，为CRD提供服务，首先创建的是APIExtensionsServer. 

回到这里:

`cmd/kube-apiserver/app/server.go:163`

```go
  // 初始化默认配置	
  kubeAPIServerConfig, insecureServingInfo, serviceResolver, pluginInitializer, admissionPostStartHook, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
	if err != nil {
		return nil, err
	}

	// 初始化APIExtensionsConfig
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(proxyTransport, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig))
	if err != nil {
		return nil, err
	}
// 创建APIExtensionsServer
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}
```



下面来看一下创建APIExtensionsServer的流程：

`cmd/kube-apiserver/app/apiextensions.go:93`

--> `vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go:130`

```go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*CustomResourceDefinitions, error) {
	genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}
  // 初始化CRD
	s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}

	apiResourceConfig := c.GenericConfig.MergedResourceConfig
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
	if apiResourceConfig.VersionEnabled(v1beta1.SchemeGroupVersion) {
		storage := map[string]rest.Storage{}
		// customresourcedefinitions
		customResourceDefintionStorage := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		storage["customresourcedefinitions"] = customResourceDefintionStorage
		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)

		apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = storage
	}

	if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
		return nil, err
	}

	crdClient, err := internalclientset.NewForConfig(s.GenericAPIServer.LoopbackClientConfig)
	if err != nil {
		// it's really bad that this is leaking here, but until we can fix the test (which I'm pretty sure isn't even testing what it wants to test),
		// we need to be able to move forward
		return nil, fmt.Errorf("failed to create clientset: %v", err)
	}
	s.Informers = internalinformers.NewSharedInformerFactory(crdClient, 5*time.Minute)

	delegateHandler := delegationTarget.UnprotectedHandler()
	if delegateHandler == nil {
		delegateHandler = http.NotFoundHandler()
	}

	versionDiscoveryHandler := &versionDiscoveryHandler{
		discovery: map[schema.GroupVersion]*discovery.APIVersionHandler{},
		delegate:  delegateHandler,
	}
	groupDiscoveryHandler := &groupDiscoveryHandler{
		discovery: map[string]*discovery.APIGroupHandler{},
		delegate:  delegateHandler,
	}
	establishingController := establish.NewEstablishingController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), crdClient.Apiextensions())
	crdHandler, err := NewCustomResourceDefinitionHandler(
		versionDiscoveryHandler,
		groupDiscoveryHandler,
		s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(),
		delegateHandler,
		c.ExtraConfig.CRDRESTOptionsGetter,
		c.GenericConfig.AdmissionControl,
		establishingController,
		c.ExtraConfig.ServiceResolver,
		c.ExtraConfig.AuthResolverWrapper,
		c.ExtraConfig.MasterCount,
		s.GenericAPIServer.Authorizer,
	)
	if err != nil {
		return nil, err
	}
	s.GenericAPIServer.Handler.NonGoRestfulMux.Handle("/apis", crdHandler)
	s.GenericAPIServer.Handler.NonGoRestfulMux.HandlePrefix("/apis/", crdHandler)

	crdController := NewDiscoveryController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), versionDiscoveryHandler, groupDiscoveryHandler)
	namingController := status.NewNamingConditionController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(), crdClient.Apiextensions())
	finalizingController := finalizer.NewCRDFinalizer(
		s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions(),
		crdClient.Apiextensions(),
		crdHandler,
	)
	var openapiController *openapicontroller.Controller
	if utilfeature.DefaultFeatureGate.Enabled(apiextensionsfeatures.CustomResourcePublishOpenAPI) {
		openapiController = openapicontroller.NewController(s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions())
	}

	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-informers", func(context genericapiserver.PostStartHookContext) error {
		s.Informers.Start(context.StopCh)
		return nil
	})
	s.GenericAPIServer.AddPostStartHookOrDie("start-apiextensions-controllers", func(context genericapiserver.PostStartHookContext) error {
		if utilfeature.DefaultFeatureGate.Enabled(apiextensionsfeatures.CustomResourcePublishOpenAPI) {
			go openapiController.Run(s.GenericAPIServer.StaticOpenAPISpec, s.GenericAPIServer.OpenAPIVersionedService, context.StopCh)
		}

		go crdController.Run(context.StopCh)
		go namingController.Run(context.StopCh)
		go establishingController.Run(context.StopCh)
		go finalizingController.Run(5, context.StopCh)
		return nil
	})
	// we don't want to report healthy until we can handle all CRDs that have already been registered.  Waiting for the informer
	// to sync makes sure that the lister will be valid before we begin.  There may still be races for CRDs added after startup,
	// but we won't go healthy until we can handle the ones already present.
	s.GenericAPIServer.AddPostStartHookOrDie("crd-informer-synced", func(context genericapiserver.PostStartHookContext) error {
		return wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
			return s.Informers.Apiextensions().InternalVersion().CustomResourceDefinitions().Informer().HasSynced(), nil
		}, context.StopCh)
	})

	return s, nil
}

```

拆解一下这个函数的流程：

1、创建GenericServer

2、实例化CRD 的APIServer

3、将CRD的GV/GVR及子资源与storage对象进行关联映射

4、注册APIGroup(####注册APIGroup)



####创建GenericServer

```go
genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
```

genericServer提供了一个通用的http server，定义了通用的模板，例如地址、端口、认证、授权、健康检查等等通用功能。无论是APIServer还是APIExtensionsServer都依赖于genericServer。



#### 实例化CRD和APIGroup

```go
	s := &CustomResourceDefinitions{
		GenericAPIServer: genericServer,
	}

	apiResourceConfig := c.GenericConfig.MergedResourceConfig
	apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(apiextensions.GroupName, Scheme, metav1.ParameterCodec, Codecs)
  // 判断配置中是否开启了apiextensions.k8s.io/v1beta1这个GV
	if apiResourceConfig.VersionEnabled(v1beta1.SchemeGroupVersion) {
		storage := map[string]rest.Storage{}
		// 生成CRD对应的RESTStorage
		customResourceDefintionStorage := customresourcedefinition.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
		storage["customresourcedefinitions"] = customResourceDefintionStorage
    // 生成CRD status子资源对应的RESTStorage
		storage["customresourcedefinitions/status"] = customresourcedefinition.NewStatusREST(Scheme, customResourceDefintionStorage)
    // 将apiGroupInfo和RESTStorage关联起来，下一步注册apiGroupInfo会用到
		apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = storage
	}
```

重点是里面的`customresourcedefinition.NewREST()`方法：

`vendor/k8s.io/apiextensions-apiserver/pkg/registry/customresourcedefinition/etcd.go:41`

```go
// NewREST returns a RESTStorage object that will work against API services.
func NewREST(scheme *runtime.Scheme, optsGetter generic.RESTOptionsGetter) *REST {
   strategy := NewStrategy(scheme)

   store := &genericregistry.Store{
      NewFunc:                  func() runtime.Object { return &apiextensions.CustomResourceDefinition{} },
      NewListFunc:              func() runtime.Object { return &apiextensions.CustomResourceDefinitionList{} },
      PredicateFunc:            MatchCustomResourceDefinition,
      DefaultQualifiedResource: apiextensions.Resource("customresourcedefinitions"),

      CreateStrategy: strategy,
      UpdateStrategy: strategy,
      DeleteStrategy: strategy,
   }
   options := &generic.StoreOptions{RESTOptions: optsGetter, AttrFunc: GetAttrs}
   if err := store.CompleteWithOptions(options); err != nil {
      panic(err) // TODO: Propagate error up
   }
   return &REST{store}
}
```

核心是`genericregistry.Store{}`结构体:

`vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go:81`

```go
// The intended use of this type is embedding within a Kind specific
// RESTStorage implementation. This type provides CRUD semantics on a Kubelike
// resource, handling details like conflict detection with ResourceVersion and
// semantics. The RESTCreateStrategy, RESTUpdateStrategy, and
// RESTDeleteStrategy are generic across all backends, and encapsulate logic
// specific to the API.
//
// TODO: make the default exposed methods exactly match a generic RESTStorage
type Store struct {
   // NewFunc returns a new instance of the type this registry returns for a
   // GET of a single object, e.g.:
   //
   // curl GET /apis/group/version/namespaces/my-ns/myresource/name-of-object
   NewFunc func() runtime.Object

   // NewListFunc returns a new list of the type this registry; it is the
   // type returned when the resource is listed, e.g.:
   //
   // curl GET /apis/group/version/namespaces/my-ns/myresource
   NewListFunc func() runtime.Object
 ... 
}
```

从注释中可以看出，Store结构体是一个对于各类k8s资源通用的REST封装的实现，可以实现CURD功能以及满足相应的CURD策略，同时实现了对接后端etcd存储进行相应的增删改查操作。

在这里注册了CRD对应的Store后，即可实现其对应的CURD方法，例如查询某个CRD对象对应的api是:

`GET /apis/apiextensions.k8s.io/v1beta1/namespaces/${namespace}/crd/${crd-name}`



#### 注册APIGroup

`vendor/k8s.io/apiextensions-apiserver/pkg/apiserver/apiserver.go:152`

--> `vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:438`

--> `vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:385`

```go
func (s *GenericAPIServer) InstallAPIGroups(apiGroupInfos ...*APIGroupInfo) error {
   ...
   // 注册APIGroup下的所有resource并注册
   for _, apiGroupInfo := range apiGroupInfos {
      if err := s.installAPIResources(APIGroupPrefix, apiGroupInfo, openAPIModels); err != nil {
         return fmt.Errorf("unable to install api resources: %v", err)
      }

      // setup discovery
      // Install the version handler.
      // Add a handler at /apis/<groupName> to enumerate all versions supported by this group.
      apiVersionsForDiscovery := []metav1.GroupVersionForDiscovery{}
      for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
         // Check the config to make sure that we elide versions that don't have any resources
         if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
            continue
         }
         apiVersionsForDiscovery = append(apiVersionsForDiscovery, metav1.GroupVersionForDiscovery{
            GroupVersion: groupVersion.String(),
            Version:      groupVersion.Version,
         })
      }
      preferredVersionForDiscovery := metav1.GroupVersionForDiscovery{
         GroupVersion: apiGroupInfo.PrioritizedVersions[0].String(),
         Version:      apiGroupInfo.PrioritizedVersions[0].Version,
      }
      apiGroup := metav1.APIGroup{
         Name:             apiGroupInfo.PrioritizedVersions[0].Group,
         Versions:         apiVersionsForDiscovery,
         PreferredVersion: preferredVersionForDiscovery,
      }

      s.DiscoveryGroupManager.AddGroup(apiGroup)
      s.Handler.GoRestfulContainer.Add(discovery.NewAPIGroupHandler(s.Serializer, apiGroup).WebService())
   }
   return nil
}
```

来看看`s.installAPIResources()`方法：

`vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:341`

```go
// installAPIResources is a private method for installing the REST storage backing each api groupversionresource
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, openAPIModels openapiproto.Models) error {
   for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
      if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
         klog.Warningf("Skipping API %v because it has no resources.", groupVersion)
         continue
      }

      apiGroupVersion := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)
      if apiGroupInfo.OptionsExternalVersion != nil {
         apiGroupVersion.OptionsExternalVersion = apiGroupInfo.OptionsExternalVersion
      }
      apiGroupVersion.OpenAPIModels = openAPIModels
      apiGroupVersion.MaxRequestBodyBytes = s.maxRequestBodyBytes

      if err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer); err != nil {
         return fmt.Errorf("unable to setup API %v: %v", apiGroupInfo, err)
      }
   }

   return nil
}
```

遍历GroupVersion，调用` apiGroupVersion.InstallREST()`方法:

`vendor/k8s.io/apiserver/pkg/endpoints/groupversion.go:99`

```go
// InstallREST registers the REST handlers (storage, watch, proxy and redirect) into a restful Container.
// It is expected that the provided path root prefix will serve all operations. Root MUST NOT end
// in a slash.
func (g *APIGroupVersion) InstallREST(container *restful.Container) error {
   prefix := path.Join(g.Root, g.GroupVersion.Group, g.GroupVersion.Version)
   installer := &APIInstaller{
      group:                        g,
      prefix:                       prefix,
      minRequestTimeout:            g.MinRequestTimeout,
      enableAPIResponseCompression: g.EnableAPIResponseCompression,
   }

   apiResources, ws, registrationErrors := installer.Install()
   versionDiscoveryHandler := discovery.NewAPIVersionHandler(g.Serializer, g.GroupVersion, staticLister{apiResources})
   versionDiscoveryHandler.AddToWebService(ws)
   container.Add(ws)
   return utilerrors.NewAggregate(registrationErrors)
}
```

调用installer.Install()安装VersionGroup内的Resource：

`vendor/k8s.io/apiserver/pkg/endpoints/installer.go:95`

```go
func (a *APIInstaller) Install() ([]metav1.APIResource, *restful.WebService, []error) {
	var apiResources []metav1.APIResource
	var errors []error
  // new 一个go-restful WebService实例
	ws := a.newWebService()

	// Register the paths in a deterministic (sorted) order to get a deterministic swagger spec.
	paths := make([]string, len(a.group.Storage))
	var i int = 0
	for path := range a.group.Storage {
		paths[i] = path
		i++
	}
	sort.Strings(paths)
	for _, path := range paths {
    // 将上一步的RESTStorage与ws绑定，通过此方法，将http 路径路由到RESTStorage对应的方法上去
		apiResource, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
		if err != nil {
			errors = append(errors, fmt.Errorf("error in registering resource: %s, %v", path, err))
		}
		if apiResource != nil {
			apiResources = append(apiResources, *apiResource)
		}
	}
	return apiResources, ws, errors
}
```

在这里，用到了上一章基础篇提到的go-restful模块，通过go-restful WebService对象，`a.registerResourceHandlers()`方法将http访问路径与上一步得到的RESTStorage对象进行绑定，即可把相应路径的请求路由到RESTStorage对应的方法上去。



APIExtensionsServer启动完成后，可以通过如下路径访问该GV下的资源列表:

```shell
# 先起一个免认证代理
root@ubuntu238:~# nohup kubectl proxy --address=0.0.0.0 --port=8000 --accept-hosts=^.* &

root@ubuntu238:~# curl http://127.0.0.1:8000/apis/apiextensions.k8s.io/v1beta1/
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apiextensions.k8s.io/v1beta1",
  "resources": [
    {
      "name": "customresourcedefinitions",
      "singularName": "",
      "namespaced": false,
      "kind": "CustomResourceDefinition",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "shortNames": [
        "crd",
        "crds"
      ],
      "storageVersionHash": "jfWCUB31mvA="
    },
    {
      "name": "customresourcedefinitions/status",
      "singularName": "",
      "namespaced": false,
      "kind": "CustomResourceDefinition",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```





## 启动中

### 创建Kube APIServer (master apiserver)

回到`cmd/kube-apiserver/app/server.go:179`

```go
kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer, admissionPostStartHook)
if err != nil {
   return nil, err
}
```

-->`cmd/kube-apiserver/app/server.go:214`

--> `pkg/master/master.go:299`

```go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Master, error) {
   if reflect.DeepEqual(c.ExtraConfig.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
      return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
   }

   s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
   if err != nil {
      return nil, err
   }

   if c.ExtraConfig.EnableLogsSupport {
      routes.Logs{}.Install(s.Handler.GoRestfulContainer)
   }

   m := &Master{
      GenericAPIServer: s,
   }

   // install legacy rest storage
   if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
      legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
         StorageFactory:              c.ExtraConfig.StorageFactory,
         ProxyTransport:              c.ExtraConfig.ProxyTransport,
         KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
         EventTTL:                    c.ExtraConfig.EventTTL,
         ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
         ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
         LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
         ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
         ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
         APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
      }
      m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
   }

   // The order here is preserved in discovery.
   // If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
   // the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
   // This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
   // with specific priorities.
   // TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
   // handlers that we have.
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

   if c.ExtraConfig.Tunneler != nil {
      m.installTunneler(c.ExtraConfig.Tunneler, corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig).Nodes())
   }

   m.GenericAPIServer.AddPostStartHookOrDie("ca-registration", c.ExtraConfig.ClientCARegistrationHook.PostStartHook)

   return m, nil
}
```



APIServer启动分为以下几步：

1、创建GenericServer

2、实例化master APIServer(Kube APIServer)

3、注册/api下的资源(LegacyAPI)

4、注册/apis下的资源

其实与上面启动APIExtensionsServer的流程基本大差不差，篇幅有限重复的地方尽量简略跳过，直接进入第三步。



#### 注册LegacyAPI

```go
// install legacy rest storage
if c.ExtraConfig.APIResourceConfigSource.VersionEnabled(apiv1.SchemeGroupVersion) {
   legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
      StorageFactory:              c.ExtraConfig.StorageFactory,
      ProxyTransport:              c.ExtraConfig.ProxyTransport,
      KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
      EventTTL:                    c.ExtraConfig.EventTTL,
      ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
      ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
      LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
      ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
      ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
      APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
   }
   m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter, legacyRESTStorageProvider)
}
```

--> `pkg/master/master.go:374`

```go
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) {
   legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)
   if err != nil {
      klog.Fatalf("Error building core storage: %v", err)
   }

   controllerName := "bootstrap-controller"
   coreClient := corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig)
   bootstrapController := c.NewBootstrapController(legacyRESTStorage, coreClient, coreClient, coreClient, coreClient.RESTClient())
   m.GenericAPIServer.AddPostStartHookOrDie(controllerName, bootstrapController.PostStartHook)
   m.GenericAPIServer.AddPreShutdownHookOrDie(controllerName, bootstrapController.PreShutdownHook)

   if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
      klog.Fatalf("Error in registering group versions: %v", err)
   }
}
```

展开`legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)`可以看到，这里注册的都是core/v1组里的资源，例如(pod/service/replicas/nodes)等等，所谓Legacy，指的即是核心组，路径是/api。后面的步骤与APIExtensionsServer里的相似，不再赘述。



#### 注册/apis下的资源

```go
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
```

注册/apis路径下的GV内的资源，例如apps/extension组等。



### 创建AggregatorServer

再回到`cmd/kube-apiserver/app/server.go:192`，进入下一步，创建AggregatorServer：

```go
// aggregator comes last in the chain
aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, proxyTransport, pluginInitializer)
if err != nil {
   return nil, err
}
aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
if err != nil {
   // we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
   return nil, err
}
```

AggregatorServer也与APIExtensionsServer类似，只不过group换成了apiregistration.k8s.io，其余基本一致，不再赘述，这个组里的资源列表可以这样查看:

```shell
root@ubuntu238:~# curl http://127.0.0.1:8000/apis/apiregistration.k8s.io/v1beta1/
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apiregistration.k8s.io/v1beta1",
  "resources": [
    {
      "name": "apiservices",
      "singularName": "",
      "namespaced": false,
      "kind": "APIService",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "storageVersionHash": "C+s2HXXP47k="
    },
    {
      "name": "apiservices/status",
      "singularName": "",
      "namespaced": false,
      "kind": "APIService",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}	
```



### 启动web服务

回到最初起点这里 `cmd/kube-apiserver/app/server.go:153`

```go
	server, err := CreateServerChain(completeOptions, stopCh)
	if err != nil {
		return err
	}
  // server chain组创建完成后，进入启动web服务环节
	return server.PrepareRun().Run(stopCh)
```

--> `vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:272 ` `s.NonBlockingRun(stopCh)`

--> `vendor/k8s.io/apiserver/pkg/server/genericapiserver.go:311` `s.SecureServingInfo.Serve()`

```go
secureServer := &http.Server{
   Addr:           s.Listener.Addr().String(),
   Handler:        handler,
   MaxHeaderBytes: 1 << 20,
   TLSConfig: &tls.Config{
      NameToCertificate: s.SNICerts,
      // Can't use SSLv3 because of POODLE and BEAST
      // Can't use TLSv1.0 because of POODLE and BEAST using CBC cipher
      // Can't use TLSv1.1 because of RC4 cipher usage
      MinVersion: tls.VersionTLS12,
      // enable HTTP2 for go's 1.7 HTTP Server
      NextProtos: []string{"h2", "http/1.1"},
   },
}
```

配置tls证书相关配置

`vendor/k8s.io/apiserver/pkg/server/secure_serving.go:126`  

```go
go func() {
   defer utilruntime.HandleCrash()

   var listener net.Listener
   listener = tcpKeepAliveListener{ln.(*net.TCPListener)}
   if server.TLSConfig != nil {
      listener = tls.NewListener(listener, server.TLSConfig)
   }

   err := server.Serve(listener)

   msg := fmt.Sprintf("Stopped listening on %s", ln.Addr().String())
   select {
   case <-stopCh:
      klog.Info(msg)
   default:
      panic(fmt.Sprintf("%s due to error: %v", msg, err))
   }
}()
```

可以看到这里使用的是go标准库内置的http模块提供web服务



## 总结

APIServer的总体启动流程分为这几个步骤:

1. 资源注册
2. 默认配置初始化
3. 注册server chain组
4. 启动服务

