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





## oracle建表语句

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


drop table T_ZHOBTCODE1 cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTCODE1                                          */
/*==============================================================*/
drop   sequence SEQ_ZHOBTCODE1;
create sequence SEQ_ZHOBTCODE1 increment by 1 minvalue 1 nocycle;
create table T_ZHOBTCODE1 
(
   obtid              varchar2(10)         not null,
   cityname           varchar2(30)         not null,
   provname           varchar2(30)         not null,
   lat                number(8)            not null,
   lon                number(8)            not null,
   height             number(8)            not null,
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTCODE1 primary key (obtid),
   constraint ZHOBTCODE1_KEYID unique (keyid)
);

comment on column T_ZHOBTCODE1.obtid is
'站点代码';

comment on column T_ZHOBTCODE1.cityname is
'城市名称';

comment on column T_ZHOBTCODE1.provname is
'省名称';

comment on column T_ZHOBTCODE1.lat is
'纬度，单位：0.01度。';

comment on column T_ZHOBTCODE1.lon is
'经度，单位：0.01度。';

comment on column T_ZHOBTCODE1.height is
'海拔高度，单位：0.1米。';

comment on column T_ZHOBTCODE1.upttime is
'更新时间。';

comment on column T_ZHOBTCODE1.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

drop table T_ZHOBTCODE2 cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTCODE2                                          */
/*==============================================================*/
create table T_ZHOBTCODE2 
(
   obtid              varchar2(10)         not null,
   cityname           varchar2(30)         not null,
   provname           varchar2(30)         not null,
   lat                number(8)            not null,
   lon                number(8)            not null,
   height             number(8)            not null,
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTCODE2 primary key (obtid),
   constraint ZHOBTCODE2_KEYID unique (keyid)
);

comment on column T_ZHOBTCODE2.obtid is
'站点代码';

comment on column T_ZHOBTCODE2.cityname is
'城市名称';

comment on column T_ZHOBTCODE2.provname is
'省名称';

comment on column T_ZHOBTCODE2.lat is
'纬度，单位：0.01度。';

comment on column T_ZHOBTCODE2.lon is
'经度，单位：0.01度。';

comment on column T_ZHOBTCODE2.height is
'海拔高度，单位：0.1米。';

comment on column T_ZHOBTCODE2.upttime is
'更新时间。';

comment on column T_ZHOBTCODE2.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

drop table T_ZHOBTCODE3 cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTCODE3                                          */
/*==============================================================*/
create table T_ZHOBTCODE3 
(
   obtid              varchar2(10)         not null,
   cityname           varchar2(30)         not null,
   provname           varchar2(30)         not null,
   lat                number(8)            not null,
   lon                number(8)            not null,
   height             number(8)            not null,
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTCODE3 primary key (obtid),
   constraint ZHOBTCODE3_KEYID unique (keyid)
);

comment on column T_ZHOBTCODE3.obtid is
'站点代码';

comment on column T_ZHOBTCODE3.cityname is
'城市名称';

comment on column T_ZHOBTCODE3.provname is
'省名称';

comment on column T_ZHOBTCODE3.lat is
'纬度，单位：0.01度。';

comment on column T_ZHOBTCODE3.lon is
'经度，单位：0.01度。';

comment on column T_ZHOBTCODE3.height is
'海拔高度，单位：0.1米。';

comment on column T_ZHOBTCODE3.upttime is
'更新时间。';

comment on column T_ZHOBTCODE3.keyid is
'记录编号，从与本表同名的序列生成器中获取。';


drop index IDX_ZHOBTMIND_3;

drop index IDX_ZHOBTMIND_2;

drop index IDX_ZHOBTMIND_1;

drop table T_ZHOBTMIND cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTMIND                                           */
/*==============================================================*/
drop   sequence SEQ_ZHOBTMIND;
create sequence SEQ_ZHOBTMIND increment by 1 minvalue 1 nocycle;
create table T_ZHOBTMIND 
(
   obtid              varchar2(10)         not null,
   ddatetime          date                 not null,
   t                  number(8),
   p                  number(8),
   u                  number(8),
   wd                 number(8),
   wf                 number(8),
   r                  number(8),
   vis                number(8),
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTMIND primary key (obtid, ddatetime),
   constraint ZHOBTMIND_KEYID unique (keyid)
);

comment on column T_ZHOBTMIND.obtid is
'站点代码。';

comment on column T_ZHOBTMIND.ddatetime is
'数据时间，精确到分钟。';

comment on column T_ZHOBTMIND.t is
'湿度，单位：0.1摄氏度。';

comment on column T_ZHOBTMIND.p is
'气压，单位：0.1百帕。';

comment on column T_ZHOBTMIND.u is
'相对湿度，0-100之间的值。';

comment on column T_ZHOBTMIND.wd is
'风向，0-360之间的值。';

comment on column T_ZHOBTMIND.wf is
'风速：单位0.1m/s。';

comment on column T_ZHOBTMIND.r is
'降雨量：0.1mm。';

comment on column T_ZHOBTMIND.vis is
'能见度：0.1米。';

comment on column T_ZHOBTMIND.upttime is
'更新时间。';

comment on column T_ZHOBTMIND.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

/*==============================================================*/
/* Index: IDX_ZHOBTMIND_1                                       */
/*==============================================================*/
create unique index IDX_ZHOBTMIND_1 on T_ZHOBTMIND (
   ddatetime ASC,
   obtid ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND_2                                       */
/*==============================================================*/
create index IDX_ZHOBTMIND_2 on T_ZHOBTMIND (
   ddatetime ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND_3                                       */
/*==============================================================*/
create index IDX_ZHOBTMIND_3 on T_ZHOBTMIND (
   obtid ASC
);


drop index IDX_ZHOBTMIND1_3;

drop index IDX_ZHOBTMIND1_2;

drop index IDX_ZHOBTMIND1_1;

drop table T_ZHOBTMIND1 cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTMIND1                                          */
/*==============================================================*/
drop   sequence SEQ_ZHOBTMIND1;
create sequence SEQ_ZHOBTMIND1 increment by 1 minvalue 1 nocycle;
create table T_ZHOBTMIND1 
(
   obtid              varchar2(10)         not null,
   ddatetime          date                 not null,
   t                  number(8),
   p                  number(8),
   u                  number(8),
   wd                 number(8),
   wf                 number(8),
   r                  number(8),
   vis                number(8),
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTMIND1 primary key (obtid, ddatetime),
   constraint ZHOBTMIND1_KEYID unique (keyid)
);

comment on column T_ZHOBTMIND1.obtid is
'站点代码。';

comment on column T_ZHOBTMIND1.ddatetime is
'数据时间，精确到分钟。';

comment on column T_ZHOBTMIND1.t is
'湿度，单位：0.1摄氏度。';

comment on column T_ZHOBTMIND1.p is
'气压，单位：0.1百帕。';

comment on column T_ZHOBTMIND1.u is
'相对湿度，0-100之间的值。';

comment on column T_ZHOBTMIND1.wd is
'风向，0-360之间的值。';

comment on column T_ZHOBTMIND1.wf is
'风速：单位0.1m/s。';

comment on column T_ZHOBTMIND1.r is
'降雨量：0.1mm。';

comment on column T_ZHOBTMIND1.vis is
'能见度：0.1米。';

comment on column T_ZHOBTMIND1.upttime is
'更新时间。';

comment on column T_ZHOBTMIND1.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

/*==============================================================*/
/* Index: IDX_ZHOBTMIND1_1                                      */
/*==============================================================*/
create unique index IDX_ZHOBTMIND1_1 on T_ZHOBTMIND1 (
   ddatetime ASC,
   obtid ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND1_2                                      */
/*==============================================================*/
create index IDX_ZHOBTMIND1_2 on T_ZHOBTMIND1 (
   ddatetime ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND1_3                                      */
/*==============================================================*/
create index IDX_ZHOBTMIND1_3 on T_ZHOBTMIND1 (
   obtid ASC
);

drop index IDX_ZHOBTMIND2_3;

drop index IDX_ZHOBTMIND2_2;

drop index IDX_ZHOBTMIND2_1;

drop table T_ZHOBTMIND2 cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTMIND2                                          */
/*==============================================================*/
create table T_ZHOBTMIND2 
(
   obtid              varchar2(10)         not null,
   ddatetime          date                 not null,
   t                  number(8),
   p                  number(8),
   u                  number(8),
   wd                 number(8),
   wf                 number(8),
   r                  number(8),
   vis                number(8),
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTMIND2 primary key (obtid, ddatetime),
   constraint ZHOBTMIND2_KEYID unique (keyid)
);

comment on column T_ZHOBTMIND2.obtid is
'站点代码。';

comment on column T_ZHOBTMIND2.ddatetime is
'数据时间，精确到分钟。';

comment on column T_ZHOBTMIND2.t is
'湿度，单位：0.1摄氏度。';

comment on column T_ZHOBTMIND2.p is
'气压，单位：0.1百帕。';

comment on column T_ZHOBTMIND2.u is
'相对湿度，0-100之间的值。';

comment on column T_ZHOBTMIND2.wd is
'风向，0-360之间的值。';

comment on column T_ZHOBTMIND2.wf is
'风速：单位0.1m/s。';

comment on column T_ZHOBTMIND2.r is
'降雨量：0.1mm。';

comment on column T_ZHOBTMIND2.vis is
'能见度：0.1米。';

comment on column T_ZHOBTMIND2.upttime is
'更新时间。';

comment on column T_ZHOBTMIND2.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

/*==============================================================*/
/* Index: IDX_ZHOBTMIND2_1                                      */
/*==============================================================*/
create unique index IDX_ZHOBTMIND2_1 on T_ZHOBTMIND2 (
   ddatetime ASC,
   obtid ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND2_2                                      */
/*==============================================================*/
create index IDX_ZHOBTMIND2_2 on T_ZHOBTMIND2 (
   ddatetime ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND2_3                                      */
/*==============================================================*/
create index IDX_ZHOBTMIND2_3 on T_ZHOBTMIND2 (
   obtid ASC
);

drop index IDX_ZHOBTMIND3_3;

drop index IDX_ZHOBTMIND3_2;

drop index IDX_ZHOBTMIND3_1;

drop table T_ZHOBTMIND3 cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTMIND3                                          */
/*==============================================================*/
create table T_ZHOBTMIND3 
(
   obtid              varchar2(10)         not null,
   ddatetime          date                 not null,
   t                  number(8),
   p                  number(8),
   u                  number(8),
   wd                 number(8),
   wf                 number(8),
   r                  number(8),
   vis                number(8),
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTMIND3 primary key (obtid, ddatetime),
   constraint ZHOBTMIND3_KEYID unique (keyid)
);

comment on column T_ZHOBTMIND3.obtid is
'站点代码。';

comment on column T_ZHOBTMIND3.ddatetime is
'数据时间，精确到分钟。';

comment on column T_ZHOBTMIND3.t is
'湿度，单位：0.1摄氏度。';

comment on column T_ZHOBTMIND3.p is
'气压，单位：0.1百帕。';

comment on column T_ZHOBTMIND3.u is
'相对湿度，0-100之间的值。';

comment on column T_ZHOBTMIND3.wd is
'风向，0-360之间的值。';

comment on column T_ZHOBTMIND3.wf is
'风速：单位0.1m/s。';

comment on column T_ZHOBTMIND3.r is
'降雨量：0.1mm。';

comment on column T_ZHOBTMIND3.vis is
'能见度：0.1米。';

comment on column T_ZHOBTMIND3.upttime is
'更新时间。';

comment on column T_ZHOBTMIND3.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

/*==============================================================*/
/* Index: IDX_ZHOBTMIND3_1                                      */
/*==============================================================*/
create unique index IDX_ZHOBTMIND3_1 on T_ZHOBTMIND3 (
   ddatetime ASC,
   obtid ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND3_2                                      */
/*==============================================================*/
create index IDX_ZHOBTMIND3_2 on T_ZHOBTMIND3 (
   ddatetime ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND3_3                                      */
/*==============================================================*/
create index IDX_ZHOBTMIND3_3 on T_ZHOBTMIND3 (
   obtid ASC
);


drop index IDX_ZHOBTMIND_HIS_3;

drop index IDX_ZHOBTMIND_HIS_2;

drop index IDX_ZHOBTMIND_HIS_1;

drop table T_ZHOBTMIND_HIS cascade constraints;

/*==============================================================*/
/* Table: T_ZHOBTMIND_HIS                                       */
/*==============================================================*/
create table T_ZHOBTMIND_HIS 
(
   obtid              varchar2(10)         not null,
   ddatetime          date                 not null,
   t                  number(8),
   p                  number(8),
   u                  number(8),
   wd                 number(8),
   wf                 number(8),
   r                  number(8),
   vis                number(8),
   upttime            date                 default sysdate not null,
   keyid              number(15)           not null,
   constraint PK_ZHOBTMIND_HIS primary key (obtid, ddatetime),
   constraint ZHOBTMIND_HIS_KEYID unique (keyid)
);

comment on column T_ZHOBTMIND_HIS.obtid is
'站点代码。';

comment on column T_ZHOBTMIND_HIS.ddatetime is
'数据时间，精确到分钟。';

comment on column T_ZHOBTMIND_HIS.t is
'湿度，单位：0.1摄氏度。';

comment on column T_ZHOBTMIND_HIS.p is
'气压，单位：0.1百帕。';

comment on column T_ZHOBTMIND_HIS.u is
'相对湿度，0-100之间的值。';

comment on column T_ZHOBTMIND_HIS.wd is
'风向，0-360之间的值。';

comment on column T_ZHOBTMIND_HIS.wf is
'风速：单位0.1m/s。';

comment on column T_ZHOBTMIND_HIS.r is
'降雨量：0.1mm。';

comment on column T_ZHOBTMIND_HIS.vis is
'能见度：0.1米。';

comment on column T_ZHOBTMIND_HIS.upttime is
'更新时间。';

comment on column T_ZHOBTMIND_HIS.keyid is
'记录编号，从与本表同名的序列生成器中获取。';

/*==============================================================*/
/* Index: IDX_ZHOBTMIND_HIS_1                                   */
/*==============================================================*/
create unique index IDX_ZHOBTMIND_HIS_1 on T_ZHOBTMIND_HIS (
   ddatetime ASC,
   obtid ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND_HIS_2                                   */
/*==============================================================*/
create index IDX_ZHOBTMIND_HIS_2 on T_ZHOBTMIND_HIS (
   ddatetime ASC
);

/*==============================================================*/
/* Index: IDX_ZHOBTMIND_HIS_3                                   */
/*==============================================================*/
create index IDX_ZHOBTMIND_HIS_3 on T_ZHOBTMIND_HIS (
   obtid ASC
);


exit;

```

