# Redis 设计与实现总结（未完成）

## 基本数据结构及应用

### 基本数据结构

**1.SDS**

由于 redis 是 C 写的，没有其余现代语言的 string 类型，因此自己实现了一个类似的。
底层是个数组，可以自动扩容

**2.list**

双向链表带了统计 len 的字段，还带了三个函数指针用于复制、释放以及比较节点的 value（用于支持多种 value 类型）。
![](http://redisbook.com/_images/graphviz-5f4d8b6177061ac52d0ae05ef357fceb52e9cb90.png)

**3.intset**

整数集合，底层是数组，可以保存 int16_t、int32_t、int64_t三种情况的整数，通过 encoding 来标识。
![](http://redisbook.com/_images/graphviz-acf7fe010d7b09c5d2500c72eb555863e67ad74f.png)

**其余TBC**

### 主要应用

这里的数据类型，其实指的是 redis set key 后面的 value 类型；redis 所有的 key 都是 string
对于 value 类型，每个 value 都是一个 redisObject:

```c
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 指向底层实现数据结构的指针
    void *ptr;

    // ...

} robj;
```

其中 type 就是指"string", "list", "hash"一类我们称为 redis 基本数据类型的内容；encoding 指 ziplist/dict 之类的后端实际实现。

| Redis数据类型 | 后端实现 | 常见命令 |
| -- | -- | -- |
| string | SDS | SET/GET/SETNX |
| list | list | LPUSH/LPOP/RPOP |
| hash | ziplist/dict | HSET/HGET |
| set | intset/dict | SCARD/SMEMBERS/SISMEMBER |
| sorted_set | ziplist/skiplist | ZCARD/ZRANGE |

## 单机数据库实现

此部分写在前面的个人见解：可以很明显看到实现上大体分为两层：1）基础数据结构 2）基础数据结构之上的分模块功能。两者相互影响。

### 数据库维护

**1.服务器全部数据库维护**

主要数据结构 redisServer,存 dbnum 以及对应的 db 的指针
![](https://img2020.cnblogs.com/blog/756647/202012/756647-20201225110414559-1859726984.png)

**2.单数据库结构**

单个数据库结构由 redisDb 维护
![](http://redisbook.com/_images/graphviz-e201646a2797e756d94777ea28a94cd2f7177806.png)

**3.过期处理**

redisDb 中的 expires字段(还是一个 dict), 存储某个 key 的过期时间(ms)
过期键的删除策略是：

1. 读时检查，如果过期，删除
2. 后台轮询，定时删除
3. 如果这个 redis server 是个 slave，则只等待 master 发送删除命令才删除，不执行上面的操作；若过期时有读操作，根据版本不同返回不一定一致（老版本有值，新版本没有）
4. RDB 持久化时，写入的 RDB 数据不包含过期键；读时分 master 还是 slave: master 不读过期键，slave 全部载入（个人理解等着 master 的命令删除)
5. AOF 持久化时，是否有记录只跟这个 key 是不是真的被执行删除有关；重写时会检查是否过期，已过期不写入

### 持久化

**1. RDB 持久化**

1. 分 save 和 bgsave 两种方式，save 阻塞主进程（此时拒绝服务），bgsave fork 主进程来持久化，不阻塞主进程
2. 不能同时存在两个持久化进程，包括 RDB 和 AOF
3. bgsave 可以做到后台自动保存，保存条件存储在 redisServer 的 saveparam 数组中，server 默认100ms 轮询一次检查是否满足保存条件（配套 dirty 计数和 lastsave 属性）
4.RDB 文件大结构如下所示，“REDIS”是个常量，用于表示这是个 RDB 文件； db_version 4 byte, 标识 RDB 文件版本号；databases中包含 DB 的具体内容；EOF 是结束符；check_sum 用于校验文件是否有损坏

| REDIS | db_version | databases | EOF | check_sum |
| -- | -- | -- | -- | -- |

**2. AOF 持久化**

1. 写原理：redis 每执行一条**写**命令，都将命令存入 AOF 缓冲区，并根据配置在每一个执行周期内来决定是否将 AOF 缓冲区内的内容刷进 AOF 文件里并进行同步
2. 恢复原理：创建一个不带网络连接的伪客户端，根据 AOF 文件将所有命令执行一遍
3. AOF rewrite 原理：根据服务器当前内存状态，将数据用一条命令代替之前的 N 条，相当于只写终态的命令。好处是可以减小 AOF 文件大小。
    * 有阻塞和非阻塞两种；阻塞情况服务器不响应客户端请求，非阻塞情况通过 fork 主进程的子进程来执行 AOF rewrite  
    * 非阻塞的情况下：1）子进程有主进程的内存备份，可以写；2）子进程执行期间主进程执行的写命令会被写入 AOF 缓冲区和 AOF rewrite 缓冲区，子进程 rewrite 完毕结束前，会将 AOF rewrite 缓冲区内的内容 append 进文件，然后替换原来的 AOF 文件。

### 事件处理

Reids 服务器是事件驱动的，事件主要分为两类：文件事件和时间事件。

**1.文件事件**

1. Reactor 模式，单线程（也就是说，套接字 ready 后产生的事件都先入队，生产/消费者模型且两端都为 1），使用 I/O 多路复用来监听多个套接字，多路复用的底层实现可选。
![文件事件处理器结构](http://redisbook.com/_images/graphviz-f0d024ca2782cbbe20e2cd1e52540d92f64b3a37.png)

![底层可选](http://redisbook.com/_images/graphviz-840bfb6ea3cc590829fecd9b9062002d59dbf673.png)

2. 文件事件分派后的处理程序（也即处理器），主要分为几类：
1）为了应答连接，需要连接应答处理器
2）为了响应命令，需要命令处理以及命令应答处理器
3）为了主从复制，需要复制处理器

**2.时间事件**

1. 时间事件分为两类：1）定时事件 2）周期性事件；存在一个时间事件处理器
2. 两者都通过周期性循环检查来实现
3. 底层数据结构是**无序列表**， 列表中每一项记录事件需要执行的时间（精度 ms）以及 handler
4. 时间事件处理器执行时，检查整个链表并进行操作；定时事件和周期性事件在这里的区别是，如果是定时事件执行完之后则将其从链表中删除；如果是周期事件则不删除二是更新其时间值。

**3.事件调度（文件事件和时间事件两者怎么配合）**

简单来说，一个 while 大循环
1）先找到最近的事件事件还有多久到达；如果还没到达则阻塞等待时间事件，如果已经到达则不用等
2）先处理所有的文件事件
3）再处理所有的时间事件。

从上述过程可见，就是**以时间事件的时间为触发器，轮询处理所有的事件，并且文件事件在前，时间事件在后**

### 客户端

数据结构

```C
struct redisServer {
//...
// all redis clients' list
list *clients;
redisClient *lua_client; //特殊 client，执行 lua 脚本的伪客户端，和 sever 的生命周期相同
//....
}
```

```C
typedef struct redisClient {
//...
int fd; //套接字描述符
robj *name; //name, 可有可无
int flags; //各种 flag
//个人理解以下几个字段和 input 有关
sds querybuf; //输入缓冲区，动态扩缩容，最大不超过 1GB，否则会被服务器关闭
int argc;
robj **argv;
struct redisCommond *cmd; //根据 argv[0]里的 cmd 指向 redis 中的 command 的 dict(key: command, value: command function ptr)
//和 output 有关
char buf[REDIS_REPLY_CHUNK_BYTES]; //固定大小缓冲区，用于保存长度较小的回复，默认 16 KB
int bufpos; //和上一字段联合使用，用于记录 buf 中已经使用的缓冲区大小
list *reply; //可变长度缓冲区

int authenticated;

time_t ctime; //create time
time_t lastinteraction; //last interaction time with server
time_t obuf_soft_limit_reached_time; //防止 client 的输出缓冲区过大占用过多资源，服务端会检查缓冲区大小，分硬性和软性限制。超过硬性限制直接关闭，超过软性限制则记录超过的起始时间（也就是本字段），如果持续超过则关闭。
}
```

### 服务器

#### 1.命令处理过程
简单来说：
1）读取 redisClient 缓冲区内的命令（见[客户端](#客户端)部分）
2）解析，比如将用户发送的命令解析指向 RedisCommand 表 （此表可以理解成各种命令的实现函数的表，是个 dict）
3）前置检查，比如身份验证，内存回收检查服务器是否处于可以响应的状态等等等
4）执行，将执行结果保存到 redisClient 的输出缓冲区
5）后置操作，如记录 log，AOF 持久化操作，被复制时将命令传播给其他服务器等。

#### 2. serverCron

serverCron 默认 100ms 执行一次
干的事情：
1）更新 redisServer 的时间缓存（精度不高的缓存，100 ms 一次，只在执行非高精度操作时使用）
2）更新 LRU 时钟(lruclock字段，10s 一次)，用于和 redisObj 的空转时间记录比较后得出空转时间
3）各种统计数据的更新，如服务器每秒执行多少次命令、内存峰值等
4）处理 sigterm 信号
5）管理客户端资源（比如释放掉连接超时的客户端）
6）管理数据库资源（比如删除过期键，对 dict 进行收缩等操作）
7）关闭输出缓冲区超限的客户端
8）执行被延迟的 BGREWRITEAOF 命令
9）检查持久化操作的运行状态
10）将 AOF 缓冲区的内容写入 AOF 文件
11）增加 cronloops 的计数值（此字段用于实现 serverCron 每执行多少次就执行一次 xxx 的功能）

1-3关于服务器状态信息；5-7 是管理 client 和 db；8-10 和持久化相关

#### 3.初始化过程
1）创建 redisServer 的数据结构实例（设置 pid，以及某些必须数据均采用默认值），初始化 LRU 时钟，创建命令表（即上文提到的 redisCommand）
2）载入配置文件，设置对应的值
3）创建其余数据结构，比如 server.Clients 链表、server.db 数组、lua 执行环境等
4）还原数据库状态，即从 RDB 文件或者 AOF 文件载入数据
5）start [main loop](#事件处理)


## 主要参考资料

1. 《Redis 设计与实现》, 黄健宏, 机械工业出版社, 2014-06-01
2. Redis 设计与实现, http://redisbook.com/
3. Redis 源码解析1: 数据库 RedisDb, https://www.cnblogs.com/chenchuxin/p/14187898.html

#CS/Redis