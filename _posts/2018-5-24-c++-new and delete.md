---
layout: post
title:
modified:
categories: Tech
tags: [cplusplus]
comments: true
---

<!-- TOC -->

- [要点](#要点)
- [set_new_handler](#set_new_handler)

<!-- /TOC -->

## 要点

## set_new_handler

std::new_handler 给机会处理 new 的异常.但是是个 global 的.

下面实现的针对某个类的.

要点:

```cpp
1. 重载new[];
2. 重载的new，其实还是::operator new, 不过是自定义下new_handler;
3. new完毕　恢复global new_handler;
```

```cpp
#include <iostream>
#include <new>
#include <exception>

/** @brief NewHandlerHoder  */
class NewHandlerHoder {
public:
	NewHandlerHoder(const std::new_handler h): holder(h) {}
	~NewHandlerHoder(){
		//恢复holder里的callback
		std::set_new_handler(holder);
	}
	//Forbid copy, =
	NewHandlerHoder(const NewHandlerHoder & ) =delete;
	NewHandlerHoder & operator=(const NewHandlerHoder &)=delete;
private:
	std::new_handler holder;
};


/** @brief NewHander  */
class NewHander {
public:
	static std::new_handler set_new_handler( std::new_handler n) {
		auto old = cur;
		cur = n;
		return old;
	}
	static void* operator new[](std::size_t size) {
		std::cout << "my new"  << std::endl;
		// 如果cur设置了 设置为cur
		//保存旧的global new_handler
		NewHandlerHoder h(std::set_new_handler(cur));
		return ::operator new(size);
		//结束后通过RAII的方式恢复old global new_handler
	}
private:
	static std::new_handler cur;
	//目的即使重写operator new?
};

//初始化
std::new_handler NewHander::cur = nullptr;


//任何继承NewHandler的类，都具备这样的能力
class A : public NewHander {
public:
private:
	int m[1000];
};

void new_error()
{
	std::cout << "new failed" << std::endl;
	//抛自定义的异常
	throw std::string("mynew_bad_alloc");
	//std::exit(1);
	//
}

void item49()
{
	//std::set_new_handler(new_error);
	try {
		int * p = new int[100000000000000l];
	} catch(std::string & e) {
		std::cout << e << std::endl;
	}
}

int main()
{
	//item49();

	// new handler just for A;
	A::set_new_handler(new_error);

	//new[] 已经重写了
	try{
		A* arr =  new A[1000000L];
	} catch (const std::string & e) {
		std::cout << e << std::endl;
	}
}
```

output:

```sh
my new
new Failed
out
```

上面代码还要问题，如果有 B,也继承自 NewHanlder

```cpp
class B : public NewHander {
private:
	int m[1000];
};

A::set_new_handler(new_error);
try{
	B* arr =  new B[1000000L];
} catch (const std::string & e) {
	std::cout << e << std::endl;
}
```

通过 A 设计的 handler,对 B 也起作用了!
因为 NewHander 不是实体各异的，也就是共享了 staitc 1 个 cur;
解决之道自然是实现 `template<typename T> Newhander`,即模板类，这样编译多态，不同的具体化的模板类，都有自己的 static cur.

```cpp
class A : public NewHandler<A> {...}
class B : public NewHandler<B> {...}
```

这种继承方法确实怪异，叫`curiously recurring template pattern`,CRTP.
