---
title: 'XMPP(5): 消息'
date: 2019-04-09 15:16:35
categories: 
    - XMPP
tags:
    - XMPP 
---

 
## Message消息体构造

属性： 
1. to ：接收方地址， JID 
2. from ： 发送方， JID
3. type 
  - chat: 一对一聊天
  - error: 出错
  - groupchat: 群聊
  - headline: 通知、临时消息这种不需要回复的系统消息
  - normal: 之前没有聊天的记录， 客户端可以回复的消息

子元素
1. body: 消息内容

```xml
<message
    from='juliet@example.com/balcony'
    id='b4vs9km4'
    to='romeo@example.net'
    type='chat'
    xml:lang='en'>
  <body>Wherefore art thou, Romeo?</body>
</message>

```
2. Subject: 聊天的话题

```xml

<message
    from='juliet@example.com/balcony'
    id='c8xg3nf8'
    to='romeo@example.net'
    type='chat'
    xml:lang='en'>
  <subject>I implore you!</subject>
  <body>Wherefore art thou, Romeo?</body>
</message>
```

3. Thread: 聊天会话的唯一标识

## Example 

对话： 

```xml
CC: <message
        from='juliet@example.com/balcony'
        to='romeo@example.net'
        type='chat'
        xml:lang='en'>
      <body>My ears have not yet drunk a hundred words</body>
      <thread>e0ffe42b28561960c6b12b944a092794b9683a38</thread>
    </message>

CC: <message
        from='juliet@example.com/balcony'
        to='romeo@example.net'
        type='chat'
        xml:lang='en'>
      <body>Of that tongue's utterance, yet I know the sound:</body>
      <thread>e0ffe42b28561960c6b12b944a092794b9683a38</thread>
    </message>

CC: <message
        from='juliet@example.com/balcony'
        to='romeo@example.net'
        type='chat'
        xml:lang='en'>
      <body>Art thou not Romeo, and a Montague?</body>
      <thread>e0ffe42b28561960c6b12b944a092794b9683a38</thread>
    </message>

UC: <message
        from='romeo@example.net/orchard'
        to='juliet@example.com/balcony'
        type='chat'
        xml:lang='en'>
      <body>Neither, fair saint, if either thee dislike.</body>
      <thread>e0ffe42b28561960c6b12b944a092794b9683a38</thread>
    </message>

CC: <message
        from='juliet@example.com/balcony'
        to='romeo@example.net/orchard'
        type='chat'
        xml:lang='en'>
      <body>How cam'st thou hither, tell me, and wherefore?</body>
      <thread>e0ffe42b28561960c6b12b944a092794b9683a38</thread>
    </message>

```

在[xmpp.js](https://github.com/xmppjs/xmpp.js/)中，客户端与服务端建立了WebSocket长链接后，发消息，需要自己构造消息体

``` js 
const {client, xml} = require('@xmpp/client')

const xmpp = client({
  service: 'ws://localhost:5280/xmpp-websocket',
  domain: 'localhost',
  resource: 'example',
  username: 'username',
  password: 'password',
})

 const message = xml(
    'message',
    {type: 'chat', to: address},
    xml('body', 'hello world')
  )
  await xmpp.send(message)

```

如果收到消息会走到一个回调里, chat-sdk就可以根据type字段来分发。


```js 
self.xmppClient.on('stanza', function (stanza: any) {
    Utils.DLog('[Chat] RECV:', stanza.toString());
    /**
     * Detect typeof incoming stanza
     * and fire the Listener
     */
    if (stanza.is('presence')) {
        self._onPresence(stanza);
    } else if (stanza.is('iq')) {
        self._onIQ(stanza);
    } else if (stanza.is('message')) {
        if (stanza.attrs.type === 'headline') {
            self._onSystemMessageListener(stanza);
        } else if (stanza.attrs.type === 'error') {
            self._onMessageErrorListener(stanza);
        } else {
            self._onMessage(stanza);
        }
    }
});
```

- ref: https://xmpp.org/rfcs/rfc6121.html#message