# 锁原理与监控

与上一章中讨论的优化器一起，锁可能是查询优化的最复杂主题。当锁显示其最糟糕的一面，他们可能会导致白发，即使是最好的专家在锁。但是，不要绝望。本章将向您介绍您需要的锁大部分知识，并可能还需要一些知识。阅读本章后，您应该能够开始调查锁，并用它来获取进一步的知识。

本章开始讨论为什么需要锁和锁访问级别。然后，本章的最大部分将浏览 MySQL 中最常遇到的锁。本章的另一半讨论了锁请求失败的原因、如何减少锁的影响以及如何监视锁。

------

**注意**：大多数示例包括用于重现输出重要部分的语句（某些数据在性质上因情况不同）。由于锁定的有趣部分通常包括多个连接，因此查询的提示已设置为指示在重要时要使用哪个连接。例如，Connection 1+ 表示查询应该由第一个连接执行。

------



## 为什么需要锁？

这似乎是一个完美的世界，不需要锁定数据库。但是，价格会如此之高，以使得只有很少的用例可以使用该数据库，而且对于通用数据库（如 MySQL）来说，这是不可能的。如果没有锁定，则不能具有任何并发性。假设只有一个连接被允许到数据库（你可以争辩说，它本身就是一个锁，因此系统不是无锁的） - 这不是非常有用的大多数应用程序。

------

**注意** 通常，在 MySQL 中称为锁的，通常是一个可以处于已授予或挂起状态的锁请求。

------

当有多个连接同时执行查询时，您需要某种方法来确保连接不会踩到彼此的要塞。这就是锁进入图片的地方。您可以像道路交通中的交通信号一样考虑锁，以调节对资源的访问，以避免事故。在道路交叉口，必须确保两辆车不交叉对方的路径和碰撞。在数据库中，必须确保两个查询对数据的访问不冲突。

由于控制交叉路口的控制级别不同（屈服，停车标志和交通信号灯），数据库中的锁类型也不同。

## 锁访问级别

锁访问级别确定给定锁允许的访问类型。它有时也称为锁类型，但由于这与锁粒度混淆，因此此处使用术语锁访问级别。

基本上有两个访问级别：共享或独占。访问级别根据他们的名字建议进行。共享锁允许其他连接也获取共享锁。这是最宽松的锁访问级别。独占锁只允许该一个连接获取锁。共享锁也称为读取锁，独占锁也称为写入锁。

MySQL 还有一个称为意向锁的概念，它指定了交易的意图。意向锁可以是共享的，也可以是独占的。当下一节将介绍隐式表锁，通过 MySQL 中的主要锁粒度级别时，将更详细地讨论意向锁。

## 锁粒度

MySQL 使用一系列不同的锁粒度（也称为锁类型）来控制对数据的访问。通过使用不同的锁粒度，可以尽可能允许并发访问数据。本节将介绍 MySQL 使用的主要粒度级别。

### User-Level Locks

用户级锁是应用程序可用于保护工作流的显式锁类型。它们不经常使用，但它们对于要序列化访问某些复杂任务非常有用。所有用户锁都是独占锁，使用最多 64 个字符的名称获取。

您可以通过一组功能来操纵用户级锁：

- **GET_LOCK(name, timeout)**：通过指定锁的名称获取锁。第二个参数是以秒为单位的超时;如果未在该时间内获取锁，则函数返回 0。如果获得锁，则返回值为 1。如果超时为负数，函数将无限期地等待锁变为可用。
- **IS_FREE_LOCK(name)**：检查命名锁是否可用。如果锁可用，函数将返回 1;如果锁不可用，则返回 0。
- **IS_USED_LOCK(name)**：这与IS_FREE_LOCK（）函数相反。如果锁正在使用（不可用），则函数返回锁定的连接 ID;如果锁未使用（可用），则返回 NULL。
- **RELEASE_ALL_LOCKS()**：释放连接持有的所有用户级锁。返回值是释放的锁数。
- **RELEASE_LOCK(name)**：使用提供的名称释放锁。如果释放锁，返回值为 1;如果锁存在，但连接不拥有，则返回值为 0;如果锁不存在，则返回值为 NULL。

以通过多次调用 GET_LOCK（） 来获取多个锁。如果这样做，请小心确保所有用户以相同的顺序获取锁，否则可能会发生死锁。如果发生死锁，ER_USER_LOCK_DEADLOCK错误（错误代码 3058）。清单18-1中显示了这方面的一个示例。

```
Listing 18-1. A deadlock for user-level locks
-- Connection 1
Connection 1> SELECT GET_LOCK('my_lock_1', -1);
+---------------------------+
| GET_LOCK('my_lock_1', -1) |
+---------------------------+
| 1                         |
+---------------------------+
1 row in set (0.0100 sec)
-- Connection 2
Connection 2> SELECT GET_LOCK('my_lock_2', -1);
+---------------------------+
| GET_LOCK('my_lock_2', -1) |
+---------------------------+
| 1                         |
+---------------------------+
1 row in set (0.0006 sec)
Connection 2> SELECT GET_LOCK('my_lock_1', -1);
-- Connection 1
Connection 1> SELECT GET_LOCK('my_lock_2', -1);
ERROR: 3058: Deadlock found when trying to get user-level lock; try rolling
back transaction/releasing locks and restarting lock acquisition.
```

当连接 2 尝试获取my_lock_1锁定时，语句将阻塞，直到连接 1 尝试获取my_lock_2触发死锁。如果获得多个锁，则应准备好处理死锁。请注意，对于用户级锁，死锁不会触发事务的回滚。

在"用户"中可以找到已授予和挂起的用户performance_schema。metadata_locks表，OBJECT_TYPE设置为用户级别锁定，如清单 18-2 所示。列出的锁假定您离开系统时，就像触发清单 18-1 中死锁时一样。请注意，某些值（OBJECT_INSTANCE_ BEGIN）将有所不同。

```
Listing 18-2. Listing user-level locks
mysql> SELECT *
 FROM performance_schema.metadata_locks
 WHERE OBJECT_TYPE = 'USER LEVEL LOCK'\G
*************************** 1. row ***************************
 OBJECT_TYPE: USER LEVEL LOCK
 OBJECT_SCHEMA: NULL
 OBJECT_NAME: my_lock_1
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2600542870816
 LOCK_TYPE: EXCLUSIVE
 LOCK_DURATION: EXPLICIT
 LOCK_STATUS: GRANTED
 SOURCE: item_func.cc:4840
 OWNER_THREAD_ID: 76
 OWNER_EVENT_ID: 33
*************************** 2. row ***************************
 OBJECT_TYPE: USER LEVEL LOCK
 OBJECT_SCHEMA: NULL
 OBJECT_NAME: my_lock_2
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2600542868896
 LOCK_TYPE: EXCLUSIVE
 LOCK_DURATION: EXPLICIT
 LOCK_STATUS: GRANTED
 SOURCE: item_func.cc:4840
 OWNER_THREAD_ID: 62
 OWNER_EVENT_ID: 25
*************************** 3. row ***************************
 OBJECT_TYPE: USER LEVEL LOCK
 OBJECT_SCHEMA: NULL
 OBJECT_NAME: my_lock_1
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2600542870336
 LOCK_TYPE: EXCLUSIVE
 LOCK_DURATION: EXPLICIT
 LOCK_STATUS: PENDING
 SOURCE: item_func.cc:4840
 OWNER_THREAD_ID: 62
 OWNER_EVENT_ID: 26
3 rows in set (0.0086 sec)
```

用户OBJECT_TYPE锁的更新是用户级锁，并且锁持续时间是显式的，因为由用户或应用程序再次释放锁。在第 1 行中，与性能架构线程 ID 76 的连接已被授予 my_lock_1 锁，在第 3 行中，线程 ID 62 正在等待（挂起）授予它。线程 ID 62 还具有已授予的锁，该锁包含在第 2 行中。

下一级别的锁涉及非数据表级锁。将讨论的第一个是齐平锁。

### Flush Locks

大多数参与备份的人都会熟悉刷新锁。当您使用 FLUSH TABLES 语句并持续使用语句时，将采用该语句，除非您添加与读取锁，在这种情况下，共享（读取）锁将保持，直到显式释放锁。在 ANALYZE TABLE 语句的末尾也会触发隐式表刷新。齐平锁是表级锁。稍后将在显式锁下讨论使用带读取锁的 FLUSH 表的读取锁。

刷新锁的锁问题的一个常见原因是长时间运行的查询。只要有打开表的查询，FLUSH TABLES 语句就不能刷新表。这意味着，如果在使用一个或多个正在刷新的表进行长时间运行的查询时执行 FLUSH TABLES 语句，则 FLUSH TABLES 语句将阻止需要任何这些表的所有其他语句，直到锁定情况得到解决。

刷新锁受lock_wait_timeout设置的约束。 如果花费多于lock_wait_timeout秒来获取锁，MySQL将放弃该锁。 如果FLUSH TABLES语句被杀死，则同样适用。 但是，由于MySQL内部的原因，在长时间运行的查询完成之前，不能始终释放称为表定义缓存（TDC）版本锁的低级锁。1这意味着确保解决锁问题的唯一方法是 终止长时间运行的查询，但要注意，如果查询更改了许多行，则回滚查询可能需要很长时间。

当刷新锁周围存在锁争用时，FLUSH TABLES语句和随后启动的查询都将状态设置为“等待强制刷新”。 清单18-3显示了一个涉及三个查询的示例。 要自己重现场景，首先执行三个查询，并设置提示为Connection N>，其中N为1、2或3代表三个不同的连接。针对sys.session的查询在第四个连接中完成。 所有查询必须在第一个查询完成之前执行（需要三分钟）。

```
Listing 18-3. Example of waiting for a flush lock
-- Connection 1
Connection 1> SELECT *, SLEEP(180) FROM world.city WHERE ID = 130;
-- Connection 2
Connection 2> FLUSH TABLES world.city;
-- Connection 3
Connection 3> SELECT * FROM world.city WHERE ID = 201;
-- Connection 4
Connection 4> SELECT thd_id, conn_id, state,
 current_statement
 FROM sys.session
 WHERE current_statement IS NOT NULL
 AND thd_id <> PS_CURRENT_THREAD_ID()\G
*************************** 1. row ***************************
 thd_id: 61
 conn_id: 21
 state: User sleep
current_statement: SELECT *, SLEEP(180) FROM world.city WHERE ID = 130
1
https://bugs.mysql.com/bug.php?id=44884
*************************** 2. row ***************************
 thd_id: 62
 conn_id: 22
 state: Waiting for table flush
current_statement: FLUSH TABLES world.city
*************************** 3. row ***************************
 thd_id: 64
 conn_id: 23
 state: Waiting for table flush
current_statement: SELECT * FROM world.city WHERE ID = 201
3 rows in set (0.0598 sec)
```

该示例使用 sys.session 视图;类似的结果可以通过performance_schema线程和显示流程列表获得。为了将输出减小到仅包括与刷新锁讨论相关的查询，将筛选出当前线程和没有持续查询的线程。

与 conn_id = 21 的连接正在执行使用世界的慢速查询。城市表（睡眠（180）用于确保它花了很长时间）。同时，conn_ id = 22 执行了世界.city 表的 FLUSH TABLES 语句。由于第一个查询仍然打开表（查询完成后将释放），因此 FLUSH TABLES 语句最终等待表刷新锁定。最后，conn_id 23 次尝试查询表，因此必须等待 FLUSH TABLES 语句。

另一个非数据表锁是元数据锁。

### Metadata Locks

元数据锁是 MySQL 中较新的锁类型之一。它们是在 MySQL 5.5 中引入的，它们的目的是保护架构，因此当查询或事务依赖于架构不变时，不会更改架构。元数据锁在表级别工作，但应视为表锁的独立锁类型，因为它们不保护表中的数据。

SELECT 语句和 DML 查询接受共享元数据锁，而 DDL 语句接受独占锁。首次使用表时，连接在表上获取元数据锁，并保持该锁直到事务结束。保留元数据锁时，不允许其他连接更改表的架构定义。但是，执行 SELECT 语句和 DML 语句的其他连接不受限制。通常，与元数据锁有关的最大问题就是阻止 DDL 语句启动其工作的空闲事务。

如果围绕元数据锁遇到冲突，您将在进程列表中看到查询状态设置为"等待表元数据锁定"。清单 18-4 中显示了一个示例，其中包括要设置的查询。

```
Listing 18-4. Example of waiting for table metadata lock
-- Connection 1
Connection 1> SELECT CONNECTION_ID();
+-----------------+
| CONNECTION_ID() |
+-----------------+
| 21              |
+-----------------+
1 row in set (0.0003 sec)
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> SELECT * FROM world.city WHERE ID = 130\G
*************************** 1. row ***************************
 ID: 130
 Name: Sydney
CountryCode: AUS
 District: New South Wales
 Population: 3276207
1 row in set (0.0005 sec)
-- Connection 2
Connection 2> SELECT CONNECTION_ID();
+-----------------+
| CONNECTION_ID() |
+-----------------+
| 22              |
+-----------------+
1 row in set (0.0003 sec)
Connection 2> OPTIMIZE TABLE world.city;
-- Connection 3
Connection 3> SELECT thd_id, conn_id, state,
 current_statement,
 last_statement
 FROM sys.session
 WHERE conn_id IN (21, 22)\G
*************************** 1. row ***************************
 thd_id: 61
 conn_id: 21
 state: NULL
current_statement: SELECT * FROM world.city WHERE ID = 130
 last_statement: SELECT * FROM world.city WHERE ID = 130
*************************** 2. row ***************************
 thd_id: 62
 conn_id: 22
 state: Waiting for table metadata lock
current_statement: OPTIMIZE TABLE world.city
 last_statement: NULL
2 rows in set (0.0549 sec)
```

在此示例中，与 conn_id = 21 的连接具有正在进行的事务，在上一个语句中查询 world.city 表（本例中的当前语句与在执行下一个语句之前未清除的语句相同）。当事务仍然处于活动状态时，conn_id = 22 已执行了一个 OPTIMIZE TABLE 语句，该语句正在等待元数据锁。（是的，优化表不会更改架构定义，但它作为 DDL 语句仍受元数据锁的影响。

当元数据锁定的原因是当前语句或最后一个语句时，这很方便。在更常规的情况下，您可以使用 performance_schema.metadata_ locks 表，将 OBJECT_TYPE列设置为 TABLE 以查找已授予和挂起的元数据锁。清单 18-5 显示了使用与上一示例中相同的设置的授予和挂起元数据锁的示例。第 22 章详细介绍了有关调查元数据锁的详情。

```
Listing 18-5. Example of metadata locks
-- Connection 3
Connection 3> SELECT *
 FROM performance_schema.metadata_locks
 WHERE OBJECT_SCHEMA = 'world'
 AND OBJECT_NAME = 'city'\G
*************************** 1. row ***************************
 OBJECT_TYPE: TABLE
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2195760373456
 LOCK_TYPE: SHARED_READ
 LOCK_DURATION: TRANSACTION
 LOCK_STATUS: GRANTED
 SOURCE: sql_parse.cc:6014
 OWNER_THREAD_ID: 61
 OWNER_EVENT_ID: 53
*************************** 2. row ***************************
 OBJECT_TYPE: TABLE
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2194784109632
 LOCK_TYPE: SHARED_NO_READ_WRITE
 LOCK_DURATION: TRANSACTION
 LOCK_STATUS: PENDING
 SOURCE: sql_parse.cc:6014
 OWNER_THREAD_ID: 62
 OWNER_EVENT_ID: 26
2 rows in set (0.0007 sec)
-- Connection 1
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0003 sec)
```

在该示例中，线程 ID 61（与 sys.session 输出中的 conn_id = 22 相同）由于正在进行的事务而在 world.city 表上拥有共享读取锁，线程 ID 62 正在等待锁，因为它正在尝试在表上执行 DDL 语句。

元数据锁的特殊情况是使用 LOCK TABLES 语句显式获取的锁。

### 显式表锁

显式表锁使用锁表和具有读取锁语句的 FLUSH 表。使用 LOCK TABLES 语句，可以获取共享或独占锁;具有读取锁的刷新表始终采用共享锁。表被锁定，直到它们使用 UNLOCK TABLES 语句显式释放。当执行具有读取锁的 FLUSH 表而不列出任何表时，将执行全局读取锁（即影响所有表）。虽然这些锁还可以保护数据，但它们在 MySQL 中被视为元数据锁。

除了与备份相关的带读取锁定的FLUSH TABLES以外，显式表锁通常不与InnoDB一起使用，因为InnoDB的复杂锁功能在大多数情况下优于自己处理锁。 但是，如果您确实需要锁定整个表，则显式锁会很有用，因为它们对于MySQL而言非常便宜。

一个连接的示例在world.country和world.countrylanguage表上具有显式的读锁定，而在world.city表上具有写锁定。

```
mysql> LOCK TABLES world.country READ,
 world.countrylanguage READ,
 world.city WRITE;
Query OK, 0 rows affected (0.0500 sec)
```

当您使用显式锁时，仅允许您根据请求的锁使用已锁定的表。 这意味着，如果您获取读锁定并尝试写入表（ER_TABLE_NOT_LOCKED_FOR_WRITE），或者尝试使用未为（ER_TABLE_NOT_LOCKED）锁定的表，则会出现错误，例如：

```
mysql> UPDATE world.country
 SET Population = Population + 1
 WHERE Code = 'AUS';
ERROR: 1099: Table 'country' was locked with a READ lock and can't be
updated
mysql> SELECT *
 FROM sakila.film
 WHERE film_id = 1;
ERROR: 1100: Table 'film' was not locked with LOCK TABLES
```

由于显式锁被认为是元数据锁，因此performance_schema.metadata_locks表中的症状和信息与隐式元数据锁相同。

另一个表级锁却被隐式处理，通常称为表锁。

### 隐式表锁

查询表时，MySQL会获取隐式表锁。 除刷新，元数据和显式锁外，表锁不对InnoDB表发挥更大的作用，因为InnoDB使用记录锁来允许并发访问表，只要事务未修改相同的行即可（大致来说-如以下小节所示- 不仅如此。

但是InnoDB确实在表级别使用了意图锁的概念，因为在调查锁问题时您可能会遇到意向锁，因此有必要熟悉一下它们。 正如在锁访问级别的讨论中提到的，意图锁标记了事务的意图。 如果您使用explicitLOCK TABLES语句，则该表将直接用您请求的访问级别锁定。

对于事务获取的锁，首先获取意图锁，然后可能需要对其进行升级。 要获得共享锁，事务首先要获取意图共享锁，然后再获取共享锁。 类似地，对于排他锁，首先要使用意图排他锁。 意图锁的一些示例如下：

- SELECT ... FOR SHARE语句将意图共享锁锁定在查询的表上。 SELECT ... LOCK IN SHARE MODE语法是同义词。
- SELECT ... FOR UPDATE语句在查询的表上使用意图互斥锁。
- DML语句（不包括SELECT）在修改后的表上使用意图互斥锁。 如果修改了外键列，则会在父表上使用意图共享锁。

两个意图锁始终彼此兼容。 这意味着即使一个事务具有意向排他锁，也不会阻止另一个事务获得意向锁。 但是，它将阻止其他事务将其意图锁升级到完全锁。 表18-1显示了锁类型之间的兼容性。共享锁分别表示为S和互斥锁X。意向锁的前缀为I，因此IS是有意共享锁，而IX是有意互斥锁。

在表中，复选标记表示两个锁兼容，而跨标记表示两个锁相互冲突。 意图锁的唯一冲突是互斥锁和共享锁。 排他锁与所有其他锁（包括两种意图锁类型）冲突。 共享锁仅与排他锁和意图排他锁冲突。

为什么意图锁甚至是必要的？ 它们允许InnoDB按顺序解析锁请求，而不会阻止兼容操作。 详细信息不在此讨论范围内。 重要的是，您知道意图存在，因此，当您看到它们时，便知道它们的来源。

表级锁可以在 performance_schema.data_locks 表中找到，LOCK_TYPE列设置为 TABLE。清单18-6显示了意向共享锁的示例。

```
Listing 18-6. Example of an InnoDB intention shared lock
-- Connection 1
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> SELECT *
 FROM world.city
 WHERE ID = 130
 FOR SHARE;
Query OK, 1 row affected (0.0010 sec)
-- Connection 2
Connection 2> SELECT *
 FROM performance_schema.data_locks
 WHERE LOCK_TYPE = 'TABLE'\G
*************************** 1. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098223824:1720:2195068346872
ENGINE_TRANSACTION_ID: 283670074934480
 THREAD_ID: 61
 EVENT_ID: 81
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2195068346872
 LOCK_TYPE: TABLE
 LOCK_MODE: IS
 LOCK_STATUS: GRANTED
 LOCK_DATA: NULL
1 row in set (0.0354 sec)
-- Connection 1
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0003 sec)
```

这显示了world.city表上的意图共享锁。 请注意，ENGINE设置为INNODB，并且LOCK_DATA为NULL。 如果您执行相同的查询，则ENGINE_LOCK_ID，ENGINE_TRANSACTION_ID和OBJECT_INSTANCE_BEGIN列的值将有所不同。

如上所述，InnoDB的主要访问级别保护处于记录级别，因此让我们来看一下。

### Record Locks

记录锁通常称为行锁。 但是，它不仅包括行锁，还包括索引锁和间隙锁。 这些通常是在谈论InnoDB锁时要使用的锁。 它们是细粒度的锁，旨在仅锁定最少量的数据，同时仍确保数据完整性。

记录锁可以是共享的或排他的，并且仅影响事务访问的行和索引。 排他锁的持续时间通常是带有异常的事务，例如，是用于INSERT INTO ... ON DUPLICATE KEY和REPLACE语句中的唯一性检查的带有删除标记的记录。 对于共享锁，持续时间可以取决于事务隔离级别，如“减少锁定问题”一节中的“事务隔离级别”所述。

可以使用performance_schema.data_locks表找到记录锁，该表还用于在表级别查找意图锁。 清单18-7显示了使用辅助indexCountryCode更新world.city表中的行的锁示例。

```
Listing 18-7. Example of InnoDB record locks
-- Connection 1
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> UPDATE world.city
 SET Population = Population + 1
 WHERE CountryCode = 'LUX';
Query OK, 1 row affected (0.0009 sec)
Rows matched: 1 Changed: 1 Warnings: 0
-- Connection 2
Connection 2> SELECT *
 FROM performance_schema.data_locks\G
*************************** 1. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098223824:1720:2195068346872
ENGINE_TRANSACTION_ID: 117114
 THREAD_ID: 61
 EVENT_ID: 121
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2195068346872
 LOCK_TYPE: TABLE
 LOCK_MODE: IX
 LOCK_STATUS: GRANTED
 LOCK_DATA: NULL
*************************** 2. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098223824:507:30:1112:2195068344088
ENGINE_TRANSACTION_ID: 117114
 THREAD_ID: 61
 EVENT_ID: 121
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: CountryCode
OBJECT_INSTANCE_BEGIN: 2195068344088
 LOCK_TYPE: RECORD
 LOCK_MODE: X
 LOCK_STATUS: GRANTED
 LOCK_DATA: 'LUX', 2452
*************************** 3. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098223824:507:20:113:2195068344432
ENGINE_TRANSACTION_ID: 117114
 THREAD_ID: 61
 EVENT_ID: 121
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2195068344432
 LOCK_TYPE: RECORD
 LOCK_MODE: X,REC_NOT_GAP
 LOCK_STATUS: GRANTED
 LOCK_DATA: 2452
*************************** 4. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098223824:507:30:1113:2195068344776
ENGINE_TRANSACTION_ID: 117114
 THREAD_ID: 61
 EVENT_ID: 121
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: CountryCode
OBJECT_INSTANCE_BEGIN: 2195068344776
 LOCK_TYPE: RECORD
 LOCK_MODE: X,GAP
 LOCK_STATUS: GRANTED
 LOCK_DATA: 'LVA', 2434
4 rows in set (0.0005 sec)
-- Connection 1
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0685 sec)
```

第一行是已经讨论过的意图排他表锁。第二行是CountryCode索引上的下一键锁（不久之后），值为（'LUX'，2452），其中'LUX'是国家/地区代码 在WHERE子句和2452中使用的是添加到非唯一二级索引的主键ID。 ID = 2452的城市是唯一匹配WHERE子句的城市，主键记录（行本身）显示在输出的第三行中。 锁定模式为X，REC_NOT_GAP，这意味着它是对记录的排他锁，而不是对间隙的锁。

什么是差距？例如在输出的第四行中。间隙锁非常重要，因此对间隙锁的讨论被分割成自己的。

### Gap Locks, Next-Key Locks, and Predicate Locks

间隙锁保护两个记录之间的空间。 它可以在聚集索引的行中，也可以在辅助索引中。 在索引页的第一条记录之前和在该页的最后一条记录之后，分别存在称为最低记录和最高记录的伪记录。 间隙锁通常是引起混乱最多的锁类型。研究锁问题的经验是熟悉它们的最佳方法。

考虑上一个示例中的查询：

```sql
UPDATE world.city
 SET Population = Population + 1
 WHERE CountryCode = 'LUX';
```

此查询更改CountryCode ='LUX'的所有城市的人口。 如果在事务的更新和提交之间插入了新的城市，该怎么办？如果UPDATE和INSERT语句的执行顺序相同，则一切都很好。 但是，如果以相反的顺序提交更改，则结果不一致，因为可以预期插入的行也会被更新。

这是间隙锁起作用的地方。 它保护将插入新记录（包括从不同位置移动的记录）的空间，因此直到持有间隙锁的事务完成后才更改。 如果查看清单18-7中示例输出的第四行的最后一列，则可以看到一个间隙锁的示例：

```
INDEX_NAME: CountryCode
OBJECT_INSTANCE_BEGIN: 2195068344776
 LOCK_TYPE: RECORD
 LOCK_MODE: X,GAP
 LOCK_STATUS: GRANTED
 LOCK_DATA: 'LVA', 2434
```

这是对国家/地区代码索引的值（“ LVA”，2434）的排他性间隙锁。由于查询请求更新所有将国家/地区代码设置为“ LUX”的行，因此间隙锁可确保没有为“ LUX”国家/地区代码。 国家/地区代码“ LVA”是CountryCode索引中的下一个值，因此，“ LUX”和“ LVA”之间的间隙受到排他锁的保护。 另一方面，仍然可以使用CountryCode ='LVA'插入新城市。 在某些地方，这被称为“记录前的间隙”，这使您更容易理解间隙锁定的工作方式。

当您使用READ COMMITTED事务隔离级别而不是REPEATABLE READ或SERIALIZABLE时，差距锁定的程度要小得多。 在“减少锁定问题”部分的“事务隔离级别”中将对此进行进一步讨论。

与间隙锁定有关的是下一键锁定和谓词锁定。 下一键锁定是记录锁定和记录之前的间隙上的间隙锁定的组合。 实际上，这是InnoDB中的默认锁类型，因此您只会在lockoutputs中看到它为S和X。 在本小节和上一小节讨论的示例中，CountryCode索引上的值（“ LUX”，2452）锁和它之前的间隔是下一键锁的示例。 清单18-7中performance_schema.data_locks表中输出的相关部分是

```
*************************** 2. row ***************************
 INDEX_NAME: CountryCode
 LOCK_TYPE: RECORD
 LOCK_MODE: X
 LOCK_STATUS: GRANTED
 LOCK_DATA: 'LUX', 2452
*************************** 3. row ***************************
 INDEX_NAME: PRIMARY
 LOCK_TYPE: RECORD
 LOCK_MODE: X,REC_NOT_GAP
 LOCK_STATUS: GRANTED
 LOCK_DATA: 2452
*************************** 4. row ***************************
 INDEX_NAME: CountryCode
 LOCK_TYPE: RECORD
 LOCK_MODE: X,GAP
 LOCK_STATUS: GRANTED
 LOCK_DATA: 'LVA', 2434
```

概括一下，第2行是下一个键锁，第3行是主键（行）上的记录锁，第4行是“ LUX”和“ LVA”之间的间隙锁（或之前的LVA间隙锁） ）

谓词锁类似于间隙锁，但适用于无法进行绝对排序的空间索引，因此间隙锁没有意义。 对于REPEATABLE READ和SERIALIZABLE事务隔离级别中的空间索引，InnoDB代替了间隙锁，InnoDB在用于查询的最小边界矩形（MBR）上创建了谓词锁。 通过防止最小边界矩形内的数据更改，这将允许一致的读取。

与记录有关的一种最终锁定类型是您应该知道的记录，这是插入意图锁定。

### Insert Intention Locks

请记住，对于表锁，InnoDB具有意图锁，用于确定事务将以共享方式还是独占方式使用表。 同样，InnoDB在记录级别具有插入意图锁。 InnoDB使用这些锁（顾名思义）与INSERT语句一起向其他事务发出信号。 这样，该锁位于尚未创建的记录上（因此它是一个间隙锁），而不是现有记录上的锁。 使用insertintention锁可以帮助增加可以执行插入操作的并发性。

除非INSERT语句正在等待授予锁，否则您不太可能在锁输出中看到插入意图锁。 您可以通过在另一个事务中创建间隙锁定来防止这种情况发生，从而阻止INSERT语句完成。 清单18-8中的示例在Connection 1中创建一个间隙锁，然后在Connection 2中尝试插入与间隙锁冲突的行。 最后，在第三连接中，检索锁定信息。

```
Listing 18-8. Example of an insert intention lock
-- Connection 1
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0004 sec)
Connection 1> SELECT *
 FROM world.city
 WHERE ID > 4079
 FOR UPDATE;
Empty set (0.0009 sec)

-- Connection 2
Connection 2> SELECT PS_CURRENT_THREAD_ID();
+------------------------+
| PS_CURRENT_THREAD_ID() |
+------------------------+
| 62                     |
+------------------------+
1 row in set (0.0003 sec)
Connection 2> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 2> INSERT INTO world.city
 VALUES (4080, 'Darwin', 'AUS',
 'Northern Territory', 146000);
-- Connection 3
Connection 3> SELECT *
 FROM performance_schema.data_locks
 WHERE THREAD_ID = 62\G
*************************** 1. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098220336:1720:2195068326968
ENGINE_TRANSACTION_ID: 117144
 THREAD_ID: 62
 EVENT_ID: 119
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2195068326968
 LOCK_TYPE: TABLE
 LOCK_MODE: IX
 LOCK_STATUS: GRANTED
 LOCK_DATA: NULL
*************************** 2. row ***************************
 ENGINE: INNODB
 ENGINE_LOCK_ID: 2195098220336:507:29:1:2195068320072
ENGINE_TRANSACTION_ID: 117144
 THREAD_ID: 62
 EVENT_ID: 119
 OBJECT_SCHEMA: world
 OBJECT_NAME: city
 PARTITION_NAME: NULL
 SUBPARTITION_NAME: NULL
 INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 2195068320072
 LOCK_TYPE: RECORD
 LOCK_MODE: X,INSERT_INTENTION
 LOCK_STATUS: WAITING
 LOCK_DATA: supremum pseudo-record
2 rows in set (0.0005 sec)
-- Connection 1
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0004 sec)
-- Connection 2
Connection 2> ROLLBACK;
Query OK, 0 rows affected (0.0004 sec)
```

连接2具有Performance Schema线程ID 62，因此在连接3中，可以仅查询该线程并排除连接1占用的锁。请注意，对于RECORD锁，该锁模式包括INSERT_INTENTION –插入意图锁 。 在这种情况下，锁定的数据是最高的伪记录，但是根据情况，也可以是主键的值。 如果您回想起有关下一个键锁定的讨论，那么X表示一个下一个键锁定，但这是一种特殊情况，因为该锁定位于最高伪记录上，并且不可能将其锁定，因此实际上只是 最高伪记录之前的间隙上的间隙锁定。

插入数据时需要注意的另一个锁是自动增量锁。

### Auto-increment Locks

当您将数据插入具有自动递增计数器的表中时，有必要保护该计数器，以确保两个事务都能获得唯一值。 如果对二进制日志使用基于语句的日志记录，则存在更多限制，因为将为所有行重新创建自动增量值，但重播语句时的第一行除外。

InnoDB支持三种锁定模式，因此您可以根据需要调整锁定量。 您可以使用innodb_autoinc_lock_mode选项选择锁定模式，该选项的值分别为0、1和2，其中MySQL 8为默认值2。它需要重新启动MySQL才能更改该值。 值的含义汇总在表18-2中。

innodb_autoinc_lock_mode的值越高，锁定越少。 要付出的代价是增加自动递增值序列中的间隔数，对于innodb_autoinc_lock_mode = 2，则需要交织值。 除非您不能使用基于行的二进制日志记录或对连续的自动增量值有特殊需要，否则建议使用值2。

到此结束了对用户级，元数据和数据级锁的讨论。 您还应该了解与备份相关的其他几个锁。

### Backup Locks

备份锁是一个实例级锁。 也就是说，它会影响整个系统。 它是MySQL 8中引入的新锁。备份锁可防止可能导致备份不一致的语句，同时仍允许其他语句与备份同时执行。 被阻止的语句包括

- 创建，重命名或删除文件的语句。 这包括CREATE TABLE，CREATE TABLESPACE，RENAME TABLE和DROP TABLE语句。
- 帐户管理语句，例如CREATE USER，ALTER USER，DROP USER和GRANT。
- 不将其更改记录到重做日志的DDL语句。 例如，这包括添加索引。

使用LOCK INSTANCE FOR BACKUP语句创建备份锁，并使用UNLOCK INSTANCE语句释放该锁。 它需要BACKUP_ADMIN特权才能执行LOCK INSTANCE FOR BACKUP。 获取备份锁并再次释放的示例是

```
mysql> LOCK INSTANCE FOR BACKUP;
Query OK, 0 rows affected (0.00 sec)
mysql> UNLOCK INSTANCE;
Query OK, 0 rows affected (0.00 sec)
```

------

**注意** 在编写本文时，使用X协议（通过mysqlx_ port指定的端口或mysqlx_socket指定的套接字连接）时，不允许使用备份锁并释放它。 尝试执行此操作将返回ER_PLUGGABLE_PROTOCOL_COMMAND_NOT_SUPPORTED错误：错误：3130：可插拔协议不支持该命令。

------

此外，与备份锁冲突的语句也将使用备份锁。 由于DDL语句有时包含几个步骤，例如，在新文件中重建表并重命名文件，因此可以在两个步骤之间释放备份锁，以避免阻塞LOCK INSTANCE FOR BACKUP的时间超过了必要。

可以在performance_schema.metadata_locks表中将OBJECT_TYPE列设置为BACKUP LOCK来找到备份锁。 清单18-9显示了一个查询的示例，该查询等待LOCK INSTANCE FOR BACKUP持有的备份锁。

```
Listing 18-9. Example of a conflict for the backup lock
-- Connection 1
Connection 1> LOCK INSTANCE FOR BACKUP;
Query OK, 0 rows affected (0.00 sec)
-- Connection 2
Connection 2> OPTIMIZE TABLE world.city;
-- Connection 3
Connection 3> SELECT *
 FROM performance_schema.metadata_locks
 WHERE OBJECT_TYPE = 'BACKUP LOCK'\G
*************************** 1. row ***************************
 OBJECT_TYPE: BACKUP LOCK
 OBJECT_SCHEMA: NULL
 OBJECT_NAME: NULL
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2520402231312
 LOCK_TYPE: SHARED
 LOCK_DURATION: EXPLICIT
 LOCK_STATUS: GRANTED
 SOURCE: sql_backup_lock.cc:101
 OWNER_THREAD_ID: 49
 OWNER_EVENT_ID: 8
*************************** 2. row ***************************
 OBJECT_TYPE: BACKUP LOCK
 OBJECT_SCHEMA: NULL
 OBJECT_NAME: NULL
 COLUMN_NAME: NULL
OBJECT_INSTANCE_BEGIN: 2520403183328
 LOCK_TYPE: INTENTION_EXCLUSIVE
 LOCK_DURATION: TRANSACTION
 LOCK_STATUS: PENDING
 SOURCE: sql_base.cc:5400
 OWNER_THREAD_ID: 60
 OWNER_EVENT_ID: 19
2 rows in set (0.0007 sec)
-- Connection 1
Connection 1> UNLOCK INSTANCE;
Query OK, 0 rows affected (0.00 sec)
```

在该示例中，线程ID为49的连接拥有备份锁，而线程ID为60的连接正在等待备份锁。 请注意，LOCK INSTANCE FOR BACKUP拥有共享锁，而DDL语句请求意图排他锁。

与备份锁定相关的是日志锁定，它也已被引入以减少备份期间的锁定。

### Log Locks

创建备份时，通常需要包括有关备份与之一致的日志位置的信息。 在MySQL 5.7和更早版本中，在获取此信息时需要全局读取锁定。 在MySQL 8中，引入了日志锁定，以允许您读取信息，例如InnoDB的已执行全局事务标识符（GTID），二进制日志位置和日志序列号（LSN），而无需获取全局读取锁定。 日志锁定可防止对日志相关信息进行更改的操作。 在实践中，这意味着提交，刷新日志等。

通过查询performance_schema.log_status表可以隐式获取日志锁定。 它需要BACKUP_ADMIN特权才能访问该表。 清单18-10显示了log_ status表的示例输出。

```
Listing 18-10. Example output of the log_status table
mysql> SELECT *
 FROM performance_schema.log_status\G
*************************** 1. row ***************************
 SERVER_UUID: 59e3f95b-e0d6-11e8-94e8-ace2d35785be
 LOCAL: {"gtid_executed": "59e3f95b-e0d6-11e8-94e8-
ace2d35785be:1-5343", "binary_log_file": "mysqlbin.000033", "binary_log_position": 3874615}
 REPLICATION: {"channels": []}
STORAGE_ENGINES: {"InnoDB": {"LSN": 7888992157, "LSN_checkpoint":
7888992157}}
1 row in set (0.0004 sec)
```

总结了MySQL中的主要锁类型。 查询请求锁但无法授予锁时会发生什么？ 让我们考虑一下。

## Failure to Obtain Locks

锁的整个想法是限制对对象或记录的访问，以避免冲突的操作并发执行。 这意味着有时无法授予锁定。 在这种情况下会发生什么？ 这取决于请求的锁定和情况。 元数据锁（包括显式请求的表锁）以超时操作。 InnoDB记录锁都支持超时和显式死锁检测。

------

**注意** 确定两个锁是否彼此兼容非常复杂。 由于这种关系不是对称的，因此变得特别有趣，也就是说，在存在另一个锁的情况下可以允许一个锁，但反之则不行。 例如，插入意图锁定必须等待间隙锁定，但是间隙锁定不必等待插入意图锁定。 另一个示例（缺少传递性）是，间隙加记录锁必须等待仅记录锁，插入意图锁必须等待间隙加记录锁，但是插入意图锁不需要等待加锁。 仅记录锁定。

------

重要的是要理解，在使用数据库时，获取锁的失败是生活中必不可少的事实。 原则上，您可以使用非常粗粒度的锁，并且可以避免超时以外的失败锁–这就是MyISAM存储引擎所做的操作，因此写入并发性非常差。 但是，在实践中，为了允许较高的并发写入工作负载，最好使用细粒度的锁，这还会带来死锁的可能性。

结论是，您应该始终使您的应用程序准备好重试以获取锁定或正常失败。 无论是显式锁还是隐式锁，这都适用。

------

**提示** 始终准备处理失败以获取锁。 未能获得锁不是灾难性的错误，通常不应将其视为错误。 就是说，如“减少锁定问题”一节中所讨论的，在开发应用程序时，有一些值得考虑的减少锁定争用的技术。

------

本章的其余部分将讨论表级超时，记录级超时和InnoDB死锁的细节。

### Metadata and Backup Lock Wait Timeouts

当您请求刷新，元数据或备份锁定时，获取锁定的尝试将在lock_wait_timeout秒后超时。 默认超时为31536000秒（365天）。 您可以动态设置lock_wait_timeout选项，也可以在全局范围和会话范围内进行设置，这使您可以根据给定进程的特定需要调整超时。

发生超时时，该语句失败，并显示错误ER_LOCK_WAIT_TIMEOUT（错误号1205）。 例如：

```
mysql> LOCK TABLES world.city WRITE;
ERROR: 1205: Lock wait timeout exceeded; try restarting transaction
```

建议的lock_wait_timeout选项设置取决于应用程序的要求。 使用较小的值来防止锁定请求长时间阻止其他查询可能是一个优点。 通常，这将要求您例如通过重试该语句来实现对锁定请求失败的处理。 另一方面，较大的值可以避免必须重试该语句。

对于FLUSH TABLES语句，还请记住它与较低级别的表定义高速缓存（TDC）版本锁进行交互，这可能意味着放弃该语句将不允许后续查询继续进行。 在这种情况下，最好为lock_wait_timeout设置一个较高的值，以使锁定关系更清晰。

### InnoDB Lock Wait Timeouts

当查询请求InnoDB中的记录级锁定时，它受到的超时类似于刷新，元数据和备份锁定的超时。 由于记录级锁争用比表级锁争用更常见，并且记录级锁增加了发生死锁的可能性，因此超时默认为50秒。 可以使用innodb_ lock_wait_timeout选项进行设置，该选项可以为全局范围和会话范围设置。

发生超时时，查询失败，并显示ER_LOCK_WAIT_TIMEOUT错误（错误号1205），就像表级锁定超时一样。 清单18-11显示了一个InnoDB锁定等待超时发生的示例。

```
Listing 18-11. Example of an InnoDB lock wait timeout
-- Connection 1
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 130;
Query OK, 1 row affected (0.0005 sec)
Rows matched: 1 Changed: 1 Warnings: 0
-- Connection 2
Connection 2> SET SESSION innodb_lock_wait_timeout = 3;
Query OK, 0 rows affected (0.0004 sec)
Connection 2> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 130;
ERROR: 1205: Lock wait timeout exceeded; try restarting transaction
-- Connection 1
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0003 sec)
```

在此示例中，连接2的锁定等待超时设置为3秒，因此不必等待通常的50秒以使超时发生。

发生超时时，innodb_rollback_on_timeout选项定义回滚事务完成的工作量。 如果禁用innodb_rollback_on_ timeout（缺省值），则仅回退触发超时的语句。 启用该选项后，整个事务将回滚。 只能在全局级别上配置innodb_ rollback_on_timeout选项，并且需要重新启动才能更改该值。

------

**警告** 处理锁等待超时非常重要，因为否则可能会使事务具有未释放的锁。 如果发生这种情况，其他事务可能无法获取所需的锁。

------

通常建议将InnoDB记录级锁定的超时保持较低。 通常，最好将其值从默认的50秒降低。 查询等待锁的时间越长，其他锁请求受到影响的可能性就越大，这也可能导致其他查询停滞。 它还使死锁更有可能发生。 如果禁用死锁检测（接下来将讨论），则应为innodb_lock_wait_timeout使用非常小的值，例如一两秒，因为您将使用超时来检测死锁。 如果没有死锁检测，还建议启用innodb_rollback_on_timeout选项。

### 死锁

死锁听起来像一个非常可怕的概念，但是您不应让这个名称吓到您。 就像锁等待超时一样，死锁是高并发数据库世界中不可或缺的事实。 真正的含义是，锁定请求之间存在循环关系。 解决僵局的唯一方法是强制其中一个请求放弃。 从这个意义上讲，死锁与锁定等待超时没有什么不同。 实际上，您可以禁用死锁检测，在这种情况下，其中一个锁将最终以锁等待超时结束。

那么，如果真的不需要死锁，那为什么还会有死锁呢？ 由于死锁是在锁定请求之间存在循环关系时发生的，因此InnoDB有可能在循环完成后立即检测到死锁。 这样，InnoDB可以立即通知用户发生了死锁，而不必等待锁等待超时发生。 被告知发生死锁也很有用，因为死锁通常提供了改善应用程序中数据访问的机会。 因此，您应该将僵局视为朋友而不是敌人。 图18-1显示了两个事务查询一个导致死锁的表的示例。

在该示例中，事务1首先更新ID = 130的行，然后更新ID = 3805的行。在这两者之间，事务2首先更新ID = 3805的行，然后更新ID = 130的行。 当事务1尝试更新ID = 3805时，事务2已对该行进行了锁定。 事务2也无法进行，因为它无法获得ID = 130的锁，因为事务1已经持有该锁。 这是一个简单的死锁的经典示例。 圆形锁定关系也显示在图18-2中。

在此图中，可以清楚地看到事务1和2持有哪个锁，请求了哪些锁，以及如何在没有干预的情况下永远无法解决冲突。 这使其成为僵局。 在现实世界中，僵局通常更为复杂。

在此处讨论的示例中，仅涉及主键记录锁。 通常，通常还涉及辅助钥匙，间隙锁和其他可能的锁类型。 可能还涉及两个以上的事务。 但是，原理保持不变。

------

**注意** 对于两个事务中的每个事务，甚至只有一个查询就可能发生死锁。 如果一个查询以升序读取记录，而另一个查询以降序读取，则可能会出现死锁。

------

当发生死锁时，InnoDB会选择“完成最少工作”的事务成为受害者。 您可以在information_ schema.INNODB_TRX视图中查看trx_weight列，以查看InnoDB使用的权重（完成的工作越多，权重越高）。 实际上，这意味着拥有最少锁的事务将被回滚。 发生这种情况时，被选择为受害者的事务中的查询将失败，并返回错误ER_LOCK_DEADLOCK（错误代码1213），并且事务将回滚以释放尽可能多的锁。 清单18-12中显示了发生死锁的示例。

```
Listing 18-12. Example of a deadlock
-- Connection 1
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 130;
Query OK, 1 row affected (0.0006 sec)
Rows matched: 1 Changed: 1 Warnings: 0
-- Connection 2
Connection 2> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 2> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 3805;
Query OK, 1 row affected (0.0006 sec)
Rows matched: 1 Changed: 1 Warnings: 0
Connection 2> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 130;
-- Connection 1
Connection 1> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 3805;
ERROR: 1213: Deadlock found when trying to get lock; try restarting
transaction
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0438 sec)
-- Connection 2
Connection 2> ROLLBACK;
Query OK, 0 rows affected (0.0438 sec)
```

在大多数情况下，自动死锁检测非常有效，可以避免查询停滞时间超过必要的时间。 死锁检测不是免费的。 对于查询并发度很高的MySQL实例，查找死锁的成本可能会变得很高，最好禁用死锁检测，这是通过将innodb_deadlock_detect选项设置为OFF来完成的。 也就是说，在MySQL 8.0.18和更高版本中，死锁检测已移至专用的后台线程，从而提高了性能。

如果确实禁用了死锁检测，建议将innodb_lock_wait_ timeout设置为非常低的值（例如1秒）以快速检测锁争用。 另外，启用innodb_rollback_on_timeout选项以确保释放锁。

既然您已经了解了锁的工作方式以及锁请求如何失败，那么您需要考虑如何减少锁的影响

## Reduce Locking Issues

在编写应用程序并设计用于其数据和访问的架构时，请牢记锁定很重要。 减少锁定的策略包括添加索引，更改事务隔离级别和抢先锁定。

提示不要在优化锁过程中迷失方向。 如果您仅偶尔遇到锁等待超时和死锁，通常最好重试查询或事务，而不要花时间来避免此问题。 太频繁的频率取决于您的工作量，但是对于许多应用程序而言，每小时重试一次并不是一个问题。

### Transaction Size and Age

减少锁定问题的重要策略是使事务保持较小状态，避免延迟，使事务保持打开状态的时间超过必要时间。锁定问题的最常见原因是修改大量行或活动时间长于必要时间的事务。

事务的大小是事务完成的工作量，特别是它需要执行的锁数量，但是事务执行所需的时间也很重要。正如本讨论中的其他一些主题将解决的那样，您可以通过索引和事务隔离级别来部分减少影响。但是，牢记整体结果也很重要。如果需要修改许多行，请问自己是否可以将工作分成较小的批处理，或者要求所有事情都在同一事务中完成。也可以拆分一些准备工作，然后在主事务之外进行。

交易的持续时间也很重要。一个常见的问题是使用autocommit = 0的连接。每次在没有活动事务的情况下执行查询（包括SELECT）时，都会启动一个新事务，并且直到执行明确的COMMIT或ROLLBACK（或关闭连接）后，该事务才会完成）。某些连接器默认情况下会禁用自动提交，因此您可能会在未意识到的情况下使用此模式，这会导致事务错误打开数小时。

提示启用自动提交选项，除非您有特定的原因要禁用它。 启用自动提交后，InnoDB还可以针对许多SELECT查询将其检测为只读事务并减少查询的开销。

另一个陷阱是在事务处于活动状态时启动事务并在应用程序中执行慢速操作。 这可以是发送回用户，交互式提示或文件I / O的数据。 当您在MySQL中没有打开活动事务时，请确保执行这些缓慢的操作。

### Indexes

索引减少了访问给定行所执行的工作量。 这样，索引是减少锁定的好工具，因为只有在执行查询时访问的记录才会被锁定。

考虑一个简单的示例，在其中查询world.city表中名称为Sydney的城市：

```sql
START TRANSACTION;
SELECT *
 FROM world.city
 WHERE Name = 'Sydney'
 FOR SHARE;
```

FOR SHARE选项用于强制查询对读取的记录采取共享锁。 默认情况下，“名称”列上没有索引，因此查询将执行全表扫描以查找结果中所需的行。 没有索引，有4103个记录锁（有些是重复的）：

```
mysql> SELECT INDEX_NAME, LOCK_TYPE,
 LOCK_MODE, COUNT(*)
 FROM performance_schema.data_locks
 WHERE OBJECT_SCHEMA = 'world'
 AND OBJECT_NAME = 'city'
 GROUP BY INDEX_NAME, LOCK_TYPE, LOCK_MODE;
+------------+-----------+-----------+----------+
| INDEX_NAME | LOCK_TYPE | LOCK_MODE | COUNT(*) |
+------------+-----------+-----------+----------+
| NULL       | TABLE     | IS        | 1        |
| PRIMARY    | RECORD    | S         | 4103     |
+------------+-----------+-----------+----------+
2 rows in set (0.0210 sec)
```

如果在“名称”列上添加索引，则锁计数将减少到总共三个记录锁：

```
mysql> SELECT INDEX_NAME, LOCK_TYPE,
 LOCK_MODE, COUNT(*)
 FROM performance_schema.data_locks
 WHERE OBJECT_SCHEMA = 'world'
 AND OBJECT_NAME = 'city'
 GROUP BY INDEX_NAME, LOCK_TYPE, LOCK_MODE;
+------------+-----------+---------------+----------+
| INDEX_NAME | LOCK_TYPE | LOCK_MODE     | COUNT(*) |
+------------+-----------+---------------+----------+
| NULL       | TABLE     | IS            | 1        |
| Name       | RECORD    | S             | 1        |
| PRIMARY    | RECORD    | S,REC_NOT_GAP | 1        |
| Name       | RECORD    | S,GAP         | 1        |
+------------+-----------+---------------+----------+
4 rows in set (0.0005 sec)
```

另一方面，更多的索引提供了更多访问相同行的方式，这可能会增加死锁的数量。

### Record Access Order

确保您尽可能多地以相同顺序访问不同事务的记录。 在本章前面讨论的死锁示例中，导致死锁的是两个事务以相反的顺序访问行。 如果他们以相同的顺序访问行，就不会有死锁。 当您访问不同表中的记录时，这也适用。

确保相同的访问顺序绝非易事。 当您执行联接并且优化器为两个查询决定不同的联接顺序时，甚至可能会发生不同的访问顺序。 如果不同的连接顺序导致过多的锁定问题，则可以考虑使用第17章中介绍的优化器提示来告诉优化器更改连接顺序，但是在这种情况下，当然您应该牢记查询性能。

### Transaction Isolation Levels

InnoDB支持几种事务隔离级别。 不同的隔离级别具有不同的锁要求：特别是可重复读取和可序列化要求的锁比读取已提交的更多。

READ COMMITTED事务隔离级别可以通过两种方式帮助锁定问题。 采取的间隙锁定要少得多，并且在DML语句期间访问但未修改的行在语句完成后会再次释放它们的锁。 对于REPEATABLE READ和SERIALIZABLE，锁仅在事务结束时释放。

------

**注意** 经常有人说READ COMMITTED事务隔离级别没有间隙锁。 这是一个神话，并不正确。 尽管使用的间隙锁少得多，但仍然需要一些间隙锁。 例如，这包括在InnoDB执行页面拆分作为更新的一部分时。 （在第25章中讨论了页面拆分。）

------

考虑一个示例，其中使用CountryCode列更改了名为Sydney的城市的人口，以将查询限制到一个国家。 可以使用以下查询完成此操作：

```sql
START TRANSACTION;
UPDATE world.city
 SET Population = 5000000
 WHERE Name = 'Sydney'
 AND CountryCode = 'AUS';
```

“名称”列上没有索引，但是“国家/地区代码”上没有索引。 因此，更新需要对CountryCode索引的一部分进行扫描。 清单18-13显示了以REPEATABLE READ事务隔离级别执行查询的示例。

```
Listing 18-13. The locks held in the REPEATABLE READ transaction isolation level
-- Connection 1
Connection 1> SET transaction_isolation = 'REPEATABLE-READ';
Query OK, 0 rows affected (0.0003 sec)
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> UPDATE world.city
 SET Population = 5000000
 WHERE Name = 'Sydney'
 AND CountryCode = 'AUS';
Query OK, 1 row affected (0.0005 sec)
Rows matched: 1 Changed: 1 Warnings: 0
-- Connection 2
Connection 2> SELECT INDEX_NAME, LOCK_TYPE,
 LOCK_MODE, COUNT(*)
 FROM performance_schema.data_locks
 WHERE OBJECT_SCHEMA = 'world'
 AND OBJECT_NAME = 'city'
 GROUP BY INDEX_NAME, LOCK_TYPE, LOCK_MODE;
+-------------+-----------+---------------+----------+
| INDEX_NAME  | LOCK_TYPE | LOCK_MODE     | COUNT(*) |
+-------------+-----------+---------------+----------+
| NULL        | TABLE     | IX            | 1        |
| CountryCode | RECORD    | X             | 14       |
| PRIMARY     | RECORD    | X,REC_NOT_GAP | 14       |
| CountryCode | RECORD    | X,GAP         | 1        |
+-------------+-----------+---------------+----------+
4 rows in set (0.0007 sec)
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0725 sec)
```

每个CountryCode索引和主键都获得了14个记录锁，而CountryCode索引则获得了一个间隙锁。 将此与在READ COMMITTED事务隔离级别中执行查询后持有的锁进行比较，如清单18-14所示。

```sql
Listing 18-14. The locks held in the READ-COMMITTED transaction isolation level
-- Connection 1
Connection 1> SET transaction_isolation = 'READ-COMMITTED';
Query OK, 0 rows affected (0.0003 sec)
Connection 1> START TRANSACTION;
Query OK, 0 rows affected (0.0003 sec)
Connection 1> UPDATE world.city
 SET Population = 5000000
 WHERE Name = 'Sydney'
 AND CountryCode = 'AUS';
Query OK, 1 row affected (0.0005 sec)
Rows matched: 1 Changed: 1 Warnings: 0
-- Connection 2
Connection 2> SELECT INDEX_NAME, LOCK_TYPE,
 LOCK_MODE, COUNT(*)
 FROM performance_schema.data_locks
 WHERE OBJECT_SCHEMA = 'world'
 AND OBJECT_NAME = 'city'
 GROUP BY INDEX_NAME, LOCK_TYPE, LOCK_MODE;
+-------------+-----------+---------------+----------+
| INDEX_NAME  | LOCK_TYPE | LOCK_MODE     | COUNT(*) |
+-------------+-----------+---------------+----------+
| NULL        | TABLE     | IX            | 1        |
| CountryCode | RECORD    | X,REC_NOT_GAP | 1        |
| PRIMARY     | RECORD    | X,REC_NOT_GAP | 1        |
+-------------+-----------+---------------+----------+
3 rows in set (0.0006 sec)
Connection 1> ROLLBACK;
Query OK, 0 rows affected (0.0816 sec)
```

在这里，记录锁被减少为对CountryCount索引和主键的每个锁。 没有缝隙锁。

并非所有工作负载都可以使用READ COMMITTED事务隔离级别。 如果必须让SELECT语句在同一事务中多次执行时返回相同的结果，或者不同的查询在时间上对应于同一快照，则必须使用REPEATABLE READ或SERIALIZABLE。 但是，在许多情况下，可以选择降低隔离级别，并且可以为不同的事务选择不同的隔离级别。 如果要从Oracle DB迁移应用程序，则已经在使用READ COMMITTED，并且还可以在MySQL中使用它。

### Preemptive Locking

将要讨论的最后一种策略是抢先锁定。 如果您有一个复杂的事务执行多个查询，则在某些情况下执行SELECT ... FOR UPDATE或SELECT ... FOR SHARE查询来锁定您知道以后需要在事务中使用的记录可能是有利的。 。 可能有用的另一种情况是确保针对不同任务以相同的顺序访问行。

抢先锁定对于减少死锁的频率特别有效。 缺点之一是您将持有这些锁的时间更长。 总体而言，抢先锁定是一种应谨慎使用的策略，但是在正确的情况下使用时，它可以有效防止死锁。

本章的最后一个主题是回顾如何监视锁。

## Monitoring Locks

已经有几个查询有关持有的锁的信息的示例。 本节将回顾已经提到的资源，并介绍一些额外的资源。 第22章将通过研究锁定问题的示例进一步介绍这一点。 监视选项可以分为四组：性能模式，sys模式，状态度量和InnoDB锁定监视。

### The Performance Schema

性能模式包含死锁以外的大多数可用锁信息的来源。 您不仅可以直接在性能模式中使用锁定信息，还可以在应用程序中使用锁定信息。 它也用于sys模式中的两个与锁相关的视图。

该信息可通过以下四个表获得：

- **data_locks**：此表包含InnoDB级别的表和锁记录的详细信息。 它显示了当前持有或待处理的所有锁。
- **data_lock_waits**：与data_locks表类似，它显示与InnoDB相关的锁，但仅显示等待中的锁，以及有关哪些线程正在阻止请求的信息。
- **metadata_locks**：此表包含有关用户级锁，元数据锁等的信息。 要记录信息，必须启用wait / lock / metadata / sql / mdl Performance Schema工具（默认情况下在MySQL 8中启用）。 OBJECT_TYPE列显示持有哪种类型的锁。
- **table_handles**：此表保存有关当前有效的表锁的信息。 必须启用wait / lock / table / sql / handler Performance Schema工具才能记录数据（这是默认设置）。 该表的使用频率低于其他表。

metadata_locks表是表中最通用的表，并且支持各种锁，从全局读取锁到低级锁（如访问控制列表（ACL））。 表18-3按字母顺序总结了OBJECT_TYPE列的可能值，并简要说明了每个值代表的锁定。

性能架构表中的数据是原始锁定数据。 通常，当您调查锁定问题或监视锁定问题时，确定是否有任何锁定等待会更有趣。 对于该信息，您需要使用sys模式。

### The sys Schema

sys模式具有两个视图，这些视图获取“性能模式”表中的信息并返回锁对，其中一个锁由于另一个锁而无法授予。 因此，它们显示了锁定等待存在的问题。 这两个视图是innodb_lock_waits和schema_table_lock_waits。

innodb_lock_waits视图使用性能模式中的data_locks和data_lock_waits视图返回所有等待InnoDB记录锁定的情况。 它显示信息，例如连接尝试获取的锁定以及涉及的连接和查询。 如果您需要信息而无需格式化，该视图也将以x $ innodb_lock_waits的形式存在。

schema_table_lock_waits视图的工作方式类似，但是使用metadata_locks表返回与模式对象相关的锁定等待。 该信息也可以在x $ schema_table_lock_waits视图中以未格式化的形式获得。 第22章包含使用这两种视图调查锁定问题的示例。

### Status Counters and InnoDB Metrics

有几个状态计数器和InnoDB度量提供有关锁定的信息。 这些主要用于全局（实例）级别，可用于检测锁定问题的总体增加。 一起监视所有这些指标的一种好方法是使用sys.metrics视图。 清单18-15显示了一个检索指标的示例。

```
Listing 18-15. Lock metrics
mysql> SELECT Variable_name,
 Variable_value AS Value,
 Enabled
 FROM sys.metrics
 WHERE Variable_name LIKE 'innodb_row_lock%'
 OR Variable_name LIKE 'Table_locks%'
 OR Type = 'InnoDB Metrics - lock';
+-------------------------------+--------+---------+
| Variable_name                 | Value  | Enabled |
+-------------------------------+--------+---------+
| innodb_row_lock_current_waits | 0      | YES     |
| innodb_row_lock_time          | 595876 | YES     |
| innodb_row_lock_time_avg      | 1683   | YES     |
| innodb_row_lock_time_max      | 51531  | YES     |
| innodb_row_lock_waits         | 354    | YES     |
| table_locks_immediate         | 4194   | YES     |
| table_locks_waited            | 0      | YES     |
| lock_deadlocks                | 1      | YES     |
| lock_rec_lock_created         | 0      | NO      |
| lock_rec_lock_removed         | 0      | NO      |
| lock_rec_lock_requests        | 0      | NO      |
| lock_rec_lock_waits           | 0      | NO      |
| lock_rec_locks                | 0      | NO      |
| lock_row_lock_current_waits   | 0      | YES     |
| lock_table_lock_created       | 0      | NO      |
| lock_table_lock_removed       | 0      | NO      |
| lock_table_lock_waits         | 0      | NO      |
| lock_table_locks              | 0      | NO      |
| lock_timeouts                 | 1      | YES     |
+-------------------------------+--------+---------+
19 rows in set (0.0076 sec)
```

如您所见，默认情况下并非所有指标都已启用。 可以使用第7章中讨论的innodb_monitor_enable选项启用那些未启用的选项。innodb_row_lock_％，lock_deadlocks和lock_timeouts度量标准是最有趣的。 行锁指标显示当前正在等待多少个锁，并统计等待获取InnoDB记录锁所花费的时间（以毫秒为单位）。 lock_deadlocks和lock_timeouts指标分别显示了遇到的死锁和锁等待超时数。

### InnoDB Lock Monitor and Deadlock Logging

InnoDB长期以来一直拥有自己的锁监视器，并在InnoDB监视器输出中返回了锁信息。 默认情况下，InnoDB监视器包含有关最新死锁以及锁等待中涉及的锁的信息。 通过启用innodb_ status_output_locks选项（默认情况下禁用），将列出所有锁； 这类似于性能模式data_locks表中的内容。

为了演示死锁和事务信息，您可以从清单18-12创建死锁，并创建一个新的正在进行的事务，该事务通过world.city表中的主键更新了一行：

```sql
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.0002 sec)
mysql> UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 130;
Query OK, 1 row affected (0.0005 sec)
Rows matched: 1 Changed: 1 Warnings: 0
```

您可以使用SHOW ENGINE INNODB STATUS语句生成InnoDB锁定监视器输出。 清单18-16显示了启用所有锁信息并生成监视器输出的示例。 完整的InnoDB监视器输出也可以从本书的GitHub存储库中的listing_18_16.txt文件中获得。

```
Listing 18-16. The InnoDB monitor output
mysql> SET GLOBAL innodb_status_output_locks = ON;
Query OK, 0 rows affected (0.0022 sec)
mysql> SHOW ENGINE INNODB STATUS\G
*************************** 1. row ***************************
 Type: InnoDB
 Name:
Status:
=====================================
2019-11-04 17:04:48 0x6e88 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 51 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 170 srv_active, 0 srv_shutdown, 62448 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 138
OS WAIT ARRAY INFO: signal count 133
RW-shared spins 1, rounds 1, OS waits 0
RW-excl spins 109, rounds 1182, OS waits 34
RW-sx spins 24, rounds 591, OS waits 18
Spin rounds per wait: 1.00 RW-shared, 10.84 RW-excl, 24.63 RW-sx
------------------------
LATEST DETECTED DEADLOCK
------------------------
2019-11-03 19:41:43 0x4b78
*** (1) TRANSACTION:
TRANSACTION 5585, ACTIVE 10 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 37, OS thread handle 28296, query id 21071 localhost ::1
root updating
UPDATE world.city
 SET Population = Population + 1
 WHERE ID = 130
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 159 page no 28 n bits 248 index PRIMARY of table
`world`.`city` trx id 5585 lock_mode X locks rec but not gap
Record lock, heap no 26 PHYSICAL RECORD: n_fields 7; compact format; info
bits 0
 0: len 4; hex 80000edd; asc ;;
 1: len 6; hex 0000000015d1; asc ;;
 2: len 7; hex 01000000f51aa6; asc ;;
 3: len 30; hex 53616e204672616e636973636f2020202020202020202020
202020202020; asc San Francisco ;
(total 35 bytes);
 4: len 3; hex 555341; asc USA;;
 5: len 20; hex 43616c69666f726e696120202020202020202020; asc
California ;;
 6: len 4; hex 800bda1e; asc ;;
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
...
------------
TRANSACTIONS
------------
Trx id counter 5662
Purge done for trx's n:o < 5661 undo n:o < 0 state: running but idle
History list length 11
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 284075292758256, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 284075292756560, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 284075292755712, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 5661, ACTIVE 60 sec
2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 40, OS thread handle 2044, query id 26453 localhost ::1
root
TABLE LOCK table `world`.`city` trx id 5661 lock mode IX
RECORD LOCKS space id 160 page no 7 n bits 248 index PRIMARY of table
`world`.`city` trx id 5661 lock_mode X locks rec but not gap
Record lock, heap no 41 PHYSICAL RECORD: n_fields 7; compact format; info
bits 0
 0: len 4; hex 80000082; asc ;;
 1: len 6; hex 00000000161d; asc ;;
 2: len 7; hex 01000001790a72; asc y r;;
 3: len 30; hex 5379646e657920202020202020202020202020202020202020202020202
0; asc Sydney ; (total 35 bytes);
 4: len 3; hex 415553; asc AUS;;
 5: len 20; hex 4e657720536f7574682057616c65732020202020; asc New South
Wales ;;
 6: len 4; hex 8031fdb0; asc 1 ;;
...
```

顶部附近是最新检测到的死锁部分，其中包含有关最新死锁及其发生时间的事务和锁的详细信息。 如果自从上次MySQL重新启动以来没有发生死锁，则忽略此部分。 第22章将提供调查僵局的示例。

------

**注意** InnoDB监视器输出中的死锁部分仅包含涉及InnoDB记录锁的死锁信息。 对于涉及用户级别锁的死锁，没有等效信息。

------

在输出的下方，有TRANSACTIONS部分列出了InnoDB事务。 请注意，不包含没有任何锁的事务（例如，纯SELECT查询）。 在该示例中，在world.city表上保留了一个意图排他锁，在主键等于130的行上具有排他锁（第一个字段的记录锁信息中的80000082表示该行的值为0x82 ，与十进制表示法中的130相同）。

提示如今，可以从performance_schema.data_locks和performance_ schema.data_lock_waits表中更好地获取InnoDB监视器输出中的锁定信息。 但是，死锁信息仍然非常有用。 InnoDB监视器输出中的死锁部分仅包含涉及InnoDB记录锁的死锁信息。 对于涉及用户级别锁的死锁，没有等效信息。

在输出的下方，有TRANSACTIONS部分列出了InnoDB事务。 请注意，不包含没有任何锁的事务（例如，纯SELECT查询）。 在该示例中，在world.city表上保留了一个意图排他锁，在主键等于130的行上具有排他锁（第一个字段的记录锁信息中的80000082表示该行的值为0x82 ，与十进制表示法中的130相同）。

------

**提示** 如今，可以从performance_schema.data_locks和performance_ schema.data_lock_waits表中更好地获取InnoDB监视器输出中的锁定信息。 但是，死锁信息仍然非常有用。

------

您可以要求每15秒将监视器输出转储到stderr。 您可以通过启用innodb_status_output选项来启用转储。 请注意，输出很大，因此如果启用它，请准备好使错误日志迅速增长。 InnoDB监视器的输出还可以轻松结束有关更严重问题的消息的隐藏。

如果要确保记录所有死锁，可以启用innodb_print_ all_deadlocks选项。 每次发生死锁时，都会使InnoDB监视器输出中的死锁信息打印到错误日志中。 如果您需要调查死锁，这可能很有用，但是建议仅按需启用死锁，以避免错误日志变得非常大并可能隐藏其他问题。

------

**注意** 如果启用InnoDB监视器的常规输出或有关所有死锁的信息，请小心。 该信息可以轻松隐藏记录到错误日志中的重要消息。

------



## 总结

锁的主题又大又复杂。希望本章可以帮助您概述为什么需要锁以及各种锁。

本章开始询问为什么需要锁。没有锁，对模式和数据进行并发访问是不安全的。隐喻地讲，数据库锁的工作方式与交通信号灯相同，而停车标志在交通中的工作方式相同。它调节对数据的访问，因此事务可以确保不会与另一个事务发生冲突而导致结果不一致。

数据有两种访问级别：共享访问（也称为读取访问）和排他访问（也称为写入访问）。这些访问级别适用于各种锁粒度，范围从全局读取锁到记录锁和间隙锁。另外，InnoDB在表级别使用意图共享和意图排他锁。

重要的是减少应用程序所需的锁数量并减少所需锁的影响。减少锁问题的策略实质上归结为通过使用索引并将大型事务拆分为较小的事务，并尽可能短地持有锁，从而使事务在事务中尽可能少地工作。对于应用程序中的不同任务，尝试以相同的顺序访问数据也很重要。否则，可能会发生不必要的死锁。

本章的最后部分介绍了性能模式，sys模式，状态度量和InnoDB监视器中的锁监视选项。最好使用性能模式表和sys模式视图来完成大多数监视。死锁例外，InnoDB监视器仍然是最佳选择。

这是第四部分的结论。现在是时候使查询分析变得更加实用了，首先要找到适合优化的查询。