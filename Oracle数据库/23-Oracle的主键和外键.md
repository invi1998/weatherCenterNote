# 一、表的主键

在现实世界中，很多数据具有唯一的特性，例如身份证号码，在国家人口基本信息表中，一定不会存在多个人用同一个身份证号码的情况，再例如手机号码、QQ号码、银行帐号等等，还有学生管理系统，学生的年级、班级和学号三个字段组合起来是唯一的标识。

如果表中一个字段或多个字段组合起来的值是唯一的，就可以作为表的主键，在创建或修改表时用primay key关键字来指定主键。一个表只能有一个主键，而且组成主键的每个字段值都不能为[空](https://baike.baidu.com/item/空值)。

主键的作用：

1）体现数据结构设计的合理性。

2）提升数据操作的速度。

3）保证数据的完整性，在表中添加或修改记录时，数据库会检查该记录主键的值，不允许与其它记录主键的值重复，这种做法有个专业的名词：主键约束。

例如超女基本信息表，编号的字段名是id，在超女选秀活动中，每个超女的编号肯定是唯一的，不可能存在两个编号相同的超女，否则会引起混乱，我们可以把id字段设置为T_GIRL表的主键，后面的工作交给数据库，如果试图往表中插入多条id相同的记录，数据库将拒绝。

指定表的主建有两种方法。

1）在create table时指定。

```sql
create table T_GIRL

(

 id    char(4)     not null,  -- 编号

 name   varchar2(30)  not null,  -- 姓名

 yz    varchar2(20)    null,  -- 颜值

 sc    varchar2(20)    null,  -- 身材

 weight  number(4,1)   not null,  -- 体重

 height  number(3)    not null,  -- 身高

 birthday date      not null,  -- 出生时间

 memo   varchar2(1000)   null,  -- 备注

 primary key(id)            -- 指定id为表的主键

);
```

2）修改已经建好的表，增加主键约束。

```sql
alter table 表名 add constraint 主键名 primary key(字段名1,字段名2,......字段名n);
```

例如：

```sql
alter table T_GIRL add constraint PK_GIRL primary key(id);
```

在Oracle数据库中，虽然主键不是必需的，但是最好为每个表都设置一个主键，不管是单字段主键还是多字段主键（复合主键），它的存在代表了表结构的完整性，主键还可以用于其他表的外键关联，外键的知识下面再介绍。

# 二、表的外键

## 1、外键的概念

外键（foreign key）是用于表达两个表数据之间的关系，将表中主键字段添加到另一个表中，再创建两个表之间的约束关系，这些字段就成为第二个表的外键。

超女选秀活动有两个数据表：

1）赛区参数表

```sql
赛区代码，赛区名称，……。
```

2）超女基本信息表

```sql
赛区代码、超女编号、姓名、颜值、身材、身高、体重、……。
```

录入超女基本信息的时候要选择赛区，为了保证数据的有效，要求录入赛区代码时，必须保证赛区参数表中有这个赛区代码，否则数据是不一致的，为了保证数据的完整性，必须在程序中判断数据的合法性。针对这种情况，在表结构设计中采用外键来约束这两个表的赛区代码字段。

对赛区参数表来说，赛区代码是该表的主键。

对超女基本信息表来说，赛区代码是该表的外键。

赛区参数表也称为**主表**，超女基本信息表也称为**从表**。

## 2、外键的作用

合理的数据结构设计，表中的数据一定有一致性约束，使用外键，让数据库去约束数据的一致，不给任何人出错的机会。不用外键会怎样？不用也不会怎么样，如果不用外键，在程序中要写代码进行判断，手工操作数据时也必须处处小心。

## 3、外键约束

**1）当对从表进行操作时，数据库会：**

a）向从表插入新记录时，如果外键值在主表中不存在，阻止插入。

b）修改从表的记录时，如果外键的值在主表中不存在，阻止修改。

**2）当对主表进行修改操作时，数据库会：**

a）主表修改主键值时，旧值在从表里存在便阻止修改。

**3）当对主表进行删除操作时，数据库会（三选一）：**

a）主表删除行时，其主键值在从表里存在便阻止删除。

b）主表删除行时，连带从表的相关行一起删除。

c）主表删除行时，把从表相关行的外键字段置为null。

## 4、创建外键

创建外键的语法：

```sql
alter table 从表名

  add constraint 外键名 foreign key (从表字段列表)

   references 主表名 (主表字段列表)

   [on delete cascade|set null];
```

说明：

外键名，Oracle的标识符，建议采用**FK_从表名_主表名**的方式命名。

主表执行删除行时，其主键值在从表里存在便阻止删除，如果on delete cascade，连带从表的相关行一起删除；如果on delete set null，把从表相关行的外键字段置为null。

## 5、删除外键

```sql
alter table 从表名 drop constraint 外键名;
```

## 6、示例脚本

```sql
/* 创建赛区参数表。 */

create table T_AREACODE

(

 areaid  number(2)  not null,  -- 赛区代码，非空。

 areaname varchar(20) not null,  -- 赛区名称，非空。

 memo   varchar(300),      -- 备注

 primary key(areaid)        -- 创建主健。

);
```



 

```sql
/* 创建超女基本信息表。 */

create table T_GIRL

(

 id    char(4)     not null,  -- 编号

 name   varchar2(30)    null,  -- 姓名

 areaid  number(2)      null,  -- 赛区代码

 yz    varchar2(20)    null,  -- 颜值

 sc    varchar2(20)    null,  -- 身材

 memo   varchar2(1000)   null,  -- 备注

 primary key(id)            -- 创建主健。

);
```



 

```sql
/* 以下三种创建外键的方式只能三选一  */

/* 为T_GIRL创建外键，无on delete选项。 */

alter table T_GIRL

  add constraint FK_GIRL_AREACODE foreign key(areaid)

   references T_AREACODE(areaid);

 

/* 为T_GIRL创建外键，采用on delete cascade选项。 */

alter table T_GIRL

  add constraint FK_GIRL_AREACODE foreign key(areaid)

   references T_AREACODE(areaid)

   on delete cascade;

 

/* 为T_GIRL创建外键，采用on delete set null选项。 */

alter table T_GIRL

  add constraint FK_GIRL_AREACODE foreign key(areaid)

   references T_AREACODE(areaid)

   on delete set null;
```

