# Oracle数据库开发基础

## connettion和sqlstatement类的使用和编译

```c++
/**************************************************************************************/
/*   程序名：_ooci.h，此程序是开发框架的C/C++操作Oracle数据库的声明文件。             */
/*   author：invi                                                                   */
/**************************************************************************************/

#ifndef __OOCI_H
#define __OOCI_H

// C/C++库常用头文件
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdarg.h>

#include <oci.h>     // OCI的头文件。

struct LOGINENV      // OCI登录环境。
{
  char user[32];     // 数据库的用户名。
  char pass[32];     // 数据库的密码。
  char tnsname[51];  // 数据库的tnsname，在ORACLE_HOME/network/admin/tnsnames.ora中配置。

  OCIEnv *envhp;     // 环境变量的句柄。
};

struct OCI_CXT       // OCI上下文句柄。
{
  OCISvcCtx  *svchp;
  OCIError   *errhp;
  OCIEnv     *envhp; // 环境变量的句柄。
};

struct OCI_HANDLE    // OCI的SQL句柄。
{
  OCISvcCtx  *svchp; // 服务器上下文的句柄引用context句柄。
  OCIStmt    *smthp;

  OCIBind    *bindhp;
  OCIDefine  *defhp;

  OCIError   *errhp; // 错误句柄引用context句柄。

  OCIEnv     *envhp; // 环境变量的句柄。
};

struct CDA_DEF       // OCI接口函数执行的结果。
{
  int      rc;          // 返回值：0-成功，其它失败。
  unsigned long rpc;    // 如果是insert、update和delete，保存影响记录的行数，如果是select，保存结果集的行数。
  char     message[2048];  // 执行SQL语句如果失败，存放错误描述信息。
};

int oci_init(LOGINENV *env);
int oci_close(LOGINENV *env); 
int oci_context_create(LOGINENV *env,OCI_CXT *cxt);
int oci_context_close(OCI_CXT *cxt);

int oci_stmt_create(OCI_CXT *cxt,OCI_HANDLE *handle);
int oci_stmt_close(OCI_HANDLE *handle);

// Oracle数据库连接类。
class connection
{
private:
  LOGINENV m_env;    // 服务器环境句柄。

  char m_dbtype[21]; // 数据库种类，固定取值为"oracle"。

  // 设置字符集，如果客户端的字符集与数据库的不一致，就会出现乱码。
  void character(char *charset);

  // 从connstr中解析username、password和tnsname。
  void setdbopt(char *connstr);
public:
  int m_state;       // 与数据库的连接状态，0-未连接，1-已连接。

  CDA_DEF m_cda;       // 数据库操作的结果或最后一次执行SQL语句的结果。

  char m_sql[10241];   // SQL语句的文本，最长不能超过10240字节。

  connection();    // 构造函数。
 ~connection();    // 析构函数。

  // 登录数据库。
  // connstr：数据库的登录参数，格式：username/password@tnsname，username-用户名，password-登录密
  // 码，tnsname-数据库的服务名，在$ORACLE_HOME/network/admin/tnsnames.ora文件中配置。
  // charset：数据库的字符集，必须与数据库保持一致，否则会出现中文乱码的情况。
  // autocommitopt：是否启用自动提交，0-不启用，1-启用，缺省是不启用。
  // 返回值：0-成功，其它失败，失败的代码在m_cda.rc中，失败的描述在m_cda.message中。
  int connecttodb(char *connstr,char *charset,int autocommitopt=0);

  // 提交事务。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int commit(); 

  // 回滚事务。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int  rollback();

  // 断开与数据库的连接。
  // 注意，断开与数据库的连接时，全部未提交的事务自动回滚。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int disconnect();

  // 执行SQL语句。
  // 如果SQL语句不需要绑定输入和输出变量（无绑定变量、非查询语句），可以直接用此方法执行。
  // 参数说明：这是一个可变参数，用法与printf函数相同。
  // 返回值：0-成功，其它失败，失败的代码在m_cda.rc中，失败的描述在m_cda.message中，
  // 如果成功的执行了非查询语句，在m_cda.rpc中保存了本次执行SQL影响记录的行数。
  // 程序中必须检查execute方法的返回值。
  // 在connection类中提供了execute方法，是为了方便程序中，在该方法中，也是用sqlstatement类来完成功能。
  int execute(const char *fmt,...);

  ////////////////////////////////////////////////////////////////////
  // 以下成员变量和函数，除了sqlstatement类，在类的外部不需要调用它。
  int m_autocommitopt; // 自动提交标志，0-关闭；1-开启。
  OCI_CXT m_cxt;       // 服务器上下文。
  void err_report();   // 获取错误信息。
  ////////////////////////////////////////////////////////////////////
};

// 操作SQL语句类。
class sqlstatement
{
  OCI_HANDLE m_handle; // SQL句柄。
  connection *m_conn;  // 数据库连接指针。
  int m_sqltype;       // SQL语句的类型，0-查询语句；1-非查询语句。
  int m_autocommitopt; // 自动提交标志，0-关闭；1-开启。
  void err_report();   // 错误报告。

  OCILobLocator *m_lob;     // 指向LOB字段的指针。
  int  alloclob();          // 初始化lob指针。
  int  filetolob(FILE *fp); // 把文件的内容导入到clob和blob字段中。
  int  lobtofile(FILE *fp); // 从clob和blob字段中导出内容到文件中。
  void freelob();           // 释放lob指针。
public:
  int m_state;         // 与数据库连接的绑定状态，0-未绑定，1-已绑定。

  char m_sql[10241];   // SQL语句的文本，最长不能超过10240字节。

  CDA_DEF m_cda;       // 执行SQL语句的结果。

  sqlstatement();      // 构造函数。
  sqlstatement(connection *conn);    // 构造函数，同时绑定数据库连接。

 ~sqlstatement();

  // 绑定数据库连接。
  // conn：数据库连接connection对象的地址。
  // 返回值：0-成功，其它失败，只要conn参数是有效的，并且数据库的游标资源足够，connect方法不会返回失败。
  // 程序中一般不必关心connect方法的返回值。
  // 注意，每个sqlstatement只需要绑定一次，在绑定新的connection前，必须先调用disconnect方法。
  int connect(connection *conn); 

  // 取消与数据库连接的绑定。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int disconnect();

  // 准备SQL语句。
  // 参数说明：这是一个可变参数，用法与printf函数相同。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  // 注意：如果SQL语句没有改变，只需要prepare一次就可以了。
  int prepare(const char *fmt,...);

  // 绑定输入变量的地址。
  // position：字段的顺序，从1开始，必须与prepare方法中的SQL的序号一一对应。
  // value：输入变量的地址，如果是字符串，内存大小应该是表对应的字段长度加1。
  // len：如果输入变量的数据类型是字符串，用len指定它的最大长度，建议采用表对应的字段长度。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  // 注意：如果SQL语句没有改变，只需要bindin一次就可以了。
  int bindin(unsigned int position,int    *value);
  int bindin(unsigned int position,long   *value);
  int bindin(unsigned int position,unsigned int  *value);
  int bindin(unsigned int position,unsigned long *value);
  int bindin(unsigned int position,float *value);
  int bindin(unsigned int position,double *value);
  int bindin(unsigned int position,char   *value,unsigned int len);

  // 绑定输出变量的地址。
  // position：字段的顺序，从1开始，与SQL的结果集一一对应。
  // value：输出变量的地址，如果是字符串，内存大小应该是表对应的字段长度加1。
  // len：如果输出变量的数据类型是字符串，用len指定它的最大长度，建议采用表对应的字段长度。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  // 注意：如果SQL语句没有改变，只需要bindout一次就可以了。
  int bindout(unsigned int position,int    *value);
  int bindout(unsigned int position,long   *value);
  int bindout(unsigned int position,unsigned int  *value);
  int bindout(unsigned int position,unsigned long *value);
  int bindout(unsigned int position,float *value);
  int bindout(unsigned int position,double *value);
  int bindout(unsigned int position,char   *value,unsigned int len);

  // 执行SQL语句。
  // 返回值：0-成功，其它失败，失败的代码在m_cda.rc中，失败的描述在m_cda.message中。
  // 如果成功的执行了非查询语句，在m_cda.rpc中保存了本次执行SQL影响记录的行数。
  // 程序中必须检查execute方法的返回值。
  int execute();
  
  // 执行SQL语句。
  // 如果SQL语句不需要绑定输入和输出变量（无绑定变量、非查询语句），可以直接用此方法执行。
  // 参数说明：这是一个可变参数，用法与printf函数相同。
  // 返回值：0-成功，其它失败，失败的代码在m_cda.rc中，失败的描述在m_cda.message中，
  // 如果成功的执行了非查询语句，在m_cda.rpc中保存了本次执行SQL影响记录的行数。
  // 程序中必须检查execute方法的返回值。
  int execute(const char *fmt,...);

  // 从结果集中获取一条记录。
  // 如果执行的SQL语句是查询语句，调用execute方法后，会产生一个结果集（存放在数据库的缓冲区中）。
  // next方法从结果集中获取一条记录，把字段的值放入已绑定的输出变量中。
  // 返回值：0-成功，1403-结果集已无记录，其它-失败，失败的代码在m_cda.rc中，失败的描述在m_cda.message中。
  // 返回失败的原因主要有两个：1）与数据库的连接已断开；2）绑定输出变量的内存太小。
  // 每执行一次next方法，m_cda.rpc的值加1。
  // 程序中必须检查next方法的返回值。
  int next();

  // 绑定clob字段。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int bindblob();

  // 绑定blob字段。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int bindclob();

  // 把文件的内容导入到clob和blob字段中。
  // filename：待导入的文件名，建议采用绝对路径。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int filetolob(char *filename);

  // 从clob和blob字段中导出内容到文件中。
  // filename：导出内容存放的文件名，建议采用绝对路径。
  // 返回值：0-成功，其它失败，程序中一般不必关心返回值。
  int lobtofile(char *filename);
};

#endif 


```

示例程序

### 建表demo

```c++
/*
 *  程序名：createtable.cpp，此程序演示开发框架操作Oracle数据库（创建表）。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 登录数据库，返回值：0-成功，其它-失败。
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8")!=0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }
  
  sqlstatement stmt(&conn); // 操作SQL语句的对象。

  // 准备创建表的SQL语句。
  // 超女表girls，超女编号id，超女姓名name，体重weight，报名时间btime，超女说明memo，超女图片pic。
  stmt.prepare("\
    create table girls(id    number(10),\
                       name  varchar2(30),\
                       weight   number(8,2),\
                       btime date,\
                       memo  clob,\
                       pic   blob,\
                       primary key (id))");
  // prepare方法不需要判断返回值。

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  // 失败代码在stmt.m_cda.rc中，失败描述在stmt.m_cda.message中。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  printf("create table girls ok.\n");
}

```

### 插入数据demo

```c++
/*
 *  程序名：inserttable.cpp，此程序演示开发框架操作Oracle数据库（向表中插入5条记录）。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

// 定义用于超女信息的结构，与表中的字段对应。
struct st_girls
{
  long id;        // 超女编号，用long数据类型对应Oracle无小数的number(10)。
  char name[30];  // 超女姓名，用char[31]对应Oracle的varchar2(30)。
  double weight;  // 超女体重，用double数据类型对应Oracle有小数的number(8,2)。
  char btime[20]; // 报名时间，用char对应Oracle的date，格式：'yyyy-mm-dd hh24:mi:ssi'。
} stgirls;

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 登录数据库，返回值：0-成功，其它-失败。
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8")!=0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }
  
  sqlstatement stmt(&conn); // 操作SQL语句的对象。

  // 准备插入表的SQL语句。
  stmt.prepare("\
    insert into girls(id,name,weight,btime) \
                values(:1,:2,:3,to_date(:4,'yyyy-mm-dd hh24:mi:ss'))");
  // prepare方法不需要判断返回值。
  // 为SQL语句绑定输入变量的地址，bindin方法不需要判断返回值。
  stmt.bindin(1,&stgirls.id);
  stmt.bindin(2, stgirls.name,30);
  stmt.bindin(3,&stgirls.weight);
  stmt.bindin(4, stgirls.btime,19);

  // 模拟超女数据，向表中插入5条测试数据。
  for (int ii=0;ii<5;ii++)
  {
    memset(&stgirls,0,sizeof(struct st_girls));         // 结构体变量初始化。

    // 为结构体变量的成员赋值。
    stgirls.id=ii;                                     // 超女编号。
    sprintf(stgirls.name,"西施%05dgirl",ii+1);         // 超女姓名。
    stgirls.weight=45.35+ii;                           // 超女体重。
    sprintf(stgirls.btime,"2021-08-25 10:33:%02d",ii); // 报名时间。

    // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
    // 失败代码在stmt.m_cda.rc中，失败描述在stmt.m_cda.message中。
    if (stmt.execute()!=0)
    {
      printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
    }

    printf("成功插入了%ld条记录。\n",stmt.m_cda.rpc); // stmt.m_cda.rpc是本次执行SQL影响的记录数。
  }

  printf("insert table girls ok.\n");

  conn.commit(); // 提交数据库事务。
}

```

### 数据查询demo

```c++
/*
 *  程序名：selecttable.cpp，此程序演示开发框架操作Oracle数据库（查询表中的记录）。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

// 定义用于超女信息的结构，与表中的字段对应。
struct st_girls
{
  long id;        // 超女编号，用long数据类型对应Oracle无小数的number(10)。
  char name[31];  // 超女姓名，用char[31]对应Oracle的varchar2(30)。
  double weight;  // 超女体重，用double数据类型对应Oracle有小数的number(8,2)。
  char btime[20]; // 报名时间，用char对应Oracle的date，格式：'yyyy-mm-dd hh24:mi:ss'。
} stgirls;

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类
  
  // 登录数据库，返回值：0-成功，其它-失败。
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8")!=0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }

  sqlstatement stmt(&conn); // 操作SQL语句的对象。

  int iminid,imaxid;  // 查询条件最小和最大的id。

  // 准备查询表的SQL语句。
  stmt.prepare("\
    select id,name,weight,to_char(btime,'yyyy-mm-dd hh24:mi:ss') from girls where id>=:1 and id<=:2");
  // prepare方法不需要判断返回值。
  // 为SQL语句绑定输入变量的地址，bindin方法不需要判断返回值。
  stmt.bindin(1,&iminid);
  stmt.bindin(2,&imaxid);
  // 为SQL语句绑定输出变量的地址，bindout方法不需要判断返回值。
  stmt.bindout(1,&stgirls.id);
  stmt.bindout(2, stgirls.name,30);
  stmt.bindout(3,&stgirls.weight);
  stmt.bindout(4, stgirls.btime,19);

  iminid=2;  // 指定待查询记录的最小id的值。
  imaxid=4;  // 指定待查询记录的最大id的值。

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  // 失败代码在stmt.m_cda.rc中，失败描述在stmt.m_cda.message中。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 本程序执行的是查询语句，执行stmt.execute()后，将会在数据库的缓冲区中产生一个结果集。
  while (1)
  {
    memset(&stgirls,0,sizeof(stgirls)); // 先把结构体变量初始化。

    // 从结果集中获取一条记录，一定要判断返回值，0-成功，1403-无记录，其它-失败。
    // 在实际开发中，除了0和1403，其它的情况极少出现。
    if (stmt.next() !=0) break;
    
    // 把获取到的记录的值打印出来。
    printf("id=%ld,name=%s,weight=%.02f,btime=%s\n",stgirls.id,stgirls.name,stgirls.weight,stgirls.btime);
  }

  // 请注意，stmt.m_cda.rpc变量非常重要，它保存了SQL被执行后影响的记录数。
  printf("本次查询了girls表%ld条记录。\n",stmt.m_cda.rpc);
}

```

### 数据更新demo

```c++
/*
 *  程序名：updatetable.cpp，此程序演示开发框架操作Oracle数据库（修改表中的记录）。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 登录数据库，返回值：0-成功，其它-失败。
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8")!=0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }

  sqlstatement stmt(&conn); // 操作SQL语句的对象。

  char strbtime[20];  // 用于存放超女的报名时间。
  memset(strbtime,0,sizeof(strbtime));
  strcpy(strbtime,"2019-12-20 09:45:30");

  // 准备更新数据的SQL语句，不需要判断返回值。
  stmt.prepare("\
    update girls set btime=to_date(:1,'yyyy-mm-dd hh24:mi:ss') where id>=2 and id<=4");
  // prepare方法不需要判断返回值。
  // 为SQL语句绑定输入变量的地址，bindin方法不需要判断返回值。
  stmt.bindin(1,strbtime,19);
  // 如果不采用绑定输入变量的方法，把strbtime的值直接写在SQL语句中也是可以的，如下：
  /*
  stmt.prepare("\
    update girls set btime=to_date('%s','yyyy-mm-dd hh24:mi:ss') where id>=2 and id<=4",strbtime);
  */

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  // 失败代码在stmt.m_cda.rc中，失败描述在stmt.m_cda.message中。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 请注意，stmt.m_cda.rpc变量非常重要，它保存了SQL被执行后影响的记录数。
  printf("本次更新了girls表%ld条记录。\n",stmt.m_cda.rpc);

  // 提交事务
  conn.commit();
}
```

### 删除语句demo

```c++
/*
 *  程序名：deletetable.cpp，此程序演示开发框架操作Oracle数据库（删除表中的记录）。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 登录数据库，返回值：0-成功，其它-失败。
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8")!=0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }

  sqlstatement stmt(&conn); // 操作SQL语句的对象。

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  // 失败代码在stmt.m_cda.rc中，失败描述在stmt.m_cda.message中。
  // 如果不需要绑定输入和输出变量，用stmt.execute()方法直接执行SQL语句，不需要stmt.prepare()。
  if (stmt.execute("delete from girls where id>=2 and id<=4") != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 请注意，stmt.m_cda.rpc变量非常重要，它保存了SQL被执行后影响的记录数。
  printf("本次从girls表中删除了%ld条记录。\n",stmt.m_cda.rpc); 

  // 提交事务
  conn.commit();
}

```

### 执行PL/SQL过程

```c++
/*
 *  程序名：execplsql.cpp，此程序演示开发框架操作Oracle数据库（执行PL/SQL过程）。
 *  author：invi
 *  说说我个人的看法，我从不在Oracle数据库中创建PL/SQL过程，也很少使用触发器，原因如下：
 *  1、在Oracle数据库中创建PL/SQL过程，程序的调试很麻烦；
 *  2、维护工作很麻烦，因为维护人员要花时间去了解数据库中的存储过程；
 *  3、采用本开发框架操作Oracle已经是非常简单，没必要去折腾存储过程；
 *  4、PL/SQL过程可移植性不好，如果换成mysql或其它数据库，比较麻烦。
 *  还有，我在C/C++程序中很少用复杂的PL/SQL过程，因为复杂的PL/SQL调试麻烦。
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 登录数据库，返回值：0-成功，其它-失败。
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8")!=0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }

  sqlstatement stmt(&conn); // 操作SQL语句的对象。

  int id=100;

  // 准备查询表的PL/SQL语句，先删除girls表中的全部记录，再插入一条记录。
  stmt.prepare("\
    BEGIN\
      delete from girls;\
      insert into girls(id,name,weight,btime)\
                 values(:1,'超女过程',55.65,to_date('2018-01-02 13:00:55','yyyy-mm-dd hh24:mi:ss'));\
    END;");
  // 注意，PL/SQL中的每条SQL需要用分号结束，END之后还有一个分号。
  // prepare方法不需要判断返回值。
  // 为PL/SQL语句绑定输入变量的地址，bindin方法不需要判断返回值。
  stmt.bindin(1,&id);
  
  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  printf("exec PL/SQL ok.\n");

  // 提交事务。
  conn.commit();
}

```



### blob大对象demo

```c++
/*
 *  程序名：filetoclob.cpp，此程序演示开发框架操作Oracle数据库。
 *  把当前目录中的memo_in.txt文件写入Oracle的CLOB字段中。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类
  
  // 连接数据库，返回值0-成功，其它-失败
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8") != 0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }
  
  sqlstatement stmt(&conn); // SQL语句操作类

  // 为了方便演示，把girls表中的记录全删掉，再插入一条用于测试的数据。
  // 不需要判断返回值
  stmt.prepare("\
    BEGIN\
      delete from girls;\
      insert into girls(id,name,memo) values(1,'超女姓名',empty_clob());\
    END;");
  
  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 使用游标从girls表中提取id为1的memo字段
  // 注意了，同一个sqlstatement可以多次使用
  // 但是，如果它的sql改变了，就要重新prepare和bindin或bindout变量
  stmt.prepare("select memo from girls where id=1 for update");
  stmt.bindclob();

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }
  
  // 获取一条记录，一定要判断返回值，0-成功，1403-无记录，其它-失败。
  if (stmt.next() != 0) return 0;
  
  // 把磁盘文件memo_in.txt的内容写入CLOB字段
  if (stmt.filetolob((char *)"memo_in.txt") != 0)
  {
    printf("stmt.filetolob() failed.\n%s\n",stmt.m_cda.message); return -1;
  }

  conn.commit(); // 提交事务
}

```

```c++
/*
 *  程序名：filetoblob.cpp，此程序演示开发框架操作Oracle数据库。
 *  把当前目录的pic_in.jpeg文件写入Oracle的BLOB字段中。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类
  
  // 连接数据库，返回值0-成功，其它-失败
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8") != 0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }
  
  sqlstatement stmt(&conn); // SQL语句操作类
  
  // 为了方便演示，把girls表中的记录全删掉，再插入一条用于测试的数据。
  // 不需要判断返回值
  stmt.prepare("\
    BEGIN\
      delete from girls;\
      insert into girls(id,name,pic) values(1,'超女姓名',empty_blob());\
    END;");
  
  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 使用游标从girls表中提取id为1的pic字段
  // 注意了，同一个sqlstatement可以多次使用
  // 但是，如果它的sql改变了，就要重新prepare和bindin或bindout变量
  stmt.prepare("select pic from girls where id=1 for update");
  stmt.bindblob();

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 获取一条记录，一定要判断返回值，0-成功，1403-无记录，其它-失败。
  if (stmt.next() != 0) return 0;
  
  // 把磁盘文件pic_in.jpeg的内容写入BLOB字段，一定要判断返回值，0-成功，其它-失败。
  if (stmt.filetolob((char *)"pic_in.jpeg") != 0)
  {
    printf("stmt.filetolob() failed.\n%s\n",stmt.m_cda.message); return -1;
  }

  conn.commit(); // 提交事
}
```

```c++
/*
 *  程序名：clobtofile.cpp，此程序演示开发框架操作Oracle数据库。
 *  把Oracle的CLOB字段的内容提取到目前目录的memo_out.txt中。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 连接数据库，返回值0-成功，其它-失败
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8") != 0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }
  
  sqlstatement stmt(&conn); // SQL语句操作类

  // 不需要判断返回值
  stmt.prepare("select memo from girls where id=1");
  stmt.bindclob();

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }
  
  // 获取一条记录，一定要判断返回值，0-成功，1403-无记录，其它-失败。
  if (stmt.next() != 0) return 0;
  
  // 把CLOB字段中的内容写入磁盘文件，一定要判断返回值，0-成功，其它-失败。
  if (stmt.lobtofile((char *)"memo_out.txt") != 0)
  {
    printf("stmt.lobtofile() failed.\n%s\n",stmt.m_cda.message); return -1;
  }
}
```

```c++
/*
 *  程序名：bolbtofile.cpp，此程序演示开发框架操作Oracle数据库。
 *  把Oracle的BLOB字段的内容提取到目前目录的pic_out.jpeg中。
 *  author：invi
*/
#include "_ooci.h"   // 开发框架操作Oracle的头文件。

int main(int argc,char *argv[])
{
  connection conn; // 数据库连接类

  // 连接数据库，返回值0-成功，其它-失败
  // 失败代码在conn.m_cda.rc中，失败描述在conn.m_cda.message中。
  if (conn.connecttodb("scott/tiger@snorcl11g_130","Simplified Chinese_China.AL32UTF8") != 0)
  {
    printf("connect database failed.\n%s\n",conn.m_cda.message); return -1;
  }

  sqlstatement stmt(&conn); // SQL语句操作类

  // 不需要判断返回值
  stmt.prepare("select pic from girls where id=1");
  stmt.bindblob();

  // 执行SQL语句，一定要判断返回值，0-成功，其它-失败。
  if (stmt.execute() != 0)
  {
    printf("stmt.execute() failed.\n%s\n%s\n",stmt.m_sql,stmt.m_cda.message); return -1;
  }

  // 获取一条记录，一定要判断返回值，0-成功，1403-无记录，其它-失败。
  if (stmt.next() != 0) return 0;

  // 把BLOB字段中的内容写入磁盘文件，一定要判断返回值，0-成功，其它-失败。
  if (stmt.lobtofile((char *)"pic_out.jpeg") != 0)
  {
    printf("stmt.lobtofile() failed.\n%s\n",stmt.m_cda.message); return -1;
  }
}
```



## Oracle的错误代码

百度...

## 其他注意事项（查询语句的结果集，数据库函数的使用）

各种数据库的函数不一样，尽量不要使用数据库提供的函数

用编程语言实现函数的功能（性能和兼容性）

用自定义函数解决数据库之间不兼容的问题

