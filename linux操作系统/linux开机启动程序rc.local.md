在CentOS7中，在/etc/rc.local脚本文件中编写启动程序的脚本，可以实现开机启动程序。

## 1、/etc/rc.local是/etc/rc.d/rc.local的软链接

执行`ls -l /etc/rc.local`看看。

​                               ![](D:\study\weatherCenterNote\linux操作系统\img\65.png)

/etc/rc.local是/etc/rc.d/rc.local文件的软链接，也就是说他们是同一个文件。

## 2、rc.local文件的原始内容

```shell
#!/bin/bash

# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES

#

# It is highly advisable to create own systemd services or udev rules

# to run scripts during boot instead of using this file.

#

# In contrast to previous versions due to parallel execution during boot

# this script will NOT be run after all other services.

#

# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure

# that this script will be executed during boot.

#

touch /var/lock/subsys/local
```

中文意思如下：

> \# 添加此文件是为了兼容。
>
> \# 强烈建议创建自己的systemd服务或udev规则，以便在引导期间运行脚本，而不是使用此文件。
>
> \# 与以前版本不同，由于在引导期间并行执行，此脚本不会在所有其他服务之后运行。
>
> \# 请注意，必须运行'chmod +x /etc/rc.d/rc.local'，以确保在引导期间执行此脚本。

明白了吧。

虽然Linux强烈建议采用自定义的系统服务实现开机自启动程序，不过我认为在rc.local中配置开机启动程序也是一个不错的方法，因为rc.local的配置更简单明了，所以仍被广泛的使用。

## 3、rc.local文件的配置

rc.local本质上是一个shell脚本文件，可以把启动时需要执行的命令写在里面，启动时将按顺序执行。

接下来我们来测试它。

1）在rc.local中添加以下脚本。

```shell
/usr/bin/date >> /tmp/date1.log # 把当前时间追加写入到/tmp/date1.log中。

/usr/bin/sleep 10 # 睡眠10秒。

/usr/bin/date >> /tmp/date2.log # 把当前时间追加写入到/tmp/date2.log中。
```

2）修改/etc/rc.d/rc.local的可执行权限。

```shell
chmod +x /etc/rc.d/rc.local
```

3）重启服务器。

4）查看日志文件/tmp/date1.log和/tmp/date2.log的内容。

 ![](D:\study\weatherCenterNote\linux操作系统\img\66.png)

## 4、应用经验

1）rc.local脚本在操作系统启动时只执行一次。

2）环境变量的问题。

在rc.local脚本中执行程序时是没有环境变量的，如果您执行的程序需要环境变量，可以在脚本中设置环境变量，也可以用su切换用户来执行，例如：

```shell
su - oracle -c "sqlplus scott/tiger @/tmp/test.sql"
```

以上命令的含义就是以oracle用户登录再执行sqlplus命令。

3）不要让rc.local挂起。

rc.local是一个脚本，是按顺序执行的，执行完一个程序后才会执行下一个程序，如果某程序不是后台程序，就应该加&让程序运行在后台，否则rc.local会挂起。

可以用以下脚本来测试，rc.local的内容如下：

```shell
/usr/bin/date >> /tmp/date1.log # 把当前时间追加写入到/tmp/date1.log中。

/usr/bin/sleep 100 # 睡眠100秒。

/usr/bin/date >> /tmp/date2.log # 把当前时间追加写入到/tmp/date2.log中。
```

如果采用了以上脚本，Linux系统在启动完成100后，才会出现以下的登录界面。

 ![](D:\study\weatherCenterNote\linux操作系统\img\67.png)

## 5、版权声明



C语言技术网原创文章，转载请说明文章的来源、作者和原文的链接。

来源：C语言技术网（[www.freecplus.net](http://www.freecplus.net)）

作者：码农有道