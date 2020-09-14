# MySQL Workbench

是 Oracle 用于查询和管理 MySQL 服务器的图形用户界面。它可以被看做是使用 MySQL 的两把瑞士军刀之一，另一把是 MySQL Shell下一章将讨论。

MySQL 工作台的主要功能是可以执行查询的查询模式。 但是，还有其他一些功能，如性能报告、可视化解释、管理配置和检查架构的能力等。

如果将 MySQL 工作台与 MySQL 企业监视器进行比较，则 MySQL 企业监视器专用于监视，是一个服务器解决方案，而 MySQL 工作台是桌面解决方案，主要是用于使用 MySQL 服务器的客户端。同样，MySQL 工作台中包含的监视都是临时监视，而 MySQL 企业监视器作为服务器解决方案包含对存储历史数据的支持。

本章将介绍 MySQL 工作台，并介绍安装、基本用法以及如何创建 EER 关系图。性能报告和可视化解释将在后几章中介绍。

------

如果您已经熟悉MySQL Workbench，则可以考虑跳过本章或略过本章。

------



## 安装

安装 MySQL 工作台的方式与其他 MySQL 程序相同，只是仅支持使用包管理器（因此没有独立安装）。MySQL 工作台版本号遵循 MySQL 服务器版本，因此 MySQL 工作台 8.0.18 与 MySQL 服务器 8.0.18 同时发布。MySQL 工作台版本支持发布时仍在维护的 MySQL 服务器版本，因此 MySQL 工作台 8.0.18 支持连接到 MySQL 服务器 5.6、5.7 和 8。

本节将介绍如何在 Microsoft Windows、"企业 Linux 7"（Oracle Linux、红帽企业 Linux 和 CentOS）和 Ubuntu 19.10 上安装 MySQL 工作台的示例。其他 Linux 平台在概念上与两个 Linux 示例类似。

### 微软视窗

在 Microsoft Windows 上，安装 MySQL 工作台的首选方式是将Windows。如果您安装了其他 MySQL 产品，您可能已经安装了 MySQL 安装程序，在这种情况下，您可以跳过这些说明的第一步，然后单击"，该屏幕将您带至图的点。

你可以从 。图显示了下载部分。

![](../附图/Figure%2011-1.png)

安装程序有两种选择。第一个称为Web安装程序只是 MySQL安装程序，还包括 MySQL 服务器。如果您也计划安装 MySQL Server，则选择包含 MySQL 安装程序和 MySQL 服务器的下载是有意义的，因为您避免等待安装程序稍后下载 MySQL 服务器安装文件。此示例假定您选择 Web 安装程序。

单击"下载按钮访问下载。如果您尚未登录，它将带您到"开始下载"页面，您可以在登录和立即开始下载进行选择。如图。

![](../附图/Figure%2011-2.png)

如果您已经拥有帐户，可以登录。否则，您可以选择注册 Oracle 帐户。您也可以选择下载安装程序，而无需登录通过点击"链接。

下载完成后，启动下载的文件。除了确认您将允许安装程序和 MySQL 安装程序修改已安装的程序外，无需执行任何操作来安装 MySQL 安装程序。安装完成后，MySQL 安装程序将自动启动安装程序安装的 MySQL 程序，如图。

![](../附图/Figure%2011-3.png)

如果您没有安装任何 MySQL 程序，您将被带到一个屏幕，要求您确认您同意许可条款。请仔细阅读许可条款，然后再继续。如果您可以接受许可证，请勾选"，然后单击标有"下一继续。

下一步是选择要安装什么。设置屏幕如图。

![](../附图/Figure%2011-4.png)

You can choose between several bundles such as the developer bundle (called *Developer Default*) which installs the products typically used in a development environment. When you choose a setup type, the description in the right of the screen includes a list of the products that will be installed. For this example, the custom installation type will be used.

The next step is to choose which products to install. That uses the selector shown in Figure [11-5](#Fig5).

![](../附图/Figure%2011-5.png)

您可以在"应用程序"下的可用产品列表中找到 MySQL 工作台。单击右侧箭头，将 MySQL 工作台添加到要安装的产品和功能列表中。随意选择其他产品;对于本书，建议也包括 MySQL 雪壳。添加所有需要的产品后，单击"下一步继续。

以下屏幕提供了要安装的产品的摘要。单击执行"以启动安装。如果 MySQL 安装程序尚未具有本地副本，安装过程包括下载产品。安装可能需要一点时间才能完成。完成后，单击"下一继续。最终屏幕列出了已安装的程序，并提供了启动 MySQL 工作台和 MySQL 外壳的选项。单击"以关闭 MySQL 安装程序。

如果您以后想要安装更多产品或执行升级或删除产品，程序，这将带您到主 MySQL 安装程序屏幕，如图。

![](../附图/Figure%2011-6.png)

选择要屏幕最右侧执行的操作。操作是

- 安装产品和功能。
- 更改现有产品的安装。这主要适用于 MySQL 服务器。
- 升级已安装的产品。
- 卸载产品。
- 更新 MySQL 安装程序可用的 MySQL 产品列表。

这五个操作允许您执行 MySQL 产品生命周期内所需的所有步骤。

### 企业 Linux 7

如果使用 Linux，则使用包管理器安装 MySQL 工作台。在 Oracle Linux、红帽企业 Linux 和 CentOS 7 上，首选的软件包管理器是因为它有助于解决您安装或升级的软件包的依赖关系。MySQL 有一个 yum 存储库，用于其社区产品。此示例将演示如何安装它，并使用它来安装 MySQL 工作台。

您可以在下一个位置找到存储库 也有 APT 和 SUSE 的存储库。选择与 Linux 发行版对应的文件，然后单击"下载图显示了企业 Linux 7 的文件。

![](../附图/Figure%2011-7.png)

如果您没有登录，它会将您带至第二个屏幕，例如在 Microsoft Windows 上安装 MySQL 工作台的示例。这将允许您登录到您的 Oracle Web 帐户、创建帐户或下载而无需登录。下载 RPM 文件并保存在要从目录中安装的目录中，或者右键单击"下载"按钮（如果您已登录），或者"否"感谢我的下载链接并复制 URL，如图。

![](../附图/Figure%2011-8.png)

现在，您可以安装，如清单。

```
Listing 11-1. Installing the MySQL community repository
shell$ wget https://dev.mysql.com/get/mysql80-community-release-el7-3.
noarch.rpm
...
HTTP request sent, awaiting response... 200 OK
Length: 26024 (25K) [application/x-redhat-package-manager]
Saving to: 'mysql80-community-release-el7-3.noarch.rpm'
100%[=========================>] 26,024 --.-K/s in 0.001s
2019-08-18 12:13:47 (20.6 MB/s) - 'mysql80-community-release-el7-3.noarch.rpm'
saved [26024/26024]
Figure 11-8. Copying the link to the repository installation file
shell$ sudo yum install mysql80-community-release-el7-3.noarch.rpm
Loaded plugins: langpacks, ulninfo
Examining mysql80-community-release-el7-3.noarch.rpm: mysql80-communityrelease-el7-3.noarch
Marking mysql80-community-release-el7-3.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package mysql80-community-release.noarch 0:el7-3 will be installed
--> Finished Dependency Resolution
Dependencies Resolved
=================================================================
 Package
 Arch Version
 Repository Size
=================================================================
Installing:
 mysql80-community-release
 noarch el7-3 /mysql80-community-release-el7-3.noarch 31 k
Transaction Summary
=================================================================
Install 1 Package
Total size: 31 k
Installed size: 31 k
Is this ok [y/d/N]: y
Downloading packages:
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
 Installing : mysql80-community-release-el7-3.noarch 1/1
 Verifying : mysql80-community-release-el7-3.noarch 1/1
Installed:
 mysql80-community-release.noarch 0:el7-3
Complete!
```

MySQL Workbench 需要来自 EPEL 存储库的一些包。在 Oracle Linux 7 上，您可以启用它，如

```
sudo yum install oracle-epel-release-el7
```



在红帽企业 Linux 和 CentOS 上，您需要从 Fedora 下载存储库定义：

```
wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install epel-release-latest-7.noarch.rpm
```

现在，您可以安装 MySQL Workbench如清单。

```
shell$ sudo yum install mysql-workbench
...
Dependencies Resolved
================================================================
 Package Arch Version Repository Size
================================================================
Installing:
 mysql-workbench-community
 x86_64 8.0.18-1.el7 mysql-tools-community 26 M
Transaction Summary
================================================================
Install 1 Package
Total download size: 26 M
Installed size: 116 M
Is this ok [y/d/N]: y
Downloading packages:
warning: /var/cache/yum/x86_64/7Server/mysql-tools-community/packages/
mysql-workbench-community-8.0.18-1.el7.x86_64.rpm: Header V3 DSA/SHA1
Signature, key ID 5072e1f5: NOKEY
Public key for mysql-workbench-community-8.0.18-1.el7.x86_64.rpm is not
installed
mysql-workbench-community-8.0.18-1. | 31 MB 00:14
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
Importing GPG key 0x5072E1F5:
 Userid : "MySQL Release Engineering <mysql-build@oss.oracle.com>"
 Fingerprint: a4a9 4068 76fc bd3c 4567 70c8 8c71 8d3b 5072 e1f5
 Package : mysql80-community-release-el7-3.noarch (@/mysql80-communityrelease-el7-3.noarch)
 From : /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql
Is this ok [y/N]: y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
 Installing : mysql-workbench-community-8.0.18-1.el7.x86 1/1
 Verifying : mysql-workbench-community-8.0.18-1.el7.x86 1/1
Installed:
 mysql-workbench-community.x86_64 0:8.0.17-1.el7
Complete!
```

例如，您的输出可能看起来不同，具体取决于已安装的包，可能会拉扯依赖项。首次从 MySQL 存储库安装包时，系统将要求您接受用于验证下载的包的 GPG 密钥。如果您从 Fedora 安装了 EPEL 存储库，则还需要接受该存储库中的 GPG 密钥。

### Debian和Ubuntu

在 Debian 和 Ubuntu 上安装 MySQL 工作台遵循与上一示例相同的原则。对于此处演示的步骤，将使用 Ubuntu 19.10。

对于 Debian 和 Ubuntu，您需要安装可以从定义文件下载的 MySQL https://dev.mysql.com/downloads/repo/apt/ 。在编写本文时，只有一个文件可用（参见它独立于体系结构，适用于所有支持的 Debian 和 Ubuntu 版本。

![](../附图/Figure%2011-9.png)

如果您未登录，您将被带到屏幕上，您可以在登录和立即开始下载之间进行选择。下载 DEB 包或右键单击下载（如果您已登录）或启动我的下载链接并复制 URL，如图。

![](../附图/Figure%2011-10.png)

现在，您可以安装 MySQL如清单。

```
shell$ wget https://dev.mysql.com/get/mysql-apt-config_0.8.14-1_all.deb
...
Connecting to repo.mysql.com (repo.mysql.com)|23.202.169.138|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 35564 (35K) [application/x-debian-package]
Saving to: 'mysql-apt-config_0.8.14-1_all.deb'
mysql-apt-config_0. 100%[==================>] 34.73K --.-KB/s in 0.02s
2019-10-26 17:16:46 (1.39 MB/s) - 'mysql-apt-config_0.8.14-1_all.deb' saved
[35564/35564]
Figure 11-10. Copying the link to the repository installation file
shell$ sudo dpkg -i mysql-apt-config_0.8.14-1_all.deb
Selecting previously unselected package mysql-apt-config.
(Reading database ... 161301 files and directories currently installed.)
Preparing to unpack mysql-apt-config_0.8.14-1_all.deb ...
Unpacking mysql-apt-config (0.8.14-1) ...
Setting up mysql-apt-config (0.8.14-1) ...
Warning: apt-key should not be used in scripts (called from postinst
maintainerscript of the package mysql-apt-config)
OK
```



在第二步命令）中，您可以选择哪些 MySQL 产品应该通过存储库提供。图。

![](../附图/Figure%2011-11.png)

默认值是启用 MySQL 服务器和群集以及工具和连接器。对于 MySQL 服务器和群集，您还可以选择要使用的版本，默认为 8。为了安装 MySQL 外壳，您需要设置为启用。在后，选择"确定"。

在开始使用存储库之前，需要执行：

```
shell$ sudo apt-get update
Hit:1 http://repo.mysql.com/apt/ubuntu eoan InRelease
Hit:2 http://au.archive.ubuntu.com/ubuntu eoan InRelease
Hit:3 http://au.archive.ubuntu.com/ubuntu eoan-updates InRelease
Hit:4 http://au.archive.ubuntu.com/ubuntu eoan-backports InRelease
Hit:5 http://security.ubuntu.com/ubuntu eoan-security InRelease
Reading package lists... Done
```

现在，您可以使用 apt-get 的安装命令安装 MySQL 产品。清单显示了安装（请注意，包名称为末尾的"-社区"很重要）。

```
Listing 11-4. Installing MySQL Workbench from the APT repository
shell$ sudo apt-get install mysql-workbench-community
Reading package lists... Done
Building dependency tree
Reading state information... Done
...
Setting up mysql-workbench-community (8.0.18-1ubuntu19.10) ...
Setting up libgail-common:amd64 (2.24.32-4ubuntu1) ...
Processing triggers for libc-bin (2.30-0ubuntu2) ...
Processing triggers for man-db (2.8.7-3) ...
Processing triggers for shared-mime-info (1.10-1) ...
Processing triggers for desktop-file-utils (0.24-1ubuntu1) ...
Processing triggers for mime-support (3.63ubuntu1) ...
Processing triggers for hicolor-icon-theme (0.17-2) ...
Processing triggers for gnome-menus (3.32.0-1ubuntu1) ...
```

输出相当详细，包括安装 MySQL Workbench 所需的其他包的更改列表。包列表取决于您已经安装过什么。

您现在可以开始使用 MySQL Workbench了。

## 创建连接

首次启动时，您需要定义与 MySQL 服务器实例的连接。如果您安装了 MySQL 通知工作台将自动为根用户创建与 MySQL 通知器监视的每个实例的连接。

您还可以根据需要创建连接。一个选项是从 MySQL 工作台连接屏幕执行此操作，如图。

![](../附图/Figure%2011-12.png)

通过单击左上角的图标显示海豚数据库，即可访问连接屏幕。下面通过行连接的表格的图标将带您到数据库建模功能，三个图标中的最后一个将打开数据迁移功能的选项卡。

屏幕截图显示连接包含欢迎消息，并且已有一个连接。您可以右键单击连接以访问连接的选项 - 这些选项包括打开连接（创建到 MySQL 实例的连接）、编辑连接、将连接添加到组等。

单击 MySQL 连接右侧的 +。图。用于创建新连接和编辑现有连接的对话框非常相似。

![](../附图/Figure%2011-13.png)

您可以使用您选择的名称来命名连接。它是一个自由格式的字符串，仅用于更轻松地标识连接的用途。其余的选项是通常的连接选项。

连接后可以从连接屏幕双击它以创建连接。

## 使用 MySQL Workbench

MySQL 工作台中使用最多的功能是执行查询的能力。这是从查询选项卡完成的，该选项卡除了执行查询的功能外，还包括多个功能。这些功能包括显示结果集、获取名为 Visual Explain 的查询计划的可视化表示、获取上下文帮助、重新格式化查询等。本节将介绍一些从概述开始的功能。

### 概述

查询选项卡由两个方面组成，其中一个是编写查询的编辑器，另一个是查询结果。还支持显示上下文帮助和查询统计信息。这两个附加区域在技术上不是查询选项卡的一部分，但由于它们大多与查询选项卡一起使用，因此此处还将讨论它们。

图显示了具有查询选项卡的 MySQL台，并且最重要的功能已编号。

![](../附图/Figure%2011-14.png)

标记为 （1） 的区域是您编写查询的地方。您可以保留多个查询，MySQL 工作台将保存它们，因此当您再次打开连接时，这些查询将被还原。这样可以方便地作为暂存板来存储最常用的查询。

使用标记为 （2） 的三个闪电图标之一执行查询或查询。左图标是一个普通的闪电符号，执行查询编辑器部分中选择的查询或查询。这与使用键盘快捷键 Ctrl+Shift+Enter 相同。带有闪电符号的中间图标和光标执行光标所在的查询。使用此图标与在编辑器中使用快捷方式 Ctrl+Enter 相同。第三个图标在闪电符号前面有一个放大镜，并在窗体中为当前放置光标的查询创建查询计划。显示查询计划的默认方式是作为可视化解释关系图。还可以使用键盘快捷方式 Ctrl+Alt+X 获取查询计划。

结果显示在查询编辑器 （3） 的下方，您可以通过使用查询结果右侧的项在几种格式之间进行选择。最后一项是（4），如果您直接从查询编辑器请求查询计划，它将以相同方式启动查询计划。

查询选项卡输出帧 （5），默认情况下，该框架显示上次执行查询的统计信息。这包括执行查询的时间、查询、找到的行数以及执行查询所用的时间。右侧有一个框架，其中有 SQL 添加 （6），默认情况下显示上下文帮助。您可以启用自动上下文帮助或使用帮助文本上方的图标手动请求它。

### 配置

MySQL可以更改多个设置，范围从颜色到行为和路径到 MySql 工作台所依赖的程序（如

有几种方法可以到达设置，如图。图显示了 MySQL 工作台窗口的左上角和右上角。

![](../附图/Figure%2011-15.png)

在左侧，您可以使用"编辑"从菜单中然后转到首选项项。或者，您可以单击窗口右侧的齿轮图标。无论使用哪种方式，您都访问图。

![](../附图/Figure%2011-16.png)

"设置包括用于语法检查器的 SQL 模式以及是否使用空格或选项卡进行缩进等设置。设置包括是否使用安全设置、是否保存编辑器以及编辑器和查询选项卡的一般行为。如果您使用捆绑二进制文件，管理设置指定要使用的路径，包括用于的路径。建模用于数据库建模功能。"设置允许您更改 MySQL 工作台的可视外观。当您使用需要连接到远程主机的 SSH 连接的功能时，将使用设置。最后，""设置包括一些不适合其他类别的设置，例如是否应使用连接在开始屏幕上显示欢迎消息。

这些包括安全设置。那是什么？

### 安全设置

MySQL 工作台默认两个安全设置，以帮助防止更改或删除表中的所有行，并避免提取太多行。安全设置意味着和语句没有子句，并且语句添加了可以配置最大行数），则"更新"和"删除"语句将被阻止。更新WHERE句不能是微不足道的。

通常最好保持这些设置处于启用状态，但对于某些查询，您需要更改设置，以使它们正常工作。选择可以在设置中更改，如前所述。限制在 SQL 编辑器下的子menu。或者，更简单的方法是使用编辑器上方的拖放框，如图。

![](../附图/Figure%2011-17.png)

这样更改限制会更新与浏览首选项相同的设置。

更新安全设置可以在最远的设置中更改。建议保持该状态，除非您确实需要更新或删除表中的所有行。请注意，禁用设置需要重新连接。

### 重新格式化查询

不太注意的一个很好的功能是查询美化工具。这对于查询调优也很有用，因为格式良好的查询可以更容易地理解查询正在做什么。

查询美化器采用查询，并将选择列表、表和筛选器拆分为单独的行并添加缩进。图。

![](../附图/Figure%2011-18.png)

第一查询是原始查询，整个查询位于一行中。第二个查询是重新格式化的查询。对于像本示例中这样的简单查询来说，美化没有什么价值，但对于更复杂的查询，它可以使查询更易于阅读。

默认情况下，美化包括将 SQL 关键字更改为大写。您可以更改是否应在首选项中的 SQL设置的菜单中发生这种情况。

## EER 图

将探讨的最后一个功能是支持反向工程架构和创建增强的实体关系 （EER） 关系图。这是获取您正在使用架构的概述的有用方法。如果已定义外键，MySQL 工作台将使用定义将表链接在一起。

可以从数据库菜单选项启动反向，然后选择反向。，Ctrl+R 键盘组合也会带您去那里。如图。

![](../附图/Figure%2011-19.png)

该向导将带您完成导入架构的步骤，从选择要使用或可选手动配置连接的存储连接开始。下一步将连接并导入第三步中显示的可用架构列表。在这里，您可以选择一个或多个逆向工程，如图。

![](../附图/Figure%2011-20.png)

在此示例中，选择了架构。接下来的步骤获取架构对象，并允许您筛选要包括的对象。最后，将对象导入并放置在关系图中，并显示确认。生成的 EER如图。

![](../附图/Figure%2011-21.png)

该图显示了世界数据库中的表。将鼠标悬停在表上时，子表与其他表的关系将突出显示为绿色，父表以蓝色突出显示。这允许您快速浏览表之间的关系，以便为您提供在需要调整查询时至关重要的知识。

## 总结

本章介绍了 MySQL 工作台，这是 MySQL 的图形用户界面解决方案。展示了如何安装 MySQL 工作台和创建连接。然后概述了主查询视图，并展示了如何配置 MySQL 工作台。默认情况下，如果没有实际无法执行和语句查询限制为 1000 行。

所讨论的两个功能是查询美化和 EER 图。这些并不是唯一的功能，以后的章节将显示性能报告和可视化解释查询计划图的示例。

下一章将讨论 MySQL Shell，这是 MySQL 提供的两把"瑞士军刀"中的第二把。
