# Ⅲ 访问文件系统

Part III Operating system access Arbitrary command execution on the back-end DBMS underlying operating system can be achieved with all of the three database softwares. The requirements are: high privileged session user and batched queries support on the web application11

第三部分操作系统访问任意命令的执行在后端 DBMS 底层操作系统上可以用所有的三种数据库软件来实现。需求是: 高特权会话用户和批处理查询支持 Web 应用11

. The techniques described in this chapter allow the execution of commands and the retrieval of their standard output via blind, UNION query or error based SQL injection technique: the command is executed via SQL injection and the standard output is also retrieved over HTTP protocol, this is an inband connection.

.本章描述的技术允许执行命令并通过盲目的、基于 UNION 查询或基于错误的 SQL 注入技术检索它们的标准输出: 命令通过 SQL 注入执行，标准输出也通过 HTTP 协议检索，这是一个带内连接。

6 User-Defined Function Wikipedia defines User-Defined Function (UDF) as follows: “In SQL databases, a user-defined function provides a mechanism for extending the functionality of the database server by adding a function that can be evaluated in SQL statements. The SQL standard distinguishes between scalar and table functions. A scalar function returns only a single value (or NULL). \[...] User-defined functions in SQL are declared using the CREATE FUNCTION statement.” On modern database management systems, it is possible to create functions from shared libraries12 located on the file system. These functions can then be called within the SELECT statement like any other built-in string function. All of the three database management systems have a set of libraries and API13 that can be used by developers to create user-defined functions. On Linux and UNIX systems the shared library is a shared object (SO) and can be compiled with GCC . On Windows it is a dynamic-link library (DLL) and can be compiled with Microsoft Visual C++ . In order to compile a shared library, it is necessary to have the specific DBMS development libraries installed on the operating system. For instance, on recent versions of Debian GNU/Linux like systems to be able to compile a UDF for PostgreSQL you need to have installed the postgresql-server-dev-8.3 package. With Windows, the development library path need to be added manually to the Microsoft Visual C++ project settings. The next step is to place the shared library in a path where the DBMS looks for them when creating functions from shared libraries: where PostgreSQL allows the shared library to be placed in any readable/writable folder on either Windows or Linux, MySQL needs the binary file to be placed in a specific location which varies depending upon the particular software version and operating system.

6用户定义函数 Wikipedia 对用户定义函数(UDF)的定义如下: “在 SQL 数据库中，用户定义函数通过添加一个可以在 SQL 语句中计算的函数，提供了一种扩展数据库服务器功能的机制。SQL 标准区分了标量函数和表函数。标量函数只返回一个值(或 NULL)。\[ ... ] SQL 中用户定义的函数使用 CREATE FUNCION 语句进行声明。”在现代数据库管理系统中，可以从位于文件系统上的共享库12创建功能。然后可以在 SELECT 语句中调用这些函数，就像任何其他内置字符串函数一样。所有这三个数据库管理系统都有一组库和 API13，开发人员可以使用这些库和 API13创建用户定义的函数。在 Linux 和 UNIX 系统上，共享库是一个共享对象(SO) ，可以使用 GCC 进行编译。在 Windows 上，它是一个动态链接库(dLL) ，可以用 Microsoft Visual C + + 编译。为了编译一个共享库，有必要在操作系统上安装特定的 DBMS 开发库。例如，在 Debian GNU/Linux 等系统的最新版本中，为了能够为 PostgreSQL 编译 UDF，您需要安装 postgreql-server-dev-8.3包。对于 Windows，开发库路径需要手动添加到 MicrosoftVisualC + + 项目设置中。下一步是将共享库放置在 DBMS 在从共享库创建函数时寻找它们的路径中: PostgreSQL 允许将共享库放置在 Windows 或 Linux 上的任何可读/可写文件夹中，MySQL 需要将二进制文件放置在特定位置，具体位置取决于特定的软件版本和操作系统。

7 UDF injection Attackers have so far under-estimated the potential of using UDF to control the underlying operating system. Yet, this over-looked area of database security potentially provides routes to achieve command execution. By exploiting a SQL injection flaw it is possible to upload a shared library which contains two user-defined functions:

7 UDF 注入攻击者到目前为止低估了使用 UDF 控制底层操作系统的潜力。然而，这个被忽视的数据库安全领域可能提供了实现命令执行的路由。通过利用 SQL 注入缺陷，可以上传包含两个用户定义函数的共享库:

• sys\_eval(cmd) - executes an arbitrary command, and returns it's standard output;

Sys \_ eval (cmd)-执行任意命令，并返回标准输出;

• sys\_exec(cmd) - executes an arbitrary command, and returns it's exit code. After uploading the binary file on a path where the back-end DBMS looks for shared libraries, the attacker can create the two user-defined functions from it: this would be UDF injection. Now, the attacker can call either of the two functions: if the command is executed via sys\_exec(), it is executed via batched queries technique and no output is returned. Otherwise, if it is executed via sys\_eval(), a support table is created, the command is run once and its standard output is inserted into the table and either the blind algorithm, the UNION query or the error based technique can be used to retrieve it by dumping the support table's first entry; after the dump, the entry is deleted and the support table is clean to be used again.

Sys \_ exec (cmd)-执行任意命令，并返回其退出代码。在后端 DBMS 寻找共享库的路径上上传二进制文件后，攻击者可以从中创建两个用户定义函数: 这将是 UDF 注入。现在，攻击者可以调用这两个函数中的任何一个: 如果命令是通过 sys \_ exec ()执行的，那么它是通过批处理查询技术执行的，不会返回任何输出。否则，如果它通过 sys \_ eval ()执行，就会创建一个支持表，命令运行一次，它的标准输出就会插入到表中，然后可以使用盲算法、 UNION 查询或基于错误的技术通过转储支持表的第一个条目来检索它; 转储之后，条目被删除，支持表就可以重新使用了。

7.1 MySQL

7.1 MySQL

7.1.1 Shared library creation On MySQL, it is possible to create a shared library that defines a user-defined function to execute commands on the underlying operating system. Marco Ivaldi demonstrated, some years ago, that his shared library defined a UDF to execute a command. However, it is clear to me, that this has two limitations:

7.1.1创建共享库在 MySQL 上，可以创建一个共享库，定义一个用户定义的函数来执行底层操作系统上的命令。几年前，Marco Ivaldi 演示了他的共享库定义了一个用于执行命令的 UDF。然而，我很清楚，这有两个限制:

• It is not MySQL 5.0+ compliant because it does not follow the new guidelines to create a proper UDF;

•它不符合 MySQL 5.0 + 标准，因为它没有遵循新的指导方针来创建适当的 UDF;

• It calls C system() function to execute the command and returns always integer 0. This expression of UDF is almost useless on new MySQL server versions because if an attacker wants to get the exit status or the standard output of the command he can not. In fact, I have found that it is possible to use UDF to execute commands and retrieve their standard output via SQL injection. I firstly focus my attention on the UDF Repository for MySQL and patched one of their codes: lib\_mysqludf\_sys by adding the sys\_eval() function to execute arbitrary commands and returns the command standard output. This code is compatible with both Linux and Windows. The patched source code is available on sqlmap subversion repository . The sys\_exec() function can be used to execute arbitrary commands and has two advantages over Marco Ivaldi's shared library:

它调用 C system ()函数来执行命令，并且总是返回整数0。这个 UDF 表达式在新的 MySQL 服务器版本中几乎没有用处，因为如果攻击者想要获得退出状态或命令的标准输出，他就无法获得。事实上，我发现可以使用 UDF 来执行命令，并通过 SQL 注入检索它们的标准输出。我首先将注意力集中在 UDF Repository for MySQL 上，并通过添加 sys \_ eval ()函数来执行任意命令并返回命令标准输出，从而修补了它们的一个代码: lib \_ mysqludf \_ sys。这段代码与 Linux 和 Windows 兼容。补丁的源代码可以在 sqlmap 子版本存储库中找到。Sys \_ exec ()函数可用于执行任意命令，并且比 Marco Ivaldi 的共享库有两个优点:

• It is MySQL 5.0+ compliant and it compiles on both Linux as a shared object and on Windows as a dynamic-link library;

•它兼容 MySQL 5.0以上版本，可以在 Linux 上作为共享对象编译，也可以在 Windows 上作为动态链接库编译;

• It returns the exit status of the executed command. A guide to create a MySQL compliant user-defined function in C can be found on the MySQL reference manual . I found also useful Roland Bouman's step by step blog post on how to compile the shared library on Windows with Microsoft Visual C++. The shared library size on Windows is 9216 bytes and on Linux it is 12896 bytes. The smaller the shared library is, the quicker it is uploaded via SQL injection. To make it as small as possible the attacker can compile it with the optimization setting enabled and, once compiled, it is possible to reduce the dynamic-link library size by using a portable executable packer like UPX on Windows. The shared object size can be reduced by discarding all symbols with strip command on Linux. The resulting binary file size on Windows is 6656 bytes and on Linux it is 5476 bytes: respectively 27.8% and 57.54% less than the initial compiled shared library. It is interesting to note that a MySQL shared library compiled with MySQL 6.0 development libraries is backward compatible with all the other MySQL versions, so by compiling one, that same binary file can be reused on any MySQL server version on the same architecture and operating system.

它返回执行命令的退出状态。在 MySQL 参考手册中可以找到用 C 语言创建与 MySQL 兼容的用户定义函数的指南。我还发现了 Roland Bouman 关于如何使用 Microsoft Visual C + + 在 Windows 上编译共享库的一步一步的博客文章。Windows 上的共享库大小是9216字节，Linux 上是12896字节。共享库越小，通过 SQL 注入上传的速度就越快。为了使它尽可能小，攻击者可以在启用了优化设置的情况下编译它，一旦编译完成，就可以通过在 Windows 上使用像 UPX 这样的动态链接库 Portable Executable 封装器来减小它的大小。通过在 Linux 上使用 Strip 命令丢弃所有符号，可以减少共享对象的大小。在 Windows 上得到的二进制文件大小是6656字节，在 Linux 上是5476字节: 分别比最初编译的共享库小27.8% 和57.54% 。值得注意的是，使用 MySQL 6.0开发库编译的 MySQL 共享库与所有其他 MySQL 版本是向后兼容的，因此通过编译一个 MySQL 共享库，相同的二进制文件可以在相同架构和操作系统上的任何 MySQL 服务器版本上重用。

7.1.2 SQL injection to command execution The session user must have the following privileges: FILE and INSERT on mysql database, write access on one of the shared library paths and the privileges needed to write a file, refer to section 5.1. The steps are:

7.1.2 SQL 注入到命令执行会话用户必须拥有以下特权: mysql 数据库上的 FILE 和 INSERT，在一个共享库路径上的写访问以及写文件所需的特权，请参阅5.1节。步骤如下:

• Via blind or UNION query are:

通过盲查询或 UNION 查询:

• Fingerprint the MySQL version for two reasons: ∗ Choose the SQL statement to test for batched queries support as explained on section 3.1; ∗ Identify a valid shared libraries absolute file path as explained in the next paragraph.

指定 MySQL 版本有两个原因: \* 选择 SQL 语句来测试对批处理查询的支持，如3.1节所述; \* 确定一个有效的共享库绝对文件路径，如下一段所述。

• Check if sys\_exec() and sys\_eval() functions already exist to avoid unwanted data overwriting.

•检查 sys \_ exec ()和 sys \_ eval ()函数是否已经存在，以避免不必要的数据覆盖。

• Test for batched queries support;

•测试批处理查询支持;

• Via batched queries:

‧透过分批查询:

• Upload the shared library to an absolute file system path where the MySQL server looks for them as described below;

将共享库上传到一个绝对文件系统路径，MySQL 服务器在该路径中查找共享库，如下所述;

• Create the two user-defined functions from the shared library;

* 从共享图书馆创建两个用户定义函数;

• Execute the arbitrary command via either sys\_exec() or sys\_eval(). Depending on the MySQL version, the shared library must be placed in different file system paths:

•通过 sys \_ exec ()或 sys \_ eval ()执行任意命令。根据 MySQL 版本的不同，共享库必须放置在不同的文件系统路径中:

• On MySQL 4.1 versions below 4.1.25, MySQL 5.0 versions below 5.0.67 and MySQL 5.1 versions below 5.1.19 the shared library must be located in a directory that is searched by your system's dynamic linker: on Windows the shared object can be uploaded to either C:\WINDOWS, C:\WINDOWS\system, C:\WINDOWS\system32, @@basedir\bin or to @@datadir14. On Linux and UNIX systems the dynamic-link library can be placed on either /lib, /usr/lib or any of the paths specified in /etc/ld.so.conf file15;

在4.1.25以下的 MySQL 4.1版本中，在5.0.67以下的 MySQL 5.0版本中，在5.1.19以下的 MySQL 5.1版本中，共享库必须位于系统动态链接器搜索的目录中: 在 WINDOWS 上，共享对象可以上传到 C: WINDOWS，C: WINDOWS system，C: WINDOWS system32,@@ basedir bin 或@@ datadir14。在 Linux 和 UNIX 系统上，动态链接库可以放在/lib、/usr/lib 或/etc/ld.so.conf 文件15中指定的任何路径上;

• MySQL 5.1 version 5.1.19 enforced the expected behavior of the system variable plugin\_dir which specifies one absolute file system path where the shared library must be located16. The same applies for all versions of MySQL

MySQL 5.1版本5.1.19强制执行系统变量 plugin \_ dir 的预期行为，该变量指定了共享库必须位于的一个绝对文件系统路径16。这同样适用于所有版本的 MySQL

6.0 . From MySQL 5.1 and MySQL 6.0 manuals: “CREATE \[AGGREGATE] FUNCTION function\_name RETURNS {STRING|INTEGER|REAL|DECIMAL} SONAME shared\_library\_name \[...] shared\_library\_name is the basename of the shared object file that contains the code that implements the function. The file must be located in the plugin directory. This directory is given by the value of the plugin\_dir system variable.”

6.0.来自 MySQL 5.1和 MySQL 6.0手册: “ CREATE \[ AGGREGATE ] FuncION FUNCTION \_ name RETURNS { STRING | INTEGER | REAL | DECIMAL } SONAME share \_ library \_ name \[ ... ] share \_ library \_ name 是包含实现该函数的代码的共享对象文件的基名。文件必须位于插件目录中。这个目录是由 plugin \_ dir 系统变量的值给出的。”

• MySQL 4.1 version 4.1.25 and MySQL 5.0 version 5.0.67 also introduced the system variable plugin\_dir: by default it is empty and the same behavior of previous MySQL versions is applied. From MySQL 5.0 manual : “ \[...] As of MySQL 5.0.67, the file must be located in the plugin directory. This directory is given by the value of the plugin\_dir system variable. If the value of plugin\_dir is empty, the behavior that is used before 5.0.67 applies: The file must be located in a directory that is searched by your system's dynamic linker.”

MySQL 4.1版本4.1.25和 MySQL 5.0版本5.0.67也引入了系统变量 plugin \_ dir: 默认情况下它是空的，并且应用了以前 MySQL 版本的相同行为。从 MySQL 5.0手册: “\[ ... ]从 MySQL 5.0.67开始，文件必须位于插件目录中。这个目录由 plugin \_ dir 系统变量的值给出。如果 plugin \_ dir 的值为空，则应用5.0.67之前使用的行为: 文件必须位于系统动态链接器搜索的目录中

7.2 PostgreSQL

7.2 PostgreSQL

7.2.1 Shared library creation On PostgreSQL, arbitrary command execution can be achieved in three ways:

7.2.1创建共享库在 PostgreSQL 上，任意命令的执行可以通过三种方式实现:

• Taking advantage of libc built-in system() function: Nico Leidecker described this technique in his paper Having Fun With PostgreSQL\[55, 56];

•利用 libc 内置的 system ()函数: Nico Leidecker 在他的论文《有趣的 PostgreSQL 》中描述了这种技术\[55,56] ;

• Creating a proper Procedural Language Function : Daniele Bellucci described the steps to go through to do that by using PL/Perl and PL/Python languages;

创建一个合适的工作语言函数: Daniele Bellucci 描述了通过使用 PL/Perl 和 PL/Python 语言来实现的步骤;

• Creating a C-Language Function (UDF): David Litchfield described this technique in his book The Database Hacker's Handbook, chapter 25 titled PostgreSQL: Discovery and Attack. The sample code is freely available from the book homepage . All of these methods have at least one limitation that make them useless on recent PostgreSQL server installations:

创建一个 C 语言函数(UDF) : David Litchfield 在他的书《数据库黑客手册》第25章《 PostgreSQL: 发现和攻击》中描述了这项技术。示例代码可以从图书主页免费获得。所有这些方法都至少有一个限制，使得它们在最近的 PostgreSQL 服务器安装中毫无用处:

• The first method only works until PostgreSQL version 8.1 and returns the command exit status, not the command standard output. Since PostgreSQL version 8.2-devel all shared libraries must include a "magic block". From PostgreSQL 8.3 manual : “A magic block is required as of PostgreSQL 8.2. To include a magic block, write this in one (and only one) of the module source files, after having included the header fmgr.h : #ifdef PG\_MODULE\_MAGIC PG\_MODULE\_MAGIC; #endif The #ifdef test can be omitted if the code doesn't need to compile against pre-8.2 PostgreSQL releases.”

•第一种方法只能工作到 PostgreSQL 8.1版，并返回命令 exit 状态，而不是命令标准输出。由于 PostgreSQL 8.2-devel 版本，所有共享库都必须包含一个“魔术块”。来自 PostgreSQL 8.3手册: “ PostgreSQL 8.2需要一个魔术块。为了包含一个魔法块，在包含头文件 fmgr.h 之后，在一个(也是唯一一个)模块源文件中编写这个代码: # ifdef PG \_ MODULE \_ MAGIC PG \_ MODULE \_ MAGIC; # endf 如果代码不需要针对8.2之前的 PostgreSQL 版本进行编译，可以省略 # ifdef 测试。”

• The second method only works if PostgreSQL server has been compiled with support for one of the procedural languages. By default they are not available, at least on most Linux distributions and Windows;

•第二种方法只有在 PostgreSQL 服务器是在支持一种过程语言的情况下编译的情况下才有效。默认情况下，它们不可用，至少在大多数 Linux 发行版和 Windows 上不可用;

• The third method works until PostgreSQL version 8.1 for the same reason of the first method and it has the same behavior: no command standard output. Anyway, it can be patched to include the "magic block" and make it work properly; also on PostgreSQL versions above 8.1. I ported the C source code of the MySQL shared library described above to PostgreSQL and created a shared library called lib\_postgresqludf\_sys with two CLanguage Function. The source code is available on sqlmap subversion repository . The shared library size on Windows is 8192 bytes and on Linux it is 8567 bytes. The smallest the shared library is, the quickest it is uploaded via SQL injection. To make it as small as possible the attacker can compile it with the optimization setting enabled and, once compiled, it is possible to reduce the dynamic-link library size by using a portable executable packer like UPC on Windows and the shared object size by discarding all symbols with strip command on Linux. The resulting binary file size on Windows is 6144 bytes and on Linux it is 5476 bytes: respectively 25% and 36.1% less than the initial compiled shared library. The shared library compiled with PostgreSQL 8.3 development libraries is not backward compatible with any other PostgreSQL version: the shared library must be compiled with the same PostgreSQL development libraries version where you want to use it.

•第三种方法在 PostgreSQL 8.1版之前一直有效，其原因与第一种方法相同，并且具有相同的行为: 没有命令标准输出。无论如何，可以对它进行修补，以包含“魔术块”并使其正常工作; 在8.1以上的 PostgreSQL 版本中也可以这样做。我将上面描述的 MySQL 共享库的 C 源代码移植到 PostgreSQL，并创建了一个名为 lib \_ postgreqludf \_ sys 的共享库，其中包含两个 CLlanguage 函数。源代码可以在 sqlmap 子版本存储库中找到。Windows 上的共享库大小是8192字节，Linux 上是8567字节。共享库越小，通过 SQL 注入上传的速度就越快。为了使它尽可能小，攻击者可以在启用了优化设置的情况下编译它，一旦编译完成，就有可能通过在 Windows 上使用像 uPC 这样的动态链接库 Portable Executable 封装程序来减小它的大小，并通过在 Linux 上使用 Strip 命令来丢弃所有符号来减小共享对象的大小。在 Windows 上得到的二进制文件大小是6144字节，在 Linux 上是5476字节: 分别比最初编译的共享库小25% 和36.1% 。使用 PostgreSQL 8.3开发库编译的共享库不能向后兼容任何其他 PostgreSQL 版本: 共享库必须使用您希望使用的相同 PostgreSQL 开发库版本进行编译。

7.2.2 SQL injection to command execution The session user must be a “super user”. The steps are:

7.2.2 SQL 注入命令执行会话用户必须是“超级用户”，步骤如下:

• Via blind or UNION query:

通过盲查询或 UNION 查询:

• Fingerprint the PostgreSQL version in order to choose the SQL statement to test for batched queries support as explained on section 3.2;

为 PostgreSQL 版本打印指纹，以便选择 SQL 语句来测试对批处理查询的支持，如3.2节所述;

• Check if sys\_exec() and sys\_eval() functions already exist to avoid unwanted data overwriting.

•检查 sys \_ exec ()和 sys \_ eval ()函数是否已经存在，以避免不必要的数据覆盖。

• Test for batched queries support;

•测试批处理查询支持;

• Via batched queries:

‧透过分批查询:

• Upload the shared library to an absolute file system path where the user running PostgreSQL has read and write access17, this can be /tmp on Linux and UNIX systems and C:\WINDOWS\Temp on Windows;

•将共享库上载到一个绝对文件系统路径，其中运行 PostgreSQL 的用户已经读写 access 17，在 Linux 和 UNIX 系统上可以是/tmp，在 WINDOWS 上可以是 C: WINDOWS Temp;

• Create the two user-defined functions from the shared library18;

* 从共享图书馆创建两个用户定义的功能18;

• Execute the arbitrary command via either sys\_exec() or sys\_eval(). It is interesting to note that PostgreSQL is more flexible than MySQL and allows to specify the absolute path where the shared library is.

•通过 sys \_ exec ()或 sys \_ eval ()执行任意命令。值得注意的是，PostgreSQL 比 MySQL 更灵活，允许指定共享库所在的绝对路径。

8 Stored procedure Wikipedia defines Stored Procedure as follows: “A stored procedure is a subroutine available to applications accessing a relational database system. Stored procedures are actually stored in the database data dictionary. \[...] Stored procedures are similar to user-defined functions. The major difference is that UDFs can be used like any other expression within SQL statements, whereas stored procedures must be invoked using the CALL statement or EXECUTE statement.” On modern database management system it is possible to create stored procedures to execute complex tasks. Some DBMS have also built-in procedures, Microsoft SQL Server and Oracle for instance. Usually stored procedures make deep use of the DBMS specific dialect: respectively Transact-SQL and PL/SQL.

存储过程维基百科对存储过程的定义如下: “存储过程是应用程序访问关系数据库系统时可用的子程序。存储过程实际上存储在数据库数据字典中。\[ ... ]存储过程类似于用户定义的函数。主要区别在于 UDF 可以像 SQL 语句中的任何其他表达式一样使用，而存储过程必须使用 CALL 语句或 EXECUTE 语句调用。”在现代数据库管理系统中，可以创建存储过程来执行复杂的任务。一些数据库管理系统也有内置的过程，例如 Microsoft SQL Server 和 Oracle。通常，存储过程会深入使用 DBMS 特有的方言: 分别是 Transact-SQL 和 PL/SQL。

8.1 Microsoft SQL Server

8.1 Microsoft SQL Server

8.1.1 xp\_cmdshell procedure Microsoft SQL Server has a built-in extended stored procedure to execute commands and return their standard output on the underlying operating system: xp\_cmdshell()\[29,

8.1.1 xp \_ cmdshell 过程 Microsoft SQL Server 有一个内置的扩展存储过程来执行命令并返回它们在底层操作系统上的标准输出: xp \_ cmdshell ()\[29,

30, 31]. This stored procedure is enabled by default on Microsoft SQL Server 2000, whereas on Microsoft SQL Server 2005 and 2008 it exists but it is disabled by default: it can be re-enabled by the attacker remotely if the session user is a member of the sysadmin server role. On Microsoft SQL Server 2000, the sp\_addextendedproc stored procedure can be used whereas on Microsoft SQL Server 2005 and 2008, the sp\_configure stored procedure can be used. If the procedure re-enabling fails, the attacker can create a new procedure from scratch using shell object if the session user has the required privileges. This technique has been illustrated numerous times and can be still used if the session user is high privileged . On all Microsoft SQL Server versions, this procedure can be executed only by users with the sysadmin server role. On Microsoft SQL Server 2005 and 2008 also users specified as proxy account can run this procedure.

30,31].这个存储过程默认在2000 Microsoft SQL Server 启用，而在2005年和2008年 Microsoft SQL Server 存在，但默认情况下是禁用的: 如果会话用户是 sysadmin 服务器角色的成员，攻击者可以远程重新启用它。在2000年 Microsoft SQL Server，可以使用 sp \_ addextdedproc 存储过程，而在2005年和2008年 Microsoft SQL Server，则可以使用 sp \_ configure 存储过程。如果过程重新启用失败，如果会话用户拥有所需的特权，攻击者可以使用 shell 对象从头创建一个新过程。这种技术已经演示过多次，如果会话用户具有高特权，仍然可以使用它。在所有 Microsoft SQL Server 版本中，此过程只能由具有 sysadmin 服务器角色的用户执行。在2005年和2008年 Microsoft SQL Server，指定为代理帐户的用户也可以运行此程序。

8.1.2 SQL injection to command execution The session user must have CONTROL SERVER permission. The first thing to do is to check if xp\_cmdshell() extended procedure exists and is enabled: re-enable it if it is disabled and create it from scratch if the creation fails. If the attacker wants the command standard output:

8.1.2 SQL 注入到命令执行会话用户必须具有 CONTROL SERVER 权限。要做的第一件事是检查 xp \_ cmdshell ()扩展过程是否存在并已启用: 如果已禁用，则重新启用它; 如果创建失败，则从头开始创建它。如果攻击者想要命令标准输出:

• Create a support table with one field, data-type text;

创建一个支持表，其中包含一个字段，数据类型为文本;

• Execute the command via xp\_cmdshell() procedure redirecting its standard output to a temporary file;

•通过 xp \_ cmdshell ()过程执行命令，将其标准输出重定向到一个临时文件;

• Use BULK INSERT statement to load the content of the temporary file as a single entry into the support table;

使用 BULK INSERT 语句将临时文件的内容作为一个条目加载到支持表中;

• Remove the temporary file via xp\_cmdshell();

•通过 xp \_ cmdshell ()删除临时文件;

• Retrieve the content of the support table's entry;

•检索支持表条目的内容;

• Delete the content of support table. Otherwise:

•删除支持表的内容，否则:

• Execute the command via xp\_cmdshell() procedure.

•通过 xp \_ cmdshell ()过程执行命令。
