

# UNIX高级环境编程（APUE）



### 1.7 出错处理

* UNIX函数出错时，长返回一个负值，而且整型变量errno通常被设计为含有附加信息的一个值
  * 如果没有出错，则其值不会被一个例程清除。因此仅当函数返回值指明出错时，才检验其值
  * 任何一个函数都不会将errno值设置为0，在<errno.h>中定义的所有常量都不为0

```c
#include<string.h>
char *strerror(int errnum);
//返回值指向消息字符串的指针
```

* 此函数将errnum（errno）映射为一个出错的信息字符串，并且返回此字符串的指针。

  

* perror函数基于errno的当前值，在标准出错上产生一条出错信息，然后返回

```c
#include<stdio.h>
void perror(const char *msg)
```

输出由msg指向的字符串，然后是一个冒号，一个空格好，接着对应与errno值的出错信息，最后是一个换行符



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

fcntl 函数可以改变已打开文件的文件属性，对文件描述符进行操作。

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



## 4.0 File and Dirtory

### 4.2 stat、fstat、fstatat、lstat

```c
#include<sys/stat.h>
int stat(const char* restrict pathname, struct stat *restrict buf);
int stat(int fd, struct stat *buf);
int stat(const char* restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char* restrict pathname, struct stat *restrict buf, int flag);
```

* pathname
  * stat函数：返回与此命名文件有关的信息结构。
  * fstat函数：获得已在描述符fd上打开文件的有关信息。
  * lstat函数：类似stat，但是当命名的文件是一个符号链接，返回符号链接有关的信息。
  * fstatat函数：相对于当前打开目录（fd参数指向）的路径返回文件统计信息。

* buf获取信息

```c
struct stat{
    mode_t	st_mode;
    ino_t			st_ino;
    dev_t			st_dev;
    dev_t			st_rdev;
    nlink_t			st_nlink;
    uid_t			st_uid;
    gid_t			st_gid;
    off_t			st_size;
    struct timespec	st_atime;
    struct timespec	st_mtime;
    struct timespec	st_ctime;
    blksize_t		st_blksize;
    blkcnt_t		st_blocks;
}
```



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

这三个文件描述符stdin、stdout和stderr加以引用



### 5.4 Buffering

由标准I/O库提供的缓冲区的目标是使得read 函数和 write 函数的调用尽可能的少（read，write 是系统调用，需要从用户态切换到内核态，开销比较大）

1. 全缓冲。在这种情况下，**当标准I/O缓冲已满时，所使用的缓冲区通常为在流上执行I/O时，有调用malloc的一个标准I/O函数获得的。**

   术语flush 描述了标准I/O缓冲区的写入。缓冲区有标准I/O例程自动刷新，或者可以调用fflush 函数来刷新流。flush在unix环境中两个用途，一是写出缓冲区的内容，二是丢弃存在缓冲区中的数据。

   **对于驻留在磁盘上的文件是由标准I/O库实施全缓冲的。**注意fflush的使用。

2. 行缓冲。

   在一个流被指向了一个终端设备时，通常使用行缓冲。

3. 无缓冲。标准I/O库不缓冲字符。

* 标准输入和标准输入通常是全缓冲的，仅除了打开一个交互类的流

* 标准错误从不全缓冲

  默认情况下：

  * 标准错误非缓冲
  * 当流打开一个终端设备时，流是行缓冲的；否则就是全缓冲的。
    * 标准错误无缓冲；**打开终端设备的标准输入输出流行缓冲；其他标准输入输出流全缓冲。**所以注意fflush的使用

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
| r+ or r+b or rb+ | 为读和写打开                            |               |
| w+ or w+b or wb+ | 把文件截断至0。或为读和写打开           |               |
| a+ or a+b or ab+ | 为在文件尾读和写而打开和创建            |               |



**除非流引用的是终端设备，否则按系统默认情况，流被打开时是全缓冲。若流引用终端设备，则该流是行缓冲的。**





### 5.6 读写流



~~~c
#include<stdio.h>
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
~~~



~~~c
#include<stdio.h>
int fgets(char *restrict buf, int n,FILE *restrict fp);
int gets(char *buf);
~~~
* fgets的执行顺序是，从流中获取sizeof(buffer)-1个数的字符（若文件末尾返回NULL），并会自动在末尾添加‘\0’
* 每次添加一个字符，并判断当前添加字符是否为‘\n’，若为‘\n’,则返回。
* 所以如果字符串长度大于缓冲区大小，则会导致‘\n’仍停留在缓冲区，导致下一次结果出错



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

    

> **Exit Function**


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



> **atexit Function**

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



环境列表是一个字符指针数组，每个指针都包含以空结尾的C字符串地址。指针数组的地址存在全局变量environ

`extern char **environ;`

![环境由五个C字符串组成](../jpg/%E7%8E%AF%E5%A2%83%E7%94%B1%E4%BA%94%E4%B8%AAC%E5%AD%97%E7%AC%A6%E4%B8%B2%E7%BB%84%E6%88%90.png)

一般通过 getenv 和 putenv （代替使用environ）获取环境变量，如果要获取全部环境，则需要用到 environ。



### 7.6 Memory Layout of a C Program

基于历史的，一个 C 程序有以下几个部分组成

* 代码段

  由供cpu执行的机器指令组成，通常，代码段是共享的，对于频繁执行的程序在内存中只有一份拷贝。此外，代码段是只读的，以防止程序被指令以外修改。

* 初始化的数据段

  程序中已经初始化过的数据。

* 未初始化的数据段

  在程序执行前，此段中数据由内核初始化为算数0 或者 null指针。

* 栈

  存储自动变量，以及每次调用函数时保存的环境信息（返回的地址，调用函数前的一些寄存器的数据信息），新调用的函数在栈上为它的自动变量和临时变量分配空间。这是递归的工作方式，每次递归函数调用自身时，都会push当前的环境信息，然后操作栈为自己提供自身所需空间。

* 堆

  在此处发生动态的内存分配，位于.bss段和栈之间。

**程序中唯一需要保存在程序文件终端部分是text segment 和 initialized data。.bss不占磁盘空间**

![传统的内存分布](../jpg/%E4%BC%A0%E7%BB%9F%E7%9A%84%E5%86%85%E5%AD%98%E5%88%86%E5%B8%83.png)



### 7.7 Shared Libraries

共享库从可执行文件中删除了公共库例程，在内存中的某个地方维护库例程的单个副本，让所有进程都引用它。

虽然减少了每个可执行文件的大小，但可能会增加运行时开销，不论是程序第一次执行时，还是第一次调用每个共享库函数时。

但是库函数版本更新时，不需要重新连接编译每个程序（参数种类和数量不变情况下）



### 7.8 Memory Allocation

* ISO C 三个内存分配的函数

  * malloc

    分配指定字节数存储区，初始值不确定

  * calloc

    为指定数量和长度的对象分配空间，初值为0

  * realloc

    更改之前分配的长度，可能需要将以前分配的内容移到另一个足够大的区域，以便在尾端增加存储区，初值不确定

    由于可能会改变空间的位置，所以晋谨慎用指针指向该存储区的对象

```c
#include<stdlib.h>
void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);
void *realloc(void *ptr, size_t newsize);
void free(void *ptr);
```

* 这些分配例程通常用 `sbrk` 系统调用实现，`sbrk`可以扩充或缩小进程的存储空间，但大多数`malloc`和`free`不减小进程的存储空间。释放的空间可供以后再分配，然而通常在malloc池中而不返回内核
* 分配的存储空间比所要求的大一点，额外空间用来记录管理信息（分配块的长度、指向下一个分配快的指针等）。
  **超过一个分配块的尾端或者在一个分配块之前进行写操作，会重写另一个块中的记录管理信息，在动态分配的缓冲区前后进行写操作，破坏的可能不仅仅是该区管理记录信息，在动态分配的缓冲区前后的存储区可能用于其他动态分配的对象**。
* free一个已经释放的那块，free的对象出错，忘记free



### 7.9 Environment Variables


环境字符串显示为：name=value

```c
#include<stdlib.h>
char *getenv(const char *name);
//返回值：指向与name关联的value的指针；否则返回NULL
```

```c
#include<stdlib.h>
int putenv(char *str);
int setenv(const char *name, const char *value, int rewrite);
int unsetenv(const char *name);
```

* putenv取形式为name=value的字符串，将其放到环境表中。如果name已经存在，则先删除原来的定义。
* setenv将name设置为value。如果name已经存在，如果rewrite非0，则删除定义，如果rewrite为0，则不删除定义，也不出错。
* unsetenv删除name的定义。即使不存在该定义也不出错。

 

### 7.10 setjmp and longjmp Functions








### 7.11 getrlimit and setrlimit Functions

### 7.12 Summary

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  

# 8.0  进程控制



### 8.2 Process Identifiers

每个进程有独一无二的进程ID（一个非负整数）。进程终止时，它们的ID会回到可重用的候选ID池中。

大部分UNIX系统实现了延迟重利用的算法，使得一个新创建的进程的ID和最近刚被终止的进程ID不同，以防止新进程被误认为使用了相同ID的前一个进程。

有一些特殊的进程：

* 进程ID 0 是一个程序调度进程

* 进程ID 1 是初始话init进程，由内核引导过程结束是调用

  旧版本中是/etc/init，新版本中是/sbin/init。分则引导内核后启动UNIX系统。init会阅读系统以来的初始化文件（/etc/rc*[unix系统的启动等级文件]，/etc/inintab[系统的默认启动等级设置文件]），启动的服务放在/etc/init.d 文件中

* 进程ID 2 是pagedaemon，负责支持虚拟内存系统的分页



```c
#include<unsitd.h>
pid_t getpid(void);
//获取当前进程id
pid_t getppid(void);
//获取当前进程父进程id
uid_t getuid(void);
//获取当前进程real user id
uid_t geteuid(void);
//获取当前进程effecitve user id
gid_t getgid(void);
//获取当前进程real group id
git_t getegid(void);
//获取当前进程effective group id
```



### 8.3 fork Function

通过`fork` 函数创造一个新的进程

```c
#include<unistd.h>
pid_t fork(void);
//子进程返回0，父进程返回子进程id，错误返回-1
```

* 这个函数被调用一次，但是返回两次。
* 父进程在创建子进程后，无法通过其他函数获取子进程id（且一个父进程可以有多个子进程）
* 子进程可以通过`getppid`函数获取父进程id（一个子进程仅有一个父进程）
* 子进程拥有父进程的数据空间、堆、栈的一个副本（拷贝并非共享）。
* 子进程和父进程共享代码段（text segment）
* 现代实现通过写时复制技术，不会对父进程执行完全的（数据、堆、栈）拷贝，父子进程共享这些域，并把它们的状态设置为只读，当任一进程尝试修改这些域时，拷贝才会发生

  * 通过	
* 父进程与子进程的执行顺序取决于内核的调用算法（之后会讨论 8.9 10.16）











> **File Sharing**

* 重定向父进程标准输出时，子进程的标准输出也被重定向。子进程的标准输出也会被重定向。
* fork会使父进程的所有打开文件描述符都被复制到子进程中，且**共享文件表项**。fork后父子进程处理描述符文件的情况：
  * 没有同步情况下，对同输出混乱。
  * 父进程等待子进程完成。
  * 父子各自执行不同的程序段。各自关闭不需要用的文件描述符。





### 8.4 vfork Function

* 用于创建一个新进程，而该进程的目的是exec一个新进程

* 在进程调用exec或者exit之前，在父进程的地址空间中运行

* 保证子进程先运行，在调用exec或者exit后，才允许父进程调度运行（这钟机制可能会导致依赖父进程操作的子进程产生死锁）

  （fflush 到底是什么意思 exit执行terminate前的冲刷缓冲区的操作是什么意思 什么是缓冲区）



### 8.5 exit Function

> **正常终止操作**

* main中执行return操作 等效于 执行exit。
* 调用exit
* _exit或者 _Exit，目的是为了提供一种无需运行终止处理程序，或者信号处理程序的终止方式，标准I/O是否被冲刷，取决于实现。
  * 大多是UNIX系统中，exit是一个标准C库函数，_exit 是系统调用。
* 进程最后一个线程启动例程中执行返回语句，返回值不会作用进程返回值，进程返回值为0。
* 进程最后一个线程嗲用pthread_exit。同上

> **异常终止操作**

* abort，会产生SIGABRT信号
* 进程接受信号时
* 最后一个线程对 取消 请求做出响应。系统默认，取消以延迟方式发生：一个线程要求需要另一个线程，一段时间后，目标线程终止。

> 孤儿进程

* 子进程终止会将退出状态返还给父进程，如果父进程先于子进程终止，那么子进程会被init进程接受。

* 内核为每个终止的子进程保存了 一定量的信息（ID 、终止状态、CPU时间总量）。
* 父进程可以通过wait或waitpid获取终止子进程的信息，之后内核释放其相关资源。
* 如果无法被父进程善后处理的进程就成为了僵尸进程。



### 8.6 wait 和 waitpid Function

子进程终止事件是异步进行的，进程正常或者异常终止时，由内核向父进程发生SIGCHLD信号。

父进程可以忽略该信号，或者捕获该信号进行其他函数调用。调用wait 或者 waitpid或出现：

* 如果其所有子进程都还在运行，则阻塞。
* 如果一个子进程已终止，正等待父进程获取其终止状态，则取得该子进程的终止状态立刻返回。
* 如果没有任何子进程，则立刻出错返回。

所以如果随意使用wait，则会导致进程阻塞。f't

```c
#include<sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int statloc, int options);
//    成功返回进程ID，0，出错返回-1
```

* 子进程的退出终止状态存储在参数指针statloc指向的区域，字节的某些部分用来存储正常退出的exit status，另一部分用于存储非正常退出的信号数字，一个bit位用于存贮是否生成核心文件。
* 四种设置在头文件<sys/wait.h>中的宏定义用来获取终止状态

| Macro                | Description |
| -------------------- | ----------- |
| WIFEXITED(status)    |             |
| WIFSIGNALED(status)  |             |
| WFSTOPPED(status)    |             |
| WIFCONTINUED(status) |             |



### 8.7 waitid Function

waitid函数类似与waitpid，且使用起来更加灵活

```c
#include<sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
//return 0 if ok,-1 on error
```



### 8.8 wait3 and wait4 Functions

提供了额外的参数用于存储被中断进程及其所有子进程所使用的资源的摘要。

```c
#include<sys/types.h>
#include<sys/wait.h>
#include<sys/time.h>
#include<sys/resource.h>
pid_t wait3(int *statloc, int options, struct rusage *rusage);
pid_t wait4(pid_t pid, int *statloc, int options, struct rusage *rusage);
//return pid if ok,or -1 on error
```



### 8.9 Race Conditions

* 无法预期fork之后，哪个进程会先开始运行（即使知道哪个进程会先运行，在该进程开始运行之后会发生什么，也取决于系统负载和内核的调度算法）



### 8.10 exec Function

* 执行exec函数时，该进程执行的程序完全替换成新程序，而新程序则从其用一个全新的程序替换了当前进程的正文、数据、堆、堆栈。

  exec后，进程ID未变。且新程序从调用进程继承了下列属性：

  * 进程ID  和 父进程ID
  * 实际用户ID 和 实际组ID（real user ID 和 real group ID）
  * 附属组ID
  * 进程组ID
  * 会话ID
  * 控制终端
  * 闹钟剩余时间
  * 当前工作目录
  * 根目录
  * 文件模式创建屏蔽字
  * 文件锁
  * 进程信号屏蔽
  * 未处理信号
  * 资源限制
  * nice值
  * tms_utime、tms_stime、tms_cutime以及tms_cstime值

* 六种exerc函数可供使用。

```c
#include<unist.h>
int execl(const char *pathname, const char *arg0,/* char *arg1,..., (char *)0 */);

int execv(const char *pathname, char *const argv[]);

int execle(const char *pathname, const char *arg0,/* char *arg1,..., (char *)0 */, char *const envp[]);

int execve(const char *pathname, char *const argv[], char *const envp[]);

。int execlp(const char *filename, const char *arg0,/* char *arg1,..., (char *)0 */);

int execvp(const char *filename, char *const argv[]); 

int fexecve(int fd, char *const argv[], char *const envp[]);
```


* 如果filename中包含 / ，就将其视为路径名；
* 否则就按PATH环境变量，在指定的各目录中搜索可执行文件

![exec族函数关系](../jpg/exec%E6%97%8F%E5%87%BD%E6%95%B0%E5%85%B3%E7%B3%BB.png)








_testinterp.exe_
```c
  #include<stdio.h>
  #include<stdlib.h>
  int main(int argc, char *argv[])
  {   
      int i;
      char **ptr; 
      extern char **environ;
      for(i = 0; i < argc; i++)
          printf("argv[%d]: %s\n",i,argv[i]);
 /*
      for(ptr = environ; *ptr != 0; ptr++)
          printf("%s\n",*ptr);
 */
      exit(0);
  
 }
```
_从该程序启动_
```c
 #include<stdio.h>
 #include<sys/wait.h>
 #include<unistd.h>
 #include<stdlib.h>
 char *env_init[] = {"USER=unknow","PATH=/home/ubuntu2004/test",NULL}    ;
 
 
 int main(void){
     pid_t pid;
     if((pid = fork())<0){
        printf("fork error");
    }
     else if(pid == 0){
 /*  
        if(execl("/home/ubuntu2004/test/testinterp","echoall","only     1 arg",(char *)0)<0)
            printf("execl error");
 */
        if(execle("testinterp","echoall","only 1 arg",(char *)0,env_    init)<0)
        printf("execle error");
   }
   exit(0);

}

```



### 8.12 Interperter file



### 8.13 system Function

system函数对系统的依赖性较强。可以直接在程序中执行命令行语句。

```c
#include<>stdlib.h>
int system(const char *cmdstring);
```

* 传入一个空指针，若支持命令行语句返回非零值。
* fork失败或者waitpid返回eintr之外的错误，返回-1，设置errno。
* 如果exec失败（不能执行shell），则其返回值如同shell执行了exit（127）。
* 如果所有函数（fork、exec、waitpid）都成功，那么system的返回值是shell的终止状态。



### 8.14 Process Accounting



### 8.15 User Identification

* 通过getpwuid函数获取运行该程序的用户的登录名字
* 系统会跟踪我们登录的名字，通过getlogin函数提供获取该登录名的方法

```c
#include<unistd.h>
char *getlogin(void);
```

* 如果进程没有附加到用户登录的终端上，此功能可能会失效



### 8.16 Process Scheduling

* 通过调整nice值以选择更低的优先级运行，返回在0~（2*NZERO）-·之间。
* NZERO是系统默认的nice值

```c
#include<unistd.h>

int nice(int incr);

//成功返回新的nice值，出错返回-1
```

incr参数会增加到调用进程的nice值上，如果incr太大/太小，系统（没有提示）自动降到合法值上。

由于返回值可以为-1，所以出入出错则需要通过判断errno来去欸的那个，在调用钱需要确定errno的值。



* getpriority函数可以像nice函数一样来获取nice，也可以获得一组相关进程的nice值

```c
#include<sys/resource.h>
int getpriority(int which, id_t who);
//成功返回0，出错返回-1
```

* which的参数可取以下值：

  * PRIO_PROCESS    进程
  * PRIO_PGRP          进程组
    * who == 0，返回优先级最高的		
  * PRIO_USER          用户ID
    * who == 0，返回进程的实际用户ID

* setpriority函数可用于设置优先级

```c
#include<sys/resource.h>
int setpriority(int which, id_t who, int value);
//成功返回0，错误返回-1
```

* 参数说明同上



### 8.17 Process Time

可以度量3个时间：时钟时间、用户CPU时间和系统CPU时间。

```c
#include<sys/times.h>
clock_t times(struct tms *buf);
```

buf指向的tms结构，结构定义如下：

```c
struct tms{
    clock_t tms_utime; //user cpu time
    clock_t tms_stime; //system cpu time
    clock_t tms_cutime;//user cpu time，terminated children
    clock_t tms_cstime;//system cpu time，terminated children
}
```















# 9.0  进程关系

# 10.0  信号

* 信号的两个状态
  * 递达：进程接收信号，转而执行某种粗粒操作。
  * 未决：信号产生到递达的中间状态
* 阻塞：信号一旦阻塞就不会递达，此时信号处于未决状态，直至解除阻塞。
* 信号实现进程控制，进程之间的通讯。

### 10.3 Function signal

```c
#include<signal.h>
void (*signal(int signo, void (*func)(int)))(int);
```

* 参数:
  * signo: 信号名
  * func: 
    * 函数指针:指向一个参数为整型的无返回值函数,收到该信号后执行的函数,也可以称为信号处理函数
    * SIG_IGN: 向内核表示忽略此信号
    * SIG_DFL: 像内核表示执行系统默认操作                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     
    * 
* 返回值:
  * 返回一个函数指针,指向一个参数为整型的无返回值函数,返回值指向的函数指向一个信号处理函数



### 10.4 Unreliable Signals

### 10.5 Interrupted System Calls

### 10.6 Reentrant Functions

* 可重入函数

### 10.7 SIGCLD Semantics

* 子进程状态改变后产生此信号，父进程需要调用一个wait函数以确定发生了什么。
  * 如果进程将该信号的配置设置为SIG_IGN，则调用进程将不产生僵死信号。如果调用该进程后随即调用wait，将阻塞直至所有进程终止，然后wait返回-1，并将errno设置为ECHILD
  * 如果将SIGCLD的配置设置为捕捉，内核检测是否有子进程是否准备好被等待，如果是这样，则调用SIGCLD处理程序。

### 10.8 Reliable-Signal Terminology and Semantics

* 事件可以引发信号
  * 硬件异常（除以0）
  * 软件条件（alarm计时器超时）
  * 终端产生的信号或调用kill函数
* 信号在产生和递送之间的时间间隔，信号是未决的
* 多次传递阻塞信号时，大部分系统不会对信号进行排队。无论阻塞期间每个信号传递多少次，在解除阻塞后，每个信号只会传递一次。





### 10.9 kill and raise Functions

> **通过系统调用`kill()`将一个信号传递给另一个进程**

```c
#include<signal.h>
int kill(pid_t pid, int signo);
int raise(int signo);
```

* 调用 `raise(signo)` 等价于 `raise(int signo)`
* pid
  * 大于0，会将信号发送给指定`pid`的进程
  * 等于0，发送信号给与调用进程同组的每个进程，包括调用进程自身。
  * 小于-1，那么会向组内ID等于该`pid`绝对值的进程组内所有下属进程发送信号
  * 等于-1，发送返回时，调用进程有权将信号发往的每个目标进程，出去init和调用进程自身。
* 如果无进程与指定`pid`匹配，那么`kill()`调用失败，会将`errno`设置为`ESRCH`
* 进程发送信号给另一个进程需要适当的权限
  * 特权级（CAP_KILL）进程可以向任何进程发送信号。
  * 以root用户和组运行的init进程（pid=1），仅能接受已安装了处理器函数的信号。（可以防止系统管理员以外杀死init进程）
  * 非特权进程可以想任意进程执行SIGCONT信号，以重启已停止的作业（进程组）![权限](../jpg/%E6%9D%83%E9%99%90.PNG)
* 如果`kill()`无权发送信号对应的pid进程，那么`kill()`将调用失败，且将errno置为EPERM。pid若为一系列进程时（进程组），只要向其中之一发送信号，则`kill()`调用成功。



> **检查进程存在**

* 向指定pid，将参数sig指定为0发送。
  * pid不存在，进程不存在
  * pid存在，进程为僵尸进程
  * pid存在，但该pid已经被其他进程利用



> raise()

* **当进程使用该函数向自身发送信号时，信号将立即传递（在raise返回之前，意味着先对信号信号进程处理）。**
* 唯一出错的可能是发生EINVAL， sig无效。

### 10.10 alarm and pause Functions

* 使用alarm函数以设置一个计时器，在将来某个指定时间该计时器会超时。超时时，产生SIGALRM信号，如果不忽略或者不捕捉该信号，默认东子是终止调用该alarm函数的进程。

```c
#include<unistd.h>
unsigned int alarm(unsigned int seconds);
```



* pause函数使调用进程挂起直至捕捉到一个信号、

  ```c
  #include<unistd.h>
  int pause(void);
  //返回-1，并将errno设置为EINTR
  ```
  
* 只有执行了一个信号处理程序从其返回时，pause才返回。在这候着那个情况下，pause返回-1，并将errno设置为EINTR。



* alarm配合pause使用，进程可以时自己休眠一段指定的时间，然后存在一些问题

```c
#include<signal.h>
#include<unistd.h>
static void sig_alrm(int signo)
{
    /*nothing to do, just return to wake up the pause*/
}
unsigned int sleep1(unsigned int nsecs)
{
    if(signal(SIGALRM, sig_alrm) == SIG_ERR)
        return(nsecs);
    alarm(nsecs);
    pause();
    return (alarm(0));
}
```

* 问题
  * 调用sleep1之前，调用者处于某种目的已设置了闹钟，则它会被sleep1中的alarm函数重写时钟，如果上次时钟超时时间大于本次设定，则需要在返回时复位预先设定的闹钟。如果小于，则只能等待上次闹钟超时。（这样不会造成某种意外，原先的闹钟产生的操作被忽略）
  * 修改了对SIGALRM的配置，导致之前的信号处理函数失效，需要调用前保存signal的返回值
  * pause和signal存在竞争关系，导致如果先进行信号处理，然后调用pause，进程会永远挂起。



### 10.11 Signal Sets

> 信号集处理函数

```c
#include<signal.h>
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
//成功返回0，出错返回-1
int sigsimember(const sigset_t *set, int signo);
//真返回1，假返回0
```

### 10.12 sigprocmask Function

* 调用函数sigprocmask可以检测或更改，或同时检测更改进程的信号屏蔽字。

```c
#include<signal.h>
int sigprocmask(int how, const sigset_t *restrict oset);
```

* 参数how
  * SIG_BLOCK 				该进程新的信号屏蔽字是当前信号屏蔽字和set指向信号集的并集	
  * SIG_UNBLOCK           该进程新的信号屏蔽字是当前信号屏蔽字和set指向信号集的补集的交集（人话是将set指向的信号集中的信号从信号屏蔽字中删除）		
  * SIG_SETMASK            该进程心的信号屏蔽字是set指向的值

* oset用于保存之前的信号屏蔽集



### 10.13 sigpending Fuction

```c
#include<signal.h>
int sigpending(sigset_t *set);
//返回值：成功返回0，失败返回-1
```

* 对调用进程而言，其中的各信号是阻塞不能传递的，因而信号是未决的。



### 10.14 sigaction Function

* 允许我们测试或者修改与特定信号相关的行为。

```c
#include<signal.h>
int sigaction(int signo, const struct sigaction *restrict act, struct sigaction *restrict oact);
//返回值：成功，出错-1
```

* 参数signo是要检测或者修改其动作的信号编号。。
* act指针非空，则修改其动作。
* oact指针非空，则有oact指针返回该信号的上一个动作。

```c
struct sigaction{
    void (*sa_handler)(int);
    sigset_t sa_mask;
    int sa_flags;
    void (*sa_sigaction)(int, siginfo_t *, void *tg)
    
}
```

* 只有sa_handler字段对应于signal()的handler参数。其所指定的值为信号处理函数的地址，或者是常量SIG_IGN、SIG_DFL之一。

* 仅当sa_handler的取值是信号处理函数时，才会对sa_mask和sa_flags字段加以处理。

  * sa_mask指定一组信号集，引发信号处理函数调用会自动将其加入到进程屏蔽集中，在信号处理函数返回时自动删除。
  * sa_flags是一个位掩码。指定用于控制信号处理过程中的各种选项。

* sa_sigaction不适用于应用程序。该字段是一个代替的信号处理程序，sa_sigaction和sa_handler字段只能使用其中一个。

  * 通常按如下方式调用信号处理程序

    `void handler(int signo);`

  * 当sa_flags中设置了SA_SIGINFO标志时，按照下列方式调用信号处理程序

    `void handler(int signo, siginfo_t *info, void *context);`

    siginfo_t结构包含了信号产生原因有关的信息。

    

### 10.15 sigsetjmp and siglongjmp

```C
#include<setjmp.h>
int sigsetjmp(sigjmp_buf。 env, int savemask);
//直接调用红返回0，从siglongjmp返回非0值。
void siglongjmp(sigjmp_buf env, int val);
```

* savemask非0，则sigsetjmp中保存进程的当前信号屏蔽字，当siglongjmp调用返回时，信号屏蔽字会从env中恢复。



### 10.16 sigsuspend Function

```c
#include<signal.h>
int sigsuspend(const sigset_t *sigmask);
```

* 将进程的屏蔽信号字设置为sigmask指向的信号集。
* 在捕捉到一个信号或者发生一个会终止该进程信号之前，该进程会挂起。
* 如果捕捉到一个信号且从信号处理程序返回，则sigsuspend返回，并且信号屏蔽字还原。

* 某些信号已经被屏蔽后，想通过该信号以实现唤醒进程，可以使用该函数（类似 pause，但是由于该信号已被屏蔽，或者解除屏蔽后立即递达，导致进程永远阻塞）。

* sigsuspend()，可以通过sigmask设定在该函数调用期间进程的信号屏蔽集，之后进程挂起，直至某个信号递达，sigsuspend()返回，然后信号屏蔽集复原。
* 利用该函数可以实现父子进程间的同步。

```c
   #include"tool1.h"
   
   
   static void sig_int(int signo){
       printf("something happend\n");   
   }   
   
   
  int main(){
      if(signal(SIGINT,sig_int));
      
      sigset_t newmask,oldmask;
  
      sigemptyset(&newmask);
      //sigemptyset(&oldmask);
      
      sigaddset(&newmask,SIGINT);
      if(sigprocmask(SIG_BLOCK, &newmask, &oldmask))
          printf("err\n");
         
      for(int i=0;i<5;i++){
          printf("%d\n",i);
          sleep(1);
      }
  
      sigsuspend(&oldmask);
  
      for(int i=0;i<5;i++){
          printf("%d\n",i);
          sleep(1);
      }
      exit(0);
  
  }
```



### 10.17 abort Function

* 使异常程序终止

```c
#include<stdlib.h>
void abort(void);
```

将SIGBRT信号发送给调用进程。调用abort将向主机环境传递一个未成功终止的通知，其方法是调用**raise(SIGABRT)**函数，这也意味着，在abort()中先传递了信号，进行处理，之后才会继续执行abort()，然后返回。

无论如何abort()都将终止程序，（除非进行了非本地跳转退出信号处理函数），因为在处理信号后，abort(),将对SIGABRT的处理操作重置为默认，然后再度发出一次SIGABRT信号，最后退出。

abort()会对所有打开的流执行fflush操作（实现执行fflush）。只有进行了信号处理函数，且在信号处理函数中通过_exit 或者 _Exit退出程序，才能取消fflush（然而内核会丢弃）。在从信号处理函数返回后，abort会调用fflush()。

实现代码：

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void
abort(void)
{
    sigset_t
    struct sigaction    action;
    /* Caller can’t ignore SIGABRT, if so reset to default */
    sigaction(SIGABRT, NULL, &action);
    if (action.sa_handler == SIG_IGN) {
        action.sa_handler = SIG_DFL;
        sigaction(SIGABRT, &action, NULL);
    }
    if (action.sa_handler == SIG_DFL)
        fflush(NULL);           /* flush all open stdio streams */
    /* Caller can’t block SIGABRT; make sure it’s unblocked */
    sigfillset(&mask);
    sigdelset(&mask, SIGABRT);  /* mask has only SIGABRT turned off */
    sigprocmask(SIG_SETMASK, &mask, NULL);
    kill(getpid(), SIGABRT);    /* send the signal */
    /* If we’re here, process caught SIGABRT and returned */
    fflush(NULL);               /* flush all open stdio streams */
    action.sa_handler = SIG_DFL;
    sigaction(SIGABRT, &action, NULL);  /* reset to default */
    sigprocmask(SIG_SETMASK, &mask, NULL);  /* just in case ... */
    kill(getpid(), SIGABRT);                /* and one more time */
    exit(1);    /* this should never be executed ... */
}
/* POSIX-style abort() function */
```



### 10.18 system Function

### 10.19 sleep, nanosleep, and clock_nanosleep Functions
### 10.20 sigqueue Function
### 10.21 Job-Control Signals
### 10.22 Signal Names and Numbers









# 11.0  线程

### 11.2 线程概念

### 11.3 线程标识

* 线程ID只是在进程上下文中才有意义。

* 该函数比较两个线程ID

  ```c
  #include<pthread.h>
  int pthread_equal(pthread_t tid1, pthread tid2);
  ```

* 该函数获取自身线程ID

  ```c
  #include<pthread.h>
  pthread_t pthread_self(void);
  ```

* 线程创建

  ```c
  #include<pthread.h>
  int pthread_create(pthread_t *restrict tidp, 
                     const pthread_attr_t *restrict attr, 
                     void *(*start_rtn)(void *), void *restrict arg);
  ```

  * 当函数成功返回时，新创建线程的线程ID会被设置成tidp指向的内存单元。
  * attr用于定制各种不同的线程属性。
  * 新创建的线程从start_rtn函数的地址开始运行，该函数只有一个无类型指针参数arg。
  * 线程可以访问进程的地址空间，继承调用线程的浮点环境和信号屏蔽字。
  * 主线程与子线程存在竞争关系

### 11.5 线程终止

* 单个线程可以通过一下三种方式退出。

  * 线程知识从启动例程中返回，返回值是线程的退出码
  * 线程可以被统一进程中的其他线程取消
  * 线程调用pthread_exit

  ```c
  #include<pthread.h>
  int pthread_join(pthread_t thread, void **rval_ptr);
  ```

  进程中的其他线程可以通过调用pthread_join函数访问到这个指针

  ```c
  #include<pthread.h>
  int pthread_join(pthread_t thread, void **rval_ptr);
  ```

  调用线程将一直阻塞v知道指定的线程调用pthead_exit\从启动例程中返回或者取消。

  如果线程只是从它的启动例程返回， rval_ptr将包含返回码。如果线程被取消，由rval_ptr指定的内存单元就置为PTHREAD_CANCELED。

  通过pthread_join自动把线程置于分离状态，这样资源就可以恢复。如果线程已处于分离状态，函数调用就会失败，返回EINVAL。

  如果对线程返回值不感兴趣，可以把rval_ptr置为NULL。在这种情况下，调用pthread_join函数将等待指定的线程终止，但并不获取线程终止状态。

  * 通过pthread_cancel函数来请求取消同一进程中的其他线程。

  ```c
  #include<pthread.h>
  int pthread_cancel(pthread_t tid);
  //成功返回0，错误返回错误编号
  ```

  默认情况下，pthread_cancel 函数会使得由tid表现为如同调用了参数为PTHREAD_CANCELED的pthread_exit函数，但是，线程可以选择忽略取消方式或是控制取消方式。

  * 类似进程退出处理函数，线程也可以安排退出时需要调用的处理函数，称为线程清理处理程序。
    * 处理顺序依旧类似，记录在栈中，执行顺序与注册顺序相反

  ```c
  #include<pthread.h>
  void pthread_cleanup_push(void (*rtn)(void *),void *arg);
  void pthread_cleanup_pop(int execute);
  ```

  * 当线程执行一下动作时调用清理函数，参数为arg，清理函数rtn的调用顺序是由pthread_cleanup_push函数安排的
    * 调用pthread_exit。
    * 响应取消请求时。
    * 用非零execute参数调用pthread_cleanup_pop时。
      * 清除或者调用后清除一个栈顶的清理函数
  
  * pthread_cleanup_pop都将删除上次pthread_cleanup_pop都将删除上次pthread_cleanup_push调用建立的清理处理程序。
  * pthread_cleanup_push 和 pthread_cleanup_pop 成对出现
  * 由于可以实现为宏，所以必须在与线程相同的作用域中以配对的心是使用
  
  ```c
  #include<pthread.h>
  int pthread_detach(pthread_t tid);
  ```
  
  * 是函数进入放分离状态。



### 11.6 线程同步

* 互斥量

```c
#include<pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destory(pthread_mutex_t *mutex);
//成功返回0,否则返回错误编号
```

互斥量本质上是一把锁,在访问共享资源前对互斥量进行枷锁,在访问完成后释放互斥量上的锁。

互斥变量用pthread_mutex_t数据类型表示，使用前需要初始化，可以把他置为常量PTHREAD_MUTEX_INITIALIZER。也可以通过pthread_mutex_init函数进行初始化。

其他试图对互斥量加锁的线程会被阻塞直至当前线程释放该互斥锁。

多个线程阻塞时，当前互斥量解锁，第一个变成运行状态的线程先加锁。

```c
#include<pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

通过pthread_mutex_lock对互斥量加锁。已上锁会阻塞线程。

通过pthread_mutex_trylock对互斥量加锁。已上锁不会阻塞，返回EBUSY，成功加锁返回0。



* 读写锁

拥有更高的并行性

读写锁也叫共享独占锁。使用前需初始化，在释放它们底层的内存前必须销毁

```c
#include<pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
```

通过pthread_rwlock_init初始化。希望读写锁有默认属性，可以传一个空指针给attr

同互斥量，需要在释放读写锁占用的内存空间前销毁锁。

```c
#include<pthread.h>
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); //读模式下读写锁的锁定
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock); //写模式下读写锁的锁定
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock); //任意模式解锁
```

类似互斥量的有条件读写锁原语版本

```c
#include<pthread.h>
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrwlock(pthread_rwlock_t *rwlock);
```

可以获取锁时，函数返回0；否则返回EBUSY。



* 条件变量

锁的数据类型为pthread_cond_t

可以将锁用PTHREAD_COND_INITALIZER初始化

通过如下函数初始化和销毁

```c
#include<pthread.h>
int pthread_cond_init(pthread_cond_t *restrict cond, pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);
```

使用pthread_cond_wait等待条件变量为真，如果在给定时间不等满足，则会返回一个代表出错码的返回变量。

```c
#include<pthread.h>
int pthread_cond_wait(pthread_cond_t *restrcit cond, pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, const struct timespec *restrict timeout);
```

函数的作用：

1. 将该线程阻塞直至接受一个条件改变的通知。
2. 释放已占用的互斥量mutex。
3. 条件改变后解除阻塞，给互斥量mutex加锁。

第一步和第二步是原子操作，第三步会阻塞直至成功加锁。

通过以下函数通知信号该线程条件已满足。

```c
#include<pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```



* 自旋锁

* 屏障

协调多个线程同步，在允许所有线程继续允许前，必须达到屏障的线程数目。

初始化和销毁

```c
#include<pthread.h>
int pthread_barrier_init(pthread_barrier_t *restrict barrier, 
                         const pthread_barrierattr_t *restrict attr, 
                         unsigned int count);
int pthread_barrier_destroy(pthread_barrier_t *barrier);
```

可以使用pthread_barrier_wait函数表明，线程已完成工作，准备等所有其他线程赶上来。

```c
#include<pthread.h>
int pthread_barrier_wait(pthread_barrier_t *barrier);
```

调用`pthread_barrier_wait`的线程在屏障计数为满足条件时，会进入休眠状态。

如果该线程是最后一个调用`pthread_barrier_wait`的线程，满足了屏障计数，则会唤醒所有线程。

对于任意线程，`pthread_barrier_wait`函数返回了`PTHREAD_BARRIER_SERIAL_THREAD`。剩下的所有线程看到的返回值是0.

这使得一个线程可以作为主线程，它可以工作在其他所有线程已完成工作结果上。

除非在调用了`pthread_barrier_destroy`后又调用了`pthread_barrier_init`进行初始化，否则计数不变。







# 12.0  线程控制





# 14.0  高级I/O

### 14.2 非阻塞I/O

### 14.3 记录锁

当一个进程正在读或者修改一个文件时，它可以阻止其他进程修改同一个文件区域。

* fcntl记录锁

```c
#include<fcntl.h>
int fcntl(int filedes, int cmd,.../*struct flock *flockptr*/);
```

对于记录锁，cmd是F_GETLK、F_SETLK或者F_SETLKW。第三个参数是一个指向flock结构的指针：

```c
struct flock{
    short l_type;
    off_t l_start;
    short l_whence;
    off_t l_len;
    pid_t l_pid;
}
```

* F_RDLCK（共享读锁）、F_WRLCK（独占写锁）、F_UNLCK（解锁）

* 加锁或者解锁的区域有l_start和l_whence两者决定
* 区域的字节长度由l_len表示
* 具有能阻塞当前进程的锁，其特有进程ID存放在l_pid中（仅由F_GETLK返回）
* l_len为0，则表示从起点开始锁至最大偏移位置

![IMG_0027(20211026-112303)](../jpg/IMG_0027(20211026-112303).PNG)
* 文件对给定字节区间上锁规则：
	* 文件给定字节区间，多个进程可以有一把共享读锁，即允许多个进程以读模式访问该字节区；
	* 文件给定字节区间，只能有一个进程有一把独占写锁，即只允许有一个进程已写模式访问该字节区；
	* 文件给定字节区间，如果有一把或多把读锁，不能在该字节区再加写锁，同样，如果有一把写锁，不能再该字节区再加任何读写锁。
  * 文件给定字节区间，单个进程只能同时拥有一把锁，**重复请求锁，前锁会替换旧锁。**
  
* 进程必须以相应权限打开文件描述符

* 死锁
  * 两个进程互相等待对方持有而且锁定的资源时，这两个进程就处于死锁状态



### 14.4 STREAMS（过时）

### 14.4 I/O 复用

* select

  * 参数告诉内核：关注的描述符、对于每个描述符我们所关注的条件（读/写/异常）、愿意等待的时间。
  * select返回时，内核告诉我们：已准备好的描述符总量、读\写\异常三个条件中哪一些准备好了。

  ```c
  #include<sys/select.h>
  int select(int maxfdpl, fd_set *restrict readfds, 
             fd_set *restrict writefds, fd_set *restrict exceptfds,
             struct timeval *restrict tvptr);
  ```

  * tvptr 指定愿意等待的时间长长度。

    * `tvptr == NULL`

      永远等待。如果捕捉到一个信号则中断，否则无限期等待。**当指定的描述符中的一个已准备好或者捕捉到一个信号则返回。** 如果捕捉到一个信号，则`select`返回-1， `errno`设置为EINTR。

    * `tvptr->tv_sec == 0 && tvptr->tv_usec == 0`

      根本不等待，测试所有指定的描述符并立刻返回。**单次轮询返回多个就绪文件描述符而不阻塞。**

    * `tvptr->tv_sec != 0 || tvptr->t_usec != 0`

      等待指定描述和微秒数。

   ```C
   #include<sys/select.h>
   int FD_ISSET(int fd, fd_set *fdset);
   void FD_CLR(int fd, fd_set *fdset);
   void FD_SET(INT fd, fd_set *fdset);
   void FD_ZERO(fd_set *fdset);
   ```

  `fd_set`数据类型是一个是一个文件字符集的数据类型，存储了可读、可写、或处于异常条件的描述符集合。对这种数据类型，仅可以分配一个变量，将这种变量值赋给同类型比那辆吗，对其进行以上四种函数操作。

  

* poll

  ```c
  #include<poll.h>
  int poll(struct pollfd fds[], nfds_t nfds, int timeout);
  ```

  * 结构体数组fds
	```c
    struct pollfd
    {
        int 		fd;
        short 	envents;
        short		revents;
    }
	```
	
* `fd`为目标文件描述符，events来指定需要检查的时间。`poll()`返回时，revents被设定为该文件描述符上实际发送的时间。

![2F85788B3F4DD71DE2A555FC5D378C47](../jpg/2F85788B3F4DD71DE2A555FC5D378C47.png)

* 如果对某个特定文件描述符不感兴趣，可以将events设置为0。

* 将fd设为相反数（负数），将导致对应的events字段被忽略，且revents字段将总返回0。

  以上两个方法用来关闭对单个文件描述符的检查。

* timeout参数
  * `timeout == -1`，一直阻塞，知道fds数组中列出的文件描述符有一个达到就绪态或者捕获一个信号。
  * `timeout == 0`，**不会阻塞，单次轮询，返回多个就绪文件描述符状态。**
  * `timeout > 0`，至多阻塞timeout毫秒，知道fds列出的文件描述符中有一个达到就绪态，或者捕获一个信号。







### 14.6 readv Function and writev Function

用于在一次函数调用中读、写多个非连续缓冲区。

```c
#include<sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int lovcnt);
```

iovec结构体数组

```c
struct iovec{
    void *iov_base;
    size_t iov_len;
}
```

iov数组中元素数有iovcnt指定，其最大值受限于`IOV_MAX`。



### 14.7 readn Function and writen Funtion

```c
//多次read
ssize_t readn(int fd, void *ptr, size_t n)
{
	size_t nleft;
	ssize_t nread;
	
	nleft = n;
	while(nleft > 0)
	{
		if((nread = read(fd, ptr, nleft)) < 0){
			if(nletf == n)
				return(-1);
			else
				break;
		}else if(nread == 0)
			break;
		nleft -= nread;
		ptr +=nread;
	}
	return(n - nleft);
}
//多次write
ssize_t writen(int fd, void *ptr, size_t n)
{
	size_t nleft = n;
	ssize_t nwrite;

	while(nleft > 0)
	{
		if((nwrite = write(fd, ptr, nleft)) < 0)
		{
			if(nwrite == n)
				return(-1);
			else
				break;
		}else if(nwrite == 0)
			break;

		nleft -= nwrite;
		ptr += nwrite;
	}
	return(n - nleft);
}
```



### 14.8 存储映射 I/O

* 存储映射I/O能够将一个磁盘文件映射到存储空间的一个缓冲区中

  * 读出缓冲区数据相当于读出文件相应数据。
  * 写入缓冲区数据相当于写入文件相应数据.

  * 在不适用`read`和`write`的情况下执行I/O.

```C
#include<sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
```

* 参数

  * addr

  指定映射存储区的起始地址，设置为0表示由系统选择该映射区的起始地址。此函数的返回值是该映射区的起始地址。

  * fd

  被映射的文件描述符，映射前需要打开该文件。len是映射的字节数，off是要映射字节在文件中的起始偏移量

  * prot

    对应映射区保护要求，**权限不能超过open函数打开文件的保护权限**（open指定RD_ONLY则不能设为PROT_WRITE）

    ![E51F9C191650DDA5ADFD83CF53E1D314](../jpg/E51F9C191650DDA5ADFD83CF53E1D314.png)

  * flag
      * MAP_FIXED
          * 返回值必须等于addr，只能映射到该地址。如果未指定且addr非0，则视为对内核的一种建议，最终内核不一定用。
          
      * MAP_SHARED
          * 对映射区的存储操作视为对文件的write。即会修改原文件。
          
      * MAP_PRIVATE
           * 对映射区的存储操作不会修改原文件，会产生一个文件私有副本，之后对该映射区的引用都是引用到副本。
      
         ![730B0074AF8E6CF396AD0F0A4DAD7E42](../jpg/730B0074AF8E6CF396AD0F0A4DAD7E42.png)
      
         具体情况视实现而定
  
* 相关信号

  * SIGVEGV
  
      进程试图访问对它不可用的存储区。设置只读却试图写入。
  
  * SIGBUS
  
      试图访问映射区某个部分，这个部分已经不存在。
  
* 子进程能fork继承存储映射区（复制父进程地址空间，该区是地址空间的一部分，所以exec不能继承）

* 更改一个现有映射页的权限

```c
#include<sys/mman.h>
int mprotect(void *addr, size_t len,int prot);
```

* 调用msync将该页冲洗到映射的文件

  * flags

    * MS_SYNC

      返回之前等待写操作完成

    * MS_ASYNC

      不等待

    * MS_INVALIDATE

      允许通知操作系统丢弃那些与底层存储器没有**同步**的页

```c
#include<sys/man.h>
int msync(void *addr, size_t len,int flags);
```

映射是私有的（MAP_PRIVATE），不修改文件。

* 解除映射

  munmap不会影响映射对象，所以为确保底层文件被成功修改，需要在这之前调用msync。进程结束也会自动解除映射，MAP_SHARED的页会有内核更新，MAP_PRIVATE的页会丢弃。

```c
#include<sys/mman.h>
int munmap(void *addr, size_t len);
```









# 15.0  进程间的通信 

### 15.2 管道

* 管道只能在具有公共祖先的两个进程之间使用。一个管道有一个进程创建，在进程fork后，这个管道就能在父进程和子进程之间使用。

```c
#include<unistd.h>
int pipe(int fd[2]);
//fd[0] 为读打开 fd[1] 为写打开。 fd[0]的输出是fd[1]的输入
```



### 15.6 XSI IPC

* 服务器进程可以指定键IPC_PRIVATE 创建一个IPC结构，将返回的标识符存在某处以便客户进程取用。

​		缺点：服务器进程要将整型标识符写入文件中，客户进程又要从文件中读出。

* 一个公共头文件定义客户进程和服务器进程都任何的键。然后服务器进程指定此键创建一个新的IPC结构。

  缺点：该键可能已与一个IPC结构相结合。

* 客户进程和服务器认同一个路径名和项目ID，调用ftok函数生成一个键。

```c
#include<sys/ipc.h>
key_t ftok(const char* path, int id);
```

​		缺点：对于不同文件路径和不同项目ID，可能产生相同键。



* 满足两个条件之一，创建一个新的IPC结构

  * key取值IPC_PRIVATE

  * key没有与特定的类型（信号量，消息队列等）结合，且flag中指定了IPC_CREAT

* 指定IPC_CREAT和O_EXCL标志 如果IPC结构已存在，则创建就会报错。









### 15.7 消息队列

```c
#include<sys/msg.h>
int msgget(key_t key, int flag);
```
执行成功返回非负队列ID

```c
#include<sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```

cmd参数说明：

* IPC_STAT

  获取此队列的msqid_ds结构，并存放在指向的buf中。

* IPC_SET

  按由buf指向结构中的值，设置此队列相关结构中的下列四个字段：msg_perm.uid、msg_perm.gid、msg_perm.mode和msg_qbytes。 有效用户ID等于cuid和uid或者具有超级用户的权力才能执行。只有超级用户才能增加msg_qbytes的值。

* IPC_RMID 

  删除该消息队列以及仍在该队列中的所有数据，如果执行，则其他进程访问该队列是会出错。



```c
struct msqid_ds{
    struct ipc_perm	msg_perm;
    msgqnum_t	msg_qnum;
    msglen_t	msg_qbytes;
    pit_t	msg_lspid;
    pid_t	msg_lrpid;
    time_t	msg_stime;
    time_t	msg_rtime;
    time_t	msg_ctime;
};
struct ipc_perm{
	uid_t	uid;
    git_t	gid;
    uid_t	cuid;
    git_t	cgid;
    mode_t	mode;
};
```

```c
#include<sys/msg.h>
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);
```



```c
#include<sys/msg.h> 
int msgrecv(int msqid, void* ptr, size_t nbytes, long type, int flag);
```







# 17.0  高级进程间通信

