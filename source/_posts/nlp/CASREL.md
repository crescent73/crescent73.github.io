---
title: <<A Novel Cascade Binary Tagging Framework for Relational Triple Extraction>>论文阅读笔记
date: 2022-05-01 16:35:36
categories:
- nlp
tags:
- ACL 2020
- relation extraction
- nlp
---
# \<\<A Novel Cascade Binary Tagging Framework for Relational Triple Extraction\>\>论文阅读笔记
论文地址：[A Novel Cascade Binary Tagging Framework for Relational Triple Extraction](https://arxiv.org/pdf/1909.03227.pdf)  
官方代码：https://github.com/weizhepei/CasRel]  
Pytorch 版复现代码：https://github.com/Onion12138/CasRelPyTorch

## 概要
这篇论文解决的问题是关系重叠（Overlappig triple problem）。  
作者首先提出，传统的关系三元组抽取，是先抽取出首尾实体，在预测实体关系，这种方式，对于关系分类来说，输入只有首尾实体的信息，对重叠实体对的三元组关系抽取效果不好。  
于是作者提出他的解决思路：改变抽取方式，先抽取首尾实体，然后根据关系类型，抽取尾实体。

关系重叠问题，作者举了几个例子：
1. Normal：没有重叠的
2. EPO(Entity Pair Overlap)：关系两端的实体相同
3. SEO(Single Entity Overlap)：关系两端有一个实体相同


![](https://pic3.zhimg.com/80/v2-34aa1bae791170bc476497ba023920f6_1440w.jpg)

## 模型
抽取方法如图所示，首先抽取实体（Subject Targger），采用的方式就是序列标注。作者构造了两个二分类器，一个标注首实体的起位置，一个标注首实体的尾位置。然后把输出结果和原始表示一起输入到关系和尾实体标注器中（Relation-Specific Object Taggers）。这个标注器，本质上是关系类型个尾实体标注器。  
模型的损失是首实体预测的损失和每种关系的尾实体预测的损失。

![](https://pic1.zhimg.com/80/v2-230cf30cc6c5467896c27eff61e983a4_1440w.jpg)

## 实现
由于模型结构实在是太简单了，我这儿就直接贴以下model部分的代码（Pytorch）  
本质上就是 首实体（2个二分类器）+ 关系数*尾实体预测（2个二分类器）
``` python
class CasRel(nn.Module):
    def __init__(self, config):
        super(CasRel, self).__init__()
        self.config = config
        self.bert = BertModel.from_pretrained(self.config.bert_name)
        self.sub_heads_linear = nn.Linear(self.config.bert_dim, 1)
        self.sub_tails_linear = nn.Linear(self.config.bert_dim, 1)
        self.obj_heads_linear = nn.Linear(self.config.bert_dim, self.config.num_relations)
        self.obj_tails_linear = nn.Linear(self.config.bert_dim, self.config.num_relations)

    def get_encoded_text(self, token_ids, mask):
        encoded_text = self.bert(token_ids, attention_mask=mask)[0]
        return encoded_text

    def get_subs(self, encoded_text):
        pred_sub_heads = torch.sigmoid(self.sub_heads_linear(encoded_text))
        pred_sub_tails = torch.sigmoid(self.sub_tails_linear(encoded_text))
        return pred_sub_heads, pred_sub_tails

    def get_objs_for_specific_sub(self, sub_head_mapping, sub_tail_mapping, encoded_text):
        # sub_head_mapping [batch, 1, seq] * encoded_text [batch, seq, dim]
        sub_head = torch.matmul(sub_head_mapping, encoded_text) # [batch_size, 1, hidden_size]
        sub_tail = torch.matmul(sub_tail_mapping, encoded_text)
        sub = (sub_head + sub_tail) / 2
        encoded_text = encoded_text + sub # encoded_text = bert_encoding_text + origin_sub_text [batch_size, seq_len, hidden_size]
        pred_obj_heads = torch.sigmoid(self.obj_heads_linear(encoded_text)) # [batch_size, seq_len, relation_num]
        pred_obj_tails = torch.sigmoid(self.obj_tails_linear(encoded_text)) # [batch_size, seq_len, relation_num]
        return pred_obj_heads, pred_obj_tails

    def forward(self, token_ids, mask, sub_head, sub_tail):
        encoded_text = self.get_encoded_text(token_ids, mask) # encoding [batch_size, seq_len, hidden_size]
        pred_sub_heads, pred_sub_tails = self.get_subs(encoded_text) # perdict subject head & tail [batch_size, seq_len, 1]
        sub_head_mapping = sub_head.unsqueeze(1) # [batch_size, 1, seq_len]
        sub_tail_mapping = sub_tail.unsqueeze(1)
        pred_obj_heads, pre_obj_tails = self.get_objs_for_specific_sub(sub_head_mapping, sub_tail_mapping, encoded_text)

        return {
            "sub_heads": pred_sub_heads,
            "sub_tails": pred_sub_tails,
            "obj_heads": pred_obj_heads,
            "obj_tails": pre_obj_tails,
        }
```

## 结果
作者从模型结构上改写了这个问题，经过修改的后的模型结构，更加适用于关系重叠的问题，比起原来的模型结构有天然的优势。作者在论文里放出了他的结果，取得了SOTA的结果。  
![](https://pic3.zhimg.com/80/v2-5f9e0c69a81e2da0a0fb75ca80196266_1440w.jpg)
针对重叠类型的句子的实验
![](https://pic3.zhimg.com/80/v2-04390d78089c7f651ee4ed6fb388205e_1440w.jpg)