---
title: Spring多模块&Vue项目部署
date: 2022-01-22 17:09:17
categories:
- web
tags:
- web
- spring
- vue
---

# Spring多模块项目部署
## 项目打包
先说明一下项目的模块划分结构，项目一共分为了三个模块，每个模块的作用如下：
+ `environmental-kg` 知识图谱相关模块
+ `environmental-main` 项目整体配置模块
+ `environmental-web` 权限管理模块

项目启动时，`environtal-main` 模块包含了 `spring` 的启动入口，即`SpringBootApplication`。所以在打包的时候，从`environmental-main` 模块打包。  

项目打包时需要在 `environmental-main` 模块下添加以下代码：
``` xml
<build>
    <finalName>${project.artifactId}</finalName><!--打jar包去掉版本号-->
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <mainClass>com.bupt.EnvironmentalApplication</mainClass>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
其中注意两点：
1. 指定main函数位置
2. repackage的作用
   + 在原始Maven打包形成的jar包基础上，进行重新打包，新形成的jar包不但包含应用类文件和配置文件，而且还会包含应用所依赖的jar包以及Springboot启动相关类（loader等），以此来满足Springboot独立应用的特性
   + 将原始Maven打包的jar重命名为XXX.jar.original作为原始文件

项目打包完成以后，会在 `environmental-main\target` 目录下找到 `environmental-main.jar` 文件，执行 `java -jar environmental-main.jar` 即可运行。

## 项目部署
后端服务部署参考实验室某学长写的部署规则。  

首先说以下三个注意事项：
1. 所有需要持续部署的非系统性服务，如后端接口、模型等，一律采用supervisor进行管理，不要自己用nohup运行
2. 后端接口如无特殊需求，一律建议在前端接口加10000获得，如前端部署在90端口，后端端口部署在10090，便于管理，其他服务自行决定，不要占用10090-10099
3. Java后端服务请添加参数 -Xms256M -Xmx256M，建议服务不常并发的话控制在256M以内，运行不了的适当放大

supervisor安装：{% post_link web/supervisor supervisor安装%}  
部署方法：  
1. 项目后端服务放置在 `/opt/projects` 文件夹中，每个项目自定义一个文件夹
2. 到 `/etc/supervisor/conf.d` 目录中，复制 `.example` 为新的配置文件，以 `项目名称.ini` 为命名，按照内容注释修改对应配置，注意以下几点：
   + 请记得修改 `program` 名称，这是 `supervisor` 管理服务的标志
   + 一个文件内可以有多个 `program` ，因此如项目有多个服务，请在一个配置文件内设置，不要设多个文件，`program` 命名可以以 `项目名称_服务名称` 的方式进行
   + 建议将log打印到项目目录里，不要放在 `/tmp` 内，便于查错
3. 配置完毕后加载对应配置，运行 `status` 查看，如为 `RUNNING` 则运行成功，否则查看错误日志重新配置，并访问应用测试
    ``` shell
    sudo supervisorctl reload # 重新加载配置文件
    sudo supervisorctl update # 重新加载配置
    sudo supervisorctl status # 查看加载状态
    sudo supervisorctl start name # 启动某一个项目
    ```
配置文件模板
``` conf
[program:environmental-api]
directory = /data/sdb1/yxy/environmental ; 程序的启动目录
command = java -jar environmental-main.jar -Xms256M -Xmx256M ;-Dsprint.config.location=./application.properties 启动命令，可以看出与手动在命令行启动的命令是一样的
autostart = true ; 在 supervisord 启动的时候也自动启动
startsecs = 5 ; 启动 5 秒后没有异常退出，就当作已经正常启动了
autorestart = true ; 程序异常退出后自动重启
startretries = 3 ; 启动失败自动重试次数，默认是 3
user = root ; 用哪个用户启动
redirect_stderr = true ; 把 stderr 重定向到 stdout，默认 false
stdout_logfile_maxbytes = 20MB ; stdout 日志文件大小，默认 50MB
; stdout 日志文件，需要注意当指定目录不存在时无法正常启动，所以需要手动创建目录（supervisord 会自动创建日志文件）
stdout_logfile = /data/sdb1/yxy/environmental/logs/enviornmental.log
```

> 这里说一下为什么要用supervisor，而不是用nohup &。nohup &本质上就是后台运行，supervisor则是将其变成守护进程(daemon)，更加安全可靠，服务挂了还可以自己重启。
> > 守护进程是一类在后台运行的特殊进程，用于执行特定的系统任务。他独立于控制终端并且周期性的执行某种任务或等待处理某些发生的事件。Linux系统的大多数服务器就是通过守护进程实现的。


# Vue项目部署
## 项目打包
vue项目打包比较简单，执行以下两个指令之一就行了
``` shell
yarn build
npm run build
```

## 项目部署
vue项目部署到nginx服务器上。nginx在ubuntu上执行`sudo apt install nginx`即可安装。  
nginx默认的配置文件地址为: `/etc/nginx/nginx.conf` 。一般不需要修改过多内容，会在 `http` 下添加配置: `include /etc/nginx/conf.d/*.conf;`。然后在conf.d文件夹中添加各个项目的配置。

配置文件模板
``` conf
server {
	listen 83;

	root /var/www/environmental;

	server_name _;

	location / {
		try_files $uri $uri/ @router;
        index  index.html;
	}

	location @router {
        rewrite ^.*$ /index.html last;
	}

	location /api {
        proxy_pass http://10.109.246.245:8090/api;
	}
}
```

具体的部署步骤：
1. 首先将前端相关文件放置在 `/var/www`下，每个项目一个文件夹
2. 在 `/etc/nginx/conf.d` 中，复制 `.example` 文件为新的项目配置文件，以 `项目名称.conf` 为文件名，例如
`sudo cp .example environmental.conf`
   + 修改对应的配置内容，主要关注：
   + listen：监听端口号
   + root：项目地址
   + localtion /api 对后端请求的转发	
3. 配置完成后重启nginx: `sudo service nginx restart`
4. 访问端口测试: `http:10.109.246.245:83`  

> 注：www文件夹权限组为root，nginx服务运行于root，因此前端通常只读不可写，如有需要在前端目录进行写入，请自行转换权限组到root，不要设置777。

