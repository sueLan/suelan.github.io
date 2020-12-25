---
title: 'XMPP(2):注册账户'
date: 2019-03-29 10:05:17
categories: 
    - XMPP
---



## XMPP注册流程


#### 1. client发送消息体, 去服务端查询注册需要的字段


```xml
<iq type='get' id='reg1' to='localhost'>
  <query xmlns='jabber:iq:register'/>
</iq>
```

xmlns是 `jabber:iq:register`, type是`get`

#### 2.1. 未注册：返回注册需要的字段

```xml
<iq type='result' id='reg1'>
  <query xmlns='jabber:iq:register'>
    <instructions>
      Choose a username and password for use with this service.
      Please also provide your email address.
    </instructions>
    <username/>
    <password/>
    <email/>
  </query>
</iq>
```

`<instructions/>` element：SHOULD contain an <instructions/> element (whose XML character data MAY be modified to reflect the fact that the entity is currently registered)

#### 2.2. 已注册：服务端的返回结果

```xml
<iq  xmlns='jabber:client' xml:lang='en' to='olivia@localhost/180244803852118156522754' from='localhost' type='result' id='reg1'>
    <query  xmlns='jabber:iq:register'>
        <username>olivia</username>
        <registered/>
        <password/>
        <instructions>Choose a username and password to register with this server</instructions>
    </query>
</iq>
```

host会根据"from"的地址判断entity是否已经注册了，IQ result消息有一个空的`<registered/>`， 标示该entiry已经注册过了。

#### 3.client 注册 

iq stanza的type是`set`, xmlns`jabber:iq:register`

```xml
<iq type='set' id='reg2'>
  <query xmlns='jabber:iq:register'>
    <username>bill</username>
    <password>Calliope</password>
    <email>bard@shakespeare.lit</email>
  </query>
</iq>
```

#### 4.1 注册成功 

```xml
<iq type='result' id='reg2'/>

```

#### 4.2 注册失败，命名冲突

```xml
<iq type='error' id='reg2'>
  <query xmlns='jabber:iq:register'>
    <username>bill</username>
    <password>m1cro$oft</password>
    <email>billg@bigcompany.com</email>
  </query>
  <error code='409' type='cancel'>
    <conflict xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>
```

#### 4.3 消息不全 ` <not-acceptable/> `

```xml
<iq type='error' id='reg2'>
  <query xmlns='jabber:iq:register'>
    <username>bill</username>
    <password>Calliope</password>
  </query>
  <error code='406' type='modify'>
    <not-acceptable xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>
```

#### 4.4 服务端访问权限问题

```xml
<iq  xmlns='jabber:client' xml:lang='en' to='olivia@localhost/180244803852118156522754' from='olivia@localhost' type='error' id='reg2'>
    <query  xmlns='jabber:iq:register'>
        <email>bard@shakespeare.lit</email>
        <username>bill</username>
        <password>Calliope</password>
    </query>
    <error  code='403' type='auth'>
        <forbidden  xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
        <text  xmlns='urn:ietf:params:xml:ns:xmpp-stanzas' xml:lang='en'>Access denied by service policy</text>
    </error>
</iq>
```

#### 5.如果用第三方注册的方式，可能需要补充一些额外的信息

客户端查询

```xml
<iq type='get'
    from='juliet@capulet.com/balcony'
    to='contests.shakespeare.lit'
    id='reg3'>
  <query xmlns='jabber:iq:register'/>
</iq>
```

#### 6.服务端返回消息， 提示需要提供的信息

```xml
<iq type='result'
    from='contests.shakespeare.lit'
    to='juliet@capulet.com/balcony'
    id='reg3'>
  <query xmlns='jabber:iq:register'>
    <instructions>
      Use the enclosed form to register. If your Jabber client does not
      support Data Forms, visit http://www.shakespeare.lit/contests.php
    </instructions>
    <x xmlns='jabber:x:data' type='form'>
      <title>Contest Registration</title>
      <instructions>
        Please provide the following information
        to sign up for our special contests!
      </instructions>
      <field type='hidden' var='FORM_TYPE'>
        <value>jabber:iq:register</value>
      </field>
      <field type='text-single' label='Given Name' var='first'>
        <required/>
      </field>
      <field type='text-single' label='Family Name' var='last'>
        <required/>
      </field>
      <field type='text-single' label='Email Address' var='email'>
        <required/>
      </field>
      <field type='list-single' label='Gender' var='x-gender'>
        <option label='Male'><value>M</value></option>
        <option label='Female'><value>F</value></option>
      </field>
    </x>
  </query>
</iq>
```

#### 7.客户端提供信息

```xml
<iq type='set'
    from='juliet@capulet.com/balcony'
    to='contests.shakespeare.lit'
    id='reg4'>
  <query xmlns='jabber:iq:register'>
    <x xmlns='jabber:x:data' type='submit'>
      <field type='hidden' var='FORM_TYPE'>
        <value>jabber:iq:register</value>
      </field>
      <field type='text-single' label='Given Name' var='first'>
        <value>Juliet</value>
      </field>
      <field type='text-single' label='Family Name' var='last'>
        <value>Capulet</value>
      </field>
      <field type='text-single' label='Email Address' var='email'>
        <value>juliet@capulet.com</value>
      </field>
      <field type='list-single' label='Gender' var='x-gender'>
        <value>F</value>
      </field>
    </x>
  </query>
</iq>
```

## Cancellation of Existing Registration

#### 1. cilent req: 
```xml
<iq type='set' from='bill@shakespeare.lit/globe' id='unreg1'>
  <query xmlns='jabber:iq:register'>
    <remove/>
  </query>
</iq>
```
跟注册不同的是 `query` 的child多了个`<remove/>`

#### 2.1. 成功注销,server response: 
  
```xml

<iq type='result' to='bill@shakespeare.lit/globe' id='unreg1'/>

```

#### 2.2.Error Case  

|Condition | Description  |
| --- | --- |
| ``<bad-request/>``|	The <remove/> element was not the only child element of the <query/> element.|
|``<forbidden/>``	| 权限不够|
|``<not-allowed/>``	|不允许用户注销账户|
|``<registration-required/>``|要注销的账户本来就不存在|
|``<unexpected-request/>``	| The host is an instant messaging server and the IQ get does not contain a 'from' address because the entity is not registered with the server.|

## 用户修改密码

#### 1. Client:
```xml
<iq type='set' to='shakespeare.lit' id='change1'>
  <query xmlns='jabber:iq:register'>
    <username>bill</username>
    <password>newpass</password>
  </query>
</iq>

```

这里的密码是明文， 要留意客户端服务端通信是否用SSL或者TLS加密，而且服务端证书可信。

#### 2.1. 成功, Server: 

```xml
<iq type='result' id='change1'/>

```


#### 2.2. 失败 Case 


|Condition | Description  |
| --- | --- |
| ``<bad-request/>``| request请求体拼写有问题，比如没带username |
|``<not-authorized/>`` | 没通过server的安全验证 |
|``<not-allowed/>`` |	server 不允许|
|``<unexpected-request/>`` | The host is an instant messaging server and the IQ set does not contain a 'from' address because the entity is not registered with the server. |

比如：
```xml
// Bad  request
<iq type='error' from='shakespeare.lit' to='bill@shakespeare.lit/globe' id='change1'>
  <error code='400' type='modify'>
    <bad-request xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>

// Not Authorized
<iq type='error' from='shakespeare.lit' to='bill@shakespeare.lit/globe' id='change1'>
  <error code='401' type='modify'>
    <not-authorized xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>

// Not Allowed
<iq type='error' from='shakespeare.lit' to='bill@shakespeare.lit/globe' id='change1'>
  <error code='405' type='cancel'>
    <not-allowed xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>
```

有时候，服务端需要更多的信息来改密码，这时候它会返回信息提示客户端

```xml
<iq type='error' from='shakespeare.lit' to='bill@shakespeare.lit/globe' id='change1'>
  <query xmlns='jabber:iq:register'>
    <x xmlns='jabber:x:data' type='form'>
      <title>Password Change</title>
      <instructions>Use this form to change your password.</instructions>
      <field type='hidden' var='FORM_TYPE'>
        <value>jabber:iq:register:changepassword</value>
      </field>
      <field type='text-single' label='Username' var='username'>
        <required/>
      </field>
      <field type='text-private' label='Old Password' var='old_password'>
        <required/>
      </field>
      <field type='text-private' label='New Password' var='password'>
        <required/>
      </field>
      <field type='text-single' label='Mother&apos;s Maiden Name' var='x-mmn'>
        <required/>
      </field>
    </x>
  </query>
  <error code='401' type='modify'>
    <not-authorized xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>

```

然后客户端返回相关信息

```xml
<iq type='set' from='bill@shakespeare.lit/globe' to='shakespeare.lit' id='change2'>
  <query xmlns='jabber:iq:register'>
    <x xmlns='jabber:x:data' type='submit'>
      <field type='hidden' var='FORM_TYPE'>
        <value>jabber:iq:register:changepassword</value>
      </field>
      <field type='text-single' var='username'>
        <value>bill@shakespeare.lit</value>
      </field>
      <field type='text-private' var='old_password'>
        <value>theglobe</value>
      </field>
      <field type='text-private' var='password'>
        <value>groundlings</value>
      </field>
      <field type='text-single' var='x-mmn'>
        <value>Throckmorton</value>
      </field>
    </x>
  </query>
</iq>
```

ref: [XEP-0077: In-Band Registration](https://xmpp.org/extensions/xep-0077.html#usecases)
