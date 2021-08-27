---
layout: post
title: Redis配置文件和常用命令
categories: [Redis, 缓存, 数据库]
description: redis config and cmd
keywords: redis, 缓存, redis配置文件
---

# Redis配置文件详解


## 基本配置

```shell

# 是否以后台进程启动
daemonize no

# 创建database的数量(默认选中的是database 0)
databases 16

# 刷新快照到硬盘中，必须满足两者要求才会触发，即900秒之后至少1个关键字发生变化。
save 900 1
# 必须是300秒之后至少10个关键字发生变化
save 300 10
# 必须是60秒之后至少10000个关键字发生变化
save 60 10000

# 后台存储错误停止写
stop-writes-on-bgsave-error yes

# 使用LZF压缩rdb文件
rdbcompression yes
# 存储和加载rdb文件时进行校验
rdbchecksum yes
# 设置rdb文件名
dbfilename dump.rdb
# 设置工作目录，rdb文件会写入该目录。
dir ./

```

## 主从配置

```shell

# 设为某台机器的从服务器
slaveof <masterip> <masterport>

# 连接主服务器的密码
masterauth <master-password>

# 当主从断开或正在复制中,slave服务器是否应答
slave-serve-stale-data yes

# slave服务器是否只读
slave-read-only yes

# salve节点 ping master节点的时间间隔,秒为单位
repl-ping-slave-period 10

# 主从超时时间(超时认为断线了),要比period大
repl-timeout 60

# 如果master不能再正常工作，那么会在多个slave中，选择优先值最小的一个slave提升为master，
# 优先值为0表示不能提升为master
slave-priority 100

#主端是否合并数据,大块发送给slave
repl-disable-tcp-nodelay no
```


## 安全

```shell
# 需要密码
requirepass foobared

# 如果公共环境, 可以重命名部分敏感命令 如config
rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52 
```



## 限制

```shell
# 最大连接数
maxclients 10000

# 最大使用内存
maxmemory <bytes>

# 内存到极限后的处理：淘汰策略
# volatile-lru -> LRU算法删除过期key
# allkeys-lru -> LRU算法删除key(不区分过不过期)
# volatile-random -> 随机删除过期key
# allkeys-random -> 随机删除key(不区分过不过期)
# volatile-ttl -> 删除快过期的key
# noeviction -> 不删除,返回错误信息
maxmemory-policy volatile-lru


# 解释 LRU ttl都是近似算法,可以选N个,再比较最适宜T踢出的数据
maxmemory-samples 3
```

## 日志模式

```shell
# 是否仅要日志
appendonly no


### appendfsync ###
# everysec ：折衷,每秒写1次
# always：系统不缓冲,直接写,慢,丢失数据少
# 系统缓冲, 统一写, 速度快
appendfsync no

# 为yes,则其他线程的数据放内存里,合并写入(速度快,容易丢失的多)
no-appendfsync-on-rewrite no

# 当前aof文件是上次重写是大N%时重写
auto-AOF-rewrite-percentage 100

# aof重写至少要达到的大小
auto-AOF-rewrite-min-size 64mb
```

## 慢查询

```shell
# 记录响应时间大于10000微秒的慢查询
slowlog-log-slower-than 10000

# 最多记录128条
slowlog-max-len 128
```


## 服务端命令

```shell
# 返回时间戳+微秒
time

# 返回key的数量
dbsize

# 重写aof
bgrewriteaof  

# 后台开启子进程dump数据
bgsave

# 阻塞进程dump数据
save
lastsave

# 做host port的从服务器(数据清空,复制新主内容)
slaveof host port

# 变成主服务器(原数据不丢失,一般用于主服失败后)
slaveof no one

# 清空当前数据库的所有数据
flushdb

# 清空所有数据库的所有数据(误用了怎么办?)
# 运维会将此类命令重命名，基本处于禁用状态
flushall

# 关闭服务器,保存数据,修改AOF(如果设置)
shutdown [save/nosave]

# 获取慢查询日志
slowlog get

# 获取慢查询日志条数
slowlog len

# 清空慢查询
slowlog reset
```


```shell
info []

# 选项(支持*通配)
config get

# 选项 值
config set

# 把值写到配置文件
config rewrite

# 更新info命令的信息
config restart

# 调试选项,看一个key的情况
debug object key

# 模拟段错误,让服务器崩溃
debug segfault
object key (refcount|encoding|idletime)

# 打开控制台,观察命令(调试用)
monitor

# 列出所有连接
client list

# 杀死某个连接  CLIENT KILL 127.0.0.1:43501
client kill

# 获取连接的名称 默认nil
client getname

# 设置连接名称,便于调试
client setname "名称" 
```



## 连接命令

```shell

# 密码登陆(如果有密码)
auth 密码

# 测试服务器是否可用
ping

# 测试服务器是否正常交互
echo "some content"

# 选择数据库
select 0/1/2...

#退出连接
quit
```
