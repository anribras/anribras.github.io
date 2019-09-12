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

- [性质](#性质)
  - [object.\_\_getitem\_\_(self,name)](#object__getitem__selfname)
  - [object.\_\_getattr\_\_(self, name)](#object__getattr__self-name)
  - [object.\_\_getattribute\_\_(self, name)](#object__getattribute__self-name)
- [AttributeDict 的例子](#AttributeDict-的例子)
- [\_\_get\_\_与描述符类](#__get__与描述符类)

<!-- /TOC -->

## 性质

### object.\_\_getitem\_\_(self,name)

给予对象类似 dict 的 obj['name']的访问能力.

正常 dict:

```py
a={"1":1,"2":2}
#__getitem__
a["1"]
#__getitem__
a["3"] =3
#__getattr__,dict报错
a.1
```

要使用`a.1`?，自然是实现`__setattr__`

### object.\_\_getattr\_\_(self, name)

发生在再访问默认 attribute 找不到，引起 AttributeError 之前:
attribute 的默认访问顺序:

```py
instance.__dict__-->type(instance).__dict__-->baseClass.__dict -->__getattr__()-->raise AttributeError
```

`__setattr__`自然是在 self.**dict**里添加内容:

```py
self.__dict__['name']=value
```

即最终调用了 self.**dict**的`__setitem__`.

### object.\_\_getattribute\_\_(self, name)

属性访问都无条件地`先通过它`. 如果重写了，最好用 object.**getattribute**(self, name)

## AttributeDict 的例子

```py
class AttributeDict(dict):
    def __setitem__(self,name,value):
        print('__setitem__,')
        return super().__setitem__(name,value)

    def __getitem__(self,name):
        print('__getitem__, find in dict data-structure')
        return super().__getitem__(name)

    def __setattr__(self,name,value):
        print('__setattr__ , put name:value in instance.__dict__')
        return super().__setattr__(name,value)

    def __getattr__(self,name):
        print('__getattr__ if finally not find anything, come for me')
        try:
            # goto self.__getitem__
            value = self[name]
        except KeyError:
            print('None exsited key')
            return None
        if isinstance(value,dict):
            value = AttributeDict(value)
        return value
    def __getattribute__(self,name):
        print('__getarrtribute__ I handle all attribute at  very first')
        #通过object显示调用__getattr__
        # 如果不显示使用，除非属性找不到，否则不再调用__getattr__
        return object.__getattribute__(self,name)
```

AttributeDict 的 key-val，通过 attribute 能访问到, 是因为上面`__getattr__`的实现里间接调用了`__getitem__`:

```py
a = AttributeDict({"name":"Bob", "year":1986})
print(a.name)
```

```py
__getarrtribute__ I handle all attribute at  very first
__getattr__ if finally not find anything, come for me
__getitem__, find in dict data-structure
Bob
```

不存在的 attribute，dict 里也没有该数据，最终也不会有:

```py
a.what
```

```py
_getarrtribute__ I handle all attribute at  very first
__getattr__ if finally not find anything, come for me
__getitem__, find in dict data-structure
None exsited key
```

增加 what 属性到 intance.**dict**中,调用了`__setattr__`:

```py
a.what = 666
```

```py
__setattr__ , put name:value in instance.__dict__
a.__dict__
{'what': 666}
```

a['what']仅查 dict 的数据，不会查 attribute,所以找不到:

```py
a['what']
```

```y
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getitem__, find in dict data-structure
---------------------------------------------------------------------------
KeyError                                  Traceback (most recent call last)
<ipython-input-639-4aceedcebab4> in <module>
----> 1 a['what']

<ipython-input-632-361f168d5d96> in __getitem__(self, name)
      6     def __getitem__(self,name):
      7         print('__getitem__, find in dict data-structure')
----> 8         return super().__getitem__(name)
      9
     10     def __setattr__(self,name,value):

KeyError: 'what'

```

'what':123 放在 dict 里:

```py
a['what'] = 123
```

```py
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__setitem__,
```

正常字典访问:

```py
a['what']
```

```py
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getitem__, find in dict data-structure
123
```

属性访问,访问到 a.**dict**中:

```py
a.what
```

```py
__getarrtribute__ I handle all attribute at  very first
666
```

属性访问不到，但最终通过**getitem**找到了 what_again 的值：

```py
a['what_again'] = 999
a['what_again']
```

```py
__getarrtribute__ I handle all attribute at  very first
__getattr__ if finally not find anything, come for me
__getitem__, find in dict data-structure
999
```

dict 里的值，并没有属性 what:666:

```py
a.items()
```

```sh
__getarrtribute__ I handle all attribute at  very first
dict_items([('name', 'Bob'), ('year', 1986), ('what', 123), ('what_again', 999)])
```

dict 的 str 和 repr 默认就是 item():

```py
str(a)
```

```py
"{'name': 'Bob', 'year': 1986, 'what': 123, 'what_again': 999}"
```

## \_\_get\_\_与描述符类

`__get__`和上面的 3 个不是 1 个概念,和是描述符有关.

什么是描述符:[python descriptors](http://martyalchin.com/2007/nov/23/python-descriptors-part-1-of-2/)

实现了下面任一函数的类，即遵循了描述符协议，是 1 个描述符类:

```py
object.__get__(self, instance, owner)
object.__set__(self, instance, value)
object.__delete__(self, instance)
object.__set_name__(self, owner, name)
```

[why owner in get ?](https://stackoverflow.com/questions/3798835/understanding-get-and-set-and-python-descriptors)

仅定义**get**(),为 non-data descriptor,让属性`read only`.python 的`classmethod staticmethod`就是 non-data descriptor.

定义了**set**(),**delete()**,为 data descriptor,`read and write`.python 的`property`就是 data descriptor.

前面讲了 attribute 的访问顺序.当访问某个属性，而这个恰好又是 1 个实现了描述符协议的 object 时，将直接调用**get**等方法，跳过默认的属性访问顺序，相当于属性访问路由到了**get**的函数调用，并由这个函数来控制这个属性的值（也即函数的返回值,在返回前,可以做定制化的操作.

类似还有个 LazyProperty,即延迟初始化的属性.
