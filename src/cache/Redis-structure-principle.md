## redis版本变化

6.0 版本之后**支持多线程**

### 6.0版本之前都是单线程 ：

即使几个客户端并发 发送过来的命令，到后面还是排队，进行操作的

（添加图片）



## **对于多值 ：mset，mget**

对于这种方法，对于一个对象的json值，如果我们采用string的方式的话，首先得获取值，然后json话数据，然后修改值，然后塞进去。

如果采用mset的方式的话，直接一步操作就可以了。



## **setnx ：**

```
setnx qkcao 1
setnx qkcao 2
```

**对于setnx的值，如果第一次qkcao的值为1，已经成功了，那么第二次setnx的时候，只要是key为qkcao，就不会成功**。



## **hash的 ：hmset，hmget**

**使用Hash 对象缓存**

（图片）

**注意 ：**

**id 与 user对应的主属性的结合**

这样都在user的key下面进行管理，如果使用string的方式的话，会散落很多很多的key，

**如果10000个用户的话，就需要10000万个key，但是当使用hash的话，只需要一个key就可以了**。



**电商购物车**

1001用户的购物车 （key），然后把商品的id放入进去（field），后面再加上商品的数量

```
hset cart:1001 10088 1
```

**1001** ：用户id

**cart** : 1001 表明购物车id

**10088** : 商品id

**1** ：商品的数量

（图片）



**购物车中增加商品数量：**

```
hincrby cart:1001 10088 1
```

（图片）



**获取商品总数 （注意是计算整个购物车的数量，不是某个商品的数量，因为后面没有带有商品ID）**

```
hlen cart:1001
```



**删除商品：**

```
hdel cart:1001 10088
```



**获取购物车所有商品：**

```
hgetall cart:1001
```



**Hash结构的优缺点**

**优点 ：**

（1）同类数据归类整合处理，方便存储

（2）相比string，消耗的cpu与cpu相对较小，更节省空间

**缺点 ：**

redis集群环境下，不适合大规模使用



## **List结构**

**有序列表**，可以存储一些列表的数据，类似粉丝列表，文章的评论列表

也可以通过**lrange**命令，读取某个闭区间的元素，可以**基于list实现分页**。

**栈 ：FILO = LPUSH + LPOP （左边进，左边出）**

first in last out

（图片）



**阻塞队列 ：**

**BRPOP  key timeout ：**

**从key列表表头弹出一个元素，若列表中没有元素，阻塞等待timeout秒，如果timeout=0，一直阻塞等待。**

左边进去，

右边阻塞，一直监听元素，一旦有元素了，就监听到了，就开始读取



## Sets 

**去重**



## Sort Sets

**去重 排序**



## **Redis的单线程和高性能 ：**

- 因为它所有的数据都在内存中，所有的运算都是内存级别的运算，因为我们常规的内存操作都是在纳秒级别的，所以说redis的QPS可以达到10W/s,因为它的每个操作都是纳秒级别的。

- 单线程避免了多线程的切换，如果只有一个CPU，多个线程进行操作，一会处理线程1，一会处理线程2，来回切换，造成性能损耗问题。

- 正因为redis是单线程，所以要小心使用redis中非常耗时的命令，比如keys，因为使用非常耗时的命令就会造成阻塞，造成redis卡顿。

- key和value尽量也不要放太大，因为这样取数据的时间也会消耗非常多的时间。



## redis单线程如何处理那么多的并发客户端连接

- redis的IO多路复用：redis利用epoll来实现IO多路复用，（就是linux内核epoll函数实现的）将连接信息和事件放到队列中，依次放到文件事件分派器，事件分派器将事件分发给事件处理器。
- nginx也是采用IO多路复用解决C10K问题

- redis在客户端采用**多路复用技术**，redis只有一个线程处理事件，但是多个事件过来时，都会注册到**事件处理器**上去，多条事件过来都会进行**排队**的，然后将事件进行分别归类到**事件处理器**上，线程如何处理呢？一个是按照先来后到的顺序，另一个是**注册事件的监听**，**谁先响应**，就先处理谁。

（图片）



## **redis的配置文件讲解**

- **单位**

  （图片）

  配置文件 对单位大小写不敏感



- 包含

  （图片）

  就好比我们学习Spring的的Import标签，jsp的include标签，将其他内容导入进来



- 通用配置

  ```yml
  daemonize yes //以守护进程的方式运行，默认是no，我们需要自己开启为yes，这样后台就能运行，我们即使关闭客户端也能运行，一退出进程就结束了。
  
  pidfile /var/run/redis_6379.pid //如果指定后台运行的方式，我们需要指定一个进程pid文件
  ```



日志：

```
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)# notice (moderately verbose, what you want in production probably) 
//生产环境
# warning (only very important / critical messages are logged)
loglevel noticelogfile ""  
//日志的文件名databases 16   
//默认的数据库数量always-show-logo yes   
//运行时，是否显示redis的logo
```



- 快照

  持久化，在规定的时间内，执行了多少次操作，则会持久化到文件 .rdb.aof，redis是内存数据库，如果没有持久化，那么数据就会断电即失。

```
save 900 1 //如果在900秒内至少进行一次key的修改，我们及进行持久话操作
save 300 10//如果在300秒内至少进行10次key的修改，我们及进行持久话操作save 60 10000
//如果在60秒内至少进行10000次key的修改，我们及进行持久话操作  就是自动保存
stop-writes-on-bgsave-error yes 
//持久化失败了，是否继续工作rdbcompression yes  
//是否压缩持久化rdb文件，需要消耗CPU资源
rdbchecksum yes //保存rdb文件的时候，进行错误校验，是否校验，如果持久化出错了，是否进行纠正dir ./  
//持久化文件保存目录
```



- SECURITY 安全

  设置密码：

  我们可以在配置文件里面进行配置，但是我们更多还是在命令行里面配置

  （图片）



- CLIENTS客户端 显示

  （图片）



- MEMORY MANAGEMENT 内存设置

  （图片）

  策略：移除一些过期的key，或者报错

  （图片）



- APPEND ONLY MODE aof配置

  （图片）

  默认是不开启的，默认是使用rdb方式持久化，在大部分所有情况下，rdb完全够用。



每秒都会执行一次，进行同步，可能会丢失1s的数据

  ```
  appendfsync always //每次修改都会进行修改，每次都要写，消耗性能
  appendfsync no //不同步
  ```

  



