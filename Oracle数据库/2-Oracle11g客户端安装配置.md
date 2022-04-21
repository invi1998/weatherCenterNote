Oracle数据库管理员和开发者一定希望在自己台式电脑的Windows系统中搭建Oracle客户端工作环境。

Oracle客户端包括两个部分：

1）Oracle数据库客户端环境，提供了Oracle客户端驱动和配置工具；

2）Oracle客户端工具，连接Oracle数据库的客户端软件比较多，如Navicat Premium，但是，PL/SQL Developer专为Oracle数据库定制开发的，功能强大，使用方便，是最佳的选择。

# 一、Oracle数据客户端环境

## 1、简易客户端库

从课件中下载instantclient-basic-windows.x64-11.2.0.4.0.zip压缩文件，在任意目录中解压，不需要安装。

从Oracle的网站下载，链接如下：

https://www.oracle.com/cn/database/technologies/instant-client/winx64-64-downloads.html

简易的客户端只包含了Oracle客户端必要的库文件。

## 2、完整的客户端安装包

从课件中下载[win64_11gR2_client.zip](https://www.oracle.com/database/technologies/112010-win64soft.html#license-lightbox)压缩文件。

**1****）运行安装程序。**

点击setup运行后，会弹出一个DOS窗口，可能需要等几十秒。

​                               

**2****）如果出现了以下窗口，选“是(Y)”继续。**

 

**3****）安装类型选择“管理员”。**

 

**4****）语言选择“简体中文”和“英语”。**

 

**5****）指定Oracle的基目录和软件安装位置，可以用缺省值，也可以如下图。**

 

**6****）执行先决条件检查。**

如果出现了“执行先决条件检查”失败，勾选“全部忽略”后再下一步。

 

**7****）确认安装信息。**

 

**8****）安装进行中。**

 

**9****）安全中心警报。**

如果安装进行中出现Windows安全中心警报，选择“允许访问”。

 

**10****）安装完成。**

 

# 二、PL/SQL Developer的安装

## 1、下载软件安装包

从课件中下载plsqldev1104x64.exe文件。

## 2、安装软件包

PL/SQL Developer软件的安装没有任何技术含量，下一步再下一步就可以了。

# 三、配置Oracle客户端环境

## 1、配置数据库连接参数

从课件中的软件安装包目录下把tnsnames.ora文件复制到（Oracle客户端软件的安装位置）\network\admin目录中。

 

 

用写字板或其它的文本编辑软件（必须以系统管理员身证运行）打开tnsnames.ora，输入数据库配置参数，如下：

 

文本内容如下：

snorcl11g_143 =

 (DESCRIPTION =

  (ADDRESS_LIST =

   (ADDRESS = (PROTOCOL = TCP)(HOST = 122.152.209.143)(PORT = 1521))

  )

  (CONNECT_DATA =

   (SID = snorcl11g)

   (SERVER = DEDICATED)

  )

 )

以上的参数中，您只需要关心四个内容。

1）数据库名，或数据库服务名，或tnsname，这个名称由您自定义，如snorcl11g_143

2）数据库服务器的ip地址，您的服务器ip是多少就填多少，如：(HOST = 122.152.209.143)

3）数据库服务器监听的端口，缺省是1521，如：(PORT = 1521)

4）数据库的SID，即ORACLE_SID，如：(SID = snorcl11g)

## 2、启动PL/SQL Developer软件

输入登录数据库的用户名、密码和数据库名。

 

## 3、打开SQL窗口

 

## 4、执行SQL语句

 

## 5、最常用的Objects窗口

 

# 四、客户端环境变量

## 1、Path环境变量

Oracle客户端软件安装完成后，会修改Windows系统变量的Path环境变量，如下：

 

## 2、注册表

 regedit.exe

HKEY_LOCAL_MACHINE -> SOFTWARE -> ORACLE -> KEY_OraClient11g_home1

 

# 五、判断客户端是否能连上数据库

## 1、打开DOS窗口

 

## 2、判断数据库的监听端口

telnet数据库服务器的1521端口。

如果成功，会出现一个空白窗口，如下：

 

如果失败，会出现如下提示：

 

如果telnet数据库的1521端口成功，表示网络和防火墙都没有问题。如果失败，有四种可能：1）Oracle数据库没有启动监听服务；2）Oracle数据库服务器的防火墙没有开通1521端口；3）云平台的安全组（或访问策略）没有开通1521端口；4）网络故障，网络不通。

## 3、tnsping判断数据库客户端配置

 

以上成功的情况，如果出现其它内容，则表示tnsnames.ora文件中的配置不正确。但是，要注意一个问题，如果tnsname中的sid配置不正确，tnsping也是成功的，所以tnsping成功，并不表示客户端可以正常连接。

## 4、Windows下的sqlplus

Oracle的客户端软件自带sqlplus工具，也可以登录数据库。

 