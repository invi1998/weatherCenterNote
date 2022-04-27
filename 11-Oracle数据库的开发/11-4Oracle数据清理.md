主要就是将mysql的数据清理程序进行Oracle改造

改造思路：

- 修改程序帮助文档
- 修改错误代码（oracle的错误代码和mysql的错误代码不一样）
- 修改keyid的处理方式（因为Oracle的自增字段是使用序列生成器，所以需要进行适配）
- 时间函数和时间格式的修改适配
- Oracle不需要为查询语句创建专门的数据库连接
- Oracle不需要MAXPARAMS宏

数据删除

```c++
/*
 *  程序名：deletetable_oracle.cpp，本程序是数据中心的公共功能模块，用于定时清理表中的数据。
*/
#include "_public.h"
#include "_ooci.h"

#define MAXPARAMS 256

struct st_arg
{
    char connstr[101];     // 数据库的连接参数。
    char tname[31];        // 待清理的表名。
    char keycol[31];       // 待清理的表的唯一键字段名。
    char where[1001];      // 待清理的数据需要满足的条件。
    char starttime[31];    // 程序运行的时间区间。
    int  timeout;          // 本程序运行时的超时时间。
    char pname[51];        // 本程序运行时的程序名。
} starg;

// 显示程序的帮助
void _help(char *argv[]);

// 把xml解析到参数starg结构中
bool _xmltoarg(char *strxmlbuffer);

CLogFile logfile;

// 判断当前时间是否在程序运行的时间区间内。
bool instarttime();

connection conn;     // 用于执行SQL语句的数据库连接。

// 业务处理主函数。
bool _deletetable();

void EXIT(int sig);

CPActive PActive;

int main(int argc,char *argv[])
{
    if (argc!=3) { _help(argv); return -1; }

    // 关闭全部的信号和输入输出
    // 处理程序退出的信号
    CloseIOAndSignal(); signal(SIGINT,EXIT); signal(SIGTERM,EXIT);

    if (logfile.Open(argv[1],"a+")==false)
    {
        printf("打开日志文件失败（%s）。\n",argv[1]); return -1;
    }

    // 把xml解析到参数starg结构中
    if (_xmltoarg(argv[2])==false) return -1;

    // 判断当前时间是否在程序运行的时间区间内。
    if (instarttime()==false) return 0;

    PActive.AddPInfo(starg.timeout,starg.pname);
    // 注意，在调试程序的时候，可以启用类似以下的代码，防止超时。
    // PActive.AddPInfo(starg.timeout*100,starg.pname);

    if (conn.connecttodb(starg.connstr,NULL,1) != 0)  // 开启自动提交。
    {
        logfile.Write("connect database(%s) failed.\n%s\n",starg.connstr,conn.m_cda.message); EXIT(-1);
    }

    // logfile.Write("connect database(%s) ok.\n",starg.connstr);

    // 业务处理主函数。
    _deletetable();
}

// 显示程序的帮助
void _help(char *argv[])
{
    printf("Using:/project/tools/bin/deletetable_oracle logfilename xmlbuffer\n\n");

    printf("Sample:/project/tools/bin/procctl 3600 /project/tools/bin/deletetable_oracle /log/idc/deletetable_oracle_ZHOBTMIND1.log \"<connstr>invi/psaaword@snorcl11g_130</connstr><tname>T_ZHOBTMIND1</tname><keycol>keyid</keycol><where>where ddatetime<sysdate-0.03</where><starttime>01,02,03,04,05,13</starttime><timeout>120</timeout><pname>deletetable_oracle_ZHOBTMIND1</pname>\"\n\n");

    printf("本程序是数据中心的公共功能模块，用于定时清理表中的数据。\n");

    printf("logfilename 本程序运行的日志文件。\n");
    printf("xmlbuffer   本程序运行的参数，用xml表示，具体如下：\n\n");

    printf("connstr     数据库的连接参数，格式：ip,username,password,dbname,port。\n");
    printf("tname       待清理数据表的表名。\n");
    printf("keycol      待清理数据表的唯一键字段名。\n");
    printf("starttime   程序运行的时间区间，例如02,13表示：如果程序运行时，踏中02时和13时则运行，其它时间\n"\
            "            不运行。如果starttime为空，本参数将失效，只要本程序启动就会执行数据迁移，为了减少\n"\
            "            对数据库的压力，数据迁移一般在数据库最闲的时候时进行。\n");
    printf("where       待清理的数据需要满足的条件，即SQL语句中的where部分。\n");
    printf("timeout     本程序的超时时间，单位：秒，建议设置120以上。\n");
    printf("pname       进程名，尽可能采用易懂的、与其它进程不同的名称，方便故障排查。\n\n");
}

// 把xml解析到参数starg结构中
bool _xmltoarg(char *strxmlbuffer)
{
    memset(&starg,0,sizeof(struct st_arg));

    GetXMLBuffer(strxmlbuffer,"connstr",starg.connstr,100);
    if (strlen(starg.connstr)==0) { logfile.Write("connstr is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"tname",starg.tname,30);
    if (strlen(starg.tname)==0) { logfile.Write("tname is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"keycol",starg.keycol,30);
    if (strlen(starg.keycol)==0) { logfile.Write("keycol is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"where",starg.where,1000);
    if (strlen(starg.where)==0) { logfile.Write("where is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"starttime",starg.starttime,30);

    GetXMLBuffer(strxmlbuffer,"timeout",&starg.timeout);
    if (starg.timeout==0) { logfile.Write("timeout is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"pname",starg.pname,50);
    if (strlen(starg.pname)==0) { logfile.Write("pname is null.\n"); return false; }

    return true;
}

void EXIT(int sig)
{
    logfile.Write("程序退出，sig=%d\n\n",sig);

    conn.disconnect();

    exit(0);
}

// 业务处理主函数。
bool _deletetable()
{
    char keyvalue[51];      // 存放从表中提取的待删除记录的唯一键的值

    CTimer Timer;

    // 从表中提取待删除记录的唯一键
    sqlstatement stmtsel(&conn);
    stmtsel.prepare("select %s from %s %s", starg.keycol, starg.tname, starg.where);
    stmtsel.bindout(1, keyvalue, 50);    // (这个唯一键一般建议使用自增字段，不建议使用其他字段)

    // 拼接绑定删除sql语句where 唯一键 in (...)字符串
    char bindstr[2001];  // 绑定删除sql语句参数的字符串
    char strtemp[11];

    memset(bindstr, 0, sizeof(bindstr));

    for(int i = 0; i < MAXPARAMS; i++)
    {
        memset(strtemp, 0, sizeof(strtemp));
        sprintf(strtemp, ":%lu,", i+1);
        strcat(bindstr, strtemp);
    }
    bindstr[strlen(bindstr) - 1] = 0;

    char kyvalue[MAXPARAMS][51];        // 存放唯一键字段的值

    // 准备删除数据的sql，一次删除MAXPARAMS条记录
    sqlstatement stmtdel(&conn);

    stmtdel.prepare("delete from %s where %s in (%s)", starg.tname, starg.keycol, bindstr);
    for(int i = 0; i < MAXPARAMS; i++)
    {
        stmtdel.bindin(i+1, kyvalue[i], 50);
    }

    int ccount = 0;

    memset(kyvalue, 0, sizeof(kyvalue));

    if(stmtsel.execute() != 0)
    {
        logfile.Write("stmtsel.execute() failed\n%s\n%s\n", stmtsel.m_sql, stmtsel.m_cda.message);
        return false;
    }

    while (true)
    {
        memset(keyvalue, 0, sizeof(keyvalue));
        // 获取结果集
        if(stmtsel.next() != 0) break;

        strcpy(kyvalue[ccount], keyvalue);
        ccount++;

        // 每 MAXPARAMS 条记录执行一次删除语句
        if(ccount == MAXPARAMS)
        {
            if(stmtdel.execute() != 0)
            {
                logfile.Write("stmtdel.execute() failed\n%s\n%s\n", stmtdel.m_sql, stmtdel.m_cda.message);
                return false;
            }
            ccount = 0;
            memset(kyvalue, 0, sizeof(kyvalue));

            PActive.UptATime();
        }
    }

    // 如果不足 MAXPARAMS 条记录的，再执行一次删除
    if(ccount > 0)
    {
        if(stmtdel.execute() != 0)
        {
            logfile.Write("stmtdel.execute() failed\n%s\n%s\n", stmtdel.m_sql, stmtdel.m_cda.message);
            return false;
        }
    }

    if(stmtdel.m_cda.rpc > 0)
    {
        logfile.Write("delete from %s %d rows in %.02fsec.\n",starg.tname,stmtsel.m_cda.rpc,Timer.Elapsed());
    }


    return true;
}

// 判断当前时间是否在程序运行的时间区间内。
bool instarttime()
{
    // 程序运行的时间区间，例如02,13表示：如果程序启动时，踏中02时和13时则运行，其它时间不运行。
    if (strlen(starg.starttime)!=0)
    {
        char strHH24[3];
        memset(strHH24,0,sizeof(strHH24));
        LocalTime(strHH24,"hh24");  // 只获取当前时间中的小时。
        if (strstr(starg.starttime,strHH24)==0) return false;
    }

    return true;
}

```

数据迁移（将数据库表1中的数据迁移到历史数据表）

```c++
/*
 *  程序名：migratetable_oracle.cpp，本程序是数据中心的公共功能模块，用于迁移表中的数据。
*/
#include "_tools_oracle.h"

#define MAXPARAMS 256

struct st_arg
{
  char connstr[101];     // 数据库的连接参数。
  char srctname[31];     // 待迁移的表名。
  char dsttname[31];     // 目的表的表名。
  char keycol[31];       // 待迁移的表的唯一键字段名。
  char where[1001];      // 待迁移的数据需要满足的条件。
  char starttime[31];    // 程序运行的时间区间。
  int  maxcount;         // 每执行一次迁移操作的记录数。
  int  timeout;          // 本程序运行时的超时时间。
  char pname[51];        // 本程序运行时的程序名。
} starg;

// 显示程序的帮助
void _help(char *argv[]);

// 把xml解析到参数starg结构中
bool _xmltoarg(char *strxmlbuffer);

CLogFile logfile;

// 判断当前时间是否在程序运行的时间区间内。
bool instarttime();

connection conn;     

// 业务处理主函数。
bool _migratetable();
 
void EXIT(int sig);

CPActive PActive;

int main(int argc,char *argv[])
{
  if (argc!=3) { _help(argv); return -1; }

  // 关闭全部的信号和输入输出
  // 处理程序退出的信号
  CloseIOAndSignal(); signal(SIGINT,EXIT); signal(SIGTERM,EXIT);

  if (logfile.Open(argv[1],"a+")==false)
  {
    printf("打开日志文件失败（%s）。\n",argv[1]); return -1;
  }

  // 把xml解析到参数starg结构中
  if (_xmltoarg(argv[2])==false) return -1;

  // 判断当前时间是否在程序运行的时间区间内。
  if (instarttime()==false) return 0;

  PActive.AddPInfo(starg.timeout,starg.pname);
  // 注意，在调试程序的时候，可以启用类似以下的代码，防止超时。
  // PActive.AddPInfo(starg.timeout*100,starg.pname);

  if (conn.connecttodb(starg.connstr,NULL) != 0)  
  {
    logfile.Write("connect database(%s) failed.\n%s\n",starg.connstr,conn.m_cda.message); EXIT(-1);
  }

  // 业务处理主函数。
  _migratetable();
}

// 显示程序的帮助
void _help(char *argv[])
{
  printf("Using:/project/tools/bin/migratetable_oracle logfilename xmlbuffer\n\n");

  printf("Sample:/project/tools/bin/procctl 3600 /project/tools/bin/migratetable_oracle /log/idc/migratetable_oracle_ZHOBTMIND1.log \"<connstr>invi/psaaword@snorcl11g_130</connstr><srctname>T_ZHOBTMIND1</srctname><dsttname>T_ZHOBTMIND_HIS</dsttname><keycol>keyid</keycol><where>where ddatetime<sysdate-0.03</where><starttime>01,02,03,04,05,13</starttime><maxcount>300</maxcount><timeout>120</timeout><pname>migratetable_oracle_ZHOBTMIND1</pname>\"\n\n");

  printf("本程序是数据中心的公共功能模块，用于迁移表中的数据。\n");

  printf("logfilename 本程序运行的日志文件。\n");
  printf("xmlbuffer   本程序运行的参数，用xml表示，具体如下：\n\n");

  printf("connstr     数据库的连接参数，格式：ip,username,password,dbname,port。\n");
  printf("srctname    待迁移数据源表的表名\n");
  printf("dsttname    迁移目的表的表名，注意，srctname和dsttname的结构必须完全相同。\n");
  printf("keycol      待迁移数据表的唯一键字段名。\n");
  printf("starttime   程序运行的时间区间，例如02,13表示：如果程序运行时，踏中02时和13时则运行，其它时间\n"\
         "            不运行。如果starttime为空，本参数将失效，只要本程序启动就会执行数据迁移，为了减少\n"\
         "            对数据库的压力，数据迁移一般在数据库最闲的时候时进行。\n");
  printf("where       待迁移的数据需要满足的条件，即SQL语句中的where部分。\n");
  printf("maxcount    每执行一次迁移操作的记录数，不能超过MAXPARAMS(256)。\n");
  printf("timeout     本程序的超时时间，单位：秒，建议设置120以上。\n");
  printf("pname       进程名，尽可能采用易懂的、与其它进程不同的名称，方便故障排查。\n\n");
}

// 把xml解析到参数starg结构中
bool _xmltoarg(char *strxmlbuffer)
{
  memset(&starg,0,sizeof(struct st_arg));

  GetXMLBuffer(strxmlbuffer,"connstr",starg.connstr,100);
  if (strlen(starg.connstr)==0) { logfile.Write("connstr is null.\n"); return false; }

  GetXMLBuffer(strxmlbuffer,"srctname",starg.srctname,30);
  if (strlen(starg.srctname)==0) { logfile.Write("srctname is null.\n"); return false; }

  GetXMLBuffer(strxmlbuffer,"dsttname",starg.dsttname,30);
  if (strlen(starg.dsttname)==0) { logfile.Write("dsttname is null.\n"); return false; }

  GetXMLBuffer(strxmlbuffer,"keycol",starg.keycol,30);
  if (strlen(starg.keycol)==0) { logfile.Write("keycol is null.\n"); return false; }

  GetXMLBuffer(strxmlbuffer,"where",starg.where,1000);
  if (strlen(starg.where)==0) { logfile.Write("where is null.\n"); return false; }

  GetXMLBuffer(strxmlbuffer,"starttime",starg.starttime,30);

  GetXMLBuffer(strxmlbuffer,"maxcount",&starg.maxcount);
  if (starg.maxcount==0) { logfile.Write("maxcount is null.\n"); return false; }
  if (starg.maxcount>MAXPARAMS) starg.maxcount=MAXPARAMS;

  GetXMLBuffer(strxmlbuffer,"timeout",&starg.timeout);
  if (starg.timeout==0) { logfile.Write("timeout is null.\n"); return false; }

  GetXMLBuffer(strxmlbuffer,"pname",starg.pname,50);
  if (strlen(starg.pname)==0) { logfile.Write("pname is null.\n"); return false; }

  return true;
}

void EXIT(int sig)
{
  logfile.Write("程序退出，sig=%d\n\n",sig);

  conn.disconnect();

  exit(0);
}

// 业务处理主函数。
bool _migratetable()
{
  CTimer Timer;

  // 从数据字典中获取表中全部的字段名。
  CTABCOLS TABCOLS;

  // 获取待迁移数据表的字段名，用starg.srctname或starg.dsttname都可以。
  if (TABCOLS.allcols(&conn,starg.dsttname)==false)
  {
    logfile.Write("表%s不存在。\n",starg.dsttname); return false;
  }

  char tmpvalue[51];    // 从数据源表查到的需要迁移记录的key字段的值。

  // 从数据源表提取待迁移记录的唯一键，无需排序。
  sqlstatement stmtsel(&conn);
  stmtsel.prepare("select %s from %s %s",starg.keycol,starg.srctname,starg.where);
  stmtsel.bindout(1,tmpvalue,50);

  // 拼接绑定删除SQL语句where 唯一键 in (...)的字符串。
  char bindstr[2001];    
  char strtemp[11];

  memset(bindstr,0,sizeof(bindstr));

  for (int ii=0;ii<starg.maxcount;ii++)
  {
    memset(strtemp,0,sizeof(strtemp));
    sprintf(strtemp,":%lu,",ii+1);       
    strcat(bindstr,strtemp);
  }

  bindstr[strlen(bindstr)-1]=0;    // 最后一个逗号是多余的。

  char keyvalues[starg.maxcount][51];   // 存放唯一键字段的值。

  // 准备插入和删除表数据的sql，一次迁移starg.maxcount条记录。
  sqlstatement stmtins(&conn);
  stmtins.prepare("insert into %s(%s) select %s from %s where %s in (%s)",starg.dsttname,TABCOLS.m_allcols,TABCOLS.m_allcols,starg.srctname,starg.keycol,bindstr);

  sqlstatement stmtdel(&conn);
  stmtdel.prepare("delete from %s where %s in (%s)",starg.srctname,starg.keycol,bindstr);

  for (int ii=0;ii<starg.maxcount;ii++)
  {
    stmtins.bindin(ii+1,keyvalues[ii],50);
    stmtdel.bindin(ii+1,keyvalues[ii],50);
  }

  int ccount=0;
  memset(keyvalues,0,sizeof(keyvalues));

  if (stmtsel.execute()!=0)
  {
    logfile.Write("stmtsel.execute() failed.\n%s\n%s\n",stmtsel.m_sql,stmtsel.m_cda.message); return false;
  }

  while (true)
  {
    memset(tmpvalue,0,sizeof(tmpvalue));

    // 获取结果集。
    if (stmtsel.next()!=0) break;

    strcpy(keyvalues[ccount],tmpvalue);
    ccount++;

    // 每starg.maxcount条记录执行一次迁移语句。
    if (ccount==starg.maxcount)
    {
      // 先插入starg.dsttname表。
      if (stmtins.execute()!=0)
      {
        logfile.Write("stmtins.execute() failed.\n%s\n%s\n",stmtins.m_sql,stmtins.m_cda.message); 
        if (stmtins.m_cda.rc!=1) return false;
      }

      // 再删除starg.srctname表。
      if (stmtdel.execute()!=0)
      {
        logfile.Write("stmtdel.execute() failed.\n%s\n%s\n",stmtdel.m_sql,stmtdel.m_cda.message); return false;
      }

      conn.commit();

      ccount=0;
      memset(keyvalues,0,sizeof(keyvalues));
    
      PActive.UptATime();
    }
  }

  // 如果不足starg.maxcount条记录，再执行一次迁移。
  if (ccount>0)
  {
    if (stmtins.execute()!=0)
    {
      logfile.Write("stmtins.execute() failed.\n%s\n%s\n",stmtins.m_sql,stmtins.m_cda.message);
      if (stmtins.m_cda.rc!=1) return false;
    }

    if (stmtdel.execute()!=0)
    {
      logfile.Write("stmtdel.execute() failed.\n%s\n%s\n",stmtdel.m_sql,stmtdel.m_cda.message); return false;
    }

    conn.commit();
  }

  if (stmtsel.m_cda.rpc>0) logfile.Write("migrate %s to %s %d rows in %.02fsec.\n",starg.srctname,starg.dsttname,stmtsel.m_cda.rpc,Timer.Elapsed());

  return true;
}

// 判断当前时间是否在程序运行的时间区间内。
bool instarttime()
{
  // 程序运行的时间区间，例如02,13表示：如果程序启动时，踏中02时和13时则运行，其它时间不运行。
  if (strlen(starg.starttime)!=0)
  {
    char strHH24[3];
    memset(strHH24,0,sizeof(strHH24));
    LocalTime(strHH24,"hh24");  // 只获取当前时间中的小时。
    if (strstr(starg.starttime,strHH24)==0) return false;
  }

  return true;
}

```

