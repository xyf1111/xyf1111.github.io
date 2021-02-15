# Linux Network 02

<!--more-->
## Socket(套接字)编程
### 套接字概念
在TCP/IP协议中，“IP地址+TCP或UDP端口号”唯一标识网络通讯中的一个进程。“IP地址+端口号”就对应一个socket。欲建立连接的两个进程各自有一个socket来标识，那么这两个socket组成的socket pair就唯一标识一个连接。因此可以用Socket来描述网络连接的一对一关系
![socket通信原理](/images/socket.png "socket通信原理")
**在网络通信中，套接字一定是成对出现的**。一端的发送缓冲区对应对端的接收缓冲区。我们使用同一个文件描述符索发送缓冲区和接收缓冲区

