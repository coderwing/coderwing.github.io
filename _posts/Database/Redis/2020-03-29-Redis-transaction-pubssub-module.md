---
layout: post
title: Redis发布订阅、事务、模块&布隆过滤器
categories: [Redis, 缓存, 数据库]
description: redis基础功能使用
keywords: redis, 缓存
---

# Redis发布订阅、事务、模块&布隆过滤器

### 发布订阅的使用
**使用的命令**

```shell
## 发布 PUBLISH
# channel 自定义的频道名称
# message 发布的消息
PUBLISH channel message

## 订阅 SUBSCRIBE
# channel1、channel2是订阅的频道
# 订阅的频道对应的发布方发布消息后，这里就会获取到相应的message
SUBSCRIBE channel1 [channel2 ...]

## 取消订阅 SUBSCRIBE
SUBSCRIBE ch1 [ch2 ...]

```

**示例**：

```shell
## 发布方
127.0.0.1:6379> PUBLISH ch1 hello
(integer) 0
127.0.0.1:6379> PUBLISH ch1 hello
(integer) 1
127.0.0.1:6379> PUBLISH ch1 world
(integer) 1

## 订阅方
127.0.0.1:6379> SUBSCRIBE ch1
Reading messages... (press Ctrl-C to quit)

1) "message"
2) "ch1"
3) "hello"
1) "message"
2) "ch1"
3) "world"

```

```
可以将redis的pub / sub用于实时消息的发送和接收，开启另外的redis-cllient去sub消息专门存储3天内的消息进行缓存；
更长时间的历史消息，使用sub到kafka中，使用DBservice保存到关系数据库。
```

### redis事务使用
**事务相关命令**：

```
MULTI 、 EXEC 、 DISCARD 和 WATCH
```

> 事务可以一次执行多个命令， 并且带有以下两个重要的保证：
> 1. 事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。
> 2. 事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。

redis处理逻辑：

```
两个客户端一起发送命令，client1发送了多个命令，client2发送了多个命令，过程中命令的顺序是有交叉的，哪个客户端先执行，取决于最后发送执行指令的先后，不同的客户端一次连接发送的多个命令是放在一个缓冲区的，其实是一个整体。
```
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/3126243158011.png)

为了防止执行出现问题，保证事务的正常执行，redis加入了一个watch机制：    
```
在执行命令之前，先去监听需要操作的key
```
**命令使用示例**：

```shell
## MULTI  开始执行多条命令模式
127.0.0.1:6379> MULTI
OK

## 使用set命令，这里是错误的写法
127.0.0.1:6379> set key 1 10
QUEUED # 表示命令排队
127.0.0.1:6379> set key2 20
QUEUED

## 执行
127.0.0.1:6379> exec
1) (error) ERR syntax error # 第一条命令执行出错
2) OK

```

#### 为什么 Redis 不支持回滚（roll back）
如果你有使用关系式数据库的经验， 那么 “Redis 在事务失败时不进行回滚，而是继续执行余下的命令”这种做法可能会让你觉得有点奇怪。

以下是这种做法的优点：
```
1、Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。
2、因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。

```

有种观点认为 Redis 处理事务的做法会产生 bug ， 然而需要注意的是， 在通常情况下， 回滚并不能解决编程错误带来的问题。 举个例子， 如果你本来想通过 INCR 命令将键的值加上 1 ， 却不小心加上了 2 ， 又或者对错误类型的键执行了 INCR ， 回滚是没有办法处理这些情况的。

**放弃事务**

当执行 DISCARD 命令时， 事务会被放弃， 事务队列会被清空， 并且客户端会从事务状态中退出：

```shell

> SET foo 1
OK
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
> GET foo
"1"
```



#### 使用 WATCH命令实现check-and-set （CAS）乐观锁

```shell

## client-1中执行
127.0.0.1:6379> set k1 10
OK

# 使用watch观察k1
127.0.0.1:6379> WATCH k1
OK
# 进入多命令模式
127.0.0.1:6379> MULTI
OK
# 加入set命令
127.0.0.1:6379> set k1 20
QUEUED

# 这时候还没有执行exec命令
```
这时候，client-2的命令执行了：

```shell
## 多命令模式
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> get k1
QUEUED

## 为k1设置了新的值
127.0.0.1:6379> set k1 11
QUEUED

## 在client-1之前执行
127.0.0.1:6379> exec
1) "10"
2) OK

## 被修改为11了（原来是10）
127.0.0.1:6379> get k1
"11"
```

这时候再执行client-1的那些命令：

```shell

## 执行并没有成功
# 被watch的k1 在watch期间被修改了，因此加入的命令全部不执行了
127.0.0.1:6379> exec
(nil) # 没有执行，返回nil，避免了发生不必要的错误

## 得到的是client-2修改的值，而并非client-1想要修改成的20
127.0.0.1:6379> get k1
"11"
```

> 如果一个事务中需要使用多个key，那么wacth多个可以即可。
```shell
redis> WATCH key1 key2 key3
OK
```

### redis moudle & 布隆过滤器
#### redis模块

> redis的模块是为了给redis增加一些扩展功能，安装对应的模块就可以根据对应模块提供的api使用相应的扩展功能。

```
进入到redis英文官网 https://redis.io/
可以看到标题栏有：Modules
```
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/5034340887406.png)

列出了非常多的模块：
> 这是一个 Redis 模块的列表，用于 Redis v4.0或更高版本，由 Github stars 排序。这个列表包含两组模块: 一组是 OSI 认可的模块，另一组是一些专有许可的模块。非 OSI 模块被明确标记为非开放源码。目前还必须在 Github 托管源代码。要在这里添加模块，请为 Redis-doc 存储库中的 modules.json 文件发送一个拉请求。

![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/3100414534913.png)
其中有一个RedisBloom模块，该模块就是一个布隆过滤器的扩展模块。

**下载安装RedisBloom模块**
需要注意：
如果直接下载master的zip包，解压后执行make命令会报错：

```shell
/usr/apps/redis/RedisBloom-master/src/rm_tdigest.c:18:21: fatal error: tdigest.h: No such file or directory
compilation terminated.
<builtin>: recipe for target '/usr/apps/redis/RedisBloom-master/src/rm_tdigest.o' failed
make: *** [/usr/apps/redis/RedisBloom-master/src/rm_tdigest.o] Error 1

```
**原因：**
> git下载发布的版本，master不适用。需要下载完重新解压，重新编译make。

```
1、下载master下拉列表后的tags列表中的版本：https://codechina.csdn.net/mirrors/RedisBloom/RedisBloom/-/archive/v2.2.6/RedisBloom-v2.2.6.zip
2、解压，进入解压目录，执行make命令（gcc错误的需要安装gcc）
```
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/5233693107454.png)

安装成功之后，会在安装目录出现一个 redisbloom.so文件：
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/2715764719157.png)

#### 布隆过滤器
以上安装好了布隆过滤器。

**加载redisbloom模块**

```shell

## 启动redis-server时加载模块并设置参数
# 布隆过滤器长度100W，容错万分之一，大小4M
redis-server /etc/redis/6379.conf --loadmodule /usr/apps/redis/RedisBloom-v2.2.6/redisbloom.so INITIAL_SIZE 1000000   ERROR_RATE 0.0001

## 重新连接cli
# 查看模块列表
127.0.0.1:6379> MODULE list
1) 1) "name"
   2) "bf"
   3) "ver"
   4) (integer) 20206

```

**在cli端加载module**

```shell
## 在连接的cli客户端使用module命令加载模块

# 查看模块列表 ：这里没有加载任何模块
127.0.0.1:6379> MODULE list
(empty list or set)

# 加载redisbloom：使用load操作符，后面跟着模块的绝对路径
127.0.0.1:6379> MODULE load /usr/apps/redis/RedisBloom-v2.2.6/redisbloom.so  INITIAL_SIZE 1000000   ERROR_RATE 0.0001
OK

# 再次查看模块列表
127.0.0.1:6379> MODULE list
1) 1) "name"
   2) "bf"
   3) "ver"
   4) (integer) 20206
```

也可以在启动redis-server的时候：  

```shell
redis-server --loadmodule /usr/apps/redis/RedisBloom-v2.2.6/redisbloom.so
```

**原理**
![](https://gitee.com/coderwing/blog-images/raw/master/数据库/redis5/redis详解.md/4190171534702.png)
```
如上图所示：当存储一个key时，将key做hash运算，得到的hash结果作为index对应到bit数组上，将对应的位置置为1；
再存入key时，计算hash，判断对应的位置是否为1，如果为1，则可能存在，如果为0，则肯定不存在；

概率性：由于hash计算存在一定的碰撞概率，不同的key计算的结果有可能是相同的，那么查询的时候就有可能返回1；
但是只要为0则肯定是不存在的。
```

**使用**

安装了bloom模块之后就可以使用bloom过滤器的功能了，提供了一组命令用来操作计划使用bloom过滤器的数据。

注意：*redis的存储/获取等操作，即便安装了bloom过滤器，默认是不适用的，需要使用模块提供的命令才有效。*


命令：
```
bf.add：添加元素到布隆过滤器中，类似于集合的sadd命令，不过bf.add命令只能一次添加一个元素，如果想一次添加多个元素，可以使用bf.madd命令。
bf.exists：判断某个元素是否在过滤器中，类似于集合的sismember命令，不过bf.exists命令只能一次查询一个元素，如果想一次查询多个元素，可以使用bf.mexists命令。
```

```shell
## 命令的命名空间为 【bf.】

# 添加k1
127.0.0.1:6379> BF.ADD k1 a
# 类型有问题
(error) WRONGTYPE Operation against a key holding the wrong kind of value
# 这里是string类型的，其实在这之前已经存在了k1，是通过set命令存储的
# 之前如果存在不是通过bf命令存储的key就会报错
127.0.0.1:6379> type k1
string

## flushall之后再操作
# 重新存储：成功
127.0.0.1:6379> BF.ADD id_filter 121
(integer) 1
127.0.0.1:6379> BF.ADD id_filter 132
(integer) 1

# 查看类型是：bf专有的类型
127.0.0.1:6379> type id_filter
MBbloom--

# 判断是否存在
127.0.0.1:6379> BF.EXISTS id_filter 121
(integer) 1
127.0.0.1:6379> BF.EXISTS id_filter 132
(integer) 1

## 存储和检查多个数据
127.0.0.1:6379> BF.MADD id_filter 156 180 191 290
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 1

# 多值检查，300不存在返回0
127.0.0.1:6379> BF.MEXISTS id_filter 156 300
1) (integer) 1
2) (integer) 0

```

**注意：** 这里的 id_filter 不单纯的是一个之前讲的redis中的一个key，这里可以理解成一个过滤器的分组名称，每一个分组中存储了不同的key，上面的121、132就是可以，比如是业务id当做key的情况。


**高级设置使用**
```
上面的例子中使用的布隆过滤器只是默认参数的布隆过滤器，它在我们第一次使用bf.add 命令时自动创建的。Redis还提供了自定义参数的布隆过滤器，想要尽量减少布隆过滤器的误判，就要设置合理的参数。

在使用bf.add 命令添加元素之前，使用bf.reserve命令创建一个【自定义的布隆过滤器】。
bf.reserve命令有三个参数，分别是：
    |-- key：键
    |-- error_rate：期望错误率，期望错误率越低，需要的空间就越大。
    |-- capacity：初始容量，当实际元素的数量超过这个初始化容量时，误判率上升。
```

高级使用：
```shell
# 创建一个自定义的bloom过滤器，灵活设置参数
127.0.0.1:6379> BF.RESERVE name_filter 0.0001 1000000
OK
```
这样就可以根据实际的业务场景需求设置不同的大小和容错率。

如果对应的key已经存在时，在执行bf.reserve命令就会报错。如果不使用bf.reserve命令创建，而是使用Redis自动创建的布隆过滤器，默认的error_rate是 0.01，capacity是 100。
布隆过滤器的error_rate越小，需要的存储空间就越大，对于不需要过于精确的场景，error_rate设置稍大一点也可以。布隆过滤器的capacity设置的过大，会浪费存储空间，设置的过小，就会影响准确率，所以在使用之前一定要尽可能地精确估计好元素数量，还需要加上一定的冗余空间以避免实际元素可能会意外高出设置值很多。总之，error_rate和 capacity都需要设置一个合适的数值。

**应用场景**

1、解决缓存穿透的问题

> 一般情况下，先查询缓存是否有该条数据，缓存中没有时，再查询数据库。当数据库也不存在该条数据时，每次查询都要访问数据库，这就是缓存穿透。缓存穿透带来的问题是，当有大量请求查询数据库不存在的数据时，就会给数据库带来压力，甚至会拖垮数据库。

> 可以使用布隆过滤器解决缓存穿透的问题，把已存在数据的key存在布隆过滤器中。当有新的请求时，先到布隆过滤器中查询是否存在，如果不存在该条数据直接返回；如果存在该条数据再查询缓存查询数据库。

2、黑名单校验

> 发现存在黑名单中的，就执行特定操作。比如：识别垃圾邮件，只要是邮箱在黑名单中的邮件，就识别为垃圾邮件。假设黑名单的数量是数以亿计的，存放起来就是非常耗费存储空间的，布隆过滤器则是一个较好的解决方案。把所有黑名单都放在布隆过滤器中，再收到邮件时，判断邮件地址是否在布隆过滤器中即可。
