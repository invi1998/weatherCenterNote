# SQL执行脚本

之前编写好测试数据入库程序，但是担心数据把磁盘撑满，就没跑服务。这里开发一个sql脚本，希望定期执行该脚本，然后亲历历史数据。

```shell
delete from T_ZHOBTMIND where ddatatime < timestampadd(minute, -30, now());

# delete from T_ZHOBTMIND1 where ddatatime < timestampadd(minute, -120, now());
# delete from T_ZHOBTMIND2 where ddatatime < timestampadd(minute, -120, now());
# delete from T_ZHOBTMIND3 where ddatatime < timestampadd(minute, -120, now());
```

假设我们有如上脚本，然后可以使用如下命令执行

```shell
mysql -uroot -ppassword -Dmysql < /project/idc/sql/cleardata.sql
```

这条命令里的 `<` 表示输入重定向。

这种命令因为我们的调度程序不支持输入重定向。所以只能自己写一个执行sql脚本的程序，使用这个工具程序来执行一些小的sql任务。

```c++
// execsql.cpp，这是一个工具程序，用于执行一个sql脚本文件。

#include "_public.h"
#include "_mysql.h"

CLogFile logfile;

connection conn;

CPActive PActive;

void EXIT(int sig);

int main(int argc, char *argv[])
{
    // 帮助文档。
    if (argc!=5)
    {
        printf("\n");
        printf("Using:./execsql sqlfile connstr charset logfile\n");

        printf("Example:/project/tools/bin/procctl 120 /project/tools/bin/execsql /project/idc/sql/cleardata.sql \"192.168.31.133,root,psaaword,mysql,3306\" utf8 /log/idc/execsql.log\n\n");

        printf("这是一个工具程序，用于执行一个sql脚本文件。\n");
        printf("sqlfile sql脚本文件名，每条sql语句可以多行书写，分号表示一条sql语句的结束，不支持注释。\n");
        printf("connstr 数据库连接参数：ip,username,password,dbname,port\n");
        printf("charset 数据库的字符集。\n");
        printf("logfile 本程序运行的日志文件名。\n\n");

        return -1;
    }

    // 关闭全部的信号和输入输出。
    // 设置信号,在shell状态下可用 "kill + 进程号" 正常终止些进程。
    // 但请不要用 "kill -9 +进程号" 强行终止。
    CloseIOAndSignal(); signal(SIGINT,EXIT); signal(SIGTERM,EXIT);

    // 打开日志文件。
    if (logfile.Open(argv[4],"a+")==false)
    {
        printf("打开日志文件失败（%s）。\n",argv[4]); return -1;
    }

    PActive.AddPInfo(500,"execsql");   // 进程的心跳，时间长的点。
    // 注意，在调试程序的时候，可以启用类似以下的代码，防止超时。
    // PActive.AddPInfo(5000,"obtcodetodb");

    // 连接数据库，不启用事务。(第三个参数：autocommitopt：是否启用自动提交，0-不启用，1-启用，缺省是不启用)，
    // 这里不启用事务，就是执行完sql后就自动提交，所以第三个参数设置为 1
    if(conn.connecttodb(argv[2], argv[3], 1) != 0)
    {
        logfile.Write("connect database(%s) failed\n%s\n", argv[2], conn.m_cda.message);
        return -1;
    }

    logfile.Write("connect database(%s) ok\n", argv[2]);

    CFile File;
    if(File.Open(argv[1], "r") == false)
    {
        logfile.Write("File.open(%s) failed\n", argv[1]);
        EXIT(-1);
    }

    char strsql[1001];     // 存取从sql文件中读取到的sql语句

    while(true)
    {
        memset(strsql, 0, sizeof(strsql));

        // 从文件中读取一行，以分号（;）结束
        if(File.FFGETS(strsql, 1000, ";") == false) break;

        // 删除sql语句最后的分号
        char* pp = strstr(strsql, ";");
        if(pp == 0) continue;
        pp[0] = 0;

        logfile.Write("%s\n", strsql);

        // 执行sql语句
        int iret = conn.execute(strsql);

        // 把sql执行结果写日志
        if(iret == 0)
        {
            logfile.Write("exec ok(rpc=%d)\n", conn.m_cda.rpc);
        }
        else
        {
            logfile.Write("exec failed(%s)\n", conn.m_cda.message);
        }

        PActive.UptATime();         // 每执行一次sql语句，做一次进程心跳更新
    }

    logfile.WriteEx("\n");

    return 0;
}

void EXIT(int sig)
{
  logfile.Write("程序退出，sig=%d\n\n",sig);

  conn.disconnect();

  exit(0);
}

```

然后待执行的sql语句（清理历史数据的脚本）

```sql
delete from T_ZHOBTMIND where ddatetime < timestampadd(minute, -120, now());
```

