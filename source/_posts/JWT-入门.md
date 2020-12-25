---
title: JWT 入门
date: 2019-04-03 10:40:16
categories: 
    - Network
tags:
    - Auth
---


## 什么是JSON Web Tokens (JWT)？ 


```
  JSON Web Token (JWT) is a compact, URL-safe means of representing
   claims to be transferred between two parties.  The claims in a JWT
   are encoded as a JSON object that is used as the payload of a JSON
   Web Signature (JWS) structure or as the plaintext of a JSON Web
   Encryption (JWE) structure, enabling the claims to be digitally
   signed or integrity protected with a Message Authentication Code
   (MAC) and/or encrypted.
   

```

## 怎么用？ 

authentication时，当user成功登录，server生成access token, 发送给user；user请求server时带上JWT，server通过JWT验证是否是可信任的客户端请求了。


![1*SSXUQJ1dWjiUrDoKaaiGLA.png](https://cdn-images-1.medium.com/max/1600/1*SSXUQJ1dWjiUrDoKaaiGLA.png)

## 结构

在客户端看来JWT是一串encode加密过的字符串,`header.payload.signature`，如下图左边。但它decode后其实是下图右边的JSON结构体

![legacy-app-auth-5.png](https://cdn.auth0.com/blog/legacy-app-auth/legacy-app-auth-5.png)

#### 1. 生成header

e.g.
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

这里，alg的值指定用HMAC-SHA256算法签名

#### 2. 生成payload

包含用户相关的信息
```
The second part of the token is the payload, which contains the claims. 
Claims are statements about an entity (typically, the user) and additional data. 
```
有三种[claims](https://tools.ietf.org/html/rfc7519#section-4.1): registered, public, and private claims.

e.g.
```json

{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022
}
```

#### 3.生成signature

```js

HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  your-256-bit-secret
) 
```
把header跟payload encode结构后，用'.'连接，生成: <span style="color:#fb015b"> eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9</span><span>.</span>
<span style="color:#d63aff"> eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ</span>

再用指定的hash算法(例子是HS256),用私钥（服务端的）生成签名:<span style="color:#00b9f1">SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c<span>


## 验证

如图1， JWT由Authentication server生成， 在client认证后发给client； client请求application server的时候带上JWT，application server在认证阶段从Authentiation server那儿拿到scret key；用同样算法生成signature， 跟client发来的JWT的signature做比较，看是否match。












[5 Easy Steps to Understanding JSON Web Tokens (JWT)](https://medium.com/vandium-software/5-easy-steps-to-understanding-json-web-tokens-jwt-1164c0adfcec)
[JSON Web Token Introduction - jwt.io](https://jwt.io/introduction/) 
[RFC 7519 - JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)