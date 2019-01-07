---
layout: post
title:
modified:
categories: Tech
tags: [fastai,deep-learning]
comments: true
---

<!-- TOC -->

- [RNN模型演进](#rnn模型演进)
    - [hidden内部展开](#hidden内部展开)
        - [输入及forward](#输入及forward)
    - [折叠起来](#折叠起来)
        - [输入及forward](#输入及forward-1)
    - [hidden的输入concat](#hidden的输入concat)
    - [pytorch里的RNN](#pytorch里的rnn)
    - [Multiout Model(SeqRNN)](#multiout-modelseqrnn)
        - [用单位I矩阵初始化h](#用单位i矩阵初始化h)

<!-- /TOC -->


## RNN模型演进



### hidden内部展开


![rnn 3char simple]({{ site.url }}/assets/img/rnn_3char_simple.png)

{: .pull-right}


```py
class BrChar3Model(nn.Module):
    def __init__(self,vocab_size,n_fac,n_hidden):
        super().__init__()
        self.em = nn.Embedding(vocab_size,n_fac)
        self.l_in = nn.Linear(n_fac,n_hidden);
        self.l_hidden = nn.Linear(n_hidden,n_hidden)
        self.l_ho = nn.Linear(n_hidden,vocab_size)
    def forward(self,c1,c2,c3):
        in1 = F.relu(self.l_in(self.em(c1)))
        in2 = F.relu(self.l_in(self.em(c2)))
        in3 = F.relu(self.l_in(self.em(c3)))
        
        h = V(torch.zeros(in1.size()).cpu())
        h = F.tanh(self.l_hidden(h+in1))
        h = F.tanh(self.l_hidden(h+in2))
        h = F.tanh(self.l_hidden(h+in3))
        
        return F.log_softmax(self.l_ho(h))
```

#### 输入及forward
let n_hidde=256,fac=42,n_input=85

1个样本的forward计算过程:

输入X0 = ['g','o','o',] predict='d'.

对每个x in X0,

`1x1`--Embedding->`1x42`--input_hidden->`1x256`.

经过add hidden后，维度不变，`1x256`-hidden_output->`1x85`

最终的结果 `1x85`-softmax->`1x85`->max->`1`,如`np.argmax(to_np(res))`

即85个结果中，最可能的那个.


**3**这个维度代表的是时间t,如[x1 x2 x3]=['g','o','o']

取1个batch的数据,此时应该为输入`512x3`按上面分析，最终得到的forward result应该是`512x85`

```py
md = ColumnarModelData.from_arrays('.', [-1], np.stack([x1,x2,x3], axis=1), y, bs=512)

it = iter(md.trn_dl)
*xs,yt = next(it)
```


### 折叠起来  

即rnn里r的含义,Recurrent,对于hidden层:这一时刻的h 的输出，联同下一时刻的系统输入，作为下一时刻h的输入

![rnn 8char recurrent]({{ site.url }}/assets/img/rnn_8char_recurrent.png)


```py
class CharLoopModel(nn.Module):
    # This is an RNN!
    def __init__(self, vocab_size, n_fac):
        super().__init__()
        self.e = nn.Embedding(vocab_size, n_fac)
        self.l_in = nn.Linear(n_fac, n_hidden)
        self.l_hidden = nn.Linear(n_hidden, n_hidden)
        self.l_out = nn.Linear(n_hidden, vocab_size)
        
    def forward(self, *cs):
        bs = cs[0].size(0)
        h = V(torch.zeros(bs, n_hidden).cpu())
        for c in cs:
            inp = F.relu(self.l_in(self.e(c)))
            h = F.tanh(self.l_hidden(h+inp))
        
        return F.log_softmax(self.l_out(h), dim=-1)
```

#### 输入及forward

输入序列`I love this game`，输入前8个时刻，预测第1个时刻。
但是下一个数据是overlap的,应该是下面这个样子的矩阵:

```
['I',' ','l','o','v','e',' ','t']
[' ','l','o','v','e',' ','t','h']
['l','o','v','e',' ','t','h','i']
['o','v','e',' ','t','h','i','s']
['v','e',' ','t','h','i','s',' ']
['e',' ','t','h','i','s',' ','g']
```

除了序列的长度变了，其余同上一个分析,因此1个batch最后的输出仍然是`512x85`


### hidden的输入concat

这样效果比上面的要更好一点.在input到hidden layer时，就将h和origin input做concat了。

![rnn 8char concat recurrent]({{ site.url }}/assets/img/rnn_8char_concat_recurrent.png)

```py
class CharLoopConcatModel(nn.Module):
    def __init__(self, vocab_size, n_fac):
        super().__init__()
        self.e = nn.Embedding(vocab_size, n_fac)
        self.l_in = nn.Linear(n_fac+n_hidden, n_hidden)
        self.l_hidden = nn.Linear(n_hidden, n_hidden)
        self.l_out = nn.Linear(n_hidden, vocab_size)
        
    def forward(self, *cs):
        bs = cs[0].size(0)
        h = V(torch.zeros(bs, n_hidden).cuda())
        for c in cs:
            inp = torch.cat((h, self.e(c)), 1)
            inp = F.relu(self.l_in(inp))
            h = F.tanh(self.l_hidden(inp))
        
        return F.log_softmax(self.l_out(h), dim=-1)
```

### pytorch里的RNN

和上面的完全一致，不过用RNN做了封装.代码更简洁

```py
class CharRnn(nn.Module):
    def __init__(self, vocab_size, n_fac):
        super().__init__()
        self.e = nn.Embedding(vocab_size, n_fac)
        self.rnn = nn.RNN(n_fac, n_hidden)
        self.l_out = nn.Linear(n_hidden, vocab_size)
        
    def forward(self, *cs):
        bs = cs[0].size(0)
        h = V(torch.zeros(1, bs, n_hidden))
        inp = self.e(torch.stack(cs))
        outp,h = self.rnn(inp, h)
        
        return F.log_softmax(self.l_out(outp[-1]), dim=-1)
```

注意: 

```py
    outp,h = self.rnn(inp, h)
```

上面的模型里，实际上rnn层的每个循环可以有中间结果的输出,多少个循环，取决于forward的输入cs被stack了多少个维度. 比如:

1个batch 512,输入`8x512` (not 512x8)? outp则为`8x512x256`, h为`1x512x256`


### Multiout Model(SeqRNN)

如果把数据的输入换成non-overlap的，然后考虑每个时刻的对下一时刻的预测,(即上面提到的中间输出)

input training data

```
SUPPOS
ING that
 Truth i
s a woma
n--what 
then? Is
 there n
ot groun
```
label data
```
SUPPOSI
NG that 
Truth is
 a woman
--what t
hen? Is 
there no
t ground
```
模型为:

```py
class CharSeqRnn(nn.Module):
    def __init__(self, vocab_size, n_fac):
        super().__init__()
        self.e = nn.Embedding(vocab_size, n_fac)
        self.rnn = nn.RNN(n_fac, n_hidden)
        self.l_out = nn.Linear(n_hidden, vocab_size)
        
    def forward(self, *cs):
        bs = cs[0].size(0)
        h = V(torch.zeros(1, bs, n_hidden))
        inp = self.e(torch.stack(cs))
        outp,h = self.rnn(inp, h)
        return F.log_softmax(self.l_out(outp), dim=-1)
```
注意forward的最后一行的输出与普通RNN的区别

#### 用单位I矩阵初始化h

对于pytorch 的RNN,如此初始化系数效果好

```
m = CharSeqRnn(vocab_size, n_fac).cuda()
opt = optim.Adam(m.parameters(), 1e-2)

m.rnn.weight_hh_l0.data.copy_(torch.eye(n_hidden))
```


