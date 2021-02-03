## 前言

在上一篇[**APIServer-P2-启动流程**](https://github.com/yinwenqin/kubeSourceCodeNote/blob/master/apiServer/Kubernetes%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-APIServer-P2-%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.md)中，把APIServer预启动、加载配置、启动 的流程大致过了一遍，其中也有提到，APIServer的认证机制分为多种，以组合的形式遍历认证，任一机制认证成功，则判定请求认证成功，顺利进入下一个授权判定的环节。那么在本篇，就来详细看看这些认证机制。



## Kubernetes用户

所有 Kubernetes 集群都有两类用户：由 Kubernetes 管理的**服务账号**和**普通用户**。

 其中服务账号(ServiceAccount)是提供给集群中的程序使用，以Secret资源保存凭据，挂载到pod中，从而允许集群内的服务调用k8s API。

而普通用户，尚不支持使用API创建，一般由证书创建，Kubernetes 使用证书中的 'subject' 的通用名称（Common Name）字段（例如，"/CN=bob"）来 确定用户名。

> 以上摘自官方文档



## 身份认证策略

Kubernetes 使用身份认证插件利用客户端证书、持有者令牌（Bearer Token）、身份认证代理（Proxy） 或者 HTTP 基本认证机制来认证 API 请求的身份。HTTP 请求发给 API 服务器时， 插件会将以下属性关联到请求本身：

- 用户名：用来辩识最终用户的字符串。常见的值可以是 `kube-admin` 或 `jane@example.com`。
- 用户 ID：用来辩识最终用户的字符串，旨在比用户名有更好的一致性和唯一性。
- 用户组：取值为一组字符串，其中各个字符串用来标明用户是某个命名的用户逻辑集合的成员。 常见的值可能是 `system:masters` 或者 `devops-team` 等。
- 附加字段：一组额外的键-值映射，键是字符串，值是一组字符串；用来保存一些鉴权组件可能 觉得有用的额外信息。

与其它身份认证协议（LDAP、SAML、Kerberos、X509 的替代模式等等）都可以通过 使用一个[身份认证代理](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#authenticating-proxy)或 [身份认证 Webhoook](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)来实现。

> 以上摘自官方文档





## 认证机制

### 流程图

首先为了加深印象，再贴一下上一章列出的认证流程图：

![](https://mycloudn.upweto.top/20210203112246.png)

>  注：认证器的执行顺序为随机的，图中流向不代表固定顺序

当客户端发送请求到达APIServer端，请求首先进入认证环节，对应处理认证的是Authentication Handler方法，

Authentication Handler方法中，遍历每一个已启用的认证器，仅需某一个认证器返回true，则认证成功结束遍历，若所有认证器全部返回false，则认证失败。

代码路径：

`staging/src/k8s.io/apiserver/pkg/server/config.go:550`

--> `vendor/k8s.io/apiserver/pkg/endpoints/filters/authentication.go:53`

```go
func WithAuthentication(handler http.Handler, auth authenticator.Request, failed http.Handler, apiAuds authenticator.Audiences) http.Handler {
  // 关闭认证则直接进入下一个环节
	if auth == nil {
		klog.Warningf("Authentication is disabled")
		return handler
	}
	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		if len(apiAuds) > 0 {
			req = req.WithContext(authenticator.WithAudiences(req.Context(), apiAuds))
		}
    // 这里进行认证
		resp, ok, err := auth.AuthenticateRequest(req)
		if err != nil || !ok {
			if err != nil {
				klog.Errorf("Unable to authenticate the request due to an error: %v", err)
			}
			failed.ServeHTTP(w, req)
			return
		}

		// TODO(mikedanese): verify the response audience matches one of apiAuds if
		// non-empty

		// authorization header is not required anymore in case of a successful authentication.
		req.Header.Del("Authorization")
		
    // 请求上下文携带上认证后的用户的信息
		req = req.WithContext(genericapirequest.WithUser(req.Context(), resp.User))
    // 计数器
		authenticatedUserCounter.WithLabelValues(compressUsername(resp.User.GetName())).Inc()
		// 处理请求
		handler.ServeHTTP(w, req)
	})
}	
```

`auth.AuthenticateRequest`是一个接口方法，看看它的引用：

`vendor/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go:35`

![](https://mycloudn.upweto.top/20210202115500.png)

引用中的union.go中，聚合了所有的Authenticator链，进去看看

`staging/src/k8s.io/apiserver/pkg/authentication/request/union/union.go:53`

```go
// AuthenticateRequest authenticates the request using a chain of authenticator.Request objects.
func (authHandler *unionAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	var errlist []error
	for _, currAuthRequestHandler := range authHandler.Handlers {
		resp, ok, err := currAuthRequestHandler.AuthenticateRequest(req)
		if err != nil {
			if authHandler.FailOnError {
				return resp, ok, err
			}
			errlist = append(errlist, err)
			continue
		}

		if ok {
			return resp, ok, err
		}
	}

	return nil, false, utilerrors.NewAggregate(errlist)
}

```

apiserver的各种request相关定义代码大多集中在这个目录里：

<img src="https://mycloudn.upweto.top/20210202145632.png" style="zoom:70%;" />

那么依次逐个来看看每一种Authenticator对应的AuthenticateRequest方法。





### RequestHeader认证

是一种代理认证方式，需要再apiserver启动时以参数形式配置，来看看官方的介绍：

API 服务器可以配置成从请求的头部字段值（如 `X-Remote-User`）中辩识用户。 这一设计是用来与某身份认证代理一起使用 API 服务器，代理负责设置请求的头部字段值。

- `--requestheader-username-headers` 必需字段，大小写不敏感。用来设置要获得用户身份所要检查的头部字段名称列表（有序）。第一个包含数值的字段会被用来提取用户名。
- `--requestheader-group-headers` 可选字段，在 Kubernetes 1.6 版本以后支持，大小写不敏感。 建议设置为 "X-Remote-Group"。用来指定一组头部字段名称列表，以供检查用户所属的组名称。 所找到的全部头部字段的取值都会被用作用户组名。
- `--requestheader-extra-headers-prefix` 可选字段，在 Kubernetes 1.6 版本以后支持，大小写不敏感。 建议设置为 "X-Remote-Extra-"。用来设置一个头部字段的前缀字符串，API 服务器会基于所给 前缀来查找与用户有关的一些额外信息。这些额外信息通常用于所配置的鉴权插件。 API 服务器会将与所给前缀匹配的头部字段过滤出来，去掉其前缀部分，将剩余部分 转换为小写字符串并在必要时执行[百分号解码](https://tools.ietf.org/html/rfc3986#section-2.1) 后，构造新的附加信息字段键名。原来的头部字段值直接作为附加信息字段的值。

例如，使用下面的配置：

```
--requestheader-username-headers=X-Remote-User
--requestheader-group-headers=X-Remote-Group
--requestheader-extra-headers-prefix=X-Remote-Extra-
```

针对所收到的如下请求：

```http
GET / HTTP/1.1
X-Remote-User: fido
X-Remote-Group: dogs
X-Remote-Group: dachshunds
X-Remote-Extra-Acme.com%2Fproject: some-project
X-Remote-Extra-Scopes: openid
X-Remote-Extra-Scopes: profile
```

会生成下面的用户信息：

```yaml
name: fido
groups:
- dogs
- dachshunds
extra:
  acme.com/project:
  - some-project
  scopes:
  - openid
  - profile
```

为了防范头部信息侦听，在请求中的头部字段被检视之前， 身份认证代理需要向 API 服务器提供一份合法的客户端证书， 供后者使用所给的 CA 来执行验证。 警告：*不要* 在不同的上下文中复用 CA 证书，除非你清楚这样做的风险是什么以及 应如何保护 CA 用法的机制。

- `--requestheader-client-ca-file` 必需字段，给出 PEM 编码的证书包。 在检查请求的头部字段以提取用户名信息之前，必须提供一个合法的客户端证书， 且该证书要能够被所给文件中的机构所验证。
- `--requestheader-allowed-names` 可选字段，用来给出一组公共名称（CN）。 如果此标志被设置，则在检视请求中的头部以提取用户信息之前，必须提供 包含此列表中所给的 CN 名的、合法的客户端证书。

> 以上摘自官方文档[身份认证代理](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#authenticating-proxy)



简而言之，就是在apiserver之前有一个代理服务器，通过代理服务器的名义，可以将相关的认证信息在请求头透传给apiserver，以通过apiserver的认证。代理服务的相关信息需要预先配置。

使用kubeadm默认部署的集群，就启用了requestheader的配置：

![](https://mycloudn.upweto.top/20210202163330.png)

来看看AuthenticateRequest方法的代码：

`vendor/k8s.io/apiserver/pkg/authentication/request/headerrequest/requestheader.go:108`

```go
func (a *requestHeaderAuthRequestHandler) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
  // 取到用户名
	name := headerValue(req.Header, a.nameHeaders)
	if len(name) == 0 {
		return nil, false, nil
	}
  // 取到用户group
	groups := allHeaderValues(req.Header, a.groupHeaders)
  // 取到其他信息
	extra := newExtra(req.Header, a.extraHeaderPrefixes)

	// clear headers used for authentication
  // 请求的header信息去除
	for _, headerName := range a.nameHeaders {
		req.Header.Del(headerName)
	}
	for _, headerName := range a.groupHeaders {
		req.Header.Del(headerName)
	}
	for k := range extra {
		for _, prefix := range a.extraHeaderPrefixes {
			req.Header.Del(prefix + k)
		}
	}
  // 返回认证成功信息
	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   name,
			Groups: groups,
			Extra:  extra,
		},
	}, true, nil
}	
```

参考代码片中注释



### BasicAuth认证

BasicAuth是一种简单的基础http认证，用户名、密码写入http请求头中，用base64编码，防君子不防小人，安全性较低，因此很少使用。快速略过

启动apiserver时，使用--basic-auth-file参数指定csv文件，csv里面以逗号切割，存放用户名、密码、uid。

看看AuthenticateRequest方法：

`vendor/k8s.io/apiserver/plugin/pkg/authenticator/request/basicauth/basicauth.go:38`

```go
// AuthenticateRequest authenticates the request using the "Authorization: Basic" header in the request
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	username, password, found := req.BasicAuth()
	if !found {
		return nil, false, nil
	}
	// 简单的账号密码比对，没有什么值得深入的
	resp, ok, err := a.auth.AuthenticatePassword(req.Context(), username, password)

	// If the password authenticator didn't error, provide a default error
	if !ok && err == nil {
		err = errInvalidAuth
	}

	return resp, ok, err
}	
```



### x509 CA认证

又称TLS双向认证，APIServer启动时使用--client-ca-file指定客户端的证书文件，用作请求的认证。

`vendor/k8s.io/apiserver/pkg/authentication/request/x509/x509.go:88`

```go
// AuthenticateRequest authenticates the request using presented client certificates
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
	if req.TLS == nil || len(req.TLS.PeerCertificates) == 0 {
		return nil, false, nil
	}

	// Use intermediates, if provided
	optsCopy := a.opts
	if optsCopy.Intermediates == nil && len(req.TLS.PeerCertificates) > 1 {
		optsCopy.Intermediates = x509.NewCertPool()
		for _, intermediate := range req.TLS.PeerCertificates[1:] {
			optsCopy.Intermediates.AddCert(intermediate)
		}
	}

	remaining := req.TLS.PeerCertificates[0].NotAfter.Sub(time.Now())
	clientCertificateExpirationHistogram.Observe(remaining.Seconds())
  // 校验客户端request携带的证书，如果是ca签名的证书，且在有效期内，则视为有效，返回对应的证书chain
	chains, err := req.TLS.PeerCertificates[0].Verify(optsCopy)
	if err != nil {
		return nil, false, err
	}

	var errlist []error
	for _, chain := range chains {
    // 这里再展开一下
		user, ok, err := a.user.User(chain)
		if err != nil {
			errlist = append(errlist, err)
			continue
		}

		if ok {
			return user, ok, err
		}
	}
	return nil, false, utilerrors.NewAggregate(errlist)
}	
```

--> `vendor/k8s.io/apiserver/pkg/authentication/request/x509/x509.go:72`

 --> `vendor/k8s.io/apiserver/pkg/authentication/request/x509/x509.go:185`

```go
// CommonNameUserConversion builds user info from a certificate chain using the subject's CommonName
var CommonNameUserConversion = UserConversionFunc(func(chain []*x509.Certificate) (*authenticator.Response, bool, error) {
	if len(chain[0].Subject.CommonName) == 0 {
		return nil, false, nil
	}
  // 取得证书chain中的CommonName/Organization作为用户和组
	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   chain[0].Subject.CommonName,
			Groups: chain[0].Subject.Organization,
		},
	}, true, nil
})
```



### BearerToken认证

这种认证方式是专为k8s节点准备的，避免每个节点都要手动配置TLS证书，在apiserver启动时指定`--enable-bootstrap-token-auth`参数来启用这种认证方式，详细说明见官方文档：

> [启动引导令牌](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#bootstrap-tokens)



bearertoken的认证方式是在请求头里放入bearer令牌，令牌的格式为 `[a-z0-9]{6}.[a-z0-9]{16}`。第一个部分是令牌的 ID；第二个部分 是令牌的 Secret。对应http请求头的格式是: 

`Authorization: Bearer xxxxxx.xxxxxxxxxxxxxxxx`

代码路径：

`vendor/k8s.io/apiserver/pkg/authentication/request/bearertoken/bearertoken.go:37`

```go
func (a *Authenticator) AuthenticateRequest(req *http.Request) (*authenticator.Response, bool, error) {
  // Authorization header
	auth := strings.TrimSpace(req.Header.Get("Authorization"))
	if auth == "" {
		return nil, false, nil
	}
  // 切割分开，第二部分就是token
	parts := strings.Split(auth, " ")
	if len(parts) < 2 || strings.ToLower(parts[0]) != "bearer" {
		return nil, false, nil
	}

	token := parts[1]

	// Empty bearer tokens aren't valid
	if len(token) == 0 {
		return nil, false, nil
	}
	// 校验token
	resp, ok, err := a.auth.AuthenticateToken(req.Context(), token)
	// if we authenticated successfully, go ahead and remove the bearer token so that no one
	// is ever tempted to use it inside of the API server
	if ok {
		req.Header.Del("Authorization")
	}

	// If the token authenticator didn't error, provide a default error
	if !ok && err == nil {
		err = invalidToken
	}

	return resp, ok, err
}

```

`AuthenticateToken`是一个接口方法：

`vendor/k8s.io/apiserver/pkg/authentication/authenticator/interfaces.go:28`

```go
type Token interface {
	AuthenticateToken(ctx context.Context, token string) (*Response, bool, error)
}
// TokenFunc is a function that implements the Token interface.
type TokenFunc func(ctx context.Context, token string) (*Response, bool, error)

// AuthenticateToken implements authenticator.Token.
func (f TokenFunc) AuthenticateToken(ctx context.Context, token string) (*Response, bool, error) {
	return f(ctx, token)
}
```

接口的实现方法在:

`plugin/pkg/auth/authenticator/token/bootstrap/bootstrap.go:94`

```go
func (t *TokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
  // 把token用'.'切割分为id和密文两段
	tokenID, tokenSecret, err := parseToken(token)
	if err != nil {
		// Token isn't of the correct form, ignore it.
		return nil, false, nil
	}
  // 从APIServer取到对应的secret资源对象，跟这个token进行信息比对
	secretName := bootstrapapi.BootstrapTokenSecretPrefix + tokenID
	secret, err := t.lister.Get(secretName)
	if err != nil {
		if errors.IsNotFound(err) {
			klog.V(3).Infof("No secret of name %s to match bootstrap bearer token", secretName)
			return nil, false, nil
		}
		return nil, false, err
	}

	if secret.DeletionTimestamp != nil {
		tokenErrorf(secret, "is deleted and awaiting removal")
		return nil, false, nil
	}

	if string(secret.Type) != string(bootstrapapi.SecretTypeBootstrapToken) || secret.Data == nil {
		tokenErrorf(secret, "has invalid type, expected %s.", bootstrapapi.SecretTypeBootstrapToken)
		return nil, false, nil
	}

	ts := getSecretString(secret, bootstrapapi.BootstrapTokenSecretKey)
	if subtle.ConstantTimeCompare([]byte(ts), []byte(tokenSecret)) != 1 {
		tokenErrorf(secret, "has invalid value for key %s, expected %s.", bootstrapapi.BootstrapTokenSecretKey, tokenSecret)
		return nil, false, nil
	}

	id := getSecretString(secret, bootstrapapi.BootstrapTokenIDKey)
	if id != tokenID {
		tokenErrorf(secret, "has invalid value for key %s, expected %s.", bootstrapapi.BootstrapTokenIDKey, tokenID)
		return nil, false, nil
	}

	if isSecretExpired(secret) {
		// logging done in isSecretExpired method.
		return nil, false, nil
	}

	if getSecretString(secret, bootstrapapi.BootstrapTokenUsageAuthentication) != "true" {
		tokenErrorf(secret, "not marked %s=true.", bootstrapapi.BootstrapTokenUsageAuthentication)
		return nil, false, nil
	}

	groups, err := getGroups(secret)
	if err != nil {
		tokenErrorf(secret, "has invalid value for key %s: %v.", bootstrapapi.BootstrapTokenExtraGroupsKey, err)
		return nil, false, nil
	}

	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   bootstrapapi.BootstrapUserPrefix + string(id),
			Groups: groups,
		},
	}, true, nil
}
```



### ServiceAccount(SA)认证

启用方式为APIServer命令使用`--service-account-key-file`参数指定一个为token签名的PEM秘钥文件。

SA认证是jwt形式的认证，使用方式与bearer token类似，也是放在请求头里，内容为Base64编码，header格式为:

`Authorization: JWT eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjo0NTk0LCJ1c2VybmFtZSI6InlpbndlbnFpbiIsImV4cCI6MTU3MDY3NDEyNywiZW1haWwiOiIifQ.djC2w5l3IiXYv7slZtGzlMzLc3_oPuR1M0dM9FwoaUU`

token哪里来呢？答案就是ServiceAccount。

SA是一种面向集群内部应用需要调用APIServer的场景所设计的认证方式。在创建ServiceAccount资源时，可以显示地设置标签将ServiceAccount绑定给某Deploy/sts/pod，也可以在Deploy/sts/pod的声明文件里显示指定ServiceAccount。ServiceAccount会自动创建Secret资源，token秘钥存放其中。

在相应的容器层面，token信息会被挂载进容器中，包含3个文件：

- namespace文件：指明命名空间
- ca.crt文件：APIServer的公钥证书，容器用来校验APIServer
- token文件: 存放在Secret里的JWT token

在集群上找一个例子，以kube-proxy举例：

![](https://mycloudn.upweto.top/20210203103806.png)

看看代码层面：

`pkg/serviceaccount/jwt.go:145`

```go
func (j *jwtTokenAuthenticator) AuthenticateToken(ctx context.Context, tokenData string) (*authenticator.Response, bool, error) {
	if !j.hasCorrectIssuer(tokenData) {
		return nil, false, nil
	}
  // 拿到token
	tok, err := jwt.ParseSigned(tokenData)
	if err != nil {
		return nil, false, nil
	}

	public := &jwt.Claims{}
	private := j.validator.NewPrivateClaims()

	var (
		found   bool
		errlist []error
	)

	for _, key := range j.keys {
		if err := tok.Claims(key, public, private); err != nil {
			errlist = append(errlist, err)
			continue
		}
		found = true
		break
	}

	if !found {
		return nil, false, utilerrors.NewAggregate(errlist)
	}

	tokenAudiences := authenticator.Audiences(public.Audience)
	if len(tokenAudiences) == 0 {
		// only apiserver audiences are allowed for legacy tokens
		tokenAudiences = j.implicitAuds
	}

	requestedAudiences, ok := authenticator.AudiencesFrom(ctx)
	if !ok {
		// default to apiserver audiences
		requestedAudiences = j.implicitAuds
	}

	auds := authenticator.Audiences(tokenAudiences).Intersect(requestedAudiences)
	if len(auds) == 0 && len(j.implicitAuds) != 0 {
		return nil, false, fmt.Errorf("token audiences %q is invalid for the target audiences %q", tokenAudiences, requestedAudiences)
	}

	// If we get here, we have a token with a recognized signature and
	// issuer string.
  // 校验token并拿到SA信息
	sa, err := j.validator.Validate(tokenData, public, private)
	if err != nil {
		return nil, false, err
	}

	return &authenticator.Response{
		User:      sa.UserInfo(),
		Audiences: auds,
	}, true, nil
}
```



### WebhookToken认证

Webhook 身份认证是一种用来验证持有者令牌的回调机制。

- `--authentication-token-webhook-config-file` 指向一个配置文件，其中描述 如何访问远程的 Webhook 服务。
- `--authentication-token-webhook-cache-ttl` 用来设定身份认证决定的缓存时间。 默认时长为 2 分钟。

配置文件使用 [kubeconfig](https://kubernetes.io/zh/docs/concepts/configuration/organize-cluster-access-kubeconfig/) 文件的格式。文件中，`clusters` 指代远程服务，`users` 指代远程 API 服务 Webhook。下面是一个例子：

```yaml
# Kubernetes API 版本
apiVersion: v1
# API 对象类别
kind: Config
# clusters 指代远程服务
clusters:
  - name: name-of-remote-authn-service
    cluster:
      certificate-authority: /path/to/ca.pem         # 用来验证远程服务的 CA
      server: https://authn.example.com/authenticate # 要查询的远程服务 URL。必须使用 'https'。

# users 指代 API 服务的 Webhook 配置
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # Webhook 插件要使用的证书
      client-key: /path/to/key.pem          # 与证书匹配的密钥

# kubeconfig 文件需要一个上下文（Context），此上下文用于本 API 服务器
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authn-service
    user: name-of-api-sever
  name: webhook
```

当客户端尝试在 API 服务器上使用持有者令牌完成身份认证（ 如[前](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#putting-a-bearer-token-in-a-request)所述）时， 身份认证 Webhook 会用 POST 请求发送一个 JSON 序列化的对象到远程服务。 该对象是 `authentication.k8s.io/v1beta1` 组的 `TokenReview` 对象， 其中包含持有者令牌。 Kubernetes 不会强制请求提供此 HTTP 头部。

要注意的是，Webhook API 对象和其他 Kubernetes API 对象一样，也要受到同一 [版本兼容规则](https://kubernetes.io/zh/docs/concepts/overview/kubernetes-api/)约束。 实现者要了解对 Beta 阶段对象的兼容性承诺，并检查请求的 `apiVersion` 字段， 以确保数据结构能够正常反序列化解析。此外，API 服务器必须启用 `authentication.k8s.io/v1beta1` API 扩展组 （`--runtime-config=authentication.k8s.io/v1beta1=true`）。

POST 请求的 Body 部分将是如下格式：

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "spec": {
    "token": "<持有者令牌>"
  }
}
```

远程服务应该会填充请求的 `status` 字段，以标明登录操作是否成功。 响应的 Body 中的 `spec` 字段会被忽略，因此可以省略。 如果持有者令牌验证成功，应该返回如下所示的响应：

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "janedoe@example.com",
      "uid": "42",
      "groups": [
        "developers",
        "qa"
      ],
      "extra": {
        "extrafield1": [
          "extravalue1",
          "extravalue2"
        ]
      }
    }
  }
}
```

而不成功的请求会返回：

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": false
  }
}
```

HTTP 状态码可用来提供进一步的错误语境信息。

> 以上摘自官方文档[Webhook 令牌身份认证](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)

代码：

`vendor/k8s.io/apiserver/plugin/pkg/authenticator/token/webhook/webhook.go:77`

```go
// AuthenticateToken implements the authenticator.Token interface.
func (w *WebhookTokenAuthenticator) AuthenticateToken(ctx context.Context, token string) (*authenticator.Response, bool, error) {
	// We take implicit audiences of the API server at WebhookTokenAuthenticator
	// construction time. The outline of how we validate audience here is:
	//
	// * if the ctx is not audience limited, don't do any audience validation.
	// * if ctx is audience-limited, add the audiences to the tokenreview spec
	//   * if the tokenreview returns with audiences in the status that intersect
	//     with the audiences in the ctx, copy into the response and return success
	//   * if the tokenreview returns without an audience in the status, ensure
	//     the ctx audiences intersect with the implicit audiences, and set the
	//     intersection in the response.
	//   * otherwise return unauthenticated.
	wantAuds, checkAuds := authenticator.AudiencesFrom(ctx)
	r := &authentication.TokenReview{
		Spec: authentication.TokenReviewSpec{
			Token:     token,
			Audiences: wantAuds,
		},
	}
	var (
		result *authentication.TokenReview
		err    error
		auds   authenticator.Audiences
	)
	webhook.WithExponentialBackoff(w.initialBackoff, func() error {
    // webhook插件向远端认证服务器发起post请求
		result, err = w.tokenReview.Create(r)
		return err
	})
	if err != nil {
		// An error here indicates bad configuration or an outage. Log for debugging.
		klog.Errorf("Failed to make webhook authenticator request: %v", err)
		return nil, false, err
	}

	if checkAuds {
		gotAuds := w.implicitAuds
		if len(result.Status.Audiences) > 0 {
			gotAuds = result.Status.Audiences
		}
		auds = wantAuds.Intersect(gotAuds)
		if len(auds) == 0 {
			return nil, false, nil
		}
	}

	r.Status = result.Status
	if !r.Status.Authenticated {
		var err error
		if len(r.Status.Error) != 0 {
			err = errors.New(r.Status.Error)
		}
		return nil, false, err
	}
  
	var extra map[string][]string
	if r.Status.User.Extra != nil {
		extra = map[string][]string{}
		for k, v := range r.Status.User.Extra {
			extra[k] = v
		}
	}
  
  // 根据远端认证服务器返回的status字段，填充请求的相关用户信息(name/uid/group等)
	return &authenticator.Response{
		User: &user.DefaultInfo{
			Name:   r.Status.User.Username,
			UID:    r.Status.User.UID,
			Groups: r.Status.User.Groups,
			Extra:  extra,
		},
		Audiences: auds,
	}, true, nil
}
```



### Anonymous认证

启用匿名请求支持之后，如果请求没有被已配置的其他身份认证方法拒绝，则被视作 匿名请求（Anonymous Requests）。这类请求获得用户名 `system:anonymous` 和 对应的用户组 `system:unauthenticated`。

例如，在一个配置了令牌身份认证且启用了匿名访问的服务器上，如果请求提供了非法的 持有者令牌，则会返回 `401 Unauthorized` 错误。 如果请求没有提供持有者令牌，则被视为匿名请求。

在 1.5.1-1.5.x 版本中，匿名访问默认情况下是被禁用的，可以通过为 API 服务器设定 `--anonymous-auth=true` 来启用。

在 1.6 及之后版本中，如果所使用的鉴权模式不是 `AlwaysAllow`，则匿名访问默认是被启用的。 从 1.6 版本开始，ABAC 和 RBAC 鉴权模块要求对 `system:anonymous` 用户或者 `system:unauthenticated` 用户组执行显式的权限判定，所以之前的为 `*` 用户或 `*` 用户组赋予访问权限的策略规则都不再包含匿名用户。

> 以上摘自官方文档[匿名请求](https://kubernetes.io/zh/docs/reference/access-authn-authz/authentication/#anonymous-requests)

代码:

`vendor/k8s.io/apiserver/pkg/authentication/request/anonymous/anonymous.go:32`

```go
func NewAuthenticator() authenticator.Request {
	return authenticator.RequestFunc(func(req *http.Request) (*authenticator.Response, bool, error) {
		auds, _ := authenticator.AudiencesFrom(req.Context())
		return &authenticator.Response{
			User: &user.DefaultInfo{
				Name:   anonymousUser,
				Groups: []string{unauthenticatedGroup},
			},
			Audiences: auds,
		}, true, nil
	})
}
```





## 总结

认证机制有许多种，每种都有不同的适用场景，一般情况下只需要了解认证流程和其中常见的几种认证器即可，不必过分深挖。认证流程通过之后，下一篇进入授权环节