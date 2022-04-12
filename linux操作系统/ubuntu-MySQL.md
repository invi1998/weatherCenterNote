## 通过apt 安装MySQL服务（推荐，会安装最新版）

```shell
#命令1 更新源
sudo apt-get update
#命令2 安装mysql服务
sudo apt-get install mysql-server
```



## 初始化配置

```shell
sudo mysql_secure_installation
```


配置项较多，如下所示：

```shell
#1
VALIDATE PASSWORD PLUGIN can be used to test passwords...
Press y|Y for Yes, any other key for No: N (选择N ,不会进行密码的强校验)

#2
Please set the password for root here...
New password: (输入密码)
Re-enter new password: (重复输入)

#3
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them...
Remove anonymous users? (Press y|Y for Yes, any other key for No) : N (选择N，不删除匿名用户)

#4
Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network...
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : N (选择N，允许root远程连接)

#5
By default, MySQL comes with a database named 'test' that
anyone can access...
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : N (选择N，不删除test数据库)

#6
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : Y (选择Y，修改权限立即生效)
```

## 检查mysql服务状态

```shell
systemctl status mysql.service
```

## 配置远程访问


在Ubuntu下MySQL缺省是只允许本地访问的，使用workbench连接工具是连不上的；
如果你要其他机器也能够访问的话，需要进行配置；

**找到 bind-address 修改值为 0.0.0.0(如果需要远程访问)**

```shell
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf #找到 bind-address 修改值为 0.0.0.0(如果需要远程访问)
sudo /etc/init.d/mysql restart #重启mysql
```


```shell
sudo mysql -uroot -p
```


输入用户密码

```shell
#切换数据库
mysql>use mysql;
#查询用户表命令：
mysql>select User,authentication_string,Host from user;
#查看状态
select host,user,plugin from user;
```


```shell
#设置权限与密码

mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '密码'; #使用mysql_native_password修改加密规则
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '密码' PASSWORD EXPIRE NEVER; #更新一下用户的密码
mysql> UPDATE user SET host = '%' WHERE user = 'root'; #允许远程访问

#刷新cache中配置 刷新权限
mysql>flush privileges; 
mysql>quit;
```

如果无法更改密码使用**flush privileges;然后再进行更改密码，修改加密规则操作。**

其中**root@localhost**，localhost就是本地访问，配置成 **%** 就是所有主机都可连接；

第二个**’密码’**为你给新增权限用户设置的密码，**%**代表所有主机，也可以是具体的**ip**；
**注意不要直接更新密码的编码格式，而不加密码，这样会把加密密码跟新了，需要携带密码**

**FLUSH PRIVILEGES;作用是：**

将当前user和privilige表中的用户信息/权限设置从mysql库(MySQL数据库的内置库)中提取到内存里。
MySQL用户数据和权限有修改后，希望在"不重启MySQL服务"的情况下直接生效，那么就需要执行这个命令。
通常是在修改ROOT帐号的设置后，怕重启后无法再登录进来，那么直接flush之后就可以看权限设置是否生效。
而不必冒太大风险。

**修改密码**

```shell
alter user 'root'@'%' identified with mysql_native_password by '密码';
```

新增用户赋权并设置远程访问
mysql8和原来的版本有点不一样，8的安全级别更高，所以在创建远程连接用户的时候，
不能用原来的命令（同时创建用户和赋权）:

```shell
#必须先创建用户（密码规则：mysql8.0以上密码策略限制必须要大小写加数字特殊符号）
mysql> CREATE USER 'sammy'@'%' IDENTIFIED WITH mysql_native_password BY 'password';
#赋权
mysql> GRANT ALL PRIVILEGES ON *.* TO 'sammy'@'%' WITH GRANT OPTION;
```

**修改加密方式：**

mysql8.0 引入了新特性 caching_sha2_password；这种密码加密方式Navicat 12以下客户端不支持；
Navicat 12以下客户端支持的是mysql_native_password这种加密方式；

```shell
update user set plugin='mysql_native_password' where user='root'
```


如果为了安全性，设置了用户验证，必须使用sudo，才能登录，出现如下情况：（尽量不要设置ubuntu用户在验证，否则会很麻烦）

解决方法：

```shell
sudo vim /etc/mysql/my.cnf
```

添加：

```shell
[mysqld]
skip-grant-tables
```

保存后重启mysql，可以正常登陆了
这样操作后，是相当于跳过了mysql的密码认证。很不安全，直接就可以登录进去。

## 新建数据库和用户

```shell
##1 创建数据库studentService
CREATE DATABASE studentService;
##2 创建用户teacher(密码admin) 并赋予其studentService数据库的远程连接权限
GRANT ALL PRIVILEGES ON teacher.* TO studentService@% IDENTIFIED BY "admin";
```



## mysql服务命令

```shell
# 检查服务状态
systemctl status mysql.service
# 或
sudo service mysql status
```


mysql服务启动停止

```shell
#停止
sudo service mysql stop
#启动
sudo service mysql start
```



## 数据库操作命令

### mysql服务操作

1、进入mysql数据库

```sql
mysql -u root -p
```


2、查看数据库版本

```sql
mysql-> status; 
```


3、退出mysql操作

```sql
mysql-> quit;
```


4、启动mysql服务

```sql
[root@szxdb etc]# service mysql start
```


5、停止mysql服务

```sql
[root@szxdb etc]# service mysql stop
```


6、重启mysql服务

```shell
 service mysql restart 
```


7、更改密码 ：mysqladmin -u用户名 -p旧密码 password 新密码

````sql
mysql-> mysqladmin -uroot -proot password 123456  
````

8、增加新用户 :grant select on 数据库.* to 用户名@登录主机 identified by “密码”

```sql
mysql-> grant all privileges on *.* to root@"%" identified by "pwd" with grant option;
```


增加一个用户test2密码为abc,让他只可以在localhost上登录，并可以对数据库mydb进行查询、插入、修改、删除的操作 （localhost指本地主机，即MYSQL数据库所在的那台主机），这样用户即使用知道test2的密码，他也无法从internet上直接访问数据 库，只能通过MYSQL主机上的web页来访问了。

```sql
mysql-> grant select,insert,update,delete on mydb.* to test2@localhost identified by "abc";
```


如果你不想test2有密码，可以再打一个命令将密码消掉。

```sqlite
mysql-> grant select,insert,update,delete on mydb.* to test2@localhost identified by "";
```


9、查看字符集

```sql
mysql-> show variables like 'character%'; 
```



### 数据库操作

创建数据库

```sql
create database 数据库名 charset=utf8;
```


删除数据库

```sql
drop database 数据库名;
```


切换数据库

```sql
use 数据库名;
```


查看当前选择的数据库

```sql
select database();
```


列出数据库

```sql
mysql-> show databases;
```



### 表操作

查看当前数据库中所有表

```sql
show tables;
```


创建表

```sql
auto_increment表示自动增长
create table 表名(列及类型);
如：
create table students(
id int auto_increment primary key,
sname varchar(10) not null
);
```



修改表

```sqlite
alter table 表名 add|change|drop 列名 类型;
# 如：
alter table students add birthday datetime;
```



删除表

```sql
drop table 表名;
```


查看表结构

```sql
desc 表名;
```


更改表名称

```sql
rename table 原表名 to 新表名;
```


查看表的创建语句

```sql
show create table '表名';
```



### 修改表结构

1、更改表得的定义把某个栏位设为主键。

```sql
ALTER TABLE tab_name ADD PRIMARY KEY (col_name) 
```


2、把主键的定义删除

```sql
ALTER TABLE tab_name DROP PRIMARY KEY (col_name) 
```


3、 在tab_name表中增加一个名为col_name的字段且类型为varchar(20)

```sql
alter table tab_name add col_name varchar(20); 
```


4、在tab_name中将col_name字段删除

```sql
alter table tab_name drop col_name;
```


5、修改字段属性，注若加上not null则要求原字段下没有数据

```sql
alter table tab_name modify col_name varchar(40) not null;

SQL Server200下的写法是：
Alter Table table_name Alter Column col_name varchar(30) not null;
```


6、如何修改表名：

```sql
alter table tab_name rename to new_tab_name;
```


7、如何修改字段名：

```sql
alter table tab_name change old_col new_col varchar(40);
```

必须为当前字段指定数据类型等属性，否则不能修改

8、 用一个已存在的表来建新表，但不包含旧表的数据

```sql
create table new_tab_name like old_tab_name;
```



### 数据操作

查询

```sql
select * from 表名
```

增加

```sql
全列插入：insert into 表名 values(…)
缺省插入：insert into 表名(列1,…) values(值1,…)
同时插入多条数据：insert into 表名 values(…),(…)…;
或insert into 表名(列1,…) values(值1,…),(值1,…)…;
主键列是自动增长，但是在全列插入时需要占位，通常使用0，插入成功后以实际数据为准
```



修改

```sql
update 表名 set 列1=值1,… where 条件
```

删除

```sql
delete from 表名 where 条件
逻辑删除，本质就是修改操作update
alter table students add isdelete bit default 0;
如果需要删除则
update students isdelete=1 where …;
```



### 数据的备份与恢复

导入外部数据文本:
1.执行外部的sql脚本

当前数据库上执行:mysql < input.sql
指定数据库上执行:mysql [表名] < input.sql

2.数据传入命令 load data local infile “[文件名]” into table [表名];
备份数据库：(dos下)

```sql
mysqldump --opt school>school.bbb 
mysqldump -u [user] -p [password] databasename > filename (备份) 
mysql -u [user] -p [password] databasename < filename (恢复) 
```



### 卸载

卸载mysql

```sql
dpkg --list|grep mysql        #在终端中查看MySQL的依赖项
sudo apt-get remove mysql-common  #卸载
sudo apt-get autoremove --purge mysql-server-8.0
##sudo apt-get autoremove --purge mysqlxxx
```


清理残留数据

```sql
dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
```


再次查看MySQL的剩余依赖项：

```sql
dpkg --list|grep mysql
```


继续删除剩余依赖项，如：`sudo apt-get autoremove --purge mysql-apt-config`

删除原先配置文件

```shell
sudo rm -rf /etc/mysql/ /var/lib/mysql
sudo apt autoremove
sudo apt autoreclean(如果提示指令有误，就把reclean改成clean)
```



自己用的是deepin系统，很不想用mariadb（虽然不了解这个东西，但好像还是更喜欢mysql），所以在网上一直找怎么去卸载mariadb，安装mysql
试了很久，都没有成功，在网上看说是deepin里全都用的是mariadb

testdb.c:3:10: fatal error: mysql.h: 没有那个文件或目录
#include “mysql.h”
^~~~~~~~~
compilation terminated.

所以自己只好去找mariadb，无法找到mysql.h的原因

找不到mysql.h的解决方法
首先切换到root用户下
输入命令1
sudo apt-get install libmariadb-dev
输入命令2
find / -name libmariadbclient.so
会输出这样的结果：
find: ‘/run/user/1000/gvfs’: 权限不够
/usr/lib/x86_64-linux-gnu/libmariadbclient.so

记住第二行的结果，一会儿会用到

输入命令3
find / -name mysql.h
输出结果：
find: ‘/run/user/1000/gvfs’: 权限不够
/usr/include/mariadb/mysql.h
/usr/include/mariadb/server/mysql.h

记住结果

用你得到的目录地址，去在最后一行命令中修改，就可以找到mysql.h头文件了
gcc testdb.c -I/usr/include/mariadb -L/usr/lib/x86_64-linux-gnu -lmariadbclient -lpthread -lm -ldl -o main
注意:修改-I 和-L后的目录为自己查找到的，就可以了



root@inviubuntu:/usr/local/mysql/bin# ./mysqld --initialize --user=invi --datadir=/usr/local/mysql/data --basedir=/usr/local/mysql
2022-04-12T02:42:14.999759Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2022-04-12T02:42:15.141882Z 0 [Warning] InnoDB: New log files created, LSN=45790
2022-04-12T02:42:15.169655Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2022-04-12T02:42:15.223704Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 299cffc6-ba0a-11ec-bae7-000c290c73ba.
2022-04-12T02:42:15.225392Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2022-04-12T02:42:15.225658Z 1 [Note] A temporary password is generated for root@localhost: gdP9jt6Tb>mT

<dN>wVz?k6fi

