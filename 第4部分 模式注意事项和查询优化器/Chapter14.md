# 索引

向表添加索引是提高查询性能的一种非常强大的方法。Anindex 允许 MySQL 快速查找查询所需的数据。当将正确的索引添加到表中时，查询性能可能会通过多个大小顺序来提高。诀窍是知道要添加哪些索引。为什么不只添加索引所有列呢？索引也有开销，因此您需要在添加随机索引之前分析您的需求。

本章开始讨论什么是索引、一些索引概念以及添加索引可以有什么缺点。然后介绍 MySQL 支持的各种索引类型和功能。本章的下一部分开始讨论InnoDB如何使用与索引组织表特别相关的索引。最后，讨论了如何选择要添加到表中的索引以及何时添加它们。

## 什么是索引？

为了能够使用索引来正确提高性能，了解索引是什么非常重要。本节将不介绍不同的索引类型（将在本章的"索引类型"部分中讨论）而是索引的更高级别想法。

索引的概念已经不是什么新鲜事了，在计算机数据库被已知之前就已经存在了。作为一个简单的例子，考虑这本书。在本书的末尾，有一些单词和术语的索引，这些单词和术语被选为本书中文本最相关的搜索术语。图书索引的工作方式在概念上与数据库索引的工作方式类似。它组织数据库中的"术语"，因此您的查询可以比通过读取所有数据并检查它是否匹配搜索条件更快地找到相关数据。此处引用术语，因为它不一定是索引由人类可读的单词。也可以对二进制数据（如空间数据）进行索引。

简而言之，索引组织数据的方式可以缩小查询需要检查的行数。从精心挑选的指数加速可能是巨大的 - 几个数量级。再次考虑这本书：如果你想阅读关于B树索引，你可以从第1页开始，继续阅读整本书，或者查找书索引中的"B树索引"一词，然后直接跳到相关页面。在查询 MySQL 数据库时，这些改进与查询比查找有关书籍中某物的信息要复杂得多的区别是一样，因此索引的重要性也会增加。

很明显，你只需要添加所有可能的索引，对不对？不。除了添加索引的管理复杂性，索引本身不仅在正确使用时提高了性能;还提高了索引本身。它们还增加了开销。所以，你需要小心选择你的索引。

另一件事是，即使可以使用索引，它并不总是比扫描整个表更有效率。如果你想阅读这本书的重要部分，查找索引中每个感兴趣的术语，找出讨论主题的地方，阅读主题最终会变得比仅仅阅读整本书从封面到封面要慢。同样，如果您的查询无论如何都需要访问表中大部分数据，则只需从一端读取整个表到另一端，就会更快。究竟是什么阈值， 它变得更便宜， 扫描整个表取决于几个因素。这些因素包括磁盘类型、与随机 I/O 相比的存储 I/O 性能、数据是否适合内存等。

在深入探讨索引的详细信息之前，值得快速了解一些关键索引概念。

## 索引概念

鉴于主题索引的很大，使用多个术语来描述索引也就不足为奇了。当然，索引类型的名称（如 B 树、全文、空间等）也有，但也有更通用的术语非常重要。索引类型将在本章的稍后部分介绍，因此这里将讨论更多的一般术语。

### Key Versus Index

您可能已经注意到，有时使用"索引"一词，有时使用"键"一词。有什么区别？索引是键的列表。但是，在 MySQL 声明中，这两个术语通常是可互换的。

一个很重要的例子是"主键"，在这种情况下，必须使用"密钥"。另一方面，当您添加索引时，可以在 ADD 表上写入 table_name添加...或更改表table_name添加键...如你愿。在这种情况下，手册使用"索引"，因此为了一致性，建议使用索引。

有几个术语来描述您使用的索引。其中第一个将讨论的是一个独特的索引。

### 唯一索引

唯一索引是一个索引，它只允许索引中每个值一行。考虑包含有关人员数据的表。您可以包括该人的社会保险号码或类似标识。任何两个人都不应共享社会保险号码，因此在存储社会保险号码的列上定义唯一索引是有意义的

从这个意义上说，"唯一"更是指约束而不是索引功能。但是，索引部分对于 MySQL 能够快速确定新值是否存在至关重要。

在 MySQL 中使用唯一索引时，一个重要的考虑因素是 NULLvalus 的处理方式。比较两个 NULL 值是未定义的（换句话说，NULLdoes 不等于 NULL），因此允许 NULL 值的列上的唯一索引不会对列的 NULL 行数设置任何限制。如果要将唯一约束限制为只允许单个 NULL 值，请使用触发器检查是否已存在 NULL 值，并使用 SIGNAL 语句引发错误。在清单14-1中可以看到触发器的示例。

```
Listing 14-1. Trigger checking for unique constraint violations
CREATE TABLE my_table (
 Id int unsigned NOT NULL,
 Name varchar(50),
 PRIMARY KEY (Id),
 UNIQUE INDEX (Name)
);

DELIMITER $$
CREATE TRIGGER befins_my_table
BEFORE INSERT ON my_table
 FOR EACH ROW
BEGIN
 DECLARE v_errmsg, v_value text;
 IF EXISTS(SELECT 1 FROM my_table WHERE Name <=> NEW.Name) THEN
 IF NEW.Name IS NULL THEN
 SET v_value = 'NULL';
 ELSE
 SET v_value = CONCAT('''', NEW.Name, '''');
 END IF;
 SET v_errmsg = CONCAT('Duplicate entry ',
 v_value,
' For key ''Name''');
 SIGNAL SQLSTATE '23000'
 SET MESSAGE_TEXT = v_errmsg,
 MYSQL_ERRNO = 1062;
 END IF;
END$$
DELIMITER ;
```

这将处理名称列的任何类型的重复值。它使用 NULL 安全条件运算符 （+） 来确定 Name 的新值是否已存在于表中。如果是，它将引用该值（如果它不是 NULL，否则不引用它）因此可以区分字符串"NULL"和 NULL 值。最后，发出 SQL 状态 23000 和 MySQL 错误编号 1062 的信号。错误消息、SQL 状态和错误编号与正常重复键约束错误相同。

一种特殊的唯一索引是主键。

### Primary Key

表的主要键是唯一定义行的索引。主键允许使用 NULL 值。如果表上有多个 NOT NULL 唯一索引，则这两个索引都可用于作为主键的目的。对于在讨论群集索引时不久将解释的原因，应为主键选择具有不可变值的一个或多个列。也就是说，旨在永远不会更改给定行的正密钥。

主键对于 InnoDB 非常特殊，而对于其他存储引擎，它可能更需要约定。但是，在所有情况下，最好始终具有唯一标识行的一些值，例如，允许复制快速确定要修改的行（第 26 章中有关此内容），并且组复制功能明确要求所有表具有主键或非 NULL 唯一索引。在 MySQL 8.0.13 及更晚sql_require_primary_key，您可以启用"sql_require_primary_key"选项，要求所有新表都必须具有主键。如果更改现有表的结构，该限制也适用。

提示 启用"sql_require_primary_key"选项（默认情况下禁用）。没有主键的表可能会导致性能问题，有时是无法期待和微妙的方式。这还可确保您的表准备就绪，如果您想要在将来使用组复制。

如果有主键，也有辅助键吗？

### Secondary Indexes

术语"辅助索引"用于不是主键的索引。它没有任何特殊含义，因此名称只是用于使其明确索引不是主键，无论是唯一索引还是非唯一索引。

如前所述，主键对于 InnoDB 具有特殊含义，因为它用于群集索引。

### Clustered Index

群集索引特定于 InnoDB，是用于 InnoDB 组织数据的术语。如果您熟悉 Oracle DB，您可能知道索引组织的表;如果您熟悉 Oracle DB，您可能知道索引组织表。描述同样的事情。

InnoDB 中的一切都是索引。行数据位于 B 树索引的叶页中（很快将介绍 B 树索引）。此索引称为群集索引。名称来自索引值聚集在一起这一事实。主键用于群集索引。如果未指定显式主键，InnoDB 将查找不允许 NULL 值的唯一索引。如果不存在，InnoDB 将使用全局（对于所有 InnoDB 表）自动递增值添加隐藏的 6 字节整数列，以生成唯一值。

选择主键也有性能影响。这些将在本章的"索引策略"部分中讨论。群集索引也可以被视为覆盖索引的特殊情况。这是怎麽？你即将发现。

### Covering Index

如果索引包含给定查询的索引表中所需的所有列，该索引即为覆盖索引。也就是说，索引是否覆盖取决于您使用索引的查询。索引可能覆盖一个查询，但不包括另一个查询。考虑索引列 （a， b） 的索引和选择这两列的查询：

```sql
SELECT a, b
 FROM my_table
 WHERE a = 10;
```

在这种情况下，查询只需要列 a 和 b，因此不需要查找行的其余部分 - 索引足以检索所有必需的数据。另一方面，如果查询也需要列 c，则索引不再覆盖。当您使用 EXPLAIN 语句分析查询（第 20 章将介绍这一点）并且表使用覆盖索引时，EXPLAIN 输出中的"额外"列将包括"使用索引"。

覆盖索引的特殊情况是 InnoDB 的群集索引（尽管 EXPLAIN 不会为它说"使用索引"）。群集索引包括叶节点中的所有行数据（即使通常实际上只有一部分列已编制索引），因此索引将始终包含所有必需的数据。某些数据库在创建可用于模拟群集索引工作的索引时支持 include 子句。

聪明地创建索引，以便将它们用作执行最多的查询的索引可以大大提高性能，这将在“索引策略”部分中进行讨论。

添加索引时，需要遵守一些限制。这些限制是下一件事要涵盖的。

## 索引限制

InnoDB 索引存在一些限制。这些范围从索引大小到表上允许的索引数。最重要的限制如下：

- 根据InnoDB行格式，B树索引的最大宽度为3072字节或767字节。 最大大小基于16 kiB InnoDB页面，对于较小的页面大小，较低的限制。
- 当指定前缀长度时，只能在全文索引以外的索引中使用Blob和text-type列。 前缀索引将在本章的“索引功能”部分中讨论。
- 功能关键部件计入表中1017列的限制。
- 每个表上最多可以有64个二级索引。
- 多列索引最多可以包含16列和功能关键部分。

您可能遇到的限制是 B 树索引的最大索引宽度。使用动态（默认）或压缩行格式时，索引不能超过 3072 字节，对于冗余和压缩行格式，索引不能超过 767 字节。使用 DYNAMIC 和 压缩行格式的表的限制减少到 8 个 KiB 页的半（1536 字节）和 4 个 KiB 页的四分之一（768 字节）。这尤其对字符串和二进制列上的索引进行限制，因为这些值在本质上是通常很大，而且也是大小计算中使用的最大可能存储量。这意味着使用 utf8mb4 字符集的 varchar（10） 将贡献 40 字节到限制，即使您从不通过列中的单字节字符存储任何内容。

将 B 树索引添加到文本或 Blob 类型的列时，必须始终提供一个键长度，指定要在索引中包含的列的前缀。这甚至适用于仅支持 256 字节数据的微小文本和小文本。对于字符、varchar、二进制列和二进制列，如果值的最大大小超过表允许的最大索引宽度，则只需指定前缀长度。

提示 对于文本和 Blob 类型的列，最好使用全文索引（稍后）编制索引，添加具有 Blob 哈希值的生成的列，或者以某种其他方式优化访问。

如果向表添加功能索引，则每个功能键部分将计入表上列的限制。如果创建包含两个功能部分的索引，则这算作两列，指向表限制。对于 InnoDB，表中最多可以有 1017 列。

最后两个限制与表中可包含的索引数以及单个索引中可以包含的列数和功能关键部分的数量相关。表上最多可以有 64 个辅助索引。实际上，如果您接近此限制，则重新考虑索引策略可能会从中受益。正如将在本章的"索引的缺点是什么？" 中讨论一样，存在与索引关联的开销，因此在所有情况下，最好将索引的数量限制为那些真正受益的查询。同样，添加到索引的部件越大，索引越大。InnoDB 限制是，您最多可以添加 16 个部分。

如果需要向表添加索引或删除多余的索引，该怎么办？索引可以与表或更晚一起创建，也可以像接下来讨论时那样删除索引。

## SQL Syntax

首次创建架构时，您通常会有一些要添加索引的想法。然后，随着时间的推移，您的监视可能会确定不再需要某些索引，但应添加其他索引。索引的这些更改可能是由于对所需索引的误解;数据可能已更改，或者查询可能已更改。

在更改表上的索引时，有三种不同的操作：在创建表本身时创建索引、向现有表添加索引或从表中删除索引。索引定义是相同的，无论您是将索引与表一起添加还是作为以下操作添加。删除索引时，只需索引名称。

本节将显示用于添加和删除索引的常规语法。在本章的其余部分中，将基于特定索引类型和功能提供进一步的示例。

### Creating Tables with Indexes

建表时，可以将索引定义添加到 CREATE TABLE 语句。索引在列之后定义。您可以选择指定索引的名称;如果没有，索引将以索引中的第一列命名。

清单 14-2 显示了创建多个索引的表的示例。如果您不知道所有索引类型都在做什么，请勿担心 ， 将在本章的稍后部分讨论。

```sql
Listing 14-2. Example of creating a table with indexes
CREATE TABLE db1.person (
 Id int unsigned NOT NULL,
 Name varchar(50),
 Birthdate date NOT NULL,
 Location point NOT NULL SRID 4326,
 Description text,
 PRIMARY KEY (Id),
 INDEX (Name),
 SPATIAL INDEX (Location),
 FULLTEXT INDEX (Description)
);
```

这将创建 db1 架构中具有四个索引的表人（必须事先存在）。第一个主键是 Id 列上的 B 树索引（有关此索引）。第二个也是 B 树索引，但它是所谓的辅助索引，索引为 Name 列。第三个索引是"位置"列上的空间索引。第四个是"描述"列上的全文索引。

您还可以创建包含多个列的索引。如果需要在多列上放条件，在第一列上放一个条件，然后按第二列排序，等等，这非常有用。若要创建多列索引，请将列名称指定为逗号分隔列表：

```sql
INDEX (Name, Birthdate)
```

列的顺序非常重要，因为它将在"索引策略"中解释。简而言之，MySQL 只能使用左侧的索引，也就是说，只有在也使用 Name 时，才能使用索引的出生日期部分。这意味着索引（名称、出生日期）与索引（出生日期、姓名）不一样。

表上的索引一般不会保持静态，因此，如果要将索引添加到现有表中，该怎么办？



### Adding Indexes

如果确定需要，可以向现有表添加索引。为此，您需要使用 ALTER TABLE 或创建 INDEX 语句。由于 ALTER TABLE 可用于表的所有修改，因此您可能需要坚持这样做;但是，完成的工作与任一语句相同。

清单 14-3 显示了如何使用 ALTER TABLE 创建索引的两个示例。第一个示例添加单个索引;第二个在一个语句上添加两个索引。

```sql
Listing 14-3. Adding indexes using ALTER TABLE
ALTER TABLE db1.person
 ADD INDEX (Birthdate);
ALTER TABLE db1.person
 DROP INDEX Birthdate;
ALTER TABLE db1.person
 ADD INDEX (Name, Birthdate),
 ADD INDEX (Birthdate);
```

第一个和最后一个 ALTER TABLE 语句使用 ADD INDEX 子句告诉 MySQL 应向表中添加索引。第三个语句添加两个这样的子句，用逗号分隔，在一个语句中添加两个索引。在两者之间，索引被丢弃，因为有重复索引是不好的做法，MySQL 也会警告它。

如果使用两个语句添加两个索引或同时添加一个语句的两个索引，这是否不同？是的，可能会有很大的不同。添加索引时，需要执行完整的表扫描以读取索引所需的所有值。对于大型表来说，完整表扫描是一项昂贵的操作，因此，最好在一个语句中添加两个索引。另一方面，只要索引可以完全保存在 InnoDB 缓冲池中，创建索引的速度就快得多。将两个索引的创建拆分为两个语句可以减轻缓冲池的压力，从而提高索引创建性能。

最后一个操作是删除不再需要的索引。



### Removing Indexes

除索引的行为类似于添加索引。您可以使用 ALTER TABLE 或 Drop INDEX 语句。使用 ALTER TABLE 时，可以将删除索引与表的其他数据定义操作相结合。

删除索引时，需要知道索引的名称。如清单14-4所示，有几种方法可以做到这一点。

```sql
Listing 14-4. Find the index names for a table
mysql> SHOW CREATE TABLE db1.person\G
*************************** 1. row ***************************
 Table: person
Create Table: CREATE TABLE `person` (
 `Id` int(10) unsigned NOT NULL,
 `Name` varchar(50) DEFAULT NULL,
 `Birthdate` date NOT NULL,
 `Location` point NOT NULL /*!80003 SRID 4326 */,
 `Description` text,
 PRIMARY KEY (`Id`),
 KEY `Name` (`Name`),
 SPATIAL KEY `Location` (`Location`),
 KEY `Name_2` (`Name`,`Birthdate`),
 KEY `Birthdate` (`Birthdate`),
 FULLTEXT KEY `Description` (`Description`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.0010 sec)

mysql> SELECT INDEX_NAME, INDEX_TYPE,
 GROUP_CONCAT(COLUMN_NAME
 ORDER BY SEQ_IN_INDEX) AS Columns
 FROM information_schema.STATISTICS
 WHERE TABLE_SCHEMA = 'db1'
 AND TABLE_NAME = 'person'
 GROUP BY INDEX_NAME, INDEX_TYPE;
+-------------+------------+----------------+
| INDEX_NAME  | INDEX_TYPE | Columns        |
+-------------+------------+----------------+
| Birthdate   | BTREE      | Birthdate      |
| Description | FULLTEXT   | Description    |
| Location    | SPATIAL    | Location       |
| Name        | BTREE      | Name           |
| Name_2      | BTREE      | Name,Birthdate |
| PRIMARY     | BTREE      | Id             |
+-------------+------------+----------------+
6 rows in set (0.0013 sec)
```

索引可能以不同的顺序列出。第一个查询使用 SHOW CREATE TABLE 语句获取完整的表定义，其中还包括索引及其名称。第二个查询查询information_schema。统计信息视图。此视图对于获取有关索引的信息非常有用，下一章将详细讨论该视图。确定要删除哪个索引后，可以使用 ALTER TABLE，如清单 14-5 所示。

```sql
Listing 14-5. Dropping an index using ATLER TABLE
ALTER TABLE db1.person DROP INDEX name_2;
```

这将删除名为name_2的索引-即（Name，Birthdate）列上的索引。

本章的其余部分将介绍什么是索引的各种详细信息，在本章末尾，"索引策略"部分讨论如何选择要编制索引的数据。首先，了解索引为什么具有开销非常重要。

## 索引的缺点是什么？

生活中很少有东西是免费的——索引也不例外。虽然索引非常适合提高查询性能，但还需要存储和保持最新。此外，执行查询时不太明显的开销，索引越多，优化器需要做的工作也越多。本节将介绍索引的这三个缺点。

### Storage

添加索引最明显的成本之一是需要存储索引，因此需要时随时可用。您不希望每次需要时首先创建索引，因为这将扼杀索引的性能优势。

磁盘存储意味着您可能需要向系统添加磁盘或块存储。如果使用复制原始表空间文件的 MySQL 企业备份 （MEB） 等备份解决方案，备份也将变大，需要更长的时间才能完成。

InnoDB 始终使用其缓冲池来读取查询所需的数据。如果缓冲池中不存在数据，则首先读取数据，然后用于查询。因此，当您使用索引时，索引和行数据一般都会读取到缓冲池中（使用覆盖索引时有一个例外）。需要放入缓冲池的越多，用于其他索引和数据的空间就越小， 除非您使缓冲池变大。当然，它比它更复杂，因为避免完整表扫描也会阻止将整个表读取到缓冲池中，从而减轻缓冲池的压力。总体收益与开销可追溯到使用索引避免检查的表量，以及其他查询是否读取索引否则避免访问的数据。

之，在添加索引时，您将需要额外的磁盘，通常您需要一个更大的 InnoDB 缓冲池来保持相同的缓冲池命中率。另一个开销是，索引仅在保持最新状态时才有用。这将在更新数据时添加工作。

### Updating the Index

无论何时对数据进行更改，索引都必须更新。这范围从在插入或删除数据时添加或删除行的链接，到在值更新时修改索引。您可能不太想这样做，但可能是一个显著的开销。事实上，在批量数据加载（如还原逻辑备份（通常包含用于创建数据的 SQL 语句（例如，使用 mysqlpump 程序创建）期间，保持索引更新的开销通常限制了插入速率。

提示 使索引保持最新开销可能很高，因此通常建议在将大量导入到空表中时删除辅助索引，然后在导入完成后重新创建索引。

对于 InnoDB，开销还取决于辅助索引是否适合缓冲池。只要整个索引位于缓冲池中，保持索引最新成本相对低廉，而且不太可能成为严重的瓶颈。如果索引不适合，InnoDB 将不得不继续在表空间文件和缓冲池之间进行洗牌，当开销成为导致严重性能问题的主要瓶颈时。

还有一个不太明显的性能开销。索引越多，优化器确定最佳查询计划的工作也更多。

### The Optimizer

当优化器分析查询以确定它认为的最佳查询执行计划时，它需要评估每个表上的索引，以确定是否应使用索引，以及是否执行两个索引的索引合并。当然，目标是让查询尽快进行评估。但是，在优化器中花费的时间通常不可忽略，在某些情况下甚至可能成为瓶颈。

考虑一个非常简单的查询示例，从一个表中选择一些行：

```sql
SELECT ID, Name, District, Population
 FROM world.city
 WHERE CountryCode = 'AUS';
```

在这种情况下，如果表城市上没有索引，则显然需要表扫描。如果有一个索引，则还需要使用索引评估查询成本，等等。如果您有一个复杂的查询，涉及多个表，每个表包含十几个可能的索引，它将进行这么多组合，以使其反映在查询执行时间中。

提示 如果在优化器中花费的时间成为问题，您可以添加优化器并加入订单提示，如第 17 章和第 24 章中讨论，以帮助优化器，因此不需要评估所有可能的查询计划。

虽然这些描述添加索引开销的页面会使索引听起来像是坏，但不要回避索引。对于频繁执行的查询具有很大选择性的索引将大有作为。但是，不要为了添加索引而添加索引。将在"索引策略"一节的章节末尾讨论选择索引的一些想法，在本书的其余部分中还将讨论索引。在得到这一步之前，值得讨论 MySQL 支持的各种索引类型以及其他索引功能。

## Index Types

对于所有用途，索引的最佳类型并不相同。优化以查找给定值范围内的行（例如，2019 年的所有日期）的索引需要与搜索大量给定单词或短语的文本的索引大相径庭。这意味着，当您选择添加索引时，必须决定需要哪种索引类型。MySQL 目前支持五种不同的索引类型：

- B树索引
- 全文索引
- 空间索引（R树索引）
- 多值索引
- 哈希索引

本节将介绍这五种索引类型，并讨论它们可用于加速哪些类型的问题

### B-Tree Indexes

到目前为止，B 树索引是 MySQL 中最常用的索引类型。事实上，所有 InnoDB 表都至少包含一个 B 树索引，因为数据在 B 树索引（群集索引）中组织。

B 树索引是有序索引，因此它善于查找与某些值相等的列、列大于或小于给定值的行，或者列位于两个值之间的行。这使得它成为许多查询非常有用的索引。

B 树索引的另一个良好功能是它们具有可预测的性能。正如名称建议的那样，索引被组织为树，从根页开始，最后以叶页结束。InnoDB 使用 B 树索引的扩展，称为 B+树。+ 表示同一级别的节点已链接，因此在到达节点中的最后一条记录时，很容易扫描索引，而无需返回父节点。

------

**注意：在MySQL中，术语B-tree和B + -tree可互换使用**

------

图 14-1 中显示了具有城市名称的索引树的示例。（该图面向从左到右的索引级别，该水平与 B 树索引的一些其他插图的 topto-底部方向不同。这主要是因为空间的原因。

在图中，文档形状表示 InnoDB 页面，并且多个文档堆叠在一起的形状（例如，级别 0 中标记为"Christchurch"的形状）表示多个页面。从左到右的箭头从根页向叶页前进。根页是索引搜索的开始位置，叶页是索引记录存在的地方。之间的页面通常称为内部页面或分支页。页面也可以称为节点。在同一级别连接页面的双箭头是 B 树和 B+树索引的区分，并允许 InnoDB 快速移动到上一个或下一个同级页面，而无需通过父页面。

对于小型索引，可能只有一个页面同时充当根页和叶页。在更一般的情况下，索引有一个根页，在图的最左侧显示。在图的最右侧是叶页。对于大型索引，两者之间可能有更多的级别。叶节点具有级别 0，其父页级别 1，直到到达根页。

在图中，为页面注意到的值（例如"Coruía"）表示树的那部分覆盖的第一个值。因此，如果您位于 1 级，并且正在寻找值"Adelaide"，则了解该值将在叶页的最上面一页，因为该页包含以"A Coruía"为起点的值，最后一个值以"北京"为顺序结束，按值排序的顺序排列。这是上一章中讨论的排序规则发挥作用的示例。

关键功能是，无论您遍历的哪个分支，级别数将始终相同。例如，在图中，这意味着无论您查找哪个值，都会读取四个页面，四个级别中每个级别一个（如果多行具有相同的值，并且对于范围扫描，则可能会读取叶级别中的更多页面）。因此，据说树是平衡的。正是此功能提供了可预测的性能，并且级别的数量可以很好地扩展 - 也就是说，级别的数量会随着索引记录的数量而缓慢增长。当需要从相对缓慢的存储（如磁盘）访问数据时，这是一个特别重要的属性。

注意 您可能也听说过 T 树索引。虽然 B 树索引针对磁盘访问进行了优化，但 T 树索引与 B 树索引类似，只是它们针对内存访问进行了优化。因此，存储内存中所有索引数据的 NDBCluster 存储引擎使用 T 树索引，即使它们处于 SQL 级别也称为 B 树索引。

在本节的开头，指出 B 树索引是迄今为止 MySQL 中最常用的索引类型。事实上，如果您有任何 InnoDB 表，即使您从未自己添加任何索引，您也使用 B 树索引。InnoDB 存储组织的数据索引 （使用群集索引），这通常只是意味着行存储在 B+树索引中。B 树索引也不仅用于关系数据库，例如，多个文件系统在 B 树结构中组织其元数据。

需要注意的 B 树索引的一个属性是，它们只能用于比较索引列或左前缀的整个值。这意味着，如果要检查索引日期的月份是否为 5 月，则无法使用索引。如果要检查索引字符串是否包含给定短语，这是相同的。

在索引中包含多个列时，同样适用同样的原则。考虑索引（名称、出生日期）：在这种情况下，您可以使用索引搜索给定名称或名称和生日的组合。但是，在不知道姓名的情况下，不能使用索引来搜索具有给定出生日期的人。

有几种方法可以处理此限制。在某些情况下，可以使用功能索引，也可以将有关该列的信息提取到可以编制索引的生成列中。在其他情况下，可以使用另一种索引类型。如下文讨论，例如，全文索引可用于搜索字符串中某处包含短语"查询性能调整"的列。

### Full Text Indexes

全文索引专门回答诸如"哪个文档包含此字符串？也就是说，它们没有经过优化来查找列与字符串完全匹配的行 - 为此，B 树索引是更好的选择。

全文索引的工作原理是标记正在编制索引的文本。具体如何完成取决于使用的解析器。InnoDB 支持使用自定义解析器，但通常使用内置解析器。默认解析器假定文本使用空格作为单词分隔符。MySQL 包括两个替代解析器：支持中文、日文和韩文的 ngram 解析器2，以及支持日语的 MeCab 解析器。

InnoDB 使用名为"FTS_DOC_ID列将全文索引链接到行，这是一个未签名的不签名列。如果添加全文索引，并且该列不存在，InnoDB 将将其添加为隐藏列。添加隐藏列需要重新生成表，因此，如果要向大型表添加全文索引，需要考虑这一点。如果您知道打算对表使用全文索引，可以自己添加列与列的唯一索引FTS_DOC_ID_INDEX索引。您还可以选择使用FTS_DOC_ID列作为主键，但请注意，FTS_DOC_ID不允许重用这些值。自己准备表的示例如下所示：

```sql
DROP TABLE IF EXISTS db1.person;
CREATE TABLE db1.person (
 FTS_DOC_ID bigint unsigned NOT NULL auto_increment,
 Name varchar(50),
 Description text,
 PRIMARY KEY (FTS_DOC_ID),
 FULLTEXT INDEX (Description)
);
```

如果没有"索引FTS_DOC_ID，并且向现有表添加全文列，MySQL 将返回一条警告，告诉已重新生成该表以添加该列：

```
Warning (code 124): InnoDB rebuilding table to add column FTS_DOC_ID
```

如果您计划使用全文索引，建议从性能角度显式添加 FTS_DOC_ID 列，并将其设置为表上的主要键，或为它创建辅助唯一索引。自己创建列的缺点是您必须自己管理值。

一种专用索引类型是空间数据。如果全文索引用于文本文档（或字符串），则空间索引用于空间数据类型。

### Spatial Indexes (R-Tree)

从历史上看，MySQL 中空间要素使用不多。但是，由于版本 5.7 中对 InnoDB 中的空间索引的支持以及其他改进（例如支持在 MySQL 8 中为空间数据指定空间参考系统标识符 （SRID），因此您可能在某个时候可能需要空间索引。

空间索引的典型用例是具有感兴趣点的表，该表包含每个点的位置以及其余信息。例如，用户可以要求在其当前位置 50 公里范围内获取所有电动汽车充电站。要尽可能高效地回答此类问题，您需要一个空间索引。

MySQL 将空间索引实现为 R 树。R 代表矩形，并提示索引的用法。R 树索引组织数据，以便空间中接近的点在索引中彼此存储。这就是确定空间值是否满足某些边界条件（例如矩形）的有效方法。

只有在列声明为"非 NULL"且已设置空间参考系统标识符时，才能使用空间索引。空间条件使用 MBRContains( ) 等函数之一指定，该函数采用两个空间值，并返回第一个值是否包含另一个值。否则，对使用空间索引没有特殊要求。清单 14-6 显示了具有空间索引的表和可以使用索引的查询的示例。

```sql
Listing 14-6. Using a spatial index
mysql> CREATE TABLE db1.city (
 id int unsigned NOT NULL,
 Name varchar(50) NOT NULL,
 Location point SRID 4326 NOT NULL,
 PRIMARY KEY (id),
 SPATIAL INDEX (Location));
Query OK, 0 rows affected (0.5578 sec)
mysql> INSERT INTO db1.city
 VALUES (1, 'Sydney',
 ST_GeomFromText('Point(-33.8650 151.2094)',
 4326));
Query OK, 1 row affected (0.0783 sec)
mysql> SET @boundary = ST_GeomFromText('Polygon((-9 112, -45 112, -45 160,
-9 160, -9 112))', 4326);
Query OK, 0 rows affected (0.0004 sec)
mysql> SELECT id, Name
 FROM db1.city
 WHERE MBRContains(@boundary, Location);
+----+--------+
| id | Name   |
+----+--------+
| 1  | Sydney |
+----+--------+
1 row in set (0.0006 sec)
```

在该示例中，具有城市位置的表在"位置"列上具有空间索引。空间参考系统标识符 （SRID） 设置为 4326 以表示地球。对于此示例，插入一行，并定义边界（如果您好奇，则边界包含澳大利亚）。您还可以直接在 MBRContains( )函数中指定多边形，但在这里，通过两个步骤完成，使查询部分更加清晰。

因此，空间索引有助于回答某些几何形状是否在某些边界内。同样，多值索引可以帮助回答给定值是否在值列表中。

### Multi-valued Indexes

MySQL 在 MySQL 5.7 中引入了对 JSON 数据类型的支持，并在 MySQL 8 中的 MySQL 文档存储扩展了此功能。可以在生成的列或功能索引上使用索引在 JSON 文档上创建索引;在生成的列或功能索引上创建索引。但是，到目前为止讨论的索引类型未涵盖的用例是搜索 JSON 数组包含某些值的文档。例如，一个城市的集合，每个城市都有一系列郊区。上一章中的示例 JSON 文档只是：

```json
{
 "name": "Sydney",
 "demographics": {
 "population": 5500000
 },
 "geography": {
 "country": "Australia",
 "state": "NSW"
 },
 "suburbs": [
 "The Rocks",
 "Surry Hills",
 "Paramatta"
 ]
}
```

如果你想搜索城市集合中的所有城市，并返回那些有一个名为"萨里山"的郊区的城市，那么你需要一个多值索引。MySQL 8.0.17 增加了对多值指数的支持。

解释多值索引如何有用的最简单方法是查看示例。清单 14-7 将国家信息表从 world_x 示例数据库中复制到 mvalue_index 表，并修改它，以便每个 JSON 文档都包含一个包含其人口和位于地区的城市数组。最后，包括一个查询，以显示检索澳大利亚的所有城市名称的示例（_id = "AUS"）。这些查询在本书的 GitHub 存储库中的文件 listing_14_7.sql 中也可用，并且可以使用命令 "源 listing_14_7.sql"在 MySQL 命令中执行。

```
Listing 14-7. Preparing the mvalue_index table for multi-valued indexes
mysql> \use world_x
Default schema set to `world_x`.
Fetching table and column names from `world_x` for auto-completion...
Press ^C to stop.
mysql> DROP TABLE IF EXISTS mvalue_index;
Query OK, 0 rows affected, 1 warning (0.0509 sec)
Note (code 1051): Unknown table 'world_x.mvalue_index'
mysql> CREATE TABLE mvalue_index LIKE countryinfo;
Query OK, 0 rows affected (0.3419 sec)
mysql> INSERT INTO mvalue_index (doc)
 SELECT doc
 FROM countryinfo;
Query OK, 239 rows affected (0.5781 sec)
Records: 239 Duplicates: 0 Warnings: 0
mysql> UPDATE mvalue_index
 SET doc = JSON_INSERT(
 doc,
 '$.cities',
 (SELECT JSON_ARRAYAGG(
 JSON_OBJECT(
 'district', district,
'name', name,
'population',
 Info->'$.Population'
 )
 )
 FROM city
 WHERE CountryCode = mvalue_index.doc->>'$.Code'
 )
 );
Query OK, 239 rows affected (3.6697 sec)
Rows matched: 239 Changed: 239 Warnings: 0
mysql> SELECT JSON_PRETTY(doc->>'$.cities[*].name')
 FROM mvalue_index
 WHERE doc->>'$.Code' = 'AUS'\G
*************************** 1. row ***************************
JSON_PRETTY(doc->>'$.cities[*].name'): [
 "Sydney",
 "Melbourne",
 "Brisbane",
 "Perth",
 "Adelaide",
 "Canberra",
 "Gold Coast",
 "Newcastle",
 "Central Coast",
 "Wollongong",
 "Hobart",
 "Geelong",
 "Townsville",
 "Cairns"
]
1 row in set (0.0022 sec)
```

列表首先将 world_x 架构设置为默认值，然后删除 mvalue_index 表（如果存在），然后使用与国家/地区信息表相同的定义和相同的数据再次创建该表。您还可以直接修改国家/地区信息表，但通过处理 mvalue_index 副本，可以通过删除world_x表轻松重置 mvalue_index 架构。该表由名为 doc 的 JSON 文档列和名为"_id，这是主键：

```
mysql> SHOW CREATE TABLE mvalue_index\G
*************************** 1. row ***************************
 Table: mvalue_index
Create Table: CREATE TABLE `mvalue_index` (
 `doc` json DEFAULT NULL,
 `_id` varbinary(32) GENERATED ALWAYS AS
(json_unquote(json_extract(`doc`,_utf8mb4'$._id'))) STORED NOT NULL,
 `_json_schema` json GENERATED ALWAYS AS (_utf8mb4'{"type":"object"}')
VIRTUAL,
 PRIMARY KEY (`_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.0006 sec)

```

UPDATE 语句使用 JSON_ARRAYAGG（） 函数创建一个 JSON 数组，其中包含三个 JSON 对象（地区、名称和人口）。最后，执行 SELECT 语句以返回澳大利亚城市的名称。

现在，您可以为城市名称添加多值索引：

```sql
ALTER TABLE mvalue_index
 ADD INDEX (((CAST(doc->>'$.cities[*].name'
 AS char(35) ARRAY))));
```

索引从文档文档根目录的城市数组的所有元素中提取名称对象。生成的数据将投射到 char（35） 值数组。数据类型被选为城市名称源自的城市表是 char （35）。在 CAST（） 函数中，对字符和 varchar 数据类型都使用 char。

新索引可用于使用 FOR 的运算符成员和 JSON_CONTAINS（） 和 JSON_OVERLAPS）函数的 WHERE 子句。运算符成员询问给定值是否为数组的成员。JSON_CONTAINS（） 非常相似，但与引用搜索成员相比，需要范围搜索。JSON_OVERLAPS（） 可用于查找至少包含多个值之一的文档。清单14-8显示了使用运算符和每个函数的示例。

```
Listing 14-8. Queries taking advantage of a multi-valued index
mysql> SELECT doc->>'$.Code' AS Code, doc->>'$.Name'
 FROM mvalue_index
 WHERE 'Sydney' MEMBER OF (doc->'$.cities[*].name');
+------+----------------+
| Code | doc->>'$.Name' |
+------+----------------+
| AUS  | Australia      |
+------+----------------+
1 row in set (0.0032 sec)
mysql> SELECT doc->>'$.Code' AS Code, doc->>'$.Name'
 FROM mvalue_index
 WHERE JSON_CONTAINS(
 doc->'$.cities[*].name',
 '"Sydney"'
 );
+------+----------------+
| Code | doc->>'$.Name' |
+------+----------------+
| AUS  | Australia      |
+------+----------------+
1 row in set (0.0033 sec)
mysql> SELECT doc->>'$.Code' AS Code, doc->>'$.Name'
 FROM mvalue_index
 WHERE JSON_OVERLAPS(
 doc->'$.cities[*].name',
 '["Sydney", "New York"]'
 );
+------+----------------+
| Code | doc->>'$.Name' |
+------+----------------+
| AUS  | Australia      |
| USA  | United States  |
+------+----------------+
2 rows in set (0.0060 sec)
```

使用"成员"和"JSON_CONTAINS（））的两个查询都查找具有名为悉尼的城市的国家/地区。使用"JSON_OVERLAPS（））的最后一个查询将搜索具有名为悉尼或纽约的城市或两者并大的国家/地区。"JSON_OVERLAPS（） （）

MySQL中还剩下一种索引类型：哈希索引。

### Hash Indexes

如果要搜索列完全等于某些值的行，可以使用 B 树索引，如本章前面讨论。不过，还有一种选择：为每个列值创建一个哈希，并使用哈希来搜索匹配的行。你为什么要那么做？答案是，这是一个非常快的方式来查找行

 MySQL 中，哈希索引使用不多。一个值得注意的例外是 NDBCluster 存储引擎，它使用哈希索引来确保主键和唯一索引的唯一性，并使用它们使用这些索引提供快速查找。就 InnoDB 而言，对哈希索引没有直接支持;但是，InnoDB 有一个称为自适应哈希索引的功能，值得考虑更多一点。

自适应哈希索引功能在 InnoDB 中自动工作。如果 InnoDB 检测到您经常使用辅助索引，并且启用了自适应哈希索引，它将在最常用值的飞函数上生成哈希索引。哈希索引专门存储在缓冲池中，因此在重新启动 MySQL 时不会持久化。如果 InnoDB 检测到内存可以更好地用于将更多页面加载到缓冲池中，它将丢弃哈希索引的一部分。当说它是一个自适应索引时，这就是意思：InnoDB 将尝试调整它，使其最适合您的查询。您可以使用"使用"innodb_adaptive_hash_index功能。

从理论上讲，自适应哈希指数是一种双赢的局面。您获得的优势是具有哈希索引，而无需考虑需要为哪些列添加它，并且内存使用情况都会自动处理。但是，启用它会有开销，并非所有工作负荷都从中受益。事实上，对于某些工作负载，开销可能会变得非常大，以致出现严重的性能问题。

有两种方法可以监视自适应哈希索引：信息INNODB_METRICS中的表和 InnoDB 监视器中的表。该INNODB_METRICS包含八个自适应哈希索引指标，其中两个默认情况下处于启用状态。清单 14-9 显示了该清单中包含的八INNODB_METRICS。

```
Listing 14-9. The metrics for the adaptive hash index in INNODB_METRICS
mysql> SELECT NAME, COUNT, STATUS, COMMENT
 FROM information_schema.INNODB_METRICS
 WHERE SUBSYSTEM = 'adaptive_hash_index'\G
*************************** 1. row ***************************
 NAME: adaptive_hash_searches
 COUNT: 10717
 STATUS: enabled
COMMENT: Number of successful searches using Adaptive Hash Index
*************************** 2. row ***************************
 NAME: adaptive_hash_searches_btree
 COUNT: 29515
 STATUS: enabled
COMMENT: Number of searches using B-tree on an index search
*************************** 3. row ***************************
 NAME: adaptive_hash_pages_added
 COUNT: 0
 STATUS: disabled
COMMENT: Number of index pages on which the Adaptive Hash Index is built
*************************** 4. row ***************************
 NAME: adaptive_hash_pages_removed
 COUNT: 0
 STATUS: disabled
COMMENT: Number of index pages whose corresponding Adaptive Hash Index
entries were removed
*************************** 5. row ***************************
 NAME: adaptive_hash_rows_added
 COUNT: 0
 STATUS: disabled
COMMENT: Number of Adaptive Hash Index rows added
*************************** 6. row ***************************
 NAME: adaptive_hash_rows_removed
 COUNT: 0
 STATUS: disabled
COMMENT: Number of Adaptive Hash Index rows removed
*************************** 7. row ***************************
 NAME: adaptive_hash_rows_deleted_no_hash_entry
 COUNT: 0
 STATUS: disabled
COMMENT: Number of rows deleted that did not have corresponding Adaptive
Hash Index entries
*************************** 8. row ***************************
 NAME: adaptive_hash_rows_updated
 COUNT: 0
 STATUS: disabled
COMMENT: Number of Adaptive Hash Index rows updated
8 rows in set (0.0015 sec)
```

默认情况下，使用自适应哈希索引（adaptive_hash_ 搜索）的成功搜索数和使用 B 树索引 （adaptive_ hash_searches_btree） 完成的搜索数。您可以使用这些来确定 InnoDB 使用哈希索引解决查询与基础 B 树索引相比的协商组。其他指标不太经常需要，因此默认情况下禁用。也就是说，如果您想要更详细地探讨自适应哈希索引的有用性，可以安全地启用六个指标。

监视自适应哈希索引的另一种方式是使用 InnoDB 监视器，如清单 14-10 所示。输出中的数据在您的情况下会有所不同。

```
Listing 14-10. Using the InnoDB monitor to monitor the adaptive hash index
mysql> SHOW ENGINE INNODB STATUS\G
*************************** 1. row ***************************
 Type: InnoDB
 Name:
Status:
=====================================
2019-05-05 17:22:14 0x1a7c INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 16 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 52 srv_active, 0 srv_shutdown, 25121 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 8
OS WAIT ARRAY INFO: signal count 11
RW-shared spins 12, rounds 12, OS waits 0
RW-excl spins 102, rounds 574, OS waits 8
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 1.00 RW-shared, 5.63 RW-excl, 0.00 RW-sx
...
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 2267, node heap has 2 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 2 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 1 buffer(s)
Hash table size 2267, node heap has 2 buffer(s)
Hash table size 2267, node heap has 3 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
...
```

要检查的第一点是信号量部分。如果自适应哈希索引是争用的主要来源，则 btr0sea.ic 文件周围会有信号量（其中自适应哈希索引在源代码中实现）。如果您偶尔看到信号量（但很少看到），这不一定是问题，但如果看到频繁和长信号量，则最好禁用自适应哈希索引。

兴趣的另一部分是插入缓冲区和自适应哈希索引的部分。这包括用于哈希索引的内存量，并使用哈希和非哈希搜索应答速率查询。请注意，这些速率用于在监视器输出顶部附近列出的期间 - 在示例中，在 2019-05-05 05 17:22:14 之前的最后 16 秒。

支持的索引类型的讨论到结束。索引仍然更多，因为有几个功能值得自己熟悉。

## Index Features

知道存在哪些类型的索引是一回事，但另一件事是能够充分利用它们。为此，您需要了解有关 MySQL 中提供的索引相关功能。这些范围从按相反顺序对索引中的值进行排序到函数索引和自动生成的索引。本节将介绍这些功能，以便您可以在日常工作中使用它们。

### Functional Indexes

到目前为止，索引已直接应用于列。这是添加索引的最常见方法，但在某些情况下，您需要使用派生值。例如，请求所有在 5 月有生日的人员的查询：

```sql
DROP TABLE IF EXISTS db1.person;
CREATE TABLE db1.person (
 Id int unsigned NOT NULL,
 Name varchar(50),
 Birthdate date NOT NULL,
 PRIMARY KEY (Id)
);
SELECT *
 FROM db1.person
 WHERE MONTH(Birthdate) = 5;
```

如果在"出生日期"列上添加索引，则不能用于应答该查询，因为日期根据其完整值存储，并且与列的最左侧不匹配。（另一方面，搜索所有出生于 1970 年的人可以使用"出生日期"列上的 B 树索引。

实现此值的一个方法就是使用派生值生成列。在 MySQL 5.7 及更晚的中，您可以告诉 MySQL 自动使列保持最新，例如：

```sql
CREATE TABLE db1.person (
 Id int unsigned NOT NULL,
 Name varchar(50) NOT NULL,
 Birthdate date NOT NULL,
 BirthMonth tinyint unsigned
 GENERATED ALWAYS AS (MONTH(Birthdate))
 VIRTUAL NOT NULL,
 PRIMARY KEY (Id),
 INDEX (BirthMonth)
);
```

在 MySQL 8.0.13 中，有一种更直接的方法来实现此目的。您可以直接索引函数的结果：

```sql
CREATE TABLE db1.person (
 Id int unsigned NOT NULL,
 Name varchar(50) NOT NULL,
 Birthdate date NOT NULL,
 PRIMARY KEY (Id),
 INDEX ((MONTH(Birthdate)))
);
```

使用函数索引的优点是，它更明确了要索引的索引，并且没有额外的"出生月"列。否则，添加功能索引的两种方法的工作方式相同。

### Prefix Indexes

表的索引部分变得大于表数据本身的情况并不少见。如果为大型字符串值编制索引，情况尤其如此。B 树索引的索引数据的最大长度也有限制 ， 使用 DYNAMIC 或 COMPRESSED 行格式的 InnoDB 表有 3072 字节，其他表的索引数据长度较小。这实际上意味着您无法索引文本列，更不用说长文本列了。缓解大型字符串索引的一种方法就是仅索引该值的第一部分。这称为前缀索引。

通过指定要索引的二进制对象的字符串字符数或字节数来创建前缀索引。如果要索引城市表中 Name 列的前十个字符（来自世界数据库），可以像

```sql
ALTER TABLE world.city ADD INDEX (Name(10));
```

请注意如何在括号中添加要索引的字符数。只要您选择足够的字符来提供良好的选择性，此索引将几乎与索引整个名称一样有效，并且上行时，它使用的存储和内存更少。您需要包含多少个字符？这完全取决于您索引的数据。您可以查询数据，了解前缀的独特性。清单 14-11 显示了检查有多少城市名称共享前十个字符的示例。

```
Listing 14-11. The frequency of city names based on the first ten characters
mysql> SELECT LEFT(Name, 10), COUNT(*),
 COUNT(DISTINCT Name) AS 'Distinct'
 FROM world.city
 GROUP BY LEFT(Name, 10)
 ORDER BY COUNT(*) DESC, LEFT(Name, 10)
 LIMIT 10;
+----------------+----------+----------+
| LEFT(Name, 10) | COUNT(*) | Distinct |
+----------------+----------+----------+
| San Pedro      | 6        | 6        |
| San Fernan     | 5        | 3        |
| San Miguel     | 5        | 3        |
| Santiago d     | 5        | 5        |
| San Felipe     | 4        | 3        |
| San José       | 4        | 1        |
| Santa Cruz     | 4        | 4        |
| São José d     | 4        | 4        |
| Cambridge      | 3        | 1        |
| Ciudad de      | 3        | 3        |
+----------------+----------+----------+
10 rows in set (0.0049 sec)
```

这表明，使用此索引前缀，您最多将读取六个城市以查找匹配项。虽然这不仅仅是一个完整的匹配，它仍然比扫描所有表好得多。在此比较中，您当然还需要验证前缀匹配数是因前缀冲突还是城市名称相同。例如，对于"剑桥"，有三个城市具有该名称，因此，无论您是索引前十个字符还是整个名称，都无区别。您可以对不同的前缀长度进行此类分析，以了解阈值，其中增加索引的大小会提供小回报。在许多情况下，索引不需要这么多字符才能正常工作。

如果您认为可以删除索引，或者想要推出索引，但不允许它立即生效，该怎么办？答案是不可见的索引。



### Invisible Indexes

MySQL 8 引入了一项称为不可见索引的新功能。它允许您有一个已维护并准备使用的索引，但优化器将忽略该索引，直到您决定使其可见。这允许您在复制拓扑中推出新索引，或禁用您认为不需要或类似索引。您可以快速启用或禁用索引，因为它只需要更新表的元数据，因此更改是"即时"。

例如，如果您认为不需要索引，则首先使其不可见，允许您在告诉 MySQL 删除索引之前监视数据库在没有它的情况下的工作方式。如果发现某些查询（例如，在您监视的期间内未执行的月度报告查询）确实需要索引，您可以快速重新创建索引。

使用"不可见"关键字将索引标记为不可见，使用"可见"关键字使不可见索引再次可见。例如，若要在 world.city 表的"名称"列上创建索引为不可见，并且要使其以后可见，可以使用

```
mysql> ALTER TABLE world.city ADD INDEX (Name) INVISIBLE;
Query OK, 0 rows affected (0.0649 sec)
Records: 0 Duplicates: 0 Warnings: 0
mysql> ALTER TABLE world.city ALTER INDEX Name VISIBLE;
Query OK, 0 rows affected (0.0131 sec)
Records: 0 Duplicates: 0 Warnings: 0
```

如果禁用索引，并且查询使用引用隐藏索引的索引提示，则查询将返回错误：

```
ERROR: 1176: Key 'Name' doesn't exist in table 'city'
```

您可以通过启用优化器开关来覆盖索引的隐身性（use_invisible_indexes关闭）。如果您遇到问题，因为索引已不可见，并且无法立即重新创建索引，或者您希望在使其普遍可用之前使用新索引进行测试，则此功能非常有用。暂时为连接启用不可见索引的示例是

```sql
SET SESSION optimizer_switch = 'use_invisible_indexes=on';
```

即使启用use_invisible_indexes优化器开关，也不允许在索引提示中引用索引。

MySQL 8 的另一个新功能是索引降序。

### Descending Indexes

在 MySQL 5.7 及更旧版本中，当您添加 B 树索引时，始终按升序排序。这非常适合查找精确匹配项、按索引的升序检索行等。但是，虽然升序索引可以加快查询按降序查找行的速度，但它们并不有效。MySQL 8 添加了降序索引，以帮助处理这些用例

利用降序索引，您不需要做任何特别操作。所有所需的是 DESC 关键字与索引一起使用，例如：

```sql
ALTER TABLE world.city ADD INDEX (Name DESC);
```

如果索引中有多列，则不需要以升序或降序包含这些列。您可以混合升序和降序列，因为它在查询中效果最佳。

### Partitioning and Indexes

如果创建分区表，则分区列必须是主键和所有唯一键的一部分。原因是 MySQL 没有全局索引的概念，因此必须确保唯一性检查只需要考虑单个分区。

关于性能调优，则分区可用于有效地使用两个索引来解决查询，而无需使用索引合并。当用于分区的列在查询中的条件中使用时，MySQL 将修剪分区，因此仅搜索条件可以匹配的分区。然后，可以使用索引解析查询的其余部分。

考虑一个t_part根据"已创建"列进行分区的表，该列是时间戳，每月有一个分区。如果在 2019 年 3 月查询值小于 2 的 val 列的所有行，则查询将首先修剪创建值上的分区，然后在 val 上使用索引。清单14-12显示了这方面的一个示例。

```
Listing 14-12. Combining partition pruning and filtering using an index
mysql> CREATE TABLE db1.t_part (
 id int unsigned NOT NULL AUTO_INCREMENT,
 Created timestamp NOT NULL,
 val int unsigned NOT NULL,
 PRIMARY KEY (id, Created),
 INDEX (val)
) ENGINE=InnoDB
 PARTITION BY RANGE (unix_timestamp(Created))
(PARTITION p201901 VALUES LESS THAN (1548939600),
 PARTITION p201902 VALUES LESS THAN (1551358800),
 PARTITION p201903 VALUES LESS THAN (1554037200),
 PARTITION p201904 VALUES LESS THAN (1556632800),
 PARTITION p201905 VALUES LESS THAN (1559311200),
 PARTITION p201906 VALUES LESS THAN (1561903200),
 PARTITION p201907 VALUES LESS THAN (1564581600),
 PARTITION p201908 VALUES LESS THAN (1567260000),
 PARTITION pmax VALUES LESS THAN MAXVALUE);
1 row in set (5.4625 sec)
-- Insert random data
-- 1546261200 is 2019-01-01 00:00:00 UTC
-- The common table expression (CTE) is just
-- a convenient way to quickly generate 1000 rows.
mysql> INSERT INTO db1.t_part (Created, val)
 WITH RECURSIVE counter (i) AS (
 SELECT 1
 UNION SELECT i+1
 FROM counter
 WHERE i < 1000)
 SELECT FROM_UNIXTIME(
 FLOOR(RAND()*(1567260000-1546261200))
 +1546261200
 ), FLOOR(RAND()*10) FROM counter;
Query OK, 1000 rows affected (0.0238 sec)
Records: 1000 Duplicates: 0 Warnings: 0
mysql> EXPLAIN
 SELECT id, Created, val
 FROM db1.t_part
 WHERE Created BETWEEN '2019-03-01 00:00:00'
 AND '2019-03-31 23:59:59'
 AND val < 2\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: t_part
 partitions: p201903
 type: range
possible_keys: val
 key: val
 key_len: 4
 ref: NULL
 rows: 22
 filtered: 11.110000610351562
 Extra: Using where; Using index
1 row in set, 1 warning (0.0005 sec)
```

使用t_part的 Unix 时间戳按范围对该表进行分区。EXPLAIN 输出（第 20 章中详细介绍了 EXPLAIN）显示，查询中仅包含 p201903 分区，val 索引将用作索引。解释的确切输出可能不同，因为该示例使用随机数据。

到目前为止，有关索引的所有讨论都是为显式创建的索引。对于某些查询，MySQL 还可以自动生成索引。这是最后要讨论的索引功能。

### Auto-generated Indexes

对于包含联接到其他表或子查询的子查询，联接可能非常昂贵，因为子查询不能包含显式索引。为了避免对子查询生成的这些临时表执行完整表扫描，MySQL 可以在联接条件上添加自动生成的索引。

例如，考虑 sakila 示例数据库中的胶片表。它有一个专栏release_year与电影上映的一年。如果要查询在有数据的每个年发布多少部电影，可以使用以下查询（是的，无需子查询即可更好地编写此查询，但以这样方式编写以演示自动生成的索引功能）：

```sql
SELECT release_year, COUNT(*)
 FROM sakila.film
 INNER JOIN (SELECT DISTINCT release_year
 FROM sakila.film
 ) release_years USING (release_year)
 GROUP BY release_year;
```

MySQL 选择在胶片表上执行完整的表扫描，并在子查询上添加自动生成的索引。当 MySQL 添加自动生成的索引时，EXPLAIN 输出将包括 <auto_key0>（或 0 替换为不同的值）作为可能的键和已用键。

自动生成的索引可以显著提高查询的性能，这些查询包括优化器无法作为正常联接重写的子查询。最好的一切是它会自动发生。

索引特征的讨论到今天结束。在讨论如何使用索引之前，还需要了解 InnoDB 如何使用索引。



## InnoDB and Indexes

InnoDB 自 1990 年代中期首次发布以来，其表的组织方式是使用群集索引来组织数据。这一事实导致了一个普遍的说法，即 InnoDB 中的一切都是索引。数据的组织实际上是一个索引。默认情况下，InnoDB 使用群集索引的主要键。如果没有主键，它将查找不允许 NULL 值的唯一索引。作为最后的手段，使用自动递增计数器将隐藏列添加到表中。

对于索引组织的表，InnoDB 中的所有内容都是索引。群集索引本身被组织为 B+树索引，包含叶页中的实际行数据。这在查询性能和索引时会产生一些后果。下一节将介绍 InnoDB 如何使用主键以及它对于辅助键意味着什么，提供一些建议，并查看索引组织表的最佳用例。

### The Clustered Index

由于数据是按照聚类索引（主键或替代项）组织的，因此选择主键非常重要。如果在现有值之间插入具有主键值的新行，InnoDB 必须重新组织数据，为新行提供空间。在最坏的情况下，InnoDB 必须将现有页面拆分为两个页面，因为页面大小固定。页面拆分会导致叶页在基础存储上出现顺序，导致更多的随机 I/O，进而导致查询性能下降。页面拆分将在第 25 章中作为 DDL 和批量数据加载的一部分进行讨论。

### Secondary Indexes

辅助索引的叶页存储对行本身的引用。由于行根据群集索引存储在 B+树索引中，因此所有辅助索引都必须包含群集索引的值。如果选择了值需要许多字节的列，例如，具有长字符串和可能多字节字符串的列，这大大增加了辅助索引的大小。

这也意味着，当您使用辅助索引执行查找时，将执行两个索引查找：首先是预期的辅助键查找，然后从叶页提取主键值，用于主键查找以获取实际数据。

对于非唯一二级索引，如果您有显式主键或 NOT NULL 唯一索引，则它是添加到索引的主键使用的列。MySQL 知道这些额外的列，即使它们尚未明确成为索引的一部分，如果 MySQL 将改进查询计划，则 MySQL 将使用它们。

### Recommendations

由于 InnoDB 使用主键的方式以及它添加到辅助索引的方式，因此最好使用使用尽可能少字节的单调递增主键。自动递增整数满足这些属性，从而产生良好的主键。

如果表没有任何合适的索引，则用于群集索引的隐藏列使用自动递增（如计数器）来生成新值。但是，由于该计数器对于具有隐藏主键的 MySQL 实例中所有 InnoDB 表都是全局的，因此它可能成为争用点。隐藏密钥也不能在复制中用于查找受事件影响的行，并且组复制需要主键或 NOT NULL 唯一索引才能进行冲突检测。因此，建议始终明确选择所有表的主要键。

另一方面，UUID 不是单调的增量，不是一个好的选择。MySQL 8 中的一个选项是使用 UUID_TO_BIN( ) 函数，第二个参数设置为 1，这将使 MySQL 交换第一组和第三组十六进制数字。第三组是 UUID 时间戳部分的高字段，因此，在 UUID 的开头，有助于确保 ID 不断增加并存储它们，因为与十六进制值相比，二进制数据需要的存储量不到十六进制值的一半。

### Optimal Use Cases

索引组织表对于使用该索引的查询特别有用。正如名称"群集索引"所暗示的，具有群集索引值相似的行彼此存储。由于 InnoDB 始终将整个页面读取到内存中，这也意味着主键的两行值相似，因此可能一起读取。如果在查询中或不久之后执行的查询中都需要同时使用，则第二行已在缓冲池中可用。

现在，您应该对 MySQL 中的索引以及 InnoDB 如何使用索引（包括其数据组织）有良好的背景知识。现在是把所有问题放在一起讨论指数策略的时候。

## Index Strategies

在索引方面，最大的问题是要索引什么，其次要使用什么样的索引以及使用哪些索引功能。无法创建最终的分步指令以确保最佳索引;为此，需要体验并充分了解架构、数据和查询。不过，可以给出一些一般准则，因为本节将讨论这一点。

首先要考虑的是何时应该添加索引;是否应在最初创建表时或以后进行。然后是主键的选择和如何选择的考虑。最后，还有辅助索引，包括要向索引添加的列数以及索引是否可以用作覆盖索引。

### When Should You Add or Remove Indexes?

索引维护是一项永无止境的任务。它从首次创建表时开始，并在表的生存期内继续。不要轻心不从心地使用索引工作 – 如前所述，编制索引的差别可能是几个数量级。您无法通过将更多的硬件资源投入到索引不佳的情况下来保护自己免受不足。索引不仅影响原始查询性能，还影响锁定（如第 18 章中将进一步讨论）、内存使用情况和 CPU 使用情况。

创建表时，应特别花时间选择良好的主键。主键在表的生命周期中通常不会更改，如果您决定更改主键，则使用索引组织的表，则必然需要完全重新生成表。随着时间的推移，辅助索引可以调整到更大程度。事实上，如果您计划为表的初始总体导入大量数据，最好等到加载数据后再添加辅助索引。可能的异常是唯一的索引，因为它们是数据验证所需的。

创建表并用其初始数据填充表后，需要监视表的使用情况。sys 架构中有两个视图可用于查找具有完整表扫描的表和语句：

- schema_tables_with_full_table_scans：此视图显示所有在不使用索引的情况下读取行且按该数字降序排序的表。如果表在不使用索引的情况下读取大量行，则可以使用此表查找查询，并查看索引是否会有所帮助。该视图基于 table_io_waits_summary_by_index_usage 性能架构表，也可以直接使用，例如，如果要执行更高级的分析，例如查找不使用索引读取的行的百分比
- statements_with_full_table_scans：此视图显示不使用索引或不使用良好索引的语句的规范化版本。语句按它们未使用索引的执行次排序，然后按它们未使用良好索引的时间数排序 - 两者均按降序排列。该视图基于性能events_ statements_summary_by_digest表。

第19章和第20章将更详细地介绍这些视图的使用以及基础性能架构表。

当您确定查询可以从其他索引中受益时，您需要评估在执行查询时是否值得获得额外收益的成本。

同时，您还需要关注您是否具有不再使用的索引。性能架构和 sys 架构对于查找未使用或未使用太多索引特别有用。三个有用的系统架构视图是

- schema_index_statistics：此视图具有使用给定索引读取、插入、更新和删除行的统计信息。与schema_tables_with_full_table_scan视图schema_index_statistics，它基于table_io_waits_ summary_by_index_usage架构表。
- schema_unused_indexes：此视图将返回自上次重置数据以来未使用的索引的名称（不会超过上次重新启动以来）。此视图还基于性能table_io_waits_summary_by_index_usage表。
- schema_redundant_indexes：如果您有两个索引覆盖同一列，则 InnoDB 的工作量将增加一倍，以使索引保持最新，并给优化器增加负担，但不会获得任何收益。如果schema_redundant_indexes，该视图可以作为名称建议用于查找冗余索引。该视图基于统计信息架构表。

使用前两个视图时，必须记住数据来自性能架构中的内存表中的数据。如果您有一些查询只是偶尔执行，统计信息可能无法反映您的整体索引需求。这是不可见索引功能可以派上用场的情况之一，因为它允许您禁用索引，同时保留索引，直到您确定可以安全地删除它。如果事实证明一些很少执行的查询需要索引，可以轻松地再次启用索引。

如前所述，首先考虑的是选择什么作为主键。您应该包括哪些列？这是下一件事要讨论。

### Choice of the Primary Key

使用索引组织的表时，主索引的选择非常重要。主键会影响随机和顺序 I/O 之间的比率、辅助索引的大小以及需要读取到缓冲池中的页数。InnoDB 表的主要键始终是 B+树索引。

与群集索引有关的最佳主键尽可能小（以字节为单位），保持单调增长，并在短时间内对频繁查询的行进行分组。实际上，在什么情况下，可能需要做出最好的妥协，因此不可能完成所有这些任务。对于许多工作负载，根据表预期的行数自动递增无符号整数（int 或 bigint）是一个不错的选择;但是，可能有一些特殊注意事项，例如跨多个 MySQL 实例的唯一性要求。主键最重要的功能是，它应尽可能按顺序排列且不可变。如果更改行主键的值，则需要将整个行移动到群集索引中的新位置。

**提示自动递增通常是作为主键的一个无符号整数。它不断单调地递增，不需要太多的存储，并且它会在群集索引中将最近的行分组在一起。**

您可能认为隐藏的主键可能与任何其他列一样是群集索引的最佳选择。毕竟，它是一个自动递增的整数。但是，隐藏密钥有两个主要缺点：它只标识该本地 MySQL 实例的行，并且计数器是全局所有 InnoDB 表（在实例中），没有用户定义的主键。隐藏密钥仅在本地有用，这意味着在复制中，隐藏值不能用于标识要更新副本的行。计数器是全局的，这意味着在插入数据时，它可能成为争用点并导致性能下降。

底线是，应始终显式定义要作为主键。对于辅助索引，有更多的选择，因为它将看到下一个。

### Adding Secondary Indexes

辅助索引是不是主键的所有索引。它们可以是唯一的，也可以不是唯一的，您可以在所有受支持的索引类型和功能之间进行选择。如何选择要添加的索引？本节将使您更容易做出该决定。

注意不要在前面的表中添加太多的索引。索引具有开销，因此当您添加最终未使用的索引时，查询和系统整体的性能将更差。这并不意味着在创建表时不应添加任何辅助索引。

只是你需要考虑一下。执行查询时，辅助索引可以通过多种方式使用。其中一些如下：

- 减少检查的行：当您具有WHERE子句或连接条件以查找所需的行而不扫描整个表时，将使用此行。
- 排序数据：B树索引可用于读取
    查询所需顺序的行，允许MySQL绕过排序步骤。
- 验证数据：这就是唯一索引的唯一性。
- 避免读取行：覆盖索引可以返回所有必需的数据，而无需读取整个行。
- 查找MIN( )和MAX( )值：对于GROUP BY查询，只需检查索引中的第一条和最后一条记录，就可以找到索引列的最小值和最大值。

主键显然也可用于所有这些目的。从查询的角度来看，主键和辅助键之间没有区别

当您需要决定是否添加索引时，您需要问自己索引需要哪些用途，以及它是否能够实现这些目的。确认为这种情况后，可以查看应为多列索引添加哪些顺序列，以及是否应添加其他列。接下来的两个小节将对此进行更详细的讨论。

### Multicolumn Index

只要不超过索引的最大宽度，最多可以向索引添加 16 列或功能部件。这既适用于主键，也适用于辅助索引。InnoDB 每个索引限制为 3072 字节。如果使用可变宽度字符集包含字符串，则计算索引宽度的最大可能宽度。

向索引添加多个列的优点是，它允许您将索引用于多个条件。这是提高查询性能的非常有效的方法。例如，考虑查询查找给定国家/地区中对城市人口的最低要求的城市：

```sql
SELECT ID, Name, District, Population
 FROM world.city
 WHERE CountryCode = 'AUS'
 AND Population > 1000000;
```

您可以使用"国家代码"列上的索引查找国家/地区代码设置为 AUS 的城市，并且可以使用"人口"列上的索引查找人口超过 100 万的城市。更妙的是，您可以将它合并为一个包含两列的索引。

如何做到这一点很重要。国家/地区代码使用相等的引用，而总体是范围搜索。索引中的列用于范围搜索或排序后，除了作为覆盖索引的一部分外，索引中的列也不再使用。对于此示例，您需要在"总体"列之前添加"国家/地区代码"列，以便对以下两个条件使用索引：

```sql
ALTER TABLE world.city
 ADD INDEX (CountryCode, Population);
```

在此示例中，索引甚至可用于使用总体对结果进行顺序。

如果需要添加多个用于相等条件的列，则需要考虑以下两点：哪些列是使用最多的列，以及列筛选数据的情况。当索引中有多列时，MySQL 将仅使用索引的左前缀。例如，如果您有一个索引 （col_a，col_b，col_c），则只能使用索引在 col_b 上筛选，如果您还对 col_a 进行筛选（这必须是相等条件）。所以你需要仔细选择订单。在某些情况下，可能需要为同一列添加多个索引，其中列顺序在索引之间有所不同。

如果无法根据使用情况决定按哪个顺序包含列，请先使用最具选择性的列添加它们。下一章将讨论索引的选择性，但简言之，列的值越明显，其选择性越高。通过首先添加最具选择性的列，可以更快地缩小索引部分包含的行数。

您可能还需要包括不用于筛选的列。你为什么要那么做？答案是，它可以帮助形成覆盖索引。

### Covering Indexes

覆盖索引是表上的索引，其中给定查询的索引包括该表中所需的所有列。这意味着，当 InnoDB 到达索引的叶页时，它拥有它所需的所有信息，并且不需要读取整行。根据表的不同，这可能会提高查询性能，尤其是使用它来排除行的很大部分（如大文本或 Blob 列） 时。

还可以使用覆盖索引来模拟辅助群集索引。请记住，群集索引只是一个 B+树索引，整个行都包含在叶页中。覆盖索引具有叶页中行的完整子集，因此模拟该列子集的群集索引。与群集索引一样，任何 B 树索引将类似的值分组在一起，因此它可用于减少读取到缓冲池中的页数，并且它有助于执行索引扫描时按顺序 I/O。

但是，与群集索引相比，覆盖索引有几个限制。覆盖索引仅模拟用于读取的群集索引。如果需要写入数据，更改始终必须访问群集索引。另一件事是，由于 InnoDB 的多版本并发控制 （MVCC），即使您使用覆盖索引，也有必要检查群集索引以验证行的另一个版本是否存在。

加索引时，值得考虑索引所针对的查询需要哪些列。即使索引不会用于筛选或排序这些列，也值得添加 select 部分中使用的任何额外列。您需要平衡覆盖索引的好处与索引的附加大小。因此，如果您只错过一两个小列，此策略非常有用。覆盖索引的好处查询量越多，可以接受添加到索引的额外数据。

## 总结

本章是索引世界的旅程。良好的索引策略可能意味着数据库停止磨削和油井机的差异。索引有助于减少查询中检查的行数，此外，覆盖索引可以避免读取整个行。另一方面，在存储和持续维护方面，都有与索引相关的开销。因此，有必要平衡对索引的需要和拥有索引的成本。

MySQL 支持几种不同的索引类型。最重要的是 B 树索引，这也是 InnoDB 使用群集索引组织其索引组织表中的行。其他索引类型包括全文索引、空间 （R 树） 索引、多值索引和哈希索引。后一种类型在 InnoDB 中很特别，因为它仅使用自适应哈希索引功能，该功能决定自动添加哪些哈希索引。

讨论了一系列索引功能。函数索引可用于索引在表达式中使用列的结果。前缀索引可用于减小文本和二进制数据类型的索引大小。在推出新索引期间或软删除现有索引时，可以使用不可见索引。降序索引可提高按降序遍历索引值的有效性。索引在分区方面也起着一定的作用，您可以使用分区有效地实现对查询中单个表使用两个索引的支持。最后，MySQL 能够自动生成与子序列相关的索引。

本章的最后一部分从 InnoDB 的具体细节以及使用索引组织表的注意事项开始。这些查询最适合主键与相关查询，但对以随机主键顺序插入的数据和按辅助索引查询数据效果不太好。

最后一节讨论了索引策略。首次创建表时，请仔细选择主键。根据指标的观察值，可以随着时间的推移在更大范围内添加和删除辅助索引。您可以使用多列索引使用索引筛选多个列和/或排序。最后，覆盖索引可用于模拟辅助群集索引。

结束讨论什么是索引以及何时使用它们。索引要多一些，在讨论索引统计信息时，下一章将可以看到。