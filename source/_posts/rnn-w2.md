---
title: Recurrent Neural Networks / Week 2
date: 2018-04-15 07:17:30
tags: [deep-learning, rnn]
mathjax: true
---

### Word Representation

One hot编码有天然的劣势，比如，

* I want a glass of orange <u>juice</u>.
* I want a glass of apple <u>juice</u>.

如果我们从上面一句学到了orange后边是juice，是无法推断出来第二句的填空题中apple后边应该填juice。因为在One hot编码中，orange和apple两个向量之间没有任何的关联关系。因此，我们需要Word representation来对词进行表达，而不是用One hot编码。

<!-- more -->

### Transfer Learning and Word Embeddings

1. Learn word embeddings from large text corpus. (1-100B words)

   (Or download pre-trained embedding online.)

2. Transfer embedding to new task with smaller training set. (Say, 100k words)

3. Optional: Continue to finetune the word embeddings with new data.

### Relation to Face Encoding

在Face encoding中，人脸图像的输入可以是任意一张图像，经过CNN之后可以变成向量的表达，这个过程叫做Encoding。而对于Word embedding来说，我们的词表是固定的，在WE模型训练的过程中决定了我们对哪些words进行表达学习。

### Properties of Word Embeddings

Word embedding有一些有趣的性质，比如，
$$
e \_ { man } - e \_ { woman } \approx e \_ { king } - e \_ { queen }
$$

### Embedding Matrix

假设Embedding matrix是$m\times n$的矩阵，其中$m$是embedding的维度，$n$是词表的大小，第$j$个词的embedding必到可以通过Embedding matrix与该词One-hot编码（$shape=n\times1$）相乘得到，
$$
E~o_j = e_j
$$
当然实际运算中不会这么计算，效率太低了。

### Word2Vec之Skip-gram Model

Skip-gram的思想是在句子中随机摘取一个词作为context，再在这个词的上下文中（一定长度的窗口）中随机摘取另一个词作为target，训练如下的神经网络结构，
$$
o\_c \rightarrow Embedding~Layer \rightarrow e\_c \rightarrow Softmax~Layer \rightarrow \hat{y}
$$
Softmax layer的概率预测如下所示，
$$
Softmax:~p(t|c) = \frac{e^{\theta\_{t}^{T}e\_c}}{\sum\_{j=1}^{Vocab~size} e^{\theta\_{j}^{T}e\_c}}
\tag{1}
$$
其中，$\theta<span class="md-search-hit">_</span>t$表示与目标词One-hot向量第$t$维有关的参数向量。

Loss function如下，
$$
\mathcal{L}(\hat{y}, y) = - \sum_{i=1}^{Vocab~size} y_i \log \hat{y}_i
$$
但是在式(1)的计算中，计算量是很大的（需要遍历词表），所以采用Hierarchical softmax来计算给定context:c的情况下target:t出现的概率，而不是采用普通的Softmax classification方法。

> [word2vec 中的数学原理详解（四）基于 Hierarchical Softmax 的模型](https://blog.csdn.net/itplus/article/details/37969979)

Word2Vec有两个版本，Skip-gram和CBOW。

 ### Negative Sampling

首先定义一个监督学习问题，给定一个词对$< c, t >$，并对该词对进行二分类。这个问题的正样本可以通过上一小节中介绍的方法得到，然后根据每个正样本可以通过随机采样词表的方式来得到$k$条负样本。这样，针对一个词对，我们可以计算词对为正样本的概率，
$$
p(y=1 | c,t) = \delta(\theta_t^{T} e_c)
$$
构造如下结构的网络，
$$
o_c \rightarrow Embedding~Layer \rightarrow e_c \rightarrow Vocab~size \times Logistic~Unit
$$
这样，针对每个样本，我们只需要从Vocab size个Logistic Unit中选择target:t所对应的那个二分类器去更新参数就好了，这样极大的减少了计算量。

### GloVe Word Vectors

Glove Word Vectors的目的是给定一个词对$< c, t >$，来进行一个回归问题，预测目标词t在c的上下文中出现的次数。它的优化目标是，
$$
minimize \sum\_{i=1}^{Vocab~size} \sum\_{j=1}^{Vocab~size} f(x\_{ii}) (\theta\_i^T e\_j + b\_i + b\_j' - \log x\_{ij})^2
$$
其中，$i, j$分别表示$t, c$在词表中的编号，$x_{ij}$表示$t$在$c$的上下文中出现的次数。$f(x_{ij})$是weight function，当$x_{ij}=0$的时候，$f(x_{ij})=0$。

在这个式子中，我们可以发现，$\theta_i$与$e_j$是对称的，所以，
$$
e_w^{final} = \frac{e_w + \theta_w}{2}
$$

### Debiasing Word Embeddings

无关主题的话，在上NG的课的时候，听到NG介绍了一些人的工作，比如说给你一幅油画，怎么把照片变成油画的风格，再比如有的人会去研究word embedding中的歧视现象（性别、种族等），发现这些人都好有意思啊，老外们不一样的科研。

