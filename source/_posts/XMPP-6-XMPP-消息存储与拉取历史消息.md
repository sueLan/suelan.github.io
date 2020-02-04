---
title: 'XMPP(6):XMPP-消息存储与拉取历史消息'
date: 2019-04-10 22:09:53
categories: 
    - XMPP
tags:
    - XMPP
---


XEP-0313定义了XMPP消息存储的规则。

### 场景需求
0313协议主要有这些场景： 
- 同账号多客户端之间的历史消息同步
- 客户端拉取历史消息，按日期排序展示（想想我们在微信的历史消息）
- 分页拉取消息

### 存储

1. 单条消息存储包括： 
- 消息发送跟接收的时间戳
- from 跟 to 的JID
- server-assigned UID
- message stanza 

2. 消息的顺序要保存： 依赖timestamp要小心，因为多条消息可能共享时间戳
3. 超过一定数量，可删除旧信息
4. 群聊记录用MAM服务
5. archive id ` <stanza-id/>`
被archived过的消息，server要给它加上stanza-id
Example 1. Client receives a message that has been archived

```xml
<message to='juliet@capulet.lit/balcony'
         from='romeo@montague.lit/orchard'
         type='chat'>
  <body>Call me but love, and I'll be new baptized; Henceforth I never will be Romeo.</body>
  <stanza-id xmlns='urn:xmpp:sid:0' by='juliet@capulet.lit' id='28482-98726-73623' />
</message>

```
stanza-id: archive ID 

### 查询

#### 1. A user queries their archive for messages
用消息UID查询

'urn:xmpp:mam:2' namespace, indicating the UID of the first and last message of the (possibly limited) result set. 

```xml
<iq type='set' id='juliet1'>
  <query xmlns='urn:xmpp:mam:2' queryid='f27' />
</iq>
```

#### 2. Their server sends the matching messages



```xml
<message id='aeb213' to='juliet@capulet.lit/chamber'>
  <result xmlns='urn:xmpp:mam:2' queryid='f27' id='28482-98726-73623'>
    <forwarded xmlns='urn:xmpp:forward:0'>
      <delay xmlns='urn:xmpp:delay' stamp='2010-07-10T23:08:25Z'/>
      <message xmlns='jabber:client' from="witch@shakespeare.lit" to="macbeth@shakespeare.lit">
        <body>Hail to thee</body>
      </message>
    </forwarded>
  </result>
</message>

```

#### 3. Server returns the result IQ to signal the end


```xml
<iq type='result' id='juliet1'>
  <fin xmlns='urn:xmpp:mam:2'>
    <set xmlns='http://jabber.org/protocol/rsm'>
      <first index='0'>28482-98726-73623</first>
      <last>09af3-cc343-b409f</last>
    </set>
  </fin>
</iq>
```

server的这条iq stanza标记查询结果结束。

### 过滤器

#### 1. 根据JID过滤

`with` 字段 + JID(Bare JID)： 会拿到to或from地址匹配JID的信息; 如果没有with, 服务端返回query指定的时间段内的消息。 

```xml 
Example 6. Querying for all messages to/from a particular JID¶
<iq type='set' id='juliet1'>
  <query xmlns='urn:xmpp:mam:2'>
    <x xmlns='jabber:x:data' type='submit'>
      <field var='FORM_TYPE' type='hidden'>
        <value>urn:xmpp:mam:2</value>
      </field>
      <field var='with'>
        <value>juliet@capulet.lit</value>
      </field>
    </x>
  </query>
</iq>
```

**使用场景：**
A想查询跟B的聊天记录，with字段的value设为B, 服务端返回的messages中，既有B发送给A的msg，也有A发送给B的msg。 

![3c4760c7.png](/img/f3eaaed1-370f-4abc-93b2-a3312d3ebcd4/3c4760c7.png)
#### 2. 根据接收时间过滤

`start` 跟 `end` 字段标记时间戳。 时间戳格式见https://xmpp.org/extensions/xep-0082.html

```xml
Example 7. Querying the archive for all messages in a certain timespan¶
<iq type='set' id='juliet1'>
  <query xmlns='urn:xmpp:mam:2'>
    <x xmlns='jabber:x:data' type='submit'>
      <field var='FORM_TYPE' type='hidden'>
        <value>urn:xmpp:mam:2</value>
      </field>
      <field var='start'>
         // UTC格式
        <value>2010-06-07T00:00:00Z</value>
      </field>
      <field var='end'>
        <value>2010-07-07T13:23:54Z</value>
      </field>
    </x>
  </query>
</iq>
```
如果`end` 缺失， server会自动认为是最近的消息的存储时间

``` xml
Example 8. Querying the archive for all messages after a certain time¶
<iq type='set' id='juliet1'>
  <query xmlns='urn:xmpp:mam:2'>
    <x xmlns='jabber:x:data' type='submit'>
      <field var='FORM_TYPE' type='hidden'>
        <value>urn:xmpp:mam:2</value>
      </field>
      <field var='start'>
        <value>2010-08-07T00:00:00Z</value>
      </field>
    </x>
  </query>
</iq>   
```

#### 3. 限定results的数量

[Result Set Management (XEP-0059)](https://xmpp.org/extensions/xep-0059.html) 

```xml
Example 9. A query using Result Set Management¶
<iq type='set' id='q29302'>
  <query xmlns='urn:xmpp:mam:2'>
    <x xmlns='jabber:x:data' type='submit'>
      <field var='FORM_TYPE' type='hidden'>
        <value>urn:xmpp:mam:2</value>
      </field>
      <field var='start'>
        <value>2010-08-07T00:00:00Z</value>
      </field>
    </x>
    <set xmlns='http://jabber.org/protocol/rsm'>
      <max>10</max>
    </set>
  </query>
</iq>
```

这个请求，指定客户端最多只能收到10条stanzas。但服务端的返回结果可能回改变`set`的内容，返回自己限定的数量，比如： 这是返回`start`时间跟`end`时间段内的20条消息。

```xml
Example 10. Server responds to client with limited results using RSM¶
<!-- result messages -->
<iq type='result' id='q29302'>
  <fin xmlns='urn:xmpp:mam:2'>
    <set xmlns='http://jabber.org/protocol/rsm'>
      <first index='0'>28482-98726-73623</first>
      <last>09af3-cc343-b409f</last>
      <count>20</count>
    </set>
  </fin>
</iq>
```

#### 4. 分页拉取消息

如果之前已经获取了m条消息，客户端可以再发送同样的请求，拉取下一页消息。`set`中要带上`after`(上次拉取到的最后一条消息的UID)

```xml
Example 11. A page query using Result Set Management¶
<iq type='set' id='q29303'>
  <query xmlns='urn:xmpp:mam:2'>
      <x xmlns='jabber:x:data' type='submit'>
        <field var='FORM_TYPE' type='hidden'><value>urn:xmpp:mam:2</value></field>
        <field var='start'><value>2010-08-07T00:00:00Z</value></field>
      </x>
      <set xmlns='http://jabber.org/protocol/rsm'>
         <max>10</max>
         <after>09af3-cc343-b409f</after>
      </set>
  </query>
</iq>
```

server返回最后一页消息，会在 fin里头带上`complete`属性，值为`ture`

```xml
Example 12. Server completes a result with the last page of messages¶
<!-- result messages -->
<iq type='result' id='u29303'>
  <fin xmlns='urn:xmpp:mam:2' complete='true'>
    <set xmlns='http://jabber.org/protocol/rsm'>
      <first index='0'>23452-4534-1</first>
      <last>390-2342-22</last>
      <count>16</count>
    </set>
  </fin>
</iq>
    
```

**使用场景：**

A客户端本地存储跟B的聊天信息， 最后一条message的id是`09af3-cc343-b409f`。 现在A想看看最近的消息（`09af3-cc343-b409f`后的message，可以发送iq请求中带上`<after>09af3-cc343-b409f</after>`。差量请求最新消息，基于游标的分页。
#### 5.其他字段的筛选

客户端查询服务端支持的其他字段
```xml
Example 13. Client requests supported query fields¶
<iq type='get' id='form1'>
  <query xmlns='urn:xmpp:mam:2'/>
</iq>
```

```xml
Example 14. Server returns supported fields¶
<iq type='result' id='form1'>
  <query xmlns='urn:xmpp:mam:2'>
    <x xmlns='jabber:x:data' type='form'>
      <field type='hidden' var='FORM_TYPE'>
        <value>urn:xmpp:mam:2</value>
      </field>
      <field type='jid-single' var='with'/>
      // 按消息received时间查询
      <field type='text-single' var='start'/>
      <field type='text-single' var='end'/>
      // 按文本查询
      <field type='text-single' var='urn:example:xmpp:free-text-search'/>
      // stanza内容
      <field type='text-single' var='urn:example:xmpp:stanza-content'/>
    </x>
  </query>
</iq>
```
### 返回的message stanza 结构

- `message`被封装在`forwarded`元素中。 [XEP-0297: Stanza Forwarding](https://xmpp.org/extensions/xep-0297.html)
- 带`result` 元素， 其属性id是这条message的UID
- delay元素 [XEP-0203: Delayed Delivery](https://xmpp.org/extensions/xep-0203.html) message被收到的时间, UTC时间戳格式


```xml
Example 16. Server returns two matching messages¶
<message id='aeb213' to='juliet@capulet.lit/chamber'>
  <result xmlns='urn:xmpp:mam:2' queryid='f27' id='28482-98726-73623'>
    <forwarded xmlns='urn:xmpp:forward:0'>
      <delay xmlns='urn:xmpp:delay' stamp='2010-07-10T23:08:25Z'/>
      <message xmlns='jabber:client'
        to='juliet@capulet.lit/balcony'
        from='romeo@montague.lit/orchard'
        type='chat'>
        <body>Call me but love, and I'll be new baptized; Henceforth I never will be Romeo.</body>
      </message>
    </forwarded>
  </result>
</message>

<message id='aeb214' to='juliet@capulet.lit/chamber'>
  <result xmlns='urn:xmpp:mam:2' queryid='f27' id='5d398-28273-f7382'>
    <forwarded xmlns='urn:xmpp:forward:0'>
      <delay xmlns='urn:xmpp:delay' stamp='2010-07-10T23:09:32Z'/>
      <message xmlns='jabber:client'
         to='romeo@montague.lit/orchard'
         from='juliet@capulet.lit/balcony'
         type='chat' id='8a54s'>
        <body>What man art thou that thus bescreen'd in night so stumblest on my counsel?</body>
      </message>
    </forwarded>
  </result>
</message>
    
```

## MUC Archive

- 存储所有发送给roomJid的message
- 不包含`private message`
- user需要权限查询群历史聊天记录
- `forward` stanza中带有`to`属性,值是roomJid，`from`值是userJid 
- `x`里有该消息的发送者Jid
```xml
Example 17. Server returns MUC messages¶
<message id='iasd207' from='coven@chat.shakespeare.lit' to='hag66@shakespeare.lit/pda'>
  <result xmlns='urn:xmpp:mam:2' queryid='g27' id='34482-21985-73620'>
    <forwarded xmlns='urn:xmpp:forward:0'>
      <delay xmlns='urn:xmpp:delay' stamp='2002-10-13T23:58:37Z'/>
      <message xmlns="jabber:client"
        from='coven@chat.shakespeare.lit/firstwitch'
        id='162BEBB1-F6DB-4D9A-9BD8-CFDCC801A0B2'
        type='groupchat'>
        <body>Thrice the brinded cat hath mew'd.</body>
        <x xmlns='http://jabber.org/protocol/muc#user'>
          <item affiliation='none'
                jid='witch1@shakespeare.lit'
                role='participant' />
        </x>
      </message>
    </forwarded>
  </result>
</message>

<message id='iasd207' from='coven@chat.shakespeare.lit' to='hag66@shakespeare.lit/pda'>
  <result xmlns='urn:xmpp:mam:2' queryid='g27' id='34482-21985-73620'>
    <forwarded xmlns='urn:xmpp:forward:0'>
      <delay xmlns='urn:xmpp:delay' stamp='2002-10-13T23:58:43Z'/>
      <message xmlns="jabber:client"
        from='coven@chat.shakespeare.lit/secondwitch'
        id='90057840-30FD-4141-AA44-103EEDF218FC'
        type='groupchat'>
        <body>Thrice and once the hedge-pig whined.</body>
        <x xmlns='http://jabber.org/protocol/muc#user'>
          <item affiliation='none'
                jid='witch2@shakespeare.lit'
                role='participant' />
        </x>
      </message>
    </forwarded>
  </result>
</message>
```
[XEP-0313: Message Archive Management](https://xmpp.org/extensions/xep-0313.html#intro)