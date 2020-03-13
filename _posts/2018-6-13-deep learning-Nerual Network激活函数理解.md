---
layout: post
title:
modified:
categories: Tech

tags: [deep-learning]

comments: true
---

<!-- TOC -->

- [kinds of](#kinds-of)
- [output layer + metric cost](#output-layer--metric-cost)
- [why cross-entry](#why-cross-entry)
- [why softmax](#why-softmax)
- [cross-entropy 与 negative likelihood log 损失的区别](#cross-entropy-与-negative-likelihood-log-损失的区别)
- [regression problem](#regression-problem)

<!-- /TOC -->

### kinds of

[kinds of activations](https://towardsdatascience.com/activation-functions-neural-networks-1cbd9f8d91d6)

### output layer + metric cost

```sh
Output: sigmoid, softmax, linear functions(回归问题,实际就是不处理),tanh
Metric cost: mse,cross-entry, log-likelihood cost
```

tanh 和 sigmoid 很像，y 取值在[-1,1]之间.

下面说的主要是`output layer + metric cost`的组合.

### why cross-entry

[the_cross-entropy_cost_function](http://neuralnetworksanddeeplearning.com/chap3.html#the_cross-entropy_cost_function)

[交叉熵理解](https://www.zhihu.com/question/41252833)

在训练过程中，当前(w,b)导致某个样本完全正确/错误分类时，希望梯度变化更快，跳出当前的局部性，去寻找最优解.
`sigmoid + mse`此时计算的(w,b)的偏导很小(sigmoid 的导数特性决定),间接上使得(w,b)的更新也很慢，甚至导致训练失败。其实就是`梯度消失`的问题.

`sigmoid + cross-entry`,正好可以解决这个问题

### why softmax

[some disscusion](https://www.zhihu.com/question/41252833)

从交叉熵的定义出发，要求 output 的值是一个概率分布，才能使用 cross-entry,对于 2 元分类，自然是满足的(p,1-p)肯定是概率分布,多元分类怎么办?

Ng 的教程里，使用的是`one-vs-all`的思想，把某类为 p，其他类加起来为(1-p),把多元分类变成若干个 2 元分类的问题.

```sh
p(a), p(not a)
p(b), p(not b)
p(c), p(not c)
final  = max(p(a),p(b),p(c))
```

但其实 p(a), p(b) , p(c)的概率加起来并不是 1, 因此当很接近的时候, 很难说到底更预测为 a 还是 b.

`softmax`可以将多元 output 后的结果变成概率分布(sum(p_i)=1).相比`sigmoid`,可以更好的解释 output 的结果

`softmax + negative likelihood log cost`作为 metric cost，得到的偏导情况和`sigmoid`+`cross_entry`类似。

likelihood cost 的计算:

$$C=-ln(a_{j}^{L})$$

其中$$a_{j}^{L}$$是 softmax 后，结果为第 j 类(最大概率)的值.越接近 1,显然 C 是越接近 0 的，越接近 0,代价函数值越高。这是符合代价函数的基本定义的

实际应用中，这个是用的最多的，因为计算相对简洁.并且概率的方式更好解释

### cross-entropy 与 negative likelihood log 损失的区别

[cross-entropy-or-log-likelihood-in-output-layer](https://stats.stackexchange.com/questions/198038/cross-entropy-or-log-likelihood-in-output-layer)

[and this one](https://stats.stackexchange.com/questions/223799/different-definitions-of-the-cross-entropy-loss-function/224491#224491)

### regression problem

不需要额外的 activation,然后用 mse 是最常见的做法，即`linear function + mse`

当然用`sigmoid + mse`也是可行的

这个作者很学术，教人如何思考问题。如他说的，神经网络里， 有很多`可意会不可言传的`的知识点,这些点尽管实践中有好的效果，但并不是确定的结论，值得进一步深入研究。如`drop out为何有效?`,`tanh为何在有些场合效果比sigmoid好?Relu为何又好一些?`

不要刻意追求理论的深度，这些启发式的结论，能指导深层次的思考.
