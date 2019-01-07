---
layout: post
title:
modified:
categories: Tech
 
tags: [fastai,deep-learning]

  
comments: true
---

<!-- TOC -->

- [cnn](#cnn)
- [multi-label classification](#multi-label-classification)
    - [data augumentation](#data-augumentation)

<!-- /TOC -->



### cnn

这个不用多讲了,倒是要看看pytorch是怎么实现的就好

### multi-label classification

```
In multi-label classification each sample can belong to one or more clases.
```

比如y可能是:

```
[1 0 1 0 ....1]
```  

最终的预测从高到低的选取多个label即可。

#### data augumentation

不同的image分类问题，适应不同的augumentaion方法，像dog and cat就不要旋转180，而planet这种卫星图片就可以.

```py
def get_data(sz):
    tfms = tfms_from_model(f_model, sz, aug_tfms=transforms_top_down, max_zoom=1.05)
    return ImageClassifierData.from_csv(PATH, 'train-jpg', label_csv, tfms=tfms,
                    suffix='.jpg', val_idxs=val_idxs, test_name='test-jpg')
```

fastai全做好了，会根据train set的label自动区分.

