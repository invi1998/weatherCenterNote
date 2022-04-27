Oracle的prepare函数和mysql的prepare函数有点不同，如果sql语句有问题，mysql会返回错误，但是Oracle的prepare不会返回错误，只有在execute执行的时候才会返回错误，所以为了兼容性考虑，建议不要判断prepare的错误代码，而是等到execute的时候再去判断其错误代码。

```c++
// 程序名：dminingmysql_oracle.cpp，本程序是数据中心的公共功能模块，用于从MySQL数据库源表抽取数据，生成xml文件。

#include "_public.h"
#include "_ooci.h"

#define MAXPARAMS   256     // 结果集字段的最大数

int imaxfieldlen = -1;     // 结果集字段值得最大长度

// 程序运行参数的结构体。
struct st_arg
{
  char connstr[101];     // 数据库的连接参数。
  char charset[51];      // 数据库的字符集。
  char selectsql[1024];  // 从数据源数据库抽取数据的SQL语句。
  char fieldstr[501];    // 抽取数据的SQL语句输出结果集字段名，字段名之间用逗号分隔。
  char fieldlen[501];    // 抽取数据的SQL语句输出结果集字段的长度，用逗号分隔。
  char bfilename[31];    // 输出xml文件的前缀。
  char efilename[31];    // 输出xml文件的后缀。
  char outpath[301];     // 输出xml文件存放的目录。
  int  maxcount;         // 输出xml文件最大记录数，0表示无限制。
  char starttime[52];    // 程序运行的时间区间
  char incfield[31];     // 递增字段名。
  char incfilename[301]; // 已抽取数据的递增字段最大值存放的文件。
  char connstr1[101];    // 已抽取数据的递增字段最大值存放的数据库的连接参数。
  int  timeout;          // 进程心跳的超时时间。
  char pname[51];        // 进程名，建议用"dminingmysql_后缀"的方式。
} starg;

char strfieldname[MAXPARAMS][31];       // 结果集字段名数组，从starg.fieldstr中解析得到
int ifieldlen[MAXPARAMS];               // 结果集字段值的最大长度, 从starg.fieldlen中解析得到
int ifieldcount;                            // strfieldname和ifieldlen数组中有效字段的个数
int incfieldpos = -1;                       // 递增字段在结果集数组中的位置

char strxmlfilename[301];                   // xml文件名

long imaxincvalue;                          // 从文件 starg.incfilename中获取到的已经抽取的数据的最大自增字段id

CLogFile logfile;

CPActive PActive;

connection conn, conn1;

// 程序退出和信号2、15的处理函数。
void EXIT(int sig);

void _help();

// 把xml解析到参数starg结构中。
bool _xmltoarg(char *strxmlbuffer);

// 数据抽取的主函数。
bool _dminingmysql();

// 判断当前时间是否在程序运行的时间区间内
bool instarttime();

// 生成xml文件名
void crtxmlfilename();

// 从starg.incfilename文件中获取已抽取数据的最大id
bool readincfile();

// 把已经抽取到的数据中的最大的自增字段的id写入到starg.infilename文件中
bool writeincfile();

int main(int argc,char *argv[])
{
    if (argc!=3) { _help(); return -1; }

    // 关闭全部的信号和输入输出。
    // 设置信号,在shell状态下可用 "kill + 进程号" 正常终止些进程。
    // 但请不要用 "kill -9 +进程号" 强行终止。
    CloseIOAndSignal(); signal(SIGINT,EXIT); signal(SIGTERM,EXIT);

    // 打开日志文件。
    if (logfile.Open(argv[1],"a+")==false)
    {
        printf("打开日志文件失败（%s）。\n",argv[1]); return -1;
    }

    // 解析xml，得到程序运行的参数。
    if (_xmltoarg(argv[2])==false) return -1;

    // 判断当前时间是否在程序运行的时间区间内
    if(instarttime() == false)
    {
        return 0;
    }

    PActive.AddPInfo(starg.timeout, starg.pname);       // 把进程心跳写入共享内存

    // 连接数据库
    if(conn.connecttodb(starg.connstr, starg.charset) != 0)
    {
        logfile.Write("connect database(%s) failed\n%s\n", starg.connstr, conn.m_cda.message);
        return -1;
    }

    logfile.Write("connect database(%s) ok\n", starg.connstr);

    if(strlen(starg.connstr1) != 0)
    {
        // 连接数据库
        if(conn1.connecttodb(starg.connstr1, starg.charset) != 0)
        {
            logfile.Write("connect database(%s) failed\n%s\n", starg.connstr1, conn1.m_cda.message);
            return -1;
        }

        logfile.Write("connect database(%s) ok\n", starg.connstr1);
    }

    // 执行上传文件功能的主函数
    _dminingmysql();

    return 0;
}

void EXIT(int sig)
{
    printf("程序退出，sig=%d\n\n",sig);

    exit(0);
}

void _help()
{
    printf("Using:/project/tools/bin/dminingmysql_oracle logfilename xmlbuffer\n\n");

    printf("Sample:/project/tools/bin/procctl 3600 /project/tools/bin/dminingmysql_oracle /log/idc/dminingoracle_ZHOBTCODE1.log \"<connstr>invi/sh269jgl105@snorcl11g_130</connstr><charset>Simplified Chinese_China.AL32UTF8</charset><selectsql>select obtid,cityname,provname,lat,lon,height from T_ZHOBTCODE1</selectsql><fieldstr>obtid,cityname,provname,lat,lon,height</fieldstr><fieldlen>10,30,30,10,10,10</fieldlen><bfilename>ZHOBTCODE1</bfilename><efilename>HYCZ</efilename><outpath>/idcdata/dmindata1</outpath><timeout>30</timeout><pname>dminingoracle_ZHOBTCODE1</pname>\"\n\n");
    printf("       /project/tools/bin/procctl   30 /project/tools/bin/dminingmysql_oracle /log/idc/dminingoracle_ZHOBTMIND1.log \"<connstr>invi/sh269jgl105@snorcl11g_130</connstr><charset>Simplified Chinese_China.AL32UTF8</charset><selectsql>select obtid,to_char(ddatetime,'yyyymmddhh24miss'),t,p,u,wd,wf,r,vis,keyid from T_ZHOBTMIND1 where keyid>:1 and ddatetime>sysdate-0.04</selectsql><fieldstr>obtid,ddatetime,t,p,u,wd,wf,r,vis,keyid</fieldstr><fieldlen>10,19,8,8,8,8,8,8,8,15</fieldlen><bfilename>ZHOBTMIND1</bfilename><efilename>HYCZ</efilename><outpath>/idcdata/dmindata1</outpath><starttime></starttime><incfield>keyid</incfield><incfilename>/idcdata/dmining/dminingoracle_ZHOBTMIND1_HYCZ.list</incfilename><timeout>30</timeout><pname>dminingoracle_ZHOBTMIND1_HYCZ</pname><maxcount>1000</maxcount><connstr1>invi/sh269jgl105@snorcl11g_130</connstr1>\"\n\n");

    printf("本程序是数据中心的公共功能模块，用于从Oracle数据库源表抽取数据，生成xml文件。\n");
    printf("logfilename 本程序运行的日志文件。\n");
    printf("xmlbuffer   本程序运行的参数，用xml表示，具体如下：\n\n");

    printf("connstr     数据库的连接参数，格式：ip,username,password,dbname,port。\n");
    printf("charset     数据库的字符集，这个参数要与数据源数据库保持一致，否则会出现中文乱码的情况。\n");
    printf("selectsql   从数据源数据库抽取数据的SQL语句，注意：时间函数的百分号%需要四个，显示出来才有两个，被prepare之后将剩一个。\n");
    printf("fieldstr    抽取数据的SQL语句输出结果集字段名，中间用逗号分隔，将作为xml文件的字段名。\n");
    printf("fieldlen    抽取数据的SQL语句输出结果集字段的长度，中间用逗号分隔。fieldstr与fieldlen的字段必须一一对应。\n");
    printf("bfilename   输出xml文件的前缀。\n");
    printf("efilename   输出xml文件的后缀。\n");
    printf("outpath     输出xml文件存放的目录。\n");
    printf("maxcount    输出xml文件的最大记录数，缺省是0，表示无限制，如果本参数取值为0，注意适当加大timeout的取值，防止程序超时。\n");
    printf("starttime   程序运行的时间区间，例如02,13表示：如果程序启动时，踏中02时和13时则运行，其它时间不运行。"\
            "如果starttime为空，那么starttime参数将失效，只要本程序启动就会执行数据抽取，为了减少数据源"\
            "的压力，从数据库抽取数据的时候，一般在对方数据库最闲的时候时进行。\n");
    printf("incfield    递增字段名，它必须是fieldstr中的字段名，并且只能是整型，一般为自增字段。"\
            "如果incfield为空，表示不采用增量抽取方案。");
    printf("incfilename 已抽取数据的递增字段最大值存放的文件，如果该文件丢失，将重新抽取全部的数据。\n");
    printf("connstr1    已抽取数据的递增字段最大值存放的数据库的连接参数。connstr1和incfilename二选一，connstr1优先。");
    printf("timeout     本程序的超时时间，单位：秒。\n");
    printf("pname       进程名，尽可能采用易懂的、与其它进程不同的名称，方便故障排查。\n\n\n");
}

// 把xml解析到参数starg结构中。
bool _xmltoarg(char *strxmlbuffer)
{
    memset(&starg,0,sizeof(struct st_arg));

    GetXMLBuffer(strxmlbuffer,"connstr",starg.connstr,100);
    if (strlen(starg.connstr)==0) { logfile.Write("connstr is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"charset",starg.charset,50);
    if (strlen(starg.charset)==0) { logfile.Write("charset is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"selectsql",starg.selectsql,1000);
    if (strlen(starg.selectsql)==0) { logfile.Write("selectsql is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"fieldstr",starg.fieldstr,500);
    if (strlen(starg.fieldstr)==0) { logfile.Write("fieldstr is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"fieldlen",starg.fieldlen,500);
    if (strlen(starg.fieldlen)==0) { logfile.Write("fieldlen is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"bfilename",starg.bfilename,30);
    if (strlen(starg.bfilename)==0) { logfile.Write("bfilename is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"efilename",starg.efilename,30);
    if (strlen(starg.efilename)==0) { logfile.Write("efilename is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"outpath",starg.outpath,300);
    if (strlen(starg.outpath)==0) { logfile.Write("outpath is null.\n"); return false; }

    GetXMLBuffer(strxmlbuffer,"starttime",starg.starttime,50);  // 可选参数。

    GetXMLBuffer(strxmlbuffer,"incfield",starg.incfield,30);  // 可选参数。

    GetXMLBuffer(strxmlbuffer,"incfilename",starg.incfilename,300);  // 可选参数。

    GetXMLBuffer(strxmlbuffer,"maxcount",&starg.maxcount);  // 可选参数。

    GetXMLBuffer(strxmlbuffer,"connstr1",starg.connstr1,100);  // 可选参数。

    GetXMLBuffer(strxmlbuffer,"timeout",&starg.timeout);   // 进程心跳的超时时间。
    if (starg.timeout==0) { logfile.Write("timeout is null.\n");  return false; }

    GetXMLBuffer(strxmlbuffer,"pname",starg.pname,50);     // 进程名。
    if (strlen(starg.pname)==0) { logfile.Write("pname is null.\n");  return false; }

    // 创建字符串拆分的对象
    CCmdStr Cmdstr;

    // 1、把starg.fieldlen解析到ifieldlen数组中；
    Cmdstr.SplitToCmd(starg.fieldlen, ",");

    // 判断字段数是否超出MAXFIELDCOUNT的限制
    if(Cmdstr.CmdCount() > MAXPARAMS)
    {
        logfile.Write("filedlen的字段数太多，超出了最大限制%d\n", MAXPARAMS);
        return false;
    }

    for(int i = 0; i < Cmdstr.CmdCount(); i++)
    {
        Cmdstr.GetValue(i, &ifieldlen[i]);
        if(ifieldlen[i] > imaxfieldlen)
        {
            // ifieldlen[i] = imaxfieldlen;     // 虽然这样做极端情况下可能会照成取到的字段被截断，但是至少不会造成内存溢出（数组边界溢出）
            // 如果觉得 imaxfieldlen 不够大，可以根据实际情况改大点

            // 如果 imaxfieldlen 是变量，这样会更灵活和节省内存
            imaxfieldlen = ifieldlen[i];
        }

    }

    ifieldcount = Cmdstr.CmdCount();

    // 2、把starg.fieldstr解析到strfieldname数组中；
    Cmdstr.SplitToCmd(starg.fieldstr, ",");

    // 判断字段数是否超出 MAXPARAMS 限制
    if(Cmdstr.CmdCount() > MAXPARAMS)
    {
        logfile.Write("filedlen的字段数太多，超出了最大限制%d\n", MAXPARAMS);
        return false;
    }

    for(int i = 0; i < Cmdstr.CmdCount(); i++)
    {
        Cmdstr.GetValue(i, strfieldname[i], 30);
    }

    // 判断 ifieldlen 和 strfieldname 这两个数组的大小是否相同
    if(ifieldcount != Cmdstr.CmdCount())
    {
        logfile.Write("fieldstr和strfieldname的元素数量不一致\n");
        return false;
    }

    // 3、获取自增字段在结果集中的位置。
    if(strlen(starg.incfield) != 0)
    {
        for(int  i = 0; i <  ifieldcount; i++)
        {
            if(strcmp(starg.incfield, strfieldname[i]) == 0)
            {
                incfieldpos = i;
                break;
            }
        }
        if(incfieldpos == -1)
        {
            logfile.Write("递增字段名%s不在列表%s中\n", starg.incfield, starg.fieldstr);
            return false;
        }

        if(strlen(starg.incfilename) == 0 && strlen(starg.connstr1) == 0)
        {
            logfile.Write("incfilename 和 connstr1 参数必须二选一（增量字段是保存在本地还是远程数据库）\n");
            return false;
        }
    }

    return true;
}

// 数据抽取的主函数。
bool _dminingmysql()
{
    // 从starg.incfilename文件中获取已抽取数据的最大id
    readincfile();

    // 创建sql执行对象
    sqlstatement stmt(&conn);
    // 准备sql语句
    stmt.prepare(starg.selectsql);
    // 绑定sql语句中的变量地址
    // 先定义一个字符串数组，把结果集绑定到这个字符串数组上(数组的大小用抽取字段的个数，然后字符串的大小用之前定义的宏+1)
    char strfieldbvalue[ifieldcount][imaxfieldlen + 1];    // 抽取数据的sql执行之后，存放结果集字段的数组
    // 采用一个循环，将sql的输出结果集绑定到对应的数组变量中
    for(int i = 0; i < ifieldcount; i++)
    {
        stmt.bindout(i+1, strfieldbvalue[i], ifieldlen[i]);
    }

    // 如果是增量抽取，绑定输入参数id
    if(strlen(starg.incfield) != 0)
    {
        stmt.bindin(1, &imaxincvalue);
    }

    if(stmt.execute() != 0)
    {
        logfile.Write("stmt.execute() failed\n%s\n%s\n", stmt.m_sql, stmt.m_cda.message);
        return false;
    }

    // 这里加一个心跳（因为执行sql语句需要时间）
    PActive.UptATime();

    CFile File;     // 用于操作xml文件

    // 循环获取结果集
    while (true)
    {
        memset(strfieldbvalue, 0, sizeof(strfieldbvalue));

        if(stmt.next() != 0) break;

        // 只有单结果集不为空的时候，才生成这个xml文件
        // 先判断文件是否打开
        if(File.IsOpened() == false)
        {
            // 打开文件之间，先把文件名拼接出来
            crtxmlfilename();           // 生成xml文件名
        
            if(File.OpenForRename(strxmlfilename, "w+") == false)
            {
                logfile.Write("File.OpenForRename(\"%s\", \"w+\") failed \n", strxmlfilename);
                return false;
            }

            // 成功打开文件之后，把xml的data标签写进去
            File.Fprintf("<data>\n");
        }

        // 拿到结果集后，将结果集的字段拼接成xml然后写入到对应的输出文件中
        for(int i = 0; i < ifieldcount; i++)
        {
            File.Fprintf("<%s>%s</%s>", strfieldname[i], strfieldbvalue[i], strfieldname[i]);
        }
        File.Fprintf("<endl/>\n");

        // 如果记录数达到starg.maxcount行就切换一个xml文件
        // 这做一个判断，如果结果集的数量超过了starg.maxcount,就关闭当前文件，
        // 这样在处理后续的结果集的时候，就会重新产生新的文件，从而实现了每个xml文件最多只记录starg.maxcount条数据
        if( starg.maxcount > 0 && stmt.m_cda.rpc%starg.maxcount == 0)
        {
            // 关闭文件之前，先写入数据结束标签
            File.Fprintf("</data>\n");

            if(File.CloseAndRename() == false)
            {
                logfile.Write("File.CloseAndRename() failed\n");
                return false;
            }

            // 成功：把xml文件名和记录总数写日志
            logfile.Write("成功生成文件%s(%d)\n", strxmlfilename, starg.maxcount);

            // 这里每写一个文件，就给他做一次心跳
            PActive.UptATime();
        }

        // 更新自增字段的最大值
        // 每次从结果集获取一条记录之后，把自增字段跟之前保存的最大id对比一下，保存更大id值
        if(strlen(starg.connstr1) != 0 && (imaxincvalue < atol(strfieldbvalue[incfieldpos]))) imaxincvalue = atol(strfieldbvalue[incfieldpos]);
    }

    if(File.IsOpened() == true)
    {
        // 关闭文件之前，先写入数据结束标签
        File.Fprintf("</data>\n");

        if(File.CloseAndRename() == false)
        {
            logfile.Write("File.CloseAndRename() failed\n");
            return false;
        }

        if(starg.maxcount == 0)
        {
            // 成功：把xml文件名和记录总数写日志
            logfile.Write("成功生成文件%s(%d)\n", strxmlfilename, stmt.m_cda.rpc);
        }
        else
        {
            // 成功：把xml文件名和记录总数写日志
            logfile.Write("成功生成文件%s(%d)\n", strxmlfilename, stmt.m_cda.rpc%starg.maxcount);
        }
    }
    
    // 把最大的自增字段的值写入到 starg.incfilename文件中
    if(stmt.m_cda.rpc > 0)     // 这里可以先做一个判断，有结果集才去写这个id
    {
        writeincfile();
    }

    return true;
}

// 判断当前时间是否在程序运行的时间区间内
bool instarttime()
{
    // 程序运行的时间区间，例如 02， 13 表示：如果程序启动时，命中02时，和13时这两个时间就运行。其他时间不运行
    if(strlen(starg.starttime) != 0)        // 如果这个参数为空，表示没有时间限制，任何时间都可以运行
    {
        char strHH24[3];
        memset(strHH24, 0, sizeof(strHH24));

        LocalTime(strHH24, "hh24");     // 获取当前小时，采用24时间制

        if(strstr(starg.starttime, strHH24) == 0) return false;     // 如果当前时间不在参数时间里，就返回false
    }

    return true;
}

// 生成xml文件名
void crtxmlfilename()
{
    // xml全路径文件名 = starg.outpath + starg.bfilename + 当前时间 + starg.efilename + 序号 + .xml
    
    // 获取当前时间
    char strLocalTime[21];
    memset(strLocalTime, 0, sizeof(strLocalTime));
    LocalTime(strLocalTime, "yyyymmddhh24miss");

    static int iseq = 0;

    SNPRINTF(strxmlfilename, 300, sizeof(strxmlfilename), "%s/%s_%s_%s_%d.xml", starg.outpath, starg.bfilename, strLocalTime, starg.efilename, iseq++);
}

// 从starg.incfilename文件中获取已抽取数据的最大id
bool readincfile()
{
    imaxincvalue = 0;

    // 如果starg.incfield这个参数为空，表示不是增量抽取
    if(strlen(starg.incfield) == 0) return true;

    if(strlen(starg.connstr1) == 0)     // 采用本地文件方式存储id
    {
        CFile File;

        if(File.Open(starg.incfilename, "r") == false)
        {
            // 如果文件打开失败，有两种可能
            // 1：程序第一次运行程序，这种情况下，不必放回失败
            // 2：也可能是保存这个最大id这个文件丢失了，如果是这种情况，那没办法，只能重新抽取
            return true;
        }

        // 从文件中读取已经抽取的数据的最大id
        char stremp[31];
        File.FFGETS(stremp, 30);

        imaxincvalue = atol(stremp);    // 将其转为整数赋值给 imaxincvalue

        // 将这个最大的id吸入到日志文件中
        logfile.Write("上次已经抽取数据的位置（%s=%ld）\n", starg.incfield, imaxincvalue);
    }
    else        // 采用数据库方式存储id
    {
        // 从数据库表中加载自增字段的最大值
        // create table T_MAXINCVALUE(pname varchar(50), maxincvalue numeric(15), primary key(pname));
        sqlstatement stmt(&conn1);

        stmt.prepare("select maxincvalue form T_MAXINCVALUE where pname = :1");

        stmt.bindin(1, starg.pname, 50);
        stmt.bindout(1, &imaxincvalue);

        // if(stmt.execute() != 0)
        // {
        //     logfile.Write("stmt.execute() faild\n%s\n%s\n", stmt.m_sql, stmt.m_cda.message);
        //     return false;
        // }
        stmt.execute();
        stmt.next();
    }

    return true;
}

// 把已经抽取到的数据中的最大的自增字段的id写入到starg.infilename文件中
bool writeincfile()
{
    // 如果starg.incfield这个参数为空，表示不是增量抽取
    if(strlen(starg.incfield) == 0) return true;

    if(strlen(starg.connstr1) == 0)
    {
        CFile File;

        if(File.Open(starg.incfilename, "w+") == false)
        {
            logfile.Write("File.Open(%s, \"w+\") failed\n", starg.incfilename);
            return false;
        }

        // 把已抽取数据的最大id写入文件
        File.Fprintf("%ld", imaxincvalue);

        // 关闭文件
        File.Close();
    }
    else
    {
        sqlstatement stmt(&conn1);

        stmt.prepare("update T_MAXINCVALUE set maxincvalue=:1 where pname=:2");

        stmt.bindin(1, &imaxincvalue);
        stmt.bindin(2, starg.pname, 50);

        if(stmt.execute() != 0)
        {
            if(stmt.m_cda.rc == 942)
            {
                // 942表示表不存在，不存在那就创建表
                conn1.execute("create table T_MAXINCVALUE(pname varchar2(50), maxincvalue number(15), primary key(pname))");
                conn1.execute("insert into T_MAXINCVALUE values('%s',%ld)",starg.pname,imaxincvalue);
                conn1.commit();
                return true;
            }
            else
            {
                logfile.Write("stmt.execute() faild\n%s\n%s\n", stmt.m_sql, stmt.m_cda.message);
                return false;
            }
        }
        if (stmt.m_cda.rpc==0)
        {
            conn1.execute("insert into T_MAXINCVALUE values('%s',%ld)",starg.pname,imaxincvalue);
        }

        conn1.commit();
    }

    
    return true;
}
```

