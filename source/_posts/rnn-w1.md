---
title: Recurrent Neural Networks / Week 1
date: 2018-04-08 07:17:30
tags: [deep-learning, rnn]
mathjax: true
---

### Why not a standard network

传统网络不可行的原因，

1. Inputs, outputs can be different lengths in different examples.
2. Doesn't share features learned across different positions of text.

RNN的网络结构如下所示，

<img src="/img/rnn-w1/rnn.jpeg" width="70%" align=center />

<!-- more -->

形式化描述如下，
$$
a^{< t >} = g_1 ( W_{aa} a^{< t - 1>} + W_{ax} x^{< t >} + b_a) \\\\
\hat{y}^{< t >} = g_2 ( W_{ya} a^{< t >} + b_y )
$$
其中，$W_{ax}$中的角标表示的是输出和输入。

为了简单起见，改写为如下形式，
$$
a^{< t >} = g_1 (W_a [a^{< t - 1 >}, x^{< t >}] + b_a) \\\\
\hat{y}^{< t >} = g_2 (W_y a^{< t >} + b_y)
$$
其中，$W_a$由$W_{aa}$和$W_{ax}$水平拼接而成，$[a^{< t - 1 >}, x^{< t >}]$表示两者的垂直拼接。

### Loss function

针对命名实体识别问题，定义单个句子上某个输出位置的损失函数，
$$
\mathcal{L}^{< t >} (\hat{y}^{< t >}, y^{< t >}) = - y^{< t >} \log \hat{y}^{< t >} - (1 - y^{< t >}) \log (1 - \hat{y}^{< t >})
$$
那么，整个句子上的损失函数为，
$$
\mathcal{L}(\hat{y}, y) = \sum_{t=1}^{T_y} \mathcal{L}^{< t >} (\hat{y}^{< t >}, y^{< t >})
$$

### Word-level VS. Character-level Language Model

使用字符级别的语言模型，可以不用关心unknown word tokens。但是缺点也很明显，

1. 很多情况下效果不如词级别的语言模型，因为模型更深，导致句子靠前位置的信息丢失严重。
2. 因为(1)的原因，导致更消耗计算资源。

### Gated Recurrent Unit (GRU)

在GRU中引入了一个新的概念memory cell来替代原先的activations（在GRU中，这两个东西是等价的，而在LSTM中不等价），我们用$c^{< t >}$来表示memory cell。下面对GRU进行形式化表述，
$$
\begin{split}
& \tilde{c}^{< t >} = \tanh( W_{c} [ \Gamma_r \ast c^{< t - 1 >}, x^{< t >}] + b_c ) \\\\
& \Gamma_{u} = \delta ( W_u [ c^{< t - 1 >}, x^{< t >} ] + b_u ) \\\\
& \Gamma_{r} = \delta ( W_r [ c^{< t - 1 >}, x^{< t >} ] + b_r ) \\\\
& c^{< t >} =  \Gamma_{u} \ast \tilde{c}^{< t >} + ( 1 - \Gamma_{u} ) \ast c^{< t - 1 >} \\\\
& a^{ < t >} = c^{< t >}
\end{split}
$$
其中，$\delta$表示sigmoid函数，$\Gamma_u$是与$ \tilde{c}^{< t >} $长度相同的向量。$\tilde{c}^{< t >}$ 是$c^{< t >}$的候选， $\Gamma_u$表示的是update gate，代表对新旧memory cell的中和取舍，可以很有效的解决vanishing gradients问题。因为 $\Gamma_u$中的元素很容易因为学习而得到一个接近0的数值（sigmoid函数，当自变量是一个很大的负数的时候），所以在更新得到$c^{< t >}$时，第一项会成为一个很小的可以忽略的项，而让$c^{< t >}$接近$c^{< t - 1 >}$。

$\Gamma_r$表示的是relavant gate，代表候选$\tilde{c}^{< t >}$与上一时刻的memory cell的相关度。

### Long Short Term Memory

LSTM是GRU的增强版本。形式化描述如下，
$$
\begin{split}
& \tilde{c}^{< t >} = \tanh( W_{c} [ c^{< t - 1 >}, x^{< t >}] + b_c ) \\\\
Update:~& \Gamma_{u} = \delta ( W_u [ c^{< t - 1 >}, x^{< t >} ] + b_u ) \\\\
Forget:~& \Gamma_{f} = \delta ( W_f [ c^{< t - 1 >}, x^{< t >} ] + b_f ) \\\\
Output:~& \Gamma_{o} = \delta ( W_o [ c^{< t - 1 >}, x^{< t >} ] + b_o ) \\\\
& c^{< t >} =  \Gamma_{u} \ast \tilde{c}^{< t >} +  \Gamma_{f}  \ast c^{< t - 1 >} \\\\
& a^{ < t >} = \Gamma_{o} \ast \tanh c^{< t >}
\end{split}
$$

### Bidirectional RNN

Bi-RNN的出现是因为在预测时候，我们只能看到历史的信息而不能看到未来的信息。为了解决这个问题，Bi-RNN采用了双RNN结构，其中一个RNN的信息从左向右流动，另一个RNN的信息从右向左流动，在预测/输出的时候，会同时考虑到两个RNN结构的输出结果，也就是，
$$
\hat{y}^{< t >} = g(W_y [a^{\rightarrow < t >}, a^{ \leftarrow < t >}] + b_y)
$$
其中，$g$表示activation function。

### Deep RNNs

Deep RNNs的目的是将若干单层RNN的结构叠加起来构造一个深层的RNN结构。不过在实际中，Deep RNNs网络的深度一般不会超过三层。举例来说明如何堆叠RNNs，
$$
a^{ [2] < 3 > } = g( W_a^{[2]} [ a^{[2]< 2 >}, a^{[1] < 3 >} ] + b_a^{[2]} )
$$
其中，$a^{ [2] < 3 > } $表示第2层在$t_3$时刻的activations。













