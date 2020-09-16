# Information Schema

当您需要优化查询时，通常需要有关查询、索引等的信息。在这种情况下，信息架构是数据的好资源。本章介绍信息架构以及它包含的视图概述。在书的其余部分中，信息架构多次使用。

## Information Schema是什么?

information数据库是几个关系数据库（包括 MySQL）的通用架构，它添加到 MySQL 5.0 中。MySQL 主要遵循 *F021 基本信息架构*的 SQL:2003 标准，并进行了必要的更改，以反映 MySQL 的附加功能，以及非标准一部分的其他视图。

------

**注意** 在没有数据存储的意义上，information schema是虚拟的。 因此，即使SHOW CREATE TABLE将其显示为常规表，本章也将所有视图和表都称为视图。 这也与对所有对象将表类型设置为SYSTEM VIEW的information_schema.TABLES视图一致

------

在 MySQL 5.5 中引入 Performance Schema 后，目标是使相对静态数据（如通过信息信息和属于性能架构的更易失数据）提供。也就是说，索引统计信息相对不稳定，但也是架构信息的一部分，这并不总是很清楚的。还有一些信息，如InnoDB指标，由于历史原因，这些指标仍然存在于信息架构中。

因此，您可以将信息架构视为描述 MySQL 实例的数据集合。在具有关系数据字典的 MySQL 8 中，一些视图是基础数据字典表上的简单 SQL 视图。这意味着 MySQL 8 中许多信息架构查询的性能将远远优于旧版本中可能体验到的性能。当查询不需要从存储引擎检索信息的架构数据时，尤其如此。

------

**注意** 如果您仍在使用 MySQL 5.7 或更早版本，请小心查询，例如信息架构中的表和列列视图。MySQL 服务器团队在博客中讨论的 MySQL 5.7 和 8 之间的信息架构性能差异示例：https://mysqlserverteam.com/mysql-8-0-scaling-and-performance-of-information_schema/。

------



## 特权

Information Schema是一个虚拟数据库，对视图的访问与其他表的工作方式有点不同。所有用户将看到架构information_schema，并且将看到所有视图。但是，查询视图的结果取决于分配给帐户的级别。例如，没有比全局使用权限具有其他权限的帐户只会在查询"信息架构"information_schema。表视图。

某些视图需要其他权限，在这种情况下，ER_SPECIFIC_ACCESS_DENIED_ERROR（错误编号 1227）错误，并说明缺少哪个权限。例如，INNODB_METRICS需要 Process 权限，因此，如果没有 Process 权限的一个用户查询该视图，则发生以下错误：

```
mysql> SELECT *
 FROM information_schema.INNODB_METRICS;
ERROR: 1227: Access denied; you need (at least one of) the PROCESS
privilege(s) for this operation
```

现在，是时候看看您可以在Information Schema视图中找到什么样的信息了。

## Views

 Information Schema中提供的数据范围从有关系统高级信息到低级 InnoDB 指标。本节概述了视图，但不会详细讨论，因为从性能调优角度讨论最重要的视图，请稍后各章的相关部分讨论这些视图。

------

**注意** 某些插件会向 Information Schema添加自己的视图。此处不考虑外页视图。

------

### 系统信息

 Information Schema中提供的最高级别信息与整个 MySQL 实例有关。这包括哪些字符集可用以及安装了哪些插件等信息。

表 7-1 总结了包含系统信息的视图

| 视图名                                 | 描述信息                                                     |
| -------------------------------------- | ------------------------------------------------------------ |
| CHARACTER_SETS                         | 可用的字符集。                                               |
| COLLATIONS                             | 每个字符集可用的排序规则。 这包括排序规则的ID，在某些情况下（例如，在二进制日志中），该ID用于唯一指定排序规则和字符集。 |
| COLLATION_CHARACTER_ SET_APPLICABILITY | 归类到字符集的映射（与COLLATIONS的前两列相同）。             |
| ENGINES                                | 已知的存储引擎以及是否已加载。                               |
| INNODB_FT_DEFAULT_ STOPWORD            | 在InnoDB表上创建全文索引时使用的默认停用词的列表。           |
| KEYWORDS                               | MySQL中关键字的列表以及是否保留关键字                        |
| PLUGINS                                | MySQL已知的插件，包括状态                                    |
| RESOURCE_GROUPS                        | 线程用于执行工作的资源组。 资源组指定线程的优先级以及它可以使用的CPU。 |
| ST_SPATIAL_REFERENCE_ SYSTEMS          | 空间参考系统列表，包括SRS_ID列，该列包含用于指定空间列参考系统的ID。 |

与系统相关的视图主要用作参考视图，而RESOURCE_GROUPS表则有所不同，因为可以添加资源组，这将在第17章中进行讨论。

例如，在测试升级时，KEYWORDS 视图很有用，因为您可以使用它来验证任何架构、表、列、例程或参数名称是否与新版本中的关键字匹配。如果是这种情况，则需要更新应用程序以引用标识符，如果情况不是这样。要查找与关键字匹配的所有列名：

```sql
SELECT TABLE_SCHEMA, TABLE_NAME,
 COLUMN_NAME, RESERVED
 FROM information_schema.COLUMNS
 INNER JOIN information_schema.KEYWORDS
 ON KEYWORDS.WORD = COLUMNS.COLUMN_NAME
 WHERE TABLE_SCHEMA NOT IN ('mysql',
 'information_schema',
'performance_schema',
'sys'
 )sql
 ORDER BY TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME;
```

该查询使用COLUMNS视图查找除系统模式以外的所有列名（如果您在应用程序或脚本中使用了它们，则可以选择包括这些列名）。 COLUMNS视图是描述模式对象的几个视图之一。



### Schema Information

包含有关架构对象的信息的视图是信息架构中最有用的视图之一。这些也是几个 SHOW 语句的来源。您可以使用视图查找从存储例程的参数到数据库名称的所有内容的信息。包含架构信息的视图在表 7-2 中汇总。

| 视图名                   | 描述信息 |
| ------------------------ | -------- |
| CHECK_CONSTRAINTS        |          |
| COLUMN_STATISTICS        |          |
| COLUMNS                  |          |
| EVENTS                   |          |
| FILES                    |          |
| INNODB_COLUMNS           |          |
| INNODB_DATAFILES         |          |
| INNODB_FIELDS            |          |
| INNODB_FOREIGN           |          |
| INNODB_FOREIGN_COLS      |          |
| INNODB_FT_BEING_DELETED  |          |
| INNODB_FT_CONFIG         |          |
| INNODB_FT_DELETED        |          |
| INNODB_FT_INDEX_CACHE    |          |
| INNODB_FT_INDEX_TABLE    |          |
| INNODB_INDEXES           |          |
| INNODB_TABLES            |          |
| INNODB_TABLESPACES       |          |
| INNODB_TABLESPACES_BRIEF |          |
| INNODB_TABLESTATS        |          |
| INNODB_TEMP_TABLE_INFO   |          |
| INNODB_VIRTUAL           |          |
| KEY_COLUMN_USAGE         |          |
| PARAMETERS               |          |
| PARTITIONS               |          |
| REFERENTIAL_CONSTRAINTS  |          |
| ROUTINES                 |          |
| SCHEMATA                 |          |
| ST_GEOMETRY_COLUMNS      |          |
| STATISTICS               |          |
| TABLE_CONSTRAINTS        |          |
| TABLES                   |          |
| TABLESPACES              |          |
| TRIGGERS                 |          |
| VIEW_ROUTINE_USAGE       |          |
| VIEW_TABLE_USAGE         |          |
| VIEWS                    |          |

几个视图密切相关，例如，列在表中，在架构中，约束引用表和列。这意味着一些列名存在于几个视图中。与这些视图相关的最常用的列名是

- **TABLE_NAME**：在非特定于InnoDB的视图中用于表名。
- **TABLE_SCHEMA**：用于非特定于InnoDB的视图中的架构名称。
- **COLUMN_NAME**：在非特定于InnoDB的视图中用于列名。
- **SPACE**：在特定于InnoDB的视图中用于表空间ID。
- **TABLE_ID**：在特定于InnoDB的视图中用于唯一标识表。 这也在InnoDB内部使用。
- **Name**：InnoDB特定的视图使用名为NAME的列来提供对象名称，而与对象类型无关。

除了使用此列表中的名称外，还有一些示例，这些列名称被稍微修改，就像在视图 KEY_COLUMN_USAGE 中一样，其中您查找了用于外键描述的列 REFERENCED_TABLE_SCHEMA、REFERENCED_TABLE_NAME、andREFERENCED_COLUMN_NAME。例如，如果要使用 KEY_COLUMN_USAGE 视图查找引用 sakila.film 表的带其他键的表，可以使用如下查询：

```sql
mysql> SELECT TABLE_SCHEMA, TABLE_NAME
 FROM information_schema.KEY_COLUMN_USAGE
 WHERE REFERENCED_TABLE_SCHEMA = 'sakila'
 AND REFERENCED_TABLE_NAME = 'film';
+--------------+---------------+
| TABLE_SCHEMA | TABLE_NAME    |
+--------------+---------------+
| sakila       | film_actor    |
| sakila       | film_category |
| sakila       | inventory     |
+--------------+---------------+
3 rows in set (0.0078 sec)
```

这表明，所有film_actorfilm_category和库存表都有作为父表的外键。例如，如果查看表定义，请film_actor：

```sql
mysql> SHOW CREATE TABLE sakila.film_actor\G
*************************** 1. row ***************************
 Table: film_actor
Create Table: CREATE TABLE `film_actor` (
 `actor_id` smallint(5) unsigned NOT NULL,
 `film_id` smallint(5) unsigned NOT NULL,
 `last_update` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (`actor_id`,`film_id`),
 KEY `idx_fk_film_id` (`film_id`),
 CONSTRAINT `fk_film_actor_actor` FOREIGN KEY (`actor_id`) REFERENCES
`actor` (`actor_id`) ON DELETE RESTRICT ON UPDATE CASCADE,
 CONSTRAINT `fk_film_actor_film` FOREIGN KEY (`film_id`) REFERENCES `film`
(`film_id`) ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.0097 sec)
```

约束fk_film_actor_film在胶片表中film_id列。您可以通过手动执行查询中针对 theKEY_COLUMN_USAGE 视图返回的每个表的查询或创建递归公共表表达式 （CTE） 来将其用作查找外键器的完整链的起点。这是留给读者的练习。

------

**提示** 对于一个示例KEY_COLUMN_USAGE递归共性表表达式中使用该视图来查找外键依赖项的seehttps://mysql.wisborg.dk/tracking-foreign-keys。

------

为了完整，图 7-1 中提供了根据胶片表通过外键显示表的可视化表示形式。

![](../附图/Figure%207-1.jpg)

该图是使用 MySQL 工作台的反向工程功能创建的。

具有特定于 InnoDB 的信息的视图使用 SPACE 和TABLE_IDto标识表空间和表。每个表空间都有一个唯一的 ID，其范围为不同的表空间类型保留。例如，数据字典表空间文件 （<datadir>/mysql.ibd） 的空间 ID 4294967294，临时表空间有 id 4294967293，撤消日志表空间从 4294967279 开始，并声明，用户空间从 1 开始。

包含 InnoDB 全文索引信息的视图很特别，因为它们要求您使用要获取信息的表的名称设置 innodb_ft_aux_table 全局变量。例如，要获取表的全文sakila.film_text配置：

```sql
mysql> SET GLOBAL innodb_ft_aux_table = 'sakila/film_text';
Query OK, 0 rows affected (0.0685 sec)
mysql> SELECT *
 FROM information_schema.INNODB_FT_CONFIG;
+---------------------------+-------+
| KEY                       | VALUE |
+---------------------------+-------+
| optimize_checkpoint_limit | 180   |
| synced_doc_id             | 1002  |
| stopword_table_name       |       |
| use_stopword              | 1     |
+---------------------------+-------+
4 rows in set (0.0009 sec)
```

INNODB_FT_CONFIG视图中的值可能与您有所不同。
InnoDB还包括带有与性能有关的信息的视图。 这些将与其他一些与性能相关的表一起讨论。

### Performance Information

与性能相关的视图组是性能调优中可能使用最多的视图组，与上一组视图的 COLUMN_STATISTICS 和"统计"视图一起使用。表 7-3 中列出了包含性能相关信息的视图。

| 视图名                           | 描述信息 |
| -------------------------------- | -------- |
| INNODB_BUFFER_PAGE               |          |
| INNODB_BUFFER_PAGE_LRU           |          |
| INNODB_BUFFER_POOL_STATS         |          |
| INNODB_CACHED_INDEXES            |          |
| INNODB_CMP                       |          |
| INNODB_CMP_RESET                 |          |
| INNODB_CMP_PER_INDEX             |          |
| INNODB_CMP_PER_INDEX_RESET       |          |
| INNODB_CMPMEM                    |          |
| INNODB_CMPMEM_RESET              |          |
| INNODB_METRICS                   |          |
| INNODB_SESSION_TEMP_ TABLESPACES |          |
| INNODB_TRX                       |          |
| OPTIMIZER_TRACE                  |          |
| PROCESSLIST                      |          |
| PROFILING                        |          |

对于包含 InnoDB 压缩表信息的视图，具有 _RESET 作为后缀的表自上次查询视图以来，将操作和计时统计信息作为增量返回。

该INNODB_METRICS包括类似于全局状态变量但具体到 InnoDB 的指标。指标被分组到子系统（SUBSYSTEM 列），对于每个指标，在 COMMENTcolum 中都有指标的说明。您可以使用全局系统变量启用、禁用和重置指标：

- **innodb_monitor_disable**：禁用一个或多个指标。
- **innodb_monitor_enable**：启用一个或多个指标。
- **innodb_monitor_reset**：为一个或多个指标重置计数器。
- **innodb_monitor_reset_all**：重置所有统计信息，包括一个或多个指标的计数器，最小值和最大值。

可以使用在"统计"列中的当前状态根据需要打开和关闭指标。将指标的名称指定为变量或innodb_monitor_enable变量innodb_monitor_disable，可以使用 % 作为通配符。值都用作影响所有指标的特殊值。清单 7-1 显示了使用匹配 %cpu% 的所有指标（恰好是 cpu 子系统中的指标）的示例。计数器值取决于查询时的工作负荷。

```sql
Listing 7-1. Using the INNODB_METRICS view
mysql> SET GLOBAL innodb_monitor_enable = '%cpu%';
Query OK, 0 rows affected (0.0005 sec)
mysql> SELECT NAME, COUNT, MIN_COUNT,
 MAX_COUNT, AVG_COUNT,
 STATUS, COMMENT
 FROM information_schema.INNODB_METRICS
 WHERE NAME LIKE '%cpu%'\G
*************************** 1. row ***************************
 NAME: module_cpu
 COUNT: 0
MIN_COUNT: NULL
MAX_COUNT: NULL
AVG_COUNT: 0
 STATUS: enabled
 COMMENT: CPU counters reflecting current usage of CPU
*************************** 2. row ***************************
 NAME: cpu_utime_abs
 COUNT: 51
MIN_COUNT: 0
MAX_COUNT: 51
AVG_COUNT: 0.4358974358974359
 STATUS: enabled
 COMMENT: Total CPU user time spent
*************************** 3. row ***************************
 NAME: cpu_stime_abs
 COUNT: 7
MIN_COUNT: 0
MAX_COUNT: 7
AVG_COUNT: 0.05982905982905983
 STATUS: enabled
 COMMENT: Total CPU system time spent
*************************** 4. row ***************************
 NAME: cpu_utime_pct
 COUNT: 6
MIN_COUNT: 0
MAX_COUNT: 6
AVG_COUNT: 0.05128205128205128
 STATUS: enabled
 COMMENT: Relative CPU user time spent
*************************** 5. row ***************************
 NAME: cpu_stime_pct
 COUNT: 0
MIN_COUNT: 0
MAX_COUNT: 0
AVG_COUNT: 0
 STATUS: enabled
 COMMENT: Relative CPU system time spent
*************************** 6. row ***************************
 NAME: cpu_n
 COUNT: 8
MIN_COUNT: 8
MAX_COUNT: 8
AVG_COUNT: 0.06837606837606838
 STATUS: enabled
 COMMENT: Number of cpus
6 rows in set (0.0011 sec)
mysql> SET GLOBAL innodb_monitor_disable = '%cpu%';
Query OK, 0 rows affected (0.0004 sec)
```

首先，使用变量的innodb_monitor_enable指标;然后检索值。除了显示的值外，还有一组列与 _RESET 后缀，当您设置 innodb_monitor_reset（仅计数器）或innodb_monitor_reset_all系统变量时重置。最后，指标再次被确定。

------

**注意** 指标的开销各不相同，因此建议您在生产中启用指标之前使用工作负载进行测试。

------

InnoDB度量标准还与全局状态变量和一些其他度量标准以及何时检索这些度量标准一起包含在sys.metrics视图中。
其余的Information Schema视图包含有关特权的信息。

### Privilege Information

MySQL 使用分配给帐户的权限来确定哪些帐户可以访问哪些架构、表和列。确定给定帐户权限的常见方式是使用 SHOW GRANTS 语句，但信息架构还包括允许您查询权限的视图。

信息架构特权视图汇总在表 7-4 中。视图从全局权限排序到列特权。

| 表名              | 描述信息 |
| ----------------- | -------- |
| USER_PRIVILEGES   |          |
| SCHEMA_PRIVILEGES |          |
| TABLE_PRIVILEGES  |          |
| COLUMN_PRIVILEGES |          |

在所有视图中，帐户称为 GRANTEE，并且格式为"用户名"@hostname，报价始终存在。清单7-2显示了一个示例，用于检索mysql.sys@localhost帐户的权限并将其与 SHOWGRANTS 语句的输出进行比较。

```sql
Listing 7-2. Using the Information Schema privilege views
mysql> SHOW GRANTS FOR 'mysql.sys'@'localhost'\G
*************************** 1. row ***************************
Grants for mysql.sys@localhost: GRANT USAGE ON *.* TO `mysql.
sys`@`localhost`
*************************** 2. row ***************************
Grants for mysql.sys@localhost: GRANT TRIGGER ON `sys`.* TO `mysql.
sys`@`localhost`
*************************** 3. row ***************************
Grants for mysql.sys@localhost: GRANT SELECT ON `sys`.`sys_config` TO
`mysql.sys`@`localhost`
3 rows in set (0.2837 sec)
mysql> SELECT *
 FROM information_schema.USER_PRIVILEGES
 WHERE GRANTEE = '''mysql.sys''@''localhost'''\G
*************************** 1. row ***************************
 GRANTEE: 'mysql.sys'@'localhost'
 TABLE_CATALOG: def
PRIVILEGE_TYPE: USAGE
 IS_GRANTABLE: NO
1 row in set (0.0006 sec)
mysql> SELECT *
 FROM information_schema.SCHEMA_PRIVILEGES
 WHERE GRANTEE = '''mysql.sys''@''localhost'''\G
*************************** 1. row ***************************
 GRANTEE: 'mysql.sys'@'localhost'
 TABLE_CATALOG: def
 TABLE_SCHEMA: sys
PRIVILEGE_TYPE: TRIGGER
 IS_GRANTABLE: NO
1 row in set (0.0005 sec)
mysql> SELECT *
 FROM information_schema.TABLE_PRIVILEGES
 WHERE GRANTEE = '''mysql.sys''@''localhost'''\G
*************************** 1. row ***************************
 GRANTEE: 'mysql.sys'@'localhost'
 TABLE_CATALOG: def
 TABLE_SCHEMA: sys
 TABLE_NAME: sys_config
PRIVILEGE_TYPE: SELECT
 IS_GRANTABLE: NO
1 row in set (0.0005 sec)
mysql> SELECT *
 FROM information_schema.COLUMN_PRIVILEGES
 WHERE GRANTEE = '''mysql.sys''@''localhost'''\G
Empty set (0.0005 sec)
```

请注意，用户名和主机名周围的单引号是如何通过将引号加乘来转义的。

虽然具有特权信息的视图不能直接用于性能调优，但它们对于维护一个稳定的系统非常有用，因为您可以使用它们来轻松识别是否有任何帐户具有它们不需要的权限。

------

**提示** 最佳做法是限制帐户只具有所需的权限，而不需要更多权限。这是确保系统安全的步骤之一。

------

关于信息架构的最后一个主题是如何缓存与索引统计相关的数据。

## Caching of Index Statistics Data

需要了解的一件事是索引统计相关视图（和等效的 SHOW 语句）中的信息的来源。大部分数据来自 MySQL 数据字典。在 MySQL 8 中，数据字典存储在 InnoDB 表中，因此视图只是数据字典顶部的正常 SQL 视图。（例如，您可以尝试执行"显示创建视图"information_schema。获取统计信息视图定义的统计信息。

但是，索引统计信息本身仍然来自存储引擎层，因此查询这些统计信息的成本相对较高。为了提高性能，统计数据字典中缓存了统计数据。您可以控制在 MySQL 刷新缓存之前，统计信息的显示时间。这是用默认为 information_schema_stats_expiry86400 秒（一天）的默认变量完成的。如果将 值设置为 0，则始终从存储引擎获取可用的最新值;如果将值设置为 0，则始终从存储引擎获取可用值。这相当于 MySQL 5.7 行为。可以在全局和会话作用域中同时设置变量，因此，如果您正在调查查看当前统计信息很重要的问题（例如，如果优化器未使用预期索引），则变量可以设置为会话的 0。

------

**提示** 使用information_schema_stats_expiry变量控制索引统计信息在数据字典中缓存的长。这仅适用于显示目的 - 优化器始终使用最新的统计信息。例如information_schema_stats_expiry将 0 设置为 0 来禁用缓存，在调查优化器使用的错误索引的问题时可能很有用。您可以根据需要在全局和会话作用域中更改该值。

------

缓存会影响表 7-5 中列出的列。显示相同数据的 SHOW 语句也会受到影响。

| 视图名     | 列名             | 描述信息 |
| ---------- | ---------------- | -------- |
| STATISTICS | CARDINALITY      |          |
| TABLES     | AUTO_INCREMENT   |          |
|            | AVG_ROW_LENGTH   |          |
|            | CHECKSUM         |          |
|            | CHECK_TIME       |          |
|            | CREATE_TIME      |          |
|            | DATA_FREE        |          |
|            | DATA_LENGTH      |          |
|            | INDEX_LENGTH     |          |
|            | MAX_DATA_ LENGTH |          |
|            | TABLE_ROWS       |          |
|            | UPDATE_TIME      |          |

您可以通过为表执行 ANALYZE 表来强制更新给定表的此数据。

有时查询数据不会更新缓存的数据：

- 当缓存的数据尚未过期时，即刷新时间少于几秒钟前的information_schema_stats_expiry
- 当information_schema_stats_expiry设置为0时
- 当MySQL或InnoDB以只读模式运行时，即启用了其中一种模式read_only，super_read_only，transaction_ read_only或innodb_read_only。
- 当查询还包含来自性能架构的数据时

## 总结

本章首先讨论什么是信息架构以及用户权限的工作方式，来介绍信息架构。本章的其余部分将介绍标准视图和缓存的工作原理。信息架构视图可以按其包含的信息类型进行分组：系统、架构、性能和权限信息。

系统信息包括字符集和排序规则、资源组、关键字和与空间数据相关的信息。这可用于使用参考手册的替代方案。

架构信息是最大的视图组，包括从架构数据到列、索引和约束的所有可用信息。这些视图以及具有指标和 InnoDB 缓冲池统计信息等信息的性能视图是性能调整中最常用的视图。与特权相关的视图并不常用于性能调整，但它们对于帮助维护一个稳定的系统非常有用。

从信息架构视图获取信息的常见快捷方式是使用 SHOW 语句。下一章将讨论这些内容。