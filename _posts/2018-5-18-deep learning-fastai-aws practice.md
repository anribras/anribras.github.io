---
layout: post
title:
modified:
categories: Tech
 
tags: [deep-learning,fastai]

  
comments: true
---

<!-- TOC -->

    - [start aws p2.xlarge instance](#start-aws-p2xlarge-instance)
    - [run jupyter notebook](#run-jupyter-notebook)
    - [kaggle cmd](#kaggle-cmd)
    - [upload ipynb](#upload-ipynb)
- [aws关闭与重启](#aws关闭与重启)
    - [7.3更新](#73更新)

<!-- /TOC -->


基本[参考](https://github.com/reshamas/fastai_deeplearn_part1/blob/master/tools/aws_ami_gpu_setup.md)就可以了.

价格还有有点小贵 光是gpu 3rmb/hour..先试一个月，豁出去了

#### start aws p2.xlarge instance

社区AMI 搜fastai 配一个.

访问:

```
ssh -i ~/.ssh/x260-2018-0517.pem ubuntu@34.201.8.230 -L8888:localhost:8888 
```

#### run jupyter notebook

```
jupyter notebook --generate-config

vim ~/.jupyter/jupyter_notebook_config.py

```

add`c.NotebookApp.ip = '*'`

```
jupter note-book &
```

然后在note-book运行的url改个aws ip即可.

#### kaggle cmd 

不得不说用aws下载kaggle data那叫一个快.

```
pip install kaggle
```

kaggle.json,到自己的kaggle账户里下到本地,传到aws

```
scp kaggle.json ubuntu@34.234.100.228:/home/ubuntu/.kaggle
```

download data:

```
mkdir -p /home/ubuntu/fastai/courses/dl1/data/dogbreeds

kaggle competitions download -c dog-breed-identification -p /home/ubuntu/fastai/courses/dl1/data/dogbreeds 

```

kaggle submit result.csv:

kaggle competitions submit -c dog-breed-identification -f  -m "1st fast ai result"



#### upload ipynb

原始版本是自己电脑上先写(gai)的,推到aws环境里:

```
scp xx.ipynb  ubuntu@xxxxx:/home/ubuntu/fastai/courses/dl1/
```
[image classifier dog and breeds](https://github.com/anribras/machine-learning/blob/master/note/lesson1_dog_breeds.ipynb) 

### aws关闭与重启

在线收费很贵的..竞价请求关闭前，可以存个快照，再取消该竞价请求.

要再用的时候,快照创建镜像，重新竞价请求即可.

#### 7.3更新

6月份被扣了个7刀，下面这俩是罪魁，自己把disk设置80G,免费的只有30GB...每天都要收钱的..

```
0.1 GB-M 的卷
0.05 GB-M 的快照
```

重新搞了个spot instance 这次用40GB的,基于社区的fast-ai

镜像的类型选择为hvm，不要选择pv.,否则下次竞价选择p2.xlarge会报错

* git clone fastai

* update pytorch

J神建议还是用0.3 ,新的0.4可能还不能完全支持

`fastai`是conda env的名字

```
conda install -y --name fastai$PYTHON_VERSION -c soumith pytorch=0.3.0
```

* cuda not ok

```
apt-get install cuda-drivers
```
重启下sudo reboot -f




