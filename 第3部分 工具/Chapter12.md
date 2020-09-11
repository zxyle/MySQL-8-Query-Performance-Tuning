# MySQL Shell

MySQL Shell是第二代命令行客户端，与传统的mysql命令行客户端相比，它通过支持X协议以及Python和JavaScript语言而脱颖而出。它还带有一些实用程序，并且高度可扩展。 这使得它成为用于日常任务以及调查性能问题的出色工具。

本章首先概述了MySQL Shell提供的功能，包括内置帮助和丰富提示。 本章的第二部分介绍如何通过使用外部代码模块，报告基础结构和插件来扩展MySQL Shell的功能。

## 概述

第一个具有常规可用性状态的 MySQL Shell 版本是在 2017 年，因此它仍然是 MySQL 工具箱中一个非常新的工具。然而，它已经拥有了大量的功能，远远超出了传统的命令行客户端的功能。这些功能不限于使用 MySQL 命令行管理程序作为 MySQL InnoDB 群集解决方案的一部分所需的功能;还有一些功能可用于日常数据库管理任务和性能优化。

MySQL Shell 比命令行客户端的优势在于 MySQL Shell 编辑器在 Linux 和 Microsoft Windows 上的行为相同，因此，如果您在这两个平台上工作，您就可获得一致的用户体验。 这意味着 Ctrl+D 在 Linux、macOS 和 Microsoft Windows 上都存在 Shell，Ctrl+W 会删除上一个单词，等等。

本节将介绍安装 MySQL Shell、调用它以及一些基本功能。但是，无法详细介绍 MySQL 命令行程序的所有功能。在使用 MySQL  Shell时在线手册，以了解更多信息。

### 安装 MySQL Shell

的安装方式与其他 MySQL 产品相同（MySQL 企业监视器除外）。你可以从它可用于微软 Windows、Linux 和 macOS，并用作源代码。对于 Microsoft Windows，您也可以通过 MySQL 安装程序安装它。

如果使用本机包格式和 Microsoft Windows 的 MySQL 安装程序安装 MySQL  Shell，则安装说明与 MySQL 工作台的安装说明相同，名称除外。有关详细信息，请参阅上一章。

您还可以在 Microsoft Windows 上使用 ZIP 存档或在 Linux 和 macOS 上使用 TAR 存档安装 MySQL  Shell。如果选择该选项，只需解压缩下载的文件，即可完成。

### 调用 MySQL Shell

MySQL Shell或 Microsoft Windows 上的二进制文件调用。 使用本机包安装 MySQL Shell 时，二进制文件将包含环境变量，因此操作系统可以在没有显式提供路径的情况下找到它。

这意味着启动 MySQL 命令行程序的最简单的方法是执行：

```
shell> mysqlsh
MySQL Shell 8.0.18
Copyright (c) 2016, 2019, Oracle and/or its affiliates. All rights
reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates.
Other names may be trademarks of their respective owners.
Type '\help' or '\?' for help; '\quit' to exit.
MySQL JS>
```



提示符看起来与此输出中不同，因为默认提示无法完全以纯文本表示。与命令行不同，MySQL 命令行不需要存在连接，默认情况下不会创建任何连接。

### 创建连接

有几种方法可以为 MySQL 命令连接，包括从命令行和从 MySQL 命令程序内部创建连接。

如果在调用时添加任何与连接相关的参数，则 MySQL 命令行程序将在启动时创建连接。未指定的任何连接选项都将使用其默认值。例如，要使用默认（以及 Linux 和 macOS 套接字）值作为根 MySQL 用户连接到本地主机上的 MySQL 实例，只需指定参数：

```
shell> mysqlsh --user=root
Please provide the password for 'root@localhost': ********
Save password for 'root@localhost'? [Y]es/[N]o/Ne[v]er (default No): yes
MySQL Shell 8.0.18
Copyright (c) 2016, 2019, Oracle and/or its affiliates. All rights
reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates.
Other names may be trademarks of their respective owners.
Type '\help' or '\?' for help; '\quit' to exit.
Creating a session to 'root@localhost'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 39581 (X protocol)
Server version: 8.0.18 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
MySQL localhost:33060+ ssl JS >
```



首次连接时，系统会要求您输入帐户的密码。如果 MySQL Shell中找到 mysql_config_editor 命令，或者您在 Microsoft Windows 上，MySQL Shell 可以使用 Windows 密钥环服务，则 MySQL Shell 将提供为您保存密码，因此您将来不需要输入密码。

或者，您可以使用 URI 指定连接选项，例如：

```
shell> mysqlsh root@localhost:3306?schema=world
```



MySQL 命令程序启动后，请注意提示是如何更改的。MySQL 命令行管理程序具有自适应提示，该提示会更改以反映连接状态。默认提示包括您连接到的端口号。如果连接到 MySQL Server 8，则使用的默认端口是 33060，而不是端口 3306，因为默认情况下，当服务器支持 X 协议时，MySQL 命令行程序使用 X 协议，而不是传统的 MySQL 协议。这就是端口号不是您所期望的原因。

您还可以从 MySQL 命令行程序创建（或更改）连接。您甚至可以有多个连接，因此可以同时处理两个或多个实例。创建会话的方法有多种，包括表。该表还包括如何设置和检索全局会话。""列显示要调用的命令或方法，具体取决于使用的语言模式。

| 方法         | 语言命令                                                     | 描述                                                         |
| :----------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 全球会话     | **所有模式：**（或短）                                       | 创建全局会话（默认会话）。这相当于中的连接。                 |
| 一般会议     | **Javascript：**mysqlx. getSession （）**Python：**mysqlx.get_session（） | 返回会话，以便可以分配给变量。可用于使用 X 协议和经典协议的连接。 |
| 经典会话     | **Javascript：**mysql. getclassicSession （）**Python：**mysql.get_classic_session（） | 与常规会话类似，但始终返回经典会话。                         |
| 设置全局会话 | **Javascript：**外壳. 设置会话 （）**Python：**shell.set_session（） | 从包含会话的变量设置全局会话。                               |
| 获取全局会话 | **Javascript：**外壳. getession （）**Python：**shell.get_session（） | 返回全局会话，以便可以分配给变量。                           |
| 重新         | **所有模式：**•重新连接                                      | 使用与现有全局连接相同的参数重新连接。                       |

创建会话的所有命令和方法都支持格式的 URI 这些方法还支持在字典中提供选项。如果您不包含密码，并且 MySQL Shell 没有帐户的存储密码，则系统将提示您以交互方式输入密码（与传统的命令行客户端不同，MySQL Shell 可以在执行命令期间提示输入信息）。例如，要作为连接到

```
MySQL JS> \connect myuser@localhost
Creating a session to 'myuser@localhost'
Please provide the password for 'myuser@localhost': *******
```



已多次提及语言模式。下一小节将研究您如何使用它。

### 语言模式

MySQL Shell 的最大功能之一是您不限于执行 SQL 语句。您拥有 JavaScript 和 Python 的全部功能，当然还有 SQL 语句。这使得 MySQL  Shell在自动化任务方面非常强大。

一次使用模式工作，但可以在 JavaScript 和 Python 中通过 API 执行查询。表总结了如何从命令行和 MySQL 命令行中选择要使用的语言模式。

| 模式       | 命令行  | MySQL Shell |
| :--------- | :------ | :---------- |
| Javascript | - - js  | \js         |
| Python     | - - py  | \py         |
| Sql        | - - sql | \sql        |

默认模式为 JavaScript。提示反映您模式，因此您始终知道您使用的模式。

在列出如何创建连接时，还有一点需要注意的是，MySQL Shell（如 X DevAPI）会尝试保留通常用于该语言的命名约定。这意味着在JavaScript模式下，函数和方法使用骆驼例而在Python模式下使用如果使用内置帮助，帮助将反映用于您使用的语言模式的名称。

### 内置帮助

很难了解 MySQL 命令行程序的所有功能以及如何使用它们。幸运的是，有一个广泛的内置帮助功能，让你获得有关功能的信息，而不必回到在线手册每次。

如果使用参数执行您将获得有关所有受支持的命令行参数的信息。启动 shell 后，您还可以获得有关命令、对象和方法的帮助。最上面的帮助是使用命令获得的。这将列出命令和全局对象以及如何获得进一步帮助。

第二级帮助用于命令和全局对象。您可以使用其中一个帮助命令指定全局对象命令的名称，以了解有关命令或对象的信息。例如：

```
mysql-js> \h \connect
NAME
 \connect - Connects the shell to a MySQL server and assigns the global
session.
SYNTAX
 \connect [<TYPE>] <URI>
 \c [<TYPE>] <URI>
DESCRIPTION
...
```



最终的帮助级别是全局对象的功能。全局对象的全局对象和模块都有一个方法，该方法为对象或模块提供帮助。方法还可以将模块或对象的方法的名称作为字符串，该字符串将返回该方法的帮助。一些示例是（输出省略，因为它相当详细 - 建议自己尝试命令以查看返回的帮助文本）：

```
MySQL JS> \h shell
MySQL JS> shell.help()
MySQL JS> shell.help('reconnect')
MySQL JS> shell.reports.help()
MySQL JS> shell.reports.help('query')
```



前两个命令检索相同的帮助文本。值得熟悉帮助功能，因为它可以大大提高您使用 MySQL Shell 的效率。

帮助的上下文感知比检测全局对象是否存在以及方法名称是否遵循 JavaScript 或 Python 约定更进一步。考虑有关"选择"的帮助请求。你的意思有几种可能性。它可以是 X DevAPI 中的方法之一，或者您可能想到 SELECT 语句。如果在 SQL 模式下请求帮助，MySQL  Shell假定您指的是 SQL 语句。但是，在 Python 和 JavaScript 模式下，系统会询问您指的是哪一种：

```
MySQL Py> \h select
Found several entries matching select
The following topics were found at the SQL Syntax category:
- SQL Syntax/SELECT
The following topics were found at the X DevAPI category:
- mysqlx.Table.select
- mysqlx.TableSelect.select
For help on a specific topic use: \? <topic>
e.g.: \? SQL Syntax/SELECT
```



MySQL Shell 可以在 SQL 模式下为 SELECT 提供帮助，而不考虑 X DevAPI 的原因是 X DevAPI 方法只能从 Python 和 JavaScript 访问。另一方面，"选择"的所有三个含义在 Python 和 JavaScript 模式下都有意义。

如前所述，存在多个全局对象。那是什么？

### 内置全局对象

MySQL 命令行对要素进行分组。使 MySQL 命令行管理器如此强大的大部分功能都可以在全局对象中找到。正如您将在"插件"部分中看到，也可以添加自己的全局对象。

内置全局对象包括

- 设置默认架构后将保留默认架构的 X DevAPI 架构对象。X DevAPI 表对象可以作为对象的属性找到（除非表或视图名称与现有属性相同）。还可以从对象获取会话对象。
- 用于管理 MySQL InnoDB 群集。
- 用于使用经典的 MySQL 协议连接到 MySQL。
- 用于使用 MySQL X 协议会话。
- 用于使用当前全局会话（连接到 MySQL 实例）。
- 各种通用方法和属性。
- 各种实用程序，如升级检查器、导入 JSON 数据以及将 CSV 文件中的数据导入关系表。

MySQL Shell的概述到此结束。接下来，您将了解有关提示以及如何自定义它进行操作的详细了解。

## 提示

将 MySQL Shell 与传统命令行客户端分开的功能之一是丰富的提示，它不仅便于查看您正在使用的主机和架构，还可以添加信息，例如您是否连接到生产实例、是否使用了 SSL 和自定义字段。

### 内置提示

MySQL 附带了多个预定义的提示模板，您可以选择这些模板。默认值是使用提供有关连接的信息并支持 256 种颜色的提示，但也有更简单的提示。

提示定义模板的位置取决于您安装 MySQL 命令行程序的方式。位置的示例包括

- 存档提示目录。
- 
- 

提示定义是 JSON 文件，其定义包含在 MySQL 命令行壳 8.0.18 中

- 彩色提示仅限于使用 16/8 颜色 ANSI 颜色和属性。
- 提示使用 256 个索引颜色。这是默认使用的。
- 但具有"不可见"的背景色（它只是使用相同的终端），并具有不同的前景颜色。
- 和一样，但有额外的符号。这需要 Powerline 修补字体，如与 Powerline 项目一起安装的字体。当您使用 SSL 连接到 MySQL 并使用"箭头"分隔符时，这将添加一个挂锁和提示。稍后将显示安装 Powerline 字体的示例。
- 但有"令人敬畏的符号"。这还需要令人敬畏的符号包含在 Powerline 字体中。稍后将展示安装真棒符号的示例。
- 这是一个非常基本的提示，只显示 或基于使用模式。
- 一个两行版本的提示.
- 一个两行版本的提示。
- 一个两行版本的提示。
- 提供完整的提示信息，但完全没有颜色。提示的示例是。

如果 shell 窗口的宽度有限，并且它们将信息放在一行上，并允许您在下一行上键入命令，而无需在完整提示之前键入命令，那么双行模板尤其有用。

有两种方法可以指定要使用哪个提示。MySQL 命令行程序首先在用户的 MySQL 命令行管理程序目录中搜索文件提示默认位置取决于您的操作系统：

- =目录中。
- 这是从用户主目录中的目录中。

您可以通过设置环境变量来目录。如果希望使用与默认值不同的提示，可以将该定义复制到目录中并命名文件。

指定提示定义位置的另一种方式是设置环境变量，例如，使用命令提示符在 Microsoft Windows 上：

```
C:\> set MYSQLSH_PROMPT_THEME=C:\Program Files\MySQL\MySQL Shell 8.0\share\
mysqlsh\prompt\prompt_256inv.json
```



在 PowerShell 中，语法略有不同：

```
PS> $env:MYSQLSH_PROMPT_THEME = "C:\Program Files\MySQL\MySQL Shell 8.0\
share\mysqlsh\prompt\prompt_256inv.json";
```



在 Linux 和 Unix 上：

```
shell$ export MYSQLSH_PROMPT_THEME=/usr/share/mysqlsh/prompt/prompt_256inv.
json
```



如果您暂时想要使用与通常提示不同的提示，这非常有用。

正如已经暗示的，大多数提示定义有几个部分。最简单的方法是查看提示的示例，如图提示。

![](../附图/Figure 12-1.png)

提示有几个部分。首先，它红色背景上生产，这是警告您已连接到生产实例。是否将实例视为生产实例取决于您连接到的主机名是否包含在。第二个元素是字符串，它没有任何特殊含义。

第三是您连接到的主机和端口、是否使用 X 协议以及是否使用 SSL。在这种情况下，端口号后有一个 +，指示 X 协议正在使用中。第四个元素是默认架构。

第五个和最后一个元素 ）是语言模式。它将显示或具体取决于您是否分别启用了 SQL、Python 或 JavaScript 模式。 此元素的背景颜色也会随语言而变化。SQL 使用橙色、Python 蓝色和 JavaScript 黄色。

通常，您不会看到提示的所有元素，因为 MySQL 命令行程序仅包括相关元素。例如，只有在设置了默认架构时才包含默认架构，并且仅在连接到实例时才显示连接信息。

当您使用 MySQL 命令行程序时，您可能会意识到您希望对提示定义进行一些更改。让我们看看如何做到这一点。

### 自定义提示定义

提示JSON 文件，没有任何内容可以阻止您编辑定义以根据您的首选项进行更改。这样做的最佳方法就是复制最接近您想要的模板，然后进行更改。

与其详细浏览规范，而是更容易查看模板并讨论该模板的某些部分。清单显示了文件的末尾是定义提示元素的位置。

```
Listing 12-1. The definition of the elements of the prompt
 "segments": [
 {
 "classes": ["disconnected%host%", "%is_production%"]
 },
 {
 "text": " My",
 "bg": 254,
 "fg": 23
 },
 {
 "separator": "",
 "text": "SQL ",
 "bg": 254,
 "fg": 166
 },
 {
 "classes": ["disconnected%host%", "%ssl%host%session%"],
 "shrink": "truncate_on_dot",
 "bg": 237,
 "fg": 15,
 "weight": 10,
 "padding" : 1
 },
 {
 "classes": ["noschema%schema%", "schema"],
 "bg": 242,
 "fg": 15,
 "shrink": "ellipsize",
 "weight": -1,
 "padding" : 1
 },
 {
 "classes": ["%Mode%"],
 "text": "%Mode%",
 "padding" : 1
 }
 ]
```



这里有一些有趣的。首先，请注意，有一个对象与类断开连接 。百分比符号中的名称是在同一文件中定义的变量，或者来自 MySQL Shell 本身的变量（它有变量，如主机和端口）。例如定义为

```
"variables" : {
 "is_production": {
 "match" : {
 "pattern": "*;%host%;*",
 "value": ";%env:PRODUCTION_SERVERS%;"
 },
 "if_true" : "production",
 "if_false" : ""
 },
```



因此，如果主机包含在环境变量中，则主机被视为生产

关于元素列表需要注意的第二件事是，有一些特殊字段，如收缩，可用于定义文本如何保持相对较短的。例如，主机元素使用因此，如果完整主机名太长，则仅显示主机名中第一个点之前的部分。或者大小可用于添加...截断值后。

第三，分别使用和 fg 元素定义前景颜色。这允许您根据颜色完全自定义提示。颜色可以通过以下方式之一指定：

- 有几个颜色是已知的名称：黑色，红色，绿色，黄色，蓝色，品红色，青色和白色。
- 介于 0 和 255 之间的值（两者包含），其中 0 为黑色、63 浅蓝色、127 品红色、193 黄色和 255 白色。
- 使用"#rrggbb"中的值。这要求终端支持真彩色。

一组值得的内置变量在某种程度上取决于环境或您连接到的 MySQL 实例。这些是

- 这使用环境变量。确定是否连接到生产服务器的方式是使用环境变量的示例。
- 这使用 MySQL 中的全局系统变量的值，即。
- 与上一个类似，但使用会话系统变量。
- 这使用来自MySQL的全局状态变量。
- 与上一个类似，但使用会话状态变量。

例如，如果您想要在提示中包括您连接到的实例的 MySQL 版本，可以添加一个元素，如

  ```
{
 "separator": "",
 "text": "%sysvar:version%",
 "bg": 250,
 "fg": 166
 },
  ```



我们鼓励您使用定义，直到获得最适合你的配色方案和元素。改进 Linux 提示的替代方法是安装 Powerline 和真棒字体。

### 电源线和真棒字体

如果您觉得正常的 MySQL Shell 提示过于方形，并且在 Linux 上使用 MySQL Shell，可以考虑使用依赖于 Powerline 和真棒字体的模板之一。默认情况下不安装字体。

此示例将向您展示如何对字体进行最小安装，Gabrielelana 在 GitHub 上令人敬畏的终端字体项目的修补策略分支安装真棒字体。

通过克隆存储库并更改为修补策略分支，可以安装真棒字体。然后是将所需的文件复制到主目录下并重建字体信息缓存文件的问题。这些步骤列在清单。输出在本书中也有可用，以便于复制命令。

```
Listing 12-2. Installing the Awesome fonts
shell$ git clone https://github.com/gabrielelana/awesome-terminal-fonts.git
Cloning into 'awesome-terminal-fonts'...
remote: Enumerating objects: 329, done.
remote: Total 329 (delta 0), reused 0 (delta 0), pack-reused 329
Receiving objects: 100% (329/329), 2.77 MiB | 941.00 KiB/s, done.
Resolving deltas: 100% (186/186), done.
shell$ cd awesome-terminal-fonts
shell$ git checkout patching-strategy
Branch patching-strategy set up to track remote branch patching-strategy
from origin.
Switched to a new branch 'patching-strategy'
shell$ mkdir -p ~/.local/share/fonts/
shell$ cp patched/SourceCodePro+Powerline+Awesome+Regular.* ~/.local/share/
fonts/
shell$ fc-cache -fv ~/.local/share/fonts/
/home/myuser/.local/share/fonts: caching, new cache contents: 1 fonts,
0 dirs
/usr/lib/fontconfig/cache: not cleaning unwritable cache directory
/home/myuser/.cache/fontconfig: cleaning cache directory
/home/myuser/.fontconfig: not cleaning non-existent cache directory
/usr/bin/fc-cache-64: succeeded
```



这需要安装下一部分是安装列名。输出在本书中也可用，以便于复制命令。

```
Listing 12-3. Installing the Powerline font
shell$ wget --directory-prefix="${HOME}/.local/share/fonts" https://github.
com/powerline/powerline/raw/develop/font/PowerlineSymbols.otf
...
2019-08-25 14:38:41 (5.48 MB/s) - '/home/myuser/.local/share/fonts/
PowerlineSymbols.otf' saved [2264/2264]
shell$ fc-cache -vf ~/.local/share/fonts/
/home/myuser/.local/share/fonts: caching, new cache contents: 2 fonts,
0 dirs
/usr/lib/fontconfig/cache: not cleaning unwritable cache directory
/home/myuser/.cache/fontconfig: cleaning cache directory
/home/myuser/.fontconfig: not cleaning non-existent cache directory
/usr/bin/fc-cache-64: succeeded
shell$ wget --directory-prefix="${HOME}/.config/fontconfig/conf.d" https://
github.com/powerline/powerline/raw/develop/font/10-powerline-symbols.conf
...
2019-08-25 14:39:11 (3.61 MB/s) - '/home/myuser/.config/fontconfig/
conf.d/10-powerline-symbols.conf' saved [2713/2713]
```



这不能完全安装 Powerline 字体，但如果只想将 Powerline 字体与 MySQL Shell 一起使用，则只需全部安装。两 命令下载字体和配置文件重建字体信息缓存文件。您需要重新启动 Linux 才能使更改生效。

重新启动完成后，您可以复制其中一模板，成为新的提示，例如：

```
shell$ cp /usr/share/mysqlsh/prompt/prompt_dbl_256pl+aw.json ~/.mysqlsh/
prompt.json
```



结果提示见图。

![](../附图/Figure 12-2.png)

此示例还显示在更改语言模式和设置默认架构时提示如何更改。关于对多个模块的支持，这就是 MySQL 雪具是一个如此强大的工具的原因，因此下一节将介绍如何使用 MySQL Shell 中的外部模块。

## 使用外部模块

对 JavaScript 和 Python 的支持使得在 MySQL  Shell中执行任务变得容易。您不仅限于核心功能，还可以同时导入标准您自己的自定义模块。本节将从使用外部模块的基础知识开始（与内置的 MySQL  Shell模块相反）。下一节将进入报告基础结构，之后将介绍插件。

在 MySQL Shell 中使用 Python 模块的方式与使用交互式 Python 解释器时相同，例如：

```
mysql-py> import sys
mysql-py> print(sys.version)
3.7.4 (default, Sep 13 2019, 06:53:53) [MSC v.1900 64 bit (AMD64)]
mysql-py> import uuid
mysql-py> print(uuid.uuid1())
fd37319e-c70d-11e9-a265-b0359feab2bb
```



确切的输出取决于 MySQL 命令行管理程序的版本以及您使用它的平台。

MySQL  Shell解释器允许您导入 Python 中包含的所有常用模块。如果要导入自己的模块，则需要调整搜索路径。您可以直接在交互式会话中这样做，例如：

```
mysql-py> sys.path.append('C:\MySQL\Shell\Python')
```



这样修改路径对于一次性使用模块来说没问题;但是，如果您创建了一个使用的模块，则不方便。

当 MySQL  Shell启动时，它读取两个配置文件，一个用于 Python，一个用于 JavaScript。对于 Python，该文件是 JavaScript MySQL 命令行程序在四个位置搜索这些文件。在 Microsoft Windows 上，路径按搜索顺序排列：

```
1. %PROGRAMDATA%\MySQL\mysqlsh\
2. %MYSQLSH_HOME%\shared\mysqlsh\
3. <mysqlsh binary path>\
4. %APPDATA%\MySQL\mysqlsh\
```



 

在 Linux 和 Unix 上：

```
1. /etc/mysql/mysqlsh/
2. $MYSQLSH_HOME/shared/mysqlsh/
3. <mysqlsh binary path>/
4. $HOME/.mysqlsh/
```



 

始终搜索所有四个路径，如果文件在多个位置找到，则将执行每个文件。这意味着，如果文件影响相同的变量，则最后找到的文件优先。如果您进行个人更改，则进行更改的最佳位置是第四个位置。可以使用第 4 步中的路径。

如果添加要定期使用的模块，可以修改"文件"中的路径。这样，您就可以将模块导入为任何其他 Python 模块。

作为一个简单的例子，考虑一个非常简单的模块，它有一个函数来掷虚拟骰子，并返回一个和六之间的值：

```
import random
def dice():
    return random.randint(1, 6)
```



该示例也可以本书中的文件配置文件中获取。如果将文件保存文件中（根据保存文件的位置调整行中的路径）：

```
import sys
sys.path.append('C:\MySQL\Shell\Python')
```



下次启动 MySQL Shell 时，您可以使用该模块，例如（因为函数返回一个随机值，因此您的输出会有所不同）：

```
mysql-py> import example
mysql-py> example.dice()
5
mysql-py> example.dice()
3
```



这是扩展 MySQL  Shell的最简单方法。另一种方法就是将报表添加到报表基础结构中。

## 报告基础架构

从 MySQL Shell 8.0.16 开始，有一个报告基础结构可用于内置报表和您自己的自定义报表。这是一种非常强大的方式，可以使用 MySQL 命令程序来监视 MySQL 实例，并在遇到性能问题时收集信息。

本节将开始介绍如何获得有关可用报表的帮助，然后讨论如何执行报表，最后讨论如何添加自己的报表。

### 报告信息和帮助

MySQL 命令行程序的内置帮助也扩展到报表，因此您可以轻松地获取有关如何使用报表的帮助。您可以开始使用无需任何参数即可获取可用报表的列表。如果将报表名称添加为参数以及，则获得该报表的详细帮助。清单显示了两种用途的示例。

```
Listing 12-4. Obtaining a list of reports and help for the query report
mysql-py> \show
Available reports: query, thread, threads.
mysql-py> \show query --help
NAME
 query - Executes the SQL statement given as arguments.
SYNTAX
 \show query [OPTIONS] [ARGS]
 \watch query [OPTIONS] [ARGS]
DESCRIPTION
 Options:
 --help, -h Display this help and exit.
 --vertical, -E
 Display records vertically.
 Arguments:
 This report accepts 1-* arguments.
```



显示有三个报表可用。这些是版本 8.0.18 的内置报表。第二个命令返回查询报表，该报表显示它需要一个或多个参数，并且有返回帮助以垂直格式返回查询结果。

内置报表

- 执行作为参数提供的查询。
- 返回有关当前连接的信息。
- 返回有关当前用户、前台线程或后台线程的所有连接的信息。

在帮助输出中，您应该注意的另一件事是，它列出了两种执行报表的方法。您可以使用也用于生成帮助的命令，也可以使用命令。您可以使用通常的内置帮助获得有关每个命令的更多帮助：

```
mysql-py> \h \show
mysql-py> \h \watch
```



帮助输出相当详细，因此此处省略了该输出。相反，下一小节将讨论如何使用这两个命令。

### 执行报告

有两种不同的报表。您可以要求执行报表一次，也可以请求以固定的时间间隔一遍又一遍地执行报表。

有两个命令可用于执行报表：

- 执行报表一次。
- 继续执行报告的时间间隔，如 Linux命令。

这两种命令都可以从任一语言模式使用。命令没有任何自己的参数（但报表可以添加特定于它的参数）。命令有两个选项，用于指定何时以及如何输出报表。这些选项是

- ：报表每次执行之间等待的秒数。该值必须在 0.1~86400（一天）秒范围内。默认值为 2 秒。
- ：输出报表结果时不要清除屏幕。这将在之前的结果下追加新结果，并允许您查看报表结果的历史记录，直到最早的滚动出视图。

使用命令执行报表时，使用 Ctrl+C 停止执行。

作为执行，请考虑您为查询提供要执行的查询。如果希望以垂直格式返回结果，可以使用 --。清单显示了首先执行报表的结果示例，该报表使用命令从视图提取活动查询，然后5 秒刷新一次，而不清除屏幕。例如，为了确保返回一些数据，可以在第二个连接中执行

```
Listing 12-5. Using the query report
mysql-sql> \show query --vertical SELECT conn_id, current_statement AS
stmt, statement_latency AS latency FROM sys.session WHERE command = 'Query'
AND conn_id <> CONNECTION_ID()
*************************** 1. row ***************************
conn_id: 34979
 stmt: SELECT SLEEP(60)
latency: 32.62 s
mysql-sql> \watch query --interval=5 --nocls --vertical SELECT conn_id,
current_statement AS stmt, statement_latency AS latency FROM sys.session
WHERE command = 'Query' AND conn_id <> CONNECTION_ID()
*************************** 1. row ***************************
conn_id: 34979
 stmt: SELECT SLEEP(60)
latency: 43.02 s
*************************** 1. row ***************************
conn_id: 34979
 stmt: SELECT SLEEP(60)
latency: 48.09 s
*************************** 1. row ***************************
conn_id: 34979
 stmt: SELECT SLEEP(60)
latency: 53.15 s
*************************** 1. row ***************************
conn_id: 34979
 stmt: SELECT SLEEP(60)
latency: 58.22 s
Report returned no data.
```



如果执行相同的命令，则输出将取决于运行报表时在其他线程中执行的语句。用于报表的查询添加了一个条件，条件是连接 ID 必须与生成报表的连接的查询 ID 不同。带查询报表的命令本身几乎没有价值，因为您可以执行查询。它对于其他报表更有用，在将查询与 \watch 命令一起使用查询。

命令更有趣，因为它允许您不断更新结果。在示例中，报表在停止之前运行五次。第四次，有另一个连接执行查询，第五次报告不生成数据。请注意，连续执行之间的查询的语句延迟将添加到语句延迟中超过五秒。这是因为 5 秒是 MySQL Shell 从一次迭代结果等待的时间，直到它再次开始执行查询。因此，两个输出之间的总时间是间隔加上查询执行时间以及处理结果的时间。

报表基础结构不仅允许您使用内置报表。您还可以添加自己的报表。

### 添加您自己的报告

报告基础结构的真正功能是它易于扩展，因此 MySQL 开发团队和您可以添加更多报告。虽然您可以使用外部模块的支持来添加报表（就像使用 Innotop 一样）但这种方法要求您自己实现报表基础结构，并且必须使用模块的语言模式来执行报表。使用报告基础结构时，将全部处理，并且报表可用于所有语言模式。

讨论如何创建自己的报表的一个好方法就是创建一个简单的报表并讨论组成报表的各个部分。清单显示了创建查询代码。在本书的 GitHub 存储库中中的文件文件也提供该代码。稍后将讨论将代码保存到哪里，以便它作为 MySQL 命令行程序中的报表可用。

```
Listing 12-6. Report querying the sys.session view
'''Defines the report "sessions" that queries the sys.x$session view
for active queries. There is support for specifying what to order by
and in which direction, and the maximum number of rows to include in
the result.'''
SORT_ALLOWED = {
 'thread': 'thd_id',
 'connection': 'conn_id',
 'user': 'user',
 'db': 'db',
 'latency': 'statement_latency',
 'memory': 'current_memory',
}
def sessions(session, args, options):
 '''Defines the report itself. The session argument is the MySQL
 Shell session object, args are unnamed arguments, and options
 are the named options.'''
 sys = session.get_schema('sys')
 session_view = sys.get_table('x$session')
 query = session_view.select(
 'thd_id', 'conn_id', 'user', 'db',
 'sys.format_statement(current_statement) AS statement',
 'sys.format_time(statement_latency) AS latency',
 'format_bytes(current_memory) AS memory')
 # Set what to sort the rows by (--sort)
 try:
 order_by = options['sort']
 except KeyError:
 order_by = 'latency'
 if order_by in ('latency', 'memory'):
 direction = 'DESC'
 else:
 direction = 'ASC'
 query.order_by('{0} {1}'.format(SORT_ALLOWED[order_by], direction))
 # If ordering by latency, ignore those statements with a NULL latency
 # (they are not active)
 if order_by == 'latency':
 query.where('statement_latency IS NOT NULL')
 # Set the maximum number of rows to retrieve is --limit is set.
 try:
 limit = options['limit']
 except KeyError:
 limit = 0
 if limit > 0:
 query.limit(limit)
 result = query.execute()
 report = [result.get_column_names()]
 for row in result.fetch_all():
 report.append(list(row))
 return {'report': report}
```



代码首先定义一个包含用于对结果进行排序的受支持的值的字典。稍后将在会话（） 函数中的代码注册报表时使用。函数是创建报表的地方。函数采用三个参数：

- 这是一个 MySQL对象（定义与 MySQL 实例的连接）。
- 传递给报表的未命名参数的列表。这是用于查询报表的，您只需指定查询，而无需在查询之前添加参数名称。
- 包含报表命名参数的字典。

会话报告使用命名选项，因此使用 args 参数。

接下来的八行使用 X DevAPI 定义基本查询。首先，从会话架构的架构对象。然后架构对象（使用获取会话视图来获取视图和表。最后，使用参数创建选择查询，这些参数指定应检索哪些列以及要用于列的别名。

接下来，参数，该参数在用作键。如果密钥不存在，则报告将回回按延迟排序。排序顺序定义为根据延迟或内存使用情况对输出进行排序的降序;否则，排序顺序是升序。方法用于将排序信息添加到查询中。此外，按延迟排序时，仅包含延迟不的会话。

参数以类似的方式处理，值 0 表示所有匹配会话。最后，执行查询。报表作为列表生成，第一个项目是列标题，其余项目是结果中的行。报表返回报表项中包含报表列表字典。

此报表返回格式为列表的结果。还有另外两种格式。总体而言，支持以下结果格式：

- 结果作为列表返回，第一项是标头，其余行应按其显示的顺序显示。标头和行本身是列表。
- 结果是包含单个项的列表。MySQL 命令行程序使用 YAML 来显示结果。
- 结果将直接打印到屏幕中。

剩下的就是登记报告。这是使用 shell 方法完成的，如清单所示（这也包含在文件）。

```
Listing 12-7. Registering the sessions report
# Make the report available in MySQL Shell.
shell.register_report(
 'sessions',
 'list',
 sessions,
 {
 'brief': 'Shows which sessions exist.',
 'details': ['You need the SELECT privilege on sys.session view and ' +
 'the underlying tables and functions used by it.'],
 'options': [
 {
 'name': 'limit',
 'brief': 'The maximum number of rows to return.',
 'shortcut': 'l',
 'type': 'integer'
 },
 {
 'name': 'sort',
 'brief': 'The field to sort by.',
 'shortcut': 's',
 'type': 'string',
 'values': list(SORT_ALLOWED.keys())
 }
 ],
 'argc': '0'
 }
)
```



方法采用定义报表的四个参数，并提供 MySQL 命令行管理程序的内置帮助功能返回的帮助信息。参数为

- ：报表的名称。您可以相对自由地选择名称，只要它是单个单词，并且它对于所有报表都是唯一的。
- 结果格式：""
- 生成报表的函数的对象，本例中。
- 描述报表的可选参数。如果提供说明，则使用字典，如描述的那样。

描述是最复杂的参数。它由包含以下键的字典组成（所有项目都是可选的）：

- 简述。
- ：作为字符串列表提供的报表的详细说明。
- ：命名参数作为字典列表。
- 未命名参数的数量。您指定，作为确切的数字，如本例中所示，星号 （） 表示任何数量的参数、具有精确数字的范围（如或具有最小参数数。

元素用于定义报表的命名参数。列表的每个字典对象都必须包含参数的名称，并且支持多个可选参数，以提供有关参数的更多信息。表列出了字典键及其默认值和说明。名称是必需的;需要名称键。其余的是可选的。

| 关键     | 默认值 | 描述                                                         |
| :------- | :----- | :----------------------------------------------------------- |
| 名字     |        | 调用报表时与双破折号（例如 -- 一起使用的参数名称。           |
| 简短     |        | 参数的简短描述。                                             |
| 细节     |        | 作为字符串列表提供的参数的详细说明。                         |
| 快捷方式 |        | 可用于访问参数的单个字母数字字符。                           |
| 类型     | 字符串 | 参数类型。写入时支持的值是字符串、布尔、整数和浮点。选择布尔时，参数用作默认为 False 的。 |
| 必填     | 假     | 参数是否为必填项。                                           |
| 值       |        | 字符串参数的允许值。如果未提供值，则支持所有值。这是示例中用于限制允许的排序选项的。 |

和注册代码保存在用户配置路径下的目录中，该目录默认为 Microsoft Windows 上的在 Linux 和 Unix与搜索配置文件的第四个路径相同）。在启动 MySQL Shell 时，所有具有文件名扩展名的脚本都将作为 Python 脚本执行（以及表示 JavaScript）。

如果将文件复制到此目录中并重新启动 MySQL Shell（请确保使用 MySQL X 端口进行连接 – 默认情况下端口 33060），可以使用会话报告，如清单。报表的结果会有所不同，因此，如果执行报表，则不会看到相同的结果。

```
mysql-py> \show
Available reports: query, sessions, thread, threads.
mysql-py> \show sessions --help
NAME
 sessions - Shows which sessions exist.
SYNTAX
 \show sessions [OPTIONS]
 \watch sessions [OPTIONS]
DESCRIPTION
 You need the SELECT privilege on sys.session view and the underlying
 tables and functions used by it.
 Options:
 --help, -h Display this help and exit.
 --vertical, -E
 Display records vertically.
 --limit=integer, -l
 The maximum number of rows to return.
 --sort=string, -s
 The field to sort by. Allowed values: thread, connection,
 user, db, latency, memory.
mysql-py> \show sessions --vertical
*************************** 1. row ***************************
 thd_id: 81
 conn_id: 36
 user: mysqlx/worker
 db: NULL
statement: SELECT `thd_id`,`conn_id`,`use ... ER BY `statement_latency` DESC
 latency: 40.81 ms
 memory: 1.02 MiB
mysql-py> \js
Switching to JavaScript mode...
mysql-js> \show sessions --vertical
*************************** 1. row ***************************
 thd_id: 81
 conn_id: 36
 user: mysqlx/worker
 db: NULL
statement: SELECT `thd_id`,`conn_id`,`use ... ER BY `statement_latency` DESC
 latency: 71.40 ms
 memory: 1.02 MiB
mysql-js> \sql
Switching to SQL mode... Commands end with ;
mysql-sql> \show sessions --vertical
*************************** 1. row ***************************
 thd_id: 81
 conn_id: 36
 user: mysqlx/worker
 db: NULL
statement: SELECT `thd_id`,`conn_id`,`use ... ER BY `statement_latency` DESC
 latency: 44.80 ms
 memory: 1.02 MiB
```



新的报表的显示方式与内置报表相同，并且具有与内置报表相同的功能，例如，支持在垂直输出中显示结果。支持垂直输出的原因是因为报表将结果作为列表返回，因此 MySQL 命令行管理程序处理格式。另请注意，即使报表是用 Python 编写的，也可以在所有三种语言模式下使用报表。

还有一种导入报表的替代方法。您可以将报表作为插件的一部分，而不是将文件保存到目录。

## 插件

MySQL Shell在 8.0.17 版中增加了对插件的支持。插件由一个或多个代码模块，这些模块可以包括报表、实用程序或其他任何可能用于您且可作为 Python 或 JavaScript 代码执行的代码。这是扩展 MySQL  Shell的最有力方式。代价是，它也相对复杂，但好处是更容易共享和导入一组功能。插件的另一个好处是，不仅可以从任何语言模式执行报告;您的代码的其余部分也可以从 Python 和 JavaScript 使用。

通过将插件名称添加到用户配置路径下的插件来创建插件，该目录默认为 Microsoft Windows 上的Linux 和 Unix与搜索配置文件的第四个路径相同）。该插件可以包含任何数量的文件和目录，但所有文件必须使用相同的编程语言。

在本书的 GitHub 存储库中称为的示例插件包含在目录/myext中。它包括图。带圆角的浅色（黄色）矩形表示目录，较深（红色）文档形状是目录中的文件列表。

![](../附图/Figure 12-3.png)

您可以查看插件的结构，如 Python 包和模块。需要注意的两个重要事情是，每个目录中必须有文件，导入执行 init.py 文件（JavaScript 模块的这意味着您必须包括必要的代码，以注册插件的公共部分。在此示例插件中，所有文件都为空。

报表 与中生成的会话报告相同，只是报表的注册在报表中完成，个同名的报表。

包括一个模块函数，该模块由 。在函数也注册清单显示了函数。

```
Listing 12-9. The get_columns() function from utils/util.py
'''Define utility functions for the plugin.'''
def get_columns(table):
 '''Create query against information_schema.COLUMNS to obtain
 meta data for the columns.'''
 session = table.get_session()
 i_s = session.get_schema("information_schema")
 i_s_columns = i_s.get_table("COLUMNS")
 query = i_s_columns.select(
 "COLUMN_NAME AS Field",
 "COLUMN_TYPE AS Type",
 "IS_NULLABLE AS `Null`",
 "COLUMN_KEY AS Key",
 "COLUMN_DEFAULT AS Default",
 "EXTRA AS Extra"
 )
 query = query.where("TABLE_SCHEMA = :schema AND TABLE_NAME = :table")
 query = query.order_by("ORDINAL_POSITION")
 query = query.bind("schema", table.schema.name)
 query = query.bind("table", table.name)
 result = query.execute()
 return result
```



的函数

函数采用表对象，并使用 X DevAPI 对数据视图。请注意函数如何通过表对象获取会话和架构。最后，返回执行查询的结果对象。

清单显示了如何函数，因此它在 提供。注册发生在。

```
Listing 12-10. Registering the get_columns() function as util.get_columns()
'''Import the utilities into the plugin.'''
import mysqlsh
from myext.utils import util
shell = mysqlsh.globals.shell
# Get the global object (the myext plugin)
try:
 # See if myext has already been registered
 global_obj = mysqlsh.globals.myext
except AttributeError:
 # Register myext
 global_obj = shell.create_extension_object()
 description = {
 'brief': 'Various MySQL Shell extensions.',
 'details': [
 'More detailed help. To be added later.'
 ]
 }
 shell.register_global('myext', global_obj, description)
# Get the utils extension
try:
 plugin_obj = global_obj.utils
except IndexError:
 # The utils extension does not exist yet, so register it
 plugin_obj = shell.create_extension_object()
 description = {
 'brief': 'Utilities.',
 'details': ['Various utilities.']
 }
 shell.add_extension_object_member(global_obj, "util", plugin_obj,
 description)
definition = {
 'brief': 'Describe a table.',
 'details': ['Show information about the columns of a table.'],
 'parameters': [
 {
 'name': 'table',
 'type': 'object',
 'class': 'Table',
 'required': True,
 'brief': 'The table to get the columns for.',
 'details': ['A table object for the table.']
 }
 ]
}
try:
 shell.add_extension_object_member(plugin_obj, 'get_columns',
 util.get_columns, definition)
except SystemError as e:
 shell.log("ERROR", "Failed to register myext util.get_columns ({0})."
 .format(str(e).rstrip()))
```



第一个重要观察是模块。shell都可以通过获取，因此在使用 MySQL Shell 中的扩展时，这是一个重要的模块。另请注意如何模块。始终需要使用从插件名称开始的完整路径来导入插件模块。

为了注册函数，首先，检查插件是否已经存在于。如果没有，则使用创建，并使用注册。这个舞蹈是必要的，因为有文件，你不应该依赖于他们执行的顺序。

接下来，和 如果你有一个大型插件，有可能最终复制代码并执行类似的步骤，因此你可以考虑创建实用程序函数，以避免重复自己。

最后，使用本身。由于参数采用对象，因此可以指定所需的对象类型。

对于模块和函数的注册，不需要代码中的名称和注册的名称相同。报告中的注册包括更改名称的示例（如果您有兴趣）。但是，在大多数情况下，保持名称相同是首选方法，以便更轻松地查找功能后面的代码。

工具添加了两个都注册的函数。有来自骰子（）函数，以及获取列信息的描述函数。清单中显示了与描述函数相关的代码部分。

```
Listing 12-11. The describe() function in tools/example.py
import mysqlsh
from myext.utils import util
def describe(schema_name, table_name):
 shell = mysqlsh.globals.shell
 session = shell.get_session()
 schema = session.get_schema(schema_name)
 table = schema.get_table(table_name)
 columns = util.get_columns(table)
 shell.dump_rows(columns)
```



需要注意的最重要的事情是对象作为和表对象。方法用于生成结果的输出。该方法采用结果对象，并采用格式（默认为表格式）。在输出结果的过程中，将消耗结果对象。

您现在已准备好尝试该插件。您需要将整个插件目录中MySQL 命令行管理程序。清单显示了帮助内容中的全局对象。

```
Listing 12-12. The global objects in the help content
mysql-py> \h
...
GLOBAL OBJECTS
The following modules and objects are ready for use when the shell starts:
- dba Used for InnoDB cluster administration.
- myext Various MySQL Shell extensions.
- mysql Support for connecting to MySQL servers using the classic MySQL
 protocol.
- mysqlx Used to work with X Protocol sessions using the MySQL X DevAPI.
- session Represents the currently open MySQL session.
- shell Gives access to general purpose functions and properties.
- util Global object that groups miscellaneous tools like upgrade checker
 and JSON import.
For additional information on these global objects use: <object>.help()
```



请注意插件如何作为全局对象显示。您可以使用插件，就像任何内置的全局对象一样。这包括获取插件子部分的帮助，如清单的。

```
Listing 12-13. Obtaining help for myext.tools
mysql-py> myext.tools.help()
NAME
 tools - Tools.
SYNTAX
 myext.tools
DESCRIPTION
 Various tools including describe() and dice().
FUNCTIONS
 describe(schema_name, table_name)
 Describe a table.
 dice()
 Roll a dice
 help([member])
 Provides help about this object and it's members
```



作为最后一个示例，请考虑 方法。清单在这两种方法。

```
Listing 12-14. Using the describe() and get_columns() methods in Python
mysql-py> myext.tools.describe('world', 'city')
+-------------+----------+------+-----+---------+----------------+
| Field       | Type     | Null | Key | Default | Extra          |
+-------------+----------+------+-----+---------+----------------+
| ID          | int(11)  | NO   | PRI | NULL    | auto_increment |
| Name        | char(35) | NO   |     |         |                |
| CountryCode | char(3)  | NO   | MUL |         |                |
| District    | char(20) | NO   |     |         |                |
| Population  | int(11)  | NO   |     | 0       |                |
+-------------+----------+------+-----+---------+----------------+
mysql-py> \use world
Default schema `world` accessible through db.
mysql-py> result = myext.util.get_columns(db.city)
mysql-py> shell.dump_rows(result, 'json/pretty')
{
 "Field": "ID",
 "Type": "int(11)",
 "Null": "NO",
 "Key": "PRI",
 "Default": null,
 "Extra": "auto_increment"
}
{
 "Field": "Name",
 "Type": "char(35)",
 "Null": "NO",
 "Key": "",
 "Default": "",
 "Extra": ""
}
{
 "Field": "CountryCode",
 "Type": "char(3)",
 "Null": "NO",
 "Key": "MUL",
 "Default": "",
 "Extra": ""
}
{
 "Field": "District",
 "Type": "char(20)",
 "Null": "NO",
 "Key": "",
 "Default": "",
 "Extra": ""
}
{
 "Field": "Population",
 "Type": "int(11)",
 "Null": "NO",
 "Key": "",
 "Default": "0",
 "Extra": ""
}
5
```



首先，方法。架构和表使用其名称作为字符串提供，结果作为表打印。然后，将当前架构设置为架构，允许您访问表作为 db 对象。然后方法将结果打印为漂亮的打印 JSON。

到此为止，MySQL Shell的讨论结束了。如果您尚未利用它提供的功能，建议您开始使用它。

## 总结

本章介绍了 MySQL Shell。它最初概述了如何安装和使用 MySQL 命令程序，包括使用连接;SQL、Python 和 JavaScript 语言模式;内置帮助;和全局对象。本章的其余部分介绍 MySQL 命令行的自定义、报告和扩展 MySQL。

MySQL 命令行程序提示不仅仅是一个静态标签。它根据连接和默认架构进行调整，您可以对其进行自定义以包含您连接到的 MySQL 版本等信息，并可以更改使用的主题。

MySQL 命令行管理器的力量来自内置的复杂功能及其对创建复杂方法的支持。扩展功能的最简单方法是将外部模块用于 JavaScript 或 Python。您还可以使用报表基础结构，包括创建自己的自定义报表。最后，MySQL Shell 8.0.17 及以后对插件的支持，您可以使用这些插件在全局对象中添加功能。报告基础结构和插件的优点是，您添加的功能与语言无关。

除非另有说明，否则在本书剩余部分使用命令行接口的所有示例都使用 MySQL 命令程序创建。为了最大限度地减少使用的空间，提示已被替换为除非语言模式很重要，在这种情况下，语言模式包括，例如 Python 模式。

关于性能转向工具的讨论到今天结束。第四部分介绍架构注意事项和查询优化器，下一章将讨论数据类型。

 