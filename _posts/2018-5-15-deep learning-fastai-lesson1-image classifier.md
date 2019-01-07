---
layout: post
title:
modified:
categories: Tech
 
tags: [fastai,deep-learning]

  
comments: true
---

<!-- TOC -->

- [data的组织形式](#data的组织形式)
- [Precompute and freeze](#precompute-and-freeze)
- [Choosing a start learning rate](#choosing-a-start-learning-rate)
- [Stochastic gradient descent with restarts.(SGDR)](#stochastic-gradient-descent-with-restartssgdr)
    - [Data augmentation](#data-augmentation)
    - [Test time augmentation(TTA)](#test-time-augmentationtta)
    - [Differential learning rate annealing](#differential-learning-rate-annealing)
    - [Analyzing results with confusion matrix](#analyzing-results-with-confusion-matrix)
    - [To sumerise](#to-sumerise)

<!-- /TOC -->

### data的组织形式

1种是 按文件夹分类,train valid下有dog和cat类; test下全是jpg,分类就是将jpg像train文件夹下面一样按类放好.

另1种是常见的image + lable分开的情况.

这两种情况,fastai都考虑到了，调用不同的接口即可:

```
ImageClassifierData.from_paths(...)

ImageClassifierData.from_csv(...)
```
还有直接从内存load的
```
ImageClassifierData.from_array(...)
```

[A blog for help](https://blog.csdn.net/gbyy42299/article/details/78977826),再理解下.

`a model created by some one else to solve a different problem`

几种境界:

>复用已经train好的model,毕竟能分类1000种 (resnet34)

>替换output layer,如调整个数，重新train weights (pretrain model)

>freeze前面n-k层，train后面k层weights

>网络基本结构不变，改下层数，重新训练所有层weights(resnet18,resnet34,reset101...)

>新的轮子!(leNet,GoogleNet,VGG,RestNet,ResNext...)

fastai的例子是在第2和第3种境界里,不过大部分场景可能都够用了.

### Precompute and freeze

* freeze

`Make all layers untrainable (i.e. frozen) except for the last layer.`

freeze意味着training过程,仅更新last output layer的weights,而不更新第1到n-1层的weights(除去output layer).

从ImamgeNet这种巨无霸训练出来的pretrained model(如resnet)仅调整last layer，对image classification这种任务，就可以有很好的效果.

```py
arch=resnet34
data = ImageClassifierData.from_paths(PATH, tfms=tfms_from_model(arch, sz))
learn = ConvLearner.pretrained(arch, data, precompute=True)
learn.fit(0.01, 2)
```

![2018-07-04-16-28-16](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-07-04-16-28-16.png)

* unfreeze

`Make all layers trainable by unfreezing. This will also set the 
precompute = False since we can not longer pre-calculate the 
activation of frozen layers.` 

unfreeze相反，training时会更新所有layer的weights,如果是pretrain,则在pretrain model的weights基础上,这其实就是普通模型traning时最一般的情况。

unfreeze 会将precompute= False; 因为这两者是冲突的，有unfreeze就没有precompute

同理，下面提到的`data augmentation`同precomputed=True也是冲突的。一旦precomputed=True,augmentation就没有效果,重点看这个data

```py
class ConvLearner(Learner)
def __init__(self,...)
    if precompute: self.save_fc1()
...
@property
    def data(self): return self.fc_data if self.precompute else self.data_
```


* precompute

precompute是pretrain前提下的一种优化手段，就是提前计算了freeze掉的1到n-1层的weights forward计算出来的值,叫`precomputed activations`

```
I perfectly understood now. The precompute is avoiding the model to recalculate the 
activation values for every image over all the previous frozen layers ( it calculates 
for the first time and saves in a temporary directory ). This way, we can just play 
around with the last (unfrozen) layers much faster. Such a smart trick…`
```

```
Precomputed = True, means reuse precomputed activations.
if we use precompute = True, then freeze / unfrozen do not work as we operate only on 
already calculated outputs of conv layers.
also data augmentation do not work
in order to use data augmentation and fine tune conv layers we need to start with 
precompute=False
in this case by default all conv layers are freezed
```

再看代码。precompute=True时 fc_data来自`save_fc1`.

`save_fc1`在__init__，or调用set_data时，执行1次. 

```py
   def set_data(self, data, precompute=False):
        super().set_data(data)
        if precompute:
            self.unfreeze()
            self.save_fc1()
            self.freeze()
            self.precompute = True
        else:
            self.freeze()
```

针对data set的每一个x, `save_fc1`计算前面提到的precomputed activations,并保存。 1个epochs也许没啥优化效果，但多个epochs时(epochs可以认为是train data重复训练多少次),就不用重复计算这些值,得到了优化。

注意的是，如果是为了train last layer，可以考虑precomputed=True,但如果是fine tune整个网络，记得precomputed=False和unfreeze.(包含了precompute=False)


```py
def save_fc1(self):
        self.get_activations()
        act, val_act, test_act = self.activations
        #top model就是除了last layer外的pretrain model
        m=self.models.top_model
        if len(self.activations[0])!=len(self.data.trn_ds):
            #这里就是在提前计算precompute activations
            predict_to_bcolz(m, self.data.fix_dl, act)
        if len(self.activations[1])!=len(self.data.val_ds):
            predict_to_bcolz(m, self.data.val_dl, val_act)
        if self.data.test_dl and (len(self.activations[2])!=len(self.data.test_ds)):
            if self.data.test_dl: predict_to_bcolz(m, self.data.test_dl, test_act)

        #这里就是把load好的activation val放到fc_data,即model.data,给后续使用
        self.fc_data = ImageClassifierData.from_arrays(self.data.path,
                (act, self.data.trn_y), (val_act, self.data.val_y), self.data.bs, classes=self.data.classes,
                test = test_act if self.data.test_dl else None, num_workers=8)
```
### Choosing a start learning rate

* lr and cost function
一种确定lr方法，逐步增加学习率，观察损失函数的翻转时期，如果发生翻转，用翻转前一点的lr作为最佳学习率

![2018-05-16-22-04-55](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-16-22-04-55.png)

### Stochastic gradient descent with restarts.(SGDR)

在接近local minima时，流行的做法是逐渐减少lr.

一个更合适的方法是用half cos函数(cosine annealing的由来)来拟合iteration和lr关系.iteration越大，lr下降的越平缓，而刚开始时，允许lr快速下降，加速迭代。

并且,为了防止掉到一些不佳的local minima中，最后的lr可以jump一下，即增加lr跳出local minima,而是restart的含义。
```
learn.fit(1e-2, 3, cycle_len=1)
```
cyclen_len是多少个epochs reset1次上面说的learning rate.

![2018-05-16-23-40-51](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-16-23-40-51.png)

#### Data augmentation

为防止overfit，可考虑在训练前增加样本，增加的样本来自原样本的slight仿射变换(太大的变换导致特征变形，不妥).旋转，平移，翻转等,

不同种类的样本，变换的形式可能不一样，dog and cat没有翻转的..icgberg就有翻转的.and so on.

对图像以外的data augmentation可不一定管用,如nlp,刚刚也在想这个问题，如果是线性回归，SVM之类的，如果改变了data的空间，势必影响分类结果.

image之所以有用，是因为人类眼睛已经锁定了图像特征，对各种变换的图像仍然会正确标注，所以通过变换来增加样本容量是resonable的.

由此想到中文nlp也是很神奇，..也可以变换!`你不会为认我下面的打字的会有什么题问...`

data augmentation也是针对last layer做的

#### Test time augmentation(TTA)

uses the original image along with 4 randomly augmented versions). It then takes the average prediction from these images.

TTA　用data augmentation的思路去扩充valadation dataset.

比如每个样本扩充了10个变形，validation set扩充10倍，each sample取平均，看`预测效果`是否改善,使得评估更准确，这里并没有改变已经训练好的model.

学生提问data augmentation里，如果是crop,要丢失信息?Howard建议是把图片弄大点再crop..这样不丢失.

#### Differential learning rate annealing

越靠前的层发现的特征比较粗犷，更有普适性，学习率可以低一点，(参数不需要调整太多),后面的层特征越精细，越需要调整.

```py
learn.unfreeze()
lr=np.array([1e-4,1e-3,1e-2]);
learn.fit(lr, 3, cycle_len=1, cycle_mult=2)
```

另外按`越到后面，速率的调整越慢,越精细`的思想:

前面SGDR方法的restart的宽度也可以调整, cycle_mult =2,即每个cycle都2倍增加jump的宽度.

![2018-05-17-12-17-23](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-17-12-17-23.png)

这个trick效果很好.

有学生提到了和`gridsearch`的区别。应该就是sklearn里的`GridSearchCv`,后者用来找最佳超参数的，而`dlra`是已经确定了３个best lr,然后用来不同的layer上

有人提到了图片的size,Howard提到了fast ai可以搞定任何size的图片...我猜应该是预处理将所有的size图片adjust to uniform value as input layer，like 224x224?

到这里，**full model has been trained.**


#### Analyzing results with confusion matrix
　应该就是真值表，precision,recall,F1这些指标. True-True,True-false False-True, False-False;

#### To sumerise

to train a better image classifier:

![2018-05-17-09-48-14](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-05-17-09-48-14.png)

