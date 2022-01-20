---
title: neo4j
date: 2021-9-20 10:26:45
categories:
- database
tags:
- database
- neo4j
---
# neo4j笔记

## 1. 介绍
### 1.1 什么是图数据库
   &emsp;&emsp;图数据库是基于图论实现的一种NoSQL数据库，其数据存储结构和数据查询方式都是以图论为基础的，图数据库主要用于存储更多的连接数据。 

对比:
  + 在关系型数据库中，Person和department之间用外键表示关系;
  + 在图数据库中，节点和关系取代表，外键和join.
  + 在图数据库中，无论何时运行类似JOIN的操作，数据库都会使用此列表并直接访问连接的节点，而无需进行昂贵的搜索和匹配计算。

  非SQL数据库类别：
  + key-value数据库：redis
  + 列存储数据库：hbase
  + 文档型数据库：mongodb
  + 图数据库：neo4j
![1632104115545.png](https://s2.loli.net/2022/01/19/a5muXwgPYK38DGl.png)
![1632104052329.png](https://s2.loli.net/2022/01/19/CAyPXfjKVvZ6pWo.png)

### 1.2 什么是neo4j
 Neo4j是一个开源的NoSQL图形数据库，2003年开始开发，使用scala和java语言，2007年开始发布。
 + 是世界上最先进的图数据库之一，提供原生的图数据存储，检索和处理;1
 + 采用属性图模型(Property graph model)，极大的完善和丰富图数据模型;
 + 专属查询语言Cypher，直观，高效;
 + 官网: https://neo4j.com/

### 1.3 neo4j数据模型
#### 图论基础
图是一组节点和连接这些节点的关系，图形以属性的形式将数据存储在节点和关系中，属性是用于表示数据的键值对。在图论中，我们可以表示一个带有圆的节点，节点之间的关系用一个箭头标记表示。

#### 属性图模型
Neo4j图数据库遵循属性图模型来存储和管理其数据。属性图模型规则
I
+ 表示节点，关系和属性中的数据。节点和关系都包含属性
关系连接节点
+ 属性是键值对
+ 节点用圆圈表示，关系用方向键表示。
+ 关系具有方向:单向和双向。
+ 每个关系包含“开始节点"或“从节点"和“到节点"或“结束节点”

### 1.4 neo4j的构建元素
Neo4j图数据库主要有以下构建元素:
+ 节点
+ 属性
+ 关系
+ 标签
+ 数据浏览器
![1632104606163.png](https://s2.loli.net/2022/01/19/ju6icYBLh7gZnFd.png)

## 2. 环境搭建
### 2.1 neo4j community server安装
jdk8可以下载Neo4j community Edition 3.5.28

文档: https://neo4j.com/docs/operations-manual/3.5/

解压到新目录（注意:目录名称不要有中文) ,比如: D:\neo4j\


```
#将Neo4j作为控制台应用程序运行
<NE04_HOME>\bin\neo4j console
#将Neo4j作为服务使用进行安装
<NE043_HOME>\bin\neo4j instal1-service
```
console:直接启动neo4j服务器

install-service | uninstall-service | update-service :安装/卸载/更新neo4j服务

start/stop/restart/status:启动/停止/重启/状态

-V 输出更多信息

进入到bin目录，执行

` neo4j console `

在浏览器中访问http://localhost:7474

使用用户名neo4j和默认密码neo4j进行连接，然后会提示更改密码。

Neo4j Browser是开发人员用来探索Neo4j数据库、执行Cypher查询并以表格或图形形式查看结果的工具。

### 2.2 docker安装
+ 7474 for HTTP.
+ 7473 for HTTPS.
+ 7687 for Bolt.

拉取镜像

`docker pu17 neo4j : 3.5.22-community`

运行镜像
```
docker run -d -p 7474:7474 -p 7687:7687 --name neo4j\ -e "NEO4J_AUTH=neo4j/123456"\
-v /usr/1oca1/soft/neo4j/data:/data \
-v /usr/loca1/soft/neo4j/logs:/logs \
-v /usr/loca1/soft/neo4j/conf:/var/lib/neo4j/conf \
-v /usr/loca1/soft/neo4j/import:/var/lib/neo4j/import \ neo4j:3.5.22-community
```

## 3. CQL使用
### 3.1 CQL 简介
Neo4j的Cypher语言是为处理图形数据而构建的，CQL代表Cypher查询语言。像Oracle数据库具有查询语言SQL，Neo4j具有CQL作为查询语言。
+ 它是Neo4j图形数据库的查询语言。
+ 它是一种声明性模式匹配语言
+ 它遵循SQL语法。
+ 它的语法是非常简单且人性化、可读的格式。
![1632108290511.png](https://s2.loli.net/2022/01/19/1EJekp2TH7z8IGN.png)

### 3.2 基本指令
查看文档：https://neo4j.com/docs/cypher-manual/3.5/clauses/match/

创建关系：
```
match (n:star {name:'张国荣'}), (m:startRelation), (s:star)
where m.subject = n.name and s.name = m.object
create (n) -[:关系{relation:m.relation}]-> (s)
```

### 3.3 常用函数
查看文档：https://neo4j.com/docs/cypher-manual/3.5/functions/

### 3.4 neo4j-admin使用
数据库备份

对Neo4j数据进行备份、还原、迁移的操作时，要关闭neo4j
```
cd %NE043_HOME%/bin
#关闭neo4j (要先install-service，才需要stop)
neo4j stop
#备份
neo4j-admin dump --database=graph.db --to=/neo4j/backup/graph_backup.dump
```
 
数据库恢复

还原、迁移之前，要关闭neo4j服务。
```
#数据导入
neo4j-admin load --from=/neo4j/backup/graph_backup.dump --database=graph.db --force
#重启服务
neo4j start
```

## 3.5 例：利用CQL构建明星关系图谱
csv存储关系数据库格式的数据，明星表和关系表，在neo4j中分别导入这两个表，然后根据关系表，创造明星之间的关系。
```
#导入明星数据
load csv from 'file:////l/明星1.csv' as line
create (:star {num: line[0],name : 1ine[1]})
1oad csv from 'fi1e:///[[明星关系数据1.csv' as line
create (:starRelation {from:line[0] , subject:1ine[1],to:1ine[2] ,object:line[3], relation:1ine[4]})

#查询明星关系
match (n:star),(m:starRelation) ,(s:star) 
where n.name='刘烨' and m.subject='刘烨' and s.name=m.object
return n.name ,m.relation,s.name

#创建关系构建所有明星关系图谱
match (n:star), (m:starRelation),(s:star) 
where n.name=m.subject and s.name=m.object
create (n)-[r:关系{relation :m.re1ation->(s)

#创建关系构建指定明星关系图谱
match (n:star), (m:starRelation),(s:star) 
where n.name='刘烨' and m.subject='刘烨' and s.name=m.object
create (n)-[r:关系{relation :m.re1ation->(s)
return n.name ,m.relation ,s.name

#查看明星关系
MATCH p=(n :star{name : '刘烨'})-[r:`关系`]->()
RETURN p
```