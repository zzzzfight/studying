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



**注意点**

1. 对同一个文件多次用open 函数，会得到不同的文件描述符，但是共享一个文件表项，所以共享偏移值



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



### 3.8 write Function

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

![打开文件的内核数据结构](https://github.com/zzzzfight/studying/blob/75d506b3cb59829a152fe9e37c227555f4b4bd9a/jpg/%E6%89%93%E5%BC%80%E6%96%87%E4%BB%B6%E7%9A%84%E5%86%85%E6%A0%B8%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.jpg)

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
   
   a）文件所有者
   
   b）文件大小
   
   c）指针指向这些文件的数据块在磁盘中的实际位置
   
   d）等等





### 3.11 Atomic Operations

* 原子操作：一系列操作要么全部成功，要么全部失败

多个线程在进程见共享文件描述符表，当同时打开文件时，每个文件指针指向各自的文件目录项，文件目录项保存了当前文件偏移以及文件状态标志，所以当多个线程同时读写文件，可能会导致错误。

所以要保证操作的原子性



### 3.12 dup and dup2 Function ！

```c++
#include<unistd.h>
int dup(int fd);
int dup2(int fd, int fd2);
//返回新的文件描述符，错误返回-1
```

* dup函数返回最低可用文件描述符，且与fd参数共享文件表项，

 

### 3.13 sync, fsync, and fdatasync Functions

### 3.14 fcntl Function

fcntl 函数可以改变已打开文件的文件属性

```c++
#include<fcntl.h>
int fcntl(int df, int cmd, .../*int arg */);
//错误返回-1 成功返回与cmd相关的值
```

* 复制一个已存在的文件描述符（cmd=F——DUPFD or F_DUPFD_CLOEXEC）
* 获取/设置文件描述符标志（cmd=F_GETFD or F_SETFD）
* 获取/设置文件状态标志（cmd=F_GETFL or F_SETFL）
* 获取/设置异步I/O所有权（cmd=F_GETOWN or F_SETOWN）
* 获取/设置记录锁（cmd=F_GETLK,F_SETLKW）

F_



## 5.0  标准I/O库

### 5.2 Standard I/O Library

* 流的定向

  对于ASCII字符集，一个字符用一个字节表示。对于国际字符集，一个字符可用多个字节表示。标准I/O文件流可用单机直接或多字节表示。**流的定向决定所读写的字符是单字节还是多字节的**。

  通过freopen 函数来清除流的定向。

  通过fwide 函数设置流的定向。

* fwide 函数

  ```c++
  #include<stdio.h>
  #include<wcher.h>
  int fwide(FILE *fp, int mode);
  //宽定向返回正值，字节定向返回负值，为定向返回0
  ```

  * 如果mode参数值为负，fwide将试图指定流字节定向
  * 如果mode参数值为正，fwide将试图指定流字节定向

* 指向FILE对象的指针

  打开一个流时。fopen返回一个指向FILE对象的指针。该指针是一个结构体，包含了管理该流的所需要的信息：

  * 用于实际I/O的文件描述符； 
  * 指向用于该流缓冲区的指针；
  * 当前在缓冲区中的字符数；
  * 出错标志；
  * 等等。

  通过文件



### 5.3 Standard Input， Standard Output， and Standard Error

这些流已被预定义且由可以由进程自动获取



### 5.4 Buffering

由标准I/O库提供的缓冲区的目标是使得read 函数和 write 函数的调用尽可能的少（read，write 是系统调用，需要从用户态切换到内核态，开销比较大）

1. 全缓冲。在这种情况下，**当标准I/O缓冲已满时，所使用的缓冲区通常为在流上执行I/O时，有调用malloc的一个标准I/O函数获得的。**

   术语flush 描述了标准I/O缓冲区的写入。缓冲区有标准I/O例程自动刷新，或者可以调用fflush 函数来刷新流。flush在unix环境中两个用途，一是写出缓冲区的内容，二是丢弃存在缓冲区中的数据。

2. 行缓冲。

   在一个流被指向了一个终端设备时，通常使用行缓冲。

3. 无缓冲。标准I/O库不缓冲字符。

* 标准输入和标准输入通常是全缓冲的，仅除了打开一个交互类的流

* 标准错误从不全缓冲

  默认情况下：

  * 标准错误非缓冲
  * 当流打开一个终端设备时，流是行缓冲的；否则就是全缓冲的。
    * 标准错误无缓冲；打开终端设备的标准输入输出流行缓冲；其他标准输入输出流全缓冲。

可以通过setbuf 函数或者setvbuf 函数设置缓冲

```c++
#include<stdio.h>
void setbuf(FILE *restrict fp, char *restrict buf);
int  setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);
//成功返回0，失败返回非零数
```

**这些操作只能在一个流被打开后且任何其他操作执行前进行。**

* setbuf

  打开或者关闭缓冲。启用缓冲，buf必须指向长度为BUFSIZ的缓冲区，这是<stdio.h>中定义的常量。通常，流被完全缓冲，但**是如果流与终端设备关联，一些系统可能会设置行缓冲。为了禁用缓冲，通过使buf设置为NULL。**

* setvbuf

  通过setvbuf 可以将buffering设置成我们要用的，

  * _IOFBF fully buffered
  * _IOLBF line buffered
  * _IONBF unbufered

  如果使用的是无缓冲流，那么buf和大小参数会被忽视。如果设置了全缓冲或者行缓冲，则可以设置缓冲及其大小。如果流是被设置需要缓冲的但是buf 设置成了NULL，那标准程序库会给流自动分配独有的缓冲区及合适的大小。

  而所谓合适的大小，一般指一个特殊的常量值BUFSIZ

  **如果在函数分配标准I/O缓冲区作为自动变量，则必须从函数返回之前关闭流**

  通过以下函数强制使刷新流

  ```c++
  #include<stdio.h>
  int fflush(FILE *fp);
  //成功返回0，否则EOF返回error
  ```

  fflush 函数使所有流中未写入的数据传至内核。

  特例：fp为NULL时，fflush 使所有输出流被刷新

  

### 5.5 Opening a Stream

三个函数打开标准I/O流

```c++
#include<stdio.h>
FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type, FILE *restrict fp);
FILE *fdopen(int fd, const char *type);
```

* fopen

  * 打开一个指定的文件

* freopen

  * 在某个流上打开一个指定文件，若流已经打开，则关闭流。若流已经定向，则清除定向
  * 将流重新联系到某个文件中，同时会清除流的定向，通常用于预定义的流的重新关联（stdin，stdout，stderr）
  
  ```c++
    #include<stdio.h>
    #include<stdlib.h>
    #define PATH1 "1.txt"
    #define PATH2 "2.txt"
    
    FILE *output1,*output2;
    char buffer[25]={'x','y','z'};
    int main(void){
    /*
        if(freopen(PATH,"a",stdout)==NULL)
            printf("error\n");
        printf("heloworld");
        fclose(stdout);     
        if(freopen(PATH,"r",stdin))
        return 0;
    */
        output1 = fopen(PATH1,"a");
        if(output1==NULL){
            printf("error");
            exit(1);
        }
        freopen(PATH2,"a",output1);
        fwrite(buffer,sizeof(char),sizeof(buffer),output1);
        fclose(output1);
        return 0; 
    }
  
  ```
  
* fdopen

  * 获取一个已存在的文件描述符（这个文件描述符通过open、dup、dup2、fcnt1、pipe、socket、socketpair或者accept函数得到此文件描述符）
  * 此函数常用于由创建管道和网络通信通道函数返回的描述符。这类特殊的文件没办法由标准I/O函数fopen 打开
  * 必须通过调用设备专用函数获取文件描述符，然后用fdopen是一个标准I/O流和该描述符相关联。

| type             | Description                             | open(2)Flages |
| ---------------- | --------------------------------------- | ------------- |
| r or rb          | 打开一个文件                            |               |
| w or wb          | 缩短文件长度至0 或者 创建一个文件来写入 |               |
| a or ab          | 扩展；打开一个文件是                    |               |
| r+ or r+b or rb+ |                                         |               |
| w+ or w+b or wb+ |                                         |               |
| a+ or a+b or ab+ |                                         |               |



## 7.0  进程环境 
### 7.2 main function

```c++
//main函数的函数原型
int main(int argc, char *argv[]);
//argc 是命令行参数的个数  argc是一个char指针类型数组
```

当内核执行一个C程序时，会在调用main函数之前调用一个特殊的例程。可执行程序文件只当该例程为程序的起始地址。编译器调用连接器时会设置这个。



### 7.3 Process Termination

一个进程可以由八种方式中断：

* 经常遇到的五种终止发生方式：

	* 由main函数返回
	* 调用 exit
	* 调用 _exit 或者 _Exit
	* 从它的开始例程返回最后一个线程
	* 在最后一个存活的线程中调用pthread_exit

* 非一般情况的三种终止发生方式

  * 调用abort（10.17）

  * 接受一个信号（11.5）

  * 最后一个存活线程对取消请求的响应（11.5 12.7）

    

**Exit Function**


```c
#include<stdlib.h>
void exit(int status);
void _Exit(int status);
#include<unistd.h>
void _exit(int status);
```

* _exit 函数和 _Exit函数会立刻返回内核，exit 函数则会执行一些清理过程

典型的，exit函数会执行关闭标准I/O库的任务：通过fclose关闭所有打开的流，这导致所有已缓冲的数据会被"冲刷"

这三个函数都需要一个所谓退出状态的整型变量作为参数。大部分UNIX系统终端都提供一个方法测试这个进程退出的状态(echo $?)。

有如果(a)任何函数在**没有退出状态**的情况下被调用，(b)main函数执行return 语句没有返回一个值，(c)mian 函数没有被声明返回一个整型量。那么进程的推出状态是无法预料的。



**atexit Function**

IOS C 下，一个进程至少可以注册32个函数由exit 自动调用。这些函数被称为**退出处理程序**

```c
#include<stdlib.h>
int atexit(void (*func)(void));
//OK返回0，错误返回非0
```

exit 调用这些函数的顺序和其注册顺序相反。在IOS C和 POSIX.1标准中exit 先调用推出处理程序，然后后关闭所有已打开的流。POSIX.1 扩展了IOS C标准，当一个程序执行exec 族的函数，任何已被安装的退出处理程序都将被清除。

![C程序的启动的终止](../jpg/C%E7%A8%8B%E5%BA%8F%E7%9A%84%E5%90%AF%E5%8A%A8%E7%9A%84%E7%BB%88%E6%AD%A2.png)

可以从图中看出：**内核执行一个程序的唯一方法是一个 exec 函数的调用，而进程的自动终止的唯一方法是 _exit 或者 _Exit 函数的调用（exit是隐式调用）。**进程可以也通过信号来非自愿终止。

图中也可以看出如果通过return返回，则会将控制权交给C执行例程，由C执行例程调用 exit 函数，最终将控制权返还给内核，这也意味着，return也会导致退出处理函数的执行。



### 7.4 Command-Line Arguments

当一个进程被执行时，exec可以传输命令行参数传给新程序。



### 7.5 Environment List







![环境由五个C字符串组成](../jpg/%E7%8E%AF%E5%A2%83%E7%94%B1%E4%BA%94%E4%B8%AAC%E5%AD%97%E7%AC%A6%E4%B8%B2%E7%BB%84%E6%88%90.png)



# 8.0  进程控制

# 9.0  进程关系

# 10.0  信号

# 11.0  线程

# 12.0  线程控制

# 14.0  高级I/O

# 15.0  进程间的通信

# 17.0  高级进程间通信

