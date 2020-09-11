# sys schema

sys schema是马克·莱思的创意，他长期以来一直是开发 MySQL 企业监视器的团队的一员。他开始了ps_helper项目，以监控理念进行实验，并展示sys schema可以做什么，同时使sys schema更简单。该项目后来重命名为 thesys 架构并移动到 MySQL 中。此后，包括这本书的作者在内的其他几个人也做出了贡献。

sys schema适用于 MySQL Server 5.6 及更晚版本。在 MySQL 5.7 中，它成为标准安装的一部分，因此您无需执行任何操作来安装系统或升级系统。截至 MySQL 8.0.18，系统架构源代码是 MySQL 服务器源的一部分。

系统架构在整本书中用于分析查询、锁等。本章将介绍 sys 架构的高级别概述，包括如何配置它、格式化函数、视图如何工作以及各种帮助程序例程。

------

**提示** sys 架构源代码 （https://github.com/mysql/mysqlserver/tree/8.0/scripts/sys_schema 和较旧的 MySQL versionshttps://github.com/mysql/mysql-sys/） 也是了解如何根据性能架构编写查询的有用资源。

------



## sys Schema配置

sys 架构使用自己的配置系统，因为它最初是独立于 MySQL 服务器实现的。有两种方法可以更改配置，取决于是永久更改设置还是只更改会话设置。

持久保留的配置存储在sys_config表中，其中包括变量名、其值以及上次设置值和由哪个用户使用。清单 6-1 显示了默认内容（set_time取决于上次安装或升级系统学。

```
Listing 6-1. The sys schema persisted configuration
mysql> SELECT * FROM sys.sys_config\G
*************************** 1. row ***************************
variable: diagnostics.allow_i_s_tables
 value: OFF
set_time: 2019-07-13 19:19:29
 set_by: NULL
*************************** 2. row ***************************
variable: diagnostics.include_raw
 value: OFF
set_time: 2019-07-13 19:19:29
 set_by: NULL
*************************** 3. row ***************************
variable: ps_thread_trx_info.max_length
 value: 65535
set_time: 2019-07-13 19:19:29
 set_by: NULL
*************************** 4. row ***************************
variable: statement_performance_analyzer.limit
 value: 100
set_time: 2019-07-13 19:19:29
 set_by: NULL
*************************** 5. row ***************************
variable: statement_performance_analyzer.view
 value: NULL
set_time: 2019-07-13 19:19:29
 set_by: NULL
*************************** 6. row ***************************
variable: statement_truncate_len
 value: 64
set_time: 2019-07-13 19:19:29
 set_by: NULL
6 rows in set (0.0005 sec)
```

目前，set_by始终为 NULL，除非 @sys.ignore_sys_config_触发器用户变量设置为计算为 FALSE 但不是 NULL 的值。

您最有可能更改的选项是 statement_truncate_len 它指定 sys 架构用于格式化视图中语句的最大长度（稍后将详细说明这些）。选择默认值 64 可增加查询视图适合控制台宽度的概率;但是，有时获取有关该语句的有用信息太少。

您可以通过更新配置设置来更新sys_config。这将保留更改并立即应用于所有连接，除非它们已设置自己的会话值（当使用设置语句的 sysschema 中的东西时，将隐式地发生）。由于sys_config是一个正常的 InnoDB 表，因此在重新启动 MySQL 后，更改也将保留。

或者，您可以只更改会话的设置。这是通过使用配置变量的名称和预用系统完成的。并把它变成一个用户可变。清单 6-2 显示了使用 sys_config 表和用户变量来更改statement_truncate_len。结果使用 format_statement（） 函数进行测试，该函数是 sys 架构用于截断声明的函数。

```
Listing 6-2. Changing the sys schema configuration
mysql> SET @query = 'SELECT * FROM world.city INNER JOIN world.city ON
country.Code = city.CountryCode';
Query OK, 0 rows affected (0.0003 sec)
mysql> SELECT sys.sys_get_config(
 'statement_truncate_len',
 NULL
 ) AS TruncateLen\G

*************************** 1. row ***************************
TruncateLen: 64
1 row in set (0.0007 sec)
mysql> SELECT sys.format_statement(@query) AS Statement\G
*************************** 1. row ***************************
Statement: SELECT * FROM world.city INNER ... ountry.Code = city.CountryCode
1 row in set (0.0019 sec)
mysql> UPDATE sys.sys_config SET value = 48 WHERE variable = 'statement_
truncate_len';
Query OK, 1 row affected (0.4966 sec)
mysql> SET @sys.statement_truncate_len = NULL;
Query OK, 0 rows affected (0.0004 sec)
mysql> SELECT sys.format_statement(@query) AS Statement\G
*************************** 1. row ***************************
Statement: SELECT * FROM world.ci ... ode = city.CountryCode
1 row in set (0.0009 sec)
mysql> SET @sys.statement_truncate_len = 96;
Query OK, 0 rows affected (0.0003 sec)
mysql> SELECT sys.format_statement(@query) AS Statement\G
*************************** 1. row ***************************
Statement: SELECT * FROM world.city INNER JOIN world.city ON country.Code =
city.CountryCode
1 row in set (0.0266 sec)
```

首先，在用户变量的@query查询。这纯粹是为了方便起见，所以很容易继续引用相同的查询。sys_get_configt( ) 函数用于获取当前配置值statement_truncate_len选项。这会说明是否设置了@sys.语句_trauncate_len 用户变量。如果提供的选项不存在，则第二个保证提供要返回的值。

format_statementt( ) 函数用于演示 @query 中的语句格式，首先默认值为 statement_truncate_len 的默认值 64，然后 updatingsys_config 的值为 48，最后将会话的值设置为 96。 请注意，在更新 sys_config 表后，@sys.语句_truncate_len 用户变量如何设置为 NULL，以使 MySQL 应用更新到会话的设置。

注意 默认情况下，某些 sysschema 功能支持一些配置选项，这些sys_config在 sys_config 表中，例如调试选项。sys 架构对象（https://dev.mysql.com/doc/refman/en/sys-schema-reference.html）的文档包括支持哪些配置选项的信息。

format_statement( ) 函数并不是 sysschema 中的唯一格式函数，因此让我们来看看所有这些格式。



## Formatting Functions

sys 架构包含四个函数，可帮助您格式化与性能架构不同的查询输出，使结果更易于阅读或占用更少的空间。由于添加了本机性能学函数以替换它们，因此在 MySQL 8.0.16 中已弃用其中两个函数。

补充表6-1

清单 6-3 显示了使用格式化函数的示例，对于 format_bytes（） 和 format_time（），结果将比较本地性能学函数。

```
Listing 6-3. Using the formatting functions
mysql> SELECT sys.format_bytes(5000) AS SysBytes,
 FORMAT_BYTES(5000) AS P_SBytes\G
*************************** 1. row ***************************
SysBytes: 4.88 KiB
P_SBytes: 4.88 KiB
1 row in set, 1 warning (0.0015 sec)
Note (code 1585): This function 'format_bytes' has the same name as a
native function
mysql> SELECT @@global.datadir AS DataDir,
 sys.format_path(
 'D:\\MySQL\\Data_8.0.18\\ib_logfile0'
 ) AS LogFile0\G
*************************** 1. row ***************************
 DataDir: D:\MySQL\Data_8.0.18\
LogFile0: @@datadir\ib_logfile0
1 row in set (0.0027 sec)
mysql> SELECT sys.format_statement(
 'SELECT * FROM world.city INNER JOIN world.city ON
country.Code = city.CountryCode'
 ) AS Statement\G
*************************** 1. row ***************************
Statement: SELECT * FROM world.city INNER ... ountry.Code = city.CountryCode
1 row in set (0.0016 sec)
mysql> SELECT sys.format_time(123456789012) AS SysTime,
 FORMAT_PICO_TIME(123456789012) AS P_STime\G
*************************** 1. row ***************************
SysTime: 123.46 ms
P_STime: 123.46 ms
1 row in set (0.0006 sec)
```

请注意，使用 sys.format_bytes（） 会触发警告（但只有第一次连接使用它），因为 sys 架构函数名与本机函数名称相同。"format_path）"函数希望在MicrosoftWindows上对路径名称进行反斜杠，在其他平台上向前斜杠。format_statement（） 函数的结果假定statement_truncate_len选项的值已重置为默认值 64。

**提示 虽然 format_time（） 和 format_bytes（） 的 sys 架构实现仍然存在，但最好使用新的本机函数，因为系统架构实现可能会在将来的版本中被删除，并且本机函数的速度要快得多。**

## The Views

sys 架构提供了许多作为预定义报表工作的视图。这些视图通常使用性能架构表，但也有少数视图使用"信息架构"。

由于视图是现成的报表，您可以使用这些报表作为数据库管理员或开发人员，因此它们使用默认顺序进行定义。这意味着使用视图的典型方式是使用纯选择 * 从 [视图名称]，例如：

```
mysql> SELECT *
 FROM sys.schema_tables_with_full_table_scans\G
*************************** 1. row ***************************
 object_schema: world
 object_name: city
rows_full_scanned: 4079
 latency: 269.13 ms

*************************** 2. row ***************************
 object_schema: sys
 object_name: sys_config
rows_full_scanned: 18
 latency: 328.80 ms
2 rows in set (0.0021 sec)
```

结果取决于已使用完整的表扫描的表。请注意，延迟的格式与 FORMAT_PICO_TIME（） 或 sys.format_time（） 函数一样。

大多数系统架构视图以两种形式存在，其中一种具有语句、路径、字节值和设置格式的计时，另一种形式返回原始数据。如果在控制台上查询视图并自己查看数据，格式化视图非常有用，而未格式化的视图在需要处理程序或想要更改默认排序时会更好地工作。MySQL 工作台中的性能报告使用未格式化的视图，因此您可以在用户界面内更改顺序。

您可以区分格式化视图和未格式化视图和名称。如果视图包含格式，则也会有一个同名的未格式化视图，但 x$ 已预加到名称中。例如，对于schema_tables_with_full_table_scans中使用的视图，未格式化的视图schema_tables_with_full_table_scans：

```
mysql> SELECT *
 FROM sys.x$schema_tables_with_full_table_scans\G
*************************** 1. row ***************************
 object_schema: world
 object_name: city
rows_full_scanned: 4079
 latency: 269131954854
*************************** 2. row ***************************
 object_schema: sys
 object_name: sys_config
rows_full_scanned: 18
 latency: 328804286013
2 rows in set (0.0017 sec)
```

sys模式的最后一个主题是提供的帮助程序功能和过程。



## Helper Functions and Procedures

sys 架构提供了多种实用程序，可帮助您使用MySQL时。其中包括执行动态创建的查询、操作列表等功能。表 6-2 中总结了最重要的帮助程序功能和程序。

补充表6-2

其中一些实用程序也在系统架构中内部使用。例程的最常规用途是存储的程序中，您需要动态处理数据和问题。

提示 系统架构函数和过程以例程注释的形式提供内置帮助。您可以通过查询数据ROUTINE_COMMENTcolumn获取information_schema。常规视图。



## 总结

本章简要介绍了系统架构，因此，当您在后几章中看到示例时，您知道它是什么以及如何使用它。sys 架构是一个有用的附加工具，它提供现成的报告和实用程序，可以简化您的日常任务和调查。sys 架构是 MySQL 5.7 及以后的系统架构，因此需要您方执行任何操作才能开始使用它。

首先，讨论了系统架构配置。全局配置存储在可更新sys.sys_config的表中，如果您喜欢与安装 MySQL 时提供的默认值不同的默认值， 则可以更新该表。还可以通过使用 sys 设置用户变量来更改会话的配置选项。前缀到配置选项的名称。

然后，在介绍系统架构格式函数时，提到了已添加本机性能架构函数作为系统架构函数替换的情况。格式函数也用于几个视图，以帮助使数据更易于人类读取。对于使用格式设置功能的视图，还有一个相应的未格式化视图，名称前缀为 x$。

最后，讨论了几种帮助程序的功能和程序。当您尝试动态地执行工作时，例如执行在存储程序中生成的查询，这些操作可以帮助您。

下一章是关于信息模式的