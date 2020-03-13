---
layout: post
title:
modified:
categories: Tech
tags: [python]
image:
comments: true
---

<!-- TOC -->

- [Enum](#Enum)
- [Structure](#Structure)
- [restype and argstype](#restype-and-argstype)
- [回调的注册方法](#回调的注册方法)
- [稍微特殊的函数](#稍微特殊的函数)

<!-- /TOC -->

参考[python libusb1](https://pypi.org/project/libusb1/)
python 如何调用 c 语言的库？如果优雅的，规范的调用 c 语言的库?
看完 python libusb1 上面的问题，自然就有答案了.

## Enum

libusb 里，定义了大量的 c 的枚举.

Enum 用来将 c 里的 cnum 转成 python 里的数据结构，并可以反查.

```py
class Enum(object):
    def __init__(self, member_dict, scope_dict=None):
        if scope_dict is None:
            # Affect caller's locals, not this module's.
            # pylint: disable=protected-access
            scope_dict = sys._getframe(1).f_locals
            # pylint: enable=protected-access
        forward_dict = {}
        reverse_dict = {}
        next_value = 0
        for name, value in list(member_dict.items()):
            if value is None:
                value = next_value
                next_value += 1
            forward_dict[name] = value
            if value in reverse_dict:
                raise ValueError('Multiple names for value %r: %r, %r' % (
                    value, reverse_dict[value], name
                ))
            reverse_dict[value] = name
            scope_dict[name] = value
        self.forward_dict = forward_dict
        self.reverse_dict = reverse_dict

    def __call__(self, value):
        return self.reverse_dict[value]

    def get(self, value, default=None):
        return self.reverse_dict.get(value, default)
```

测试如下:

```sh
from libusb1 import Enum as ReverseEnum
e = ReverseEnum({})
e = ReverseEnum({"111":1,"222":2,"333":3})
e.forward_dict['111']
>>1
e(2)
>>'222'
e.get(3)
>>'333
```

## Structure

自然就是用 ctypes 里的 Structure 定义了,同时定义 1 个 pinter 类型:

```py
class libusb_endpoint_descriptor(Structure):
    _fields_ = [
        ('bLength', c_uint8),
        ('bDescriptorType', c_uint8),
        ('bEndpointAddress', c_uint8),
        ('bmAttributes', c_uint8),
        ('wMaxPacketSize', c_uint16),
        ('bInterval', c_uint8),
        ('bRefresh', c_uint8),
        ('bSynchAddress', c_uint8),
        ('extra', c_void_p),
        ('extra_length', c_int)]
libusb_endpoint_descriptor_p = POINTER(libusb_endpoint_descriptor)
```

也可以这样定义，可以使用前向声明(比如 c 里的链表 Head 结构):

```py
_libusb_transfer_fields = [
    ('dev_handle', libusb_device_handle_p),
    ('flags', c_uint8),
    ('endpoint', c_uchar),
    ('type', c_uchar),
    ('timeout', c_uint),
    ('status', c_int), # enum libusb_transfer_status
    ('length', c_int),
    ('actual_length', c_int),
    ('callback', libusb_transfer_cb_fn_p),
    ('user_data', c_void_p),
    ('buffer', c_void_p),
    ('num_iso_packets', c_int),
    ('iso_packet_desc', libusb_iso_packet_descriptor)
]
class libusb_transfer(Structure):
    pass
libusb_transfer_p = POINTER(libusb_transfer)
libusb_transfer._fields_ = _libusb_transfer_fields
```

- python 还有能干这个活的内置 struct

[比较 struct and ctypes Structure](https://stackoverflow.com/questions/52004279/python-similar-functionality-in-struct-and-array-vs-ctypes)

struct 用来 pack/unpack binarydata,是 ctypes Structure 的底层实现.

ctypes 的 Structure 的优点，像 c 一样使用结构体，比较清晰.

ctypes 的 Structure 的缺点:uses the byte order that is native to C.当需要处理复杂的 bigendian littleendian 时，最好用 struct,而且效率也高些.

## restype and argstype

给从 so 里 load 到的函数，定义返回值和参数:

```py
libusb_get_device_list = libusb.libusb_get_device_list
libusb_get_device_list.argtypes = [libusb_context_p, libusb_device_p_p_p]
libusb_get_device_list.restype = c_ssize_t
```

## 回调的注册方法

c 里的回调，在 python 里处理，自然是用的 FUNCTYPE

```py
#typedef int (*libusb_hotplug_callback_fn)(libusb_context *ctx,
#        libusb_device *device, libusb_hotplug_event event, void *user_data);
libusb_hotplug_callback_fn_p = CFUNCTYPE(
    c_int, libusb_context_p, libusb_device_p, c_int, c_void_p)

#int libusb_hotplug_register_callback(libusb_context *ctx,
#        libusb_hotplug_event events, libusb_hotplug_flag flags,
#        int vendor_id, int product_id, int dev_class,
#        libusb_hotplug_callback_fn cb_fn, void *user_data,
#        libusb_hotplug_callback_handle *handle);
try:
    libusb_hotplug_register_callback = libusb.libusb_hotplug_register_callback
except AttributeError:
    pass
else:
    libusb_hotplug_register_callback.argtypes = [
        libusb_context_p,
        c_int, c_int,
        c_int, c_int, c_int,
        libusb_hotplug_callback_fn_p, c_void_p,
        POINTER(libusb_hotplug_callback_handle),
    ]
    libusb_hotplug_register_callback.restype = c_int
```

## 稍微特殊的函数

其实也就是用到了 cast.

```py
def libusb_fill_control_setup(
        setup_p, bmRequestType, bRequest, wValue, wIndex, wLength):
    # 一般情况下，setup_p应该就是libusb_control_setup_p,
    # 但是也许有pointer(create_string_buffer())创建的Array pointer，也是可以强制转换的
    setup = cast(setup_p, libusb_control_setup_p).contents
    setup.bmRequestType = bmRequestType
    setup.bRequest = bRequest
    setup.wValue = libusb_cpu_to_le16(wValue)
    setup.wIndex = libusb_cpu_to_le16(wIndex)
    setup.wLength = libusb_cpu_to_le16(wLength)
```

上面的几个用法搞明白，python 调用 c 不再有什么问题了.
