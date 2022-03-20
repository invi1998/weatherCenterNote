# 一、access库函数

access函数用于判断当前操作系统用户对文件或目录的存取权限。

包含头文件：

```c
#include <unistd.h>
```

函数声明：

```c
int access(const char *pathname, int mode);
```

参数说明：

pathname文件名或目录名，可以是当前目录的文件或目录，也可以列出全路径。

mode 需要判断的存取权限。在头文件unistd.h中的预定义如下：

```c
#define R_OK 4   // R_OK 只判断是否有读权限

#define W_OK 2  // W_OK 只判断是否有写权限

#define X_OK 1   // X_OK 判断是否有执行权限

#define F_OK 0   // F_OK 只判断是否存在
```

返回值：

当pathname满足mode的条件时候返回0，不满足返回-1。

在实际开发中，access函数主要用于判断文件或目录是否是存在。

# 二、stat库函数

## 1、stat结构体

struct stat结构体用于存放文件和目录的状态信息，如下：

```c
struct stat

{

 dev_t st_dev;  // device 文件的设备编号

 ino_t st_ino;  // inode 文件的i-node

 mode_t st_mode;  // protection 文件的类型和存取的权限

 nlink_t st_nlink;  // number of hard links 连到该文件的硬连接数目, 刚建立的文件值为1.

 uid_t st_uid;  // user ID of owner 文件所有者的用户识别码

 gid_t st_gid;  // group ID of owner 文件所有者的组识别码

 dev_t st_rdev; // device type 若此文件为设备文件, 则为其设备编号

 off_t st_size; // total size, in bytes 文件大小, 以字节计算

 unsigned long st_blksize; // blocksize for filesystem I/O 文件系统的I/O 缓冲区大小.

 unsigned long st_blocks; // number of blocks allocated 占用文件区块的个数, 每一区块大小为512 个字节.

 time_t st_atime; // time of lastaccess 文件最近一次被存取或被执行的时间, 一般只有在用mknod、 utime、read、write 与tructate 时改变.

 time_t st_mtime; // time of last modification 文件最后一次被修改的时间, 一般只有在用mknod、 utime 和write 时才会改变

 time_t st_ctime; // time of last change i-node 最近一次被更改的时间, 此参数会在文件所有者、组、 权限被更改时更新

};
```

struct stat结构体的成员变量比较多，对程序员来说，重点关注st_mode、st_size和st_mtime成员就可以了。注意st_mtime是一个整数表达的时间，需要程序员自己写代码转换格式。

st_mode成员的取值很多，或者使用如下两个宏来判断。

```c
 S_ISREG(st_mode) // 是否为一般文件 

 S_ISDIR(st_mode) // 是否为目录 
```

## 2、stat库函数

包含头文件：

```c
#include <sys/types.h>

#include <sys/stat.h>

#include <unistd.h>
```

函数声明：

```c
int stat(const char *path, struct stat *buf);
```

stat函数获取path指定文件或目录的信息，并将信息保存到结构体buf中，执行成功返回0，失败返回-1。

**示例（book145.c）**

```c
/*

 \* 程序名：book145.c，此程序演示目录和文件的存取权限和状态信息

*/

\#include <stdio.h>

\#include <sys/stat.h>

\#include <unistd.h>

 

// 本程序运行要带一个参数，即文件或目录名

int main(int argc,char *argv[])

{

 if (argc != 2) { printf("请指定目录或文件名。\n"); return -1; }

 

 if (access(argv[1],F_OK) != 0) { printf("文件或目录%s不存在。\n",argv[1]); return -1; }

 

 struct stat ststat;

 

 // 获取文件的状态信息

 if (stat(argv[1],&ststat) != 0) return -1;

 

 if (S_ISREG(ststat.st_mode)) printf("%s是一个文件。\n",argv[1]);

 if (S_ISDIR(ststat.st_mode)) printf("%s是一个目录。\n",argv[1]);

}
```



**运行效果**

![](./img/2.png)

# 三、utime库函数

utime函数用于修改文件的存取时间和更改时间。

包含头文件：

```c
#include <utime.h>
```

函数声明：

```c
int utime(const char *filename, const struct utimbuf *times);
```

函数说明：utime()用来修改参数filename 文件所属的inode 存取时间。如果参数times为空指针(NULL), 则该文件的存取时间和更改时间全部会设为目前时间。结构utimbuf 定义如下：

```c
struct utimbuf

{

 time_t actime;

 time_t modtime;

};
```

返回值：执行成功则返回0，失败返回-1。

# 四、rename库函数

rename函数用于重命名文件或目录，相当于操作系统的mv命令，对程序员来说，在程序中极少重命名目录，但重命名文件是经常用到的功能。

包含头文件：

```c
#include <stdio.h>
```

函数声明：

```c
int rename(const char *oldpath, const char *newpath);
```

参数说明：

oldpath 文件或目录的原名。

newpath 文件或目录的新的名称。

返回值：0-成功，-1-失败。

# 五、remove库函数

remove函数用于删除文件或目录，相当于操作系统的rm命令。

包含头文件：

```c
#include <stdio.h>
```

函数声明：

```c
int remove(const char *pathname);
```

参数说明：

pathname 待删除的文件或目录名。

返回值：0-成功，-1-失败。