---
layout: post
title:
modified:
categories: Tech

tags: [linux]

comments: true
---

<!-- TOC -->

- [装载](#装载)
  - [两个视图](#两个视图)
  - [其他 VMA](#其他-VMA)
- [段地址对齐](#段地址对齐)

<!-- /TOC -->

## 装载

装载到哪里？内存，进程上看，是映射到了虚拟进程空间.

装载谁？ 程序运行的实体代码，数据，来自 elf，共享库，OS 等.

怎么装载?利用`程序局部性`原理: 运行某段程序时，很可能也要运行最近的 1 个代码块，于是先提前装载到内存.

内存肯定不够用，不过通过`页映射`，需要时(缺页异常)，再从磁盘 load 到内存，做替换.

管理这个装载过程的，就是 os 的存储管理器(MMU).

具体过程:

1. 建立虚拟进程空间，
2. 读取 elf 文件头，建立好 elf 与虚拟空间映射关系,(为后续缺页装载准备).虚拟进程空间有个区域叫 VMA, 对应映射的是 elf 的.text 段
3. cpu pc 跳转到入口，开始运行.
4. 发现空页，缺页异常，正式装载磁盘页到映射好的物理内存，虚拟进程空间也看的到

![Screenshot from 2019-01-22 09-44-55-afff0265-55a1-4c88-bda8-fe795b025193](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2009-44-55-afff0265-55a1-4c88-bda8-fe795b025193.png)

### 两个视图

链接的角度,elf 的段按`Section`划分，(链接试图), 从装载的角度,elf 按`Segment`(执行试图), 后者为了装载方便，不浪费内存，可能进行了段的合并优化等.

![Screenshot from 2019-01-22 09-55-32-6d3b5f39-efa0-48e0-8a74-f340ae2ed1e1](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2009-55-32-6d3b5f39-efa0-48e0-8a74-f340ae2ed1e1.png)

这就是 elf 中`program header`的由来，描述 elf 该如何被 os 装载到进程空间.

```sh
➜  compile-link-lib readelf -l ab

Elf file type is DYN (Shared object file)
Entry point 0x5a0
There are 9 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000001f8 0x00000000000001f8  R      0x8
  INTERP         0x0000000000000238 0x0000000000000238 0x0000000000000238
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000950 0x0000000000000950  R E    0x200000
  LOAD           0x0000000000000db0 0x0000000000200db0 0x0000000000200db0
                 0x0000000000000264 0x0000000000000268  RW     0x200000
  DYNAMIC        0x0000000000000dc0 0x0000000000200dc0 0x0000000000200dc0
                 0x00000000000001f0 0x00000000000001f0  RW     0x8
  NOTE           0x0000000000000254 0x0000000000000254 0x0000000000000254
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_EH_FRAME   0x00000000000007e0 0x00000000000007e0 0x00000000000007e0
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x0000000000000db0 0x0000000000200db0 0x0000000000200db0
                 0x0000000000000250 0x0000000000000250  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr
    .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .plt.got
    .text .fini .rodata .eh_frame_hdr .eh_frame
   03     .init_array .fini_array .dynamic .got .data .bss
   04     .dynamic
   05     .note.ABI-tag .note.gnu.build-id
   06     .eh_frame_hdr
   07
   08     .init_array .fini_array .dynamic .got
```

Segment 的内容:

```sh
segment类型有LOAD,DYNAMIC,INTERP,GNU_XXX等;
Offset是segment在文件里的偏移;
Virtual Addr就是映射到的虚拟进程空间.
Phy Addr是物理装载地址，一般与上面的一致.
Flag: R,W,X
Align: 对齐属性.(k bytes)
    - 对齐是为了页处理的方便,但是可能带来空间浪费.
```

### 其他 VMA

VMA 不止有映射代码的，还有映射堆，栈，数据，内核等.

![Screenshot from 2019-01-22 10-03-03-29ddbddf-966c-4ef9-af51-f2a14d6000ec](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2010-03-03-29ddbddf-966c-4ef9-af51-f2a14d6000ec.png)

程序实现的视图,还是来自[这篇](http://www.cnblogs.com/huxiao-tee/p/4660352.html).

![Screenshot from 2019-01-22 10-04-02-c4cede78-b535-4ffd-a722-0758a160a1ce](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2010-04-02-c4cede78-b535-4ffd-a722-0758a160a1ce.png)

## 段地址对齐

1 个例子，3 个段的 Segment 情况:

![Screenshot from 2019-01-22 10-48-04-e31eb07a-3dfe-495e-bcae-c79fb5ea330c](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2010-48-04-e31eb07a-3dfe-495e-bcae-c79fb5ea330c.png)

合并前，映射了 5 个物理内存页:

![Screenshot from 2019-01-22 10-46-20-bda73b46-9e24-45b8-8fc6-d45b822e5fd6](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2010-46-20-bda73b46-9e24-45b8-8fc6-d45b822e5fd6.png)

合并后，仅映射 3 个物理内存页,(但是虚拟空间多了映射):

![Screenshot from 2019-01-22 10-47-26-97e797fe-8dbc-4bec-986d-e07887275984](https://images-1257933000.cos.ap-chengdu.myqcloud.com/Screenshot%20from%202019-01-22%2010-47-26-97e797fe-8dbc-4bec-986d-e07887275984.png)
