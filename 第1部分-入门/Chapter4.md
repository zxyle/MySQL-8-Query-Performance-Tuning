# 测试数据

测试是性能调优工作中非常重要的一部分，因为在将更改应用到生产系统之前，请确保已确认所做的更改很重要，这一点很重要。 验证变更的最佳数据与生产数据密切相关； 但是，为了探索MySQL的工作方式，最好使用一些通用的测试数据。 本章介绍了四个带有安装说明的标准数据集，以及一些可用的其他数据集。

------

**提示** 本书的其余部分将使用world数据库，world_x数据库和sakila数据库作为测试数据。

------

但是，首先，您需要知道如何下载数据库。

## 下载示例数据库

本章将详细讨论的示例数据库的共同点是可以从https://dev.mysql.com/doc/index-other.html下载它们，或者有指向可以从中下载它们的链接。 对于其中的一些数据库，此页面还链接了联机文档和PDF文件。 该页面的相关部分如图4-1所示。

![](../附图/Figure%204-1.png)

员工数据（员工数据库）是从Giuseppe Maxia的GitHub存储库下载的，而其他数据库是从Oracle MySQL站点下载的。 包含员工数据的下载内容还包括sakila数据库的副本。 对于员工数据，world数据库和sakila数据库，还有可用的文档。

------

**注意** 如果您没有使用最新版本的数据，则在安装测试数据库时，可能会看到有关不推荐使用的功能的警告。 您可以忽略这些警告，但是建议获取最新版本的数据。

------

menagerie数据库是一个很小的两表数据库，总共少于20行，这是针对MySQL手册中的tutorials部分创建的。 不再赘述。

## world数据库

world样本数据库是用于简单测试的最常用数据库之一。 它由具有几百到几千行的三个表组成。 这使其成为一个小的数据集，这意味着即使在小型测试实例上也可以轻松使用它。

### 模式

该数据库由city，country和countrylanguage组成。 表之间的关系如图4-2所示。

![](../附图/Figure%204-2.jpg)

国家地区表包括有关 239 个国家/地区的信息，并在城市和国家表。数据库中共有4079个城市，国家语言组合有984。

country表包含有关239个国家/地区的信息，并用作来自city表和countrylanguage表的外键中的父表。 数据库中共有4079个城市，并且984个国家和语言组合。

### 安装

下载的文件包含一个名为world.sql.gz或world.sql.zip的文件，具体取决于您选择的是Gzip还是Zip链接。 无论哪种情况，下载的归档文件都包含一个文件world.sql。 数据的安装非常简单，因为只需执行脚本即可。

如果您在2020年1月左右或之前将MySQL Shell与world数据库的副本一起使用，则需要使用传统协议，因为X协议（默认）需要UTF-8，而world数据库则使用Latin 1。 \source命令从MySQL Shell加载数据：

```
MySQL [localhost ssl] SQL> \source world.sql
```

如果使用旧版命令行客户端，请改为命令：

```
mysql> SOURCE world.sql
```

在这两种情况下，如果位于启动 MySQL Shell 或 mysql 的目录中，请将其添加到。

在任何一种情况下，如果该路径都不位于启动MySQL Shell或mysql的目录中，则将其添加到world.sql文件。

另一个相关的数据库是world_x，它包含与world相同的数据，但是组织方式不同

## world_x数据库

MySQL 8增加了对MySQL文档存储的支持，该文档存储支持以JavaScript对象符号（JSON）文档的形式存储和检索数据。 world_x数据库将一些数据存储在JSON文档中，从而为您提供一个测试数据库，该数据库可以轻松用于包含JSON使用的测试。

### 模式

包含与世界数据库相同的表，尽管列略有不同，例如具有人口（而不是人口的 JSON"信息省略了几列。相反，有一表，它是一个纯文档存储 - 类型表，否则信息将从国家/中删除。图。

world_x数据库包含与world数据库相同的三个表，尽管各列略有不同，例如，city表包含JSON列Info而不是人口列，而People列包含Country列，而country表则省略了几列。 取而代之的是，有一个countryinfo表，它是一个纯文档存储类型的表，其中的信息否则已从country表中删除。 该架构图如图4-3所示。

![](../附图/Figure%204-3.png)

尽管没有来自city和countryinfo表的外键，但是可以分别使用CountryCode列和doc->>'$.Code'值将它们连接到country表。 countryinfo表的_id列是存储的生成列的示例，其中该值是从doc列中的JSON文档中提取的。

### 安装

world_x数据库的安装与world数据库非常相似。 您可以下载world_x-db.tar.gz或world_x-db.zip文件并将其解压缩。 提取的文件包括一个名为world_x.sql的文件以及一个README文件。 world_x.sql文件包含创建模式所需的所有语句。

由于world_x模式使用UTF-8，因此您可以使用任何一种MySQL协议进行安装。 例如，使用MySQL Shell：

```
MySQL [localhost+ ssl] SQL> \source world_x.sql
```

如果该路径不在当前目录中，则将其添加到world_x.sql文件。

world和world_x数据库非常简单，因此易于使用。 但是，有时您会需要sakila数据库可以提供的更复杂的功能。

## sakila数据库

sakila数据库是一个现实的数据库，其中包含电影租赁业务的架构，其中包含有关电影，库存，商店，员工和客户的信息。 它添加了全文索引，空间索引，视图和存储的程序，以提供使用MySQL功能的更完整示例。 数据库大小仍然非常适中，使其适合小型实例。

### 模式

sakila数据库包含16个表，七个视图，三个存储过程，三个存储函数和六个触发器。 这些表可以分为三组：客户数据，业务和库存。 为简便起见，图中未包含所有列，并且未显示大多数索引。 图4-4显示了表，视图和存储例程的完整概述。

![](../附图/Figure%204-4.jpg)

带有客户相关数据的表（加上工作人员和商店的地址）在左上角的区域中。 左下方的区域包含与业务相关的数据，右上方的区域包含有关胶片和库存的信息。 右下角用于视图和存储的例程(stored routines)。

------

**提示** 您可以通过打开MySQL Workbench中的安装随附的sakila.mwb文件来查看整个图（尽管格式不同）。 这也是一个很好的例子，说明了如何在MySQL Workbench中使用增强的实体关系（EER）图来记录架构。

------

由于存在相对大量的对象，因此在讨论架构时，它们将分为五个组（每个表组，视图和存储的例程）。 第一组是与客户相关的数据，其表如图4-5所示。

![](../附图/Figure%204-5.png)

有四个表，其中包含与客户相关的数据。 客户表是主表，地址信息存储在address表，city表和country表中。

客户和业务组之间有外键，外键从customer表到业务组中的store表。 从业务组表到地址表和客户表，还有四个外键。 业务组如图4-6所示。

![](../附图/Figure%204-6.png)

业务表包含有关商店、员工、租金和付款的信息。商店和两个方向都有外键，员工属于一家商店，商店有一名经理，是员工的一部分。租金和付款由工作人员处理，因此与商店间接挂钩，付款用于租赁。

业务表包含有关商店，员工，租金和付款的信息。 store表和staff表在两个方向上都具有外键，其中职员属于商店，并且商店具有作为职员一部分的经理。 租金和付款由工作人员处理，因此间接链接到商店，付款是针对租金的。

![](../附图/Figure%204-7.png)

库存组中的主表是电影表，其中包含有关商店提供的电影的元数据。 此外，还有带有标题和描述以及带有全文索引的film_text表。

电影与类别表和演员表之间存在多对多关系。 最后，从库存表到业务组中的商店表有一个外键。

它涵盖了sakila数据库中的所有表，但也有一些视图，如图4-8所示。

![](../附图/Figure%204-8.png)

这些视图可以像报表一样使用，并且可以分为两类。 film_list，nice_but_slower_film_list和actor_info视图与存储在数据库中的电影有关。 第二类包含与sales_by_store，sales_by_film_category，staff_list和customer_list视图中的商店有关的信息。

为了完成数据库，还有存储的功能和过程，如图4-9所示。

![](../附图/Figure%204-9.png)

film_in_stock( )和film_not_in_stock( )过程根据胶片是否有存货，返回一个结果集，该结果集包含给定电影和商店的库存ID。 找到的库存条目总数作为out参数返回。 rewards_report( )过程根据上个月的最低支出生成报告。

get_customer_balance( )函数返回给定客户在给定数据上的余额。 剩下的两个函数使用库存清单状态来检查库存清单ID的状态，该方法返回当前正在租借该项目的客户的客户编码（如果没有客户在租用，则返回NULL），以及是否要检查给定的库存清单是否有库存 ，您可以使用ventory_in_stock( )函数。

### 安装

下载的文件扩展到包含三个文件的目录中，其中两个创建模式和数据，最后一个文件包含MySQL Workbench使用的格式的ETL图。

------

**注意** 本节和本书后面的示例使用从MySQL主页下载的sakila数据库的副本。

------

文件是

- **sakila-data.sql**：填充表和触发器定义所需的INSERT语句。
- **sakila-schema.sql**：模式定义语句
- **sakila.mwb**：MySQL Workbench ETL图。 这类似于图4-4所示的内容，图4-5至4-9对此进行了详细说明。

通过先采购sakila-schema.sql文件，然后再采购sakila-data.sql文件，来安装sakila数据库。 例如，以下使用MySQL Shell：

```
MySQL [localhost+ ssl] SQL> \source sakila-schema.sql
MySQL [localhost+ ssl] SQL> \source sakila-data.sql
```

如果文件不在当前目录中，则将其添加到文件中。

到目前为止，这三个数据集的共同点是它们包含的数据很少。 尽管在许多情况下这是一个不错的功能，因为它使使用起来更容易，但在某些情况下，您需要更多的数据来探索查询计划的差异。 employees数据库是具有更多数据的选项。

## employees数据库

employees数据库（在MySQL文档下载页面上称为员工数据； GitHub存储库的名称为test_db）最初是由Fusheng Wang和Carlo Zaniolo创建的，并且是MySQL主页中链接的最大的测试数据集。 对于非分区版本，数据文件的总大小约为180 MiB，对于分区版本，数据文件的总大小约为440 MiB。

### 模式

employees数据库由六个表和两个视图组成。 您可以选择另外安装两个视图，五个存储函数和两个存储过程。 这些表如图4-10所示。

![](../附图/Figure%204-10.jpg)

可以选择按from_date列的年份对薪金和职称表进行分区，如清单4-1所示。

```
Listing 4-1. The optional partitioning of the salaries and titles tables
PARTITION BY RANGE COLUMNS(from_date)
(PARTITION p01 VALUES LESS THAN ('1985-12-31') ENGINE = InnoDB,
 PARTITION p02 VALUES LESS THAN ('1986-12-31') ENGINE = InnoDB,
 PARTITION p03 VALUES LESS THAN ('1987-12-31') ENGINE = InnoDB,
 PARTITION p04 VALUES LESS THAN ('1988-12-31') ENGINE = InnoDB,
 PARTITION p05 VALUES LESS THAN ('1989-12-31') ENGINE = InnoDB,
 PARTITION p06 VALUES LESS THAN ('1990-12-31') ENGINE = InnoDB,
 PARTITION p07 VALUES LESS THAN ('1991-12-31') ENGINE = InnoDB,
 PARTITION p08 VALUES LESS THAN ('1992-12-31') ENGINE = InnoDB,
 PARTITION p09 VALUES LESS THAN ('1993-12-31') ENGINE = InnoDB,
 PARTITION p10 VALUES LESS THAN ('1994-12-31') ENGINE = InnoDB,
 PARTITION p11 VALUES LESS THAN ('1995-12-31') ENGINE = InnoDB,
 PARTITION p12 VALUES LESS THAN ('1996-12-31') ENGINE = InnoDB,
 PARTITION p13 VALUES LESS THAN ('1997-12-31') ENGINE = InnoDB,
 PARTITION p14 VALUES LESS THAN ('1998-12-31') ENGINE = InnoDB,
 PARTITION p15 VALUES LESS THAN ('1999-12-31') ENGINE = InnoDB,
 PARTITION p16 VALUES LESS THAN ('2000-12-31') ENGINE = InnoDB,
 PARTITION p17 VALUES LESS THAN ('2001-12-31') ENGINE = InnoDB,
 PARTITION p18 VALUES LESS THAN ('2002-12-31') ENGINE = InnoDB,
 PARTITION p19 VALUES LESS THAN (MAXVALUE) ENGINE = InnoDB)
```

表4-1显示了employees数据库中表的行数和表空间文件的大小（请注意，加载数据时大小可能会有所不同）。 该大小假定您加载了未分区的数据。 分区表有些大。

| 表名         |    行数 | 表空间大小 |
| :----------- | ------: | ---------: |
| departments  |       9 |    128 kiB |
| dept_emp     |  331603 |  25600 kiB |
| dept_manager |      24 |    128 kiB |
| employees    |  300024 |  22528 kiB |
| salaries     | 2844047 | 106496 kiB |
| titles       |  443308 |  27648 kiB |

按照今天的标准，它仍然是相对少量的数据，但是它足够大，您可以开始看到不同查询计划的一些性能差异。

图4-11总结了视图和例程。

![](../附图/Figure%204-11.jpg)

dept_emp_latest_date和current_dept_emp视图与表一起安装，而其余对象则分别安装在objects.sql文件中。 存储的例程带有它们自己的内置帮助，您可以通过使用employee_usage( )函数或employees_help( )过程获得该帮助。 清单4-2中显示了后者。

```
mysql> CALL employees_help()\G
*************************** 1. row ***************************
info:
    == USAGE ==
    ====================
    PROCEDURE show_departments()
        shows the departments with the manager and
        number of employees per department
    FUNCTION current_manager (dept_id)
        Shows who is the manager of a given departmennt
    FUNCTION emp_name (emp_id)
        Shows name and surname of a given employee

    FUNCTION emp_dept_id (emp_id)
        Shows the current department of given employee
1 row in set (0.00 sec)
Query OK, 0 rows affected (0.02 sec)
```

### 安装

您可以下载包含安装所需文件的ZIP文件，也可以在https://github.com/datacharmer/test_db克隆GitHub存储库。 在撰写本文时，只有一个分支称为master。 如果您已经下载了ZIP文件，它将解压缩到名为test_db-master的目录中。

有几个文件。 与在MySQL 8中安装employee数据库有关的两个是employee.sql和employee_partitioned.sql。 区别在于薪金和职称表是否已分区。 （也有employees_ partitioned_5.1.sql，它用于MySQL 5.1，其中不支持employees_partitioned.sql中使用的分区方案。）

通过使用SOURCE命令获取.dump文件来加载数据。 在撰写本文时，MySQL Shell不支持SOURCE命令，因此您将需要使用旧版mysql命令行客户端来导入数据。 转到包含源文件的目录，然后根据是否要使用分区来选择employees.sql或employees_partitioned.sql文件，例如：

```
mysql> SOURCE employees.sql
```

导入需要一点时间，并通过显示花费了多长时间来完成：

```
+---------------------+
| data_load_time_diff |
+---------------------+
| 00:01:51            |
+---------------------+
1 row in set (0.44 sec)
```

（可选）您可以通过采购objects.sql文件来加载一些额外的视图和存储的例程：

```
mysql> SOURCE objects.sql
```

除了此处讨论的数据集之外，还有其他一些选择来获取要使用的示例数据。

## 其他数据库

可能发生的情况是，您需要执行需要数据的测试，而这些数据的某些要求是到目前为止讨论的标准示例数据库无法满足的。 幸运的是，还有其他选择。

------

**提示**  不要低估创建自己的自定义示例数据库的可能性，例如，通过在生产数据上使用数据屏蔽。

------

如果您要查找一个非常大的真实示例，则可以按照https://en.wikipedia.org/wiki/ Wikipedia:Database_download中的说明下载Wikipedia数据库。 自2019年9月20日起，英语Wikipedia转储为bzip2压缩XML格式的16.3 GiB。

如果要查找JSON数据，则可以选择以美国地质调查局（USGS）的地震信息提供，该信息以GeoJSON格式提供，并可以选择下载过去一小时，一天，一周或一个月的地震信息（可选） 受地震的影响。 格式说明和提要的链接可以在https://earthquake.usgs.gov/earthquakes/feed/v1.0/geojson.php中找到。 由于数据包含GeoJSON格式的地理信息，因此对于需要空间索引的测试很有用。

上一章中介绍的基准测试工具还包括测试数据或支持创建测试数据。 此数据也可能对您自己的测试有用。

如果您搜索Internet，则还有其他可用的示例数据库。 最后，需要考虑的重要事项是数据是否适合测试，以及是否使用了所需的功能。

## 总结

本章介绍了四个标准示例数据库和一些其他测试数据示例。 讨论的四个标准数据库是world，world_x，sakila和employee。 这些都可以通过MySQL手册（https://dev.mysql.com/doc/index-other.html）找到。 除employee外，除非另有说明，否则这些数据库均用于本书中的示例。

world和world_x数据库是最简单的数据库，不同之处在于world_x使用JSON来存储一些信息，而world数据库是纯粹的关系数据库。 这些数据库包含的数据不多，但是由于它们的大小和简单性，它们对于简单的测试和示例很有用。 特别是本书中世界数据库被广泛使用。

sakila数据库具有更为复杂的架构，其中包括不同的索引类型，视图和存储的例程。 这使其更现实，并允许进行更复杂的测试。 但是，数据的大小仍然足够小，甚至可以在小型MySQL实例上使用。 本书也广泛使用它。

employee数据库的架构复杂度介于world数据库和sakila数据库之间，但是拥有大量数据，因此更适合测试各种查询计划之间的差异。 如果您需要在实例上产生一些负载，例如使用表扫描，它也很有用。 employee数据库在本书中并未直接使用，但是如果您要重现一些需要一定负载的示例，那么这是使用四个标准测试数据库中最好的一个。

您不应该局限于考虑标准测试数据库。 您可能可以创建自己的数据库，使用基准测试工具创建数据库，或者查找Internet上可用的数据。 维基百科的数据库和来自美国地质调查局（USGS）的地震数据就是可以下载的数据示例。

这样就完成了MySQL查询性能调优的介绍。 第二部分从Performance Schema开始，介绍了与诊断性能问题有关的常见信息源。