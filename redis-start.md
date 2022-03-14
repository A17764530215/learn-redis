## 参考链接

https://pdai.tech/md/db/nosql-redis/db-redis-introduce.html

https://www.bilibili.com/video/BV1GY41187d5?spm_id_from=333.999.0.0

## 什么是Redis

Redis是一种支持key-value等多种数据结构的存储系统。可用于缓存，事件发布或订阅，高速队列等场景。支持网络，提供字符串，哈希，列表，队列，集合结构直接存取，基于内存，可持久化。



使用场景：

1. `缓存`
   * 读取前，先去读Redis，如果没有数据，读取数据库，将数据拉入Redis。
   * 插入数据时，同时写入Redis。
   
2. `限时业务`

   redis中可以使用`expire命令`设置一个<u>键的生存时间</u>，到时间后redis会删除它。利用这一特性可以运用在限时的优惠活动信息、手机验证码等业务场景

3. `计数器相关问题`

   redis由于`incrby命令`可以实现<u>原子性的递增</u>，所以可以运用于高并发的秒杀活动、分布式序列号的生成、具体业务还体现在比如限制一个手机号发多少条短信、一个接口一分钟限制多少请求、一个接口一天限制调用多少次等等

4. `分布式锁`

   这个主要利用redis的`setnx命令`进行，setnx："`set if not exists`"就是如果不存在则成功设置缓存同时返回1，否则返回0 ，这个特性在很多后台中都有所运用，因为我们服务器是集群的，定时任务可能在两台机器上都会运行，所以在定时任务中首先 通过setnx设置一个lock， 如果成功设置则执行，如果没有成功设置，则表明该定时任务已执行。 当然结合具体业务，我们可以给这个lock加一个过期时间，比如说30分钟执行一次的定时任务，那么这个过期时间设置为小于30分钟的一个时间就可以，这个与定时任务的周期以及定时任务执行消耗时间相关。

   在分布式锁的场景中，主要用在比如秒杀系统等

5. `延时系统`

   比如在订单生产后我们占用了库存，10分钟后去检验用户是否真正购买，如果没有购买将该单据设置无效，同时还原库存。 由于redis自2.8.0之后版本提供`Keyspace Notifications`功能，允许客户订阅Pub/Sub频道，以便以某种方式接收影响Redis数据集的事件。 所以我们对于上面的需求就可以用以下解决方案，`我们在订单生产时，设置一个key，同时设置10分钟后过期， 我们在后台实现一个监听器，监听key的实效，监听到key失效时将后续逻辑加上`。

   当然我们也可以利用rabbitmq、activemq等消息中间件的延迟队列服务实现该需求。

6. `排行榜相关问题`

   关系型数据库在排行榜方面查询速度普遍偏慢，所以可以借助redis的SortedSet进行热点数据的排序。

   比如点赞排行榜，做一个SortedSet, 然后以用户的openid作为上面的username, 以用户的点赞数作为上面的score, 然后针对每个用户做一个hash, 通过zrangebyscore就可以按照点赞数获取排行榜，然后再根据username获取用户的hash信息，这个当时在实际运用中性能体验也蛮不错的。

7. `点赞、好友等相互关系的存储`

   Redis 利用集合的一些命令，比如求交集、并集、差集等。

   在微博应用中，每个用户关注的人存在一个集合中，就很容易实现求两个人的共同好友功能。

8. `简单队列`

   由于Redis有list push和list pop这样的命令，所以能够很方便的执行队列操作



## 安装redis

```bash
sudo apt update
sudo apt install redis-server
sudo service redis-server start 
redis-cli
```

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315004722528.png" alt="image-20220315004722528" style="zoom:50%;" />

我们输入一个ping，会返回一个PONG，就表示安装成功了

## 基本操作

### 数据库操作

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315005326464.png" alt="image-20220315005326464" style="zoom: 33%;" />

我们用select进行选择数据库：

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315005211124.png" alt="image-20220315005211124" style="zoom: 50%;" />

我们用dbsize来查看当前key-value数量：

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315005505731.png" alt="image-20220315005505731" style="zoom:50%;" />

flushdb清空当前数据库，flushall清空所有数据库：

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315005554676.png" alt="image-20220315005554676" style="zoom:50%;" />

用save将db保存到磁盘,用bgsave来后台保存到磁盘，用lastsave查看最后一次保存的unix时间

### 通用数据操作

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315010022558.png" alt="image-20220315010022558" style="zoom: 33%;" />

keys + 通配符:返回符合通配符的所有的key

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315010236798.png" alt="image-20220315010236798" style="zoom:50%;" />

```bash
# exists + 若干个key的名字：返回存在这些key的个数
127.0.0.1:6379> exists k1 k2 a3
(integer) 2

# type + key的名字:查看该key对应的value的类型
127.0.0.1:6379> type k1
string
127.0.0.1:6379> type kkk
none

# del + 若干个key的名字:删除键值对
127.0.0.1:6379> del k1 k2
(integer) 2    
127.0.0.1:6379> keys *
1) "a2"
2) "a1"

# rename key1 key2:将key1重命名为key2. 注意:如果key2本来就存在,会覆盖掉key2的value
127.0.0.1:6379> rename a1 a2
OK
127.0.0.1:6379> keys *
1) "a2"
127.0.0.1:6379> get a2
"av1"

# renamenx key1 key2 [nx表示not exist]:只有当key2不存在时才会去重命名
mset k1 v1 k2 v2 a1 av1 a2 av2 # 我们先重新插入数据,来试验
127.0.0.1:6379> renamenx k1 k2
(integer) 0     
127.0.0.1:6379> get k1
"v1"
127.0.0.1:6379> renamenx k1 newk1
(integer) 1     
127.0.0.1:6379> get newk1
"v1"

# move + key的名字 + db编号：将这个键值对移到这个数据库
127.0.0.1:6379> move k2 3	# k2 移到3号数据库
(integer) 1     
127.0.0.1:6379> keys *
1) "a2"
2) "newk1"      
3) "a1"
127.0.0.1:6379> select  3
OK
127.0.0.1:6379[3]> keys *
1) "k2"

# copy key1 key2:将key1的value赋值给key2的value  [这个是6.2版本后可用的] 
```



## 五种基本数据类型

https://redis.io/topics/data-types 这是官网对这里的介绍，可供参考

首先对redis来说，所有的key（键）都是字符串。我们在谈基础数据结构时，讨论的是`存储VALUE的数据类型`，主要包括常见的5种数据类型，分别是：`String、List、Set、Zset、Hash`

![img](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/db-redis-ds-1.jpeg)

![image-20220315014358621](https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315014358621.png)

### 字符串String

String是redis中最基本的数据类型，是二进制安全的，这意味着redis的string可以包含任何数据。如数字，字符串，jpg图片或者序列化的对象

下面是一些常用命令：

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315014558315.png" alt="image-20220315014558315" style="zoom:50%;" />

<img src="https://gitee.com/xu-kaijian/pic-bed/raw/master/img/image-20220315014823865.png" alt="image-20220315014823865" style="zoom:50%;" />

实战场景：

1. **缓存**： 经典使用场景，把常用信息，字符串，图片或者视频等信息放到redis中，redis作为缓存层，mysql做持久化层，降低mysql的读写压力。
2. **计数器**：redis是单线程模型，一个命令执行完才会执行下一个，同时数据可以一步落地到其他的数据源。
3. **session**：常见方案spring session + redis实现session共享

