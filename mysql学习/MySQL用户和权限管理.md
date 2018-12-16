# MySQL用户和权限管理

### **一、MySQL权限体系**

MySQL的认证是“用户”加“主机”形式，而权限是访问资源对象，MySQL服务器通过权限表来控制用户对数据库的访问，权限表存放在mysql数据库中，初始化数据库时会初始化这些权限表。存储账户权限信息表主要有：user，db，tables_priv，columns_priv，procs_priv这五张表（5.6之前还有host表，现在已经把host内容整合进user表）。

[官方文档](https://dev.mysql.com/doc/refman/5.7/en/privileges-provided.html)对权限有比较详细的描述，为了方便我把其中的表格列在下面。第一列表示所有的权限，可以在 Grant 语句中指定的，第二列是对应权限存储在系统数据库 mysql 几张表中的定义，第三列表示权限作用的范围，其中 Global（Server administration）对应 mysql.user 表，Database 对应 mysql.db 表，Tables 对应 mysql.tables_priv 表，Columns 对应 mysql.columns_priv 表，Stored routines 对应 mysql.procs_priv 表。

| Privilege               | Column                       | Context                               |
| ----------------------- | ---------------------------- | ------------------------------------- |
| ALL [PRIVILEGES]        | Synonym for “all privileges” | Server administration                 |
| **ALTER**               | Alter_priv                   | Tables                                |
| ALTER ROUTINE           | Alter_routine_priv           | Stored routines                       |
| **CREATE**              | Create_priv                  | Databases, tables, or indexes         |
| CREATE ROUTINE          | Create_routine_priv          | Stored routines                       |
| CREATE TABLESPACE       | Create_tablespace_priv       | Server administration                 |
| CREATE TEMPORARY TABLES | Create_tmp_table_priv        | Tables                                |
| CREATE USER             | Create_user_priv             | Server administration                 |
| CREATE VIEW             | Create_view_priv             | Views                                 |
| **DELETE**              | Delete_priv                  | Tables                                |
| **DROP**                | Drop_priv                    | Databases, tables, or views           |
| EVENT                   | Event_priv                   | Databases                             |
| EXECUTE                 | Execute_priv                 | Stored routines                       |
| FILE                    | File_priv                    | File access on server host            |
| GRANT OPTION            | Grant_priv                   | Databases, tables, or stored routines |
| INDEX                   | Index_priv                   | Tables                                |
| **INSERT**              | Insert_priv                  | Tables or columns                     |
| LOCK TABLES             | Lock_tables_priv             | Databases                             |
| PROCESS                 | Process_priv                 | Server administration                 |
| PROXY See               | proxies_priv                 | table Server administration           |
| REFERENCES              | References_priv              | Databases or tables                   |
| RELOAD                  | Reload_priv                  | Server administration                 |
| REPLICATION CLIENT      | Repl_client_priv             | Server administration                 |
| REPLICATION SLAVE       | Repl_slave_priv              | Server administration                 |
| **SELECT**              | Select_priv                  | Tables or columns                     |
| SHOW DATABASES          | Show_db_priv                 | Server administration                 |
| SHOW VIEW               | Show_view_priv               | Views                                 |
| SHUTDOWN                | Shutdown_priv                | Server administration                 |
| SUPER                   | Super_priv                   | Server administration                 |
| TRIGGER                 | Trigger_priv                 | Tables                                |
| **UPDATE**              | Update_priv                  | Tables or columns                     |
| **USAGE**               | Synonym for “no privileges”  | Server administration                 |

[GRANT语句](https://dev.mysql.com/doc/refman/5.7/en/grant.html)赋予对应用户相应的权限，会根据不同的语法存储到不同的表中，以链接中官方文档中的语句为例：

- **Global Privileges**

| Privilege               | Column                       | Context                               |
| ----------------------- | ---------------------------- | ------------------------------------- |
| ALL [PRIVILEGES]        | Synonym for “all privileges” | Server administration                 |
| ALTER                   | Alter_priv                   | Tables                                |
| ALTER ROUTINE           | Alter_routine_priv           | Stored routines                       |
| CREATE                  | Create_priv                  | Databases, tables, or indexes         |
| CREATE ROUTINE          | Create_routine_priv          | Stored routines                       |
| CREATE TABLESPACE       | Create_tablespace_priv       | Server administration                 |
| CREATE TEMPORARY TABLES | Create_tmp_table_priv        | Tables                                |
| CREATE USER             | Create_user_priv             | Server administration                 |
| CREATE VIEW             | Create_view_priv             | Views                                 |
| DELETE                  | Delete_priv                  | Tables                                |
| DROP                    | Drop_priv                    | Databases, tables, or views           |
| EVENT                   | Event_priv                   | Databases                             |
| EXECUTE                 | Execute_priv                 | Stored routines                       |
| FILE                    | File_priv                    | File access on server host            |
| GRANT OPTION            | Grant_priv                   | Databases, tables, or stored routines |
| INDEX                   | Index_priv                   | Tables                                |
| INSERT                  | Insert_priv                  | Tables or columns                     |
| LOCK TABLES             | Lock_tables_priv             | Databases                             |
| PROCESS                 | Process_priv                 | Server administration                 |
| PROXY See               | proxies_priv                 | table Server administration           |
| REFERENCES              | References_priv              | Databases or tables                   |
| RELOAD                  | Reload_priv                  | Server administration                 |
| REPLICATION CLIENT      | Repl_client_priv             | Server administration                 |
| REPLICATION SLAVE       | Repl_slave_priv              | Server administration                 |
| SELECT                  | Select_priv                  | Tables or columns                     |
| SHOW DATABASES          | Show_db_priv                 | Server administration                 |
| SHOW VIEW               | Show_view_priv               | Views                                 |
| SHUTDOWN                | Shutdown_priv                | Server administration                 |
| SUPER                   | Super_priv                   | Server administration                 |
| TRIGGER                 | Trigger_priv                 | Tables                                |
| UPDATE                  | Update_priv                  | Tables or columns                     |
| USAGE                   | Synonym for “no privileges”  | Server administration                 |

[GRANT语句](https://dev.mysql.com/doc/refman/5.7/en/grant.html)赋予对应用户相应的权限，会根据不同的语法存储到不同的表中，以链接中官方文档中的语句为例：

- **Global Privileges**

```
GRANT ALL ON *.* TO 'someuser'@'somehost';
GRANT SELECT, INSERT ON *.* TO 'someuser'@'somehost';
```

其中 *.* 表示所有数据的所有表，对应的权限会保存在 mysql.user 表中，和 user 相关联。

**user表**

user表是MySQL中最重要的一个权限表，记录允许连接到服务器的账号信息，里面的权限是全局级的。例如：一个用户在user表中被授予了DELETE权限，则该用户可以删除MySQL服务器上所有数据库的任何记录。user表中大概有45个字段，这些字段大概可以分为4类，分别是用户列、权限列、安全列和资源控制列，详细解释如下：

1）Host、User：表示主机和用户，是user表的主键。

2）*-priv：此类型的字段都是权限列，权限列的字段决定了用户的权限，描述了全局范围内允许对数据和数据库进行的操作。包括查询、修改、删除等普通权限，还有包括了关闭服务器、超级权限和加载用户等高级权限。user表对应的权限是针对所有用户数据库的，这些字段的类型为ENUM，可以取的值只能为Y或N，Y表示该用户有对应的权限；N表示没有。

3）ssl_type、ssl_cipher、x509_issuer、x509_subject：这几个字段是安全连接相关的，SSL和证书。

4）max_questions、max_updates、max_connections、max_user_connections：这几个字段是**用户资源限制相关的**，如此用户某个时间内最大查询、更新、连接等操作。

5）authentication_string、account_locked、password_lifetime、password_last_changed：这几个字段是跟用户密码相关的，部分都是MySQL 5.7新增字段，作用分别是标记账号锁定、密码存活时间和密码改变时间；

MySQL 5.7去掉了password字段，由authentication_string替换，因为5.7更换了密码插件，因此存储密码的字段更换了。

- **Database Privileges**

```
GRANT ALL ON mydb.* TO 'someuser'@'somehost';
GRANT SELECT, INSERT ON mydb.* TO 'someuser'@'somehost';
```

其中 mydb.* 表示 mydb 数据库下的所有表，对应的权限会保存在 mysql.db 表中，和 db 相关联。

**db表**

db表存储了用户对某个数据库的操作权限，决定用户能从哪个主机哪个用户来操作哪个数据库。User表中存储了某个主机和用户对数据库的操作权限，配置和db权限表对给定主机上数据库级操作权限做更细致的控制。这个权限表不受GRANT和REVOKE语句的影响，字段大致可以分为两类：用户列和权限列，详细解释如下：

1）Host、Db、User：表示主机、数据库和用户，是db表的主键。

2）*-priv：此类型的字段都是权限列，权限列的字段决定了用户的权限。user表的权限时针对所有数据库的，全局的；但如果希望某个用户只对某个数据库有相应的查询、修改、删除等普通权限，那么就需要在db表中设定，而user表只有对应的主机、用户和密码等信息。这些字段的类型为ENUM，可以取的值只能为Y或N，Y表示该用户有对应的权限；N表示没有。

- **Table Privileges**

```
GRANT ALL ON mydb.mytbl TO 'someuser'@'somehost';
GRANT SELECT, INSERT ON mydb.mytbl TO 'someuser'@'somehost';
```

对应的权限保存在 mysql.tables_priv 中，和 db , user 关联。

**tables_priv表**

tables_priv表用来对表设置操作权限，有几个字段分别是Host、Db、User、Table_name、Grantor、Timestamp、Table_priv和Column_priv，各个字段说明如下：

1）Host、Db、User和Table_name这几个字段分表示主机名、数据库名、用户名和表名。

2）Grantor字段表示修改该记录的用户。

3）Timestamp字段表示修改该记录的时间。

4）Table_priv字段表示对表的操作权限，包括set(‘Select’,’Insert’,’Update’,’Delete’,’Create’,’Drop’,’Grant’,’References’,’Index’,’Alter’,’Create View’,’Show view’,’Trigger’)。

5）Column_priv字段表示对表中的列的操作权限，包括set(‘Select’,’Insert’,’Update’,’References’)。

- **Column Privileges**

```
GRANT SELECT (col1), INSERT (col1,col2) ON mydb.mytbl TO 'someuser'@'somehost';
```

对应的权限保存在 mysql.tables_priv 中，和 db, table, user 关联。

**columns_priv表**

columns_priv表用来对表设置操作权限，有这么几个字段分别是Host、Db、User、Table_name、Timestamp、Column_name和Column_priv，各个字段说明如下：

1）Host、Db、User和Column_name这几个字段分表示主机名、数据库名、用户名和列名。

2）Timestamp字段表示修改该记录的时间。

3） Column_priv字段表示对表中的列的操作权限，包括set(‘Select’,’Insert’,’Update’,’References’)。

- **Stored Routine Privileges**

```
GRANT CREATE ROUTINE ON mydb.* TO 'someuser'@'somehost';
GRANT EXECUTE ON PROCEDURE mydb.myproc TO 'someuser'@'somehost';
```

对应的权限保存在 mysql.procs_priv 中，和 routine_name， db，user 关联。

**procs_priv表**

存储过程和存储函数相关的权限，分别是Host、Db、User、Routine_name、Routine_type、Grantor、Proc_priv和Timestamp，各个字段的说明如下：

1）Host、Db和User字段分别表示主机名、数据库名和用户名；Routine_name表示存储过程或函数的名称。

2）Routine_type字段表示存储过程或函数的类型。

3）Routine_type字段有两个值，分别是FUNCTION和PROCEDURE。FUNCTION表示这是一个函数；PROCEDURE表示这是一个存储过程。

4）Grantor字段记录是插入或修改该记录的用户。

5）Proc_priv字段表示拥有的权限，包括Execute、Alter Routine、Grant这3种。

6）Timestamp字段表示记录更新时间。

### **二、MySQL****访问控制两阶段**

阶段1：客户端连接核实阶段

阶段2：客户端操作核实阶段

客户端连接核实阶段，当连接MySQL服务器时，服务器基于用户的身份以及用户是否能通过正确的密码身份验证，来接受或拒绝连接。即客户端用户连接请求中会提供用户名称、主机地址和密码，MySQL使用user表中的三个字段（Host、User、Password）执行身份检查，服务器只有在user表记录的Host和User字段匹配客户端主机名和用户名，并且提供正确的面貌时才接受连接。如果连接核实没有通过，服务器完全拒绝访问；否则，服务器接受连接，然后进入阶段2等待用户请求。

客户端操作核实阶段，当客户端的连接请求被MySQL服务器端通过其身份认证后。那么接下来就可以发送数据库的操作命令给服务器端处理，服务器检查用户要执行的操作，在确认权限时，MySQL首先检查user表，如果指定的权限没有在user表中被授权；MySQL将检查db表，db表时下一安全层级，其中的权限限定于数据库层级，在该层级的SELECT权限允许用户查看指定数据库的所有表中的数据；如果在该层级没有找到限定的权限，则MySQL继续检查tables_priv表以及columns_priv表，如果所有权限表都检查完毕，但还是没有找到允许的权限操作，MySQL将返回错误信息，用户请求的操作不能执行，操作失败。

### **三、MySQL****用户及密码管理**

MySQL提供许多语句用来管理用户账号，这些语句可以用来管理包括登陆和退出MySQL服务器、创建用户、删除用户、密码管理和权限管理等内容。MySQL数据库的安全性，需要通过账户管理来保证。下面介绍四种用来管理账号密码的方式：

**3.1 通过mysqladmin工具（只能改密码）**

```
# 给root@localhost用户登录mysql设置密码为"redhat";
$ mysqladmin -u root -h localhost password "redhat" 
 
# 修改root@localhost用户登录mysql数据库的密码;
$ mysqladmin -u root -h localhost password "new passwd" -p "old passwd"
```

**3.2 通过CREATE USER语句**

```
mysql> create user 'USERNAME'@'HOST' identified by 'PASSWORD';
```

创建登录用户，MySQL的登录用户必须是’USERNAME’@’HOST’（用户名加主机名），如’mysql’@’172.16.16.1′，含义是只有在172.16.16.1这台主机上才可以使用mysql用户登录MySQL数据库（还可以指定只允许登录那个数据库）。

**HOST的表现方式：**

\1. IP地址，如172.16.16.1；

\2. 主机名，如localhost；

\3. 网络地址，如172.16.0.0

\4. 通配符，如

%：匹配任意字符

_：匹配任意单个字符如172.16.16._(允许172.16.16.1-172.16.16.9)

```
# 创建用户允许任何主机登录以USERNAME用户登录;
mysql> create user 'USERNAME'@'%' identified by 'PASSWORD';   
# 更改用户名;
mysql> rename user OLD_NAME to NEW_NAME; 
# 删除登录用户;
mysql> drop user 'USERNAME'@'HOST'; 
# 删除MySQL默认的无用账户;
mysql> drop user 'root'@'localhost.localdomain';
# 删除MySQL默认的无用账户;
mysql> drop user 'root'@'127.0.0.1';
```

**3.3 通过直接修改mysql.user表的用户记录**

```
# MySQL 5.6
mysql> update mysql.user set password=PASSWORD('redhat') where user='root';

 # MySQL 5.7
mysql> update mysql.user set authentication_string=PASSWORD('redhat') where user='root';
```

或

```
mysql> set password for 'root'@'localhost'=PASSWORD('redhat');
```

改完记得刷新内存中现有的表，另外这种形式在MySQL 5.7中使用会被警告，会告诉你这是一个即将被移除的特性。MySQL 5.7提供了新的更改密码的方式ALTER USER语句。

```
mysql> use mysqlmysql> alter user root@'localhost' identified by '123456';
```

**4）通过GRANT指令（只能用于添加新用户）**

```
# 创建tom用户并对此库下的所有表赋予所有权限;mysql> grant all on DB_NAME.* to 'tom'@'localhost' identified by '1234';
```

同样，通过grant创建用户在MySQL 5.7中使用会被警告，是一个即将被移除的特性。推荐使用CREATE USER语句。

虽然介绍了好几种方法创建用户，但真正在使用中，最好按照规范使用CREATE USER创建用户，GRANT设置权限，ALTER USER更改密码，而不要直接将用户信息插入user表中，因为user表中存储了全局级别的权限以及其他的账户信息，如果意外破坏了user表中的记录，则可能会对MySQL服务器造成很大的影响。

### **四、MySQL**管理员密码找回

**4.1 关闭MySQL**

```
$ service mysqld stop
```

**4.2 在配置文件中[mysqld]字段添加skip-grant-tables指令，跳过授权表**

```
$ cat /etc/my.cnf 
[mysqld]
skip-grant-tables
```

**4.3 给root用户登录mysql设置密码为redhat并以加密方式**

```
Mysql> use mysql;Mysql> update user set password=PASSWORD('redhat') where user='root';

MySQL5.7修改密码
mysql&gt; update mysql.user set authentication_string=PASSWORD('redhat') where user='root';
```



### **五、MySQL**权限管理

权限管理主要是对登录到MySQL的用户进行权限验证，所有用户的权限都存储在MySQL的权限表中，不合理的权限规划会给MySQL服务器带来安全隐患。数据库管理员要对所有用户的权限进行合理规划管理。MySQL权限系统的主要功能时证实连接到一台给定主机的用户，并且赋予该用户在数据库上的SELECT/INSERT/UPDATE和DELETE权限。

**1）MySQL权限说明**

账户权限信息被存储在MySQL数据库的几张权限表中，在MySQL启动时，服务器将这些数据库表中权限信息的内容读入内存。其中GRANT和REVOKE语句所涉及的常用权限大致如下这些：CREATE、DROP、SELECT、INSERT、UPDATE、DELETE、INDEX、ALTER、CREATE、ROUTINE、FILE等，还有一个特殊的proxy权限，是用来赋予某个用户具有给他人赋予权限的权限。

**2）MySQL用户授权**

授权就是为某个用户授予权限，合理的授权可以保证数据库的安全，MySQL中可以使用GRANT语句为用户授予权限。授权可以分为多个层次：

全局层级：全局权限适用于一个给定服务器中的所有数据库，这些权限存储在mysql.user表中。

数据库层级：数据库权限适用于一个给定数据库中的所有目标，这些权限存储在mysql.db表中。

表层级：表权限适用于一个给定表中的所有列，这些权限存储在mysql.tables_priv表中。

列层级：列权限使用于一个给定表中的单一列，这些权限存储在mysql.columns_priv表中。

子程序层级：CREATE ROUTINE、ALTER ROUTINE、EXECUTE和GRANT权限适用于已存储的子程序。这些权限可以被授予为全局层级和数据库层级。而且，除了CREATE ROUTINE外，这些权限可以被授予子程序层级，并存储在mysql.procs_priv表中。

PS：MySQL中必须拥有GRANT权限的用户才可以执行GRANT语句。

**2.1 GRANT赋予用户权限**

```
# 定义对已经存在的用户可以操作此库下的所有表及所有权限；
Mysql> grant all privileges on DB_NAME.* to 'USERNAME'@'HOST';
 
# 创建tom用户并赋予select权限对此库下的所有表；
Mysql> grant select on DB_NAME.* to 'tom'@'localhost' identified by '1234';
 
# 定义tom用户赋予insert权限对db库下的xsb表； 
Mysql> grant insert on db.xsb to 'tom'@'localhost'; 
 
# 定义tom用户赋予update权限对db库下的xsb表；                      
Mysql> grant update on db.xsb to 'tom'@'localhost'; 
 
# 定义tom用于赋予update权限对db库下的xsb表中的AGE字段；
Mysql> grant update(AGE) on db.xsb to 'tom'@'localhost';
 
# 定义tom用于赋予super权限在*.*上(super权限可以对全局变量更改)；
Mysql> grant super on *.* to 'tom'@'%';
 
# 通过GRANT语句中的USAGE权限，你可以创建账户而不授予任何权限；它可以将所有全局权限设为'N'，假定你将在以后将具体权限授予该账户；
Mysql> grant usage on *.* to 'tom'@'%';
```

all表示赋予用户全部权限（包含存储过程、存储函数等创建和执行）。当数据库名称.表名称被*.*代替，表示赋予用户操作服务器上所有数据库所有表的权限。用户地址可以是localhost，也可以是ip地址、机器名字、域名。也可以用’%’表示从任何地址连接。而’连接口令’不能为空，否则创建失败。

**2.2 REVOKE移除用户权限**

```
# 移除tom用户对于db.xsb的权限;Mysql> revoke all on db.xsb from 'tom'@'localhost'; # 刷新授权表;Mysql> flush privileges;
```

**3）SHOW查看用户的权限**

```
Mysql> show grants for 'USERNAME'@'HOST';
```

PS：使用REVOKE收回权限之后，用户帐户的记录将从db、host、tables_priv、columns_priv表中删除，但是用户帐号记录依然在user表中保存。

**3）PROXY特殊权限**

如果想让某个用户具有给他人赋予权限的能力，那么就需要proxy权限了。当你给一个用户赋予all权限之后，你查看mysql.user表会发现Grant_priv字段还是为N，表示其没有给他人赋予权限的权限。

我们可以查看一下系统默认的超级管理员权限：

```
mysql> show grants for 'root'@'localhost';
+---------------------------------------------------------------------+
| Grants for root@localhost                                           |
+---------------------------------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION |
| GRANT PROXY ON ''@'' TO 'root'@'localhost' WITH GRANT OPTION        |
+---------------------------------------------------------------------+
2 rows in set (0.00 sec)
```

可以看到其本身有PROXY权限，并且这个语句跟一般授权语句还不太一样。所以如果想让一个远程用户有给他人赋予权限的能力，就需要给此用户PROXY权限，如下：

```
mysql> grant all on *.* to 'test'@'%' identified by '123456';
mysql> GRANT PROXY ON ''@'' TO 'test'@'%' WITH GRANT OPTION;
```

**4）数据库开发人员，创建表、索引、视图、存储过程、函数等权限授权**

grant创建、修改、删除MySQL数据表结构权限

```
grant create on testdb.* to developer@'192.168.0.%';

grant alter on testdb.* to developer@'192.168.0.%';

grant drop on testdb.* to developer@'192.168.0.%';
```

grant操作MySQL外键权限

```
grant references on testdb.* to developer@'192.168.0.%';
```

grant操作MySQL临时表权限。

```
grant create temporary tables on testdb.* to developer@'192.168.0.%';
```

grant操作MySQL索引权限

```
grant index on testdb.* to developer@'192.168.0.%';
```

grant操作MySQL视图、查看视图源代码 权限

```
grant create view on testdb.* to developer@'192.168.0.%';
grant show view on testdb.* to developer@'192.168.0.%';
```

grant操作MySQL存储过程、存储函数权限

```
grant create routine on testdb.* to developer@'192.168.0.%';
grant alter routine on testdb.* to developer@'192.168.0.%';
grant execute on testdb.* to developer@'192.168.0.%';
```

