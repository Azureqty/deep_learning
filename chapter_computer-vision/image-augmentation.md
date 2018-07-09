# 图片增广

在[“深度卷积神经网络：AlexNet”](../chapter_convolutional-neural-networks/alexnet.md)小节里我们提到过，大规模数据集是成功使用深度网络的前提。图片增广（image augmentation）技术通过对训练图片做一系列随机变化，来产生相似但又有不同的训练样本，从而扩大训练数据集规模。图片增广的另一种解释是，通过对训练样本做一些随机变形，可以降低模型对某些属性的依赖，从而提高泛化能力。例如我们可以对图片进行不同的裁剪，使得感兴趣的物体出现在不同的位置中，从而使得模型减小对物体出现位置的依赖性。也可以调整亮度色彩等因素来降低模型对色彩的敏感度。在AlexNet的成功中，图片增广技术功不可没。本小节我们将讨论这个在计算机视觉里被广泛使用的技术。

## 常用增广方法

我们首先读取一张$400\times 500$的图片作为样例。

```{.python .input  n=1}
%matplotlib inline
import sys
sys.path.insert(0, '..')
import gluonbook as gb
from mxnet import nd, image, gluon, init
from mxnet.gluon.data.vision import transforms

img = image.imread('../img/cat1.jpg')
gb.plt.imshow(img.asnumpy())
```

因为大部分的增广方法都有一定的随机性。接下来我们定义一个辅助函数，它对输入图片`img`运行多次增广方法`aug`并显示所有结果。

```{.python .input  n=2}
def apply(img, aug, num_rows=2, num_cols=4, scale=1.5):
    Y = [aug(img) for _ in range(num_rows*num_cols)]
    gb.show_images(Y, num_rows, num_cols, scale)
```

### 变形

左右翻转图片通常不物体的类别，它是最早也是最广泛使用的一种增广。下面我们使用transform模块里的`RandomFlipLeftRight`类来实现按0.5的概率左右翻转图片：

```{.python .input  n=3}
apply(img, transforms.RandomFlipLeftRight())
```

上下翻转不如水平翻转通用，但是至少对于样例图片，上下翻转不会造成识别障碍。

```{.python .input  n=4}
apply(img, transforms.RandomFlipTopBottom())
```

我们使用的样例图片里，猫在图片正中间，但一般情况下可能不是这样。[“池化层”](../chapter_convolutional-neural-networks/pooling.md)一节里我们解释了池化层能弱化卷积层对目标位置的敏感度，另一方面我们可以通过对图片随机剪裁来让物体以不同的比例出现在不同位置。

下面代码里我们每次随机裁剪一片面积为原面积10%到100%的区域，其宽和高的比例在0.5和2之间，然后再将高宽缩放到200像素大小。

```{.python .input  n=5}
shape_aug = transforms.RandomResizedCrop(
    (200, 200), scale=(0.1, 1), ratio=(0.5, 2))
apply(img, shape_aug)
```

### 颜色变化

另一类增广方法是变化颜色。我们可以从四个维度改变图片的颜色：亮度、对比、饱和度和色相。在下面的例子里，我们随机亮度改为原图的50%到150%。

```{.python .input  n=6}
apply(img, transforms.RandomBrightness(0.5))
```

类似的，我们可以修改色相。

```{.python .input  n=7}
apply(img, transforms.RandomHue(0.5))
```

或者用使用`RandomColorJitter`来一起使用。

```{.python .input  n=8}
color_aug = transforms.RandomColorJitter(
    brightness=0.5, contrast=0.5, saturation=0.5, hue=0.5)
apply(img, color_aug)
```

### 使用多个增广

实际应用中我们会将多个增广叠加使用。Compose类可以将多个增广串联起来。

```{.python .input  n=9}
augs = transforms.Compose([
    transforms.RandomFlipLeftRight(), color_aug, shape_aug])
apply(img, augs)
```

## 使用图片增广来训练

接下来我们来看一个将图片增广应用在实际训练中的例子，并比较其与不使用时的区别。这里我们使用CIFAR-10数据集，而不是之前我们一直使用的FashionMNIST。原因在于FashionMNIST中物体位置和尺寸都已经归一化了，而CIFAR-10中物体颜色和大小区别更加显著。下面我们展示CIFAR-10中的前32张训练图片。

```{.python .input  n=10}
gb.show_images(gluon.data.vision.CIFAR10(train=True)[0:32][0], 4, 8,
               scale=0.8);
```

我们通常将图片增广用在训练样本上，但是在预测的时候并不使用随机增广。这里我们仅仅使用最简单的随机水平翻转。此外，我们使用`ToTensor`变换来将图片转成MXNet需要的格式，即格式为（批量，通道，高，宽）以及类型为32位浮点数。

```{.python .input  n=11}
train_augs = transforms.Compose([
    transforms.RandomFlipLeftRight(),
    transforms.ToTensor(),
])

test_augs = transforms.Compose([
    transforms.ToTensor(),
])
```

接下来我们定义一个辅助函数来方便读取图片并应用增广。Gluon的数据集提供`transform_first`函数来对数据里面的第一项（数据一般有图片和标签两项）来应用增广。另外图片增广将增加计算复杂度，我们使用两个额外CPU进程加来加速计算。

```{.python .input  n=12}
def load_cifar10(is_train, augs, batch_size):
    return gluon.data.DataLoader(gluon.data.vision.CIFAR10(
        train=is_train).transform_first(augs),
        batch_size=batch_size, shuffle=is_train, num_workers=2)
```

### 模型训练

我们使用ResNet 18来训练CIFAR-10。训练的的代码与[“残差网络：ResNet”](../chapter_convolutional-neural-networks/resnet.md)中一致，除了使用所有可用的GPU和不同的学习率外。

```{.python .input  n=13}
def train(train_augs, test_augs, lr=0.1):
    batch_size = 256
    ctx = gb.try_all_gpus()
    net = gb.resnet18(10)
    net.initialize(ctx=ctx, init=init.Xavier())
    trainer = gluon.Trainer(net.collect_params(), 'sgd', {'learning_rate':lr})
    loss = gluon.loss.SoftmaxCrossEntropyLoss()
    train_data = load_cifar10(True, train_augs, batch_size)
    test_data = load_cifar10(False, test_augs, batch_size)
    gb.train(train_data, test_data, net, loss, trainer, ctx, num_epochs=8)
```

首先我们看使用了图片增广的情况。

```{.python .input  n=14}
train(train_augs, test_augs)
```

作为对比，我们尝试只对训练数据做中间剪裁。

```{.python .input  n=15}
train(test_augs, test_augs)
```

可以看到，即使是简单的随机翻转也会有明显的效果。图片增广类似于正则项，它使得训练精度变低，但可以提高测试精度。

## 小结

* 图片增广基于现有训练数据生成大量随机图片来有效避免过拟合。

## 练习

* 尝试在CIFAR-10训练中增加不同的增广方法。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/1666)

![](../img/qr_image-augmentation.svg)
