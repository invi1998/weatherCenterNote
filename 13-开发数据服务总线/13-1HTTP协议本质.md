# HTTP协议本质

## http协议的基础知识

超文本传输协议（英文：HyperText Transfer Protocol，缩写：HTTP）是一种用于[分布式](https://so.csdn.net/so/search?q=分布式&spm=1001.2101.3001.7020)、协作式和超媒体信息系统的应用层协议。HTTP是万维网的数据通信的基础。

HTTP是一个客户端终端（用户）和服务器端（网站）请求和应答的标准（TCP）。通过使用网页浏览器、网络爬虫或者其它的工具，客户端发起一个HTTP请求到服务器上指定端口（默认端口为80）。

HTTP是一个客户端终端（用户）和服务器端（网站）请求和应答的标准（TCP）。通过使用网页浏览器、网络爬虫或者其它的工具，客户端发起一个HTTP请求到服务器上指定端口（默认端口为80）。

## HTTP工作原理

————————————————————————————————————————————————————
HTTP协议定义Web客户端如何从Web服务器请求Web页面，以及服务器如何把Web页面传送给客户端。HTTP协议采用了请求/响应模型。客户端向服务器发送一个请求报文，请求报文包含请求的方法、URL、协议版本、请求头部和请求数据。服务器以一个状态行作为响应，响应的内容包括协议的版本、成功或者错误代码、服务器信息、响应头部和响应数据。

以下是 HTTP 请求/响应的步骤：

1. 客户端连接到Web服务器
   一个HTTP客户端，通常是浏览器，与Web服务器的HTTP端口（默认为80）建立一个TCP套接字连接。例如，http://www.luffycity.com。
2. 发送HTTP请求
   通过TCP套接字，客户端向Web服务器发送一个文本的请求报文，一个请求报文由请求行、请求头部、空行和请求数据4部分组成。
3. 服务器接受请求并返回HTTP响应
   Web服务器解析请求，定位请求资源。服务器将资源复本写到TCP套接字，由客户端读取。一个响应由状态行、响应头部、空行和响应数据4部分组成。
4. 释放连接TCP连接
   若connection 模式为close，则服务器主动关闭TCP连接，客户端被动关闭连接，释放TCP连接;若connection 模式为keepalive，则该连接会保持一段时间，在该时间内可以继续接收请求;
5. 客户端浏览器解析HTML内容
   客户端浏览器首先解析状态行，查看表明请求是否成功的状态代码。然后解析每一个响应头，响应头告知以下为若干字节的HTML文档和文档的字符集。客户端浏览器读取响应数据HTML，根据HTML的语法对其进行格式化，并在浏览器窗口中显示。

例如：在浏览器地址栏键入URL，按下回车之后会经历以下流程：

```
1. 浏览器向 DNS 服务器请求解析该 URL 中的域名所对应的 IP 地址;`
`2. 解析出 IP 地址后，根据该 IP 地址和默认端口 80，和服务器建立TCP连接;`
`3. 浏览器发出读取文件(URL 中域名后面部分对应的文件)的HTTP 请求，该请求报文作为 TCP 三次握手的第三个报文的数据发送给服务器;`
`4. 服务器对浏览器请求作出响应，并把对应的 html 文本发送给浏览器;`
`5. 释放 TCP连接;`
`6. 浏览器将该 html 文本并显示内容;
```

![](.\img\QQ截图20220503114734.png)

## http请求和响应报文的格式

![img](.\img\20210421220329611.png)

```c++
/*
 * 程序名：demo26.cpp，此程序用于演示采用TcpServer类实现http通讯的服务端。
 * author：invi
*/
#include "../_public.h"
 
int main(int argc,char *argv[])
{
  if (argc!=2)
  {
    printf("Using:./demo26 port\nExample:./demo26 5005\n\n"); return -1;
  }

  CTcpServer TcpServer;

  // 服务端初始化。
  if (TcpServer.InitServer(atoi(argv[1]))==false)
  {
    printf("TcpServer.InitServer(%s) failed.\n",argv[1]); return -1;
  }

  // 等待客户端的连接请求。
  if (TcpServer.Accept()==false)
  {
    printf("TcpServer.Accept() failed.\n"); return -1;
  }

  printf("客户端（%s）已连接。\n",TcpServer.GetIP());

  char buffer[102400];
  memset(buffer, 0, sizeof(buffer));

  // 接收http客户端发送过来的报文
  recv(TcpServer.m_connfd, buffer, 1000, 0);

  printf("%s\n", buffer);
}

```

上面这个程序就是简单的打印http接收到的报文内容，编译运行，在浏览器输入ip加端口和url，服务端的响应打印结果如下

```http
[invi@192 socket]$ ./demo26 8080
客户端（192.168.31.186）已连接。
GET /lesson/546.html HTTP/1.1
Host: 192.168.31.133:8080
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/100.0.4896.127 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
```

![img](.\img\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2Fsb2trYQ==,size_16,color_FFFFFF,t_70)

如下程序用于模拟向一个服务端发起htpp请求，并将响应报文输出到html文件中。

```c++
/*
 * 程序名：demo27.cpp，此程序用于演示采用TcpClient类演示http通讯的客户端。
 * author：invi
*/
#include "../_public.h"
 
int main(int argc,char *argv[])
{
	if (argc!=3)
	{
		printf("Using:./demo27 ip port\nExample:./demo27 www.baidu.com 80\n\n"); return -1;
	}

	CTcpClient TcpClient;

	// 向服务端发起连接请求。
	if (TcpClient.ConnectToServer(argv[1],atoi(argv[2]))==false)
	{
		printf("TcpClient.ConnectToServer(%s,%s) failed.\n",argv[1],argv[2]); return -1;
	}

	char buffer[102400];
	
	// 生成http请求报文(注意，请求报文每一块内容上都需要加上回车换行\r\n)
	// GET /lesson/546.html HTTP/1.1
	// Host: 192.168.31.133:8080
	sprintf(buffer, "GET / HTTP/1.1\r\n"
			"Host: %s:%s\r\n"
			"\r\n", argv[1], argv[2]
	);

	// 用原生的send函数把http报文发送给服务端
	send(TcpClient.m_connfd, buffer, strlen(buffer), 0);

	// 接收服务端返回的网页内容
	while (true)
	{
		/* code */
		memset(buffer, 0, sizeof(buffer));
		if(recv(TcpClient.m_connfd, buffer, 102400, 0) <= 0)
		{
			return -1;
		};
		printf("%s", buffer);
	}
	
}

```

编译运行，结果如下(这里我们访问百度的站点)

![](.\img\QQ截图20220503130244.png)



## http协议的客户端工具和字符集的转换

