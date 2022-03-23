# Linux exec 系列函数：execl execv等

## exec函数族

exec函数族提供了一个在进程中启动另一个程序执行的方法。
实际上他们的功能都是差不多的, 因为要用于接受不同的参数所以要用不同的名字区分它们, 毕竟c语言没有函数重载的功能

### exec 重要说明！！！

exec函数会取代执行它的进程, 也就是说, 一旦exec函数执行成功, 它就不会返回了, 进程结束.
但是如果exec函数执行失败, 它会返回失败的信息, 而且进程继续执行后面的代码!
通常exec会放在fork() 函数的子进程部分, 来替代子进程执行啦, 执行成功后子程序就会消失, 但是执行失败的话, 必须用exit()函数来让子进程退出!

### 使用exec函数族主要有两种情况：

(1)当进程认为自己不能再为系统和用户做出任何贡献时，就可以调用exec函数族中的任意一个函数让自己重生。
(2)如果一个进程想执行另一个程序，那么它就可以调用fork函数新建一个进程，然后调用exec函数族中的任意一个函数，
这样看起来就像通过执行应用程序而产生了一个新进程(这种情况非常普遍)。

## exec函数族共有6种不同形式的函数

### exec 函数族6个函数可以划分为两组：

这两组函数的不同在于exec后的第一个字符，

`execl、execle和execlp。`

第一组是l，在此称，为execl系列；这里的l是list(列表)的意思，表示execl系列函数需要将每个命令行参数作为函数的参数进行传递；

`execv、execve和execvp`

第二组是v，在此称为execv系列。而v是vector(矢量)的意思，表示execv系列函数将所有函数包装到一个矢量数组中传递即可。

### exec 函数的原型如下：

```c
int execl(const char * path，const char * arg，…)；
int execle(const char * path，const char * arg，char * const envp[])；
int execlp(const char * file，const char * arg，…)；
int execv(const char * path，char * const argv[])；
int execve(const char * path，char * const argv[]，char * const envp[])；
int execvp(const char * file，char * const argv[])；
```

## exec 命名规律

`exec[l or v][p][e]`

### exec函数里的参数可以分成3个部分

执行文件部分, 命令参数部分, 环境变量部分.

例如我要执行1个命令 `ls -l /home/gateman`
执行文件部分就是 `/usr/bin/ls`
命令参数部分就是 “ls”,"-l","/home/gateman",NULL 见到是以ls开头 每1个空格都必须分开成2个部分, 而且以NULL结尾的啊.
环境变量部分, 这是1个数组,最后的元素必须是NULL 例如 char * env[] = {“PATH=/home/gateman”, “USER=lei”, “STATUS=testing”, NULL};

### 命名规则:

`e 结尾`, 参数必须带环境变量部分, 环境变零部分参数会成为执行exec函数期间的环境变量, 比较少用
`l 结尾,` 命令参数部分必须以"," 相隔, 最后1个命令参数必须是NULL
例如： `execl("/bin/ls", “ls”, “-l”, NULL);`
`v 结尾`, 命令参数部分必须是1个以NULL结尾的字符串指针数组的头部指针.
例如char * pstr就是1个字符串的指针, char * pstr[] 就是数组了, 分别指向各个字符串.
`execv(szcmd, ps_argv);`
`p 结尾`, 执行文件部分可以不带路径, exec函数会在$PATH中找

### 参数说明：

`path`：要执行的程序路径。可以是绝对路径或者是相对路径。在execv、execve、execl和execle这4个函数中，使用带路径名的文件名作为参数。
`file`：要执行的程序名称。如果该参数中包含“/”字符，则视为路径名直接执行；否则视为单独的文件名，系统将根据PATH环境变量指定的路径顺序搜索指定的文件。
`argv`：命令行参数的矢量数组。
`envp`：带有该参数的exec函数可以在调用时指定一个环境变量数组。其他不带该参数的exec函数则使用调用进程的环境变量。
`arg`：程序的第0个参数，即程序名自身。相当于`argv[O]`。
`…`：命令行参数列表。调用相应程序时有多少命令行参数，就需要有多少个输入参数项。注意：在使用此类函数时，在所有命令行参数的最后应该增加一个空的参数项(NULL)，表明命令行参数结束。

例如： `execl("/bin/ls", “ls”, “-l”, NULL);`

### exec 返回值

-1表明调用exec失败，无返回表明调用成功。

## execl execv 代码示例

### execl 代码示例

#### execl 执行成功不返回/命令执行失败返回-1

原因：
在进程的创建上Unix采用了一个独特的方法，它将进程创建与加载一个新进程映象分离。
execl将当前进程替换掉，所有最后那条打印语句不会输出

```c
#include <stdio.h>
int main(int argc, char* argv[]){
    int a = execl("/bin/ls", "ls", "-l", NULL);
    printf("%d\n", a);

    printf("exiting...\n");
    return 0;
}
```

结果

```shell
root@glusterfs home]# gcc test.c 
[root@glusterfs home]# ll
total 16
-rwxr-xr-x 1 root root 8584 Dec  2 17:10 a.out
-rw-r--r-- 1 root root  175 Dec  2 17:10 test.c
[root@glusterfs home]# ./a.out 
total 16
-rwxr-xr-x 1 root root 8584 Dec  2 17:10 a.out
-rw-r--r-- 1 root root  175 Dec  2 17:10 test.c
[root@glusterfs home]# 
```

4.1.2 execl + fork

```c++
#include <stdio.h>
#include <errno.h>
#include <unistd.h>

int main(int argc, char* argv[]){
	int childpid;
	pid_t pid;

	pid = fork();
	if(pid > 0)
	{
		printf("parent execl done\n");
		wait(&childpid);
	}else if(0 == pid)
	{
		execl("/bin/ls", "ls", "-l", NULL);
		printf("pid execl done\n");
	}else{
		printf("error pid < 0 \n");
	}
	printf("exiting...\n");
	return 0;

}
```


结果：

```shell
[root@glusterfs home]# gcc test.c 
[root@glusterfs home]# ./a.out 
parent execl done
total 52
-rwxr-xr-x 1 root root 8632 Dec  3 14:14 a.out
-rw-r--r-- 1 root root  444 Dec  3 14:14 test.c
exiting...
[root@glusterfs home]#
```



### execv 代码示例

```c++
#include <stdio.h>
#include <errno.h>
#include <unistd.h>

int main(int argc, char* argv[]){
	int childpid;
	pid_t pid;

	char szcmd[] = "/bin/ls";
	char *ps_argv[3];
	ps_argv[0] = "ls";
	ps_argv[1] = "-l";
	ps_argv[2] = NULL;
	
	pid = fork();
	if(pid > 0)
	{
		printf("parent execv done\n");
		wait(&childpid);
	}else if(0 == pid)
	{
		execv(szcmd, ps_argv);
		printf("pid execv done\n");
	}else{
		printf("error pid < 0 \n");
	}
	printf("exiting...\n");
	return 0;

}
```

结果：

```shell
[root@glusterfs home]# gcc execv.c 
[root@glusterfs home]# ./a.out 
parent execv done
total 48
-rwxr-xr-x 1 root root 8632 Dec  3 14:19 a.out
-rw-r--r-- 1 root root  550 Dec  3 14:19 execv.c
-rwxr-xr-x 1 root root 8648 Dec  2 17:39 sys
-rw-r--r-- 1 root root  192 Dec  2 17:39 sys.c
-rwxr-xr-x 1 root root 8632 Dec  3 11:49 test
-rw-r--r-- 1 root root  444 Dec  3 14:14 test.c
exiting...
[root@glusterfs home]#
```



### execv execle execlp 等不再阐述

