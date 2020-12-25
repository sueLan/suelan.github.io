---
title: 'XMPP(4):Search 和 vCard'
date: 2019-03-31 15:35:51
categories: 
    - XMPP
---
`jabber:iq:search`协议用来查找用户信息。

1. 我们先查询可以用哪些字段查找用户
<!-- more -->

# XMPP Search 

`jabber:iq:search`协议用来查找用户信息。

1. 我们先查询可以用哪些字段查找用户

```xml
// Requesting Search Fields

<iq type='get'
    from='romeo@montague.net/home'
    to='characters.shakespeare.lit'
    id='search1'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'/>
</iq>
```

2. service 返回

```xml
// Receiving Search Fields
<iq type='result'
    from='characters.shakespeare.lit'
    to='romeo@montague.net/home'
    id='search1'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'>
    <instructions>
      Fill in one or more fields to search
      for any matching Jabber users.
    </instructions>
    <first/>
    <last/>
    <nick/>
    <email/>
  </query>
</iq>
```
3. 服务端返回，可以用`first` `last` `nick` `email` 这几个字段找人。接着就用last查人.

```xml
// Submitting a Search Request

<iq type='set'
    from='romeo@montague.net/home'
    to='characters.shakespeare.lit'
    id='search2'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'>
    <last>Capulet</last>
  </query>
</iq>
```

服务端可以能会返回好多个last匹配的item
```xml
<iq type='result'
    from='characters.shakespeare.lit'
    to='romeo@montague.net/home'
    id='search2'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'>
    <item jid='juliet@capulet.com'>
      <first>Juliet</first>
      <last>Capulet</last>
      <nick>JuliC</nick>
      <email>juliet@shakespeare.lit</email>
    </item>
    <item jid='tybalt@shakespeare.lit'>
      <first>Tybalt</first>
      <last>Capulet</last>
      <nick>ty</nick>
      <email>tybalt@shakespeare.lit</email>
    </item>
  </query>
</iq>
```
没有结果的话，query就没有子元素

```xml
<iq type='result'
    from='characters.shakespeare.lit'
    to='romeo@montague.net/home'
    id='search2'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'/>
</iq>
```

XMPP Search 

`jabber:iq:search`协议用来查找用户信息。

我们先查询可以用哪些字段查找用户

```xml
// Requesting Search Fields

<iq type='get'
    from='romeo@montague.net/home'
    to='characters.shakespeare.lit'
    id='search1'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'/>
</iq>
```

service 返回

```xml
// Receiving Search Fields
<iq type='result'
    from='characters.shakespeare.lit'
    to='romeo@montague.net/home'
    id='search1'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'>
    <instructions>
      Fill in one or more fields to search
      for any matching Jabber users.
    </instructions>
    <first/>
    <last/>
    <nick/>
    <email/>
  </query>
</iq>
```
服务端返回，可以用`first` `last` `nick` `email` 这几个字段找人。接着就用last查人.

```xml
// Submitting a Search Request

<iq type='set'
    from='romeo@montague.net/home'
    to='characters.shakespeare.lit'
    id='search2'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'>
    <last>Capulet</last>
  </query>
</iq>
```

服务端可以能会返回好多个last匹配的item
```xml
<iq type='result'
    from='characters.shakespeare.lit'
    to='romeo@montague.net/home'
    id='search2'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'>
    <item jid='juliet@capulet.com'>
      <first>Juliet</first>
      <last>Capulet</last>
      <nick>JuliC</nick>
      <email>juliet@shakespeare.lit</email>
    </item>
    <item jid='tybalt@shakespeare.lit'>
      <first>Tybalt</first>
      <last>Capulet</last>
      <nick>ty</nick>
      <email>tybalt@shakespeare.lit</email>
    </item>
  </query>
</iq>
```
没有结果的话，query就没有子元素

```xml
<iq type='result'
    from='characters.shakespeare.lit'
    to='romeo@montague.net/home'
    id='search2'
    xml:lang='en'>
  <query xmlns='jabber:iq:search'/>
</iq>
```

# vCard 
vCard协议主要负责用户信息存储，就像个人名片。

1. 查看自己的vCard
如果客户端想查询自己的vCard, 需要发送IQ-set stanza，注意没有to地址哦。

```xml
<iq from='stpeter@jabber.org/roundabout'
    id='v1'
    type='get'>
  <vCard xmlns='vcard-temp'/>
</iq>
```

2. 返回信息
接着服务端返回一堆的用户信息

```xml

<iq id='v1'
    to='stpeter@jabber.org/roundabout'
    type='result'>
  <vCard xmlns='vcard-temp'>
    <FN>Peter Saint-Andre</FN>
    <N>
      <FAMILY>Saint-Andre</FAMILY>
      <GIVEN>Peter</GIVEN>
      <MIDDLE/>
    </N>
    <NICKNAME>stpeter</NICKNAME>
    <URL>http://www.xmpp.org/xsf/people/stpeter.shtml</URL>
    <BDAY>1966-08-06</BDAY>
    <ORG>
      <ORGNAME>XMPP Standards Foundation</ORGNAME>
      <ORGUNIT/>
    </ORG>
    <TITLE>Executive Director</TITLE>
    <ROLE>Patron Saint</ROLE>
    <TEL><WORK/><VOICE/><NUMBER>303-308-3282</NUMBER></TEL>
    <TEL><WORK/><FAX/><NUMBER/></TEL>
    <TEL><WORK/><MSG/><NUMBER/></TEL>
    <ADR>
      <WORK/>
      <EXTADD>Suite 600</EXTADD>
      <STREET>1899 Wynkoop Street</STREET>
      <LOCALITY>Denver</LOCALITY>
      <REGION>CO</REGION>
      <PCODE>80202</PCODE>
      <CTRY>USA</CTRY>
    </ADR>
    <TEL><HOME/><VOICE/><NUMBER>303-555-1212</NUMBER></TEL>
    <TEL><HOME/><FAX/><NUMBER/></TEL>
    <TEL><HOME/><MSG/><NUMBER/></TEL>
    <ADR>
      <HOME/>
      <EXTADD/>
      <STREET/>
      <LOCALITY>Denver</LOCALITY>
      <REGION>CO</REGION>
      <PCODE>80209</PCODE>
      <CTRY>USA</CTRY>
    </ADR>
    <EMAIL><INTERNET/><PREF/><USERID>stpeter@jabber.org</USERID></EMAIL>
    <JABBERID>stpeter@jabber.org</JABBERID>
    <DESC>
      More information about me is located on my
      personal website: http://www.saint-andre.com/
    </DESC>
  </vCard>
</iq>
```
如果没有相关vCard，会返回error
```xml
// item-not-found
<iq id='v1'
    to='stpeter@jabber.org/roundabout'
    type='error'>
  <vCard xmlns='vcard-temp'/>
  <error type='cancel'>
    <item-not-found xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>
```

```xml
// empty element
<iq id='v1'
    to='stpeter@jabber.org/roundabout'
    type='result'>
  <vCard xmlns='vcard-temp'/>
</iq>

```

3. 查看别人的vCard

用IQ-get stanza, 带上to地址

```xml 

<iq from='stpeter@jabber.org/roundabout'
    id='v3'
    to='jer@jabber.org'
    type='get'>
  <vCard xmlns='vcard-temp'/>
</iq>
```

```xml
<iq from='jer@jabber.org'
    to='stpeter@jabber.org/roundabout'
    type='result'
    id='v3'>
  <vCard xmlns='vcard-temp'>
    <FN>JeremieMiller</FN>
    <N>
      <GIVEN>Jeremie</GIVEN>
      <FAMILY>Miller</FAMILY>
      <MIDDLE/>
    </N>
    <NICKNAME>jer</NICKNAME>
    <EMAIL><INTERNET/><PREF/><USERID>jeremie@jabber.org</USERID></EMAIL>
    <JABBERID>jer@jabber.org</JABBERID>
  </vCard>
</iq>

```

4. 更新vCard

客户端可以用IQ-set stanza 更新自己的vCard信息

```xml
<iq id='v2' type='set'>
  <vCard xmlns='vcard-temp'>
    <FN>Peter Saint-Andre</FN>
    <N>
      <FAMILY>Saint-Andre</FAMILY>
      <GIVEN>Peter</GIVEN>
      <MIDDLE/>
    </N>
    <NICKNAME>stpeter</NICKNAME>
    <URL>http://www.xmpp.org/xsf/people/stpeter.shtml</URL>
    <BDAY>1966-08-06</BDAY>
    <ORG>
      <ORGNAME>XMPP Standards Foundation</ORGNAME>
      <ORGUNIT/>
    </ORG>
    <TITLE>Executive Director</TITLE>
    <ROLE>Patron Saint</ROLE>
    <TEL><WORK/><VOICE/><NUMBER>303-308-3282</NUMBER></TEL>
    <TEL><WORK/><FAX/><NUMBER/></TEL>
    <TEL><WORK/><MSG/><NUMBER/></TEL>
    <ADR>
      <WORK/>
      <EXTADD>Suite 600</EXTADD>
      <STREET>1899 Wynkoop Street</STREET>
      <LOCALITY>Denver</LOCALITY>
      <REGION>CO</REGION>
      <PCODE>80202</PCODE>
      <CTRY>USA</CTRY>
    </ADR>
    <TEL><HOME/><VOICE/><NUMBER>303-555-1212</NUMBER></TEL>
    <TEL><HOME/><FAX/><NUMBER/></TEL>
    <TEL><HOME/><MSG/><NUMBER/></TEL>
    <ADR>
      <HOME/>
      <EXTADD/>
      <STREET/>
      <LOCALITY>Denver</LOCALITY>
      <REGION>CO</REGION>
      <PCODE>80209</PCODE>
      <CTRY>USA</CTRY>
    </ADR>
    <EMAIL><INTERNET/><PREF/><USERID>stpeter@jabber.org</USERID></EMAIL>
    <JABBERID>stpeter@jabber.org</JABBERID>
    <DESC>
      Check out my blog at https://stpeter.im/
    </DESC>
  </vCard>
</iq>
```

服务端返回结果

```xml
<iq id='v2'
    to='stpeter@jabber.org/roundabout'
    type='result'/>
```

ref: https://xmpp.org/extensions/xep-0054.html#intro
ref: https://xmpp.org/extensions/xep-0055.html#intro