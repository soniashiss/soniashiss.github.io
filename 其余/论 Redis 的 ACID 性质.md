# 论 Redis 的 ACID 性质

最近看《Redis 设计与实现》一书，讲到 Redis 的 ACID 性质，书中结论是 Redis 满足 ACI，D 看配置可能满足也可能不满足，**表示不敢苟同**。

论述之前，先说明以下所有定义均来自 Wikipedia 英文版，理由是：

1）最权威的定义应该是[ISO/IEC 10026-1:1992](https://www.iso.org/standard/27121.html),但需要花钱买，懒得花这个钱，英文版的 Wiki 在这种基础概念上应该不会有太大错误。
2) 书中作者参考的定义也是这个页面.

## 定义
在讨论为什么我不认为书中结论正确之前，先考虑一个问题，**我们为什么要讨论 ACID？**

个人浅见，讨论 ACID 的目的是，在我写一段逻辑的时候，如果我知道我使用的事务具有 ACID 性质，那么能够简化我的编程操作，我知道这么一个事务操作下去（即使中间有出错）数据依然是可靠的，就不用考虑特别复杂的各种出错怎么处理。这是 ACID 的意义所在。

而实际上，Wikipedia 对于 ACID 的描述，首先也写了这么一段话，大意和我上面的理解类似：

> In computer science, ACID (atomicity, consistency, isolation, durability) is a set of properties of database transactions intended to guarantee data validity despite errors, power failures, and other mishaps.

那么接下来讨论，我为什么认为书中结论不对。

### 1. Atomicity

定义：

> Transactions are often composed of multiple statements. Atomicity guarantees that each transaction is treated as a single "unit", which either succeeds completely or fails completely: if any of the statements constituting a transaction fails to complete, the entire transaction fails and the database is left unchanged. An atomic system must guarantee atomicity in each and every situation, including power failures, errors, and crashes.

粗略翻译一下：事务经常由多个操作组成。原子性保证每个事务可以被看成一个单独的“单元”：里面的操作要么全部成功，要么全部失败。也即，如果成功，需要全部成功；如果有失败，那么数据库应该保证没有被改变。原子性要求（声称自己具有原子特性的）系统在任何情况下都能保证上述特性，这个任何情况包括各种错误或者 crash。

那么我们参考定义来看看 Redis 的事务是不是具有原子性：

1）要么全部成功，要么全部不成功： 否。如果某个 Redis 执行出错，那么会继续执行下去，没有回滚。书中认为的原子性是要么全部执行要么全部不执行，我只能说这个定义理解不对。**当我们说”要么全部执行要么全部不执行“的时候，我们实际上说的是”要么全部执行成功要么全部执行不成功”**，这才是更准确的定义。

2）任何情况下都满足定义 ：否。一种情况上面说了；另一种情况是 crash。此情况下，假设执行到一半的时候 Redis 挂了，Redis 也没有回滚机制存在，在恢复后不会恢复数据到原来，因此不满足。

综合上述两点，Redis 不具备原子性。

### 2.Consistency

定义：

> Consistency ensures that a transaction can only bring the database from one consistent state to another, preserving database invariants: any data written to the database must be valid according to all defined rules, including constraints, cascades, triggers, and any combination thereof. This prevents database corruption by an illegal transaction. Referential integrity guarantees the primary key–foreign key relationship.

粗略翻译：一致性要求事务只能使得数据库从一个一致性状态迁移到另一个一致性状态：被写入数据库的数据必须合法，这个合法指数据满足所有预设的数据限制、触发器、级联回滚，以及这些限制造成的组合限制。

触发器、级联回滚Redis 没有，姑且不讨论。只讨论预设的数据限制这一条。

书中认为 Redis 满足一致性，理由是：

1)出错情况下，要么不执行，要么出错的语句不会对数据造成更改
2)服务器停机的情况下，如果没有持久化，那么数据库是空的所以满足一致性；如果有持久化并且恢复了，那么数据也还是一致的。

对于上面的理由，我只能说这纯粹是**偷换概念**。**上面的概念里的一致性，其实可以认为是数据物理上的完整性，而当我们讨论数据库数据的一致性时，我们说的是逻辑上的数据完整性**。理由在于，Wikipedia 在描述 Consistency 的定义时，点击 constraints 这个单词会跳到如下[页面](https://en.wikipedia.org/wiki/Data_integrity)，页面中清清楚楚明明白白写着：

> Data integrity also includes rules defining the relations a piece of data can have to other pieces of data, such as a Customer record being allowed to link to purchased Products, but not to unrelated data such as Corporate Assets. Data integrity often includes checks and correction for invalid data, based on a fixed schema or a predefined set of rules.

这段定义写着，我们需要将数据之间的关联关系纳入数据是否“正确”与“合法“的考虑。是，我承认 Redis 是非关系型数据库，没有约束关联关系正确性的机制，那**不等于我们就可以把数据一致性这个概念偷换成数据物理上的完整性**了。

如一开始所说，我们讨论 ACID，是为了简化编程模型，而在这里强行让 Redis 满足一致性，很可能最后我们记一个结论叫 "Redis AOF 持久化开了 always 的时候事务满足 ACID 性质"，然后记着这个结论去写一些带有数据关联关系的代码，呵呵，那乐子就大了。

个人结论，Redis 也不满足一致性。

### 3. Isolation

定义：

> Transactions are often executed concurrently (e.g., multiple transactions reading and writing to a table at the same time). Isolation ensures that concurrent execution of transactions leaves the database in the same state that would have been obtained if the transactions were executed sequentially. Isolation is the main goal of concurrency control; depending on the isolation level used, the effects of an incomplete transaction might not be visible to other transactions.

事务并发执行的结果和事务顺序执行后的结果任何时候一致。

Redis 单线程，满足，没什么可说的。

### 4. Durability

定义：

> Durability guarantees that once a transaction has been committed, it will remain committed even in the case of a system failure (e.g., power outage or crash). This usually means that completed transactions (or their effects) are recorded in non-volatile memory.

事务如果 commit 了，那么它的执行结果是持久化的。

书中论述 Redis 运行在 AOF 持久化模式下并且 appednfsync 选项值为 always 时具有持久性，**个人认为这个结论也是错的**。

理由很简单，AOF 持久化的原理是命令先执行再把命令记录下来，这里一定有一个时间差。假设命令执行后，命令记录还没来得及刷进 AOF 文件的时候 Redis 就挂了，那么这条命令的执行结果就没了。要保证持久性，这种操作必然不合适；所有需要保证持久性的系统，必然都是先记录再执行的，这点绕不过去。

# 结论

**Redis 除了满足 Isolation，剩下的 ACD 特性全部不满足**。

# 参考资料
1. [ACID](https://en.wikipedia.org/wiki/ACID), wikipedia
2. [Data integrity](https://en.wikipedia.org/wiki/Data_integrity)
3. 第 19 章 事务, 《Redis 设计与实现》, 黄健宏, 机械工业出版社, 2014-06-01