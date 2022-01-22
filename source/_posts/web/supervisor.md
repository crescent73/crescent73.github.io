---
title: supervisor安装与配置
date: 2022-01-22 18:54:27
categories:
- web
tags:
- web
- supervisor
---

# supervisor安装与配置
后端部署时，一般都用的nohup直接后台运行了，之前师兄说建议用supervisor，正好试一下&学习下。  
supervisor官方文档：http://supervisord.org/installing.html

## 安装
服务器为ubuntu，两个方式安装apt-get或者pip，apt-get不知道为啥安不下来，最后用的pip安的。指令如下：
``` shell
sudo apt-get install supervisor
pip3 install supervisor
```
下面继续说pip安装的后续操作。  

## 配置
将 `supervisor` 的所有配置都放到 `/etc/supervisor` 目录下，因此执行以下指令：
``` shell
user$ sudo su # 切换root账号
root# mkdir /etc/supervisor # 创建目录
root# echo_supervisord_conf > /etc/supervisor/supervisord.conf # 复制配置文件
root# mkdir conf.d # 创建每个项目的配置文件文件夹
root# mkdir logs
```
`supervisord.conf` 配置文件主要修改内容如下：
``` conf
[unix_http_server]
file=/etc/supervisor/supervisor.sock   ; the path to the socket file

[supervisord]
logfile=/etc/supervisor/logs/supervisord.log ; main log file; default $CWD/supervisord.log
pidfile=/etc/supervisor/supervisord.pid ; supervisord pidfile; default supervisord.pid

[supervisorctl]
serverurl=unix:///etc/supervisor/supervisor.sock ; use a unix:// URL  for a unix socket

[include]
files = /etc/supervisor/conf.d/*.ini
```
其中最重要的时配置 `include` ，这里有个小坑，就是 `[include]` 前面也需要取消注释，否则 `include` 设置不生效。

`conf.d` 文件夹名格式为 `xxx.ini` 。每个文件的配置模板如下：
``` conf
[program:echo_date_server]
directory = /root/scripts ; 程序的启动目录
command =  sh echo_date.sh ; 启动命令，可以看出与手动在命令行启动的命令是一样的
autostart = true ; 在 supervisord 启动的时候也自动启动
startsecs = 5 ; 启动 5 秒后没有异常退出，就当作已经正常启动了
autorestart = true ; 程序异常退出后自动重启
startretries = 3 ; 启动失败自动重试次数，默认是 3
user = bupt ; 用哪个用户启动
redirect_stderr = true ; 把 stderr 重定向到 stdout，默认 false
stdout_logfile_maxbytes = 20MB ; stdout 日志文件大小，默认 50MB
; stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录（supervisord 会自动创建日志文件）
stdout_logfile = /tmp/echo_stdout.log
```
## 启动
执行以下命令启动supervisor: `sudo supervisord -c /etc/supervisor/supervisord.conf`  
服务启动后可以使用supervisorctl命令进行进程管理。

supervisorctl常用指令
``` shell
supervisorctl status        //查看所有进程的状态
supervisorctl stop es       //停止es
supervisorctl start es      //启动es
supervisorctl restart       //重启es
supervisorctl update        //配置文件修改后使用该命令加载新的配置
supervisorctl reload        //重新启动配置中的所有程序
```