---
title: Convolutional Neural Networks / Week 3
date: 2018-04-06 11:24:21
tags: [cnn, deep-learning]
mathjax: true
---

### Object Localization

Object localization用来识别图像中是否包含特定对象以及该对象的位置，并最终使用一个矩形框在图像中标出该特定对象。为了简化问题，在这里我们假设图片中最多包含一个待识别的对象。下面对问题进行形式化描述。

定义目标变量$y$ (同时也是神经网络的输出层)，
$$
y = [P_c~b_x~b_y~b_h~b_w~C_1~C_2~C_3]^T
\tag{1}
$$
<!-- more -->

其中，$P_c$表示图像中是否包含特定对象，$(b_x, b_y)$表示特定对象的中心位置在图像中的坐标（图像左上角坐标为$(0,0)$，右下角坐标为$(1,1)$），$b_h,b_w$分别表示特定对象的高度和宽度，$C_1-C_3$表示特定对象的类型（行人，汽车，摩托车）。

定义损失函数$\mathcal{L(\hat{y}, y)}$，
$$
\mathcal{L}(\hat{y}, y) = 
\begin{split}
\begin{cases}
\sum_{i=1}^{i=8} (\hat{y}_i - y_i)^2,&~if~y_1=1 \\\\
(\hat{y}_1 - y_1)^2,&~if~y_1=0
\end{cases}
\end{split}
$$
这里针对不同的维度都使用了平方差损失函数，可以针对不同的维度使用不同的损失函数。

### Landmark Detection

 有时候我们需要识别图中的一些关键点的坐标，这些坐标称为Landmarks。这时候，我们可以定义如下的目标变量$y$，
$$
y = [P~l_{1x}~l_{1y}~\dots~l_{nx}~l_{ny}]^T
$$
以识别人面部眼角嘴角为例，其中$P$代表是否包含人脸，$(l_{ix}, l_{iy})$代表关键点的坐标。

### Object Detection

 Object Detction的其中一种办法叫做Sliding windows detection，采用不同尺寸的矩形框，从左至右、从上到下遍历枚举图像的子图，判断子图中是否包含需要的目标对象。很明显，这种办法比较笨，需要消耗大量的计算量。

### Convolutional Implementation of Sliding Windows

全连接是可以通过卷积来实现的，并且两者直接是等价的。例如，如果$5\times 5 \times 16$的卷积层之后接的是一个$400$个神经元的全连接层，那么它等价于$5 \times 5 \times 16$的卷积层之后采用400个$5 \times 5 \times 16$的filters得到的$1 \times 1 \times 400$的卷积层。

得益于卷积的存在，Sliding windows detection可以做到同时预测同一张图像不同子图中是否包含特定对象。但是这样做的不利条件是预测出来的对象的边界（bounding box）会相对不准确。因为这种办法采用的是子图的边界来作为待预测对象的边界。 

### Bounding Box Predictions

这里介绍了YOLO algorithm (You Only Look Once)，该算法用来识别同一张图像上的多个目标简单。它将图像切分为了$M \times N$的网格并在此基础上构造了卷积神经网络。该网络的输入依然为整张图片，切分并不影响输入，而是决定了网络的输出尺寸为$M \times N \times 8$。这样，每个子图就拥有了一个$1 \times 1 \times 8$的预测结果，用来表示图像中是否包含特定的对象，如果包含的话，该特定对象的中心位置、长宽以及类别分别是什么。

该算法利用了卷积操作提高了对同一张图像上不同子图的模型训练预测的效率，使得一次训练就可以完成对多个子图的建模（这里有个假设，就是每个子图上只包含最多一个特定对象）。

### Intersection Over Union

$$
Intersection over Union (loU) = \frac{size~of~intersection}{size~of~union}
$$

通过loU，我们可以知道两个矩形在大小和位置上的相像程度。这样，我们就可以用它来评价object detection算法的优劣。

### Non-max Suppression

有时候，我们的算法会将相同的对象识别多次，non-max suppression算法用来解决这个问题。举例，

假设卷积神经网络最后的输出为$19 \times 19 \times 5$，也就是说图像被切分为了$19 \times 19$的子图，每个子图的预测结果为一个5维的向量，该向量如下，
$$
y = [p_c~b_x~b_y~b_h~b_w]^T
$$
那么，在训练结束之后，non-max suppression算法会执行如下步骤，

1. 扔掉所有$p_c \le 0.6$的bounding boxes
2. 取出剩余bounding boxes中$p_c$最大的那个bounding box，作为新检测到的目标
3. 删除剩余所有与该box的loU值$\ge 0.5$的bounding boxes
4. 重复(2-3)步，直到没有bounding boxes剩余

从上可以看出，non-max suppression其实是个简单的贪心算法。

### Anchor Boxes

在Object detection问题中，还有一个难点就是图像划分出网格后，每个网格中只能最多识别一个对象。为了让单个网格识别多个对象，可以采用Anchor boxes方法。

Anchor boxes方法的思想很简单，将式(1)改为如下形式，
$$
y = [P_c~b_x~b_y~b_h~b_w~C_1~C_2~C_3~P_c~b_x~b_y~b_h~b_w~C_1~C_2~C_3]^T 
\tag{2}
$$
式(2)表示在识别的过程中采用了两个Anchor box。每个Anchor box都负责识别所有类别的对象。



