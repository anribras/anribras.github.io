---
layout: post
title:
modified:
categories: Tech
 
tags: [libevent,backend]

  
comments: true
lang: zh
---
<!-- TOC -->

- [simple list](#simple-list)
- [list](#list)
- [simple queue](#simple-queue)
- [tail queue](#tail-queue)
- [circur queue](#circur-queue)
- [evmap](#evmap)
- [min_heap](#min_heap)

<!-- /TOC -->

数据结构的灵魂是`在何时的场合选择最高效的结构`。
数据结构往往要实现对数据的`增删查改`,高效的数据结构就体现在这些操作的时间和空间复杂度上。
不同数据结构特性不一样，适用的场景也不一样。
libevent中大量使用的结构包括:
* 哈希表 evmap
* 最小堆 min_heap
* 链表　list
* 队列 queue

double list链表是个很常用的数据结构，libevent中使用了更高效的方式，也是bsd unix源码中使用的方式。从`simple list`到`list`,从`simple queue`到`tail queue`的演进，可以看出是如何逐步实现高效的数据结构的。　

### simple list
就是最简单的单向forward-list。提供操作有:
* SLIST_INSERT_AFTER
* SLIST_INSERT_HEAD
* SLIST_REMOVE_HEAD

header是个单指针
```
#define SLIST_HEAD(name, type)						\
struct name {								\
	struct type *slh_first;	/* first element */			\
}
```

### list

比与simple-list比，提供了O(1)的insert和remove,以及insert_before.
操作有:
* LIST_INSERT_AFTER o(1)
* LIST_INSERT_BEFORE o(1)
* LIST_INSERT_HEAD o(1)
* LIST_REMOVE o(1)
* LIST_REPLACE o(1)

header是个单指针。
```
#define LIST_HEAD(name, type)						\
struct name {								\
	struct type *lh_first;	/* first element */			\
}
```
entry如下，用了二级指针，不是通常的*prev
```
#define LIST_ENTRY(type)						\
struct {								\
	struct type *le_next;	/* next element */			\
	struct type **le_prev;	/* address of previous next element */	\
}

//通常意义的理解
#define LIST_ENTRY(type)						\
struct {								\
	struct type *le_next;	/* next element */			\
	struct type *le_prev;	/* prev element */	\
}
```
为什么list的entry用一个指向前一个结点的next指针的二级指针？
先说结论:
>性能更好 操作更丰富
>insert remove达到O(1)的效果.

这一篇[Stackoverflow][ref1]解释了数据结构为何如此重视性能。
虽然相比纯粹的双向链表，只提升了微弱的性能，但在大型系统和海量数据面前里，微弱的性能也是很重要的。在链表操作里，如果有if判断是可以优化的对象，汇编的指令越少越好。(跳转，压栈etc)

再仔细分析。如果么有该二级指针，就是普通的单向链表。其实`**prev`二级指针带来最大的好处是允许了**前插**,相当与记住了前驱。 链表的插入总是O(1)时间复杂度的(查找是O(n))。
```
insert_after(entry* list_elem, entry* elem, type val);
insert_head(head_entry* head, entry* elem, type val);
insert_before(entry* list_elem, entry* elem, type val);
```
下面的图解释了为何允许了前插：
![]( http://osvo72tet.bkt.clouddn.com/list.png)
```
list_insert_before(entry* list_elem, entry* elem, type val)
{
    elem->prev = list_elem->prev;
	elem->next = list_elem;
    *(list_elem->prev) = elem;
    list_elem->prev = &(elem)->next;
}
// insert_after是forward_list就可以做到的
list_insert_after(entry* list_elem, entry* elem, type val)
{
	if( ((elem)->next= (list_elem)->next) != NULL ) {
    	(elem_list)->next->prev = &(elem)->next;
    }
    (list_elem)->next =  elem;
    elem->prev =  &(list_elem)->next;
}
```
remove类似insert_before,另外就是性能了，不需要多一个head函数型参，函数压栈也是操作。
```
//正常move需要head,需要while
list_move(header* head, entry* pos)
{
	entry = head->first;
	while(entry->next != pos) {
    	...
        entry =  entry->next;
    }
    entry->next=  pos->next;
    free(pos)
}
//二级指针o(1)级别的move
list_move(entry* elem)
{
	if( (*(elem->prev) = elem->next) ) = NULL)
    	*(elem->prev) = NULL;
      
     free(pos);
}
```

### simple queue
还是基于链表的，允许头插，后插，尾插，但是只允许头删。
区别英文`privious and last! 后者是最后一个，前者是上一个`
HEAD除了单指针，还有一个指针随时指向链表最后一个成员的next指针的地址,才能按O(1)实现关键的**尾插。**
这种结构也决定了不可能尾删，只能头删.
```
#define SIMPLEQ_INSERT_TAIL(head, elm, field) do {			\
	(elm)->field.sqe_next = NULL;					\
	*(head)->sqh_last = (elm);					\
	(head)->sqh_last = &(elm)->field.sqe_next;			\
} while (0)
```
*(head)->sqh_last 等于last elem next 也即链接了上一个elem和新插入的elem.

![]( http://osvo72tet.bkt.clouddn.com/simple_queue.png)
### tail queue
这么看综合下list和simple queue的实现，应该会比较强大。就是tail queue了！
head和entry:
```
#define TAILQ_HEAD(name, type)						\
struct name {								\
	struct type *tqh_first;	/* first element */			\
	struct type **tqh_last;	/* addr of last next element */		\
}

#define TAILQ_ENTRY(type)						\
struct {								\
	struct type *tqe_next;	/* next element */			\
	struct type **tqe_prev;	/* address of previous next element */	\
}
```
可以推测操作是全支持了：
* TAILQ_INSERT_HEAD
* TAILQ_INSERT_TAIL
* TAILQ_INSERT_AFTER
* TAILQ_INSERT_BEFORE
* TAILQ_REMOVE
* TAILQ_REPLACE...

这个结构既可以做stack也可以做queue.
stack时，头插，头删；
queue时，头插，尾删.
可以推测stl里queue/stack和dequeue的实现也是类似。

### circur queue
看下header和entry:
```
#define CIRCLEQ_HEAD(name, type)					\
struct name {								\
	struct type *cqh_first;		/* first element */		\
	struct type *cqh_last;		/* last element */		\
}

#define CIRCLEQ_ENTRY(type)						\
struct {								\
	struct type *cqe_next;		/* next element */		\
	struct type *cqe_prev;		/* previous element */		\
}

就是循环双链表实现,tail queue的功能它全部可以实现，可以预见的是其性能不如tail queue。
```

### evmap
libevent中，需要将大量的监听事件event进行归类存放，比如一个文件描述符fd可能对应多个监听事件，对大量的事件event采用监听的所采用的数据结构是event_io_map，其实现通过哈希表。不同event也有可能在同一个bucket里。
既然是哈希结构，从全局上必然一个是键key-value，libevent的哈希结构中，主要实现的是文件描述符fd（key）到该文件描述符fd所关联的事件event（value）之间的映射。
http://www.cnblogs.com/yangang92/p/5857096.html

linux 下并没有用evmap,而是event_signal_map,应该是个链表数组
![]( http://osvo72tet.bkt.clouddn.com/20170809-114233.png)

### min_heap
就是传统的最小堆，推排序,很快。
注意排序比较的对象是event->timerout的时间结构。

`int min_heap_reserve(min_heap_t* s, unsigned n)`
用来分配元素空间，元素是event*,即指向event的地址.

`void min_heap_shift_up_(min_heap_t* s, unsigned hole_index, struct event* e)` 
从index开始shift_up快速插入，root node 永远是mini.

`void min_heap_shift_down_(min_heap_t* s, unsigned hole_index, struct event* e)`
从index_shiftdown,取mini后，把尾部值放到index,用shiftdown重排;
`struct event* min_heap_pop(min_heap_t* s)` 取mini值
`int min_heap_push(min_heap_t* s, struct event* e)` 快速插入

[ref1]: (https://stackoverflow.com/questions/3058592/use-of-double-pointer-in-linux-kernel-hash-list-implementation)

