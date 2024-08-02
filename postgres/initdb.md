# Postgresql的initdb的过程

## initdb

由于笔者现在在给openGauss开发，所以还会额外添加一些openGauss的流程，不过还是以PG为主。
先看PG的吧。主要流程都是顺序执行的，所以我们下面列出来的点就按照执行顺序。

- 解析命令参数
- 检查权限authmethod和authhost
- 设置PGDATA,PATH,LC等环境变量
- 创建数据文件目录
  - 设置信号处理函数
  - 创建xlog目录
  - 创建数据文件目录的各个子目录
  - 设置配置文件
  - 创建template1
  - 写入version文件
  - 初始化设置权限信息和密码
  - 初始化system_views, constraints, functions等系统对象
  - 加载PLSQL模块
  - 执行vacuumdb压缩数据库空间
  - 创建template1和postgres默认初始数据库

创建的这些数据库template1, template0, postgres初始数据库的作用是：
template1: 最初始的模板数据库，这个数据库的配置通过initdb完成，一旦initdb完成则配置完成；以后创建的任何数据库默认模板都会采用这个。
template0: 干净的备份数据库模板。
postgres: 为用户提供的缺省的默认数据库，以template1为模板创建。
我们在执行SQL时也可以创建自己的模板数据库并使用这个模板数据库创建数据库，这个略。

initdb时，各个可选参数的含义如下，执行initdb -?即可显示详情。
-A, --auth=METHOD         default authentication method for local connections
    --auth-host=METHOD    default authentication method for local TCP/IP connections
    --auth-local=METHOD   default authentication method for local-socket connections
  
顾名思义，鉴权使用的方法，有trust, reject, md5, sha256, sm3等数字签名算法
-D, --pgdata          数据目录
-E, --encoding        默认编码
-k, --data-checksums  启用数据页的校验和
-lc-xx    设置地区与字符集信息
--pwfile  在文件中读取密码，把文件中的第一行视为密码
-U，-W    账密相关选项
-X        WAL数据目录
-c        设置服务器变量
-n, -d    调试选项
-V        版本号            

作为基于PG的数据库openGauss，initdb的新增参数可以在OG的文档找到，本文略。[点我打开](https://docs-opengauss.osinfra.cn/zh/docs/6.0.0-RC1/docs/ToolandCommandReference/gs_initdb.html)

## pg_ctl

在initdb完成后，也可以简单介绍下pg_ctl。
pg_ctl是Postgresql服务端的控制工具，支持直接在pg_ctl中执行initdb, start, stop, restart, reload等命令操作数据。
直接看使用-?选项打出来的命令参数及解释即可。

Usage:
  pg_ctl init[db]   [-D DATADIR] [-s] [-o OPTIONS]
  pg_ctl start      [-D DATADIR] [-l FILENAME] [-W] [-t SECS] [-s]
                    [-o OPTIONS] [-p PATH] [-c]
  pg_ctl stop       [-D DATADIR] [-m SHUTDOWN-MODE] [-W] [-t SECS] [-s]
  pg_ctl restart    [-D DATADIR] [-m SHUTDOWN-MODE] [-W] [-t SECS] [-s]
                    [-o OPTIONS] [-c]
  pg_ctl reload     [-D DATADIR] [-s]
  pg_ctl status     [-D DATADIR]
  pg_ctl promote    [-D DATADIR] [-W] [-t SECS] [-s]
  pg_ctl logrotate  [-D DATADIR] [-s]
  pg_ctl kill       SIGNALNAME PID

Common options:
  -D, --pgdata=DATADIR   location of the database storage area
  -s, --silent           only print errors, no informational messages
  -t, --timeout=SECS     seconds to wait when using -w option
  -V, --version          output version information, then exit
  -w, --wait             wait until operation completes (default)
  -W, --no-wait          do not wait until operation completes
  -?, --help             show this help, then exit
If the -D option is omitted, the environment variable PGDATA is used.

Options for start or restart:
  -c, --core-files       allow postgres to produce core files
  -l, --log=FILENAME     write (or append) server log to FILENAME
  -o, --options=OPTIONS  command line options to pass to postgres
                         (PostgreSQL server executable) or initdb
  -p PATH-TO-POSTGRES    normally not necessary

Options for stop or restart:
  -m, --mode=MODE        MODE can be "smart", "fast", or "immediate"

Shutdown modes are:
  smart       quit after all clients have disconnected
  fast        quit directly, with proper shutdown (default)
  immediate   quit without complete shutdown; will lead to recovery on restart

Allowed signal names for kill:
  ABRT HUP INT KILL QUIT TERM USR1 USR2

感觉已经写得比较明白了。

## 步骤细化

### bootstrap_template1

功能：通过在bootstrap模式下运行BKI脚本来生成template1
流程：
1. 使用`readfile`函数读取一个文件，将每一行存储在`bki_lines`数组中。
2. 检查读取的文件内容是否符合预期的版本号。通过比较文件的第一行与预定义的版本号。
3. 替换`bki_lines`数组中的一些占位符，替换规则如下：
   - 将`NAMEDATALEN`替换为`NAMEDATALEN`的值。
   - 将`SIZEOF_POINTER`替换为`sizeof(Pointer)`的值。
   - 将`ALIGNOF_POINTER`替换为条件表达式 `(sizeof(Pointer) == 4) ? "i" : "d"`的结果。
   - 将`FLOAT8PASSBYVAL`替换为`FLOAT8PASSBYVAL`的值，如果为真则替换为"true"，否则替换为"false"。
   - 将`POSTGRES`替换为`username`的值。
   - 将`ENCODING`替换为`encodingid`的字符串表示。
   - 将`LC_COLLATE`替换为`lc_collate`的值。
   - 将`LC_CTYPE`替换为`lc_ctype`的值。
   - 将`ICU_LOCALE`替换为`icu_locale`的值，如果为非空则替换为对应的字符串，否则替换为"_null_"。
   - 将`ICU_RULES`替换为`icu_rules`的值，如果为非空则替换为对应的字符串，否则替换为"_null_"。
   - 将`LOCALE_PROVIDER`替换为`locale_provider`的值。
4. 使用`unsetenv`函数取消设置`PGCLIENTENCODING`环境变量。
5. 使用`snprintf`函数构建一个命令字符串，并将其存储在`cmd`数组中。
6. 使用`PG_CMD_OPEN`打开一个与命令相关联的子进程。
7. 使用循环将`bki_lines`数组中的每一行写入到子进程的输入流。
8. 使用`PG_CMD_CLOSE`关闭与子进程相关联的输入流，并等待子进程的结束。
9. 释放`bki_lines`数组中每一行的内存空间。
10. 检查子进程的执行结果。

### load_plpgsql

执行过程与template1类似，都是通过命令的形式，在template1模板库里创建plpgsql插件，需要注意的是只有加载了plpgsql，才能使用plpgsql语言创建函数（包括创建package等），所以执行上会有顺序依赖的问题，在添加数据库对象需要注意该问题。类似的几个函数都非常简单，都是在函数内部调用SQL语句来执行相应的命令。以load_plpgsql函数为例，主要功能使用几个PG_CMD开头的宏来实现，其中PG_CMD_PUTS为执行SQL语句的接口，在load_plpgsql函数中执行的语句是CREATE EXTENSION plpgsql；功能与作用与在server中执行的SQL语句一致。

### 安装系统视图

通过setup_sysviews函数实现，执行system_views.sql文件来创建系统视图，需要说明的是因为一些历史原因，该文件里还存在大量创建package或function的SQL语句，该部分是所有兼容模式共享的，后续的工作中（2.2.14版本以后），应尽量避免该类操作，需要使用SQL语句创建的函数或包，应放在对应兼容模式下的XX_object.sql文件里。

### 安装区分兼容模式的对象（openGauss特有）

通过setup_XX_object函数实现，本质是调用XX_object.sql文件，值得注意的是，文件不止能够创建plpgsql语言编写的函数，也可以使用指定so文件路径的方式，创建C语言的函数。除了函数之外，其他与兼容模式高度绑定的对象，如：表、视图、数据类型、数据类型转换、函数、存储过程、触发器等数据对象，均可以在对应的sql文件里创建。

### 创建其他数据库

template0、postgres均是通过copy template1得到。
