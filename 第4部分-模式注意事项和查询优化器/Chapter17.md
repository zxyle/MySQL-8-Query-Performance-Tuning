# 查询优化器

当您向 MySQL 提交查询以执行时，它不像只读取数据并返回数据那么简单。诚然，对于从单个表中请求所有数据的简单查询，使用检索数据的选项并不多。但是，大多数查询都更为复杂（有些则复杂得多）。执行查询完全按照编写的方式执行绝不是获得结果最有效的方法。在阅读有关索引时，您已经触及了其中一些复杂性。您可以添加到索引的选择、联接顺序、用于执行联接的算法、各种联接优化等。这就是优化器发挥作用的地方。

优化器的主要工作是准备查询以执行并确定最佳查询计划。第一阶段涉及对查询进行转换，目的是以比原始查询更低的成本执行重写的查询。第二阶段包括计算可以执行查询的各种方法的成本，并确定最便宜的选项。

本章首先讨论转换和基于成本的优化。然后，本章继续讨论基本联接算法，然后是其他优化功能，如批处理密钥访问。本章的最后一部分介绍如何配置优化器以及如何使用资源组来确定查询的优先级。

## 转换

人类自然地发现编写查询的方式可能与在 MySQL 中执行查询的最佳方式不一样。优化器知道多个更改查询，同时仍返回相同的结果，因此查询对于 MySQL 更加最佳。

当然，原始查询和重写查询返回相同的结果至关重要。幸运的是，关系数据库基于数学集理论，因此许多转换可以使用标准的数学规则来确保查询的两个版本返回相同的结果（bar 实现错误）。

优化器进行的最简单转换类型之一是恒定传播。作为示例，请考虑以下查询：

```sql
SELECT *
 FROM world.country
 INNER JOIN world.city
 ON city.CountryCode = country.Code
 WHERE city.CountryCode = 'AUS';
```

此查询有两个条件：/地区代码"列必须等于"AUS"，代码"列/地区表的列。从这两个条件，可以得出，列还必须等于"AUS"。优化器使用此知识直接筛选国家/地区表。由于"列是国家，这意味着优化器知道只有一行匹配条件，并且优化器可以将视为常量。实际上，查询最终以国家/地区表中的列作为常量在选择列表中执行，并扫描城市表中的条目，使用国家/地区代码=

```sql
SELECT 'AUS' AS `Code`,
 'Australia' AS `Name`,
 'Oceania' AS `Continent`,
 'Australia and New Zealand' AS `Region`,
 7741220.00 AS `SurfaceArea`,
 1901 AS `IndepYear`,
 18886000 AS `Population`,
 79.8 AS `LifeExpectancy`,
 351182.00 AS `GNP`,
 392911.00 AS `GNPOld`,
 'Australia' AS `LocalName`,
 'Constitutional Monarchy, Federation' AS `GovernmentForm`,
 'Elisabeth II' AS `HeadOfState`,
 135 AS `Capital`,
 'AU' AS `Code2`,
 city.*
 FROM world.city
 WHERE CountryCode = 'AUS';
```



从性能的角度来看，这是一个安全的转换。其他转换更为复杂，并不总是能提高性能。因此，无论是否启用优化，都可以进行配置。配置使用"optimizer_switch器提示"完成，这些提示在以及如何配置优化器时进行了讨论。

优化器确定要执行哪些转换后，需要确定如何执行重写的查询，接下来将讨论。

## 基于成本的优化

MySQL 使用基于成本的查询优化。这意味着优化器计算执行查询所需的各种操作的成本，然后合并这些部分成本来计算可能的查询计划的总体查询成本，并选择最便宜的计划。本节介绍估计查询计划成本的原则。

### 基础知识：单表选择

无论查询如何，计算成本的原则都是一样的，但很明显，查询越复杂，成本估算就变得越复杂。作为一，请考虑查询索引列上具有子句的单个表的查询：

```sql
SELECT *
 FROM world.city
 WHERE CountryCode = 'IND';
```

在"国家代码"列上具有辅助一索引，从表定义中可以看到：

```sql
mysql> SHOW CREATE TABLE world.city\G
**************************** 1. row ****************************
 Table: city
Create Table: CREATE TABLE `city` (
 `ID` int(11) NOT NULL AUTO_INCREMENT,
 `Name` char(35) NOT NULL DEFAULT ",
 `CountryCode` char(3) NOT NULL DEFAULT ",
 `District` char(20) NOT NULL DEFAULT ",
 `Population` int(11) NOT NULL DEFAULT '0',
 PRIMARY KEY (`ID`),
 KEY `CountryCode` (`CountryCode`),
 CONSTRAINT `city_ibfk_1` FOREIGN KEY (`CountryCode`) REFERENCES `country`
(`Code`)
) ENGINE=InnoDB AUTO_INCREMENT=4080 DEFAULT CHARSET=utf8mb4
COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.0008 sec)
```

优化器可以选择两种方式来获取匹配的行。一种方法就是使用国家代码上的索引中的匹配行，然后查找请求的行值。另一种方式是执行完整的表扫描并检查每行以确定它是否满足筛选器条件。

这些访问哪一个具有最低成本（最快）并不像看起来那么简单。这取决于几个因素：

- 通过辅助索引读取行需要首先查找索引中的行，然后可能（请参阅下一项）进行主键查找以获取该行。这意味着使用辅助索引检查和检索行比直接读取行更昂贵，并且索引访问比表扫描总体便宜，索引必须显著减少要检查的行数。索引越有选择性，使用它就相对便宜。
- 如果索引包含查询所需的所有列，则可以跳过实际行中的读取，使其更有利于使用索引。
- 这再次取决于几个因素，例如索引和行数据是否已在缓冲池中，如果没有，记录可以从磁盘读取的速度。在读取索引和读取聚类索引之间切换时，使用索引将需要更多的随机 I/O，因此查找记录所涉及的寻线时间变得非常重要。

MySQL 8 中的一个新功能是，优化器可以询问 InnoDB 查询所需的记录是否可以在缓冲池中找到，或者是否需要从磁盘读取。这极大地有助于改进查询计划。

由于 MySQL 不知道硬件的性能特征，因此读取记录所涉及的成本问题更为复杂。默认情况下，MySQL 假定从磁盘读取的成本是内存的四倍。这可按"配置优化器"部分中的"发动机成本"部分中所述进行配置。

一旦将第二个表引入查询，优化器还需要决定以什么顺序加入表。

### 表联接顺序

对于比单个表 SELECT 语句优化器不仅需要考虑访问每个表的成本，还需要考虑每个表包括的顺序以及每个表要使用的索引。

对于外部联接和直线联接是固定的，但对于内部联接，优化器可以自由选择顺序，因此优化器必须计算每个组合的成本。可能的组合数量是 N！（因子），它缩放非常差。如果有五个表参与内部联接，则优化器可以选择五个表作为第一个表，然后选择第二个表的四个表，第三个表的三个表，第四个表的两个表，最后一个表选择一个表：

组合 = 5 * 4 * 3 * 2 * 1 * 5！120

MySQL 支持联接多达 61 个表，在这种情况下，可能有 5.1E83 个组合来计算成本过高，并且可能需要比执行查询本身更长的时间。因此，默认情况下，优化器根据对成本的部分评估修剪查询计划，因此仅对最有希望的计划进行充分评估。还可以告诉优化器在包含给定数量的表后停止评估成本。修剪和搜索深度分别配置了和选项，这将在"配置优化器"部分中讨论。

最佳联接顺序与表的以及筛选器减少每个表中的行数的工作效果有关。

### 默认筛选效果

当您联接两个或多个表时，优化器需要知道每个表中包含多少行，以便能够确定最佳联接顺序。这绝不是一件微不足道的任务。

使用索引时，当筛选器与其他表不相关时，优化器可以非常准确地估计索引的匹配行数。如果没有索引，可以使用直方图统计信息来获取良好的筛选估计值。当筛选的列没有统计信息时，会出现困难。在这种情况下，优化器将回放在内置的默认估计值上。表包括筛选效果的示例，这些示例在没有可以使用索引或直方图统计信息时使用。

| 类型       | 过滤器  %               | 备注/示例                                  |
| :--------- | :---------------------- | :----------------------------------------- |
| All        | 100                     | 当按索引筛选或没有筛选条件时，使用此选项。 |
| Equality   | 10                      | 名称 = "悉尼"                              |
| Not Equal  | 90                      | 名称 <> "悉尼"                             |
| Inequality | 33.33                   | 人口 > 4000000                             |
| Between    | 11.11                   | 人口在 1000000 和 4000000 之间             |
| IN         | 分钟 （#items * 10，50) | 名称 IN（"悉尼"，"墨尔本"）                |

筛选效果基于 Selinger 等人的文章"关系数据库管理系统中的访问路径有时会看到不同的筛选值。一些示例包括

- 这包括项位数据类型。考虑中的"。这是一个个值的步答，因此优化器将估计，对于 WHERE 子句（如大陆 = "）的筛选1/7 。
- 如果表中的行数少于 10 行，并且添加相等条件，则筛选估计值将为对于不相等的筛选估计值。。
- 如果将筛选器合并到多个非索引列上，则估计筛选效果是组合效果。例如， 对于，过滤器名称采取 10% 的行，由于性，由于对人口的不平等33% 的行，因此综合效果是。

此列表并非详尽无遗，但它应该让您了解 MySQL 如何到达筛选估计值。默认筛选效果显然不是很准确，尤其是对于大型表，因为数据不遵循这种严格的规则。这就是为什么索引和直方图对于获得良好的查询计划如此重要的原因。

在确定查询计划结束时，对和整个查询都有成本估算。这些内容可以丰富，以了解优化器如何到达查询执行计划。

### 查询成本

如果要检查优化器找到的成本，则需要使用树-（包括）或 JSON 格式的、MySQL 工作台或优化器跟踪。这些都在第。

作为一个简单的示例，请考虑一个数据库的国家表：

```sql
SELECT *
 FROM world.country
 INNER JOIN world.city
 ON CountryCode = Code;
```

图显示了查询的可视化解释图，包括城市表详细信息。

![](../附图/Figure%2017-1.png)

该图显示了优化器如何决定执行查询。如何阅读图表将在第。这里的重要部分是箭头指向的数字。这些是优化器已针对查询执行的各个部分以更低的成本到达的成本估计值，越好。该示例显示，成本估算是为非常特定的任务（如读取数据、评估筛选器条件等）计算的。在关系图的顶部，总查询成本估计为 1535.43。

执行查询后，还可以从用户状态变量获取成本。清单显示了对图。

```
Listing 17-1. Obtaining the estimated query cost after executing a query
mysql> SELECT *
 FROM world.country
 INNER JOIN world.city
 ON CountryCode = Code;
...
mysql> SHOW SESSION STATUS LIKE 'Last_query_cost';
+-----------------+-------------+
| Variable_name   | Value       |
+-----------------+-------------+
| Last_query_cost | 1535.425669 |
+-----------------+-------------+
1 row in set (0.0013 sec)
```



查询结果已从输出中删除，因为在此讨论中这并不重要。 有关Last_query_cost的重要注意事项是它是估计成本，这就是为什么它在Visual Explain图中显示与总成本相同的值的原因。 如果需要有关执行查询的实际成本的信息，则需要使用EXPLAIN ANALYZE。

Visual Explain图提到使用*嵌套循环*执行查询。 那只是MySQL支持的连接算法之一。

## Join Algorithms

A join is a very broad concept in MySQL – so much that you can argue that everything is a join. Even querying a single table is considered a join. That said, the most interesting joins are those between two or more tables. In this discussion a table can also be a derived table.

When a query is executed, and two tables need to be joined, MySQL has support for three different algorithms. The algorithms are

- Nested loop
- Block nested loop
- Hash join

------

Note The timings shown in this section are for illustrative purposes only. The timings you see on your system will be different, and there may also be differences in the timings relative to each other.

------

This section and the next will reference several names of optimizer switches and optimizer hints. The optimizer switches refer to the optimizer_switch configuration option, and the optimizer hints refer to the /*+ ... */ comments that can be added to queries to tell the optimizer how you would like the query to be executed. Both concepts and how to use them will be discussed further in the section “Configuring the Optimizer” later in this chapter.

### Nested Loop

The nested loop algorithm is the simplest of the algorithms used in MySQL. Until MySQL 5.6 it was also the only algorithm available. As the name suggests, it works by nesting loops with one loop for each table in the join. Not only is the nested join algorithm very simple; it also works well for index lookups.

Consider a query on the world.country table joining on the world.city table querying for countries and cities in Asia. You can write the query in the following way:

```sql
SELECT CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country
 INNER JOIN world.city
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

它将使用嵌套循环执行，该循环在国家/地区，子句中的筛选器，后在城市表上查找。在树表示法中，查询如下所示

```
-> Nested loop inner join
 -> Filter: (country.Continent = 'Asia')
 -> Table scan on country
 -> Index lookup on city using CountryCode
 (CountryCode=country.`Code`)
```

您也可以将此编写为伪代码。使用与 Python 一样语法，可以像以下代码段一样编写嵌套循环联接：

```python
result = []
for country_row in country:
 if country_row.Continent == 'Asia':
 for city_row in city.CountryCode['country_row.Code']:
 result.append(join_rows(country_row, city_row))
```

在伪代码中国家地区和城市城市的国家/地区。国家/是上的"国家代码和"国家代码"表示一行。函数用于表示将所需的列从两行合并到结果集中的一行的过程。

图显示了使用关系图的相同嵌套循环。为简单起见和专注于联接，即使从国家/地区表读取所有行，也只包含匹配行的主要值。

![](../附图/Figure%2017-2.png)

该图显示 MySQL 扫描国家/地区表，直到找到与子句匹配的行。在图中，第一个匹配行是 AFG（对于阿富汗）。然后找到国家代码行等于 1、2、3 和 4），并且每个组合都用于在结果中形成一行。这与等于 ARE（阿拉伯联合酋长国）的国家代码一起继续，直到 YEM（也门）。

在国家/地区表中扫描行索引中扫描的行取决于索引定义以及优化器、执行器和存储引擎中的内部。除非有显式的 ORDER BY 子句，否则绝不应依赖。

通常，联接可能比此示例中更为复杂，因为可能还有其他筛选器。然而，这一概念保持不变。

虽然简单通常是一个很好的属性，但嵌套循环联接有一些限制。它不能用于执行完全外部联接，因为嵌套循环联接需要第一个表返回行，而完全外部联接的情况并不总是如此。解决方法是将完整的外部联接编写为左侧联接和右外部联接的联接。考虑查询所有国家和城市，包括那些国家/地区没有城市，城市没有国家/地区的情况。可以编写为完整的外部联接（在 MySQL 中无效）：

```sql
SELECT *
 FROM world.country
 FULL OUTER JOIN world.city
 ON city.CountryCode = country.Code;
```

为了在 Mysql 中执行， 您可以使用国家联盟城市和国家喜欢

```sql
SELECT *
 FROM world.country
 LEFT OUTER JOIN world.city
 ON city.CountryCode = country.Code
 UNION
SELECT *
 FROM world.country
 RIGHT OUTER JOIN world.city
 ON city.CountryCode = country.Code;
```

另个限制是嵌套循环联接对于无法使用索引的联接不是很有效。由于嵌套循环一次从联接中的第一个表连接一行，因此需要对第一个表中的每一行进行第二个表的完整扫描。这很快就会变得过于昂贵。考虑较早一点检查的查询，其中发现亚洲所有城市：

```sql
mysql> SELECT PS_CURRENT_THREAD_ID();
+------------------------+
| PS_CURRENT_THREAD_ID() |
+------------------------+
| 30                     | 
+------------------------+
1 row in set (0.0017 sec)
SELECT CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country
 INNER JOIN world.city
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

通过国家/地区表（239 行）上的表扫描和城市表上的索引查找，将检查总共 2005 行（在第二个连接中执行此查询）：

```sql
mysql> SELECT rows_examined, rows_sent,
 last_statement_latency AS latency
 FROM sys.session
 WHERE thd_id = 30\G
*************************** 1. row ***************************
rows_examined: 2005
 rows_sent: 1766
 latency: 4.36 ms
1 row in set (0.0539 sec)
```

thd_id 上的需要匹配执行查询的连接的性能架构线程 ID（这可以在 MySQL 8.0.16 及更晚的函数中找到）。2005年检查的行来自检查国家表中的239行，进行完整的表格扫描，然后阅读亚洲国家城市的1766行。

如果 MySQL 不能对联接使用索引，则查询性能将发生巨大变化。您可以使用嵌套循环联接执行查询，而无需以以下方式使用注释是优化器提示）：

```sql
SELECT /*+ NO_BNL(city) */
 CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

忽略 子句是一个索引提示，它告诉 MySQL 忽略括号之间给出的索引。此版本的查询统计信息显示，现在检查超过 200，000 行，并且查询的执行时间比之前长大约十倍（执行此测试的方式与上一个查询相同，其中查询查找亚洲城市在一个连接中执行，中执行 thd_id = 更改为使用第一个连接的线程 ID）：

```sql
mysql> SELECT rows_examined, rows_sent,
 last_statement_latency AS latency
 FROM sys.session
 WHERE thd_id = 30\G
**************************** 1. row ****************************
rows_examined: 208268
 rows_sent: 1766
 latency: 44.83 ms
```



有51个国家与"，这意味着有51个完整的表格扫描的城市表。由于城市表中有 4079 行，因此总共提供了 51 * 4079 = 239 = 208268 行，必须检查。额外的 239 来自包含 239乡村表扫描。

为什么有必要在示例中添加的注释？BNL 代表块嵌套循环，它可以帮助改进没有索引的联接，并且注释禁用该优化。通常，您确实希望保持启用，如下文所述。

### 块嵌套循环

块嵌套循环算法是嵌套循环算法的延伸。它也被称为 BNL 算法。联接缓冲区不是一个提交联接中第一个表中的行，而是用于收集尽可能多的行，并在第二个表的一次扫描中比较所有行。这可以大大提高嵌套循环算法中某些查询的性能。

如果考虑用作嵌套循环算法示例的相同查询，但禁用索引的使用（模拟两个没有索引的表），并且不允许哈希联接（在 8.0.18 及更晚）中），则有一个可以使用块嵌套循环算法的查询。是

```sql
SELECT /*+ NO_HASH_JOIN(country,city) */
 CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

在 MySQL 8.0.17 及更早版本中器提示删除注释。

清单显示了使用Python样代码的块嵌套循环算法的伪代码实现。

```python
Listing 17-2. Pseudo code representing a block nested loop join
result = []
join_buffer = []
for country_row in country:
 if country_row.Continent == 'Asia':
 join_buffer.append(country_row.Code)
 if is_full(join_buffer):
 for city_row in city:
 CountryCode = city_row.CountryCode
 if CountryCode in join_buffer:
 country_row = get_row(CountryCode)
 result.append(
 join_rows(country_row, city_row))
 join_buffer = []
if len(join_buffer) > 0:
 for city_row in city:
 CountryCode = city_row.CountryCode
 if CountryCode in join_buffer:
 country_row = get_row(CountryCode)
 result.append(join_rows(country_row, city_row))
 join_buffer = []
```

列表表示存储联接所需的列的联接缓冲区。在伪代码中，使用。对于用作示例的查询，只需要地区表中"列。这是需要注意的一件重要的事情，不久将进一步讨论。当联接缓冲区已满时，将在城市表上执行扫描;如果城市表的"国家列与联接缓冲区中之一匹配，则构造结果行。

图显示了表示联接的。为简单起见，即使对两个表执行完整表扫描，也只包含联接所需的行的主要键值。

![](../附图/Figure%2017-3.png)

该图显示了如何一起读取存储来自国家/地区表中的行并存储在联接缓冲区中。每次联接缓冲区已满时，将执行城市表扫描，并逐步生成结果。在图中，一次有六行适合联接缓冲区。由于"列每行只需要 3 个字节，因此实际上联接缓冲区将能够保存所有国家/地区代码，除非使用尽可能小的

使用联接缓冲区缓冲地区代码如何影响查询统计信息？至于前面的示例，首先，执行查询，在一个连接中查找亚洲城市：

```sql
SELECT /*+ NO_HASH_JOIN(country,city) */
 CountryCode, country.Name AS Country,
 city.Name AS City, city.District
Figure 17-3. An example of a block nested loop join
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

然后在另一个连接中获取检查的行数和查询延迟以使用第一个连接的线程 ID）：

```sql
mysql> SELECT rows_examined, rows_sent,
 last_statement_latency AS latency
 FROM sys.session
 WHERE thd_id = 30\G
**************************** 1. row ****************************
rows_examined: 4318
 rows_sent: 1766
 latency: 16.87 ms
1 row in set (0.0490 sec)s
```

结果假定为 join_buffer_size的默认值。统计数据显示，块嵌套循环的性能明显优于嵌套循环算法，而无需使用索引。相比之下，使用索引执行查询检查了 2005 行，并测量了大约 4 ms，而使用不带索引的嵌套循环联接检查了 208268 行，并且大约需要 45 ms。这看起来与查询执行时间不相干，但国家表都很小。对于大型表，差异将非线性增长，可能意味着查询完成和似乎永远运行之间的差异。

关于块嵌套循环，您应该注意一些要点，因为它可以帮助您以最佳方式使用它。这些点包括

- 联接所需的仅列存储在联接缓冲区中。这意味着联接缓冲区所需的内存将少于最初预期。
- 联接缓冲区的大小配置了join_buffer_size是缓冲区的最小大小！即使国家/地区代码值少于 1 KiB 将存储在讨论示例中的联接缓冲区设置为 1 GiB，则将分配 1 GiB。因此，将仅根据需要增加。"配置优化器"部分包括有关如何仅针对单个查询更改联接缓冲区大小的信息。
- 使用块嵌套循环算法为每个联接分配一个联接缓冲区。
- 每个联接缓冲区都分配到查询的整个持续时间。
- 块嵌套循环算法可用于完整表扫描、全索引扫描和范围扫描。
- 块嵌套循环算法永远不会用于常量表以及第一个非恒定表。这意味着，在按唯一索引筛选后，需要两个具有一行多行的表之间的联接，才能使用块嵌套循环算法。

您可以通过设置优化器开关来配置是否允许优化器。默认值是启用它。对于单个查询，可以使用 优化器提示为特定联接启用或禁用块嵌套循环。

虽然块嵌套循环是非索引联接的一大改进，但在大多数情况下，使用哈希联接可以做得更好。

### 哈希联接

哈希联接算法是 MySQL 的最新添加，在 MySQL 8.0.18 及更晚版本中受支持。它标志着与嵌套循环联接的传统（包括块嵌套循环变体）的显著突破。它对于没有索引的大型联接特别有用，但在某些情况下甚至可以优于索引联接。

MySQL 实现了经典内存中哈希联接和磁盘上 GRACE 哈希联接算法之间的混合。如果可以存储内存中的所有哈希，则使用纯内存中实现。联接缓冲区用于内存部分，因此可用于哈希的内存量受当联接不适合内存时，联接将溢出到磁盘，但实际联接操作仍在内存中执行。

内存中哈希联接算法包括两个步骤：

1. 联接中的一个表被选为生成。哈希值为联接所需的列计算，并加载到内存中。这称为生成。

    

2. 联接中的另一个表是。对于此表，一次读取一个行，并计算哈希。然后对从生成表计算的哈希执行哈希键查找，并且联接的结果从匹配的行生成。这称为探。

    

当生成表的哈希不适应内存时，MySQL 会自动切换为使用磁盘上实现（基于 GRACE 哈希联接算法）。如果联接缓冲区在生成阶段已满，则从内存中算法切换到磁盘上算法。三个步骤组成：

1. 计算生成表和探测表中所有行的哈希值，并将其存储在由哈希分区的几个小文件中的磁盘上。选择分区数可使探测表的每个分区都适合联接缓冲区，但最多限制为 128 个分区。

    

2. 将生成表的第一个分区加载到内存中，并迭代探测表中的哈希值，其方式与内存中算法的探测阶段相同。由于步骤 1 中的分区对生成表和探测表使用相同的哈希函数，因此只需遍及探测表的第一个分区。

    

3. 清除内存中缓冲区，并一个个继续处理其余分区。

    

内存中算法和磁盘上算法都函数，该函数称为快速，同时仍然提供高质量的哈希（减少哈希冲突的数量）。为了获得最佳性能，联接缓冲区必须足够大，以适应生成表中的所有哈希值。也就是说，哈希联接的相同注意事项。

MySQL will use the hash join whenever the block nested loop would otherwise be chosen, and the hash join algorithm is supported for the query. At the time of writing, the following requirements exist for the hash join algorithm to be used:

- The join must be an inner join.
- The join cannot be performed using an index, either because there is no available index or because the indexes have been disabled for the query.
- All joins in the query must have at least one equi-join condition between the two tables in the join, and only columns from the two tables as well as constants are referenced in the condition.
- As of 8.0.20, anti, semi, and outer joins are also supported.[3](#Fn3). If you join the two tables t1 and t2, then examples of join conditions that are supported for hash join include
- t1.t1_val = t2.t2_val
- t1.t1_val = t2.t2_val + 2
- t1.t1_val1 = t2.t2_val AND t1.t1_val2 > 100
- MONTH(t1.t1_val) = MONTH(t2.t2_val)

如果考虑此部分的重复示例查询，可以通过忽略可用于联接的表上的索引来使用哈希联接来执行它：

```sql
SELECT CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

执行此联接的伪代码与块嵌套循环的伪代码类似，只不过联接所需的列已散列，并且支持溢出到磁盘。伪在清单。

```python
result = []
join_buffer = []
partitions = 0
on_disk = False
for country_row in country:
 if country_row.Continent == 'Asia':
 hash = xxHash64(country_row.Code)
 if not on_disk:
 join_buffer.append(hash)
 if is_full(join_buffer):
 # Create partitions on disk
 on_disk = True
 partitions = write_buffer_to_disk(join_buffer)
 join_buffer = []
 else
 write_hash_to_disk(hash)
if not on_disk:
 for city_row in city:
 hash = xxHash64(city_row.CountryCode)
 if hash in join_buffer:
 country_row = get_row(hash)
 city_row = get_row(hash)
 result.append(join_rows(country_row, city_row))
else:
 for city_row in city:
 hash = xxHash64(city_row.CountryCode)
 write_hash_to_disk(hash)
 for partition in range(partitions):
 join_buffer = load_build_from_disk(partition)
 for hash in load_hash_from_disk(partition):
 if hash in join_buffer:
 country_row = get_row(hash)
 city_row = get_row(hash)
 result.append(join_rows(country_row, city_row))
 join_buffer = []
```

伪代码从国家/地区表开始，并计算值，并存储在联接缓冲区中。如果缓冲区已满，则代码将切换到磁盘上算法，并从缓冲区中写出哈希。这也是确定分区数的地方。在此之后，将散/地区表的其余部分。

下一部分，对于内存中算法，在城市表中的行，将哈希值与缓冲区中的哈希值进行比较。对于计算城市表的值并存储在磁盘上;然后一个处理分区。

比配置更多的内存。

图显示了内存中哈希联接。为简单起见，即使对两个表执行完整表扫描，也只包含联接所需的行的主要键值。

![](../附图/Figure%2017-4.png)

来自国家/地区表中的匹配行的的值将进行哈希处理并存储在联接缓冲区中。然后，对城市表执行表，并计算每代码"哈希值，并且结果由匹配的行构造。

通过首先在一个连接中执行查询，可以像以前算法一样检查查询的统计信息：

```sql
SELECT CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

Then you can look at the Performance Schema statistics for the query by querying the sys.session view in a second connection (change thd_id = 30 to use the thread id of the first connection):

```
mysql> SELECT rows_examined, rows_sent,
 last_statement_latency AS latency
 FROM sys.session
 WHERE thd_id = 30\G
rows_examined: 4318
 rows_sent: 1766
 latency: 3.53 ms
1 row in set (0.0467 sec)
```

您可以看到查询执行得非常好，哈希联接检查的行数与块嵌套循环相同，但它比索引联接快。这不是一个错误：在某些情况下，哈希联接甚至可能优于索引联接。可以使用以下规则来估计哈希联接算法与索引和块嵌套循环联接相比的性能：

- 对于不使用索引的联接，哈希联接通常比块嵌套联接快得多，除非添加了 子句。观察到超过1000个因素的改进。
- 对于没有存在子句的索引的联接，块嵌套循环可以在找到足够的行时退出，而哈希联接将完成整个联接（但可以跳过提取行）。如果与联接找到的子句而包含的行数较小，则块嵌套循环可能更快。
- 对于支持索引的联接，如果索引的选择性较低，哈希联接算法可能会更快。

使用哈希联接的最大好处是，到目前为止，对于没有索引和没有 LIMIT 子句。最后，只有测试才能证明哪个联接策略最适合您的查询。

您可以使用优化器开关启用和禁用。此外优化器开关。默认情况下，两者都处于启用状态。如果要为特定联接配置哈希联接的使用，可以使用 优化器提示。

结束关于 MySQL 中支持的三个高级联接策略的讨论结束。还有一些较低级别的优化也值得考虑。

## 联接优化

MySQL 可以使用联接优化来改进上一节中讨论的联接算法的基本概念，或决定如何执行查询的某些部分。 本节将详细介绍索引合并、多范围读取 （MRR） 和批处理密钥访问 （BKA） 优化。这三个优化是最有可能帮助您帮助优化器使查询计划成为最佳的优化。其余可配置优化在本节末尾介绍，但细节较少。

### 索引合并

通常 MySQL 将只使用每个表的单个索引。但是，如果对同一表中的多个列具有条件，并且没有覆盖所有列的单个索引，则这不是最佳条件。对于这些情况，MySQL 支持索引合并。

支持三种索引合并算法。表总结了的使用时间以及查询计划中包含的信息。

| 算法         | 用例           | EXPLAIN Extra字段  | JSON字段        |
| :----------- | :------------- | :----------------- | :-------------- |
| Intersection | AND            | 使用intersect(...) | intersect(...)  |
| Union        | OR             | 使用union(...)     | union(...)      |
| Sort-Union   | OR with ranges | sort_union(...)    | sort_union(...) |

除了表中列出的信息外，访问类型也

用例指定加入条件的运算符。联合算法和排序联合算法的区别在于联合算法用于相等条件，排序联合算法用于范围条件。对于输出，与索引合并一起使用的索引的名称列在括号中。

在讨论这三种算法时，考虑使用每种算法的实际查询可能很有用。表可用于此目的。 是

```sql
CREATE TABLE `payment` (
 `payment_id` smallint unsigned NOT NULL,
 `customer_id` smallint unsigned NOT NULL,
 `staff_id` tinyint unsigned NOT NULL,
 `rental_id` int(DEFAULT NULL,
 `amount` decimal(5,2) NOT NULL,
 `payment_date` datetime NOT NULL,
 `last_update` timestamp NULL,
 PRIMARY KEY (`payment_id`),
 KEY `idx_fk_staff_id` (`staff_id`),
 KEY `idx_fk_customer_id` (`customer_id`),
 KEY `fk_payment_rental` (`rental_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

默认值、自动增量信息和外键定义已从表中删除，以专注于列和索引。该表有四个索引，全部在单个列上，这使得它成为索引合并优化的一个很好的候选项。

索引合并讨论的其余部分将经历每个索引合并算法以及性能注意事项以及如何配置索引合并的使用。这些示例都只包括两列的条件，但算法确实支持涉及更多列的索引合并。

#### 交集算法

当AND 分隔的多个索引列上具有条件时，使用交集。使用交集索引合并算法的两个查询示例是

```sql
SELECT *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75;
SELECT *
 FROM sakila.payment
 WHERE payment_id > 10
 AND customer_id = 318;
```

第一个查询在两个辅助索引上具有相等条件，第二个查询在主键上具有范围条件，在辅助索引上具有相等条件。索引合并优化与第二个查询独占地使用 InnoDB 表。清单显示了使用两种不同格式的两个查询中的第一个的输出。

```
Listing 17-4. Example of an EXPLAIN output for an intersection merge
mysql> EXPLAIN
 SELECT *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75\G
**************************** 1. row *****************************
 id: 1
 select_type: SIMPLE
 table: payment
 partitions: NULL
 type: index_merge
possible_keys: idx_fk_staff_id,idx_fk_customer_id
 key: idx_fk_customer_id,idx_fk_staff_id
 key_len: 2,1
 ref: NULL
 rows: 20
 filtered: 100
 Extra: Using intersect(idx_fk_customer_id,idx_fk_staff_id); Using
where 1 row in set, 1 warning (0.0007 sec)
mysql> EXPLAIN FORMAT=TREE
 SELECT *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75\G
**************************** 1. row ****************************
EXPLAIN: -> Filter: ((sakila.payment.customer_id = 75) and (sakila.payment.
staff_id = 1)) (cost=14.48 rows=20)
 -> Index range scan on payment using intersect(idx_fk_customer_id,idx_
fk_staff_id) (cost=14.48 rows=20)
1 row in set (0.0004 sec)
```

请注意中的"intersect(...)"消息，以及树格式输出中的索引范围扫描。 这表明索引索引的索引。 传统输出还包括键列中的两个并在"字符串"列中提供长度。

#### 联合算法

当OR 分隔的表存在一系列相等条件时，使用联合。可以使用联合算法的两个查询示例是

```sql
SELECT *
 FROM sakila.payment
 WHERE staff_id = 1
 OR customer_id = 318;
SELECT *
 FROM sakila.payment
 WHERE payment_id > 15000
 OR customer_id = 318;
```

第一个查询在辅助索引上具有两个相等条件，而第二个查询在主键上具有范围条件，在辅助索引上具有相等条件。第二个查询将仅对 InnoDB 表使用索引合并。清单显示了第一个查询的相应 EXPLAIN 输出的示例。

```
Listing 17-5. The EXPLAIN output for a union merge
mysql> EXPLAIN
 SELECT *
 FROM sakila.payment
 WHERE staff_id = 1
 OR customer_id = 318\G
**************************** 1. row *****************************
 id: 1
 select_type: SIMPLE
 table: payment
 partitions: NULL
 type: index_merge
possible_keys: idx_fk_staff_id,idx_fk_customer_id
 key: idx_fk_staff_id,idx_fk_customer_id
 key_len: 1,2
 ref: NULL
 rows: 8069
 filtered: 100
 Extra: Using union(idx_fk_staff_id,idx_fk_customer_id); Using where
1 row in set, 1 warning (0.0008 sec)
mysql> EXPLAIN FORMAT=TREE
 SELECT *
 FROM sakila.payment
 WHERE staff_id = 1
 OR customer_id = 318\G
**************************** 1. row ****************************
EXPLAIN: -> Filter: ((sakila.payment.staff_id = 1) or (sakila.payment.
customer_id = 318)) (cost=2236.18 rows=8069)
 -> Index range scan on payment using union(idx_fk_staff_id,idx_fk_
customer_id) (cost=2236.18 rows=8069)
1 row in set (0.0010 sec)
```

请注意中的"使用（..."）和树格式输出中的索引范围扫描。这表明索引索引的索引。

#### 排序联合算法

类似于使用联合算法的查询，但条件是范围条件而不是相等条件。可以使用排序联合算法的两个查询示例是

```sql
SELECT *
 FROM sakila.payment
 WHERE customer_id < 30
 OR rental_id < 10;
SELECT *
 FROM sakila.payment
 WHERE customer_id < 30
 OR rental_id > 16000;
```

两个查询在两个辅助索引上都有范围条件。清单第一个查询的传统和树格式的相应 EXPLAIN 输出。

```sql
Listing 17-6. The EXPLAIN output using a sort-union merge
mysql> EXPLAIN
 SELECT *
 FROM sakila.payment
 WHERE customer_id < 30
 OR rental_id < 10\G
**************************** 1. row *****************************
 id: 1
 select_type: SIMPLE
 table: payment
 partitions: NULL
 type: index_merge
possible_keys: idx_fk_customer_id,fk_payment_rental
 key: idx_fk_customer_id,fk_payment_rental
 key_len: 2,5
 ref: NULL
 rows: 826
 filtered: 100
 Extra: Using sort_union(idx_fk_customer_id,fk_payment_rental);
Using where 1 row in set, 1 warning (0.0009 sec)
mysql> EXPLAIN FORMAT=TREE
 SELECT *
 FROM sakila.payment
 WHERE customer_id < 30
 OR rental_id < 10\G
**************************** 1. row *****************************
EXPLAIN: -> Filter: ((sakila.payment.customer_id < 30) or (sakila.payment.
rental_id < 10)) (cost=1040.52 rows=826)
 -> Index range scan on payment using sort_union(idx_fk_customer_id,fk_
payment_rental) (cost=1040.52 rows=826)
1 row in set (0.0005 sec)
```

请注意在"sort_union（...）格式输出中的索引范围扫描使用这表明索引索引的索引。

#### 性能注意事项

优化器很难知道索引合并何时比仅仅使用单个索引更理想。一开始，对更多列使用索引似乎总是一种胜利，但索引合并的开销很大，因此只有当索引的索引选择性正确组合存在时，它们才有用。当因过期的索引统计信息而选择索引合并时，会发生严重性能回归的更常见原因之一。

如果优化器选择索引合并且查询性能不佳（例如，与通常性能相比）时，的表执行 ANALYZE TABLE。这通常会改进查询计划。否则，可能需要更改优化器配置以确定是否使用索引合并。

#### 配置

索引合并功能四个优化器开关进行控制，其中一个控制整体功能，另外三个控制三个算法。选项是

- 是否完全启用或禁用索引合并。
- 是否启用交集算法。
- 是否启用联合算法。
- 是否启用排序联合算法。

默认情况下，所有索引合并优化器开关都启用。

此外，还有两个优化器。这两个提示都将表名作为参数，也可以选择应考虑或忽略的索引。例如，如果您想要执行查询，查找1，customer_id 设置为 75而不使用索引合并，可以使用以下查询之一执行：

```
SELECT /*+ NO_INDEX_MERGE(payment) */
 *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75;
SELECT /*+ NO_INDEX_MERGE(
 payment
 idx_fk_staff_id,idx_fk_customer_id) */
 *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75;
```

由于索引合并被视为范围优化的特殊情况优化器提示也会禁用索引合并。可以使用 EXPLAIN 输出确认合并不再使用，如清单。

```sql
Listing 17-7. The EXPLAIN output when index merges are unselected
mysql> EXPLAIN
 SELECT /*+ NO_INDEX_MERGE(payment) */
 *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75\G
**************************** 1. row *****************************
 id: 1
 select_type: SIMPLE
 table: payment
 partitions: NULL
 type: ref
possible_keys: idx_fk_staff_id,idx_fk_customer_id
 key: idx_fk_customer_id
 key_len: 2
 ref: const
 rows: 41
 filtered: 50.0870361328125
 Extra: Using where
1 row in set, 1 warning (0.0010 sec)
mysql> EXPLAIN FORMAT=TREE
 SELECT /*+ NO_INDEX_MERGE(payment) */
 *
 FROM sakila.payment
 WHERE staff_id = 1
 AND customer_id = 75\G
**************************** 1. row ****************************
EXPLAIN: -> Filter: (sakila.payment.staff_id = 1) (cost=26.98 rows=21)
 -> Index lookup on payment using idx_fk_customer_id (customer_
id=75) (cost=26.98 rows=41)
1 row in set (0.0006 sec)
```

另一个优化是多范围读取。

### 多范围读取 （MRR）

多优化旨在减少辅助索引上范围扫描引起的随机 I/O 量。优化首先读取索引，根据行 ID（InnoDB 的群集索引）对键进行排序，然后按行的存储顺序检索行。多范围读取优化可用于使用索引的范围扫描和等联接。它不支持虚拟生成的列上的辅助索引。

使用 InnoDB 的多范围读取优化的主要用例是用于没有覆盖索引的磁盘绑定查询。优化的效果取决于需要多少行和存储的寻寻时间。MySQL 将尝试估计优化何时有用;但是，成本估算是过于悲观而不是过于乐观的一面，因此可能需要提供信息来帮助优化者做出正确的决策。

多范围读取优化由两个优化器开关控制：

- **mrr**：是否允许优化器使用多范围读取优化。默认值为"
- **mrr_cost_based**: 使用多范围读取优化的决定是否基于成本。您可以禁用此选项，以在受支持时始终使用优化。默认值为"

或者，您可以使用 优化器开关根据每个表或索引启用和禁用多范围读取优化。

可以从查询计划中查看是否使用了多范围读取优化。在这种情况下，传统的输出指定在"额外"列中使用输出将字段为。 清单显示了使用多范围优化时传统格式的完整 EXPLAIN 输出的示例。

```sql
Listing 17-8. The EXPLAIN output for a query using Multi-Range Read
mysql> EXPLAIN
 SELECT /*+ MRR(city) */
 *
 FROM world.city
 WHERE CountryCode BETWEEN 'AUS' AND 'CHN'\G
**************************** 1. row *****************************
 id: 1
 select_type: SIMPLE
 table: city
 partitions: NULL
 type: range
possible_keys: CountryCode
 key: CountryCode
 key_len: 3
 ref: NULL
 rows: 812
 filtered: 100
 Extra: Using index condition; Using MRR
1 row in set, 1 warning (0.0006 sec)
```

有必要使用优化器提示显式请求使用读取优化，或者禁用 mrr_cost_based 优化器切换，因为示例查询的估计行数太小，不能将多范围读取优化与基于成本的优化一起使用来选择它。

使用优化时使用随机读取缓冲区来存储索引。缓冲区的大小使用"read_rnd_buffer_size"。

相关的优化是批处理密钥访问优化。

### 批处理密钥访问 （BKA）

批 优化结合了块嵌套循环和多范围读取优化。这样可以以与非索引联接类似的方式对索引联接使用联接缓冲区，并使用多范围读取优化来减少随机 I/O 的数量。

批处理密钥 Access 最有用的查询类型是大型磁盘绑定查询，但没有确定优化何时有帮助以及何时导致性能下降的明确指南。当优化效果最佳时，它会将查询执行时间减少 2~10。但是，当它执行最差时，查询执行时间可能会增加 2~3。

由于批处理密钥访问优化主要有利于相对狭窄的查询范围，并且其他查询的性能可能会降低，因此默认情况下禁用该优化。启用优化的最佳方法是在查询中使用优化器提示，其中已找到优化以提供增益。

如果要使用 optimizer_switch 变量启用启用优化器开关（默认情况下禁用）、禁用 mrr_cost_based 优化器开关默认情况下启用），并确保优化器开关（默认情况下启用）。若要为会话启用批处理密钥访问，可以使用以下查询执行以下操作：

```
SET SESSION
 optimizer_switch
 = 'mrr=on,mrr_cost_based=off,batched_key_access=on';
```

当优化已启用此方式时，您还可以使用 优化器提示来影响是否应使用优化。使用时，传统 EXPLAIN中的列批处理密钥访问），在 JSON字段设置为。清单显示了使用批处理访问时的完整 EXPLAIN 输出的示例。

```
Listing 17-9. The EXPLAIN output with Batched Key Access
mysql> EXPLAIN
 SELECT /*+ BKA(ci) */
 co.Code, co.Name AS Country,
 ci.Name AS City
 FROM world.country co
 INNER JOIN world.city ci
 ON ci.CountryCode = co.Code\G
**************************** 1. row *****************************
 id: 1
 select_type: SIMPLE
 table: co
 partitions: NULL
 type: ALL
possible_keys: PRIMARY
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 239
 filtered: 100
 Extra: NULL
**************************** 2. row *****************************
 id: 1
 select_type: SIMPLE
 table: ci
 partitions: NULL
 type: ref
possible_keys: CountryCode
 key: CountryCode
 key_len: 3
 ref: world.co.Code
 rows: 18
 filtered: 100
 Extra: Using join buffer (Batched Key Access)
2 rows in set, 1 warning (0.0007 sec)
```

在此示例中，使用"国家代码索引在城市表上的联接器提示访问。

联接缓冲区的大小配置了"join_buffer_size由于批处理密钥访问优化主要用于大型联接，因此联接缓冲区通常应配置相对较大，通常为 4 MB 或更大。由于大型联接缓冲区是大多数查询的糟糕选择，因此建议仅增加使用批处理密钥访问优化的查询的大小。

### 其他优化

MySQL 包括对其他几个优化的支持。当优化器对查询有利时，它们会自动使用，并且很少需要手动禁用优化。了解优化是什么仍然很有用，这样您就可以知道在 EXPLAIN 输出中时意味着什么，并且知道当极少数情况下优化器需要朝着正确的方向推进时，如何更改行为。

本小节将按字母顺序处理一些剩余的优化，重点介绍可以配置的优化。对于每个优化，都包含优化器开关、格式（额外）和 JSON 格式的 EXPLAIN 输出详细信息。

#### 条件筛选

当表两个或多个与其关联的条件，并且索引可用于条件的一部分时，使用条件筛选优化。启用条件筛选时，在估计表的整体筛选时，将考虑其余条件的筛选效果。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用
- 没有
- 无

#### 派生合并

可以将派生表、视图引用和通用表表达式合并到它们参与的查询块中。优化的替代方法是实现表、视图引用或公共表表达式。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用。
- 
- ：查询计划反映派生表已合并。

#### 发动机状态俯下

此优化将到存储引擎。它目前仅支持引擎。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用。
- 没有。
- ：包括有关已向下推送的条件的信息。

#### 索引条件下拉

MySQL 可以使用单个索引中的列确定的条件，但索引只能直接筛选部分条件。例如，当您的条件（如名称喜欢索引的一部分时，将发生这种情况。优化还用于辅助索引上的范围条件。对于 InnoDB，索引条件下拉仅支持辅助索引。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用。
- 
- 传统格式在中的索引条件设置"索引"字段。

#### 索引扩展

InnoDB 中的所有辅助非统一索引都附加在索引上的主要键列。启用优化后，MySQL 将考虑主键列作为索引的一部分。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用
- 没有
- 无

#### 索引可见性

当表具有不可见索引，默认情况下，优化器在创建查询计划时不会考虑它。如果启用了索引可见性优化器开关，则将考虑不可见索引。例如，这可用于测试已添加但尚未显示的索引的效果。

优化器开关、提示详细信息如下所示：

- = 默认情况下禁用
- 没有
- 

#### 松散索引扫描

在某些情况下可以使用索引的一部分来提高聚合数据或包含子句的查询的性能。这要求用于按形式对数据进行分组的列为多列索引的左前缀，以及不用于分组的其他列。当存在组句时，只允许和聚合函数。

优化器开关、提示详细信息如下所示：

- 没有。
- 禁用松散的索引扫描优化以及索引合并和范围扫描。
- 传统格式在的索引。JSON 格式将字段。

#### 范围访问方法

范围与其他优化略有不同，因为它被视为访问方法。MySQL 将只扫描表或索引的一个或多个部分，而不是执行完整的表或索引扫描。范围访问方法通常用于涉及运算符 等条件，以及类似运算符。

优化器开关、提示详细信息如下所示：

- 没有。
- \- 这也禁用松散的索引扫描和索引合并优化。但是，它不会禁用跳过扫描优化，即使这也使用范围访问。
- ：访问方法设置为。

您可以使用选项来限制用于范围访问的内存量。默认值为 8 MiB。如果将值设置为 0，则意味着可以使用无限量的内存。

#### 半连接

用于 IN。有四种支持的策略：物化、重复杂草、第一次匹配和松散扫描（不要与松散的索引扫描优化混淆）。启用子查询物化时，半连体优化将尽可能使用物化策略。对于半参与优化仅在 MySQL 8.0.16 及更晚版本中支持，（类似 – 也称为反连体），MySQL 8.0.17 或更晚是必需的。

可以使用半连接优化器开关控制优化，以完全启用或禁用优化。 优化器提示可用于使用一个或多个"材料化和单个查询。

物化策略与子序列物化优化相同。有关详细信息，请参阅此。

重复的除义策略执行半联接，就像它是一个普通联接一样，并使用临时表删除重复项。优化器开关、提示详细信息如下所示：

- 默认情况下启用。
- 
- 对于所涉及的表，传统"额外列中"和"结束临时"。JSON 格式的输出使用名为"duplicates_removal

第一个匹配策略返回每个值的第一个匹配项，而不是所有值。优化器开关、提示详细信息如下所示：

- = 默认情况下启用。
- 
- ：传统格式在"额外"列中具有其中括号之间的值是引用表的名称。 JSON 格式将"first_match设置为引用表的名称。

松散扫描策略使用索引从每个子查询的值组中选择单个值。优化器开关、提示详细信息如下所示：

- 松散扫描 + 。
- 
- 传统在"列和指示索引的哪些部分用于松散扫描。JSON 格式将设置为。

#### 跳过扫描

跳过MySQL 8.0.13 中是新的，其效果与松散索引扫描类似。当多列索引的第二列存在范围条件，但第一列上没有条件时，使用它。跳过扫描优化将完整索引扫描转换为一系列范围扫描（索引中第一列的每个值进行一次范围扫描）。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用。
- 
- 传统格式在具有using 索引，JSON 格式字段设置。

#### 子查询实现

策略将子查询的结果存储到内部临时表中。如果可能，优化器将在临时表上添加自动生成的哈希索引，从而快速将其加入到查询的其余部分。

优化器开关、提示详细信息如下所示：

- = 默认情况下启用。
- 
- ：传统格式选择类型实现。JSON 格式创建一个名为 materialized_from_subquery。

启用优化器开关（默认值）时，优化器将使用估计的成本来决定在子查询物化子查询将条件重写为）。当开关关闭时，优化器始终选择子查询物化。

正如最后两节中显而易见的，配置优化器的可能性非常大。下一节将仔细研究这一点。

## 配置优化器

您可以通过多种方式配置 MySQL 来影响优化器。您已经遇到一些配置选项、优化器开关和优化器提示。本节将开始介绍如何配置与不同操作关联的引擎和服务器成本，然后通过配置选项介绍有关优化器交换机的其他详细信息。最后，将讨论优化器提示。

### 引擎成本

引擎成本提供有关读取数据的成本信息。由于数据可以从内存或磁盘获取，并且不同的存储引擎读取数据的成本可能不同，因此它并非一刀切。因此，MySQL 允许您配置每个存储引擎从内存和磁盘读取的成本。

您可以使用表来更改读取数据的成本。该表具有以下列：

- 成本数据用于的存储引擎。值用于表示没有特定数据的所有存储引擎。
- 当前未使用，必须具有值 0。
- 成本的名称。目前，有两个受用于基于磁盘用于基于内存的读取。
- 读取操作的成本。NULL 值默认值）表示使用存储在。
- 上次更新行的时间。时间在会话变量设置的返回。
- 可以提供的可选注释，为成本更改的原因提供上下文。注释最多可以 1024 个字符长。
- 用于操作的默认成本。这是只读列。默认值为 1，io_block_read_cost为 0.25，memory_block_read_cost 。

主键由列组成。引擎成本对于如在 MySQL 8 中，InnoDB 可以为优化器提供数据在缓冲池中还是需要从磁盘读取数据的估计值。

您可以使用 UPDATE 语句更新现有成本。如果要插入存储引擎的估计值，请使用语句;如果要删除自定义成本值，请使用语句。在这两种情况下，都必须执行语句才能使更改对新连接生效（现有连接继续使用旧值）。例如，如果要为 InnoDB 添加特定数据，假设主机具有慢速磁盘 I/O 和非常快的内存，可以使用以下语句，例如

```sql
mysql> INSERT INTO mysql.engine_cost
 (engine_name, device_type, cost_name,
 cost_value, comment)
 VALUES ('InnoDB', 0, 'io_block_read_cost',
 2, 'InnoDB on non-local cloud storage'),
 ('InnoDB', 0, 'memory_block_read_cost',
 0.15, 'InnoDB with very fast memory');
Query OK, 2 rows affected (0.0887 sec)
Records: 2 Duplicates: 0 Warnings: 0
mysql> FLUSH OPTIMIZER_COSTS;
Query OK, 0 rows affected (0.0877 sec)
```

如果要更改成本值，建议将值大约翻倍或一半，并评估效果。由于引擎成本是全局的，因此应确保在更改个良好的监视基线，并在更改后比较查询性能，以检测更改是否具有预期效果。

MySQL 还具有一些更通用的服务器成本，可用于影响与查询相关的各种操作。

### 服务器成本

MySQL 使用基于成本的方法来确定最佳查询计划。要想让这一切尽可能好地工作，它必须知道各种类型的操作有多昂贵。计算最重要的部分是相对成本是正确的，这幸运的是有帮助。然而，由于系统而异，成本如何，以及它如何影响工作负载。

您可以使用表更改多个操作的成本。该表具有以下列：

- 操作的名称。
- 执行操作的成本。如果成本设置为，则使用默认成本（default_value成本以浮点编号提供。
- 成本上次更新时。时间在会话变量设置的返回。
- 可以提供的可选注释，为成本更改的原因提供上下文。注释最多可以 1024 个字符长。
- 用于操作的默认成本。这是只读列。

当前可以在该表中配置六操作。这些是

- 在磁盘上创建内部临时表的成本。服务器和越有可能优化器选择需要磁盘上临时表的查询计划。默认成本为 20。
- 在磁盘上创建的内部临时表的行操作成本。默认成本为 0.5。
- 比较记录键的成本。如果使用基于非索引排序的文件排序查询计划出现问题，则可能会增加这些操作的成本。默认成本为 0.05。
- 在内存中创建内部临时表的成本。存储和越有可能优化器选择需要内存内部临时表的查询计划。默认成本为 1。
- 在内存中创建的内部临时表的行操作成本。默认成本为 0.1。
- 评估行条件的一般成本。成本越低，MySQL 就越倾向于检查许多行，例如使用完整的表扫描。成本越高，MySQL 将尝试减少检查的行数，并使用更多的索引查找和范围扫描。默认成本0.1。

如果要更改服务器成本之一，则需要使用常规语句，后跟OPTIMIZER_COSTS。然后，这些更改将影响新连接。例如，如果将磁盘上的内部临时表存储在 RAM 磁盘（共享内存磁盘）上，并且希望降低成本以反映

```sql
mysql> UPDATE mysql.server_cost
 SET cost_value = 1,
 Comment = 'Stored on memory disk'
 WHERE cost_name = 'disk_temptable_create_cost';
Query OK, 1 row affected (0.1051 sec)
Rows matched: 1 Changed: 1 Warnings: 0
mysql> UPDATE mysql.server_cost
 SET cost_value = 0.1,
 Comment = 'Stored on memory disk'
 WHERE cost_name = 'disk_temptable_row_cost';
Query OK, 1 row affected (0.1496 sec)
Rows matched: 1 Changed: 1 Warnings: 0
mysql> FLUSH OPTIMIZER_COSTS;
Query OK, 0 rows affected (0.1057 sec)
```

更改成本可能并不总是影响查询计划，因为优化器可能别无选择，只能使用给定的查询计划，或者计算的成本是如此不同，因此更改服务器成本以影响查询计划可能会对其他查询产生太大的影响。请记住，所有连接的服务器成本都是全局的，因此只有在存在系统问题时，才应更改成本。如果问题只影响几个查询，最好使用优化器提示来影响查询计划。

影响查询计划的另一个选项是优化器开关。

### 优化器开关

本章都提到了优化器开关。它们通过"optimizer_switch优化器开关的工作方式与其他配置选项略有不同，因此值得更深入地了解其使用。

选项是一个复合选项，所有优化器交换机使用相同的选项，但有可能在不包括您不想更改的交换机的情况下更改单个交换机。将要更改的开关设置为打开关闭启用或禁用它。可以在影响所有新连接的全局范围内或在会话级别更改优化器交换机。例如，如果要禁用当前优化器开关，可以使用以下语句：

```
mysql> SET SESSION optimizer_switch = 'derived_merge=off';
Query OK, 0 rows affected (0.0003 sec)
```

如果要永久更改该值，可以使用"或方式使用：

```
mysql> SET PERSIST optimizer_switch = 'derived_merge=off';
Query OK, 0 rows affected (0.0431 sec)
```

如果您希望将值存储在 MySQL 配置文件中，则同样适用，例如：

```
[mysqld]
optimizer_switch = "derived_merge=off"
```

表列出了截至 MySQL 8.0.18 可用的优化器交换机及其默认值以及交换机操作的简要摘要。优化器开关按它们在"自动"选项中排序。

| 优化器开关                          | 默认值 | 描述                                                        |
| :---------------------------------- | :----- | :---------------------------------------------------------- |
| index_merge                         | 上     | 整体开关控制索引合并。                                      |
| index_merge_union                   | 上     | 联合索引合并策略。                                          |
| index_merge_sort_union              | 上     | 排序联合索引合并策略。                                      |
| index_merge_intersection            | 上     | 交集索引合并策略。                                          |
| engine_condition_pushdown           | 上     | 将条件推送到存储引擎。                                      |
| index_condition_pushdown            | 上     | 将索引条件向下推送到存储引擎。                              |
| mrr                                 | 上     | 多范围读取优化。                                            |
| mrr_cost_based                      | 上     | 是否使用多范围读取优化应基于成本估算。                      |
| block_nested_loop                   | 上     | 块嵌套循环联接算法。这与控制是否可以使用哈希联接。          |
| batched_key_access                  | 关闭   | 批处理密钥访问优化。还需要启用，并禁用的分批密钥访问。      |
| 物化                                | 上     | 是否可以使用物化子挤压。这也会影响物化半连体优化是否可用。  |
| 半人                                | 上     | 整体开关启用或禁用半连接优化。                              |
| 松扫描                              | 上     | 半连体松散扫描策略。                                        |
| 第一场比赛                          | 上     | 半连第一场比赛策略。                                        |
| 重复杂草                            | 上     | 半连重复的杂草策略。                                        |
| subquery_materialization_cost_based | 上     | 是否使用子查询物化是基于成本估算。                          |
| use_index_extensions                | 上     | InnoDB 添加到非统一二级索引的主要键列是否用作索引的一部分。 |
| condition_fanout_filter             | 上     | 筛选估计是否包括访问方法未处理的条件。                      |
| derived_merge                       | 上     | 派生的合并优化。                                            |
| use_invisible_indexes               | 关闭   | 是否应使用不可见索引。                                      |
| skip_scan                           | 上     | 跳过扫描优化。                                              |
| hash_join                           | 上     | 哈希联接算法。要启用哈希联接，启用"标记"开关。              |

本章前面将更详细地介绍各种优化、策略和算法。

如果要全局更改设置，或在会话期间更改设置，则"策略"选项非常大;但是，在许多情况下，您只需要更改优化器开关或单个查询的设置。在这种情况下，优化器提示是更好的选择。

### 优化器提示

优化器提示功能在 MySQL 5.7 中引入，并在 MySQL 8 中扩展。它允许您向优化器提供信息，以便影响查询计划的结束。与打开选项的"优化器"选项不同，可以按查询块、表或索引设置优化器提示等效项。此外，还支持在查询期间更改配置选项的值。当优化器无法完全获得最佳查询计划或需要查询来执行时，这是提高查询性能的有力方法，例如，某些选项的值大于全局默认值。

优化器提示使用特殊注释、或句之后。 语法在注释开始后立即使用带 + 的内联注释，例如：

```
SELECT /*+ MAX_EXECUTION_TIME(2000) */
 id, Name, District
 FROM world.city
 WHERE CountryCode = 'AUS';
```

本示例将查询的最大执行时间设置为 2000 毫秒。

表列出了截至 MySQL 8.0.18 的优化器提示，包括每个提示支持的范围和简要说明。对于许多提示，有两个版本，一个用于启用该功能，另一个用于禁用该功能;这些列在一起。提示按字母顺序列出，根据启用该功能的提示和没有相应的提示启用该功能。

| 提示                       | 范围        | 描述                                                         |
| :------------------------- | :---------- | :----------------------------------------------------------- |
| BKA NO_BKA                 | 查询块表    | 批处理密钥访问优化。                                         |
| BNL NO_BNL                 | 查询块表    | 块嵌套循环联接算法。                                         |
| HASH_JOIN NO_HASH_JOIN     | 查询块表    | 哈希联接算法。                                               |
| INDEX_MERGE NO_INDEX_MERGE | 表指数      | 索引合并优化。                                               |
| JOIN_FIXED_ORDER           | 查询块      | 强制按查询中列出的顺序执行查询块中的所有联接。这与使用       |
| JOIN_ORDER                 | 查询块      | 强制按特定顺序联接两个或多个表。优化器可以自由更改未列出的表的联接顺序。 |
| JOIN_PREFIX                | 查询块      | 强制指定表为联接的第一个表，并按给定顺序联接它们。           |
| JOIN_SUFFIX                | 查询块      | 强制指定表作为联接的最后一个表，并按给定顺序联接它们。       |
| MAX_EXECUTION_ TIME        | 全球        | 限制 SELECT 语句的查询时间。该值以毫秒为单位。               |
| MERGE NO_MERGE             | 表          | 派生的合并优化。                                             |
| MRR NO_MRR                 | 表指数      | 多范围读取优化。                                             |
| NO_ICP                     | 表指数      | 索引条件下拉优化。                                           |
| NO_RANGE_ OPTIMIZATION     | 表指数      | 不要使用范围访问表和/或索引。这还可以禁用索引合并和松散索引扫描。如果查询会导致许多范围扫描，并且它会导致性能或资源问题，它最有用。 |
| QB_NAME                    | 查询块      | 设置查询块的名称。该名称可用于引用其他优化器提示中的查询块。 |
| RESOURCE_GROUP             | 全球        | 用于查询的资源组。下一节将讨论资源组。                       |
| SEMIJOIN NO_SEMIJOIN       | 查询块      | 半连体优化。                                                 |
| SKIP_SCANNO_SKIP_SCAN      | 表指数      | 跳过扫描优化。                                               |
| SET_VAR                    | Global      | 设置查询持续时间的配置变量的值。                             |
| SUBQUERY                   | Query block | 子查询是否可以使用物化优化或。                               |

本章前面讨论联接算法和优化时，已经遇到一些优化器提示。作用域指定提示应用于查询的哪一部分。包括

- 提示适用于整个查询。
- 提示适用于一组联接。例如，查询的顶层是查询块;如果查询是查询块，则查询的顶层是查询块。子查询是另一个查询块。在某些情况下，应用于查询块的提示还可以采用联接的表名，以将提示限制为特定联接。
- 提示适用于特定表。
- 提示适用于特定索引的使用。

指定表时，需要使用该表在查询中使用的名称。如果为表指定了别名，则需要使用别名而不是表名，以确保查询块中的所有表都可以唯一标识。

优化器提示的指定方式与在括号中指定的参数的函数调用相同。当优化器提示不采用任何参数时，使用一组空括号。您可以为同一查询指定多个优化器提示，在这种情况下，您可以使用空格来分隔它们。如果指定多个参数（前导查询块名称除外），则必须使用逗号分隔参数（但请注意，在某些情况下，使用空格将两个信息块合并到一个参数中，例如，在指定索引时，表和索引名称由空格分隔）。

对于具有多个查询块的复杂查询，命名很有用，以便可以指定优化器提示应应用于的查询块。使用 器提示设置查询块的名称：

```
SELECT /*+ QB_NAME(payment) */
 rental_id
 FROM sakila.payment
 WHERE staff_id = 1 AND customer_id = 75;
You can then refer to the query block by adding an @ in front of the query block
name when specifying a hint:
SELECT /*+ NO_INDEX_MERGE(@payment payment) */
 rental_id, rental_date, return_date
 FROM sakila.rental
 WHERE rental_id IN (
 SELECT /*+ QB_NAME(payment) */
 rental_id
 FROM sakila.payment
 WHERE staff_id = 1 AND customer_id = 75
 );
```

该示例将 IN 条件中的查询块为。然后在顶层引用此块名称，以禁用付款查询块中的索引合并功能。当您使用这种查询块名称时，提示中列出的所有表都必须来自同一个查询块。指定查询块的替代表示法是在表名之后添加它，例如：

```
SELECT /*+ NO_INDEX_MERGE(payment@payment) */
 rental_id, rental_date, return_date
 FROM sakila.rental
 WHERE rental_id IN (
 SELECT /*+ QB_NAME(payment) */
 rental_id
 FROM sakila.payment
 WHERE staff_id = 1 AND customer_id = 75
 );
```

这与上一示例中的相同，但它的优点是，您可以将一个提示用于不同查询块中的表。

优化器提示的一大用途是更改查询期间配置变量的值。这对于最好以但某些查询的值越大，可以提高性能的选项特别有用。使用优化器提示，参数为变量赋值。在参考手册中，可与器提示一起使用的应用：是"。例如，要将 1 0（此选项将很快解释），您可以使用

```
SELECT /*+ SET_VAR(join_buffer_size = 1048576)
 SET_VAR(optimizer_search_depth = 0) */
 CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code
 WHERE Continent = 'Asia';
```

从示例中需要注意几点。首先提示不支持在同一提示中设置两个选项，因此您需要为每个选项指定一次提示。其次，不支持表达式或单位，因此对于需要直接以字节形式提供值。

优化器提示不能帮助您的一件事。如果您对优化器的索引选择不满意，则需要使用索引提示。

### 索引提示

提示在 MySQL 中已经存在很长时间了。您可以使用它们为每个表指定允许优化器使用的索引以及应忽略哪些索引。在禁用用于块和哈希联接算法的示例的示例的索引时，您已经遇到忽略索引提示。

MySQL 支持三个索引提示：

- ：完全不允许优化器使用命名索引。
- 如果使用索引，优化器应使用其中一个命名索引。
- ：这与 USE ，但如果完全可以使用其中一个命名索引，应始终避免表扫描。

使用其中一个索引提示时，需要在括号内逗号分隔列表中提供应受提示影响的索引的名称。索引提示放在表名之后。如果为表添加别名，请将索引提示放在别名之后。例如，若要在不使用国家/地区表上的主键或城市的国家索引的情况下城市，可以使用以下查询：

```
SELECT ci.CountryCode, co.Name AS Country,
 ci.Name AS City, ci.District
 FROM world.country co IGNORE INDEX (Primary)
 INNER JOIN world.city ci IGNORE INDEX (CountryCode)
 ON ci.CountryCode = co.Code
 WHERE co.Continent = 'Asia';
```

请注意主键如何。在示例中，索引提示应用于可以使用表索引的所有操作。可以通过添加 FOR JOIN、"按顺序"或"组 BY"来限制范围以联接、分组，例如：

```
SELECT *
 FROM world.city USE INDEX FOR ORDER BY (Primary)
 WHERE CountryCode = 'AUS'
 ORDER BY ID;
```

虽然在大多数情况下最好限制索引提示的使用以便优化器可以随着索引和数据的变化而自由更改查询计划，但索引提示是可用的功能最强大的工具之一，您不应回避在需要时使用它们。

影响优化器的最后一种方法是使用配置选项。

### 配置选项

有几个配置选项会影响优化器，但 这些选项控制优化器搜索最佳查询计划的详尽性，以及是否应使用优化器跟踪功能跟踪其步骤。优化器跟踪功能将推迟到第，其中与 EXPLAIN 语句一讨论。

这里将讨论的两个选项是

- optimizer_prune_level
- optimizer_search_depth

选项的值可以是 0 或 1。默认值为 1。它确定优化器是否会修剪查询计划以避免执行详尽的搜索。值为 1 可进行修剪。如果遇到修剪阻止优化器查找足够好的查询计划的更改会话的查询。全局值几乎应始终为 1。

选项确定在搜索最佳查询计划时应包含多少表（联接）。允许的值为 0=62，默认值为 62。由于一个查询块允许的表数为 61，因此值为 62 意味着除了修剪删除的搜索路径外，将进行详尽的搜索。值 0 表示 MySQL 选取最大搜索深度;当前与将值设置为 7 相同。

如果查询块包含许多由内部联接联接的表，并且与查询执行时间相比，确定查询计划设置为 0 或值低于 62。另一种选择是使用 优化器提示来锁定部分查询的联接顺序。

到目前为止，讨论一直围绕着优化过程和优化器拥有的选项。还有一个级别需要考虑：在执行查询时应使用哪个资源组。

## 资源组

资源组的概念在 MySQL 8 中是新的，它允许您为查询或一组查询可以使用的资源使用设置规则。这可能是提高高并发系统上性能的有力方法，并允许您优先处理某些查询的优先级。本节介绍如何获取有关现有资源组的信息、管理资源组以及如何使用它们。

### 检索有关资源组的信息

有关现有资源组的信息，请参阅视图，它是存储资源组的数据字典表顶部的视图。该视图包括以下列：

- 资源组的名称。
- 资源组是用于还是级线程。由系统线程使用，用户使用用户。
- 资源组是否启用。
- 哪个虚拟 CPU。虚拟 CPU 会考虑物理 CPU 内核、超线程、硬件线程等。
- 使用资源组的线程的线程优先级。值越低，优先级越高。

清单显示了 MySQL 安装后默认资源组的资源组信息。"VCPU_IDS值取决于您系统中的虚拟 CPU 数。

```
mysql> SELECT *
 FROM information_schema.RESOURCE_GROUPS\G
*************************** 1. row ***************************
 RESOURCE_GROUP_NAME: USR_default
 RESOURCE_GROUP_TYPE: USER
RESOURCE_GROUP_ENABLED: 1
 VCPU_IDS: 0-7
 THREAD_PRIORITY: 0
*************************** 2. row ***************************
 RESOURCE_GROUP_NAME: SYS_default
 RESOURCE_GROUP_TYPE: SYSTEM
RESOURCE_GROUP_ENABLED: 1
 VCPU_IDS: 0-7
 THREAD_PRIORITY: 0
2 rows in set (0.0007 sec)
```

默认情况下有两个资源组连接的优先级组和线程的一个资源组。两个组配置相同，允许使用所有 CPU。这两个组既不能删除也不能修改。但是，您可以创建自己的资源组。

### 管理资源组

只要不尝试修改或删除其中一个默认组，就可以创建、更改和删除资源组。这允许您创建资源组，可用于在查询之间划分资源。它要求创建、更改或删除资源组的权限。

以下语句可用于管理资源组：

- 创建新资源组
- ：修改现有资源组
- ：删除资源组

对于所有三个语句，必须始终指定组名称，并且指定组名称时不提供任何参数名称（示例将很快跟随）。表显示了三个语句的参数。其中值指定 N 或 M-N，M 和 N 表示整数。

| 选项     | 语法            | 值                               | 操作              |
| :------- | :-------------- | :------------------------------- | :---------------- |
| Name     |                 | 最多 64 个字符                   | CREATE ALTER DROP |
| Type     | TYPE = ...      | 系统用户                         | CREATE            |
| CPUs     | VCPU = ...      | N or M-N in comma-separated list | CREATE ALTER      |
| Priority | THREAD_PRIORITY | N                                | CREATE ALTER      |
| Status   |                 | 启用禁用                         | CREATE ALTER      |
| Force    | FORCE           |                                  | ALTER DROP        |

对于优先级，值的有效范围取决于组类型。可以具有 -20 和 0 之间的类型可以具有 -20 和 19 之间的优先级。优先级的含义遵循 Linux 中 nice 功能的原则，这意味着优先级的值越低，线程的优先级越高。因此，-20 是最高优先级，而 19 的优先级最低。在 Microsoft Windows 上，有五个可用的本机优先级。表列出了从资源组优先级到 Microsoft Windows 优先级的映射。

| 开始优先级 | 结束优先级 | 微软视窗优先级               |
| :--------- | :--------- | :--------------------------- |
| -20        | -10        | THREAD_PRIORITY_HIGHEST      |
| -9         | -1         | THREAD_PRIORITY_ABOVE_NORMAL |
| 0          | 0          | THREAD_PRIORITY_NORMAL       |
| 1          | 10         | THREAD_PRIORITY_BELOW_NORMAL |
| 11         | 19         | THREAD_PRIORITY_LOWEST       |

创建新资源组时，必须设置组的名称和类型。其余参数是可选的。默认值是将包括主机上的所有可用 CPU，将优先级设置为 0，然后启用组。为可以使用7 的用户连接创建名为 my_group 的已启用组的示例是（这要求主机至少有八个虚拟 CPU），如下所示：

```
CREATE RESOURCE GROUP my_group
 TYPE = USER
 VCPU = 2-3,6,7
THREAD_PRIORITY = 0
ENABLE;
```

显示如何一个列出 CPU 或使用范围。资源组名称被视为标识符，因此您只需在与架构和表名称相同的情况下用背子引用它。

语句类似于创建，但不能更改组名称或组类型。例如，要更改名为"已创建"的组的 CPU 和

```
 ALTER RESOURCE GROUP my_group
 VCPU = 2-5
THREAD_PRIORITY = 10;
```

如果需要删除资源组，可以使用只需的 DROP 资源组语句，例如：

```sql
DROP RESOURCE GROUP my_group;
```

对于 语句，有一个可选参数 。这指定了当有线程使用资源组时，MySQL 应如何处理案例。表总结了该行为。

| Forcing | ALTER                                                        | DROP                                 |
| :------ | :----------------------------------------------------------- | :----------------------------------- |
| 不强迫  | 当使用组的所有现有线程都终止时，更改将生效。在此之前，任何新线程都无法使用资源组。 | 如果将任何线程分配给组，则发生错误。 |
| 迫使    | 现有线程将基于线程类型移动到默认组。                         | 现有线程将基于线程类型移动到默认组。 |

修改和删除资源组时，如果具有选项，则分配给该组的现有线程将重新分配到默认组。这意味着用户的一个组和线程的一个组。对于，也指定了"禁用"选项才能使用"强制"选项。

现在，您可以向线程分配资源组了。

### 分配资源组

有两种方法可以为线程设置资源组。您可以显式设置线程的资源组，也可以使用优化器提示为单个查询设置资源组。它要求或权限将线程分配给资源组，而不管使用的方法如何。

首先，重新组（这一次只使用一个 CPU 使其在所有系统上工作）：

```
CREATE RESOURCE GROUP my_group
 TYPE = USER
 VCPU = 0
THREAD_PRIORITY = 0
ENABLE;
```

使用语句将线程分配给资源组。这适用于系统和用户线程。若要将连接本身分配给资源组，请使用资源组名称为唯一参数的 语句，例如：

```
SET RESOURCE GROUP my_group;
```

如果要更改一个或多个其他线程的资源组，请添加末尾的关键字，然后添加要分配给该组的性能架构线程 ID 的逗号分隔列表。例如，要将线程 47、49 和 50在本例中，线程 ID 显然会有所不同 - 替换为系统中存在的线程）

```
SET RESOURCE GROUP my_group FOR 47, 49, 50;
```

作为替代方法，您可以使用 优化器提示在查询期间将资源组分配给线程，例如：

```
SELECT /*+ RESOURCE_GROUP(my_group) */
 *
 FROM world.city
 WHERE CountryCode = 'USA';
```

优化器提示通常是使用资源组的最佳方法，因为它允许您按查询设置它，并且当您使用。它也可以与 MySQL 重写插件或代理（如 ProxySQL）结合使用，该代理支持将优化器提示注释添加到查询中。

可以使用表列查看每个线程使用的资源组。例如，若要查看使用之前使用组

```
mysql> SELECT THREAD_ID, RESOURCE_GROUP
 FROM performance_schema.threads
 WHERE THREAD_ID IN (47, 49, 50);
+-----------+----------------+
| THREAD_ID | RESOURCE_GROUP |
+-----------+----------------+
| 47        | my_group       |
| 49        | my_group       |
| 50        | my_group       |
+-----------+----------------+
3 rows in set (0.0008 sec)
```



这留下了如何使用资源组的问题。

### 性能注意事项

使用资源组的效果取决于几个因素。默认值是，所有线程都可以在任何 CPU 上执行，并且具有相同的中端优先级，这与 MySQL 5.7 及更早版本中的行为相同。当 MySQL 开始遇到资源争用时，对资源组使用不同的配置的主要好处就来了。

不可能就如何使用资源组最最佳给出具体建议，因为它在很大程度上取决于硬件和查询工作负载的组合。随着对 MySQL 代码进行新的改进，资源组的最佳使用也会发生变化。这意味着，与一如既往，您需要使用监视来确定更改资源组及其使用的效果。

也就是说，对于如何使用资源组来提高性能或用户体验，可以提出。这些包括但不限于

- 为不同的连接提供不同的优先级。例如，这可以是确保批处理作业不会影响与前端应用程序相关的查询太多，也可以为不同的应用程序提供不同的优先级。
- 将不同应用程序的线程分配给不同的 CPU 集，以减少它们之间的干扰。
- 将写入线程和读取线程分配给不同的 CPU 集，以设置不同任务的最大同意率。例如，如果写入线程遇到资源争用，这非常有用，可以限制它们并发。
- 对执行需要许多锁的事务的线程给予高优先级，以便事务可以尽快完成并再次释放锁。

根据经验，资源组很有用，如果没有足够的 CPU 资源并行执行所有内容，或者写入并发性变得过高，并且通过限制可以使用哪些 CPU 处理写入工作负荷来避免争用来限制资源组。对于低并发工作负载，通常最好使用默认资源组。

## 总结

本章介绍优化器的工作原理、联接算法和可用的优化、如何配置优化器以及资源组。

MySQL 使用基于成本的优化器，其中估计查询执行的每个部分的成本，并选择总体查询计划以最大限度地降低成本。作为优化的一部分，优化器将使用各种转换重写查询，找到最佳联接顺序，并做出其他决策，例如应使用哪些索引。

MySQL 支持三种联接算法。最简单的算法（也是原始算法）是嵌套循环联接，它只需在最外层表中的行上迭代，然后为下一个表设置一个嵌套循环，等等。块嵌套循环是非索引联接可以使用联接缓冲区减少内部表的表扫描次数的扩展。MySQL 8.0.18 中的新功能是哈希联接算法，该算法也用于不使用索引的联接，并且对于它所支持的联接非常有效 - 因此，对于低选择性索引，它可以优于索引联接。

可以使用一系列其他优化。特别关注索引合并、多范围读取和批处理密钥访问优化。索引合并优化允许 MySQL 每个表使用多个索引。多范围读取优化用于减少由辅助索引读取引起的随机 I/O 量。批处理密钥访问优化结合了块嵌套循环和多范围读取优化。

有几种方法可以更改 MySQL 的配置以影响优化器。mysql.engine_cost从内存和磁盘读取的成本信息。这可以通过每个存储引擎进行设置。该各种操作（如使用内部临时表和比较记录）的基本成本估算。配置选项用于启用或禁用各种优化器功能，如块嵌套循环、批处理密钥访问等。

影响优化器的两个灵活选项是使用优化器提示和索引提示。优化器提示可用于启用或禁用功能，以及设置查询选项，甚至更精细地向下调整到索引级别。索引提示可用于启用或禁用表的索引。可选地，索引提示可以限制为特定操作，如排序。最后选项可用于限制优化器为查找最佳查询计划而将执行多少工作。

最后一个涵盖的功能是在 MySQL 8 中添加的资源组。资源组可用于指定允许线程使用哪些 CPU 以及线程应执行哪个优先级。这对于某些线程的优先级高于其他线程或防止资源争用非常有用。

下一章将介绍锁在 MySQL 中的工作原理。
