---
title: screen 指令的使用
date: 2022-04-26 15:08:03
categories:
- linux
tags:
- linux
- screen
---
# screen指令使用
## 功能
+ 会话恢复：只要Screen本身没有终止，在其内部运行的会话都可以恢复。
+ 多窗口：在Screen环境下，所有的会话都独立的运行，并拥有各自的编号、输入、输出和窗口缓存。
+ 会话共享：Screen可以让一个或多个用户从不同终端多次登录一个会话，并共享会话的所有特性。

## 查看终端列表
``` shell
screen -ls
```

## 新建终端
``` shell
screen -R name # 如果有同名终端会直接进入之前的终端，推荐
screen -S name # 不检录之前终端，直接创建新终端
```

## 退出终端
按 `ctrl + a`，再按 `d` ，保持这个screen后台运行并回到主终端

## 清除终端
``` shell
# 在终端内
exit
# 主终端内
screen -R [pid/Name] -X quit
```

# 参考
1. https://cloud.tencent.com/developer/article/1844735
