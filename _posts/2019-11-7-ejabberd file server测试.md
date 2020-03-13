---
layout: post
title:
modified:
categories: Tech
tags: [web,ejabberd]
comments: true
---


<!-- TOC -->

- [准备](#准备)
- [流程](#流程)
- [把file_server架设在外部](#把file_server架设在外部)

<!-- /TOC -->


## 准备

windows客户端 : eyeCU

工具: eyeCU xml-console

file_server协议: <https://xmpp.org/extensions/xep-0363.html>

server module配置: <https://docs.ejabberd.im/admin/configuration/#mod-http-upload>


## 流程

* Client sends service discovery request to server

```xml
<iq from='admin@your@server'
    id='step_01'
    to='your@server'
    type='get'>
  <query xmlns='http://jabber.org/protocol/disco#items'/>
</iq>
```

Response: Server replies to service discovery request

```xml
  <iq xmlns="jabber:client" xml:lang="zh" xmlns:xml="http://www.w3.org/XML/1998/namespace" to="admin@your@server/eyeCU" type="result" from="your@server" id="step_01">
    <query xmlns="http://jabber.org/protocol/disco#items">
      <item xmlns="http://jabber.org/protocol/disco#items" jid="conference.your@server"/>
      <item xmlns="http://jabber.org/protocol/disco#items" jid="proxy.your@server"/>
      <item xmlns="http://jabber.org/protocol/disco#items" jid="pubsub.your@server"/>
      <item xmlns="http://jabber.org/protocol/disco#items" jid="upload.your@server"/>
      <item xmlns="http://jabber.org/protocol/disco#items" jid="vjud.your@server"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="通知" jid="your@server" node="announce"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="配置" jid="your@server" node="config"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="用户管理" jid="your@server" node="user"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="在线用户" jid="your@server" node="online users"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="所有用户" jid="your@server" node="all users"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="出站 s2s 连接" jid="your@server" node="outgoing s2s"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="运行中的节点" jid="your@server" node="running nodes"/>
      <item xmlns="http://jabber.org/protocol/disco#items" name="已经停止的节点" jid="your@server" node="stopped nodes"/>
    </query>
  </iq>
```

* Client sends service discovery request to upload service

```xml
<iq from='admin@your@server'
    id='step_02'
    to='upload.your@server'
    type='get'>
  <query xmlns='http://jabber.org/protocol/disco#info'/>
</iq>
```

Response:

```xml
  <iq xmlns="jabber:client" xml:lang="zh" xmlns:xml="http://www.w3.org/XML/1998/namespace" to="admin@your@server/eyeCU" type="result" from="upload.your@server" id="step_02">
    <query xmlns="http://jabber.org/protocol/disco#info">
      <identity xmlns="http://jabber.org/protocol/disco#info" type="file" name="HTTP File Upload" category="store"/>
      <feature xmlns="http://jabber.org/protocol/disco#info" var="urn:xmpp:http:upload"/>
      <feature xmlns="http://jabber.org/protocol/disco#info" var="urn:xmpp:http:upload:0"/>
      <feature xmlns="http://jabber.org/protocol/disco#info" var="eu:siacs:conversations:http:upload"/>
      <feature xmlns="http://jabber.org/protocol/disco#info" var="vcard-temp"/>
      <feature xmlns="http://jabber.org/protocol/disco#info" var="http://jabber.org/protocol/disco#info"/>
      <feature xmlns="http://jabber.org/protocol/disco#info" var="http://jabber.org/protocol/disco#items"/>
      <x xmlns="jabber:x:data" type="result">
        <field xmlns="jabber:x:data" type="hidden" var="FORM_TYPE">
          <value xmlns="jabber:x:data">urn:xmpp:http:upload</value>
        </field>
        <field xmlns="jabber:x:data" label="Maximum file size" type="text-single" var="max-file-size">
          <value xmlns="jabber:x:data">104857600</value>
        </field>
      </x>
      <x xmlns="jabber:x:data" type="result">
        <field xmlns="jabber:x:data" type="hidden" var="FORM_TYPE">
          <value xmlns="jabber:x:data">urn:xmpp:http:upload:0</value>
        </field>
        <field xmlns="jabber:x:data" label="Maximum file size" type="text-single" var="max-file-size">
          <value xmlns="jabber:x:data">104857600</value>
        </field>
      </x>
      <x xmlns="jabber:x:data" type="result">
        <field xmlns="jabber:x:data" type="hidden" var="FORM_TYPE">
          <value xmlns="jabber:x:data">http://jabber.org/network/serverinfo</value>
        </field>
      </x>
    </query>
  </iq>
```

* Client requests a slot on the upload service

```xml
<iq from='admin@your@server'
    id='step_03'
    to='upload.your@server'
    type='get'>
  <request xmlns='urn:xmpp:http:upload:0'
    filename='1.jpg'
    size='191453'
    content-type='image/jpeg' />
</iq>
```

这步申请槽，可以得到最终的put地址和get地址

Reponse:

```xml
    <iq xmlns="jabber:client" xml:lang="zh" xmlns:xml="http://www.w3.org/XML/1998/namespace" to="admin@your@server/eyeCU" type="result" from="upload.your@server" id="step_03">
    <slot xmlns="urn:xmpp:http:upload:0">
      <get xmlns="urn:xmpp:http:upload:0" url="http://your@server:5280/upload/18835e3044c959a859c7fd62cda467e563ba216a/tLJ0KIbEq64xYkk1paEo9F7xAkTmaugbXoHxMQNA/1.jpg"/>
      <put xmlns="urn:xmpp:http:upload:0" url="http://your@server:5280/upload/18835e3044c959a859c7fd62cda467e563ba216a/tLJ0KIbEq64xYkk1paEo9F7xAkTmaugbXoHxMQNA/1.jpg"/>
    </slot>
  </iq>
```

然后用post-man put到地址:

注意server 的 docroot目录(/opt/ejabberd/static)是ejabberd user权限:

```sh
ejabberd ejabberd 4.0K Nov  7 15:21 static
```

上传后看到文件来了:

```sh
/opt/ejabberd/static/18835e3044c959a859c7fd62cda467e563ba216a/tLJ0KIbEq64xYkk1paEo9F7xAkTmaugbXoHxMQNA
```

## 把file_server架设在外部

<https://modules.prosody.im/mod_http_upload_external.html>

1个python3-flask的示例:
