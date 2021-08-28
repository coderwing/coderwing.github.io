---
layout: post
title: Redis持久化
categories: [Redis, 缓存, 数据库]
description: redis持久化
keywords: redis, 缓存, AOF, RDB
---


# redis持久化

### Linux父子进程
父进程的变量数据，经过 【export 变量】之后，子进程就可以拿到对应的变量，子进程就可以操作了；
这时，父进程修改该变量，子进程中的变量不受影响；同样的，子进程修改该变量，父进程的变量也不会受影响；
因此，子父进程的数据是相互隔离的。
使用echo 和 export演示。
这个操作就是在内存中fork了一个变量副本，fork之后变量会对进程相互隔离。

**操作原理**
在fork的时候其实还是只有一份数据，只不过在父进程操作的时候会发生CopyOnWrite（cow），父进程会将原来的变量数据copy一份write到新的内存地址，这时候就变成了两份数据了。

![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/1268256698641.png)


### fork()
> 在redis的持久化中，进行持久化时，需要指定持久化的时间节点，那么就要保证持久化出来的文件中的数据是对应时间节点的数据。

比如8:00进行持久化：

**1、阻塞式持久化**
> 持久化线程将阻塞redis服务，等待持久化之后redis服务才能正常运行，如果数据量较大，持久化时间较长，对业务影响较大，该方案不可用。

**2、非阻塞多线程**
> redis提供服务的同时进行持久化，问题在于，数据是持续变化的，持久化的数据不能保证都是8:00时间节点的数据，因此该方案更不可行。

**3、非阻塞fork子进程**
> 通过fork子进程，将redis的数据在内存中快速fork出副本（指针引用），然后通过子进程进行持久化，父进程可以正常提供redis服务，父子进程的数据相互隔离。


### copyonwrite

顾名思义，就是在write的时候，copy一份同样的数据来操作；这样不会影响原来的数据，这也是上面讲到的fork出一个子进程之后，对数据的操作方式，遵循CoW。


### RBD
Redis DB 是redis的快照。

使用了bgsave操作，bgsave也是一个同步命令；执行bgsave之后为fork一个子进程出来完成持久化操作。

对应的配置文件配置项：

```shell
# 刷新快照到硬盘中，必须满足两者要求才会触发，即900秒之后至少1个关键字发生变化。
save 900 1
# 必须是300秒之后至少10个关键字发生变化
save 300 10
# 必须是60秒之后至少10000个关键字发生变化
save 60 10000
```

同样的还有一个**SAVE**命令：
> SAVE 命令用于创建当前数据库的备份;
该命令将在 redis 安装目录中创建dump.rdb文件。

SAVE命令在服务器需要关机维护、重启前执行。

恢复数据：

```shell
# 如果需要恢复数据，只需将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可；
# 获取 redis 目录可以使用 CONFIG 命令，如下所示：
redis 127.0.0.1:6379> CONFIG GET dir
1) "dir"
2) "/usr/local/redis/bin"
```
注意：redis的持久化产生的文件永远只有一个，只有一个dump.rdb文件


#### RDB优缺点

**RDB的优点**

```
1、RDB是一个非常紧凑的文件,它保存了某个时间点得数据集,非常适用于数据集的备份,比如你可以在每个小时报保存一下过去24小时内的数据,同时每天保存过去30天的数据,这样即使出了问题你也可以根据需求恢复到不同版本的数据集.
2、RDB是一个紧凑的单一文件,很方便传送到另一个远端数据中心或者亚马逊的S3（可能加密），非常适用于灾难恢复.
3、RDB在保存RDB文件时父进程唯一需要做的就是fork出一个子进程,接下来的工作全部由子进程来做，父进程不需要再做其他IO操作，所以RDB持久化方式可以最大化redis的性能.
4、与AOF相比,在恢复大的数据集的时候，RDB方式会更快一些.
```
**RDB的缺点**

```
1、如果你希望在redis意外停止工作（例如电源中断）的情况下丢失的数据最少的话，那么RDB不适合你.虽然你可以配置不同的save时间点(例如每隔5分钟并且对数据集有100个写的操作),是Redis要完整的保存整个数据集是一个比较繁重的工作,你通常会每隔5分钟或者更久做一次完整的保存,万一在Redis意外宕机,你可能会丢失几分钟的数据.
2、RDB 需要经常fork子进程来保存数据集到硬盘上,当数据集比较大的时候,fork的过程是非常耗时的,可能会导致Redis在一些毫秒级内不能响应客户端的请求.如果数据集巨大并且CPU性能不是很好的情况下,这种情况会持续1秒,AOF也需要fork,但是你可以调节重写日志文件的频率来提高数据集的耐久度.
```

### AOF
AOF：Append Only File
aof持久化只会向文件中追加写操作的数据。

RDB和AOF可以同时开启，但是两个都开启的话，恢复数据的话只有AOF起作用，因为AOF记录会更全面，会包含RDB的全量数据。

**AOF重写**

> 重新在老版本中会使用一个命令：BGREWRITEAOF，表示后台重写AOF文件；
aof在进行append的时候会重复的将同一个key记录下来比如set key1 a会写入aof文件中，然后set key1 b，也会写入aof文件中，执行重写后，最后只会剩下最后的set key1 b命令，从而实现使aof文件缩小的目的。

#### AOF优缺点

**AOF优点**
```
1、使用AOF 会让你的Redis更加耐久: 你可以使用不同的fsync策略：无fsync,每秒fsync,每次写的时候fsync.使用默认的每秒fsync策略,Redis的性能依然很好(fsync是由后台线程进行处理的,主线程会尽力处理客户端请求),一旦出现故障，你最多丢失1秒的数据.
2、AOF文件是一个只进行追加的日志文件,所以不需要写入seek,即使由于某些原因(磁盘空间已满，写的过程中宕机等等)未执行完整的写入命令,你也也可使用redis-check-aof工具修复这些问题.
3、Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写： 重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
4、AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单： 举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。
```


**AOF缺点**

```
1、对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积，因为AOF是不断追加的。
2、根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。
```

RDB和AOF具体参考：http://redis.cn/topics/persistence.html

### RDB&AOF持久化演示

```shell
## 使用到的命令
BGSAVE # rdb持久化命令
LASTSAVE # 用来检查BGSAVE的执行结果，得到最后一次同步RDB的时间
BGREWRITEAOF AOF重写操作（混合模式）缩减AOF文件

# 对于AOF开启后会启动执行aof写入操作
```

**演示**

> 6379 redis服务数据目录：/var/lib/redis/6379

```shell

## 之慈宁宫三次set k1的写命令
127.0.0.1:6379> set k1 a
OK
127.0.0.1:6379> set k1 b
OK
127.0.0.1:6379> set k1 c
OK
```

查看aof数据
```shell
vim appendonly.aof

## aof文件规则，以set k1 c 命令为例：
*3^M # *表示命令记录的开始，3表示该条命令有3个元素组成
$3^M # $3 表示后面的元素长度为3，set是三个字符组成
set^M
$2^M # $2 表示后面的元素由2个字符组成，k1由2个字符组成
k1^M
$1^M
c^M

## aof文件内容展示
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂ¹V*aú^Hused-memÂx^E^M^@ú^Laof-preambleÀ^Aÿû/Öì1gÈ¨*2^M
$6^M
SELECT^M
$1^M
0^M
*3^M
$3^M
set^M
$2^M
k1^M
$1^M
a^M
*3^M
$3^M
set^M
$2^M
k1^M
$1^M
b^M
*3^M
$3^M
set^M
$2^M
k1^M
$1^M
c^M
~
```
可以看到上面的命令记录k1的set操作被记录了3次，这就是append

**同步RDB文件**

```shell
## 直接执行BGSAVE操作即可同步RDB文件
# SAVE命令也可以，只不过SAVE命令是阻塞的
127.0.0.1:6379> BGSAVE
Background saving started

# 查看RDB文件内容

root@iZ2ze4jt2xg0kcbwp242yxZ:/var/lib/redis/6379# vi dump.rdb

# 内容 1,1           All
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂfX*aú^Hused-memÂÈ^E^M^@ú^Laof-preambleÀ^@þ^@û^A^@^@^Bk1^Acÿ<87>-¬:³^Q^@n


```

**重写AOF文件（缩减）**

```shell
## 重写AOF
127.0.0.1:6379> BGREWRITEAOF
Background append only file rewriting started
```

查看AOF和RDB文件内容



```shell

## 查看AOF文件
root@iZ2ze4jt2xg0kcbwp242yxZ:/var/lib/redis/6379# vim appendonly.aof

# 内容
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂïX*aú^Hused-memÂÐ^E^M^@ú^Laof-preambleÀ^Aþ^@û^A^@^@^Bk1^AcÿçªJ<8d><8c>K :


## 查看RDB文件
root@iZ2ze4jt2xg0kcbwp242yxZ:/var/lib/redis/6379# vi dump.rdb

# 内容
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂfX*aú^Hused-memÂÈ^E^M^@ú^Laof-preambleÀ^@þ^@û^A^@^@^Bk1^Acÿ<87>-¬:³^Q^@n

```

这时候RDB和AOF都是最小化了。

**再次加入数据**
```shell
## AOF文件内容
REDIS0009ú      redis-ver^E5.0.5ú
redis-bitsÀ@ú^EctimeÂïX*aú^Hused-memÂÐ^E^M^@ú^Laof-preambleÀ^Aþ^@û^A^@^@^Bk1^AcÿçªJ<8d><8c>K :*2^M
$6^M
SELECT^M
$1^M
0^M
*3^M
$3^M
set^M
$2^M
k1^M
$1^M
d^M
*3^M
$3^M
set^M
$2^M
k2^M
$1^M
q^M

```
> 可以看到AOF的内容前半部分是同步了之前的数据，后面是后来追加的命令。

**混合持久化**
在配置文件中开启：

```
# 是否开启混合持久化模式4.0之后支持
aof-use-rdb-preamble yes
```

> 执行重写后，再append命令，这样可以实现快速送rdb中同步，然后把最新的命令追加进来，既保证速度，又保证全量。
混合持久化的缺点就是：版本兼容性差，支持4.0之后的版本。