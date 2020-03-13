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

<!-- /TOC -->


## 准备

windows客户端 : eyeCU

工具: eyeCU xml-console

协议 xep-054: <https://xmpp.org/extensions/xep-0054.html>

## 流程

* 获取vcard

```xml
Start sending user stanza...

>>>> admin@49.233.92.47/eyeCU 16:23:11 +0 >>>>
  <iq id="v1" type="get">
    <vCard xmlns="vcard-temp"/>
  </iq>

User stanza sent.

<<<< admin@49.233.92.47/eyeCU 16:23:11 +87 <<<<
  <iq xmlns="jabber:client" id="v1" type="result" xml:lang="zh" xmlns:xml="http://www.w3.org/XML/1998/namespace" to="admin@49.233.92.47/eyeCU" from="admin@49.233.92.47">
    <vCard xmlns="vcard-temp">
      <PHOTO xmlns="vcard-temp">
        <BINVAL xmlns="vcard-temp">/9j/4AAQSkZJRgABAQAASABIAAD/4QBYRXhpZgAATU0AKgAAAAgAAgESAAMAAAABAAEAAIdpAAQAAAABAAAAJgAAAAAAA6ABAAMAAAABAAEAAKACAAQAAAABAAAAIqADAAQAAAABAAAAIgAAAAD/7QA4UGhvdG9zaG9wIDMuMAA4QklNBAQAAAAAAAA4QklNBCUAAAAAABDUHYzZjwCyBOmACZjs+EJ+/8AAEQgAIgAiAwEiAAIRAQMRAf/EAB8AAAEFAQEBAQEBAAAAAAAAAAABAgMEBQYHCAkKC//EALUQAAIBAwMCBAMFBQQEAAABfQECAwAEEQUSITFBBhNRYQcicRQygZGhCCNCscEVUtHwJDNicoIJChYXGBkaJSYnKCkqNDU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6g4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2drh4uPk5ebn6Onq8fLz9PX29/j5+v/EAB8BAAMBAQEBAQEBAQEAAAAAAAABAgMEBQYHCAkKC//EALURAAIBAgQEAwQHBQQEAAECdwABAgMRBAUhMQYSQVEHYXETIjKBCBRCkaGxwQkjM1LwFWJy0QoWJDThJfEXGBkaJicoKSo1Njc4OTpDREVGR0hJSlNUVVZXWFlaY2RlZmdoaWpzdHV2d3h5eoKDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uLj5OXm5+jp6vLz9PX29/j5+v/bAEMABgYGBgYGCgYGCg4KCgoOEg4ODg4SFxISEhISFxwXFxcXFxccHBwcHBwcHCIiIiIiIicnJycnLCwsLCwsLCwsLP/bAEMBBwcHCwoLEwoKEy4fGh8uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLi4uLv/dAAQAA//aAAwDAQACEQMRAD8A+ktW1aLTIhkbpW+6v9T7V59d6rf3hPnSnB/hHC/kKNVuzeX8s2cruIX/AHRwKqQQmZ9mdoALEnsB1r1aNGMI3e54WIxEqk+WL0H295dWpzbyMn0PFdvoviD7YwtbvAlP3WHAb2PvXDTxRx4Mbls9iNrD6iokZkYOpwQcgirqUo1EZ0q86UtGez0VhQa9aNBG0rYcqC31xzUv9uWH9+vO+rz7HsfW6fc//9D0m8tza3Ulu38DEfhTrKcQSksdoZSu7rjPQ/ga7XX9Fa8/0u1GZQMMv94DuPeuBZGQlXBBHUGvYpzVSB89WpSo1C3dSq0UcRk811LEvz0OMDJ5PrVLrR1rqND0OS5kW6ul2wryAf4v/rVUpKnG7IjCVWVki1B4Z8yCORmILKCR6ZFTf8IsP79djRXn/W5nrfUIH//R+qawtdgha0aVo1LgfeIGfzrdrH1z/jwet8P8aObF/wANmB4Zghk3NJGrEHgkAkV29cb4W+69dlV4v4jLAfAFFFFcp3H/2Q==</BINVAL>
        <TYPE xmlns="vcard-temp">image/jpeg</TYPE>
      </PHOTO>
      <NICKNAME xmlns="vcard-temp">admin boy</NICKNAME>
    </vCard>
  </iq>
```

PHOTO就是头像,base64编码