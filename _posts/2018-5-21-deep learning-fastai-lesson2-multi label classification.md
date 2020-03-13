---
layout: post
title:
modified:
categories: Tech

tags: [fastai, deep-learning]

comments: true
---

<!-- TOC -->

- [cnn](#cnn)
- [multi-label classification](#multi-label-classification)
  - [data augumentation](#data-augumentation)

<!-- /TOC -->

### cnn

这个不用多讲了,倒是要看看 pytorch 是怎么实现的就好

### multi-label classification

```
In multi-label classification each sample can belong to one or more clases.
```

比如 y 可能是:

```
[1 0 1 0 ....1]
```

最终的预测从高到低的选取多个 label 即可。

#### data augumentation

不同的 image 分类问题，适应不同的 augumentaion 方法，像 dog and cat 就不要旋转 180，而 planet 这种卫星图片就可以.

```py
def get_data(sz):
    tfms = tfms_from_model(f_model, sz, aug_tfms=transforms_top_down, max_zoom=1.05)
    return ImageClassifierData.from_csv(PATH, 'train-jpg', label_csv, tfms=tfms,
                    suffix='.jpg', val_idxs=val_idxs, test_name='test-jpg')
```

fastai 全做好了，会根据 train set 的 label 自动区分.
