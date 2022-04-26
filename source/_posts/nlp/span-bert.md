---
title: <<SpanBERT, Improving Pre-training by Representing and Predicting Spans>>论文阅读笔记
date: 2022-04-26 10:48:14
categories:
- nlp
tags:
- span bert
- pretrained language model
---
# \<\<SpanBERT: Improving Pre-training by Representing and Predicting Spans\>\>论文阅读笔记
论文地址：[SpanBERT: Improving Pre-training by Representing and Predicting Spans](https://arxiv.org/abs/1907.10529)  
官方代码：https://github.com/facebookresearch/SpanBERT

## 概要
论文主要做了以下件事  
1. 将原来BERT基于字符级别的Mask换成基于Span（词）级别的
2. 去掉了原BERT的NSP（Next Sentence Perdiction）任务
3. 添加一个SBO（Span Boundary Objective）任务

## 新的MLM（Mask Language Model) 任务
先说一下inspiration。原始的BERT基于每个字符（subword）进行mask，但这样，有时候会拆散单词的语义信息。于是Google提出了基于整个单词的Mask模型，Google BERT WWM（Whole Word Masking）。那么，既然可以基于每个单词Mask，那么可不可以基于整个实体Mask呢（实体由多个单词组成），于是百度提出了ERNIE。这样看，效果好像更好了，但是MASS模型表明，不一定需要引入词的边界信息，随机Mask的效果可能也很好。  
于是，SpanBert提出了根据几何分布的Mask方法。

Mask的概率分布计算方法如下（为了保证随机效果）：  
根据几何分布，先随机选择一段Span的长度，再根据均匀随机分布，随机选择这一段的起始位置，最后按照长度遮盖。  
论文选取的几何分布概率为0.2，最大长度的Span为10（更长的Span舍弃）。几何分布的公式为  
![几何分布](https://bkimg.cdn.bcebos.com/formula/7c20672eab7d7f90571734166b7a3f42.svg)  
所以，论文里实际计算的时候，计算的是p取0.2，k取1-10的概率，由于k大于10的情况的概率被舍弃，所以最终会做一个归一化。 
![Snipaste_2022-04-26_11-31-32.png](https://s2.loli.net/2022/04/26/l3E6GPO4YKkLred.png)  
这个公式的计算结果和论文里给的图的结果也是吻合的
![](https://pic3.zhimg.com/80/v2-4b4d741e7adc751874299a7e2ce3e252_1440w.jpg)  

### 代码实现
概率分布代码实现：
``` python
# 计算几何分布概率
self.len_distrib = [self.p * (1-self.p)**(i - self.lower) for i in range(self.lower, self.upper + 1)] if self.p >= 0 else None
self.len_distrib = [x / (sum(self.len_distrib)) for x in self.len_distrib]

# 对不同长度的span，指定概率
span_len = np.random.choice(self.lens, p=self.len_distrib) # 对长度进行不同权重的抽样，有放回
```

在具体Mask的时候，和BERT一样，总共选取15%的字符Mask掉，其中80%的字符被替换成Mask标记，10%的字符被替换成随机的token，10%的字符保持不变。
这部分的代码实现如下：
``` python
rand = np.random.random()
for i in range(start, end + 1):
    assert i in mask
    target[i] = sentence[i]
    if replacement == 'word_piece':
        rand = np.random.random()
    if rand < 0.8: # 80%的概率被替换成[MASK]
        sentence[i] = mask_id
    elif rand < 0.9: # 10%的概率被替换成随机token
        # sample random token according to input distribution
        sentence[i] = np.random.choice(tokens)
```

### 实验结果
作者做了一系列的对比实验，实验里对比了其他模型的Mask方法，实验表明，处理在Conference Resolution任务上，Random Span的Mask方法普遍更优。
![](https://pic4.zhimg.com/80/v2-0b78ddcb47528cef8a81dba7c1ab1f17_1440w.jpg)

## SBO（Span Boundary Objective）任务
SBO任务的目的是希望被遮盖Span边界的词向量，也能学习到Span的内容。让模型在一些需要Span的下游任务取得更好的成果。  
![](https://pic4.zhimg.com/80/v2-38328bde767c733e90bd76af091a5d6b_1440w.jpg)
具体的实现思路是，对于mask掉的span，取其前后位置的隐藏层信息拼接，并添加上预测的token在span中的位置信息，输入两个线性层作为最终结果。
![](https://pic4.zhimg.com/80/v2-8a5bec12575460bfe928a925c20053ab_1440w.jpg)  
模型的loss是MLM任务的损失和SBO损失之和。

### 代码实现
``` python
class BertPairTargetPredictionHead(nn.Module):
    def __init__(self, config, bert_model_embedding_weights, max_targets=20, position_embedding_size=200):
        super(BertPairTargetPredictionHead, self).__init__()
        self.position_embeddings = nn.Embedding(max_targets, position_embedding_size)
        self.mlp_layer_norm = MLPWithLayerNorm(config, config.hidden_size * 2 + position_embedding_size)
        # The output weights are the same as the input embeddings, but there is
        # an output-only bias for each token.
        self.decoder = nn.Linear(bert_model_embedding_weights.size(1),
                                 bert_model_embedding_weights.size(0),
                                 bias=False)
        self.decoder.weight = bert_model_embedding_weights
        self.bias = nn.Parameter(torch.zeros(bert_model_embedding_weights.size(0)))
        self.max_targets = max_targets

    def forward(self, hidden_states, pairs):
        """
        取出hidden_states 对应的左右边界的向量表示，
        对于一个batch来说，有num_pair个左和右的向量表示，之后复制max_targets个，和位置拼接，送入解码器。
        """
        bs, num_pairs, _ = pairs.size()
        bs, seq_len, dim = hidden_states.size()
        # pair indices: (bs, num_pairs)
        left, right = pairs[:,:, 0], pairs[:, :, 1]
        # (bs, num_pairs, dim)
        left_hidden = torch.gather(hidden_states, 1, left.unsqueeze(2).repeat(1, 1, dim))
        # pair states: bs * num_pairs, max_targets, dim
        left_hidden = left_hidden.contiguous().view(bs * num_pairs, dim).unsqueeze(1).repeat(1, self.max_targets, 1)
        right_hidden = torch.gather(hidden_states, 1, right.unsqueeze(2).repeat(1, 1, dim))
        # bs * num_pairs, max_targets, dim
        right_hidden = right_hidden.contiguous().view(bs * num_pairs, dim).unsqueeze(1).repeat(1, self.max_targets, 1)

        # (max_targets, dim)
        position_embeddings = self.position_embeddings.weight
        hidden_states = self.mlp_layer_norm(torch.cat((left_hidden, right_hidden, position_embeddings.unsqueeze(0).repeat(bs * num_pairs, 1, 1)), -1))
        # target scores : bs * num_pairs, max_targets, vocab_size
        # bert的decoder就是FC+GeLU+LN
        target_scores = self.decoder(hidden_states) + self.bias
        return target_scores
```

### 实验结果
加上SBO后效果普遍提高
![](https://pic4.zhimg.com/80/v2-2229a8e0f916045940f4cf0cd3e2dfaf_1440w.png)

## NSP(Next Sentence Perdiction)任务没有用的解释
1. 比起拼接两句话，一句长句，模型可以获得更长的上下文
2. NSP负例情况下，基于另一个文档来预测词，会给MLP任务带来很大噪音

# 参考
1. https://zhuanlan.zhihu.com/p/75893972
2. https://zhuanlan.zhihu.com/p/387547270?msclkid=9e9e0202c50411eca69b091099e2a31c
3. https://zhuanlan.zhihu.com/p/103203220