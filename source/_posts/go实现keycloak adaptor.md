---
title: go实现keycloak adaptor
date: 2021-12-09 10:11:41
categories:
    - GO
tags:
    - go
    - 认证
---

## 概述

**IAM：**Identity and Access Management的缩写，即身份识别与访问管理，具有单点登录(SSO-Single Sign On)，认证管理，基于策略的集中式授权和审计，动态授权等功能。

**keycloak：**是IAM的解决方案，用于管理用户的注册、登录、单点登录Single Sign On（SSO，开放app接口的授权等。

## 实现

keycloak并没有提供go的接入适配器，所以需要我们自行实现。接入keycloak有两种常用的认证方式：

1. OpenID Connect（oidc）：在oauth2.0上的一种授权认证协议
2. SAML：认证授权协议

两种方式都可接入，oidc会相对简单一点。（当然，还得看我们iam是否有配置两种协议的endpoint）

我们选择用oidc的认证方式接入IAM。OIDC是OpenID Connect的简称，OIDC=(Identity, Authentication) + OAuth 2.0。它在OAuth2上构建了一个身份层，是一个基于OAuth2协议的身份认证标准协议。具体可以参考文档：https://www.cnblogs.com/linianhui/p/openid-connect-core.html#auto-id-0

go生态有go-oidc和oauth2.0组件，可以帮助我们解决和iam交互工作，我们只需负责提供认证流程需要的接口即可。认证流程图：

TODO

### **oidc和oauth初始化**

```go
type IAM struct {
	oauth2Cfg      *oauth2.Config
	oidcVerifier   *oidc.IDTokenVerifier
	state          string //随机状态 目前是固定的
	feRedirectURL  string //前端回调地址
	serverLoginUrl string //服务端登录接口
}

type Iam struct {
	ClientID            string `protobuf:"bytes,1,opt,name=ClientID,proto3" json:"ClientID,omitempty"`                       // 客户端名称从keycloak 中获取，resource
	ClientSecret        string `protobuf:"bytes,2,opt,name=ClientSecret,proto3" json:"ClientSecret,omitempty"`               // 客户端密钥从keycloak 中获取，credentials.secret
	LogoutCallbackAddr  string `protobuf:"bytes,3,opt,name=LogoutCallbackAddr,proto3" json:"LogoutCallbackAddr,omitempty"`   // IAM登出之后的回调地址
	LoginCallbackAddr   string `protobuf:"bytes,4,opt,name=LoginCallbackAddr,proto3" json:"LoginCallbackAddr,omitempty"`     // IAM登录成功之后 应该回调的前端地址
	FeLoginCallbackAddr string `protobuf:"bytes,5,opt,name=FeLoginCallbackAddr,proto3" json:"FeLoginCallbackAddr,omitempty"` // 登录成功之后，前端回调地址
	AuthServer          string `protobuf:"bytes,6,opt,name=AuthServer,proto3" json:"AuthServer,omitempty"` // IAM授权地址
	ServerLoginUrl      string `protobuf:"bytes,9,opt,name=ServerLoginUrl,proto3" json:"ServerLoginUrl,omitempty"` // 后端登录接口地址
}

func NewIAMClient(params *conf.Iam) (*iamClient, error) {
	ctx := context.Background()
	provider, err := oidc.NewProvider(ctx, params.AuthServer) //获取endpoint信息，但是我们的iam架构之间加了一层， 所以并没有使用这个返回结果
	if err != nil {
		return nil, err
	}
	oauth2Config := &oauth2.Config{
		ClientID:     params.ClientID,
		ClientSecret: params.ClientSecret,
		RedirectURL:  params.LoginCallbackAddr,
		// Discovery returns the OAuth2 endpoints.
		Endpoint: provider.Endpoint(),
		// "openid" is a required scope for OpenID Connect flows.
		Scopes: []string{oidc.ScopeOpenID, "profile", "email"},
	}
	oidcConfig := &oidc.Config{
		ClientID:          params.ClientID,
		SkipClientIDCheck: true,
	}
	verifier := provider.Verifier(oidcConfig)
	return &iamClient{
		oauth2Cfg:      oauth2Config,
		oidcVerifier:   verifier,
		state:          strconv.FormatInt(time.Now().Unix(), 10), //理论上需要随机生成，随机生成 需要用额外的存储 才能校验
		feRedirectURL:  params.FeLoginCallbackAddr,
		serverLoginUrl: params.ServerLoginUrl,
	}, nil
}
```

### 路由接口实现

```go
type UserRepo struct {
	Iam *IAM
}
// LoginUrlHandler 获取iam登录地址接口
func (u *UserRepo) LoginUrlHandler(ctx context.Context) string {
   return u.Iam.oauth2Cfg.AuthCodeURL(u.Iam.state)
}

// LoginCallback 授权码验证接口
func (u *UserRepo) LoginCallback(ctx context.Context, state, code string) {
 	//验证iam返回的state
	if state != u.Iam.state {
		//认证失败 自定义返回错误
    return
	}
	//iam返回的授权码code 去交换token
	oauth2Token, err := u.Iam.oauth2Cfg.Exchange(ctx, code)
	if err != nil {
		//认证失败 自定义返回错误
    return
	}
	//取出access_token 因为iam从oidc退化了 所以 取access_token 正常是取 id_token
  //rawIDToken, ok := oauth2Token.Extra("id_token").(string)
	accessToken, ok := oauth2Token.Extra("access_token").(string)
	if !ok {
		//认证失败 自定义返回错误
    return
	}
	//从access token中解析 用户信息
	idToken, err := u.Iam.oidcVerifier.Verify(ctx, accessToken)
	if err != nil {
		//认证失败 自定义返回错误
    return
	}
  //iam认证成功 可以开始创建本地token
  // ... 自己撸代码
}
```
