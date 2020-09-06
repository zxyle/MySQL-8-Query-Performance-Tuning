# show语句

SHOW 语句是 MySQL 中用于数据库管理员获取有关架构对象和系统上发生的情况的信息的好老主力。虽然今天大部分信息都可以在信息架构或性能信息中找到，但由于其短裤，SHOW 命令仍然非常流行，适合交互式使用。

**提示建议查询基础信息架构视图和性能架构表。这尤其适用于对数据的非交互式访问。查询基础源也更强大，因为它允许您联接到其他视图和表。**

本章首先概述 SHOW 语句如何与信息架构视图和性能架构表匹配。课程的其余部分包括信息信息学和性能架构中没有视图或表的 SHOW 语句，包括通过 SHOW ENGINEINNODB 状态语句提供的 InnoDB 监视器输出的更深入的视图获取引擎状态信息，以及获取复制和二进制日志信息。

## Relationship to the Information Schema

对于返回有关架构对象或特权的信息的 SHOW 语句，可以在信息架构中找到相同的信息。表 8-1 列出了从信息架构视图中获取信息的SHOW语句，以及可以在其中找到哪些视图。

在 SHOW 语句和应答信息架构视图之间，信息并不总是相同的。在某些情况下，使用视图可以获得更多信息，并且一般情况下视图更灵活。

还有几个 SHOW 语句，其中的基础数据可以在性能架构中找到。

## Relationship to the Performance Schema

引入性能架构后，信息架构中原始放置的一些信息已移动到性能架构，而性能架构在逻辑上属于该架构。这也反映在与 SHOW 声明的关系中，其中现在有多个表，如表 8-2 所示，这些表从性能架构表中获取数据。

SHOW 主状态包括有关在将事件编写到二进制日志时启用筛选的信息。此信息无法从性能信息中获得，因此，如果您使用的是 binlog-do-db 或 binlog-ignore-db 选项（因为它们可以防止时间点恢复），则仍然需要使用SHOW MASTER 状态。

在"显示从属状态"输出中，在性能架构表中找不到几列。其中一些可以在 mysql 架构slave_master_infoand slave_relay_log_info的表（如果master_info_repository andrelay_log_info_repository已设置为默认的 TABLE）。

对于"显示状态"和"显示变量"，一个区别是，如果没有会话值，SHOW 语句返回会话范围值将包括全局值。查询session_statussession_variables时，仅返回属于请求范围的值。此外，SHOW STATUS 语句包括 Com_% 计数器，而直接查询性能架构时，这些计数器对应于 events_statements_summary_global_by_event_name 和 events_statements_summary_by_thread_by_event_name 表中的事件（取决于是否查询全局或会话范围）。

还有一些 SHOW 语句没有任何相应的表。将讨论的第一组是引擎状态。



## Engine Status

SHOW ENGINE 语句可用于获取存储引擎的特定信息。它目前为 InnoDB、Performance_Schema和 NDBCluster 引擎实现。对于所有三个引擎，可以请求状态，对于 InnoDB 引擎，也有可能获取互斥信息。

SHOW ENGINE PERFORMANCE_SCHEMA状态语句可用于获取有关性能架构的一些状态信息，包括表的大小及其内存使用情况。（内存使用情况也可以从内存图获得。

到目前为止，使用最多的引擎状态声明是 SHOW 引擎 INNODB 状态，它提供了一份称为 InnoDB 监视器报告的全面报告，其中包括无法从其他来源获取的一些信息。本节的其余部分将介绍 InnoDB 监视器报告。

**提示 还可以通过启用系统变量，使 InnoDB 以定期间隔将监视器报告innodb_status_output日志。设置innodb_status_output_locks时，InnoDB 监视器（无论是由于 innodb_status_output = ON 或使用 SHOW ENGINEINNODB 状态）都包含其他锁信息。**

InnoDB 监视器报告以标题开头，并注释显示平均值涵盖的时间长：

```
mysql> SHOW ENGINE INNODB STATUS\G
*************************** 1. row ***************************
 Type: InnoDB
 Name:
Status:
=====================================
2019-09-14 19:52:40 0x6480 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 59 seconds
```

报告本身分为几个部分，包括

- 背景线程：由主后台线程完成的工作。
- 信号量：信号量统计。在争用导致长信号量等待的情况下，该节是最重要的，在这种情况下，该节可用于获取有关锁和谁持有锁的信息。
- 最新外键错误：如果已对外键错误进行计数，本节将包含该错误的详细信息。否则，将省略这些词。
- 最新检测到的死锁：如果发生死锁，本节将包含导致死锁的两个事务和锁的详细信息。否则，将省略该节。
- 事务：有关 InnoDB 事务的信息。仅包含已修改 InnoDB 表的交易。如果theinnodb_status_output_locks，则列出每个事务的锁;如果启用了"theinnodb_status_output_locks，则列出每个事务的锁。如果启用了该选项，则列出每个事务的锁。否则， 它只是锁涉及锁等待。一般来说，最好使用information_schema。INNODB_TRX查询事务信息和用于锁定信息以使用performance_schema。DATA_LOCKS andperformance_schema.DATA_LOCK_WAITS表。
- 文件 I/O：有关 InnoDBind 使用的 I/O 线程的信息，包括插入缓冲区线程、日志线程、读取线程和写入线程。
- 插入缓冲区和自适应哈希索引：有关更改缓冲区（这以前称为插入缓冲区）和适应哈希索引的信息。
- 日志：有关重做日志的信息。
- 缓冲区池和内存：有关 InnoDB 缓冲池的信息。此信息最好从information_schema。INNODB_BUFFER_POOL_STATS视图。
- 个人缓冲池信息：innodb_buffer_pool_instancesis大于 1，本节包含有关个人缓冲池实例的信息，其信息与上一节中的全球摘要的信息相同。否则，将包含该部分。此信息最好从information_schema。INNODB_BUFFER_POOL_STATS视图。
- 行操作：本节显示有关InnoDB的各种信息，包括当前活动、主线程正在做什么以及插入、更新、删除和读取的行活动。

当几个部分的内容用于分析性能或锁定问题时，将在后几章中使用。


## Replication and Binary Logs

在使用复制时，SHOW 语句始终很重要。虽然性能架构复制表现在基本上取代了 SHOW SLAVESTATUS 和 SHOW 主状态语句，但如果您想要查看连接哪些副本并检查来自 MySQL 内部的二进制日志或中继日志中的事件，则仍然需要使用 SHOW 语句。

### Listing Binary Logs

SHOW BINARY LOGS 语句可用于检查存在哪些二进制日志。如果您想知道二进制日志占用的空间、它们是否加密，以及基于位置的复制副本是否仍然存在，这可能非常用。

输出看起来像一个例子

```
mysql> SHOW BINARY LOGS;
+---------------+-----------+-----------+
| Log_name      | File_size | Encrypted |
+---------------+-----------+-----------+
| binlog.000044 | 2616      | No        |
| binlog.000045 | 886       | No        |
| binlog.000046 | 218       | No        |
| binlog.000047 | 218       | No        |
| binlog.000048 | 218       | No        |
| binlog.000049 | 575       | No        |
+---------------+-----------+-----------+
6 rows in set (0.0018 sec)
```

MySQL 8.0.14中添加了Encrypted列，并支持加密的二进制日志。

通常，文件大小将大于示例中，因为在编写事务后，当大小超过 max_binlog_size（默认为 1 GiB）时，将自动修改二年日志文件。由于事务不会在文件之间拆分，因此如果您有大型事务，则文件可能会变得比文件max_binlog_size。



### Viewing Log Events

显示 BINLOG 事件和显示 RELAYLOG 事件语句分别读取二进制 logand 中继日志，并返回与参数匹配的事件。有四个限制，其中一个仅适用于中继日志事件：

- IN：要从中读取事件的二进制日志或中继日志文件的名称。
- FROM：以字节为单位开始读取的位置。
- LIMIT（事件）：要包括的事件数，带有可选的偏移量。 语法与SELECT语句相同：[offset]，row_count。
- FOR CHANNEL：对于中继日志，是要为其读取事件的复制通道。

所有参数都是可选的。如果未给出 IN 参数，则返回第一个日志中的事件。使用 SHOW BINLOG 事件的示例列在清单 8-1 中。如果您希望尝试该示例，则需要替换二进制日志文件名、位置和限制。

```
Listing 8-1. Using SHOW BINLOG EVENTS
mysql> SHOW BINLOG EVENTS IN 'binlog.000049' FROM 195 LIMIT 5\G
*************************** 1. row ***************************
 Log_name: binlog.000049
 Pos: 195
 Event_type: Gtid
 Server_id: 1
End_log_pos: 274
 Info: SET @@SESSION.GTID_NEXT= '4d22b3e5-a54f-11e9-8bdb-ace2d35785be:603'
*************************** 2. row ***************************
 Log_name: binlog.000049
 Pos: 274
 Event_type: Query
 Server_id: 1
End_log_pos: 372
 Info: BEGIN
*************************** 3. row ***************************
 Log_name: binlog.000049
 Pos: 372
 Event_type: Table_map
 Server_id: 1
End_log_pos: 436
 Info: table_id: 89 (world.city)
*************************** 4. row ***************************
 Log_name: binlog.000049
 Pos: 436
 Event_type: Update_rows
 Server_id: 1
End_log_pos: 544
 Info: table_id: 89 flags: STMT_END_F
*************************** 5. row ***************************
 Log_name: binlog.000049
 Pos: 544
 Event_type: Xid
 Server_id: 1
End_log_pos: 575
 Info: COMMIT /* xid=44 */
5 rows in set (0.0632 sec)
```

该示例说明了使用 SHOW 语句检查二次日志和中继日志的一些限制。结果是来自查询的正常结果集，并且由于文件大小通常约为 1 GiB，这意味着结果可以同样大。您可以在示例中执行仅选择特定事件，但了解有趣的事件从该开始位置并不总是微不足道的，并且无法按事件类型或它们影响哪些表进行筛选。最后，默认事件格式（binlog_format 选项）是row格式，从结果中的第三行和第四行可以看到，从 SHOW BINGOG 事件中，您只能看到事务更新了 world.city 表。您无法查看更新了哪些行以及值是什么。

实际上，如果您有权访问文件系统，在大多数情况下最好使用 MySQL 附带的 mysqlbinlog 实用程序。（在受控测试或复制停止时，显示 BINLOG 事件和 SHOWRELAYLOG 事件语句仍然很有用，您快速想要检查导致错误的事件。使用 mysqlbinlog 实用程序到上一个 SHOW BINLOG 事件语句的等效命令显示在清单 8-2 中。该示例还使用详细标志显示更新 world.city 表的基于行的事件的之前和之后的图像。

```
Listing 8-2. Inspecting the binary log using the mysqlbinlog utility
shell> mysqlbinlog -v --base64-output=decode-rows --start-position=195
--stop-position=575 binlog.000049
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 124
#190914 20:38:43 server id 1 end_log_pos 124 CRC32 0x751322a6 Start:
binlog v 4, server v 8.0.18 created 190914 20:38:43 at startup
# Warning: this binlog is either in use or was not closed properly.
ROLLBACK/*!*/;
# at 195
#190915 10:18:45 server id 1 end_log_pos 274 CRC32
0xe1b8b9a1 GTID last_committed=0 sequence_number=1
rbr_only=yes original_committed_timestamp=1568506725779031
immediate_commit_timestamp=1568506725779031 transaction_length=380
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
# original_commit_timestamp=1568506725779031 (2019-09-15 10:18:45.779031
AUS Eastern Standard Time)
# immediate_commit_timestamp=1568506725779031 (2019-09-15 10:18:45.779031
AUS Eastern Standard Time)
/*!80001 SET @@session.original_commit_timestamp=1568506725779031*//*!*/;
/*!80014 SET @@session.original_server_version=80018*//*!*/;
/*!80014 SET @@session.immediate_server_version=80018*//*!*/;
SET @@SESSION.GTID_NEXT= '4d22b3e5-a54f-11e9-8bdb-ace2d35785be:603'/*!*/;
# at 274
#190915 10:18:45 server id 1 end_log_pos 372 CRC32 0x2d716bd5 Query
thread_id=8 exec_time=0 error_code=0
SET TIMESTAMP=1568506725/*!*/;
SET @@session.pseudo_thread_id=8/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0,
@@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1168113696/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_
offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=45,@@session.collation_connection=45,
@@session.collation_server=255/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
BEGIN
/*!*/;
# at 372
#190915 10:18:45 server id 1 end_log_pos 436 CRC32 0xb62c64d7 Table_map:
`world`.`city` mapped to number 89
# at 436
#190915 10:18:45 server id 1 end_log_pos 544 CRC32 0x62687b0b
Update_rows: table id 89 flags: STMT_END_F
### UPDATE `world`.`city`
### WHERE
### @1=130
### @2='Sydney'
### @3='AUS'
### @4='New South Wales'
### @5=3276207
### SET
### @1=130
### @2='Sydney'
### @3='AUS'
### @4='New South Wales'
### @5=3276208
# at 544
#190915 10:18:45 server id 1 end_log_pos 575 CRC32 0x149e2b5c Xid = 44
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_
```

-v 参数请求详细模式，并可最多提供两次以增加包含的信息量。单 -v 是从位置 436 开始的事件中使用伪查询生成注释的函数。--base64 输出=解码行参数告诉 mysqlbinlog 不要以行格式包含事件的 base64 编码版本。--开始位置和 --停止位置参数指定以字节为单位的开始和停止偏移。

事务中最有趣的事件是从注释# 436 开始的事件，这意味着事件从偏移量 436（以字节为单位）开始。它编写为伪更新语句，其中 WHERE 部分显示更改前的值，更新后 SET 部分显示值。这也称为图像之前和之后。

**注意 如果您使用加密的二进制日志，则不能直接使用 mysqlbinlog 来读取文件。一个选项是使 mysqlbinlog 连接到服务器并读取返回未加密的日志的日志。如果使用加密插件存储keyring_file的另一个选项是使用 Python 或标准 Linux 工具来加密文件。这些方法在https://mysql.wisborg.dk/decrypt-binary-logshttps://mysqlhighavailability.com/howto-manually-decrypt-an-encrypted-binary-log-file/。**



### Show Connected Replicas

另一个有用的命令是请求复制源列出连接到它的所有副本。这可用于在监视工具中自动发现复制拓扑。

列出连接的副本的命令是SHOW SLAVE HOSTS，例如：

```
mysql> SHOW SLAVE HOSTS\G
*************************** 1. row ***************************
 Server_id: 2
 Host: replica.example.com
 Port: 3308
 Master_id: 1
Slave_UUID: 0b072c80-d759-11e9-8423-ace2d35785be
1 row in set (0.0003 sec)
```

如果在执行语句时未连接任何副本，则结果将为空。Server_idMaster_id列是副本和源server_id的可用系统的值。主机是使用"使用"report_host副本的主机名。同样，Port 列是副本的sreport_port值。最后，Slave_UUID列是副本上的 @@global.server_uuid 的值。

剩下的唯一一组 SHOW 语句由各种语句组成，用于获取有关权限、用户、打开的表、警告和错误的信息。



## Miscellaneous Statements

有几个 SHOW 语句很有用，但不适合到目前为止讨论过的任何组。它们可用于列出可用权限、返回帐户的 CREATE USER 语句、列出打开的表以及在执行语句后列出警告 orerors。这些语句总结于表8-3。

杂项 SHOW 语句中最常用的三个语句是"显示创建用户"、"显示授权"和"显示警告"。

"显示创建用户"语句可用于检索帐户的创建用户语句。这对于检查帐户的元数据而不直接查询基础 mysql.user 表非常有用。允许所有用户为当前用户执行语句。例如：

```
mysql> SET print_identified_with_as_hex = ON;
Query OK, 0 rows affected (0.0200 sec)
Table 8-3. Miscellaneous SHOW statements
SHOW Statement Description
PRIVILEGES Lists the available privileges, which context they apply to, and for some
privileges a description of what the privilege controls.
CREATE USER Returns the CREATE USER statement for an account.
GRANTS Lists the assigned privileges for the current account or another account.
OPEN TABLES Lists the tables in the table cache, the number of table locks or lock requests,
and whether the name of the table is locked (happens during DROP TABLE or
RENAME TABLE).
WARNINGS Lists the warnings and errors and if sql_notes is enabled (the default) notes
for the last executed statement.
ERRORS Lists the errors for the last executed statement.
mysql> SHOW CREATE USER CURRENT_USER()\G
*************************** 1. row ***************************
CREATE USER for root@localhost: CREATE USER 'root'@'localhost' IDENTIFIED
WITH 'caching_sha2_password' AS 0x24412430303524377B743F5E176E1A77494F574
D216C41563934064E58364E385372734B77314E43587745314F506F59502E747079664957
776F4948346B526B59467A642F30 REQUIRE NONE PASSWORD EXPIRE DEFAULT ACCOUNT
UNLOCK PASSWORD HISTORY DEFAULT PASSWORD REUSE INTERVAL DEFAULT PASSWORD
REQUIRE CURRENT DEFAULT
1 row in set (0.0003 sec)
```

print_identified_with_as_hex（在 8.0.17 及更晚版本中可用）可以返回十六进制表示法中的密码摘要。这是将值返回到控制台时的首选，因为摘要可能包含不可打印字符。SHOW CREATE 用户输出等效于用户创建的方式，可用于创建具有相同设置（包括密码）的新用户。

**注意：仅在 MySQL 8.0.17 及更晚一些中支持在创建用户时指定十六进制表示法中的身份验证摘要。**

SHOW GRANTS 语句通过返回分配给帐户的"显示创建用户"来补充"显示创建用户"。默认值是返回当前用户，但如果您具有 mysql 系统数据库的 SELECT 权限，您还可以获取分配给其他帐户的特权。例如，要列出该帐户root@localhost：

```
mysql> SHOW GRANTS FOR root@localhost\G
*************************** 1. row ***************************
Grants for root@localhost: GRANT SELECT, INSERT, UPDATE, DELETE, CREATE,
DROP, RELOAD, SHUTDOWN, PROCESS, FILE, REFERENCES, INDEX, ALTER, SHOW
DATABASES, SUPER, CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE,
REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW, SHOW VIEW, CREATE
ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER, CREATE TABLESPACE,
CREATE ROLE, DROP ROLE ON *.* TO `root`@`localhost` WITH GRANT OPTION
*************************** 2. row ***************************
Grants for root@localhost: GRANT APPLICATION_PASSWORD_ADMIN,AUDIT_
ADMIN,BACKUP_ADMIN,BINLOG_ADMIN,BINLOG_ENCRYPTION_ADMIN,CLONE_
ADMIN,CONNECTION_ADMIN,ENCRYPTION_KEY_ADMIN,GROUP_REPLICATION_
ADMIN,INNODB_REDO_LOG_ARCHIVE,PERSIST_RO_VARIABLES_ADMIN,REPLICATION_
APPLIER,REPLICATION_SLAVE_ADMIN,RESOURCE_GROUP_ADMIN,RESOURCE_GROUP_
USER,ROLE_ADMIN,SERVICE_CONNECTION_ADMIN,SESSION_VARIABLES_ADMIN,SET_USER_
ID,SYSTEM_USER,SYSTEM_VARIABLES_ADMIN,TABLE_ENCRYPTION_ADMIN,XA_RECOVER_
ADMIN ON *.* TO `root`@`localhost` WITH GRANT OPTION
*************************** 3. row ***************************
Grants for root@localhost: GRANT PROXY ON “@” TO 'root'@'localhost' WITH
GRANT OPTION
3 rows in set (0.0129 sec)
```

SHOW 警告语句是 MySQL 中使用最未充分利用的语句之一。如果MySQL遇到问题，但能够继续，它将生成一个警告，但另一方面完成语句的执行。虽然语句完成时没有错误，但警告可能是更大问题的迹象，最佳做法是始终检查警告，并力求在应用程序执行的查询中永远不会发出警告。

**注意 MySQL Shell 不支持 SHOW 警告语句，因为如果启用了 \W 模式（默认值），并且不使警告可用，它将自动获取警告。但是，该语句在旧版 mysql 命令行客户端和某些连接器（如 MySQLConnector/Python）中仍然很有用。**

清单 8-3 显示了一个示例，其中 SHOW 警告与旧版 mysqlcommand 线路客户端一起使用，以标识架构定义和数据不匹配。

```
Listing 8-3. Using SHOW WARNINGS to identify problems
mysql> SELECT @@sql_mode\G
*************************** 1. row ***************************
@@sql_mode: ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,
NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
1 row in set (0.0004 sec)
mysql> SET sql_mode = sys.list_drop(
 @@sql_mode,
'STRICT_TRANS_TABLES'
 );
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql> SHOW WARNINGS\G
*************************** 1. row ***************************
 Level: Warning
 Code: 3135
Message: 'NO_ZERO_DATE', 'NO_ZERO_IN_DATE' and 'ERROR_FOR_DIVISION_BY_ZERO'
sql modes should be used with strict mode. They will be merged with strict
mode in a future release.
1 row in set (0.00 sec)
mysql> UPDATE world.city
 SET Population = Population/0
 WHERE ID = 130;
Query OK, 0 rows affected, 2 warnings (0.00 sec)
Rows matched: 1 Changed: 0 Warnings: 2
mysql> SHOW WARNINGS\G
*************************** 1. row ***************************
 Level: Warning
 Code: 1365
Message: Division by 0
*************************** 2. row ***************************
 Level: Warning
 Code: 1048
Message: Column 'Population' cannot be null
2 rows in set (0.00 sec)
mysql> SELECT *
 FROM world.city
 WHERE ID = 130\G
*************************** 1. row ***************************
 ID: 130
 Name: Sydney
CountryCode: AUS
 District: New South Wales
 Population: 0
1 row in set (0.03 sec)
```

该示例从将 SQL 模式设置为 MySQL 8 中的默认值开始。首先，使用 sys.list_drop（） 函数更改 SQLmode 以删除触发警告的 STRICT_TRANS_TABLES 模式，因为禁用严格模式应与其他模式一起完成，因为它们将在以后合并在一起。然后更新世界上一个城市的人口.城市表，但计算结束除以0，这将触发两个警告。一个警告是按 0 划分，未定义，因此 MySQL 使用 NULL 值，该值会导致第二个警告，因为"总体"列是非 NULL 列。结果是，0 的人口与城市是分配的，这可能不是应用程序中的预期。这也解释了为什么启用严格的 SQL 模式很重要，因为这将使分区为零错误并阻止更新。

**注意 不要禁用 STRICT_TRANS_TABLES SQL 模式，因为它使表中的数据更有可能最终出现无效数据。**



## 总结

本章介绍了可追溯到信息信息和性能架构实施前的 SHOW 语句。如今，最好在信息架构和性能架构中使用基础数据源。

还有一些 SHOW 语句返回无法通过其他源访问的数据。常用的功能是 InnoDB 监视器报告，该报告来自 INNODBobo 与 SHOW 引擎 INNODB 状态语句一起包含。报告分为几个部分，其中一些将在调查性能和锁定问题时使用。

还有一些用于复制的语句和有用的二进制日志。其中最常用的语句是 SHOW BINARY LOGS，它列出了 MySQL 为该实例知道的二进制日志。该信息包括日志的大小以及日志是否加密。您还可以在二进制日志或中继日志中列出事件，但在实践中，mysqlbinlog 实用程序通常是更好的选项。

最后，介绍了一组杂项的 SHOW 语句。其中三个使用最多的是"显示创建用户"来显示可用于重新创建用户的语句，返回分配给用户的权限的 SHOW GRANTS 和显示警告，它们列出了错误、警告以及上次执行查询时发生的默认注释。检查警告是执行查询时经常忽略的一个方面，因为警告可能表明查询的结果不是您所期望的。值得称赞的是，始终检查警告并启用STRICT_TRANS_TABLESSQL模式。

关于信息源的最后一章是关于慢查询日志的信息