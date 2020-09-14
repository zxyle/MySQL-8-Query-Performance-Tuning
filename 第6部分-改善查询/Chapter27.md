# 缓存

最便宜的查询是您根本不执行的查询。 本章研究如何使用缓存来避免执行查询或降低查询的复杂性。 首先，将讨论缓存如何到处存在以及如何有不同类型的缓存。 然后介绍了如何使用缓存表和近似值在MySQL内部使用缓存。 接下来的两个部分将介绍提供缓存的两种流行产品：Memcached和ProxySQL。 最后，讨论了一些缓存技巧。

## 缓存无处不在

即使您认为没有实现缓存，也已经在多个地方使用了缓存。 这些缓存是透明的，并在硬件，操作系统或MySQL级别进行维护。 这些缓存中最明显的是InnoDB缓冲池。

图27-1显示了整个系统中如何存在缓存的示例，以及如何添加自定义缓存的示例。 这幅图（包括交互作用）绝不是完整的，但它可以说明常见的缓存程度以及可以在多少地方发生。

在左下角，有一个CPU，它具有几级缓存，用于缓存指令和用于CPU指令的数据。操作系统实现了一个I / O缓存，InnoDB有其缓冲池。所有这些缓存都是返回最新数据的缓存的示例。

也有可能提供稍微陈旧的数据的缓存。这包括在MySQL中实现缓存表，在ProxySQL中缓存查询结果或直接在应用程序中缓存数据。在这些情况下，通常需要定义一个时间段来考虑数据是否足够新鲜，并且当数据达到给定的期限（生存时间（TTL））时，缓存条目将无效。 Memcached解决方案很特别，因为它有两个版本。常规的Memcached守护程序使用时间生存，或者当数据太旧时会使用某些依赖于应用程序的逻辑将数据逐出。但是，还有一个特殊的MySQL版本可作为插件使用，并且可以从InnoDB缓冲池中获取数据并将数据写回到缓冲池中，因此数据永远不会过时。

在应用程序中使用可能过时的数据似乎是错误的。但是，在许多情况下，这是完全可以的，因为不需要精确的数据。如果您有一个显示销售数字仪表盘的应用程序，那么如果该数据在执行查询时是最新的或者是几分钟后才出现，那会有什么不同？到用户读完这些数字时，它们可能总是有点过时了。重要的是，销售数据必须一致并定期更新。

------

**提示** 请仔细考虑您的应用程序的需求，并记住，从宽松的需求开始就必须更新数据，并使其更加严格，这比说服用户不再拥有最新数据要容易得多。第二个结果。如果您使用的缓存数据没有自动更新为最新值，则可以考虑存储数据的最新时间并将其显示给用户，以便用户知道上一次刷新数据的时间。

------

接下来的三节将介绍更具体的缓存示例，从在MySQL中实现自己的缓存开始。

## Caching Inside MySQL

在MySQL内部，实现缓存的合理位置。 如果将缓存的数据与其他表一起使用，这将特别有用。 不利之处在于，它仍然需要从应用程序到数据库的往返查询数据，并且还需要执行查询。 本节介绍了两种在MySQL中缓存数据的方式：缓存表和直方图统计信息。

### Cache Tables

缓存表可用于预先计算数据，例如报告或仪表板。它对于经常需要的复杂聚合很有用。

有几种使用缓存表的方法。您可以选择创建一个表，该表存储与其一起使用的功能的结果。这使其使用起来便宜，但又相对不灵活，因为它只能与该功能一起使用。或者，您可以创建需要连接在一起的构建基块，以便可以将它们用于多个功能。这会使查询的开销增加一点，但是您可以重用缓存的数据，并避免重复数据。这取决于您的应用程序，哪种方法是最好的，并且您最终可能会选择一个混合表，其中一些表是单独使用的，而另一些表是连接在一起的。

填充缓存表有两种主要策略。您可以定期完全重建表，也可以使用触发器使数据连续不断地更新。完全重建表的最佳方式是创建一个新的缓存表副本，并在重建结束时使用RENAME TABLE交换表，因为这样可以避免删除事务中潜在的大量行，并且可以避免随着时间的流逝而产生碎片。或者，您可以使用触发器来更新缓存的数据，因为该数据取决于更改。在大多数情况下，如果可以使用不完全最新的数据是可以接受的，则重建缓存表是首选方法，因为这样不太容易出错，并且刷新是在后台完成的。

------

**提示**：如果通过删除事务中的现有数据来重建缓存表，请禁用索引统计信息的自动重新计算，并在重建结束时使用ANALYZE TABLE或启用innodb_stats_ include_delete_marked选项。

------

特殊情况是表中包含的缓存列，否则不缓存数据。 缓存列有用的一个示例是存储属于某个组的最新事件的时间，状态或ID。 想象一下，您的应用程序支持发送文本消息，并且对于每条消息，您都存储了历史记录，例如何时在应用程序中创建历史记录，何时发送历史记录以及收件人何时确认消息。 在大多数情况下，仅需要最新状态以及何时达到该状态，因此您可能希望将其与消息记录本身一起存储，而不必显式查询。 在这种情况下，您可以使用两个表来存储状态：

```sql
CREATE TABLE message (
 message_id bigint unsigned NOT NULL auto_increment,
 message_text varchar(1024) NOT NULL,
 cached_status_time datetime(3) NOT NULL,
 cached_status_id tinyint unsigned NOT NULL,
 PRIMARY KEY (message_id)
);
CREATE TABLE message_status_history (
 message_status_id bigint unsigned NOT NULL auto_increment,
 message_id bigint unsigned NOT NULL,
 status_time datetime(3) NOT NULL,
 status_id tinyint unsigned NOT NULL,
 PRIMARY KEY (message_status_id)
);
```

在现实世界中，可能会有更多的列和外键，但例如，此信息就足够了。 当消息的状态更改时，会将一行插入到message_status_history表中。 您可以在消息的最新行中查找最新状态，但是此处已创建了业务规则，用于使用最新状态和更改时间来更新消息表中的cached_ status_time和cached_status_id。 这样，要返回消息的应用程序详细信息（需要历史记录时除外），只需查询消息表。 您可以通过应用程序或触发器来更新缓存的列，或者如果不需要缓存的状态为最新，则可以使用后台作业。

提示请使用一种命名方案，使之可以清楚地了解哪些数据被缓存，哪些数据未被缓存。 例如，您可以为缓存表和列添加cached_前缀。

您可以考虑缓存的另一种情况是直方图统计信息。

### Histogram Statistics

回顾一下第16章，直方图统计信息是如何统计列中每个值遇到频率的频率。 您可以利用此优势，并将直方图统计信息用作缓存。 如果该列最多包含1024个唯一值，这是最有用的，因为这是支持的最大存储桶数，因此1024个是可与单身直方图一起使用的最大值数。

清单27-1显示了一个示例，该示例使用直方图在世界数据库中返回印度的城市数（CountryCode = IND）。

```
Listing 27-1. Using histograms as a cache
-- Create the histogram on the CountryCode
-- column of the world.city table.
mysql> ANALYZE TABLE world.city
 UPDATE HISTOGRAM on CountryCode
 WITH 1024 BUCKETS\G
*************************** 1. row ***************************
 Table: world.city
 Op: histogram
Msg_type: status
Msg_text: Histogram statistics created for column 'CountryCode'.
1 row in set (0.5909 sec)
mysql> SELECT Bucket_Value, Frequency
 FROM (
 SELECT (Row_ID - 1) AS Bucket_Number,
 SUBSTRING_INDEX(Bucket_Value, ':', -1)
 AS Bucket_Value,
 (Cumulative_Frequency
 - LAG(Cumulative_Frequency, 1, 0)
 OVER (ORDER BY Row_ID))
 AS Frequency
 FROM information_schema.COLUMN_STATISTICS
 INNER JOIN JSON_TABLE(
 histogram->'$.buckets',
Chapter 27 Caching
923
 '$[*]' COLUMNS(
 Row_ID FOR ORDINALITY,
Bucket_Value varchar(42) PATH '$[0]',
Cumulative_Frequency double PATH '$[1]'
 )
 ) buckets
 WHERE SCHEMA_NAME = 'world'
 AND TABLE_NAME = 'city'
 AND COLUMN_NAME = 'CountryCode'
 ) stats
 WHERE Bucket_Value = 'IND';
+--------------+---------------------+
| Bucket_Value | Frequency           |
+--------------+---------------------+
| IND          | 0.08359892130424124 |
+--------------+---------------------+
1 row in set (0.0102 sec)
mysql> SELECT TABLE_ROWS
 FROM information_schema.TABLES
 WHERE TABLE_SCHEMA = 'world'
 AND TABLE_NAME = 'city';
+------------+
| TABLE_ROWS |
+------------+
| 4188       |
+------------+
1 row in set (0.0075 sec)
mysql> SELECT 0.08359892130424124*4188;
+--------------------------+
| 0.08359892130424124*4188 |
+--------------------------+
| 350.11228242216231312    |
+--------------------------+
1 row in set (0.0023 sec)
mysql> SELECT COUNT(*)
 FROM world.city
 WHERE CountryCode = 'IND';
+----------+
| COUNT(*) |
+----------+
| 341      |
+----------+
1 row in set (0.0360 sec)
```

如果您认为针对COLUMN_STATITICS的查询看起来很熟悉，则该查询源自第16章中列出单例直方图的存储桶信息时使用的查询。必须在子查询中收集直方图信息，否则将不计算频率。

您还将需要总行数。您可以使用information_schema.TABLES视图中的近似值，也可以缓存表的SELECT COUNT（*）结果。在此示例中，估计城市表中有4188行（您的估计可能有所不同），再加上印度的频率，表明该表中大约有350个印度城市。确切的计数显示有341。偏差来自总行数估计（城市表中有4079行）。

使用直方图作为缓存对于具有最多1024个唯一值的列的大型表尤其有用，尤其是在该列上没有索引的情况下。这意味着它与所有许多用例都不匹配。但是，它确实显示了一个跳出思路的示例–当您尝试查找缓存解决方案时，这非常有用。

对于更高级的缓存解决方案，您需要查看第三方解决方案或在应用程序中实现自己的解决方案。

## Memcached

Memcached是一个简单但具有高度可扩展性的内存中键值存储，已广泛用作缓存工具。 传统上，它通常与Web服务器一起使用，但可以由任何类型的应用程序使用。 Memcached的一个优点是它可以分布在多个主机上，这使您可以创建大型缓存。

注意Memcached仅在Linux和Unix上正式受支持。

通过MySQL使用Memcached有两种方法。 您可以使用常规的独立Memcached，也可以使用MySQL InnoDB Memcached插件。 本节将显示一个简单的示例，将两者同时使用。 有关完整的Memcached文档，请参见https://memcached.org/上的官方主页以及https://github.com/memcached/memcached/wiki上的官方Wiki。

### Standalone Memcached

独立的Memcached是https://memcached.org/中的官方守护程序。 它允许您将其用作分布式缓存，或者使缓存与应用程序非常接近（可能在同一主机上），从而降低了查询缓存的成本。

有几种安装Memcached的选项，包括使用操作系统的程序包管理器和从源代码进行编译。 最简单的方法是在Oracle Linux，Red Hat Enterprise Linux和CentOS 7上使用软件包管理器：

```bash
shell$ sudo yum install memcached libevent
```

包含libevent软件包，因为memcached需要它。 在Ubuntu Linux上，该软件包称为libevent-dev。 您可能已经安装了libevent和/或memcached，在这种情况下，程序包管理器将让您知道无事可做。

您可以使用memcached命令启动守护程序。 例如，使用所有默认选项启动它

```bash
shell$ memcached
```

如果在生产环境中使用它，则应配置systemd或用于在操作系统启动和关闭时启动和停止守护程序的任何服务管理器。 对于测试，可以只从命令行启动它。

------

**警告** Memcached中没有安全支持。 将缓存的数据限制为非敏感数据，并确保您的Memcached实例仅在内部网络中可用，并使用防火墙来限制访问。 一种选择是将Memcached与应用程序部署在同一主机上，并阻止远程连接。

------

现在，您可以通过将从MySQL检索到的数据存储在缓存中来使用Memcached。 Memcached支持多种编程语言。 在此讨论中，Python将与pymemcache模块1和MySQL Connector / Python一起使用。 清单27-2显示了如何使用pip安装模块。 输出可能看起来有些不同，具体取决于所使用的Python的确切版本以及您已经安装的Python，Python命令的名称取决于您的系统。 在撰写本文时，pymemcache支持Python 2.7、3.5、3.6和3.7。 该示例使用在Oracle Linux 7上作为额外软件包安装的Python 3.6。

```
Listing 27-2. Installing the Python pymemcache module
shell$ python3 -m pip install --user pymemcache
Collecting pymemcache
 Downloading https://files.pythonhosted.org/packages/20/08/3dfe193f9a1dc6
0186fc40d41b7dc59f6bf2990722c3cbaf19cee36bbd93/pymemcache-2.2.2-py2.py3-
none-any.whl (44kB)
 |█████████████████████
███████████| 51kB 3.3MB/s
Requirement already satisfied: six in /usr/local/lib/python3.6/sitepackages (from pymemcache) (1.11.0)
Installing collected packages: pymemcache
Successfully installed pymemcache-2.2.2
shell$ python36 -m pip install --user mysql-connector-python
Collecting mysql-connector-python
 Downloading https://files.pythonhosted.org/packages/58/ac/
a3e86e5df84b818f69ebb8c89f282efe6a15d3ad63a769314cdd00bccbbb/mysql_
connector_python-8.0.17-cp36-cp36m-manylinux1_x86_64.whl (13.1MB)
 |█████████████████████
███████████| 13.1MB 5.6MB/s
Requirement already satisfied: protobuf>=3.0.0 in /usr/local/lib64/
python3.6/site-packages (from mysql-connector-python) (3.6.1)
Requirement already satisfied: setuptools in /usr/local/lib/python3.6/sitepackages (from protobuf>=3.0.0->mysql-connector-python) (39.0.1)
Requirement already satisfied: six>=1.9 in /usr/local/lib/python3.6/sitepackages (from protobuf>=3.0.0->mysql-connector-python) (1.11.0)
Installing collected packages: mysql-connector-python
Successfully installed mysql-connector-python-8.0.17
```

在您的应用程序中，您可以通过键查询Memcached。 如果找到了密钥，则Memcached返回与密钥一起存储的值，如果找不到，则需要查询MySQL并将结果存储在缓存中。 清单27-3给出了一个查询world.city表的简单示例。 也可以在本书的GitHub存储库中包含的listing_27_3.py文件中找到该程序。 如果要执行该程序，则需要更新connect_args中的连接参数以反映用于连接到MySQL实例的设置。

```python
Listing 27-3. Simple Python program using memcached and MySQL
from pymemcache.client.base import Client
import mysql.connector
connect_args = {
 "user": "root",
 "password": "password",
 "host": "localhost",
 "port": 3306,
}
db = mysql.connector.connect(**connect_args)
cursor = db.cursor()
memcache = Client(("localhost", 11211))
sql = "SELECT CountryCode, Name FROM world.city WHERE ID = %s"
city_id = 130
city = memcache.get(str(city_id))
if city is not None:
 country_code, name = city.decode("utf-8").split("|")
 print("memcached: country: {0} - city: {1}".format(country_code, name))
else:
 cursor.execute(sql, (city_id,))
 country_code, name = cursor.fetchone()
 memcache.set(str(city_id), "|".join([country_code, name]), expire=60)
 print("MySQL: country: {0} - city: {1}".format(country_code, name))
memcache.close()
cursor.close()
db.close()
```

该程序开始创建到MySQL和memcached守护程序的连接。 在这种情况下，将对连接参数和要查询的ID进行硬编码。 在实际程序中，您应该从配置文件或类似文件中读取连接参数。

------

**注意** 切勿将连接详细信息存储在应用程序中。 尤其不要对密码进行硬编码。 将连接详细信息存储在应用程序中既不灵活也不安全。

------

然后，程序尝试从Memcached获取数据。 请注意，由于Memcached使用字符串作为键，整数是如何转换为字符串的。 如果找到了密钥，则通过在|处分割字符串，从缓存的值中提取国家代码和名称。 字符。 如果在缓存中找不到密钥，则从MySQL获取城市数据并将其存储在缓存中，同时将缓存中的值保持为60秒。 为每种情况添加了打印语句，以显示从何处获取数据。

每次重新启动memcached后第一次执行程序时，它将最终查询MySQL：

```
shell$ python3 listing_27_3.py
MySQL: country: AUS - city: Sydney
```

在长达一分钟的后续执行中，将在缓存中找到数据：

```
shell$ python3 listing_27_3.py
memcached: country: AUS - city: Sydney
```

测试完Memcached后，可以在运行Memcached的会话中使用Ctrl + C停止它，或者通过发送SIGTEM（15）信号来停止它，例如：

```bash
shell$ kill -s SIGTERM $(pidof memcached)
```

如本例中所示，直接使用Memcached的优点是您可以拥有一个守护程序池，并且可以在应用程序附近运行该守护程序，甚至可以在与应用程序相同的主机上运行该守护程序。 缺点是您必须自己维护缓存。 一种替代方法是使用MySQL提供的memcached插件，该插件将为您管理缓存，甚至自动将写入持久保存到缓存中。

### MySQL InnoDB Memcached Plugin

InnoDB Memcached插件是在MySQL 5.6中引入的，它是一种访问InnoDB数据的方式，而无需解析SQL语句。该插件的主要用途是让InnoDB通过其缓冲池处理缓存，而仅将Memcached用作查询数据的机制。以这种方式使用插件的一些不错的功能是：对插件的写入将写入基础的InnoDB表，数据始终是最新的，并且您可以同时使用SQL和Memcached来访问数据。

------

**注意** 安装MySQL InnoDB Memcached插件之前，请确保已停止独立的Memcached进程，因为它们默认情况下使用相同的端口。如果不这样做，您将继续连接到独立进程。

------

在安装MySQL memcached守护程序之前，必须确保已安装libevent软件包，就像独立的Memcached安装一样。一旦安装了libevent，就需要安装innodb_memcache模式，该模式包括用于配置的表。您可以通过获取MySQL分发中包含的share / innodb_memcached_config.sql文件来执行安装。该文件是相对于MySQL基本目录的，您可以通过basedir系统变量找到它，例如：

```sql
mysql> SELECT @@global.basedir AS basedir;
+---------+
| basedir |
+---------+
| /usr/   |
+---------+
1 row in set (0.00 sec)
```

如果您已使用https://dev.mysql.com/downloads/中的RPM安装了MySQL，则命令为

```bash
mysql> SOURCE /usr/share/mysql-8.0/innodb_memcached_config.sql
```

------

**注意** 请注意，此命令在MySQL Shell中不起作用，因为该脚本包含USE命令，而MySQL脚本不支持分号。

------

该脚本还创建了test.demo_test表，该表将在本讨论的其余部分中使用。

innodb_memcache模式包含三个表：

- cache_policies：缓存策略的配置，用于定义缓存的工作方式。 默认是将其保留给InnoDB。 通常建议这样做，并确保您永远不会读取过时的数据。

- config_options：插件的配置选项。 这包括在返回值的多个列和表映射定界符时使用的分隔符。

- 容器：到InnoDB表的映射的定义。 您必须为要与InnoDB memcached插件一起使用的所有表添加一个映射。

容器表是您最常使用的表。 默认情况下，该表包含test.demo_test表的映射：

```sql
mysql> SELECT * FROM innodb_memcache.containers\G
*************************** 1. row ***************************
 name: aaa
 db_schema: test
 db_table: demo_test
 key_columns: c1
 value_columns: c2
 flags: c3
 cas_column: c4
 expire_time_column: c5
unique_idx_name_on_key: PRIMARY
1 row in set (0.0007 sec)
```

查询表时，可以使用该名称引用db_schema和db_table定义的表。 key_columns列定义InnoDB表中用于键查找的列。 您可以在value_columns列中指定要包括在查询结果中的列。 如果包含多列，则使用在行config_options表中配置的名称为=分隔符（默认为|）的分隔符来分隔列名。

cas_column和expire_time_column列很少需要，在此不再赘述。 最后一列unique_idx_name_on_key是表中唯一索引的名称，最好是主键

------

**提示** 有关表及其使用的详细说明，请参见https://dev.mysql.com/doc/refman/en/innodb-memcachedinternals.html。

------

现在，您准备安装插件本身。 您可以使用INSTALL PLUGIN命令执行此操作（请记住，这在Windows上不起作用）：

```sql
mysql> INSTALL PLUGIN daemon_memcached soname "libmemcached.so";
Query OK, 0 rows affected (0.09 sec)
```

该语句必须使用旧版MySQL协议（默认端口3306）执行，因为X协议（默认端口33060）不允许您安装插件。 就是这样– InnoDB memcached插件现在可以进行测试了。 测试它的最简单方法是使用telnet客户端。 清单27-4显示了一个简单的示例，该示例显式指定容器并使用默认容器。

```sql
Listing 27-4. Testing InnoDB memcached with telnet
shell$ telnet localhost 11211
Trying ::1...
Connected to localhost.
Escape character is '^]'.
get @@aaa.AA
VALUE @@aaa.AA 8 12
HELLO, HELLO
END

get AA
VALUE AA 8 12
HELLO, HELLO
END
```

为了便于查看这两个命令，在每个命令之前插入了一个空行。 第一个命令使用@@在键值之前指定容器名称。 第二个命令使用默认容器依赖于Memcached（当按容器名称按字母升序排序时，第一个条目）。 通过按Ctrl +]退出telnet，然后按quit命令：

```
^]
telnet> quit
Connection closed.
```

对于独立的Memcached实例，该守护程序默认使用端口11211。 如果要更改端口或其他任何Memcached选项，则可以使用daemon_memcached_option选项，该选项将字符串与memcached选项一起使用。 例如，将端口设置为22222

```
[mysqld]
daemon_memcached_option = "-p22222"
```

该选项只能在MySQL配置文件或命令行中设置，因此需要重新启动MySQL才能使更改生效。

如果将新条目添加到容器表或更改现有条目，则需要重新启动memcached插件以使其再次读取定义。 您可以通过重新启动MySQL或卸载并安装插件来做到这一点：

```
mysql> UNINSTALL PLUGIN daemon_memcached;
Query OK, 0 rows affected (4.05 sec)
mysql> INSTALL PLUGIN daemon_memcached soname "libmemcached.so";
Query OK, 0 rows affected (0.02 sec)
```

实际上，您将主要使用应用程序中的插件。 如果您习惯使用Memcached，用法非常简单。 例如，清单27-5显示了一些使用pymemcache模块的Python命令。 请注意，该示例假定您将端口设置回11211。

```
Listing 27-5. Using the InnoDB memcached plugin with Python
shell$ python3
Python 3.6.8 (default, May 16 2019, 05:58:38)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-36.0.1)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from pymemcache.client.base import Client
>>> client = Client(('localhost', 11211))
>>> client.get('@@aaa.AA')
b'HELLO, HELLO'
>>> client.set('@@aaa.BB', 'Hello World')
True
>>> client.get('@@aaa.BB')
b'Hello World'
```

交互式Python环境用于通过memcached插件查询test.demo_test表。 创建连接后，使用get（）方法查询现有行，并使用set（）方法插入新行。 在这种情况下，无需设置超时，因为set（）方法最终直接写入InnoDB。 最后，再次检索新行。 请注意，与需要自己维护缓存的常规Memcached相比，此示例非常简单。

您可以通过在MySQL中查询新行来验证新行是否确实已插入到表中：

```
mysql> SELECT * FROM test.demo_test;
+----+--------------+----+----+----+
| c1 | c2           | c3 | c4 | c5 |
+----+--------------+----+----+----+
| AA | HELLO, HELLO | 8  | 0  | 0  |
| BB | Hello World  | 0  | 1  | 0  |
+----+--------------+----+----+----+
2 rows in set (0.0032 sec)
```

还有更多使用MySQL InnoDB Memcached插件的信息。 如果打算使用它，建议您阅读参考手册中的“ InnoDB memcached插件”部分，网址为https://dev.mysql.com/doc/refman/en/innodb-memcached.html。

另一个支持缓存的流行实用程序是ProxySQL。

## ProxySQL

ProxySQL项目由René Cannaò创建，是一个高级代理，支持负载平衡，基于查询规则的路由，缓存等。 缓存功能基于查询规则进行缓存，例如，您可以设置要缓存具有给定摘要的查询。 缓存将根据您为查询规则设置的生存时间值自动过期。

您可以从https://github.com/sysown/proxysql/releases/下载ProxySQL。 在撰写本文时，最新发行版是版本2.0.8，这是示例中使用的发行版。

------

注意ProxySQL仅在Linux上正式受支持。 有关包括受支持发行版的安装说明的完整文档，请参阅https://github.com/sysown/proxysql/wiki。

------

清单27-6显示了使用ProxySQL GitHub存储库中的RPM在Oracle Linux上安装ProxySQL 2.0.8的示例。 在其他Linux发行版中，使用package命令进行分发的安装过程类似（但是，根据所使用的package命令，输出将有所不同）。 安装完成后，将启动ProxySQL。

```
Listing 27-6. Installing and starting ProxySQL
shell$ wget https://github.com/sysown/proxysql/releases/download/v2.0.8/
proxysql-2.0.8-1-centos7.x86_64.rpm
...
Length: 9340744 (8.9M) [application/octet-stream]
Saving to: 'proxysql-2.0.8-1-centos7.x86_64.rpm'
100%[===========================>] 9,340,744 2.22MB/s in 4.0s
2019-11-24 18:41:34 (2.22 MB/s) - 'proxysql-2.0.8-1-centos7.x86_64.rpm'
saved [9340744/9340744]
shell$ sudo yum install proxysql-2.0.8-1-centos7.x86_64.rpm
Loaded plugins: langpacks, ulninfo
Examining proxysql-2.0.8-1-centos7.x86_64.rpm: proxysql-2.0.8-1.x86_64
Marking proxysql-2.0.8-1-centos7.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package proxysql.x86_64 0:2.0.8-1 will be installed
--> Finished Dependency Resolution
Dependencies Resolved
==============================================================
 Package Arch Version Repository Size
==============================================================
Installing:
 proxysql x86_64 2.0.8-1 /proxysql-2.0.8-1-centos7.x86_64 35 M
Transaction Summary
==============================================================
Install 1 Package
Total size: 35 M
Installed size: 35 M
Is this ok [y/d/N]: y
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
 Installing : proxysql-2.0.8-1.x86_64 1/1
warning: group proxysql does not exist - using root
warning: group proxysql does not exist - using root
Created symlink from /etc/systemd/system/multi-user.target.wants/proxysql.
service to /etc/systemd/system/proxysql.service.
 Verifying : proxysql-2.0.8-1.x86_64 1/1
Installed:
 proxysql.x86_64 0:2.0.8-1
Complete!
shell$ sudo systemctl start proxysql
```

您只能通过ProxySQL的管理界面来对其进行配置。 它使用mysql命令行客户端，并且对MySQL管理员具有熟悉的感觉。 默认情况下，ProxySQL使用端口6032作为管理界面，管理员用户名为admin，密码设置为admin。 清单27-7显示了连接到管理界面并列出可用的架构和表的示例。

```
Listing 27-7. The administration interface
shell$ mysql --host=127.0.0.1 --port=6032 \
 --user=admin --password \
 --default-character-set=utf8mb4 \
 --prompt='ProxySQL> '
Enter password:
Welcome to the MySQL monitor. Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.5.30 (ProxySQL Admin Module)
Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
ProxySQL> SHOW SCHEMAS;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)

ProxySQL> SHOW TABLES;
+--------------------------------------------+
| tables                                     |
+--------------------------------------------+
| global_variables                           |
| mysql_aws_aurora_hostgroups                |
| mysql_collations                           |
| mysql_galera_hostgroups                    |
| mysql_group_replication_hostgroups         |
| mysql_query_rules                          |
| mysql_query_rules_fast_routing             |
| mysql_replication_hostgroups               |
| mysql_servers                              |
| mysql_users                                |
| proxysql_servers                           |
| runtime_checksums_values                   |
| runtime_global_variables                   |
| runtime_mysql_aws_aurora_hostgroups        |
| runtime_mysql_galera_hostgroups            |
| runtime_mysql_group_replication_hostgroups |
| runtime_mysql_query_rules                  |
| runtime_mysql_query_rules_fast_routing     |
| runtime_mysql_replication_hostgroups       |
| runtime_mysql_servers                      |
| runtime_mysql_users                        |
| runtime_proxysql_servers                   |
| runtime_scheduler                          |
| scheduler                                  |
+--------------------------------------------+
24 rows in set (0.00 sec)
```

将表分组在架构中时，您可以直接访问表而无需引用架构。 SHOW TABLES的输出显示主模式中的表，这些表与ProxySQL的配置相关联。

配置是一个分为两个阶段的过程，在此过程中，您首先准备新的配置，然后应用它。 如果您要保留更改并将其加载到运行时线程，则应用更改意味着将其保存到磁盘。

名称中带有runtime_前缀的表用于将配置推送到运行时线程。 配置ProxySQL的一种方法是使用SET语句，类似于在MySQL中设置系统变量，但是您也可以使用UPDATE语句。 第一步应该是更改管理员密码（以及可选的管理员用户名），这可以通过设置admin-admin_credentials变量来完成，如清单27-8所示。

```
Listing 27-8. Setting the password for the administrator account
ProxySQL> SET admin-admin_credentials = 'admin:password';
Query OK, 1 row affected (0.01 sec)
ProxySQL> SAVE ADMIN VARIABLES TO DISK;
Query OK, 32 rows affected (0.02 sec)
ProxySQL> LOAD ADMIN VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)
ProxySQL> SELECT @@admin-admin_credentials;
+---------------------------+
| @@admin-admin_credentials |
+---------------------------+
| admin:password            |
+---------------------------+
1 row in set (0.00 sec)
```

admin-admin_credentials选项的值是用冒号分隔的用户名和密码。 SAVE ADMIN VARIABLES TO DISK语句保留更改，而LOAD ADMIN VARIABLES TO RUNTIME命令将更改应用于运行时线程。 由于性能原因，ProxySQL会在每个线程中保留变量的副本，因此有必要将变量加载到运行时线程中。 您可以查询当前值（无论已应用还是待处理），就像在MySQL中查询系统变量一样。

您配置了MySQL后端实例，ProxySQL可以使用这些后端实例来指导mysql_servers表中的查询。 在此讨论中，将使用与ProxySQL在同一主机上的单个实例。 清单27-9显示了如何将其添加到ProxySQL可以路由到的服务器列表中。

```
Listing 27-9. Adding a MySQL instance to the list of servers
ProxySQL> SHOW CREATE TABLE mysql_servers\G
*************************** 1. row ***************************
 table: mysql_servers
Create Table: CREATE TABLE mysql_servers (
 hostgroup_id INT CHECK (hostgroup_id>=0) NOT NULL DEFAULT 0,
 hostname VARCHAR NOT NULL,
 port INT CHECK (port >= 0 AND port <= 65535) NOT NULL DEFAULT 3306,
 gtid_port INT CHECK (gtid_port <> port AND gtid_port >= 0 AND gtid_port
<= 65535) NOT NULL DEFAULT 0,
 status VARCHAR CHECK (UPPER(status) IN ('ONLINE','SHUNNED','OFFLINE_SOFT',
'OFFLINE_HARD')) NOT NULL DEFAULT 'ONLINE',
 weight INT CHECK (weight >= 0 AND weight <=10000000) NOT NULL DEFAULT 1,
 compression INT CHECK (compression IN(0,1)) NOT NULL DEFAULT 0,
 max_connections INT CHECK (max_connections >=0) NOT NULL DEFAULT 1000,
 max_replication_lag INT CHECK (max_replication_lag >= 0 AND
max_replication_lag <= 126144000) NOT NULL DEFAULT 0,
 use_ssl INT CHECK (use_ssl IN(0,1)) NOT NULL DEFAULT 0,
 max_latency_ms INT UNSIGNED CHECK (max_latency_ms>=0) NOT NULL DEFAULT 0,
 comment VARCHAR NOT NULL DEFAULT ",
 PRIMARY KEY (hostgroup_id, hostname, port) )
1 row in set (0.01 sec)
ProxySQL> INSERT INTO mysql_servers
 (hostname, port, use_ssl)
 VALUES ('127.0.0.1', 3306, 1);
Query OK, 1 row affected (0.01 sec)
ProxySQL> SAVE MYSQL SERVERS TO DISK;
Query OK, 0 rows affected (0.36 sec)
ProxySQL> LOAD MYSQL SERVERS TO RUNTIME;
Query OK, 0 rows affected (0.01 sec)
```

该示例说明如何使用SHOW CREATE TABLE获取有关mysql_servers表的信息。 该表定义包含有关您可以包括的设置和允许的值的信息。 除主机名外，所有设置都具有默认值。 清单的其余部分在localhost端口3306上为MySQL实例插入一行，并要求使用SSL。 然后，更改将保存到磁盘并加载到运行时线程中。

注意SSL只能从ProxySQL到MySQL实例使用，而不能在客户端和ProxySQL之间使用。

您还需要指定哪些用户可以使用该连接。 首先，在MySQL中创建一个用户：

```
mysql> CREATE USER myuser@'127.0.0.1'
 IDENTIFIED WITH mysql_native_password
 BY 'password';
Query OK, 0 rows affected (0.0550 sec)
mysql> GRANT ALL ON world.* TO myuser@'127.0.0.1';
Query OK, 0 rows affected (0.0422 sec)
```

ProxySQL目前不支持caching_sha2_password身份验证插件，这是使用MySQL Shell连接时MySQL 8中的默认身份验证插件（但使用mysql命令行客户端提供支持），因此您需要使用mysql_native_password插件创建用户。 然后在ProxySQL中添加用户：

```
ProxySQL> INSERT INTO mysql_users
 (username,password)
 VALUES ('myuser', 'password');
Query OK, 1 row affected (0.00 sec)
ProxySQL> SAVE MYSQL USERS TO DISK;
Query OK, 0 rows affected (0.06 sec)
ProxySQL> LOAD MYSQL USERS TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)
```

现在，您可以通过ProxySQL连接到MySQL。 默认情况下，SQL接口使用端口6033。除了端口号和可能的主机名以外，您可以通过ProxySQL以与通常相同的方式进行连接：

```
shell$ mysqlsh --user=myuser --password \
 --host=127.0.0.1 --port=6033 \
 --sql --table \
 -e "SELECT * FROM world.city WHERE ID = 130;"
+-----+--------+-------------+-----------------+------------+
| ID  | Name   | CountryCode | District        | Population |
+-----+--------+-------------+-----------------+------------+
| 130 | Sydney | AUS         | New South Wales | 3276207    |
+-----+--------+-------------+-----------------+------------+

```

ProxySQL以类似于性能模式的方式收集统计信息。 您可以在stats_mysql_query_digest和stats_mysql_query_digest_重置表中查询统计信息。 这两个表之间的区别在于，自上次查询该表以来，后者仅包含摘要。 例如，要按查询的总执行时间对其进行排序

```
ProxySQL> SELECT count_star, sum_time,
 digest, digest_text
 FROM stats_mysql_query_digest_reset
 ORDER BY sum_time DESC\G
*************************** 1. row ***************************
 count_star: 1
 sum_time: 577149
 digest: 0x170E9EDDB525D570
digest_text: select @@sql_mode;
*************************** 2. row ***************************
 count_star: 1
 sum_time: 5795
 digest: 0x94656E0AA2C6D499
digest_text: SELECT * FROM world.city WHERE ID = ?
2 rows in set (0.01 sec)
```

如果看到要缓存其结果的查询，则可以基于查询的摘要添加查询规则。 假设您要缓存通过ID（摘要0x94656E0AA2C6D499）查询world.city表的结果，则可以添加如下规则：

```
ProxySQL> INSERT INTO mysql_query_rules
 (active, digest, cache_ttl, apply)
 VALUES (1, '0x94656E0AA2C6D499', 60000, 1);
Query OK, 1 row affected (0.01 sec)
ProxySQL> SAVE MYSQL QUERY RULES TO DISK;
Query OK, 0 rows affected (0.09 sec)
ProxySQL> LOAD MYSQL QUERY RULES TO RUNTIME;
Query OK, 0 rows affected (0.01 sec)
```

活动列指定在评估可以使用的规则时，ProxySQL是否应考虑该规则。 摘要是要缓存的查询的摘要，cache_ttl指定在认为结果过期之前应使用毫秒数，然后刷新结果。 生存时间已设置为60000毫秒（1分钟），以便您有时间在使缓存无效之前执行几次查询。 设置为1表示当查询与此规则匹配时，不会评估更高的规则。

如果在一分钟内几次执行查询，则可以在表stats_mysql_global中查询缓存统计信息，以了解如何使用缓存。 输出的示例是

```
ProxySQL> SELECT *
 FROM stats_mysql_global
 WHERE Variable_Name LIKE 'Query_Cache%';
+--------------------------+----------------+
| Variable_Name            | Variable_Value |
+--------------------------+----------------+
| Query_Cache_Memory_bytes | 3659           |
| Query_Cache_count_GET    | 6              |
| Query_Cache_count_GET_OK | 5              |
| Query_Cache_count_SET    | 1              |
| Query_Cache_bytes_IN     | 331            |
| Query_Cache_bytes_OUT    | 1655           |
| Query_Cache_Purged       | 0              |
| Query_Cache_Entries      | 1              |
+--------------------------+----------------+
8 rows in set (0.01 sec)
```

您的数据很可能会有所不同。 它显示高速缓存使用3659字节，并且对高速缓存进行了六个查询，在其中五种情况下，结果是从高速缓存返回的。 六个查询中的最后一个要求对MySQL后端执行查询。

您可以设置两个选项来配置缓存。 这些是

- mysql-query_cache_size_MB：缓存的最大大小，以兆字节为单位。 这是清除线程使用的软限制，用于确定要从缓存清除的查询数量。 因此，内存使用量可能暂时大于配置的大小。 默认值为256。

- mysql-query_cache_stores_empty_result：是否缓存没有行的结果集。 默认值为true。 也可以在查询规则表中针对每个查询进行配置。

您更改配置的方式类似于之前更改管理员密码的方式。 例如，将查询缓存限制为128 MB

```
ProxySQL> SET mysql-query_cache_size_MB = 128;
Query OK, 1 row affected (0.00 sec)
ProxySQL> SAVE MYSQL VARIABLES TO DISK;
Query OK, 121 rows affected (0.04 sec)
ProxySQL> LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)
```

这首先准备配置更改，然后将其保存到磁盘，最后将MySQL变量加载到运行时线程中。

如果要使用ProxySQL，建议您查阅ProxySQL GitHub项目上的Wiki，网址为https://github.com/sysown/proxysql/wiki。

## Caching Tips

如果您决定为MySQL实例实施缓存，则需要考虑一些事项。本节研究一些常规的缓存技巧。

最重要的考虑因素是要缓存的内容。本章前面的缓存单行主键查找结果的示例并不是一个受益于缓存的查询类型的好例子。通常，查询越复杂和昂贵，并且执行查询的频率越高，则查询越好。使缓存更有效的一件事是将复杂的查询分成较小的部分。这样，您可以分别缓存复杂查询的每个部分的结果，这使得它更有可能被重用。

您还应该考虑查询返回多少数据。如果查询返回一个较大的结果集，您可能最终会使用所有可用于单个查询的缓存的内存。

另一个考虑因素是在哪里拥有缓存。您可以将缓存放置到应用程序中越近，它就越有效，因为它减少了花费在网络通信上的时间。缺点是，如果您有多个应用程序实例，则必须在复制缓存和拥有远程共享缓存之间进行选择。例外是，如果您需要将缓存的数据与其他MySQL表一起使用。在这种情况下，最好以缓存表或类似的形式将缓存保留在MySQL中。

## 总结

本章概述了使用MySQL进行缓存。它开始描述了如何在从CPU内部到专用缓存过程的各处找到缓存。然后讨论了如何使用缓存表和直方图在MySQL内部进行缓存。

讨论了使用Memcached和ProxySQL进行缓存的两个主要部分。 Memcached是可以在应用程序中使用的内存中键值存储，也可以使用MySQL附带的特殊版本，该特殊版本允许您直接与InnoDB进行交互。 ProxySQL结合了路由器和缓存机制，可根据您定义的查询规则透明地存储结果集。

最后，介绍了有关缓存的一些注意事项。执行查询的频率越高，执行查询的成本就越高，则从缓存中获得的收益就越大。第二个考虑因素是，您可以将缓存放置到应用程序中越近越好。

总结了MySQL 8查询性能调优过程的最后一章。希望这是一段有意义的旅程，并且您随时准备在工作中使用这些工具和技术。请记住，您练习查询调整的次数越多，您就越能从中受益。快乐的查询调优。