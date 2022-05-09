#  简介

## IO复用

- 传统方法：每个线程、进程只能管理一个客户端连接
- 线程池：每个线程也只能管理一个客户端连接，但是不用频繁的创建线程
- I/O复用：在单进程/单线程中同时监听多个连接（监听，读，写）

## I/O复用的三种模型

- `select`模型：只能管理1024个客户端连接
- `poll`模型：可以管理更多的客户端连接，但是连接越多，性能会呈现线性下架
- `epoll`模型：只要内容足够，管理的连接数没有上限，性能不会下降

## 正向代理&方向代理

- 正向代理：A->B->C->C->D， A->D。就是如果A能访问B，B能访问C，C能访问D，通过正向代理，A就能访问D
- 方向代理：A->B<-C， A->C。如果A能访问B，C能访问B，那么通过方向代理，A就能访问C
- 比如：QQ远程桌面，微信语音、视频连线都是方向代理

# Select 模型

## 事件

- 新客户端的连接请求 accept （可读）
- 客户端有报文到达 recv （可读）
- 客户端连接已经断开 （可读）
- 可以向客户端发送报文 send (可写)

select() 等待事件的发生

TCP有缓冲区，如果缓冲区已经填满，send函数会阻塞

如果发送端关闭了socket，缓冲区中的数据会继续发送给接收端

**函数原型**

```c++
/* According to POSIX.1-2001 */
 #include <sys/select.h>
 /* According to earlier standards */
 #include <sys/time.h>
 #include <sys/types.h>
 #include <unistd.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

**函数参数**

参数 含义

1. `nfds`  需要监视的最大的文件描述符值+1
2. `readfds`  需要检测的可读文件描述符的集合
3. `writefds`  需要检测的可写文件描述符的集合
4. `exceptfds`  需要检测的异常文件描述符的集合
5. `timeout`  超时时间

函数返回值

失败返回-1，获取到数据返回>0，超过时间返回0。

Select工作流程

- 1：用FD_ZERO宏来初始化我们感兴趣的`fd_set`

  也就是select函数的第二三四个参数。

- 2：用FD_SET宏来将套接字句柄分配给相应的`fd_set`。

  如果想要检查一个套接字是否有数据需要接收，可以用FD_SET宏把套接接字句柄加入可读性检查队列中

- 3：调用select函数。

  如果该套接字没有数据需要接收，select函数会把该套接字从可读性检查队列中删除掉，

- 4：用FD_ISSET对套接字句柄进行检查。

  如果我们所关注的那个套接字句柄仍然在开始分配的那个`fd_set`里，那么说明马上可以进行相应的IO操 作。比如一个分配给select第一个参数的套接字句柄在select返回后仍然在select第一个参数的`fd_set`里，那么说明当前数据已经来了， 马上可以读取成功而不会被阻塞。

| No   | 参数                             | 含义                                      |
| ---- | -------------------------------- | ----------------------------------------- |
| 1    | `FD_ZERO(fd_set *fdset)`         | 清空文件描述符集                          |
| 2    | `FD_SET(int fd,fd_set *fdset)`   | 设置监听的描述符(把监听的描述符设置为1)   |
| 3    | `FD_CLR(int fd,fd_set *fdset)`   | 清除监听的描述符(把监听的描述符设置为0)   |
| 4    | `FD_ISSET(int fd,fd_set *fdset)` | 判断描述符是否设置(判断描述符是否设置为1) |
| 5    | FD_SETSIZE                       | 256                                       |

结构体`fd_set` 描述符号集

```c++
typedef struct fd_set {
        u_int   fd_count;               /* how many are SET? */
        SOCKET  fd_array[FD_SETSIZE];   /* an array of SOCKETs */
} fd_set;
```

TCPselect 服务端程序

```c++
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>

// 初始化服务端的监听端口
int initserver(int port);

int main(int argc, char *argv[])
{
    if(argc != 2)
    {
        printf("usage: ./tcpselect port\n");
        return -1;
    }

    // 初始化服务端用于监听的socket
    int listensock = initserver(atoi(argv[1]));

    printf("listensock = %d\n", listensock);

    if(listensock < 0)
    {
        printf("初始化监听端口失败..\n");
        return -1;
    }

    fd_set readfds;         // 声明一个socket集合，包括socket和客户端连接上来的socket
    FD_ZERO(&readfds);      // 初始化读事件socket的集合
    FD_SET(listensock, &readfds);       // 把监听的socket加入到这个socket集合中

    int maxfd = listensock;             // 记录集合中socket的最大值

    while (true)
    {
        // 事件：1)新客户端的连接请求accept；2)客户端有报文到达recv，可以读；3)客户端连接已断开；
        //       4)可以向客户端发送报文send，可以写。
        // 可读事件  可写事件
        // select() 等待事件的发生(监视哪些socket发生了事件)。

        struct timeval timeout;
        timeout.tv_sec=10;          // 10s
        timeout.tv_usec=0;          // 0微秒

        fd_set tmpfds = readfds;
        int infds = select(maxfd+1, &tmpfds, NULL, NULL, &timeout);     // 这里把socket集合的副本传给select

        // 返回失败
        if(infds < 0)
        {
            perror("select() failed\n");
            break;
        }

        // 超时，在本程序中，select函数最后一个参数为空，不存在超时的情况，单已下代码还是留着‘
        if(infds == 0)
        {
            printf("select timeout\n");
            continue;
        }

        // 如果infds > 0.表示有事件发生的socket的数量
        for(int eventfd = 0; eventfd <= maxfd; eventfd++)
        {
            // 使用FD_ISSET来检测socket集合中是否有事件
            if(FD_ISSET(eventfd, &tmpfds) <= 0)
            {
                continue;       // 没有事件，继续循环
            }

            // 接下来就是表示找到了发生事件的socket

            // 如果发生的事件是listensock。表示有新的客户端连接上来
            // 监听socket只会用于监听客户端的连接请求，不会接收客户端的数据通信报文
            if(eventfd == listensock)
            {
                struct sockaddr_in client;
                socklen_t len = sizeof(client);
                int clientsock = accept(listensock, (struct sockaddr*)&client, &len);
                if(clientsock < 0)
                {
                    perror("accept() failed\n");
                    continue;
                }

                printf("accept client(socket=%d) ok\n", clientsock);

                // 然后把新客户端的socket加入可读socket的集合
                FD_SET(clientsock, &readfds);
                if(maxfd < clientsock) 
                {
                    maxfd = clientsock;     // 更新maxfd的值
                }
            }
            else
            {
                // 如果是客户端连接的socket有事件，表示有报文发送过来，或者连接已经断开
                char buffer[1024];      // 存放从客户端读取的数据
                memset(buffer, 0, sizeof(buffer));

                if(recv(eventfd, buffer, sizeof(buffer), 0) <= 0)
                {
                    // 客户端连接已经断开
                    printf("client(enventfd = %d) disconnected\n", eventfd);
                    close(eventfd);
                    // 然后用这个宏把已经关闭的socket从socket集合中删除
                    FD_CLR(eventfd, &readfds);
                    // 从集合中删除一个socket之后，要重新计算maxfd的值
                    if(maxfd == eventfd)
                    {
                        for(int i = maxfd; i > 0; i--)      // 从后往前找
                        {
                            if(FD_ISSET(i, &readfds))
                            {
                                maxfd = i;
                                break;
                            }
                        }
                    }
                }
                else
                {
                    // 客户端有报文发送过来
                    printf("recv(eventfd=%d):%s\n", eventfd, buffer);
                    // 这里把接收到的报文原封不动的发回去
                    fd_set tmpfds;
                    FD_ZERO(&tmpfds);
                    FD_SET(eventfd, &tmpfds);
                    if(select(eventfd+1, NULL, &tmpfds, NULL, NULL) < 0)
                    {
                        perror("select（） failed");
                    }
                    else
                    {
                        send(eventfd, buffer, strlen(buffer), 0);
                    }
                }
            }
        }
    }
    

    return 0;
}

// 初始化服务端的监听端口
int initserver(int port)
{
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    if(sock < 0)
    {
        perror("socket() failed");
        return -1;
    }

    int opt = 1;
    unsigned int len = sizeof(opt);

    struct sockaddr_in servadr;
    servadr.sin_family = AF_INET;
    servadr.sin_addr.s_addr = htonl(INADDR_ANY);
    servadr.sin_port = htons(port);

    if (bind(sock, (struct sockaddr *)&servadr, sizeof(servadr)) < 0)
    {
        perror("bind() faied");
        close(sock);
        return -1;
    }

    if(listen(sock, 5) != 0)
    {
        perror("listen() failed");
        close(sock);
        return -1;
    }
    
    return sock;
}
```

客户端测试程序

```c++
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char *argv[])
{
  if (argc != 3)
  {
    printf("usage:./client ip port\n"); return -1;
  }

  int sockfd;
  struct sockaddr_in servaddr;
  char buf[1024];
 
  if ((sockfd=socket(AF_INET,SOCK_STREAM,0))<0) { printf("socket() failed.\n"); return -1; }
 
  memset(&servaddr,0,sizeof(servaddr));
  servaddr.sin_family=AF_INET;
  servaddr.sin_port=htons(atoi(argv[2]));
  servaddr.sin_addr.s_addr=inet_addr(argv[1]);

  if (connect(sockfd, (struct sockaddr *)&servaddr,sizeof(servaddr)) != 0)
  {
    printf("connect(%s:%s) failed.\n",argv[1],argv[2]); close(sockfd);  return -1;
  }

  printf("connect ok.\n");

  for (int ii=0;ii<5000000000;ii++)
  {
    // 从命令行输入内容。
    memset(buf,0,sizeof(buf));
    printf("please input:"); scanf("%s",buf);
    // strcpy(buf,"1111111111111111111111111");

    if (send(sockfd,buf,strlen(buf),0) <=0)
    { 
      printf("write() failed.\n");  close(sockfd);  return -1;
    }
  
    memset(buf,0,sizeof(buf));
    if (recv(sockfd,buf,sizeof(buf),0) <=0) 
    { 
      printf("read() failed.\n");  close(sockfd);  return -1;
    }

    printf("recv:%s\n",buf);
  }
} 


```

编译上面两个程序，然后运行服务端，启动多几个客户端。测试通讯过程

![](.\img\QQ截图20220509155258.png)

# select模型的缺点

- 支持的连接数太小（1024），调整的意义不大
- select是内核函数，每次调用select，要把fdset从用户态拷贝到内核，调用select之后，把fdset从内核态拷贝到用户态中
- select返回后，需要遍历bitmap，效率比较低

