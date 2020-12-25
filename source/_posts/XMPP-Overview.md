---
title: XMPP Overview
date: 2019-03-26 09:58:25
categories: 
    - Network
---



跟朋友做一个项目，想快速开发，选了XMPP协议。它是一套通信协议。分为两部分，[XMPP Core Services](https://xmpp.org/rfcs/rfc6121.html#A%20Sample%20Session) 和 XMPP Extension Protocols. 核心由基础feature组成，扩展协议就非常丰富，而且一直在发展。Wiki上有张各种IM协议的汇总表，推荐！

- [Comparison of instant messaging protocols - Wikipedia](https://en.wikipedia.org/wiki/Comparison_of_instant_messaging_protocols)


## XMPP Addressing 

这是一张Client-Server的图，图里的server、client都遵循XMPP协议。叫 XMPP entity. 它们有各自唯一的Address, 格式如'username@server.com', 叫 JID (Jaber ID)
 [RFC 7622 - Extensible Messaging and Presence Protocol (XMPP): Address Format](https://datatracker.ietf.org/doc/rfc7622/)
 
 ![28a215f7.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/28a215f7.png)
 
其中resource是拿来做同一账号多客户端标记的， 比如图中`User1` 从 pc ,phone1 和 phone2登录同一账号，resource分别是 `pc`, `iphone1`,`iphone2`
 
 
 ## XMPP Client- Server Streams
 
 客户端与服务端通过长链接方式通信，现在多用WebSocket。当客户端跟服务端握手成功，它们开始用 XML stream通信。
 
 ![f1565a2e.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/f1565a2e.png)\

 
XML stream 总是以  ``<stream>`` 开头， ``</stream>`` tag结尾。是xml消息的容器。

```
An XML stream is a container for the exchange of XML elements between any two entities over a network. 
During the life of the stream, the entity that initiated it can send an unbounded number of XML elements over the stream, either elements used to negotiate the stream (e.g., to complete TLS negotiation or SASL negotiation) or XML stanzas. 
```

下面是client跟server的一次消息交互， 绿色来自client的，黑色消息来自server

 
  ![f97e583b.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/65d38868.png)

 ### XML stanza
 An XML stanza is the basic unit of meaning in XMPP. A stanza is a first-level element (at depth=1 of the stream) whose element name is "message", "presence", or "iq" and whose qualifying namespace is 'jabber:client' or 'jabber:server'. 
 
 
 ### XMPP Communication Primitives

A `stanza` is the smallest piece of XML data a client can send to a server ( server send to client) in one package.

xmpp中，服务端、客户数据交换时，最小XML数据单位 叫 stanza。如上图，绿色的就是一个stanza，黑色的也是一个stanza。Stanza有几种类型: `message`, `iq`, `presence`。 

#### The Message Stanza

The <message/> stanza is meant to be used to send data between XMPP entities.

![6fe8a15e.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/6fe8a15e.png)

 - from：发送方
 - to： 接收方
 - body: 消息内容
 - type 有几种类型:
     -`<message type=”chat”/>` ( chat message stanza) 
     - `< message type=”groupchat”/>` ( groupchat message stanza)
     - `< message type=”error”/>` (error message stanza)

#### The Presence Stanza

用来表示在线状态的
 

![0fbe995b.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/0fbe995b.png)

`show` 标签里可能会有的几种状态: 
`chat` : online and available for chat ; 
`away` : 暂时离开
`xa` : 长时间离开
`dnd`: 请勿打扰

如果你想知道别的状态，需要先发消息给Server，subscribe别人。 


#### The IQ stanza
 
 The IQ( Info/Query) stanza is used to get some information from the server ( info about the server or its registered clients) or to apply some settings to the server.
 
 用来获取消息，或者请求设置
  
Type属性中的类型 :get ,set ,result or error. 
- `< iq type=”get”/>` stanzas are used to get(ask) some information ( from the server). 
- `<iq type=”set”/>` stanzas are used to apply some settings to the server.When you send get/set IQ stanzas to the server ,
- it can reply either with an `< iq type=”result”/>` stanza when your request has been successfully processed by the server or 
- `<iq type=”error”/>` stanza when something has gone wrong with your request.The figure below shows an IQ stanza that we send to the server and the reply we get from the server.


![30c96f66.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/30c96f66.png)


The client sends an IQ get stanza to the server to request its contact list.We know it is asking for the contact list because of the `jabber:iq:roster` XML namespace.

The XMPP engine in the server is programmed to know that when a client sends `jabber:iq:roster` namespaced IQ ,it wants to retrieve its contact list.There are other `namespaces` in XMPP for other uses and you will surely come accross them in your XMPPing journey.

The server responds with a list of the JID’s contacts wraped within a `jabber:iq:roster` namespaced `<query/>`tag.


## 本地搭建 Server 

我搭的是ejabberd. 官方安装教程: [Installing ejabberd \| ejabberd Docs](https://docs.ejabberd.im/admin/installation/#install-on-macos)

#### 启动服务

```
cd /Applications/ejabberd-19.02
//开启服务
./bin/ejabberdctl start  
//状态
./bin/ejabberdctl status  

// help 查看更多功能哦
./bin/ejabberdctl help 
```

#### 注册账户

打开 [admin 页面](http://localhost:5280/admin/), 虚拟主机 -> localhost(可能你的名字不一样) -> 用户。 现在你可以自己创建账户了。

![578b88b6.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/578b88b6.png)


如果有自定义需求,配置教程 [Configuring ejabberd \| ejabberd Docs](https://docs.ejabberd.im/admin/configuration/#mod-http-ws) 
 
#### 客户端玩起来

客户端有很多[选择](https://xmpp.org/software/clients.html)，不过大多数都是渣。如果是WebSocket，用这个 [GitHub - processone/xmpp-websocket-client: Test XMPP Websocket client](https://github.com/processone/xmpp-websocket-client) 调试可以看到stanza，挺方便的。

如果Mac用户报auth问题，可以打开`vim conf/ejabberd.yml`, `tls`配置成`false`
![5202ee46.png](/img/32c16f22-9862-45e8-b15f-1b1eceb7b30f/5202ee46.png)

#### 关于js lib
打算用React Native写，lib选了 [GitHub - xmppjs/xmpp.js: XMPP for JavaScript](https://github.com/xmppjs/xmpp.js) 。当然 Web多用框架 Strophe.js。这儿有个简单比较[How do you compare to strophe.js · Issue #217 · xmppjs/xmpp.js · GitHub](https://github.com/xmppjs/xmpp.js/issues/217)

### 其他资料

- 简单介绍 [A friendly introduction to XMPP – blikoon](https://www.blikoontech.com/xmpp/xmpp-a-soft-friendly-introduction)

- 官方协议很详细，例子也很形象。 [Extensible Messaging and Presence Protocol (XMPP): Core](https://xmpp.org/rfcs/rfc6120.html#tls)

- 如何选择即时通讯应用的数据传输格式 [如何选择即时通讯应用的数据传输格式-其它分享/专项技术区 - 即时通讯开发者社区!](http://www.52im.net/thread-276-1-1.html)
- 强列建议将Protobuf作为你的即时通讯应用数据传输格式 [强列建议将Protobuf作为你的即时通讯应用数据传输格式-其它分享/专项技术区 - 即时通讯开发者社区!](http://www.52im.net/thread-277-1-1.html) 




