---
layout: post
title:
modified:
categories: Tech

tags: [deep-learning]

comments: true
---

<!-- TOC -->

- [参考](#参考)
- [基本结构](#基本结构)
- [4 大特点](#4-大特点)
- [关于 Convlayer 的 depth](#关于-Convlayer-的-depth)
- [1x1xk 的卷积核能干什么](#1x1xk-的卷积核能干什么)
- [CNN 如何 back propogation](#CNN-如何-back-propogation)
- [FullConnection layer 看做是 卷积层](#FullConnection-layer-看做是-卷积层)
- [合理的 CNN 参数](#合理的-CNN-参数)

<!-- /TOC -->

### 参考

[Nerual Networks and Deep learning ch6](http://neuralnetworksanddeeplearning.com/chap6.html)

[cs231 cnn](http://cs231n.github.io/convolutional-networks/)

[ConvNetJS](https://cs.stanford.edu/people/karpathy/convnetjs/demo/cifar10.html)

### 基本结构

ConvLayer ,Relu, Pooling, FullConnection ,Output

不是 Convlayer 后一定就是 Rele+Pooling，很多结构可以多做几次 Conv 后，再 Polling，比如 Conv+Relu + Conv+Relu + Pooling

Pooling 下采样.最新的一些研究建议不要 pooling 了.

```sh
INPUT -> [[CONV -> RELU]*N -> POOL?]*M -> [FC -> RELU]*K -> FC //*N表示重复N次
```

最新的 googlenet,resnet 已经有更多的变化，不再是上面这种简单的线性结构了

### 4 大特点

局部连接／权值共享／池化操作／多层次结构

### 关于 Convlayer 的 depth

depth=1 `ConvLayer`好理解.

相对全连接，卷积层是针对局部进行计算，没有那么多 weights，并且进行窗口滑动(步长的概念),针对输入的每个局部，weights 都是一样的,即`Parameter sharing scheme`

但是 depth > 1 时，看了不少资料都没有讲清楚。按 cs231 里的来定义:

```sh
输入维度:W1 x H1 x input depth = (11x11x9)
卷积层参数:
spatial extent F =3;
补0宽度Padding:P = 0;
步长Stride = 2;
number of filters K=2，也可以理解为feature map的层数

计算feature map层输出维度:
W2=(W1 - F + 2P)/S +1 =  5
H2=(H1 - F + 2P)/S +1 =  5
depth = K ; //重点在这里,为何为K，　为何不是D1xK?
```

```sh
 Each neuron in the convolutional layer is connected only to a local region in the input volume spatially, but to the full depth
```

如果把 filter 看做一个 neuron,它的 depth 总是它的输入一致，上面的例子，
一个 conv neuron 有`3x3x3`个 weights + `1`个 bias.经过一个 neuron 的 map 后得到的`feature map`为`5x5x1`, K 个 filter 得到自然是`5x5xk`的`feature map`,depth=K

注意 2 个参数.

计算卷积层参数个数即:`FxFxinput depth + 1`,

计算输出层个数即:`W2xH2xK`

### 1x1xk 的卷积核能干什么

假设输入层的 input depth=n,按上面的理解，1x1(spaitial extent=1)xk 的卷积层显然是可以不改变输入层 width,height 的情况下(一般,stride=1,padding=0),进行升维(k > n)或者降维(k < n)处理.

这个维度，也可以理解为卷积层输出`feature map`的个数。

当 n 比较大时，可能卷积层参数较多，为计算方便，可以先用 1x1xk 的卷积层降维;

当 n 比较小，作为输出层时，为了增强表达，可以用 1x1xk 的卷积层增强表达~

当 k=1 时，可以看做对输入 n 层的信息加权融合,还是有 n 个 weights(and 1 bias)的; 比如`32x32x3`会变成`32x32x1`

在 VGG,Google net,Resnet 里，大量用到了以上技术。

![2018-06-28-11-23-07](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-06-28-11-23-07.png)

[reference](https://zhuanlan.zhihu.com/p/31319322)

### CNN 如何 back propogation

关键就是 Convention layer 和 Pooling layer 如何 bp

### FullConnection layer 看做是 卷积层

假设输入层`7x7x512`,full connecton layer 为`4096`. 可以把 full connection layer 看做是`F=7,P=0,S=1,K=4096.`的 Conv layer 的针对输入(7x7x512)的输出(1x1x4096)

AlexNet 里就这个干的,一共有 3 个 FC,全都转换成 Convlayer:

1st FC: input `7x7x512` output `1x1x4096`

2nd FC: input `1x1x4096` output `1x1x4096`

维度不变

3rd FC: input `1x1x4096` output `1x1x1000`

即用之前的降维思想，用(1x1x1000)的卷积层得到(1x1x1000)的 output layer

好处是什么? 可以更大的图片上`滑动`的使用已有的 ConvNet,计算方便!

```sh
It turns out that this conversion allows us to “slide” the original ConvNet very efficiently across many spatial positions in a larger image, in a single forward pass.
```

假设新的图片为`384x384`(12x12x512)可以看做是`6x6`个`224x224`(即 7x7x512),即以 32 为步长。

同样还是经过上面的转换过的 FC layer 得到的是`6x6x1000`的 output layer,其含义就是 384x384 里 36 个`224x224`子图像经过 FC layer 的结果，很直观。在目标检测这类算法里，更方便。

另外举了个例子的方便:

```sh
it is common to resize an image to make it bigger, use a converted ConvNet to evaluate the class scores at many spatial positions and then average the class scores.
```

### 合理的 CNN 参数

网络应该保持 image size ，也就是 Convlayer 不要改变 input 的大小，可选择的参数组合如:

输入 image 应该能被 2 整除;8 的倍数最好咯

`Padding=1,F=3,` or `Padding=2,F=5,`

`Stride=1`

`Max pooling F=2`

考虑内存,中间变量的大小直接和网络选取的这些参数有关.

看 VggNet 结构:

```sh
INPUT: [224x224x3]        memory:  224*224*3=150K   weights: 0
CONV3-64: [224x224x64]  memory:  224*224*64=3.2M   weights: (3*3*3)*64 = 1,728
CONV3-64: [224x224x64]  memory:  224*224*64=3.2M   weights: (3*3*64)*64 = 36,864
POOL2: [112x112x64]  memory:  112*112*64=800K   weights: 0
CONV3-128: [112x112x128]  memory:  112*112*128=1.6M   weights: (3*3*64)*128 = 73,728
CONV3-128: [112x112x128]  memory:  112*112*128=1.6M   weights: (3*3*128)*128 = 147,456
POOL2: [56x56x128]  memory:  56*56*128=400K   weights: 0
CONV3-256: [56x56x256]  memory:  56*56*256=800K   weights: (3*3*128)*256 = 294,912
CONV3-256: [56x56x256]  memory:  56*56*256=800K   weights: (3*3*256)*256 = 589,824
CONV3-256: [56x56x256]  memory:  56*56*256=800K   weights: (3*3*256)*256 = 589,824
POOL2: [28x28x256]  memory:  28*28*256=200K   weights: 0
CONV3-512: [28x28x512]  memory:  28*28*512=400K   weights: (3*3*256)*512 = 1,179,648
CONV3-512: [28x28x512]  memory:  28*28*512=400K   weights: (3*3*512)*512 = 2,359,296
CONV3-512: [28x28x512]  memory:  28*28*512=400K   weights: (3*3*512)*512 = 2,359,296
POOL2: [14x14x512]  memory:  14*14*512=100K   weights: 0
CONV3-512: [14x14x512]  memory:  14*14*512=100K   weights: (3*3*512)*512 = 2,359,296
CONV3-512: [14x14x512]  memory:  14*14*512=100K   weights: (3*3*512)*512 = 2,359,296
CONV3-512: [14x14x512]  memory:  14*14*512=100K   weights: (3*3*512)*512 = 2,359,296
POOL2: [7x7x512]  memory:  7*7*512=25K  weights: 0
FC: [1x1x4096]  memory:  4096  weights: 7*7*512*4096 = 102,760,448
FC: [1x1x4096]  memory:  4096  weights: 4096*4096 = 16,777,216
FC: [1x1x1000]  memory:  1000 weights: 4096*1000 = 4,096,000

TOTAL memory: 24M * 4 bytes ~= 93MB / image (only forward! ~*2 for bwd)
TOTAL params: 138M parameters
```

实际计算过程，bp,各种 activations,中间变量，gradient descent 算法，会让内存消耗在 GB 以上...
