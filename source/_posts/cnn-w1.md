---
title: Convolutional Neural Networks / Week 1
date: 2018-01-10 23:53:50
tags: [deep-learning, cnn]
mathjax: true
---

### Computer Vision

Computer Vision Problems include:

* Image Classication
* Object Detection
* … ...

One of the challenges of computer vision problems is that the input can be very big. For example, a 1000 by 1000 image can have $1000 \times  64 \times 3 = 12288$ dimensions because there are three color channels. If the size of hidden layer is 1000, the number of parameters from input layer to hidden layer could be 3 billion. This will cause these problems:

1. data size requirements;
2. computational requirements;
3. memory requirements.

<!-- more -->

### Padding

The problems of convolutional operation:

1. shrinking output
2. throwing away a lot of information from the edges of the image

In order to fix these problems, what we need to do is **pad the image**.

通常有两种padding的方式：

* Valid convolution: 意思是不采用padding的方式。
* Same convolution：意思是输出的尺寸和输入的尺寸相同。
  * 在这种情况下可以推导出$p = \frac{f-1}{2}$，所以在计算机视觉的模型中，filter的尺寸通常是奇数而不是偶数。
  * Filter的尺寸是奇数还有另外一个好处，就是filter可以有中心像素，可以很方便的用来定位filter的位置。

### Strided Convolutions

给定如下条件，
$$
\begin{split}
&n\times n~image~~~~&f\times f~filter\\\\
&padding~p&stride~s
\end{split}
$$
经过Strided Convolutions之后得到的tensor的尺寸为，
$$
\lfloor \frac{n + 2p - f}{s} + 1 \rfloor \times \lfloor \frac{n + 2p - f}{s} + 1 \rfloor
$$

> NG在这里提到，我们所谓的convolution并不是真正意义上的卷积，而是应该称为cross-correlation，它之前实际上应该有一个针对卷积核的变换操作，这些操作再加上cross-correlation才是真正的convolution。但是这个变换操作没什么用处，所以通常情况下就省略了。

### Convolutions Over Volume

在RGB类型的多通道图像中使用Multiple filters：
$$
n \times n \times n_c \ast f\times f \times n_c \rightarrow (n - f + 1) \times (n - f + 1) \times {n_c}'
$$
其中，$n$表示图像的长宽，$f$表示filter的长宽，$n_c$表示图像的通道数，$n_c'$表示filter的个数。

### One Layer of a Convolutional Network

普通的BP神经网络的数学表达形式如下：
$$
\begin{split}
z^{[1]} &= w^{[1]} a^{[0]} + b^{[1]} \\\\
a^{[1]} &= g(z^{[1]}) 
\end{split}
$$
在CNN中，convolution operation相当于$w^{[1]}a^{[0]}$，也就是充当了原先线性变换的角色。

这里对卷积层中涉及到的符号进行总结，
$$
\begin{split}
f^{[l]} &= filter~size \\\\
p^{[l]} &= padding \\\\
s^{[l]} &= stride \\\\
n_{C}^{[l]} &= number~of~filters
\end{split}
\tag{1}
$$
接着定义卷积层的输入和输出表示，
$$
\begin{split}
Input:~&n_H^{[l-1]} \times n_W^{[l-1]} \times n_C^{[l-1]} \\\\
Output:~&n_H^{[l]} \times n_W^{[l]} \times n_C^{[l]}
\end{split}
\tag{2}
$$

基于(1)和(2)，我们可以进行如下定义，
$$
\begin{split}
&Each~filter~is:~	&f^{[l]} \times f^{[l]} \times n_C^{[l-1]} \\\\
&Activations:~		&a^{[l]} \rightarrow n_H^{[l]} \times n_W^{[l]} \times n_C^{[l]} \\\\
&Weights:~		&f^{[l]} \times f^{[l]} \times n_C^{[l-1]} \times n_C^{[l]} \\\\
&bias:~			&n_C^{[l]}
\end{split}
\tag{3}
$$
在式(2)中，$n_H^{[l]}$与$n_H^{[l-1]}$的关系如下，
$$
n_H^{[l]} = \lfloor \frac{n_H^{[l - 1]} + 2p^{[l]} - f^{[l]}}{s^{l}} + 1 \rfloor
$$
在式(3)中，Activations是单个样本的形式，batch的形式如下，
$$
A^{[l]} \rightarrow m \times n_H^{[l]} \times n_W^{[l]} \times n_C^{[l]}
$$

### Simple Convolution Network Example

以图像分类为例（识别图片中是否有猫），经过若干卷积层之后，为了得到最终的$0/1$分类结果，会将最后一层卷积的tensor展开并拉长成vector，经过logistic/softmax单元后得到代表预测结果的概率值。

在使用ConvNet的过程中，比较麻烦的地方在于如何确定超参。有一个常用的指导方针是，activations的长和宽需要越来越小（也就是图片的尺寸越来越小），同时通道数需要越来越多（也就是activations的第三个维度）。之后会详细介绍怎么需选择超参。

在ConvNet中，通常有三种类型的网络层，

* Convolution (CONV)
* Pooling (POOL)
* Fully connected (FC)

### Pooling Layers

Pooling Layer有如下好处，

* 减少图像representation的尺寸，提高计算速度
* 提高鲁棒性

很有意思的地方在于，对于pooling layer来说，我们只需要确定超参数$f^{[l]}$和$s^{[l]}$，以及是max pooling 还是average pooling，并不需要进行参数的学习。

在pooling layer中，超参$p^{[l]}$通常设置为0。

### CNN Example

神经网络中常用的一种模式是，若干卷积层之后加池化层，再若干层卷积层之后接池化层，然后接全连接层，最后给softmax单元。NG在课上画了一个例子如下，

<div align=center>

![cnn-example](/img/cnn-w1/cnn-example.jpeg)

</div>

### Why Convolutions

卷积层最显著的特点就是参数数量大大小于全连接层的数量，因为：

1. Parameter sharing: 图像的不同位置共享filters。
2. Sparsity of connections: 每个输出值只取决于很小的一部分输入。这样也降低了过拟合的风险。

卷积神经网络的损失函数定义如下所示，
$$
Cost~J = \frac{1}{m} \sum_{i-1}^{m} \mathcal{L}(\hat{y}^{(i)}, y^{(i)})
$$

### References

- [MarkDown中使用Latex数学公式](http://www.cnblogs.com/nowgood/p/Latexstart.html)
- [Hexo博客(13)添加MathJax数学公式渲染](http://masikkk.com/article/hexo-13-MathJax/)
  - 解释了Markdown和Mathjax渲染冲突问题







