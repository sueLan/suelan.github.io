---
title: XMPP(3):Roster&联系人
date: 2019-03-31 11:49:32
categories: 
    - XMPP
tags: 
    - XMPP
---




XMPP中联系人模块协议是`jabber:iq:roster`. Roster直接翻译叫花名册，其实它就是联系人列表啦。

## 客户端获取联系人列表

比较简单，发送IQ stanza给server. xmlns=`jabber:iq:roster`;type='get'

```xml

<iq from='user@server.com/balcony'
       id='bv1bs71f'
       type='get'>
    <query xmlns='jabber:iq:roster'/>
  </iq>

```
返回结果的item中有联系人Jid

```xml
<iq id='bv1bs71f'
       to='user@server.com/balcony'
       type='result'>
    <query xmlns='jabber:iq:roster' ver='ver7'>
      <item jid='contact1@server.com'/>
      <item jid='contact2@server.com'/>
    </query>
  </iq>

```

## 添加联系人(加好友）的流程 

方法有两种，第一种用IQ set, 见[rfc6121](https://xmpp.org/rfcs/rfc6121.html#roster-add).

1. 客户端请求添加联系人

xmlns用`jabber:iq:roster`; 带上想添加的用户jid. name可以不带; `group`分组用。


```xml
<iq from='user@server.com/balcony' type='set' id='roster_2'>
  <query xmlns='jabber:iq:roster'>
    <item jid='contact@server.com'
          name='contact'>
      <group>Servants</group>
    </item>
  </query>
</iq>
```

2.1. server通知同一个账户关联的所有客户端: 联系人列表更新了。

```xml

<iq to='user@server.com/balcony'
    type='set'
    id='a78b4q6ha463'>
  <query xmlns='jabber:iq:roster'>
    <item jid='contact@server.com'
          name='contact'
          subscription='none'>
      <group>Servants</group>
    </item>
  </query>
</iq>

<iq to='user@server.com/chamber'
    type='set'
    id='a78b4q6ha464'>
  <query xmlns='jabber:iq:roster'>
    <item jid='contact@server.com'
          name='contact'
          subscription='none'>
      <group>Servants</group>
    </item>
  </query>
</iq>
```

server回复IQ stanza给请求添加联系人的客户端balcony
```xml
<iq to='user@server.com/balcony' type='result' id='roster_2'/>
```


##  删除联系人

给server发送个IQ set， subscription一定是'remove'.

```xml

<iq from='user@server.com/balcony' type='set' id='roster_4'>
  <query xmlns='jabber:iq:roster'>
    <item jid='contact@server.com' subscription='remove'/>
  </query>
</iq>

```

## Presence

增删联系人的另一种方法是Presence订阅机制.Presence stanza其实有两种功能：
- 广播online/offline状态, [之前文章](https://suelan.github.io/2019/03/26/XMPP-Overview/#The-Presence-Stanza)提过
- 控制联系人订阅. 就是增删好友功能咯

我们用type来区分这两种功能。type是`available| unavailable`， presence stanza表达online/offline状态。type若是`subscribe | subscribed | unsubscribe| unsubscribed`，就跟联系人有关啦。


subscribtion有四种状态：
- NONE :  
- TO  :  user订阅contact的状态
- FROM : contact被user订阅
- BOTH : user跟contact相互subcribe

![flow](https://www.blikoontech.com/wp-content/uploads/2018/03/XMPP_Subscription_Flow.png)

如上图：一开始user跟contact没啥关系，subscription状态都是none。 接着user发送了一条Presence stanza给contact，想subscribe他的状态。如下：
```xml
// from user
<presence to='contact@server.com' type='subscribe'/>
```
现在user用`jabber:iq:roster` 查询所有联系人的时候，会发现item多了一条, contact还没确认, 所以 ask='subscribe', subscribtion='none'

```xml
// user's roster
<item ask='subscribe' subscription='none' jid='contact@server.com'/>
```
server要将消息转发给contact客户端, contact登录时，会收到一条来自user的presence stanza; type是'subscribe'。 我们可以用这条消息来做“收到来自user添加好友的请求”这样的功能
```xml
<presence from='user@server.com' to='contact@server.com' type='subscribe' xmlns='jabber:client'></presence>
```

同时contact/dev设备会收到Roster更新的信息. 
```xml
<iq  from='contact@server.com' to='contact@server.com/dev' id='13a99ca5' type='result' xmlns='jabber:client'>
    <query  xmlns='jabber:iq:roster'>
         <item  ask='subscribe' subscription='none' jid='user@server.com'/>
       </query>
</iq>
```
#### 接受请求
如果contact接受请求，他要发送一条presence给user. type值是'subscribed'

```xml
<presence to='user@server.com' type='subscribed'/>
```

user这边的roster会更新
```xml
// user's roster
<item subscription='to' jid='contact@server.com'/>
```
这时在contact的roster列表里，user的subscription是from。 ```xml
// contact's roster
<item ask='subscribe' subscription='from' jid='user@server.com'/>
```

接着contact也请求订阅user 

```xml
<iq from='user@server.com/balcony' type='set' id='roster_2'>
  <query xmlns='jabber:iq:roster'>
    <item jid='contact@server.com'
          name='contact'>
      <group>Servants</group>
    </item>
  </query>
</iq>
```

Contact同样流程后，他两的subscription都变成了both。

#### 拒绝
如果contact想拒绝user的请求，也是发送presence 
```xml
<presence to='user@server.com' type='unsubscribed'/>
```
如果user想取消对contact的订阅, 发送presence stanza，type 是unsubscribed
```xml
<presence to='contact@server.com' type='unsubscribed'/>
```


ref: https://xmpp.org/rfcs/rfc3921.html#roster