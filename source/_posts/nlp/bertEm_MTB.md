---
title:  <<Matching the Blanks,Distributional Similarity for Relation Learning>>论文阅读笔记
date: 2022-04-21 15:32:44
categories:
- nlp
tags:
- relation extraction
- ACL 2019
- nlp
- bertEm
- paper
- mtb
---


# \<\<Matching the Blanks: Distributional Similarity for Relation Learning\>\>论文阅读笔记
论文地址：["Matching the Blanks: Distributional Similarity for Relation Learning"](https://arxiv.org/pdf/1906.03158.pdf)  
复现代码（非官方）：["BERT-Relation-Extraction"](https://github.com/plkmo/BERT-Relation-Extraction)  
核心思想：BertEm + MTB

## 概要
这篇论文主要做了两件事情，第一个是BertEm（提出了一种基于语言模型的更好的关系表示的方式），第二个是MTB（提出一种无监督的关系表示训练方式）。

## BertEm （EM, Entity Markers)
对于给定实体对，预测实体之间关系的任务，论文将Bert的输入中，给实体添加了起止标记（[E1],[\E1]）。然后将这个句子的关系表征认为是这两个实体标记的hidden表示的向量拼接。

![Snipaste_2022-04-23_18-00-04.png](https://s2.loli.net/2022/04/23/75tUzrh2DiGs18e.png)  

### BertEm 做 关系分类预测
方法：在Bert的输出后面添加一个线性层，输出的维度的关系的类别数。 
![Snipaste_2022-04-23_17-59-53.png](https://s2.loli.net/2022/04/23/4kV2XQWzyKpET9x.png)

具体的实现如下：
1. 给定一个实体关系对 `r = (x, s1, s2)`。其中 `x` 是文本， `s1` 和 `s2` 是两个标记实体。
2. 将 `x` 文本中，`s1` 和 `s2` 标记的位置前后，添加 `[E1]` 和 `[\E1]` 标记符。
3. 将 `x` 输入到 `bert` 里 `tokenize & padding`
4. 拿到 `bert` 的 `encoder` 输出 （ `shape` 为 `[batch, seq_len, hidden_size]` ）
5. 从 `encoder` 中取出 `seq` 位置为 `[E1]` 和 `[E2]` 的隐向量，拼接作为关系表示
6. 将拼接后的关系表示添加一个线性层，做关系分类

部分核心代码如下：
``` python
# embedding
embedding_output = self.embeddings(input_ids=input_ids, position_ids=position_ids, token_type_ids=token_type_ids, inputs_embeds=inputs_embeds)

# bert encoder
encoder_outputs = self.encoder(embedding_output,
                                attention_mask=extended_attention_mask,
                                head_mask=head_mask,
                                encoder_hidden_states=encoder_hidden_states,
                                encoder_attention_mask=encoder_extended_attention_mask)
sequence_output = encoder_outputs[0] # [batch, seq_len, hidden_size]
pooled_output = self.pooler(sequence_output)

### two heads: LM and blanks
#  e1_e2_start [batch,2]-> [batch,2,hidden_size]
e1_e2_start = e1_e2_start[:,:,None].expand(-1, -1, sequence_output.size(-1))
# v1v2 [batch,2,hidden_size]
v1v2 = torch.gather(sequence_output, 1, e1_e2_start)
# v1v2 [batch, hidden_size*2]
v1v2 = v1v2.reshape(sequence_output.size(0),-1)

# classification [batch, class_size]
self.classification_layer = nn.Linear(hidden_size, n_classes_)
classification_logits = self.classification_layer(v1v2)
```
### BertEm 做 few-shot 关系分类
few-shot 关系分类目标是通过比较少量的样本数据就可以对新的关系进行分类。论文里使用的小样本学习数据集是FewRel。  
目前的研究中主要应用N way K shot设定，即对于N种关系，每种关系的支持集中包含K个样本。举个例子，5-way 1-shot 指的是，有五个分类，每个类别样本数量只有一个。

![](https://img-blog.csdnimg.cn/img_convert/d2b240b17811ed0e6a06bbca50089a7a.png)

这种时候，模型的输出就不能直接是一个线性层了。论文中使用的方法是，把预测句子从模型输出的关系表示，和候选类别从模型输出的关系表示做点积运算，作为相似度得分，然后用softmax取最大，和真正类别做交叉熵损失作为训练损失。
![](https://pic1.zhimg.com/80/v2-303cc98bd97be21655c6ec5c4ba385d8_1440w.jpg)

具体实现的时候，可以把batch设成class_num+1，每个batch训练每个类别的数据和要预测的数据。


### 论文解释
作者做了一系列的对比实验（改变bert的输入和输出）认为，添加实体的边界位置信息对关系表征有提升作用，这种方法比单纯的使用pool后的cls表征信息效果更好。 

直观的理解一下，我们认为关系和两个实体之间的联系是更直接和紧密的，所以在输入的时候，输出这两个位置的hidden信息，提取出的向量信息，肯定比直接从cls的输出关系质量更高（cls的输出关注的是整个句子，并没有特别关注这两个实体的信息）。

## Matching the Blanks (MTB)
提出一种不需要有标签训练数据的关系表示方式。核心思想是做一个关系的映射学习（和词向量相似），相似的关系在embedding后的空间内距离应该更加相似。更加具体的来说，论文中认为，相同的实体，拥有的关系表示应该更相似，不用实体拥有的关系表示不相似。基于这种思想，制作训练集&训练关系表征的编码器。  
具体实现的时候，相同实体拥有的关系是相似的，这是个强假设，实际上，并不是拥有相同实体就会拥有相似的关系。所以，作者在训练是，随机使用 [BLANK] 标记替换实体（概率0.7），保证模型学到的不是一个强表征。此外，在训练的时候，模型其实同时还做了一个词预测任务（next token perdiction）。这个任务，就是bert的一个下游任务（bert模型在训练的时候就使用），随机mask掉一些词，然后让模型预测这个词。最后模型的损失，是两部分损失之和。

下面详细的说一下，具体训练时候的模型输出和损失的计算方法。因为同时做了两个任务（预测关系相似度的任务和预测mask的词的任务），所以有两个输入和输出。下面分别对这两任务进行介绍。
### 数据集的构造  
样本集的构造，分为两大部分，正样本和负样本。正样本就是拥有相同实体的样本们，它们的label标签为1；负样本，就是拥有的实体和正样本拥有的不一样的样本，它们的label标签为0。  
在具体训练的时候，每个batch，给定一个实体对，筛选出基于这个实体对构造的正负样本数据。然后对这批数据，随机的进行 [BLANK] 和 [MASK] 操作。这样就生成了两个标签数据，一个标签数据标记关系类型（0，1），一个标签标记mask掉的数据的真正的值。对于模型来说，就是一个输入，两个输出。
### 模型的输入输出
模型的输入是经过 [BLANK] 和 [MASK] 处理，并经过 tokenize & padding 后的数据。模型的输出，应该是抽取出来的关系表示和预测的 mask 的词。   
实现代码如下：
``` python
# embedding
embedding_output = self.embeddings(input_ids=input_ids, position_ids=position_ids, token_type_ids=token_type_ids, inputs_embeds=inputs_embeds)

# bert encoder
encoder_outputs = self.encoder(embedding_output,
                                attention_mask=extended_attention_mask,
                                head_mask=head_mask,
                                encoder_hidden_states=encoder_hidden_states,
                                encoder_attention_mask=encoder_extended_attention_mask)
sequence_output = encoder_outputs[0] # [batch, seq_len, hidden_size]
pooled_output = self.pooler(sequence_output)

### two heads: LM and blanks
#  e1_e2_start [batch,2]-> [batch,2,hidden_size]
e1_e2_start = e1_e2_start[:,:,None].expand(-1, -1, sequence_output.size(-1))
# v1v2 [batch,2,hidden_size]
v1v2 = torch.gather(sequence_output, 1, e1_e2_start)
# v1v2 [batch, hidden_size*2]
v1v2 = v1v2.reshape(sequence_output.size(0),-1)

# 这里直接走了个激活函数，作为关系表示
# blanks_logits [batch, hidden_size*2]
blanks_logits = self.activation(v1v2) # 直接走激活函数

# 这里做vocab预测
self.cls = bertOnlyMLMHead(config) # 这里直接调的是bert预测下游任务的方法
# sequence_output [batch, seq_len, hidden_size]
# lm_logits [batch, seq_len, vovab_size]
lm_logits = self.cls(sequence_output) 
```

### Loss 的计算
1. 预测mask的词的任务的loss  

token预测的Loss特别好算，直接一个交叉熵就行了（本质上还是个多分类）。
``` python
# 交叉熵损失判断模型预测的词对不对
lm_loss = self.LM_criterion(lm_logits, lm_labels) # CrossEntropyLoss
```

2. 预测关系相似度的任务的loss

这部分loss的计算比较复杂。先说一下，论文的思想。论文先给出了给定两个关系，计算相似度的公式（这里我觉得应该就是一个sigmoid函数，论文里打掉了个负号）
![](https://www.zhihu.com/equation?tex=p%5Cleft%28l%3D1+%5Cmid+%5Cmathbf%7Br%7D%2C+%5Cmathbf%7Br%7D%5E%7B%5Cprime%7D%5Cright%29%3D%5Cfrac%7B1%7D%7B1%2B%5Cexp+f_%7B%5Ctheta%7D%28%5Cmathbf%7Br%7D%29%5E%7B%5Ctop%7D+f_%7B%5Ctheta%7D%5Cleft%28%5Cmathbf%7Br%7D%5E%7B%5Cprime%7D%5Cright%29%7D)  
根据这个相似度函数，计算损失的公式如下：
![](https://pic3.zhimg.com/80/v2-f4278ae69b155c059b6981fc171247de_1440w.jpg)

看着挺复杂的，我的理解是，给定两个向量，做点积运算，然后把运算结果输出到sigmoid做二分类，最后计算的损失是二分类损失。复杂的地方在于，需要根据正负样本采样算损失（论文里说的是用的噪声对比估计方法）。

代码实现：  
具体计算loss的时候，是对正样本做了个排列组合，正负样本排列组作为计算loss的样本数，最后把这些数据一起计算的loss。（可能因为样本数量太少了？所以需要排列组合？）
``` python
pos_idxs = [i for i, l in enumerate(blank_labels.squeeze().tolist()) if l == 1] # 8 batch里，表示的关系意义相同的index
neg_idxs = [i for i, l in enumerate(blank_labels.squeeze().tolist()) if l == 0] # 16 batch里，表示的关系意义不同的index
# 对预测的关系进行处理
if len(pos_idxs) > 1:
    # positives 起止节点相同的关系，表示应该相似
    pos_logits = [] # 排列组合 C(pos_idxs,2) = pos_idxs * (pos_idxs - 1) / 2, (28, hidden_size * 2)
    for pos1, pos2 in combinations(pos_idxs, 2): # combinations 对 pos_idxs 进行排列组合操作
        pos_logits.append(self.p_(blank_logits[pos1, :], blank_logits[pos2, :])) # 对 start & end 的 向量表示对位相乘并相加
    pos_logits = torch.stack(pos_logits, dim=0) # 28
    pos_labels = [1.0 for _ in range(pos_logits.shape[0])] # 28，全是1， Kronecker delta function，如果i和j相等则是1，否则是0
else:
    pos_logits, pos_labels = torch.FloatTensor([]), []
    if blank_logits.is_cuda:
        pos_logits = pos_logits.cuda()

# negatives 对batch中的正负样本的关系表示，再次设置了排列组合，训练关系表示
neg_logits = [] # shape pos_idxs*neg_idxs 8*16
for pos_idx in pos_idxs:
    for neg_idx in neg_idxs:
        neg_logits.append(self.p_(blank_logits[pos_idx, :], blank_logits[neg_idx, :]))
neg_logits = torch.stack(neg_logits, dim=0) # 128
neg_labels = [0.0 for _ in range(neg_logits.shape[0])] # 128 全是 0

blank_labels_ = torch.FloatTensor(pos_labels + neg_labels) # 156 C(pos_idxs,2) + pos_idxs*neg_idxs

if blank_logits.is_cuda:
    blank_labels_ = blank_labels_.cuda()
# BCE 损失判断模型预测的关系对不对
blank_loss = self.BCE_criterion(torch.cat([pos_logits, neg_logits], dim=0), blank_labels_) # BCELoss
```

3. 整体loss  
最后的loss就是两部分loss相加
``` python
total_loss = lm_loss + blank_loss
```

## 实验结果
论文里的实验结果表明，使用BertEM+MTB可以显著的提升效果，比起只用BertEM的模型来说，可以更快更早的达到更高的准确率（我的理解是因为把详细关系的向量拉近了）。  
![](https://pic3.zhimg.com/80/v2-87c9fc916c2263e68f5d8c1438c5ee22_1440w.jpg)
对于Few-Shot的任务，能够有非常好的结果，甚至比人类标注的结果更好。（人类标注的结果本来就差异比较大，FewRel数据集里也说过，因为样本数量太少的原因）

因为官方没有开源代码，所以只找到GitHub的一个开源库，这个开源库我没有跑，但是根据项目结果来看，BertEM确实可以效果比之前好，MTB的效果，达不到论文中的程度。（作者说因为算力原因，他的训练集比较小。论文里用了维基百科的外部引用信息，训练的数据集也大，效果好也正常？）