# Redis 设计与实现总结

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
其中 type 就是指"string", "list", "hash"一类我们成为 redis 基本数据类型的内容；encoding 指 ziplist/dict 之类的后端实际实现。

| Redis数据类型 | 后端实现 | 常见命令 |
| -- | -- | -- |
| string | SDS | SET/GET/SETNX |
| list | list | LPUSH/LPOP/RPOP |
| hash | ziplist/dict | HSET/HGET |
| set | intset/dict | SCARD/SMEMBERS/SISMEMBER |
| sorted_set | ziplist/skiplist | ZCARD/ZRANGE |

## 数据库实现
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
**4. RDB持久化**

1. 分 save 和 bgsave 两种方式，save 阻塞主进程（此时拒绝服务），bgsave fork 主进程来持久化，不阻塞主进程
2. 不能同时存在两个持久化进程，包括 RDB 和 AOF
3. bgsave 可以做到后台自动保存，保存条件存储在 redisServer 的 saveparam 数组中，server 100ms 轮询一次检查是否满足保存条件

## 主要参考资料
1. 《Redis 设计与实现》, 黄健宏, 机械工业出版社, 2014-06-01
2. Redis 设计与实现, http://redisbook.com/
3. Redis 源码解析1: 数据库 RedisDb, https://www.cnblogs.com/chenchuxin/p/14187898.html