## **缓存做持久化：**

是为了防止缓存挂了，导致缓存数据丢失



## **RDB快照 ：**

https://blog.csdn.net/u022812849/article/details/108409469

Redis是内存数据库，如果不将内存中的数据库状态保存到磁盘，那么一旦服务器进程退出，服务器中的数据库状态也会消失，所以Redis提供了持久化功能。

rdb是在**指定时间间隔内，对全量数据**进行快照存储。

由于rdb是对**全量数据**进行备份，所以这种操作不可能很频繁，不然会影响IO性能，

所以RDB会有一个时间间隔，假如这个时间间隔为5分钟，8点整进行一次rdb，在8点05分刚要进行rdb的时候，redis意外停止工作（例如停电），这样就会导致**8点到8点五分这个时间段得数据会丢失**。如果能够容忍这段数据的丢失，就可以使用rdb，一般来说，redis用来缓存，数据丢失了没关系，可以从数据库中再次加载到redis中，但是如果**把redis当成数据库**的话，那么就**不适合**使用rdb这种持久化方式了。



- RDB(redis DataBase)

  （1）在指定的时间间隔将内存中的数据集体快照写入磁盘，也就是行话讲的snapshot**快照**，它会**恢复时**是将**快照文件直接读到内存里**。

  **快照**：

  记录某一时刻的东西，rdb快照就是记录某一个瞬间的内存数据，，记录的是实际数据



（2）redis会单独创建一个子进程进行持久化，当数据集比较大的时候，**fork的过程是非常耗时的**，可能会**导致redis在一些毫秒级内不能响应客户端的请求**。如果数据集比巨大并且cpu性能不是很好的情况下，这种情况会持续1秒，**AOF也需要fork**，但是你可以**调节重写日志文件的频率**来调节。会先将数据写到一个**临时文件**中，待持久化过程都结束了，再用这个临时文件**替换和删除**上次持久化好的文件。整个过程，**主进程不进行任何IO操作**的，这就确保了极高的性能。如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式比AOF方式更加高效，RDB的缺点就是最后一次持久化后的数据可能会丢失。



## **fork的原理**

**rdb时需要考虑两个问题**：

（1）假如8点开始，整个过程需要持续5分钟，那么在这5分钟有些key被修改了，有些key被删除了，那么这些key会不会被存储到rdb文件中去呢？

（2）如果有用另外一个进程来负责持久化，那么是不是需要将整个redis进程的内存拷贝一份，时间成本会不会很大，内存空间是不是够用？



**fork是linux提供的一个系统调用：**

```c
#include <unistd.h>
pid_t fork(void);
```

pid_t是进程描述，实质就是一个int，如果fork函数调用失败，返回一个负数-1，调用成功的话，返回两个值，一个是0，另外一个是子进程ID。

```java
fork系统调用之后，父进程和子进程一般会交替执行，并且它们处于不同空间中。
fork()的子进程并不是从头开始，因为在fork()之前，父进程已经为子进程搭建好了运行环境了，所以直接从有效代码处开始。

fork底层实现采用了写时拷贝（COW，copy_on_write）技术，父进程和子进程共享页帧而不是复制页帧。
然而，只要页帧被共享，它们就不能被修改，即页帧被保护。
无论父进程还是子进程何时试图写一个共享的页帧，就产生一个异常，这时内核就把这个页复制到一个新的页帧中并标记为可写。
原来的页帧仍然是写保护的：当其他进程试图写入时，内核检查写进程是否是这个页帧的唯一属主，
如果是，就把这个页帧标记为对这个进程是可写的。
```



(图片)

我们默认是RDB，一般不需要进行修改这个配置



有时候我们在生产环境中要保存这个文件，备份这个文件，rdb保存的文件是dump.rdb  都是在快照中进行配置的

（图片）



**要生成具体的哪一个位置是在 ：**

（图片）



**dir ：**表示rdb文件保存的位置

**rdb-del-sync-files : **在**没有持久化**的情况下**删除**复制中使用的rdb文件，通常情况下默认即可。



## **配置文件中与rdb相关的配置** ：

```yml
# 60秒内有至少有1000个键被改动时，自动触发RDB
# 300秒内有至少有10个键被改动时，自动触发RDB
# 900秒内有至少有1个键被改动时，自动触发RDB
save 900 1
save 300 10
save 60 10000

# 异步重写失败了，会停止写入
stop-writes-on-bgsave-error yes

# 开启rdb文件压缩
rdbcompression yes

# 加载rdb和保存rdb时文件进行校验
rdbchecksum yes

# rdb存储的文件名，保存路径在下面
dbfilename dump.rdb

# 在没有持久化的情况下删除复制中使用的RDB文件，通常情况下保持默认即可。
rdb-del-sync-files no

# RBD文件保存的目录
dir /usr/local/redis/data/6379
```

（1）开启rdb文件**压缩**

（2）加载rdb和保存rdb时文件进行**校验**

（3）**rdb**存储的**文件名**，保存在路径下面

（4）在没有持久化的情况下**删除**复制中使用的rdb文件，通常情况下保存默认即可。

（5）rdb文件**保存**的目录

（6）异步重写失败了，会**停止**写入

redis默认开启rdb，除了配置文件这配置的自动保存规则外，也可以通过调用 **同步保存 或者 异步保存**

(图片)



只要触发rdb操作，就会进行快照备份。

但是如果在60s内进行4次操作，而且redis挂掉了，这时数据就会丢失了。所以这种方式在工作使用的比较少。

**触发机制：**

(1)save的规则满足的情况下，会自动触发rdb规则

(2)执行flushall,也会触发我们的rdb规则

(3)退出redis，也会产生rdb文件

备份就会自动产生一个dump.rdb文件

（图片）



## **如何恢复rdb文件**

（1）只需要**将rdb文件放在我们redis启动目录就可以**了，redis启动的时候会自动检查dump.rdb文件，恢复其中的数据

（2）查看需要存在的位置

（图片）



## **save与bgsave**

（图片）



## **AOF持久化**

redis可以在AOF文件体积变得过大时，重新后的新AOF文件包含了恢复当前数据集所需的最小命令集合。

整个重写得操作是**绝对安全的**，及时宕机，**AOF也不会丢失**，而一旦新的AOF文件创建完毕，redis就会从旧的AOF文件**切换到新的AOF文件**中，并开始对新的AOF文件进行追加操作。

AOF文件有序的保存了对数据库执行的所有写入操作，这些写入操作以redis协议的格式保存，因此AOF文件的内容非常容易被人读懂，对文件进行分析也很轻松，导出AOF文件也非常简单，如果你不小心执行了flushall命令，但**只要AOF文件未被重写，那么只要停止服务器，移除AOF文件的末尾的flushall命令，并重启redis，就可以将数据集恢复到flushall执行之前的状态**。

```yml
# AOF默认关闭
appendonly no

# The name of the append only file (default: "appendonly.aof")
# AOF保存的文件名
appendfilename "appendonly.aof"

# 同步的模式
# always：每收到一个写请求就同步到文件中，这样就不会丢失数据，但是性能低（每次都有磁盘IO）。
# everysec：每秒同步一次，最多丢失一秒的数据，性能介于always和no之间。
# no：no并不是不同步，而是redis只把数据写到操作系统的缓冲区中，具体什么时候写到磁盘依赖操作系统，linux下缓存区大小为512k，缓冲区买了才会刷新到磁盘，性能最好。

# appendfsync always
appendfsync everysec
# appendfsync no

# AOF重写期间不进行同步，避免大量的磁盘IO，影响性能
no-appendfsync-on-rewrite no

# 配置重写触发机制，当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 如果该配置启用，在加载时发现aof尾部不正确时会向客户端写入一个log，但是会继续执行，如果设置为no，发现错误就会停止，必须修复后才能重新加载。
aof-load-truncated yes

# AOF文件的内容前面是否使用RDB
aof-use-rdb-preamble yes
```

AOF**默认关闭**

AOF保存的**文件名称**

**同步模式**：always，everysync,no

AOF**重写期间不进行同步**，避免大量的磁盘IO，影响性能

配置重写**触发机制**

\# **配置重写触发机制**，当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发

auto-aof-rewrite-percentage 100

auto-aof-rewrite-min-size 64mb



**AOF重写的目的 ：**就是解决AOF文件体积膨胀的问题



AOF文件内容**是否使用rdb**:

（1）**老版本**的AOF，**重写操作只会删除过期的key**，**合并相同的key**，最终生成的文件还是一个纯指令的日志文件

（2）**新版本**的AOF，重写操作就会将**老的数据以rdb的格式存储到aof文件中的前面**，后面将**增量的数据**以redis协议的格式append到**文件末尾**。此时AOF文件是一个**混合体**。



**AOF中如果存在自增或者其他比较多的操作时**:

如果发生宕机的话，进行恢复的时候，指令很多，就会很慢，这时AOF提供了一个方法 ： **重写**

意思就是说将**上面多条命令换做 执行一条命令**



**AOF也可以手动进行触发 （触发重写）**:

**bgrewriteaof**



**那么手动触发会不会遇到阻塞？**

**一边手动触发，一边进行set，get操作**

(图片)



每当redis执行一个改变数据集的命令时，这个命令就会被追加到AOF文件的末尾。这样redis重启时，程序就可以通过重新执行AOF中的命令达到重建数据的目的。

（图片）

默认不开启的，需要我们手动去配置

（图片）

意思是每秒都会执行一次持久化的操作。在出现故障时，只会丢失一秒钟的数据。和RDB差不多。

always：只是保存修改数据的命令，只会保存set，不会保存get命令

重启redis即可生效



如果rof文件有错误，redis是启动不起来的，这时我们需要进行修复这个aof文件，这时redis提供一个工具进行修复，**redis-check-aof**

（图片）



优点：**每次修改都同步**，从不同步，效率最高

缺点：**aof是一个IO操作**，运行效率要比rdb慢

**有一个问题就是**：日志文件会越来越大



## **RDB和AOF的选用问题 ：**

如果同时开启的话，会优先选用AOF，因为AOF更完整。



## **redis 4.0 提供了混合持久化 ：**

使用rdb会丢失很多数据，但是使用AOF的话，redis启动就会特别慢，

```yml
aof-use-rdb-preamble no
```

如果将no改为**yes**，就会**开启混合持久化**，**AOF在重写时**，不再是单纯的将内存数据转换为RESP命令写入AOF文件，而是将**重写 这一刻之前** 的内存做**RDB快照处理（就是将内存中的所有重写到AOF中）**，并且将RDB内容和增量的AOF修改内存数据的命令放在一起，都写入新的AOF文件，等到重新完新的AOF才会进行改名，新的文件一开始不叫appendonly.aof,完成新旧aof的交替。

于是在**redis重启的时候，可以先加载rdb的内容，然后再重放增量AOF日志就可以完全替代之前的全量的AOF文件**。



**混合持久化AOF的结构 ：**

(图片)



**比如下面的操作 ：**

比如现在**oppendonly.aof** 文件如下内容：

（图片）



**然后执行** **bgrewriteaof** **命令：**

然后再查看appendonly.aof文件

（图片）

上面的内容类似乱码的格式，就是rdb的格式，二进制的压缩数据，就是当前内存的数据。

上面就是将rdb放在aof当内存去存储



**现在又有redis命令来执行了，查看一下appendonly.aof的变化** **：**

（图片）

看到上面的变化，

如果现在**再重写**（**bgrewriteaof**）的话，**那么下面新写入的数据还会变成rdb格式的**。

**所以这种混合持久化方式的话，速度比之前快的多 ：**

比如回头恢复appendonly.aof 文件，**前面1G的rdb数据直接放到内存中，后面的aof命令直接重新执行一下就可以**了。



## **redis的主从架构**

**主从复制的原理 ：**

从redis 2.8 开始支持部分复制的命令psync去master同步数据

如果你为master配置了一个slave，那么不管这个slave是否是第一次连接上master，它都会向master发送一个sync的命令，请求复制数据（redis 2.8 版本之前的命令）。



### **全量复制 ：**

（图片）

全量复制 和 部分复制

第一次肯定是全量复制

（1）从节点发起一个同步命令，同步数据

（2）主节点接到这个命令之后，会快速的bgsave生成一个最新的rdb快照数据

（全量同步就是同步主节点的rdb快照数据的）

（3）主节点马上将rdb数据发送给从节点

bgsave的过程中也有可能会有新的数据，这时发送的是rdb是不是最新的数据？

其实master主节点会维护一个**最近数据的缓存**（repl_back_buffer）,意思就是说master会将最新的命令写到这个缓存中去，**当上面发送rdb数据完之后，就会将最近数据的缓存发送给从节点（也就是上面的第4步）**。

（4）发送最新数据的缓存

（5）**slave收到master发送的rdb之后，如果salve本地有一个rdb数据的话，有可能是别的master发过来的，这时salve就会将这个rdb删除**，然后将上面的 **rdb** 和 **最新数据的缓存** 进行合并，然后**重新加载到slave内存**中去。



**不管是开启rbd还是开启的aof，与这个没关系 ：**

因为上面的执行的命令是**bgsave**，**这个命令就会就会生成rdb文件，并且将已经存在的rdb文件覆盖掉**。



### **增量同步 ：**

（图片）

（1）salve与master的连接已经断开了

（2）当slave与master再次重新连接

（3）重新连接之后，就会执行psync（offset）命令

内存的每一条数据都有一个偏移量，同步数据的时候，就会将偏移量发送给master

（4）master首先偏移量offset之后，它就会判断，判断这个offset

（**值越大，偏移量越大，比如master上面有最大偏移量有2000，当同步1500的时候，突然断开了，当重新连接的时候，从节点就会将这个偏移量为1500的发送给主节点，主节点master就会去比对，比较一下从节点发过来的偏移量是否在 自己的偏移量之间，如果是的话，就会将1501到2000的数据直接同步给从节点，从节点拿到数据之后，加载到内存中去。**

**但是当主节点的缓存区比价小的话，从节点的偏移量比较小，就是上次同步的数据比较少，还剩下很多数据没有同步，从节点发送的偏移量很小，小到 小于主节点的偏移量范围，这时就会触发一次全量的同步**）



### **演示一下** ：

**复制配置文件（cp）之后 ：**

(1)首先更改端口

```
port 6380
```

(2)更改进程文件id

```
pidfile /var/run/redis_6380.pid
```

(3)更改日志文件

```
logfile "6380.log"
```

(4) 目录更改一下

```
dir 6380./
```

(5) 配置主从复制

```yml
replicaof 192.168.0.60 6379 // 从6379的机器上复制数据 replica-read-only yes // 只读不写
```

(6)启动从节点

```
redis-server redis.conf
```

(7) **连接从节点**

```
redis-cli -p 6380
```

注意 ： 在从节点上只能读数据，不能写的

查看数据 ：**keys \***

(图片)



(8)在6379上写数据，在6380上查看是否同步数据





## **redis哨兵架构**

（图片）

**redis 客户端连接的是哨兵集群**

它会监听每一个节点

**第一次连接**的时候 **可能是通过哨兵找到的主节点信息，返回给客户端**，

**下次连接的时候，直接通过节点信息进行连接主节点，但是不会每次都是通过sentinel代理访问主节点的，**

**当主节点发生变化的时候，哨兵会第一个感知到，并且将新的主节点信息返回给客户端（redis客户端实现了订阅功能，订阅了sentinel发布的节点变动的消息）**



### **高可用 ：**

主节点挂了之后，哨兵就会从节点中选出一个新的节点作为主节点



### **哨兵模式搭建 ：**

使用的是redis中的 **sentinel.conf** 配置文件

（1）端口

（2）daemonize yes 后台启动方式

（3）pidfile 进程id

（4）日志文件地址 logfile

（5）dir **日志文件生成的目录**

（6）sentinel monitor mymaster 192.168.0.60 6379 2

**指的是主节点的 ip + port, 2 表示当有多少sentinel 认为 master失效了**，master才算是正在的失效。

值一般为 ： sentinel总数/2 + 1

（7）启动哨兵

```
src/redis-sentinel sentinel-26379.conf
```

**启动客户端的话 ：**

```
src/redis-cli -p 26379
```

客户端启动之后，使用 **info命令** 可以查看哨兵集群的信息



