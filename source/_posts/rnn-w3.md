---
title: Recurrent Neural Networks / Week 3
date: 2018-04-21 10:49:44
tags: [deep-learning, rnn]
mathjax: true
---

### Basic Models

Sequence to sequence mdoel的构成：
$$
Encode~Network \rightarrow Decode~Network
$$
它的思想可以用在Image captioning上，结构如下，
$$
ConvNet (delete~softmax~layer) \rightarrow Decode Network
$$
<!-- more -->

### Picking the most likely sentence

Machine translation与Language model在网络结构上有相似之处，他俩的不同之处在于，

* Language model在生成句子的过程中，有根据概率分布随机挑选下一个词语的过程。
* Machine tranlation在生成句子的过程中，不会根据概率分布随机挑选下一个词语，而是会选择概率最大的那个翻译结果的句子。

考虑到结果的空间复杂度，所以算法不可能去枚举计算每一种可能结果的概率，然后选择最大的，所以通常都会求近似最优解。在寻找近似最优解的过程中，并没有使用贪心算法。

考虑以下两个翻译结果，

1. Jane is visiting Africa in September.
2. Jane is going to be visiting Africa in September.

很明显，句(1)的翻译质量高于句(2)。如果我们采用贪心策略的话，在计算第3个词语的时候，$P(going | Jane~is)$的概率是要高于$P(visiting|Jane~is)$的，因为going这个词比visiting要常见。所以如果单纯的采用贪心算法，很容易陷入局部最优解。

### Beam Search

Beam Search是为了解决在Machine translation中如果使用贪心的策略来寻找最有可能的翻译结果的话容易陷入局部最优解的问题。

Beam search算法设置了Beam width作为超参数(简记B)：

1. 在计算$p(y^{< 1 >} |x)$的时候，Beam search会从词表中挑选B个最有可能出现的词语作为备选（贪心策略只选一个）。
2. 在计算$p(y^{< 2 >} | x, y^{< 1 >})$的时候，Beam search会基于上次得到的三个结果分别挑选B个最有可能出现的词语作为备选，这样就出现了$B^2$个可能的结果。对$p(y^{< 1 >}, y^{< 2 >} | x)$进行排序，保留概率最大的B个。
3. 与(2)类似，每次计算完成后进行剪枝，只保留B个概率最大的结果，直到句子结束。

### Refinements to Beam Search

Beam search的优化目标如下，
$$
\arg \max _ {y} \prod _ {t=1}^{T_y} P(y^{< t >} | x, y^{< 1 >}, ..., y^{< t - 1 >})
$$
因为连乘的存在，所以容易浮点溢出，对上式进行改进，
$$
\arg \max_y \sum_{t=1}^{T_y} \log P(y^{< t >} | x, y^{< 1 >}, ..., y^{< t - 1 >})
\tag{1}
$$
从优化目标我们可以看出，对于结果的选择倾向于选择句子较短的，因为句子越长，式(1)中的项越多，而每一项都是负数，所以最终结果可能会更小。为了解决这个问题，所以对式(1)进行了**Length normalization**，
$$
\frac{1}{T_y^\alpha} \arg \max_y \sum_{t=1}^{T_y} \log P(y^{< t >} | x, y^{< 1 >}, ..., y^{< t - 1 >})
\tag{2}
$$
其中，$T_y$是句子的长度。$\alpha$是另一个超参数。

### Error analysis in beam search

在Machine translation中，有两个步骤构成，RNN + Beam search，那么问题来了，如果在调模型的时候，我们应该调RNN还是调Beam search呢？

从dev data set中挑选一批负例，假设人工翻译的结果在模型中的概率是$P(\hat{y}|x)$，而模型翻译的结果的概率是$P(y^{\star}|x)$，那么，

* 如果$P(\hat{y}|x) > P(y^{\star}|x)$，说明RNN是没有问题的，Beam search的过程中陷入了局部最优解。
* 如果$P(\hat{y}|x) < P(y^{\star}|x)$，说明RNN已经出现了问题，没有让人工翻译的结果概率最大。

观察dev data set中哪种情况出现的多，那么对应哪部分的问题就更大，然后着重对待那部分。

### Bleu Score

如果对于同一个句子，有若干人工翻译的结果，那么怎样基于这些人工翻译的结果来对机器翻译结果进行打分呢？Bleu score解决了这个问题。

首先定义Modified precision，
$$
P_n = \frac{ \sum_{ n-grams \in \hat{y} } Count_{clip}( n-gram ) }{ \sum_{ n-grams \in \hat{y} } Count( n-gram ) }
$$
Combined Bleu score：
$$
BP~\exp( \frac{1}{4} \sum_{n=1}^4 P_n)
$$
其中，BP表示brevity penalty，
$$
BP = 
\begin{split}
\begin{cases}
1,~if~MT\_output\_length > reference\_output\_length \\\\
\exp(1 - MT\_output\_length / reference\_output\_length), ~otherwith
\end{cases}
\end{split}
$$

BP的出现是因为Combined Bleu score倾向于选择长度更短的翻译结果，因为这样机器翻译结果中的词语更有可能在人工翻译的结果中都出现（多说多错，少说少错）。

### Attention Model Intuition

在Machine translation模型（Encoder-Decoder）中，会先使用Encoder学习待翻译的文本的表达，再在Decoder中生成翻译后的句子。这样的模型对于短文本来说是很有效的，但是在长文本翻译中，会因为整句读取完毕之后再翻译会“遗忘”掉之前的信息而导致翻译效果下降。所以，Attention model引入了$\alpha^{< t, t' >}$来表示在生成第$t$个单词时，你需要对原文的第$t'$个单词付出多少的关注度。

### Attention Model

 在Attetion model中，我们定义了$\alpha^{< t, t' >}$，它表示的是，

> Amount of "attention" $y^{< t >}$ should pay to $a^{< t' > }$.

通过下式计算得到，
$$
\alpha^{< t, t' >} = \frac{ \exp (e^{ <t, t'> }) }{ \sum_{t'=1}^{T_x} \exp (e^{ <t, t'> }) }
$$
其中，$ e^{ <t, t'> } $可以通过下面的简单网络结构得到，

<img src="/img/rnn-w3/alpha-cal.jpg" width="50%" align=center />

### Speech Recognition

在Speech recognition中有一个很有意思的方法叫做CTC cost。

我们都知道，语音识别中，$T_x$要远大于$T_y$（假设一段10s的音频，采集频率是100帧，那么输入的长度就是1000，而输出的字母个数远小于1000）。CTC cost允许输出重复的字母以及空白符等，这样就允许输出的长度与输入的长度一致，之后再进行输出的压缩。

### Conclusion

终终终终终终终终终终终终终终于，学完了！菜鸡从刚开始跟不上字幕，到后面可以脱离字幕也是不容易。Pursue your dreams！