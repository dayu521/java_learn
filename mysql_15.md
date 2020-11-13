### 15.7.1 InnoDB Locking 

这部分描述了InnoDB使用的锁的类型 .
• Shared and Exclusive Locks
• Intention Locks
• Record Locks
• Gap Locks
• Next-Key Locks
• Insert Intention Locks
• AUTO-INC Locks
• Predicate Locks for Spatial Indexes

#### Shared and Exclusive Locks(共享和排他锁)

InnoDB实现了标准的行级加锁(row-level)，有两种锁类型，共享锁( shared (S) locks)和排他锁(exclusive (X) locks).

- 共享锁允许持有此锁的事务去读取一行.
- 排他锁允许持有此锁的事务去更新或删除一行.

如果事务T1在行r上持有共享锁S，那么对于一些不同的事务T2来说，请求行r上的锁需要按照下面进行处理:

- T2请求如果是共享锁，那么可以被立即授予.因此T1和T2都持有r上的一把锁.
- T2如果请求排他锁，那么立即否决.

如果T1持有r上的排他锁(X)，来自其他不同事务请求任何锁类型都立即拒绝.相应地，事务T2只能等待事务T1释放行r上的锁.

#### Intention Locks(意向锁)

InnoDB支持多种粒度的加锁，这允许行锁和表锁可以共存.例如，语句`LOCK TABLES ... WRITE`也占用某个特定表上的排他锁.为了让在多个粒度级别的加锁可行，InnoDB使用Intention Locks.意向锁是表级锁(table-level)，它表明一个事务在一个表中的某行中，稍后需要哪种类型的锁(共享锁或排他锁).有两种类型的意向锁:

- intention shared lock (IS)表明一个事物打算在某个表中的一些个别行上设置一个共享锁.
- intention exclusive lock (IX)类似，表明打算设置排他锁.

例如，, SELECT ... FOR SHARE 设置 IS锁, 而 SELECT ... FOR UPDATE设置 IX锁.

这种将要打算加锁的协议如下所示:

- 在一个事务能够获得表中一行上的共享锁之前，它必须先获取IS锁或更强的IX锁.
- 在一个事务能够获得表中一行上的排他锁之前，他必须获得IX锁.

表级锁类型的兼容性总结如下

|      | X    | IX   | S    | IS   |
| ---- | ---- | ---- | ---- | ---- |
| X    | 冲突 | 冲突 | 冲突 | 冲突 |
| IX   | 冲突 | 兼容 | 冲突 | 兼容 |
| S    | 冲突 | 冲突 | 兼容 | 兼容 |
| IS   | 冲突 | 兼容 | 兼容 | 兼容 |

如果一个事务请求与已存的锁兼容的锁，那么可以被授予.但如果冲突就不行.事务只能等到相冲突的锁被释放才继续.如果锁请求与已存锁冲突，且是由于会造成死锁而不能授予，那么会发生错误.

意向锁什么也不阻塞，除了全表请求(例如,LOCK TABLES ... WRITE).意向锁的主要目的是展示某人正在锁住一行，或打算锁住表中的一行.

意向锁的事务数据产生类似于 SHOW ENGINE INNODB STATUS和InnoDB monitor中的输出:

```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

#### Record Locks(记录锁)

记录锁是索引记录上的锁.例如,`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;`防止其他事务插入，更新或删除`t.cl`值为10的行.

记录锁总是在索引记录上加锁，甚至表没有定义任何索引.对于这样的情况，InnoDB会创建一个隐藏的聚集索引，把这个索引用于记录锁.参考小节 15.6.2.1,“Clustered and Secondary Indexes”.

记录锁的事务数据类似以下输出:

```text
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
0: len 4; hex 8000000a; asc ;;
1: len 6; hex 00000000274f; asc 'O;;
2: len 7; hex b60000019d0110; asc  ;;
```

#### Gap Locks(间隙锁)

间隙锁是在索引记录之间的间隙上的锁,或者是在第一个索引记录之前的间隙上的锁，或最后一个索引记录之后的间隙上的锁.例如，` SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;`防止其他事务插入15进入列`t.c1`，无论是否列中已有此值，因为此范围内所有值之间的间隙都被锁住了.

间隙可以跨过单个索引值，多个索引值，甚至不跨任何索引.

间隙锁是性能与并发之间的部分权衡，只被用在某些事务隔离级别，其他事务不适用。

间隙锁是不需要的，如果一个sql语句对某一行加锁，且它使用唯一索引搜寻唯一一行。例如，如果id列有一个唯一索引，对于某个行的id值为100,下面的语句只使用唯一一个索引记录锁.是否有其他会话在id为100的行的之前的间隙中插入某些行是无关紧要的:

```sql
SELECT * FROM child WHERE id = 100;
```

如果id没有建立索引，或不是唯一索引，这条语句确实会锁住前面的间隙.

这里还值得注意的是，在间隙上有冲突的锁可以由不同的事务持有。例如，事务A在一个间隙上持有共享的间隙锁，同时事务B在同一个间隙上持有排他的间隙锁。可以允许有冲突的间隙锁的原因是如果一个记录被从一个索引清除，被不同事务所持有的此记录上的间隙锁必须被合并.

> 此处没明白

InnoDB的间隙锁是“纯抑制性的”，意味着它们的唯一目的是防止其他事务插入数据到间隙中。间隙锁可以共存。一个事务占有的间隙锁不会阻止其他事物占有相同间隙的间隙锁。共享的和排他的间隙锁之间没什么不同，它们互相之间不冲突，都是执行相同的功能.

间隙锁能被显式禁用.如果你把事务级别改成`READ COMMITTED`，事务锁就被禁用了.在这种情况下间隙锁禁止用于搜寻以及索引扫描，只能用于外键约束检查和重复建(duplicate-key )检查.

使用`READ COMMITTED`隔离级别也会有其他影响。当mysql计算WHERE条件之后，不匹配的行上的记录锁会被释放。对于UPDATE语句，InnoDB会做“半一致性”(semi-consistent)读，因此它返回最新提交的版本给mysql以便mysql可以决定是否当前行匹配UPDATE语句的WHERE条件.

