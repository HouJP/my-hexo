---
title: Convolutional Neural Networks / Week 2
date: 2018-04-02 20:50:35
tags: [deep-learning, cnn]
mathjax: true
---

### Classic Networks

LeNet-5的网络结构如下，

<img src="/img/cnn-w2/lenet-5.jpeg" width="80%" align=center />

<!-- more -->

AlexNet的网络结构如下，

<img src="/img/cnn-w2/alexnet.jpeg" width="80%" align=center />

VGG-16的网络结构如下，

<img src="/img/cnn-w2/vgg-16.jpeg" width="80%" align=center />

### ResNets

在一般的神经网络中，两层神经网络的数学表达如下，
$$
\begin{split}
& z^{[l+1]} = W^{[l+1]}a^{[l]} + b^{[l+1]} \\\\
& a^{[l+1]} = g(z^{[l+1]}) \\\\
& z^{[l+2]} = W^{[l+2]}a^{[l + 1]} + b^{[l+2]} \\\\
& a^{[l+2]} = g(z^{[l+2]})
\end{split}
$$
而在ResNets中，修改了$a^{[l+2]}$的生成方式，变成了，
$$
a^{[l+2]} = g(z^{[l + 2]} + a^{[l]}) 
\tag{1}
$$
这样的两层神经网络称为Residual block，将这样的Residual block串联起来就构成了ResNets。

ResNets解决了传统神经网络中存在的层数不能过深的问题。在传统的神经网络中，随着深度的增加，训练误差会先降后升，而对于ResNets，随着网络层数达到一百甚至一千层，训练误差也可以平缓的下降（也可能出现收敛的现象）。

### Why ResNets Work

首先来解释一下为什么在传统的网络后面加一个Residual block不会降低原有网络的性能，
$$
\begin{split}
a^{[l+2]} &= g(z^{[l + 2]} + a^{[l]}) \\\\
	&= g((w^{[l+2]}a^{[l + 1]} + b^{[l + 2]}) + a^{[l]})
\end{split}
$$
假设我们采用的激活函数为RELU，同时$w^{[l+2]}$和$b^{[l+2]}$为0，那么，
$$
a^{[l+2]} = g(a^{[l]}) = a^{[l]}
$$
所以由于Residual block的存在，网络在第$l+2$层的时候，很容易退回到$l$层去。这样可以达到一个效果，在最差情况下，后边加上的Residual block仿佛不存在一样，这样就不会影响原先的效果。

Residual block中还有一点值得注意，对于式(1)来说，$z^{[l+2]}$和$a^{[l]}$的维度需要一致，那如果出现维度不一致的情况怎么办呢？增加一个$W_s$矩阵，
$$
a^{[l+2]} = g(z^{[l + 2]} + W_s a^{[l]}) 
$$
$W_s$有两种方式生成，

1. 随机生成的参数矩阵，跟随其他参数一起训练学习。
2. $a^{[l]}$的基础上采用padding操作生成，比如补0。

### Networks in Networks and 1x1 Convolutions

1x1 Convolutons也称为Networks in networks，它是一个1x1的filters并使用了Relu非线性变换。

### Inception Network Motivation

在构造神经网络的时候，我们有时候会很困惑，用$1\times1$的卷积效果好，还是$f^{[l]}\times f^{[l]}$的卷积效果好，或者用Max-pooling效果会更好呢？Inception Network的思想是，那就把他们在同一层中都用一遍，这样就会得到若干tensor的输出，然后再把这些tensor在channel的纬度上拼在一起，组合一个大的tensor。最后，用数据去训练学习，决定这些filters的参数。

这样会有如下问题，

1. 如何让这些tensor在除了channel之外的其他维度上保持尺寸一致？
2. 会不会造成计算量的显著增加？

对于问题(1)其实很好解决，采用padding的方式就可以让这些tensor的$n_H$和$n_W$保持一致。

对于问题(2)可以通过在两层网络中间增加一层$1\times1$ filter来解决。下面详细描述原理。

假设我们有这样两层网络，

<img src="/img/cnn-w2/inception-2-layer.jpeg" width="50%" title="图 1" align=center />

那么从左到右需要的乘法运算的数量为，
$$
(28 \times 28 \times 32) \times (5 \times 5 \times 192) \approx 120~million
$$
如果我们在图(1)两层网络中间加入一层使用了$1\times1$ filter的卷积层，如图(2)所示，

<img src="/img/cnn-w2/inception-3-layer.jpeg" width="50%" title="图 2" align=center />

那么从左到右所需要的乘法运算的数量为，
$$
\begin{split}
&1st~layer \rightarrow 2nd~layer：&(28 \times 28 \times 16) \times (1 \times 1 \times 192) \approx 2.4~million \\\\
&2nd~layer \rightarrow 3rd~layer：&(28 \times 28 \times 32) \times (5 \times 5 \times 16) \approx 10~million
\end{split}
$$
也就是共需要$12.4~million$的乘法运算。从中可以看到，加入"bottleneck layer"之后，所需要的计算量减少为了原来的十分之一。

### Transfer Learning

当我们有一个比较小的训练数据集的时候，我们可以在别人训练好的模型的基础上来达到我们的目的：删除最后的softmax layers，保留之前的layers的模型结构和权重，并在之后增加我们自己的softmax layers。然后用较小的训练数据集来训练我们新增加的layers的参数。这样就可以用较少的数据来得到不错的预测效果。

其中，我们可以预先存储训练数据集中的样本经过之前的layers (删除原先的softmax layers)之后得到的activations，这样就不用在之后训练新的softmax layers参数的时候反复计算，从而节省计算量并提高效率。

随着我们拥有的训练数据的增加，我们可以保留较少层数的参数不发生改变，其余网络层以原先权重为初始参数，然后在新的训练数据集上进行训练调整。如果我们的训练数据足够大，那么原先所有层的参数都可以只作为初始参数，让它们在新的数据集上进行训练调整。

### Data Augmentation

通常在机器学习中，我们需要大量的训练数据，因此有一些常用的有效的增加数据集的方法，

* Mirroring: 镜像处理
* Random Cropping: 随机图像裁剪
* Rotation: 图像旋转
* Shearing: 
* Local warping: 局部变形
* Color Shifting: 在不同的颜色通道上增减一定的数值，例如$R+20, G-20, B+20$



