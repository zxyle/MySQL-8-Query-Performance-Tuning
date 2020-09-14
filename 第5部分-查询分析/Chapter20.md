# 分析查询

在上一章中，您学习了如何查找优化候选查询。现在是采取下一步的时间 - 分析查询，以确定它们为什么不能执行预期。分析过程中的主要工具是语句，该语句显示优化器将使用的查询计划。相关是可用于调查优化器最终与查询计划一起结束的优化器跟踪。另一种可能性是使用性能架构中的语句和阶段信息来查看存储过程或查询花费的时间最多的位置。本章将讨论这三个主题。

解释声明是迄今为止本章的最大部分，分为四个部分：

- EXPLAIN 语句法。
- 可以查看查询计划每种格式的详细信息。这包括在 MySQL 工作台使用的语句和解释中显式选择的两种格式。
- 对查询计划中可用信息的讨论。
- 使用 EXPLAIN 语句返回的数据的一些示例。

## 解释用法

EXPLAIN 语句返回 MySQL 优化器将用于给定查询的查询计划的概述。它同时非常简单，也是查询调优中更复杂的工具之一。这很简单，因为您只需要在要调查的查询之前添加命令，并且很复杂，因为了解信息需要了解 MySQL 及其优化器的工作方式。您可以将与显式指定的查询和当前由另一个连接执行的查询一起使用 EXPLAIN。本节介绍 EXPLAIN 语句的基本。

### 显式查询的用法

通过在查询来生成查询计划，可以选择添加选项以指定是希望以传统表格式、使用 JSON 格式还是树样式格式返回结果。支持选择、插入和语句。查询不执行（但请参阅下一节关于的异常），因此可以安全地获取查询计划。

如果需要分析复合查询（如存储过程和存储函数），首先需要将执行拆分为单个查询的每个查询使用 EXPLAIN。确定存储程序中各个查询的一个方法是使用性能架构。本章稍后将介绍实现此目的的示例。

解释的最简单是使用分析的查询指定 EXPLAIN：

```
mysql> EXPLAIN <query>;
```

在示例中是您要分析的查询。使用不选项语句将返回传统表格式的结果。如果要指定格式，可以通过添加 

```
FORMAT=TRADITIONAL|JSON|TREE:
mysql> EXPLAIN FORMAT=TRADITIONAL <query>
mysql> EXPLAIN FORMAT=JSON <query>
mysql> EXPLAIN FORMAT=TREE <query>
```



首选的格式取决于您的需求。当您需要查询计划概述、使用的索引以及有关查询计划的其他基本信息时，传统格式更易于使用。JSON 格式提供了更多详细信息，并且更易于应用程序使用。例如，MySQL 工作台中的可视化解释使用 JSON 格式的输出。

树格式是最新的格式，在 MySQL 8.0.16 及更晚版本中受支持。它要求使用 Volcano 表示执行器执行查询，在编写时，并非所有查询都支持该执行器。树格式的特殊用途解释语句。

### 解释分析

解释语句 是 MySQL 8.0.18 的新功能，是使用树的标准 EXPLAIN 语句的扩展。关键区别在于 EXPLAIN实际上执行查询，并在执行查询时收集执行的统计信息。执行语句时，将禁止显示查询的输出，因此仅返回查询计划和统计信息。与树输出格式一样，需要使用火山器执行器。

解释用法与您已经看到的 EXPLAIN 语句：

```
mysql> EXPLAIN ANALYZE <query>
```

EXPLAIN将在本章后面与树格式输出一起讨论。

从性质仅处理显式查询，因为它需要从头到尾监视查询。另方面，纯 EXPLAIN 语句也可用于正在进行的查询。

### 连接用法

假设您正在调查性能不佳的问题，您注意到存在已运行了几个小时的查询。您知道这不应该发生，所以您要分析为什么查询如此缓慢。一个选项是复制查询并为此执行但是，这可能无法提供所需的信息，因为索引统计信息可能已更改，因为慢速查询启动后，因此现在分析查询不会显示导致性能缓慢的实际查询计划。

更好的解决方案是请求用于慢速查询的实际查询计划。您可以使用解释语句的变体获得如果要尝试，则需要长时间运行的查询，例如：

```sql
SELECT * FROM world.city WHERE id = 130 + SLEEP(0.1);
```

这将需要大约 420 秒（世界每行 0.1）。

您将需要要调查的查询的连接 ID，并将它作为参数传递给。可以从进程信息获取连接 ID。例如，如果使用，可以在"链接"列conn_id

```
mysql> SELECT conn_id, current_statement,
 statement_latency
 FROM sys.session
 WHERE command = 'Query'
 ORDER BY time
 DESC LIMIT 1\G
*************************** 1. row ***************************
 conn_id: 8
current_statement: SELECT * FROM world.city WHERE id = 130 + SLEEP(0.1)
statement_latency: 4.22 m
1 row in set (0.0551 sec)
```

为了保持输出简单，它仅限于此示例的感兴趣连接。查询的连接 ID 为 8。您可以使用它获取查询的执行计划，如下所示：

```
mysql> EXPLAIN FOR CONNECTION 8\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: city
 partitions: NULL
 type: ALL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 4188
 filtered: 100
 Extra: Using where
1 row in set (0.0004 sec)
```

您可以选择添加需要格式，其方式与显式指定查询时相同。如果使用的客户端与 MySQL 命令行管理程序不同，则筛选列可能会显示在讨论输出的含义之前，值得熟悉输出格式。

## 解释格式

当您需要检查查询计划时，可以在几种格式之间进行选择。你选择哪一个主要取决于你的喜好。也就是说，JSON 格式确实包含的信息比传统格式和树格式的信息更多。如果您喜欢查询计划的可视化表示形式，则 MySQL 工作台的可视化解释是一个不错的选择。

本节将讨论每种格式，并显示以下计划的输出：

```
SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5;
```



该查询按欧洲地区查找欧洲十个最小国家/地区的五个最大城市，按城市人口按降序排列。选择此查询的原因是它显示各种输出格式如何表示子查询、排序和限制。EXPLAIN 语句将在本节中讨论;推迟到"解释部分。

输出相当详细。为了便于比较输出，本节中的示例与本书的 GitHub 存储库中的查询结果相结合。对于树输出格式（包括在列名称和查询计划之间添加了一个额外的新行，以使树层次结构显示更清晰：

```
*************************** 1. row ***************************
EXPLAIN:
-> Limit: 5 row(s)
 -> Sort: <temporary>.Population DESC, limit input to 5 row(s) per chunk

Instead of:
*************************** 1. row ***************************
EXPLAIN: -> Limit: 5 row(s)
 -> Sort: <temporary>.Population DESC, limit input to 5 row(s) per chunk
```

本章中都使用此约定。

### 传统格式

当您在没有的情况下执行 EXPLAIN或将格式设置为，输出将返回为表，就像查询过普通表一样。当您想要查询计划的概述，并且是检查输出的人工数据库管理员或开发人员时，这非常有用。

输出中有 12 列。如果字段没有任何值，则下一节每一列的含义。清单显示了示例查询的传统输出。

```
Listing 20-1. Example of the traditional EXPLAIN output
mysql> EXPLAIN FORMAT=TRADITIONAL
 SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5\G
*************************** 1. row ***************************
 id: 1
 select_type: PRIMARY
 table: <derived2>
 partitions: NULL
 type: ALL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 10
 filtered: 100
 Extra: Using temporary; Using filesort
*************************** 2. row ***************************
 id: 1
 select_type: PRIMARY
 table: ci
 partitions: NULL
 type: ref
possible_keys: CountryCode
 key: CountryCode
 key_len: 3
 ref: co.Code
 rows: 18
 filtered: 100
 Extra: NULL
*************************** 3. row ***************************
 id: 2
 select_type: DERIVED
 table: country
 partitions: NULL
 type: ALL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 239
 filtered: 14.285715103149414
 Extra: Using where; Using filesort
3 rows in set, 1 warning (0.0089 sec)
Note (code 1003): /* select#1 */ select `world`.`ci`.`ID` AS
`ID`,`world`.`ci`.`Name` AS `Name`,`world`.`ci`.`District` AS
`District`,`co`.`Name` AS `Country`,`world`.`ci`.`Population` AS
`Population` from `world`.`city` `ci` join (/* select#2 */ select
`world`.`country`.`Code` AS `Code`,`world`.`country`.`Name` AS
`Name` from `world`.`country` where (`world`.`country`.`Continent`
= 'Europe') order by `world`.`country`.`SurfaceArea` limit 10)
`co` where (`world`.`ci`.`CountryCode` = `co`.`Code`) order by
`world`.`ci`.`Population` desc limit 5
```

请注意第一个表如何称为。这是用于国家/地区表查询，数字 2 是指的 id 列的值。""列包含查询临时表和文件排序等信息。输出的末尾是优化器重写后查询。在许多情况下，更改并不多，但在某些情况下，优化器可能能够对查询进行重大更改。在重写的查询中，请注意注释（例如） 如何用于显示该部分使用哪个 ID 值。重写的查询中可能还有其他提示，以说明如何执行查询。重写的查询由 SHOW 警告作为注释，由 MySQL Shell 隐式执行）。

输出可能看起来势不可挡，而且很难理解如何使用这些信息来分析查询。一旦讨论了其他输出格式、选择类型和联接类型的详细信息以及额外信息，就会有一些信息。

如果要以编程方式分析查询计划该怎么办？您可以像正常查询那样处理，也可以请求 JSON 格式的信息，其中包括一些附加信息。

### JSON 格式

自 MySQL 5.6 以来，可以使用格式请求输出。与传统表格式相比，JSON 格式的一个优点是，JSON 格式的附加灵活性已用于以更合乎逻辑的方式对信息进行分组。

JSON 输出中的基本概念是查询。查询定义查询的一部分，并可能反过来包括其自己的查询块。这允许 MySQL 向它们所属的查询块指定查询执行的详细信息。这也从清单。

```
mysql> EXPLAIN FORMAT=JSON
 SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5\G
*************************** 1. row ***************************
EXPLAIN: {
 "query_block": {
 "select_id": 1,
 "cost_info": {
 "query_cost": "247.32"
 },
 "ordering_operation": {
 "using_temporary_table": true,
 "using_filesort": true,
 "cost_info": {
 "sort_cost": "180.52"
 },
 "nested_loop": [
 {
 "table": {
 "table_name": "co",
 "access_type": "ALL",
 "rows_examined_per_scan": 10,
 "rows_produced_per_join": 10,
 "filtered": "100.00",
 "cost_info": {
 "read_cost": "2.63",
 "eval_cost": "1.00",
 "prefix_cost": "3.63",
 "data_read_per_join": "640"
 },
 "used_columns": [
 "Code",
 "Name"
 ],
 "materialized_from_subquery": {
 "using_temporary_table": true,
 "dependent": false,
 "cacheable": true,
 "query_block": {
 "select_id": 2,
 "cost_info": {
 "query_cost": "25.40"
 },
 "ordering_operation": {
 "using_filesort": true,
 "table": {
 "table_name": "country",
 "access_type": "ALL",
 "rows_examined_per_scan": 239,
 "rows_produced_per_join": 34,
 "filtered": "14.29",
 "cost_info": {
 "read_cost": "21.99",
 "eval_cost": "3.41",
 "prefix_cost": "25.40",
 "data_read_per_join": "8K"
 },
 "used_columns": [
 "Code",
 "Name",
 "Continent",
 "SurfaceArea"
 ],
 "attached_condition": "(`world`.`country`.`Continent` =
'Europe')"
 }
 }
 }
 }
 }
 },
 {
 "table": {
 "table_name": "ci",
 "access_type": "ref",
 "possible_keys": [
 "CountryCode"
 ],
 "key": "CountryCode",
 "used_key_parts": [
 "CountryCode"
 ],
 "key_length": "3",
 "ref": [
 "co.Code"
 ],
 "rows_examined_per_scan": 18,
 "rows_produced_per_join": 180,
 "filtered": "100.00",
 "cost_info": {
 "read_cost": "45.13",
 "eval_cost": "18.05",
 "prefix_cost": "66.81",
 "data_read_per_join": "12K"
 },
 "used_columns": [
 "ID",
 "Name",
 "CountryCode",
 "District",
 "Population"
 ]
 }
 }
 ]
 }
 }
}
1 row in set, 1 warning (0.0061 sec)
```

如您所见，输出相当冗长，但结构使得相对容易看到哪些信息属于一起，以及查询的各个部分如何相互关联。在此示例中，有一个包含两个表和的嵌套循环。共本身包括一个新的查询块，该查询块是使用国家/地区表实现化的查询。

JSON 格式还包括其他信息，例如每个部件的估计。成本可用于查看优化器认为查询最昂贵的部分在哪里。例如，如果您看到查询的一部分成本非常高，但您了解数据意味着您知道数据应该很便宜，则可以表明索引统计信息不是最新的，或者需要直方图。

使用 JSON 格式输出的最大问题是，有这么多的信息和这么多的输出行。一种非常方便的方法来绕过它，使用 MySQL 工作台中的可视化解释功能，该功能在讨论树格式输出后介绍。

### 树格式

树格式侧重于描述如何根据查询部分与执行部分的顺序之间的关系执行查询。从这个意义上说，它可能听起来类似于JSON输出;但是，树格式读的更简单，而且没有多少细节。树格式在 MySQL 8.0.16 中作为实验功能引入，它依赖于火山流器执行器。从 MySQL 8.0.18 开始，树格式也用于 功能。

清单显示了使用示例格式的输出。此输出是非分析版本。对于同一查询显示 EXPLAIN ANALYZE 的输出示例，以便您看到差异。

```
Listing 20-3. Example of the tree EXPLAIN output
mysql> EXPLAIN FORMAT=TREE
 SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5\G
*************************** 1. row ***************************
EXPLAIN:
-> Limit: 5 row(s)
 -> Sort: <temporary>.Population DESC, limit input to 5 row(s) per chunk
 -> Stream results
 -> Nested loop inner join
 -> Table scan on co
 -> Materialize
 -> Limit: 10 row(s)
 -> Sort: country.SurfaceArea, limit input to 10
row(s) per chunk (cost=25.40 rows=239)
 -> Filter: (country.Continent = 'Europe')
 -> Table scan on country
 -> Index lookup on ci using CountryCode
(CountryCode=co.`Code`) (cost=4.69 rows=18)
```

输出提供了如何执行查询的大致概述。通过从内向和从内到外在一定程度上读取输出，可以更容易地理解执行。对于嵌套循环，您有两个表，其中第一个表是 co 上的表（缩进已减小）：

```
-> Table scan on co
 -> Materialize
 -> Limit: 10 row(s)
 -> Sort: country.SurfaceArea, limit input to 10 row(s) per
chunk (cost=25.40 rows=239)
 -> Filter: (country.Continent = 'Europe')
 -> Table scan on country
```

在这里，您可以看到表是如何通过首先在该国表上执行表扫描，然后应用大陆筛选器，然后根据表面积进行排序，然后将结果限制在 10 行来创建的物化子查询。

嵌套循环的第二部分更简单，因为它只是使用表（表）上查找：

```
-> Index lookup on ci using CountryCode (CountryCode=co.`Code`) (cost=4.69
rows=18)
```

当使用内部联接解析嵌套循环时，结果将流到排序（即，未实现），并返回前五行：

```
-> Limit: 5 row(s)
 -> Sort: <temporary>.Population DESC, limit input to 5 row(s) per chunk
 -> Stream results
 -> Nested loop inner join
```

虽然这不像 JSON 输出那样详细，但它仍然包含大量有关查询计划的信息。这包括每个表的估计成本和估计行数。例如，从国家表面积上的排序步骤

```
(cost=25.40 rows=239)
```

一个很好的问题是，这与查询表的实际成本有什么关系。您可以使用解释语句。清单显示了为示例查询生成的输出示例。

```
Listing 20-4. Example of the EXPLAIN ANALYZE output
mysql> EXPLAIN ANALYZE
 SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5\G
*************************** 1. row ***************************
EXPLAIN: -> Limit: 5 row(s) (actual time=34.492..34.494 rows=5 loops=1)
 -> Sort: <temporary>.Population DESC, limit input to 5 row(s) per
chunk (actual time=34.491..34.492 rows=5 loops=1)
 -> Stream results (actual time=34.371..34.471 rows=15 loops=1)
 -> Nested loop inner join (actual time=34.370..34.466 rows=15
loops=1)
 -> Table scan on co (actual time=0.001..0.003 rows=10
loops=1)
 -> Materialize (actual time=34.327..34.330 rows=10
loops=1)
 -> Limit: 10 row(s) (actual time=34.297..34.301
rows=10 loops=1)
 -> Sort: country.SurfaceArea, limit input to
10 row(s) per chunk (cost=25.40 rows=239)
(actual time=34.297..34.298 rows=10 loops=1)
 -> Filter: (world.country.Continent =
'Europe') (actual time=0.063..0.201
rows=46 loops=1)
 -> Table scan on country (actual
time=0.057..0.166 rows=239 loops=1)
 -> Index lookup on ci using CountryCode
(CountryCode=co.`Code`) (cost=4.69 rows=18) (actual
time=0.012..0.013 rows=2 loops=10)
1 row in set (0.0353 sec)
```

这与 FORMAT+TREE树，只是每个步骤都有有关性能的信息。如果查看 ci 表的，可以看到有两个计时，行数和循环数（重新格式化以提高可读性）：

```
-> Index lookup on ci using CountryCode
 (CountryCode=co.`Code`)
 (cost=4.69 rows=18)
 (actual time=0.012..0.013 rows=2 loops=10)
```

此处，预期 18 行（每个循环）的估计成本为 4.69。实际统计信息显示，第一行在 0.012 毫秒后读取，所有行在 0.013 毫秒后读取。有十个循环（每十个国家一个），每个循环平均取两行，总共 20 行。因此，在这种情况下，估计不是很准确（因为查询只选择小国）。

如果在 MySQL 8.0.18 及以后有使用哈希联接的查询，则需要使用树格式的输出来确认何时使用哈希联接算法。例如，如果使用哈希将城市表与地区表

```
mysql> EXPLAIN FORMAT=TREE
 SELECT CountryCode, country.Name AS Country,
 city.Name AS City, city.District
 FROM world.country IGNORE INDEX (Primary)
 INNER JOIN world.city IGNORE INDEX (CountryCode)
 ON city.CountryCode = country.Code\G
*************************** 1. row ***************************
EXPLAIN:
-> Inner hash join (world.city.CountryCode = world.
country.`Code`) (cost=100125.16 rows=4314)
 -> Table scan on city (cost=0.04 rows=4188)
 -> Hash
 -> Table scan on country (cost=25.40 rows=239)
1 row in set (0.0005 sec)
```

请注意联接是"内部并且国家/地区扫描是使用哈希

迄今为止，所有示例都使用了基于文本的输出。特别是 JSON 格式的输出可能很难用于获取查询计划的概述。对于这个视觉解释是一个更好的选择。

### 视觉解释

可视化解释功能是 MySQL 工作台的一部分，通过将 JSON 格式的查询计划转换为图形表示来工作。在研究向 sakila.film 表添加直方图的效果时，您已经在第

通过单击闪电符号前面的放大镜图标，获得视觉解释图，如图。

![](../附图/Figure%2020-1.png)

如果查询需要很长时间才能执行或查询修改数据，则这是生成查询计划的特别有用方法。如果已执行查询，也可以单击结果"执行计划"图标，如图。

![](../附图/Figure%2020-2.png)

可视化解释关系图创建为流程图块和表有一个矩形。数据处理使用其他形状（如联接的菱形）进行描述。图显示了可视化解释中使用的每个基本形状的示例。

![](../附图/Figure%2020-3.png)

在图中，查询块为灰色，而表的两个示例（子查询中的单行查找和完整表扫描）分别是蓝色和红色。例如，在联合的情况下，还使用灰色块。表格框下面的文本以标准文本显示表名或别名，以粗体文本显示索引名称。具有圆角的矩形描述行上的操作，如排序、分组、不同操作等。

左上角的数字是该表、操作或查询块的相对成本。表和联接的右上右侧的数字是估计要转发的行数。操作的颜色用于显示应用操作的成本。表还使用基于表访问类型的颜色，主要是对类似的访问类型进行分组，其次用于指示访问类型的成本。使用从可视化，颜色和成本之间的关系可以在图。

![](../附图/Figure%2020-4.png)

蓝色 （1） 是最便宜的;绿色 （2）、黄色 （3） 和橙色 （4） 代表低至中等成本;和最昂贵的访问类型和操作是红色象征高 （5） 到非常高 （6） 成本。

颜色组之间有很多重叠。每个成本估算都考虑一个"平均"用例，因此不应将成本估算视为绝对事实。是复杂的，有时一种通常比另一种方法便宜的方法对于一个特定的查询结果会提供更好的性能。

对于表，成本与访问类型关联，访问类型是输出列 JSON 格式输出中的"类型"字段中的值。图显示了 Visual Explain 如何表示当前存在的 12 种访问类型。访问类型的推迟到下一节。

![](../附图/Figure%2020-5.png)

此外，Visual Explain 具有"未知"访问类型为黑色，以防遇到未知访问类型。访问类型按颜色和大致成本从左到右排序，然后从上到下排序。

图将所有这些内容放在一以显示本节中使用的示例查询的查询计划。

![](../附图/Figure%2020-6.png)

从左下到右，然后向上阅读图表。因此，该图显示，首先执行国家表上具有完整表扫描的子查询，然后使用非唯一索引查找在物化的co 表一个完整表扫描，与 ci （） 表上的行连接。最后，使用临时表和文件排序对结果进行排序。

如果想要比图表最初显示的更多的详细信息，您可以将鼠标悬停在查询计划想要了解更多部分的上。图显示了 ci 表包含的示例。

![](../附图/Figure%2020-7.png)

弹出式框架不仅显示 JSON 输出中可用的其余详细信息，还提供了有助于了解数据含义的提示。所有这些都意味着 Visual Explain 是开始通过查询计划分析查询的一种很好的方法。获得经验后，您可能更喜欢使用基于文本的输出，尤其是如果您更喜欢从 shell 工作，但不要忽略 Visual Explain，因为您认为最好使用基于文本的输出格式。即使对于专家来说，Visual Explain 也是了解查询执行方式的一个很好的工具。

希望讨论输出格式可以让您了解 EXPLAIN 可以哪些信息。然而，要充分理解它并利用它，有必要更深入地了解信息的含义。

## 解释输出

解释输出中有很多可用的信息，因此值得深入研究此信息的含义。本节首先概述传统和 JSON 格式输出中包含的字段;然后，选择类型和访问类型和额外的信息将更详细地介绍。

### 解释字段

在您的工作中建设性地使用语句来改进查询的第一步是了解哪些信息可用。信息范围从用于查询部件的 ID 到有关可用于查询的索引的详细信息，以及与使用哪些 ID 和应用哪些优化器功能相比。

如果您第一次阅读定义后无法回忆所有详细信息，则不必担心。大多数字段都是非常不言自明的，因此您可以对它们表示的数据进行限定的猜测。当您自己分析某些查询时，您也会很快熟悉这些信息。表列出了传统格式中包含的所有字段以及 JSON 格式中的一些公共字段。

| 传统          | Json                   | 描述                                                         |
| :------------ | :--------------------- | :----------------------------------------------------------- |
| Id            | select_id              | 显示表或子查询的查询的哪一部分的数字标识符。顶级表有 id 第一个子查询有等等。对于联合，ID 将为表值设置为另表列），用于表示联合结果聚合的行。 |
| select_type   |                        | 这显示了表如何包含在整体语句中。在"选择类型"部分的稍后部分将讨论已知的选择类型。对于 JSON 格式，选择类型由 JSON 文档的结构以及从属和可缓存。 |
|               | 依赖                   | 它是否是依赖子查询，也就是说，它取决于查询的外部部分。       |
|               | 缓存                   | 子查询的结果是否可以缓存，或者必须为外部查询中的每一行重新评估子查询的结果。 |
| 表            | table_name             | 表或子查询的名称。如果已指定别名，则使用的是别名。这可确保每个表名对于 id 列的给定值都的。特殊情况包括联合、派生表和具体化子查询，其中表名为，其中 N 和 M 分别引用查询计划前面部分的 |
| 分区          | 分区                   | 将为查询包含的分区。您可以使用它来确定分区修剪是否按预期应用。 |
| 类型          | access_type            | 如何访问数据。这显示了优化器如何决定限制表中检查的行数。这些类型将在"访问类型"部分中讨论。 |
| possible_keys | possible_keys          | 要用于表的候选索引的列表。使用架构的键名称表示自动生成的索引可用。 |
| 关键          | 关键                   | 为表选择的索引。使用架构键名称意味着使用自动生成的索引。     |
| key_len       | key_length             | 索引使用的字节数。对于由多个列组成的索引，优化器可能只能使用列的子集。在这种情况下，可以使用密钥长度来确定索引的多少对于此查询有用。如果索引中的列支持值，则与 NOT NULL 列的情况相比，将 1中。 |
|               | used_key_parts         | 使用的索引中的列。                                           |
| 裁判          | 裁判                   | 对执行筛选时执行的操作。例如，对于"等条件，这可以是常量。    |
| 行            | rows_examined_per_scan | 包含表的结果的行数的估计值。对于联接到较早表的表，它是估计每个联接找到的行数。特殊情况是当引用是表上的主要键或唯一键时，在这种情况下，行估计值正好是 1。 |
|               | rows_produced_per_join | 联接产生的估计行数。实际上，预期循环数与的行的百分比相乘。   |
| 过滤          | 过滤                   | 这是将包含多少个已检查行的估计值。该值以百分比表示，因此对于值 100.00，将返回所有检查的行。值 100.00 是最佳值，最差值为。注意：传统格式的值的舍入取决于您使用的客户端。例如，MySQL 命令程序将返回命令行客户端返回。 |
|               | cost_info              | 具有查询部件成本细分的 JSON 对象。                           |
| 额外          |                        | 有关优化器决策的其他信息。这可能包括有关使用的排序算法、是否使用覆盖索引等的信息。支持的最常见值将在"额外信息"部分中讨论。 |
|               | 消息                   | 对于 JSON中没有专用字段的传统输出的"额外"列中的信息。一个例子是。 |
|               | using_filesort         | 是否使用文件排序。                                           |
|               | using_index            | 是否使用覆盖索引。                                           |
|               | using_temporary_table  | 操作（如子查询或排序）是否需要内部临时表。                   |
|               | attached_condition     | 与查询部分关联的子句。                                       |
|               | used_columns           | 表中所需的列。这很有用，可以查看您是否接近能够使用覆盖索引。 |

某些信息最初在 JSON 格式中显示缺失，因为该字段仅存在于传统格式中。事实并非如此;相反，信息是使用其他方法提供的，例如中的几个消息在 JSON 格式中有自己的字段。其他消息使用字段。本节稍后讨论"额外"列中的信息时，将提及 JSON 输出包含的一些字段。

通常，如果值为 false，则省略 JSON 格式输出中的布尔;一个例外，因为与可缓存案例相比，不可缓存的子查询或联合表示成本更高。

对于 JSON，也有用于对操作信息进行分组的字段。操作范围从访问表到对多个操作进行分组的复杂操作。一些常见的操作，以及触发它们的示例

- 访问表。这是操作的最低级别。
- 具有一个查询块对应于传统格式的的最高级别概念。所有查询至少有一个查询块。
- 联接操作。
- 例如，由组 BY 子句操作。
- 例如，为 ORDER BY 子句。
- 例如，使用 DISTINCT 关键字时操作。
- 使用窗口函数产生的操作。
- 执行子查询并实现结果。
- 附加到查询其余部分的子查询。例如，这发生在 IN 子句中的子查询的查询的子查询的子查询等句例中。
- 对于使用合并两个或多个查询结果的查询。在块中，该块包含联合中每个查询的定义。

表字段和复杂操作列表对于 JSON 格式并不全面，但它应该让您了解可用的信息。通常，字段名称本身包含信息，并且与发生这些名称的上下文相结合通常足以理解字段的含义。不过，某些字段的值值得更多关注 - 从选择类型开始。

### 选择类型

选择显示查询的每个部分的查询块类型。在此上下文中，查询的一部分可以包含多个表。例如，如果您有一个简单的查询加入表列表，但不使用构造（如子查询），则所有表都将在查询的同一（也仅）部分。查询的每个部分都获取自己的 JSON 输出中）。

有几种选择类型。对于大多数，JSON 输出中没有直接字段;但是，可以从结构和某些其他字段派生选择类型。表显示了当前存在的选择类型，提示如何从 JSON 输出派生类型。在表中，"选择类型的值是用于传统输出列的值。

| 选择类型         | Json                       | 描述                                                       |
| :--------------- | :------------------------- | :--------------------------------------------------------- |
| 简单             |                            | 对于不使用派生表、子查询、联合或类似方法的查询。           |
| 主要             |                            | 对于使用子查询或联合的查询，主要部分是最外层的部分。       |
| 插入             |                            | 对于语句。                                                 |
| 删除             |                            | 用于语句。                                                 |
| 更新             |                            | 对于语句。                                                 |
| 取代             |                            | 用于语句。                                                 |
| 联盟             |                            | 对于联合语句，第二个或以后语句。                           |
| 依赖联合         | 依赖=真实                  | 对于联合语句，第二个或更晚的语句依赖于外部查询。           |
| 联合结果         | union_result               | 聚合联合语句的结果的查询部分。                             |
| 子奎里           |                            | 用于列表中的 SELECT 语句。                                 |
| 依赖子查询       | 依赖=真实                  | 对于从属子挤压，第一个语句。                               |
| 派生             |                            | 派生表 - 通过查询创建的表，但其行为类似于普通表。          |
| 依赖派生         | 依赖=真实                  | 依赖于另一个表的派生表。                                   |
| 物化             | materialized_from_subquery | 物化子查询。                                               |
| 不可缓存的子查询 | 可缓存=假                  | 必须为外部查询中的每一行计算结果的子查询。                 |
| 不可缓存的联盟   | 可缓存=假                  | 对于联合语句，第二个或更晚的语句是不可缓存子查询的一部分。 |

某些选择可以视为信息，以便更轻松地了解您正在查看的查询的哪个部分。例如，这包括。但是，某些选择类型指示它是查询的昂贵部分。这尤其适用于不可缓存的类型。依赖类型还意味着优化器在确定执行计划中添加表的哪个部分时的灵活性较低。如果查询速度较慢，并且看到无法缓存或从属部分，则值得研究是否可以重写这些部分或将查询拆分为两个部分。

另一个重要的是如何访问表。

### 访问类型

讨论时已经遇到表访问类型。它们显示查询是否使用索引、扫描和类似访问表。由于与每种访问类型相关的成本差异很大，因此它也是输出中查找的重要值之一，以确定查询的哪些部分用于提高性能。

本小节的其余部分总结了 MySQL 中的访问类型。标题是传统格式的类型中使用的值。对于每种访问类型，都有一个使用该访问类型的示例。

#### 系统

系统访问与只有一行的表一起使用。这意味着表可以视为常量。可视化解释成本、消息和颜色如下所示：

- **成本：非常低**
- 单行（系统常量）
- 蓝色

使用系统访问类型的示例是

选择 *

从 （选择 1） my_table

访问类型是。

#### 常量

表最多一行，例如，主键或唯一索引的单个值上存在筛选器。可视化解释成本、消息和颜色如下所示：

- 非常低
- 单行（常量）
- 蓝色

使用访问类型的查询示例是

选择 *

来自世界. city

其中 ID = 130;

#### eq_ref

该是联接中的右侧表，其中表上的条件位于主键或不空的唯一索引上。可视化解释成本、消息和颜色如下所示：

- 低
- 唯一的键查找
- 绿色

使用访问类型eq_ref示例是

选择 *

来自世界. city

STRAIGHT_JOIN世界. 国家

关于国家代码 = 代码;

访问是 ref 访问类型的专用，每个查找只能返回一行。

#### 裁判

由非统一辅助索引筛选。可视化解释成本、消息和颜色如下所示：

- 低到中
- 非唯一密钥查找
- 绿色

使用 ref 访问类型的示例是

选择 *

来自世界. city

国家代码 = "澳大利亚";

#### ref_or_null

与，但筛选的列也可能为。可视化解释成本、消息和颜色如下所示：

- 低到中
- 键查找 = 获取 NULL 值
- 绿色

使用访问类型ref_or_null是

选择 *

从 sakila. 付款

在哪里rental_id # 1

或rental_id为空;

#### index_merge

优化选择两个或多个索引的组合，以解析包含不同索引中列中的或的筛选器。可视化解释成本、消息和颜色如下所示：

- 中等
- 索引合并
- 绿色

使用访问类型

选择 *

从 sakila. 付款

在哪里rental_id # 1

或customer_id = 5;

虽然成本被列为中等成本，但最常见的严重性能问题之一是查询通常使用单个索引或执行完整表扫描，并且索引统计信息变得不准确，因此优化器选择索引合并。如果使用索引合并的性能不佳，请尝试告诉优化器忽略索引合并优化或使用的索引，并查看是否有帮助或分析表以更新索引统计信息。或者，查询可以重写为两个查询的联合，每个查询使用筛选器的一部分。这方面的一个示例将在第。

#### 全文

优化器选择全文索引来筛选表。可视化解释成本、消息和颜色如下所示：

- 低
- 全文索引搜索
- 黄色

使用全文访问类型的为：

选择 *

从 sakila.film_text

匹配的地方（标题、描述）

反对（布尔模式下的"循环"）;

#### unique_subquery

对于 IN 运算符，其中子查询返回主键或唯一索引的值。在 MySQL 8 中，这些查询通常由优化优化器交换机。可视化解释成本、消息和颜色如下所示：

- 低
- 对子查询表的唯一键查找
- 橙

使用访问类型unique_subquery示例是

设置optimizer_switch = "实现=关闭，半连";

选择 *

来自世界. city

国家代码在 （

选择代码

来自世界. 国家

大陆 = "大洋洲"）;

设置optimizer_switch = "实现= 打开，半连";

对于或唯一索引的情况访问方法是一种特殊情况。

#### index_subquery

对于 IN 运算符中的子，其中子查询返回辅助非统一索引的值。在 MySQL 8 中，这些查询通常由优化器连接优化器交换机。可视化解释成本、消息和颜色如下所示：

- 低
- 非独特键查找到子查询表
- 橙

使用访问类型index_subquery示例是

设置optimizer_switch = "实现=关闭，半连";

选择 *

来自世界. 国家

代码在 （

选择国家/地区代码

来自世界. city

名称 = "悉尼"）;

设置optimizer_switch = "实现= 打开，半连";

#### 范围

当用于按顺序或组查找多个值时，使用范围访问类型。它既用于显式范围，如 ID 1 和，用于子句，也用于同一列上的多个条件由。可视化解释成本、消息和颜色如下所示：

- 中等
- 索引范围扫描
- 橙

使用范围访问类型的示例是

选择 *

来自世界. city

其中ID在（130，3805）;

使用范围访问的成本在很大程度上取决于范围中包含的行数。在一个极端情况下，范围扫描仅使用主键匹配一行，因此成本非常低。另一方面，范围扫描包括使用辅助索引的表的很大一部分，在这种情况下，执行完整表扫描可能更便宜。

访问类型与索引访问相关，区别在于是否需要部分扫描或完全扫描。

#### 指数

优化器已选择执行完整的索引扫描。这可以通过使用覆盖索引的组合进行选择。可视化解释成本、消息和颜色如下所示：

- 高
- 全索引扫描
- 红

使用索引访问类型的示例是

选择 ID，国家/地区代码

来自世界.城市;

由于索引扫描需要使用主键进行第二次查找，因此，除非索引是查询的覆盖索引，否则可能会非常昂贵，以至于执行完整表扫描的成本最终更低。

#### 所有

最基本的访问类型是扫描的所有行。这也是最昂贵的访问类型，因此该类型是用所有大写编写的。可视化解释成本、消息和颜色如下所示：

- 非常高
- 全表扫描
- 红

使用访问类型的查询示例是

选择 *

来自世界.城市;

如果使用完整表扫描看到第一个表以外的表，则通常是一个红色标志，指示表上缺少条件或没有可以使用的索引。是否为第一个表的合理访问类型取决于查询所需的表量;所需的表部分越大，完整的表扫描越合理。

现在讨论访问类型到此结束。在本书的稍后部分查看示例时，以及在本书的稍后部分查看优化查询时（例如，在第 24 章 中）中，将再次。同时，让我们看看"额外"列中。

### 额外信息

传统格式中的"额外"列是一个"全集"，用于使用没有自己的列的信息。引入 JSON 格式时，没有理由保留该格式，因为引入其他字段很容易，也没有必要包括每个输出的所有字段。因此，JSON 格式没有"额外字段，而是具有一系列字段。一些剩余的消息已留给一个泛字段。

一些更常见的消息包括

- 当使用覆盖索引时。对于 JSON 格式字段设置为。
- ：当使用索引测试是否需要读取完整行时。例如，当索引列上存在范围条件时，使用此选项。对于 JSON 格式字段设置与筛选器条件。
- 当子句应用于表而不使用索引时。这可能表示表上的索引不是最佳的。在 JSON 格式中字段设置为筛选器条件。
- 使用松散索引扫描来解决组DISTINCT。在 JSON 格式字段设置为。
- 这意味着在不使用索引的地方进行联接，因此改为使用联接缓冲区。包含此消息的表是添加索引的候选项。对于 JSON 格式字段设置为需要注意的一件事是，当使用哈希联接时，传统和 JSON 格式的输出仍将显示使用块嵌套循环。若要查看它是实际的块嵌套循环联接还是哈希联接，需要使用树格式的输出。
- ）：这意味着联接使用批优化。若要启用批处理密钥访问优化，为默认值关闭）默认优化器开关。 优化需要联接的索引，因此与使用块嵌套循环的联接缓冲区不同，使用批处理密钥访问算法并不是对表进行昂贵访问的标志。对于 JSON 格式设置为批。
- 优化。这有时用于减少需要完整行的辅助索引上范围条件的随机 I/O 量。优化由 mrr 和默认情况下都启用）。 对于 JSON 格式设置为
- 使用通道来确定如何以正确的顺序检索行。例如，这发生在按辅助索引排序时;索引不是覆盖索引。对于 JSON 格式字段设置为。
- ：内部临时表用于存储子查询的结果、排序或分组。对于排序和分组，有时可以通过添加索引或重写查询来避免使用内部临时表。对于 JSON 格式字段设置为。
- 这三条消息用于索引合并，以表示如何执行索引合并。对于任一消息，有关索引合并中涉及的索引的信息都包含在括号中。对于 JSON 格式字段指定使用的方法和索引。
- 表是递归公共表表达式 （CTE） 的一部分。对于 JSON 格式字段设置为。
- 如果第二个表的索引列上存在依赖于第一个表中列的值，例如上的索引：选择 * 从 t1 内部联接 t2 其中则会发生这种情况;这就是触发性能语句事件表中的计数器递增的触发因素。索引映射是一个位掩，指示哪些索引是范围检查的候选索引。索引号基于 1，如 SHOW 。当您写出位掩码时，具有位集的索引号是候选项。对于 JSON 格式字段设置为索引映射。
- 当有一为 true 的筛选器时，例如 WHERE 。如果筛选器中的值超出数据类型支持的范围，则同样适用，例如表示数据类型。对于 JSON 格式，消息将。
- 不可能， 除了它适用于使用系统const 访问后。例如，对于 JSON 格式，消息将。
- 与不可能，因为它适用于 HAVING句。对于 JSON 格式，消息将。
- ：当优化器选择使用类似于松散索引扫描的多个范围扫描时。例如，它可用于覆盖索引，其中索引的第一列不用于筛选器条件。此方法在 MySQL 8.0.13 及更晚版本中可用。对于 JSON 格式字段设置为。
- 此消息意味着 MySQL 能够从查询中删除该表，因为只会生成一行，并且该行可以从一组确定性行生成。通常，当表中仅需要索引的最小值和/或最大值时，将发生此情况。对于 JSON 格式，消息将。
- 对于不涉及任何表的子查询，例如，从双表中选择 1;对于不涉及任何表的子查询，对于不涉及任何表的子查询，从双表中选择 1 个。对于不涉及对于 JSON 格式，消息将。
- 访问类型的表，但没有匹配条件的行。对于 JSON 格式，消息将。

结束关于解释语句输出含义结束。剩下的就是开始使用它来检查查询计划。

## 解释示例

要完成对查询计划的讨论，值得通过几个例子来更好地了解如何将所有这些计划放在一起。此处的示例旨在作为导言。本书的其余部分，特别是第24章，将会发生。

### 单个表，表扫描

作为第一个示例，考虑对数据库中的城市查询，该查询的条件在非索引列。由于没有可以使用的索引，因此需要完整的表扫描来评估查询。匹配这些要求的查询示例是

```
SELECT *
 FROM world.city
 WHERE Name = 'London';
```



清单显示了的传统EXPLAIN 输出。

```
Listing 20-5. The EXPLAIN output for a single table with a table scan
mysql> EXPLAIN
 SELECT *
 FROM world.city
 WHERE Name = 'London'\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: city
 partitions: NULL
 type: ALL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 4188
 filtered: 10
 Extra: Using where
1 row in set, 1 warning (0.0007 sec)
```



输出具有设置为的访问类型，这也是预期，因为索引的列上没有条件。估计将检查 4188 行（实际数字为 4079），对于每行，将应用子句中的条件。预计所检查的行的 10% 将匹配子句（请注意，根据使用的客户端，可能会显示或）。回想第，优化器使用默认值来估计各种条件的筛选效果，因此不能直接使用筛选值来估计索引是否有用。

在图。

![](../附图/Figure%2020-8.png)

完整表扫描由红色"框显示，可以看到成本估计为 425.05。

此查询只返回两行（该表有一个英国伦敦和加拿大安大略省）。如果请求一个国家/地区的所有城市，会发生什么？

### 单个表，索引访问

第二个示例类似于第一个示例，但筛选器条件已更改为使用非统一索引的"国家代码"列。这应该会使得访问匹配行更便宜。对于此示例，将德国城市：

```
SELECT *
 FROM world.city
 WHERE CountryCode = 'DEU';
```



清单显示了的传统 EXPLAIN 输出。

```
Listing 20-6. The EXPLAIN output for a single table with index lookups
mysql> EXPLAIN
 SELECT *
 FROM world.city
 WHERE CountryCode = 'DEU'\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: city
 partitions: NULL
 type: ref
possible_keys: CountryCode
 key: CountryCode
 key_len: 3
 ref: const
 rows: 93
 filtered: 100
 Extra: NULL
1 row in set, 1 warning (0.0008 sec)
```



这一显示代码索引可用于查询列显示使用索引。访问为以反映表访问使用非统一索引。估计将访问 93 行，这非常精确，因为优化器会询问 InnoDB 将匹配多少行。筛选列显示索引在筛选表方面做得很好。相应的视觉解释图如图。

![](../附图/Figure%2020-9.png)

尽管返回的行数是第一个示例的 45 倍以上，但成本估计仅为 28.05 或不到完整表扫描成本的十分之一。

如果仅使用和国家，会发生什么？

### 两个表和一个覆盖索引

如果有一个包含表中所需的所有列，则称为覆盖。MySQL 将使用此功能来避免检索整行。由于"国家代码"索引是非统一索引，因此它列，因为它是主键。为使查询更逼真，查询还将包括国家/地区表，并基于大陆筛选包含的国家/地区。此类查询的一个示例是

```
SELECT ci.ID
 FROM world.country co
 INNER JOIN world.city ci
 ON ci.CountryCode = co.Code
 WHERE co.Continent = 'Asia';
```



清单显示了的传统 EXPLAIN 输出。

```
Listing 20-7. The EXPLAIN output for a simple join between two tables
mysql> EXPLAIN
 SELECT ci.ID
 FROM world.country co
 INNER JOIN world.city ci
Figure 20-9. Visual Explain diagram for a single table with index lookup
 ON ci.CountryCode = co.Code
 WHERE co.Continent = 'Asia'\G
*************************** 1. row ***************************
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
 filtered: 14.285715103149414
 Extra: Using where
*************************** 2. row ***************************
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
 Extra: Using index
```



查询计划显示，优化器已选择从 co ） 表上的扫描开始， （城市）索引。这里特别的是，"额外""使用因此，没有必要阅读城市表的完整。另请注意，键长度为 3（字节），这是"国家代码"列宽度。在图。

![](../附图/Figure%2020-10.png)

key_len不包括索引的主要关键部分，即使它被使用。然而，查看多列索引是很有用的。

### 多列指数

国家有一个主键，其中包括国家和列。假设您想要找到在一个国家/地区使用的所有语言;在这种情况下，你需要过滤国家代码索引仍可用于执行筛选，您可以使用查看索引的使用量。一个可用于查找中国所有语言的查询是

```
SELECT *
 FROM world.countrylanguage
 WHERE CountryCode = 'CHN';
```



清单显示了的传统 EXPLAIN 输出。

```
Listing 20-8. The EXPLAIN output using part of a multicolumn index
mysql> EXPLAIN
 SELECT *
 FROM world.countrylanguage
 WHERE CountryCode = 'CHN'\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: countrylanguage
 partitions: NULL
 type: ref
possible_keys: PRIMARY,CountryCode
 key: PRIMARY
 key_len: 3
 ref: const
 rows: 12
 filtered: 100
 Extra: NULL
```



主键的总宽度为"国家/地区语言"列的 3，来自"语言"列的 30。由于列显示仅使用 3 个字节，因此可以得出结论筛选的索引的国家语言部分（索引的已用部分始终是最左侧的部分）。在可视化解释中，您需要将鼠标悬停在有关表上，以获得如图。

![](../附图/Figure%2020-11.png)

在图中，在"键/：主要"下。这直接显示仅代码列。

作为最后一个示例，让我们在浏览 EXPLAIN 格式时用作查询。

### 两个表具有子查询和排序

本章前面广泛使用的示例查询将用于结束有关的讨论。查询使用各种功能的组合，因此它触发了已讨论的信息的几个部分。它也是具有多个查询块的查询的示例。作为提醒，查询在这里重复。

在清单。

```
mysql> EXPLAIN
 SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5\G
*************************** 1. row ***************************
 id: 1
 select_type: PRIMARY
 table: <derived2>
 partitions: NULL
 type: ALL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 10
 filtered: 100
 Extra: Using temporary; Using filesort
*************************** 2. row ***************************
 id: 1
 select_type: PRIMARY
 table: ci
 partitions: NULL
 type: ref
possible_keys: CountryCode
 key: CountryCode
 key_len: 3
 ref: co.Code
 rows: 18
 filtered: 100
 Extra: NULL
*************************** 3. row ***************************
 id: 2
 select_type: DERIVED
 table: country
 partitions: NULL
 type: ALL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 239
 filtered: 14.285715103149414
 Extra: Using where; Using filesort
```



图 。在继续阅读输出分析之前，我们鼓励您自己研究它。

![](../附图/Figure%2020-12.png)

查询计划从使用国家/地区表查找按地区查找个最小国家/地区的子查询开始。子查询为表标签，因此您需要查找 id 的行（可能是其他查询的几行），在这种情况下，该行为第 3 行。第 3 行的选型设置为，因此它是派生表;这是通过查询创建的表，但其行为类似于普通表。派生表扫描（类型生成，子句应用于每行，后跟文件排序。生成的派生表被物化（从视觉解释中可见），称为。

一旦派生表构造完成，它即用作与 ci （ 城市 ） 表一表.从行中第 1 行中的行的顺序中可以看到，第 2和 ci 的顺序。对于派生表中的每一行，估计使用"国家代码"索引在检查 18。国家索引是一个非统一索引，可以从可视化解释中的表框的标签具有值。据估计，联接将返回来自派生表中的 10 行的 180 行乘以 ci 表中每个索引查找的值。

最后，使用内部临时表和文件排序对结果进行排序。查询的总成本估计为 247.32。

到目前为止，人们一直在讨论查询计划最终是什么。如果您想知道优化器是如何到达那里的，则需要检查优化器跟踪。

## 优化器跟踪

优化器跟踪并不经常需要，但有时当您遇到意外的查询计划时，了解优化器是如何到达那里会很有用的。这就是优化器跟踪显示的。

通过将"optimizer_trace 1 启用。这使得优化器记录后续查询的跟踪再次禁用），并且信息通过表。保留的最大跟踪数使用"默认"选项默认值为 1）。

您可以选择执行所需的查询优化器跟踪和使用获取查询计划。后者非常有用，因为它同时为您提供查询计划和优化器跟踪。获取查询优化器跟踪如下所示：

1. 1.

   为启用"会话"选项。

    

2. 2.

   调查的查询的 EXPLAIN。

    

3. 3.

   再次选项。

    

4. 4.

   从数据检索优化表。

    

包含四列：

- 原始查询。
- 包含跟踪信息的 JSON 文档。不久将有更多关于跟踪。
- 记录跟踪的大小（以字节为单位）限制为 8 中的默认值为 1 MiB）。此列显示记录完整跟踪所需的内存量。如果该值大于 0，则使用增加该选项。
- 是否缺少生成优化器跟踪的权限。

该表作为临时表创建，因此跟踪对会话是唯一的。

清单显示了获取查询优化器跟踪的示例（与前面各节中重复的示例查询相同）。优化器跟踪输出已被截断，因为它超过 15000 个字符和近 500 行长。同样语句的输出被省略，因为它与前面显示的相同，并且对于此讨论并不重要。完整输出包含在文件中的跟踪本身。

```
Listing 20-10. Obtaining the optimizer trace for a query
mysql> SET SESSION optimizer_trace = 1;
Query OK, 0 rows affected (0.0003 sec)
mysql> EXPLAIN
 SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5\G
...
mysql> SET SESSION optimizer_trace = 0;
Query OK, 0 rows affected (0.0002 sec)
mysql> SELECT * FROM information_schema.OPTIMIZER_TRACE\G
*************************** 1. row ***************************
 QUERY: EXPLAIN
SELECT ci.ID, ci.Name, ci.District,
 co.Name AS Country, ci.Population
 FROM world.city ci
 INNER JOIN
 (SELECT Code, Name
 FROM world.country
 WHERE Continent = 'Europe'
 ORDER BY SurfaceArea
 LIMIT 10
 ) co ON co.Code = ci.CountryCode
 ORDER BY ci.Population DESC
 LIMIT 5
 TRACE: {
...
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
 INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.0436 sec)
```



跟踪是结果中最有趣的。虽然有很多可用的信息，幸运的是，这在很大程度上是不言自明的，如果你已经熟悉了JSON格式输出，有一些相似之处。大部分信息是关于执行查询的各个部分的成本估算。如果有多个可能的选项，优化器将计算每个选项的成本并选择最便宜的选项。此跟踪的一个此类示例是用于访问 （） 表.这可以通过国家代码索引表扫描完成。此决策的跟踪部分显示在清单缩进已减小）。

```
Listing 20-11. The optimizer trace for choosing the access type for the ci table
"table": "`city` `ci`",
"best_access_path": {
 "considered_access_paths": [
 {
 "access_type": "ref",
 "index": "CountryCode",
 "rows": 18.052,
 "cost": 63.181,
 "chosen": true
 },
 {
 "rows_to_scan": 4188,
 "filtering_effect": [
 ],
 "final_filtering_effect": 1,
 "access_type": "scan",
 "using_join_cache": true,
 "buffers_needed": 1,
 "resulting_rows": 4188,
 "cost": 4194.3,
 "chosen": false
 }
 ]
},
```



这表明，在使用成本为 63.181 的国家代码索引检查超过 18 行。 对于完整表扫描预计需要检查 4188 行，总成本为 4194.3。"元素显示访问类型已被选中。

虽然很少需要深入了解优化器如何到达查询计划的详细信息，但了解优化器的工作原理可能很有用。有时，查看查询计划的其他选项的估计成本，以了解未选择它们的原因也很有用。

到目前为止，除了解释分析之外整个讨论是关于在查询执行前阶段分析查询。如果要检查实际性能，选择是解释分析。另一个选项是使用性能架构。

## 性能架构事件分析

性能架构允许您分析在检测的每个事件上花费的时间。您可以使用它来分析执行查询时花费的时间。本节将介绍如何使用性能架构分析存储过程，以查看该过程中哪个语句的用量最长，以及如何使用阶段事件分析单个查询。在本节末尾，将介绍如何使用过程创建线程完成的工作图，以及如何使用收集具有给定摘要的语句的统计信息。

### 检查存储过程

检查存储过程完成的工作可能具有挑战性，因为您不能直接在过程上使用而且过程将执行哪些查询可能并不明显。相反，您可以使用性能架构。它记录执行的每个语句，并在表中历史记录。

除非需要存储每个线程的最后十个查询，否则无需执行任何操作来开始分析。如果该过程生成十多个语句事件增加 performance_schema_events_statements_history_size 选项的值（需要重新启动）、表或使用过程，如下文所述。本讨论的剩余部分假定您可以使用

作为检查存储过程执行的查询的示例，请考虑清单。该过程在文件中也可用，它可以源到任何架构中。

```
Listing 20-12. An example procedure
CREATE SCHEMA IF NOT EXISTS chapter_20;
DELIMITER $$
CREATE PROCEDURE chapter_20.testproc()
 SQL SECURITY INVOKER
 NOT DETERMINISTIC
 MODIFIES SQL DATA
BEGIN
 DECLARE v_iter, v_id int unsigned DEFAULT 0;
 DECLARE v_name char(35) CHARSET latin1;
 SET v_id = CEIL(RAND()*4079);
 SELECT Name
 INTO v_name
 FROM world.city
 WHERE ID = v_id;
 SELECT *
 FROM world.city
 WHERE Name = v_name;
END$$
DELIMITER ;
```



该过程执行三个查询。第一个查询 1 和 4079 之间的整数表中的可用 ID 值）。第二个查询获取具有该 ID 的城市的名称。第三个查询查找所有名称与在第二个查询中发现的城市同名的城市。

如果在连接中调用此过程，则随后可以分析由该过程触发的查询以及这些查询的性能。例如：

```
mysql> SELECT PS_CURRENT_THREAD_ID();
+------------------------+
| PS_CURRENT_THREAD_ID() |
+------------------------+
| 83                     |
+------------------------+
1 row in set (0.00 sec)
mysql> CALL chapter_20.testproc();
+------+--------+-------------+----------+------------+
| ID   | Name   | CountryCode | District | Population |
+------+--------+-------------+----------+------------+
| 2853 | Jhelum | PAK         | Punjab   | 145800     |
+------+--------+-------------+----------+------------+
1 row in set (0.0019 sec)
Query OK, 0 rows affected (0.0019 sec)
```



该过程的输出是随机的，因此每次执行都会有所不同。然后，您可以使用函数（在 MySQL 8.0.15 和更早版本使用找到的线程 ID 来确定执行了哪些查询。

清单显示了如何进行此分析。您必须在不同的连接中执行此分析以使用找到的线程以使用第一个查询中的事件 ID。一些细节已从输出中删除，以关注最感兴趣的值。

```
Listing 20-13. Analyzing the queries executed by a stored procedure
mysql> SELECT *
 FROM performance_schema.events_statements_history
 WHERE THREAD_ID = 83
 AND EVENT_NAME = 'statement/sql/call_procedure'
 ORDER BY EVENT_ID DESC
 LIMIT 1\G
*************************** 1. row ***************************
 THREAD_ID: 83
 EVENT_ID: 64
 END_EVENT_ID: 72
 EVENT_NAME: statement/sql/call_procedure
 SOURCE: init_net_server_extension.cc:95
 TIMER_START: 533823963611947008
 TIMER_END: 533823965937460352
 TIMER_WAIT: 2325513344
 LOCK_TIME: 129000000
 SQL_TEXT: CALL testproc()
 DIGEST: 72fd8466a0e05fe215308832173a3be50e7edad960
408c70078ef94f8ffb52b2
 DIGEST_TEXT: CALL `testproc` ( )
...
1 row in set (0.0008 sec)
mysql> SELECT *
 FROM performance_schema.events_statements_history
 WHERE THREAD_ID = 83
 AND NESTING_EVENT_ID = 64
 ORDER BY EVENT_ID\G
*************************** 1. row ***************************
 THREAD_ID: 83
 EVENT_ID: 65
 END_EVENT_ID: 65
 EVENT_NAME: statement/sp/set
...
*************************** 2. row ***************************
 THREAD_ID: 83
 EVENT_ID: 66
 END_EVENT_ID: 66
 EVENT_NAME: statement/sp/set
...
*************************** 3. row ***************************
 THREAD_ID: 83
 EVENT_ID: 67
 END_EVENT_ID: 67
 EVENT_NAME: statement/sp/set
...
*************************** 4. row ***************************
 THREAD_ID: 83
 EVENT_ID: 68
 END_EVENT_ID: 68
 EVENT_NAME: statement/sp/set
...
*************************** 5. row ***************************
 THREAD_ID: 83
 EVENT_ID: 69
 END_EVENT_ID: 70
 EVENT_NAME: statement/sp/stmt
 SOURCE: sp_head.cc:2166
 TIMER_START: 533823963993029248
 TIMER_END: 533823964065598976
 TIMER_WAIT: 72569728
 LOCK_TIME: 0
 SQL_TEXT: SELECT Name
 INTO v_name
 FROM world.city
 WHERE ID = v_id
 DIGEST: NULL
 DIGEST_TEXT: NULL
 CURRENT_SCHEMA: db1
 OBJECT_TYPE: PROCEDURE
 OBJECT_SCHEMA: db1
 OBJECT_NAME: testproc
 OBJECT_INSTANCE_BEGIN: NULL
 MYSQL_ERRNO: 0
 RETURNED_SQLSTATE: 00000
 MESSAGE_TEXT: NULL
 ERRORS: 0
 WARNINGS: 0
 ROWS_AFFECTED: 1
 ROWS_SENT: 0
 ROWS_EXAMINED: 1
CREATED_TMP_DISK_TABLES: 0
 CREATED_TMP_TABLES: 0
 SELECT_FULL_JOIN: 0
 SELECT_FULL_RANGE_JOIN: 0
 SELECT_RANGE: 0
 SELECT_RANGE_CHECK: 0
 SELECT_SCAN: 0
 SORT_MERGE_PASSES: 0
 SORT_RANGE: 0
 SORT_ROWS: 0
 SORT_SCAN: 0
 NO_INDEX_USED: 0
 NO_GOOD_INDEX_USED: 0
 NESTING_EVENT_ID: 64
 NESTING_EVENT_TYPE: STATEMENT
 NESTING_EVENT_LEVEL: 1
 STATEMENT_ID: 25241
*************************** 6. row ***************************
 THREAD_ID: 83
 EVENT_ID: 71
 END_EVENT_ID: 72
 EVENT_NAME: statement/sp/stmt
 SOURCE: sp_head.cc:2166
 TIMER_START: 533823964067422336
 TIMER_END: 533823965880571520
 TIMER_WAIT: 1813149184
 LOCK_TIME: 0
 SQL_TEXT: SELECT *
 FROM world.city
 WHERE Name = v_name
 DIGEST: NULL
 DIGEST_TEXT: NULL
 CURRENT_SCHEMA: db1
 OBJECT_TYPE: PROCEDURE
 OBJECT_SCHEMA: db1
 OBJECT_NAME: testproc
 OBJECT_INSTANCE_BEGIN: NULL
 MYSQL_ERRNO: 0
 RETURNED_SQLSTATE: NULL
 MESSAGE_TEXT: NULL
 ERRORS: 0
 WARNINGS: 0
 ROWS_AFFECTED: 0
 ROWS_SENT: 1
 ROWS_EXAMINED: 4080
CREATED_TMP_DISK_TABLES: 0
 CREATED_TMP_TABLES: 0
 SELECT_FULL_JOIN: 0
 SELECT_FULL_RANGE_JOIN: 0
 SELECT_RANGE: 0
 SELECT_RANGE_CHECK: 0
 SELECT_SCAN: 1
 SORT_MERGE_PASSES: 0
 SORT_RANGE: 0
 SORT_ROWS: 0
 SORT_SCAN: 0
 NO_INDEX_USED: 1
 NO_GOOD_INDEX_USED: 0
 NESTING_EVENT_ID: 64
 NESTING_EVENT_TYPE: STATEMENT
 NESTING_EVENT_LEVEL: 1
 STATEMENT_ID: 25242
6 rows in set (0.0008 sec)
```



分析由两个查询组成。第一个确定通过查询语句/sql/call_procedure 事件排序）该事件是调用过程的事件。

第二个查询请求具有语句相同的线程的事件。这些是过程执行的语句。通过按按执行顺序返回语句。

第二个查询的查询结果显示，该过程从四个开始。其中一些是预料之中的，但也有一些是由隐式设置变量触发的。最后两行是本讨论最有趣的，因为它们显示执行了两个查询。首先，表由其 ID 列主键）查询。正如预期的那样，它检查一行。由于结果保存在变量中计数器递增，。

第二个查询执行得不好。它还查询城市表但按名称查询没有索引。这将导致检查 4080 行以返回单个行。列设置为 1，以反映已执行完整表扫描。

使用检查存储过程的一个缺点是，如您所见，它可以快速使用历史记录表中的所有 10 行。另一种选择是启用并在其他空闲测试系统上测试过程，或禁用其他连接的历史记录日志记录。这允许您分析执行最多 10000 个语句事件的过程。另一种选择是使用过程，该过程也使用较长历史记录，但在执行过程时支持轮询，因此它可以收集事件，即使表不够大，无法在过程持续时间内保存所有事件。

此示例一直使用语句事件来分析性能。有时，您需要知道在更细粒度级别会发生什么，在这种情况下，您需要开始查看阶段事件。

### 分析阶段事件

如果需要获取查询花费时间位置的更细粒度的详细信息，第一步是查看阶段事件。也可以包括等待事件。由于处理等待事件的步骤与阶段事件的步骤基本相同，因此留给读者分析查询的等待事件是一个练习。

生成的阶段事件数比语句事件的数量大得多。这意味着，为了避免阶段事件从历史记录表中消失，建议对空闲测试系统执行分析，。默认情况下，此表处于禁用状态;默认情况下，此表已禁用。要启用它，请启用相应的使用者：

```
mysql> UPDATE performance_schema.setup_consumers
 SET ENABLED = 'YES'
 WHERE NAME IN ('events_stages_current',
 'events_stages_history_long');
Query OK, 2 rows affected (0.0008 sec)
Rows matched: 2 Changed: 2 Warnings: 0
```



使用者取决于因此您需要同时同时启用这两个使用者。默认情况下，仅启用与进度信息相关的阶段事件。对于一般分析，您需要启用所有阶段事件：

```
mysql> UPDATE performance_schema.setup_instruments
 SET ENABLED = 'YES',
 TIMED = 'YES'
 WHERE NAME LIKE 'stage/%';
Query OK, 125 rows affected (0.0011 sec)
Rows matched: 125 Changed: 109 Warnings: 0
```



此时，分析可以像分析存储过程时大致相同的方式进行。例如，请考虑以下查询由与性能架构线程 ID 等于 83 的连接执行：

```
SELECT *
 FROM world.city
 WHERE Name = 'Sydney';
```



假设这是最后一个执行的查询，您可以获取每个阶段所花费的时间量，如清单。您需要执行这是一个单独的连接，并更改来使用连接的线程 ID。除了时间明显不同，那么查询经历的阶段列表也可能不同。

```
Listing 20-14. Finding the stages for the last statement of a connection
mysql> SET @thread_id = 83;
Query OK, 0 rows affected (0.0004 sec)
mysql> SELECT EVENT_ID,
 SUBSTRING_INDEX(EVENT_NAME, '/', -1) AS Event,
 FORMAT_PICO_TIME(TIMER_WAIT) AS Latency
 FROM performance_schema.events_stages_history_long
 WHERE THREAD_ID = @thread_id
 AND NESTING_EVENT_ID = (
 SELECT EVENT_ID
 FROM performance_schema.events_statements_history
 WHERE THREAD_ID = @thread_id
 ORDER BY EVENT_ID DESC
 LIMIT 1);
+----------+--------------------------------------+-----------+
| EVENT_ID | Event                                | Latency   |
+----------+--------------------------------------+-----------+
| 7193     | Executing hook on transaction begin. | 200.00 ns |
| 7194     | cleaning up                          | 4.10 us   |
| 7195     | checking permissions                 | 2.60 us   |
| 7196     | Opening tables                       | 41.50 us  |
| 7197     | init                                 | 3.10 us   |
| 7198     | System lock                          | 6.50 us   |
| 7200     | optimizing                           | 5.30 us   |
| 7201     | statistics                           | 15.00 us  |
| 7202     | preparing                            | 12.10 us  |
| 7203     | executing                            | 1.18 ms   |
| 7204     | end                                  | 800.00 ns |
| 7205     | query end                            | 500.00 ns |
| 7206     | waiting for handler commit           | 6.70 us   |
| 7207     | closing tables                       | 3.30 us   |
| 7208     | freeing items                        | 70.30 us  |
| 7209     | cleaning up                          | 300.00 ns |
+----------+--------------------------------------+-----------+
16 rows in set (0.0044 sec)
```



事件 ID、阶段名称（为了简洁起见删除完整事件名称的两个前部分函数（使用 MySQL sys.format_time 8.0.15 及更早版本格式的。WHERE 子句在执行查询的连接的线程 ID 和嵌套事件 ID 上进行筛选。嵌套事件 ID 设置为线程 ID 等于 83 的连接的最新执行语句的事件 ID。结果显示，查询最慢的部分是发送，这是存储引擎查找和发送行的阶段。

这样分析查询的主要问题是，您要么受到默认情况下保存的每个线程的 10 个事件限制，要么在检查完之前，可能会从长历史记录表中清除事件。创建过程是为了帮助解决此问题。

### 使用sys.ps_trace_thread() 程序进行分析

当您需要分析执行多个的复杂查询或存储程序时，可以使用在执行过程中自动收集信息的工具。从 sys 架构中执行的选项是过程。

该过程循环一段时间，轮询新事务、语句、阶段和等待事件的长历史记录表。可选地，该过程还可以设置性能架构以包括所有事件，使使用者能够记录事件。但是，由于包含所有事件通常太多，因此建议自己设置性能架构来检测和使用分析感兴趣的事件。

另一个可选功能是在监视开始时重置性能架构表。如果可以删除长历史记录表的内容，这可大。

调用该过程时，必须提供以下：

- 要监视的性能架构线程 ID。
- 要将结果写入的文件。结果使用点图描述语言创建。这要求选项已设置为允许将文件写入目标目录，并且文件不存在，并且执行该过程的用户具有权限。
- 以秒为单位监视的最大时间。支持使用 1/100 秒精度指定该值。如果该值设置为则运行时设置为 60 秒。
- 轮询历史记录表之间的间隔。可以以 1/100 秒的精度设置该值。如果该值设置为则轮询间隔将设置为一秒。
- 布尔是否重置用于分析的性能架构表。
- 布尔是否启用所有工具和消费者，可由程序使用。启用后，当过程完成后，将还原当前设置。
- 布尔是否包含其他，例如在源代码中触发事件的地点。这在包括等待事件时非常有用。

在清单中可以看到使用过程的示例。执行该过程时，的线程调用来自早期测试proc（） 过程。该示例假定您从默认的性能架构设置开始。

```
Listing 20-15. Using the ps_trace_thread() procedure
Connection 1> UPDATE performance_schema.setup_consumers
 SET ENABLED = 'YES'
 WHERE NAME = 'events_statements_history_long';
Query OK, 1 row affected (0.0074 sec)
Rows matched: 1 Changed: 1 Warnings: 0
-- Find the Performance Schema thread id for the
-- thread that will be monitored.
Connection 2> SELECT PS_CURRENT_THREAD_ID();
+-----------------+
| PS_THREAD_ID(9) |
+-----------------+
| 32              |
+-----------------+
1 row in set (0.0016 sec)
-- Replace the first argument with the thread id
-- just found.
--
-- Once the procedure returns
-- "Data collection starting for THREAD_ID = 32"
-- (replace 32 with your thread id) invoke the
-- chapter_20.testproc() chapter from connection 2.
-- The example is set to poll for 10 seconds. If you
-- need more time, change the third argument to the
-- number of seconds you need.
Connection 1> CALL sys.ps_trace_thread(
 32,
 '/mysql/files/thread_32.gv',
 10, 0.1, False, False, False);
+-------------------+
| summary           |
+-------------------+
| Disabled 1 thread |
+-------------------+
1 row in set (0.0316 sec)
+---------------------------------------------+
| summary                                     |
+---------------------------------------------+
| Data collection starting for THREAD_ID = 32 |
+---------------------------------------------+
1 row in set (0.0316 sec)
-- Here, sys.ps_trace_id() blocks – execute the
-- query you want to trace. The output is random.
Connection 2> CALL chapter_20.testproc();
+------+--------+-------------+----------+------------+
| ID | Name | CountryCode | District | Population |
+------+--------+-------------+----------+------------+
| 3607 | Rjazan | RUS | Rjazan | 529900 |
+------+--------+-------------+----------+------------+
1 row in set (0.0023 sec)
Query OK, 0 rows affected (0.0023 sec)
-- Back in connection 1, wait for the sys.ps_trace_id()
-- procedure to complete.
+--------------------------------------------------+
| summary                                          |
+--------------------------------------------------+
| Stack trace written to /mysql/files/thread_32.gv |
+--------------------------------------------------+
1 row in set (0.0316 sec)
+----------------------------------------------------------+
| summary                                                  |
+----------------------------------------------------------+
| dot -Tpdf -o /tmp/stack_32.pdf /mysql/files/thread_32.gv |
+----------------------------------------------------------+
1 row in set (0.0316 sec)
+----------------------------------------------------------+
| summary                                                  |
+----------------------------------------------------------+
| dot -Tpng -o /tmp/stack_32.png /mysql/files/thread_32.gv |
+----------------------------------------------------------+
1 row in set (0.0316 sec)
+------------------+
| summary          |
+------------------+
| Enabled 1 thread |
+------------------+
1 row in set (0.0316 sec)
Query OK, 0 rows affected (0.0316 sec)
```



在此示例中，仅启用使用者。这将允许记录调用就像之前手动完成一样。将监视的线程 ID 使用MySQL 8.0.15 及更早版本

为过程，输出写入。该过程每 0.1 秒轮询 10 秒，并且禁用所有可选功能。

您将需要一个程序，了解点格式，以将其转换为图像。一个选项是 ，它可通过包存储库从多个 Linux 发行版获得。也可以从项目的主页微软Windows，macOS，Solaris和自由BSD下载。该过程的输出显示了如何将具有点图定义的文件转换为 PDF 或 PNG 文件的示例。图显示了 CALL 图形。

![](../附图/Figure%2020-13.png)

语句图包含与手动分析过程时相同的信息。对于像生成图形的优势是有限的，但对于更复杂的过程或对于启用较低级别的事件的分析查询，它可以是可视化执行流的好方法。

另可以帮助您分析查询的 sys 架构过程是过程。

### 使用ps_trace_statement_digest() 程序进行分析

作为使用性能架构分析查询一个示例，将演示架构中的过程。它需要一个摘要，然后监视与该摘要的语句相关的事件。分析结果包括汇总数据以及详细信息，如运行时间最长的查询的查询计划。

该过程采用五个参数，这些参数都是必填项的。为

- 要监视的摘要。如果语句的摘要与提供的架构的摘要匹配，则将监视语句，而不考虑默认架构。
- 以秒为单位的监视时间。不允许使用小数。
- 轮询历史记录表之间的间隔。该值可以设置为精度为 1/100 秒，并且必须小于 1 秒。
- 布尔是否重置用于分析的性能架构表。
- 布尔是否启用所有工具和消费者，可由程序使用。启用后，当，将还原当前设置。

例如，您可以开始使用过程进行监视，并在监视进行期间执行以下查询（监视示例如下）：

```
SELECT * FROM world.city WHERE CountryCode = 'AUS';
SELECT * FROM world.city WHERE CountryCode = 'USA';
SELECT * FROM world.city WHERE CountryCode = 'CHN';
SELECT * FROM world.city WHERE CountryCode = 'ZAF';
SELECT * FROM world.city WHERE CountryCode = 'BRA';
SELECT * FROM world.city WHERE CountryCode = 'GBR';
SELECT * FROM world.city WHERE CountryCode = 'FRA';
SELECT * FROM world.city WHERE CountryCode = 'IND';
SELECT * FROM world.city WHERE CountryCode = 'DEU';
SELECT * FROM world.city WHERE CountryCode = 'SWE';
SELECT * FROM world.city WHERE CountryCode = 'LUX';
SELECT * FROM world.city WHERE CountryCode = 'NZL';
SELECT * FROM world.city WHERE CountryCode = 'KOR';
```



从执行到执行，这些查询中哪个查询最慢。

清单显示了使用过程监视选择特定国家/地区所有城市的查询的示例。在该示例中，使用函数找到摘要，但您可能也通过基于"events_statements_summary_by_digest表监视。它将被留给过程，以启用所需的工具和使用者，并将重置受监视的表，以避免包括监视开始前执行的语句的发生。轮询频率设置为每 0.5 秒轮询一次。为了减小输出的宽度，舞台事件名称已删除前缀，并且 EXPLAIN 输出线已变短。在本书的 GitHub 存储库中文件中可以找到未修改的输出。

```
Listing 20-16. Using the ps_trace_statement_digest() procedure
mysql> SET @digest = STATEMENT_DIGEST('SELECT * FROM world.city WHERE
CountryCode = ''AUS''');
Query OK, 0 rows affected (0.0004 sec)
-- Execute your queries once the procedure has started.
mysql> CALL sys.ps_trace_statement_digest(@digest, 60, 0.5, TRUE, TRUE);

+-------------------+
| summary |
+-------------------+
| Disabled 1 thread |
+-------------------+
1 row in set (1 min 0.0861 sec)
+--------------------+
| SUMMARY STATISTICS |
+--------------------+
| SUMMARY STATISTICS |
+--------------------+
1 row in set (1 min 0.0861 sec)
+------------+-----------+-----------+-----------+---------------+---------
------+------------+------------+
| executions | exec_time | lock_time | rows_sent | rows_affected | rows_
examined | tmp_tables | full_scans |
+------------+-----------+-----------+-----------+---------------+---------
------+------------+------------+
| 13 | 7.29 ms | 1.19 ms | 1720 | 0 |
1720 | 0 | 0 |
+------------+-----------+-----------+-----------+---------------+---------
------+------------+------------+
1 row in set (1 min 0.0861 sec)
+--------------------------------------+-------+-----------+
| event_name | count | latency |
+--------------------------------------+-------+-----------+
| Sending data | 13 | 2.99 ms |
| freeing items | 13 | 2.02 ms |
| statistics | 13 | 675.37 us |
| Opening tables | 13 | 401.50 us |
| preparing | 13 | 100.28 us |
| optimizing | 13 | 66.37 us |
| waiting for handler commit | 13 | 64.18 us |
| closing tables | 13 | 54.70 us |
| System lock | 13 | 54.34 us |
| cleaning up | 26 | 45.22 us |
| init | 13 | 29.54 us |
| checking permissions | 13 | 23.34 us |
| end | 13 | 10.21 us |
| query end | 13 | 8.02 us |
| executing | 13 | 4.01 us |
| Executing hook on transaction begin. | 13 | 3.65 us |
+--------------------------------------+-------+-----------+
16 rows in set (1 min 0.0861 sec)
+---------------------------+
| LONGEST RUNNING STATEMENT |
+---------------------------+
| LONGEST RUNNING STATEMENT |
+---------------------------+
1 row in set (1 min 0.0861 sec)
+-----------+-----------+-----------+-----------+---------------+----------
-----+------------+-----------+
| thread_id | exec_time | lock_time | rows_sent | rows_affected | rows_
examined | tmp_tables | full_scan |
+-----------+-----------+-----------+-----------+---------------+----------
-----+------------+-----------+
| 32 | 1.09 ms | 79.00 us | 274 | 0
| 274 | 0 | 0 |
+-----------+-----------+-----------+-----------+---------------+----------
-----+------------+-----------+
1 row in set (1 min 0.0861 sec)
+----------------------------------------------------+
| sql_text |
+----------------------------------------------------+
| SELECT * FROM world.city WHERE CountryCode = 'USA' |
+----------------------------------------------------+
1 row in set (59.91 sec)
+--------------------------------------+-----------+
| event_name | latency |
+--------------------------------------+-----------+
| Executing hook on transaction begin. | 364.67 ns |
| cleaning up | 3.28 us |
| checking permissions | 1.46 us |
| Opening tables | 27.72 us |
| init | 2.19 us |
| System lock | 4.01 us |
| optimizing | 5.11 us |
| statistics | 46.68 us |
| preparing | 7.66 us |
| executing | 364.67 ns |
| Sending data | 528.41 us |
| end | 729.34 ns |
| query end | 729.34 ns |
| waiting for handler commit | 4.38 us |
| closing tables | 16.77 us |
| freeing items | 391.29 us |
| cleaning up | 364.67 ns |
+--------------------------------------+-----------+
17 rows in set (1 min 0.0861 sec)
+--------------------------------------------------+
| EXPLAIN                                          |
+--------------------------------------------------+
| {
 "query_block": {
 "select_id": 1,
 "cost_info": {
 "query_cost": "46.15"
 },
 "table": {
 "table_name": "city",
 "access_type": "ref",
 "possible_keys": [
 "CountryCode"
 ],
 "key": "CountryCode",
 "used_key_parts": [
 "CountryCode"
 ],
 "key_length": "3",
 "ref": [
 "const"
 ],
 "rows_examined_per_scan": 274,
 "rows_produced_per_join": 274,
 "filtered": "100.00",
 "cost_info": {
 "read_cost": "18.75",
 "eval_cost": "27.40",
 "prefix_cost": "46.15",
 "data_read_per_join": "19K"
 },
 "used_columns": [
 "ID",
 "Name",
 "CountryCode",
 "District",
 "Population"
 ]
 }
 }
} |
+--------------------------------------------------+
1 row in set (1 min 0.0861 sec)
+------------------+
| summary          |
+------------------+
| Enabled 1 thread |
+------------------+
1 row in set (1 min 0.0861 sec)
Query OK, 0 rows affected (1 min 0.0861 sec)
```



输出从分析期间找到的所有查询的摘要开始。总共使用 7.29 毫秒检测到 13 次执行。总体摘要还包括各个阶段所花费的时间的聚合。输出的下一部分是 13 个执行中最慢的详细信息。输出以 JSON 格式的查询计划结束，该查询计划最慢。

生成查询计划时应注意一个限制。语句将执行，默认架构设置为与执行过程位置相同。这意味着，如果查询在不同的架构中执行，并且它不使用完全限定的表名（即包括架构名称），语句将失败，并且该过程不会输出查询计划。

## 总结

本章介绍了如何分析您认为可能需要优化的查询。本章的大部分内容重点介绍，它是分析查询的主要工具。本章的其余部分包括优化器跟踪以及如何使用性能架构分析查询。

EXPLAIN 语句支持几种不同的格式，可帮助您以最适合您的格式获取查询计划。传统格式使用标准表输出，JSON 格式返回详细的 JSON 文档，树格式显示相对简单的执行树。树格式仅在 MySQL 8.0.16 及更晚时受支持，并且要求使用 Volcano 执行器执行器执行查询。JSON 格式是 MySQL 工作台中的可视化解释功能用于创建查询计划的图表。

EXPLAIN 输出中有大量有关查询计划的信息。讨论了传统格式的字段以及 JSON 最常遇到的字段。这包括详细讨论选择类型和访问类型以及额外信息。最后，使用了一系列示例来演示如何使用此信息。

优化器跟踪可用于获取有关优化器如何结束 EXPLAIN 语句返回的信息。通常不需要为最终用户使用优化器跟踪，但它们对于了解有关优化器以及导致查询计划的决策过程的信息非常有用。

本章的最后一部分展示了如何使用性能架构事件来确定语句所花的时间。首先展示了如何将存储过程分解为单个语句，然后以及如何将语句分解为阶段。最后过程用于自动分析并创建事件过程用于收集给定语句摘要的统计信息。

本章分析了查询。有时有必要考虑整个事务。下一章将介绍如何分析交易记录。
