
---
layout: post
title: Redis单机安装的正确姿势
categories: [Redis, 缓存, 数据库]
description: redis单机安装
keywords: redis, 缓存
---

# 单机redis正确安装姿势

## 下载redis包
> 下载URL：https://download.redis.io/releases/redis-5.0.5.tar.gz
> 如果需要下载其他版本,参考: https://download.redis.io/releases/

在自定义的redis目录下安装：
```shell
wget https://download.redis.io/releases/redis-5.0.5.tar.gz
```
等待执行完成：
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/1136340615710.png)

## 安装

### 1、解压tar包
```shell
## 解压
tar xf redis-5.0.5.tar.gz
## 进入解压后的目录
cd redis-5.0.5
```
### 2、查阅README
<img src="https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/5654767941461.png" style="zoom:80%;" />

> 在README文件中提供好了非常详细的官方安装步骤

查阅后提取README关键步骤内容：  
#### 普通安装
```shell
## 安装
make  # 32位使用 mkae 32bit

## 如果构建中出现了一些问题使用该命名
make distclean # 构建时使用的可能性比较大

## 安装完成后进入src目录
cd src
## 进入src后启动redis服务
./redis-server

## 可以指定配置文件启动服务
./redis-server /path/to/redis.conf

## 还可以指定端口ip等启动服务
./redis-server --port 9999 --replicaof 127.0.0.1 6379
./redis-server /etc/redis/6379.conf --loglevel debug # 指定日志级别

## 启动命令行交互式命令
./redis-cli
```
注意：
> 以上的make安装是将服务安装到了当前的目录使用，启动server需要先进入到src中才可以执行redis-server命令


#### 安装到bin中
```shell
## 安装到bin中
make install

## 将可执行文件单独提取到其他路径目录
make install PREFIX=/usr/apps/bins/redis5 # PREFIX指定其他目录
```
** **安装后需要配置系统路径**


#### 本地服务安装
> 在README中提到了安装目录中的utils目录（上图中已指出）
```shell
cd utils
./install_server.sh
```
通过install_server.sh可以实现将redis服务安装成本地服务，并且开机启动

### 3、正式安装

有了上面的参考，可以顺利的进行单机的安装。
```shell
## 执行make
make

## 可能会发生GCC的警告错误，如果发生就安装GCC
apt-get install gcc

## 然后执行clean
mkae distclean # 清理掉之前的安装数据

## 执行install，指定可执行命令文件单独路径
make install PREFIX=/usr/apps/bins/redis5 # 会自动创建不存在的路径

## 配置环境变量
    # 编辑profile
vim /etc/profile 

    #  写入内容
export REDIS_HOME=/usr/apps/bins/redis5
export PATH=$PATH:$REDIS_HOME/bin

    # 刷新profile
source /etc/profile


## 进入utils目录
cd utils

## 执行install_server.sh
./install_server.sh # 后面有参考图步骤

```

##### 单独可执行文件路径

![单独可执行文件路径](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/1643303899865.png)


##### 进入utils后的操作
第一步：指定端口号，默认6379
![指定端口号](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/275140586507.png)

回车后，需要指定配置文件，使用默认，会根据输入的端口号自动生成一个配置文件
![指定配置文件](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/3424540132985.png)

指定日志文件，同样根据端口号生成
![生成日志文件](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/2392189947329.png)

指定data数据目录
![指定data数据目录](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/29900817515.png)

再次确认server可执行文件目录：该目录就是指定的独立可执行文件目录
![server可执行文件](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/5489932911655.png)


再次去人输入或选定的配置信息：确定**回车**，取消**Ctrl+C**
![再次确认配置信息](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/706830533195.png)

> 【记录配置信息】
Selected config:
Port           : 6379
Config file    : /etc/redis/6379.conf
Log file       : /var/log/redis_6379.log
Data dir       : /var/lib/redis/6379
Executable     : /usr/apps/bins/redis5/bin/redis-server
Cli Executable : /usr/apps/bins/redis5/bin/redis-cli
Is this ok? Then press ENTER to go on or Ctrl-C to abort.

回车后会输出启动服务的过程：
![启动服务过程](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/3688943659031.png)

过程：  
> 第一步：复制配置文件到/etc/init.d中，作为系统服务的配置使用
第二步：将redis安装到系统服务
第三步：启动系统服务


### 4、安装多个redis
> 根据上面使用utils目录中的命令安装的逻辑步骤，可以使用不同的端口安装多个redis服务
#### 安装6380端口redis
![指定新端口启动服务](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/3951896559571.png)  
根据上图，我们再次执行utils目录下载install_server.sh，指定了6380新端口，又启动了一个新的redis服务。

使用ps命令查看运行的redis服务：
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/单机redis正确安装姿势.md/2412094761973.png)
可以看到有两个redis服务