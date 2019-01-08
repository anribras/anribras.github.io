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
    - [object.\_\_getitem\_\_(self,name)](#object\_\_getitem\_\_selfname)
    - [object.\_\_getattr\_\_(self, name)](#object\_\_getattr\_\_self-name)
    - [object.\_\_getattribute\_\_(self, name)](#object\_\_getattribute\_\_self-name)
- [AttributeDict的例子:](#attributedict的例子)
- [\_\_get\_\_与描述符类](#\_\_get\_\_与描述符类)

<!-- /TOC -->

## 性质


### object.\_\_getitem\_\_(self,name)

给予对象类似dict的 obj['name']的访问能力.

正常dict:
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
发生在再访问默认attribute找不到，引起AttributeError之前:
attribute的默认访问顺序:
```
instance.__dict__-->type(instance).__dict__-->baseClass.__dict -->__getattr__()-->raise AttributeError
```
`__setattr__`自然是在self.__dict__里添加内容:
```py
self.__dict__['name']=value
```
即最终调用了self.__dict__的`__setitem__`.

### object.\_\_getattribute\_\_(self, name)
属性访问都无条件地`先通过它`. 如果重写了，最好用 object.__getattribute__(self, name)


## AttributeDict的例子:

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

AttributeDict的key-val，通过attribute能访问到, 是因为上面`__getattr__`的实现里间接调用了`__getitem__`:
```py
a = AttributeDict({"name":"Bob", "year":1986})
print(a.name)
```
```
__getarrtribute__ I handle all attribute at  very first
__getattr__ if finally not find anything, come for me
__getitem__, find in dict data-structure
Bob
```

不存在的attribute，dict里也没有该数据，最终也不会有:

```py
a.what
```
```
_getarrtribute__ I handle all attribute at  very first
__getattr__ if finally not find anything, come for me
__getitem__, find in dict data-structure
None exsited key
```
增加what属性到 intance.__dict__中,调用了`__setattr__`:

```py
a.what = 666
```
```
__setattr__ , put name:value in instance.__dict__
a.__dict__
{'what': 666}
```
a['what']仅查dict的数据，不会查attribute,所以找不到:
```py
a['what']
```
```
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

'what':123 放在dict里:
```py
a['what'] = 123
```
```
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
```
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getarrtribute__ I handle all attribute at  very first
__getitem__, find in dict data-structure
123
```
属性访问,访问到a.__dict__中:
```py
a.what
```
```
__getarrtribute__ I handle all attribute at  very first
666
```

属性访问不到，但最终通过__getitem__找到了what_again的值：
```py
a['what_again'] = 999
a['what_again'] 
```
```
__getarrtribute__ I handle all attribute at  very first
__getattr__ if finally not find anything, come for me
__getitem__, find in dict data-structure
999
```
dict里的值，并没有属性what:666:
```py
a.items()
```
```sh
__getarrtribute__ I handle all attribute at  very first
dict_items([('name', 'Bob'), ('year', 1986), ('what', 123), ('what_again', 999)])
```

dict 的str和repr默认就是item():
```py
str(a)
```
```
"{'name': 'Bob', 'year': 1986, 'what': 123, 'what_again': 999}"
```

## \_\_get\_\_与描述符类

`__get__`和上面的3个不是1个概念,和是描述符有关.

什么是描述符:[python descriptors](http://martyalchin.com/2007/nov/23/python-descriptors-part-1-of-2/)

实现了下面任一函数的类，即遵循了描述符协议，是1个描述符类:
```
object.__get__(self, instance, owner)
object.__set__(self, instance, value)
object.__delete__(self, instance)
object.__set_name__(self, owner, name)
```
[why owner in get ?](https://stackoverflow.com/questions/3798835/understanding-get-and-set-and-python-descriptors)

仅定义__get__(),为non-data descriptor,让属性`read only`.python的`classmethod staticmethod`就是non-data descriptor.

定义了__set__(),__delete()__,为data descriptor,`read and write`.python的`property`就是data descriptor.

前面讲了attribute的访问顺序.当访问某个属性，而这个恰好又是1个实现了描述符协议的object时，将直接调用__get__等方法，跳过默认的属性访问顺序，相当于属性访问路由到了__get__的函数调用，并由这个函数来控制这个属性的值（也即函数的返回值,在返回前,可以做定制化的操作.

类似还有个LazyProperty,即延迟初始化的属性.
