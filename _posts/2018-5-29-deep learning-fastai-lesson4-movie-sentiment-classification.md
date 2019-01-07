---
layout: post
title:
modified:
categories: Tech
 
tags: [fastai,deep-learning]

  
comments: true
---
<!-- TOC -->

- [模型](#模型)

<!-- /TOC -->


# 模型

问题:通过imdb的电影英文评论,区分是positive还是negative.

这个数据集本就经过了各种特殊处理，如拿掉中立评论，对评论进行`bag of words`建模等等

train test各自分文件夹，里面有pos negative子目录，train/negative 下的样本如下:
```
0_0.txt : 0表示id为0的电影，0表示评分为0.
0_1.txt : id=0的电影可以有多个低的评分.

0_0.txt的评价:
I admit, the great majority of films released before say 1933 are just not for me. Of the dozen or so "major" silents I have viewed, one I loved (The Crowd), and two were very good (The Last Command and City Lights, that latter Chaplin circa 1931).<br /><br />So I was apprehensive about this one, and humor is often difficult to appreciate (uh, enjoy) decades later. I did like the lead actors, but thought little of the film.<br /><br />One intriguing sequence. Early on, the guys are supposed to get "de-loused" and for about three minutes, fully dressed, do some schtick. In the background, perhaps three dozen men pass by, all naked, white and black (WWI ?), and for most, their butts, part or full backside, are shown. Was this an early variation of beefcake courtesy of Howard Hughes?

```
id=9, 其评论的bag of words模型,0:9表示词库里的单词0出现了9次:
```
9 0:9 1:1 2:4 3:4 4:6 5:4 6:2 7:2 8:4 10:4 12:2 26:1 27:1 28:1 29:2 32:1 41:1 45:1 47:1 50:1 54:2 57:1 59:1 63:2 64:1 66:1 68:2 70:1 72:1 78:1 100:1 106:1 116:1 122:1 125:1 136:1 140:1 142:1 150:1 167:1 183:1 201:1 207:1 208:1 213:1 217:1 230:1 255:1 321:5 343:1 357:1 370:1 390:2 468:1 514:1 571:1 619:1 671:1 766:1 877:1 1057:1 1179:1 1192:1 1402:2 1416:1 1477:2 1940:1 1941:1 2096:1 2243:1 2285:1 2379:1 2934:1 2938:1 3520:1 3647:1 4938:1 5138:4 5715:1 5726:1 5731:1 5812:1 8319:1 8567:1 10480:1 14239:1 20604:1 22409:4 24551:1 47304:1
```

要分析评论，光靠单个词汇是不够，要考虑词汇的结构性，比如2个单词才能表达一个完整的意思.

要分析结构性，自然要建立语言模型，能够预测当前语句的下一个词汇.

要建立语言模型，自然需要把英文的单词tokenlized,直接处理单词是困难的，nlp的基础.

fastai用到`spacy` + `torchtext`，貌似也支持中文的.初始化好后作为参数
```py
spacy_tok = spacy.load('en')
TEXT = data.Field(lower=True, tokenize="spacy")
bs=64; bptt=70
FILES = dict(train=TRN_PATH, validation=VAL_PATH, test=VAL_PATH)
md = LanguageModelData.from_text_files(PATH, TEXT, **FILES, bs=bs, bptt=bptt, min_freq=10)
```
用pickle吧text的vocabulary dump保存，就是一个单词库，每个单词都有单独的id.
```py
pickle.dump(TEXT, open(f'{PATH}models/TEXT.pkl','wb'))
```
把train sets的某个样本的单词id化:
```py
TEXT.numericalize([md.trn_ds[0].text[:12]])
Variable containing:
   12
   35
  227
  480
   13
   76
   17
    2
 7319
  769
    3
    2
[torch.cuda.LongTensor of size 12x1 (GPU 0)]
```

