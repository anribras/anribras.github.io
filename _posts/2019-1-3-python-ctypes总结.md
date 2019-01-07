---
layout: post
title: 
modified:
categories: Tech
tags: [python]
comments: true
---

<!-- TOC -->

- [核心概念](#核心概念)
- [实例1: ctypes + socket](#实例1-ctypes--socket)
- [实例2 libusb使用](#实例2-libusb使用)
- [末了](#末了)

<!-- /TOC -->


## 核心概念

[ctypes官方](https://docs.python.org/3/library/ctypes.html)

* 类型一览 

ctypes是python定义的为实现类型转换的中间层，是纯python数据类型与c类型通信的桥梁.
![Screenshot from 2019-01-03 11-33-04-fecf6500-f845-4dd4-bc06-2deb65be7f33](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-03%2011-33-04-fecf6500-f845-4dd4-bc06-2deb65be7f33.png)

除了None,integer,string，bytes，(隐式转换), 其他都需要转换成ctypes类型作为参数.
```sh
None -->NULL
string bytes -- > char* wchar_t*
integer -- > int
```
```py
#c_double 必须显示转换
printf(b'Hello %s %f\n', b'Wisdom', c_double(1.23))
```
标准类型里唯一要注意的是c_char_p,很像C里的字符串类型， 它只能转换integer或者bytes作为输入.

普通的'char *'类型用POINTER(c_char)
```sh
class ctypes.c_char_p
Represents the C char * datatype when it points to a zero-terminated string. For a general character pointer that may also point to binary data, POINTER(c_char) must be used. The constructor accepts an integer address, or a bytes object.
```

* 严格区分类型

不像c指针那么万能,python里一切皆对象.数组，指针都是要按不同的类型来分开的。

```sh
1 int a = 100;  
    ----> ct_a = c_int(100)
2 int array[100] = {0,1,2,...} 
    ---->ct_array = (c_int* 100)(0,1,2,...)
         for i in ct_array: print(i)
3 int *ptr = &a;  
    ----> ct_ptr = POINTER(c_int)(ct_a)
          ct_ptr.contents
          c_int(10)
4 *ptr = 200 ;
// ptr[0] = 200
    ----> ct_ptr[0] = 200 or ct_ptr.contents.value = 300
5 int *ptr_arr = array;
    ---->  ctype做不到! 对python 来讲，数组和指针是不同的类型 
```

* Access dll function and values

function见下面例2.

value:
```sh
>>> opt_flag = c_int.in_dll(pythonapi, "Py_OptimizeFlag")
>>> print(opt_flag)
c_long(0)
```

* resttype argstype

load c的lib后，必须告诉Python,一个ctype函数的形参类型和返回的值的类型, 函数才能正确地在python里调用.
```py
libusb_alloc_transfer = libusb.libusb_alloc_transfer
libusb_alloc_transfer.argtypes = [c_int]
libusb_alloc_transfer.restype = libusb_transfer_p
```

* convert python type to ctype

1.构造ctype对象时输入;

2.通过value输入输入;

```sh
ct_int = ctypes.c_int(100)
ct_int.value
100
ct_int.value = 200
ct_int
c_int(200)
```
如果是ctype指针，也是要通过赋值:
但是python的bytes是immutable的，也就是bytes内容变化，是指向了另外的内存，而不是内存内容本身发生了变化，这和c的一些要求是不符合的:
```sh
ct_str = ctypes.c_wchar_p('Hi!')
ct_str.value
'Hi!'
ct_str
c_wchar_p(140093580736096)
ct_str = ctypes.c_wchar_p('Hi again')
ct_str.value
'Hi again'
ct_str
c_wchar_p(140093530479112)
```
用下面的 create_string_buffer解决问题

* create_string_buffer
创建可修改的ctype内存空间
```sh
>>buf = create_string_buffer(b'12345')
>>buf.value
>>b'12345'
```

* byref and pointer and POINTER 

`byref`:

拿到ctype instance的指针对象，,offset是偏移的地址. 但是只能用作ctype函数的参数,比pointer(obj)更快.

`pointer`:

pointer则是构造ctype对象的ctype pointer实例对象,`普通对象变成指针对象`
```sh
ctypes.pointer(obj)
Pointer instances are created by calling the pointer() function on a ctypes type.
This function creates a new pointer instance, pointing to obj. The returned object is of the
type POINTER(type(obj)).
```
用contents取出内容,类似于`*a`:

* POINTER

POINTER仅仅是ctype层面的指针类型转换，比如1个ctypes类型转成其指针的类型,`类型变成指针类型` 
```sh
ctypes.POINTER(type)
This factory function creates and returns a new ctypes pointer type. Pointer types are
cached and reused internally, so calling this function repeatedly is cheap. type must be a
ctypes type.
```
例子:
```sh
libc.printf(b'%x',byref(c_int(123)))
>>bf60f9188
type(byref(c_int(123)))

ct_int = ct_int(123)
>> <class 'CArgObject'>
ip = pointer(ct_int)
type(ip)
>> <class '__main__.LP_c_int'>
ip.contents
>>c_int(0)
type_int_p = POINTER(c_int)
type(type_int_p)
>> <class '_ctypes.PyCPointerType'>
ip_1 = type_int_p(c_int(123))
ip_1.contents
>>c_int(123)
type(ip_1)
<class '__main__.LP_c_int'>
>>False
```

* Structure

参考下面例子里定义.

1.注意一点，字节序是native bytes order.

2.前向声明结构体的方式:

```sh
>>> from ctypes import *
#先声明1个空的
>>> class cell(Structure):
...     pass
...
#直接赋值_fields_的属性
>>> cell._fields_ = [("name", c_char_p),
...                  ("next", POINTER(cell))]
>>>
```


* Array

不同类型再乘以1个数即得到数组.
```sh
ct_int_array = c_int * 10
type(ct_int_array)
<class '_ctypes.PyCArrayType'>
```

* memmove memset

```sh
ctypes.memmove(dst, src, count)
Same as the standard C memmove library function: copies count bytes from src to dst. dst and src must be integers or ctypes instances that can be converted to pointers.

ctypes.memset(dst, c, count)
Same as the standard C memset library function: fills the memory block at address dst with count bytes of value c. dst must be an integer specifying an address, or a ctypes instance.
```

* cast:

类似c里的强制指针类型转换,将1个可转换为指针类型的object，转换为第2个参数指定的ctype pointer类型, 返回新的ctype pointer的实例.

必须是`One POINTER to another POINTER`.如Array POINTER object转Structure POINTER的object:

(理论上，可以Structure转Array,但是没有这样的接口，只有cast通过POINTER转换才能做到)


```py
length = 100
#某个Structure pointer object，转为Array pointer object.
p = cast(pointer(s), POINTER(c_char * length))
#得到Structure的原始数据
raw = p.contents.raw

arr = (c_char * len(raw))()  
arr.raw = raw

s1 = cast(pointer(arr),POINTER(Structure))
#又把bytes 恢复成了ctype的structure
s1.contents
```
相当于c里的:
```c
Structure * s = new Structure();
char** p = (char*)&s;
// (*p)相当于Array ,Array POINTER应该是**p;
Structure * s1 = (Structure*)(*p);
```

* addressof

获得数据的内存存储地址
```sh
a = c_int(100)
addressof(a)
139795135170432
hex(addressof(a))
'0x7f24975f8780'

byref(a)
<cparam 'P' (0x7f24975f8780)>
```

**神奇的几个地址**

下面3个地址分别是什么?

```sh
ct_arr_ptr =  pointer((c_int*5)(1,2,3,4,5))
# 这是ct_arr_ptr对象的地址
<__main__.LP_c_int_Array_5 object at 0x7f2496542d90>

# 这是ct_arr_ptr所指向的内容在内存中的真实地址 
hex(addressof(ct_arr_ptr.contents))
'0x7f24966101e0'
hex(addressof(ct_arr_ptr.contents))
'0x7f24966101e0'
# 这是contents对象的地址，每次都临时生成新的，但是都引用上面那个不变的内存里的东西.
ct_arr_ptr.contents
<__main__.c_int_Array_5 object at 0x7f24965ae158>
ct_arr_ptr.contents
<__main__.c_int_Array_5 object at 0x7f24975f8620>

```


* CFUNCTYPE

用来定义ctype的函数指针，指向python函数,实际是传给c语言调用的.

到这可以看到python和c的相互调用方式了:

```sh
1 python里loadlibrary方式调用c; 
2 python提供FUNCTYPE的python回调给c语言调用;
```

```sh
libc = CDLL('/lib/x86_64-linux-gnu/libc.so.6')
qsort = libc.qsort
qsort.restype
>> <class 'ctypes.c_int'>
CMPFUNCP = CFUNCTYPE(c_int,POINTER(c_int),POINTER(c_int))
type(CMPFUNCP)
>> <class '_ctypes.PyCFuncPtrType'>
def python_cmp(a,b):
    print('cmp in python')
    return a[0]-b[0]
ia = (c_int * 10)(1,3,5,7,9,2,4,6,8,10)
qsort(id,len(ia),sizeof(c_int),CMPFUNCP(python_cmp()))
ia
1 2 3 4 5 6 7 8 9 10 <__main__.c_int_Array_10 object at 0x7f6a121c26a8>
#更简单的写法是用decorator:
@CFUNCTYPE(c_int,POINTER(c_int),POINTER(c_int))
def python_cmp(a,b):
    print('%s: %s',a[0],b[0])
    return a[0]-b[0]
qsort(ia,len(ia),sizeof(c_int),python_cmp)

```
* FUNCTYPE 和PFUNCTYPE的区别?

FUNCTYPE是封装的python函数是给c语言调用的，它将不受GIL影响，纯c的.

而PFUNCTYPE封装的python函数仍然受GIL影响，这个区别还是很大的.

c希望调用的python不要卡在GIL里的话，用FUNCTYPE;如果有些GIL操作再c里不可忽略，用PFUNCTYPE.


## 实例1: ctypes + socket

1. 对端用c语言tcp socket发送 

2. python socket.recv , 得到bytes(b'xxxx') 

3. ctypes Structure

4. 转换bytes to ctypes 解析数据,用Structure按c的方式解析数据.

```py
def struct2stream(s):
    length = ctypes.sizeof(s)
    p = ctypes.cast(ctypes.pointer(s), ctypes.POINTER(ctypes.c_char * length))
    return p.contents.raw


def stream2struct(string, stype):
    if not issubclass(stype, ctypes.Structure):
        raise ValueError('Not a ctypes.Structure')
    length = ctypes.sizeof(stype)
    stream = (ctypes.c_char * length)()
    stream.raw = string
    p = ctypes.cast(stream, ctypes.POINTER(stype))
    return p.contents


class CommandHeader(ctypes.Structure):
    _pack_ = 4
    _fields_ = [
        # Size of this descriptor (in bytes)
        ('MsgCommand', ctypes.c_int),
        ('MsgParam', ctypes.c_int),
        ('unkown', ctypes.c_short),
        ('unkown1', ctypes.c_short),
        ('startx', ctypes.c_short),
        ('starty', ctypes.c_short),
        ('width', ctypes.c_short),
        ('height', ctypes.c_short),
        ('len', ctypes.c_int)
    ]

class StructConverter(object):
    def __init__(self):
        pass

    @classmethod
    def encoding(cls, raw, structs):
        """
            'encode' means raw binary stream to ctype structure. 
        """
        if raw is not None and structs is not None:
            return stream2struct(raw, structs)
        else:
            return None

    @classmethod
    def decoding(cls, data):
        """
            'decode means ctpye structure to raw binary stream
        """
        if data is not None:
            return struct2stream(data)
        else:
            return None
```

收发过程:

```py
#receive
try:
    raw = fds.recv(ctypes.sizeof(CommandHeader))
except socket.error as e:
    exit(1)
header = StructConverter.encoding(raw, CommandHeader)

#send
resp = CommandHeader()
try:
    fds.send(StructConverter.decoding(data=resp))
except socket.error as e:
    LOGGER.info('%s', e)
    return
```

## 实例2 libusb使用

参考[python libusb1](https://pypi.org/project/libusb1/)

并以自己实现的python调用libusb底层库的实现为例子:

```py
def aoa_update_point(self, action, x, y, ops=0):

    global report
    if ops == 0:
        # left = up =0
        x, y = self.axis_convert(x, y)
        real_x = x - self.ref_x
        real_y = y - self.ref_y
    else:
        real_x = x
        real_y = y

    # LOGGER.info('real point(%d %d)',real_x,real_y)

    if action == 'move' or action == 'down':
        report = Report(REPORT_ID, 1, 0, 0, 0, int(real_x), int(real_y))
    if action == 'up':
        report = Report(REPORT_ID, 0, 0, 0, 0, int(real_x), int(real_y))

    if ops == 0:
        self.set_ref(x, y)

    #transfer是1个Structurue pointer obj
    transfer = U.libusb_alloc_transfer(0)
    # contents是实体 
    transfer.contents.actual_length = sizeof(Report)
    # p_report = cast(pointer(report), c_void_p)
    transfer.contents.buffer = cast(pointer(report), c_void_p)

    # put report buffer into control_buffer
    control_buffer = create_string_buffer(sizeof(Report) + LIBUSB_CONTROL_SETUP_SIZE)

    # python级别的内存填充,memmove + addressof
    memmove(addressof(control_buffer) +
            LIBUSB_CONTROL_SETUP_SIZE, addressof(report),
            sizeof(report))

    # 可以看出这个是signed char 0x1 ---> 0x1 0x0 小端!
    # 实际调用了:
    # setup = cast(addressof(control_buffer), libusb_control_setup_p).contents
    U.libusb_fill_control_setup(
        addressof(control_buffer),
        U.LIBUSB_ENDPOINT_OUT | U.LIBUSB_REQUEST_TYPE_VENDOR,
        AndroidA0AProtocl.AOA_SEND_HID_EVENT.value[0],
        1,
        0,
        6)

    # LOGGER.info(control_buffer.raw)

    U.libusb_fill_control_transfer(
        transfer,
        self.aoa_handle,
        pointer(control_buffer),
        null_callback,
        None,
        0)

    transfer.contents.flags = U.LIBUSB_TRANSFER_FREE_BUFFER | U.LIBUSB_TRANSFER_FREE_TRANSFER

    rets: int = U.libusb_submit_transfer(transfer)
    if rets < 0:
        LOGGER.info(U.libusb_error_name(rets))
        return rets
    return rets
```
## 末了

```sh

_CData---->_CSimpleData---->(c_int,c_char,....c_long) which has value attributes
      ---->_Pointers , which has contents attributes.
      ---->Array
      ---->Structure
      ---->Union

```