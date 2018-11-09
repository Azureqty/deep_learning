# 区域卷积神经网络（R-CNN）系列


区域卷积神经网络（region-based CNN或regions with CNN features，简称R-CNN）是将深度模型应用于目标检测的开创性工作之一 [1]。本节中，我们将介绍R-CNN和它的一系列改进方法：快速的R-CNN（fast R-CNN）[3]、更快的R-CNN（faster R-CNN）[4] 以及掩码R-CNN（mask R-CNN）[5]。限于篇幅，这里只介绍这些模型的设计思路。


## R-CNN

R-CNN首先对图像选取若干提议区域（例如锚框也是一种选取方法）并标注它们的类别和边界框（例如偏移量）。然后，用卷积神经网络对每个提议区域做前向计算抽取特征。之后，我们用每个提议区域的特征预测类别和边界框。图9.5描述了R-CNN模型。

![R-CNN模型。](../img/r-cnn.svg)

具体来说，R-CNN主要由以下四步构成：

1. 对输入图像使用选择性搜索（selective search）来选取多个高质量的提议区域 [2]。这些提议区域通常是在多个尺度下选取的，并具有不同的形状和大小。每个提议区域将被标注类别和真实边界框。
1. 选取一个预训练的卷积神经网络，并将其在输出层之前截断。将每个提议区域变形为网络需要的输入尺寸，并通过前向计算输出抽取的提议区域特征。
1. 将每个提议区域的特征连同其标注的类别作为一个样本，训练多个支持向量机对目标分类。其中每个支持向量机用来判断样本是否属于某一个类别。
1. 将每个提议区域的特征连同其标注的边界框作为一个样本，训练线性回归模型来预测真实边界框。

R-CNN虽然通过预训练的卷积神经网络有效抽取了图像特征，但它的主要缺点是速度慢。想象一下，我们可能从一张图像中选出上千个提议区域，对该图像做目标检测将导致上千次的卷积神经网络的前向计算。这个巨大的计算量令R-CNN难以在实际应用中被广泛采用。


## Fast R-CNN

R-CNN的主要性能瓶颈在于需要对每个提议区域独立抽取特征。由于这些区域通常有大量重叠，独立的特征抽取会导致大量的重复计算。Fast R-CNN对R-CNN的一个主要改进在于只对整个图像做卷积神经网络的前向计算。

![Fast R-CNN模型。](../img/fast-rcnn.svg)

图9.6描述了Fast R-CNN模型。它的主要计算步骤如下：

1. 与R-CNN相比，Fast R-CNN用来提取特征的卷积神经网络的输入是整个图像，而不是各个提议区域。而且，这个网络通常会参与训练，即更新模型参数。设输入为一张图像，将卷积神经网络的输出的形状记为$1 \times c \times h_1 \times w_1$。
1. 假设选择性搜索生成$n$个提议区域。这些形状各异的提议区域在卷积神经网络的输出上分别标出形状各异的兴趣区域。这些兴趣区域需要抽取出形状相同的特征（假设高和宽均分别指定为$h_2,w_2$），从而便于连结。Fast R-CNN引入兴趣区域池化层（Region of Interest Pooling，简称RoI池化层），将卷积神经网络的输出和提议区域作为输入，输出连结后的各个提议区域抽取的特征，形状为$n \times c \times h_2 \times w_2$。
1. 通过全连接层将输出形状变换为$n \times d$，其中$d$是超参数。
1. 类别预测时，将全连接层的输出的形状再变换为$n \times q$并使用softmax回归（$q$为类别个数）。边界框预测时，将全连接层的输出的形状再变换为$n \times 4$。也就是说，我们为每个提议区域预测类别和边界框。

Fast R-CNN中提出的兴趣区域池化层跟我们之前介绍过的池化层有所不同。在池化层中，我们通过设置池化窗口、填充和步幅来控制输出形状。而兴趣区域池化层对每个区域的输出形状是可以直接指定的，例如指定每个区域输出的高和宽分别为$h_2,w_2$。假设某一兴趣区域窗口的高和宽分别为$h$和$w$，该窗口将被划分为形状为$h_2 \times w_2$的子窗口网格，且每个子窗口的大小大约为$(h/h_2) \times (w/w_2)$。任一子窗口的高和宽要取整，其中的最大元素作为该子窗口的输出。因此，兴趣区域池化层可从形状各异的兴趣区域中均抽取出形状相同的特征。

图9.7中，我们在$4 \times 4$的输入上选取了左上角的$3\times 3$区域作为兴趣区域。对于该兴趣区域，我们通过$2\times 2$的兴趣区域池化层得到一个$2\times 2$的输出。四个划分后的子窗口分别含有元素0、1、4、5（5最大），2、6（6最大），8、9（9最大），10。

![$2\times 2$兴趣区域池化层。](../img/roi.svg)

我们使用`ROIPooling`函数来演示兴趣区域池化层的计算。假设卷积神经网络抽取的特征`X`的高和宽均为4且只有单通道。

```{.python .input  n=4}
from mxnet import nd

X = nd.arange(16).reshape((1, 1, 4, 4))
X
```

假设图像的高和宽均为40像素。假设选择性搜索在图像上生成了两个提议区域：每个区域由五个元素表示，分别为区域目标类别、左上角的$x,y$轴坐标和右下角的$x,y$轴坐标。

```{.python .input  n=5}
rois = nd.array([[0, 0, 0, 20, 20], [0, 0, 10, 30, 30]])
```

由于`X`的高和宽是图像高和宽的$1/10$，以上两个提议区域中的坐标先按`spatial_scale`自乘0.1，然后在`X`上分别标出兴趣区域`X[:,:,0:3,0:3]`和`X[:,:,1:4,0:4]`。最后对这两个兴趣区域分别划分子窗口网格并抽取高和宽为2的特征。

```{.python .input  n=6}
nd.ROIPooling(X, rois, pooled_size=(2, 2), spatial_scale=0.1)
```

## Faster R-CNN：更快速的区域卷积神经网络

Faster R-CNN 对Fast R-CNN做了进一步改进，它将Fast R-CNN中的选择性搜索替换成区域提议网络（region proposal network，简称RPN）[4]。RPN以锚框为起始点，通过一个小神经网络来选择提议区域。图9.8描述了Faster R-CNN模型。

![Faster R-CNN模型。](../img/faster-rcnn.svg)

具体来说，RPN里面有四个神经层。

1. 卷积网络抽取的特征首先进入一个填充数为1、通道数为256的 $3\times 3$ 卷积层，这样每个像素得到一个256长度的特征表示。
1. 以每个像素为中心，生成多个大小和比例不同的锚框和对应的标注。每个锚框使用其中心像素对应的256维特征来表示。
1. 在锚框特征和标注上面训练一个两类分类器，判断其含有感兴趣目标还是只有背景。
1. 对每个被判断成含有目标的锚框，进一步预测其边界框，然后进入RoI池化层。

可以看到RPN通过标注来学习预测跟真实边界框更相近的提议区域，从而减小提议区域的数量同时保证最终模型的预测精度。

## Mask R-CNN：使用全连接卷积网络的Faster RCNN

如果训练数据中我们标注了每个目标的精确边框，而不是一个简单的方形边界框，那么Mask R-CNN能有效的利用这些详尽的标注信息来进一步提升目标识别精度 [5]。具体来说，Mask R-CNN使用额外的全连接卷积网络来利用像素级别标注信息，这个网络将在稍后的[“语义分割”](fcn.md)这一节做详细介绍。图9.9描述了Mask R-CNN模型。

![Mask R-CNN模型。](../img/mask-rcnn.svg)

注意到RPN输出的是实数坐标的提议区域，在输入到RoI池化层时我们将实数坐标定点化成整数来确定区域中的像素。在计算过程中，我们将每个区域分割成多块然后同样定点化区域边缘到最近的像素上。这两步定点化会使得定点化后的边缘和原始区域中定义的有数个像素的偏差，这个对于边界框预测来说问题不大，但在像素级别的预测上则会带来麻烦。

Mask R-CNN中提出了RoI对齐层（RoI Align）。它去掉了RoI池化层中的定点化过程，从而使得不管是输入的提议区域还是其分割区域的坐标均使用实数。如果边界不是整数，那么其元素值则通过相邻像素插值而来。

然后有

$$f(x,y) = (\lfloor y \rfloor + 1-y)f(x, \lfloor y \rfloor) + (y-\lfloor y \rfloor)f(x, \lfloor y \rfloor + 1).$$


## 小结

* R-CNN对每张图像选取多个提议区域，然后使用卷积层来对每个区域抽取特征，之后对每个区域进行目标分类和真实边界框预测。
* Fast R-CNN对整个图像进行特征抽取后再选取提议区域来提升计算性能，它引入了兴趣区域池化层将每个提议区域提取同样大小的输出以便输入之后的神经层。
* Faster R-CNN引入区域提议网络来进一步简化区域提议流程。
* Mask R-CNN在Faster R-CNN基础上进入一个全卷积网络可以借助像素粒度的标注来进一步提升模型精度。


## 练习

* 介于篇幅原因这里没有提供R-CNN系列模型的实现。有兴趣的读者可以参考Gluon CV工具包（https://gluon-cv.mxnet.io/ ）来学习它们的实现。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/7219)

![](../img/qr_rcnn.svg)



## 参考文献

[1] Girshick, R., Donahue, J., Darrell, T., & Malik, J. (2014). Rich feature hierarchies for accurate object detection and semantic segmentation. In Proceedings of the IEEE conference on computer vision and pattern recognition (pp. 580-587).

[2] Uijlings, J. R., Van De Sande, K. E., Gevers, T., & Smeulders, A. W. (2013). Selective search for object recognition. International journal of computer vision, 104(2), 154-171.

[3] Girshick, R. (2015). Fast r-cnn. arXiv preprint arXiv:1504.08083.

[4] Ren, S., He, K., Girshick, R., & Sun, J. (2015). Faster r-cnn: Towards real-time object detection with region proposal networks. In Advances in neural information processing systems (pp. 91-99).

[5] He, K., Gkioxari, G., Dollár, P., & Girshick, R. (2017, October). Mask r-cnn. In Computer Vision (ICCV), 2017 IEEE International Conference on (pp. 2980-2988). IEEE.
