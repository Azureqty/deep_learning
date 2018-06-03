# 长短期记忆（LSTM）——从零开始


本节将介绍另一种常用的门控循环神经网络：长短期记忆（long short-term memory，简称LSTM）[1]。它比门控循环单元的结构稍微更复杂一点。


## 长短期记忆

我们先介绍长短期记忆的构造。长短期记忆的隐藏状态包括隐藏层变量$\boldsymbol{H}$和细胞$\boldsymbol{C}$（也称记忆细胞）。它们形状相同。


### 输入门、遗忘门和输出门


假定隐藏状态长度为$h$，给定时间步$t$的一个样本数为$n$特征向量维度为$x$的批量数据$\boldsymbol{X}_t \in \mathbb{R}^{n \times x}$和上一时间步隐藏状态$\boldsymbol{H}_{t-1} \in \mathbb{R}^{n \times h}$，输入门（input gate）$\boldsymbol{I}_t \in \mathbb{R}^{n \times h}$、遗忘门（forget gate）$\boldsymbol{F}_t \in \mathbb{R}^{n \times h}$和输出门（output gate）$\boldsymbol{O}_t \in \mathbb{R}^{n \times h}$的定义如下：

$$\boldsymbol{I}_t = \sigma(\boldsymbol{X}_t \boldsymbol{W}_{xi} + \boldsymbol{H}_{t-1} \boldsymbol{W}_{hi} + \boldsymbol{b}_i)$$

$$\boldsymbol{F}_t = \sigma(\boldsymbol{X}_t \boldsymbol{W}_{xf} + \boldsymbol{H}_{t-1} \boldsymbol{W}_{hf} + \boldsymbol{b}_f)$$

$$\boldsymbol{O}_t = \sigma(\boldsymbol{X}_t \boldsymbol{W}_{xo} + \boldsymbol{H}_{t-1} \boldsymbol{W}_{ho} + \boldsymbol{b}_o)$$

其中的$\boldsymbol{W}_{xi}, \boldsymbol{W}_{xf}, \boldsymbol{W}_{xo} \in \mathbb{R}^{x \times h}$和$\boldsymbol{W}_{hi}, \boldsymbol{W}_{hf}, \boldsymbol{W}_{ho} \in \mathbb{R}^{h \times h}$是可学习的权重参数，$\boldsymbol{b}_i, \boldsymbol{b}_f, \boldsymbol{b}_o \in \mathbb{R}^{1 \times h}$是可学习的偏移参数。函数$\sigma$自变量中的三项相加使用了[广播](../chapter_crashcourse/ndarray.md)。

和[门控循环单元](gru-scratch.md)中的重置门和更新门一样，这里的输入门、遗忘门和输出门中每个元素的值域都是$[0, 1]$。


### 候选细胞

和[门控循环单元](gru-scratch.md)中的候选隐藏状态一样，长短期记忆中的候选细胞$\tilde{\boldsymbol{C}}_t \in \mathbb{R}^{n \times h}$也使用了值域在$[-1, 1]$的双曲正切函数tanh做激活函数：

$$\tilde{\boldsymbol{C}}_t = \text{tanh}(\boldsymbol{X}_t \boldsymbol{W}_{xc} + \boldsymbol{H}_{t-1} \boldsymbol{W}_{hc} + \boldsymbol{b}_c)$$

其中的$\boldsymbol{W}_{xc} \in \mathbb{R}^{x \times h}$和$\boldsymbol{W}_{hc} \in \mathbb{R}^{h \times h}$是可学习的权重参数，$\boldsymbol{b}_c \in \mathbb{R}^{1 \times h}$是可学习的偏移参数。


### 细胞

我们可以通过元素值域在$[0, 1]$的输入门、遗忘门和输出门来控制隐藏状态中信息的流动：这通常可以应用按元素乘法符$\odot$。当前时间步细胞$\boldsymbol{C}_t \in \mathbb{R}^{n \times h}$的计算组合了上一时间步细胞和当前时间步候选细胞的信息，并通过遗忘门和输入门来控制信息的流动：

$$\boldsymbol{C}_t = \boldsymbol{F}_t \odot \boldsymbol{C}_{t-1} + \boldsymbol{I}_t \odot \tilde{\boldsymbol{C}}_t$$

需要注意的是，如果遗忘门一直近似1且输入门一直近似0，过去的细胞将一直通过时间保存并传递至当前时间步。这个设计可以应对循环神经网络中的梯度衰减问题，并更好地捕捉时序数据中间隔较大的依赖关系。


### 隐藏状态

有了细胞以后，接下来我们还可以通过输出门来控制从细胞到隐藏层变量$\boldsymbol{H}_t \in \mathbb{R}^{n \times h}$的信息的流动：

$$\boldsymbol{H}_t = \boldsymbol{O}_t \odot \text{tanh}(\boldsymbol{C}_t)$$

需要注意的是，当输出门近似1，细胞信息将传递到隐藏层变量；当输出门近似0，细胞信息只自己保留。



输出层的设计可参照[循环神经网络](rnn-scratch.md)中的描述。


## 实验


为了实现并展示门控循环单元，我们依然使用周杰伦歌词数据集来训练模型作词。这里除长短期记忆以外的实现已在[循环神经网络](rnn-scratch.md)中介绍。


### 数据处理

我们先读取并对数据集做简单处理。

```{.python .input}
import sys
sys.path.append('..')
import gluonbook as gb
from mxnet import nd
import zipfile
```

```{.python .input  n=1}
with zipfile.ZipFile('../data/jaychou_lyrics.txt.zip', 'r') as zin:
    zin.extractall('../data/')
with open('../data/jaychou_lyrics.txt') as f:
    corpus_chars = f.read()

corpus_chars = corpus_chars.replace('\n', ' ').replace('\r', ' ')
corpus_chars = corpus_chars[0:20000]
idx_to_char = list(set(corpus_chars))
char_to_idx = dict([(char, i) for i, char in enumerate(idx_to_char)])
corpus_indices = [char_to_idx[char] for char in corpus_chars]
vocab_size = len(char_to_idx)
```

我们使用onehot来将字符索引表示成向量。

### 初始化模型参数

以下部分对模型参数进行初始化。参数`hidden_dim`定义了隐含状态的长度。

```{.python .input  n=3}
ctx = gb.try_gpu()
input_dim = vocab_size
num_hiddens = 256
output_dim = vocab_size

def get_params():
    # 输入门参数.
    W_xi = nd.random_normal(scale=0.01, shape=(input_dim, num_hiddens),
                            ctx=ctx)
    W_hi = nd.random_normal(scale=0.01, shape=(num_hiddens, num_hiddens),
                            ctx=ctx)
    b_i = nd.zeros(num_hiddens, ctx=ctx)
    # 遗忘门参数。
    W_xf = nd.random_normal(scale=0.01, shape=(input_dim, num_hiddens),
                            ctx=ctx)
    W_hf = nd.random_normal(scale=0.01, shape=(num_hiddens, num_hiddens),
                            ctx=ctx)
    b_f = nd.zeros(num_hiddens, ctx=ctx)
    # 输出门参数。
    W_xo = nd.random_normal(scale=0.01, shape=(input_dim, num_hiddens),
                            ctx=ctx)
    W_ho = nd.random_normal(scale=0.01, shape=(num_hiddens, num_hiddens),
                            ctx=ctx)
    b_o = nd.zeros(num_hiddens, ctx=ctx)
    # 候选细胞参数。
    W_xc = nd.random_normal(scale=0.01, shape=(input_dim, num_hiddens),
                            ctx=ctx)
    W_hc = nd.random_normal(scale=0.01, shape=(num_hiddens, num_hiddens),
                            ctx=ctx)
    b_c = nd.zeros(num_hiddens, ctx=ctx)
    # 输出层参数。
    W_hy = nd.random_normal(scale=0.01, shape=(num_hiddens, output_dim),
                            ctx=ctx)
    b_y = nd.zeros(output_dim, ctx=ctx)

    params = [W_xi, W_hi, b_i, W_xf, W_hf, b_f, W_xo, W_ho, b_o, W_xc, W_hc,
              b_c, W_hy, b_y]
    for param in params:
        param.attach_grad()
    return params
```

## 定义模型

我们将前面的模型公式翻译成代码。

```{.python .input  n=4}
def lstm_rnn(inputs, state_h, state_c, *params):
    [W_xi, W_hi, b_i, W_xf, W_hf, b_f, W_xo, W_ho, b_o, W_xc, W_hc, b_c,
     W_hy, b_y] = params
    H = state_h
    C = state_c
    outputs = []
    for X in inputs:        
        I = nd.sigmoid(nd.dot(X, W_xi) + nd.dot(H, W_hi) + b_i)
        F = nd.sigmoid(nd.dot(X, W_xf) + nd.dot(H, W_hf) + b_f)
        O = nd.sigmoid(nd.dot(X, W_xo) + nd.dot(H, W_ho) + b_o)
        C_tilda = nd.tanh(nd.dot(X, W_xc) + nd.dot(H, W_hc) + b_c)
        C = F * C + I * C_tilda
        H = O * nd.tanh(C)
        Y = nd.dot(H, W_hy) + b_y
        outputs.append(Y)
    return (outputs, H, C)
```

### 训练模型

下面我们开始训练模型。我们假定谱写歌词的前缀分别为“分开”、“不分开”和“战争中部队”。这里采用的是相邻批量采样实验门控循环单元谱写歌词。

```{.python .input  n=5}
get_inputs = gb.to_onehot
num_epochs = 150
num_steps = 35
batch_size = 32
lr = 0.25
clipping_theta = 5
prefixes = ['分开', '不分开']
pred_period = 30
pred_len = 100

gb.train_and_predict_rnn(lstm_rnn, False, num_epochs, num_steps, num_hiddens,
                         lr, clipping_theta, batch_size, vocab_size,
                         pred_period, pred_len, prefixes, get_params,
                         get_inputs, ctx, corpus_indices, idx_to_char,
                         char_to_idx, is_lstm=True)
```

## 小结

* 长短期记忆的提出是为了更好地捕捉时间序列中时间步距离较大的依赖关系。


## 练习

* 调调超参数，观察并分析对运行时间、困惑度以及创作歌词的结果造成的影响。
* 在相同条件下，比较长短期记忆、门控循环单元和循环神经网络的运行时间。

## 扫码直达[讨论区](https://discuss.gluon.ai/t/topic/4042)

![](../img/qr_lstm-scratch.svg)

## 参考文献

[1] Hochreiter, Sepp, and Jürgen Schmidhuber. "Long short-term memory." Neural computation 9.8 (1997): 1735-1780.

