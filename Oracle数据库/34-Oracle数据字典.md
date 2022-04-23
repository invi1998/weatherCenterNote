# 一、概述

 Oracle通过数据字典来管理和展现数据库信息，数据字典储存数据库的元数据，是数据库的“数据库”。数据字典由4部分组成：内部RDBMS（X$）表、数据字典表、动态性能视图（V$）和（静态）数据字典视图。

数据字典系统表，保存在system表空间中。执行以下语句可以查询所有数据字典：

```sql
select * from dictionary;
```

# 二、内部RDBMS（X$）表

X$表示Oracle数据库的核心部分，这些表用于跟踪数据库内部信息，维持数据库的正常运行。X$表是加密命名的，而且Oracle不做文档说明，也不允许sysdba以外的用户直接访问，显示授权不被允许。X$表是Oracle数据库的运行基础，在数据库启动时由Oracle应用程序动态创建。

# 三、动态性能视图

动态性能视图记录了数据库运行时信息和统计数据，大部分动态性能视图被实时更新以及反映数据库当前状态。在数据库启动时，Oracle动态创建X$表，在此基础上，Oracle创建了GV$和V$视图，GV$即Global V$，除了一些特例外，每个V$都对应一个GV$。GV$产生是为了OPS/RAC环境的需要，每个V$都是基于GV$的，只是GV$多了INST_ID列来显示实例ID。

v$database数据库信息

v$datafile数据文件信息

v$controlfile控制文件信息

v$logfile重做日志信息

v$instance数据库实例信息

v$log日志组信息

v$loghist日志历史信息

v$sga数据库SGA信息

v$parameter初始化参数信息

v$process数据库服务器进程信息

v$bgprocess数据库后台进程信息

v$controlfile_record_section控制文件记载的各部分信息

v$thread线程信息

v$datafile_header数据文件头所记载的信息

v$archived_log归档日志信息

v$archive_dest归档日志的设置信息

v$logmnr_contents归档日志分析的DMLDDL结果信息

v$logmnr_dictionary日志分析的字典文件信息

v$logmnr_logs日志分析的日志列表信息

v$tablespace表空间信息

v$tempfile临时文件信息

v$filestat数据文件的I/O统计信息

v$undostatUndo数据信息

v$rollname在线回滚段信息

v$session会话信息

v$transaction事务信息

v$rollstat回滚段统计信息

v$pwfile_users特权用户信息

v$sqlarea当前查询过的sql语句访问过的资源及相关的信息

v$sql与v$sqlarea基本相同的相关信息

v$sysstat数据库系统状态信息

# 四、数据字典表

数据字典表（Data dictionary table）用以存储表、索引、约束以及其它数据库结构的信息，这些对象通常以“$”结尾（例如：TAB$、OBJ$、TS$等），在创建数据库的时候通过运行$ORACLE_HOME/rdbms/admin/sql.bsq脚本来创建。

# 五、静态数据字典视图

由于X$表和数据字典表通常不能直接被用户访问，Oracle创建了静态数据字典视图来提供用户对于数据字典信息的访问，由于这些信息通常相对稳定，不能直接修改，所以又被称为静态数据字典视图。静态数据字典视图在创建数据库时由$ORACLE_HOME/rdbms/admin/catagory.sql脚本创建。

静态数据字典视图按照前缀的不同通常分成三类，在本质上是为了实现权限控制。在Oracle数据库中，每个用户与方案（Schema）是对应的，Schema是用户所拥有的对象的集合。数据库通过Schema将不同用户的对象隔离开来，用户可以自由的访问自己的对象，但是要访问其他Schema对象就需要相关的授权。

## 1、USER_*（用户所拥有的相关对象信息）

user_objects用户对象信息

user_source数据库用户的所有资源对象信息

user_segments用户的表段信息

user_tables用户的表对象信息

user_tab_columns用户的表列信息

user_constraints用户的对象约束信息

user_sys_privs当前用户的系统权限信息

user_tab_privs当前用户的对象权限信息

user_col_privs当前用户的表列权限信息

user_role_privs当前用户的角色权限信息

user_indexes用户的索引信息

user_ind_columns用户的索引对应的表列信息

user_cons_columns用户的约束对应的表列信息

user_clusters用户的所有簇信息

user_clu_columns用户的簇所包含的内容信息

user_cluster_hash_expressions散列簇的信息

## 2、ALL_*（用于有权限访问的所有对象的信息）

all_users数据库所有用户的信息

all_objects数据库所有的对象的信息

all_def_audit_opts所有默认的审计设置信息

all_tables所有的表对象信息

all_indexes所有的数据库对象索引的信息

## 3、DBA_（数据库所有相关对象的信息）

dba_users数据库用户信息

dba_segments表段信息

dba_extents数据区信息

dba_objects数据库对象信息

dba_tablespaces数据库表空间信息

dba_data_files数据文件设置信息

dba_temp_files临时数据文件信息

dba_rollback_segs回滚段信息

dba_ts_quotas用户表空间配额信息

dba_free_space数据库空闲空间信息

dba_profiles数据库用户资源限制信息

dba_sys_privs用户的系统权限信息

dba_tab_privs用户具有的对象权限信息

dba_col_privs用户具有的列对象权限信息

dba_role_privs用户具有的角色信息

dba_audit_trail审计跟踪记录信息

dba_stmt_audit_opts审计设置信息

dba_audit_object对象审计结果信息

dba_audit_session会话审计结果信息

dba_indexes用户模式的索引信息