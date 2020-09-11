# 索引统计

在上一章中，您了解了索引。有人提到，优化器会评估每个索引，以决定是否使用索引。它是如何做到这一点？这在很大程度上是本章的主题，其中介绍了索引统计信息、如何查看有关索引统计信息的信息以及如何维护统计信息。

本章首先讨论了什么是索引统计信息以及 InnoDB 如何处理索引统计信息。然后，您将了解瞬态和持久性统计信息。本章的其余部分介绍如何监视统计信息并更新统计信息。

## 什么是索引统计？

当 MySQL 决定是否使用索引时，归结起来就是 MySQL 认为索引对查询的成效如何。请记住，当您使用辅助索引时，实际上将有一个额外的主键查找来访问数据。辅助索引的排序方式与行的方式不一样，因此使用索引通常意味着随机 I/O（这可以帮助使用覆盖索引）。另一方面，表扫描是更大的顺序 I/O。因此，行对行，执行表扫描比使用辅助索引查找同一行便宜。

这意味着，索引要有效，必须筛选出表的很大一部分。必须筛选出多少取决于硬件的性能特征、缓冲池中的表数量、表定义等。在旧旋转磁盘时代，经验法则是，如果需要超过 30% 的行，则首选表扫描。内存中的行数越高，磁盘的随机 I/O 性能越好，此阈值越高。

------

**注意** 覆盖索引会更改此图片，因为它们减少了从跳到实际行数据所需的随机 I/O 量。

------

这就是索引统计信息的图。优化器（是决定使用哪个查询计划的 MySQL 部分）需要一些简单的方法来确定索引对于给定查询计划的作用。优化器显然知道索引包括哪些列，但此外，它需要一些度量索引筛选行的程度。此信息是索引统计信息提供的信息。因此，索引统计是衡量索引选择性的指标。有两个主要统计信息：唯一值的数量和一定范围内的值数。

在讨论索引统计信息时，唯一值的数量是最常见的问题。这称为索引的基数。基数越高，唯一值越高。对于不允许 NULL 值的主要键和其他唯一索引，基数是表中的行数，因为所有值都必须是唯一的。

优化器在查询查询的基础上请求给定范围内的行数。这适用于范围条件，如 WHERE val = 5 以及 IN（） 条件或一系列 OR 条件。MySQL 8 支持的直方图 是为单个查询临时收集此信息的一个例外。下一章将讨论直方图。

简而言之，索引统计信息是有关索引中数据分布的近似信息。在 MySQL 中，存储引擎负责提供索引统计信息。因此，值得进一看看 InnoDB 如何处理索引统计信息。

## InnoDB and Index Statistics

它是向服务器层和优化器提供索引统计信息的存储引擎。因此，了解 InnoDB 如何确定其统计信息非常重要。InnoDB 支持两种存储统计信息的方法：持久和瞬态。无论哪种方式，统计的确定方式都是一样的。本节首先讨论如何收集统计数据，然后介绍持久性和瞬态性统计信息的具体细节。

### How Statistics Are Collected

InnoDB 通过分析索引的随机叶页来计算其索引统计信息。例如，可能是对 20 个随机索引页进行了采样（也称为 20 个索引潜水），并检查这些页面包含的索引值。然后，InnoDB 会根据索引的总大小进行缩放。

这一个重要含义是 InnoDB 索引统计信息并不精确。当您看到给定的查询条件意味着将读取 100 行时，它仅是基于分析的样本的估计值。这甚至包括主键和其他唯一索引以及在"索引"中报告的information_schema。表视图。表中的行的估计数与主键的估计基数相同。

另一个考虑是如何处理 NULL 值，因为 NULL 具有不等于 NULL 的属性。因此，在收集统计信息时，应将所有 NULL 值分组到一个存储桶中，还是将它们视为单独的值？最佳解决方案取决于您的查询。将所有 NULL 值视为不同的值会增加索引的基数，尤其是在索引列中有许多具有 NULL 的行时。这对于查找非 NULL 值的查询是好事。另一方面，如果将所有 NULL 视为相同，则会降低基数，这对于包含 NULL 的查询来说很有意义。您可以使用"无名小数"选项选择 InnoDB innodb_stats_method值。它可以采取三个值之一：

- nulls_equal：在这种情况下，所有NULL值都视为相同。 这是默认值。 如果不确定要选择哪个值，请选择nulls_equal。
- nulls_unequal：在这种情况下，NULL值被认为是不同的值。
- nulls_ignored：在这种情况下，收集统计信息时将忽略NULL值。

为什么使用估计值而不是精确统计（表示完整的索引扫描）？原因是性能。对于大型索引，执行完整的索引扫描需要很长时间。一般来说，它还包括磁盘 I/O，这使得性能问题更加严重。为了避免计算索引统计信息对查询性能有负面影响，已选择将扫描限制为相对较少的页数。

### Sample Pages

使用近似统计的缺点是，它们并不总是值的实际分布的一个很好的表示形式。发生这种情况时，优化器可能会选择错误的索引或错误的联接顺序，导致速度低于必要的查询。但是，也可以调整随机指数潜水的数量。如何做到这一点取决于是否使用持久统计或瞬态统计信息：

- 持久统计信息使用innodb_stats_persistent_sample_页选项作为要采样的默认页数。表STATS_SAMPLE_PAGES可用于指定给定表的页数。
- 瞬态统计信息使用所有表的innodb_ stats_transient_sample_pages页数。

关于持久性和瞬态性统计的两个小节详细介绍了处理这两种处理索引统计信息的方法的细节。

将示例页数设置为给定值是什么意思？这取决于索引中的列数。如果只有一列，则该值实际上意味着对叶页数进行采样。但是，对于多列索引，页数是每列。例如，如果将示例页数设置为 20，并且索引中包含四列，则总共采样 4*20=80 页。

------

**注意** 在实践中，索引统计信息采样比本章中描述的要复杂得多。例如，并不总是需要一直下降到叶页。考虑两个相邻的非叶节点何时具有相同的值。然后可以得出结论，最左侧（根据顺序）零件的所有叶页具有相同的值。如果您有兴趣了解更多，一个好的起点是源代码中存储/innobase/dict/dict0stats.cc 文件顶部的注释：https://github.com/mysql/mysql-server/blob/8.0/ 存储/innobase/dict/dict0stats.cc。

------

必须检查多少页才能得到良好的估计？这取决于表。如果数据是统一的，也就是说，每个索引值的行数大致相同，则只需要检查相对较少的页数，并且默认的页数通常就足够了。另一方面，如果您的数据分布非常不规则，则可能需要增加采样页数。非常不规则的数据示例是队列中任务的状态。随着时间的推移，大多数任务将状态为已完成。在最坏的情况下，您可能会体验到所有随机潜水都看到相同的状态，使 InnoDB 得出结论，只有一个值，并且索引作为筛选器毫无价值。

------

**提示** 对于只有几行且值用于筛选的数据，下一章中讨论的直方图对于改进查询计划非常有用。

------

表大小也是需要考虑的一个因素。表格越大，一般必须检查的页面越大，才能得到良好的估计。原因是表越大，整个叶页就越有可能指向索引值相同的行。这会降低每个采样页的值，因此为了进行补偿，需要采样更多页面。

特殊情况是，InnoDB 已配置为进行比叶页更多的索引潜水。在这种情况下，InnoDB 将检查所有叶页，并在该点停止。这将提供尽可能准确的统计数据。如果在分析期间没有活动事务，则该时间点的统计数据将精确。这包括表中的页数。在本章的稍后部分，您将学习如何在索引和表中查找表的叶页数

实际上，不可能使用精确的值。InnoDB 支持多版本控制，以便允许事务高并发，即使它们涉及写入。由于每个事务都有自己的数据视图，因此确切的统计信息将意味着每个事务都有自己的索引统计信息。这是不可行的， 那么 Innodb 如何处理呢？这是下一件事要考虑。

### Transaction Isolation Level

一个相关的问题是，在收集统计信息时使用什么事务隔离级别。InnoDB 支持四个隔离级别：读取未提交、读取已提交、可重复读取（默认值）和可序列化。收集索引统计信息时，已选择使用未提交的读取。这是有道理的，因为这是一个很好的假设，大多数事务最终被提交，或者如果他们失败，他们重试。统计信息用于将来的查询，因此没有理由在收集统计信息时添加维护读取视图的开销

但是，这确实对对表进行大更改的事务有影响。对于极端（但不太可能）的情况，请考虑缓存表，其中数据由由两个步骤组成的事务刷新：

- 1.从表中删除所有现有数据。
- 2.使用更新的数据重建表。

默认情况下，当表的"大部分"已更改时，索引统计信息会更新。（本章稍后的"持久指数统计"和"瞬态指数统计"部分将介绍构成"很大一部分"的"大部分"。这意味着，当步骤 1 完成时，InnoDB 将重新计算统计信息。这很容易 - 表是空的，所以没有。如果查询仅执行该点，则优化器将表视为空。但是，除非查询在读取未提交的事务隔离级别中执行，否则查询仍将读取所有旧行，并且查询计划可能导致查询执行效率低下。

对于刚刚讨论的问题，您需要持久统计，因为有更好的配置选项来处理特殊情况。在讨论持久统计的详细信息之前，值得学习如何在持久统计和瞬态统计之间进行选择。



### Configuring Statistics Type

如前所述，InnoDB 有两种方法可以存储索引统计信息。它可以使用持久存储，也可以使用瞬态存储。您可以使用"未执行"选项为innodb_stats_persistent方法。当设置为 1 或 ON（默认值）时，将使用持久统计信息;然后使用持久统计信息。如果将持久统计信息设置为 1 或 ON（默认值），则使用持久统计信息。将其设置为 0 或 OFF 会将该方法更改为瞬态统计信息。还可以使用"持久"表选项为每个STATS_方法。例如，要启用 world.city 表的持久统计信息，可以使用 ALTER 表，例如

```sql
ALTER TABLE world.city
 STATS_PERSISTENT = 1;
```

使用STATS_PERSISTENT表语句创建新表时，还可以设置"创建"选项。对于STATS_PERSISTENT 0 和 1 可用作值。

自引入持久索引统计信息以来，持久索引统计信息一直是默认的，并且也是推荐的选择，除非您遇到测试显示瞬态统计信息可以解决的问题。持久性统计和瞬态统计之间存在一些差异，这些差异非常重要。接下来将讨论这些差异。



## Persistent Index Statistics

MySQL 5.6 中引入了持久索引统计信息，以使查询计划比较旧的瞬态索引统计信息更稳定。如名称建议，启用持久索引统计信息后，将保存统计信息，以便当 MySQL 重新启动时不会丢失统计信息。与单单坚持性更多的差异，尽管将变得清晰。

除了稳定的查询计划，持久统计允许详细配置要采样的页面数并具有良好的监视，您甚至可以直接查询保存统计信息的表。由于监视与瞬态统计信息有较大的重叠，因此将推迟到本章的稍后部分，因此本节将重点介绍持久性统计信息和存储统计信息的表的配置。

### Configuration

可以配置持久统计信息，在收集统计信息的成本和统计的准确性之间实现良好的平衡。与瞬态统计信息不同，可以配置全局级别和每个表的行为。未设置特定于表的选项时，全局配置用作默认值。

有三个全局选项特定于持久统计信息。这些是

- **innodb_stats_persistent_sample_pages**：要采样的页数。页面数越高，统计数据越准确，成本也越高。如果该值大于索引的叶页数，则对整个索引进行采样。默认值为 20。
- **innodb_stats_auto_recalc**：是否在表中超过 10% 的行已更改时自动更新统计信息。默认值已启用（打开）。
- **innodb_stats_include_delete_marked**：是否包括标记为已删除但尚未在统计信息中已提交的行。此选项将稍后讨论。默认值处于禁用状态（关闭）。

还可以innodb_stats_persistent_sample_pagesinnodb_stats_auto_recalc设置"表"和"表"选项。这允许您根据与特定表关联的大小、数据分布和工作负载微调需求。虽然不建议进行微观管理，但它可用于处理前面讨论的缓存表方案以及其他常规默认值无法涵盖的表等情况。

建议尝试为用户找到一个很好的折衷innodb_stats_ persistent_sample_pages，从而提供足够好的统计信息，以便优化器可以确定最佳查询计划，同时避免过度扫描来计算统计信息。如果您发现查询性能很差，因为不准确的索引统计信息导致优化器选择低效计划，则需要增加采样页数。另一方面，如果 ANALYZE TABLE 时间过长，可以考虑减少采样页数。然后，可以使用描述的特定于表的选项来减少或增加特定表的采样页数。

对于大多数表，建议启用innodb_stats_auto_recalc。这将有助于确保统计数据不会由于大量更改而过时。自动重新计算在后台进行，因此它不会延迟对触发更新的应用程序的响应。当超过 10% 的表已更改时，该表将排队等待索引统计信息更新。为了避免不断重新计算小表的统计信息，还要求每个索引统计信息更新之间必须至少有 10 秒。

当然，有些例外不需要自动重新计算统计信息，例如，如果您有一个缓存表来使报告查询执行得更快，并且缓存表中的数据会不时完全重新创建，但否则不会更改。在这种情况下，禁用统计信息的自动重新计算并在重建完成时显式重新计算统计信息可能是一种优势。另一个选项是在统计信息中包括删除标记的行。

请记住，索引统计信息是使用读取未提交的事务隔离级别计算的。虽然在大多数情况下，这是提供最佳统计信息的，但有一个例外。当事务暂时完全更改数据的分布时，可能会导致不正确的统计信息。对表进行完全重建是最极端的情况，也是人们最经常看到的问题。就是在引入这样的innodb_stats_ include_delete_marked选择。InnoDB 仍将将它们包括在统计信息中，而不是将未提交的已删除行视为已删除。该选项仅作为全局选项存在，因此，即使您只有一个表存在此问题，它也会影响所有表。如前所述，另一种选择是禁用受影响表的统计信息自动重新计算，并自己处理。

------

**提示** 如果事务对表进行了大的更改（如删除所有行，然后重新生成表），请考虑禁用表的索引统计信息自动重新计算或启用innodb_stats_include_delete_marked。

------

迄今为止，只提到全球选择。如何更改表的索引统计信息设置？由于可以使用 STATS_PERSISTENT 表选项覆盖表的 innodb_stats_persistent 的全局值，因此有一些选项来控制表的持久统计信息的行为方式。表选项是

- **STATS_AUTO_RECALC**：覆盖是否
    该表已启用索引统计信息的自动重新计算。
- **STATS_SAMPLE_PAGES**：覆盖为该表采样的页面数。

可以使用创建表时设置这些选项，也可以在以后使用 ALTER 表设置这些选项，如清单 15-1 所示。

```sql
Listing 15-1. Setting the persistent statistics options for a table
mysql> CREATE SCHEMA IF NOT EXISTS chapter_15;
Query OK, 1 row affected (0.4209 sec)
mysql> use chapter_15
Default schema set to `chapter_15`.
Fetching table and column names from `chapter_15` for auto-completion...
Press ^C to stop.
mysql> CREATE TABLE city (
 City_ID int unsigned NOT NULL auto_increment,
 City_Name varchar(40) NOT NULL,
 State_ID int unsigned DEFAULT NULL,
 Country_ID int unsigned NOT NULL,
 PRIMARY KEY (City_ID),
 INDEX (City_Name, State_ID, City_ID)
 ) STATS_AUTO_RECALC = 0,
 STATS_SAMPLE_PAGES = 10;
Query OK, 0 rows affected (0.0637 sec)

mysql> ALTER TABLE city
 STATS_AUTO_RECALC = 1,
 STATS_SAMPLE_PAGES = 20;
Query OK, 0 rows affected (0.0280 sec)
Records: 0 Duplicates: 0 Warnings: 0
```

首先，创建表城市时禁用自动重新计算并创建 10 个示例页。然后更改设置以启用自动重新计算，并增加示例页数至 20。请注意 ALTER 表如何返回受影响的 0 行。更改持久统计选项只会更改表的元数据，因此它们会立即发生，并且不会影响数据。这意味着您可以根据需要更改设置，而不必担心执行昂贵的操作。例如，您可能希望在批量操作期间禁用自动重新计算。

有机会调整索引统计信息，因此能够查看收集的数据非常重要。在讨论瞬态统计之后，"监视"部分将讨论一些一般方法。但是，使持久性统计信息持久性的是，它们存储在表中，并且这些统计信息也提供了有价值的信息。

### Index Statistics Tables

InnoDB 在 mysql 架构中使用两个表来存储与持久统计信息相关的数据。这些不仅可用于调查统计数据和采样数据，而且有助于了解有关一般索引的更多信息。

通常有用的表是表innodb_index_stats。此表每个 B 树索引包含几行，提供有关索引每个部分的唯一值数（基数）、索引中的叶页数和索引的总大小的信息。表 15-1 汇总了表中的列。

主键由列、database_nametable_nameindex_name和stat_name。数据库、表和索引名称定义统计信息的索引。"last_update列可用于查看统计信息上次更新以来的过去多长时间。stat_namestat_value是为您提供实际统计数据的。sample_size是为确定统计信息而检查的叶页数。这将是索引中的叶页数和为表设置的示例页数的较小。最后，stat_description提供了有关统计的一些更多信息。对于基数，说明显示索引中包括哪些列，每列将有一行（不久将提供一个示例）。

如前所述，在该表中包括innodb_index_stats统计数据。该名称可以具有以下值之一：

- **n_diff_pfxNN**：索引中第一个 NN 列的基数。NN 是基于 1 的，因此对于具有两列的索引，n_diff_pfx01和n_diff_pfx02。对于具有这些统计信息的stat_，则描述包括统计信息中包含的列。

- **n_leaf_pages**：索引中的叶页总数。您可以将此值与数据库统计信息的n_diff_pfxNN进行比较，以确定已采样的索引的分数。

- **size**：索引中的总页数。这包括非叶页。

查看示例以更好地了解此数据表示的项数可能很有用。world.city 表有两个索引：ID 列上的主要键和"国家代码"列上的"国家代码"索引。清单15-2显示了两个索引的统计信息。请注意，如果执行同一查询，统计信息值可能不同，并且如果第 14 章中添加了额外的索引，则将有更多的行

```
Listing 15-2. The innodb_index_stats table for the world.city table
mysql> SELECT index_name, stat_name,
 stat_value, sample_size,
 stat_description
 FROM mysql.innodb_index_stats
 WHERE database_name = 'world'
 AND table_name = 'city'\G
*************************** 1. row ***************************
 index_name: CountryCode
 stat_name: n_diff_pfx01
 stat_value: 232
 sample_size: 7
stat_description: CountryCode
*************************** 2. row ***************************
 index_name: CountryCode
 stat_name: n_diff_pfx02
 stat_value: 4079
 sample_size: 7
stat_description: CountryCode,ID
*************************** 3. row ***************************
 index_name: CountryCode
 stat_name: n_leaf_pages
 stat_value: 7
 sample_size: NULL
stat_description: Number of leaf pages in the index
*************************** 4. row ***************************
 index_name: CountryCode
 stat_name: size
 stat_value: 8
 sample_size: NULL
stat_description: Number of pages in the index
*************************** 5. row ***************************
 index_name: PRIMARY
 stat_name: n_diff_pfx01
 stat_value: 4188
 sample_size: 20
stat_description: ID
*************************** 6. row ***************************
 index_name: PRIMARY
 stat_name: n_leaf_pages
 stat_value: 24
 sample_size: NULL
stat_description: Number of leaf pages in the index
*************************** 7. row ***************************
 index_name: PRIMARY
 stat_name: size
 stat_value: 25
 sample_size: NULL
stat_description: Number of pages in the index
7 rows in set (0.0007 sec)
```

第 1 行+4 表示国家代码索引，而第 5 行+7 表示主键。首先要注意的是，国家代码索引n_diff_pfx01n_diff_pfx02和数据统计信息。为什么，考虑到索引只包括一列？请记住，InnoDB 使用群集索引，并且非独特索引始终会追加主键，因为无论如何，查找实际行都需要主键。这就是您在这里看到的，n_diff_pfx01国家代码列，n_diff_pfx02国家代码和 ID 列的组合。

"国家代码"索引是八页大，其中七页是叶节点。这意味着索引具有两个级别，叶节点为级别 0，根节点为级别 1。我们鼓励您回到上一章中有关 B 树索引的讨论，并在查看表中某些索引的大小统计信息时查看它。

主键更简单，因为它只包含一列。这里有 24 个叶页，因此只采样了索引的子集。（请记住，对于主键，索引是表。这样做的后果是统计数据并不精确。主n_diff_pfx01预测 4188 个唯一值。由于它是主键，因此这也是行总数的估计值。但是，如果您查看国家/地区代码的统计数据，则预计国家/地区代码和 ID 值有 4079 种不同的组合。由于"国家代码"索引只有七个叶页，因此已检查所有页，并且行估计值精确。

与持久统计信息相关的另一个innodb_table_stats。它类似于innodb_index_stats，只是它是包含的整个表的聚合统计信息。表 innodb_table_stats15-2 中总结了这些列的列。

主键由列和database_nametable_name。表统计信息需要注意的一个重要点是，它们与索引统计信息一样近似。表中的行数只是主键的估计基数。同样，群集索引大小与来自表中主键innodb_index_stats相同。辅助索引页数是每个辅助索引的大小的总和。清单15-3显示了innodb_table_stats表内容的示例，该表使用与上一示例中相同的索引统计信息。

```
Listing 15-3. The innodb_table_stats table for the world.city table
mysql> SELECT *
 FROM mysql.innodb_table_stats
 WHERE database_name = 'world'
 AND table_name = 'city'\G
*************************** 1. row ***************************
 database_name: world
 table_name: city
 last_update: 2019-05-25 13:51:40
 n_rows: 4188
 clustered_index_size: 25
sum_of_other_index_sizes: 8
1 row in set (0.0005 sec)
```

------

**提示** innodb_index_stats 和innodb_table_stats是常规表。在备份中包含表非常有用，因此如果查询计划突然更改，可以返回并比较统计信息。还可以为具有 UPDATE 权限的用户更新表。这似乎是一个非常有用的属性，但要小心。如果不知道正确的统计信息，您最终会有非常差的查询计划。手动修改索引统计信息几乎不应完成。如果完成，则更改仅在刷新表后生效。

------

如果您觉得innodb_index_ 统计数据和 innodb_table_stats 中有关信息的讨论与 SHOW INDEX 语句以及表和统计信息架构表中可能用于查看的信息类似，那么您就是对的。有一些重叠。由于这些来源也适用于瞬态统计，因此将推迟到讨论瞬态索引统计之后。

## Transient Index Statistics

瞬态索引统计信息是 InnoDB 中用于处理索引统计信息的原始方法。正如名称建议的那样，统计信息不是永久性的，也就是说，当 MySQL 重新启动时，它们不会持久化。相反，当表首次打开（在其他时间）并且仅保留在内存中时，将计算统计信息。由于统计信息未持久化，因此它们不太稳定，因此更有可能看到查询计划的更改。

有两个配置选项用于影响瞬态统计的行为。这些是

- **innodb_stats_transient_sample_pages**：更新索引统计信息时要采样的页数。默认值为 8。
- **innodb_stats_on_metadata**：是否查询表的元数据时重新计算统计信息。默认值为 OFF，自 MySQL 5.6 以来一直如此。

"innodb_stats_transient_sample_pages与"innodb_stats_ persistent_sample_pages，但适用于使用瞬态统计信息的表。使用瞬态统计信息的表不仅在首次打开时重新计算统计信息，而且在只有 6.25% （1/16） 的行发生更改时，还要求至少发生 16 次更新。此外，在自动重新计算统计信息时，瞬态统计信息不使用后台线程，因此更新更有可能影响性能。因此，innodb_stats_ transient_sample_pages的默认值。

如果要更频繁地更新瞬态索引统计信息，可以启用innodb_stats_on_metadata选项。启用此选项后，查询信息架构中的表和统计信息表或使用等效的 SHOW 语句将触发索引统计信息的更新。实际上，这很少有原因，并且可以安全地关闭该选项。

没有可用于瞬态统计的特殊表。但是，MySQL 中的所有表都可用表和语句。

## Monitoring

索引统计信息对于优化器非常重要，有助于确定执行查询的最佳方法。因此，了解如何检查表的索引统计信息也很重要。已经讨论过，对于持久统计，有mysql.innodb_index_statsmysql.innodb_table_stats表。但是，也有一般的方法，这些将在这里讨论。

------

**提示** 请记住，information_schema_stats_expiry变量会影响数据字典刷新其与索引统计信息相关的数据视图的检查时间。

------



### Information Schema STATISTICS View

获取有关索引统计信息的详细信息的主表是信息架构中的 STATISTICS 视图。该视图不仅包含索引统计信息本身，还包含有关索引的元信息。实际上，您可以基于 STATISTICS 视图中的数据重新创建索引定义。这是上一章中用于查找表上的索引名称的视图。

表 15-3 包含视图中列的摘要。您通常只需要列的子集，但可以方便地访问所有需要时的情况信息。"基数"列是唯一受"information_schema_stats_expiry列。

STATISTICS 视图不仅与索引统计信息有关，而且对于索引本身也很有用，并且它包括有关所有索引的信息，而与索引类型无关。例如，您可以使用它查找不可见索引和用于函数索引的表达式。关于索引统计信息，最有趣的列是基数，它是估计存在于索引中的唯一值的数量。

当您查询"统计"视图时，建议按TABLE_SCHEMA、TABLE_NAMEINDEX_NAME列SEQ_IN_INDEX结果。这将将相关行分组在一起，对于多列索引，这些行将按索引中列的顺序返回。清单15-4显示了世界索引的示例。在这种情况下，排序仅在索引名称和索引中的序列上，因为表架构和表名是固定的。由于这些值本质上是不准确的，因此结果可能会有所不同。

```
Listing 15-4. The STATISTICS view for the world.countrylanguage table
mysql> SELECT INDEX_NAME, NON_UNIQUE,
 SEQ_IN_INDEX, COLUMN_NAME,
 CARDINALITY, INDEX_TYPE,
 IS_VISIBLE
 FROM information_schema.STATISTICS
 WHERE TABLE_SCHEMA = 'world'
 AND TABLE_NAME = 'countrylanguage'
 ORDER BY INDEX_NAME, SEQ_IN_INDEX\G
*************************** 1. row ***************************
 INDEX_NAME: CountryCode
 NON_UNIQUE: 1
SEQ_IN_INDEX: 1
 COLUMN_NAME: CountryCode
 CARDINALITY: 233
 INDEX_TYPE: BTREE
 IS_VISIBLE: YES
*************************** 2. row ***************************
 INDEX_NAME: PRIMARY
 NON_UNIQUE: 0
SEQ_IN_INDEX: 1
 COLUMN_NAME: CountryCode
 CARDINALITY: 233
 INDEX_TYPE: BTREE
 IS_VISIBLE: YES
*************************** 3. row ***************************
 INDEX_NAME: PRIMARY
 NON_UNIQUE: 0
SEQ_IN_INDEX: 2
 COLUMN_NAME: Language
 CARDINALITY: 984
 INDEX_TYPE: BTREE
 IS_VISIBLE: YES
3 rows in set (0.0010 sec)
```

国家语言表有两个索引。"国家/地区代码"和"语言"列上有一个主键，并且仅国家/地区代码上也有一个辅助索引。与mysql.innodb_index_stats表不同，主键附加到辅助非统一索引时也有行，则"统计信息"视图不包括该信息。

------

**注意** "国家代码"列上的辅助索引本身是冗余的，因为"国家代码"列是主键中的第一列。这意味着主键可以与辅助索引一样使用。最佳做法是避免冗余索引。

------

您可能希望在"统计信息"视图中保留数据记录，并比较数据随时间的变化。突然的变化可能表明数据发生了意外情况，或者索引统计信息的最新重新计算可能导致不同的查询计划。

统计视图中的一些信息也可通过 SHOW INDEX 语句获得。

### The SHOW INDEX Statement

SHOW INDEX 语句是获取有关 MySQL 中索引信息的原始方式。今天，它从与用户相同的源获取information_schema。统计，所以您可以使用，因为它最适合你。统计视图的一个主要优点是，您可以选择您想要的信息以及如何订购这些信息;使用 SHOW INDEX 语句时，您始终获取单个表的索引，并订购了基于可用字段进行筛选的选项。

SHOW INDEX 返回的列与 STATISTICS 视图中的列相同，只是省略了表目录、表架构和索引架构。另一方面，SHOW INDEX 可以选择使用扩展关键字，该关键字包括有关索引隐藏部分的信息。这不应与不可见索引混淆，而应该是附加部分，如附加到辅助索引的主要键。标准和扩展输出具有与通用行相同的信息。

清单 15-5 显示了世界 SHOW INDEX 输出的示例。城市表（结果假定第 14 章中的索引已被删除）。首先，返回标准输出，然后是扩展输出。由于扩展输出是几页长，因此通过删除某些列和行来缩写它。若要查看完整输出，请自己执行语句或查看listing_15_5 GitHub 存储库中可用的 listing_15_5.txt 文件。

```
Listing 15-5. The SHOW INDEX output for the world.city table
mysql> SHOW INDEX FROM world.city\G
*************************** 1. row ***************************
 Table: city
 Non_unique: 0
 Key_name: PRIMARY
 Seq_in_index: 1
 Column_name: ID
 Collation: A
 Cardinality: 4188
 Sub_part: NULL
 Packed: NULL
 Null:
 Index_type: BTREE
 Comment:
Index_comment:
 Visible: YES
 Expression: NULL
*************************** 2. row ***************************
 Table: city
 Non_unique: 1
 Key_name: CountryCode
 Seq_in_index: 1
 Column_name: CountryCode
 Collation: A
 Cardinality: 232
 Sub_part: NULL
 Packed: NULL
 Null:
 Index_type: BTREE
 Comment:
Index_comment:
 Visible: YES
 Expression: NULL
2 rows in set (0.0013 sec)
mysql> SHOW EXTENDED INDEX FROM world.city\G
*************************** 1. row ***************************
 Non_unique: 0
 Key_name: PRIMARY
 Seq_in_index: 1
 Column_name: ID
 Cardinality: 4188
*************************** 2. row ***************************
 Non_unique: 0
 Key_name: PRIMARY
 Seq_in_index: 2
 Column_name: DB_TRX_ID
 Cardinality: NULL
*************************** 3. row ***************************
 Non_unique: 0
 Key_name: PRIMARY
 Seq_in_index: 3
 Column_name: DB_ROLL_PTR
 Cardinality: NULL
*************************** 4. row ***************************
 Non_unique: 0
 Key_name: PRIMARY
 Seq_in_index: 4
 Column_name: Name
 Cardinality: NULL
...
*************************** 8. row ***************************
 Non_unique: 1
 Key_name: CountryCode
 Seq_in_index: 1
 Column_name: CountryCode
 Cardinality: 232
*************************** 9. row ***************************
 Non_unique: 1
 Key_name: CountryCode
 Seq_in_index: 2
 Column_name: ID
 Cardinality: NULL
9 rows in set (0.0013 sec)
```

请注意列名称与 STATISTICS 视图使用的名称不同。但是，列的顺序相同，名称相似，因此很容易将两个输出映射到彼此

在扩展输出中，主键在 InnoDB 内部有两个隐藏列：DB_TRX_ID 这是 6 字节事务标识符和 DB_ROLL_PTR 这是指向写入回滚段的撤消日志记录的 7 字节滚动指针。这些是 InnoDB 多版本控制支持的一部分。这反映 InnoDB 使用群集索引进行其行，因此主键是行

对于国家/地区代码上的辅助索引，主键现在显示为索引的第二部分。这是预料之中的，反映了在 mysql 中也看到的东西。innodb_index_stats表。

虽然扩展输出在调查性能问题时通常不感兴趣，但探索 InnoDB 的工作原理时，扩展输出的价值不高。

使用索引统计信息时有用的另一个信息架构视图是INNODB_TABLESTATS视图。

### The Information Schema INNODB_TABLESTATS View

信息INNODB_TABLESTATS中的视图是 InnoDB 内部内存结构上的视图，该结构包含有关索引的信息。它不包含任何可用于验证未包含在已描述的表和视图中的索引的基数和大小的信息。但是，它确实提供了对索引统计信息的状态以及自上次分析表以来的修改数的一些见解。该视图包括所有 InnoDB 表的信息，无论它们是使用持久统计还是瞬态统计信息。表 15-4 汇总了视图INNODB_TABLESTATS列。

初始化状态可能会导致混淆。这显示索引统计信息和相关元数据（如此视图中公开）是否已加载到内存中。即使存在统计信息，状态始终以未初始化状态开始。当某些连接或后台线程需要数据时，InnoDB 会将数据加载到内存中，状态将变为初始化。每当没有线程持有对表的引用时，InnoDB 可以自由再次逐出信息，并且状态将变为"未初始化"。例如，当为表刷新或为表执行 ANALYZE 表时，可能会发生这种情况。

修改的计数器很有趣，因为它可用于查看自上次更新索引统计信息以来更改的行数。只有当 DML 查询影响索引时，计数器才增加。这意味着，如果更新非索引列，并保留行，则计数器不会递增。计数器与在进行给定数量的更改时触发的自动更新相关。

清单 15-6 提供了世界INNODB_TABLESTATS视图的示例输出。如果执行相同的查询，表 ID、行数和引用计数可能不同。

```
Listing 15-6. The INNODB_TABLESTATS view for the world.city table
mysql> SELECT *
 FROM information_schema.INNODB_TABLESTATS
 WHERE NAME = 'world/city'\G
*************************** 1. row ***************************
 TABLE_ID: 1670
 NAME: world/city
STATS_INITIALIZED: Initialized
 NUM_ROWS: 4188
 CLUST_INDEX_SIZE: 25
 OTHER_INDEX_SIZE: 8
 MODIFIED_COUNTER: 0
 AUTOINC: 4080
 REF_COUNT: 2
1 row in set (0.0009 sec)
```

输出显示索引统计信息是最新的，因为自上次分析以来没有修改过行。行数以及群集索引和辅助索引的大小与使用"索引"和"mysql.innodb_index_相同。这些表大小与相关数字也用于information_。表视图和"显示表状态"语句。

### The Information Schema TABLES View and SHOW TABLE STATUS

索引统计信息集合也是用于填充用户使用的表中某些列information_schema。表视图和"显示表状态"语句。这包括行数的估计值以及数据和索引的大小。

表 15-5 显示了表视图中列的摘要。除了"显示"、"TABLE_SCHEMA、TABLE_CATALOG TABLE_TYPETABLE_TYPE列TABLE_COMMENT之外，显示表状态语句的输出中具有相同的列，并且有几个列的名称略有不同。标有星号 （*） 的列受变量information_schema_stats_expiry。

在现有信息中，行数以及数据和索引的大小与索引统计信息关系最密切。TABLE 视图不仅可用于查询表大小的估计值，还可用于查询哪些表具有显式设置的持久统计信息变量。清单 15-7 显示了一个 chapter_15.t1 表示例，用正好 100 万行填充表，然后查询表的 TABLE 视图的内容。

```
Listing 15-7. The TABLES view for the table chapter_15.t1
mysql> CREATE TABLE chapter_15.t1 (
 id int unsigned NOT NULL auto_increment,
 val varchar(36) NOT NULL,
 PRIMARY KEY (id)
 ) STATS_PERSISTENT=1,
 STATS_SAMPLE_PAGES=50,
 STATS_AUTO_RECALC=1;
Query OK, 0 rows affected (0.5385 sec)
mysql> SET SESSION cte_max_recursion_depth = 1000000;
Query OK, 0 rows affected (0.0003 sec)
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.0002 sec)
mysql> INSERT INTO chapter_15.t1 (val)
 WITH RECURSIVE seq (i) AS (
 SELECT 1
 UNION ALL
 SELECT i + 1
 FROM seq WHERE i < 1000000
 )
 SELECT UUID()
 FROM seq;
Query OK, 1000000 rows affected (15.8552 sec)
Records: 1000000 Duplicates: 0 Warnings: 0
mysql> COMMIT;
Query OK, 0 rows affected (0.8306 sec)
mysql> SELECT *
 FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = 'chapter_15'
 AND TABLE_NAME = 't1'\G
*************************** 1. row ***************************
 TABLE_CATALOG: def
 TABLE_SCHEMA: chapter_15
 TABLE_NAME: t1
 TABLE_TYPE: BASE TABLE
 ENGINE: InnoDB
 VERSION: 10
 ROW_FORMAT: Dynamic
 TABLE_ROWS: 996442
 AVG_ROW_LENGTH: 64
 DATA_LENGTH: 64569344
MAX_DATA_LENGTH: 0
 INDEX_LENGTH: 0
 DATA_FREE: 7340032
 AUTO_INCREMENT: 1048561
 CREATE_TIME: 2019-11-02 11:48:28
 UPDATE_TIME: 2019-11-02 11:49:25
 CHECK_TIME: NULL
TABLE_COLLATION: utf8mb4_0900_ai_ci
 CHECKSUM: NULL
 CREATE_OPTIONS: stats_sample_pages=50 stats_auto_recalc=1 stats_
persistent=1
 TABLE_COMMENT:
1 row in set (0.0653 sec)
```

该表使用递归公共表表达式填充随机数据，以确保插入正好 100 万行。要这样做，必须将 cte_max_recursion_depth设置为 10000000，否则公共表表达式将失败，而递归深度过高。

请注意，估计行数仅 996442 行，比实际行数少 0.3% 左右。这在预期范围内 - 差异高达 10% 或更多并不罕见。该表还设置了多个表选项，用于显式配置启用自动重新计算和使用 50 个示例页的表使用持久统计信息。

如果希望改为使用 SHOW TABLE STATUS 语句，则无需参数即可使用它，在这种情况下，将返回默认架构中所有表的表状态。或者，您可以添加一个 LIKE 子句，以仅包含表的子集。若要检索非默认架构中的表的表状态，请使用 FROM 子句指定架构名称。例如，将世界架构作为默认值，然后以下查询将全部返回城市表的表状态：

```
mysql> use world
mysql> SHOW TABLE STATUS LIKE 'city';
mysql> SHOW TABLE STATUS LIKE 'ci%';
mysql> SHOW TABLE STATUS FROM world LIKE 'city';
```

前两个查询依赖于默认架构来了解在什么位置查找表。第三个查询显式搜索世界架构中的城市表。

如果索引统计信息没有数据，如何更新它们？这是本章结束之前要探讨的最后一个主题。

## Updating the Statistics

最新的索引统计信息对于优化器实现最佳查询执行计划非常重要。索引有两种更新方式：自动更新，因为对表的更改已足够，可以触发统计信息的重新计算，并手动触发更新。

### Automatic Updates

在涉及持久性和瞬态性统计时，已经在某种程度上讨论了自动更新机制。表 15-6 根据索引统计信息类型汇总了该功能。

摘要显示，持久性统计信息一般更新频率较低，并且影响较小，因为自动更新在后台进行。持久统计信息还具有更好的配置选项。

也可以手动触发索引统计信息的更新。可以使用 ANALYZE TABLE 语句或 mysqlcheck 命令行程序，如下各节所述。

### The ANALYZE TABLE Statement

当您在 mysql 命令行客户端或 MySQL 命令程序中工作或更新将由存储过程触发时，使用 ANALYZE TABLE 语句非常方便。该语句可以更新索引统计信息和直方图。后者将在下一章中讨论，因此这里只介绍索引统计信息的更新。

Analyze TABLE 有一个参数，即是否将语句记录到二进制日志。如果在 ANALYZE 和 TABLE NO_WRITE_TO_BINLOG指定"分析"或"本地"，则该语句将仅应用于本地实例，而不是写入二进制日志。

执行 ANALYZE TABLE 时，它会强制刷新索引统计信息和表缓存值，否则这些值受information_schema_stats_expiry变量的影响。因此，如果强制更新索引统计信息，则不需要更改information_schema_stats_expiry要information_schema。统计视图和类似地反映更新的值。

您可以选择指定多个表以更新其索引统计信息。通过将表列在逗号分隔列表中来实现此目的。在清单15-8中可以看到更新世界架构中三个表的统计信息的示例。

```
Listing 15-8. Analyzing the index statistics for the tables in the world schema
mysql> ANALYZE LOCAL TABLE
 world.city, world.country,
 world.countrylanguage\G
*************************** 1. row ***************************
 Table: world.city
 Op: analyze
Msg_type: status
Msg_text: OK
*************************** 2. row ***************************
 Table: world.country
 Op: analyze
Msg_type: status
Msg_text: OK
*************************** 3. row ***************************
 Table: world.countrylanguage
 Op: analyze
Msg_type: status
Msg_text: OK
3 rows in set (0.0248 sec)
```

在示例中，LOCAL 关键字用于避免将语句记录到二进制日志。如果未将架构名称与表名称（例如，城市而不是 world.city）一起指定，则 MySQL 将在当前默认架构中显示该表。

------

**注意**：虽然可以与 ANALYZE 表同时查询表，但请注意，作为最后一步（返回到客户端后），将刷新分析表（隐式 FLUSH TABLES 语句）。只有在所有进行中的查询完成后，才能进行表刷新，因此在具有长时间运行的查询时，不应使用 ANALYZE TABLE（或 mysqlcheck）。

------

ANALYZE TABLE 语句非常适合临时更新，以及您确切地知道要分析的表时。它对于分析给定架构中的所有表或实例中的所有表不太有用。为此，接下来讨论的 mysqlcheck 是一个更好的选择。

### The mysqlcheck Program

例如，如果您想要通过 cron 守护程序或 Windows 任务计划程序从 shell 脚本触发更新，则 mysqlcheck 程序非常方便。它不仅可用于更新单个表或多个表（如 ANALYZE TABLE）上的索引统计信息，还可以告诉 mysqlcheck 更新架构中所有表或实例中所有表的索引统计信息。mysqlcheck 所做是为符合您条件的表执行 ANALYZE TABLE，因此从索引统计信息的角度来看，手动执行 ANAYZE 表和使用 mysqlcheck 之间没有区别。

------

**注意** mysqlcheck 程序可以不仅仅分析表来更新索引统计信息。此处仅介绍分析功能。若要阅读 mysqlcheck 程序的完整文档，请参阅https://dev.mysql。com/doc/refman/en/mysqlcheck.html.

------

使用 --analyze 选项使 mysqlcheck 更新索引统计信息和 --write-binlog/--跳过-写入 binlog 参数来判断您是否希望将语句记录到二进制日志中。默认值是记录语句。您还需要告诉如何连接到 MySQL;为此，您使用标准连接选项

有三种方法可以指定要分析的表。默认值是分析同一架构中的一个或多个表，如 ANALYZE TABLE 语句。如果选择，则不需要添加任何额外的选项，并且指定的第一个值将解释为架构名称和可选参数作为表名。清单 15-9 演示了如何以两种方式分析世界架构中的所有表：通过显式列出表名而不列出表。

```
Listing 15-9. Using mysqlcheck to analyze all tables in the world schema
shell$ mysqlcheck --user=root --password --host=localhost --port=3306
--analyze world city country countrylanguage
Enter password: ********
world.city OK
world.country OK
world.countrylanguage OK
shell$ mysqlcheck --user=root --password --host=localhost --analyze world
Enter password: ********
world.city OK
world.country OK
world.countrylanguage OK
```

在这两种情况下，输出都列出了被分析的三个表。

如果要分析多个架构中的所有表，但仍列出要包括的架构，可以使用 --数据库参数。当存在时，命令行上列出的所有对象名称都解释为架构名称。清单 15-10 显示了分析 sakila 和世界架构中所有表的示例。

```
Listing 15-10. Analyze all tables in the sakila and world schemas
shell$ mysqlcheck --user=root --password --host=localhost --port=3306
--analyze --databases sakila world
Enter password: ********
sakila.actor OK
sakila.address OK
sakila.category OK
sakila.city OK
sakila.country OK
sakila.customer OK
sakila.film OK
sakila.film_actor OK
sakila.film_category OK
sakila.film_text OK
sakila.inventory OK
sakila.language OK
sakila.payment OK
sakila.rental OK
sakila.staff OK
sakila.store OK
world.city OK
world.country OK
world.countrylanguage OK
```

最后一个选项是使用  --all-databases 选项来分析所有表，而不管它们位于哪个架构中。这将包括系统表，但信息架构和性能架构除外。清单 15-11 显示了将 mysqlcheck 与  --all-databases一起使用的示例。

```
Listing 15-11. Analyzing all tables
shell$ mysqlcheck --user=root --password --host=localhost --port=3306
--analyze --all-databases
Enter password: ********
mysql.columns_priv OK
mysql.component OK
mysql.db OK
mysql.default_roles OK
mysql.engine_cost OK
mysql.func OK
mysql.general_log
note : The storage engine for the table doesn't support analyze
mysql.global_grants OK
mysql.gtid_executed OK
mysql.help_category OK
mysql.help_keyword OK
mysql.help_relation OK
mysql.help_topic OK
mysql.innodb_index_stats OK
mysql.innodb_table_stats OK
mysql.password_history OK
mysql.plugin OK
mysql.procs_priv OK
mysql.proxies_priv OK
mysql.role_edges OK
mysql.server_cost OK
mysql.servers OK
mysql.slave_master_info OK
mysql.slave_relay_log_info OK
mysql.slave_worker_info OK
mysql.slow_log
note : The storage engine for the table doesn't support analyze
mysql.tables_priv OK
mysql.time_zone OK
mysql.time_zone_leap_second OK
mysql.time_zone_name OK
mysql.time_zone_transition OK
mysql.time_zone_transition_type OK
mysql.user OK
sakila.actor OK
sakila.address OK
sakila.category OK
sakila.city OK
sakila.country OK
sakila.customer OK
sakila.film OK
sakila.film_actor OK
sakila.film_category OK
sakila.film_text OK
sakila.inventory OK
sakila.language OK
sakila.payment OK
sakila.rental OK
sakila.staff OK
sakila.store OK
sys.sys_config OK
world.city OK
world.country OK
world.countrylanguage OK
```

请注意，有两个表如何回复其存储引擎不支持分析。mysqlcheck 程序尝试分析所有表，而不考虑其存储引擎，因此预期中的示例中的消息。默认情况下mysql.general_log和mysql.slow_log都使用 CSV 存储引擎，该引擎不支持索引，因此两者都不支持 ANALYZE 表。



## 总结

本章通过查看 InnoDB 如何处理索引统计信息来了解上一章的结尾。InnoDB 有两种方法可以存储统计信息：在数据库中持久mysql.innodb_index_statsmysql.innodb_table_stats或内存中的瞬态。持久统计信息通常首选，因为它们提供更一致的查询计划，允许采样更多页面，在后台更新，并且可以配置到更大程度，包括支持表级选项。

有几个表、视图和 SHOW 语句可用于调查和了解 InnoDB 索引及其统计信息。特别令人感兴趣的是information_schema。统计信息视图，其中包含 MySQL 中所有索引的详细信息。information_schema。INNODB_TABLESTATS和information_schema。还讨论了表视图、显示索引和显示表状态语句。

您可以通过两种方式更新索引统计信息：使用 ANALYZE TABLE 语句或 mysqlcheck 程序。前者在交互式客户端或存储过程内很有用，而后者对 shell 脚本和更新一个或多个架构中的所有表更有用。这两种方法还强制更新表元数据的缓存值和 MySQL 数据字典中的索引基数。

在讨论 ANALYZE TABLE 语句时，有人提到 MySQL 也支持直方图。这些与索引相关，是下一章的主题。