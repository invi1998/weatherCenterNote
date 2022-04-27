# Oracle数据入库

准备测试数据（从MySQL数据库中抽取）

```shell
# 每隔1小时把T_ZHOBTCODE表中全部的数据抽取出来，放在/idcdata/xmltodb/vip2目录，给xmltodb_oracle入库到Oracle。
/project/tools/bin/procctl 3600 /project/tools/bin/dminingmysql /log/idc/dminingmysql_ZHOBTCODE_vip2.log "<connstr>127.0.0.1,root,sh269jgl105,mysql,3306</connstr><charset>utf8</charset><selectsql>select obtid,cityname,provname,lat,lon,height from T_ZHOBTCODE</selectsql><fieldstr>obtid,cityname,provname,lat,lon,height</fieldstr><fieldlen>10,30,30,10,10,10</fieldlen><bfilename>ZHOBTCODE</bfilename><efilename>HYCZ</efilename><outpath>/idcdata/xmltodb/vip2</outpath><timeout>30</timeout><pname>dminingmysql_ZHOBTCODE_vip2</pname>"

# 每30秒从T_ZHOBTMIND表中增量抽取出来，放在/idcdata/xmltodb/vip2目录，给xmltodb_oracle入库到Oracle。
/project/tools/bin/procctl   30 /project/tools/bin/dminingmysql /log/idc/dminingmysql_ZHOBTMIND_vip2.log "<connstr>127.0.0.1,root,sh269jgl105,mysql,3306</connstr><charset>utf8</charset><selectsql>select obtid,date_format(ddatetime,'%%Y-%%m-%%d %%H:%%i:%%s'),t,p,u,wd,wf,r,vis,keyid from T_ZHOBTMIND where keyid>:1 and ddatetime>timestampadd(minute,-65,now())</selectsql><fieldstr>obtid,ddatetime,t,p,u,wd,wf,r,vis,keyid</fieldstr><fieldlen>10,19,8,8,8,8,8,8,8,15</fieldlen><bfilename>ZHOBTMIND</bfilename><efilename>HYCZ</efilename><outpath>/idcdata/xmltodb/vip2</outpath><incfield>keyid</incfield><timeout>30</timeout><pname>dminingmysql_ZHOBTMIND_vip2</pname><maxcount>1000</maxcount><connstr1>127.0.0.1,root,sh269jgl105,mysql,3306</connstr1>"

# 清理/idcdata/xmltodb/vip2bak和/idcdata/xmltodb/vip2err目录中文件，防止把空间撑满。
/project/tools/bin/procctl 300 /project/tools/bin/deletefiles /idcdata/xmltodb/vip2    "*" 0.02
/project/tools/bin/procctl 300 /project/tools/bin/deletefiles /idcdata/xmltodb/vip2bak "*" 0.02
/project/tools/bin/procctl 300 /project/tools/bin/deletefiles /idcdata/xmltodb/vip2err "*" 0.02
```

在Oracle数据库中创建表（用户权限，序列生成器，powerDesigner）

```sql
drop table T_ZHOBTCODE cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTCODE                                           */
/*==============================================================*/
drop   sequence SEQ_ZHOBTCODE;
create sequence SEQ_ZHOBTCODE increment by 1 minvalue 1 nocycle;
create table T_ZHOBTCODE 
(
   obtid              varchar2(10)         not null,
   cityname           varchar2(30)         not null,
   provname           varchar2(30)         not null,
   lat                number(8)            not null,
   lon                number(8)            not null,
   height             number(8),
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTCODE primary key (obtid),
   constraint ZHOBTCODE_KEYID unique (keyid)
);

comment on column T_ZHOBTCODE.obtid is
'站点代码';

comment on column T_ZHOBTCODE.cityname is
'城市名称';

comment on column T_ZHOBTCODE.provname is
'省名称';

comment on column T_ZHOBTCODE.lat is
'纬度，单位：0.01度。';

comment on column T_ZHOBTCODE.lon is
'经度，单位：0.01度。';

comment on column T_ZHOBTCODE.height is
'海拔高度，单位：0.1米。';

comment on column T_ZHOBTCODE.upttime is
'更新时间，数据被插入或更新的时间。';

comment on column T_ZHOBTCODE.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

```

新增xmltodb_oracle.cpp程序，把xml文件入库到Oracle的表中

