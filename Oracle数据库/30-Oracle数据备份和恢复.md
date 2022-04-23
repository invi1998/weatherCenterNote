Oracle数据库的备份和恢复有很多种方法，是一个很大的话题，足可以写一本书，但是，本文只介绍采用exp和imp进行数据备份和恢复，这也是程序员最常用的方法。

本文涉及的备份与恢复的其它概念都是狭义的，不完整的或不完全准确的，仅供参考。

# 一、备份与恢复的概念

## 1、什么是备份

备份就是把数据库中的数据复制到存储设备的过程，存储设备包括磁盘、磁带、光盘等，随着存储技术的发展，磁带和光盘已经很少使用。

## 2、备份的方法

1）物理备份

对数据库操作系统的物理文件（数据文件，控制文件和日志文件）的备份。物理备份又可以分为脱机备份（冷备份）和联机备份（热备份），前者是在关闭数据库的时候进行的，后者是以归档日志的方式对运行的数据库进行备份。可以使用oracle的恢复管理器（rman）或操作系统命令进行数据库的物理备份。

2）逻辑备份

对数据库逻辑组件（如表和存储过程等数据库对象）的备份。逻辑备份的手段很多，如传统的exp，数据泵（expdp），数据库闪回技术等第三方工具，都可以进行数据库的逻辑备份。

## 3、备份的策略

1）完全备份

每次对数据库进行完整备份，当发生数据丢失的灾难时，完全备份无需依赖其他信息即可实现100%的数据恢复，其恢复时间最短且操作最方便。

2）增量备份

只有那些在上次完全备份或增量备份后被修改的文件才会被备份。优点是备份数据量小，需要的时间短，缺点是恢复的时候需要依赖以前备份记录，出问题的风险较大。

3）差异备份

备份那些自从上次完全备份之后被修改过的文件。从差异备份中恢复数据的时间较短，只需要最后一次完整备份和最后一次差异备份的数据，缺点是每次备份需要的时间较长。

## 4、什么是恢复

恢复就是发生故障后，利用已备份的数据文件或控制文件，重新建立一个完整的数据库。

## 5、恢复分类

1）实例恢复

当oracle实例出现失败后，oracle自动进行的恢复。

2）介质恢复

当存放数据库的介质出现故障时所作的恢复，介质恢复又分为完全恢复和不完全恢复。

完全恢复：将数据库恢复到数据库失败时的状态。这种恢复是通过装载数据库备份并应用全部的重做日志做到的。

不完全恢复：将数据库恢复到数据库失败前的某一时刻的状态。这种恢复是通过装载数据库备份并应用部分的重做日志做到的。进行不完全恢复后，必须在启动数据库时用resetlogs选项重设联机重做日志。

# 二、逻辑备份和恢复

Oracle逻辑备份和恢复的工具是exp（导出）和imp（导入），我根据应用场景来介绍这两个命令使用方法。

## 1、exp命令

在shell下输入exp -help或exp help=y获取exp命令帮助，如下：

```sql
exp -help
```

```sql
Export: Release 11.2.0.4.0 - Production on 星期四 2月 6 10:04:26 2020

Copyright (c) 1982, 2011, Oracle and/or its affiliates. All rights reserved.
```

通过输入 EXP 命令和您的用户名/口令, 导出

操作将提示您输入参数: 

   例如: EXP SCOTT/TIGER

或者, 您也可以通过输入跟有各种参数的 EXP 命令来控制导出

的运行方式。要指定参数, 您可以使用关键字: 

   格式: EXP KEYWORD=value 或 KEYWORD=(value1,value2,...,valueN)

   例如: EXP SCOTT/TIGER GRANTS=Y TABLES=(EMP,DEPT,MGR)

​        或 TABLES=(T1:P1,T1:P2), 如果 T1 是分区表

USERID 必须是命令行中的第一个参数。

| 关键字            | 说明 (默认值)                            | 关键字               | 说明 (默认值)                   |
| ----------------- | ---------------------------------------- | -------------------- | ------------------------------- |
| USERID            | 用户名/口令                              | FULL                 | 导出整个文件 (N)                |
| BUFFER            | 数据缓冲区大小                           | OWNER                | 所有者用户名列表                |
| FILE              | 输出文件 (EXPDAT.DMP)                    | TABLES               | 表名列表                        |
| COMPRESS          | 导入到一个区 (Y)                         | RECORDLENGTH         | IO  记录的长度                  |
| GRANTS            | 导出权限 (Y)                             | CONSTRAINTS          | 导出的约束条件 (Y)              |
| INDEXES           | 导出索引 (Y)                             | INCTYPE              | 增量导出类型                    |
| DIRECT            | 直接路径 (N)                             | RECORD               | 跟踪增量导出 (Y)                |
| LOG               | 屏幕输出的日志文件                       | TRIGGERS             | 导出触发器 (Y)                  |
| ROWS              | 导出数据行 (Y)                           | STATISTICS           | 分析对象 (ESTIMATE)             |
| CONSISTENT        | 交叉表的一致性 (N)                       | PARFILE              | 参数文件名                      |
| OBJECT_CONSISTENT | 只在对象导出期间设置为只读的事务处理 (N) | RESUMABLE_TIMEOUT    | RESUMABLE 的等待时间            |
| FEEDBACK          | 每 x 行显示进度 (0)                      | TTS_FULL_CHECK       | 对 TTS 执行完整或部分相关性检查 |
| FILESIZE          | 每个转储文件的最大大小                   | RESUMABLE_NAME       | 用于标识可恢复语句的文本字符串  |
| FLASHBACK_SCN     | 用于将会话快照设置回以前状态的 SCN       | VOLSIZE              | 写入每个磁带卷的字节数          |
| FLASHBACK_TIME    | 用于获取最接近指定时间的 SCN 的时间      | TABLESPACES          | 要导出的表空间列表              |
| QUERY             | 用于导出表的子集的 select 子句           | TRANSPORT_TABLESPACE | 导出可传输的表空间元数据 (N)    |
| RESUMABLE         | 遇到与空格相关的错误时挂起 (N)           | TEMPLATE             | 调用 iAS 模式导出的模板名       |



## 2、imp命令

在shell下输入imp -help或imp help=y获取imp命令帮助，如下：

```sql
imp -help

 

Import: Release 11.2.0.4.0 - Production on 星期四 2月 6 10:06:42 2020

Copyright (c) 1982, 2011, Oracle and/or its affiliates. All rights reserved.


```

通过输入 IMP 命令和您的用户名/口令, 导入

操作将提示您输入参数: 

   例如: IMP SCOTT/TIGER

或者, 可以通过输入 IMP 命令和各种参数来控制导入

的运行方式。要指定参数, 您可以使用关键字: 

 

   格式: IMP KEYWORD=value 或 KEYWORD=(value1,value2,...,valueN)

   例如: IMP SCOTT/TIGER IGNORE=Y TABLES=(EMP,DEPT) FULL=N

​        或 TABLES=(T1:P1,T1:P2), 如果 T1 是分区表

 

USERID 必须是命令行中的第一个参数。

 

关键字  说明 (默认值)    关键字   说明 (默认值)

\--------------------------------------------------------------------------

USERID  用户名/口令      FULL    导入整个文件 (N)

BUFFER  数据缓冲区大小    FROMUSER  所有者用户名列表

FILE   输入文件 (EXPDAT.DMP) TOUSER   用户名列表

SHOW   只列出文件内容 (N)   TABLES   表名列表

IGNORE  忽略创建错误 (N)  RECORDLENGTH IO 记录的长度

GRANTS  导入权限 (Y)     INCTYPE   增量导入类型

INDEXES  导入索引 (Y)     COMMIT    提交数组插入 (N)

ROWS   导入数据行 (Y)    PARFILE   参数文件名

LOG   屏幕输出的日志文件  CONSTRAINTS  导入限制 (Y)

DESTROY        覆盖表空间数据文件 (N)

INDEXFILE       将表/索引信息写入指定的文件

SKIP_UNUSABLE_INDEXES 跳过不可用索引的维护 (N)

FEEDBACK        每 x 行显示进度 (0)

TOID_NOVALIDATE     跳过指定类型 ID 的验证 

FILESIZE        每个转储文件的最大大小

STATISTICS       始终导入预计算的统计信息

RESUMABLE       在遇到有关空间的错误时挂起 (N)

RESUMABLE_NAME     用来标识可恢复语句的文本字符串

RESUMABLE_TIMEOUT   RESUMABLE 的等待时间 

COMPILE        编译过程, 程序包和函数 (Y)

STREAMS_CONFIGURATION 导入流的一般元数据 (Y)

STREAMS_INSTANTIATION 导入流实例化元数据 (N)

DATA_ONLY       仅导入数据 (N)

VOLSIZE        磁带的每个文件卷上的文件的字节数

 

下列关键字仅用于可传输的表空间

TRANSPORT_TABLESPACE 导入可传输的表空间元数据 (N)

TABLESPACES 将要传输到数据库的表空间

DATAFILES 将要传输到数据库的数据文件

TTS_OWNERS 拥有可传输表空间集中数据的用户

## 3、数据库实例导出和导入

导出数据库实例需要DBA权限，包括Oracle系统和全部的用户、索引、存储过程、权限等。应用场景极少，不建议。

1）导出

```sql
exp scott/tiger file=/tmp/expdata.dmp full=y
```

2）导入

```sql
imp scott/tiger file=/tmp/expdata.dmp full=y
```

## 4、用户的导出和导入

导出/导入某用户全部的对象。普通用户只能操作本用户，DBA用户可以操作其他用户。

1）用scott/tiger登录数据库，导出scott用户，数据保存在/tmp/expscott.dmp文件中，日志保存在/tmp/expscott.log文件中。

```sql
exp scott/tiger owner=scott file=/tmp/expscott.dmp log=/tmp/expscott.log
```

2）用scott/tiger登录数据库，从/tmp/expscott.dmp文件中导入数据，指明是从scott用户导出的，现在要导入到scott用户中。

```sql
imp scott/tiger file=/tmp/expscott.dmp fromuser=scott touser=scott
```

3）用system/systempwd登录数据库（system具有DBA权限），，从/tmp/expscott.dmp文件中导入数据，指明是从scott用户导出的，现在要导入到girl用户中。

```sql
imp system/systempwd file=/tmp/expscott.dmp fromuser=scott touser=girl
```

## 5、表的导出和导入

导出/导入某用户全部的表。普通用户只能操作本用户的表，DBA用户可以操作其他用户的表。如果要导出多个表，需要把表名写在括号中，括号需要用\转义。

1）用scott/tiger登录数据库，导出EMP和DEPT表，数据保存在/tmp/empdept.dmp文件中，日志保存在/tmp/empdtpe.log文件中。

```sql
exp scott/tiger file=/tmp/empdept.dmp log=/tmp/empdept.log tables=\(EMP,DEPT\)
```

2）用scott/tiger登录数据库，从/tmp/empdept.dmp文件中导入数据，指明是从scott用户导出的，现在要导入到scott用户中。

```sql
imp scott/tiger file=/tmp/empdept.dmp fromuser=scott touser=scott
```

3）用system/systempwd登录数据库（system具有DBA权限），，从/tmp/expscott.dmp文件中导入数据，指明是从scott用户导出的，现在要导入到girl用户中。

```sql
imp system/systempwd file=empdept.dmp fromuser=scott touser=girl
```



## 6、注意事项

1）exp和imp的参数比较多，但用到的却很少，您可以多尝试。

2）exp和imp的时候，如果没有任何警告和错误，会出现成功终止导出/导入, 没有出现警告。

3）log参数很重要，把导出/导入的信息写在日志文件中，您可以用程序来判断导出是否成功。备份工作很重要，必须保证成功。

4）owner参数支持多个用户名的书写，要用括号，括号前加\转义。

5）tables参数支持多个表名的书写，要用括号，括号前加\转义。

6）rows参数很重要，如果rows=n，则只导出/导入表结构，不导出数据。

7）imp的ignore参数很重要，如果ignore=n，导入数据的时候，如果表已存在，就报错并且不会向表中导入数据，如果ignore=y，导入数据的时候，如果表已存在，会报错但是仍会向表中导入数据。

8）增量导出功能看上去很棒，但是可操作性很差。

9）其它的参数看看就行，了解了解。

# 三、应用经验



重要的业务系统一定会有非常完善的备份方案，由DBA去执行。

对程序员来说，采用exp/imp的场景主要有：

1）对一些数据量比较小的系统做备份，例如备份某数据库用户；

2）只备份数据库对象（不备份数据），缩短故障恢复的时间；

3）用于表的数据迁移；

4）程序员还可能自己编写程序来备份数据。