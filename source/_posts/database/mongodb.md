---
title: mongodb
date: 2021-10-28 9:23:18
categories:
- database
tags:
- database
- mongodb
---

# MongoDB
目标：
+ 理解MongoDB的业务场景、熟悉MongoDB的简介、特点和体系结构、数据类型等。
+ 能够在Windows和Linux下安装和启动MongoDB、图形化管理界面Compass的安装使用
+ 掌握MongoDB基本常用命令实现数据的CRUD
+ 掌握MongoDB的索引类型、索引管理、执行计划。
+ 使用Spring Data MongoDB完成文章评论业务的开发

## 1. MongoDB相关概念
### 1.1 业务场景
传统的关系型数据库（如MySQL)，在数据操作的"三高"需求以及应对Web2.0的网站需求面前，显得力不从心。
解释:"三高"需求:
+ High performance -对数据库高并发读写的需求。（双十一）
+ Huge Storage -对海量数据的高效率存储和访问的需求。（朋友圈）
+ High Scalability && High Availability-对数据库的高可扩展性和高可用性的需求。（不用固定列）

**而MongoDB可以对应三高需求**

具体的应用场景有：
1. 社交场景，使用MongoDB存储存储用户信息，以及用户发表的朋友圈信息，通过地理位置索引实现附近的人、地点等功能。
2. 游戏场景，使用MongoDB存储游戏用户信息，用户的装备、积分等直接以内嵌文档的形式存储，方便查询、高效率存储和访问。
3. 物流场景，使用MongoDB存储订单信息，订单状态在运送过程中会不断更新，以MongoDB内嵌数组的形式来存储，一次查询就能将订单所有的变更读取出来。
4. 物联网场景，使用MongoDB存储所有接入的智能设备信息，以及设备汇报的日志信息，并对这些信息进行多维度的分析。
5. 视频直播，使用MongoDB存储用户信息、点赞互动信息等。

这些应用场景中，数据操作方面的共同特点是:
1. 数据量大
2. 写入操作频繁(读写都很频繁)
3. 价值较低的数据，对事务性要求不高（银行转账类的就不适合）
对于这样的数据，我们更适合使用MongoDB来实现数据的存储。

**什么时候选择MongoDB?**

在架构选型上，除了上述的三个特点外，可以考虑以下的一些问题:
+ 应用不需要事务及复杂join支持
+ 新应用，需求会变，数据模型无法确定，想快速迭代开发
+ 应用需要2000-3000以上的读写QPS(更高也可以)
+ 应用需要TB甚至PB级别数据存储应用发展迅速，需要能快速水平扩展
+ 应用要求存储的数据不丢失
+ 应用需要99.999%高可用
+ 应用需要大量的地理位置查询、文本查询

如果上述有1个符合，可以考虑MongoDB，2个及以上的符合，选择MongoDB绝不会后悔。

思考:如果用MySQL呢?

答:相对MySQL，可以以更低的成本解决问题（包括学习、开发、运维等成本)

### 1.2 MongoDB简介
MongoDB是一个开源、高性能、无模式的文档型数据库，当初的设计就是用于简化开发和方便扩展，是NoSQL数据库产品中的一种。是最像关系型数据库(MySQL)的非关系型数据库。

它支持的数据结构非常松散，是一种类似于JSON的格式叫BSON,所以它既可以存储比较复杂的数据类型，又相当的灵活。

MongoDB中的记录是一个文档，它是一个由字段和值对(field:value)组成的数据结构。MongoDB文档类似于JSON对象，即一个文档认为就是一个对象。字段的数据类型是字符型，它的值除了使用基本的一些类型外，还可以包括其他文档、普通数组和文档数组。

### 1.3 体系结构
MySQL和MongoDB对比

![1635384807966.png](https://s2.loli.net/2022/01/19/tcJoOd1S7gMWfPm.png)
![1635384984939.png](https://s2.loli.net/2022/01/19/OfAt7b5M9CWrRTg.png)

### 1.4 数据类型
MongoDB的最小存储单位就是文档(document)对象。文档(document)对象对应于关系型数据库的行。数据在MongoDB中以BSON (Binary-JSON)文档的格式存储在磁盘上。

BSON (Binary Serialized Document Format)是一种类json的一种二进制形式的存储格式，简称Binary JSON.BSON和JSON一样，支持内嵌的文档对象和数组对象，但是BSON有JSON没有的一些数据类型，如Date和BinData类型。

BSON采用了类似于C语言结构体的名称、对表示方法，支持内嵌的文档对象和数组对象，具有轻量性、可遍历性、高效性的三个特点，可以有效描述非结构化数据和结构化数据。这种格式的优点是灵活性高，但它的缺点是空间利用率不是很理想。

Bson中，除了基本的JSON类型: string,integer,boolean,double,null,array和object，mongo还使用了特殊的数据类型。这些类型包括date,object id,binary data,regular expression和code。每一个驱动都以特定语言的方式实现了这些类型，查看你的驱动的文档来获取详细信息。

![1635385636427.png](https://s2.loli.net/2022/01/19/n7yoIeM5KkSh4YL.png)

### 1.5 MongoDB的特点
+ 高性能：提供高性能的数据持久性。特别是对嵌入式数据模型的支持减少了数据库系统上的IO活动。
+ 高可用性：MongoDB的复制工具成为副本集（replica set），它可提供自动故障转移和数据冗余。
+ 高扩展性：MongoDB提供了水平可扩展性作为其核心功能的一部分。
+ 丰富的查询支持：MongoDB支持丰富的查询语言，支持读和写操作（CRUD）。
+ 其他特点：如无模式（动态模式）、灵活的文档模型等。

## 2. MongDB安装
### 2.1 Windows
1. 第一步：下载安装包，下载zip格式的解压缩就能使用。
    https://www.mongodb.com/try/download/community

    版本的选择: MongoDB的版本命名规范如:x.y.Z;
    + y为奇数时表示当前版本为开发版，如:1.5.2、4.1.13;
    + y为偶数时表示当前版本为稳定版，如:1.6.3、4.0.10;
    + z是修正版本号，数字越大越子。
2. 第二步：解压安装启动
在解压目录中手动建立一个目录用于存储数据文件，如data/db。

方式1:命令行参数方式启动服务

在bin目录中打开命令行提示符，输入如下命令:

` mongod --dbpath=..\data\db `

我们在启动信息中可以看到,mongoDB的默认端口是27017，如果我们想改变默认的启动端口，可以通过--port来指定端口。

为了方便我们每次启动，可以将安装目录的bin目录设置到环境变量的path中， bin目录下是一些常用命令，比如mongod启动服务用的，mongo客户端连接服务用的。

方式2:配置文件方式启动服务

在解压目录中新建conf文件夹，该文件夹中新建配置文件mongod.conf，内如参考如下:
```
systemLog:
  #MongoDB发送所有日志输出的目标指定为文件
  destination: file
  path: "/usr/local/mongodb/log/mongod.log"
  logAppend: true
storage:
  #mongod实例存储其数据的目录
  dbPath: "/usr/local/mongodb/data"
  journal:
    #启用或禁用持久性日志以确保数据文件保持有效和可恢复。 
    enabled: true
processManagement: 
   #启用在后台运行mongos或mongod进程的守护进程模式。 
   fork: true
net:
   #服务实例绑定的IP，默认是localhost 
   bindIp: 0.0.0.0
   port: 27017
```

启动方式:
` mongod -f ../conf/mongod.conf `

**windows 安装服务**
安装服务：
```
mongod -f mongodbPath\conf\mongod.conf --logappend --directoryperdb --serviceName MongoDB --install
```
启动服务：
```
net start MongoDB
```
删除服务
```
sc delete MongoDB
```
### 2.2 Shell连接（mongo命令）
在shell中输入mongo命令

### 2.3 Compass 图形化界面客户端
https://www.mongodb.com/try/download/compass

### 2.4 Ubuntu安装
https://docs.mongodb.com/v4.4/tutorial/install-mongodb-on-ubuntu/

默认config地址：/etc/mongod.conf
```
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0

```

启动服务
```
sudo systemctl start mongod
sudo systemctl status mongod
sudo systemctl daemon-reload
sudo systemctl restart mongod
sudo service mongod stop
```
查看进程：`ps -ef | grep mongod`

## 3. 基本常用命令
### 3.1 数据库操作
#### 3.1.1 选择和创建数据库
+ 选择和创建数据库：`use databaseName`，如果数据库不存在会自动创建
+ 查看所有数据库：` show dbs`或`show databases`
> 注意：在MongoDB中，集合只有在内容插入之后才回创建。也就是说，创建集合后要再插入一个文档，集合才会真正创建。
+ 查看当前数据库：`db`
> MongoDB中默认的数据库为test，如果你没有选择数据库，集合将存放在test数据库中。

数据库名的命名规则：可以是满足以下条件的任意UTF-8字符串。
+ 不能是空字符串("")。
+ 不得含有' '(空格)、.、$、/、\和\0(空字符)。
+ 应全部小写。
+ 最多64字节。

有一些数据库名是保留的，可以直接访问这些有特殊作用的数据库。
+ admin: 从权限的角度来看，这是"root"数据库。要是将一个用户添加到这个数据库，这个用户自动继承所有数据库的权限。一些特定的服务器端命令也只能从这个数据库运行，比如列出所有的数据库或者关闭服务器。
+ local:这个数据永远不会被复制，可以用来存储限于本地单台服务器的任意集合
+ config:当Mongo用于分片设置时，config数据库在内部使用，用于保存分片的相关信息。

#### 3.1.2 删除数据库
删除数据库：`db.dropDatabase()`
> 注意：主要用于删除已经持久化的数据库

### 3.2 集合操作
集合，类似关系型数据库中的表
可以显示的创建，也可以隐式的创建
#### 3.2.1 集合的显示创建（了解）
`db.createCollection(name)`
参数说明：name是要创建集合的名称

`show collections`

#### 3.2.2集合的隐式创建
当向一个集合中插入一个文档的时候，如果集合不存在，则会自动创建集合。
> 通常使用隐式创建文档即可。

#### 3.2.3 集合的删除
```
db.collection.drop()
db.集合.drop()
```
返回值：如果成功删除选定集合，则drop()方法返回true，否则返回false。

### 3.3 文档基本CRUD
文档(document)的数据结构和JSON 基本—样。
所有存储在集合中的数据都是BSON格式。

https://docs.mongodb.com/v4.4/crud/
#### 3.3.1 文档的插入
1. 单个文档插入
使用insert()或 save()方法向集合中插入文档，语法如下:
```
db.co1lection.insert(
    <document or array of documents>,
    {
        writeConcern: <document>,
        ordered: <boolean>
    }
)
```
2. 批量插入
```
dh.co1lectioh.insertMany(
    [ <document 1> , <document 2>，... ],
    {
        writeConcern: <document> ,
        ordered: <boolean>
    }
)

```

### 3.3.4 常用命令小结
```
选择切换数据库:use articledb
插入数据:db.comment.insert({bson数据}
查询所有数据:db. comment.find();
条件查询数据:db.comment.find({条件})
查询符合条件的第一条记录:db.oomment.findone({条件})
查询符合条件的前几条记录:db.comment.find({条件}).limit(条数)
查询符合条件的跳过的记录:db.comment.find({条件}).skip(条数)
修改数据: db.comment.update({条件},{修改后的数据])或db.comment.update({条件}, {$set:{要修改部分的字段:数据})
修改数据并自增某字段值:db.comment.update({条件3，{$inc:{自增的字段:步进值}})
删除数据:db.comment.remove({条件})
统计查询:db.comment.count({条件})
模糊查询: db.comment.find({字段名:/正则表达式/)
条件比较运算:db.comment.find({字段名:{$gt:值}})
包含查询: db.comment.find({字段名:{$in:[值1，值2]}})或db .comment.find({字段名:$nin:[值1，值2]31)条件连接查询: db.comment.find({$and :[{条件1]},{条件2}]])或db.comment.find({$or :[{条件1},{条件2}]})

```

## 3. 安全认证
### 3.1 MongoDB的用户和角色权限简介
默认情况下，MongoDB实例启动运行时是没有启用用户访问权限控制的，也就是说，在实例本机服务器上都可以随意连接到实例进行各种操作，MongoDB不会对连接客户端进行用户验证，这是非常危险的。

mongodb官网上说，为了能保障mongodb的安全可以做以下几个步骤:

1. 使用新的端口，默认的27017端口如果一旦知道了ip就能连接上，不太安全。
2. 设置mongodb的网络环境，最好将mongodb部署到公司服务器内网，这样外网是访问不到的。公司内部访问使用vpn等。
3. 开启安全认证。认证要同时设置服务器之间的内部认证方式，同时要设置客户端连接到集群的账号密码认证方式。

为了强制开启用户访问控制(用户验证)，则需要在MongoDB实例启动时使用选项--auth或在指定启动配置文件中添加选项auth=true 。

在开始之前需要了解一下概念
1. 启用访问控制

    MongoDB使用的是基于角色的访问控制(Role-Based Access Control,RBAC)来管理用户对实例的访问。通过对用户授予一个或多个角色来控制用户访问数据库资源的权限和数据库操作的权限，在对用户分配角色之前，用户无法访问实例。

2. 角色

    在MongoDB中通过角色对用户授予相应数据库资源的操作权限，每个角色当中的权限可以显式指定，也可以通过继承其他角色的权限，或者两都都存在的权限。
3. 权限

    权限由指定的数据库资源(resource)以及允许在指定资源上进行的操作(action)组成。
    1. 资源(resource)包括:数据库、集合、部分集合和集群;
    2. 操作(action)包括:对资源进行的增、删、改、查(CRUD)操作。

角色说明：

![1635410436876.png](https://s2.loli.net/2022/01/19/prknOBv3Uz6gdx7.png)

### 3.2 添加用户和权限
1. 关闭已经开启的服务

`sudo service mongod stop`

2. 使用mongo客户端登录
3. 创建两个管理员用户，一个系统的超级管理员myroot，一个是admin库的管理员用户myadmin:
```
> use admin
// 创建系统超级用户 myroot，设置密码123456，设置角色root
> db.createUser({user:"myroot",pwd:"123456",roles:["root"]})
// 创建专门用于管理admin库的账号myadmin，只用来作为用户权限的管理
> db.createUser({user:"myadmin",pwd:"123456",roles:[{role:"userAdminAnyDatabase",db:"admin"}]})
// 查看已经创建了的用户情况
> db.system.users.find()
// 删除用户
> db.dropUser("myadmin")
// 修改密码
> db.changeUserPassword("myroot","123456")
```

    提示:
    1. 本案例创建了两个用户，分别对应超管和专门用来管理用户的角色，事实上，你只需要一个用户即可。如果你对安全要求很高，防止超管泄漏，则不要创建超管用户。
    2. 和其它数据库（MySQL)一样，权限的管理都差不多一样，也是将用户和权限信息保存到数据库对应的表中。Mongodb存储所有的用户信息在admin数据库的集合system.users中，保存用户名、密码和数据库信息。
    3. 如果不指定数据库，则创建的指定的权限的用户在所有的数据库上有效，如{role:"userAdminAnyDatabase"，db: ""}

4. 认证测试
```
// 切换到admin
> use admin
// 测试密码
> db.auth("myroot","123456")
```
5. 创建普通用户

创建普通用户可以在没有开启认证的时候添加，也可以在开启认证之后添加，但开启认证之后，必须使用有操作admin库的用户登录认证后才能操作。底层都是将用户信息保存在了admin数据库的集合system.users中。
```
// 切换到将来要操作的数据库articledb
> use articledb
// 创建用户，拥有articledb数据库的读写权限readWrite，密码是123456
> db.createUser({user:"user",pwd:"123456",roles:[{role:"readWrite",db:"environmental"}]})
```

    提示:
    如果开启了认证后，登录的客户端的用户必须使用admin库的角色，如拥有root角色的myadmin用户，再通过myadmin用户去创建其他角色的用户

### 3.3 在服务端开启认证和客户端连接登录
1. 关闭已经开启的服务
2. 启动时开启权限认证

    两种方式：
    1. 在启动时添加指定参数 --auth
    2. 配置文件方式

    在配置文件中添加
    ```
    security:
      #开启授权认证
      authorization: enabled
    ```
3. 开启认证后客户端登录

此时登录仍然可以正常连接，但可以发现打印的日志变少了，相关操作需要认证才可以。
认证方式：`db.auth("myadmin","123456")`

4. 远程连接
uri: `mongodb://user:123456@localhost:27017/articledb
`