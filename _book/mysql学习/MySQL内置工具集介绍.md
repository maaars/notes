# MySQL内置工具集介绍

### **MySQL**内置工具集

MySQL数据库管理系统提供了许多命令行工具，这些工具可以用来管理MySQL服务器、对数据库进行访问控制、管理MySQL用户以及数据库备份和恢复工具等。而且MySQL提供图形化的管理工具，这使得对数据库的操作更加简单。下面将介绍常用工具的作用。

**MySQL**安装相关程序

```
mysql_install_db：初始化MySQL数据目录程序

mysql_secure_installation：MySQL数据库安装完用来提高安全性程序

mysql_safe：这是一个脚本程序，用来安全启动Mysqld程序

mysql_tzinfo_to_sql：加载时区表

mysql_upgrade：检查升级MySQL表
```



**MySQL**客户端程序

```
mysql：MySQL命令行程序

mysqladmin：用于管理MySQL服务器

mysqldump：逻辑备份程序

mysqlcheck：一个表检查维护工具

mysqlimport：数据导入程序数据工具

mysqlshow：显示数据库，表，列信息

mysqlslap：附在仿真客户端
```

**MySQL**管理和实用程序

```
mysqlaccess：检查访问权限的客户端

mysqlbinlog：用于处理二进制日志文件

mysqldumpslow：用于慢查询日志文件

mysqlhotcopy：数据备份工具
```

这里写出来的都是一些基本常用的MySQL程序，当然还有一部分没有写出来，也不太常用。具体可以查看官网: http://dev.mysql.com/doc/refman/5.6/en/programs-installation.html

下面先介绍一些现在需要用到的MySQL程序，更多的程序工具在后面哪里用到就会讲到。

### 连接MySQL

连接MySQL操作是一个连接进程和MySQL数据库实例进行通信，从程序设计上来说，本质上是进程通信。如果对进程通信比较了解，可以知道常用的进程通信方式有管道、命令管道、TCP/IP套接字、UNIX域套接字。

**TCP/IP**

TCP/IP套接字方式是MySQL数据库在任何平台下都提供的连接方式，也是网络中使用得最多的一种方式。这种方式在TCP/IP连接上建立一个基于网络的连接请求，一般情况下客户端在一台服务器上，而MySQL实例在另一台服务器上，这两台服务器通过一个TCP/IP网络连接。例如在Windows下使用Sqlyog工具进行连接远程MySQL实例就是通过TCP/IP连接。（需要输入MySQL的IP地址及端口号，也称为一个套接字。同时MySQL实例必须要允许远程的IP地址和用户名进程连接访问，默认无法连接。）

**共享内存**

MySQL提供共享内存的连接方式看着是通过在配置文件中添加–shared-memory实现的，如果想使用共享内存（属于进程间通信）的方式，在MySQL命令行客户端连接时还必须使用–protocol=memory选项即可。

**UNIX域套接字**

在Linux和Unix环境下，还可以使用Unix域套接字。Unix域套接字不是一个网络协议，所以只能在MySQL客户端和数据库实例在一台服务器上的情况下使用。可以再配置文件中指定套接字文件的路径，如**--socket=/tmp/mysql.sock**，一般在数据库安装时就指定好了路径。当数据库实例启动后，用户就可以通过Uuix域套接字进行连接。

### Mysql：MySQL命令行程序

MySQL命令行是MySQL提供的一个命令行客户端工具，MySQL客户端工具并不具体指某个客户端软件，事实上MySQL客户端是一种复合概念。包含不同程序语言编写的前端应用程序，如Sqlyog是一个非常好用的图形化界面客户端连接工具，MySQL自带的基于命令行的客户端，phpmyadmin是一个web界面的客户端，以及调用API接口等都称为MySQL客户端。

**mysql命令行连接服务器**

```
[root@localhost ~]#

--help：显示帮助信息

-uroot：指定登录用户

-predhat：指定登录密码（密码与-p之间不能有空格，否则无法识别）

-h localhost：指定要登录的主机（这里登录的主机名必须在Mysql库中有授权才可以）

-e "select * from mysql.user;"：在Linux命令行中执行SQL语句

-s -e ""：可以去掉MySQL显示边框

-N -s -e ""：可以去掉MySQL显示边框和标题信息

--socke=/tmp/mysql.sock：指定套接字文件（不指定会去默认位置查找）

--port：指定端口

--protocol=memory：指定连接协议{tcp|socket|pipe|memory}

--html：以HTML格式显示数据内容

--xml：以XML格式显示数据内容
```

示例：

```
[root@localhost ~]# mysql -uroot -predhat -h localhost --port 3306 --socket=/tmp/mysql.sock -s -N -e "select @@VERSION" 5.5.28
```

**mysql命令**

MySQL命令行客户端命令执行有两个模式：交互式模式和批处理模式

交互式模式

客户端命令(不需要在服务器上执行而是由客户端从服务端获取信息)

```
mysql> help
  #查看命令帮助
mysql> charset utf8
  #改变默认字符集
mysql> status
  #获取MYSQL的工作状态，如当前默认数据，字符集等
mysql> use DATABASE_NAME
  #设置默认数据库
mysql> delimiter \\
  #设置SQL语句结束符,存储过程中常用（默认是;号）
mysql> quit
  #退出MySQL
mysql> system
  #在Mysql登录提示符下操作Linux主机如"system ls /root"命令
mysql> prompt MY>
  #修改MySQL的登录提示符为“my>”
mysql> select * from mysql.user
  -> \c
  #\c是结束命令，一般用于退出当忘记使用SQL结束符进入的模式
mysql> select * from mysql.user\G
  #\G做SQL语句的结束符并以纵向显示输出结果
```

批处理模式

```
一般用于导入以.sql结尾的命令集，有以下三种方式：
1.[root@localhost ~]# mysql -u root -h localhost < /root/mysql.sql
2.mysql> source /root/mysql.sql
3.mysql> \. /root/mysql.sql
 
命令行编辑功能
mysql> ctrl+a：快速移动光标至行首
mysql> ctrl+e：快速移动光标至行尾
mysql> ctrl+w：删除光标之前的单词
mysql> ctrl+u：删除行首至光标处的所有内容
mysql> ctrl+y：粘贴使用ctrl+w或ctrl+u删除的内容
```

### Mysqladmin：管理MySQL服务器客户端工具

```
[root@localhost ~]# mysqladmin -u root -h localhost -p [Command] [arguments]
-u root：指定登录用户
-h localhost：指定要登录的主机（本机可省略）
-p：指定密码，密码可以直接跟在-p后面
 
[Command]
passwd 'new password'  [-p 'old password']
  #设置密码，其中“-p ’old password’”只有更换密码时才需要执行指定旧密码，第一次指定密码时不需要
create DATABASE_NAME
  #创建数据库
drop DATABASE_NAME
  #删除数据库
processlist
  #显示当前主机正在执行的任务列表
status
  #显示Mysql的服务器状态信息，如启动时间
status --sleep 2 --count 2
  #2秒显示一次一共显示2此
extended-status
  #显示MYSQL状态变量及其值
variables
  #显示MYSQL服务器变量及其值
shutdown
  #终止MySQL服务
flush-privileges
  #刷新授权表
flush-threads
  #清空线程缓存
flush-status
  #重置大多数的服务器状态变量
flush-logs
  #刷新日志
start-slave
stop-slave
  #启动和关闭Slave
version
  #查看MYSQL的版本和状态信息
```

### Mysql_safe：这是一个脚本程序，用来安全启动Mysqld服务

在上面我们查看mysql进程的时候发现，不光有mysqld还有一个使用bash执行的mysqld_safe程序。mysqld_safe是用来安全启动mysqld程序的，一般直接运行mysqld程序来启动MySQL服务的方法很少见，mysqld_safe增加了一些安全特性，例如当出现错误时重启服务器并向错误日志文件写入运行时间信息。

在类UNIX系统中推荐使用mysqld_safe来启动mysqld服务器。mysqld_safe从选项配置文件的[mysqld]、[server]和 [mysqld_safe]部分读取所有选项。其实我们平时使用mysql.server脚本其实也是调用mysqld_safe脚本去启动MySQL服务器的，所以如果你直接去启动mysqld程序而不是用脚本的话一般都会有问题。

mysqld_safe通常做如下事情：

1.检查系统和选项。

2.检查MyISAM表。

3.保持MySQL服务器窗口。

4.启动并监视mysqld，如果因错误终止则重启。

5.将mysqld的错误消息发送到数据目录中的host_name.err 文件。

6.将mysqld_safe的屏幕输出发送到数据目录中的host_name.safe文件。