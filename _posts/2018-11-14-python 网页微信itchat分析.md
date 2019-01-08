---
layout: post
title:
modified:
categories: Tech
 
tags: [python]

  
comments: true
---

<!-- TOC -->

- [itchat可以做什么](#itchat可以做什么)
- [itchat实现分析](#itchat实现分析)
    - [AbstractUserDict](#abstractuserdict)
    - [Contactlist](#contactlist)
    - [Storage](#storage)

<!-- /TOC -->

[itchat](https://itchat.readthedocs.io/zh/latest/)

## itchat可以做什么

itchat封装了微信web api，蛮好用的，可以做一些有意思的事情:

另外还有个库itchatmp和公众号管理相关的

脑洞可以这么打开:
* 管理

所有聊天记录备份

聊天记录监控(一个打入敌人内部的人...加了n多群)

群聊管理,自动拉群，定时群发(拜年)

红包监控,但是不能直接抢...

监控公众号推送

撤回消息保存

谁删了我?

* 账号云图分析

联系人统计分析，个人关注的公众号，做口味分析

好友签名云图

头像整合?分析

聊天记录分析?-->词云

* 控制入口，后台的空间任意想象
webapi app-->微信入口--->any后台

聊天机器人

一秒变逗逼--->nlp

可指定在哪个群，变成哪种人(逗逼，暴躁男，文艺男...),

收到问题-->后台--->回复问题

收到文本-->后台-->回复各种萌萌哒语音

收到语音-->后台tts,分析后-->回复各种指令，控制

收到图像-->后台分析--->给出反馈，如美化，识别结果 等


自动问题告警

## itchat实现分析

封装的还是不错的，可以剖析下

### AbstractUserDict

先搞清楚以下继承链:
```sh
list->AttributeDict->AbstractUserDict-->(User, ChatRoom,MassivePlatform)

list->增加list.name的方式，和list['name']一致-->增加了很多方法，send相关的-->
具体的实现子类，User需要send,聊天室需要send,公众号需要send

update
set_alias
set_pinned
verify
get_head_image
delete_member
add_member
send_raw_msg
send_msg
send_file
send_image
send_vedio
send
search_member
```

AbstractUserDict里的send，基本都是调用core里的,core是通过property设置的一个属性

```py
def send_image(self, fileDir, mediaId=None):
    return self.core.send_image(fileDir, self.userName, mediaId)
```
core类里也不是真正实现函数地方，还有其他地方
```py
  def send_msg(self, msg='Test Message', toUserName=None):
        ''' send plain text message
            for options
                - msg: should be unicode if there's non-ascii words in msg
                - toUserName: 'UserName' key of friend dict
            it is defined in components/messages.py
        '''
        raise NotImplementedError()
```
应该是core.py最后的load_components,各种的实现又放在各自的py里，如contact.py,hotreload.py等

具体的requets用法，都在这些文件里，好好看懂，基本就是这个了.
```py
from .contact import load_contact
from .hotreload import load_hotreload
from .login import load_login
from .messages import load_messages
from .register import load_register

def load_components(core):
    load_contact(core)
    load_hotreload(core)
    load_login(core)
    load_messages(core)
    load_register(core)
```

在load_xxx指向了真正的调用者，这样的好处?封装，解耦把? 还有绝的比较好的,就是简化import
```py
def load_contact(core):
    core.update_chatroom             = update_chatroom
    core.update_friend               = update_friend
    core.get_contact                 = get_contact
    core.get_friends                 = get_friends
    core.get_chatrooms               = get_chatrooms
    core.get_mps                     = get_mps
    core.set_alias                   = set_alias
    core.set_pinned                  = set_pinned
    core.add_friend                  = add_friend
    core.get_head_img                = get_head_img
    core.create_chatroom             = create_chatroom
    core.set_chatroom_name           = set_chatroom_name
    core.delete_member_from_chatroom = delete_member_from_chatroom
    core.add_member_into_chatroom    = add_member_into_chatroom
```


### Contactlist
还有1个contactlist，继承自list, 也是把core当成了一个属性
```sh
list->Contactlist
```
ContactList 有1个ContactClass成员，就是上面的AbstractUserDict的子类实现,通过set_default_value赋值:
```py
def set_default_value(self, initFunction=None, contactClass=None):
    if hasattr(initFunction, '__call__'):
        self.contactInitFn = initFunction
    if hasattr(contactClass, '__call__'):
        self.contactClass = contactClass

```
调用ContactList.append时，会用到:
```py
 def append(self, value):
        contact = self.contactClass(value)
        contact.core = self.core
        if self.contactInitFn is not None:
            contact = self.contactInitFn(self, contact) or contact
        super(ContactList, self).append(contact)

```

### Storage
哪个类又用到了ContactList?是Storage.存1个用户所有的联系人信息?核心是实现了各种search

```py
class Storage(object):
    def __init__(self, core):
        self.userName          = None
        self.nickName          = None
        self.updateLock        = Lock()
        self.memberList        = ContactList()
        self.mpList            = ContactList()
        self.chatroomList      = ContactList()
        self.msgList           = Queue(-1)
        self.lastInputUserName = None
        self.memberList.set_default_value(contactClass=User)
        self.memberList.core = core
        self.mpList.set_default_value(contactClass=MassivePlatform)
        self.mpList.core = core
        self.chatroomList.set_default_value(contactClass=Chatroom)
        self.chatroomList.core = core
    def search_friends(self, name=None, userName=None, remarkName=None, nickName=None,
            wechatAccount=None):
    def search_chatrooms(self, name=None, userName=None):
    def search_mps(self, name=None, userName=None):
```

Storage又被谁用？自然是core里引用:
```py
class Core(object):
    def __init__(self):
        ''' init is the only method defined in core.py
            alive is value showing whether core is running
                - you should call logout method to change it
                - after logout, a core object can login again
            storageClass only uses basic python types
                - so for advanced uses, inherit it yourself
            receivingRetryCount is for receiving loop retry
                - it's 5 now, but actually even 1 is enough
                - failing is failing
        '''
        self.alive, self.isLogging = False, False
        self.storageClass = storage.Storage(self)
        self.memberList = self.storageClass.memberList
        self.mpList = self.storageClass.mpList
        self.chatroomList = self.storageClass.chatroomList
        self.msgList = self.storageClass.msgList
        self.loginInfo = {}
        self.s = requests.Session()
        self.uuid = None
        self.functionDict = {'FriendChat': {}, 'GroupChat': {}, 'MpChat': {}}
        self.useHotReload, self.hotReloadDir = False, 'itchat.pkl'
        self.receivingRetryCount = 5
```