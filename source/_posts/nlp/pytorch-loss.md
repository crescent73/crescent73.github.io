---
title: pytorch-loss
date: 2022-04-22 11:39:03
categories:
- pytroch
tags:
- pytorch
- loss
- deep learning
---

# pytorch 中的 loss
最近在写代码的时候，遇到了几种损失函数，做下笔记区分一下。  
下面对以下几种loss做区分：CrossEntropyLoss、NLLLoss、BCEloss、BCEWithLogitsLoss。

首先，概括性的说一下，这几个函数都是计算交叉熵的函数。CrossEntropyLoss 和 NLLLoss是计算多分类的交叉熵（当然也可以计算二分类），BCEloss 和BCEWithLogitsLoss 是计算二分类的交叉熵。

从pytroch接口的角度来说，CrossEntropyLoss() 内部将 input 先做了softmax 后在与 label 做 交叉熵。 而 NLLLoss() 内部不先做softmax就与label做交叉熵。和BCEWithLogitsLoss内部先将input做了sigmoid在做交叉熵，BCEloss直接将input做交叉熵。简单理解以下就是:
```
CrossEntropyLoss = softmax + NLLLoss
BCEWithLogitsLoss = sigmoid + BCEloss
```
这样说还挺简单的，但是感觉，理解的还不是很深入啊，下面开始一波深挖。
# sigmoid 函数 和 softmax 函数
## sigmoid
sigmoid函数公式：  
![](https://www.zhihu.com/equation?tex=Sigmoid%28x%29%3D%5Cfrac%7B1%7D%7B1%2Be%5E%7B-x%7D%7D)  
图像  
![](https://pic1.zhimg.com/80/v2-17889947ad7bc75ba672e41363b54788_1440w.jpg)  
导函数：
![](https://www.zhihu.com/equation?tex=Sigmoid%5E%7B%27%7D%28x%29%3DSigmoid%28x%29%5Ccdot+%281-Sigmoid%28x%29%29)  
输出的值在[0,1]之间。比较适合多标签分类，多个正确答案，非独占输出（设定阈值）。
## softmax
softmax被称为归一化指数函数，函数公式：
![](https://www.zhihu.com/equation?tex=Softmax%28x%29%3D%5Cfrac%7Be%5E%7Bx_%7Bi%7D%7D%7D%7B%5Csum_%7Bj%3D1%7D%5E%7Bn%7D%7Be%5E%7Bx_%7Bj%7D%7D%7D%7D)  
softman函数是二分类函数sigmoid在多分类上的推广，目的是将多分类结果以概率的形式展现出来。比较适合多类别分类问题，只有一个正确答案（取概率最大的）。  
softmax函数通过指数函数，拉大了向量元素之间的差异，然后再进行归一化操作，所以对于分类问题来说，它使各个类别的概率差异更加显著，最大值产生的概率更加接近于1，输出的分布形式更加合理。
## 区别与联系
从概率学的角度来说，sigmoid的函数公式进行变形可以得到二项分布的公式，softmax的函数公式变形，可以得到伯努利分布（这部分内容PRML第二章里有讲，都属于指数族，相关博客 [链接](https://www.zhihu.com/equation?tex=Softmax%28x%29%3D%5Cfrac%7Be%5E%7Bx_%7Bi%7D%7D%7D%7B%5Csum_%7Bj%3D1%7D%5E%7Bn%7D%7Be%5E%7Bx_%7Bj%7D%7D%7D%7D)）。  

从数学的角度来说，当分类的个数为2的时候，这两个公式是效果是一样的（伯努利分布退化为二项分布）。下面直接放一个博主的解释  
![图源https://blog.csdn.net/yyhhlancelot/article/details/104260794](https://s2.loli.net/2022/04/23/QmSKWJhkrewoXUC.png)

## 总结
1. sigmoid函数用来解决**多标签**问题，softmax函数用来解决**单标签**问题。  
2. 对于分类来说，如果softmax函数能用的话，sigmoid函数也能用。

# sigmoid 交叉熵损失 和 softmax 交叉熵损失
## 交叉熵损失
熵的概念：用来衡量信息的不确定性，信息的不确定性越大，熵就越大。
最原始交叉熵损失的公式：
![](https://img-blog.csdnimg.cn/20190206214003719.png)
对于这个公式来说，交叉熵就是衡量两个不同的概率分布p和q的相似情况，p和q越相似，计算出来的交叉熵越小。对于模型来说，p一般是标签值，q是模型的预测值，它们的交叉熵越小，代表模型预测的值越准确，所以在模型训练的时候，一般都用交叉熵作为损失函数。
## sigmoid 交叉熵损失
sigmoid的交叉熵损失函数公式：
![](https://img-blog.csdnimg.cn/20190206230609565.png)

一般来说，对于二分类的情况，模型在输出层会经过一个nn.Linear()全连接层压缩至1维，然后接Sigmoid，然后使用BCELoss（也可以用BCEWithLogitsLoss），做一个二分类损失计算。
对于多分类的情况，模型输出的是多个标签维度，还是使用Sigmoid+BCELoss或BCEWithLogitsLoss，模型会自动使用上面的公式计算。

## softmax 交叉熵损失
sigmoid的交叉熵损失函数公式：
![](https://img-blog.csdnimg.cn/20190206223310433.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2MzY4Mzg4,size_16,color_FFFFFF,t_70)
这里公式这么简单的原因在于，对于单标签分类来说，只有一个目标标签的y值是1，其他值是0，所以在计算交叉熵的时候都做0处理，相当于只需要计算目标类别的损失。
对于二分类的情况，一般模型输出层会会经过一个nn.Linear()全连接层压缩至2维，然后接softmax，然后使用NLLLoss（也可以用CrossEntropyLoss），做一个二分类损失计算。多个类别同理。

# 参考
1. https://blog.csdn.net/yyhhlancelot/article/details/104260794
2. https://zhuanlan.zhihu.com/p/356976844
3. https://blog.csdn.net/qq_36368388/article/details/86769443

