# Linux高性能服务器编程



## 4

### 4.3 使用tcpdump抓取传输数据包

```bash
 tcpdump -s 2000 -i eth0 -ntX '(src XXX.XXX.XXX.XXX)or(dst XXX.XXX.XXX.XXX)or(arp)'
 字节数  物理网卡名称 源地址XXX.XXX.XXX.XXX 目标地址XXX.XXX.XXX.XXX
```


### socket地址API

#### 5.1.1 主机字节序和网路字节序

```c++
#include<netinet/in.h>
unsigned long int htonl(unsigned long int hostlong);
unsigned short int htons(unsigned short int hostshort);
unsigned long int ntohl(unsigned long int netlong);
unsigned short int ntohl(unsigned short int netshort);
```

主机字节序与网络字节序之间的转换函数



#### 5.1.2 通用socket地址

```c
#include<bits/socket.h>
struct sockaddr{
    sa_family_t sa_family; //地址族
    char sa_data[14];      //地址值
}
```

上述通用地址结构体不够使用,地址值只有14个字节,但是多数协议不止这个数字了.

```c
#include<bits/socket.h>
struct sockaddr_storage{
    sa_family_t sa_family;
    unsigned long int __ss_align;
    char __ss_padding[128-sizeof(__ss_align)];
}
```

内存对齐



#### 5.1.3 专用socket地址

IPV4 

```C
struct sockaddr_in{
    sa_family_t sin_family; //地址族
    u_int16_t sin_port; //端口号
    struct in_addr sin_addr;//ipv4结构体
}

struct in_addr{
    u_int32_t s_addr //ipv4地址,用网络字节序来表示
}
```



#### 5.1.4 IP地址转换函数

```c
#include<arpa/inet.h>
in_addr_t inet_addr(const char* strpty);
int inet_aton(const char* cp, struct in_addr* inp);
char inet_ntoa(struct in_addr* inp);
```

网络字节序整数表示的地址与点分十进制表示的地址之间的转换.

```c
#include<arpa/inet.h>
int inet_pton(int af, char* src, void* dst);
const char* inet_ntop(int af, const void* src, char *dst, socklen_t cnt);
```





### 5.2 创建socket

```c
#include<sys/types.h>
#include<sys/socket.h>
int socket(int domain, int type, int protocol);
```

调用成功返回socket文件描述符,失败返回-1并设置errno.



### 5.3 命名socket

```c
#include<sys/types.h>
#include<sys/socket.h>
int bind(int sockfd, const struct sockaddr* my_addr, socklen_t addrlen);
```

将该socket描述符与地址绑定.

bind成功返回0，失败返回-1并设置errno

* errno
  * EACCE:被绑定地址是受保护地址，仅超级用户可以访问，如绑定端口号到（1~1023）
  * EADDRINUSS:被绑定地址正在使用中。比如一个处于TIME_WAIT状态下的是socket地址。



### 5.4 监听socket

```c
#include<sys/socket.h>
int listen(int sockfd,int backiog);

```

sockfd 指定监听的套接字。

backlog 提示内核监听队列的最大长度。

成功返回0，失败返回-1并设置errno。



### 5.5 接受socket

```c
#include<sys/socket.h>
int accept(int sockfd, struct sockaddr* addr, socklen_t *addrlen);
```

accept 成功时返回一个新的连接socket，该socket唯一的标识了被接受的这个连接，服务器通过该socket与被接受的对应的客户端通信。

失败返回-1并设置errno。

**accept只是从监听队列中取出连接，而不论连接处于 何种状态（如上面的ESTABLISHED状态和CLOSE_WAIT状态），更 不关心任何网络状况的变化。**



### 5.6 发起连接

```c
#include<sys/types.h>
#include<sys/socket.h>
int connect(int sockfd, struct sockaddr* serv_addr, socklen_t addrlen);
```



### 5.7 关闭连接

```c
#include<unistd.h>
int close(int fd);
```

close的 系统调用并非立即关闭一个socket，而是将fd 的引用计数减一，直至fd的引用计数为0时，才真正关闭该socket。



```c
#include<sys/socket.h>
int shutdown(int sockfd, int howto);
```

立刻以某种模式关闭该socket

* howto

  * SHUT_RD

    关闭读这一半，处于半关闭状态，不能对该socket进行读操作，且该接受缓冲区中数据全被抛弃

  * SHUT_WR

    关闭写这一半，处于半关闭状态，不能对该socket进行写操作，且该接受缓冲区中数据全被抛弃

  * SHUT_RDWR

    同时关闭读写

成功返回0。失败返回-1。



### 5.8 数据读写

#### 5.8.1 TCP数据读写

read和write同时适用于socket。

基于TCP数据流读写的系统调用

```c
#include<sys/types.h>
#include<sys/socket.h>
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

recv读取sockfd上的数据， buf和len参数分别指定读缓冲区的位置和大小。

send读取sockfd上的数据， buf和len参数分别指定读缓冲区的位置和大小。



#### 5.8.2 UDP数据读写

```c
#include<sys/types.h>
#include<sys/socket.h>
ssize_t recvfrom(int sockfd, void* buf, size_t len, int flags, struct sockaddr* src_addr, socklen_t *addr_len);
ssize_t sendto(int sockfd, const void* buff, size_t len, int flags, const struct sockaddr* dest_addr, socklen_t addrlen);
```

UDP中没有连接的概念，所以每次发送数据都需要目标的地址信息（dest_addr, addrlen)



#### 5.8.3 通用数据读写函数

```c
#include<sys/socket.h>
ssize_t recvmsg(int sockfd, struct msghdr* msg, int flags);
ssize_t sendmsg(int sockfd, struct msghdr* msg, int flags);
```





#### 5.9 带外标记

```c
#include<sys/socket.h>
int sockatmask(int sockfd);
```

通过该函数确认sockfd是否处于带外标记状态



#### 5.10 地址信息函数

```c
#include<sys/socket.h>
int getsockname(int sockfd, struct sockaddr* address, socklen_t* address_len);
int getpeername(int sockfd, struct sockaddr* address, socklen_t* address_len);
```

本端socket地址，以及远端socket地址的获取



#### 5.11 socket选项

```c
#include<sys/socket.h>
int getsockopt(int sockfd, int level, int option_name, void* option_value, socklen_t* restrict option_len);
int setsockopt(int sockfd, int level, int option_name, const void* option_value, socklen_t* option_len);
```



##### 5.11.1 SO_REUSEADDR选项

##### 5.11.2 SO_RCVBUF和SO_SNDBUF选项

##### 5.11.3 SO_RCVLOWAT和SO_SNDLOWAT选项

##### 5.11.4 SO_LINGER选项





'

### 6.4 sendfile 函数

```c
#include<sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t* offset, size_t count);
```

in_fd参数是待读出内容的文件描述符，out_fd参数是待写入内容的文件描述符。offset参数指定从读入文件流的哪个位置开始读，如果为空，则使用读入文件流默认的起始位置。count指定传输的字节数，成功返回传输的字节数，失败返回-1，设置errno

in_fd必须是一个支持类似mmap函数的文件描述符，必须指向真实的文件，不能是socket和管道



### 6.6 splice 函数

```c
#include<fcntl.h>
ssize_t splice(int fd_in, loff_t* off_in, int fd_out, loff_t* off_out, size_t len, unsigned int flags);
```

**splice函数用于在两个文件描述符之间移动数据，也是零拷贝操作。**

fd_in 和 fd_out 其中有一个必须是管道







### 6.8 fcntl 函数

对文件描述符进行操作。

```c
#include<fcntl.h>
int fcntl(int fd, int cmd,...);
```



#### 7.1.2 syslog 函数

```c
#include<syslog.h>
void syslog(int priority, const char* message, ...);



#include＜syslog.h＞
#define LOG_EMERG 0/*系统不可用*/
#define LOG_ALERT 1/*报警，需要立即采取动作*/
#define LOG_CRIT 2/*非常严重的情况*/
#define LOG_ERR 3/*错误*/
#define LOG_WARNING 4/*警告*/
#define LOG_NOTICE 5/*通知*/
#define LOG_INFO 6/*信息*/
#define LOG_DEBUG 7/*调试*/


void openlog(const char*ident,int logopt,int facility);

#logopt 对后续调用行为进行配置，取下值的按位或

#define LOG_PID 0x01/*在日志消息中包含程序PID*/
#define LOG_CONS 0x02/*如果消息不能记录到日志文件，则打印至终端*/
#define LOG_ODELAY 0x04/*延迟打开日志功能直到第一次调用syslog*/
#define LOG_NDELAY 0x08/*不延迟打开日志功能*/

#facility参数可用来修改syslog函数中的默认设施值。

```



设置日志掩码

```c
#include<syslog.h>
int setlogmask(int maskpri); //设置日志掩码 返回值为先前日志掩码

void closelog(); //关闭日志
```





#### 7.2.1 UID、EUID、GID和EGID



```bash
sudo  chmod +s XX #可以对可执行文件升级权限。
sudo  chmod u+s XX #即让执行文件的用户以该文件所属组的权限去执行；
sudo  chmod g+s XX #即让执行文件的用户以该文件所属组的权限去执行；

```



```c
#include＜sys/types.h＞
#include＜unistd.h＞
uid_t getuid();/*获取真实用户ID*/
uid_t geteuid();/*获取有效用户ID*/
gid_t getgid();/*获取真实组ID*/
gid_t getegid();/*获取有效组ID*/
int setuid(uid_t uid);/*设置真实用户ID*/
int seteuid(uid_t uid);/*设置有效用户ID*/
int setgid(gid_t gid);/*设置真实组ID*/
int setegid(gid_t gid);/*设置有效组ID*/
```



#### 7.3.1 进程组

```c
#include＜unistd.h＞
pid_t getpgid(pid_t pid);
```



Linux下每个进程都隶属于一个进程组，因此它们除了PID信息 外，还有进程组ID（PGID）。我们可以用如下函数来获取指定进程的 PGID



#### 7.3.2 会话组

```c
#include<unistd.h>
pid_t setsid(void);
pid_t getsid(pid_t pid);
```

该函数不能由进程组的首领进程调用，否则将产生一个错误。对 于非组首领的进程，调用该函数不仅创建新会话，而且有如下额外效 果： 

* 调用进程成为会话的首领，此时该进程是新会话的唯一成员。 

* 新建一个进程组，其PGID就是调用进程的PID，调用进程成为该 组的首领。 

* 调用进程将甩开终端（如果有的话）。





### 7.6 服务器程序后台化

```c
#include<unistd.h>
int daemon(int nochdir, int noclose);
```







## 8. 高性能服务器架构

将服务器解构为如下三个主要模块：

* I/O处理单元。

  I/O处理单元的四种I/O模型和两种高效事件处理模式。 

* 逻辑单元。本章将介绍逻辑单元的两种高效并发模式，以及高 效的逻辑处理方式——有限状态机。 

* 存储单元。本书不讨论存储单元，因为它只是服务器程序的可 选模块，而且其内容与网络编程本身无关。





### 9.3 epoll系列系统调用

```c
#include<sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
```

op参数指定操作类型：

* EPOLL_CTL_ADD 往事件表中注册fd上的事件
* EPOLL_CTL_MOD 修改fd上的注册事件
* EPOLL_CTL_DEL 删除fd上的注册事件

event参数指定事件，它是epoll_event结构指针类型x



```c
#include<sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```



#### 9.3.3 ET和LT模式

