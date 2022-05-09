# poll模型

poll模型和select本质上没有任何区别，弃用了bitmap，采用数组表示法

## poll 函数

```c++
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

（1）struct pollfd *fds ：结构体数组
（2）nfds_t nfds ：数组里面有效的元素的个数，相当于select 里面的最大有效文件描述符个数，或者说需要监视的描述符的个数。
（3）int timeout ：超时机制（单位毫秒）

文件描述符数组

（1）int fd ：文件描述符

（2）short events ：让内核去监视的事件，就像是 select 中让内核去监视读事件，写事件，异常事件

（3）short revents ：返回的事件。调用 poll 函数，当监视的文件描述符有事件发生，内核修改的是 revents，而不修改 events 和 fd，这样做events 和 fd 就可以重用；poll 返回的是 revents 。也就是在调用poll的时候，把需要监听的socket和关心的事件传进去（这里就比select好，select在事件发生的时候，是直接修改bitmap，导致bitmap不可重用，所以我们在使用select的时候，才会需要给select传递一个socket集合的拷贝）

```c++
struct pollfd
{
 int fd; 
 short events;
 short revents;
};

```

系统对打开文件描述符的数量是有限制的，使用 `ulimit -a`命令可以查看，可以看到，缺省情况下是1024

![](.\img\QQ截图20220509183939.png)

**事件 events**

- POLLIN 有数据可读
- POLLRDNORM 有普通数据可读
- POLLRDBAND 有优先数据可读
- POLLPRI 有紧急数据可读
- POLLOUT 数据可写
- POLLWRNORM 普通数据可写
- POLLWRBAND 优先数据可写
- POLLMSGSIGPOLL 消息可用

**返回事件 revent**

- POLLERR 指定描述符发生错误
- POLLHUP 指定文件描述符挂起事件
- POLLNVAL 指定描述符非法

从下面的poll服务端代码也可以看出来，和select真的没有什么本质区别

```c++
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>
#include <poll.h>

#define MAXNFDS 1024

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

    // 声明poll的结构体数组
    struct pollfd fds[MAXNFDS];     // fds用于存放需要监视的socket

    for(int i =0; i < MAXNFDS; i++)
    {
        fds[i].fd = -1;     // 初始化数组，把全部的fd设置为-1
    }

    // 把监听的socket和读事件添加到数组中
    fds[listensock].fd = listensock;
    fds[listensock].events = POLLIN;
    
    int maxfd = listensock;             // 记录集合中socket的最大值

    while (true)
    {
        // poll第3个参数，如果希望没有超时机制，就填-1,如果希望超时就填毫秒数
        int infds = poll(fds, maxfd+1, 5000);     // 这里把socket集合的副本传给select

        // 返回失败
        if(infds < 0)
        {
            perror("poll() failed\n");
            break;
        }

        // 超时，在本程序中，poll函数最后一个参数为-1，不存在超时的情况，单已下代码还是留着‘
        if(infds == 0)
        {
            printf("poll timeout\n");
            continue;
        }

        // 如果infds > 0.表示有事件发生的socket的数量
        for(int eventfd = 0; eventfd <= maxfd; eventfd++)
        {
            // 遍历查看pollsocket数组中的每个socket的fd查看是否有事件
            if(fds[eventfd].fd < 0)
            {
                continue;  // 如果fd为负，忽略他
            }

            if((fds[eventfd].revents&POLLIN) == 0)
            {
                continue;   // 如果没有事件，continue
            }

            // 接下来就是表示找到了发生事件的socket
            // 注意：先把revent清空
            fds[eventfd].revents = 0;

            // 如果发生的事件是listensock。表示有新的客户端连接上来
            // 监听socket只会用于监听客户端的连接请求，不会接收客户端的数据通信报文
            if(eventfd==listensock)
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
                // 注意：这里数组采用的是 3放在索引3的位置，4放在索引4的位置，n放在索引n的位置
                fds[clientsock].fd = clientsock;
                fds[clientsock].events = POLLIN;
                fds[clientsock].revents = 0;
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
                    // 然后把已经关闭的socket从socket集合中删除
                    fds[eventfd].fd = -1;
                    // 从集合中删除一个socket之后，要重新计算maxfd的值
                    if(maxfd == eventfd)
                    {
                        for(int i = maxfd; i > 0; i--)      // 从后往前找
                        {
                            if(fds[i].fd != -1)
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
                    send(eventfd, buffer, strlen(buffer), 0);
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

编译运行，启动几个客户端程序

![](.\img\QQ截图20220509192306.png)
