# Adam --- 使用Gluon


在`Gluon`里，使用Adam很容易。我们无需重新实现它。

```{.python .input  n=1}
import mxnet as mx
from mxnet import autograd
from mxnet import gluon
from mxnet import nd
import random

# 为方便比较同一优化算法的从零开始实现和Gluon实现，将输出保持确定。
random.seed(1)
mx.random.seed(1)

# 生成数据集。
num_inputs = 2
num_examples = 1000
true_w = [2, -3.4]
true_b = 4.2
X = nd.random_normal(scale=1, shape=(num_examples, num_inputs))
y = true_w[0] * X[:, 0] + true_w[1] * X[:, 1] + true_b
y += .01 * nd.random_normal(scale=1, shape=y.shape)

# 创建模型和定义损失函数。
net = gluon.nn.Sequential()
net.add(gluon.nn.Dense(1))
```

我们需要在`gluon.Trainer`中指定优化算法名称`adam`并设置学习率。

```{.python .input  n=2}
%matplotlib inline
%config InlineBackend.figure_format = 'retina'
import numpy as np
import sys
sys.path.append('..')
import utils
```

使用Adam，最终学到的参数值与真实值较接近。

```{.python .input  n=3}
net.collect_params().initialize(mx.init.Normal(sigma=1), force_reinit=True)
trainer = gluon.Trainer(net.collect_params(), 'adam',
                        {'learning_rate': 0.1})
utils.optimize(batch_size=10, trainer=trainer, num_epochs=3, decay_epoch=None,
               log_interval=10, X=X, y=y, net=net)
```

## 结论

* 使用`Gluon`的`Trainer`可以轻松使用Adam。



## 练习

* 试着使用其他Adam初始学习率，观察实验结果。



## 赠诗一首以总结优化章节


> 梯度下降可沉甸，  随机降低方差难。

> 引入动量别弯慢，  Adagrad梯方贪。

> Adadelta学率换， RMSProp梯方权。

> Adam动量RMS伴，  优化还需己调参。


注释：

* 梯方：梯度按元素平方
* 贪：因贪婪故而不断累加
* 学率：学习率
* 换：这个参数被换成别的了
* 权：指数加权移动平均

**吐槽和讨论欢迎点**[这里](https://discuss.gluon.ai/t/topic/2280)
