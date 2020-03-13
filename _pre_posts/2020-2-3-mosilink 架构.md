---
layout: post
title:
modified:
categories: Tech
tags: [linux]
comments: true
---
<!-- TOC -->

- [Carlife Vehicle](#carlife-vehicle)
    - [Module](#module)
    - [CTranRecvPackageProcess](#ctranrecvpackageprocess)
    - [ConnectionManager](#connectionmanager)
- [AOA 简单协议](#aoa-简单协议)
- [Carlife Phone](#carlife-phone)
    - [启动流程](#启动流程)
    - [ConnectionClient](#connectionclient)
    - [Connect Service](#connect-service)
    - [ConnectService](#connectservice)
        - [ConnectServiceProxy](#connectserviceproxy)
    - [A0A](#a0a)
        - [AOAConnectManager](#aoaconnectmanager)
        - [AOAAccessorySetup](#aoaaccessorysetup)
        - [SocketReadThread](#socketreadthread)
        - [AOAAccessorySetup](#aoaaccessorysetup-1)
    - [MsgHandlerCenter](#msghandlercenter)
    - [待确认项](#待确认项)

<!-- /TOC -->


# Carlife Vehicle

## Module

```
1. connectionSetupModule
    socket create
2. xxxChannelModule
    核心是tranRecvPackageProcess
```
## CTranRecvPackageProcess

具体的的xxx通道行为,也是定义在这里面,不符合面向对象的原则.比如
```c
int sendCmdHUProtoclVersion(S_HU_PROTOCOL_VERSION* version);
int sendCmdHUInfro(S_HU_INFO* huInfo);
int cmdHUBTPairInfro(S_BT_PAIR_INFO* info);
int sendCmdVideoEncoderInit(S_VIDEO_ENCODER_INIT* initParam);
int sendCmdVideoEncoderStart();
int sendCmdVideoEncoderPause();
int sendCmdVideoEncoderReset();
```

放在Module里更好

## ConnectionManager

这个类封装了所有的socket handle,自然就是包括socket的动作,发送,接收等

也就是1个单例:
```c
CConnectManager::getInstance()->writeXXXData
CConnectManager::getInstance()->recvXXXData
CConnectManager::getInstance()->createXXXSocket
```

# AOA 简单协议

usb 最外层:

![Screenshot from 2020-02-05 19-13-30-cad280e0-c409-4a07-bd08-35c1fa07548c](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202020-02-05%2019-13-30-cad280e0-c409-4a07-bd08-35c1fa07548c.png)

carlife message:

![Screenshot from 2020-02-05 19-14-12-8ed989a0-e520-43f1-822d-4777970c3e41](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202020-02-05%2019-14-12-8ed989a0-e520-43f1-822d-4777970c3e41.png)

data就是pb序列流

# Carlife Phone

非官方,借鉴下别人的思路.

## 启动流程

OnCreate->startCarlifeActivity:
```
由MsgHandlerCenter dispatchMessage出MSG_ANDROID_DEBUG_CONNECT_STATUS_LISTENED.
```

Handler mMainActivityHandler收到MSG_ANDROID_DEBUG_CONNECT_STATUS_LISTENED:
```
1 ConnectClient init
    1.1 建立 Service启动后的回调ServceConnection;
    ConnectionClientHandler->HandlerThread->Messenger,作为上面mConnection的replyTo,
    跨进程通信,对Service的回应注册了回调Handler,可以收到Service回发的信息,
    具体是在ConnectClientHandler的handleMessage里.
    1.2 注册2个Receiver,
        1.2.1 ConnectServiceReceiver
            收到信息后,发送到1.1的handler,就是将广播转成了Message:
                MSG_CONNECT_SERVICE_MSG_START
                    作用:bindService
                MSG_CONNECT_SERVICE_MSG_STOP
                    作用:unbindService
        1.2.2  UsbConnectStateReceiver
            系统广播USB_STATE_ACTION, USB状态变化
            改变isUsbConnected这个变量
            拔出检测更好?
        
2 ConnectService: 收到CARLIFE_CONNECT_SERVICE_START broadcast
    广播被1.1注册的ConnectClientHandler接收
```

兜转了半天,就是来bind或者unbind Service!


## ConnectionClient

除启动Service外,作为Client, 还负责和Service通信,很直白了.

重点看sendMsgToService:
```
1  onServiceConnected里发送MSG_REC_REGISTER_CLIENT
2  unbindService里发送MSG_REC_UNREGISTER_CLIENT
//下面是具体应用,sdk角度来讲不用关心了
3 EncryptSetupManager
4 CarDataManager
5 CarlifeProtocolVersionInfoManager
6 BtHfpProtocolManager
7 CarlifeDeviceInfoManager
```


## Connect Service

3个类ConnectManager ConnectService ConnectServiceProxy

但作用好像划分的不太准确, 可以优化

## ConnectService
拥有ConnectManager和ConnectServeProxy的对象,感觉它才是Manager

```
1 Cached Message
    部分消息在Service没初始化好,就发过来了,需要Cache
2 OnCreate
    2.0 AOA init, AOA转发线程启动, 其他应用init;
    2.1 new ConnectServiceProxy对象
    2.2 建立2个handler,处理消息
     2.2.1 mConnectServiceHandler
        Service收到的消息,先到这个,再转给Proxy
     2.2.2 初始化ConnectServiceProxy后拿到handler
        ConnectServiceProxy是真正的和Service通信的消息处理者.
    2.3 启动ConnectionManager里的线程
        AOAConnectThread 

```

### ConnectServiceProxy

真正的Service消息处理的代理.

完成Service到AOA底层架构的转发

## A0A

### AOAConnectManager

AOAConnectManager:
```
3种线程:
AOAConnectThread
    这个线程run后就执行1次scanUsbDevices,即openAccessary,然后break;
    执行时机正确?
AOAReadThread
    usb原始线程,收到数据send给各大ScoketReadThread
SocketReadThread
    具体的协议处理
```
###  AOAAccessorySetup

相关的操作:
```
AOAAccessorySetup.getInstance().init(mContext);

//connect thread
AOAAccessorySetup.getInstance().scanUsbDevices()

// aoa read
AOAAccessorySetup.getInstance().bulkTransferIn


AOAAccessorySetup.getInstance().unInit();
```

### SocketReadThread
UsbRead后
```java
ConnectManager.getInstance().writeCmdData(msg, lenMsg);
ConnectManager.getInstance().writeAndroidDebugData(msg, lenMsg);
```
write到了mConnectSocket. 其实就是客户端的socket.

```java
connectSocket(SERVER_SOCKET_PORT, CommonParams.SERVER_SOCKET_NAME);
connectSocket(SERVER_SOCKET_VIDEO_PORT, CommonParams.SERVER_SOCKET_VIDEO_NAME);
connectSocket(SERVER_SOCKET_TOUCH_PORT, CommonParams.SERVER_SOCKET_TOUCH_NAME);
connectSocket(SERVER_SOCKET_AUDIO_VR_PORT, CommonParams.SERVER_SOCKET_AUDIO_VR_NAME);
connectSocket(SERVER_SOCKET_ANDROID_DEBUG_PORT, CommonParams.SERVER_SOCKET_ANDROID_DEBUG_NAME);
```

然后Server端就能收到,也就是SocketReadThread

### AOAAccessorySetup

2个receiver监控usb插入的:
```
OpenAccessoryReceiver
UsbDetachedReceiver
```
2个steam,在插入,读取相应的descriptor后,进行IO操作的:
```
mFileInputStream = new FileInputStream(fileDescriptor);
mFileOutputStream = new FileOutputStream(fileDescriptor);
```

## MsgHandlerCenter

所有添加到handlercenter的hander,可以收到相同的消息.方法是center里的dispatchmessage

作用就是通过1个center可以通知多个handler拥有者(即线程), 即同时通知多个线程.

相当于管理多个(handler)线程.

3个hanlder,最主要的就是MainActivityHandler:

它处理MSG_ANDROID_DEBUG_CONNECT_STATUS_LISTENED,这个消息是最初一定会发的消息

处理过程是:
```
MSG_ANDROID_DEBUG_CONNECT_STATUS_LISTENED:

```


## 待确认项
```sh
1. SDK 要不要包含整个Service的设计?
2. 如果要, SDK 回调的必定是Service Client的数据?
```