---
title: <<GDPNet: Refining Latent Multi-View Graph for Relation Extraction>>论文阅读笔记
date: 2022-05-13 16:35:14
categories:
- nlp
tags:
- AAAI 2021
- nlp
- paper
- relation extraction
- DialogRE
---
# \<\<GDPNet: Refining Latent Multi-View Graph for Relation Extraction\>\>论文阅读笔记
论文地址：["GDPNet: Refining Latent Multi-View Graph for Relation Extraction"](https://www.aaai.org/AAAI21Papers/AAAI-3290.XueF.pdf)    
复现代码：https://github.com/XueFuzhao/GDPNet  
核心思想：图卷积网络

## 概要
这篇论文，感觉主要是针对DialogRE设计了一个模型。论文中提出了一种图卷积网络，通过这个图卷积网络可以在DialogRE和TRACED上取得比较好的效果。  
作者提出的三个他的创新点：
1. 提出了一种通过计算高斯分布的KL散度初始化图的边的方法
2. 提出一种图池化的方法
3. 提出了GDPNet，并在DialogRE和TRACED上取得SOTA效果

这里说一点，DialogRE是2020年新提出的一个基于对话的关系抽取的数据集。目前做的人比较少，所以这篇论文效果比较好？途中比较的方法都是很基础的模型方法。  
DialogRE这个数据集，个人感觉，虽然说是Dialog，但是目前并没有很好的基于Dialog的建模方法，因为他数据集了并没有什么基于对话的标注信息。DST（Dialog State Tracking）这个研究方向是可以针对对话建模的，因为这个数据集里标注了对话的轮次，和当前论文对话所属的槽位。但是DialogRE并没有针对对话的数据标注，只包含整个对话项的关系类型。所以DialogRE目前的模型，都是把它作为一个长文本关系抽取来做的。个人觉得这是一个可以挖掘的点。  

然后再说下这篇论文，这篇论文，个人感觉，就是建立了一个图神经网络（作者自称的）。  
下面说一说，作者是怎么建图和训练图的（模型设计）

## 模型设计
先放一下论文中的模型结构图  
![Snipaste_2022-05-13_17-30-38.png](https://s2.loli.net/2022/05/13/q4wjrhoCvZ1kbNP.png)
模型整体上分为三个部分，先用BERT做一个Embedding，然后输入作者设计的神经网络里训练，最后放到一个多标签分类器里分类。  
作者设计的神经网络，他自己又分成了三部分，初始化（Gaussian Graph Generator），卷积（Multi-view Graph Convolution）和池化（Dynamic Time Wraping Pooling）（这里的卷积和池化不是CV里的卷积和池化概念）  


先对着图说下整体的结构吧。首先，把文本Embedding以后输入到BERT中，拿到BERT Encoder之后的结果，作为初始的node的Embedding值。然后把node送入Graph Module里去进行图运算。这里的图运算分了三块，首先把node初始化，构建出图的点和边，用的是他说的Gaussian Graph Generator方法。然后，他做了一个8层的迭代，迭代结构大致如下： `Conv -> Pool -> Gonv -> Pool -> Attn -> Conv -> Pool -> Attn -> Conv -> Pool`。迭代完成以后，把node值再做一些线性变化，最后做一个多标签分类，这就是Loss1。作者在论文里还提了一个Loss的计算方法，就是把图第一次的Node和最后一次的Node的值，用SoftDTW（SoftDTW是计算动态时间序列的一个loss）算一个Loss2。最后把这两部分Loss加起来作为整个模型训练的Loss。  

代码：
``` python
layer_list = []
# Gaussian Graph Convolution
gcn_inputs, attn_adj_list = self.GGG(gcn_inputs,adj)
outputs = gcn_inputs
# multi-view Graph Convolution
for i in range(len(self.layers)):
    if i < 4:
        attn_adj_list, outputs, src_mask = self.layers[i](attn_adj_list, outputs, src_mask) # i为偶数层，是做MultiGraphConvLayer，i为奇数层是做SAGPool_Multi
        if i==0:
            src_mask_input = src_mask # 存初始mask
        if i%2 !=0: # 把卷积层结果存下来
            layer_list.append(outputs)
    else:
        attn_tensor = self.attn(outputs, outputs, src_mask) # 根据Attention重新计算邻接矩阵
        attn_adj_list = [attn_adj.squeeze(1) for attn_adj in torch.split(attn_tensor, 1, dim=1)]
        attn_adj_list, outputs, src_mask = self.layers[i](attn_adj_list, outputs, src_mask) # i为偶数层，是做MultiGraphConvLayer，i为奇数层是做SAGPool_Multi
        if i%2 !=0:
            layer_list.append(outputs)
# 把四层的输出结果拼接            
aggregate_out = torch.cat(layer_list, dim=2)
# 做线性变换
dcgcn_output = self.aggregate_W(aggregate_out)
```

下面对每个部分进行介绍
### GGG(Gaussian Graph Generator)
这部分论文的解释是，将每个view的node分布视为一个高斯分布，然后建图的时候，将高斯分布的KL散度（衡量高斯分布的相关性，KL散度越接近0越相关）作为图的边。  
个人觉得，这个地方用KL散度初始化边权值，有点牵强。第一，为什么能建模成一个高斯分布；第二，为什么拿KL散度算相似度；第三，后面重构边权值用的是Attention的方法，那这里为什么不用Attenion做？  
然后，这个部分的目的其实是很清楚的，就是初始化图的点和边（虽然我觉得，不把它当成图，当成一个普通的神经网络，也没什么问题。）

代码：
``` python
def kl_div_gauss(mean_1, mean_2, std_1, std_2):
    kld_element = 0.5*(2*torch.log(std_2) - 2*torch.log(std_1) + (std_1.pow(2) + (mean_1 - mean_2).pow(2))/std_2.pow(2) -1)
    return kld_element

class GGG(nn.Module):
    def __init__(self, config, args):
        super().__init__()
        self.num_heads = args.heads
        self.hid_dim = args.graph_hidden_size
        self.R = self.hid_dim // self.num_heads
        self.max_len_left = args.max_offset
        self.max_len_right = args.max_offset


        self.transform = nn.Linear(self.hid_dim*2, self.hid_dim)
        self.gauss = nn.Linear(self.hid_dim, self.num_heads * 2)

        self.pool_X = nn.MaxPool1d(args.max_offset*2 + 1, stride=1, padding=args.max_offset)
        self.dropout = nn.Dropout(args.input_dropout)


    def forward(self, x, adj, mask = None):
        B, T, C = x.size() # B batch_size; T real_length; C graph_hidden_size (300)
        H = self.num_heads

        x_pooled = self.pool_X(x) # 过max pooling [batch, real_length, pool_size] (300)
        x_new = torch.cat([x_pooled,x],dim=2) # [batch, real_length, graph_hidden_size+pool_size] (600)
        x_new = self.transform(x_new) # FC [batch, real_length, hidden_dim] (300)
        x_new = self.dropout(x_new) # Dropout

        gauss_parameters = self.gauss(x_new) # FC [batch, real_length, heads_num * 2] *2表示方差和标准差
        gauss_mean, gauss_std = gauss_parameters[:,:,:H], F.softplus(gauss_parameters[:,:,H:]) # gauss_mean 方差，gauss_std标准差过了一个 Softplus 激活函数
        # 计算KL散度
        kl_div = kl_div_gauss(gauss_mean.unsqueeze(1).repeat(1,T,1,1),gauss_mean.unsqueeze(2).repeat(1,1,T,1),gauss_std.unsqueeze(1).repeat(1,T,1,1),gauss_std.unsqueeze(2).repeat(1,1,T,1))
        adj_multi = kl_div
        # 返回 heads 个临界矩阵
        attn_adj_list = [attn_adj.squeeze(3) for attn_adj in torch.split(adj_multi, 1, dim=3)]

        return x_new, attn_adj_list

```

### Multi-view Graph Convolution
这个地方，论文里说的是他直接用了别人的模型 [DCGCN](https://aclanthology.org/Q19-1019.pdf), 2019年在TACL上发的。这篇DCCGN解决的问题是Graph2Sequence learning，模型结构参考了DenseNet。基本上就是 DensetNet + GCN 结合。[说明](https://zhuanlan.zhihu.com/p/105999287)

[DenseNet](https://arxiv.org/pdf/1608.06993.pdf) 是清华团队在2017年的CVPR上发的。DenseNet 整体的模型结构和ResNet有一点类似（原论文里也给出了对比），个人感觉DensetNet通过拼接变量的方式将信息在多层之间传递，多重网络之间可以信息共享。[说明](https://blog.csdn.net/qq_52302919/article/details/122788473)

然后，具体的看了他的代码，个人感觉，用的就是DCGCN block的部分。这篇论文里说GCN是直接在Graph结构上操作的神经网络（个人感觉公式好像就只做了个线性变换再加了个激活函数，那和Graph有什么关系呢？然后加了残差，加了DensetNet里的连接。）
这部分的感觉，更像一个更加密集连接的神经网络，和图卷积没多大关系（可能也是我对GCN了解不够深入）。

代码：  
这个网络，输入是node，adj和mask，实际运算只是根据adj修改了node的值。
``` python
class MultiGraphConvLayer(nn.Module):
    """ A GCN module operated on dependency graphs. """

    def __init__(self, args, mem_dim, layers, heads):
        super(MultiGraphConvLayer, self).__init__()
        self.args = args
        self.mem_dim = mem_dim
        self.layers = layers
        self.head_dim = self.mem_dim // self.layers
        self.heads = heads
        self.gcn_drop = nn.Dropout(args.gcn_dropout)
        # dcgcn layer
        self.Linear = nn.Linear(self.mem_dim * self.heads, self.mem_dim)
        self.weight_list = nn.ModuleList()

        for i in range(self.heads):
            for j in range(self.layers):
                self.weight_list.append(nn.Linear(self.mem_dim + self.head_dim * j, self.head_dim))
        self.weight_list = self.weight_list.cuda()
        self.Linear = self.Linear.cuda()

    def forward(self, adj_list, gcn_inputs, src_mask):

        multi_head_list = []
        for i in range(self.heads):
            adj = adj_list[i] # [batch, real_len, real_len]
            denom = adj.sum(2).unsqueeze(2) + 1 # 度矩阵 [batch, real_len, 1]
            outputs = gcn_inputs # [batch, real_len, hidden_dim]
            cache_list = [outputs]
            output_list = []
            for l in range(self.layers):
                index = i * self.layers + l

                Ax = adj.bmm(outputs) # [batch, real_len, hidden_dim]
                AxW = self.weight_list[index](Ax) # [batch, real_len, head_dim]
                AxW = AxW + self.weight_list[index](outputs)  # self loop # [batch, real_len, head_dim]
                AxW = AxW / denom  # 归一化操作
                gAxW = F.relu(AxW) # 非线性变化 [batch, real_len, head_dim]
                cache_list.append(gAxW)
                outputs = torch.cat(cache_list, dim=2) # 在第二个layer中线性层维度变为 hid_dim + head_dim
                output_list.append(self.gcn_drop(gAxW))

            gcn_ouputs = torch.cat(output_list, dim=2)  # [batch, real_len, hidden_dim]
            gcn_ouputs = gcn_ouputs + gcn_inputs  # 和input相加
            multi_head_list.append(gcn_ouputs)
        final_output = torch.cat(multi_head_list, dim=2)  # [batch, real_len, hidden_dim* heads]
        out = self.Linear(final_output)  # [batch, real_len, hidden_dim]
        return adj_list,out, src_mask
```

### Dynamic Time Wraping Pooling
这部分，作者又说用了 [SAGPool](https://arxiv.org/abs/1904.08082)，2019年ICML。SAGPool 里的做法我感觉还是很靠谱的，首先通过图卷积的方法计算self-attention值，将这个值做个线性变换作为节点的权值。池化的方法是选取前TopK个节点，然后将这TopK个节点和self-attention计算的权值相乘，更新node值。整体的模型结构为Conv和Pool交替，Conv更新node值，Pool筛选节点。[说明](https://zhuanlan.zhihu.com/p/94677841)

SAGPool感觉挺有道理的，作者这里，把SAGPool改成了多个View的SAGPool，筛选节点的时候，是选K和最小节点丢掉，每个View都被丢掉的节点才是真正的被丢掉的。但看作者的代码实现，卷积部分用的是上面的卷积，不是原作者的卷积，但我感觉上面的卷积本身效果就是个？

代码：
``` python
class SAGPool_Multi(torch.nn.Module):
    def __init__(self,args, ratio=0.5,non_linearity=torch.tanh, heads = 3):
        super(SAGPool_Multi,self).__init__()
        #self.in_channels = in_channels
        self.ratio = ratio
        self.score_layer = GCN_Pool_for_Single(args, args.graph_hidden_size,1)
        self.linear_transform = nn.Linear(args.graph_hidden_size, args.graph_hidden_size//heads)
        self.non_linearity = non_linearity

    def forward(self, adj_list, x, src_mask):
        '''if batch is None:
            batch = edge_index.new_zeros(x.size(0))'''
        # x = x.unsqueeze(-1) if x.dim() == 1 else x
        adj_list_new = []
        src_mask_list = []
        #src_mask_source = src_mask
        x_list = []
        x_select_list = []
        for adj in adj_list:
            score = self.score_layer(adj, x) # 做一个一层的 Graph Conv
            _, idx = Top_K(score, self.ratio) # 选取最小的k个节点
            x_selected = self.linear_transform(x) # 将x分割为三个view
            x_select_list.append(x_selected)
            # 随机mask掉一部分node
            for i in range(src_mask.size(0)): # batch 
                for j in range(idx.size(1)): # selected node
                    src_mask[i][0][idx[i][j]] = False # 随机mask调30%
            src_mask_list.append(src_mask)
            adj_list_new.append(adj)
        src_mask_out = torch.zeros_like(src_mask_list[0]).cuda()

        x = torch.cat(x_select_list, dim=2) # 三个view的视图合并
        for src_mask_i in src_mask_list: # 只有三个view中都被drop的节点才被真的drop
            src_mask_out = src_mask_out + src_mask_i
        return adj_list_new, x, src_mask_out
```
## 实验结果
作者的实验效果很好，但是我在跑他的代码时候，碰到一些问题。最大的问题就是那个SoftDTW Loss，在他的代码里，这个损失是把第一个卷积图的node和最后一个卷积图的node作为SoftDTW的输入计算Loss的，但是实际训练的时候，这部分损失值，要么就是一个特别大的数 e19 次方这种量级的，要么就是特别小，好像就是没有？然后训练的时候，要么就是Loss直接变成nan，要么就是loss在0.1上下浮动，没有变化。感觉模型根本没被训练。。。不知道是我的问题还是他论文的问题。问了下师兄，感觉他这论文理论也不是很可靠，SoftDTW Loss用的很奇怪（可能是想让他的Graph Module收敛？），就不继续看继续跑了，下一篇吧。

这里说一个后续，我在看这个作者另外一篇论文SimpleRE的时候，也跑了一下他SimpleRE的代码，发现和这个论文跑出来一样，都是准确率不动，然后我就把Bert换成他github提供的bert，发现模型跑出来正常了。两个模型跑出来的结果都是62+，和这个论文里issue复现的结果差不多。然后SimpleRE在test上的结果还比GDPNet好一点。所以。。。懂得都懂。SimpleRE的笔记就不写了，这儿放下链接。  
SimpleRE：[An Embarrassingly Simple Model for Dialogue Relation Extraction](https://arxiv.org/abs/2012.13873)  
代码：https://github.com/XueFuzhao/SimpleRE