---
title: 基于docker搭建MySQL开发环境
date: 2019-06-12 23:40:53
categories:
- wiki
tags:
- mysql
- docker
- sqlpad
---
最近需要做一些实验，需要mysql环境，想到用docker来搭建应该很快，于是尝试了下，整个过程的确比较快，不仅完成了mysql环境搭建，还顺带解决了界面化sql的问题，整个过程解决了不少问题，想记录下来备忘，于是有了这篇笔记。

# 安装docker
Mac下安装docker，直接到docker官网下载docker-desktop安装包安装即可。不过docker-desktop的安装包比较大，建议用下载软件下载。笔者下载的时候大约安装包超过了500M。
## docker镜像加速
因为网络原因，从docker官方仓库下载镜像速度比较慢，可以通过阿里云进行镜像加速。方法是注册一个阿里云账号并登陆，访问[镜像加速器](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)获得阿里云分配的Docker加速器地址。如下图：
![阿里云docker镜像加速器](aliyun_mirror.jpg "阿里云docker镜像加速器")

之后打开docker-desktop的preference，将上面获得的镜像加速链接，配置到docker中并生效：
![](docker_config.jpg "docker配置镜像加速地址")

## 启动mysql
docker安装好之后，就可以进行docker实例启动了，我选择了mysql的[官方镜像](https://hub.docker.com/_/mysql)，用下面的命令下载和启动msyql镜像：
```sh
docker run -p 3306:3306 --name test-mysql -e MYSQL_ROOT_PASSWORD=password -d mysql:latest
```
上面的命令中`-p 3306:3306`将docker中的3306端口映射到宿主机的3306端口，这样我就能在外部访问mysql docker实例的服务了。
之后连接到mysql docker实例上，登陆数据库做一些必要的配置：
```sh
## 在docker实例上执行交互式bash
docker exec -it test-mysql bash
## 以root身份连接mysql
mysql -uroot -ppassowrd

```
配置root账号可以远程登陆mysql（主要是方便测试，实际工程当然是不推荐的）：
```sql
-- 允许root账户远程登陆
grant all PRIVILEGES on *.* to root@'%' WITH GRANT OPTION;
```

修改root账号的密码加密模式，这是因为mysql8默认采用`caching_sha2_password`而不是`mysql_native_password`，这会导致下面要介绍的sqlpad或者其他mysql客户端，无法通过mysql的鉴权，报类似下面的错误：
```text
MYSQL：ER_NOT_SUPPORTED_AUTH_MODE:Client does not support authentication protocol
```
因此需要修改root账号的密码加密方案：
```sql
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'test';
```

# 安装配置sqlpad
习惯了用浏览器直接操作db，实在不像安装客户端，偶然发现了sqlpad，经过简单的配置就能用起来，简直惊为天人。首先安装sqlpad：
```sh
npm install sqlpad -g
```
切换到存储sqlpad配置数据的目录，执行下面的命令启动sqlpad：
```sh
sqlpad --dir ./ --port 3000 --passphrase secret-encryption-phrase
```
之后在浏览器访问http://127.0.0.1:3000 就能开始使用sqlpad。先注册账号，在用账号登陆，这里注册的账号都是存储在本地文件中的。登陆之后的界面如下：
![](sqlpad_create_connection.jpg "sqlpad界面")

点击上图的connections创建连接：
![](connection.jpg "添加mysql连接")

可以用下方的test进行连接可用性测试，添加好连接之后，就可以使用啦，如下图：
![](example.jpg "sqlpad执行样例")

# 参考资料
1. [sqlpad安装文档](http://rickbergfalk.github.io/sqlpad/installation-and-administration/)
2. [docker mysql 镜像](https://hub.docker.com/_/mysql)
3. [docker-desktop](https://www.docker.com/products/docker-desktop)
4. [使用阿里云进行docker镜像加速](https://yq.aliyun.com/articles/29941)
5. [sqlpad: 可视化sql交互](https://github.com/rickbergfalk/sqlpad)
6. [mysql官方docker镜像](https://hub.docker.com/_/mysql)
