# DDL 和批量数据负载

不时地执行架构更改或将大量数据导入表中。这可能是为了适应新功能、还原备份、导入由第三方进程生成的数据或类似功能。虽然原始磁盘写入性能自然非常重要，但您也可以在 MySQL 端执行几项操作来提高这些操作的性能。

本章开始讨论架构更改，然后继续讨论有关加载数据的一些一般注意事项。当您一次插入单行时，这些注意事项也适用。本章的其余部分介绍如何通过按主键顺序插入来提高数据加载性能，缓冲池和辅助索引如何影响性能、配置和调整语句本身。最后，演示了 MySQL Shell的并行导入功能。

## 架构更改

当您需要对架构执行更改时，存储引擎可能需要大量工作，这可能涉及制作一个全新的表副本。本节将介绍您可以执行哪些操作来加快此过程，从支持架构更改的算法开始，然后是其他注意事项（如配置）。

### 算法

MySQL 支持 ALTER 的多种算法，该算法决定如何执行架构更改。某些架构更改可以通过更改表定义"立即"进行，而在频谱的另一端，某些更改需要将整个表复制到新表中。

按所需量的顺序，算法为

- 对表定义进行更改。虽然变化不是即时的，但它非常快。INSTANT算法在 MySQL 8.0.12 及更晚版本中可用。
- 一般在现有表空间文件中进行（表空间 ID 不更改），但除了一些由），这更像是 COPY，但允许并发数据更改。这可能是一个相对便宜的操作，但也可能涉及复制所有数据。
- 将复制到新的表空间文件。这是影响最大的算法，因为它通常需要更多的锁，导致更多的 I/O，并且需要更长的时间。

通常算法允许并发数据更改，从而减少对其他连接的影响，而至少需要读取锁。MySQL 会根据请求的更改选择影响最少的算法，但您也可以显式请求特定算法。例如，如果您希望确保 MySQL 不继续更改，如果不支持您选择的算法，这非常有用。使用 算法关键字指定，例如：

```
mysql> ALTER TABLE world.city
 ADD COLUMN Council varchar(50),
 ALGORITHM=INSTANT;
If the change cannot be performed using the requested algorithm, the statement fails
with an ER_ALTER_OPERATION_NOT_SUPPORTED error (error number 1845), for example:
mysql> ALTER TABLE world.city
 DROP COLUMN Council,
 ALGORITHM=INSTANT;
ERROR: 1845: ALGORITHM=INSTANT is not supported for this operation. Try
ALGORITHM=COPY/INPLACE.
```



如果可以使用 INSTANT 算法显然将获得最佳的 ALTER 性能。在编写本文时，允许使用 INSTANT 算法执行操作：

- 将新列添加为表中的最后一列。
- 添加生成的虚拟列。
- 删除生成的虚拟列。
- 为现有列设置默认值。
- 删除现有列的默认值。
- 更改使用列或设置数据类型的列表。要求是列的存储大小不会更改。
- 更改是否为现有索引显式设置了索引类型（例如

还有一些是需要注意的：

- 行格式不能为。
- 该表不能具有全文索引。
- 不支持临时表。
- 数据字典中的表不能使用 INSTANT 算法。

性能明智，就地更改通常（但并不总是）比复制更改更快。此外，当架构更改联机时必须跟踪在执行架构更改期间所做的更改。 这会增加开销，并且需要一段时间来应用在操作结束时的架构更改期间所做的更改。如果能够在表上获取共享或独占锁通常可以比允许并发更改获得更好的性能。

### 其他注意事项

由于就地或复制非常密集，因此对性能的最大影响是磁盘的速度以及架构更改期间有多少其他写入活动。这意味着，从性能角度来看，最好选择在实例和主机上几乎没有其他写入活动时执行需要复制或移动大量数据的架构更改。这包括备份本身可能非常 I/O 密集型备份。

如果创建或重建辅助索引（包括 OPTIMIZE 和其他重建表的语句选项指定每个排序缓冲区可以使用的内存量。请注意，单个将创建多个缓冲区，因此请注意不要将值设置得太大。默认值为 1 MiB，允许的最大值为 64 MiB。在某些情况下，较大的缓冲区可能会提高性能。

创建全文索引时，可以使用选项指定 InnoDB 用于生成搜索索引的线程数。默认值为 2，支持的值介于 1 和 32 之间。如果要在大型表上创建全文索引，则增加其值可能是一

需要考虑的一个特殊的 DDL 操作是删除或截断表。

### 删除或截断表

似乎没有必要考虑删除表的性能。似乎所有必需的只是删除表空间文件并删除对表的引用。实际上，它并不太简单。

删除或截断表时的主要问题是缓冲区池中对表数据的所有引用。特别是，自适应哈希索引可能会导致问题。因此，您可以通过在操作期间禁用自适应哈希索引，从而在删除或截断大型表时显著提高性能，例如：

```
mysql> SET GLOBAL innodb_adaptive_hash_index = OFF;
Query OK, 0 rows affected (0.1008 sec)
mysql> DROP TABLE <name of large table>;
mysql> SET GLOBAL innodb_adaptive_hash_index = ON;
Query OK, 0 rows affected (0.0098 sec)
```



禁用自适应哈希索引将使从哈希索引运行中受益的查询运行速度变慢，但对于大小为几百千兆字节或更大的表，禁用自适应哈希索引的减速速度相对较小，通常优先于发生潜在停滞的情况，因为删除对正在删除或截断的表的引用的开销。

执行架构更改的讨论到此结束。本章的其余部分将讨论加载数据。

## 一般数据加载注意事项

在讨论如何提高批量插入的性能之前，值得进行一个小测试并讨论结果。在测试中， 200 ， 000 行入到两个表中。其中一个表具有自动递增计数器作为主键，另一个表对主键使用随机整数。两个表的行大小相同。

数据加载完成后，清单中的脚本可用于确定表空间文件中按日志序列号（LSN） 衡量的每个页面的年龄。日志序列号越高，页面修改得越新。此脚本的灵感来自innodb_ruby科尔，并生成了类似于地图。但是，innodb_ruby不支持 MySQL 8，因此开发了单独的 Python 程序。该程序已经通过 Python 2.7 （Linux） 和 3.6 （Linux 和微软 Windows） 进行了测试。在本书的 GitHub 存储库listing_25_1.py 中的文件中也提供它。

```
Listing 25-1. Python program to map the LSN age of InnoDB pages
'''Read a MySQL 8 file-per-table tablespace file and generate an
SVG formatted map of the LSN age of each page.
Invoke with the --help argument to see a list of arguments and
Usage instructions.'''
import sys
import argparse
import math
from struct import unpack
# Some constants from InnoDB
FIL_PAGE_OFFSET = 4 # Offset for the page number
FIL_PAGE_LSN = 16 # Offset for the LSN
FIL_PAGE_TYPE = 24 # Offset for the page type
FIL_PAGE_TYPE_ALLOCATED = 0 # Freshly allocated page
def mach_read_from_2(page, offset):
 '''Read 2 bytes in big endian. Based on the function of the same
 name in the InnoDB source code.'''
 return unpack('>H', page[offset:offset + 2])[0]
1
https://github.com/jeremycole/innodb_ruby
def mach_read_from_4(page, offset):
 '''Read 4 bytes in big endian. Based on the function of the same
 name in the InnoDB source code.'''
 return unpack('>L', page[offset:offset + 4])[0]
def mach_read_from_8(page, offset):
 '''Read 8 bytes in big endian. Based on the function of the same
 name in the InnoDB source code.'''
 return unpack('>Q', page[offset:offset + 8])[0]
def get_color(lsn, delta_lsn, greyscale):
 '''Get the RGB color of a relative lsn.'''
 color_fmt = '#{0:02x}{1:02x}{2:02x}'
 if greyscale:
 value = int(255 * lsn / delta_lsn)
 color = color_fmt.format(value, value, value)
 else:
 # 0000FF -> 00FF00 -> FF0000 -> FFFF00
 # 256 + 256 + 256 values
 value = int((3 * 256 - 1) * lsn / delta_lsn)
 if value < 256:
 color = color_fmt.format(0, value, 255 - value)
 elif value < 512:
 value = value % 256
 color = color_fmt.format(value, 255 - value, 0)
 else:
 value = value % 256
 color = color_fmt.format(255, value, 0)
 return color
def gen_svg(min_lsn, max_lsn, lsn_age, args):
 '''Generate an SVG output and print to stdout.'''
 pages_per_row = args.width
 page_width = args.size
 num_pages = len(lsn_age)
 num_rows = int(math.ceil(1.0 * num_pages / pages_per_row))
 x1_label = 5 * page_width + 1
 x2_label = (pages_per_row + 7) * page_width
 delta_lsn = max_lsn - min_lsn
 print('<?xml version="1.0"?>')
 print('<svg xmlns="http://www.w3.org/2000/svg" version="1.1">')
 print('<text x="{0}" y="{1}" font-family="monospace" font-size="{2}" '
 .format(x1_label, int(1.5 * page_width) + 1, page_width) +
 'font-weight="bold" text-anchor="end">Page</text>')
 page_number = 0
 page_fmt = ' <rect x="{0}" y="{1}" width="{2}" height="{2}" fill="{3}" />'
 label_fmt = ' <text x="{0}" y="{1}" font-family="monospace" '
 label_fmt += 'font-size="{2}" text-anchor="{3}">{4}</text>'
 for i in range(num_rows):
 y = (i + 2) * page_width
 for j in range(pages_per_row):
 x = 6 * page_width + j * page_width
 if page_number >= len(lsn_age) or lsn_age[page_number] is None:
 color = 'black'
 else:
 relative_lsn = lsn_age[page_number] - min_lsn
 color = get_color(relative_lsn, delta_lsn, args.greyscale)
 print(page_fmt.format(x, y, page_width, color))
 page_number += 1
 y_label = y + page_width
 label1 = i * pages_per_row
 label2 = (i + 1) * pages_per_row
 print(label_fmt.format(x1_label, y_label, page_width, 'end', label1))
 print(label_fmt.format(x2_label, y_label, page_width, 'start', label2))
 # Create a frame around the pages
 frame_fmt = ' <path stroke="black" stroke-width="1" fill="none" d="'
 frame_fmt += 'M{0},{1} L{2},{1} S{3},{1} {3},{4} L{3},{5} S{3},{6} {2},{6}'
 frame_fmt += ' L{0},{6} S{7},{6} {7},{5} L{7},{4} S{7},{1} {0},{1} Z" />'
 x1 = int(page_width * 6.5)
 y1 = int(page_width * 1.5)
 x2 = int(page_width * 5.5) + page_width * pages_per_row
 x2b = x2 + page_width
 y1b = y1 + page_width
 y2 = int(page_width * (1.5 + num_rows))
 y2b = y2 + page_width
 x1c = x1 - page_width
 print(frame_fmt.format(x1, y1, x2, x2b, y1b, y2, y2b, x1c))
 # Create legend
 x_left = 6 * page_width
 x_right = x_left + pages_per_row * page_width
 x_mid = x_left + int((x_right - x_left) * 0.5)
 y = y2b + 2 * page_width
 print('<text x="{0}" y="{1}" font-family="monospace" '.format(x_left, y) +
 'font-size="{0}" text-anchor="start">{1}</text>'.format(page_width,
 min_lsn))
 print('<text x="{0}" y="{1}" font-family="monospace" '.format(x_right, y) +
 'font-size="{0}" text-anchor="end">{1}</text>'.format(page_width,
 max_lsn))
 print('<text x="{0}" y="{1}" font-family="monospace" '.format(x_mid, y) +
 'font-size="{0}" font-weight="bold" text-anchor="middle">{1}</text>'
 .format(page_width, 'LSN Age'))
 color_width = 1
 color_steps = page_width * pages_per_row
 y = y + int(page_width * 0.5)
 for i in range(color_steps):
 x = 6 * page_width + i * color_width
 color = get_color(i, color_steps, args.greyscale)
 print('<rect x="{0}" y="{1}" width="{2}" height="{3}" fill="{4}" />'
 .format(x, y, color_width, page_width, color))
 print('</svg>')
def analyze_lsn_age(args):
 '''Read the tablespace file and find the LSN for each page.'''
 page_size_bytes = int(args.page_size[0:-1]) * 1024
 min_lsn = None
 max_lsn = None
 lsn_age = []
 with open(args.tablespace, 'rb') as fs:
 # Read at most 1000 pages at a time to avoid storing too much
 # in memory at a time.
 chunk = fs.read(1000 * page_size_bytes)
 while len(chunk) > 0:
 num_pages = int(math.floor(len(chunk) / page_size_bytes))
 for i in range(num_pages):
 # offset is the start of the page inside the
 # chunk of data
 offset = i * page_size_bytes
 # The page number, lsn for the page, and page
 # type can be found at the FIL_PAGE_OFFSET,
 # FIL_PAGE_LSN, and FIL_PAGE_TYPE offsets
 # relative to the start of the page.
 page_number = mach_read_from_4(chunk, offset + FIL_PAGE_OFFSET)
 page_lsn = mach_read_from_8(chunk, offset + FIL_PAGE_LSN)
 page_type = mach_read_from_2(chunk, offset + FIL_PAGE_TYPE)
 if page_type == FIL_PAGE_TYPE_ALLOCATED:
 # The page has not been used yet
 continue
 if min_lsn is None:
 min_lsn = page_lsn
 max_lsn = page_lsn
 else:
 min_lsn = min(min_lsn, page_lsn)
 max_lsn = max(max_lsn, page_lsn)
 if page_number == len(lsn_age):
 lsn_age.append(page_lsn)
 elif page_number > len(lsn_age):
 # The page number is out of order - expand the list first
 lsn_age += [None] * (page_number - len(lsn_age))
 lsn_age.append(page_lsn)
 else:
 lsn_age[page_number] = page_lsn
 chunk = fs.read(1000 * page_size_bytes)
 sys.stderr.write("Total # Pages ...: {0}\n".format(len(lsn_age)))
 gen_svg(min_lsn, max_lsn, lsn_age, args)
def main():
 '''Parse the arguments and call the analyze_lsn_age()
 function to perform the analysis.'''
 parser = argparse.ArgumentParser(
 prog='listing_25_1.py',
 description='Generate an SVG map with the LSN age for each page in an' +
 ' InnoDB tablespace file. The SVG is printed to stdout.')
 parser.add_argument(
 '-g', '--grey', '--greyscale', default=False,
 dest='greyscale', action='store_true',
 help='Print the LSN age map in greyscale.')
 parser.add_argument(
 '-p', '--page_size', '--page-size', default='16k',
 dest='page_size',
 choices=['4k', '8k', '16k', '32k', '64k'],
 help='The InnoDB page size. Defaults to 16k.')
 parser.add_argument(
 '-s', '--size', default=16, dest='size',
 choices=[4, 8, 12, 16, 20, 24], type=int,
 help='The size of the square representing a page in the output. ' +
 'Defaults to 16.')
 parser.add_argument(
 '-w', '--width', default=64, dest='width',
 type=int,
 help='The number of pages to include per row in the output. ' +
 'The default is 64.')
 parser.add_argument(
 dest='tablespace',
 help='The tablespace file to analyze.')
 args = parser.parse_args()
 analyze_lsn_age(args)
if __name__ == '__main__':
 main()
```



页号、日志序列号和页类型按每个页面定义的"FIL_PAGE_OFFSET、FIL_PAGE_LSN和量以字节为单位） 提取。如果具有 FIL_PAGE_TYPE_ALLOCATED 常，则意味着尚未使用，因此可以跳过该页 - 这些页面在日志序列号映射中为黑色。

您可以通过使用 --help 参数调用程序来帮助。唯一需要的参数是要分析的表空间文件的路径。除非将 innodb_page_size 选项设置为字节以外的其他内容，否则除非您想要更改生成的地图的尺寸和大小，否则所有需要的可选参数的默认值都是您所需要的。

现在，您可以生成测试表。清单显示了表的创建方式。这是具有自动递增主键的表。

```
Listing 25-2. Populating a table with an auto-incrementing primary key
mysql-sql> CREATE SCHEMA chapter_25;
Query OK, 1 row affected (0.0020 sec)
mysql-sql> CREATE TABLE chapter_25.table_autoinc (
 id bigint unsigned NOT NULL auto_increment,
 val varchar(36),
 PRIMARY KEY (id)
 );
Query OK, 0 rows affected (0.3382 sec)
mysql-sql> \py
Switching to Python mode...
mysql-py> for i in range(40):
 session.start_transaction()
 for j in range(5000):
 session.run_sql("INSERT INTO chapter_25.table_autoinc
(val) VALUES (UUID())")
 session.commit()
Query OK, 0 rows affected (0.1551 sec)
```



该表有一主键和填充了 UUID 以创建一些随机数据。MySQL 外壳的 Python 语言模式用于插入数据。方法在版本 8.0.17 及更晚版本中提供。最后，您可以执行脚本，以可扩展矢量图形 （SVG） 格式生成表空间年龄图：

```
shell> python listing_25_1.py <path to datadir>\chapter_25\table_autoinc.
ibd > table_autoinc.svg
Total # Pages ...: 880
```



程序的输出显示表空间中有 880 页，文件末尾可能有一些未使用的页面。

图显示了表的日志

![](../附图/Figure%2025-1.png)

在图中，左上角表示表空间的第一页。当您从左到右和从上到下浏览图形时，页面会进一步进入表空间文件，右下角表示最后一页。该图显示，除第一页页，页面的年龄模式遵循与图底部的 LSN 年龄比例相同的模式。这意味着，当您浏览表空间时，页面的年龄会变年轻。前几个页面是例外，例如，它们包括表空间标头。

此模式显示数据按顺序插入到表空间中，使其尽可能紧凑。它还使得查询尽可能可能从按顺序逻辑读取多个页面中的数据，那么它们也是表空间文件中的物理顺序。

那么， 如果按随机顺序插入， 它看起来如何？随机顺序插入的常见示例是 UUID 作为主键，但为了确保两个表的行大小相同，改为使用随机整数。清单显示了的填充。

```
Listing 25-3. Populating a table with a random primary key
mysql-py> \sql
Switching to SQL mode... Commands end with ;
mysql-sql> CREATE TABLE chapter_25.table_random (
 id bigint unsigned NOT NULL,
 val varchar(36),
 PRIMARY KEY (id)
 );
Query OK, 0 rows affected (0.0903 sec)
mysql-sql> \py
Switching to Python mode...
mysql-py> import random
mysql-py> import math
mysql-py> maxint = math.pow(2, 64) - 1
mysql-py> random.seed(42)
mysql-py> for i in range(40):
 session.start_transaction()
 for j in range(5000):
 session.run_sql("INSERT INTO chapter_25.table_random
VALUE ({0}, UUID())".format(random.randint(0, maxint)))
 session.commit()
Query OK, 0 rows affected (0.0185 sec)
```



Python模块用于生成 64 位随机无符号整数。种子被显式设置，因为它（通过实验）知道 42 的种子连续生成 200，000 个不同的数字，因此不会发生重复的密钥错误。填充表时，执行脚本：

```
shell> python listing_25_1.py <path to datadir>\chapter_25\table_random.ibd
> table_random.svg
Total # Pages ...: 1345
```



脚本的输出显示，此表空间有 1345 页。生成的年龄图如图。

![](../附图/Figure%2025-2.png)

这一次，日志序列号年龄模式完全不同。除未使用的页面外，所有页面的年龄颜色与最新日志序列号的颜色相对应。这意味着所有包含数据的页面都大约在同一时间更新，换句话说，它们都写入到批量加载结束之前。与使用自动递增主键的表中使用的 880 页相比，包含数据的页数为 1345。页面数超过 50%。

以随机顺序插入的原因是 InnoDB 在插入数据时填充了页面。当数据按顺序主，这意味着下一行将始终连续为上一行，因此当按主键顺序排序行时，这工作得很好。图。

![](../附图/Figure%2025-3.png)

图中显示了正在插入的两个新行。id = 1005 的行可以完全适合页面 N，因此当插入 id = 1006 的行时，它将插入到下一页中。在这种情况下，一切都很好，很紧凑。

当行以随机顺序到达时，有时需要将行插入已满的页面中，以便没有新行的空间。在这种情况下，InnoDB 将现有页面一分为二，在两个页面拆分后的每一页中，原始页面的数据，因此有新行的空间。如图。

![](../附图/Figure%2025-4.png)

在这种情况下，插入了 id = 3500 的行，但在逻辑上属于它的页面 N 中不再有空间。因此，第 N 页被拆分为第 N 页和 N+1 页，每个页面中大约有一半的数据。

页面拆分有两个直接后果。首先，以前占用一个页面的数据现在使用两个页面，这就是为什么以随机顺序插入最终占用 50% 以上的页面，这也意味着相同的数据需要更多的空间在缓冲池中。附加页面的一个重要副作用是，B 树索引最终包含更多的叶页和树中可能更多的级别，并且鉴于树中的每个级别意味着在访问该页时有额外的寻线，这会导致额外的 I/O。

其次，以前一起读取到内存中的行现在位于磁盘上不同位置的两个页面中。当 InnoDB 增加表空间文件的大小时，它通过分配一个新的范围，当页面大小为 16 KiB 或更少时，分配 1 MiB 的新范围。这有助于使磁盘 I/O 更具顺序性（新范围获得磁盘上的连续扇区）。页面多，页面不仅在一个范围内，而且分布在多个范围内，导致随机磁盘 I/O 的分布量也越大。当由于页面拆分而创建新页面时，它很可能位于磁盘的完全不同的部分，因此在读取，随机 I/O 的数量会增加。图。

![](../附图/Figure%2025-5.png)

在图中描绘了三个范围。为简单起见，每个扩展区只显示五个页面（默认页面大小为 16 KiB，每个扩展区有 64 页）。已参与页面拆分的页面将突出显示。第 11 页被拆分时，唯一的后期页面是第 13 页，因此第 11 页和第 12 页仍然相对接近。但是，当创建了几个额外的页面时，第 15 页被拆分，这意味着第 16 页最终出现在下一个范围。

更深占用缓冲池中空间的页面更多以及更随机的 I/O 的组合意味着以随机主键顺序插入行的表的性能将不如以主键顺序插入数据的等效表好。性能差异不仅适用于插入数据;还适用于插入数据。它还适用于数据的后续使用。因此，以主键顺序插入数据对于最佳性能非常重要。接下来将讨论如何实现这一目标。

## 插入主键顺序

如前面的讨论所显示，以主键顺序插入数据具有很大的优势。实现此目的的最简单方法是使用无符号整数并声明列以自动递增来自动生成主键值。或者，您需要确保数据以主键顺序插入。本节将调查这两个案例。

### 自动递增主键

确保将数据插入键顺序的最简单方法是允许 MySQL 使用自动递增主键来分配值本身。通过在创建表键列指定属性，可以做到这一点。也可以使用自动递增列与多列主键连接;在这种情况下，自动递增列必须是索引中的第一列。

清单显示了创建两个表的示例，这些表使用自动递增列以主键顺序插入数据。

```
Listing 25-4. Creating tables with an auto-increment primary key
mysql> \sql
Switching to SQL mode... Commands end with ;
mysql> DROP SCHEMA IF EXISTS chapter_25;
Query OK, 0 rows affected, 1 warning (0.0456 sec)
mysql> CREATE SCHEMA chapter_25;
Query OK, 1 row affected (0.1122 sec)
mysql> CREATE TABLE chapter_25.t1 (
 id int unsigned NOT NULL auto_increment,
 val varchar(10),
 PRIMARY KEY (id)
 );
Query OK, 0 rows affected (0.4018 sec)
mysql> CREATE TABLE chapter_25.t2 (
 id int unsigned NOT NULL auto_increment,
 CreatedDate datetime NOT NULL
 DEFAULT CURRENT_TIMESTAMP(),
 val varchar(10),
 PRIMARY KEY (id, CreatedDate)
 );
Query OK, 0 rows affected (0.3422 sec)
```



表只有主键的单个列，该值是自动递增的。使用无符号整数而不是符号整数的原因是自动递增值始终大于 0，因此在用尽可用值之前，使用无符号整数允许两倍于数值。这些示例使用 4 字节整数，如果使用所有值，则允许的行数小于 43 亿。如果这还不够，您可以将列声明为 bigint 无8 个字节并允许 1.8E19 行。

表将添加到主键中，例如，如果要在创建行时进行分区，该键可能很有用。自动递增仍可确保使用唯一主键创建键中的第一个，因此即使主键中的后续列本质上是随机的，行仍将按主键顺序插入。

使用自动递增主sys 架构中的递增值的使用，并监视是否有任何表接近耗尽其值。清单显示了输出。

```
Listing 25-5. Using the sys.schema_auto_increment_columns view
mysql> SELECT *
 FROM sys.schema_auto_increment_columns
 WHERE table_schema = 'sakila'
 AND table_name = 'payment'\G
*************************** 1. row ***************************
 table_schema: sakila
 table_name: payment
 column_name: payment_id
 data_type: smallint
 column_type: smallint(5) unsigned
 is_signed: 0
 is_unsigned: 1
 max_value: 65535
 auto_increment: 16049
auto_increment_ratio: 0.2449
1 row in set (0.0024 sec)
```



从输出中可以看到，该表对的自动递增值使用小未。下一个自动增量值为 16049，因此使用了 24.49% 的可用值。

如果从外部源插入数据您可能已经为主键列分配了值（即使使用自动递增主键）。让我们看看在这种情况下你可以做什么。

### 插入现有数据

无论您是需要插入生成的数据、还原备份还是使用不同的存储引擎转换表，最好在插入表之前确保表按主键顺序排列。如果生成数据或数据已存在，则可以在插入数据之前考虑对数据进行排序。或者，使用 语句在导入完成后重新生成表。

重建的一个示例是

```
mysql> OPTIMIZE TABLE chapter_25.t1\G
*************************** 1. row ***************************
 Table: chapter_25.t1
 Op: optimize
Msg_type: note
Msg_text: Table does not support optimize, doing recreate + analyze instead
*************************** 2. row ***************************
 Table: chapter_25.t1
 Op: optimize
Msg_type: status
Msg_text: OK
2 rows in set (0.6265 sec)
```



对于大型表，重建可能需要大量时间，但该过程是联机的，但启动和结束时需要锁以确保一致性的持续时间很短。

如果使用程序创建备份使添加包含主键中的列的子句没有等效选项）。如果使用使用所谓的堆组织数据（如 MyISAM）的存储引擎创建表，以便还原到 InnoDB 表（使用数据的索引组织），这尤其有用。

如果将数据从一个复制到另一个表，可以使用相同的原则。清单显示了将 world 的行

```
Listing 25-6. Ordering data by the primary key when copying it
mysql> CREATE TABLE world.city_new
 LIKE world.city;
Query OK, 0 rows affected (0.8607 sec)
mysql> INSERT INTO world.city_new
 SELECT *
 FROM world.city
 ORDER BY ID;
Query OK, 4079 rows affected (2.0879 sec)
Records: 4079 Duplicates: 0 Warnings: 0
```



作为最后一个案例，请考虑何时将 UUID 作为主键。

### UUID 主键

例如，如果主键的因为无法更改应用程序以支持自动递增主键，则可以通过交换 UUID 组件并将其存储在二进制列中来提高性能。

使用 UUID 版本 1）包括时间戳和序列号（如果时间戳向后移动（例如，在夏令时更改期间）和 MAC 地址，则保证唯一性。

时间戳是一个 60 位值，使用 UTC 进行 UTC 自 1582 年 10 月 15 日午夜（当公历投入使用时）以来的 100 纳秒间隔数。它分为三个部分，第一部分和最后最重要的部分最不重要。（时间戳的高字段还包括位。UUID 的组件也如图。

![](../附图/Figure%2025-6.png)

时间戳的低部分表示高达 4，294，967，295 （0xffff） 的间隔 100 纳秒或不到 430 秒。这意味着每 7 分钟和 10 秒以下一点，时间戳的低部分滚动，使 UUID 从排序角度重新开始。这就是为什么普通 UUID 不能很好地用于索引组织的数据，因为这意味着插入将在很大程度上位于主键树中的随机位置。

MySQL 8 包括两个用于操作 UUID 的新功能，使其更适合作为 InnoDB 中的主要 。这些函数将 UUID 从十六进制表示形式分别转换为二进制表示和背面。他们接受相同的两个参数：要转换的 UUID 值以及是否交换时间戳的低点和高部分。清单显示了一个插入数据并使用函数检索数据的示例。

```
Listing 25-7. Using the UUID_TO_BIN() and BIN_TO_UUID() functions
mysql> CREATE TABLE chapter_25.t3 (
 id binary(16) NOT NULL,
 val varchar(10),
 PRIMARY KEY (id)
 );
Query OK, 0 rows affected (0.4413 sec)
mysql> INSERT INTO chapter_25.t3
 VALUES (UUID_TO_BIN(
 '14614d6e-b5a8-11e9-ae6e-080027b7c106',
 TRUE
 ), 'abc');
Query OK, 1 row affected (0.2166 sec)
mysql> SELECT BIN_TO_UUID(id, TRUE) AS id, val
 FROM chapter_25.t3\G
*************************** 1. row ***************************
 id: 14614d6e-b5a8-11e9-ae6e-080027b7c106
val: abc
1 row in set (0.0004 sec)
```



这种方法的优点是双重的。由于具有交换的低时间和高时间组件，因此它变得单调增加，使其更适合索引组织的行。二进制存储意味着 UUID 只需要 16 字节的存储，而不是十六进制版本中的 36 字节，并具有破折号来分隔 UUID 的各个部分。请记住，由于数据由主键组织，因此主键将添加到辅助索引，以便从索引转到行，因此存储主键所需的字节越少，辅助索引越小。

## InnoDB 缓冲池和辅助索引

批量数据加载性能最重要的因素是。本节讨论为什么缓冲池对批量数据加载很重要。

将数据插入表时，InnoDB 需要能够将数据存储在缓冲池中，直到数据写入表空间文件。在缓冲池中存储的数据越多，InnoDB 可以越有效地执行将脏页刷新到表空间文件。但是，还有第二个原因就是维护辅助索引。

插入数据时需要维护辅助索引，但辅助索引的排序顺序与主键的顺序不一样，因此在插入数据时将不断重新排列它们。只要索引可以在内存中保持，插入速率可以保持高，但当索引不再适合缓冲池时，它们的成本会突然增加，插入速率也会显著降低。图说明了性能如何取决于缓冲辅助索引的可用性。

![](../附图/Figure%2025-7.png)

该图显示了插入速率在一段时间内大致保持不变，在此期间，越来越多的缓冲池用于辅助索引。当无法将更多索引存储在缓冲池中时，插入速率会突然下降。在将数据加载到包含包含整个行且没有别的情况下的表中的极端情况下，当辅助索引使用接近一半的缓冲池（其余为主键）时，将下降。

您可以使用以确定索引在缓冲池中使用的空间。例如，要查找缓冲区池中使用的内存量请按表上的"国家代码查找

```
mysql> SELECT COUNT(*) AS NumPages,
 IFNULL(SUM(DATA_SIZE), 0) AS DataSize,
 IFNULL(SUM(IF(COMPRESSED_SIZE = 0,
 @@global.innodb_page_size,
COMPRESSED_SIZE
 )
 ),
 0
 ) AS CompressedSize
 FROM information_schema.INNODB_BUFFER_PAGE
 WHERE TABLE_NAME = '`world`.`city`'
 AND INDEX_NAME = 'CountryCode';
+----------+----------+----------------+
| NumPages | DataSize | CompressedSize |
+----------+----------+----------------+
| 3 | 27148 | 49152 |
+----------+----------+----------------+
1 row in set (0.1027 sec)
```



结果将取决于您使用索引的多少，因此一般来说，您的结果会有所不同。查询最好在测试系统使用，因为在该表上查询

当辅助索引无法放入缓冲池时，避免性能受到冲击的三种策略如下：

- 增加缓冲池的大小。
- 插入数据时删除辅助索引。
- 对表进行分区。

在进行期间增加缓冲池大小是最明显的策略，也是最不可能有用的策略。将数据插入已经具有大量数据的表中时，它主要很有用，并且您知道在数据加载期间，您可以占用其他进程所需的一些内存，并将其用于缓冲池。在这种情况下，支持动态调整缓冲池的大小非常有用。例如，将缓冲池大小设置为 256 MiB

```
mysql> SET GLOBAL innodb_buffer_pool_size = 256 * 1024 * 1024;
Query OK, 0 rows affected (0.0003 sec)
```



数据加载完成后，您可以将缓冲池大小设置回常规值（如果使用默认值，则为 134217728）。

如果要插入到空表中，非常有用的策略是删除所有辅助索引（可能为数据验证留下唯一索引），然后重新添加索引。在大多数情况下，这比在加载数据时尝试维护索引效率更高，如果实用程序创建备份，它也就是这样做。

最后一个策略是对表进行分区。这很有帮助，因为索引是分区的本地（这就是分区键必须是所有唯一索引的一部分的原因），因此，如果以分区顺序插入数据，InnoDB将只需要维护当前分区中数据的索引。这使得每个索引更小，因此它们更容易放入缓冲池中。

## 配置

您可以通过执行加载的会话的配置影响负载性能。这包括考虑关闭约束检查、如何生成自动增量 ID 等。

表总结了与散列数据性能（缓冲区池大小）相关的最重要的配置选项。范围是该选项是否可以在会话级别更改，或者该选项仅在全局范围内可用。

| 选项名称                       | 范围 | 描述                                                         |
| :----------------------------- | :--- | :----------------------------------------------------------- |
| foreign_key_checks             | 会话 | 指定是否检查新行是否违反外键。禁用此选项可以提高带外键的表的性能。 |
| unique_checks                  | 会话 | 指定是否检查新行是否违反唯一约束。禁用此选项可以提高具有唯一索引的表的性能。 |
| innodb_autoinc_lock_mode       | 全球 | 指定 InnoDB 如何确定下一个自动增量值。将此选项设置为 2（MySQL 8，但牺牲了潜在的非缩放自动递增值。需要重新启动 MySQL。 |
| innodb_flush_log_at_trx_commit | 全球 | 确定 InnoDB 刷新对数据文件所做的更改的频率。如果使用许多小型事务导入数据，则将此选项设置为 0 或 2 可以提高性能。 |
| sql_log_bin                    | 会话 | 设置为 0 或 OFF 时禁用二进制。这将大大减少写入的数据量。     |
| transaction_isolation          | 会话 | 设置事务隔离级别。如果您未读取 MySQL 中的现有数据，请考虑将隔离 |

所有选项都有副作用，因此请仔细考虑更改设置是否适合您。例如，如果要将数据从现有实例导入新实例，并且知道外键和唯一键约束没有问题，和选项。另一方面，如果您从源导入，而您不确定数据的完整性，则最好保持启用约束检查，以确保数据的质量，即使这意味着加载性能变慢。

对于选项，您需要考虑丢失最后一秒左右的已提交交易的风险是否可接受。如果数据加载进程是实例上的唯一事务，并且很容易重做导入设置为 0 或 2 以减少刷新数。更改对小型事务最有用。如果导入提交时间少于一次，则更改不会获得很少的收获。如果更改则请记住在导入后将值设置回 1。

对于二进制日志，禁用写入导入的数据非常有用，因为它大大减少了必须写入磁盘的数据更改量。如果二进制日志与重做日志和数据文件位于同一磁盘上，则这尤其有用。如果无法修改导入禁用 sql_log_bin ，可以考虑使用跳过日志 bin 选项重新启动 MySQL完全禁用二进制日志，但请注意，这也会影响系统上的所有其他事务。如果在导入过程中禁用二进制日志记录，则在导入后立即创建完整备份可能很有用，因此可以再次使用二进制日志进行时间点恢复。

您还可以通过选择导入数据的语句和使用事务方式来提高加载性能。

## 事务和加载方法

事务表示一组更改，InnoDB 在提交事务之前不会完全应用更改。每个提交都涉及将数据写入重做日志，并包括其他开销。如果您有非常小的事务（如一次插入一行），此开销会显著影响负载性能。

最佳事务大小没有黄金法则。对于小行大小，通常几千行是好的，而对于较大的行大小选择较少的行。最终，您需要对系统和数据进行测试，以确定最佳事务规模。

对于加载有两个主要选择语句或语句。通常的性能优于语句，因为分析较少。对于语句，使用扩展插入语法的优点是使用单个语句而不是多行语句插入多行。

使用 LOAD 一个优点是 MySQL 命令行程序可以自动并行执行加载。

## MySQL Shell 并行加载数据

将数据加载到 MySQL 时，可能会遇到的一个问题是单个线程无法将 InnoDB 推到其所能承受的极限。如果将数据拆分为批处理并使用多个线程加载数据，则可以提高总体加载速率。自动执行此操作的一个选项是使用 MySQL 命令行管理程序 8.0.17 及更晚的并行数据加载功能。

并行加载功能可通过 Python 实用程序和方法提供。本讨论将假定您正在使用 Python 模式。第一个参数是文件名，第二个（可选）参数是包含可选参数的字典。您可以使用 实用程序获取帮助文本，例如

```
mysql-py> util.help('import_table')
```



帮助文本包括可以通过第二个参数中指定的字典提供的所有设置的详细说明。

MySQL Shell 禁用重复密钥和外键检查，并设置未提交执行导入的连接，以尽可能减少导入期间的开销。

默认值是将数据插入当前架构中的表中，其名称与没有扩展名的文件相同。例如，如果文件名为则默认表。清单中显示了将文件加载到表中一个简单示例。文件可从本书的 GitHub 存储库中

```
Listing 25-8. Using the util.import_table() utility with default settings
mysql> \sql
Switching to SQL mode... Commands end with ;
mysql-sql> CREATE SCHEMA IF NOT EXISTS chapter_25;
Query OK, 1 row affected, 1 warning (0.0490 sec)
mysql-sql> DROP TABLE IF EXISTS chapter_25.t_load;
Query OK, 0 rows affected (0.3075 sec)
mysql-sql> CREATE TABLE chapter_25.t_load (
 id int unsigned NOT NULL auto_increment,
 val varchar(40) NOT NULL,
 PRIMARY KEY (id),
 INDEX (val)
 );
Query OK, 0 rows affected (0.3576 sec)
mysql> SET GLOBAL local_infile = ON;
Query OK, 0 rows affected (0.0002 sec)
mysql> \py
Switching to Python mode...
mysql-py> \use chapter_25
Default schema set to `chapter_25`.
mysql-py> util.import_table('D:/MySQL/Files/t_load.csv')
Importing from file 'D:/MySQL/Files/t_load.csv' to table `chapter_25`.`t_load`
in MySQL Server at localhost:3306 using 2 threads
[Worker000] chapter_25.t_load: Records: 721916 Deleted: 0 Skipped: 0 Warnings: 0
[Worker001] chapter_25.t_load: Records: 1043084 Deleted: 0 Skipped: 0 Warnings: 0
100% (85.37 MB / 85.37 MB), 446.55 KB/s
File 'D:/MySQL/Files/t_load.csv' (85.37 MB) was imported in 1 min 52.1678 sec
at 761.13 KB/s
Total rows affected in chapter_25.t_load: Records: 1765000 Deleted: 0
Skipped: 0 Warnings: 0
```



创建架构时取决于您是否更早创建了架构。请注意，您必须启用选项才能使实用程序正常工作。

该示例最有趣的部分是导入的执行。当您不指定任何内容时，MySQL 命令行程序将文件拆分为 50 MB 块，并使用最多 8 个线程。在这种情况下，文件为 85.37 MB（MySQL Shell 使用指标文件大小 = 85.37 MB 与 81.42 MiB 相同），因此它给出两个区块，其中第一个区块为 50 MB，第二个区块为 35.37 MB。这不是一个可怕的好分布。

您可以选择告诉 MySQL 外壳要拆分的大小。最优的是每个线程最终处理相同数量的数据。例如，如果要除以 85.37 MB 数据，请将区块大小设置为大小的一半多一点，例如 43 MB。如果为大小指定了十进制值，则该值将向下舍入。还可以设置其他几个选项，清单显示了设置其中一些选项的示例。

```
Listing 25-9. Using util.import_table() with several custom settings
mysql-py> \sql TRUNCATE TABLE chapter_25.t_load
Query OK, 0 rows affected (1.1294 sec)
mysql-py> settings = {
 'schema': 'chapter_25',
 'table': 't_load',
 'columns': ['id', 'val'],
 'threads': 4,
 'bytesPerChunk': '21500k',
 'fieldsTerminatedBy': '\t',
 'fieldsOptionallyEnclosed': False,
 'linesTerminatedBy': '\n'
 }
mysql-py> util.import_table('D:/MySQL/Files/t_load.csv', settings)
Importing from file 'D:/MySQL/Files/t_load.csv' to table `chapter_25`.
`t_load` in MySQL Server at localhost:3306 using 4 threads
[Worker001] chapter_25.t_load: Records: 425996 Deleted: 0 Skipped: 0 Warnings: 0
[Worker002] chapter_25.t_load: Records: 440855 Deleted: 0 Skipped: 0 Warnings: 0
[Worker000] chapter_25.t_load: Records: 447917 Deleted: 0 Skipped: 0 Warnings: 0
[Worker003] chapter_25.t_load: Records: 450232 Deleted: 0 Skipped: 0 Warnings: 0
100% (85.37 MB / 85.37 MB), 279.87 KB/s
File 'D:/MySQL/Files/t_load.csv' (85.37 MB) was imported in 2 min 2.6656
sec at 695.99 KB/s
Total rows affected in chapter_25.t_load: Records: 1765000 Deleted:
0 Skipped: 0 Warnings: 0
```



在这种情况下，将显式指定目标架构、表和列，将文件拆分为四个大致相等的区块，线程数设置为四个。CSV 文件的格式也包含在设置中（指定的值为默认值）。

最佳线程因硬件、数据和其他正在运行的查询而有很大差异。您需要进行试验，以找到系统的最佳设置。

## 总结

本章讨论了决定 DDL 语句和批量数据加载性能的指标。第一个主题是模式更改的交替。当您进行架构更改时，支持三种不同的算法。性能最好的算法是算法，它可用于在行的末尾添加列和几个元数据更改。第二个最佳在大多数情况下，它修改现有表空间文件中的数据。最终的算法，一般来说是最昂贵的算法是。

在无法使用，将存在大量 I/O，因此磁盘性能很重要，而需要磁盘 I/O 的其他工作也越好。它也可能有助于锁定表，因此 MySQL 不需要跟踪数据更改并在架构更改结束时应用它们。

对于插入数据，有人讨论以主键顺序插入非常重要。如果插入顺序是随机的，则会导致更大的表、群集索引的更深的 B 树索引、更多的磁盘查找以及更多的随机 I/O。以主键顺序插入数据的最简单的方法是使用自动递增主键，让 MySQL 确定下一个值。对于 UUID，MySQL 8 添加了和函数，允许您将 UUID 所需的存储量减至 16 字节，并交换时间戳的低阶和高阶部分，使 UUID 单调增加。

插入数据时，插入速率突然减慢的典型原因是辅助索引不再适合缓冲池。如果插入到空表中，则在导入期间删除索引是一种优势。分区也可能有帮助，因为它将索引拆分为每个分区的一部分，因此一次只需要索引的一部分。

在某些情况下，您可以禁用约束检查，减少重做日志的刷新，禁用二进制日志记录，并减少事务隔离。这些配置更改都有助于减少开销;但是，所有也有副作用，因此您必须仔细考虑更改是否对您的系统来说是可以接受的。您还可以通过调整事务大小来平衡处理大型事务的提交开销和开销的减少，影响性能。

对于批量插入，有两个加载数据的选项。您可以使用常规，也可以使用 LOAD 语句。后者一般是首选方法。它还允许您使用 MySQL 命令行管理程序 8.0.17 及更晚的并行表导入功能。

下一章将学习提高复制性能。
