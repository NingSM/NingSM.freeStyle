---
layout: post
title: "Window10下Anaconda安装Tensorflow+Keras"
comments: true
date: 2017-11-4
description: "anaconda+tensorflow+keras"
tag: Deep Learning

---

# Window10下Anaconda安装Tensorflow+Keras
之前作比赛用到了深度学习中的LSTM，最近又要用到，发现实验室tensorflow等环境被删除，因此又得重新安装，记录一下，免得忘掉。
由于实验室服务器的GPU+Cuda是配置好的，所以没有进行这块的配置。

---
## 1  Anaconda

1>首先在Anaconda![官网](https://www.continuum.io/downloads)下载最新版的anaconda，我用的3.6版本，版本其实都无需关心，只需关心到时候自己所有的python版本即可。
安装过程中提示failed to create anacoda menue错误时参考http://www.cnblogs.com/chuckle/p/7429624.html。

2>安装成功后，打开终端 Anaconda Prompt进行清华镜像的更改，输入一下命令：
```
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --set show_channel_urls yes
```

## 2  安装Tensorflow+Keras
打开Anaconda的Navigator，可以看到环境中有一个叫做root的环境名，点击root选择Open Terminal。此命令界面中可以使用conda install ***  来进行package的安装。
首先安装Tensorflow GPU版本：
```
conda install tensorflow-gpu
```
如果想使用CPU版本的话，请直接敲入：
```
conda install tensorflow
```

安装过程中需要用户手动输入一次y来确认。安装过程需要花费几分钟。

安装成功后，在root上点击选择open with python，输入` import tensorflow  `，如果不报错，说明安装成功。

同样的，在Terminal中输入conda install keras来安装keras。安装成功后同样在python终端输入`  import keras `来检查。



---

**转载请注明原址，谢谢**。
