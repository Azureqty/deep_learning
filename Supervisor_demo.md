# ΢�ż��MXNetѵ��
>��ϵ����szha�ظ�Ӱ�죬���˰���ʱ�佫��̳���Ӹ�Ϊcallback��ʵ����һ���ܡ����Խ�����У����Ͼ����֣�����bug���١�contributeϣ��MXNe
t�Ŷ��ܹ�����֧�֣�����

## �����Ķ�

�ڹ�����utils.py���������¸Ķ���

1.����itchat��threading

```{.python .input  n=1}
#pip install itchat
import itchat
import threading
```

2.��ӻ�������

```{.python .input  n=2}
lock = threading.Lock()
running = False

batch_size = 256
learning_rate = 0.5
training_iters = 2
```

3.����`lock_start()`��`chat_inf()`��`lock_end()`��`chat_supervisor()`

## ����ʵ��

��[��������� �� ʹ��Gluon](https://zh.gluon.ai/cnn-gluon.html)һ��Ϊ�������ǽ�ʵ��cnn��itchat��ϡ�

���ȣ���ԭ�������ѵ�����ַ�װ��`nn_train`��,���м���`utils.lock_start()`��`utils.chat_inf()`��`utils.l
ock_end()`��Ϊ���ƺ������

```{.python .input  n=3}
def nn_train(wechat_name,params):
    learning_rate, training_iters, batch_size = params
    train_data, test_data = utils.load_data_fashion_mnist(batch_size)
    softmax_cross_entropy = gluon.loss.SoftmaxCrossEntropyLoss()
    net = nn.Sequential()
    with net.name_scope():
        net.add(nn.Conv2D(channels=20, kernel_size=5, activation='relu'))
        net.add(nn.MaxPool2D(pool_size=2, strides=2))
        net.add(nn.Conv2D(channels=50, kernel_size=3, activation='relu'))
        net.add(nn.MaxPool2D(pool_size=2, strides=2))
        net.add(nn.Flatten())
        net.add(nn.Dense(128, activation="relu"))
        net.add(nn.Dense(10))
    ctx = utils.try_gpu()
    net.initialize(ctx=ctx)
    trainer = gluon.Trainer(net.collect_params(),'sgd',{'learning_rate':learning_rate})
    epoch = 1
    while utils.lock_start() and epoch < training_iters:
        train_loss = 0
        train_acc = 0
        for data,label in train_data:
            label = label.as_in_context(ctx)
            with autograd.record():
                output = net(data.as_in_context(ctx))
                loss = softmax_cross_entropy(output,label)
            loss.backward()
            trainer.step(batch_size)
            train_acc += utils.accuracy(output,label)
            train_loss += nd.mean(loss).asscalar()
        test_acc = utils.evaluate_accuracy(test_data, net, ctx)
        print("Epoch %d. Loss: %f, Train acc %f, Test acc %f\n" % (epoch, train_loss/len(train_data),train_acc/len(train_data), test_acc))
        utils.chat_inf(wechat_name,epoch,train_loss,train_acc,train_data,test_acc)
        epoch += 1
    utils.lock_end(wechat_name)
```

2.Ȼ����main��������� `utils.chat_supervisor()` ��ʵ�ֽ�����

```{.python .input  n=4}
if __name__ == '__main__':
    utils.chat_supervisor(nn_train)
    itchat.auto_login(hotReload=True)
    itchat.run()
```

3.�����������£�

```{.python .input  n=5}
from mxnet.gluon import nn
import utils
import itchat
from mxnet import autograd
from mxnet import gluon
from mxnet import nd

def nn_train(wechat_name,params):
    learning_rate, training_iters, batch_size = params
    train_data, test_data = utils.load_data_fashion_mnist(batch_size)
    softmax_cross_entropy = gluon.loss.SoftmaxCrossEntropyLoss()
    net = nn.Sequential()
    with net.name_scope():
        net.add(nn.Conv2D(channels=20, kernel_size=5, activation='relu'))
        net.add(nn.MaxPool2D(pool_size=2, strides=2))
        net.add(nn.Conv2D(channels=50, kernel_size=3, activation='relu'))
        net.add(nn.MaxPool2D(pool_size=2, strides=2))
        net.add(nn.Flatten())
        net.add(nn.Dense(128, activation="relu"))
        net.add(nn.Dense(10))
    ctx = utils.try_gpu()
    net.initialize(ctx=ctx)
    trainer = gluon.Trainer(net.collect_params(),'sgd',{'learning_rate':learning_rate})
    epoch = 1
    while utils.lock_start() and epoch < training_iters:
        train_loss = 0
        train_acc = 0
        for data,label in train_data:
            label = label.as_in_context(ctx)
            with autograd.record():
                output = net(data.as_in_context(ctx))
                loss = softmax_cross_entropy(output,label)
            loss.backward()
            trainer.step(batch_size)
            train_acc += utils.accuracy(output,label)
            train_loss += nd.mean(loss).asscalar()
        test_acc = utils.evaluate_accuracy(test_data, net, ctx)
        print("Epoch %d. Loss: %f, Train acc %f, Test acc %f\n" % (epoch, train_loss/len(train_data),train_acc/len(train_data), test_acc))
        utils.chat_inf(wechat_name,epoch,train_loss,train_acc,train_data,test_acc)
        epoch += 1
    utils.lock_end(wechat_name)
            
if __name__ == '__main__':
    utils.chat_supervisor(nn_train)
    itchat.auto_login(hotReload=True)
    itchat.run()
```

## ʵ��Ч��
ɨ���¼΢��A��ͨ��΢��B��΢��Aʵ�ֽ�����

1.ͨ������**param\_name value**�ķ�ʽ��learning_rate, training_iters,
batch_size�������ֱ��дΪlr��ti��bs�����в����޸ģ�

2.����**��ʼ**����ʼ����ѵ����

3.����**ֹͣ**���ݶ�ѵ��������������ʾ��

<center>

![](https://i.imgur.com/y5ZukZb.png)

</center>
