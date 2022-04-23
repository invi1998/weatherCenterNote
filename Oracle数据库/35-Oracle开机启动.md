如果服务器断电重启或计划内重启，在服务器的操作系统启动后，需要手工启动数据库实例和监听，本文介绍如何把Oracle数据库的启动和关闭配置成系统服务，在操作系统启动/关闭时，自动启动/关闭Oracle实例和监听。

假设ORACLE_HOME环境变量的值是/oracle/home。

## 1、启动Oracle数据库实例的shell脚本

启动Oracle数据库的脚本为/oracle/home/bin/dbstart，内容如下：

```sh
sqlplus / as sysdba <<EOF

startup;

EOF
```

修改脚本的权限为可执行。

```shell
chmod +x /oracle/home/bin/dbstart
```

## 2、重启Oracle数据库实例的shell脚本

启动Oracle数据库的脚本为/oracle/home/bin/dbrestart，内容如下：

```shell
sqlplus / as sysdba <<EOF

shutdown immediate;

startup;

EOF
```

修改脚本的权限为可执行。

```sh
chmod +x /oracle/home/bin/dbrestart
```

## 3、关闭Oracle数据库实例的shell脚本

启动Oracle数据库的脚本为/oracle/home/bin/dbshut，内容如下：

```sh
sqlplus / as sysdba <<EOF

shutdown immediate;

EOF
```

修改脚本的权限为可执行。

```sh
chmod +x /oracle/home/bin/dbshut
```

## 4、编写Oracle实例的系统服务配置文件

如果把系统服务命名为oracle，则创建服务配置文件/usr/lib/systemd/system/oracle.service，内容如下：

```sh
[Unit]

Description=Oracle RDBMS

After=network.target

 

[Service]

Type=simple

ExecStart=/usr/bin/su - oracle -c "/oracle/home/bin/dbstart >> /tmp/oracle.log"

ExecReload=/usr/bin/su - oracle -c "/oracle/home/bin/dbrestart >> /tmp/oracle.log"

ExecStop=/usr/bin/su - oracle -c "/oracle/home/bin/dbshut >> /tmp/oracle.log"

RemainAfterExit=yes

 

[Install]

WantedBy=multi-user.target
```



## 5、编写Oracle监听的系统服务配置文件

如果把系统服务命名为lsnrctl，则创建服务配置文件/usr/lib/systemd/system/lsnrctl.service，内容如下：

```sh
[Unit]
Description=Oracle RDBMS
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/su - oracle -c "/oracle/home/bin/lsnrctl start >> /tmp/lsnrctl.log"
ExecReload=/usr/bin/su - oracle -c "/oracle/home/bin/lsnrctl reload >> /tmp/lsnrctl.log"
ExecStop=/usr/bin/su - oracle -c "/oracle/home/bin/lsnrctl stop >> /tmp/lsnrctl.log"
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

```



## 6、重新加载服务配置文件

```sh
systemctl daemon-reload
```

每次修改了服务配置文件后，需要重新加载它。

## 7、启动/停止/启重oracle和lsnrctl服务

```shell
systemctl start oracle # 启动oracle服务。

systemctl restart oracle # 重启oracle服务。

systemctl stop oracle # 关闭oracle服务。

systemctl start lsnrctl # 启动lsnrctl服务。

systemctl restart lsnrctl # 重启lsnrctl服务。。

systemctl stop lsnrctl # 关闭lsnrctl服务。
```



## 8、把oracle和lsnrctl服务设置为开机/关机自启动/停止

```shell
systemctl enable oracle # 把Oracle实例服务设置为开机自启动。

systemctl enable lsnrctl # 把Oracle监听服务设置为开机自启动。
```



## 9、查看Oracle实例和监听启动/停止的日志

Oracle实例启动的日志在/tmp/oracle.log文件中。

监听的启动日成在/tmp/lsnrctl.log文件中。

注意，只有通过systemctl启动/关闭Oracle实例和监听才会写日志，手工执行脚本不写日志。