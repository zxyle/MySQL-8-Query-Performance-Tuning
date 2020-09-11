# 事务

事务是报表的老大哥。 它们将多个更改组合在一起（无论是在单个语句中还是在多个语句中），因此它们可以作为单个单元应用或放弃。 通常，事务只不过是事后才想到的事情，只是在有必要一起应用多个语句时才进行考虑。 那是考虑交易的一种坏方法。 它们对于确保数据完整性非常重要，如果使用不当，可能会导致严重的性能问题。

本章开始讨论为什么需要通过回顾事务对锁和性能的影响，从性能角度认真对待事务。本章的其余部分侧重于分析事务，首先使用表，然后是 InnoDB 监视器、InnoDB 指标，最后是性能架构。

## 交易的影响

如果您认为事务是用于对查询进行分组的容器，那么它们可能看起来是一个无辜的概念。但是，重要的是要了解，由于事务为查询组提供原子性，因此事务处于活动状态的时间越长，与查询关联的资源持有的时间越长，在事务中完成的工作越多，所需的资源就更多。在提交事务之前，仍在使用的查询使用哪些资源？主要两种是锁定和撤消日志。

### 锁

当查询执行时，它需要，并且当您使用默认事务隔离级别 -时，所有锁将保留到提交事务。使用 READ级别时，可能会释放某些锁，但至少会保留那些涉及已更改记录的锁。锁本身是一个资源，但它也需要内存来存储有关锁的信息。对于正常工作负载，您可能不太想这样做，但巨大的事务最终可能会使用太多的内存，以使事务在

```
ERROR: 1206: The total number of locks exceeds the lock table size
```

从记录到错误日志的警告消息（更短的时间）中可以看到，锁所需的内存从缓冲池中获取。因此，您持有的锁越多，持有的时间越长，可用于缓存数据和索引的内存就更少。

错误之前在错误日志中发出警告，指出超过 67% 的缓冲池用于锁或自适应哈希索引：

```
2019-07-06T03:23:04.345256Z 10 [Warning] [MY-011958] [InnoDB] Over 67
percent of the buffer pool is occupied by lock heaps or the adaptive hash
index! Check that your transactions do not set too many row locks. Your
buffer pool size is 7 MB. Maybe you should make the buffer pool bigger?.
Starting the InnoDB Monitor to print diagnostics, including lock heap and
hash index sizes.
```



警告后是 InnoDB 监视器的定期重复输出，因此您可以确定哪些事务是罪魁祸首。事务 InnoDB 监视器输出将在"InnoDB 监视器"部分中讨论。

在事务，经常忽略的一个锁类型是元数据锁。当语句查询表时，将获取共享元数据锁，并且该元数据锁将一直保留到事务结束。虽然表上存在元数据锁，但任何连接都无法对表执行任何 DDL表）。如果 DDL 语句被长时间运行的事务阻止，它将反过来阻止所有新查询使用该表。第章将展示调查此类问题的示例，包括使用本章中的一些方法。

当事务处于活动状态时，锁将保持。但是，即使事务已完成撤消日志，它仍可能产生影响。

### 撤消日志

如果您选择回滚事务，则事务期间存储到所需的状态。这很容易理解。更令人惊讶的是，即使没有进行任何更改的事务也会使来自其他事务的撤消信息继续存在。当事务需要读取视图（一致快照）时，将发生这种情况，在使用重复读取事务隔离级别时就是这种情况。读取视图意味着事务将返回对应于事务启动时对应的行数据，无论其他事务是否更改数据。为了能够实现这一点，有必要保留在事务生存期内更改的行的旧值。具有读取视图的长期运行事务是最终出现大量撤消日志的最常见原因，在 MySQL 5.7 和更早版本中，文件最终很大。（在 MySQL 8 中，撤消日志始终存储在可截断的单独的撤消表空间中。

撤消日志的活动部分的大小在历史记录列表长度。历史记录列表长度是尚未清除撤消日志时所提交事务的事务数。这意味着不能使用历史记录列表长度来测量行更改的总量。它告诉您执行查询时必须考虑的旧行（每个事务一个单位）中有多少旧行单位（每个事务一个单位）。此链接列表越长，查找每行的正确版本的成本越高。最后，如果您有一个大型的历史记录列表，它会严重影响所有查询的性能。

什么是大历史列表长度？没有关于这个严格的规则——只是越小越好。通常，当列表长度大约为 1000 万到 1000 万事务时，性能问题开始显示，但它成为瓶颈的点取决于在撤消日志中提交事务的点以及历史记录列表长度较大的工作负载。

当不再需要时，InnoDB 会自动在后台清除历史记录列表。有两个选项可以控制清除，还有两个选项可以影响在无法完成清除时发生的情况。选项是

- 每个批处理清除的撤消日志页数。批处理在清除线程之间分配。此选项不应在生产系统上更改。默认值为 300，有效值介于 1 和 5000 之间。
- 要并行使用的清除线程数。如果数据更改跨越多个表，较高的并行性可能很有用。另一方面，如果所有更改都集中在少数表中，则首选低值。更改清除线程数需要重新启动 MySQL。默认值为 4，有效值介于 1 和 32 之间。
- 当历史记录列表长度超过延迟添加到更改数据的操作中，以降低历史记录列表以牺牲较高的语句延迟为代价的增长速率。默认值为 0，这意味着永远不会添加延迟。有效值为 0=4294967295。
- 当历史记录列表长度大于 D. 时，可添加到 DML 查询。

通常不需要更改这些设置中的任何一个;然而，在特殊情况下，它可以是有用的。如果清除线程跟不上，您可以尝试根据被修改的表数更改清除线程的数量;如果清除线程跟不上，可以尝试根据修改的表数更改清除线程数。修改的表越多，清除线程的用值就越大。更改清除线程数时，在更改之前监视以基线开始的效果非常重要，以便查看更改是否进行了改进。

最大清除延迟选项可用于减慢修改数据的 DML 语句的速度。当写入仅限于特定连接且延迟不会导致创建额外的写入线程以保持相同的吞吐量时，它最有用。

如何监视事务有多老，锁使用多少内存，以及历史记录列表有多长？您可以使用信息架构、InnoDB 监视器和性能架构来获取此信息。

## INNODB_TRX

信息中的表是有关 InnoDB 事务的最专用信息源。它包括事务启动时间、修改行数以及持有锁数等信息。INNODB_TRX还使用表提供有关锁等待问题中涉及的事务的一些信息。表汇总了表中的列。

| **列/数据类型**                        | **描述**                                                     |
| :------------------------------------- | :----------------------------------------------------------- |
| trx_id瓦尔查尔（18）                   | 事务 ID。当引用事务或与 InnoDB 监视器的输出进行比较时，这非常有用。否则，ID 应纯粹在内部处理，不应赋予其任何意义。ID 仅分配给已修改数据或锁定行的事务;否则，ID 将分配给已修改数据或锁定行的事务。仅执行只读 SELECT 语句的具有虚拟 ID，如 421124985258256，如果事务开始修改或锁定记录，该 ID 将更改。 |
| trx_statevarchar(13)                   | 事务的状态。这可能是一个运行，和。                           |
| trx_startedDatetime                    | 使用时区启动事务时。                                         |
| trx_requested_lock_idvarchar(105)      | 当时，此列显示事务正在等待的锁的 ID。                        |
| trx_wait_startedDatetime               | 当，此列显示锁等待何时开始使用系统时区。                     |
| trx_weight大无符号                     | 事务在修改行和持有的锁方面完成了多少工作。这是用于确定死锁时回滚哪个事务的权重。重量越高，完成的工作就更多。 |
| trx_mysql_thread_id大无符号            | 执行事务的连接的连接性能架构表中的一个连接列）。             |
| trx_queryvarchar(1024)                 | 事务当前执行的查询。如果事务处于空闲状态，则查询为。         |
| trx_operation_statevarchar(64)         | 事务执行的当前操作。即使正在执行查询，也可能为               |
| trx_tables_in_use大无符号              | 事务使用的表数。                                             |
| trx_tables_locked大无符号              | 事务包含行锁的表数。                                         |
| trx_lock_structs大无符号               | 事务创建的结构数。                                           |
| trx_lock_memory_bytes大无符号          | 事务持有的锁使用的内存量（以字节为单位）。                   |
| trx_rows_locked大无符号                | 事务持有的记录锁数。虽然称为行锁，但它还包括索引锁。         |
| trx_rows_modified大无符号              | 事务修改的行数。                                             |
| trx_concurrency_tickets大无符号        | 当 0 时，事务将分配在必须允许其他事务执行任务之前，可以使用该票证。一个票证对应于访问一行。此列显示还剩下多少张票证。 |
| trx_isolation_levelvarchar(16)         | 用于事务的事务隔离级别。                                     |
| trx_unique_checksInt                   | 是否连接启用了该变量。                                       |
| trx_foreign_key_checksInt              | 是否连接启用了该变量。                                       |
| trx_last_foreign_key_errorvarchar(256) | 遇到的上次（如果有）外键错误的错误消息。                     |
| trx_adaptive_hash_latchedInt           | 事务是否锁定了自适应哈希索引的一部分。共有一此列实际上是布尔值。 |
| trx_adaptive_hash_timeout大无符号      | 是否在整个多个查询中保留自适应哈希索引的锁。如果自适应哈希索引只有一个部件，并且没有争用，则超时将倒计时，当超时达到 0 时，将释放锁。当存在争用或有多个部分时，每次查询后始终释放锁，超时值为 0。 |
| trx_is_read_onlyInt                    | 事务是否为只读事务。事务可以通过显式声明或启用自动提交的单语句事务（InnoDB 可以检测到查询将仅读取数据）为只读。 |
| trx_autocommit_non_lockingInt          | 当事务是单语句非锁定时，此列设置为 1。当此列和为 1 时，InnoDB 可以优化事务以减少开销。 |

从该表的信息可以确定哪些事务的影响最大。清单显示了为两个事务返回的信息的示例。

```
Listing 21-1. Example output of the INNODB_TRX table
mysql> SELECT *
 FROM information_schema.INNODB_TRX\G
*************************** 1. row ***************************
 trx_id: 5897
 trx_state: RUNNING
 trx_started: 2019-07-06 11:11:12
 trx_requested_lock_id: NULL
 trx_wait_started: NULL
 trx_weight: 4552416
 trx_mysql_thread_id: 10
 trx_query: UPDATE db1.t1 SET val1 = 4
 trx_operation_state: updating or deleting
 trx_tables_in_use: 1
 trx_tables_locked: 1
 trx_lock_structs: 7919
 trx_lock_memory_bytes: 1417424
 trx_rows_locked: 4552415
 trx_rows_modified: 4544497
 trx_concurrency_tickets: 0
 trx_isolation_level: REPEATABLE READ
 trx_unique_checks: 1
 trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
 trx_is_read_only: 0
trx_autocommit_non_locking: 0
*************************** 2. row ***************************
 trx_id: 421624759431440
 trx_state: RUNNING
 trx_started: 2019-07-06 11:46:55
 trx_requested_lock_id: NULL
 trx_wait_started: NULL
 trx_weight: 0
 trx_mysql_thread_id: 8
 trx_query: SELECT COUNT(*) FROM db1.t1
 trx_operation_state: counting records
 trx_tables_in_use: 1
 trx_tables_locked: 0
 trx_lock_structs: 0
 trx_lock_memory_bytes: 1136
 trx_rows_locked: 0
 trx_rows_modified: 0
 trx_concurrency_tickets: 0
 trx_isolation_level: REPEATABLE READ
 trx_unique_checks: 1
 trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
 trx_is_read_only: 1
trx_autocommit_non_locking: 1
2 rows in set (0.0023 sec)
```



第一行显示修改数据的事务示例。在检索信息时，已修改了 4，544，497 行，并且记录锁更多。您还可以看到事务仍在主动执行查询语句）。

第二行是在启用自动提交时执行的 SELECT。由于启用了自动提交，因此事务中只能有一个语句（显式 START禁用自动提交）。列显示它是一个查询，没有任何锁子句，因此它是一个只读语句。这意味着 InnoDB 可以跳过某些操作，例如准备保存锁定和撤消事务的信息，从而降低事务的开销。列设置为 1 以反映这一点。

您应该担心哪些事务取决于您系统的预期工作负载。如果您有 OLAP 工作负荷，则预期会有相对较长的运行查询。对于纯 OLTP 工作负载，任何运行超过几秒钟且修改多个行的事务都可能是问题的迹象。例如，要查找超过一分钟的事务，可以使用以下查询：

```
SELECT *
 FROM information_schema.INNODB_TRX
 WHERE trx_started < NOW() - INTERVAL 1 MINUTE;
```



与数据库是 InnoDB 监视器中的事务列表。

## InnoDB 监视器

是一种瑞士军刀，包含 InnoDB 信息，还包括交易信息。监视器输出中的"交易"部分专用于事务信息。此信息不仅包括事务列表，还包括历史记录列表长度。清单显示了 InnoDB 监视器的摘录，其中示例为事务部分，该部分是在上一个输出的表。

```
Listing 21-2. Transaction information from the InnoDB monitor
mysql> SHOW ENGINE INNODB STATUS\G
*************************** 1. row ***************************
 Type: InnoDB
 Name:
Status:
=====================================
2019-07-06 11:46:58 0x7f7728f69700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 6 seconds
...
------------
TRANSACTIONS
------------
Trx id counter 5898
Purge done for trx's n:o < 5894 undo n:o < 0 state: running but idle
History list length 3
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421624759429712, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421624759428848, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 5897, ACTIVE 2146 sec updating or deleting
mysql tables in use 1, locked 1
7923 lock struct(s), heap size 1417424, 4554508 row lock(s), undo log
entries 4546586
MySQL thread id 10, OS thread handle 140149617817344, query id 25 localhost
127.0.0.1 root updating
UPDATE db1.t1 SET val1 = 4
```



"事务"的顶部显示事务 ID 计数器的当前值，后跟从撤消日志中清除内容的信息。它显示小于 5894 的事务 ID 的撤消日志已清除。此清除的后面越大，历史记录列表长度越大（在节的第三行）。从 InnoDB 监视器输出中读取历史记录列表长度是获取历史记录列表长度的传统方法。下一节将介绍在用于监视目的时如何以更好的方式获取值。

本节的其余部分是事务列表。请注意，虽然输出的生成方式与列表仅包括一个活动事务）。在 MySQL 5.7 及更晚的中，只读非锁定事务不包括在 InnoDB 监视器事务列表中。因此，如果需要包括所有活动最好使用"表"。

如前所述，还有一种获取历史记录列表长度的替代方法。您需要为此使用 InnoDB 指标。

## INNODB_METRICS和系统指标

InnoDB 监视器报告对于数据库管理员获取 InnoDB 中所做所为的概述非常有用，但对于监视，它不像它需要分析才能以监视可以使用的方式获取数据那样有用。您在章节前面看到如何从数据获取有关但历史列表长度等指标如何？

包括多个指标，用于显示有关视图。这些指标都位于事务子系统中。清单显示了，它们是否默认启用，以及解释指标量值的简短注释。

```
Listing 21-3. InnoDB metrics related to transactions
mysql> SELECT NAME, COUNT, STATUS, COMMENT
 FROM information_schema.INNODB_METRICS
 WHERE SUBSYSTEM = 'transaction'\G
*************************** 1. row ***************************
 NAME: trx_rw_commits
 COUNT: 0
 STATUS: disabled
COMMENT: Number of read-write transactions committed
*************************** 2. row ***************************
 NAME: trx_ro_commits
 COUNT: 0
 STATUS: disabled
COMMENT: Number of read-only transactions committed
*************************** 3. row ***************************
 NAME: trx_nl_ro_commits
 COUNT: 0
 STATUS: disabled
COMMENT: Number of non-locking auto-commit read-only transactions committed
*************************** 4. row ***************************
 NAME: trx_commits_insert_update
 COUNT: 0
 STATUS: disabled
COMMENT: Number of transactions committed with inserts and updates
*************************** 5. row ***************************
 NAME: trx_rollbacks
 COUNT: 0
 STATUS: disabled
COMMENT: Number of transactions rolled back
*************************** 6. row ***************************
 NAME: trx_rollbacks_savepoint
 COUNT: 0
 STATUS: disabled
COMMENT: Number of transactions rolled back to savepoint
*************************** 7. row ***************************
 NAME: trx_rollback_active
 COUNT: 0
 STATUS: disabled
COMMENT: Number of resurrected active transactions rolled back
*************************** 8. row ***************************
 NAME: trx_active_transactions
 COUNT: 0
 STATUS: disabled
COMMENT: Number of active transactions
*************************** 9. row ***************************
 NAME: trx_on_log_no_waits
 COUNT: 0
 STATUS: disabled
COMMENT: Waits for redo during transaction commits
*************************** 10. row ***************************
 NAME: trx_on_log_waits
 COUNT: 0
 STATUS: disabled
COMMENT: Waits for redo during transaction commits
*************************** 11. row ***************************
 NAME: trx_on_log_wait_loops
 COUNT: 0
 STATUS: disabled
COMMENT: Waits for redo during transaction commits
*************************** 12. row ***************************
 NAME: trx_rseg_history_len
 COUNT: 45
 STATUS: enabled
COMMENT: Length of the TRX_RSEG_HISTORY list
*************************** 13. row ***************************
 NAME: trx_undo_slots_used
 COUNT: 0
 STATUS: disabled
COMMENT: Number of undo slots used
*************************** 14. row ***************************
 NAME: trx_undo_slots_cached
 COUNT: 0
 STATUS: disabled
COMMENT: Number of undo slots cached
*************************** 15. row ***************************
 NAME: trx_rseg_current_size
 COUNT: 0
 STATUS: disabled
COMMENT: Current rollback segment size in pages
15 rows in set (0.0403 sec)
```



这些指标中最重要的列表长度。这也是默认情况下启用的唯一指标。与提交和回滚相关的指标可用于确定您有多少读写、只读和非锁定只读事务，以及它们提交和回滚的频繁发生。许多回滚表明存在问题。如果您怀疑重做日志是一个指标来衡量事务提交期间等待重做日志的事务量度。

查询 InnoDB 指标的替代一种方式是使用，该视图还包括全局状态变量。清单显示了使用获取当前值以及是否启用指标的示例。

```
Listing 21-4. Using the sys.metrics view to get the transaction metrics
mysql> SELECT Variable_name AS Name,
 Variable_value AS Value,
 Enabled
 FROM sys.metrics
 WHERE Type = 'InnoDB Metrics - transaction';
+---------------------------+-------+---------+
| Name | Value | Enabled |
+---------------------------+-------+---------+
| trx_active_transactions | 0 | NO |
| trx_commits_insert_update | 0 | NO |
| trx_nl_ro_commits | 0 | NO |
| trx_on_log_no_waits | 0 | NO |
| trx_on_log_wait_loops | 0 | NO |
| trx_on_log_waits | 0 | NO |
| trx_ro_commits | 0 | NO |
| trx_rollback_active | 0 | NO |
| trx_rollbacks | 0 | NO |
| trx_rollbacks_savepoint | 0 | NO |
| trx_rseg_current_size | 0 | NO |
| trx_rseg_history_len | 45 | YES |
| trx_rw_commits | 0 | NO |
| trx_undo_slots_cached | 0 | NO |
| trx_undo_slots_used | 0 | NO |
+---------------------------+-------+---------+
15 rows in set (0.0152 sec)
```



这表明历史记录列表长度为 45，这是一个很好的低值，因此撤消日志中几乎没有开销。其余指标将被禁用。

迄今为止，对交易信息的讨论是关于所有交易或单个交易的汇总统计数据。如果要更深入地了解事务已完成的工作，则需要使用性能架构。

## 性能架构事务

性能架构支持 MySQL 5.7 及更晚中的事务监视，默认情况下在 MySQL 8 中启用该监视。除了与 XA 事务和性能架构中可用的保存点相关的事务详细信息以外的表获取这些存储点。但是，性能架构事务事件的优点是，您可以将它们与其他事件类型（如语句）相结合，以获取有关事务所完成工作的信息。这是本节的主要焦点。此外，性能架构提供包含聚合统计信息的汇总表。

### 事务事件及其语句

用于调查性能架构中的事务的主要表是事务事件。有三个表用于记录当前或它们包含表。

| 列/数据类型                             | 描述                                                         |
| :-------------------------------------- | :----------------------------------------------------------- |
| THREAD_ID大无符号                       | 执行事务的连接的性能架构线程 ID。                            |
| EVENT_ID大无符号                        | 事件的事件 ID。可以使用事件 ID 将线程的事件或作为外键与事件表之间的线程 ID 一起排列。 |
| END_EVENT_ID大无符号                    | 事务完成时的事件如果事件 ID 为，则事务仍在进行中。           |
| EVENT_NAMEvarchar(128)                  | 事务事件名称。目前，此列始终具有值。                         |
| 状态枚举                                | 事务的状态。可能的值为                                       |
| TRX_ID大无符号                          | 这当前未使用，并将始终为。                                   |
| 格蒂德varchar(64)                       | 交易记录的 GTID。当自动确定 GTID 时（通常），返回。这与执行事务的的变量相同。 |
| XID_FORMAT_IDInt                        | 对于 XA 事务，格式 ID。                                      |
| XID_GTRIDvarchar(130)                   | 对于 XA 事务，gtrid 值。                                     |
| XID_BQUALvarchar(130)                   | 对于 XA 交易记录，bqual 值。                                 |
| XA_STATEvarchar(64)                     | 对于，事务的状态。它可以是活动，或。                         |
| 源varchar(64)                           | 记录事件的源代码文件和行号。                                 |
| TIMER_START大无符号                     | 事件开始的时间（以皮秒为单位）。                             |
| TIMER_END大无符号                       | 事件完成时的时间（以皮秒为单位）。如果事务尚未完成，则该值对应于当前时间。 |
| TIMER_WAIT大无符号                      | 执行事件所用的总时间（以皮秒为单位）。如果事件尚未完成，则该值对应于事务处于活动状态的时间。 |
| ACCESS_MODE枚举                         | 事务是只读 （只读 ）读写 （） 模式.                          |
| ISOLATION_LEVELvarchar(64)              | 事务的事务隔离级别。                                         |
| 自动通信枚举                            | 事务根据自动提交选项，以及是否已启动显式事务。可能的值为"和  |
| NUMBER_OF_SAVEPOINTS大无符号            | 在事务中创建的保存点数。                                     |
| NUMBER_OF_ROLLBACK_TO_SAVEPOINT大无符号 | 事务回滚到保存点的时间。                                     |
| NUMBER_OF_RELEASE_SAVEPOINT大无符号     | 事务释放保存点的时间。                                       |
| OBJECT_INSTANCE_BEGIN大无符号           | 此字段当前未使用，始终设置为。                               |
| NESTING_EVENT_ID大无符号                | 触发事务的事件的事件 ID。                                    |
| NESTING_EVENT_TYPE枚举                  | 触发事务的事件的事件类型。                                   |

如果您正在使用 XA 事务，则当您需要恢复事务时，事务事件表非常好，因为格式 ID、gtrid 和 bqual 值直接从表中可用，必须分析输出。同样，如果您使用保存点，可以获取有关保存点使用情况的统计信息。否则，该信息与表中的可用信息

对于使用表可以启动两个事务。第一个事务是更新多个城市人口的正常事务：

```
START TRANSACTION;
UPDATE world.city SET Population = 5200000 WHERE ID = 130;
UPDATE world.city SET Population = 4900000 WHERE ID = 131;
UPDATE world.city SET Population = 2400000 WHERE ID = 132;
UPDATE world.city SET Population = 2000000 WHERE ID = 133;
```



第二个事务是 XA 事务：

```
XA START 'abc', 'def', 1;
UPDATE world.city SET Population = 900000 WHERE ID = 3805;
```



清单显示了显示当前表的示例输出。

```
mysql> SELECT *
 FROM performance_schema.events_transactions_current
 WHERE STATE = 'ACTIVE'\G
*************************** 1. row ***************************
 THREAD_ID: 54
 EVENT_ID: 39
 END_EVENT_ID: NULL
 EVENT_NAME: transaction
 STATE: ACTIVE
 TRX_ID: NULL
 GTID: AUTOMATIC
 XID_FORMAT_ID: NULL
 XID_GTRID: NULL
 XID_BQUAL: NULL
 XA_STATE: NULL
 SOURCE: transaction.cc:219
 TIMER_START: 488967975158077184
 TIMER_END: 489085567376530432
 TIMER_WAIT: 117592218453248
 ACCESS_MODE: READ WRITE
 ISOLATION_LEVEL: REPEATABLE READ
 AUTOCOMMIT: NO
 NUMBER_OF_SAVEPOINTS: 0
NUMBER_OF_ROLLBACK_TO_SAVEPOINT: 0
 NUMBER_OF_RELEASE_SAVEPOINT: 0
 OBJECT_INSTANCE_BEGIN: NULL
 NESTING_EVENT_ID: 38
 NESTING_EVENT_TYPE: STATEMENT
*************************** 2. row ***************************
 THREAD_ID: 57
 EVENT_ID: 10
 END_EVENT_ID: NULL
 EVENT_NAME: transaction
 STATE: ACTIVE
 TRX_ID: NULL
 GTID: AUTOMATIC
 XID_FORMAT_ID: 1
 XID_GTRID: abc
 XID_BQUAL: def
 XA_STATE: ACTIVE
 SOURCE: transaction.cc:219
 TIMER_START: 488977176010232448
 TIMER_END: 489085567391481984
 TIMER_WAIT: 108391381249536
 ACCESS_MODE: READ WRITE
 ISOLATION_LEVEL: REPEATABLE READ
 AUTOCOMMIT: NO
 NUMBER_OF_SAVEPOINTS: 0
NUMBER_OF_ROLLBACK_TO_SAVEPOINT: 0
 NUMBER_OF_RELEASE_SAVEPOINT: 0
 OBJECT_INSTANCE_BEGIN: NULL
 NESTING_EVENT_ID: 9
 NESTING_EVENT_TYPE: STATEMENT
2 rows in set (0.0007 sec)
```



第 1 行中的事务是常规事务，而第 2 行中的事务是 XA 事务。这两个事务都是由从嵌套事件类型中可以看到的语句启动的。如果要查找触发事务的语句，可以使用该语句查询，如

```
mysql> SELECT SQL_TEXT
 FROM performance_schema.events_statements_history
 WHERE THREAD_ID = 54
 AND EVENT_ID = 38\G
*************************** 1. row ***************************
SQL_TEXT: START TRANSACTION
1 row in set (0.0009 sec)
```



这表明，由 THREAD_ID 的事务是使用的。由于表仅包含连接的最后 10 个语句，因此不能保证启动事务的语句仍在历史记录表中。如果在禁用自动提交时查看单语句事务或第一个语句（仍在执行时），则需要查询表。

事务和语句之间的关系也相反。给定事务事件 ID 和线程 ID，可以使用语句事件历史记录和当前表查询为该事务执行的最后 10 个语句。清单和来自清单的第 1 行）的示例，其中包括启动事务的语句和后续语句。

```
Listing 21-6. Finding the last ten statements executed in a transaction
mysql> SET @thread_id = 54,
 @event_id = 39,
 @nesting_event_id = 38;
mysql> SELECT EVENT_ID, SQL_TEXT,
 FORMAT_PICO_TIME(TIMER_WAIT) AS Latency,
 IF(END_EVENT_ID IS NULL, 'YES', 'NO') AS IsCurrent
 FROM ((SELECT EVENT_ID, END_EVENT_ID,
 TIMER_WAIT,
 SQL_TEXT, NESTING_EVENT_ID,
 NESTING_EVENT_TYPE
 FROM performance_schema.events_statements_current
 WHERE THREAD_ID = @thread_id
 ) UNION (
 SELECT EVENT_ID, END_EVENT_ID,
 TIMER_WAIT,
 SQL_TEXT, NESTING_EVENT_ID,
 NESTING_EVENT_TYPE
 FROM performance_schema.events_statements_history
 WHERE THREAD_ID = @thread_id
 )
 ) events
 WHERE (NESTING_EVENT_TYPE = 'TRANSACTION'
 AND NESTING_EVENT_ID = @event_id)
 OR EVENT_ID = @nesting_event_id
 ORDER BY EVENT_ID DESC\G
*************************** 1. row ***************************
 EVENT_ID: 43
 SQL_TEXT: UPDATE city SET Population = 2000000 WHERE ID = 133
 Latency: 291.01 us
IsCurrent: NO
*************************** 2. row ***************************
 EVENT_ID: 42
 SQL_TEXT: UPDATE city SET Population = 2400000 WHERE ID = 132
 Latency: 367.59 us
IsCurrent: NO
*************************** 3. row ***************************
 EVENT_ID: 41
 SQL_TEXT: UPDATE city SET Population = 4900000 WHERE ID = 131
 Latency: 361.03 us
IsCurrent: NO
*************************** 4. row ***************************
 EVENT_ID: 40
 SQL_TEXT: UPDATE city SET Population = 5200000 WHERE ID = 130
 Latency: 399.32 us
IsCurrent: NO
*************************** 5. row ***************************
 EVENT_ID: 38
 SQL_TEXT: START TRANSACTION
 Latency: 97.37 us
IsCurrent: NO
9 rows in set (0.0012 sec)
```



子查询（派生表）从表和表查找线程语句事件。有必要包括当前事件，因为交易可能有持续语句。语句通过作为事务的子级或事务的嵌套事件（EVENT_ID。这将包括从启动事务的语句开始的所有语句。如果有持续声明，最多会有 11 个声明，否则最多有 10 个声明。

END_EVENT_ID 用于确定语句当前是否正在执行，并且语句使用 EVENT_ID 进行行中，第 5 行中最早的语句语句）。

这种类型的查询不仅有助于调查仍在执行查询的事务。当您遇到空闲事务，并且想知道事务在被放弃之前做了什么时，它也非常有用。查找活动事务的另一个相关方式是使用 表来包含有关每个连接的事务状态的信息。清单显示了一个查询活动事务的示例，不包括执行查询的连接的行。

```
Listing 21-7. Finding active transactions with sys.session
mysql> SELECT *
 FROM sys.session
 WHERE trx_state = 'ACTIVE'
 AND conn_id <> CONNECTION_ID()\G
*************************** 1. row ***************************
 thd_id: 54
 conn_id: 16
 user: mysqlx/worker
 db: world
 command: Sleep
 state: NULL
 time: 690
 current_statement: UPDATE world.city SET Population = 2000000 WHERE ID = 133
 statement_latency: NULL
 progress: NULL
 lock_latency: 281.76 ms
 rows_examined: 341
 rows_sent: 341
 rows_affected: 0
 tmp_tables: 0
 tmp_disk_tables: 0
 full_scan: NO
 last_statement: UPDATE world.city SET Population = 2000000 WHERE ID = 133
last_statement_latency: 391.80 ms
 current_memory: 2.35 MiB
 last_wait: NULL
 last_wait_latency: NULL
 source: NULL
 trx_latency: 11.49 m
 trx_state: ACTIVE
 trx_autocommit: NO
 pid: 23376
 program_name: mysqlsh
*************************** 2. row ***************************
 thd_id: 57
 conn_id: 18
 user: mysqlx/worker
 db: world
 command: Sleep
 state: NULL
 time: 598
 current_statement: UPDATE world.city SET Population = 900000 WHERE ID = 3805
 statement_latency: NULL
 progress: NULL
 lock_latency: 104.00 us
 rows_examined: 1
 rows_sent: 0
 rows_affected: 1
 tmp_tables: 0
 tmp_disk_tables: 0
 full_scan: NO
 last_statement: UPDATE world.city SET Population = 900000 WHERE ID = 3805
last_statement_latency: 40.21 ms
 current_memory: 344.76 KiB
 last_wait: NULL
 last_wait_latency: NULL
 source: NULL
 trx_latency: 11.32 m
 trx_state: ACTIVE
 trx_autocommit: NO
 pid: 25836
 program_name: mysqlsh
2 rows in set (0.0781 sec)
```



这表明第一行中的事务已处于活动状态超过 11 分钟，并且自上次执行查询以来是 690 秒（11.5 分钟）（您的值会有所不同）。系统可用于确定连接执行的最后一个查询。这是一个废弃的事务示例，它阻止 InnoDB 清除其撤消日志。废弃事务的最常见原因是数据库管理员以交互方式启动事务并分心，或者提交被禁用，并且没有意识到事务已启动。

您可以回滚事务以避免更改任何数据。对于第一个（正常）事务：

```
mysql> ROLLBACK;
Query OK, 0 rows affected (0.0841 sec)
```

对于 XA 事务：

```
mysql> XA END 'abc', 'def', 1;
Query OK, 0 rows affected (0.0003 sec)
mysql> XA ROLLBACK 'abc', 'def', 1;
Query OK, 0 rows affected (0.0759 sec)
```

性能架构表可用于分析事务的另一种方式是使用汇总表获取聚合数据。

### 事务摘要表

与使用语句摘要所执行语句的报告一样，也有可用于分析事务使用的事务汇总表。虽然它们不像对语句对应项那样有用，但它们确实提供了对哪些连接和帐户以不同的方式使用事务的见解。

有五个事务汇总表按全局或按帐户、主机、线程或用户对数据进行分组。所有摘要也按事件名称分组，但由于目前只有一个事务事件（），因此它是一个零操作。表是

- 所有事务聚合。此表中只有一行。
- 按用户名和主机名分组的事务。
- 按帐户的主机名分组的事务。
- 按线程分组的事务。仅包含当前存在的线程。
- 按帐户的用户名部分分组的事件。

每个表都包含事务统计信息分组的列和三组列：总计、读写事务和只读事务。对于这三组列，都有事务总数以及总、最小、平均和最大延迟。清单显示了来自

```
Listing 21-8. The events_transactions_summary_global_by_event_name table
mysql> SELECT *
 FROM performance_schema.events_transactions_summary_global_by_
event_name\G
*************************** 1. row ***************************
 EVENT_NAME: transaction
 COUNT_STAR: 1274
 SUM_TIMER_WAIT: 13091950115512576
 MIN_TIMER_WAIT: 7293440
 AVG_TIMER_WAIT: 10276255661056
 MAX_TIMER_WAIT: 11777025727144832
 COUNT_READ_WRITE: 1273
SUM_TIMER_READ_WRITE: 13078918924805888
MIN_TIMER_READ_WRITE: 7293440
AVG_TIMER_READ_WRITE: 10274091697408
MAX_TIMER_READ_WRITE: 11777025727144832
 COUNT_READ_ONLY: 1
 SUM_TIMER_READ_ONLY: 13031190706688
 MIN_TIMER_READ_ONLY: 13031190706688
 AVG_TIMER_READ_ONLY: 13031190706688
 MAX_TIMER_READ_ONLY: 13031190706688
1 row in set (0.0005 sec)
```



当您研究输出有多少事务（尤其是读）时，您可能会感到惊讶。请记住，在查询 InnoDB 表时，即使您未显式指定一个表，一切都是事务。因此，即使是查询单个行的简单语句也算作事务。关于读写事务和只读事务之间的分布，则性能架构仅在显式启动事务时才考虑事务只读：

仅开始事务读取;

当 InnoDB 确定自动提交单语句事务可以被视为只读事务时，该事务仍计入性能架构中的读写统计信息。

## 总结

事务是数据库中的一个重要概念。它们有助于确保您可以将更改作为一个单元应用于多行，并且可以选择是应用更改还是回滚更改。

本章开始讨论为什么了解交易是如何被使用的很重要。虽然它们可以被视为更改的容器，但锁将保留到提交或回滚事务，并且它们可以阻止撤消日志被清除。即使查询未在其中一个事务中执行，也会影响查询和大型撤消日志的性能，从而导致大量锁或大量撤消日志。Locks 使用从缓冲池获取的内存，因此可用于缓存数据和索引的内存较少。以历史记录列表长度衡量的大量撤消日志意味着在 InnoDB 执行语句时必须考虑更多的行版本。

本章的其余部分将介绍如何分析正在进行的和过去的事务。信息中的"信息架构"表是正在进行的事务的最佳信息来源。InnoDB 监视器和 InnoDB 指标补充了这一点。对于使用保存点的 XA 事务和事务，或者当您需要调查哪些语句作为事务的一部分执行时，您需要使用性能架构事务事件表。性能架构还包括汇总表，可用于获取有关谁花时间在读写和只读事务上的信息。

锁在交易讨论中扮演了重要的角色。下一章将介绍如何分析一系列锁问题。