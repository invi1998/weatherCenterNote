# epoll网络模型

- epoll性能卓越，是高性能网络框架的基础（nginx，Redis，rpc，libevent）
- epoll，没有内存拷贝，没有轮询，没有遍历

## epoll的3个主要函数

- 创建句柄：`int epoll_create(int size);`.epoll句柄可以理解为socket的集合，就像select中的bitmap和poll中的数组一样。
- 注册事件：`int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);`，意思就是告诉epoll需要监视哪些socket的哪些事件
- 等待事件：`int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);`,epoll事件返回之后，后续的处理和select和poll也是类似的。

## epoll_event

```c++
struct epoll_event {
    uint32_t events;   // epoll events
    epoll_data_t data;   // user data variable
};
```

epoll_event是一个结构体，他的第二个成员是用户数据。如下所示，它是一个联合体。

## epoll_data

```c++
typedef union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

epoll_data_t 是一个共同体，epoll在添加事件的时候，把用户数据传给epoll，epoll返回的时候，会把这些数据原封不动的带回来。

## epoll控制函数

```c++
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event ); // 成功，返0，失败返-1
```

控制某个epoll监控的文件描述符上的事件：注册，修改，删除

**参数释义：**

- epfd：为epoll的句柄
- op:表示动作，用3个宏来表示
  - ··· EPOLL_CTL_ADD（注册新的 fd 到epfd）
  - ··· EPOLL_CTL_DEL（从 epfd 中删除一个 fd）
  - ··· EPOLL_CTL_MOD（修改已经注册的 fd 监听事件）
- fd：表示epoll监听的soket
- event：告诉内核需要监听的事件

```c++
typedef union epoll_data
{
    void* ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;  /* 保存触发事件的某个文件描述符相关的数据 */
 
struct epoll_event
{
    __uint32_t events;  /* epoll event */
    epoll_data_t data;  /* User data variable */
};
/* epoll_event.events:
  EPOLLIN  表示对应的文件描述符可以读
  EPOLLOUT 表示对应的文件描述符可以写
  EPOLLPRI 表示对应的文件描述符有紧急的数据可读
  EPOLLERR 表示对应的文件描述符发生错误
  EPOLLHUP 表示对应的文件描述符被挂断
  EPOLLET  设置ET模式
*/
```

## epoll消息读取

```c++
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

等待所监控文件描述符上有事件的产生

**参数释义**：

- epfd：epoll句柄
- events：用来从内核得到事件的集合。已发生事件的数组，也就是说，如果发生了事件，epoll会把已经发生事件的清单放在这个数组里面，这个数组的内存空间需要我们在程序中预先分配好
- maxevent：用于告诉内核这个event有多大，这个maxevent不能大于创建句柄时的size
- timeout：超时时间，单位微秒
  - ··· -1：阻塞
  - ··· 0：立即返回
  - ···>0：指定微秒

成功返回有多少个文件描述符准备就绪，时间到返回0，出错返回-1.

```c++
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/fcntl.h>
#include <sys/epoll.h>

// 初始化服务端的监听端口
int initserver(int port);

int main(int argc, char *argv[])
{
    if(argc != 2)
    {
        printf("usage: ./tcpepoll port\n");
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

    // 创建epoll句柄
    int epollfd = epoll_create(1);

    // 为监听socket准备可读事件
    epoll_event ev;             // 声明事件的数据结构
    ev.events = EPOLLIN;        // 对结构体的events成员赋值，表示关心读事件
    ev.data.fd = listensock;    // 对结构体的用户数据成员data的fd成员赋值，指定事件的自定义数据，会随着epoll_wait()返回的事件一并返回

    // 事件准备好后，把它加入到epoll的句柄中
    // 第一个参数：epoll句柄
    // 第二个参数：ctl的动作，这里是添加事件，用EPOLL_CTL_ADD
    // 第三个参数：epoll监听的socket，这里是给监听socket添加epoll，所以填listensock
    // 第四个参数：epoll事件的地址
    epoll_ctl(epollfd, EPOLL_CTL_ADD, listensock, &ev);

    struct epoll_event evs[10];     // 声明一个用于存放epoll返回事件的数组
   
    while (true)
    {
        // 在循环中调用epoll_wait监视sock事件发生
        int infds = epoll_wait(epollfd, evs, 10, -1);

        // 返回失败
        if(infds < 0)
        {
            perror("epoll() failed\n");
            break;
        }

        // 超时，在本程序中，epoll_wait函数最后一个参数-1，不存在超时的情况，但已下代码还是留着‘
        if(infds == 0)
        {
            printf("epoll timeout\n");
            continue;
        }

        // 如果infds > 0.表示有事件发生的socket的数量
        // 遍历epoll返回的已经发生事件的数组evs
        for(int i = 0; i < infds; i++)
        {
            // 注意，先前我们已经将事件的socket通过用户数据成员传递进epoll，那么在这里事件发生的时候，我们就可以通过事件将这个socket带回来
            // 也就是说，我们可以通过这个事件知道是哪个socket发生了事件
            printf("events=%d, data.fd=%d,\n", evs[i].events, evs[i].data.fd);

            // 如果发生的事件是listensock。表示有新的客户端连接上来
            // 监听socket只会用于监听客户端的连接请求，不会接收客户端的数据通信报文
            if(evs[i].data.fd == listensock)
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

                // 为客户端准备可读事件，并添加到epoll中
                ev.data.fd = clientsock;
                ev.events = EPOLLIN;        // 可读事件
                epoll_ctl(epollfd, EPOLL_CTL_ADD, clientsock, &ev);

            }
            else
            {
                // 如果是客户端连接的socket有事件，表示有报文发送过来，或者连接已经断开
                char buffer[1024];      // 存放从客户端读取的数据
                memset(buffer, 0, sizeof(buffer));

                if(recv(evs[i].data.fd, buffer, sizeof(buffer), 0) <= 0)
                {
                    // 客户端连接已经断开
                    printf("client(enventfd = %d) disconnected\n", evs[i].data.fd);
                    close(evs[i].data.fd);
                    // 然后把已经关闭的socket从epoll中删除
                    // 注意，这里已经关闭的socket不需要调用epoll_ctl函数把这个socket从epoll中删除，系统会自动处理已经关闭的socket
                }
                else
                {
                    // 客户端有报文发送过来
                    printf("recv(eventfd=%d):%s\n", evs[i].data.fd, buffer);
                    // 这里把接收到的报文原封不动的发回去
                    send(evs[i].data.fd, buffer, strlen(buffer), 0);
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

编译运行服务端，然后启动多几个客户端，进行通信测试，如下

![](.\img\QQ截图20220510122437.png)
