# UNIX高级环境编程（APUE）

## 3.0  文件I/O

### 3. 1 介绍

* `unbuffered I/O` 意味着每次进行`read`或者`write`均设计到在内核中的系统调用。

* 多个进程之间在共享的资源池中进行操作时，原子性的操作这一概念十分重要。（本书从文件`I/O`和`open`函数的参数来研究）



### 3.2 文件描述符 （file descriptors）

* 在内核中，所有打开的文件都由文件描述符确定
* 文件描述符是一个非负的整数
* 它作为`read`和`write`的参数或者作为`open`和`creat`的返回值
* 按规定：文件描述符0为标准输入的进程，文件描述符1为标准输出的进程，文件描述符2为标准错误
* 文件描述符的范围从`0~OPEN_MAX-1`



### 3.3 `open`和`openat`函数

* open函数

```c++
#include<fcntl.h>
int open (const char *pathname, int oflag, .../* mode_t mode */);
//pathname  是要打开或创建文件的名字。
```

> **以下参数必须一个且只能指定一个**
>
> > O_RDONLY		 只读打开
> >
> > O_WRONLY		只写打开
> >
> > O_RDWR			 读写打开
> >
> > O_EXEC			   只执行打开
> >
> > O_SEARCH		 搜索打开
>
> **以下参数可选**
>
> > O_APPEND		 每次从结尾开始写。
> >
> > O_CLOEXEC		设置FD_CLOEXEC 文件描述符。
> >
> > O_CREAT			 创建文件如果不存在。用第三个选项来设置文件访问权限
> >
> > O_DIRECTORY    产生一个error如果路径未指向一个目录
> >
> > O_EXCL			   产生一个error如果O_CREAT 被指定了，且文件存在，用于测试文件是否存在。
> >
> > O_TRUNC		   如果文件存在且以只写或读写形式打开，长度截断为0。
> >
> > O_NOCTTY		 如果path指向一个终端设备，不会是该设备作为该进程的控制终端(?)
> >
> > O_NOFOLLOW  如果路径指向符号连接，返回error
> >
> > O_NONBLOCK   如果路径指向FIFO，一个块特殊文件、一个字符特殊文件，打开操作和后序I/O操作设置为非阻塞模式。
> >
> > O_SYNC			  每次write等待物理I/O完成，包括write操作引起的对文件属性的更新所需I/O
>
> **同步输入输出的一部分**
>
> > O_RSYNC			是每一个一文件描述符作为参数的read操作等待。直至任何对文件同一部分进行的write操作都完成
> >
> > O_DSYNC		    每次write等待物理I/O操作完成

* openat函数

  fd这个参数将open和openat这两个区别开来
```c++
#include<fcntl.h>
int openat (int fd,const char *pathname, int oflag, .../* mode_t mode */);
//pathname  是要打开或创建文件的名字。
```
1. 如果path参数为绝对路径，fd参数会被忽略，openat和open作用相当。
2. 如果path参数为相对路径，fd参数为计算该相对路径的在文件系统中的起始位置，fd参数通过打开计算相对路径的目录获得。
3. 如果path参数为相对路径，fd参数中有特殊值`AT_FDCWD`，pathname从当前工作目录开始计算，openat和open作用相当。



**为线程提供了一种使用相对路径名打开当前工作目录以外目录中的文件的方法**

**提供了一种避免检查到使用时间错误(TOCTTOU)的方法**

TOCTTOU:如果一个程序执行两个基于文件的函数调用，其中第二个调用依赖于第一个调用的结果，那么该程序就很容易受到攻击。

这一连串操作非原子性，所以当两个函数调用中文件被修改，就容易产生错误的结果。



### 3.4 create Function

* 由create函数创建一个新文件

```c++
#include<fcntl.h>
int creat (const char *path, /* mode_t mode */);
//成功返回fd 失败返回-1
```

等价于

```c++
open(path, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

对于短期文件可以使用
```c++
open(path, O_RDWR | O_CREAT | O_TRUNC, mode);
```



### 3.5 close Function

* 关闭文件

```c++
#include<unistd.h>
int close(int fd);
//成功0 失败-1
```

进程终止时，所有被打开的文件会由内核自动关闭



### 3.6 lseek Function

* 一个被打开的文件可以由该函数设置文件偏移量

每个打开的文件都有一个相关的当前文件偏移量，通常这个数字非负，用来度量从文件开始的偏移位置。读写操作从文件偏移量开始，默认偏移量为“0”，除非O_APPEND被设置。

```c++
#include<unistd.h>
int  lseek(int fd, off_t offset, int whence);
//成功返回新文件偏移量 失败返回-1
```

1. whence为SEEK_SET, 则返回文件开始加offset
2. whence为SEEK_CUR, 则返回当前偏移量加offset， offset可正可负
3. whence为SEEK_END, 则返回文件末尾加offset， offset可正可负



我们可以通过设置offset为“0”，来获取当前位置偏移量。

也可以用来确定当前文件是否可以设置偏移量。如果fd是一个管道、FIFO或网络套接字，则lseek返回-1，并且errno设置为ESPIPE

因为offset可以为负值，所以在比较返回值是应该测试其是否等于-1，而不是是否小于0 

lseek仅用来记录内核中档期啊那文件的偏移量，不会导致任何I/O发生。然后下次读写操作使用这个偏移量

偏移量可以大于文件大小，这样在write中会产生"hole"，这是被允许的，"hole"以多个"\0"的形式存在

hole 不占实际的硬盘空间，改变显示大小，但不改变实际磁盘块



### 3.7 read Function

用来打开文件
```c++
#include<unistd.h>
ssize_t read (int fd, void *buf, size_t nbytes);
//成功返回读取文件大小，如果达到文件结尾返回0，如果错误返回-1
```

1. 当读取一般文件时，如果在达到读取大小前已经读取到文件结尾，那么会返回当前读取的字节个数，在下次读取时，返回0。
2. fd为终端设备是，每次只读取一行。
3. 从网络中读取时，网络内的蝗虫可能导致返回的数量少去请求数量。
4. 当从管道或者FIFO，如果管道包含的字符少于请求字符，返回可能的字符。
5. when reading from a record-oriened device. Some record-oriented devices, such as magnetic tape, can return up to a single record at a time.
6.  

read操作开始与文件当前偏移。 在成功返回之前，偏移会随着实际读取的字符而增长。



### 3.8 read Function

写入一个打开的文件

```c++
#include<unistd.h>
ssize_t write (int fd,const void *buf, size_t nbytes);
//成功返回吸入文件大小，如果错误返回-1
```

返回值一般等与nbytes参数值，如果不等于则会发生错误。

write error 一般发生于磁盘空间被占满，或者超过给定进程的文件大小限制。



### 3.9 I/O Efficiency

  ### 3.10 File Sharing

![打开文件的内核数据结构](F:\_github\studying\jpg\打开文件的内核数据结构.jpg)

* 内核用三种数据结构表示打开的文件，其之间关系决定了文件共享方面的一个进程对另一个进程可能产生的影响

1. 进程表项

   每个进程在进程表中都有个记录项，每个记录项包含一张文件描述符表，每个文件描述符表关联两个数据：

   a）文件描述符标志位

   b） 一个指向文件表项的指针

2. 文件表项：

   a） 文件状态标志位，如read状态、write状态、append状态、sync（同步）状态、nonblocking状态

   b） 当前文件偏移

   c） 一个指向文件v-node表项的指针

3. v节点表

   文件在磁盘中的一些基础信息

发陈法蓉rrrrrrrrrrrrrrrrrrr

# 5.0  标准I/O库

# 7.0  进程环境 

# 8.0  进程控制

# 9.0  进程关系

# 10.0  信号

# 11.0  线程

# 12.0  线程控制

# 14.0  高级I/O

# 15.0  进程间的通信

# 17.0  高级进程间通信

