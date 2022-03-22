我们知道Windows系统之间可以共享文件系统和打印机，Linux系统采用Samba来实现共享文件系统和打印机的功能。通过SMB协议，Windows和Linux系统之间的文件系统和打印机可以互相访问。

SMB（Server Messages Block）协议是一种在局域网上共享文件系统和打印机的TCP应用层协议， 它为局域网内的不同计算机之间提供文件系统和打印机的共享服务。SMB协议是客户/服务器型， Samba是在Linux系统上实现SMB协议的一个免费软件。

# 一、安装软件包

Samba涉及到四个软件包，有些功能您可能用不上，但是安装了也不会有问题。

1. samba：Samba服务器。
2. samba-client：Samba客户端。
3. samba-common：Samba服务器和客户端相关的软件。
4. cifs-utils：通用的Internet文件系统实用程序，支持与Windows、OS X和其他Unix系统进行跨平台文件共享。

```shell
yum -y install samba samba-client samba-common cifs-utils
```

# 二、修改系统配置

## 1、关闭SELINUX

修改/etc/selinux/config文件，把SELINUX参数的值改为disabled。

```shell
SELINUX =disabled
```

重启linux系统或执行 setenforce 0 使修改马上生效。

## 2、开通防火墙端口

Samba 涉及到以四个端口：UDP 137、UDP 138、TCP 139、TCP 445。

1）防火墙开通samba服务。

```shell
firewall-cmd --zone=public --add-service=samba --permanent
```

2）重启防火墙。

```shell
systemctl restart firewalld
```

## 3、启用smb服务

Samba的服务名是smb。

1）启动smb服务。

```shell
systemctl start smb
```

2）把smb服务设置为开机自启动。

```shell
systemctl enable smb
```

# 三、Samba服务的参数文件

Samba服务的参数文件是/etc/samba/smb.conf，在CentOS7版本的系统中，该文件的原始内容如下：

```shell
\# See smb.conf.example for a more detailed config file or

\# read the smb.conf manpage.

\# Run 'testparm' to verify the config is correct after

\# you modified it.

[global]

​    workgroup = SAMBA

​    security = user

​    passdb backend = tdbsam

​    printing = cups

​    printcap name = cups

​    load printers = yes

​    cups options = raw

[homes]

​    comment = Home Directories

​    valid users = %S, %D%w%S

​    browseable = No

​    read only = No

​    inherit acls = Yes

[printers]

​    comment = All Printers

​    path = /var/tmp

​    printable = Yes

​    create mask = 0600

​    browseable = No

[print$]

​    comment = Printer Drivers

​    path = /var/lib/samba/drivers

​    write list = @printadmin root

​    force group = @printadmin

​    create mask = 0664

​    directory mask = 0775
```

`[global]`组是全局参数，根据不同的需求，我们会修改部分参数。

`[homes]`组是用户主目录参数，我们不关心它。

`[printers]`和`[print$]`组是共享打印机参数，我们不关心它。

`testparm`命令可以测试`smb.conf`配置是否正确。

`testparm -v`命令可以详细的列出`smb.conf`支持的配置参数。

`smb.conf`文件的配置比较麻烦，网上有很多资料，但大部分不准确。我先不介绍`smb.conf`文件中参数的含义，我用实际应用的场景来介绍它的配置。

**共享文件系统的应用场景主要有两种：**

**1）匿名方式：不需要输入用户名和密码，任何人都可以访问共享文件系统；**

**2）用户名/密码方式：需要用户名和密码成功登录后才可以访问共享文件系统。**

# 四、配置任何人都可以访问的共享文件系统

例如您想把服务器/tmp/docs目录共享出来。

## 1、创建测试目录和文件

执行以下脚本，创建/tmp/docs目录，生成测试和文件，并指定用户和组。

```shell
mkdir /tmp/docs # 创建/tmp/docs目录。

ls /usr > /tmp/docs/usr.txt # 把ls /usr的结果输出到/tmp/docs/usr.txt文件。

ls /etc > /tmp/docs/etc.txt # 把ls /etc的结果输出到/tmp/docs/etc.txt文件。

chown -R nobody:nobody /tmp/docs # 修改/tmp/docs目录及文件用户和组。
```

## 2、配置/etc/samba/smb.conf文件

在[global]组中增加以下行：

```shell
map to guest = Bad User
```

在文件最后增加以下行：

```shell
[docs]

​    comment = Fully shared directory

​    path = /tmp/docs

​    public = yes

​    read only = no
```

`[docs]`为待共享的文件系统起个名称，不要求与目录名相同，在windows下将看到这个名称。

`comment`参数是说明文字。

`path`是待共享的Linux目录。

`public`指定guest用户可以访问。

`read only`是否为只读，yes或no。

完整的`/etc/samba/smb.conf`文件的内容如下：

```shell
[global]

​    workgroup = SAMBA

​    security = user

​    map to guest = Bad User

[docs]

​    comment = Fully shared docs(read/write)

​    path = /tmp/docs

​    public = yes

​    read only = no
```

和打印机相关的参数我删除掉了，留着也没用。

## 3、重启samba服务，验证结果

每次修改/etc/samba/smb.conf文件后，要重启smb服务。

```shell
systemctl restart smb
```

在windows上的我的电脑中，输入\\服务器IP，不需要输入服户名和密码就可以访问Linux共享文件系统，如下：

​              ![](.\img\35.png)

您可以修改docs目录中的文件，也可以创建新的目录和文件。

## 4、注意事项

1. security参数要用user，不能用share，share已不支持；
2. map to guest = Bad User，这个配置的意思是将所有用户都映射成guest用户，所以访问共享文件时就不再需要用户名和密码了。
3. 待共享的目录和文件的用户和组要设置成nobody。
4. /etc/samba/smb.conf文件中，global只能一组，共享目录可以配置多个。

# 五、配置需要用户名/密码才能访问的共享文件系统

## 1、创建操作系统用户

操作系统用户可以是普通的用户，也可以是简单的、无需登录的、没有HOME目录的用户，如下：

```shell
useradd -M -s /sbin/nologin test -g bin # 创建test用户，指定组为bin（其它组也行）。
```

## 2、创建测试目录和文件

执行以下脚本，创建/tmp/docs目录，生成测试和文件，并指定用户和组。

```shell
mkdir /tmp/docs # 创建/tmp/docs目录。

ls /usr > /tmp/docs/usr.txt # 把ls /usr的结果输出到/tmp/docs/usr.txt文件。

ls /etc > /tmp/docs/etc.txt # 把ls /etc的结果输出到/tmp/docs/etc.txt文件。

chown -R test:bin /tmp/docs # 修改/tmp/docs目录及文件用户和组。
```

## 3、添加 Samba 用户

操作系统用户不能直接用于Samba服务的登录 ，采用smbpasswd命令把操作系统用户添加到Samba的用户中。

```shell
smbpasswd -a test
```

执行以上命令后，按系统提示两次输入密码，注意，输入的是用于登录Samba服务器的密码，与操作系统的密码没有关系。

smbpasswd 命令是用于维护 Samba 服务器的用户帐号的，具体如下：

1）添加 Samba 用户帐号。

```shell
smbpasswd -a sambauser 
```

2）删除 Samba 用户帐号。

```shell
smbpasswd -x sambauser
```

3）禁用 Samba 用户帐号。

```shell
smbpasswd -d sambauser
```

4）启用 Samba 用户帐号。

```shell
smbpasswd -e sambauser
```

## 4、配置/etc/samba/smb.conf文件

/etc/samba/smb.conf文件的内容如下：

```shell
[global]

​    security = USER

​    workgroup = SAMBA

[docs]

​    comment = Shared docs(read/write)

​    path = /tmp/docs

​    read only = No
```

注意：在/etc/samba/smb.conf文件中，global只能一组，共享目录可以配置多个。

## 5、重启samba服务，验证结果

每次修改/etc/samba/smb.conf文件后，要重启smb服务。

```shell
systemctl restart smb
```

在windows上的我的电脑中，输入\\服务器IP后，按提示输入用户名和密码就可以访问Linux共享文件系统，如下：

 ![](.\img\36.png)

# 六、smb.conf文件详解

smb.conf文件的参数非常多，也很麻烦，如果您有更多的需求，请阅读/etc/samba/smb.conf.example文件，或man 5 smb.conf查看帮助。

我从网上找了一些说明文字，供大家参考，但是，我不保证这些说明文字是正确的，我没有测试验证。

```txt
\# 全局参数

\# ==================Global Settings =================== #

[global]

config file = /usr/local/samba/lib/smb.conf.%m

说明：config file可以让你使用另一个配置文件来覆盖缺省的配置文件。如果文件不存在，则该项无效。这个参数很有用，可以使得samba配置更灵活，可以让一台 samba服务器模拟多台不同配置的服务器。比如，你想让PC1（主机名）这台电脑在访问Samba Server时使用它自己的配置文件，那么先在/etc/samba/host/下为PC1配置一个名为smb.conf.pc1的文件，然后在 smb.conf中加入：config file = /etc/samba/host/smb.conf.%m。这样当PC1请求连接Samba Server时，smb.conf.%m就被替换成smb.conf.pc1。这样，对于PC1来说，它所使用的Samba服务就是由 smb.conf.pc1定义的，而其他机器访问Samba Server则还是应用smb.conf。

 

workgroup = WORKGROUP

说明：设定 Samba Server 所要加入的工作组或者域。

 

server string = Samba Server Version %v

说明：设定 Samba Server 的注释，可以是任何字符串，也可以不填。宏%v表示显示Samba的版本号。

 

netbios name = smbserver

说明：设置Samba Server的NetBIOS名称。如果不填，则默认会使用该服务器的DNS名称的第一部分。netbios name和workgroup名字不要设置成一样了。

 

interfaces = lo eth0 192.168.12.2/24 192.168.13.2/24

说明：设置Samba Server监听哪些网卡，可以写网卡名，也可以写该网卡的IP地址。

 

hosts allow = 127. 192.168.1. 192.168.10.1

说明：表示允许连接到Samba Server的客户端，多个参数以空格隔开。可以用一个IP表示，也可以用一个网段表示。hosts deny 与hosts allow 刚好相反。

例如：hosts allow=172.17.2.EXCEPT172.17.2.50

表示容许来自172.17.2.*的主机连接，但排除172.17.2.50

hosts allow=172.17.2.0/255.255.0.0

表示容许来自172.17.2.0/255.255.0.0子网中的所有主机连接

hosts allow=M1，M2

表示容许来自M1和M2两台计算机连接

hosts allow=@pega

表示容许来自pega网域的所有计算机连接

 

max connections = 0

说明：max connections用来指定连接Samba Server的最大连接数目。如果超出连接数目，则新的连接请求将被拒绝。0表示不限制。

 

deadtime = 0

说明：deadtime用来设置断掉一个没有打开任何文件的连接的时间。单位是分钟，0代表Samba Server不自动切断任何连接。

 

time server = yes/no

说明：time server用来设置让nmdb成为windows客户端的时间服务器。

 

log file = /var/log/samba/log.%m

说明：设置Samba Server日志文件的存储位置以及日志文件名称。在文件名后加个宏%m（主机名），表示对每台访问Samba Server的机器都单独记录一个日志文件。如果pc1、pc2访问过Samba Server，就会在/var/log/samba目录下留下log.pc1和log.pc2两个日志文件。

 

max log size = 50

说明：设置Samba Server日志文件的最大容量，单位为kB，0代表不限制。

 

security = user

说明：设置用户访问Samba Server的验证方式，一共有四种验证方式。

1）share：用户访问Samba Server不需要提供用户名和口令, 安全性能较低，已不兼容。

2）user：Samba Server共享目录只能被授权的用户访问,由Samba Server负责检查账号和密码的正确性。账号和密码要在本Samba Server中建立。

3）server：依靠其他Windows NT/2000或Samba Server来验证用户的账号和密码,是一种代理验证。此种安全模式下,系统管理员可以把所有的Windows用户和口令集中到一个NT系统上,使用 Windows NT进行Samba认证, 远程服务器可以自动认证全部用户和口令,如果认证失败,Samba将使用用户级安全模式作为替代的方式。

4）domain：域安全级别,使用主域控制器(PDC)来完成认证。

 

passdb backend = tdbsam

说明：passdb backend就是用户后台的意思。目前有三种后台：smbpasswd、tdbsam和ldapsam。sam应该是security account manager（安全账户管理）的简写。

1）smbpasswd：该方式是使用smb自己的工具smbpasswd来给系统用户（真实用户或者虚拟用户）设置一个Samba密码，客户端就用这个密码来访问Samba的资源。smbpasswd文件默认在/etc/samba目录下，不过有时候要手工建立该文件。

2）tdbsam： 该方式则是使用一个数据库文件来建立用户数据库。数据库文件叫passdb.tdb，默认在/etc/samba目录下。passdb.tdb用户数据库 可以使用smbpasswd –a来建立Samba用户，不过要建立的Samba用户必须先是系统用户。我们也可以使用pdbedit命令来建立Samba账户。pdbedit命令的 参数很多，我们列出几个主要的。

　　pdbedit –a username：新建Samba账户。

　　pdbedit –x username：删除Samba账户。

　　pdbedit –L：列出Samba用户列表，读取passdb.tdb数据库文件。

　　pdbedit –Lv：列出Samba用户列表的详细信息。

　　pdbedit –c “[D]” –u username：暂停该Samba用户的账号。

　　pdbedit –c “[]” –u username：恢复该Samba用户的账号。

3）ldapsam：该方式则是基于LDAP的账户管理方式来验证用户。首先要建立LDAP服务，然后设置“passdb backend = ldapsam:ldap://LDAP Server”

 

encrypt passwords = yes/no

说明：是否将认证密码加密。因为现在windows操作系统都是使用加密密码，所以一般要开启此项。不过配置文件默认已开启。

 

smb passwd file = /etc/samba/smbpasswd

说明：用来定义samba用户的密码文件。smbpasswd文件如果没有那就要手工新建。

 

username map = /etc/samba/smbusers

说明：用来定义用户名映射，比如可以将root换成administrator、admin等。不过要事先在smbusers文件中定义好。比如：root = administrator admin，这样就可以用administrator或admin这两个用户来代替root登陆Samba Server，更贴近windows用户的习惯。

 

guest account = nobody

说明：用来设置guest用户名。

 

socket options = TCP_NODELAY SO_RCVBUF=8192 SO_SNDBUF=8192

说明：用来设置服务器和客户端之间会话的Socket选项，可以优化传输速度。

 

domain master = yes/no

说明：设置Samba服务器是否要成为网域主浏览器，网域主浏览器可以管理跨子网域的浏览服务。

 

local master = yes/no

说明：local master用来指定Samba Server是否试图成为本地网域主浏览器。如果设为no，则永远不会成为本地网域主浏览器。但是即使设置为yes，也不等于该Samba Server就能成为主浏览器，还需要参加选举。

 

preferred master = yes/no

说明：设置Samba Server一开机就强迫进行主浏览器选举，可以提高Samba Server成为本地网域主浏览器的机会。如果该参数指定为yes时，最好把domain master也指定为yes。使用该参数时要注意：如果在本Samba Server所在的子网有其他的机器（不论是windows NT还是其他Samba Server）也指定为首要主浏览器时，那么这些机器将会因为争夺主浏览器而在网络上大发广播，影响网络性能。

如果同一个区域内有多台Samba Server，将上面三个参数设定在一台即可。

 

os level = 200

说明：设置samba服务器的os level。该参数决定Samba Server是否有机会成为本地网域的主浏览器。os level从0到255，winNT的os level是32，win95/98的os level是1。Windows 2000的os level是64。如果设置为0，则意味着Samba Server将失去浏览选择。如果想让Samba Server成为PDC，那么将它的os level值设大些。

 

domain logons = yes/no

说明：设置Samba Server是否要做为本地域控制器。主域控制器和备份域控制器都需要开启此项。

 

logon script = %u.bat

说明：当使用者用windows客户端登陆，那么Samba将提供一个登陆档。如果设置成%u.bat，那么就要为每个用户提供一个登陆档。如果人比较多， 那就比较麻烦。可以设置成一个具体的文件名，比如start.bat，那么用户登陆后都会去执行start.bat，而不用为每个用户设定一个登陆档了。 这个文件要放置在[netlogon]的path设置的目录路径下。

 

wins support = yes/no

说明：设置samba服务器是否提供wins服务。

 

wins server = wins服务器IP地址

说明：设置Samba Server是否使用别的wins服务器提供wins服务。

 

wins proxy = yes/no

说明：设置Samba Server是否开启wins代理服务。

 

dns proxy = yes/no

说明：设置Samba Server是否开启dns代理服务。

 

load printers = yes/no

说明：设置是否在启动Samba时就共享打印机。

 

printcap name = cups

说明：设置共享打印机的配置文件。

 

printing = cups

说明：设置Samba共享打印机的类型。现在支持的打印系统有：bsd, sysv, plp, lprng, aix, hpux, qnx

 

\# 共享参数

\# ================== Share Definitions ================== #

 

[共享名]

comment = 任意字符串

说明：comment是对该共享的描述，可以是任意字符串。

 

path = 共享目录路径

说 明：path用来指定共享目录的路径。可以用%u、%m这样的宏来代替路径里的unix用户和客户机的Netbios名，用宏表示主要用于[homes] 共享域。例如：如果我们不打算用home段做为客户的共享，而是在/home/share/下为每个Linux用户以他的用户名建个目录，作为他的共享目 录，这样path就可以写成：path = /home/share/%u; 。用户在连接到这共享时具体的路径会被他的用户名代替，要注意这个用户名路径一定要存在，否则，客户机在访问时会找不到网络路径。同样，如果我们不是以用 户来划分目录，而是以客户机来划分目录，为网络上每台可以访问samba的机器都各自建个以它的netbios名的路径，作为不同机器的共享资源，就可以 这样写：path = /home/share/%m 。

 

browseable = yes/no

说明：browseable用来指定该共享是否可以浏览。

 

writable = yes/no

说明：writable用来指定该共享路径是否可写。

 

available = yes/no

说明：available用来指定该共享资源是否可用。

 

admin users = 该共享的管理者

说明：admin users用来指定该共享的管理员（对该共享具有完全控制权限）。

例如：admin users =david，sandy（多个用户中间用逗号隔开）。

 

valid users = 允许访问该共享的用户

说明：valid users用来指定允许访问该共享资源的用户。

例如：valid users = david，@dave，@tech（多个用户或者组中间用逗号隔开，如果要加入一个组就用“@组名”表示。）

 

invalid users = 禁止访问该共享的用户

说明：invalid users用来指定不允许访问该共享资源的用户。

例如：invalid users = root，@bob（多个用户或者组中间用逗号隔开。）

 

write list = 允许写入该共享的用户

说明：write list用来指定可以在该共享下写入文件的用户。

例如：write list = david，@dave

 

public = yes/no

说明：public用来指定该共享是否允许guest账户访问。

 

guest ok = yes/no

说明：意义同public。
```
