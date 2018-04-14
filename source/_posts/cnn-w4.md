---
title: Convolutional Neural Networks / Week 4
date: 2018-04-06 22:40:27
tags: [cnn, deep-learning]
mathjax: true
---

### What is Face Recognition

这里区分两个概念，`Face verification`和`Face recognition`，

- Verification
  - 输入：图像 + 名字/ID
  - 输出：输入的图像上是否有名字/ID表示的那个人
- Recognition
  - 数据库中有$K$个人
  - 输入：图像
  - 输出：图像上有的数据库中的人的名字/ID

很明显，Recognition比Verification的难度要大得多。

<!-- more -->

### One Shot Learning

什么是One-shot learning，

> Learning from one example to recognize the person again.

很多时候公司的数据库中只有一张员工的照片，那么我们在做人脸识别系统的时候怎么根据这一张照片来再次识别相同的人的影像？如果用传统的卷积神经网络来做的话，因为训练数据很小，所以通常效果并不理想。

在这种情况下，我们应该学习"similarity" function,
$$
d(img1, img2) = degree~of~difference~between~images 
$$
然后可以根据Similarity function完成verification,
$$
\begin{split}
If~d(img1, img2) &\le \tau~same\\\\
&> \tau~different
\end{split}
$$

### Siamese Network

Parameters of NN define an encoding $f(x^{(i)})$

Learn parameters so that:

- If $x^{(i)}, x^{(j)}$ are the same person, $\| f(x^{(i)}) - f(x^{(j)}) \|^2$ is small
- If $x^{(i)}, x^{(j)}$ are the different person, $\| f(x^{(i)}) - f(x^{(j)}) \|^2$ is large

那么，该网络学习的目标函数应该怎么定义呢？

### Triple Loss

在Triple loss中，基准人脸图像称为Anchor image，正样本为Positive image，负样本为Negative image，那么我们希望得到的是，
$$
\|f(A) - f(P) \| ^2 \le \| f(A) - f(N) \| ^2
$$
也就是，
$$
\|f(A) - f(P) \| ^2 - \| f(A) - f(N) \| ^2 \le 0
$$
这里有个问题，如果$f$始终预测0，那么上述条件始终满足。为了防止这种情况发生，所以需要增加一个超参$\alpha$，也称为margin，
$$
\|f(A) - f(P) \| ^2 - \| f(A) - f(N) \| ^2 + \alpha \le 0
$$
下面对Loss function进行形式化定义，给定3个图像 A, P, N，
$$
\mathcal{L}(A, P, N) = max(\|f(A) - f(P)\|^2 -\|f(A) - f(N)\|^2  + \alpha, 0)
$$
在整体样本上的损失为，
$$
J = \sum_{i=1}^{M} \mathcal{L}(A^{(i)}, P^{(i)}, N^{(i)})
$$
很明显，在训练样本中，一个人需要有多张照片才能组合出这样的三元组。

那么，应该怎么选择三元组呢，

* During training, if A, P, N are chsen randomly, $d(A, P) + \alpha \le d(A, N)$ is easily satisfied.
  * 这种方法容易选择，但是训练出来的模型效果一般
* Choose triplets that're "hard" to train on.
  * 这种方法不容易选择，但是模型能学习到更多的信息。

### Face Verification and Binary Classification

另一种similarity function，以一对图像作为输入，
$$
\hat{y} = \delta ( \sum_{k=1}^{128} w_i | f(x^{(i)})_k - f(x^{(j)})_k   | + b  )
$$
其中，$f(x^{(i)})$表示对第i张图像的128维的embedding表达。

### What are deep ConvNets Learning

这里介绍了怎么将卷积神经网络的hidden layer可视化。以第一层为例，

> Pick a unit in layer 1. Find the nine image patches that maximize the unit's activation.
>
> Repeat for other units.

可以看到，每个unit's activation针对的方向不同，有的是颜色，有的是不同方向的边。随着网络深度的增加，每个unit's activation可以看到的图像的范围越来越大。 

### Neural Style Transfer Cost Function

原始内容图片为C，风格图片为S，目标图片为G，那么，
$$
J(G) = \alpha J_{content}(C, G) + \beta J_{style}(S, G)
$$
下面介绍Style matrix,
$$
\begin{split}
& Let~a_{i,j,k}^{[l]} &= activation~at~(i,j,k). G^{[l]}~is~n_c^{[l]} \times n_c^{[l]} \\\\
\rightarrow & G_{kk'}^{[l]} &= \sum_{i=1}^{n_H^{[l]}} \sum_{j=1}^{[n_W]^{[l]}} a_{ijk}^{[l]} a_{ijk'}^{[l]}
\end{split}
$$
其中，$G^{[l]}$为Style matrix。