# Ⅲ 访问操作系统

访问任意命令的执行在后端 DBMS 底层操作系统上可以用所有的三种数据库软件来实现。需求是：高特权会话用户和 Web 应用支持批量查询。  
本章描述的技术允许执行命令，通过盲注、基于 UNION 查询或基于错误的 SQL 注入技术来获取它们的标准输出：这是一个带内连接，命令通过 SQL 注入执行，标准输出也通过 HTTP 协议返回。

## 用户定义函数

Wikipedia 对用户定义函数 (UDF) 的定义如下: 

> 在 SQL 数据库中，用户定义函数通过添加一个可以在 SQL 语句中计算的函数，提供了一种扩展数据库服务器功能的机制。SQL 标准区分了标量函数和表函数。标量函数只返回一个值或 NULL。  
> …… SQL 中用户定义的函数使用 CREATE FUNCION 语句进行声明。

在现代数据库管理系统中，可以从位于文件系统上的共享库创建功能。然后可以在 `SELECT` 语句中调用这些函数，就像任何其他内置字符串函数一样。所有这三个数据库管理系统都有一组库和 API，开发人员可以使用这些库和 API 创建用户定义的函数。  
在 Linux 和 UNIX 系统上，共享库是一个共享对象(SO) ，可以使用 GCC 进行编译。在 Windows 上，它是一个动态链接库(DLL) ，可以用 Microsoft Visual C++ 编译。
为了编译一个共享库，有必要在操作系统上安装特定的 DBMS 开发库。例如，在 Debian GNU/Linux 等系统的最新版本中，为了能够为 PostgreSQL 编译 UDF，您需要安装 `postgreql-server-dev-8.3` 包。对于 Windows，开发库路径需要手动添加到 Microsoft Visual C++ 项目设置中。
下一步是将共享库放置在 DBMS 在从共享库创建函数时寻找它们的路径中: PostgreSQL 允许将共享库放置在 Windows 或 Linux 上的任何可读/可写文件夹中，MySQL 需要将二进制文件放置在特定位置，具体位置取决于特定的软件版本和操作系统。

## UDF 注入

注入攻击者到目前为止低估了使用 UDF 控制底层操作系统的潜力。然而，这个被忽视的数据库安全领域可能提供了实现命令执行的路由。通过利用 SQL 注入缺陷，可以上传包含两个用户定义函数的共享库:

- `sys_eval(cmd)` - 执行任意命令，并返回标准输出。
- `sys_exec(cmd)` - 执行任意命令，并返回其退出代码。

在后端 DBMS 寻找共享库的路径上上传二进制文件后，攻击者可以从中创建两个用户定义函数: 这将是 UDF 注入。

现在，攻击者可以调用这两个函数中的任何一个: 如果命令是通过 `sys_exec()` 执行的，那么它是通过批处理查询技术执行的，不会返回任何输出。否则，如果它通过 ``sys_eval()`` 执行，就会创建一个支持表，命令运行一次，它的标准输出就会插入到表中，然后可以使用盲注、UNION 查询或基于错误的技术通过转储支持表的第一个条目来检索它；转储之后，条目被删除，支持表就可以重新使用了。

### MySQL

#### 创建共享库

在 MySQL 上，可以创建一个共享库，定义一个用户定义函数 (UDF) 来执行底层操作系统上的命令。几年前，Marco Ivaldi 演示了他的共享库定义了一个用于执行命令的 UDF。然而，我很清楚，这有两个限制:

- 它不符合 MySQL 5.0+ 标准，因为它没有遵循新的 MySQL 语言标准来创建正确的 UDF；
- 它调用 C `system()` 函数来执行命令，并且总是返回整数 0。

这个 UDF 表达式在新的 MySQL 服务器版本中几乎没有用处，因为如果攻击者想要获得退出状态或命令的标准输出，他做不到。

事实上，我发现可以使用 UDF 来执行命令，并通过 SQL 注入获取它们的标准输出。
我首先将注意力集中在 UDF Repository for MySQL 上，我修改了他们的一段代码`lib_mysqludf_sys`，通过添加 `sys_eval()` 函数来执行任意命令并返回命令标准输出。这段代码与 Linux 和 Windows 兼容。  
补丁的源代码可以在 sqlmap 子版本存储库中找到。

`sys_exec()` 函数可用于执行任意命令，并且比 Marco Ivaldi 的共享库有两个优点:

- 它兼容 MySQL 5.0以上版本，可以在 Linux 上作为共享对象（Shared Object, `.so`）编译，也可以在 Windows 上作为动态链接库（`.dll`）编译;
- 它返回执行命令的退出状态（exit status）。

在 MySQL 参考手册中可以找到用 C 语言创建与 MySQL 兼容的用户定义函数的指南。我还发现了 Roland Bouman 关于[如何使用 Microsoft Visual C++ 在 Windows 上编译共享库的博客文章](http://rpbouman.blogspot.com/2007/09/creating-mysql-udfs-with-microsoft.html)。

Windows 上的共享库大小是 9216 字节，Linux 上是 12896 字节。共享库越小，通过 SQL 注入上传的速度就越快。为了使它尽可能小，攻击者可以在编译它时设置更高等级的__优化__，一旦编译完成，就可以通过在 Windows 上使用像 UPX 这样的 PE 文件打包器来减小它的大小。通过在 Linux 上使用 Strip 命令丢弃所有符号，可以减少共享对象的大小。在 Windows 上得到的二进制文件大小是 6656 字节，在 Linux 上是 5476 字节: 分别比最初编译的共享库小 27.8% 和 57.54%。

值得注意的是，使用 MySQL 6.0 开发库编译的 MySQL 共享库与所有其他 MySQL 版本是向后兼容的，因此通过编译一个 MySQL 共享库，相同的二进制文件可以在相同架构和操作系统上的任何 MySQL 服务器版本上重用。

#### SQL 注入到命令执行

会话用户必须拥有以下特权: mysql 数据库上的 `FILE` 和 `INSERT`，在一个共享库路径上的写访问以及写文件所需的权限（参阅 5.1 节）。

步骤如下:

1. 通过盲注或 UNION 查询:
    1. 收集 MySQL 版本。有两个原因: 
        - 选择 SQL 语句来测试对批处理查询的支持，如 3.1 节所述; 
        - 确定一个有效的共享库绝对文件路径，如下一段所述。
    2. 检查 `sys_exec()` 和 `sys_eval()` 函数是否已经存在，以避免不必要的数据覆盖。

2. 测试是否支持批量查询;

3. 通过批量查询：
    1. 将共享库上传到一个文件系统的绝对路径下，MySQL 服务器需要能够在他的查找路径下找到这个库；（如下所述）
    2. 从共享库创建两个用户定义函数；
    3. 通过 `sys_exec()` 或 `sys_eval()` 执行任意命令。

根据 MySQL 版本的不同，共享库必须放置在不同的文件系统路径中：

- 在 4.1.25 以下的 MySQL 4.1 版本中
- 在 5.0.67 以下的 MySQL 5.0 版本中
- 在 5.1.19 以下的 MySQL 5.1 版本中  
共享库必须位于系统动态链接器搜索的目录中：
- 在 WINDOWS 上，共享对象可以上传到：
    - `C:\WINDOWS`
    - `C:\WINDOWS\system`
    - `C:\WINDOWS\system32`
    - `@@basedir\bin`
    - `@@datadir`
- 在 Linux 和 UNIX 系统上，动态链接库可以放在
    - `/lib`
    - `/usr/lib`
    - `/etc/ld.so.conf` 文件中指定的任何路径上;

MySQL 5.1 版本 5.1.19 使用系统变量 `plugin_dir` 强制指定了共享库必须位于的一个绝对文件系统路径。这同样适用于所有版本的 MySQL 6.0。  
来自 MySQL 5.1 和 MySQL 6.0 手册: 

> ```sql
> CREATE [AGGREGATE] FUNCTION function_name RETURNS
> {STRING|INTEGER|REAL|DECIMAL} SONAME shared_library_name
> ```
> `share_library_name` 是包含实现该函数的代码的共享对象文件的 basename（没有目录路径和后缀名的文件名）。**文件必须位于插件目录中。这个目录是由 `plugin_dir` 系统变量的值给出的。**

- MySQL 4.1 版本 4.1.25 和 MySQL 5.0 版本 5.0.67 也引入了系统变量 `plugin_ dir`：默认情况下它是空的，并且应用了以前 MySQL 版本的相同行为。

MySQL 5.0 手册中指出：

> [……] 从 MySQL 5.0.67 开始，文件必须位于插件目录中。这个目录由 `plugin_dir` 系统变量的值给出。如果 `plugin_dir` 的值为空，则应用 5.0.67 之前使用的行为：**文件必须位于系统动态链接器搜索的目录中。**

### PostgreSQL

#### 创建共享库

在 PostgreSQL 上，任意命令的执行可以通过三种方式实现:

- 利用 libc 内置的 `system()` 函数：Nico Leidecker 在他的论文[《有趣的 PostgreSQL》](http://leidecker.info/downloads/Having_Fun_With_PostgreSQL.pdf)中描述了这种技术；

- 创建一个合适的过程语言函数（[Procedural Language](http://www.postgresql.org/docs/8.3/static/xplang.html) Function，PLF）：[Daniele Bellucci](http://daniele.bellucci.googlepages.com/) 描述了通过使用 PL/Perl 和 PL/Python 语言来实现的步骤;

- 创建一个 [C语言函数 (UDF)](http://www.postgresql.org/docs/8.3/static/xfunc-c.html)：[David Litchfield](http://www.davidlitchfield.com/) 在他的书[《数据库黑客手册》](http://eu.wiley.com/WileyCDA/WileyTitle/productCd-0764578014.html) 第25章《PostgreSQL：发现和攻击》中描述了这项技术。[示例代码](http://media.wiley.com/product_ancillary/14/07645780/DOWNLOAD/578014_Code.zip)可以从图书主页免费获得。

所有这些方法都有至少一个限制，使得它们在较新的 PostgreSQL 服务器套件中毫无用处:

- 第一种方法只有效到 PostgreSQL 8.1 版，并返回命令的 exit status，而不是命令标准输出。由于 PostgreSQL 8.2-devel 版本，所有共享库都必须包含一个“magic 块”。

    PostgreSQL 8.3 手册中提到:

    > PostgreSQL 8.2 需要一个 magic 块。要 include 这个 magic block，你需要在包含头文件 fmgr.h 之后，在（唯一）一个模块的源文件中加入这段代码: 
    > 
    > ```C
    > #ifdef PG_MODULE_MAGIC
    > PG_MODULE_MAGIC;
    > # endf
    > ```
    > 
    > 如果代码不需要针对 8.2 之前的 PostgreSQL 版本进行编译，可以省略 `#ifdef` 测试。

- 第二种方法只有在 PostgreSQL 服务器在编译时设置了至少支持一种 PL(过程语言) 的情况下才有效。默认情况下，它们不可用，至少在大多数 Linux 发行版和 Windows 上不可用;

- 第三种方法在 PostgreSQL 8.1版之前一直有效，其原因与第一种方法相同，并且具有相同的行为：没有命令标准输出。不过可以修改它，使其包含“魔术块”后正常工作；在8.1 以上的 PostgreSQL 版本中也可以这样做。

我将上面描述的 MySQL 共享库的 C 源代码移植到 PostgreSQL，并创建了一个名为 `lib_postgreqludf_sys` 的共享库，其中包含两个 C 语言函数。源代码可以在 [sqlmap 子版本存储库](https://svn.sqlmap.org/sqlmap/trunk/sqlmap/)中找到。

Windows 上的共享库大小是 8192 字节，Linux 上是 8567 字节。共享库越小，通过 SQL 注入上传的速度就越快。为了使它尽可能小，攻击者可以在启用了__优化__的情况下编译它，一旦编译完成，就有可能通过在 Windows 上使用像 UPX 这样的打包程序来减小它的大小，并通过在 Linux 上使用 Strip 命令来丢弃所有符号来减小共享对象的大小。在 Windows 上得到的二进制文件大小是 6144 字节，在 Linux 上是5476字节：分别比最初编译的共享库小 25% 和 36.1% 。

使用 PostgreSQL 8.3 开发库编译的共享库不能向后兼容任何其他 PostgreSQL 版本：共享库必须使用您希望使用的相同 PostgreSQL 开发库版本进行编译。

#### SQL 注入命令执行

会话用户必须是“超级用户”，步骤如下:

1. 通过盲查询或 UNION 查询:

    1. 指纹探测 PostgreSQL 的版本，以便选择 SQL 语句来测试对批处理查询的支持（参阅3.2节）所述;
    2. 检查 `sys_exec()` 和 `sys_eval()` 函数是否已经存在，以避免不必要的数据覆盖。

2. 测试是否支持批量查询;

3. 通过批量查询：

    1. 将共享库上载到一个绝对文件系统路径，需要运行 PostgreSQL 的用户有读写权限，在 Linux 和 UNIX 系统上可以是` /tmp`，在 WINDOWS 上可以是 `C:\WINDOWS\Temp`；

    2. 从共享库中创建两个用户定义函数 UDF;

    3. 通过 `sys_exec()` 或 `sys_eval()` 执行任意命令。

有趣的是，PostgreSQL 比 MySQL 更灵活，允许指定共享库所在的绝对路径。

## 存储过程

维基百科对存储过程的定义如下: 

> 存储过程是应用程序访问关系数据库系统时可用的子程序。存储过程实际上存储在数据库数据字典中。
> 
> …… 存储过程类似于用户定义的函数。主要区别在于 UDF 可以像 SQL 语句中的任何其他表达式一样使用，而存储过程必须使用 `CALL` 语句或 `EXECUTE` 语句调用。

在现代数据库管理系统中，可以创建存储过程来执行复杂的任务。一些数据库管理系统也有内置的过程，例如 Microsoft SQL Server 和 Oracle。通常，存储过程会深入使用 DBMS 特有的语言: 分别是 Transact-SQL 和 PL/SQL。

### Microsoft SQL Server

#### `xp_cmdshell` 过程

Microsoft SQL Server 有一个内置的扩展存储过程来执行命令并返回它们在底层操作系统上的标准输出: `xp_cmdshell()`。

这个存储过程默认在 Microsoft SQL Server 2000 启用，同时也存在于 Microsoft SQL Server 2005 和 2008，但默认情况下是禁用的：如果会话用户是 `sysadmin` 服务器角色的成员，攻击者可以远程重新启用它。在 Microsoft SQL Server 2000，可以使用 `sp_addextdedproc` 存储过程，而在 Microsoft SQL Server 2005 和 2008，则可以使用 `sp_configure` 存储过程。  
如果过程重新启用失败而会话用户又有相应的权限，攻击者可以使用 shell 对象重新创建一个新过程。这种技术已经演示过多次，并且如果会话用户具有高权限的话仍然可用。在所有 Microsoft SQL Server 版本中，此过程只能由具有 `sysadmin` 服务器角色的用户执行。在 Microsoft SQL Server 2005 和 2008，指定为__代理帐户__（proxy account）的用户也可以运行此过程。


#### SQL 注入到命令执行

会话用户必须具有 `CONTROL SERVER` 权限。  
要做的第一件事是检查 `xp_cmdshell()` 扩展过程是否存在并已启用: 如果已禁用，则重新启用它; 如果创建失败，则从头开始创建它。

如果攻击者需要命令的标准输出:

1. 创建一个支持表，包含一个字段，数据类型为 `text`;
2. 通过 `xp_cmdshell()` 过程执行命令，将其标准输出重定向到一个临时文件;
3. 使用 `BULK INSERT` 语句将临时文件的内容作为一个条目加载到支持表中;
4. 通过 `xp_cmdshell()` 删除临时文件;
5. 检索支持表条目的内容;
6. 删除支持表的内容

否则可以直接:

1. 通过 `xp_cmdshell()` 过程执行命令。
