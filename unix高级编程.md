---
title: unix高级编程
date: 2019-12-18 22:17:48
tags: -C/C++
categories: -Linux
mathjax:
    true
description: UNIX下高级编程，目前正在学习中，不断更新。
---

<center/> <font size = 15>UNIX环境高级编程</font></center>
# 第一章 UNIX基础知识

## UNIX体系结构

操作系统是一个软件, 控制计算机硬件资源, 提供程序运行环境. 也叫做内核.  内核接口是系统调用. 公共函数库构建在系统之上. 应用程序可以调用系统调用也可以调用公共函数. shell是一个特殊的应用程序, 为运行其他应用程序提供了一个接口. 

### shell

shell是一个命令行解释器, 读取用户输入, 然后执行命令. shell的输入通常来自终端(交互式shell), 有时来源于文件(shell脚本).

## 文件和目录

目录是一个包含目录项的文件. 

创建新目录是会自动创建两个文件名: .(点) ..(点点) 点指向当前目录, 点点指向父目录.

ex: 将工作空间转到父目录的父目录.

```shell
cd ../..
```

### 工作目录

每个进程都有一个工作目录, 称为当前工作目录. 所有相对路径都是从当前目录开始解释. 可以通过chdir函数更改工作目录.

## 输入和输出

### 文件描述符

文件描述符通常是一个小的非负整数, 内核用其来标识特定进程正在访问的文件. 当内核打开或者创建文件时都会返回一个文件描述符. 读写文件时使用文件描述符.

### 标准输入, 标准输出和标准错误

当程序执行时, 所有shell都会打开三个文件描述符, 即标准输入, 标准输出和标准错误. 默认情况下三个描述符都指向终端(即输入输出和错误都通过终端进行交互). 同时可以将一个或者3个描述符重定向到指定文件. "> file_name": 将标准输出重定向到file_name文件中(如果没有就会创建). "<file_name": 将标准输入重定向到file_name中.

ex:  

```
ls > files_list -a
```

 将当前目录下的文件输出到files_list文件中.

对于一个可执行文件a.out

```
./a.out < input_file > output_file 
```

此时程序中标准输入就会从input_file读取, 标准输出就会到output_file中.

### 不带缓冲的I/O

函数open, read, write, lseek以及close提供了不带缓冲的I/O. 均使用文件描述符.  

```
read(文件描述符, char [], BUFFSIZE); //从文件描述符连接的文件读字符串
write(文件描述符, char [], BUFFSIZE); //向文件描述符连接的文件写字符串
```



## 程序与进程

### 程序:

存储在磁盘的可执行文件, 内核使用exec()(7个), 将程序读入内存并执行.

### 进程与进程ID

进程: 程序的执行实例. UNIX保证每个进程都有唯一的一个数字标识符. 即进程ID. 通过getpid()函数获取进程ID.

### 进程管理

三个主要函数: fork, exec和waitpid函数.

#### fork

创建一个新的进程, 返回两个pid_t,  对父进程返回子进程的ID号, 对子进程返回0. 当调用此函数时, 新进程调用父进程的一个副本. 相当于将父进程进行了一份拷贝(?), 在调用该命令之前的信息两个进程完全一致(并不是资源共享, 只是资源的值一致, 对于变量来说, 指向地址不同,而是值相同). 该命令之后的两个进程分别执行. 对于返回的两个不同的值作为用来作为后续执行代码的选择区分.

#### waitpid

等待进程执行结束. 参数为进程ID, 返回进程的终止状态.

ex: simple_shell.c

```c
#include"apue.h"
#include<sys/wait.h>

int main(void)
{
    char buf[MAXLINE];
    pid_t pid;
    int statue;
    printf("%%");
    while(fgets(buf, MAXLINE, stdin) != NULL) //每次读取标准输入的一行, 以空(ctrl+D)作为结尾
    {
        if(buf[strlen(buf)-1] == '\n') //替换换行符为空字符
            buf[strlen(buf)-1] = 0;
        if((pid = fork())<0) // 创建新进程, if后的程序会在父进程与新进程中执行两遍, 按照pid选择执行方式.
            err_sys("fork error");
        else
        {
            if(pid == 0) //执行子进程
            {
                execlp(buf, buf, (char*)0);
                err_sys("coundn't execute: %s", buf);
                exit(127); // 结束子进程
            }
        }
        if((pid = waitpid(pid, &statue, 0))<0) // 由于子进程在之前已经结束, 子进程无法执行到此, 所以pid只能是子进程ID(父进程返回为0), 该语句为等待子进程结束,statue用来返回子进程终止状态。
            err_sys("waitpid error");
        printf("%%");
    }
    exit(0);
}

```

### 线程和线程ID

线程: 某一时刻执行的一组机器指令. 一个进程内所有线程共享同一地址空间, 文件描述符, 栈以及与进程相关的属性. 线程也有线程标识, 线程ID只能所属进程中有效.



## 出错处理

当UNIX出错时, 通常会返回一个负值, 部分整型变量erron通常被设置为具有特定含义的值. <errno.h>中定义了erron以及可以赋予它的常量. 

c标准库定义了两个函数用于打印出错信息.

### strerror()

```c
#include<string.h>
void char* strerror(int errnum); //根据输入的整数(代表错误类型)获得错误信息.
```



### perror()

```c
#inclide<stdio.h>
void perror(const char *mag) ; //向标准输出中打印mag信息: 错误信息.错误信息由erron指明.
```

ex:

```c
#includ"apue.h"
#include<stdio.h>
int main(argc, *argv[])
{
    fprintf(stderr, "EACCES: %s \n", strerror(EACCES)); // EACCES为头文件中包含指明错误的常量.
    erron = ENOENT; // erron同样是被定义在头文件中.用来指明perror打印的错误类型.
    perror(argv[0]);
}
```

## 用户ID

### 用户ID

口令文件登录项中用户ID是一个数值, 标识不同用户. ID= 0 为超级用户. 如果进程具有超级用户权限则大多数权限检测都不用进行.

### 组ID

/etc/group文件中.

### 附属ID



## 信号(signal)

信号用于通知进程发生了某种情况. 进程有三种处理信号的方式:

1. 忽略信号
2. 按默认方式处理.
3. 通过一个函数, 当信号发生时调用该函数, 称为捕获信号.



## 时间值

### 日历时间

从1970年1月1日00:00:00到指定时间进过多少秒.系统基本数据类型time_t存储这种时间值.

### 进程时间

用于度量进程使用CPU资源. 进程时间以时钟滴答计算. 每秒可以有不同时间滴答数取值.

度量一个进程执行的时间时 UNIX系统为一个进程维护3个进程时间值.

1. 时钟时间(CPU时间): 进程运行总时间.
2. 用户CPU时间: 执行用户指令所用的时间量.
3. 系统CPU时间: 进程中调用内核程序时所使用的时间.

用户CPU时间和系统CPU时间之和被称为CPU时间.

获得进程时间方式:  在执行程序的指令前加上time即可.

ex: time ls -a;

## 系统调用和库函数

操作系统提供的服务的入口点被称为系统调用. 

UNIX使用技术为为每个系统调用都在标准C库中设置一个具有同样名字的函数. 用户进程用标准C调用序列来调用这些函数, 这些函数又用系统所需要的技术调用相应的内核服务.

# 第三章 文件I/O

## 文件描述符

对于内核而言, 所有打开的文件都是通过文件描述符引用. 文件描述符为非负整数, 打开文件或者创建一个新文件时将返回一个文件描述符. UNIX系统shell将文件描述符0与标准输入关联(STDIN_FILENO). 1与标准输出关联(STDOUT_FILENO), 2与标准错误关联(STDERR_FILENO).

## 函数open和openat

open和openat用于打开或者创建一个文件. 原型如下:

```c++
#include<fcntl.h>
int open(const char *path, int oflag, .../*mode_t mode */);
int openat(int fd, const char *path, int oflag, .../*mode_t mode */);
...表示最后一个参数, 表明余下的参数数量和类型均不定. 对与open只有在创建文件时才会使用到最后的参数;
path表示要打开或者创建的文件名. oflag表示此函数的多个选项, 使用下列一个或对个常量进行或运算构成oflag参数, 均被定义在<fcntl.h>头文件中:
oflag参数:
// 下面五个只能选一个
O_RDONLY  read only;
O_WRONLY  write only;
O_RDWR    write and read;
O_EXEC    only execute;
O_SEARCH  only search; // linux don't exist.
// 下面的可以任性选择
O_APPEND     每次写时追加到末尾
O_CLOEXEC    把FD_CLOEXEC常量设置为文件描述标识符
O_CREAT      若此文件不存在则创建, 创建时将使用最后的参数指明新文件的访问权限
O_DIRECTORY  如果path引用得到不是目录就会出错
O_EXCL       如果同时指定了O_CREAT而文件已经存在则会报错, 用于测试文件是否已经存在. 使得测试和创建成为一个原子操作
O_NOCTTY     如果path引用的是终端设备, 则不将该设备分配作为此进程的控制终端
O_NOFOLLOW   如果path引用的是一个符号连接将会出错
O_NONBLOCK   如果path引用的是一个FIFO, 一个块特殊文件或者一个字符特殊文件, 则此选项为文本的本次打开和后续I/O为非阻塞式.
O_SYNC       使每次write等待物理I/O操作完成, 包括由该write引起的文件属性更新.
O_TRUNC      如果此文件存在, 且为写或读写方式打开,则将长度截断为0
O_TTY_INIT   如果打开一个未打开的终端设备, 设置非标准termios参数值, 使其符合Single UNIX Specificatation
O_DSYNC      使每次write要等待物理I/O操作完成, 但是如果改写操作并不影响读刚写入的数据,则不需要等待文件属性被更改.
O_RSYNC      使每一个以文件描述符作为参数进行的read操作等待, 直到所有对文件同一部分挂起的写操作完成.

```

fd参数将open与openat函数区分开, 主要有三种情况:

1. path为绝对路径, fd被忽略, open与openat一致.
2. path指定相对路径, fd指出相对路径名在文件系统中的开始地址. fd参数通过打开相对路径名所在的目录名来获取.
3. path参数指定了相对路径名, fd参数具有特殊值AT_FDCWD, 在这种情况下, 路径名在当前工作目录中获取, openat函数在操作上与open类似.

## 函数creat

creat创建一个新文件, 原型:

```c++
#include<fcntl.h>
int creat(const char *path, mode_t mode);
// 成功返回只写打开的文件描述符, 错误返回-1;

```

## 函数close

close关闭一个打开的文件. 原型:

```c++
#include<fcntl.h>
int close(int fd);

```

关闭文件还会释放加在该文件上的记录锁.

## 函数lseek

每一个打开的文件都有一个与其相关联的"当前字节偏移量". 通常是一个非负整数, 用以度量从文件开始处计算的字节数. 读写操作都是从当前文件偏移量处开始. 当打开一个文件时, 除显示使用O_APPEND选项, 否则偏移量为零. 可以通过lseek设置偏移量. 原型:

```c++
#include<unistd.h>
off_t lseek(int fd, off_t offset, int whence);
// 成功返回新的文件偏移量, 否则返回-1;
参数offset与whence有关:
1 当whence是SEEK_SET, 则偏移量设置为从文件开始到offset个字节处.
2 当whence是SEEK_CUR, 偏移量为当前值加上offset当前位置.
3 当whence是SEEK_END, 则偏移量设置为文件长度加上offset.

```

文件偏移量可以大于文件的当前长度, 此时对该文件的下一次写操作将加长该文件, 并在该文件中构成一个空洞, 位于文件中但没有写过的字节都被度为0, 空洞并不占用磁盘空间.

## 函数read

read从打开文件读取数据, 原型:

```c++
#include<unistd.h>
ssize_t read(int fd, void *buf, size_t nbytes);
//返回类型, 读到的字节数, 若已经达到文件末尾则返回0; 若出错返回-1;

```

## 函数write

write函数向一个打开的文件写数据, 原型:

```c++
#include<unistd.h>
ssize_t write(int fd, const void *buf, size_t nbytes);

```

## 文件共享

UNIX系统支持不同进程共享打开的文件. 内核使用三种结构表示打开的文件, 他们的关系决定了在文件共享方面一个进程对另一个进程可能产生的影响.

(1) 每个进程在进程表中都有一个记录项, 记录项中包含一张打开文件描述符表, 与文件表项向关联的是:

1. 文件描述符标志
2. 指向一个文件表项的指针.

(2) 内核为所有打开文件维持一张文件表. 每个文件表包含:

1. 文件状态标志(读, 写, 添加, 同步, 阻塞等)
2. 当前文件偏移量
3. 指向该文件v节点表项的指针

(3) 每个打开文件(或设备)都有一个v节点结构. v节点包含了文件类型和对此文件进行各种操作函数的指针. 大多数文件, v节点还包含该文件i节点的指针.

下图展示了三者之间的关系:

![结构](https://s2.ax1x.com/2019/06/24/ZA0K9s.png)

如果两个独立进程各自打开同一文件, 则有下图关系:

![打开同一文件](https://s2.ax1x.com/2019/06/24/ZA0M3n.png)

注意：<font color = red>可能有多个文件描述符项指向同一文件表项</font>.这会在dup或fork函数后产生. 注意: 文件描述符标志与文件状态标志在作用范围上的区别: 前者只用于一个进程的一个描述符, 而后者则应用与指向该文件表项的任意进程中的所有描述符.(有点类型指针和底层存储的区别)



## 原子操作

原子操作指的是由多步组成的一个操作. 如果该操作原子的执行, 则要么执行完所有的步骤, 要么一步也不执行, 不可能只执行所以步骤中的一个子集. 任何要求多于一个函数调用的操作都不是原子操作, 因为在两个函数调用之间, 内核有可能会临时挂起进程.

## 函数dup和dup2

dup和dup2都用来赋值一个现有的文件描述符. 原型:

```c++
#include<unistd.h>
int dup(int fd);
int dup2(int fd, int fd2);
//函数返回值, 若成功返回新的文件描述符, 否则返回-1

```

由dup返回的新文件描述符一定是当前可用文件描述符中最小数值. dup2可以通过fd2指定新的文件描述符的值. 如果fd2已经打开则先关闭. 如果fd2与fd一致,则不关闭直接返回fd2.  执行dup函数后可能的结果如图:

![dup](https://s2.ax1x.com/2019/06/24/ZAOi2d.png)

## 函数sync, fsync, fdatasync

传统的UNIX系统实现内核中设有缓冲区高速缓存或页高速缓存. 向文件写入数据时, 内核首先将数据复制到缓冲区, 然后排入队列. 当内核需要重用缓冲区来存放其他磁盘块数据时, 它会把延迟写数据块写入磁盘, 为了保证磁盘上实际文件与缓冲区内容一致, 可以使用sync, fsync, fdatasync函数. 原型:

```c++
#include<unistd.h>
int fsync(int fd);
int fdatasync(int fd);
void sync(void);

```

sync将所有修改过的缓冲区排入写队列, 然后返回, 并不等待实际写磁盘结束.

fsync函数只对指定文件描述符起作用, 并且等待写磁盘结束才返回.

fdatasync与fsync类似, 不过只影响文件的数据部分, 而fsync除了数据部分还会同步更新文件属性.

## 函数fcntl

fcntl函数可以改变已经打开文件的属性. 原型:

```c++
#include<fcntl.h>
int fcntl(int fd, int cmd, .../* int arg */);

```

fcntl函数有以下5中功能:

1. 赋值一个已有的文件描述符(cmd = F_DUPFD或F_DUPFD_CLOEXEC).
2. 获取/设置文件描述符标志(cmd = F_GETFD或F_SETFD).
3. 获取/设置文件状态标志(cmd = F_GETFL或F_SETFL).
4. 获取/设置异步I/O所有权(cmd = F_GETOWN或F_SETOWN).
5. 获取/设置记录锁(cmd = F_GETLK, F_SETLK或F_SETLKW).

下面对上述参数进行解释:

<table>
    <tr>
        <td>F_DUPFD</td>
        <td>复制文件描述符fd. 新文件描述符作为返回值. 新描述符与fd共享同一文件表, 但新文件描述符有自己的一套文件描述符标志,其中FD_CLOEXEC文件描述标志被清除(表示该描述符在exec时任有效)</td>
    </tr>
    <tr>
        <td>F_DUPFD_CLOEXEC</td>
        <td>复制文件描述符, 设置与新描述符关联的FD_CLOEXEC文件描述符标志的值</td>
    </tr>
    <tr>
        <td>F_GETFD</td>
        <td>对应于fd的文件描述符标志作为函数返回值. 当前只定义了一个文件描述符标志FD_CLOEXEC. 由于五个基本的访问方式标志不是各占一位, 因此我们需要使用屏蔽字O_ACCMODE取得访问标志位, 然后将结果与五个值对比.</td>
    </tr>
    <tr>
        <td>F_SETFD</td>
        <td>对于fd设置文件描述符标志, 新标志值按第三个参数设置</td>
    </tr>
    <tr>
        <td>F_GETFL</td>
        <td>对应fd的文件状态标志作为函数返回值</td>
    </tr>
    <tr>
        <td>F_SETFL</td>
        <td>将文件状态标志设为第三个参数的值</td>
    </tr>
    <tr>
        <td>F_GETOWN</td>
        <td>获取当前接收SIGIO和SIGURG信号进程ID或进程组ID</td>
    </tr>
    <tr>
        <td>F_SETOWN</td>
        <td>设置接收SIGIO和SIGURG信号的进程ID或进程组ID</td>
    </tr>
</table>

fcntl返回值与命令有关, 如果出错则都返回-1, 否则返回某个其它值.

下表列出了文件状态标志(与open时描述的一样)：

![fd](https://s2.ax1x.com/2020/02/18/3ichM8.png)

例: 查看文件状态标志

```c
#include"apue.h"
#include<fcntl.h>
int main(int argc,char *argv[])
{
    int val;
    if(argc!=2)
        err_quit("usage: a.out<descriptor#>");
    if((val = fcntl(atoi(argv[1]), F_GETFL, 0))<0)
        err_sys("fcntl error for fd %d", atoi(argv[1]));
    switch(val & O_ACCMODE)
    {
        case O_RDONLY:
            printf("read only");
            break;
        case O_WRONLY:
            printf("write only");
            break;
        case O_RDWR:
            printf("read write");
            break;
        default:
            err_dump("unknow access mode");
    }
    if(val & O_APPEND)
        printf(", append");
    if(val & O_NONBLOCK)
        printf(", nonblock");
    if(val & O_SYNC)
        printf(", sunchronous write");
# if !defined(_POSIX_C_SOURCE) && defined(O_FSYNC) && (O_FSYNC != O_SYNC)
    if(val & O_FSYNC)
        printf(", synchronous write");
#endif
    putchar('\n');
    exit(0);
}

//output:
chst@wyk-GL63:~/study_file/unix编程$ ./getfl.o 0 < /dev/tty
read only
chst@wyk-GL63:~/study_file/unix编程$ ./getfl.o 1 > temp.foo
chst@wyk-GL63:~/study_file/unix编程$ cat temp.foo
write only
chst@wyk-GL63:~/study_file/unix编程$ ./getfl.o 2 2>>temp.foo
write only, append
chst@wyk-GL63:~/study_file/unix编程$ ./getfl.o 5 5<>temp.foo
read write

```

子句`5<>temp.foo`表示在文件描述符5上打开文件temp.foo以供读写。

## 函数ioctl

函数ioctl是I/O操作的杂物箱：

```c
#include<unistd.h> /* system v */
#include<sys/icotl.h> /* BSD and linux */ 

int ioctl(int fd, int request, ...);
//出错返回-1， 成功返回其他值
```

下表总结FreeBSD支持的通用ioctl命令的一些类别：

![icotl](https://s2.ax1x.com/2020/02/18/3ig3SP.png)

## /dev/fd

较新的系统提供/dev/fd目录, 其目录项是名0, 1, 2等的文件. 打开文件/dev/fd/n等效于复制描述符n. 

例:

```c
fd = open("/dev/fd/o", mode);
==
fd = dup(0);

```

/dev/fd文件主要用于shell, 它允许使用路径名作为调用参数的程序. 例如cat将'-'解释为标准输入.

```shell
filter file2 | cat file1 - file3 | lpr
==
filer file2 | cat file1 /dev/fd/0 file3 | lpr
// |表示通道, 即前一个命令的输出作为下一个命令的输入

```

这里'-'别替换为filter file2的输出。

# 第四章 文件和目录

## 函数stat, fstat, fstatat和lstat

函数原型：

```c
#include<sys/stat.h>
int stat(const char *restrict pathname, struct stat *restrict buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);

```

　　一旦给出pathname，stat函数返回与此命名文件有关的信息结构。fstat获得已在描述符fd中打开的文件有关信息。lstat与stat类似，但当命名文件是一个符号链接时，lstat返回该链接对应的有关信息而不是链接指向的文件。

　　fstatat返回相对于当前打开目录（由fd参数指向）的路径名的文件统计信息。flag参数控制是否跟随着一个符号链接。当AT_SYLINK_NOFOLLOW被置位时，不跟随符号链接，只返回符号链接本身的文件信息。否则，在默认情况下，返回的是符号链接指向的文件对于的信息。如果fd参数是AT_FDCWD，并且pathname是一个相对路径，则会计算相对于当前目录的pathname参数，返回对应文件信息。如果pathname是绝对路径，fd会被忽略。

　　buf是一个指针，指向我们必须提供的结构。函数来填充内容。结构的基本形式为：

```c
struct stat{
    mode_t st_mode; //文件类型和mode(权限)
    ino_t  st_ino; //i节点数量
    dev_t  st_dev;//设备数量
    dev_t st_rdev;//对于特殊文件来说设备数量
    nlink_t st_nlink;//链接数量
    uid_t st_uid;//拥有者用户ｉｄ
    gid_t st_gid;//拥有者组ｉｄ
    off_t st_size;//大小(bytes)
    struct timespec st_atime;//最后一次访问时间
    struct timespec st_mtime;//最后一次更改时间
    struct timespec st_time;//最后一次文件stat更改时间。
    blksize_t st_blksize;//best I/O block size
    blkcnt_t st_blocks;//number of disk bocks allocated
}

```

timespec结构类型按照秒和纳秒定义了时间，至少包含以下两个字段：

```c
time_t tv_sec;
long tv_nsec;
```

stat中使用到的类型大都属于基本统计类型。使用前

```c
include<sys/types.h>
```



## 文件类型





|    文件类型    |                             说明                             |
| :------------: | :----------------------------------------------------------: |
|    普通文件    |               最常见的文件，包含某种形式数据。               |
|    目录文件    | 包含了其他文件的名字以及指向这些文件有关信息的指针。对于一个目录文件具有度权限的任一进程都能读取目录内容，但只有内核能够写目录文件。 |
|   块特殊文件   |   提供对设备带缓冲的访问，每次访问以以固定长队为单位进行。   |
|  字符特殊文件  | 提供对设备不带缓冲的访问，每次访问长度可变。系统中的所以设备要么是字符特殊文件，要么是块特殊文件。 |
|   FIFO(管道)   |                       用于进程间通讯。                       |
| 套接字(socket) |                     用于进程间网路通信。                     |
|   符号连接。   |                       指向另一个文件。                       |

文件类型信息包含在stat结构中的st_mode中。使用如下宏来确定文件类型：参数均为stat结果中的st_mode。

| S_ISREG()  |   普通文件   |
| :--------: | :----------: |
| S_ISDIR()  |   目录文件   |
| S_ISCHR()  | 字符特殊文件 |
| S_ISBLK()  |  块特殊文件  |
| S_ISFIFO() |     管道     |
| S_ISLNK()  |   符号连接   |
| S_ISSOCK() |    套接字    |



## 设置用户ID和组ID

一个进程关联的ID有六个或更多。

| 实际用户ID/实际组ID             | 我们实际上是谁       |
| ------------------------------- | -------------------- |
| 有效用户ID/有效组ID/附属组ID    | 用于文件访问权限检查 |
| 保存的设置用户ID/保存的设置组ID | 由exec函数保存       |

通常有效用户ID等于实际用户ID，有效组ID等于实际组ID。所以者和所有者组由stat中st_uid和st_gid指定。

实际用户ID和实际组ID表示我们究竟是谁.这两个字段在登录时取自口令文件的登录项(应该是由执行该文件的用户决定).

当执行一个程序文件时,通常进程的有效用户ID就是实际用户ID,有效组ID通常是实际组ID. 但我们可以在文件模式字(st_mode)中设置一个标志,其含义是"当执行次文件时,将进程有有效用户ID设置为文件所有者的用户ID",与次类似,在文件模式字中,可以设置另一位,它将执行文件的进程的有效组ID设置为文件所有者组ID.这两个位分别为设置用户ID位(set-user-id)和设置组ID位(set-group-ID).

## <a id="文件访问权限 ">文件访问权限</a>

st_mode值也包含了对文件的访问权限.这里的文件是指上述所有七种文件.

每个文件有几个访问位权限,可以分为三类:

|        st_mode屏蔽        |      含义      |
| :-----------------------: | :------------: |
| S_IRUSER/S_IWUSER/S_IXUSE | 用户读/写/执行 |
|  S_IRGRP/S_IWGRP/S_IXGRP  |  组读/写/执行  |
|  S_IROTH/S_IWORT/S_IXOTH  | 其他读/写/执行 |

用户指的是所有者.chomd命令用来修改这九个权限.该命令允许我们用u表示用户,用g表示组,用o表示其他.

使用规则:

### 一

当我们使用名字打开一个文件时,我们对该名字中包含的每一个目录,包括它可能隐藏的当前的工作目录都应该具有执行权限.这也是为何对目录执行权限位通常被称为搜索位.

注意:对于目录的读权限和执行权限的意义是不同的.读权限允许我们读目录,获得在该目录下所以文件名的列表.当一个目录是我们要访问文件路径名的一部分时,对该目录的执行权限使得我们可以通过该目录.

### 二

对于一个文件的读权限决定了我们能否打开文件进行读操作.

### 三

对于一个文件的写权限决定了我们能否打开文件进行写操作.

### 四

为了在open函数中对一个文件指定O_TRUNC标志,必须对该文件具有写权限.

### 五

为了在一个目录下创建一个新文件,需要对该目录具有写和执行权限.

### 六

为了删除一个文件,需要对该文件所在目录具有写和执行权限而不必对文件本身具有相应权限.

### 七

如果使用七个exec函数执行某个文件,需要对该文件具有执行权限.

进程每次<font color=red>打开,创建,删除</font>一个文件时,内核就会进行文件访问权限测试,而这种测试可能涉及文件所有者(st_uid和st_gid),进程的有效ID(有效用户ID和有效组ID)已经进程的附属组ID.<font color=red>两个所有者ID是文件的性质,而两个有效ID和附属ID则是进程的性质</font>. 内核测试具体如下:

1. 若进程有效ID是0(超级用户),则允许访问.
2. 若进程的有效用户ID等于文件所有者ID(即进程拥有此文件),则判断所有者是否具有进程将要操作的权限,如果没有则拒绝.
3. 若进程的有效组ID或进程的附属组ID之一等于文件的组ID,那么组适当的权限被置位则允许访问.
4. 若其他用户适当的访问权限被置位,则允许访问.

按顺序执行这四步.需要注意,这四步是截断的,即一个条件被满足就不会继续向下进行.

## 新文件和目录的所有权

新文件的用户ID设置为进程的有效用户ID,新文件的组ID可以是进程的有效组ID,也可以是它所在目录的组ID.

## 函数access和faccess

access和faccess是按照进程实际用户ID和实际组ID进行权限测试的.

函数原型:

```c
#include<unistd.d>
int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);
//成功返回0，失败返回-1
```

当要测试文件是否存在时,mode是F_OK,否则mode是下面常量按位或.

| mode |              |
| :--: | :----------: |
| R_OK |  测试读权限  |
| W_OK |  测试写权限  |
| X_OK | 测试执行权限 |

当pathname是绝对路径和当fd是AT_FDCWD而pathname是相对路径时,faccessat与acess是相同的.

flag参数可以用于改变faccessat行为,如果flag设置为AT_ACCESS访问检测用的是有效用户ID和有效进程ID.

## 例:acess.cpp

```c
#include"apue.h"
#include<fcntl.h>
int main(int argc, char *argv[])
{
    if(argc!=2)
    {
        err_quit("usage: a.out<pathname>");
    }
    if(access(argv[1], R_OK)<0)
    {
        err_ret("access error for%s",argv[1]);
    }
    else
    {
        printf("read access ok\n");
    }
    if(open(argv[1], O_RDONLY)<0)
    {
        err_ret("open error for %s",argv[1]);
    }
    else
    {
        printf("open for reading ok\n");
    }
    exit(0);
}

```

```
$g++ access.cpp -o access.o -lapue
$ls -l access.o
-rwxrwxr-x 1 chst chst 13328 Sep  8 15:40 access.o
$./access.o access.cpp
read access ok
open for reading ok
$ls -l /etc/shadow
-rw-r----- 1 root shadow 1395 May  6 19:00 /etc/shadow
$ ./access.o /etc/shadow
access error for/etc/shadow: Permission denied
open error for /etc/shadow: Permission denied
$ sudo chown root access.o   //更改文件用户为超级用户
[sudo] password for chst: 
$ sudo chmod u+s access.o //打开设置用户位,即使得进程的有效ID等于文件的用户ID,即超级用户.
$ ls -l access.o
-rwsrwxr-x 1 root chst 13328 Sep  8 15:40 access.o //这里s表示设置用户位被置位
$exit //退出超级用户
$ ./access.o /etc/shadow
access error for/etc/shadow: Permission denied
open for reading ok

```

这里解释一下最后的输出,在执行access.o时,我们是以普通用户进行的,此时进程的实际ID即为普通用户ID,但由于设置用户位被置位,此时进程的有效用户ID为超级用户ID,因为在打开文件时,是使用有效用户来进行判断的,因此此时可以打开文件,但是我们实际用户ID是普通用户,因此使用access进行检查时,会显示权限错误,因此尽管我们可以打开文件,但可以确定实际用户不能正常读指定文件.但该程序现在是可以正常读取指定文件的.

## 函数umask(文件模式创建屏蔽字)

umask函数为进程设置文件模式创建屏蔽字,并返回之前的值.

函数原型:

```
#include<sys/stat.h>
mode_t umask(mode_t cmask);

```

其中cmask为之前<a href="#文件访问权限 ">表格</a>里面的9个常量或的结果.在进程创建一个新文件和新目录时,就一定会使用文件模式创建屏蔽字.(open和creat函数都有参数mode,其就是用来指定新文件的访问权限).

例:

```
#include"apue.h"
#include<fcntl.h>
#define RWRWRW (S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH)
int main()
{
    umask(0);
    if(creat("foo", RWRWRW)<0)
    {
        err_sys("creat foo error!\n");
    }
    umask(S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
    if((creat("bar", RWRWRW)<0))
    {
        err_sys("creat bar error!\n");
    }
    exit(0);
}

```

执行

```
chst@wyk-GL63:~/study_file/unix编程$ umask //查看当前屏蔽字, 0002表示只有其他写被屏蔽
0002
$ ./umask.o
$ ls -l foo bar
-rw------- 1 chst chst 0 Sep  8 17:34 bar
-rw-rw-rw- 1 chst chst 0 Sep  8 17:34 foo

```

更改环境文件创建屏蔽字:

```
$umask -S //打印符号格式
u=rwx,g=rwx,o=rx
$umask 0027 //更改屏蔽字,屏蔽用户组读和其他的所以权限
$umask -S
u=rwx,g=rx,o=

```

## 函数chmod,fchmod和fchmodat

这三个函数使得我们可以更改现有文件的访问权限.

函数原型:

```
#include<sys/stat.h>
int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag)

```

chmod操作指定文件,fchmod操作打开的文件, 当pathname为绝对路径或fd参数为AT_FDCWD而pathname为相对路径时,fchmodat与chmod一样. flag参数用来改变fchmodat行为,当设置了AT_SYMLIN_NOFOLLOW标志时,fchmodat不会跟随符号链接.

为了改变一个文件的权限位,进程的有效用户ID必须等于文件所有者ID,或者进程拥有超级用户权限.参数mode是如下常量取与或:

|  mode   |        说明        |
| :-----: | :----------------: |
| S_ISUID |  执行时设置用户ID  |
| S_ISGID | 执行时设置用户组ID |
| S_ISVTX |                    |

| S_IRWXU | 用户(所有者)读写和执行 |
| :-----: | :--------------------: |
| S_IRUSR |         用户读         |
| S_IWUSR |         用户写         |
| S_IXUSR |        用户执行        |

| S_IRWXG | 用户组读写和执行 |
| :-----: | :--------------: |
| S_IRGRP |     用户组读     |
| S_IWGRP |     用户组写     |
| S_IXGRP |    用户组执行    |

| S_IRWXO | 其他读写和执行 |
| :-----: | :------------: |
| S_IROTH |     其他读     |
| S_IWOTH |     其他写     |
| S_IXOTH |    其他执行    |

命令行添加设置用户ID和设置组ID方式为:

```
$chmod u+s filename
$chmod g+s filename
-rwSrwSrw- //这里S表示设置ID开启.

```

## 函数chown,fchown,fchownat和lchown

这几个函数是用来更改文件用户ID和组ID的.函数原型:

```
#include<unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);

```

当owner或group任意一个是-1,则对应的ID不变.

除了所引用的文件是符号链接以外,这4个函数操作类似.在符号连接下,lchown与fchownat(设置了AT_SYMLINK_NOFOLLOW)更改符号链接本身而不是连接指向的文件.

当pathname为绝对路径或fd参数为AT_FDCWD而pathname为相对路径时,fchownat与chmod一样. flag参数用来改变fchmodat行为,当设置了AT_SYMLIN_NOFOLLOW标志时,fchmodat不会跟随符号链接.

## 文件系统

在说明文件链接前先介绍一下文件系统,这里主要介绍的是UFS系统.

我们把一个磁盘分成一个或多个分区,每个分区都包含一个文件系统.细节见下图:

![文件系统](https://s2.ax1x.com/2019/09/09/ntqGge.png)

仔细观察柱面`i`节点和数据块的部分,会存在下图的关系:

![i节点](https://s2.ax1x.com/2019/09/09/ntqJjH.png)

注意细节:

1. 图中有两个目录项指向同一个`i`节点.每个`i`节点都有一个连接计数,其值是指向该`i`节点的<font color = red>目录项数</font>.只有当链接计数等于0的时候才可以删除文件(释放该文件所占用的数据块).在stat中,链接计数包含在st_nlink中,基本数据类型是nlink_t.这种链接为硬链接.
2. 还有一种链接为符号链接.符号链接链接文件的实际内容(在数据块中)包含了该符号链接所执向的文件的名字.
3. i节点包含了文件的所以信息:文件类型,访问权限,文件长度和指向文件数据块的指针等. stat中大多数内容取自i节点,只有文件名和i节点编号放在目录项中.
4. 当在不更换文件系统的情况下为一个文件重命名时,该文件实际内容并未移动,只需要构造一个指向当前i节点的新目录项,并删除老目录项即可,连接计数不会改变.

目录文件的计数说明:

使用`mkdir testdir`创建一个新目录时,结果如下:

![创建目录](https://s2.ax1x.com/2019/09/09/ntLpxe.png)

该图显示的展现出了`.`和`..`.

任何一个叶目录(不包含目录的目录)连接计数均为2.数值2来自于命名该目录的目录项和在该目录中的`.`.编号为1267的i节点,链接计数大于等于3.这是由于,一个是命名它的目录项,一个是自己目录下的`.`,还有则是新建的目录testdir中的`..`(目录下的目录中的`..`都是对父目录的硬链接,会增加i节点计数).

## 函数link, linkat, unlink和remove

使用link和linkat函数创建一个指向当前文件的链接.函数原型:

```
#include<unistd.h>
int link(const char *existingpath, const char *newpath);
int linkat(int efd, const char *existingpath, int nfg, const char *newpath, int flag);
// 函数返回值0正常,-1出错.

```

两个函数创建一个新目录项newpath,它引用现有文件existingpath. 创建新目录项和增加链接计数应该是原子操作.

为了删除一个现在的目录项, 可以使用unlink和unlinkat函数:

```
#include<unistd.h>
int unlink(const char *pathname);
int unlink(int fd, const char *pathname, int flag);

```

两个函数删除目录项,并将有pathname所引用文件的链接计数减一.为了解除对文件的链接,我们必须对该目录具有写和执行权限.只有当链接计数达到0的时候该文件内容才会被删除.

注意:只要有进程打开了该文件,其内容也不会被删除.关闭一个文件时,内核首先检测打开该文件的进程数目,如果这个数值等于0,再去查看链接计数,如果链接计数也达到0,才删除文件.利用该特性,unlink常常被用来确保在程序崩溃的情况下删除临时创建的文件.进程使用open或creat创建一个文件,然后立即调用unlink,由于该文件仍旧是打开的,因此不会被立即删除,只有当进程终止时,文件才会被删除.

fd和pathname用来确定路径的.flag给出一种方式,当AT_REMOVEDIR被设置时,unlinkat函数类似与rmdir一样删除目录.

如果pathname给出的是符号链接,则只能删除符号链接本身,当是符号链接时,没有能够直接删除符号链接所引用的文件的函数.

可以使用remove解除对一个文件或目录的链接. 对于目录,remove与rmdir类型,对于文件,remove与unlink类型.

函数原型:

```
#include<stdio.h>
int remove(const char *pathname);

```

例:

```
#include"apue.h"
#include<fcntl.h>
int main()
{
    if(open("temp.foo", O_RDWR)<0)
        err_sys("open eror");
    if(unlink("temp.foo")<0)
    {
        err_sys("unlink error");
    }
    printf("file unlinked\n");
    sleep(15);
    printf("done\n");
    exit(0);
}

```

运行:

```shell
chst@wyk-GL63:~/study_file/unix编程$ ls -l temp.foo
-rw-rw-r-- 1 chst chst 8 Sep 10 23:59 temp.foo
chst@wyk-GL63:~/study_file/unix编程$ ./unlink.o & //后台运行程序
[1] 9597
chst@wyk-GL63:~/study_file/unix编程$ file unlinked
done
 ls -l temp.foo
ls: cannot access 'temp.foo': No such file or directory //目录项已被删除(数据块未被删除)
[1]+  Done                    ./unlink.o
chst@wyk-GL63:~/study_file/unix编程$

```



## 函数rename和renameat

文件或目录可以使用rename和renameat来命名,函数原型:

```
#include<stdio.h>
int rename(const char *oldname, const char *newname);
int renameat(int oldfd, const char *oldname, int newfd, const char *newname);

```

## 符号链接

符号链接是对一个文件的间接指针,它与上一节的硬链接(直接指向`i`节点)不同,符号链接指向的是目录项.这是为了规避硬链接的一些限制:

1. 硬链接通常要求链接和文件位于同一文件系统下.
2. 只有超级用户才能创建指向目录的硬链接.

对符号链接以及它指向何种对象并无任何限制,任何用户都可以创建指向目录的符号链接.符号链接是为了将一个文件或整个目录移到系统的另一个位置.

使用符号链接可能造成循环:

```
chst@wyk-GL63:~/study_file/unix编程$ mkdir loop
chst@wyk-GL63:~/study_file/unix编程$ touch loop/a  //创建一个空文件a
chst@wyk-GL63:~/study_file/unix编程$ ln -s ../loop loop/testdif //在目录下创建一个符号链接指向目录本身
chst@wyk-GL63:~/study_file/unix编程$ ls -l loop
total 0
-rw-rw-r-- 1 chst chst 0 Sep 11 00:37 a
lrwxrwxrwx 1 chst chst 7 Sep 11 00:37 testdif -> ../loop

```

此时就会造成循环,因为目录下的符号链接指向目录本身.

![Screenshot from 2019-09-11 00-40-23](/home/chst/Pictures/Screenshot from 2019-09-11 00-40-23.png)

此时使用Solares中的ftw以降序遍历文件结构,打印每一个遇到的路径名,结果为:

![Screenshot from 2019-09-11 00-40-42](/home/chst/Pictures/Screenshot from 2019-09-11 00-40-42.png)

这个循环是十分容易消除的,因为unlink不跟随符号链接,可以使用unlink文件foo/testdir.但如果创建一个构成这样的硬链接,就很难消除(难吗?直接删除testdir不就好了?).因此link不允许一般用户(linux下超级用户也不行)构造指向目录的链接.

## 创建和读取符号连接

可以使用symlink或symlinkat函数创建一个符号链接.函数原型:

```c
#include<unistd.h>
int symlink(const char *actualpath, const char *sympath);
int symlinkat(const char *actualpath,int fd,const char *sympath);
//两个函数成功返回0,出错返回-1
// // 创建符号链接$ln -s actualpath sympath

```

函数创建一个指向actualpath的新目录项sympath.并不要求actualpath已经存在,且两个不必在同一个文件系统中.

open函数会打开链接指向内容,因此需要一种方式打开链接本身,并读该链接中的名字.函数readlink和readlinkat提供这一功能.函数原型:

```c
#include<unistd.h>
ssize_t readlink(const char *restatrict pathname,char *restrict buf,size_t bufsize);
ssize_t readlinkat(int fd,const char *pathname,char *restrict buf, size_t bufsize);
//成功返回buf中读取字节数,否则返回-1

```

两个函数组合了open,read,和close的所有操作.buf返回的符号链接不以null为结尾.

## 文件时间

每个文件维护三个时间字段:

|  字段   |          说明          |    例子     | ls选项 |
| :-----: | :--------------------: | :---------: | :----: |
| st_atim |  文件数据最后访问时间  |    read     |   -u   |
| st_mtim | 文件数据的最后修改时间 |    write    |  默认  |
| st_ctim |  i节点最后的更该时间   | chmod,chown |   -c   |

注意: 修改时间(st_mtim)与状态更改时间(st_ctim)的区别.修改时间是指文件内容修改时间(数据块),状态更改时间是该文件i节点最后被修改时间.状态更改时间包括更改访问权限,用户ID,连接计数.

## 函数futimens,utimensat,utimes函数

函数原型:

```c
#include<sys/stat.h>
int futimens(int fd, const struct timespec times[2]);
int utimenstat(int fd,const char *path,const struct timespec times[2], int flag);

```

这两个函数用于更改文件访问和修改时间.times数组参数第一个元素包含访问时间,第二个元素包含修改时间,均是时间戳.

时间戳按照下列四种方式之一进行指定:

1. 如果times参数是空指针,则访问时间和修改时间都设置为当前时间.
2. 如果times指向两个timespec结构的数组,任一数组元素的tv_nesc字段值为UTIME_NOW,相应的时间戳就设置为当前时间,忽略相应的tv_sec字段.
3. 如果times指向两个timespec结构的数组,任一数组元素的tv_nesc字段值为UTIME_OMIT,相应的时间戳保持不变,忽略相应的tv_sec字段.
4. 如果times指向两个timespec结构的数组,任一数组元素的tv_nesc字段值即不为UTIME_OMIT也不是UTIME_NOW,相应的时间戳设置为对应的两个字段值.

utims对目录名时间进行操作,函数原型:

```c
#include<sys/time.h>
int utimes(const char *pathname, const struct timeval times[2]);
struct timeval{
time_t tv_sec;
long tv_usec;//毫秒
}

```

我们不能更改状态更改时间st_ctim指定一个值,因为调用这三个函数时,此字段会被自动更新.

## 函数mkdir,mkdirat和rmdir

用mkdir,mkdirat,用rmdir函数删除目录.函数原型:

```c
#include<sys/stat.h>
int mkdir(const char *pathname,mode_t mode);
int mkdirat(int fd,const char *pathname, mode_t mode);

```

两个函数创建一个新的空目录.其中`.`和`..`被自动创建.所指定的文件访问权限mode由进程的文件模式创建屏蔽字修改.常见错误是指定与文件一样的mode(只指定读写).对于目录来说,者少应该添加执行权限来允许访问目录中的文件名.

使用rmdir函数删除一个空目录:

```c
#include<unsid.h>
int rmdir(const char *pathname);

```

如果调用该命令使得目录的链接计数达到0,并且也没有进程打开该目录,则释放次目录占用的空间.如果此时有进程打开该目录,则在进程结束前删除最后一个链接及`.`和`..`,在此目录下不能创建文件,但在最后一个打开该目录的进程结束前不会释放次目录.

## 读目录

对某个目录具有访问权限的任意用户都可以读目录,但只有内核可以写目录.一个目录的写权限决定了在该目录下能否创建新文件以及删除文件,它们不代表能否写目录本身.

相关函数:

```c
#include<dirent.h>
DIR *opendir(const char *pathname);
DIR *fdopendir(int fd);
//成功返回指针,出错返回NULL

struct dirent *readdir(DIR, *dp);
//成功返回指针,出错返回NULL

void rewinddir(DIR *dp);

int closedir(DIR *dp);
//成功返回0,错误返回-1

long telldir(DIR *dp);
//返回与dp关联的目录中的当前位置

void seekdir(DIR *dp,long loc);

```

例:

```c
#include"apue.h"
#include<dirent.h>
#include<limits.h>

//一个静态函数,传参为函数指针
static int myftw(char *, int (*Myfunc)(const char *, const struct stat *, int));
static int dopath(int (*Myfunc)(const char *, const struct stat *, int));
static long nreg, ndir, nblk, nchr, nfifo, nslink, nsock, ntot;
static int Myfunc(const char *pathname, const struct stat *statprt, int type);

int main(int argc, char *argv[])
{
    int ret;
    if(argc != 2)
    {
        err_quit("usage: ftw <starting-pathname>");
    }
    ret = myftw(argv[1], Myfunc);
    ntot = nreg + ndir + nblk + nchr + nfifo + nslink + nsock;
    // 避免除以0
    if(ntot == 0)
    {
        ntot = 1; 
    }
    printf("regular file   = %7ld, %5.2f%%\n", nreg, nreg*100.0/ntot);
    printf("directories    = %7ld, %5.2f%%\n", ndir, ndir*100.0/ntot);
    printf("block special  = %7ld, %5.2f%%\n", nblk, nblk*100.0/ntot);
    printf("char special   = %7ld, %5.2f%%\n", nchr, nchr*100.0/ntot);
    printf("FIFOs          = %7ld, %5.2f%%\n", nfifo, nfifo*100.0/ntot);
    printf("symbolic link  = %7ld, %5.2f%%\n", nslink, nslink*100.0/ntot);
    printf("sockets        = %7ld, %5.2f%%\n", nsock, nsock*100.0/ntot);
    exit(ret);
}

#define FTW_F 1 //普通文件
#define FTW_D 2 //目录
#define FTW_DNR 3 //无法读取的目录
#define FTW_NS 4 //没有stat的文件

static char *fullpath; //文件绝对路径
static size_t pathlen;

static int myftw(char *pathname, int (*Myfunc)(const char *, const struct stat *, int))
{
    fullpath = (char*)malloc(PATH_MAX+1); //分配PATH_MAX+1字节
    if(pathlen<=strlen(pathname))
    {
        pathlen = strlen(pathname) * 2;
        fullpath = (char*)malloc(pathlen);
    }
    strcpy(fullpath, pathname);
    printf("fullpath1 = %s\n", fullpath);
    return dopath(Myfunc);
}

//深度优先搜索DFS,遍历每个文件和目录
static int dopath(int (*Myfunc)(const char *, const struct stat *, int))
{
    struct stat statbuf;
    struct dirent *dirp;
    DIR *dp;
    int ret, n;

    //递归终止的2个条件,1:文件相关信息无法获取, 2:传递的绝对路径不是目录而是文件
    if(lstat(fullpath, &statbuf)<0)
    {
        return Myfunc(fullpath, &statbuf, FTW_NS);
    }
    if(S_ISDIR(statbuf.st_mode)==0)
    {
        return Myfunc(fullpath, &statbuf,FTW_F);
    }

    //如果fullpath是目录
    //如果返回表示0表示出错
    if((ret=Myfunc(fullpath, &statbuf, FTW_D)) != 0)
    {
        return ret;
    }
    n = strlen(fullpath);
    printf("fullname long = %d\n",n);
    if(n+NAME_MAX+2>pathlen)
    {
        pathlen *= 2;
        char *save = fullpath;
        fullpath = (char*)malloc(pathlen);
        for(int j=0;j<n;j++)
        {
            fullpath[j] = save[j];
        }
    }
    fullpath[n++]='/';
    fullpath[n] = 0;
    
    printf("fullpath2=%s\n", fullpath);
    if((dp = opendir(fullpath)) == NULL)
    {
        printf("fullpath3=%s\n", fullpath);
        return Myfunc(fullpath, &statbuf, FTW_DNR);
    }
    
    // 逐个获取目录下文件名,直到空表示终止
    while ((dirp = readdir(dp))!=NULL)
    {
        if(strcmp(dirp->d_name, ".") == 0 || strcmp(dirp->d_name, "..")==0)
        {
            continue;
        }
        strcpy(&fullpath[n], dirp->d_name);
        if((ret=dopath(Myfunc))!=0)
        {
            break;
        }
    }
    fullpath[n-1] = 0; //恢复初始路径,深度优先搜索回溯
    if(closedir(dp)<0)
    {
        err_ret("can't close directory %s", fullpath);
    }
    return ret;
}

static int Myfunc(const char *pathname, const struct stat *statprt, int type)
{
    switch (type)
    {
    case FTW_F:
        switch (statprt->st_mode & S_IFMT)
        {
            case S_IFREG: nreg++;break;
            case S_IFBLK: nblk++;break;
            case S_IFCHR: nchr++;break;
            case S_IFIFO: nfifo++;break;
            case S_IFLNK: nslink++;break;
            case S_IFSOCK: nsock++;break;
            case S_IFDIR: err_dump("for S_IFDIR for %s", fullpath); 
        }
        break;
    case FTW_D:
        ndir++;
        break;
    case FTW_DNR:
        err_ret("can't read1 directory %s", fullpath);
        break;
    case FTW_NS:
        err_ret("stat error for %s", fullpath);
        break;
    default:
        err_dump("unknown type %d for pathname %s", type, fullpath);
        break;
    }
    return 0;
}

```

```
$ ./readdir.o /home
regular file   =  321788, 91.13%
directories    =   25916,  7.34%
block special  =       0,  0.00%
char special   =       0,  0.00%
FIFOs          =       1,  0.00%
symbolic link  =    5390,  1.53%
sockets        =       0,  0.00%

```

## 函数chdir,fchdir和getcwd

每个进程都有一个当前工作目录,此目录是搜索所有相对路径的起点.当用户登录到UNIX时,器当前工作目录通常是口令文件(/etc/passwd)中该用户登录项的第六个字段--用户起始目录.当前工作目录是进程的一个属性,起始目录则是登录名的一个属性.进程调用chdir或fchdir函数更改当前工作目录:

```c
#include<unistd.h>
int chdir(const char *pathname);
int fchdir(int fd);
//返回0表示成功,返回-1表示错误.

```

获取当前工作目录的绝对路径:

```c
#include<unistd.h>
char *getpwd(char *buf, size_t size);
//成功返回buf,失败返回NULL

```

参数buf是缓冲区地址,size是缓冲区长度,缓冲区必须有足够长度以容纳绝对路径名再加上一个null字节.



# 第五章 标准I/O库

## 流和FILE对象

对于标准I/O库,操作都是围绕流进行的.当用标准库打开或创建一个文件时,我们已近使用一个流与其关联.

流的定向决定了所读写的是单字节还是多字节.如若在未定向的流上使用多字节I/O函数,则将该流的定向设置为宽定向的,若在未定向的流上使用一个单字节I/O函数,则将该该流设置为字节定向的.

fwide函数用于设置流的定向:

```c
#include<stdio.h>
#include<wchar.h>
int fwide(FILE *fp, int mode);
//宽定向返回正值,字节定向返回负值,为定向返回0

```

mode为负值,试图将流指定为字节定向,mode为正值,试图将流指定为宽定向,mode为0,不指定定向.

fwide不改变已定向的流的定向.

当打开一个流时,标准I/O函数fopen返回一个指向FILE对象的指针.该对象通常是一个结构,它包含了标准I/O库为管理该流需要的所有信息,包括用于实际I/O的文件描述符,指向用于该缓冲区的指针,缓冲区的长度,当前在缓冲区的长度以及出错标志等.

## 标准输入,标准输出与标准错误

对一个进程预定义了三个流,标准输入,标准输出与标准错误.这三个流进程可以自动使用.

这三个标准I/O通过预定义文件指针stdin,stdout.stderr加以引用,这三个文件指针被定义在头文件<stdio.h>中.

## 缓冲

标准I/O库提供缓冲的目的是为了尽可能的减少使用read和write次数.标准库提供了三种缓冲类型.

(1)全缓冲.在这种情况下,在填满标准I/O缓冲区后才进行实际I/O操作.在一个流上第一次执行I/O操作时,相关标准I/O函数通常调用malloc获得需要的缓冲区.

　　术语冲洗说明标准I/O写操作.缓冲区可向标准I/O自动冲洗,或者可以调用fflush冲洗一个流.flush存在两种意思,在I/O方面,flush表示将缓冲区写入磁盘,在终端驱动程序方面,flash表示丢弃已存储在缓冲区的数据.

(2)行缓冲. 在输入和输出遇到换行符时,标准I/O库执行I/O操作.这允许我们一次输出一个字符,但只在写了一行后才进行实际I/O操作.终端中(涉及标准输入输出),通常使用行缓冲.

　　对于行缓冲通常有两个限制.第一:I/O库的缓冲区是有限制的,如果一行太长,填满了缓冲区,即使没有到达换行符,也进行I/O操作.第二:任何时候,通过标准I/O库要求从(a)一个不带缓冲的流,或者(b)一个行缓冲流得到数据,那么就会冲洗所以行输出流.

(3)不带缓冲.标准I/O不对字符进行缓冲存储.标准错误流stderr通常是不带缓冲的,这就是使得错误信息可以立即显式出来.

ISO C要求缓冲特征:

1. 当且仅当标准输入和标准输出并不指向交互设备时,他们才是全缓冲的.
2. 标准错误绝不是全缓冲的.

一般系统默认缓冲类型:

1. 标准错误是不带缓冲的.
2. 若是指向终端设备的流,则是行缓冲的,否则是全缓冲.

可以使用下列两个函数更改缓冲类型:

```c
#include<stdio.h>
void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf,int mode, size_t size);
//返回0成功,否则失败

```

参数解释:

|  函数   |  mode  | buf  |        缓冲区及长度         |    缓冲类型    |
| :-----: | :----: | :--: | :-------------------------: | :------------: |
| setbuf  |        | 非空 | 长度为BUFSIZ的用户缓冲区buf | 全缓冲或行缓冲 |
| setbuf  |        | NULL |          无缓冲区           |    不带缓冲    |
| setvbuf | _IOFBF | 非空 |    长度为size的缓冲区buf    |     全缓冲     |
| setvbuf | _IOFBF | NULL |   合适长度的系统缓冲区buf   |     全缓冲     |
| setvbuf | _IOLBF | 非空 |    长度为size的缓冲区buf    |     行缓冲     |
| setvbuf | _IOLBF | NULL |   合适长度的系统缓冲区buf   |     行缓冲     |
| setvbuf | _IONBF | 忽略 |           无缓冲            |    不带缓冲    |

任何时候,我们可以强制刷新一个流:

```c
#include<stdio.h>
int fflush(FILE *fp);
//成功返回0,否则返回EOF

```

## 打开流

函数原型:

```c
#include<stdio.h>
FILE *fopen(const char *restrict pathnem, const char *restrict type);
FILE *freopen(const char *restrict pathnem, const char *restrict type,FILE *restrict fp);
FILE *fdopen(int fd,const char *type);

```

fopen打开路径名为pathname的文件.

freopen在一个指定流上打开文件,如果流已经被打开,则先关闭该流.如果流已经定向,则清除定向,此函数通常将一个指定的文件绑定到一个指定的流上:标准输入输出错误.

fdopen取一个文件描述符,并使一个标准I/O流与该描述符结合.此函数通常用于创建管道和网路通信通道函数返回的描述符.

type有15种取值:

|       type       |                 说明                 |          open标准           |
| :--------------: | :----------------------------------: | :-------------------------: |
|     `r`/`rb`     |              为读而打开              |          O_RDONLY           |
|     `w`/`wb`     |     把文件截断为0长,或为写而创建     | O_WRONLY\|O_CREAT\|O_TRUNC  |
|     `a`/`ab`     | 追加:为在文件尾写而打开,或为写而创建 | O_WRONLY\|O_CREAT\|O_APPEND |
| `r+`/`r+b`/`rb+` |             为读和写创建             |          O_RDONLY           |
| `w+`/`w+b`/`wb+` |     把文件截断为0长,或为写而创建     | O_WRONLY\|O_CREAT\|O_TRUNC  |
| `a+`/`a+b`/`ab+` | 追加:为在文件尾写而打开,或为写而创建 | O_WRONLY\|O_CREAT\|O_APPEND |

调用fclose关闭一个流:

```c
#include<stdio.h>
int fclose(FILE *fp);
//成功返回0,否则返回EOF

```

## 读和写流

打开流后,可以使用三种不同类型的非格式化I/O对其进行读写操作.

(1)每次一个字符的I/O

(2)每次一行的I/O

(3)直接I/O.fread和fwrite函数支持这种类型I/O.常用于从二进制文件中每次读写一个结构.

### 输入函数(一次一个字符)

```c
#include<stdio.h>
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
//若成功返回下一个字符,若已到达文件末尾或出错,返回EOF

```

函数getchar等于getc(stdin)(标准输入).前两个函数的区别是,getc可被实现为宏,而fgetc不能.

这三个函数在返回下一个字符时,将其`unsigned char`转换为`int`.要求返回整型的原因是,这样就可以返回所以可能的字符再加上一个出错或者到达文件末尾的指示值. EOF通常是一个负值,一般是-1.

不管出错还是到达文件末尾,三个函数都是返回相同的值,这时候想要区分就需要调用下面的函数:

```c
#include<stdio.h>
int ferror(FILE *fp);
int feof(FILE *fp);
//函数返回非0,表示为真,否则为假

void cleareer(FILE *fp);

```

每个流在FILE对象中维护了两个标志:

1. 出错标志
2. 文件结束标志

调用cleareer可以清除这两个标志.

### 输出函数(一次一个字符)

```c
#include<stdio.h>
int putc(int c,FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
//成功返回c,否则返回EOF

```

puchar(c)等于putc(c,stdout).

## 每次一行I/O

### 输入一行

```c
#include<stdio.h>
char *fgets(char *restrict buf, int n, FILE *restrict fp);
char *gets(char *buf);
//成功返回buf,到达文件末尾或出错返回NULL

```

gets从标准输入中读取,fgets从指定流中读取.fgets需要指定缓冲的长度n.此函数一直到下一个换行符为止,但不超过n-1个字符,读入的字符被送入缓冲区.缓冲区总是以null字节结尾.对于超过n-1个字符的行,fgets只返回一个不完整的行,下次调用会继续处理这一行.

gets不能指定缓冲区长度,不推荐使用.gets和fgets的一个区别是,gets并不将换行符存入缓冲区中.

### 输出一行

```c
#include<stdio.h>
int fputs(const char *restrict str, FILE *restrict fp);
int puts(const char *str);
//成功返回非负值,到达文件末尾或否则返回EOF

```

函数fputs将一个以null字节作为结尾的字符串写到指定的流,尾端的null不写出.fputs不一定是一次输出一行,因为字符串不必最后一个非null字符为换行符.

## 输入输出举例

### 按字节输入输出

```c
#include"apue.h"
static int count = 0;
int main(void)
{
    int c;
    while((c=fgetc(stdin))!=EOF)
    {
        count++;
        if(fputc(c,stdout)==EOF)
        {
            err_sys("output error\n");
        }
    }
    if(ferror(stdin))
    {
        err_sys("input error\n");
    }
    printf("count=%d\n",count);
}

```

由count可以看出,标准输入输出行缓冲的时候会根据换行符作为终止,同时会将换行符传入流中.

### 按行输入输出

```c
#include"apue.h"
#include"string.h"
int main(void)
{
    char buf[MAXLINE];
    while(fgets(buf,MAXLINE,stdin)!=NULL)
    {
        int num = strlen(buf);
        printf("strlen = %d\n",num);
        if(fputs(buf,stdout)==EOF)
        {
            err_sys("output error\n");
        }
    }
    if(ferror(stdin))
    {
        err_sys("input error\n");
    }
    exit(0);
}

```

由于strlen不计算字符串末尾的空字符,因此通过count我们也能发现按行读取时,换行符会被读到标准输入,这是我们如果将末尾的换行符替换成空字符,输出就不是按行了.

## 二进制I/O

二进制I/O主要用于一次读写一个结构.下面两个函数提供了二进制I/O操作

```c
#include<stdio.h>
size_t fread(void *restrict ptr,size_t size, size_t nobj, FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj, FILE *restarict fp);
//函数返回值为读写对象的数量.

```

这两个函数有以下两种常见用法.

(1)读或写一个二进制数组.如将一个浮点数组的第2-5个元素写到一个文件.

```c
float data[10];
if(fwrite(&data[1],sizeof(float),4,fp)!=4)
    err_sys("fwrite error\n");

```

(2)读或写一个结构

```c
struct{
    short count;
    long total;
    char name[NAMESIZE];
} item;

if(fwrite(&item,sizeof(item),1,fp)!=1)
    err_sys("fwrite error\n");

```

例:

```c
#include<stdio.h>
#include<string.h>
#include"apue.h"
#define NAME_SIZE 20
struct bio
{
    int score;
    char name[NAME_SIZE];
};

int main(int argc, char *argv[])
{
    bio self;
    FILE *fp = fopen(argv[1], "w");
    char name[] = "chst";
    for(int i=0;i<strlen(name);i++)
    {
        self.name[i] = name[i];
    }
    self.name[strlen(name)] = 0;
    self.score = 30;
    if(fwrite(&self,sizeof(bio),1,fp)!=1)
    {
        err_sys("fwrite error\n");
    }
    fclose(fp);
    FILE *fd = fopen(argv[1], "r");
    bio self2;
    if(fread(&self2,sizeof(bio), 1, fp)!=0)
    {
        printf("name:%s,score=%d\n",self2.name, self2.score);
    }
    else
    {
        err_sys("fread error\n");
    }
    exit(0);
}

```

## 格式化I/O

### 格式化输出

```c
#include<stdio.h>
int printf(const char *restrict format,...);
int fprintf(FILE *restrict fp, const char *restrict format,...);
int dprintf(int fd,const char *restrict format,...);
//若成功,返回输出字符数,若失败,返回负值

int sprintf(char *restrict buf,const char *restrict format,...);
//若成功,返回存入数组的字符数,若编码错误,返回负值

int snprintf(char *restrict buf,size_t n,const char *restrict format,...);
//若缓冲区足够大,返回将要存入数组的字符数,若编码错误,返回负值.

```

sprintf将格式化的字符输出到数组buf中,会在数组的尾端加上一个null.sprintf函数可能导致缓冲区buf溢出.为了解决缓冲区溢出问题,引入了snprintf函数,在该函数中,缓冲区是一个显式参数,超过缓冲区长度的部分会被丢弃,与sprintf相同,返回值不包括结尾的null字节.

格式说明控制其余参数如何编写,以后又该如何显示.每个参数按照转换说明编写,转换说明以百分号%开始,除转换说明外,格式字符串的其他字符将按原样,不经任何修改被复制输出.一个转换说明有4个可选部分:

```
%[flags][fldwidth][precision][lenmodifier]convtype

```

|  标志  |                           说明                           |
| :----: | :------------------------------------------------------: |
|  `＇`  |               (撇号)将整数按照千位分组字符               |
|  `-`   |                    在字段内左对齐输出                    |
|  `+`   |                总是显示带符号转换的正负号                |
| (空格) |      如果第一个字符不是正负号，则在其前面加一个空格      |
|  `#`   | 指定另一中转换形式（例如，对于十六进制格式，加０ｘ前缀） |
|   0    |                    添加前导０进行填充                    |

fldwidth说明最小字段宽度.转换后参数若小于宽度,则多余字符使用空格填充.宽度是一个非负十进制数或`*`.

precision说明整型转换后最少输出数字位数,浮点数转换后小数点后的最少位数,字符串转换后最大字节数.精度是一个`.`,其后更随一个可选的非负十进制数或一个`*`.

lenmodifier说明参数长度:

| 长度修饰符 | 说明                                               |
| ---------- | -------------------------------------------------- |
| `hh`       | 将相应参数按照signed或者unsigned char类型输出      |
| `h`        | 将相应参数按照signed或者unsigned short类型输出     |
| `l`        | 将相应参数按照signed或者unsigned long类型输出      |
| `ll`       | 将相应参数按照signed或者unsigned long long类型输出 |
| `j`        | intmax_t或uintmax_t                                |
| `z`        | size_t                                             |
| `t`        | ptrdiff_t                                          |
| `L`        | long double                                        |

convtype不是可选的,它控制如何解释参数.

| 转换类型 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| `d`/`i`  | 有符号十进制                                                 |
| `o`      | 无符号八进制                                                 |
| `u`      | 无符号十进制                                                 |
| `x`/`X`  | 无符号十六进制                                               |
| `f`/`F`  | 双精度浮点数                                                 |
| `e`/`E`  | 指数格式双精度浮点数                                         |
| `g`/`G`  | 根据转换后的值解释为`f`/`F`/`e`/`E`                          |
| `a`/`A`  | 十六进制指数格式双精度浮点数                                 |
| `c`      | 字符(若带长度修饰符1,为宽字符)                               |
| `s`      | 字符串(若带长度修饰符1,为宽字符)                             |
| `p`      | 指向void的指针                                               |
| `n`      | 到目前为止,次printf调用输出的子符的数目将被写到指针说指向的带符号整型中 |
| `%`      | 一个%字符                                                    |
| `C`      | 宽字符,等价于1c                                              |
| `S`      | 宽字符串,等价于1s                                            |

### 格式化输入

```c
#include<stdio.h>
int scanf(const char *restrict format,...);
int fscanf(FILE *restrict fp,const char *restrict format,...);
int sscanf(const char *restrict buf,const char *restrict format,...);
//赋值的输入项数,若错误或者在任一转换前已经到达文件末尾则返回EOF

```

scanf族用于分析输入字符串,并将字符序列转换为指定类型变量.在格式之后包含了变量的地址(因此使用&a),用转换结果对这些变量赋值.

格式说明控制如何转换参数,以便对他们赋值.转换说明以%开始.除转换说明和空格外,格式字符中的其他字符必须与输入一致.若存在一个字符不匹配,则停止后续处理.

一个转换说明有三个可选部分:

```
%[*][fldwidth][m][lenmodifier]convtype

```

可选的(*)是抑制转换,按照转换说明的其余部分对输入进行转换,但转换后的结果并不放到结果参数中.

可选项m是赋值分配符.可以用于`%C`,`%S`以及`%[`转换符,迫使内存缓冲区分配空间以接纳字符串.此时,相关参数必须是指针地址,分配的缓冲区地址必须赋值给该指针.如果调用成功,该缓冲区域不再使用时,由用户负责调用free来释放该缓冲区.

| 转换类型                        | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `d`                             | 符号十进制                                                   |
| `i`                             | 有符号十进制                                                 |
| `O`                             | 无符号八进制                                                 |
| `u`                             | 无符号十进制                                                 |
| `x`/`X`                         | 无符号十六进制                                               |
| `a`/`A`/`e`/`E`/`f`/`F`/`g`/`G` | 浮点数                                                       |
| `c`                             | 字符(若带长度修饰符1,为宽字符)                               |
| `s`                             | 字符串(若带长度修饰符1,为宽字符)                             |
| `[`                             | 匹配列出的字符序列,以]终止                                   |
| `[^`                            | 匹配除列出了来的字符以外的所有字符,以]终止                   |
| `p`                             | 指向void的指针                                               |
| `n`                             | 将到目前为止该函数调用读取的字符数写入到指针所指向的无符号整型中 |
| `%`                             | 一个%符号                                                    |
| `C`                             | 宽字符,等效与1c                                              |
| `S`                             | 宽字符,等效于ls                                              |

## 实现细节

每个标准I/O流都有一个与其相关的文件描述符,可以对一个流调用fileno函数来获得其描述符:

```
#include<stdio.h>
int fileno(FILE *fp);

```

# 第六章 系统数据文件和信息

## 口令文件

UNIX系统口令文件包含了下列的个字段(linux不包含最后三个字段),这些字段包含在<pwd.h>中定义的passwd结构中.

| 说明                | struct passwd成员 |
| :------------------ | :---------------- |
| 用户名              | char *pw_name     |
| 加密口令            | char *pw_passwd   |
| 数值用户ID          | uid_t pw_uid      |
| 数值组ID            | gid_t pw_gid      |
| 注释字段            | char *pw_gecos    |
| 初始工作目录        | char *pw_dir      |
| 初始shell(用户程序) | char *pw_shell    |
| 用户访问类          | char *pw_class    |
| 下次更改口令时间    | time_t pw_change  |
| 账户有效期时间      | time_t pw_expire  |

口令文件是`/ect/passwd`.每一行包含上述各字段,字段之间用冒号分隔.

关于登录项,需要注意:

1. 通常有一个用户名为root的登录项,其用户ID是0(超级用户).
2. 加密口令字段包含了一个占位符.
3. shell字段包含了一个可执行程序名,它被用来作为该用户的登录shell.若为空,使用系统默认值,一般是/bin/shell.
4. 为了阻止一个特定用户登录系统.可以在初始shell中使用/dev/null或者/bin/false或在/bin/true禁止一个账户.
5. 使用nobody用户名的一个目的是,使任何人都能够登录至系统,但其用户ID(65534)和用户组ID(65534)不提供任何权限,只可以访问人人都可以读写的文件.

下面的两个函数可以获得口令文件项:

```
#include<pwd.h>
struct passwd *getpwuid(uid_t uid);
struct passwd *getpwnam(const char *name);

```

getpwuid函数由ls程序使用,它将`i`节点中的数字用户ID映射为用户登录名.在键入登录名时,getpwnam函数由login程序调用.passwd结构通常是函数内部的静态变量,只要调用任一相关函数,其内容就会被重写.

当程序想要查看整个口令文件时,可以使用下列3个函数:

```c
#include<pwd.h>
struct passwd *getpwent(void);
//若成功,返回指针,若错误或者到达文件末尾返回NULL

void setpwent(void);
void endpwent(void);

```

每次调用getpwend时,其返回口令文件的下一个记录项.setpwent用来将getpwent()的读写地址指向口令文件的开头,endpwent则关闭这些文件.在使用getpwent后一定要使用endpwent关闭这些文件.

getpwnam的一个实现:

```c
#include<stdio.h>
#include<pwd.h>
#include<stddef.h>
#include<string.h>
passwd *getpwnam(const char *name)
{
    passwd *ptr;
    setpwent(); //确保文件以及被关闭
    while((ptr=getpwent())!=NULL)
    {
        if(strcmp(name,ptr->pw_name)==0)
        {
            break;
        }
    }
    endpwent();
    return ptr;
}
int main(int argc, char *argv[])
{
    passwd *ptr = getpwnam(argv[1]);
    if(ptr!=NULL)
    {
        printf("name:%-10s; passwd:%-10s; uid:%5d, gid=%-5d; gecos=%-20s; dir=%-10s; shell=%-10s\n",ptr->pw_name
        ,ptr->pw_passwd,ptr->pw_uid,ptr->pw_gid,ptr->pw_gecos,ptr->pw_dir,ptr->pw_shell);
    }
    return 0;
}

```



## 阴影口令

加密口令是经过单向加密算法处理过的用户副本.因为此算法是单向的,所以不能从加密口令猜测到原来的口令.

为了使一般用户无法获得加密口令,系统将加密口令放在另一个通常称为阴影口令的文件中,该文件至少要包含用户名与加密口令.与该口令有关的信息也可以放在该文件中:

| 说明                     | struct spwd成员      |
| ------------------------ | -------------------- |
| 用户登录名               | char *sp_name        |
| 加密口令                 | char *sp_pwdp        |
| 上次更改口令以来经过时间 | int sp_lstchg        |
| 经多少天后允许更改       | int sp_min           |
| 要求更改剩余天数         | int sp_max           |
| 超期警告天数             | int sp_warn          |
| 账户不活动之前剩余天数   | int sp_inact         |
| 账户超期天数             | int sp_expire        |
| 保留                     | unsigned int sp_flag |

阴影口令文件不是一般用户可以读取的.仅少数几个程序需要访问加密口令,如login和passwd,这些用户常常设置用户ID为root.与访问口令文件相似,存在访问阴影口令文件的一组函数:

```c
#include<shadow.h>
struct spwd *getspnam(const char *name);
struct spwq *getspent(void);
//成功返回指针,失败返回NULL

void setspent(void);
void endspent(void);

```

## 组文件

UNIX组文件包含了下面所列字段,这些字段包含在<grp.h>中所定义的group中:

| 说明                   | struct group成员 |
| ---------------------- | ---------------- |
| 组名                   | char *gr_name    |
| 加密口令               | char *gr_passwd  |
| 数值组ID               | int gr_gid       |
| 指向个用户名指针的数值 | char **gr_mem    |

下列两个函数可以查看组名或组ID:

```c
#include<grp.h>
struct group *getgrgid(gid_t gid);
struct group *getgrnma(const char *name);

```

与口令文件类似,这里的group也是静态变量的指针.

如果需要搜索整个组文件:

```
#include<grp.h>
struct group *getgrent(void);
void setgrent(void);
void endgrent(void);

```

## 附属组ID

我们不仅可以属于口令文件记录项中的组ID所对应的组,也可以属于多至16个另外的组.文件访问权限被修改为:不仅将进程有效ID与文件的组ID进行比较,而且也将所以附属组ID与文件的组ID进行比较.使用附属组ID的一个好处是不用经常更改组.

为了获取和设置附属组ID,提供了下面三个函数:

```c
#include<unistd.h>
int getgroups(int gidsetsize,gid_t grouplist[]);
//成功返回附属组ID数量,出错返回-1

#include<grp.h> //on linux
#include<unistd.h> //on freebsd, mac os x, solaris
int setgroups(int ngroups,const gid_t grouplist[]);

#include<grp.h> //on linux
#include<unistd.h> //on freebsd, mac os x, solaris
int initgroups(const char *username,gid_t basegid);
//两个函数成功返回0,失败返回-1

```

getgroup将进程所属用户的各附属组ID填写到数组grouplist中,填入该数组的附属组ID最多gidsetsize个,实际填写的数量由函数返回.

setgroups可由超级用户调用以便为调用进程设置附属组ID表,grouplist是组ID数组,ngroups说明数组中元素个数.

通常只有initgroups函数调用setgroups,initgroups读整个组文件,然后对username确定其组的成员关系,然后调用setgroups,以便为该用户初始化附属组ID表.

## 其他数据文件

一般情况下,对每个数据文件至少有三个函数:

(1) get函数:读下一条记录,如果需要还会打开该文件,一般返回静态存储类结构的指针.

(2) set函数:打开对应数据文件,然后反绕该文件.

(3) end函数:关闭相关数据文件.

另外,如果数据文件支持某种形式的键搜索,则也提供搜索具有指定键的记录的例程.

下面列出一些常用的数据文件

| 说明 | 数据文件       | 头文件     | 结构     | 附加键搜索函数                   |
| ---- | -------------- | ---------- | -------- | -------------------------------- |
| 口令 | /etc/passwd    | <pwd.h>    | passwd   | getpwnam, getpwuid               |
| 组   | /etc/group     | <grp.h>    | group    | getgrnam, getgrgid               |
| 阴影 | /etc/shadow    | <shadow.h> | spwd     | getspnam                         |
| 主机 | /etc/hosts     | <netdb.h>  | hostent  | getnameinfo, getaddrinfo         |
| 网络 | /etc/networks  | <netdb.h>  | netent   | getnetbyname, getnetbyaddr       |
| 协议 | /etc/protocols | <netdb.h>  | protoent | Getprotobyname, getprotobynumber |
| 服务 | /etc/services  | <netdb.h>  | servent  | getservbyname, getservbyport     |

## 登录账户记录

UNIX下提供了两个数据文件:utmp文件记录当前登录到系统的各个用户;wtmp文件跟踪各个登录和注销事件.每次写入的是包含下列结构的一个二进制记录:

```c
struct utmp{
    char ut_line[8];
    char ut_name[8];
    long ut_time;
};

```

登录时,login程序填写此类型数据结构,然后将其写入到utml文件,同时也添加到wtmp文件.注销时,init进程将utmp文件中相应记录删除,并将一个新记录添加到wtmp文件中.

## 系统标识

```c
#include<sys/utsname.h>
int uname(struct utsname *name);

```

uname函数返回与主机和操作系统相关的信息.该函数向其中传递一个utsname地址,该函数会填充结构内容.结构包含如下信息:

```c
struct{
    char sysname[];
    char nodename[];
    char release[];
    char version[];
    char matchine[];
};

```

获取主机名:

```
#include<unistd.h>
int gethostname(char *name,int namelen);

```

该名字通常就是TCP/IP网络上主机的名字.

## 时间和日期例程

UNIX内核提供的基本时间服务是计算自协调世界时(UTC)公元1970年1月1号00:00:00这一特定时间以来经过的秒数.这种秒数是以数据类型time_t表示的(第三章),我们称之为日历时间.日历时间包含时间和日期.UNIX特点是:(1)以协调统一时间而非本地时间计时;(2)可自动进行转换;(3)将时间和日期作为一个量值保存.

time函数返回当前时间和日期:

```c
#include<time.h>
time_t time(time_t *calptr);

```

POSXI.1的实时扩展增加了对多个系统时钟的支持.时钟通过clockid_t类型进行标识.

| 标识符                   | 选项                   | 说明                     |
| ------------------------ | ---------------------- | ------------------------ |
| CLOCK_REALTIME           |                        | 实时系统时间             |
| CLOCK_MONTONIC           | _POSIX_MONOTONIC_CLOCK | 不带负跳数的实时系统时间 |
| CLOCK_PROCESS_CPUTIME_ID | _POSIX_CPUTIME         | 调用进程的CPU时间        |
| CLOCK_THREAD_CPUTIME_ID  | _POSIX_THREAD_CPUTIME  | 调用线程的CPU时间        |

clock_gettime函数可用来获取指定时钟时间,返回timespec结构(第四章),其把时间表示为秒和纳秒:

```c
#include<sys/time.h>
int clock_gettime(clockid_t clock_id,struct timespec *tsp);

```

当时钟ID设置为CLOCK_REALTIME时,clock_gettime函数提供了与time函数类似的功能,不过clock_gettime可能比time函数的精度高.

```c
#include<sys/time.h>
int clock_getres(clockid_t clock_id,struct timespec *tsp);

```

clock_getres函数将tsp指向的timespec结构初始化为与clock_id对应的时钟精度.

如果需要对特定的时钟设置时间,可以调用clock_settime函数:

```c
#include<sys/time.h>
int clock_settime(clockid_t clock_id, const struct timespec *tsp);

```

下图展示了各种时间函数之间的关系:

![time](https://s2.ax1x.com/2019/09/22/u9LVq1.png)

图中虚线表示的三个函数localtime,mktime和strftime都受到环境变量TZ的影响.两个函数localtime和gmtime将日历时间转换成分解的时间,并将这些存放在一个tm结构中:

```c
struct tm{
    int tm_sec; //秒:[0,60]
    int tm_min;//分钟:[0,59]
    int tm_hour;//小时:[0,23]
    int tm_mday;//一个月的某一天:[1,31]
    int tm_mon;//月:[0-11]
    int tm_year;//年,从1970年开始到现在
    int tw_wday;//一周的某一天[0,6]
    int tw_yday;//一年的某一天[0,365]
    int tm_isdst;//夏令时标志
};

```

从日历时间获得分解时间:

```c
#incude<time.h>
struct tm *gmtime(const time_t *calptr);
struct tm *localtime(const time_t *calptr);
//成功返回指针,出错返回NULL

```

localtime和gmtime的区别是,localtime将日历转为本地时间,而gmtime将日历时间转换为协调统一时间.

从分解时间转换的日历时间:

```c
#include<time.h>
time_t mktime(struct tm *tmptr);
//成功返回日历时间,出错返回-1

```

打印时间:

```c
#include<time.h>
size_t strftime(char *restrict buf,size_t maxsize,const char *restrict format,const struct tm *tmptr);
size_t strftime_l(char *restrict buf,size_t maxsize,const char *restrict format,const struct tm *restrict tmptr,locale_t local);
//若有空间则返回存入数组字符数,否则返回0

```

strftime_l将区域指定为参数,除此之外两个函数完全一致.strftime使用环境变量TZ指定区域.

format参数控制了时间值的格式.形式是在一个百分号后更随一个特定字符,其他字符原样输出,不存在字段宽度修饰符.

| 格式 | 说明                                  | 实例                     |
| ---- | ------------------------------------- | ------------------------ |
| `%a` | 缩写的周日名                          | Thu                      |
| `%A` | 周日名                                | Thursday                 |
| `%b` | 缩写的月名                            | Jan                      |
| `%B` | 月名                                  | January                  |
| `%c` | 日期和时间                            | Thu Jan 19 21:24:52 2012 |
| `%C` | 年/100(00-99)                         | 20                       |
| `%d` | 月日(01-31)                           | 19                       |
| `%D` | 日期(MM/DD/YY)                        | 01/19/12                 |
| `%e` | 月日(一位数字前加空格)(1-31)          | 21                       |
| `%F` | ISO 8601日期格式(YYYY-MM-DD)          | 2012-01-09               |
| `%g` | ISO 8601基于周的年的最后两位数(00-99) | 12                       |
| `%G` | ISO 8601基于周的年                    | 2012                     |
| `%h` | 与`%b`相同                            | Jan                      |
| `%H` | 小时(24)(00-23)                       | 21                       |
| `%I` | 小时(12)(00-11)                       | 09                       |
| `%j` | 年日(001-366)                         | 019                      |
| `%m` | 月(01-12)                             | 01                       |
| `%M` | 分(01-59)                             | 23                       |
| `%n` | 换行符                                |                          |
| `%p` | AM/PM                                 | PM                       |
| `%r` | 本地时间(12)                          | 09:24:52 PM              |
| `%R` | 与`"%H:%M"`相同                       | 21:24                    |
| `%S` | 秒[00-60]                             | 52                       |
| `%t` | 水平制表符                            |                          |
| `%T` | 与`"%H:%M:%S"`相同`                   | 21:24:52                 |
| `%u` | ISO 8601周几(1-7)                     | 4                        |
| `%U` | 星期日周数(00-53)                     | 03                       |
| `%V` | ISO 周数(01-53)                       | 03                       |
| `%w` | 周几(0-6)                             | 03                       |
| `%W` | 星期一周数(00-53)                     | 03                       |
| `%x` | 本地日期                              | 01/19/12                 |
| `%X` | 本地时间                              | 21:24:52                 |
| `%y` | 年的最后两位数(00-99)                 | 12                       |
| `%Y` | 年                                    | 2012                     |
| `%z` | ISO 8601格式的UTC偏移量               | -0500                    |
| `%Z` | 时区名                                | EST                      |
| `%%` | 翻译为一个%                           | %                        |

打印时间例子:

```c
#include<time.h>
#include<stdio.h>
#include<stdlib.h>
int main(void)
{
    time_t t;
    struct tm *tmp;
    char buf1[16];
    char buf2[64];
    time(&t);
    tmp = localtime(&t);
    if(strftime(buf1,16,"time and date:%r, %a %b %d, %Y", tmp)==0)
    {
        printf("buffer length 16 is too small\n");
    }
    else
    {
        printf("%s\n",buf1);
    }
    if(strftime(buf2,64,"time and date:%r, %a %b %d, %Y", tmp)==0)
    {
        printf("buffer length 64 is too small\n");
    }
    else
    {
        printf("%s\n",buf2);
    }
    exit(0);
}

```

strptime函数是strftime的反过来的版本,把字符串时间转换为分解时间:

```c
#include<time.h>
char *strptime(const char *restrict buf, const char *restrict format, struct tm *restrict tmptr);

```

格式说明符与上述类似.

# 第七章 进程环境

## main函数

```c
int main(int argc,int *argv[]);

```

内核执行C程序时(使用一个exec函数),在调用main前先调用一个特殊的启动例程.可执行程序文件将此启动例程指定为程序的起始地址.启动例程从内核获取环境变量值和命令行参数.

## 进程终止

共有八种进程终止方式,其中五种正常终止:

(1)从main函数返回;

(2)调用`exit`;

(3)调用`_exit`或`_Exit`

(4)最后一个线程从其启动例程返回;

(5)从最后一个线程调用`pthread_exit`;

三种异常终止:

(6)调用abort;

(7)接到一个信号;

(8)最后一个线程对取消请求做出响应.

启动例程一般是从main函数返回后立即调用exit函数,大概是:

```
exit(main(argc,argv));

```

### 1. 退出函数

3个函数用于正常终止一个程序:

```
#include<stdlib.h>
void exit(int status);
void _Exit(int status);
#include<unistd.h>
void _exit(int status);

```

其中`_Exit`和`_exit`立即进入内核,`exit`则先执行一些清理,在返回内核.`exit`总是执行I/O库的清理关闭操作.

3个函数都带一个整型参数,称为终止状态.如果(a)调用这些函数时不带终止状态;(b)main执行了一个无返回的`return`语句;(c)main未申明返回类型为整型,则进程终止状态是未定义的.但若main返回类型为整型,并且main执行到最后一句返回(隐式返回也可以),那么进程终止状态是0.

main函数调用`exit(0)`与`return 0`是等价的.

打印终止状态(程序执行之后):

```
$echo $?

```

### 2. 函数atexit

一个进程可以登录多至32个程序,这些函数将由`exit`自动调用,这些函数称为终止处理程序,并调用`atexit`函数来登记这些函数:

```c
#include<stdlib.h>
int atexit(void (*func)(void));
// 成功返回0,否则非0

```

参数为函数地址,调用函数时无需传递任何参数,也不期待存在返回值.`exit`调用这些函数的顺序与他们登记的顺序相反,同一个函数如果登录多次也会被执行多次.下图展示了一个C程序如何启动:

![c程序](https://s2.ax1x.com/2019/09/22/u9Lma6.png)

<font color =red/>注意:内核使程序执行的唯一方法是调用一个`exec`函数.程序自愿终止的唯一方法是显示或隐式地(通过调用`exit`)调用`_exit`或`_Exit`.进程也可以非自愿的由一个信号使其终止.</font>

例:

```c
#include<stdlib.h>
#include"apue.h"
static void myexit1(void);
static void myexit2(void);
int main(void)
{
    if(atexit(myexit2)!=0)
    {
        err_sys("can't register myexit2");
    }
    if(atexit(myexit1)!=0)
    {
        err_sys("can't register myexit1");
    }
    if(atexit(myexit1)!=0)
    {
        err_sys("can't register myexit1");
    }
    printf("main is done!\n");
    return 0;
}

static void myexit1()
{
    printf("first exit handler\n");
}

static void myexit2()
{
    printf("second exit handler\n");
}

```

执行结果:

```
main is done!
first exit handler
first exit handler
second exit handler

//myexit1和myexit2均是在函数返回时调用exit时才执行,执行顺序与添加顺序相反,加入多次会执行多次.

```

## 命令行参数

当执行一个程序时,调用`exec`的进程可以将命令行参数传递给该新进程.

## 环境表

每个程序都接收一张环境表.环境表也是一字符指针数组,其中每个指针包含一个以null为结尾的字符串的地址.全局变量environ包含了该指针数组的地址:

```
extern char **environ;

```

environ为环境指针,指针数组为环境表,其中各个指针指向的字符串为环境字符串.环境由`name=value`这样的字符组成,如下图:

![环境表](https://s2.ax1x.com/2019/09/22/u9LAM9.png)

## C程序存储空间分布

C程序由下列几部分组成:

1. 正文段.由CPU执行的机器指令.通常正文段是可共享的,在存储器中只需要一个副本,同时正文段是只读的,防止程序由于意外而修改其指令.
2. 初始化数据段.通常称为数据段,包含了程序中明确地赋初值的变量,如C程序任意函数外申明`int maxcount = 99;`.
3. 未初始化数据段,通常称为bss,在程序开始执行前,内核将此段中的数据初始化为0或空指针.如函数外的申明:`long sum[1000]`.
4. 栈.自动变量以及每次函数调用时保存的信息都存放在次段中.每次函数调用时,其返回地址以及调用者环境信息都放在栈中.最近被调用的函数在栈上为其自动变量和临时变量分配存储空间.递归函数调用自身时,就会使用一个新的栈帧,因此一次函数调用实例中的变量集不会影响另一次函数调用实例中的变量.
5. 堆.通常在堆中进行动态内存分配.

![存储空间分配](https://s2.ax1x.com/2019/09/22/u9LErR.png)

未初始化数据段的内容并不会存放在磁盘程序文件(可执行文件).内核在运行程序前将他们置0.需要存放在磁盘文件的只有正文段和初始化数据段.

size目录报告正文段,数据段和bss段的长度(字节),如:

```
$ size ./atexit.o
   text    data     bss     dec     hex filename
   4911     688      48    5647    160f ./atexit.o

```

第4列和第5列分别是以十进制和十六进制表示的三个文件总长度.

## 共享库

共享库使得可执行文件中不在需要包含公用的库函数,而只需在所有进程都可引用的存储区中保存这种库例程的副本.程序第一次执行或者第一次调用某个库函数时,用动态链接方法将程序与共享库函数相连接.这减少了每个可执行文件的长度,但增加了一些运行的开销,这种开销发生在第一次执行程序或第一次调用库函数.共享库的另一个优点是可以用库函数的新版本代替老版本而不用对使用该库的程序重新连接编辑.

例:

使用共享库进行编译:

```
$ g++ atexit.cpp -o atexit.o
$ ls -l atexit.o
-rwxrwxr-x 1 chst chst 13384 Sep 24 00:17 atexit.o
$ size atexit.o
   text    data     bss     dec     hex filename
   4911     688      48    5647    160f atexit.o

```

无共享库进行编译:

```
g++ -static atexit.cpp -o atexit.o //阻止使用共享库-static
$ ls -l atexit.o
-rwxrwxr-x 1 chst chst 849744 Sep 24 00:19 atexit.o
$ size atexit.o
   text    data     bss     dec     hex filename
 746297   21068    5984  773349   bcce5 atexit.o

```

可以明显看出,使用共享减少了大量空间.

## 存储空间分配

ISO C说明了三种用于存储空间分配的函数:

```c
#include<stdlib.h>
void *malloc(size_t size);
void *calloc(size_t nobj,size_t size);
void *realloc(void *ptr,size_t newsize);
// 成功返回非空指针,否则返回NULL

void free(void *ptr);

```

`malloc`分配指定字节的存储区域,初始值不定.`calloc`为指定数量指定长度的对象分配存储空间,该空间的每一位(bit)都是0.`realloc`增加或减少以前分配器的长度,参数是newsize是改变后的长度而不是改变的长度.如果是增大空间,可能需要将以前分配的内容移到另一个更大的区域,以便在尾部提供增加的区域,新区域的初始值不定.

`free`释放ptr指向的存储空间.

大多数实现所分配的存储空间都比所要求的稍微大一些,额外的开销用来记录管理信息--分配块的长度,指向下一个个块的指针等.这意味着,如果超过一个已分配的尾端或者在已分配区起始位置之前进行写操作,则会改写另一块的管理信息,这种错误是灾难性的,但不会很快暴露出来,所以很难发现.

## 环境变量

环境字符串形式：

```
name=value

```

ISO C提供一个函数getenv来获取环境变量值：

```c
#include<stdlib.h>
char *getenv(const char *name);
//返回指向name关联的value指针，如果未找到，返回NULL；

```

下面列出了环境变量内容：

| 变量        | 说明                           |
| ----------- | ------------------------------ |
| COLUMNS     | 终端宽度                       |
| DATEMSK     | getdate模板文件路径名          |
| HOME        | home起始目录                   |
| LANG        | 本地名                         |
| LC_ALL      | 本地名                         |
| LC_COLLATE  | 本地排序名                     |
| LC_CTYPE    | 本地字符分类名                 |
| LC_MESSAGES | 本地消息名                     |
| LC_MONETART | 本地货币编辑名                 |
| LC_NUMERIC  | 本地数字编辑名                 |
| LC_TIME     | 本地日期/时间格式名            |
| LINES       | 终端高度                       |
| LOGNAME     | 登录名                         |
| MSGVERB     | fmtmsg处理的消息组成部分       |
| NLSPATH     | 消息类模板序列                 |
| PATH        | 搜索可执行文件的路径前缀列表   |
| PWD         | 当前工作路径的绝对路径名       |
| SHELL       | 用户首选的shell名              |
| TERM        | 终端类型                       |
| TMPDIR      | 在其中创建临时文件的目录路径名 |
| TZ          | 时区信息                       |

有时，我们也需要设置环境变量或者增加新的环境变量（我们能够影响的只是当前进程及其后生成的和调用的任何子进程的环境，但不影响父进程的环境），此时我们可以使用下面的函数：

```
#include<stdlib.h>
int putenv(char *str);
//成功返回0，否则非0

int setenv(const char *name, const char *value, int rewrite);
int unsetenv(const char *name);
//成功返回0，否则-1

```

`putenv`取形式为`name=value`的字符串，将其放到环境表中，如果`name`已经存在则先删除。

`setenv`将`name`设置为`value`，如果环境中`name`已经存在，那么是否重写取决于`rewrite`。

`unsetenv`删除`name`的定义，即使不存在`name`的定义也不会出错。

修改环境表是如何操作的？

环境表和环境字符串通常占用的是进程地址空间的顶部（见C程序存储空间分布图），此时删除一个是十分简单的，但是增加或者修改一个是相对复杂的。这是因为它不能够再向高地址（向上）扩展，同时也不能移动在它下面的各栈帧，所以也不能向低地址（向下）扩展。

（1）如果修改一个现有的name：

1. 如果新的value长度不大于现在value长度，则只将新字符串复制到原字符串所在位置。
2. 如果新的value长度大于原长度，则必须使用malloc为新字符串分配空间，然后将新字符串复制到该空间，接着使用环境表中针对name的指针指向新分配区。

（2）新增加一个name，必须调用malloc为name=value字符串分配空间，而后将字符串复制到该空间。

1. 如果是第一次添加，则必须调用malloc为新的指针表分配空间。接着将原来的环境表分配到新分配区，并将name=value字符串的指针存放在该指针表的末尾，然后将一个空指针存放在其后。最后使environ指向新的指针表。此时，原来指针表位于栈顶之上，那么必须将次表移到堆中，但此时表中大多数指针仍指向栈顶的各name=value。
2. 如果不是第一次增加，则只要调用realloc以分配比原空间多存放一个指针的空间，然后将指向新的name=value的指针放到末尾，后面接一个空指针。

## 函数setjmp和longjmp

C语言中goto不能跨越函数，而执行此类跳转是函数setjmp和longjmp。这两个函数用于很深层嵌套函数调用中出错是十分有效的。

考察下面的程序：

```c
#include"apue.h"
#define TOK_ADD 5

void do_line(char *);
void cmd_add(void);
int get_token(void);

int main()
{
    char line[MAXLINE];
    while(fgets(line,MAXLINE,stdin)!=NULL)
    {
        do_line(line);
    }
}

char *tok_ptr;

void do_line(char *ptr)
{
    int cmd;
    tok_ptr = ptr;
    while((cmd=get_token())>0)
    {
        switch (cmd)
        {
        case TOK_ADD:
            cmd_add();
            break;
        }
    }
}

void cmd_add(void)
{
    int token;
    token = get_token();
    // 接下来处理相对应的指令
}

int get_token(void)
{
    //从tok_ptr*中获取下一条指令;
}

```

程序的基本骨架在读命令，确定命令类型，然后调用响应函数处理每一条指令。下图展示了调用到cmd_add之后栈的大致使用情况：

![栈使用情况](https://s2.ax1x.com/2019/09/22/u9LeVx.png)

自动变量存储在每个函数的栈帧中，数组line存储在main的栈帧中，cmd存储在do_line栈帧中，token在cmd_add栈帧中。

当发生一个非致命性错误时，例如，如果cmd_add函数发生一个错误，那么可能会先打印一个错误，然后忽略接下来的输入，返回main函数并读取下一行。如果出现在C函数的深层嵌套中，处理起来是十分麻烦的，我们不得不以检测返回值的形式逐层返回。

解决这种问题的一个方法是使用非局部goto--setjmp和longjmp函数。非局部是指，这不是普通的goto在一个函数中跳转，而是在栈上跳过若干调用帧，返回到当前函数调用路径上的某个函数上。

```
#include<setjmp.h>
int setjmp(jmp_buf env);
//若直接调用返回0，若从longjmp返回，则为非0

void longjmp(jmp_buf env,int val);

```

在希望返回到的位置调用setjmp。参数env的类型是一个特殊的jmp_buf。因为需要在另一个函数中引用env变量，通常将其定义为全局变量。

当检测到错误使用两个参数调用longjmp函数，第一个是setjmp的env，第二个是一个非0val，它将成为setjmp的返回值，可以用来判断出错的位置和类型。

利用setjmp和longjmp对之前的程序进行更改：

```c
#include"apue.h"
#include<setjmp.h>
#define TOK_ADD 5

jmp_buf jmpbuff;

void do_line(char *);
void cmd_add(void);
int get_token(void);

int main()
{
    char line[MAXLINE];
    if(setjmp(jmpbuff)!=0)
    {
        printf("error!\n");
    }
    while(fgets(line,MAXLINE,stdin)!=NULL)
    {
        do_line(line);
    }
}

char *tok_ptr;

void do_line(char *ptr)
{
    int cmd;
    tok_ptr = ptr;
    while((cmd=get_token())>0)
    {
        switch (cmd)
        {
        case TOK_ADD:
            cmd_add();
            break;
        }
    }
}

void cmd_add(void)
{
    int token;
    token = get_token();
    if(token<0)
    {
        longjmp(jmpbuff,1);
    }
    // 接下来处理相对应的指令
}

int get_token(void)
{
    //从tok_ptr*中获取下一条指令;
}

```

执行main函数时，调用setjmp，它将所需的信息记入变量jmpbuff中并返回0,。随后调用do_line，它又调用cmd_add，当出现错误时，调用longjmp后会丢弃cmd_add和do_line的栈帧，同时造成main函数中setjmp返回1。调用后的栈帧为：

![longjmp](https://s2.ax1x.com/2019/09/22/u9LnIK.png)

### 自动变量、寄存器变量和易失变量

调用longjmp后栈帧如上所述，但此时main函数中自动变量、寄存器变量和易失变量的状态又该如何？是否能够恢复到以前调用setjmp时的状态（回滚），或者保持不变。回答是不确定的。大多数都不回滚，但所以实现都声称不确定。当有一个自动变量又不想让其回滚，可以定义为具有volatile属性。申明为全局变量或静态变量的值在执行完longjmp不回滚。

下面通过实例说明自动变量、全局变量、寄存器变量、静态变量和易失变量的不同情况：

```c
#include"apue.h"
#include<setjmp.h>
static void f1(int, int, int,int);
static void f2(void);

static jmp_buf jmpbuffer;
static int globval;

int main(void)
{
    int autoval;
    register int regival;//寄存器变量
    volatile int volaval;//易失变量
    static int statval;
    globval =1; autoval = 2; regival =3;volaval=4;statval=5;
    if(setjmp(jmpbuffer)!=0)
    {
        printf("after longjmp:\n");
        printf("global=%d, autoval=%d, regival=%d, volaval=%d, statval=%d\n", 
        globval, autoval,regival,volaval,statval);
        exit(0);
    }
    globval =95; autoval = 96; regival =97;volaval=98;statval=99;
    f1(autoval,regival,volaval,statval);
    exit(0);
}

static void f1(int i, int j, int k, int l)
{
    printf("f1():\n");
    printf("global=%d, autoval=%d, regival=%d, volaval=%d, statval=%d\n", 
        globval, i,j,k,l);
    f2();
}

static void f2()
{
    longjmp(jmpbuffer,1);
}

```

执行：

```
$ g++ jmpval.cpp -lapue
$ ./a.out
f1():
global=95, autoval=96, regival=97, volaval=98, statval=99
after longjmp:
global=95, autoval=96, regival=3, volaval=98, statval=99

```

### 自动变量的潜在问题

自动变量存在一个潜在出错情况，基本规则是申明自动变量的函数已经返回后，不能再引用这些自动变量。

例如：

```c
FILE *open_data(void)
{
    FILE *fp;
    char databuf[BUFSIZ];
    if((fp=fopen("datafile","r"))==NULL)
    {
        return NULL;
    }
    if(setvbuf(fp, databuf,_IOLBF,BUFSIZ)!=0)
    {
        return NULL;
    }
    return fp;
}

```

当open_data返回时，他在栈上使用的空间将由下一个被调用函数的栈帧使用。但标准I/O还将使用这部分存储空间作为缓冲区（databuf）。这就会产生冲突和混乱，为了解决这个问题，应该在全局存储空间静态地（如static或extern）或者动态的（malloc）为数组databuf分配空间。

## 函数getrlimit和setrlimit

每一个进程都存在一组资源限制，其中一些可以使用getrlimit和setrlimit函数来查询和更改：

```c
#include<stdio.h>
int getlimit(int resure,struct rlimit *rlptr);
int setrlimit(int resurce,const struct rlimit *rlptr);
//成功返回0，否则返回非0

```

进程的资源环境通常由0进程来建立，然后由后续进程继承。函数调用制定一个资源以及一个指向`rlimit`结构的指针：

```c
struct rlimit{
    rlimit_t rlim_cur; // soft limit:current limit
    rlimit_t rlim_max; //hard limit:maximun value for rlim_cur 
};

```

更改资源限制时需要遵守下列三条限制：

1. 任何一个进程都可以将软限制调整到不大于硬限制。
2. 任何一个进程都可以降低硬限制，但必须大于或等于软限制，这种降低对于普通用户而言是不可逆的。
3. 只有超级进程可以提高硬限制值。

常量`RLIM_INFINITY`指定了一个无限量的限制。

| 限制                | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| `RLIMIT_AS`         | 进程可以使用的存储空间最大的长度（字节）。影响到sbrk和mmap函数。 |
| `RLIMIT_CORE`       | core文件的最大长度，0表示阻止生成core文件。                  |
| `RLIMIT_CPU`        | CPU时间的最大秒数，当超过此限制时，向该进程发送SIGXCPU信号。 |
| `RLIMIT_DATA`       | 数据段的最大字节长度，是初始化数据、非初始以及堆的总和。     |
| `RLIMMIT_FSIZE`     | 可以创建的文件的最大长度，超过此限制将会向进程发送信号SIGFSZ信号。 |
| `RLIMIT_MEMLOCK`    | 一个进程可以使用mlock能够锁定在存储空间的最大字节长度。      |
| `RLIMIT_MSGQUEUE`   | 进程为POSIX消息队列可分配的最大存储字节数。                  |
| `RLIMIT_NICE`       | 为了影响进程的调度优先级，nice值能够设置的最大限制。         |
| `RLIMIT_NPTS`       | 用户可以同时打开的伪终端的最大限制。                         |
| `RLIMIT_NOFILE`     | 每个进程可以打开的最多的文件数。                             |
| `RLIMIT_NPROC`      | 每个实际用户ID可拥有的最大子进程数量。                       |
| `RLIMIT_RSS`        | 最大驻内存集字节长度，如果可用的物理存储器非常少，则内核将从进程处取回超过RSS的部分。 |
| `PLIMIT_SBSIZE`     | 在任一给定时刻，一个用户可以占用的套接字的缓冲区的最大长度（字节）（linux上不存在） |
| `RLIMIT_SIGPENDING` | 一个进程可排队的信号的最大数量。                             |
| `RLIMMIT_STACK`     | 栈的最大字节数。                                             |
| `RLIMIT_SWAP`       | 用户可消耗的交换空间最大字节数。                             |
| `RLIMIT_VMEM`       | 与`RLIMIT_AS`相同。                                          |

获取限制代码：

```c
#include<sys/resource.h>
#include"apue.h"

#define doit(name) pr_limits(#name,name)

static void pr_limits(char *,int);

int main(void)
{
    #ifdef RLIMIT_AS
    doit(RLIMIT_AS);
    #endif

    doit(RLIMIT_CORE);
    doit(RLIMIT_CPU);
    doit(RLIMIT_DATA);
    doit(RLIMIT_FSIZE);
    
    #ifdef RLIMIT_MEMLOCK
    doit(RLIMIT_MEMLOCK);
    #endif
    
    #ifdef RLIMIT_MSGQUEUE
    doit(RLIMIT_MSGQUEUE);
    #endif
    
    #ifdef RLIMIT_NICE
    doit(RLIMIT_NICE);
    #endif
    
    #ifdef RLIMIT_NOFILE
    doit(RLIMIT_NOFILE);
    #endif
    
    #ifdef RLIMIT_NPROC
    doit(RLIMIT_NPROC);
    #endif
    
    #ifdef RLIMIT_NPTS
    doit(RLIMIT_NPTS);
    #endif
    
    #ifdef RLIMIT_RSS
    doit(RLIMIT_RSS);
    #endif
    
    #ifdef RLIMIT_SBSIZE
    doit(RLIMIT_SBSIZE);
    #endif
    
    #ifdef RLIMIT_SIGPENDING
    doit(RLIMIT_SIGPENDING);
    #endif
    
    #ifdef RLIMIT_STACK
    doit(RLIMIT_STACK);
    #endif
    
    #ifdef RLIMIT_SWAP
    doit(RLIMIT_SWAP);
    #endif
    
    #ifdef RLMIT_VMEM
    doit(RLMIT_VMEM);
    #endif
    exit(0);
}

static void pr_limits(char *name, int resource)
{
    rlimit limit;
    unsigned long long lim;
    if(getrlimit(resource,&limit)<0)
    {
        err_sys("getrlimit error for %s", name);
    }
    printf("%-14s ",name);
    if(limit.rlim_cur == RLIM_INFINITY)
    {
        printf("(infinite) ");
    }
    else{
        lim = limit.rlim_cur;
        printf("%10lld ",lim);
    }
    if(limit.rlim_max == RLIM_INFINITY)
    {
        printf("(infinite) ");
    }
    else{
        lim = limit.rlim_max;
        printf("%10lld ",lim);
    }
    putchar((int)'\n');
}

```

doit中使用了ISO C的字符串创建算符（#），以便为每个资源名产生字符串值：

```c
doit(RLIMIT_CORE);
//被C预处理为：
pr_limits("RLIMIT_CORE", RLIMIT_CORE);

```

# 第八章 进程控制

## 进程标识

每个进程存在一个非负整型表示的唯一进程ID。由于唯一性，常用来作为其他标识符的一部分以保证其唯一性。大多数UNIX实现延迟复用，使得新建进程的ID不同于最近终止进程所有的ID。

ID为0的进程通常是调度进程，常常被称为交换进程，该进程是内核的一部分并不执行磁盘上的任何程序。ID为1的进程通常是init进程，在自举过程结束时由内核调用。此进程负责在自举后启动一个UNIX系统。init进程绝对不会终止。

除了进程ID，进程还有其他标识：

```c
#includ<unsid.h>
pid_t getpid(void);
// 函数调用进程的进程ID

pid_t getppid(void);
//调用进程的父进程ID

pid_t getuid(void);
//调用进程的实际用ID

pid_t geteuid(void);
//调用进程的有效用户ID

pid_t getgid(void);
//调用进程的实际组ID

pid_t getegid(void);
//调用进程的有效组ID

```



## 函数fork

一个现有进程调用fork进程创建一个新进程：

```c
#include<unistd.h>
pid_t fork(void);
//子进程返回0，父进程返回子进程的ID，出错返回-1

```

子进程是父进程的副本，子进程获得父进程数据空间、堆和栈的副本。这是子进程拥有的副本，与父进程并不共享这些存储空间部分。父进程和子进程共享正文段。

由于fork后经常更随着exec，所以现在很多实现并不执行一个父进程的数据段、堆和栈的完全副本，而是采用写时复制的策略。即这些区域子进程与父进程共享，内核将其访问权限更改为只读，当子进程或者父进程要试图修改这些区域时，内核才对要修改的区域那块内存赋值一个副本，通常是虚拟存储系统中的一页。

例：

```c
#include"apue.h"
int globvar = 6;
char buf[] = "a write to stdout\n";

int main(void)
{
    int var;
    pid_t pid;
    var = 88;
    if(write(STDOUT_FILENO,buf,sizeof(buf)-1)!=sizeof(buf)-1)
    {
        err_sys("write error\n");
    }
    printf("before fork\n");
    if((pid = fork())<0)
    {
        err_sys("fork error\n");
    }
    else
    {
        if(pid == 0)
        {
            globvar++;
            var++;
        }
        else
        {
            sleep(2);
        }
        
    }
    printf("PID = %ld, glob = %d, var = %d\n", (long)getpid(), globvar, var);
    exit(0);
}

```

这里使用两种不同的运行方式，将会获得两种不同输出：

```shell
$./fork.o
a write to stdout
before fork
PID = 14487, glob = 7, var = 89
PID = 14486, glob = 6, var = 88
$ ./fork.o > tem.txt
$ cat tem.txt
a write to stdout
before fork
PID = 14491, glob = 7, var = 89
before fork
PID = 14489, glob = 6, var = 88

```

fork之后是父进程先执行还是子进程先执行是不确定的。sizeof计算字符串包含的终止null，因此需要减一（strlen不包含）。对于strlen来说，每次执行就调用响应函数，而对于sizeof来说，因为缓冲区已用已知字符串进行初始化，其长度是固定的，因此sizeof是编译时计算缓冲区长度。

对于两种不同运行方式输出不同，这是由于：对于连接到终端的标准输出来说，缓冲方式为行缓冲，此时在调用fork之前，缓冲区已近被清空，此时调用fork，子进程缓冲区也是空的，因此只会输出一次（在父进程）。但是在非连接到终端的标准输出来说，采用的是全缓冲，此时在调用fork之前，父进程的缓冲区并未被清空（未输出），调用fork后，子进程获得父进程缓冲区的一份拷贝，最终两个进程输出时都会打印（“before fork”）。

父进程和子进程每个相同的打开的文件描述符共享一个文件表项：

![文件描述符](https://s2.ax1x.com/2019/10/01/uUgGX4.png)

<font color =red/>父进程和子进程共享同一个文件偏移量。</font>

fork之后处理文件描述符有下列两种情况：

1. 父进程等待子进程完成。此时父进程无需对其文件描述符进行任何操作。子进程处理完成后，它进行过读写的共享描述符的偏移量以及做了相应更新。
2. 父进程和子进程执行不同的代码段。此时，在fork之后子进程与父进程各自关闭不用的文件描述符，这样就不会干扰对方使用的文件描述符。

子进程继承于父进程的内容：

1. 实际用户ID，实际组ID，有效用户ID，有效组ID。
2. 附属组ID。
3. 进程组ID。
4. 会话ID。
5. 控制终端。
6. 设置用户ID标志和设置组ID标志。
7. 当前工作目录。
8. 根目录。
9. 文件模式创建屏蔽字。
10. 信号屏蔽和安排。
11. 对任一打开文件描述符的执行时关闭（close-on-exce）。
12. 环境。
13. 连接的共享存储字段。
14. 存储映射。
15. 资源限制。

父进程和子进程的区别：

1. fork返回值。
2. 进程ID。
3. 父进程ID不同。
4. 子进程的tms_utime,tms_stime,tms_cutime和tms_ustime被设置为0。
5. 子进程不继承父进程设置的文件锁。
6. 子进程未处理的闹钟被清除。
7. 子进程的未处理信号集设置为空集。

fork有以下两种用法：

1. 一个进程希望复制自己，使父进程与子进程执行不同的代码段，这在网络服务进程中是最常见的。
2. 一个进程要执行一个不同的程序。这对shell来说是常见的。

## 函数exit

进程存在八种终止方式。其中五种正常终止：

1. main函数中执行return语句，这等效于调用exit。
2. 调用exit函数。包括调用终止处理程序（atexit登记）。因为ISO C并不处理文件描述符、多进程以及作业控制，所以这一定义对于UNIX是不完整的。
3. 调用`_exit`或 `_Exit`函数。`_Exit`函数为进程提供了一种不用运行终止处理程序或者信号处理程序而终止的方法。`_exit`与`_Exit`是同义的。
4. 进程的最后一个线程在其启动例程中执行return语句。该线程的返回值不作为进程的返回值。
5. 进程的最后一个线程调用`pthread_exit`函数。

三种异常终止：

1. 调用abort。它产生SIGABRT信号，其为下一中情况的特例。
2. 当进程收到某些信号时。信号可由进程自身（如调用abort）、其他进程或者内核产生。
3. 最后一个线程对“取消”请求作出相应。

不管进程如何终止，最终都会执行内核中的同一段代码为相应进程关闭所打开的文件描述符。

对任一种终止情况，我们都希望进程能够通知父进程其是如何终止的。在任意一张情况下，都可以使用`wait`或`waitpid`函数来获得其终止信息。

“退出状态”和“终止状态”的区别：在最后调用`_exit`时，内核将退出状态转换为终止状态。如果子进程正常终止，则父进程可以获得退出状态，否则只能获得终止状态。

注意：<font color = red/>对于父进程终止的所以进程，他们的父进程都转换为`init`进程</font>。我们称这些进程被`init`收养。操作方式为：当一个进程终止时，内核检查所有活动进程，以判断其父进程是否为终止的进程，如果是则将其父进程ID更改为1。这样能够保证每个进程都存在父进程。被`init`收养的进程将会被调用`wait`函数处理。

内核为每一个终止子进程保存了一定量的信息，所以当终止进程的父进程调用`wait`或`waitpid`时，可以获得这些信息。这些信息者少包含进程ID，该进程的终止状态以及该进程使用的CPU时间总量。内核可以关闭其所打开文件和释放终止进程所使用的存储器。一个已近终止、但其父进程未对其进行善后处理（获取终止进程相关信息、释放它（信息）所占用的资源）的进程称为僵死进程。ps命令将僵死进程打印为Z。

## 函数wait和waitpid

当一个进程终止时，内核就会向其进程发送SIGCHLD信号。父进程可以选择忽略该信号或者提供一个该信号发生时即被调用执行的函数（信号处理程序）。系统默认忽略。当调用`wait`或者`waitpid`时情况：

1. 如果其所以子进程都还在运行，则阻塞。
2. 如果一个子进程已经终止，正在等待父进程获取其终止状态，则取得该子进程的终止状态立即返回。
3. 如果不存在任何子进程，则立即出错返回。

```c
#include<sys/wait.h>
pid_t wait(int *statloc);

pid_t waitpid(pid_t pid, int *statloc, int options);

//成功返回子进程ID，否则返回0

```

函数区别为：

1. 在一个子进程终止前，`wait`使调用者堵塞，而`waitpid`存在选项使调用者不堵塞。
2. `waitpid`并不等待在其调用之后的第一个终止进程，它有若干选项，可以控制等待的进程。

`statloc`是一个整型指针。如果`statloc`不是空，则将进程终止状态放在其所指向的整型中。通过宏来判断终止状态：

| 宏                     | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `WIFEXITED(status)`    | 若为正常终止进程返回状态，则为真。对于这种情况可执行`WEXITSTATUS（status）`获取子进程传递给`exit`或`_exit`参数的低八位。 |
| `WIFSIGNALED(status)`  | 若为异常终止子进程的返回状态，则为真（接到一个不捕捉的信号）。可执行`WTERMSIG（status）`，获得子进程终止的信号编号。有些实现存在`WCOREDUMO（status）`，可通过次宏来判断是否生成了终止进程的core文件。 |
| `WIFSTOPPED(status)`   | 若为当前暂停的子进程返回的状态，则为真。此时可执行`WSTOPSIG(status)`获取使紫禁城暂停的信号编号。 |
| `WIFCONTINUED(status)` | 若在作用控制暂停后已近继续的子进程返回状态，则为真（仅用于waitpid）。 |



`waitpid`等待特定进程，其中pid参数用法为：

| pid     | 说明                                   |
| ------- | -------------------------------------- |
| pid==-1 | 等待任一进程，此时与wait等效。         |
| pid>0   | 等待进程ID与pid一致的子进程。          |
| pid==0  | 等待组ID等于调用进程组ID的任一子进程。 |
| pid<-1  | 等待组ID等于pid绝对值的任一子进程。    |

如果`waitpid`指定的进程不存在或者不是调用者的子进程就会报错。

通过options可以进一步控制waitpid操作，该参数要么是0（0是下面三个参数相与的结果），要么是下列参数位运算的结果：

| 常量       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| WCONTINUED | 若实现支持作业控制，那么由pid指定的任一子进程在停止后已经继续，但其状态尚未报告，则返回其状态。 |
| WNOHANG    | 若由pid指定的子进程并不是立即可用的，则waitpid不阻塞，此时返回值为0。 |
| WUNTRACED  | 若实现支持作业控制，而由pid指定的任一子进程已经处于停止状态，而且其状态自停止以来还未报告过，则返回其状态。WIFSTOPPED宏确定一个返回值是否是一个停止的子进程。 |

打印终止状态程序（将在下面经常用到）：

```c
#include"apue.h"
#include<sys/wait.h>

void pr_exit(int status)
{
    if(WIFEXITED(status))
    {
        printf("normal termination, exit status=%d\n",WEXITSTATUS(status));
    }
    else
    {
        if(WIFSIGNALED(status))
        {
            printf("abnormal termination,signal number =%d%s\n",WTERMSIG(status),
            #ifdef WCOREDUMP
            WCOREDUMP(status)?" (core file generated)":"");
            #else
            "");
            #endif
        }
        else
        {
            if(WIFSTOPPED(status))
            {
                printf("child stopped,signal number = %d\n", WSTOPSIG(status));
            }
        }
    }
    
}

```

利用上述函数展示终止状态：

```c
#include"apue.h"
#include<sys/wait.h>
int main(void)
{
    pid_t pid;
    int status;
    if((pid=fork())<0)
    {
        err_sys("fork error\n");
    }
    else
    {
        if(pid==0) //终止子进程
        {
            exit(7);
        }
    }
    if(wait(&status)!=pid) //获取子进程终止状态
    {
        err_sys("wait error\n");
    }
    else{
        pr_exit(status);
    }

    if((pid=fork())<0)
    {
        err_sys("fork error\n");
    }
    else
    {
        if(pid==0) //终止子进程
        {
            abort();
        }
    }
    if(wait(&status)!=pid) //获取子进程终止状态
    {
        err_sys("wait error\n");
    }
    else{
        pr_exit(status);
    }

    if((pid=fork())<0)
    {
        err_sys("fork error\n");
    }
    else
    {
        if(pid==0) //终止子进程
        {
            status/=0;
        }
    }
    if(wait(&status)!=pid) //获取子进程终止状态
    {
        err_sys("wait error\n");
    }
    else{
        pr_exit(status);
    }
    exit(0);
}

```

再来考虑僵死进程，如果一个进程fork了一个子进程，但是并不像自己去等待进程终止也不想让其成为僵死进程直到父进程终止。此时，一个好的方式是调用两次fork。第一次创建一个子进程，第二次使用子进程再次创造一个子进程的子进程并且立即终止子进程，这样父进程不用等待子进程可以直接调用`wait`。而对于子进程的子进程来说，其父进程已经终止，其会被init收养，当退出时，init将会处理，使其不会成为一个僵死进程。例如下面的程序：

```c
#include"apue.h"
#include<sys/wait.h>
int main()
{
    pid_t pid;
    if((pid = fork())<0)
    {
        err_sys("fork error\n");
    }
    else{
        if(pid==0) // 子进程
        {
            if((pid=fork())<0)
            {
                err_sys("fork error\n");
            }
            else
            {
                if(pid>0) // 子进程终止，使子进程的子进程被init收养
                exit(0);
                else{
                    sleep(2); //保证子进程终止
                    printf("second child,parent pid = %ld\n", (long)(getppid()));
                    exit(0); //子进程的子进程终止，被init处理
                }
            }
        }
    }
    if(waitpid(pid, NULL,0)!=pid) //等待子进程
    {
        err_sys("waitpid error\n");
    }
    exit(0);
}

```

运行：

```shell
$ ./fork_seconds.o 
$ second child,parent pid = 1

```

当原先的进程（父进程）终止时，shell打印其提示符，这在子进程的子进程打印其父进程ID之前。

## 函数waitid、wait3和wait4

```c
#include<sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);

```

`waitid`允许指定一个要等待的进程ID。使用两个单独的参数表示要等待的子进程所属类型。idtype选项如下：

| 常量   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| P_PID  | 等待一个特定进程：id为要等待的进程ID。                       |
| P_PGID | 等待一特定进程组的任一子进程：id包含要等待子进程的进程组ID。 |
| P_ALL  | 等待任一进程，忽略id。                                       |

options参数是下列标志的位运算：

| 常量       | 说明                                                 |
| ---------- | ---------------------------------------------------- |
| WCONTINUED | 等待一进程，它曾经被停止，此后又已继续，但尚未报告。 |
| WEXITED    | 等待已退出的进程。                                   |
| WNOHANG    | 如无可用的子进程退出状态，立即返回而不阻塞。         |
| WNOWAIT    | 不破坏子进程退出状态。                               |
| WSTOPPED   | 等待一个子进程，它已经停止但尚未报告。               |

`wait3`和`wait4`在上述几个`wait`基础上多加了一个参数，使得内核返回由终止进程及其所有子进程使用的资源概况。

```c
#include<sys/wait.h>
#include<sys/types.h>
#include<sys/time.h>
#include<sys/resource.h>

pid_t wait3(int *statloc, int option, struct rusage *rusage);

pid_t wait4(pid_t pid, int *statloc, int option, struct rusage *rusage);

//成功返回进程ID，出错返回-1

```

## 竞争条件

当多个进程都企图对共享数据进行某些处理，而最后的处理的结果又取决于进程运行的顺序时，我们认为发生了竞争条件。如果一个进程要等待子进程终止，则它必须使用`wait`函数中的一个。如果一个子进程要等待父进程的终止，可以使用下列的代码：

```c
while(getppid()!=1)
    sleep(1);

```

该方式被称为轮询，问题是浪费了CPU时间。

为了避免竞争和轮询，在多个进程之间需要有某种信号发送和接收的方法。可以使用信号机制，也开始使用进程间通信。均会在后面介绍。这里为了展示竞争条件，我们先使用下列几个函数，这些函数的实现都会在接下来的内容中讲解：

```c
#include"apue.h"
TELL_WAIT(); //设置，为TELL_xxx和WAIT_xx做准备。
if((pid=fork())<0)
{
    err_sys("fork error\n");
}
else
{
    if(pid==0)
    {
        /*子进程处理程序*/
        TELL_PARENT(getppid); //通知父进程自己处理完成
        WAIT_PARENT(); //等待父进程
        /*子进程等待父进程后接下来要处理的程序*/
        exit(0);
    }
    /*父进程要处理的代码*/
    TELL_CHILD(pid); //告诉子进程自己处理完成
    WAIT_CHILD(); //等待子进程
    /* 父进程再等待子进程后需要处理的代码*/
    exit(0);
}

```

出现竞争条件的程序：

```c
#include"apue.h"

static void charatatime(char *);

int main()
{
    pid_t pid;
    if((pid=fork())<0)
    {
        printf("fork error\n");
    }
    else{
        if(pid==0)
        {
            charatatime("output from child\n");
        }
        else{
            charatatime("output from parent\n");
        }
    }
    exit(0);
}

static void charatatime(char *str)
{
    char *ptr;
    int c;
    setbuf(stdout, NULL);//设置标准输出无缓冲，更方便看到竞争
    for(ptr=str;(c=*ptr++)!=0;)
    {
        putc(c,stdout);
    }
}

```

执行：

```shell
$ ./compare.o 
output from parenotu
tput from child

```

通过使用TELL和WAIT函数来解决竞争，程序变为：

```c
#include"apue.h"

static void charatatime(char *);

int main()
{
    pid_t pid;
    
    TELL_WAIT();

    if((pid=fork())<0)
    {
        printf("fork error\n");
    }
    else{
        if(pid==0)
        {
            WAIT_PARENT();
            charatatime("output from child\n");
        }
        else{
            charatatime("output from parent\n");
            TELL_CHILD(pid);
        }
    }
    exit(0);
}

static void charatatime(char *str)
{
    char *ptr;
    int c;
    setbuf(stdout, NULL);//设置标准输出无缓冲，更方便看到竞争
    for(ptr=str;(c=*ptr++)!=0;)
    {
        putc(c,stdout);
    }
}

```

这里先让父进程打印再打印子进程。这里的程序应该是无法执行的，这几个函数作者只在apue中给出了定义，并未实现，应该是希望通过之后的学习自己实现。

## 函数exec

当进程调用`exec`函数时，该进程执行的程序完全替换为新程序，而新程序则从main函数开始执行。因为`exec`并不创建新的进程，所以前后的进程ID并未改变。`exec`只是用磁盘上一个新的程序替换的当前进程的正文段、数据段、堆段和栈。<font color =red/>基本进程原语是：使用`fork`创建新进程，用`exec`初始执行新的程序。`exit`和`wait`函数处理终止和等待。</font>

```c
#include<unsid.h>

int execl(const char *pathname, const char *arg0, .../*(char *)0*/);

int execv(const char *pathname, char *const argv[]);

int execle(const char *pathname, const char *arg0, .../*(char *)0,char *const envp[]*/);

int execve(const char *pathname, char *coonst argv[], char *const envp[]);

int execlp(const char *filename, const char *arg0, .../*(char*)0*/);

int execvp(const char *filename, const char *argv[]);

int fexecve(int fd, char *const argv[], char *const envp[]);

//函数成功，不返回，否则返回-1

```

这七个函数之间的第一个区别为：前4个函数取路径名作为参数，后两个则使用文件名作为参数，最后一个取文件描述符作为参数。当使用filename时：

1. 如果filname中存在`/`，就被视为路径名。
2. 否则按照PATH环境变量，在其所指定的各个目录下搜索可执行文件。

```shell
PATH:
/usr/bin/:/usr/local/cuda-9.0/bin:/home/chst/.local/bin:/usr/bin/:/usr/local/cuda-9.0/bin:/home/chst/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games

```

如果`execlp`或`execvp`使用路径前缀中的一个找到一个可执行文件，但该文件不是连接编译器产生的机器可执行文件，则就认为是shell脚步，于是试着调用`/bin/sh`，并以filename作为输入。

第二个区别是：参数表的传递，`l`表示list，`v`表示矢量vector。`execl`、`execlp`和`execle`要求将新程序的每个命令行参数都说明为一个单独参数。这种参数表以空指针结尾。另外四个则是先构造一个指向各个参数的指针数组，然后传递数组地址。

第三个区别：向新进程传递参数表。以`e`结尾的函数传递一个指向环境字符串指针数组的指针。其他四个函数使用调用进程的`environ`变量为新程序赋值现有环境。

执行`exec`后，新进程从调用的进程继承了下列属性：

1. 进程ID和父进程ID。
2. 实际用户ID和实际组ID。
3. 附属组ID。
4. 进程组ID。
5. 会话ID。
6. 控制终端。
7. 闹钟尚余留时间。
8. 当前工作目录。
9. 根目录。
10. 文件模式创建屏蔽字。
11. 文件锁。
12. 进程信号屏蔽。
13. 未处理信号。
14. 资源限制。
15. nice值。
16. `tms_utime`、`tms_stime`、`tms_cutime`以及`tms_cstime`。

对于打开的文件的处理与每个描述符的执行时关闭标志相关。详见第三章的`fcntl`函数节。如果设置了该标志，则执行`exec`时关闭，否则保持打开。对于打开的目录，POSIX.1要求`exec`时关闭。

`exec`前后实际用户ID和时间组ID保持不变，而有效ID是否改变取决于所执行的程序的设置用户ID位和设置组ID位。

七个`exec`只有`execve`是内核调用的，另外的函数都是库函数，它们最终都要调用内核函数。其关系如下：

![exec](https://s2.ax1x.com/2019/10/01/uUg31U.png)

例：

```c
#include"apue.h"
#include<sys/wait.h>

char *env_init[] = {"USER=unknow","PATH=/tmp", NULL};

int main(void)
{
    pid_t pid;
    if((pid=fork())<0)
    {
        err_sys("fork error\n");
    }
    else{
        if(pid==0){
            if(execle("/home/chst/study_file/unix编程/test.o", "test.o", "myarg1", "MY ARG2", (char *)0, env_init)<0)
            {
                err_sys("execle error\n");
            }
        }
    }
    if(waitpid(pid,NULL, 0)<0)
    {
        err_sys("waitpid error\n");
    }
    
    if((pid=fork())<0)
    {
        err_sys("fork error\n");
    }
    else{
        if(pid==0){
            if(execlp("/home/chst/study_file/unix编程/test.o", "test.o", "myarg1", "MY ARG2", (char *)0)<0)
            {
                err_sys("execle error\n");
            }
        }
    }
    exit(0);
}

```

其中`/home/chst/study_file/unix编程/test.o`代码为：

```c
#include"apue.h"
int main(int argc, char *argv[])
{
    int i;
    char **ptr;
    extern char **environ;
    for(i=0;i<argc;i++)
    {
        printf("argv[%d]: %s\n",i, argv[i]);
    }

    for(ptr = environ; *ptr!=0; ptr++)
    {
        printf("%s\n", *ptr);
    }
    exit(0);
}

```

## 更改用户ID和更改组ID

在UNIX中，特权以及访问控制是基于用户ID和组ID的。一般而言，在设计应用时，我们总是试图使用最小特权模型。依照此模型，我们总是给程序完成任务所需要的最小特权。

可以使用`setuid`设置实际用户ID和有效用户ID，用`setgid`函数来设置实际组ID和有效组ID：

```c
#include<unistd.h>

int setuid(uid_t uid);

int setgid(gid_t gid);
//成功返回0，否则返回-1.

```

更改用户ID规则为：

1. 若进程拥有超级用户权限，则`setuid`函数将实际用户ID、有效用户ID以及保存的设置用户ID设置为uid。
2. 若进程没有超级用户权限，但uid等于实际用户ID或者保存用户ID，则`setuid`只将有效用户ID设置为uid而不改变实际用户ID和保存的设置用户ID。
3. 如果上面两个都不满足，则erron设置为EPERM，并返回-1。

更改3个用户ID的方法：

| ID               | exec             | exec                   | setuid（uid） | setuid（uid） |
| ---------------- | ---------------- | ---------------------- | ------------- | ------------- |
|                  | 设置用户ID位关闭 | 设置用户ID为开启       | 超级用户      | 非特权用户    |
| 实际用户ID       | 不变             | 不变                   | 设为uid       | 不变          |
| 有效用户ID       | 不变             | 设置为程序文件的用户ID | 设为uid       | 设为uid       |
| 保存的设置用户ID | 从有效用户ID复制 | 从有效用户ID复制       | 设为uid       | 不变          |

函数`setreuid`和`setregid`用来交换实际用户ID和有效用户ID。

```
#include<unistd.h>
int setreuid(uid_t ruid, uid_t euid);
int setregid(gid_t rgid, gid_t egid);

```

如果任一参数值为-1，表示相应的ID应当保持不变。任意一个非特权用户都可以交换实际用户ID和有效用户ID。

函数`seteuid`和函数`setegid`类似于`setuid`和`setgid`，但只更改有效用户ID和有效组ID。

```c
#include<unistd.h>
int seteuid(uid_t uid);

int setegid(gid_t gid);
//成功返回0，否则返回-1

```

不同函数更改ID的方式：

![ID](https://s2.ax1x.com/2019/10/01/uUg8cF.png)

## 解释器文件

解释器文件是文本文件，其起始行的形式为：

```
#! pathname [optional-agrument]

```

最常见的解释器文件以下列行开始：

```
#! /bin/sh

```

pathname通常是绝对路径。内核使调用`exec`函数的进程实际执行的不是该解释器文件，而是在该解释器文件第一行中pathname所指定的文件。

实例：

```c
#include"apue.h"
#include<sys/wait.h>

int main(void)
{
    pid_t pid;
    if((pid=fork())<0)
    {
        err_sys("fork error");
    }
    else{
        if(pid==0)
        {
            if(execl("/home/chst/study_file/unix编程/inter", "inter", "myarg1", "MY ARG2", (char *)0)<0)
            {
                err_sys("excel error");
            }
        }
    }
    if(waitpid(pid, NULL, 0)!=pid)
    {
        err_sys("waitpid error");
    }
    exit(0);
}

```

其中`/home/chst/study_file/unix编程/inter`内容为：

```
#! /home/chst/study_file/unix编程/test.o foo

```

注意：需要使用chmod设置`inter`可执行

`/home/chst/study_file/unix编程/test.o`代码与上面第七节的一致，打印参数和环境表。

执行结果：

```
$ ./interpreter1.o 
argv[0]: /home/chst/study_file/unix编程/test.o
argv[1]: foo
argv[2]: /home/chst/study_file/unix编程/inter
argv[3]: myarg1
argv[4]: MY ARG2
......

```

从结果中可以看出，内核调用`exec`解释器时，`argv[0]`是解释器的pathname，`argv[1]`是解释器的可选参数。其余的参数是`execl`输入的参数。内核取第一个参数是pathname，而不是`test.o`。

在解释器后可以跟随可选参数，例如可以执行python脚步：

```c
#include"apue.h"
#include<sys/wait.h>

int main(void)
{
    pid_t pid;
    if((pid=fork())<0)
    {
        err_sys("fork error\n");
    }
    else{
        if(pid == 0)
        {
            if(execl("/home/chst/study_file/unix编程/inter", "inter", (char *)0)<0)
            {
                err_sys("execl error\n");
            }
        }
    }
    if(waitpid(pid, NULL, 0)!=pid)
    {
        err_sys("waitpid error\n");
    }
    printf("C++ printf pid = %ld\n", (long)pid);
    exit(0);
}

```

其中`/home/chst/study_file/unix编程/inter`内容为：

```c
#! /usr/bin/python /home/chst/study_file/unix编程/getpid.py

```

其中`/home/chst/study_file/unix编程/getpid.py`代码为：

```python
import os
w = os.getpid()
print("python printf pid = ",w,'\n')

```

执行后输出为：

```
$ ./interpreter.o 
python printf pid =  26401 

C++ printf pid = 26401

```

这里也进一步验证了执行`exec`并不会额外创建新的进程。

之所以使用解释器有如下理由：

1. 有些程序是用某些语言写的脚本，解释器可以将这个事实隐藏起来。
2. 解释器在效率上提供了好处。
3. 解释器脚本使我们可以使用除了`/bin/sh`以外的其他`shell`来编写`shell`脚本。当`execlp`找到一个非机器可执行文件时，它总是调用`/bin/sh`来解释执行该文件。但，用解释脚本则可以简单的写成：`#! /bin/csh`(在解释文件后跟随C shell)。

## 函数system

```c
#include<stdlib.h>

int system(const char *cmdstring);

```

如果cmdstring是一个空指针，则仅当命令处理程序可用时，system返回非0，这一特征可以确定一个给定的操作系统上是否支持system。

由于system在其实现中调用了`fork`、`exec`和`waitpid`，因此返回值有三种。

1. `fork`失败或者`waitpid`返回除EINTR之外的出错，则system返回-1，并且设置erron以指示错误。
2. 如果`exec`失败（表示不能执行`shell`），则其返回值如同`shell`执行了`exit（127）`一样。
3. 否则三个函数都成功，那么system的返回值是`shell`的终止状态。

system函数的一种实现：

```c
#include<sys/wait.h>
#include<errno.h>
#include<unistd.h>

int system(const char *cmdstring)
{
    pid_t pid;
    int status;
    if(cmdstring == NULL)
    {
        return 1;
    }

    if((pid = fork())<0)
    {
        status = -1;
    }
    else{
        if(pid==0)
        {
            execl("/bin/sh", "sh", "-c", cmdstring, (char*)0); //如果程序正常执行，则会自己调用exit函数，否则使用下一条命令。
            _exit(127);
        }
        else{
            while (waitpid(pid, &status, 0)<0)
            {
                if(errno != EINTR)
                {
                    status = -1;
                    break;
                }
            }
        }
    }
    return status;
}

```

`shell`的`-c`选项告诉`shell`程序取下一个命令行参数（这里是cmdstring）作为命令行输入。

使用上面的system函数测试：

```c
#include<sys/wait.h>
#include<errno.h>
#include<unistd.h>
#include"apue.h"
#include"pr_exit.h"
#include"system.h"

int main(void)
{
    int status;
    if((status = system("date"))<0)
    {
        err_sys("system() error");
    }
    pr_exit(status);
    if((status = system("nosuchcommand"))<0)
    {
        err_sys("system() error");
    }

    pr_exit(status);

    if((status = system("who; exit 44"))<0)
    {
        err_sys("system() error");
    }

    pr_exit(status);

    exit(0);
}

```

执行结果：

```shell
$ ./system.o 
Thu  3 Oct 01:05:35 +08 2019
normal termination, exit status=0
sh: 1: nosuchcommand: not found
normal termination, exit status=127
chst     :0           2019-10-02 23:24 (:0)
normal termination, exit status=44

```

注意：设置用户ID或设置用户组ID的程序决不应该调用system函数。

## 进程标识

```
getpwuid(getuid());

```

获得运行该程序用户的登录名。

```c
#include<unistd.h>

char *getlongin(void);

//正常返回指针，否则返回NULL

```

还有该函数可以获得用户登录时使用的名字。

获得登录名后可以使用`getpwnam`（第六章）获得口令文件。

## 进程调度

调度策略和调度优先级由内核决定。进程可以通过调整`nice`值来选择以更低优先级运行（通过调整`nice`值降低对CPU的占有）。只有特权进程允许提高调度权限。

`nice`的值在`0～（2*NZERO-1）`之间。`nice`值越小，优先级越高。

通过`nice`函数，我们可以更改进程`nice`值：

```c
#include<unistd.h>
int nice(int incr);
//incr为在当前nice值基础上增加的数量。正确返回返回新的nice值，否则返回-1

```

由于-1是合法返回值，，因此判断错误应该使用`errno`

`getpriority`函数获得进程`nice`值，还可以获取一组相关进程的`nice`值：

```c
#include<sys/resource.h>

int getpriority(int which, id_t who);
//正常返回nice值，错误返回-1

```

`which`参数为：`PRIO_PROCESS`表示进程，`PRIO_PGRP`表示进程组，`PRIO_USER`表示用户ID。`which`控制参数`who`如何解释，`who`参数选择感兴趣的一个或多个进程。如果`who`参数为0，表示调用进程、进程组或者用户（取决与`which`）。如果`which`参数设置为`PRIO_USER`且`who`为0，使用调用进程的实际用户ID。如果`which`作用于多个进程，则返回进程中优先级最高的（nice最小的）。

`setpriority`函数用于为进程、进程组和属于特定用户ID的所以进程设置优先级：

```c
#include<sys/resource.h>

int setpriority(int which, id_d who, int value);

```

例：

```c
#include"apue.h"
#include<sys/time.h>
#include<errno.h>

#if defined(MACOS)
#include<sys/syslimits.h>
#elif defined(SOLARIS)
#include<limits.h>
#elif defined(BSD)
#include<sys/param.h>
#endif

unsigned long long count;
timeval end;

void checktime(char *str)
{
    timeval tv;
    gettimeofday(&tv, NULL);
    if(tv.tv_sec > end.tv_sec && tv.tv_usec > end.tv_usec){
        printf("%s count = %lld\n", str, count);
        exit(0);
    }
}

int main(int argc, char *argv[])
{
    pid_t pid;
    char *s;
    int nzero,ret;
    int adj = 0;
    setbuf(stdout, NULL);

    #if defined(NZERO)
    nzero = NZERO;
    #elif defined(_SC_NZERO)
    nzero = sysconf(_SC_NZERO);
    #else
    #error NZERO undefined
    #endif

    printf("NZERO = %d\n", nzero);
    if(argc == 2)
    {
        adj = strtol(argv[1], NULL, 10);//字符串转换为长整型
    }
    gettimeofday(&end, NULL);
    end.tv_sec += 3;

    if((pid=fork())<0)
    {
        err_sys("fork error");
    }
    else{
        if(pid == 0){
            s = "child";
            printf("current nice value in child is %d, adjusting by %d\n", nice(0)+nzero, adj);
            errno = 0;
            if((ret=nice(adj))==-1 && errno != 0)
            {
                err_sys("child set nice error");
            }
            printf("child now nice value is %d\n", ret+nzero);
        }
        else{
            s = "parent";
            printf("current nice value of parent is %d\n", nice(0)+nzero);
        }
        while(1)
        {
            if(++count==0)
            {
                err_sys("%s conter warp",s);
            }
            checktime(s);
        }
    }
}


```

例子通过将子进程的nice增加来展示两个进程累加次数，但在我的计算机上好像没啥区别：

```shell
$ ./nice.o 20
NZERO = 20
current nice value of parent is 20
current nice value in child is 20, adjusting by 20
child now nice value is 39
child count = 590759158
parent count = 589967528

$ ./nice.o 20
NZERO = 20
current nice value of parent is 20
current nice value in child is 20, adjusting by 20
child now nice value is 39
parent count = 590076641
child count = 590084625

```

## 进程时间

我们可以度量的有三个时间：墙上时钟时间、用户CPU时间和系统CPU时间。任一进程都可以调用`times`函数获得自己和已经终止子进程的上述值：

```c
#include<sys/times.h>
clock_t times(struct tms *buf);
//成功返回流逝的墙上时钟时间（以时钟滴答数为计数单位），出错返回-1

```

`buf`结构为：

```c
struct tms{
    clock_t tms_utime;//用户CUP时间
    clock_t tms_stime;//系统CPU时间
    clock_t tms_cutime;//已经终止的子进程的CPU时间
    clock_t tms_cstime;//已经终止的子进程的系统时间
}

```

例程：

```c
#include"apue.h"
#include<sys/times.h>
#include"pr_exit.h"

static void pr_times(clock_t ,tms *, tms *);
static void do_cmd(char *);

int main(int argc, char*argv[])
{
    int i;
    setbuf(stdout, NULL);
    for(i=1;i<argc; i++)
    {
        do_cmd(argv[i]);
    }
    exit(0);
}

static void do_cmd(char *cmd)
{
    tms tmsstart, tmsend;
    clock_t start, end;
    int status;

    printf("\ncommend:%s\n",cmd);
    if((start=times(&tmsstart))==-1)
    {
        err_sys("times error");
    }

    if((status = system(cmd))<0)
    {
        err_sys("system error");
    }

    if((end = times(&tmsend))<0)
    {
        err_sys("times error");
    }
    pr_times(end-start, &tmsstart, &tmsend);
    pr_exit(status);
}

static void pr_times(clock_t real, tms *tmsstart, tms *tmsend)
{
    static long clktck = 0;
    if(clktck == 0)
    {
        if((clktck = sysconf(_SC_CLK_TCK))<0) //获取每秒时钟滴答数
        {
            err_sys("sysconf error");
        }
    }

    printf(" real: %7.2f\n", real/double(clktck));
    printf(" user: %7.2f\n", (tmsend->tms_utime - tmsstart->tms_utime)/(double)clktck);
    printf(" sys: %7.2f\n", (tmsend->tms_stime-tmsstart->tms_stime)/(double)clktck);
    printf(" child user: %7.2f\n", (tmsend->tms_cutime-tmsstart->tms_cutime)/double(clktck));
    printf(" child sys: %7.2f\n", (tmsend->tms_cstime-tmsstart->tms_cstime)/(double)clktck);
}

```

该程序执行输入参数的命令行命令并输出对于时间信息：

```
$ ./ptime.o "sleep 5" "date" "man bash > /home/chst/study_file/unix编程/output.txt"

commend:sleep 5
 real:    5.01
 user:    0.00
 sys:    0.00
 child user:    0.00
 child sys:    0.00
normal termination, exit status=0

commend:date
Thu  3 Oct 13:54:01 +08 2019
 real:    0.00
 user:    0.00
 sys:    0.00
 child user:    0.00
 child sys:    0.00
normal termination, exit status=0

commend:man bash > /home/chst/study_file/unix编程/output.txt
 real:    0.21
 user:    0.00
 sys:    0.00
 child user:    0.33
 child sys:    0.04
normal termination, exit status=0

```

书中题目8.6：

```c
#include<sys/wait.h>
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<string.h>

int main()
{
    pid_t pid;
    if((pid = fork())<0)
    {
        printf("fork error\n");
    }
    else{
        if(pid==0)
        {
            exit(0);
        }
        else{
            sleep(2); //等待子进程结束
            char w[100] = "ps ";
            int n = strlen(w);
            printf("%d\n", n);
            char id[10];
            int i=0;
            long int p= long(pid);
            while(p)
            {
                id[i++] = char(p%10 + '0');
                p /= 10;
            }
            for(int k = n; k<n+i;k++)
            {
                w[k] = id[n+i-1-k];
            }
            w[n+i] = '\0';
            printf("command = %s\n", w);
            int status = 0;
            if((status = system(w))<0)
            {
                printf("system error\n");
            }
        }
    }
    exit(0);
}

```

# 第九章 进程关系

## 终端登录

系统管理者创建通常名为`/etc/ttys`的文件（我的Ubuntu并没有这个文件），其中每个终端设备都有一行，每一行说明设备名和传到`getty`程序的参数。当程序自举时，内核创建进程为1的`init`进程。`init`进程使系统进入多用户模式。`init`读取文件`/etc/ttys`，对每一个允许登录的终端设备，`init`调用一次`fork`，所生成的子进程`exec getty`程序。如下图9-1：

![init](https://s2.ax1x.com/2019/10/01/uUgQhV.png)

`getty`对终端设备调用`open`函数，以读写方式将终端打开。一旦设备被打开，文件描述符0、 1、 2就会被设置到该设备上。然后`getty`输出`login：`之类提示符等待用户输入用户名。随后以类似于下面的方式调用`login`程序：

```
execle("/bin/login", "login", "-p", username, (char*)0, envp);

```

`init`以空环境表调用`getty`。`getty`以终端和在`gettytab`中说明的环境字符串为`login`创建一个环境。`-p`标志通知`login`保留传递给它的环境，也可以将其他环境字符串加到该环境中，但不要替换它。上图9-2显示了`login`刚被调用后这些进程的状态。

图9-2下面的三个进程的ID相同，因为调用`exec`并不创建新的进程。

`login`会调用`getpwnam`获取响应用户的口令文件登录项。调用`getpass`提示输入`password`。对比用户输入密码与登录项的`pw_passwd`是否一致。如果多次都无效，则`login`以1调用`exit`表示登录失败。父进程`init`了解到后，将再次调用`fork`，其后执行`getty`。

用户正常登录，`login`就将完成如下工作：

1. 将当前工作目录更改为该用户的起始目录（`chdir`）。

2. 调用`chown`更改该终端的所有权，使登录用户成为它的所有者。

3. 将对该终端设备的访问权限改变为“用户读和写”。

4. 调用`setgid`及`initgroups`设置进程的进程组ID。

5. 用`login`得到的所以信息初始化环境：起始目录（`HOME`），shell（`SHELL`），用户名（`USER`和`LOGNAME`）以及一个系统默认路径（`PATH`）。

6. `login`进程更改为登录用户的用户ID（`setuid`），并调用该用户的登陆shell，其方式类似于：

   ```c
   execl("/bin/sh", "-sh",(char*)0);
   
   ```

至此，登录shell开始运行。其父进程ID是`init`，所以当此登录shell终止时，`init`会得到通知（SIGCHILD信号），它会重复上述全部过程。如下图：

![login](https://s2.ax1x.com/2019/10/01/uUgN7R.png)

## 网络登录

通过串行登录至系统和经由网络登陆至系统两者主要区别是：网络登录时，在终端和计算机之间的连接不再是点到点的。在网络登陆下，`login`仅仅是一种可用的服务，这与其他网路服务（如FTP或SMTP）的性质相同。为了同一个软件既能处理终端登录，又能处理网络登录，系统使用了一种称为伪终端的软件驱动程序。

在上一节的终端登录中，`init`知道那些终端设备可以用来进行登录，并为每一个设备生成一个`getty`进程。但网络登录情况下，所以登录均由内核的网路接口驱动程序，而且事先并不知道会有多少这样的登录。因此必须等待一个网路连接请求的到达，而不是使一个进程等待每一个可能的登录。

在BSD中，有一个`inetd`进程（因特网超级服务器），它等待大多数网络连接。`init`调用shell，使其执行shell脚本`/etc/rc`。由此shell脚本启动一个守护进程`inetd`。一旦此shell脚本终止，`inetd`的父进程就变成`init`。`inetd`等待TCP/IP连接请求达到主机，而当一个连接请求到达主机时，它执行一次fork，然后子进程`exec`适当的程序。

假定一个TELNET服务进程的TCP连接请求到达。TELNET是使用TCP协议的远程登录应用程序。客户进程通过`telnet hostname`启动登录过程。该客户进程打开一个到`hostname`主机的TCP连接，在`hostname`主机上启动的程序被称为TELNET服务进程。然后，客户进程和服务进程之间通过使用TELNET应用协议通过TCP连接交换数据。下图展示了这一过程：

![TELNET](https://s2.ax1x.com/2019/10/01/uUgtB9.png)

随后`telnetd`进程打开一个伪终端，并`fork`分成两个进程。父进程处理通过网络传输的信息，子进程执行`login`程序。父进程和子进程通过终端相连。在调用`exec`之前，子进程使其文件描述符0、 1、 2与伪终端相连。如果登录正确则执行上一节所述步骤。然后`login`调用`exec`将其自身替换为登录用户的登录shell。下图展示了这一过程：

![login](https://s2.ax1x.com/2019/10/01/uUgN7R.png)

注意：当通过终端或网络登录时，我们得到一个登录shell，其标准输入、标准输出、标准错误要么连接到一个终端，要么连接到伪终端设备上。

## 进程组

每个进程除了有一个进程ID之外，还属于一个进程组进程组是一个或多个进程的合集，通常他们是在同一作业中结合起来的，<font color=red/>同一进程组中的进程接收来自统一终端的各种信号。</font>每个进程组有一个唯一的进错组ID。进程组ID是一个正整数，保存在`pid_t`数据类型中。函数`getpgrp`返回调用进程的进错组ID：

```c
#include<unistd.h>

pid_t getpgrp(void);
//成功返回进程组ID，否则返回-1

```

`getpgid`函数可以传递进程ID，获取进程组ID：

```c
#include<unistd.h>

pid_t getpgid(pid_t pid);
//成功返回进程组ID，否则返回-1

```

如pid是0，返回调用进程的进错组ID，于是`getpgid(0) = getpgrp()`。

每个进程组存在一个组长进程。组长进程的进程组ID等于其进程ID。

注意：<font color = red/>进程组组长可以创建一个进程组（`fork`生成的子进程其进程组ID也会从父进程中继承过来，且`exec`不改变进错组ID）、创建该组中的进程，然后终止。只要在某一个进程组中有一个进程存在，则该进程组就存在，这与其组长进程是否终止无关。</font>从进程组创建开始到其中最后一个进程离开为止的时间称为进程组的生命周期。某个进程组的最后一个进程可以终止，也可以转移到另一个进程组中。

进程调用`setpgid`可以加入一个现有的进程组或者创建一个新的进程组：

```c
#include<unistd.h>
int setpgid(pid_t pid, pid_t pgid);
//成功返回0，失败返回-1

```

`setpgid`将`pid`进程的进程组ID设置为`pgid`。如果两个参数相等，则由`pid`指定的进程变成进程组组长。如果`pid`是0，则使用调用者的进程ID。如果`pgid`是0，则由`pid`指定的进程ID作为进程组ID。

一个进程只能为它自己或它的子进程设置进程组ID。在它的子进程调用`exec`后，它就不再更改子进程的进程组ID。

在大多数作业控制shell中，在`fork`之后调用此函数，使父进程设置子进程的进程组ID，并且也使子进程设置其自己的进程组ID。这两个调用有一个是冗余的，但为了保证设置确实发生了。

## 会话

会话是一个或多个进程组的集合，例如下图：

![会话](https://s2.ax1x.com/2019/10/01/uUgaA1.png)

通常是由shell的管道将几个进程编写成一组的，上图的安排可能是下面的命令：

```
proc1 | proc2 &
proc3 | proc4 | proc5

```

进程调用`setsid`函数建立一个新的会话：

```c
#include<unistd.h>

pid_t setsid(void);
//成功返回进程组ID，否则返回-1

```

如果调用该函数的进程不是一个进程组的组长，则此函数创建一个新会话。具体会发生下面三件事：

1. 该进程变成新会话的会话首进程（会话首进程是创建该会话的进程）。此时，该进程是新会话的唯一进程。
2. 该进程成为一个新进程组的组长进程，进程组ID为该进程的进程ID。
3. 该进程没有控制终端（下一节讨论控制终端）。如果在调用`setsid`之前该进程有一个控制终端，那么这种联系也会被切断。

如果该调用进程是一个进程组的组长，则会报错。为了防止这种情况发生，一般先调用`fork`创建新进程，然后使父进程终止，在子进程中调用该函数。

将会话首进程的进程ID视为会话ID。`getsid`获得会化首进程的进程组ID：

```c
#include<unistd.h>

pid_t getsid(pid_t pid);
//成功返回会话首进程的进程组ID，否则返回-1

```

如果`pid`是0，则返回调用进程的会话首进程的进程组ID。

## 控制终端

会话和进程组还有如下特性：

1. 一个会话可以有一个控制终端。这通常是终端设备或伪终端设备。
2. 建立与控制终端连接的会话首进程被称为控制进程。
3. 一个会话中的几个进程组可被分为一个前台进程组和一个或者多个后台进程组。
4. 如果一个会话有一个控制终端，则它有一个前台进程组，其它进程组为后台进程组。
5. 无论何时键入终端的中断键（常常是`Delete`或者`Ctrl+C`，都会将中断信号发送至前台进程组的所以进程。
6. 无论何时键入终端的退出键（常常是`Ctrl+\`），都会将退出信号发送到前台进程组的每个进程。
7. 如果终端接口检测到调制解调器（或网络）已经断开连接，则将挂断信号发送至控制进程（会话首进程）。

特性展示如下图：

![终端](https://s2.ax1x.com/2019/10/01/uUgdtx.png)

通常，我们不必担心控制终端，登录时，将自动创建控制终端。有时不管标注输入输出是否重定向，我们都需要与终端交互。<font color = red/>保证程序能够与控制终端对话的方法是`open`文件`/dev/tty`。在内核中，次特殊文件是控制终端的同义语。当程序没有控制终端，则对此文件的打开会失败。</font>

i会话分配控制终端的两种方式：一、当会话首进程用`TIOCSCTTY`作为request参数（第三个参数为空指针）调用`ioct1`时，系统会为会话分配控制终端。二、当会话首进程打开第一个未与会话关联的终端设备时，只要在调用`open`时不指定`O_NOCTTY`，系统将次作为控制终端分配给次会话。

## 函数tcgetpgrp、tcsetpgrp和tcgetsid

需要有一种方法告诉内核哪一个进程组是前台进程组，这样，终端设备驱动程序就能知道将终端输入和终端产生的信号发送到何处：

```c
#include<unistd.h>
pid_t tcgetpgrp(int fd);
//成功返回前台进程组ID，否则返回-1；

int tcsetpgrp(int fd, pid_t pgrpid);
//成功返回0，否则返回-1

```

函数`tcgetpgrp`返回前台进程组ID，它与在fd上打开的终端相关联。

如果进程有一个控制终端，则该进程可以调用`tcsetpgrp`将前台进程组ID设置为``pgrpid`。`pgrpid`应当是在同一会话中的一个进程组ID。`fd`必须引用该会话控制终端。

给出控制TTY的文件描述符，通过`tcgetsid`函数，应用程序就能获得会话首进程的进程组ID：

```c
#include<termios.h>
pid_t tcgetsid(int fd);
//成功返回会话首进程的进程组ID，否则-1

```

## 作业控制

作业控制运行在一个终端上启动多个作业（进程组）。器控制哪个进程可以访问该终端以及那些作业在后台运行。作业控制要求以下三种支持：

1. 支持作业控制的shell。
2. 内核中的终端驱动程序必须支持作业控制。
3. 内核必须提供对某些作业控制信号的支持。

通常，在shell里面输入命令默认产生的是前台进程，在一般命令后加上一个`&`即将其转换为后台进程。

如：

```
vim mian.cpp
//前台进程

pr *.c | lpr &
make all &
//两个后台进程

```

当启动一个后台进程是，shell将会赋予它一个作业标识符，并打印一个或多个进程ID：

如：

```shell
$ ls -l > file.txt &
[1] 29107
$ cp file.txt file2.txt &
[2] 29110
[1]   Done                    ls --color=auto -l > file.txt
$ 
[2]+  Done                    cp file.txt file2.txt

```

`ls`是的编号作业是1，`cp`的作用编号是2。当作业完成且键入回车时，shell通知作业已经完成。

有三个特殊字符可使终端驱动程序产生信号，并将他们发送到前台进程组：

1. 中断字符（`Delete`或`Ctrl+c`）产生SIGINT。
2. 退出字符（`Ctrl+\`）产生SIGQUIT。
3. 挂起字符（一般采用`Ctrl+Z`）产生SIGTSTP。

只有前台作业接收终端输入。如果后台作业试图读终端，并不是一个错误，但终端会检测到这种情况，并且向后台作业发送一个特定信号SIGTTIN。该信号会停止次后台作业，而shell则向有关用户发出这种情况的通知，然后用户使用shell指令将次作业转换为前台作业，于是就可以正常读终端了。

如：

```
$ cat > temp.foo &
[1] 30285
$ 

[1]+  Stopped                 cat > temp.foo
$ fg %1 //将作业转换为前台作业
cat > temp.foo

hello word
^D //键入文件结束符
$ cat temp.foo

hello word


```

`cat > temp.foo`命令是读取终端输入，输出到`temp.foo`文件里。当其想要读取时，终端驱动知道其为后台作业，发送信号SIGTTIN使作业停止。当将其转换为前台进程后，终端驱动发送继续信号SIGCONT给进程组。

对于后台工作输出到终端，这是一个我们可以允许或禁止的选项。可以使用`stty`改变这一选项：

```
$ cat temp.foo &
[1] 370
$ 
hello word

[1]+  Done                    cat temp.foo
$ stty tostop
$ cat temp.foo &
[1] 385
$ 

[1]+  Stopped                 cat temp.foo
$ fg %1
cat temp.foo

hello word

```

`stty tostop`禁止后台程序输出到控制终端，此时驱动程序发现该写操作来自于后台进程，于是向该作业发送SIGTTOU信号，`cat`信号阻塞。当使用`fg %1`将进程转为前台时，作业继续执行。

下图展示了作业控制的功能：

![作业控制](https://s2.ax1x.com/2019/10/01/uUgB9K.png)

## shell执行程序

执行下面的指令：

```shell
$ ps -o pid,ppid,pgid,sid,tpgid,comm
  PID  PPID  PGID   SID TPGID COMMAND
 4150 32758  4150 32758  4150 ps
32758 32752 32758 32758  4150 bash

```

可以看出，`ps`进程是`bash`的子进程，但shell将前台作业（`ps`)放入自己的进程组。`ps`是进程组组长，也是该进程组唯一进程，此进程具有控制终端，因此是前台进程组。

再执行下面的命令：

```
$  ps -o pid,ppid,pgid,sid,tpgid,comm &
[1] 4241
$   
 PID  PPID  PGID   SID TPGID COMMAND
 4233  4227  4233  4233  4233 bash
 4241  4233  4241  4233  4233 ps

[1]+  Done                    ps -o pid,ppid,pgid,sid,tpgid,comm

```

可以看到，此时前端进程组是`bash`。

再来看下面的指令：

```
$ ps -o pid,ppid,pgid,sid,tpgid,comm | cat
  PID  PPID  PGID   SID TPGID COMMAND
 4233  4227  4233  4233  4296 bash
 4296  4233  4296  4233  4296 ps
 4297  4233  4296  4233  4296 cat

```

`ps`和`cat`都在一个新的进程组组中，这是一个前台进程。注意：对于管道来说，上一条指令的输出是下一条指令的输入，因此，管道的最后一个进程（最后一个命令生成的进程）是shell的子进程，而执行管道中其他目录的进程则是该最后进程的子进程。可以理解为最后的一条指令生成的进程再fork一个新进程，执行之前的指令，再讲执行结果作为执行本身进程的输入。

## 孤儿进程组

定义：该组中每个成员的父进程要么是该组的一个成员，要么不是该组所属会话的成员。另一种描述为：一个进程不是孤儿进程的条件是：该组中存在一个进程，其父进程在属于同一会话的另一个组中。如果进程不是孤儿进程组，那么在属于同一会话的另一个组中的父进程就有机会重新启动该组中停止的进程。

POSIX.1要求向孤儿进程组中处于停止状态的每一个进程发送挂断信号（SIGHUP），接着又向其发送继续信号。

例程：

```c
#include"apue.h"
#include<error.h>

//接受到SIGHUP的处理函数，如果没有该函数，默认是终端进程
static void sig_hup(int signo)
{
    printf("SIGHUP received, pid = %ld\n", (long)getpid());
}

static void pr_ids(char *name)
{
    printf("%s: pid =%ld, ppid=%ld, tgprg = %ld\n", name, (long)getpid(), (long)getppid(),
    (long)tcgetpgrp(STDIN_FILENO)); //通过标准输入的fd获取会话首进程
    fflush(stdout);
}

int main(void)
{
    char c;
    pid_t pid;

    pr_ids("parent");

    if((pid = fork())<0)
    {
        err_sys("fork error");
    }
    else{
        if(pid>0)
        {
            sleep(5); // 暂停5秒，使子进程暂停
        }
        else{
            pr_ids("child");
            signal(SIGHUP, sig_hup); //绑定信号与处理函数
            kill(getpid(), SIGTSTP); //自己给自己发送信号，使自己暂停
            pr_ids("child"); //当接受到内核的SIGHUP和SIGCONT后继续执行。
            if(read(STDIN_FILENO, &c, 1)!=1)
            {
                printf("read error %d on controlling TTY\n",errno);
            }
        }

    }
    exit(0);
}

```

执行：

```
$ ./opg.o 
parent: pid =7646, ppid=6151, tgprg = 7646
child: pid =7647, ppid=7646, tgprg = 7646
SIGHUP received, pid = 7647
child: pid =7647, ppid=1, tgprg = 6151
read error 5 on controlling TTY

```

这里，当父进程终止时，子进程就会变为后台进程组，因为父进程是由shell作为前台作业执行的。当子进程继续执行时，企图从终端读取输入，但此时已经变成在后台进程组，于是内核向其发送SIGTTIN，但此时其为孤儿进程，如果进程是由该信号停止它，则此进程再也不会继续。

## FreeBSD实现

下图展示了进程，进程组，会话和控制终端是如何实现的：

![实现](https://s2.ax1x.com/2019/10/01/uUg6nH.png)

从`session`结构开始说明。每个会话都会分配一个`session`结构。

1. `s_count`是当前会话中的进程组数。当到达0时即可以释放该结构。
2. `s_leader`是指向会话首进程`proc`结构的指针。
3. `s_ttyvp`是指向控制终端`vnode`结构的指针。
4. `s_ttyp`是指向控制终端`tty`结构的指针。
5. `s_sid`是会话ID。

在调用`setsid`时，在内核中分配一个新的`session`结构。`s_count`设置为1，`s_leader`设置为调用进程`proc`结构的指针，`s_sid`设置为进程ID，由于新会话没有控制终端，所以`s_ttyvp`和`s_ttyp`设置为空指针。

接着说`TTY`结构。每个终端设备和每个伪终端设备均会在内核分配这样的一种结构。

1. `t_session`指向将此终端作为控制终端的`session`结果。终端在失去载波信号时使用此指针将挂起信号发送给会话首进程。
2. `t_pgrp`指向前台进程组的`pgrp`结构。终端驱动用次字段将信号发送到前台进程组。
3. `t_termios`包含所以这些特殊字符和与终端有关信息的结构。
4. `t_winsize`是包含终端窗口大小的`winsize`型结构。

为了找到特定前台进程，内核从会话开始，使用`s_ttyp`得到控制终端的`tty`结构，再用`t_pgrp`得到前台进程组的`pgrp`结构。

`pgrp`包含特定进程组信息。

1. `pg_id`是进程组ID。
2. `pg_session`指向此进程所属会话的`session`结构。
3. `pg_members`指向次进程组的`proc`结构表的指针`proc`代表进程组成员，`proc`结构中的`p_pglist`是一个双向链表，指向该组中的下一个和上一个进程。直到遇到最后一个进程，它的`proc`中`p_pglist`为空。

`proc`包含一个进程的信息

1. `p_pid`进程ID。
2. `p_pptr`指向父进程`proc`的指针。
3. `p_pgrp`指向本进程所属的进程组`pgrp`结构的指针。
4. `p_pglist`是一个结构。包含两个指针，指向进程组中上一个和下一个进程

最后还有一个`vnode`结构。在打开控制终端设备时分配此结构。进程对`/dev/tty`的所以访问都是通过`vnode`结构。

# 第十章 信号

## 信号概念

信号是软件中断。信号提供了一种处理异步事件的方法。每个信号都有名字，名字都以`SIG`开头。

很多条件可以产生信号：

1. 当用户按下某些终端键时，引发终端产生的信号。
2. 硬件异常产生信号：除数为0、无效的内存引用等。这些条件通常由硬件检测到，并通知内核。内核为进程产生适当的信号。
3. 进程调用`kill`函数可以将任意信号发送到另一个进程或者进程组。对此存在限制：发送信号的进程所有者应该与接受信号的进程所有者一致，或者发送信号的进程所有者为超级用户。
4. 用户可以使用`kill`指令将信号发送到其他进程。该指令是`kill`函数的接口。常用此命令终止一个失控的后台进程。
5. 当检测到某些软件条件已经发送，并应将其通知有关进程时也产生信号。这里指的是软件条件，如进程设置的定时闹钟已超时。

信号是异步事件的经典实例。产生信号的事件对进程而言是随机出现的。进程不能简单的测试一个变量来判断是否发送了一个信号，而是告诉内核“在此信号发生时，请执行下列操作”。

当某个信号发生时，可以告诉内核按下列3中方式之一进行处理：

1. 忽略次信号。大多数内核按照这种方式进行处理。但两种信号绝对不能忽略。是`SIGKILL`和`SIGSTOP`。它们向内核和超级用户提供了使进程终止或者停止的可靠方法。另外，如果忽略某些硬件异常的信号，则进程的行为是未定义的。
2. 捕捉信号。为了做到这一点，要通知内核在某种信号发生时，调用一个用户函数。在用户函数中，可执行用户希望对这种事件的处理。
3. 执行默认动作。大多数信号的系统默认动作是终止进程。

在系统默认动作中，“终止+`core`”表示在进程当前工作目录的`core`文件中复制该进程的内存映像。大多数UNIX系统调试程序都使用`core`文件检查进程终止时的状态。

下面列出各个信号相关信息：

| 信号         | 说明                            | 默认动作          | 详细说明                                                     |
| ------------ | ------------------------------- | ----------------- | ------------------------------------------------------------ |
| `SIGABRT`    | 异常终止（`abort`）             | 终止+`core`       | 调用`abort`时，产生次信号。进程异常终止。                    |
| `SIGALRM`    | 定时器超时（`alarm`）           | 终止              | 当用`alarm`函数设置的定时器超时，或由`setitimer`函数设置的间隔时间已经超时时，产生次信号。 |
| `SIGBUS`     | 硬件故障                        | 终止+`core`       | 指示一个实现定义的硬件故障。当出现某种类型的内存故障时，实现常常产生此信号。 |
| `SIGCANCEL`  | 线程库内使用                    | 忽略              | `Solaris`线程库内使用。                                      |
| `SIGCHLD`    | 子进程改变状态                  | 忽略              | 在一个进程终止或者暂停时，该信号被发送到其父进程。如果父进程希望被告知其子进程这种状态，则应该捕捉信号，在捕捉函数中调用一种`wait`函数以获得子进程ID和状态。 |
| `SIGCONT`    | 使暂停程序继续                  | 忽略              | 如果接到此信号的进程处于停止状态，在系统默认动作是进程继续执行，否则忽略此信号。 |
| `SIGEMT`     | 硬件故障                        | 终止+`core`       | 指示一个实现定义的硬件异常。                                 |
| `SIGFPE`     | 算数异常                        | 终止+`core`       | 算数运算异常，如除0、浮点数溢出。                            |
| `SIGFREEZE`  | 检查点冻结                      | 忽略              | 仅由`Solaries`定义。                                         |
| `SIGHUP`     | 连接断开                        | 终止              | 如果终端接口检测到一个连接断开，将此信号送给与终端进程相关的控制进程。接到此信号的会话首进程可能在后台。如果会话首进程已经终止，也产生此信号，则将信号发送给前台进程组。通常用此信号通知守护进程再次读取他们的配置文件。 |
| `SIGILL`     | 非法硬件指令                    | 终止+`core`       | 进程已执行一条非法硬件指令。                                 |
| `SIGINFO`    | 键盘状态指令                    | 忽略              | 一种`BSD`硬件指令。当用户按状态键（`Ctrl+T`）时发送信号到前段进程组。 |
| `SIGINT`     | 终端中断符                      | 终止              | 当用户按终端键（`Delete`或`Ctrl+C`）时，终端驱动程序发送信号取前台进程组中每一个进程。当一个进程失控时常使用该方式结束进程。 |
| `SIGIO`      | 异步I/O                         | 终止/忽略         | 此信号指示一个一个异步I/O                                    |
| `SIGLIOT`    | 硬件故障                        | 终止+`core`       | 指示一个实现定义的硬件故障                                   |
| `SIGJVM1`    | Java虚拟机内部使用              | 忽略              | `Solaris`为Jave虚拟机预留的信号。                            |
| `SIGJVM2`    | Java虚拟机内部使用              | 忽略              | `Solaris`为Jave虚拟机预留的信号。                            |
| `SIGKILL`    | 终止                            | 终止              | 杀死进程                                                     |
| `SIGLOST`    | 资源丢失                        | 终止              | 只在`Solaris`中存在。                                        |
| `SIGLWP`     | 线程库内使用                    | 终止/忽略         | `Solaris`内线程库内使用。                                    |
| `SIGPIPE`    | 写至无读进程的管道              | 终止              | 在管道的读进程已经终止时写管道，或类型为`SOCK_STREAM`的套接字已不再连接时，写该套接字会产生此信号。 |
| `SIGPOLL`    | 可轮询时间（poll）              | 终止              | 当一个可轮询事件发生一个特定事件时产生此信号。               |
| `SIGPROF`    | 梗概时间超时（`setitimer`）     | 终止              | 当`setitimer`函数设置的梗概统计间隔定时器已经产生超时信号时产生。 |
| `SIGPWR`     | 电源失效/重启动                 | 终止/忽略         | 接到蓄电池电压过低信息的进程将信号`SIGPWR`发送给`init`进程，而后`init`进程处理停机操作。 |
| `SIGQUIT`    | 终端退出符                      | 终止+`core`       | 当用户在终端按下退出键（`Ctrl+\`），中断驱动程序产生此信号，并发送给前台进程组的所有进程。 |
| `SIGSEGV`    | 无效内存引用                    | 终止+`core`       | 进程进行了一次无效的进程引用，通常说明程序有错。             |
| `SIGSTKFLT`  | 协处理器栈故障                  | 终止              | 并非由内核产生，只在早期`Linux`中存在。                      |
| `SIGSTOP`    | 停止                            | 停止进程          | 作业控制信号，停止进程。不能被捕捉或忽略。                   |
| `SIGSYS`     | 无效系统调用                    | 终止+`core`       | 指示一个无效的系统调用，指令指示系统调用类型参数是无效的。常发生在不同系统间。 |
| `SIGTREM`    | 终止                            | 终止              | 这是由`kill`命令发送的系统默认终止信号。该信号是可以捕获的，相对与`SIGKILL`，我们可以在终止前进行必要的处理。 |
| `SIGTHAW`    | 检查点解冻                      | 忽略              | 由`Solaris`定义。                                            |
| `SIGTHR`     | 线程库内部使用                  | 忽略              | `FreeBSD`预留线程库信号。                                    |
| `SIGTRAP`    | 硬件故障                        | 终止+`core`       | 指示一个实现定义的硬件故障                                   |
| `SIGTSTP`    | 终端停止符                      | 停止进程          | 交互停止信号，当用户在终端上按挂起键（`Ctrl+Z`）时，终端驱动程序产生此信号，该信号发送至前段进程组的所以进程。 |
| `SIGTTIN`    | 后台读控制`tty`                 | 停止进程          | 后台进程组进程试图读其控制终端时，终端驱动产生此信号。下列两个情况不产生：1. 读进程忽略或阻塞此信号。2.进程所属为孤儿进程组，读进程返回错误，`errno`设置为`EIO`。 |
| `SIGTTOU`    | 后台写向控制`tty`               | 停止进程          | 与删一条类似，不过是后端向所属控制终端写。                   |
| `SIGURG`     | 紧急情况（套接字）              | 忽略              | 通知进程已经发生一个紧急情况。在网络连接上接到带外的数据时，可选择的产生此信号。 |
| `SIGUSER1`   | 用户定义信号                    | 终止              | 用户定义信号，可用于程序                                     |
| `SIGUSER2`   | 用户定义信号                    | 终止              | 用户定义信号，可用于程序                                     |
| `SIGVTALRM`  | 虚拟时间闹钟（`setitimer`）     | 终止              | 当一个由`setitimer`函数设置的虚拟时间间隔时间已经超时时产生此信号。 |
| `SIGWAITING` | 线程库内使用                    | 忽略              | 由`Solaris`线程库内部使用。                                  |
| `SIGWINCH`   | 终端窗口改变                    | 忽略              | 如果进程用`ioct1`的设置窗口大小命令更改了窗口大小，则内核将此信号发送至前台进程组。内核维持与每个终端与伪终端相关联的窗口大小。 |
| `SIGXCPU`    | 超过CPU限制（`setrlimit`）      | 终止或终止+`core` | 进程超过其软CPU限制，会产生该信号。                          |
| `SIGXFSZ`    | 超过文件长度限制（`setrlimit`） | 终止或终止+`core` | 如果进程超过其软文件长度限制，则产生该信号。                 |
| `SIGXRES`    | 超过资源限制                    | 忽略              | 仅由`Solaris`定义。                                          |





## 函数signal

```c
#include<signal.h>

void (*signal(int signo, void(*func)(int)))(int);
//成功返回以前的信号处理配置，若出错，返回SIG_ERR

```

`signo`是信号名，`fun`是常量SIG_IGN、常量SIG_DFL或当接到此信号后要调用的函数的地址。如果指定SIG_IGN则忽略此信号，如果指定SIG_DEF则表示接收此信号后的动作是系统默认动作。当指定函数，则接到信号执行相应函数。此函数称为信号处理程序或者信号捕捉函数。

函数返回为一个函数指针，即返回函数指针的函数。返回的函数也有一个int参数，该参数为信号，返回的函数无返回值。因此信号处理程序都是只有一个int参数且无返回值的函数。

例程：

```c
#include"apue.h"
static void sig_usr(int);

int main(void)
{
    if(signal(SIGUSR1, sig_usr) == SIG_ERR)
    {
        err_sys("signal error");
    }

    if(signal(SIGUSR2, sig_usr) == SIG_ERR)
    {
        err_sys("signal error");
    }
    while(1){
        pause();
    }
}

static void sig_usr(int signo)
{
    if(signo == SIGUSR1)
    {
        printf("received SIGUSR1\n");
    }

    else if(signo == SIGUSR2)
    {
        printf("received SIGUSR2\n");
    }

    else {
        err_dump("received signal %d\n", signo);
    }

    
}

```

程序执行：

```
$ ./signal.o & //后台执行
[1] 8011
$ kill -USR1 8011 //发送SIGUSR1
received SIGUSR1
$ kill -USR2 8011 //发送SIGUSR2
received SIGUSR2
$ kill 8011 //发送SIGTERM
$ kill -USR2 8011
bash: kill: (8011) - No such process
[1]+  Terminated              ./signal.o

```

### 程序启动

`exec`函数将原先设置为要捕捉的信号都恢复为默认动作，其它信号则不变，这是因为当执行`exec`后，原来捕捉函数的地址可能对于新程序来说是无意义的。

`signal`存在的一个限制：不改变信号的处理方式就不知道当前其处理方式。

### 进程创建

当一个进程调用`fork`时，其子进程继承父进程的信号处理方式，由于子进程复制了父进程内存镜像，所以捕捉函数的地址在子进程中也是有意义的。

## 不可靠信号

在早期的UNIX版本中，信号是不可靠的。不可靠是指，信号可能丢失：一个信号发生了，但进程却可能一直不知道这一点。

早期的另一个问题是，再进程每次接到信号对其进行处理时，随即将信号的动作重置为默认值。因此早期处理中断的代码中可能是这样：

```c
int sig_int();
.
.
.
signal(SIGINT, sig_int);

sig_int()
{
    signal(SIGINT, sig_int);
    .
    .
    .
}

```

这存在两个问题。1：在第一个信号发生进行处理，和再重新设置捕捉函数之间如果再次出现该信号，则处理的方式是按默认值，可能与我们期望不一致。2：对于表示默认忽略的信号，如果我们希望设置为忽略是无法实现的，只能在捕捉函数中进行忽略。

## 中断的系统调用

早期的UNIX系统的一个特征是：如果执行一个低速系统调用而阻塞期间捕捉到一个信号，则系统调用就中断不再继续执行。该系统调用返回出错，其`errno`设置为`EINTR`。注意，这里是内核中的系统调用中断

系统调用分为两类：低速系统调用和其他系统调用。低速系统调用是可能会使进程永远阻塞的一类系统调用：

1. 如果某些类型文件（如读管道、终端设备和网路设备）的数据不存在，则读操作可能会使调用者永远阻塞。
2. 如果这些数据不能被相同类型的文件立即接受，则写操作可能会使调用者永远阻塞。
3. 在某些条件发生之前打开某些文件，可能会发生阻塞（例如打开一个终端设备，需要先等待与之连接的调制解调器应答）。
4. `pause`函数（使进程休眠直至捕捉到一个函数）和`wait`函数。
5. 某些`ioct1`操作。
6. 某些进程间通信函数。

我们必须显示的处理出错返回。如：存在一个读操作，它被中断，我们希望从新启动它，则可能是如下代码：

```c
again:
if((n=read(fd, buf, BUFFSIZE))<0)
{
    if(errno == EINTR)
    {
        goto again;
    }
}

```

4.2BSD引进了某些系统调用的自动启动。自动启动的系统调用包括：`ioct1`、`read`、`readv`、`write`、`writev`、`wait`和`waitpid`。但这也是有问题的，某些程序并不希望这些函数被中断后重新启动。需要注意的是，不同系统实现是不一样的，别的系统并不一定有自动重启。在我的Ubuntu18.04上`read`是可以自动重启的。

## 可重入信号

进程捕捉信号并对其进行处理时，进程正在执行的指令序列就被信号处理程序临时中断，它首先执行该信号处理程序中的指令。如果从信号处理程序返回（例如没有调用`exit`或`longjmp`），则继续执行在捕捉到信号时进程正在执行的正常指令序列。这时会有两个问题，1：如果被中断的进程正在执行`malloc`在堆中分配空间，而调用的信号捕捉函数内部也调用了`malloc`分配空间，则此时可能对进程造成破坏，因为`malloc`通常为它所分配的存储器维护一个链表，而插入执行信号处理程序时，进程可能正在更改此表。2：若中断的进程正在执行`getpwnam`这种将其结果存放在静态存储单元中的函数，在其插入的信号捕捉函数中又调用此函数，则正常调用信息可能被信号处理函数的结果覆盖。

下面列出来的函数是不会发生写情况，这些函数是可重入的并被称为是异步信号安全的。除了可重入外，在处理信号期间，它会阻塞任何引起不一致的信号发送。

![可重入函数](https://s2.ax1x.com/2019/10/06/ucmGx1.png)

不在上图中的，一般都是不可重入的，他们一般是（a）：已知他们使用静态数据结构。（b）：他们调用`malloc`或`free`。（c）：他们是标准I/O。

由于每个线程只有一个`errno`变量，所以信号处理函数可能会更改其原来的值。因此，作为一个通用规则，当在信号处理程序中调用上图中的函数，应该先保存`errno`，在调用后恢复`errno`。

非可重入例程：

```c
#include"apue.h"
#include<pwd.h>

static void my_alarm(int signo)
{
    passwd *rootptr;
    printf("in signal handler\n");
    if((rootptr=getpwnam("root")) == NULL)
    {
        err_sys("getpwnam(root) error");
    }
}
int main(void)
{
    passwd *ptr;
    signal(SIGALRM, my_alarm);
    alarm(1);
    while(1)
    {
        if((ptr=getpwnam("chst")) == NULL)
        {
            err_sys("getpwnam error");
        }
        if(strcmp(ptr->pw_name, "chst") != 0)
        {
            printf("return value corrupted!, pw_name=%s\n",ptr->pw_name);
        }
    }
}

```



## SIGCLD语义

`SIGCLD`与`SIGCHLD`两个信号很容易混淆。`SIGCLD`是`System V`的一个信号。其与`SIGCHLD`不同。

`SIGCLD`早期处理方式是：

(1）如果进程明确地将信号的配置设置为`SIG_IGN`，则调用进程将不产生僵死进程。这里与默认动作（`SIG_DFL`）忽略不同。子进程在终止时，将其状态丢弃。如果调用进程随后调用一个`wait`函数，则会等待到所以子进程都终止，然后返回-1。并将其`errno`设置为`ECHILD`。

（2）如果将`SIGCLD`设置为捕捉，则内核检查是否有子进程准备好被等待，如果是这样则调用SIGCLD处理程序。这里是应该是一个漏洞，在后面的例子可以看出来，应该是出现该信号时才调用此信号处理函数。

例程：

```c
#include"apue.h"
#include<sys/wait.h>

static void sig_cld(int);

int main()
{
    pid_t pid;
    if(signal(SIGCLD, sig_cld) == SIG_ERR)
    {
        perror("signal error");
    }

    if((pid = fork())<0)
    {
        perror("fork error");
    }
    else{
        if(pid==0)
        {
            sleep(2);
            _exit(0);
        }
    }
    pause();
    exit(0);
}

static void sig_cld(int signo)
{
    pid_t pid;
    int status;
    printf("SIGCLD received\n");

    if(signal(SIGCLD, sig_cld) == SIG_ERR)
    {
        perror("signal error");
    }

    if((pid = wait(&status)) < 0)
    {
        perror("wait error");
    }

    printf("pid = %d\n", pid);

}

```

该程序存在的问题是，在旧的UNIX系统上，信号处理程序使用一次就会被重置为默认处理方式，因此在信号处理函数中要再次绑定，但是绑定的位置放到了`wait`函数之前，此时内核会检查是否存在一个需要等待的子进程，而这时，wait还没被调用，子进程状态并未被释放，条件满足，于是会立即再次调用信号处理函数，这样就会不断迭代调用，知道达到资源限制。解决方法是，将`wait`函数放到重新绑定信号处理函数之前。这个问题在较新的系统上已经不存在了，一方面，现在的系统不会调用一次信号处理函数就将其恢复为默认处理方式，因此不用重新绑定，再次，现在都是检测函数是否出现，而不是检测是否有需要等待的进程。因此在我的电脑上执行结果为：

```
$ ./sigcld.o
SIGCLD received
pid = 13604

```

## 可靠信号术语和语义

当造成信号的事件发生时，未进程<font color =red/>产生（generation）</font>一个信号。当一个信号产生时，内核通常在进程表中以某种形式设置一个标志。

当对信号采取了这种动作时，我们说向进程<font color = red/>递送（delivery）</font>了一个信号。在信号产生和递送之间的时间间隔内，称信号是<font color = red/>未决的（pending）</font>。

进程可以选用“阻塞信号递送”。如果一个进程产生了一个阻塞的信号，而且对该信号的动作是系统默认动作或者捕捉该信号，则为该进程将此信号保持为未决状态，直到该进程对此信号解除阻塞或者设置此信号的动作为忽略。

如果对一个信号解除阻塞前，该信号发了多次，如果递送该信号多次，则称这些信号进行了排队。除非支持POSIX.1实时扩展，否则大部分UNIX并不多信号排队而仅递送一次。

如果有多个信号要递送给一个进程，POXIS.1并未规定这些信号的递送顺序。但建议是在其他信号之前递送与进程当前状态有关的信号。

每个进程都有一个信号屏蔽字，它规定了当前要阻塞递送到该进程的信号集。对于每一种可能的信号，该屏蔽字都有一位与之对应，如果该位被设置，则对应的信号应该是阻塞的。

信号编号可能会超过一个整型所包含的二进制位数，因此POSIX.1定义了一个新数据类型`sigset_t`，它容纳一个信号集。

## 函数kill和raise

`kill`函数将信号发送到指定进程或进程组。`raise`函数则允许进程向自生发送信号：

```c
#include<signal.h>

int kill(pid_t pid, int signo);

int raise(int signo); 

//成功返回0，否则返回-1

```

调用`raise（signo）`等同于调用`kill（gitpid（），signo）`。

`kill`的参数有以下四种情况：

1. pid > 0： 将该信号发送给进程ID为pid的进程。
2. pid == 0：将信号发送给发送进程同属一个进程组的所以进程。而且发送进程有权限向其发送信号的所以进程。
3. pid < 0：将信号发送给进程组ID为pid绝对值的进程组。而且发送进程有权限向其发送信号的所以进程。
4. pid == -1：将该信号发送到发送进程有权限向他们发送信号的所以进程。

进程将信号发送给其它进程需要权限。超级用户可以将信号发送任一进程。对于非超级用户，其基本规则为是：发送者的实际用户ID和有效用户ID必须等于接收者的实际用户ID或有效用户ID。

POSIX.1将信号编号为0定义为空信号，signo如果是0，则`kill`仍执行正常的错误检查，但不发生信号。常用来检查特定进程是否依然存在。如果一个不存在的进程发送信号，则`kill`返回-1，`errno`被设置为`ESRCH`。

测试进程存在不是原子操作。在`kill`向调用者返回结果时，原来存在的进程可能已经终止了。

如果`kill`为调用者产生信号，而且此信号是不被阻塞的，那么在`kill`返回之前，`signo`或者某个其他未决的、非阻塞信号被传送至该进程。即如果进程向自身发送`SIGKILL`信号，则在返回之前进程已经终止了。

## 函数alarm和pause

使用`alarm`函数可以设置一个定时器（闹钟时间），在某个时刻该定时器会超时。当定时器超时时，产生`SIGALRM`信号，如果忽略或不捕捉该信号，进程终止。

```c
#include<unistd.h>

unsigned int alarm(unsigned int seconds);

//返回值,0或以前设置的闹钟时间的余留秒数。

```

每个进程只能有一个闹钟时间。如果调用`alarm`时，之前已经为该进程注册的闹钟时间还没有超时，则闹钟时间会被新值替代，而旧的剩余时间会被返回。

如果有以前注册的尚未超时的闹钟时间，而且本次调用的`second`值为0，则取消之前的闹钟时间，其剩余时间作为返回值。

`pause`函数使调用进程挂起直至<font color = red/>捕捉</font>到一个信号:

```c
#include<unistd.h>

int pause(void);
//返回值：-1， errno设置为EINTR

```

注意：<font color = red/>只有处理了一个信号处理程序并从其返回时，`pause`才返回。</font>因此，如果被捕捉的函数执行耗时很长，将一值阻塞。在这种情况下，`pause`返回-1，`errno`设置为`EINTR`。

使用`alarm`和`pause`实现`sleep`：

```c
#include"apue.h"
#include<unistd.h>

static void sig_alarm(int signo)
{
    //什么都不用做，只是为了从pause唤醒进程
}

unsigned int sleep1(unsigned int seconds)
{
    if(signal(SIGALRM, sig_alarm) == SIG_ERR)
    {
        return seconds;
    }
    alarm(seconds);
    pause();
    return alarm(0);
}

```

程序存在三个问题：

1. 如果在调用`sleep1`之前已经设置了闹钟，则会被`sleep1`中重设删除。处理方式：检查第一次调用`alarm`的返回值，如果小于`seconds`，则只等到之前设置的闹钟超时，如果返回值大于`seconds`，则应该在`sleep1`返回之前重置闹钟，使原来的闹钟不会被清除。
2. 该程序修改了`SIGALRM`的配置，如果编写了一个函数供其他函数调用，则在函数被调用时应该先保留原来的配置（`sleep1`中`signal`的返回），在该函数返回前恢复配置。
3. 调用`alarm`与`pause`之间存在竞争条件。可能`alarm`在调用`pause`之前超时，此时，调用者可能被永久挂起。

前两个问题解决比较简单，对于第三个问题的解决需要后面学习。

使用`setjmp`和`longjmp`解决第三个问题：

```c
#include<setjmp.h>
#include<unistd.h>
#include<signal.h>

static jmp_buf env_alrm;

static void sig_alarm(int signo)
{
    longjmp(env_alrm, 1);
}

unsigned int sleep1(unsigned int seconds)
{
    if(signal(SIGALRM, sig_alarm) == SIG_ERR)
    {
        return seconds;
    }
    if(setjmp(env_alrm) == 0)
    {
        alarm(seconds);
        pause();
    }
    return alarm(0);
}

```

该函数基本解决第三个问题，但会存在新的问题：如果`SIGALRM`中断了某个其他信号的处理程序，则调用`longjmp`将会提早终止该信号处理程序。如下程序：

```c
#include<setjmp.h>
#include<unistd.h>
#include<signal.h>
#include"apue.h"

static jmp_buf env_alrm;

static void sig_alarm(int signo)
{
    longjmp(env_alrm, 1);
}

unsigned int sleep2(unsigned int seconds)
{
    if(signal(SIGALRM, sig_alarm) == SIG_ERR)
    {
        return seconds;
    }
    if(setjmp(env_alrm) == 0)
    {
        alarm(seconds);
        pause();
    }
    return alarm(0);
}

static void sig_int(int signo)
{
    int i,j;
    volatile int k;
    printf("\nsig_int starting\n");
    for(i = 0;i<300000; i++)
    {
        for(j=0;j<4000;j++)
        {
            k += i*j;
        }
    }

    printf("sig_int finish\n");
}

int main()
{
    unsigned int unslept;
    if(signal(SIGINT, sig_int) == SIG_ERR)
    {
        err_sys("signal(SIGINT) error");
    }
    unslept = sleep2(5);
    printf("sleep2 return :%u\n", unslept);
    exit(0);
}

```

执行结果：

```shell
$ ./sleep.o
^C
sig_int starting
sig_int finish
sleep2 return :1
$ ./sleep.o
^C
sig_int starting
sleep2 return :0

```

两次执行差异，第一次执行后，直接按`Ctrl+C`。第二次，执行程序后过一会再按`Ctrl+C`。解释：对于第一次来说，直接按下`Ctrl+C`会执行`sig_int`函数，而且在`alarm`到达之前就执行完了，于是`pause`返回，进程继续执行，由于`alarm`还未超时，此时调用`alarm(0)`会返回上一次（5）设置的时间的剩余时间，这里我的运行结果是还剩下1秒。对于第二次来说，过一阵按`Ctrl+C`时，在执行`sig_int`时，`alarm`设置的5秒超时，于是暂停执行`sig_int`函数，执行`sig_alarm`函数，这时，由于调用了`longjmp`，因此`sig_int`将不会再执行了，就造成了提前终止了`SIGINT`信号的处理程序。在执行完`sig_int`函数后，调用`alarm(0)`，由于上一个设置的闹钟已经执行完成，因此返回是0。

使用`alarm`和`setjmp`对可能阻塞的操作，设置时间上限：

```c
#include"apue.h"
#include<setjmp.h>

static void sig_alarm(int);
static jmp_buf env_alarm;

int main()
{
    int n;
    char line[MAXLINE];
    if(signal(SIGALRM, sig_alarm) == SIG_ERR)
    {
        err_sys("signal(SIGALRM) error");
    }

    if(setjmp(env_alarm)!=0)
    {
        err_quit("read timeout");
    }

    alarm(5);

    if((n=read(STDIN_FILENO, line, MAXLINE))<0)
    {
        err_sys("read error");
    }
    alarm(0);

    write(STDOUT_FILENO, line, n);
    exit(0);
}

static void sig_alarm(int signo)
{
    longjmp(env_alarm, 1);
}

```

该程序也存在问题：如果`SIGALRM`中断了某个其他信号的处理程序，则调用`longjmp`将会提早终止该信号处理程序。

## 信号集

我们需要一个能够表示多个信号：信号集的数据类型。POSIX.1定义数据类型`sigset_t`以包含一个信号集，并定义了下面5个处理信号集的函数：

```c
#include<signal.h>

int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signo);
int sigdelset(sigset_t *set, int signo);
//四个函数，成功返回0， 否则返回-1。

int sigismember(const sigset_t *set, int signo);
//真返回1， 否则返回0

```

函数`sigemptyset`初始化由`set`指向的信号集，清除其中所以信号。函数`sigfillset`初始化由`set`指向的信号集，使其包含所以信号。所以程序在使用信号集之前都必须调用两个函数中的至少一个。

## 函数sigprocmask

进程的信号屏蔽字规定了当前阻塞而不传递给该进程的信号集。调用函数`sigprocmask`可以检测和修改，或同时进行检测和修改：

```c
#include<signal.h>
int sigprocmask(int how, const sigset_t *restrict set, sigset_t *restrict oset);
//成功返回0，出错返回-1

```

首先，若`oset`是非空指针，那么进程的当前信号屏蔽字通`oset`返回。

其次，若`set`是非空指针，则参数`how`决定如何修改当前信号屏蔽字。下表说明了可选参数和含义：

| `how`         | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| `SIG_BLOCK`   | 进程的信号屏蔽字是当前进程信号屏蔽字和`set`指向信号集的并集，`set`包含了希望阻塞附加信号。 |
| `SIG_UNBLOCK` | 进程的信号屏蔽字是当前进程信号屏蔽字和`set`指向信号集补集的交集，`set`包含了希望解除阻塞的信号。 |
| `SIG_SETMASK` | 进程新的信号屏蔽字是`set`指向的值。                          |

在调用`sigprocmask`后如果有任何未决的、不在阻塞的信号，则在`sigprocmask`返回之前，至少将其中之一递送给该进程。

## 函数sigpending

`sigpending`函数返回当前进程中阻塞的，未递送的信号（已经产生了）：

```c
#include<signal.h>

int sigpending(sigset_t *set);
//成功返回0，否则返回-1

```

例程：

```c
#include"apue.h"
static void sig_quit(int);

int main()
{
    sigset_t newmask, oldmask, pendmask;
    if(signal(SIGQUIT, sig_quit) == SIG_ERR)
    {
        err_sys("can't catch SIGQUIT");
    }
    sigemptyset(&newmask);
    sigaddset(&newmask, SIGQUIT);
    if(sigprocmask(SIG_BLOCK, &newmask, &oldmask)<0)
    {
        err_sys("SIG_BLOCK error");
    }
    sleep(5);
    if(sigpending(&pendmask)<0)
    {
        err_sys("sigpending error");
    }

    if(sigismember(&pendmask, SIGQUIT))
    {
        printf("\nSIGQUIT pending\n");
    }

    if(sigprocmask(SIG_SETMASK, &oldmask, NULL)<0)
    {
        err_sys("SIG_SETMARK error");
    }
    printf("SIGQUIT unblock\n");
    sleep(5);
    exit(0);
}

static void sig_quit(int signo)
{
    printf("catch SIGQUIT\n");
    if(signal(SIGQUIT, SIG_DFL)==SIG_ERR)
    {
        err_sys("signal error");
    }
}

```

执行：

```
$ ./sigpending.o 
^\^\^\^\^\^\
SIGQUIT pending
catch SIGQUIT
SIGQUIT unblock
^\Quit (core dumped)

```

## 函数sigaction

`sigaction`函数用来检查或修改与指定信号相关联的处理动作：

```c
#include<signal.h>

int sigaction(int signo, const struct sigaction *restrict act, struct sigaction *restrict oact);

```

`signo`是信号编号。若`act`指针为空，则要修改其动作，若`oact`不为空，则系统由`oact`返回该信号的上一个动作。其中结构体为：

```c
struct{
    void (*sa_handler)(int); //信号处理函数地址或者SIG_DEL/SIG_IGN
    sigset_t sa_mask; //添加的阻塞信号
    int sa_flags; //信号选择
    void (*sa_sigaction)(int,siginfo_t *,void *);//
}

```

​		`sa_mask`字段说明了一个信号集，在调用（进入）该信号捕捉函数之前，这一信号集要加入到进程的信号屏蔽字中。仅当从捕捉函数中返回时，再将进程的信号屏蔽字恢复为原值。这样就可以在执行捕捉函数时阻塞某些信号。在一个信号处理程序被调用时，操作系统建立的新信号屏蔽字包括正在被递送的信号。因此保证在处理一个信号时，如果该信号再次发生，那么会阻塞到对前一个信号的处理结束。若同一个信号多次发生，通常并不会将他们加入队列，所以如果在某种信号被阻塞是，若发生了多次，那么对信号解除阻塞后，其信号处理函数只会被调用一次。

​		`act`结果的`sa_flags`字段指定对信号进行处理的各个选项：

| 选项           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `SA_INTERRUP`  | 由此信号中断的系统调用不自动重启动。（sigaction默认处理方式） |
| `SA_NOCLDSTOP` | 若`signo`是`SIGCHLD`，当子进程停止是，不产生此信号。当子进程终止时，仍旧产生信号。 |
| `SA_NODEFER`   | 若`signo`是`SIGCHLD`时，子进程终止时，不创建僵死进程。如调用进程随后调用`wait`，则阻塞到所以子进程终止，返回-1.`errno`设置为`ECHLD`。 |
| `SA_ONSTACT`   | 当捕捉到该信号时，在执行其信号捕捉函数时，系统不自动阻塞此信号（除非`sa_mark`包含了此信号） |
| `SA_ONSTACK`   |                                                              |
| `SA_RESETHAND` |                                                              |
| `SA_RESTART`   | 由此信号中断的系统调用自动重启动。                           |
| `SA_SIGINFO`   | 对信号处理程序提供了一个附加信息：一个指向`siginfo`结构的指针以及一个指向进程上下文标识符的指针。 |

​		`sa_sigaction`字段是一个替代的信号处理程序，在`sigaction`的结构中使用了`SA_SIGINFO`标志时，使用该信号处理程序。

​		`siginfo`包含了信号产生的原因的有关信息：

```c
struct siginfo
{
    int si_signo; //信号编号
    int si_erron; //错误编号
    int si_code;//
    pid_t si_pid; //发送进程ID
    uid_t si_uid; //发送进程实际用户ID
    void *si_addr; //导致错误地址
    int si_status; //
    union sigval si_value; //
};

sigval联合包含下列字段
int sival_int;
void *sival_ptr;

```

​		应用程序在`si_value.sival_int`中传递一个整数或者在`si_value.sigval_ptr`中传递一个指针。

​		下图展示了各种信号的`si_code`:

![si_code](https://s2.ax1x.com/2019/10/11/uqzccD.png)

​		若信号是`SIGCHLD`，则设置`si_pid`、`si_status`和`si_uid`字段。若信号是`SIGBUS`、`SIGILL`、`SIGFPE`或`SIGSEGC`，则`si_addr`包含故障的根地址。

​		信号处理程序的`context`参数是无类型指针，它可以被强制转换成`ucontext_t`结构类型，该结构标识信号传递时进程的上下文。至少包含下面字段：

```
ucontext_t *uc_link;//
sigset_t un_sigmask;//
stack_t un_stack; //
mcontext_t un_mcontext;

// uc_satck字段描述了当前上下文使用的栈，至少包含下列成员
void *ss_sp;
size_t ss_size;
int ss_flags;

```

使用`sigaction`实现`signal`函数：

```c
#include"apue.h"
/* typedef	void	Sigfunc(int);*/
Sigfunc *signal(int signo, Sigfunc *func)
{
    struct sigaction act, oact;
    act.sa_handler = func;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    if(signo == SIGALRM)
    {
        #ifdef SA_INTERRUPT
        act.sa_flags |= SA_INTERRUPT
        #endif
    }
    else{
        act.sa_flags |= SA_RESTART;
    }

    if(sigaction(signo, &act, &oact)<0)
    {
        return SIG_ERR;
    }
    return oact.sa_handler;
}

```

## 函数sigsetjmp和siglongjmp

​		在执行信号处理程序时，对应信号会被自动加入到信号屏蔽字中，此时如果调用`longset`函数，对于该信号是否从信号屏蔽字中恢复是未指定的，而是定义了`sigsetjmp`和`siglongjmp`函数来指定这种操作：

```
#inclue<setjmp.h>

int sigsetjmp(sigjmp_buf env, int savemask);
//直接调用返回0，从siglongjmp调用返回非0
void siglongjmp(sigjmp_buf env, int val);

```

​		这两个函数和`setjmp`、`longjmp`的唯一区别是`sigsetjmp`增加了一个参数。如果`savemask`非0，则`sigsetjmp`在`env`中保存进程的当前信号屏蔽字。调用`siglongjmp`时，如果带非0`savemark`的`sigsetjmp`调用已经保存了`env`，则`siglongjmp`从其中恢复保存的信号屏蔽字。

实例：

```c
#include"apue.h"
#include<setjmp.h>
#include<time.h>

static void sig_usr1(int);
static void sig_alrm(int);
static sigjmp_buf jmpbuf;
static volatile sig_atomic_t canjmp;
void pr_mask(const char *str)
{
    sigset_t sigset;
    int errno_save;
    errno_save = errno;
    if(sigprocmask(0, NULL, &sigset)<0)
    {
        err_ret("sigprocmask error");
    }
    else{
        printf("%s", str);
        if(sigismember(&sigset, SIGINT))
        {
            printf(" SIGINT");
        }
        if(sigismember(&sigset, SIGQUIT))
        {
            printf(" SIGQUIT");
        }
        if(sigismember(&sigset, SIGUSR1))
        {
            printf(" SIGUSR1");
        }
        if(sigismember(&sigset, SIGALRM))
        {
            printf(" SIGALRM");
        }
        printf("\n");
    }
    errno = errno_save;
}

int main()
{
    if(signal(SIGUSR1, sig_usr1) == SIG_ERR)
    {
        err_sys("signal(SIGUSR1) error");
    }
    if(signal(SIGALRM, sig_alrm) == SIG_ERR)
    {
        err_sys("signal(SIGALRM) error");
    }
    pr_mask("starting main: ");

    if(sigsetjmp(jmpbuf, 1))
    {
        pr_mask("ending main: ");
        exit(0);
    }

    canjmp = 1;
    while(1){
        pause();
    }

}

static void sig_usr1(int signo)
{
    time_t starttime;
    if(canjmp == 0)
    {
        return;
    }

    pr_mask("starting sig_usr1: ");

    alarm(3);
    starttime = time(NULL);
    while(1)
    {
        if(time(NULL) > starttime +5)
        {
            break;
        }
    }

    pr_mask("finish sig_usr1: ");

    canjmp = 0;
    siglongjmp(jmpbuf, 1);
}

static void sig_alrm(int signo)
{
    pr_mask("in sig_alrm: ");
}

```

执行：

```
$ ./sigsetjmp.o &
[1] 6705
$ starting main: 

$ kill -USR1 6705
starting sig_usr1:  SIGUSR1
$ in sig_alrm:  SIGUSR1 SIGALRM
finish sig_usr1:  SIGUSR1
ending main: 

[1]+  Done                    ./sigsetjmp.o

```



## 函数sigsuspend

​		`sigsuspend`函数是一个原子操作，该函数的作用是，先恢复信号屏蔽字，然后使进程休眠：

```c
#include<signal.h>
int sigsuspend(const sigset_t *sigmask);
//返回-1，并将errno设置为EINTR

```

​		进程的信号屏蔽字设置为`sigmask`指向的值。<font color = red>在捕捉到一个信号或发生了一个会终止该进程的信号之前，该进程被挂起。如果捕捉到一个信号而且从该信号处理程序返回，则`sigsuspend`返回，并且该进程的信号屏蔽字设置为调用`sigsuspend`之前的值。</font>

例程：捕捉中断信号和退出信号，但只有当是退出信号时时才唤醒进程：

```c
#include"apue.h"

volatile sig_atomic_t quitflag = 0;

static void sig_int(int signo)
{
    if(signo == SIGINT)
    {
        printf("\ninterrupt\n");
    }
    else{
        quitflag = 1;
    }
}

int main()
{
    sigset_t newmask,zeromask,oldmask;
    if(signal(SIGINT, sig_int) == SIG_ERR)
    {
        err_sys("signal error");
    }
    if(signal(SIGQUIT, sig_int)==SIG_ERR)
    {
        err_sys("signal error");
    }

    sigemptyset(&newmask);
    sigemptyset(&zeromask);
    sigaddset(&newmask, SIGQUIT);

    if(sigprocmask(SIG_BLOCK, &newmask,&oldmask)<0)
    {
        err_sys("SIG_BOCK error");
    }

    while(quitflag == 0)
    {
        sigsuspend(&zeromask);
    }

    quitflag = 0;

    if(sigprocmask(SIG_SETMASK, &oldmask, NULL)<0)
    {
        err_sys("SIG_SETMASK error");
    }
    exit(0);
}

```

执行：

```
$ ./sigsuspend1.o 
^C
interrupt
^C
interrupt
^C
interrupt
^C
interrupt
^\$

```

考虑在第八章中，竞争条件的例程，其中我们使用了`TELL_**`和`WAIT_**`。这里我们可以使用信号来实现：

```c
#include"apue.h"

static void charatatime(char *);

static sigset_t newmask, oldmask;
static volatile sig_atomic_t sigflags = 0;

static void sig_usr(int signo)
{
    sigflags = 1;
}

void TELL_WAIT()
{
    sigemptyset(&newmask);
    sigaddset(&newmask, SIGUSR1);
    sigaddset(&newmask, SIGUSR2);
    sigprocmask(SIG_BLOCK, &newmask, &oldmask);
    if(signal(SIGUSR1, sig_usr)==SIG_ERR)
    {
        err_sys("siganl error");
    }
    if(signal(SIGUSR2, sig_usr)==SIG_ERR)
    {
        err_sys("siganl error");
    }
}

void WAIT_PARENT()
{
    sigset_t zeromask;
    sigemptyset(&zeromask);
    while(sigflags == 0)
    {
        sigsuspend(&zeromask);
    }
    sigflags = 0;
}

void TELL_CHILD(pid_t pid)
{
    kill(pid, SIGUSR2);
}

int main()
{
    pid_t pid;
    
    TELL_WAIT();

    if((pid=fork())<0)
    {
        printf("fork error\n");
    }
    else{
        if(pid==0)
        {
            WAIT_PARENT();
            charatatime("output from child\n");
        }
        else{
            charatatime("output from parent\n");
            TELL_CHILD(pid);
        }
    }
    exit(0);
}

static void charatatime(char *str)
{
    char *ptr;
    int c;
    setbuf(stdout, NULL);//设置标准输出无缓冲，更方便看到竞争
    for(ptr=str;(c=*ptr++)!=0;)
    {
        putc(c,stdout);
    }
}

```

## 函数abort

​		`abort`函数使程序异常终止：

```c
#include<stdlib.h>
void abort(void);

```

​		其方法是调用`raise（SIGABRT）`函数。

​		让进程捕捉`SIGABRT`的意图是：在进程终止之前由其执行所需清理操作。如果进程并不在信号处理程序中终止自己，POSIX.1申明当信号处理程序返回时，`abort`终止进程。

POSIX.1中`abort`的实现：

```c
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

void abort(void)
{
    sigset_t mask;
    struct  sigaction action;
    sigaction(SIGABRT, NULL, &action);
    if(action.sa_handler == SIG_IGN)
    {
        action.sa_handler = SIG_DFL;
        sigaction(SIGABRT, &action, NULL);
    }
    if(action.sa_handler == SIG_DFL)
    {
        fflush(NULL);
    }
    sigfillset(&mask);
    sigdelset(&mask, SIGABRT);
    sigprocmask(SIG_SETMASK, &mask, NULL);
    kill(getpid(), SIGABRT);

    fflush(NULL);
    action.sa_handler = SIG_DFL;
    sigaction(SIGABRT, &action, NULL);
    sigprocmask(SIG_SETMASK, &mask, NULL);
    kill(getpid(), SIGABRT);
    exit(0);
    


```

## 函数system

​		POSIX.1要求`system`或略`SIGINT`和`SIGQUIT`，阻塞`SIGCHLD`。对其原因解释的部分没看看明白。实现代码如下：

```c
#include<sys/wait.h>
#include<errno.h>
#include<unistd.h>
#include<signal.h>

int system(const char *cmdstring)
{
    pid_t pid;
    int status;
    struct sigaction ignore, saveintr, savequit;
    sigset_t chldmask, savemask;
    if(cmdstring == NULL)
    {
        return 1;
    }

    //忽略SIGQUIT SIGINT信号
    ignore.sa_handler = SIG_IGN;
    sigemptyset(&ignore.sa_mask);
    ignore.sa_flags = 0;
    if(sigaction(SIGINT, &ignore, &saveintr)<0)
    {
        return -1;
    }

    if(sigaction(SIGQUIT, &ignore, &saveintr)<0)
    {
        return -1;
    }

    // 阻塞SIGCHLD
    sigemptyset(&chldmask);
    sigaddset(&chldmask, SIGCHLD);
    if(sigprocmask(SIG_BLOCK, &chldmask, &savemask)<0)
    {
        return -1;
    }


    if((pid = fork())<0)
    {
        status = -1;
    }
    else{
        if(pid==0)
        {
            //子进程中恢复各个信号的处理
            sigaction(SIGINT, &saveintr, NULL);
            sigaction(SIGQUIT, &savequit, NULL);
            sigprocmask(SIG_SETMASK, &savemask, NULL);

            execl("/bin/sh", "sh", "-c", cmdstring, (char*)0); //如果程序正常执行，则会自己调用exit函数，否则使用下一条命令。
            _exit(127);
        }
        else{
            while (waitpid(pid, &status, 0)<0)
            {
                if(errno != EINTR)
                {
                    status = -1;
                    break;
                }
            }
        }
    }

    //恢复父进程的三个信号处理
    if(sigaction(SIGQUIT, &savequit, NULL)<0)
    {
        return -1;
    }
    if(sigaction(SIGINT, &saveintr, NULL)<0)
    {
        return -1;
    }
    if(sigprocmask(SIG_SETMASK, &savequit, NULL)<0)
    {
        return -1;
    }
    return status;
}

```

​		`system`返回值为`shell`终止状态。对于由于信号而终止的情况，终止状态为信号编号加上128。

## 函数`sleep`、`nanosleep`和`clock_nanosleep`

```c
#include<unistd.h>
unsigned int sleep(unsigned int seconds);
//返回0或未休眠完的秒数。

```

​		此函数将进程挂起，直到满足下面条件中的一个：

​		（1）：过了`seconds`设置的墙上时钟时间。

​		（2）：调用进程捕捉到了一个信号并从信号处理程序返回。

​		在第一中情形下，返回值是0，当由于捕捉到某个信号而提早返回时，返回值是未休眠完的秒数。由于其他系统活动（调用信号处理程序花费的时间），实际返回时间比要求要迟一些。

POSIX.1中的sleep的实现：

```c
#include"apue.h"

static void sig_alrm(int signo)
{
    //什么都不用做，只是用来唤醒进程
}

unsigned int sleep(unsigned int seconds)
{
    struct sigaction newact, oldact;
    sigset_t newmask, oldmask,suspmask;
    unsigned int unslept;

    //设置处理函数，保存原信息
    newact.sa_handler = sig_alrm;
    sigemptyset(&newact.sa_mask);
    newact.sa_flags = 0;
    sigaction(SIGALRM, &newact, &oldact);

    //阻塞SIGALRM并保存当前信号屏蔽字
    sigemptyset(&newmask);
    sigaddset(&newmask, SIGALRM);
    sigprocmask(SIG_BLOCK, &newmask, &oldmask);

    alarm(seconds);

    suspmask = oldmask;

    //确保SIGALRM不被阻塞
    sigdelset(&suspmask, SIGALRM);

    //等待任何信号被捕获
    sigsuspend(&suspmask);

    //当有信号被捕获后继续执行

    unslept = alarm(0);

    //恢复之前捕获函数
    sigaction(SIGALRM, &oldact, NULL);

    //恢复原来的信号屏蔽字
    sigprocmask(SIG_SETMASK, &oldmask, NULL);
    return unslept;
}

```

​		`nanosleep`和`sleep`函数类似，但提供了纳秒级的精度：

```c
#include<time.h>
int nanosleep(const struct timespce *reqtp, struct timespec *remtp);
//休眠到指定时间返回0，否则返回-1

```

`reqtp`指定休眠时间，提前返回时`remtp`返回剩余时间。

​		多系统时钟的引入，需要使用相对于特定时钟的延迟时间来挂起调用线程。`clock_nanosleep`提供了这种功能：

```c
#include<time.h>
int clock_nanosleep(clockid_t clock_id, int flags, const struct timespec *reqtp, struct timespec *remtp);
//达到休眠时间返回0，若出错，返回错误码

```

​		`clock_id`指定了计算延迟时间基于的时钟（6.10节）。`flags`控制延迟时间是相对还是绝对:0是相对时间（希望休眠时长），`TIMER_ABSTIME`是绝对（希望休眠到何时）。剩下两个参数与`nanosleep`一样。

## 函数sigqueue

​		在POSIX.1的实时扩展中，有些系统已经开始支持信号排队。

​		使用排队信号必须做一下几个操作：

1. 使用`sigaction`函数安装信号处理装置时指定`SA_SIGINFO`标志。
2. 在`sigaction`中的`sa_sigaction`成员中提供信号处理程序。
3. 使用`sigqueue`函数发送信号。

```c
#include<siganl.h>
int sigqueue(pid_t pid, int signo, const union sigval value);
//成功返回0，出错返回-1

```

## 作业控制信号

​		六个与作业控制有关的信号：

1. `SIGCHLD`：子进程停止或终止。

2. `SIGCONT`：如果进程停止，使进程继续运行。

3. `SIGSTOP`：停止信号（不能被捕捉或忽略）。

4. `SIGTSTP`：交互式停止信号。

5. `SIGTTIN`：后台进程组成员读控制终端。

6. `SIGTTOU`：后台进程组成员写控制终端。

   ​	当键入挂起字符（`Ctrl+Z`）时，`SIGTSTP`被送至前台进程组的所以进程。如果进程是停止的，则`SIGCONT`的默认动作是继续该进程，否则忽略该信号。当对一个停止的进程产生一个`SIGCONT`信号时，该进程就继续，即使该信号是阻塞或忽略。

## 信号名和编号

​		某些系统提供数组：

```
extern char *sys_siglist[];

```

可以使用`psignal`函数可移植地打印以信号编号对于的字符串：

```
#include<signal.h>
void psignal(int signo, const char*msg);

```

​		字符串`msg`（通常是程序名）输出到标准错误文件，后面跟随一个冒号和一个空格，再后面对该信号的说明，最后一个换行符。

​		如果在`sigaction`信号处理程序中有`siginfo`结构，可以使用`psiginfo`函数打印信号信息：

```
#include<signal.h>
void psiginfo(const siginfo_t *info, const char *msg);

```

​		如果只需要信号的字符描述部分，不需要写到标准错误文件中，可以使用`strsignal`函数：

```
#include<signal.h>
char *strsignal(int signo);

```

## 习题

10.6：使用`TELL_***`和`WAIT_***`写一个程序，父进程与子进程交替往一个文件中写入一个数和进程ID，数是递增的。

```c
#include"apue.h"
#include<stdio.h>
#include<fcntl.h>

static void charatatime(char *);

static sigset_t newmask, oldmask;
static volatile sig_atomic_t sigflags = 0;

static void sig_usr(int signo)
{
    sigflags = 1;
}

void TELL_WAIT()
{
    sigemptyset(&newmask);
    sigaddset(&newmask, SIGUSR1);
    sigaddset(&newmask, SIGUSR2);
    sigprocmask(SIG_BLOCK, &newmask, &oldmask);
    if(signal(SIGUSR1, sig_usr)==SIG_ERR)
    {
        err_sys("siganl error");
    }
    if(signal(SIGUSR2, sig_usr)==SIG_ERR)
    {
        err_sys("siganl error");
    }
}

void WAIT_PARENT()
{
    sigset_t zeromask;
    sigemptyset(&zeromask);
    while(sigflags == 0)
    {
        sigsuspend(&zeromask);
    }
    sigflags = 0;
    if(sigprocmask(SIG_SETMASK, &oldmask, NULL)<0)
    {
        err_sys("set mask error\n");
    }
}

void WAIT_CHILD()
{
    sigset_t zeromask;
    sigemptyset(&zeromask);
    while(sigflags == 0)
    {
        sigsuspend(&zeromask);
    }
    sigflags = 0;
    if(sigprocmask(SIG_SETMASK, &oldmask, NULL)<0)
    {
        err_sys("set mask error\n");
    }
}

void TELL_CHILD(pid_t pid)
{
    kill(pid, SIGUSR2);
}

void TELL_PARENT(pid_t pid)
{
    kill(pid, SIGUSR1);
}

int main()
{
    pid_t pid;
    
    TELL_WAIT();

    int num = 0;
    FILE *fp;
    if((fp =fopen("temp.txt", "r+b"))==NULL)
    {
        printf("fopen error\n");
        return 0;
    }
    fprintf(fp, "num = %d, pid = %ld\n", num, (long)getpid());
    fflush(fp);
    if((pid=fork())<0)
    {
        printf("fork error\n");
    }
    else{
        int start = 0;
        while(num < 100)
        {
            if(pid==0)
            {
                if(start == 0)
                {
                    num++;
                    start = 1;
                }
                else {
                    WAIT_PARENT();
                    num += 2;
                }
                fprintf(fp, "num = %d, pid = %ld\n", num, (long)getpid());
                fflush(fp);
                TELL_PARENT(getppid());
            }
            else{
                WAIT_CHILD();
                num += 2;
                fprintf(fp, "num = %d, pid = %ld\n", num, (long)getpid());
                fflush(fp);
                TELL_CHILD(pid);
            }
        }
    }
    fclose(fp);
    exit(0);
}

```

10_11:

```c
#include<stdio.h>
#include<unistd.h>
#include<sys/resource.h>
#include<fcntl.h>
#include<signal.h>
#include<stdlib.h>

static void sig_xfsz(int signo)
{
    printf("rlimit of xdsz is too small\n");
    exit(0);
}

int main()
{
    char buf[100];
    int n;
    rlimit rtp;
    rtp.rlim_cur = 1024;
    rtp.rlim_max = RLIM_INFINITY;
    if(signal(SIGXFSZ, sig_xfsz)==SIG_ERR)
    {
        perror("signal error\n");
        return 0;
    }
    if(setrlimit(RLIMIT_FSIZE, &rtp)==0)
    {
        int cur = open("CMakeCache.txt", O_RDONLY);
        int cp = creat("tp.txt", O_RDWR);
        unlink("tp.txt");
        while ((n=read(cur, buf, 100))>0)
        {
            if(write(cp, buf, n)!=n)
            {
                perror("write error\n");
            }
            else{
                printf("%d\n", n);
            }
        }
        close(cur);
        close(cp);
    }
    else
    {
        perror("setrlimt error\n");
    }
}

```

# 第十一章 线程

## 线程概念

多线程的好处：

1. 通过为每种事件类型分配单独的处理线程，可以简化处理异步事件的代码。每个线程在进行事件处理时可以采用同步编程模式，这比异步编程模式简单很多。
2. 多个进程需要使用操作系统提供的复杂机制才能实现内存和文件描述符的共享，而多线程自动共享存储空间和文件描述符。
3. 有些问题可以分解从而提高整个程序的吞吐量。
4. 交互的程序可以通过多线程来改善响应时间，多线程可以把处理用户输入输出的部分与其他部分分离。

每个线程都包含有执行环境所必须的信息，其中包括进程中标识线程的线程ID、一组寄存器值、栈、调度优先级和策略、信号屏蔽字、`errno`变量以及线程私有数据。一个进程的所有信息对该进程的所有线程都是共享的，包括可执行程序的代码、程序的全局内存和堆内存、栈以及文件描述符。

## 线程标识

线程ID只在它所属的进程上下文中才有意义。线程ID使用`pthread_t`数据类型来表示。函数`pthread_equal`函数可以用来比较两个线程ID：

```c
#include<pthread.h>
int pthread_equal(pthread_t tid1, pthread_t tid2);
//相等返回非0，否则返回0

```

调用`pthread_self()`可以获得自身的线程ID：

```c
#include<pthread.h>
int pthread_self();
//返回自身线程ID

```

## 线程创建

函数`pthread_create`用来创建新的线程：

```c
#include<pthread.h>
int pthread_create(pthread_t *restrict tidp, const pthread_attr_t *restrict attr, void *(*start_rtn)(void *), void *restarict arg);
//成功返回0，否则返回错误编码

```

当成功返回时，新创建的线程ID会被设置为`tidp`指向的内存单元。`attr`参数用于定制各种不同的线程属性，具体会在第十二章讨论，使用NULL则是创建一个默认属性的线程。

新创建的线程从`start_rtn`函数的地址开始运行，该函数只有一个无类型指针参数`arg`。如果要向`start_rtn`传递的参数有一个以上，那么需要将参数放到一个结果中，将结果的地址作为`arg`参数传入。

`pthread`函数调用之后通常会返回错误码，这一点并不像其他POSIX函数一样设置`errno`。每个线程都提供`errno`的副本，这只是为了与使用`errno`的现有函数兼容。

例程：打印线程ID

```c
#include<pthread.h>
#include<stdio.h>
#include<unistd.h>
#include"apue.h"

pthread_t ntid;

void printids(const char *s)
{
    pid_t pid;
    pthread_t tid;

    pid = getpid();
    tid = pthread_self();
    printf("%s pid %lu tid %lu (0x%lx)\n", s, (unsigned long)pid, (unsigned long)tid, (unsigned long)tid);
}

void *thr_fn(void *arg)
{
    printids("new thread: ");
    return (void *)0;
}

int main(void)
{
    int err;
    err = pthread_create(&ntid, NULL, thr_fn, NULL);
    if(err!=0)
    {
        err_exit(err, "can't create thread");
    }
    printids("main thread: ");
    sleep(1);
    exit(0);
}

```

由于`pthread`不是linux下默认的库，因此需要添加链接：

```
$ g++ a.cpp -o a.o -lapue -lthread

```

这里需要注意两个问题：第一，要让主线程sleep一秒，这是要等待新线程执行完毕，如果主线程返回了，而新线程还没执行完成，则整个进程返回，新线程将不会执行了。第二，在新线程中获取线程ID也要使用`pthread_self`而不能直接使用存储在`ntid`中的，这是因为不能保证在执行子线程时，函数`pthread_create`已经返回了，如果这时候直接使用`ntid`，则可能是未初始的内容。

在Linux下运行结果：

```
$ ./pthread1.o 
main thread:  pid 9416 tid 140693008582464 (0x7ff5a4cc8740)
new thread:  pid 9416 tid 140693000173312 (0x7ff5a44c3700)

```

## 线程终止

如果进程中的任意线程调用了`exit`、`_Exit`或者`_exit`，那么整个进程就会终止。单个线程可以通过3种方式退出，可以在不终止整个进程的情况下，停止它的控制流：

1. 线程可以简单的从启动例程中返回，返回值是线程的退出码（即返回的指针存储的内容）。

2. 线程可以被同一进程中的其他线程取消。

3. 线程调用`pthread_exit`：

   ```c
   #include<pthread.h>
   void pthread_exit(void *rval_ptr);
   
   ```

   `rval_ptr`参数是一个无类型参数指针，与传递给启动例程的单个参数类似。进程中的其他线程也可以通过调用`pthread_join`函数访问到这个指针：

   ```c
   #include<pthread.h>
   int pthread_join(pthread_t thread, void **reval_ptr);
   //成功返回0，否则返回错误编号
   
   ```

   调用线程将一直阻塞，直到指定的线程调用`pthread_exit`、从启动例程中返回或者被取消。如果是简单的从例程中返回，`rval_ptr`包含返回码。如果线程被取消，由`rval_ptr`指定的内存单元就设置为`PTHREAD_CANCELED`。

   可以通过调用`pthread_join`自动将进程置于分离状态（随后讨论），这样资源就可以恢复。如果线程已经处于分离状态，`pthread_join`就会失败，返回`EINVAL`。对线程返回不感兴趣，可以将`rval_ptr`设置为NULL。此时调用`pthread_join`函数等待指定线程终止，不获取线程终止状态。

   例程：

   ```c
   #include"apue.h"
   #include<pthread.h>
   
   void *thr_fn1(void *arg)
   {
       printf("thread 1 returning\n");
       return (void*)1;
   }
   
   void *thr_fn2(void *arg)
   {
       printf("thread 2 returning\n");
       pthread_exit((void*)2);
   }
   
   int main(void)
   {
       int err;
       pthread_t tid1, tid2;
       void *tret;
       err = pthread_create(&tid1, NULL, thr_fn1, NULL);
       if(err!=0)
       {
           err_exit(err, "can't create thread 1");
       }
       err = pthread_create(&tid2, NULL, thr_fn2, NULL);
       if(err!=0)
       {
           err_exit(err, "can't create thread 1");
       }
       err = pthread_join(tid1, &tret);
       if(err!=0)
       {
           err_exit(err, "can't join with thread 1");
       }
       printf("thread 1 exit code %ld\n", (long)tret);
       err = pthread_join(tid2, &tret);
       if(err!=0)
       {
           err_exit(err, "can't join with thread 2");
       }
       printf("thread 2 exit code %ld\n", (long)tret);
       exit(0);
   }
   
   ```

   `pthread_create`和`pthread_exit`函数的无类型指针参数可以传递的参数的值不止一个，这个指针可以传递包含复杂信息的结构的地址，但，这个结构所使用的内存在调用者完成调用后必须仍然是有效的。如果是在栈中分配的空间，则其他线程在使用这个结构时内存内容可能已经改变了。

   例如:

   ```c
   #include<pthread.h>
   #include"apue.h"
   
   struct foo{
       int a,b,c,d;
   };
   
   void printfoo(const char *s, const foo *fp)
   {
       printf("%s",s);
       printf(" structure at 0x%lx\n",(unsigned long)fp);
       printf(" foo.a = %d\n", fp->a);
       printf(" foo.b = %d\n", fp->b);
       printf(" foo.c = %d\n", fp->c);
       printf(" foo.d = %d\n", fp->d);
   }
   
   void *thr_fn1(void *arg)
   {
       foo fo = {1, 2, 3, 4};
       printfoo("thread 1:\n", &fo);
       pthread_exit((void*)&fo);
   }
   
   void *thr_fn2(void *arg)
   {
       printf("thread 2: ID is %lu\n", (unsigned long)(pthread_self()));
       pthread_exit((void*)0);
   }
   
   int main(void)
   {
       int err;
       pthread_t tid1, tid2;
       foo *fp;
       err = pthread_create(&tid1, NULL, thr_fn1, NULL);
       if(err!=0)
       {
           err_exit(err, "can't create thread");
       }
   
       err = pthread_join(tid1, (void **)&fp);
   
       if(err!=0)
       {
           err_exit(err, "can't join with thread 1");
       }
   
       sleep(1);
   
       printf("parent starting second thread\n");
   
       err = pthread_create(&tid2, NULL, thr_fn2, NULL);
       if(err!=0)
       {
           err_exit(err, "can't create thread");
       }
       sleep(2);
       printfoo("parent:\n",fp);
       exit(0);
   }
   
   ```

   执行结果：

   ```
   $ ./pthread_error.o 
   thread 1:
    structure at 0x7f48ad896ed0
    foo.a = 1
    foo.b = 2
    foo.c = 3
    foo.d = 4
   parent starting second thread
   thread 2: ID is 139950125840128
   parent:
    structure at 0x7f48ad896ed0
    foo.a = -1379403936
    foo.b = 32584
    foo.c = -1381808558
    foo.d = 32584
   
   ```

   线程可以调用`pthread_cancel`函数来请求取消同一进程的其他线程：

   ```c
   #include<pthread.h>
   int pthread_cancel(pthread_t tid);
   //成功返回0，否则返回错误编号
   
   ```

   默认情况下，`pthread_cancel`函数会使得由`tid`标识的线程行为表现为如同调用了参数为`PTHREAD_CANCELED`的`pthread_exit`函数，但线程可以选择忽略取消或者控制如何被取消。`pthread_cancel`函数并不等待线程终止，仅仅是提出请求。

   线程可以安排其退出时需要调用的函数，这与进程在退出时可以用`atexit`函数安排退出类似。这样的函数称为线程清理处理程序。一个线程可以建立多个清理处理程序。处理程序记录在栈中，即执行顺序与注册顺序相反：

   ```c
   #include<pthread.h>
   void pthread_cleanup_push(void (*rtn)(void *), void *arg);
   void pthread_cleanup_pop(int execute);
   
   ```

   当线程执行以下动作时，清理函数`rtn`由`pthread_cleanup_push`函数调度的，调用时只有一个参数`arg`：

   1. 调用`pthread_exit`时；
   2. 响应取消请求时；
   3. 用非零`execute`参数调用`pthrea_cleanup_pop`函数时。

   如果`execute`参数设置为0，清理函数不会被调用。不管发生哪种情况，`pthread_cleanup_pop`都将删除上次`pthread_cleanup_push`调用建立的清理处理程序。

   例程：

   ```c
   #include"apue.h"
   #include<pthread.h>
   
   void cleanup(void *arg)
   {
       printf("cleanup: %s\n", (char*)arg);
   }
   
   void *thr_fn1(void *arg)
   {
       printf("thread 1 start\n");
       pthread_cleanup_push(cleanup, (void *)"thread 1 first handler");
       pthread_cleanup_push(cleanup, (void *)"thread 1 second handler");
       printf("thread 1 push complete\n");
       if((int*)arg)
       {
           return (void*)1;
       }
       pthread_cleanup_pop(0);
       pthread_cleanup_pop(0);
       return (void*)1;
   }
   
   void *thr_fn2(void *arg)
   {
       printf("thread 2 start\n");
       pthread_cleanup_push(cleanup, (void *)"thread 2 first handler");
       pthread_cleanup_push(cleanup, (void*)"thread 2 second handler");
       printf("thread 2 push complete\n");
       if((int*)arg)
       {
           pthread_exit((void*)2);
       }
       pthread_cleanup_pop(0);
       pthread_cleanup_pop(0);
       pthread_exit((void*)2);
   }
   
   int main(void)
   {
       int err;
       pthread_t tid1, tid2;
       void *tret;
       err = pthread_create(&tid1, NULL, thr_fn1, (void*)1);
       if(err!=0)
       {
           err_exit(err, "can't create thread 1");
       }
       err = pthread_create(&tid2, NULL, thr_fn2, (void*)1);
       if(err!=0)
       {
           err_exit(err, "can't create thread 2");
       }
       err = pthread_join(tid1, &tret);
       if(err!=0)
       {
           err_exit(err, "can't join thread 1");
       }
       printf("thread 1 exit code %ld\n", (long)tret);
   
       err = pthread_join(tid2, &tret);
       if(err!=0)
       {
           err_exit(err, "can't join thread 2");
       }
       printf("thread 2 exit code %ld\n", (long)tret);
       exit(0);
   }
   
   ```

执行结果：

```
$ ./pthread_clean.o 
thread 1 start
thread 1 push complete
thread 2 start
thread 2 push complete
cleanup: thread 1 second handler
cleanup: thread 1 first handler
cleanup: thread 2 second handler
cleanup: thread 2 first handler
thread 1 exit code 1
thread 2 exit code 2

```

与书中所述存在差异，书中说只有第二个新进程执行了线程清理处理程序，认为线程如果通过启动例程中返回而终止，就不会执行清理处理函数，但在当前Linux下，好像也会执行，似乎是新版的Linux进行了改变。

进程与线程存在很多相似之处，可以使用下表总结：

| 进程原语  | 线程原语               | 描述                           |
| --------- | ---------------------- | ------------------------------ |
| `fork`    | `pthread_create`       | 创建新的控制流。               |
| `exit`    | `pthread_exit`         | 从现有的控制流中退出。         |
| `waitpid` | `pthread_jooin`        | 从控制流中获取退出状态。       |
| `atexit`  | `pthread_cleanup_push` | 注册在退出控制流时调用的函数。 |
| `getpid`  | `pthread_self`         | 获取控制流ID。                 |
| `abort`   | `pthread_cancel`       | 请求控制流的非正常退出。       |

在默认情况下，线程的终止状态会保存直到对线程调用`pthread_join`。如果线程已经被分离，线程的底层存储资源可以在线程终止时立即被收回。在线程被分离后，我们不能用`pthread_join`函数等待它的终止状态，此后会产生未定义的行为。

可以调用`pthread_detach`分离线程：

```c
#include<pthread.h>
int pthread_detach(pthread_t tid);
//成功返回0，否则返回错误编号。

```

## 线程同步

当多个控制线程共享相同的内存时，需要确保每个线程看到一致的数据视图。当一个线程可以修改的变量，其他线程也可以读取或者修改的时候，我们需要对这些线程进行同步，确保他们在访问变量的存储内容时不会访问到无效的值。

### 互斥量

可以使用`pthread`的互斥接口来保存数据，确保同一时间只有一个线程访问数据。互斥量（`mutex`）从本质上来书其实是一把锁，在访问共享资源前对互斥进行设置（加锁），在访问完成后释放（解锁）互斥量。当对互斥量加锁后，任何其他视图再次对互斥量加锁的线程将会被阻塞直到当前线程释放该互斥锁。

互斥变量是用`pthread_mutex_t`数据类型表示的。在使用互斥变量之前，必须首先对它进行初始化，可以将其设置为常量`PTHREAD_MUTEX_INITIALIZER`（只适用于静态分配的互斥量），也可以调用`pthread_mutex_init`函数进行初始化。如果动态分配互斥量（例如使用`malloc`），在释放内存前需要调用`pthread_mutex_destroy`。

```c
#include<pthread.h>
int pthread_mutex_init(pthread_mutex_t *restrict mutex, const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destory(pthread_mutex_t *mutex);
//成功返回0，否则返回错误编号

```

要用默认属性初始化互斥量，只需把`attr`设置为NULL。

对互斥量加锁，使用`pthread_mutex_lock`，如果互斥量已经上锁，则调用线程阻塞直到互斥量被解锁。对互斥量解锁需要调用`pthread_mutex_unlock`。

```c
#include<pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
//成功返回0，错误返回错误编号

```

如果不希望线程阻塞，可以使用`pthread_mutex_trylock`尝试对互斥量加锁。如果调用`pthread_mutex_trylock`时互斥量处于未锁状态，那么`pthread_mutex_trylock`会锁住互斥量，不会出现阻塞直接返回0，否则`pthread_mutex_trylock`就会失败，不能锁住互斥量，返回`EBUSY`。

例程：保护某个结构的互斥量，当一个以上线程需要访问动态分配的对象时，我们在对象中加入引用计数，确保在所以使用该对象的线程完成访问数据之前，该对象空间不会被释放：

```c
#include<stdlib.h>
#include<pthread.h>

struct foo
{
    int f_cout;
    pthread_mutex_t f_lock;
    int f_id;
};

struct foo * foo_alloc(int id)
{
    foo *fp;
    if((fp = (foo*)malloc(sizeof(foo)))!=NULL)
    {
        fp->f_cout = 1;
        fp->f_id = id;
        if(pthread_mutex_init(&fp->f_lock, NULL)!=0)
        {
            free(fp);
            return NULL;
        }
    }
};

void foo_hold(foo *fp)
{
    pthread_mutex_lock(&fp->f_lock);
    fp->f_cout++;
    pthread_mutex_unlock(&fp->f_lock);
}

void foo_rele(foo *fp)
{
    pthread_mutex_lock(&fp->f_lock);
    if(--fp->f_cout == 0)
    {
        pthread_mutex_unlock(&fp->f_lock);
        pthread_mutex_destroy(&fp->f_lock);
        free(fp);
    }
    else{
        pthread_mutex_unlock(&fp->f_lock);
    }
}

```

这里忽略了线程在调用`foo_hold`之前是如何找到对象的。同时，如果有另一个线程正在调用`foo_hold`时阻塞等待互斥锁，这时即使该对象引用计数达到0，`foo_rele`释放该对象依旧是不对的。

### 避免死锁

如果一个线程企图对一个互斥量加锁两次，那么它自身就会陷入死锁状态。当存在一个以上互斥量时，如果允许一个线程一直占用一个互斥量，并且在试图锁住第二个互斥量时处于阻塞状态，但是拥有第二个互斥量的线程也在试图锁住第一个互斥量。由于两个线程都在相互请求另一个线程用于的资源，所以两个线程都无法前进，于是产生死锁。

<font color =red>可能出现死锁只会发生在一个线程试图锁住另一个线程以相反的顺序锁住的互斥量。</font>

更新上一节的例程，添加一个散列表用来实现线程获取结构，同时需要对散列表加锁，在同时需要两个互斥锁时，总是以相同的顺序加锁，这样可以避免死锁：

```c
#include<stdlib.h>
#include<pthread.h>

#define NHASH 29
#define HASH(id) (((unsigned long)id)%29)

struct foo
{
    int f_cout;
    pthread_mutex_t f_lock;
    int f_id;
    foo *f_next;
};

foo *fh[NHASH];
pthread_mutex_t hashlock = PTHREAD_MUTEX_INITIALIZER;

struct foo * foo_alloc(int id)
{
    foo *fp;
    int idx;
    if((fp = (foo*)malloc(sizeof(foo)))!=NULL)
    {
        fp->f_cout = 1;
        fp->f_id = id;
        if(pthread_mutex_init(&fp->f_lock, NULL)!=0)
        {
            free(fp);
            return NULL;
        }
        idx = HASH(id);
        pthread_mutex_lock(&hashlock);
        fp->f_next = fh[idx];
        fh[idx] = fp;
        pthread_mutex_lock(&fp->f_lock);
        pthread_mutex_unlock(&hashlock);
        pthread_mutex_unlock(&fp->f_lock);
    }
};

void foo_hold(foo *fp)
{
    pthread_mutex_lock(&fp->f_lock);
    fp->f_cout++;
    pthread_mutex_unlock(&fp->f_lock);
}

foo *foo_find(int id)
{
    foo *fp;
    pthread_mutex_lock(&hashlock);
    for(fp=fh[HASH(id)]; fp!=NULL; fp = fp->f_next)
    {
        if(fp->f_id == id)
        {
            foo_hold(fp);
            break;
        }
    }
    pthread_mutex_unlock(&hashlock);
    return fp;
}

void foo_rele(foo *fp)
{
    foo *tfp;
    int idx;
    pthread_mutex_lock(&fp->f_lock);
    if(fp->f_cout == 1)
    {
        pthread_mutex_unlock(&fp->f_lock);
        pthread_mutex_lock(&hashlock);
        pthread_mutex_lock(&fp->f_lock);
        if(fp->f_cout != 1)
        {
            fp->f_cout--;
            pthread_mutex_unlock(&fp->f_lock);
            pthread_mutex_unlock(&hashlock);
            return;
        }
        idx = HASH(fp->f_id);
        tfp = fh[idx];
        if(tfp == fp)
        {
            fh[idx] = fp->f_next;
        }
        else{
            while(tfp->f_next != fp)
            {
                tfp = tfp->f_next;
            }
            tfp->f_next = fp->f_next;
        }
        pthread_mutex_unlock(&hashlock);
        pthread_mutex_unlock(&fp->f_lock);
        pthread_mutex_destroy(&fp->f_lock);
        free(fp);
    }
    else{
        fp->f_cout--;
        pthread_mutex_unlock(&fp->f_lock);
    }
}

```

这里主要交互函数为`foo_find`、`foo_alloc`和`foo_rele`。这里这三个函数都是先锁住散列表再锁住指定元素。因此不会发生死锁。但我认为这里存在一个问题：1. 如果有一个或多个线程调用`foo_find`查找指定id的元素，当其不存在时，应该会调用`foo_alloc`进行初始化一个，而这时可能有多个线程调用`foo_alloc`函数创建同一个对象，此时可能造成生成多个重复元素。解决的办法我想到两个，第一个方式是在调用`foo_find`如果未找到指定id的元素，不释放散列表的锁，直接进行创建新元素。而后在释放散列表的锁，此时，只有第一个查询的进程会创建，其后的进程再使用`foo_find`时就存在该元素了，只会在其上引用计数上加1。第二种方式是，在函数`foo_alloc`获得散列表的锁之后，再次检查指定id的元素是否存在，如果存在就不创建了，只在其引用计数上加1。（其实这里问题不大，即使创建了多个，在之后查找的时候也会能找到的，但是这将对其他数据进行多余的拷贝，而且可能出现其他问题，个人拙见，还望更明白的人赐教）。

上述代码还可以进行简化，考虑每次操作都是先获取散列表锁，再获得元素锁。由于三个函数都是这样，因此其实我们可以只获取散列表锁即可：

```c
#include<stdlib.h>
#include<pthread.h>

#define NHASH 29
#define HASH(id) (((unsigned long)id)%29)

struct foo
{
    int f_cout;
    // pthread_mutex_t f_lock;
    int f_id;
    foo *f_next;
};

foo *fh[NHASH];
pthread_mutex_t hashlock = PTHREAD_MUTEX_INITIALIZER;

struct foo * foo_alloc(int id)
{
    foo *fp;
    int idx;
    if((fp = (foo*)malloc(sizeof(foo)))!=NULL)
    {
        fp->f_cout = 1;
        fp->f_id = id;
        if(pthread_mutex_init(&fp->f_lock, NULL)!=0)
        {
            free(fp);
            return NULL;
        }
        idx = HASH(id);
        pthread_mutex_lock(&hashlock);
        fp->f_next = fh[idx];
        fh[idx] = fp;
        // pthread_mutex_lock(&fp->f_lock);
        pthread_mutex_unlock(&hashlock);
        // pthread_mutex_unlock(&fp->f_lock);
    }
};

void foo_hold(foo *fp)
{
    pthread_mutex_lock(&hashlock);
    fp->f_cout++;
    pthread_mutex_unlock(&hashlock);
}

foo *foo_find(int id)
{
    foo *fp;
    pthread_mutex_lock(&hashlock);
    for(fp=fh[HASH(id)]; fp!=NULL; fp = fp->f_next)
    {
        if(fp->f_id == id)
        {
            foo_hold(fp);
            break;
        }
    }
    pthread_mutex_unlock(&hashlock);
    return fp;
}

void foo_rele(foo *fp)
{
    foo *tfp;
    int idx;
    pthread_mutex_lock(&hashlock);
    if(--fp->f_cout == 0)
    {
        // pthread_mutex_unlock(&fp->f_lock);
        // pthread_mutex_lock(&hashlock);
        // pthread_mutex_lock(&fp->f_lock);
        // if(fp->f_cout != 1)
        // {
        //     fp->f_cout--;
        //     pthread_mutex_unlock(&fp->f_lock);
        //     pthread_mutex_unlock(&hashlock);
        //     return;
        // }
        idx = HASH(fp->f_id);
        tfp = fh[idx];
        if(tfp == fp)
        {
            fh[idx] = fp->f_next;
        }
        else{
            while(tfp->f_next != fp)
            {
                tfp = tfp->f_next;
            }
            tfp->f_next = fp->f_next;
        }
        pthread_mutex_unlock(&hashlock);
        // pthread_mutex_unlock(&fp->f_lock);
        // pthread_mutex_destroy(&fp->f_lock);
        free(fp);
    }
    else{
        fp->f_cout--;
        pthread_mutex_unlock(&hashlock);
    }
}

```

多线程软件设计涉及两者之间的折中。如果锁的粒度太粗，就会出现很多线程等待相同的锁，这可能不能改善并发性，如果锁的粒度太细，那么过多的锁开销会使系统性能受到影响，并且代码变得复杂。

### 函数pthread_mutex_timedlock

`pthread_mutex_timedlock`与`pthread_mutex_lock`基本是等价的，区别在于，前者可以设定一个时间值，如果超时，就不会对互斥量进行加锁了，而是返回错误码`ETIMEOUT`。

```c
#include<pthread.h>
#include<time.h>
int pthread_mutex_timedlock(pthread_mutex_t *restarict mutex, const struct timespec *restrict tspr);
//成功返回0，否则返回错误编号

```

这里的时间值为绝对时间，即愿意等待到何时而并不是愿意等待多久。

例程：

```c
#include"apue.h"
#include<pthread.h>

int main(void)
{
    pthread_mutex_t tid;
    pthread_mutex_init(&tid,NULL);
    timespec tmp;
    char buf[64];

    int err;
    pthread_mutex_lock(&tid);
    printf("tid has been locked\n");
    clock_gettime(CLOCK_REALTIME,&tmp);
    tm *time = localtime(&tmp.tv_sec);
    strftime(buf, sizeof(buf), "%r", time);
    printf("current time is : %s\n", buf);
    tmp.tv_sec += 5;
    printf("again try lock tid\n");
    err = pthread_mutex_timedlock(&tid, &tmp);

    clock_gettime(CLOCK_REALTIME,&tmp);
    time = localtime(&tmp.tv_sec);
    strftime(buf, sizeof(buf), "%r", time);
    printf("the time is now : %s\n", buf);
    if(err!=0)
    {
        printf("can't lock tid again: %s\n", strerror(err));
    }
    else{
        printf("tid lock again\n");
    }
    exit(0);

}

```

这里尝试在同一个线程对同一个互斥量加两次锁，以此来验证超时。

### 读写锁

互斥锁只有两种状态，要么是锁住，要么是未锁，而且一次只能有一个线程可以对其加锁。读写锁有三个状态：读模式下加锁，写模式下加锁，不加锁状态。一次只能有一个线程可以占有写模式的读写锁，但可以有多个线程可以同时占有写模式的读写锁。

当读写锁是写状态加锁时，在锁状态被解锁之前，所有试图对这个锁加锁的线程都会被阻塞。当读写锁在读加锁状态时，所有试图对其加锁的线程都可以得到访问权限，但是任何希望以写模式进行加锁的进程会阻塞，直到所有的线程释放它们的读锁为止。当读写锁处于读模式锁住状态，而这是有一个线程试图以写模式获取锁时，读写锁会阻塞随后的读模式锁请求，这样可以避免读模式锁长期占用，而写模式锁请求一直无法满足。

读写锁适合于对数据结构读的次数远大于写的情况。读写锁又叫共享互斥锁，当读模式锁住时，可以说是共享模式锁住的，以写模式锁住时，可以说是互斥模式锁住的。

读写锁在使用之前必须初始化，在释放底层内存之前必须销毁：

```c
#include<pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock, const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destory(pthread_rwlock_t *rwlock);
//成功返回0，否则返回错误编号

```

当`attr`为`NULL`时，采用默认初始化。获取读写锁和释放读写锁采用下面的函数：

```c
#include<pthread.h>
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pyhread_rwlock_unlock(pthread_rwlock_t *rwlock);
//成功返回0，否则返回错误编号

```

标准还定义了读写锁原语的条件版本：

```c
#include<pthread.h>
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
///成功返回0，否则返回错误编号

```

可以获取锁时，函数返回0，否则返回错误`EBUSY`。

例程：

```c
#include<pthread.h>
#include"apue.h"

//任务
struct job{
    job *j_next;
    job *j_prev;
    pthread_t pid; //分配给哪个线程
};

//任务队列
struct queue{
    job *q_head;
    job *q_tail;
    pthread_rwlock_t q_lock;
};

//初始化队列
int queue_init(queue *qp)
{
    int err;
    qp->q_head = NULL;
    qp->q_tail = NULL;
    if((err = pthread_rwlock_init(&qp->q_lock, NULL))!=0)
    {
        return err;
    }
    return 0;
}

//队列前添加任务
void job_inser(queue *qp, job *jp)
{
    pthread_rwlock_wrlock(&qp->q_lock);
    jp->j_next = qp->q_head;
    if(qp->q_head != NULL)
    {
        qp->q_head->j_prev = jp;
    }
    else{
        qp->q_tail = jp;
    }
    qp->q_head = jp;
    jp->j_prev = NULL;
    pthread_rwlock_unlock(&qp->q_lock);
}

//队尾添加任务
void job_append(queue *qp, job *jp)
{
    pthread_rwlock_wrlock(&qp->q_lock);
    jp->j_next = NULL;
    jp->j_prev = qp->q_tail;
    if(qp->q_tail != NULL)
    {
        qp->q_tail->j_next = jp;
    }
    else{
        qp->q_head = jp;
    }
    qp->q_tail = jp;
    pthread_rwlock_unlock(&qp->q_lock);
}

void job_move(queue *qp,job *jp)
{
    pthread_rwlock_wrlock(&qp->q_lock);
    if(jp == qp->q_head)
    {
        qp->q_head = jp->j_next;
        if(jp == qp->q_tail)
        {
            qp->q_tail = NULL;
        }
        else{
            jp->j_next->j_prev = jp->j_prev;
        }
    }
    else{
        if(jp == qp->q_tail)
        {
            qp->q_tail = jp->j_prev;
            qp->q_tail->j_next = NULL;
        }
        else{
            jp->j_next->j_prev = jp->j_prev;
            jp->j_prev->j_next = jp->j_next;
        }
    }
    pthread_rwlock_unlock(&qp->q_lock);
}

//寻找任务列表中第一个线程为id的任务
job *find_job(queue *qp, pthread_t id)
{
    job *jp;
    if(pthread_rwlock_rdlock(&qp->q_lock)!=0)
    {
        return NULL;
    }
    for(jp = qp->q_head; jp!=NULL; jp = jp->j_next)
    {
        if(pthread_equal(jp->pid, id) == 0)
        {
            break;
        }
    }
    pthread_rwlock_unlock(&qp->q_lock);
    return jp;
}

```

这里实现一个简单的任务队列，可以队列的任务都被分配给指定进程，对队列写时要获得读锁，读时获取读锁。

### 带有超时的读写锁

为了避免获取读写锁时一直处于堵塞状态，标准定义了带有超时的读写锁：

```c
#include<pthread.h>
#include<time.h>
int pthread_rwlock_timedrdlock(pthread_rwlock_t *restrict rwlock, const struct timespec *restrict tsptr);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *restrict rwlock, const struct timespec *restrict tsptr);
//成功返回0，否则返回错误编号

```

时间依旧是绝对值。

### 条件变量

条件变量是另一种同步机制。其为多个线程提供了一个会和的场所。条件变量与互斥量一起使用时，允许线程以无竞争的方式等待特定条件发生。

条件变量本身是由互斥量保护的。线程在改变条件状态之前必须首先锁住互斥量。其他线程在获得互斥量之前不会察觉到这种改变，因为互斥量必须在锁住之后才能计算条件。

`pthread_cond_t`类型为条件变量，初始化和反初始化方式如下，常量可以使用`PTHREAD_COND_INITIALIZER`直接赋值：

```c
#include<pthread.h>
int pthread_cond_init(pthread_cond_t *restrict cond, const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);
//成功返回0，否则返回错误编号

```

`attr`为`NULL`时表示默认初始化。

我们使用`pthread_cond_wait`等待条件变为真，如果指定时间内不能满足，则返回错误码：

```c
#include<pthread.h>
int pthread_cond_wait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond, pthread_mutex_t *restrict mutex, const timespec *restrict tspr);
//成功返回0，否则返回错误编号

```

传递给` pthread_cond_wait`的互斥量对条件进行保护。调用者把锁住的互斥量传递给函数，函数随后自动把调用线程放到等待条件的线程列表上，对互斥量解锁。` pthread_cond_wait`返回时，互斥量将再次被锁住。这里互斥量与条件变量没有太大关系，只是提供一个保护，一般应该是一个在该条件满足后，后续执行的代码要求获取的一个互斥量，真正与条件绑定的还是条件变量。

如果超时条件还未出现，`pthread_cond_timedwait`将重新获得互斥量，然后返回错误`ETIMEOUT`。从`pthread_cond_timedwai`或`pthread_cond_wait`调用成功返回，需要重新计算条件，因为另一个线程可能已经在运行并改变了条件。（具体参看下面的例子）

有两个函数可以通知线程条件已经满足：

```c
#include<pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
//成功返回0，否则返回错误编号

```

`pthread_cond_signal`至少能唤醒一个等待该条件的线程，`pthread_cond_broadcast`则唤醒等待该条件的所以线程。调用二者时，我们说这是在给线程或者条件发信号，必须要在改变条件状态以后再给线程发信号。

例程：

```c
#include<pthread.h>

//消息类
struct msg{
    struct msg *m_next;
};

//消息链表
msg *workq;

pthread_cond_t qready = PTHREAD_COND_INITIALIZER;

pthread_mutex_t qlock = PTHREAD_MUTEX_INITIALIZER;

void process_mag(void)
{
    msg *mp;
    while (1)
    {
        pthread_mutex_lock(&qlock);
        /*这里使用while，就是上文所述，当从pthread_cond_timedwai或
        pthread_cond_wait调用成功返回，需要重新计算条件，
        因为另一个线程可能已经在运行并改变了条件。
        这里条件变量绑定的条件就是消息链表非空*/
        while(workq == NULL)
        {
            pthread_cond_wait(&qready, &qlock);
        }
        mp = workq;
        workq = mp->m_next;
        pthread_mutex_unlock(&qlock);
        /* process msg*/
    }
    
}

//向消息链表的头部加一个消息，并向等待条件的进程进行广播
void equeue_msg(msg *mp)
{
    pthread_mutex_lock(&qlock);
    mp->m_next = workq;
    workq = mp;
    pthread_mutex_unlock(&qlock);
    pthread_cond_signal(&qready);
}

```

### 自旋锁

好像实用性不大，书中说一般用不到，偷个懒。

### 屏障

屏障是用户协调多个线程并行工作的同步机制。屏障允许每个线程等待，直到所有的合作线程到达某一点，然后从改点继续执行。`pthread_join`就是一种屏障，允许一个线程等待，直到另一个线程退出。

`pthread_barrier_init`是屏障类，下面的函数可以进行初始化和反初始化：

```c
#include<pthread.h>
int pthread_barrier_init(pthread_barrier_t *restrict barrier, const pthread_barrierattr_t *restrict attr, unsigned int count);
int pthread_barrier_destory(pthread_barrier_t *barrier);
//成功返回0，否则返回错误编号

```

初始化屏障时，使用`count`参数指定，在允许所以线程运行之前，必须到达屏障的线程数目。

使用函数`pthread_barrier_wait`函数来表面调用线程已完成任务，等待其他线程赶来：

```c
#include<pthread.h>
int pthread_barrier_wait(pthread_barrier_t *barrier);
//成功返回0或者PTHREAD_BARRIER_SERIAL_THREAD，否则返回错误编号。

```

调用`pthread_barrier_wait`的线程在屏障计数未满足条件时，会进入休眠状态。如果该线程是最后一个调用`pthread_barrier_wait`的线程，就满足了屏障计数，所以线程就会被唤醒。

对于任意一个线程，`pthread_barrier_wait`返回了`PTHREAD_BARRIER_SERIAL_THREAD`。剩下的进程看到的返回值是0，。这使得一个线程可以作为主线程，它可以工作在其他所有线程已完成的结果上。

一旦达到屏障计数，而且线程处于非阻塞状态，屏障就可以被重用，但除非在反初始化之后又重新进行了初始化，否则屏障计数不变。

例程：八个线程对一个数组进行堆排序，将数组拆成八份，最后利用归并的方法进行合并：

```c
#include<pthread.h>
#include"apue.h"
#include<limits.h>
#include<sys/time.h>
#undef max
#undef min
#include<algorithm>
using namespace std;

#define NTHR 8 //线程数
#define NUMNUM 8000000L //数组大小
#define TNUM (NUMNUM/NTHR) //每个线程排序数量

long nums[NUMNUM];

long snums[NUMNUM];

#ifdef SOLARIS
#define heapsort qsort
#else
extern int heapsort(void *, size_t, size_t, int (*)(const void *, const void *));
#endif

pthread_barrier_t b;

bool complong(const long &arg1, const long &arg2)
{
    if(arg1<arg2)
    {
        return 1;
    }
    return 0;
}

void *thr_fn(void *arg)
{
    long idx = (long)arg;
    sort(nums+idx, nums+idx+TNUM,complong);
    
    pthread_barrier_wait(&b);
    return ((void*)0);
}

void merge()
{
    long idx[NTHR];
    long i, minidx, sidx, num;
    for(int i=0;i<NTHR;i++)
    {
        idx[i] = i*TNUM;
    }
    for(sidx = 0;sidx<NUMNUM; sidx++)
    {
        num = LONG_MAX;
        for(i = 0; i<NTHR; i++)
        {
            if(idx[i]<(i+1)*TNUM && (nums[idx[i]]<num))
            {
                num = nums[idx[i]];
                minidx = i;
            }
        }
        snums[sidx] = nums[idx[minidx]];
        idx[minidx]++;
    }
}

int main()
{
    unsigned long i;
    timeval start, end;
    long long startsec, endsec;
    double elapsed;
    int err;
    pthread_t tid;

    srandom(1);
    for(i =0 ;i<NUMNUM; i++)
    {
        nums[i] = random();
    }
    gettimeofday(&start, NULL);
    pthread_barrier_init(&b, NULL, NTHR+1);
    
    for(i = 0;i<NTHR;i++)
    {
        err = pthread_create(&tid, NULL, thr_fn, (void*)(i*TNUM));
        if(err!=0)
        {
            err_exit(err, "can't create thread");
        }
    }
    pthread_barrier_wait(&b);
    merge();
    gettimeofday(&end, NULL);
    startsec = start.tv_sec * 1000000 + start.tv_usec;
    endsec = end.tv_sec * 1000000 + end.tv_usec;
    elapsed = (double)(endsec-startsec)/1000000.0;
    printf("sort1 took %.4f seconds\n", elapsed);

    srandom(1);
    for(i =0 ;i<NUMNUM; i++)
    {
        nums[i] = random();
    }
    gettimeofday(&start, NULL);
    sort(nums, nums+NUMNUM, complong);
    gettimeofday(&end, NULL);
    startsec = start.tv_sec * 1000000 + start.tv_usec;
    endsec = end.tv_sec * 1000000 + end.tv_usec;
    elapsed = (double)(endsec-startsec)/1000000.0;
    printf("sort2 took %.4f seconds\n", elapsed);
    exit(0);
}

```

执行结果：

```
$ ./pthread_barrier.o 
sort1 took 0.7863 seconds
sort2 took 2.4713 seconds

```



# 十二章 线程控制

## 线程限制

线程相关限制有下面所述：，这些参数都可以使用`sysconf`函数获得

| 限制名称                        | 描述                                                   | name参数                           |
| ------------------------------- | ------------------------------------------------------ | ---------------------------------- |
| `PTHREAD_DESTRUCTOR_ITERATIONS` | 线程退出时操作系统实现试图销毁线程特定数据的最大次数。 | `_SC_THREAD_DESTRUCTOR_ITERATIONS` |
| `PTHREAD_KEYS_MAX`              | 进程可以创建的键的最大数目。                           | `_SC_THREAD_KEYS_MAX`              |
| `PTHREAD_STACT_MIN`             | 一个线程栈可用的最小字节数。                           | `_SC_THREAD_START_MIN`             |
| `PTHREAD_THREADS_MAX`           | 进程可以创建的最大线程数。                             | `_SC_THREAD_THREADS_MAX`           |



## 线程属性

`pthread`接口允许我们通过设置每个对象关联的不同属性来细调线程和同步对象的行为，管理这些属性的函数都遵循相同的模式：

（1）每个对象与它自己类型的属性对象进行关联（线程与线程属性关联，互斥量与互斥量属性关联等等）。一个属性对象可以代表多种属性。属性对象对应程序来说是不透明的。需要提供相应的函数来管理属性。

（2）有一个初始化函数，把属性设置为默认值。

（3）还有一个销毁对象的函数，如果初始化函数分配了与属性相关的资源，销毁函数负责释放这些资源。

（4）每一个属性都有一个从属性对象中获取属性值的函数。

（5）每一个函数都有一个设置属性值的函数，在这种情况下，属性值作为参数按值传递。

在`pthread_create`函数中，我们可以使用`phread_attr_t`对线程属性进行设置，下面两个函数负责默认初始化和反初始化`pthread_attr_t`变量：

```c
#include<pthread.h>
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destory(pthread_attr_t *attr);
//成功返回0，否则返回错误编号

```

`pthread_attr_destory`会销毁属性对象的动态分配的空间（如果是的话），同时还会用无效的值初始化属性对象。

线程属性类型如下：

| 名称          | 描述                                 |
| ------------- | ------------------------------------ |
| `detachstate` | 线程分离状态属性。                   |
| `guardsize`   | 线程栈末尾的警戒缓冲区大小（字节）。 |
| `stackaddr`   | 线程栈的最低地址。                   |
| `stacksize`   | 线程栈的最小长度（字节）。           |

分离线程在上一章已经介绍过了，如果对现有的某个线程的终止状态不感兴趣，可以使用`pthread_detach`函数让操作系统在线程退出时就收回它所占用的资源。

可以修改`pthread_attr_t`中`detachstate`属性决定线程分离状态。`detachstate`有两个合法值，`PTHREAD_CREATE_DETACHED`，以分离状态创建进程，或`PTREAD_CREATE_JOINABLE`，正常启动线程，应用程序可以获取线程的终止状态(这两个均是int类型指针)。下面两个函数分别用来获取和设置`pthread_attr_t`的相应属性：

```c
#include<pthread.h>
int pthread_attr_getdetachstate(const pthread_attr_t *restrict attr, int *detachstate);
int pthread_attr_setdetachstate(pthread_attr_t *attr, int *detachstate);
//成功返回0，否则返回错误编号

```

使用下面两个函数对线程栈属性进行更改：

```c
#include<pthread.h>
int pthread_attr_getstack(const pthread_attr_t *restrict attr, void **restrict stackaddr, size_t *restrict stacksize);
int pthread_attr_setstack(pthread_attr_t *attr, void *stackaddr, size_t stacksize);
//成功返回0，否则返回错误编号

```

对于进程来说，虚地址空间的大小是固定的。进程只有一个栈，其大小不是问题。但对于线程来说，相同大小的虚地址空间必须被所以线程栈共享。如果应用程序使用了许多线程，以至于这些线程栈的累计大小超过了可用的虚地址空间，就需要减少默认线程栈大小。另一方面，如果线程调用的函数分配了大量自动变量，或者调用的函数涉及许多很深的栈帧，那么需要的栈大小可能比默认的大。

如果线程栈的虚地址空间用完了，可以使用`malloc`或者`mmap`（十四章）来为可替代的栈分配空间，并用`pthread_attr_setstack`函数来改变新建线程的栈位置。由`stackaddr`参数指定的地址可以用作线程栈的内存范围的最低可寻址地址，改地址与处理器结构相应的边界对齐。这里要假设`malloc`与`mmap`所用的虚地址范围与线程栈当前使用的虚地址范围不同。

`stackaddr`被定义为栈的最低内存地址，但并不一定是栈的开始位置。对于一个给定的处理器结构来说，如果栈是从高地址向低地址方向增长，那么`stackaddr`线程属性将是3栈的结尾位置。

应用程序也可以通过下面两个函数获取和设置`stacksize`：

```c
#include<pthread.h>
int pthread_attr_getstacksize(const pthread_attr_t *restrict attr, size_t *restrict stacksize);
int pathread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
//成功返回0，否则返回错误编号

```

希望改变默认的栈的大小，又不想自己处理线程栈的分配问题，使用`pathread_attr_setstacksize`函数十分有用。设置`stacksize`不能小于`PTHREAD_STACK_MIN`。

线程属性`guardsize`控制线程栈末尾之后用以避免栈溢出的扩展内存大小，默认取决于系统实现，通常是系统页大小。可以把`guardsize`线程属性设置为0，不允许属性的这种行为发生：在这种情况下，不不提供警戒缓冲区。如果修改了线程属性`stackaddr`，系统就认为我们自己管理栈，进而使栈警戒缓冲机制失效，这等同于把`guardsize`设置为0。

下面的函数可以获取和设置`guardsize`属性：

```c
#include<pthread.h>
int pthread_attr_getguardsize(const pthread_attr_t *restrict attr, size_t *restrict guardsize);
int ptrhead_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);
//成功返回0，否则返回错误编号

```

如果`guardsize`线程属性被修改，操作系统可能会把它取为页大小的整数倍。如果线程的栈指针溢出到警戒区，应用程序就可能通过信号接收到出错信息。

## 同步属性

### 互斥量属性

对于互斥量非默认属性，可以使用下面函数进行初始化和反初始化：

```c
#include<pthread.h>
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
int pthread_mutexattr_destory(pthread_mutexattr_t *attr);
//成功返回0，否则返回错误编号

```

`pthread_mutexattr_init`将用默认的互斥量属性初始化`pthread_mutexattr_t`结构。值得注意的三个属性是：进程共享属性、健壮属性和类型属性。

在进程中，多个进程可以访问同一个同步对象。这是默认行为，在这种情况下，进程共享互斥量属性需设置为`PTHREAD_PROCESS_PRIVATE`。

在下面的章节，我们将看到存在这样的机制：允许相互独立的多个进程把同一个内存数据块映射到它们各自独立的地址空间中。和多个线程访问共享数据一样，多个进程访问共享数据通常也需要同步。如果进程共享互斥量属性设置为`PTHREAD_PROCESS_SHARED`，从多个进程彼此之间共享的内存数据块中分配的互斥量就可以用于这些进程的同步。

可以使用下面的函数来获取和设置进程共享属性：

```c
#include<pthread.h>
int pthread_mutexattr_getpshared(const pthread_mutexattr_t *restrict attr, int *restrict pshared);
int pthread_mutexattr_setpshared(const pthread_mutexattr_t *restrict attr, int pshared);
//成功返回0，否则返回错误编号

```

互斥量的健壮属性与在多个进程共享的互斥量有关。这意味着，当持有互斥量的进程终止时，需要解决互斥量状态恢复的问题。在这种情况下，互斥量处于锁定状态，恢复起来很困难。其他阻塞在这个锁的进程将会一直阻塞下去。

可以使用下面的函数获取和设置互斥量的健壮属性：

```c
#include<pthread.h>
int pthread_mutexattr_getrobust(const pthread_mutexattr_t *restrict attr, int *restrict robust);
int pthread_mutexattr_setrobust(const pthread_mutexattr_t *restrict attr, int robust);
//成功返回0，否则返回错误编号

```

健壮属性取值有两种情况，默认值是`PTHREAD_MUTEX_STALLED`，这意味着持有互斥量的进程终止时不采取特别的动作。另一个取值是`PTHREAD_MUTEX_ROBUST`，这个值将导致线程调用`pthread_mutex_lock`获取锁，而该锁被另一个进程持有，但终止时并未对该锁进行解锁，此时线程会阻塞，从`pthread_mutex_lock`返回值为`EOWNERDEAD`而不是0。应用程序可以通过这个特殊值获知，若有可能，不管它们保护的互斥量状态如何，都需要进行恢复。

使用健壮性改变了使用`pthread_mutex_lock`的方式，因为必须要检查三个值：不需要恢复的成功，需要恢复的成功以及失败。

如果应用状态无法恢复，在线程对互斥量解锁以后，该互斥量将处于永久不可用状态，为了避免这样的问题，线程可以调用`pthread_mutex_consistent`函数，指明与该互斥量相关的状态在互斥量解锁之前是一致的。

```c
#include<pthread.h>
int pthread_mutex_consistent(pthread_mutex_t *mutex);
//成功返回0，否则返回错误编号

```

如果线程没有先调用`pthread_mutex_consistent`就对互斥量解锁，那么其他试图获取该互斥量的阻塞线程将会得到错误码`ENOTRECOERABLE`。如果发生这种情况，互斥量将不在可用。线程通过提前调用`pthread_mutex_consistent`，就能让互斥量正常工作，这样就可以持续被使用。

类型互斥量属性控制着互斥量的锁定特性：

| 互斥量类型                 | 特性                                                         | 没有解锁时重新加锁 | 不占用时的解锁 | 在已解锁时解锁 |
| -------------------------- | ------------------------------------------------------------ | ------------------ | -------------- | -------------- |
| `PTHREAD_MUTEX_NORMAL`     | 标准互斥量，不做错误检测和死锁检测                           | 死锁               | 未定义         | 未定义         |
| `PTHREAD_MUTEX_ERRORCHECK` | 提供错误检查                                                 | 返回错误           | 返回错误       | 返回错误       |
| `PTHREAD_MUTEX_RECURSIVE`  | 运行同一个线程在互斥量解锁之前对该互斥量多次加锁。递归互斥量维护锁的计数。（加几次就一个解锁几次）。 | 允许               | 返回错误       | 返回错误       |
| `PTHREAD_MUTEX_DEFAULT`    | 可以提供默认特性和行为。操作系统实现时把该类型自由映射到其他互斥量类型中的一种。 | 未定义             | 未定义         | 未定义         |

“不占用时加锁”是指，一个线程对另一个线程加锁的互斥量解锁，“已解锁时解锁”是指，一个线程对已经解锁的互斥量进行解锁。

使用下面的函数可以获取和设置互斥量类型属性：

```c
#include<pthread.h>
int pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict attr, int *restrict type);
int pthread_mutexattr_settype(const pthread_mutexattr_t *restrict attr, int *restrict type);
//成功返回0，否则返回错误编号

```

例程：使用递归互斥量的情况，超时函数，允许安排另一个函数在未来某个时间运行，线程资源如果不是很昂贵，就可以为每一个挂起的超时函数创建一个线程，线程在未到时间时一直等待，时间到了再调用请求函数。

```c
#include<pthread.h>
#include"apue.h"
#include<time.h>
#include<sys/time.h>

extern int makethread(void *(*)(void*), void *);

struct to_info{
    void (*to_fn)(void *); //function
    void *to_arg;
    timespec to_wait;
};

#define SECTONSEC 1000000000 /* second to nanoseconds*/

#if !defined(CLOCK_REALTIME) || defined(BSD)
#define CLOCK_nanosleep(ID, FL, REQ, REM) nanosleep((REQ), (REM))
#endif

#ifndef CLOCK_REALTIME
#define CLOCK_REALTIME 0
#define USECTONSEC 1000 /* microseconds to nanosecond*/
void clock_gettime(int id, timespec *tsp)
{
    timeval tv;
    gettimeofday(&tv, NULL);
    tsp->tv_sec = tv.tv_sec;
    tsp->tv_nsec = tv.tv_usec*USECTONSEC
}
#endif

void *timeout_helper(void *arg)
{
    to_info *tip = (struct to_info *)arg;
    clock_nanosleep(CLOCK_REALTIME, 0, &tip->to_wait, NULL);
    (*tip->to_fn)(tip->to_arg);
    free(arg);
    return 0;
}

void timeout(const timespec *when, void (*func)(void *), void *arg)
{
    timespec now;
    to_info *tip;
    int err;
    clock_gettime(CLOCK_REALTIME, &now);
    if(when->tv_sec > now.tv_sec || (when->tv_sec == now.tv_sec && when->tv_nsec > now.tv_nsec))
    {
        tip = (to_info*)malloc(sizeof(to_info));
        if(tip != NULL)
        {
            tip->to_fn = func;
            tip->to_arg = arg;
            tip->to_wait.tv_sec = when->tv_sec - now.tv_sec;
            if(when->tv_nsec >= now.tv_nsec)
            {
                tip->to_wait.tv_nsec = when->tv_nsec - now.tv_nsec;
            }
            else{
                tip->to_wait.tv_nsec = when->tv_nsec - now.tv_nsec + SECTONSEC;
                tip->to_wait.tv_sec--;
            }
            err = makethread(timeout_helper, (void*)tip);
            if(err == 0)
            {
                return;
            }
            else{
                free(tip);
            }
        }
    }
    //如果when<=now，或者malloc失败，或创建线程失败，我们应该直接调用函数
    (*func)(arg);
}

pthread_mutexattr_t attr;
pthread_mutex_t mutex;

void retry(void *arg)
{
    pthread_mutex_lock(&mutex);
    /* preform retry steps ...*/

    pthread_mutex_unlock(&mutex);
}

int main(void)
{
    int err, condition, arg;
    timespec when;

    if((err=pthread_mutexattr_init(&attr))!=0)
    {
        err_exit(err, "pthread_mutexattr_init error");
    }
    if((err=pthread_mutexattr_settype(%attr, PTHREAD_MUTEX_RECURSIVE))!=0)
    {
        err_exit(err, "can't set recursive type");
    }
    if((err = pthread_mutex_init(&mutex, &attr))!=0)
    {
        err_exit(err, "can't create recursive mutex");
    }

    /*continue process ...*/

    pthread_mutex_lock(&mutex);
    if(condition)
    {
        clock_gettime(CLOCK_REALTIME, &when);
        when.tv_sec += 10;
        timeout(&when, retry, (void*)((unsigned long)arg));
    }

    pthread_mutex_unlock(&mutex);
    exit(0);

}

```

`makethread`函数以分类状态创建线程。由于传递给`timeout`函数的`func`函数参数将在未来运行，因此我们不希望一直空等待线程结束。

`timeout`的调用者需要占有互斥锁来检查条件，并且把`retry`函数安排为原子操作。`retry`函数试图对同一个互斥量进行加锁，如果互斥量不是递归的，会导致死锁。

### 读写锁属性

下面的函数用来对读写锁默认初始化和反初始化：

```c
#include<pthread.h>
int pthread_rwlockattr_init(pthread_relockattr_t *attr);
int pthread_rwlockattr_destory(pthread_relockattr_t *attr);
//成功返回0，否则返回错误编号

```

读写锁唯一支持的属性是进程共享属性，其与互斥量的进程共享属性一致。下面的函数用来获取和设置读写锁的进程属性：

```c
#include<pthread.h>
int pthread_rwlockattr_getpshared(const pthread_rwlockattr_t *restrict attr, int *restrict pshared);
int pthread_rwlockattr_setpshared(const pthread_rwlockattr_t *attr, int pshared);
//成功返回0，否则返回错误编号

```

### 条件变量属性

条件变量存在两个属性：进程共享属性和时钟属性。

下面的函数用来默认初始化和反初始化条件变量：

```c
#include<pthread.h>
int pthread_condattr_init(pthread_condattr_t *attr);
int pthread_condattr_destory(pthread_condattr_t *attr);
//成功返回0，否则返回错误编号

```

条件变量的进程属性控制条件变量是被单进程的多线程使用还是多进程的线程使用。下面的函数获取和设置进程共享属性：

```c
#include<pthread.h>
int pthread_condattr_getpshared(const pthread_condattr_t *restrict attr, int *restrict pshared);
int pthread_condattr_setpshared(const pthread_condattr_t *attr, int pshared);
//成功返回0，否则返回错误编号

```

时钟属性控制计算`pthread_cond_timedwait`函数的超时参数（`tsptr`）采用的哪个时钟。合法值为第六章时间和日期例程中第一个表中的值。下面的函数用来获取和设置时钟属性：

```c
#include<pthread.h>
int pthread_condattr_getclock(const pthread_condattr_t *restrict attr, clockid_t *restrict clock_id);
int pthread_condattr_setclock(const pthread_condattr_t *attr, clockid_t clock_id);
//成功返回0，否则返回错误编号

```



### 屏障属性

下面的函数用来对屏障属性对象初始化和反初始化：

```c
#include<pthread.h>
int pthread_barrierattr_init(pthread_barrierattr_t *attr);
int pthread_barrierattr_destory(pthread_barrierattr_t *attr);
//成功返回0，否则返回错误编号

```

屏障属性只有进程共享，与互斥量类似，下面的函数用来获取和设置进程共享属性：

```c
#include<pthread.h>
int pthread_barrierattr_getpshared(const pthread_barrierattr_t *restrict attr, int *restrict pshared);
int pthread_barrierattr_setpshared(const pthread_barrierattr_t *attr, int *pshared);
//成功返回0，否则返回错误编号

```



## 重入

如果一个函数在相同的时间点可以被多个线程安全的调用，就称之为线程安全的。在标准的定义的函数除了下图列出来的函数，其他都保证是线程安全的。

![线程不安全函数](https://s2.ax1x.com/2019/10/27/KsB7ZR.png)

对POSIX.1中的一些非线程安全函数，它会提供可替代的线程安全版本。下图列出了这些替代版本：

![线程安全版本](https://s2.ax1x.com/2019/10/27/KsBHd1.png)

如果一个函数对于多个线程来说是可重入的，就说这个函数是线程安全的。但并不能说明对信号处理程序来说该函数是可重入的。如果函数对异步信号处理程序的重入是安全的，那么就说函数是异步信号安全的。

除了上图，POSIX.1提供了以线程安全的方式管理FILE对象的方法。可以使用`flockfile`和`ftrylockfile`获取给定FILE对象关联的锁。这个锁是递归的。虽然这种锁的具体实现无规定，但要求所以操作FILE对象的标准I/O例程动作行为必须看起来就像他们内部调用了`flockfile`和`funlockfile`：

```c
#include<stdio.h>
int ftrylockfile(FILE *fp);
//成功返回0，如果不能获取锁，返回非0数组

void flockfile(FILE *fp);
void funlockfile(FILE *fp);

```

如果标准I/O例程都获取各自的锁，那么每次做一次一个字符的I/O时就会出现严重的性能下降。为了避免这种开销，出现了不加锁版本的基于字符的标准I/O例程：

```c
#include<stdio.h>
int getchar_unlock(void);
int getc_unlocked(FILE *fp);
//成功，返回下一个字符，遇到文件结尾或出错，返回EOF

int putchar_unlocked(int c);
int putc_unlocked(int c, FILE *fp);
//成功返回c，出错返回EOF

```

除非被`flockfile`和`funlockfile`包围，否则尽量不要调用上面四个函数，因为它们会导致不可预期的结果。

第七节显示了一个`getevn`的可能实现，不过这个版本是不可重入的。如果两个线程同时调用这个函数，就会看到不一样的结果，因为所以`getenv`的线程返回的字符串都存储在同一静态缓冲区中：

```c
#include<limits.h>
#include<string.h>

#define MAXSTRINGSZ 4096

static char envbuf[MAXSTRINGSZ];

extern char **environ;

char *getenv(const char *name)
{
    int i, len;
    len = strlen(name);
    for(i=0;environ[i]!=NULL;i++)
    {
        if((strncmp(name, environ[i],len)) && (environ[i][len] == '='))
        {
            strncpy(envbuf, &environ[i][len+1], MAXSTRINGSZ-1);
            return envbuf;
        }
    }
    return NULL;
}

```

下面给出了`getenv`的可重入版本。使用了`pthread_once`函数来确保不管多少线程同时竞争`getenv_r`，每个进程只调用`thread_init`函数一次，下一节会详细介绍`pthread_once`

```c
#include<string.h>
#include<errno.h>
#include<pthread.h>
#include<stdlib.h>

extern char **environ;

pthread_mutex_t env_mutex;

static pthread_once_t init_done = PTHREAD_ONCE_INIT;

static void thread_init(void)
{
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    pthread_mutex_init(&env_mutex, &attr);
    pthread_mutexattr_destroy(&attr);
}

int getenv_t(const char *name, char *buf, int buflen)
{
    int i, len, olen;
    pthread_once(&init_done, thread_init);
    len = strlen(name);
    pthread_mutex_lock(&env_mutex);
    for(i=0;environ[i] != NULL;i++)
    {
        if((strncmp(name, environ[i],len)) && (environ[i][len] == '='))
        {
            olen = strlen(&environ[i][len+1]);
            if(olen >= buflen)
            {
                pthread_mutex_unlock(&env_mutex);
                return ENOSPC;
            }
            else{
                strcpy(buf, &environ[i][len+1]);
                pthread_mutex_unlock(&env_mutex);
                return 0;
            }
        }
    }
    pthread_mutex_unlock(&env_mutex);
    return ENOSPC;
}

```

这里改变了原来`getenv`的接口，调用者必须提供自己的缓冲区，这样每个线程可以使用不同的缓冲区避免互相干扰。



## 线程特定数据

线程特定数据，也称为线程私有数据，是存储和查询某个特定线程相关数据的一种机制。对于这种数据，我们希望每个线程可以访问它自己的数据副本而不需要担心与其他线程同步访问问题。

线程需要特定数据的原因有两个：

1. 线程ID不能保证是小而连续的整数，所以不能简单分配一个每组线程数组，用线程ID作为数组的索引。即使线程ID是小而连续的整数，我们可能希望有一些额外的保护，防止某个线程的数据与其他线程的数据相混乱。
2. 线程特定数据提供了让基于进程的接口适应多线程环境的机制。一个典型的例子就是`erron`。

在分配线程特定数据之前，需要创建与该数据关联的键。这个键将用于获取对线程特定数据的访问。使用`pthread_key_create`创建一个键：

```c
#include<pthread.h>
int pthread_key_create(pthread_key_t *keyp, void (*destructor)(void*));
//成功返回0，否则返回错误编号

```

创建的键存储在`keyp`指向的内存单元中，这个键可以被进程中的所有线程使用，但每个线程把这个键与不同的线程特定数据地址关联。创建新键时，每个线程的数据地址设为空地址。

除了创建键以为，`pthread_key_create`可以关联一个析构函数。当线程退出时，如果数据地址已经被置为非空值，那么析构函数将会被调用。当线程调用了`pthread_exit`或者线程执行返回，正常退出时，析构函数就会被调用。线程取消时，只有在最后清理处理程序返回之后，析构函数才会被调用。如果线程调用了`exit`、`_Exit`或`abort`，或出现其他非正常的退出时，就不会调用析构函数。

线程通常使用`malloc`为线程特定数据分配内存。

对于所以的线程，我们通常可以调用`pthread_key_delete`来取消键与特定数据值之间的关联关系：

```c
#include<pthread.h>
int pthread_key_delete(pthread_key_t key);
//成功返回0，否则返回错误编号

```

调用`pthread_key_delete`并不会激活与键关联的析构函数。要释放响应空间应该在应用程序中采取额外步骤。

对于同一个线程特定数据，`pthread_key_create`应该在一个进程中只执行一次，如果将`pthread_key_create`放在每个线程内执行，会导致不同线程看到的是不同的键值。解决这种竞争的办法是使用`pthread_once`：

```c
#include<pthread.h>
pthread_once_t initflag = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *initflag, void (*initfn)(void));
//成功返回0，否则返回错误编号

```

`initflag`必须是非本地变量（如全局变量或静态变量），而且必须初始化为`PTHREAD_ONCE_INIT`。

如果每个线程都调用`pthread_once`，系统就能保证初始化例程`initfn`只被调用一次，即系统首次调用` pthread_once`时。

键一旦被创建后，就可以通过调用`pthread_setspecific`函数把键和线程特定数据关联起来。可以通过`pthread_getspecific`函数获取线程特定数据的地址：

```c
#include<pthread.h>
void *pthread_getspecific(pthread_key_t key);
//返回线程特定数据，如果没有值与该键相关联，返回NULL

int pthread_setspecific(pthread_key_t key, const void *value);
//返回值，成功返回0，否则返回错误编号

```

例程：getenv函数的另一个版本，之前我们改变了函数接口，这里不该函数接口

```c
#include<pthread.h>
#include<limits.h>
#include<stdlib.h>
#include<string.h>

#define MAXSTRINGSZ 4096

static pthread_key_t key;
static pthread_once_t init_done = PTHREAD_ONCE_INIT;
pthread_mutex_t env_mutex = PTHREAD_MUTEX_INITIALIZER;

extern char **environ;

static void thread_init(void)
{
    pthread_key_create(&key, free);
}

char *getenv(const char *name)
{
    int i, len;
    char *envbuf;
    pthread_once(&init_done, thread_init);
    pthread_mutex_lock(&env_mutex);
    envbuf = (char*)pthread_getspecific(key);
    if(envbuf == NULL)
    {
        envbuf = (char*)malloc(MAXSTRINGSZ);
        if(envbuf == NULL)
        {
            pthread_mutex_unlock(&env_mutex);
            return NULL;
        }
    }
    len = strlen(name);
    for(i=0;environ[i] != NULL;i++)
    {
        if((strncmp(name, environ[i],len)) && (environ[i][len] == '='))
        {
            strncpy(envbuf, &environ[i][len+1], MAXSTRINGSZ-1);
            pthread_mutex_unlock(&env_mutex);
            return envbuf;
        }
    }
    pthread_mutex_unlock(&env_mutex);
    return NULL;
}

```



## 取消选项

有两个线程属性并没有包含在`pthread_attr_t`结构中，它们是可取消状态和可取消类型。这两个属性影响着线程在响应`pthread_cansel`函数调用时所呈现的行为。

可取消状态属性可以是`PTHREAD_CANCEL_ENABLE`，也可以是`PTHREAD_CANCEL_DISABLE`。线程可以通过下面的函数修改可取消状态：

```c
#include<pthread.h>
int pthread_setcancelstate(int state, int *oldstate);
//成功返回0，否则返回错误编号

```

`pthread_cancel`调用并不等待进程终止。在默认情况下，线程在取消请求发出后还是继续运行，直到运行到某个取消点。取消点是线程检查它是否被取消的一个位置，如果取消了，按请求行事。下列函数为执行完成后会出现取消点的函数：

![取消点](https://s2.ax1x.com/2019/10/27/KyV1pD.png)

线程默认的可取消状态为`PTHREAD_CANCEL_ENABLE`。当状态为`PTHREAD_CANCEL_DISABLE`时，对`pthread_cancel`调用并不会杀死进程。相反，取消请求对于这个线程来说还处于挂起状态，当取消状态再次变为`PTHREAD_CANCEL_ENABLE`时，线程将在下一个取消点上对所以挂起的取消请求进行处理。

可以调用`pthread_testcancel`函数在程序中添加自己的取消点：

```
#include<pthread.h>
void pthread_testcancel(void);

```

调用`pthread_testcancel`时，如果有某个取消请求正处于挂起状态，而且取消并没有置为无效，那么线程会立即取消。

我们所描述的默认的取消类型也称为推迟取消。调用`pthread_cancel`以后，在线程达到取消点以前，并不会真正的取消。可以通过调用下面的函数来修改取消类型：

```c
#include<pthread.h>
int pthread_setcanceltype(int type, int *oldtype);
//成功返回0，否则返回错误编号

```

`type`可以是`PTHREAD_CANCEL_DEFERRED`(延迟取消)或`PTHREAD_CANCEL_ASYNCHRONOUS`(异步取消)。

异步取消可以在任意时间取消线程。

## 线程和信号

每个线程都有自己的信号屏蔽字，但是信号的处理是进程中所以进程共享的。进程中的信号是递送到单个进程的。如果一个信号与硬件故障相关，那么该信号一般会被发送到引起该事件的线程中去，而其他信号则被发送到任意一个线程。

第十章讨论了进程使用`sigpromask`函数来阻止信号发送。然而`sigpromask`的行为并未在多线程中定义，线程必须使用`pthread_sigmask`：

```c
#include<signal.h>
int pthread_sigmask(int how, const sigset_t *restrict set, sigset_t *restrict oset);
//成功返回0，否则返回错误编号

```

`pthread_sigmask`除了返回值，其它与`sigpromask`基本相同。

线程可以调用`sigwait`等待一个或多个信号的出现：

```c
#include<signal.h>
int sigwait(const sigset_t *restrict set, int *restrict signop);
//成功返回0，否则返回错误编号

```

`set`参数指定了线程等待的信号集。返回时，`signop`指向的整数将包含发生的信号。

如果信号集中某个信号在`sigwait`调用的时候处于挂起状态，那么`sigwait`将无阻塞的返回。在返回之前，`sigwait`将从进程中移除那些处于挂起等待的信号。

为了避免错误发生，线程在调用`sigwait`之前，必须阻塞那些正在等待的信号。`sigwait`会原子地取消信号集的阻塞状态，直到有新的信号被递送。在返回之前，`sigwait`将恢复线程的信号屏蔽字。如果信号在`sigwait`被调用的时候没有被阻塞，那么在线程完成对`sigwait`的调用之前会出现一个时间窗，在这个时间窗中，信号就可以被发送给线程。

使用`sigwait`的好处是可以简化信号处理，允许把异步产生的信号用同步的方式处理。为了防止信号中断线程，可以把信号加到每一个线程的信号屏蔽字中。然后安排专用线程处理信号。这些专用线程可以进行函数调用，不需要担心在信号处理程序中调用哪些函数是安全的，因为这些函数调用来自正常的线程上下文，而非会中断线程正常执行的传统信号处理程序。

如果多个线程在`sigwait`的调用中因等待同一信号而阻塞，那么在信号递送的时候，就只有一个线程可以从`sigwait`中返回。具体由操作系统来决定如何递送信号。要把信号发送给线程，可以使用下面的函数：

```c
#include<signal.h>
int pthread_kill(pthread_t thread, int signo);
//成功返回0，否则返回错误编号

```

闹钟定时器是进程资源，并且所以的线程共享相同的闹钟，所以进程的多个线程不可能互不干扰的使用闹钟定时器。

例程：实现第十章函数`sigsuspend`的捕捉中断信号和退出信号，但只有当是退出信号时时才唤醒进程，使用单独的线程处理信号：

```c
#include"apue.h"
#include<pthread.h>
int quitflag;
sigset_t mask;

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t waitloc = PTHREAD_COND_INITIALIZER;

void *thr_fn(void*arg)
{
    int err, signo;

    while(1)
    {
        err = sigwait(&mask, &signo);
        if(err != 0)
        {
            err_exit(err, "sigwait failed");
        }
        switch(signo)
        {
            case SIGINT:
            printf("\niterrupt\n");
            break;

            case SIGQUIT:
            pthread_mutex_lock(&lock);
            quitflag = 1;
            pthread_mutex_unlock(&lock);
            pthread_cond_signal(&waitloc);
            return 0;

            default:
            printf("unexcepted signal %d\n", signo);
            exit(1);

        }
    }
}

int main(void)
{
    int err;
    sigset_t oldmask;
    pthread_t tid;

    sigemptyset(&mask);
    sigaddset(&mask, SIGINT);
    sigaddset(&mask, SIGQUIT);

    //主线程屏蔽中断信号和退出信号，新键线程会继承该信号屏蔽。
    if((err = pthread_sigmask(SIG_BLOCK, &mask, &oldmask))!=0)
    {
        err_exit(err, "sig_block errpr");
    }

    err = pthread_create(&tid, NULL, thr_fn, 0);
    if(err!=0)
    {
        err_exit(err, "can't create thread");
    }

    //这里的lock只是起保护作用。
    pthread_mutex_lock(&lock);
    while(quitflag == 0)
    {
        pthread_cond_wait(&waitloc, &lock);
    }
    pthread_mutex_unlock(&lock);

    quitflag = 0;

    //恢复信号屏蔽字
    if(sigprocmask(SIG_SETMASK, &oldmask, NULL)<0)
    {
        err_exit(err, "sig_setmask error");
    }
    exit(0);
}

```



## 线程和fork

当线程调用`fork`时，为子进程创建整个进程地址空间的副本。在第八章讲过写时复制策略，子进程与父进程是完全不同的进程，只要二者都没有对内存做出修改，父进程和子进程共享内存页的副本。

子进程通过继承整个地址空间的副本，还从父进程那里继承了每个互斥量、读写锁和条件变量的状态。如果父进程包含一个以上线程，子进程在fork之后如果不是立即调用`exec`的话，需要立即清理锁状态。

子进程内部，只存在一个线程，即父进程中调用frok的线程的副本构成的。如果父进程中的线程占用锁，子进程将同样占用这些锁。但子进程并不包含占有锁的线程的副本，所以子进程没有办法知道它占有的哪些锁、需要释放哪些锁。这样就导致子进程无法再使用父进程中的锁了，但他们占用的资源却不会被释放，这是极大的浪费。

如果子进程从`fork`返回后马上调用其中一个`exec`函数，就可以避免这样的问题。此时，旧的地址空间将被丢弃，所以锁的状态无关紧要。这里考虑的问题主要就是子进程的问题，对于父进程来说是无所谓的，父进程设计的合理时，自己会解锁的，而对于子进程来说，其并没有父进程前面的处理，因此对于子进程来说，锁是一个完全未知的状态，要想子进程能够正常使用父进程的锁，就应该让生成的子进程获取所以的锁的锁，这样，锁状态对于子进程就不是未知的了。

要清除锁状态，可以通过调用`pthread_atfork`函数建立`fork`处理程序：

```c
#include<pthread.h>
int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));
//成功返回0，否则返回错误编号

```

`pthread_atfork`可以安装三个帮助清理锁的函数。`parpare`处理函数程序由父进程在`fork`创建子进程之前调用。该函数是获取父进程定义的所以锁。`parent`函数处理程序在`fork`创建子进程之后、在返回之前在父进程的上下文中调用的。这个处理程序用来释放`prepare`获取的锁。`child`处理程序在fork返回之前在子进程上下文中调用。与`parent`函数一样用来处理`prepare`获得的锁。执行过程为：

1. 父进程获取所有的锁。
2. 子进程获取所有的锁。
3. 父进程释放它的锁。
4. 子进程释放它的锁。

可以调用`pthread_atfork`参数从而设置多套`fork`处理函数。当某个函数床单为NULL时，表示不需要处理该部分。使用多个`fork`处理程序时，处理程序的调用顺序并不相同。`parent`和`child`处理程序是以他们注册时的顺序进行调用的，而`prepare`处理程序函数的调用顺序与注册的顺序相反。这样可以允许多个模块注册它们自己的`fork`处理程序，而且可以可保持锁的层次。

假设模块A调用模块B中的函数，而且每个模块有自己的一套锁。如果锁的层次是A在B之前，模块B必须在模块A之前设置它的`fork`处理程序。当父进程调用`fork`时：

1. 调用子模块A的`prepare`的函数。
2. 调用模块B的`prepare`函数。
3. 创建子进程。
4. 调用模块B的`child`函数。
5. 创建模块A的`child`函数。
6. `fork`函数返回到子进程。
7. 调用模块B的`parent`函数。
8. 调用模块A的`parent`函数。

虽然`pthread_atfork`机制的意图是是`fork`之后的锁状态保存一致，但它还是存在一些问题：

1. 没有很好的办法对复杂同步对象（条件变量和屏障）进行状态的重新初始化。
2. 某些错误检查的互斥量实现在`child`处理程序试图对被加锁的互斥量进行解锁时会发生错误。
3. 递归互斥量不能在`child`程序中被清理，由于没有办法知道被加锁次数。

例程：

```c
#include "apue.h"
#include <pthread.h>

pthread_mutex_t lock1 = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t lock2 = PTHREAD_MUTEX_INITIALIZER;

void prepare()
{
    int err;
    printf("prepare fn lock lock2\n");
    if ((err = pthread_mutex_lock(&lock2)) != 0)
    {
        err_exit(err, "prepare can't lock lock1");
    }
}

void parent()
{
    int err;
    printf("parent fn unlock lock2\n");
    if ((err = pthread_mutex_unlock(&lock2)) != 0)
    {
        err_exit(err, "parent can't unlock lock1");
    }
}

void child()
{
    int err;
    printf("child fn unlock lock2\n");
    if ((err = pthread_mutex_unlock(&lock2)) != 0)
    {
        err_exit(err, "child can't unlock lock2");
    }
    printf("child fn unlock lock1\n");
    if ((err = pthread_mutex_unlock(&lock1)) != 0)
    {
        err_exit(err, "child can't unlock lock1");
    }
}

void *child_thr_fn(void *arg)
{
    int err;
    printf("child 2 lock lock1\n");
    if ((err = pthread_mutex_lock(&lock1)) != 0)
    {
        err_exit(err, "child 2 can't lock lock1");
    }
    printf("child 2 lock lock2\n");
    if ((err = pthread_mutex_lock(&lock2)) != 0)
    {
        err_exit(err, "child 2 can't lock lock2");
    }
    printf("child process here\n");

    printf("child 2 lock unlock2\n");
    if ((err = pthread_mutex_unlock(&lock2)) != 0)
    {
        err_exit(err, "child 2 can't unlock lock2");
    }
    printf("child 2 lock unlock1\n");
    if ((err = pthread_mutex_unlock(&lock1)) != 0)
    {
        err_exit(err, "child 2 can't unlock lock1");
    }
    return (void *)0;
}

void *parent_thr_fn(void *arg)
{
    int err;
    printf("parent lock lock1\n");
    if ((err = pthread_mutex_lock(&lock1)) != 0)
    {
        err_exit(err, "prepare can't lock lock1");
    }
    if ((err = pthread_atfork(prepare, parent, child)) != 0)
    {
        err_exit(err, "atfork error");
    }
    pid_t pid = fork();
    // child
    if (pid == 0)
    {
        pthread_t tid;
        int err2 = pthread_create(&tid, NULL, child_thr_fn, NULL);
        if (err2 != 0)
        {
            err_exit(err2, "child can't create pthread");
        }
        sleep(1);//等待新线程结束
        int err3;
        printf("child 2 lock lock1\n");
        if ((err3 = pthread_mutex_lock(&lock1)) != 0)
        {
            err_exit(err, "child 1 can't lock lock1");
        }
        printf("child 1 lock lock2\n");
        if ((err3 = pthread_mutex_lock(&lock2)) != 0)
        {
            err_exit(err, "child 1 can't lock lock2");
        }
        printf("child 1 process here\n");

        printf("child 1 lock unlock2\n");
        if ((err3 = pthread_mutex_unlock(&lock2)) != 0)
        {
            err_exit(err, "child 1 can't unlock lock2");
        }
        printf("child 1 lock unlock1\n");
        if ((err3 = pthread_mutex_unlock(&lock1)) != 0)
        {
            err_exit(err, "child 1 can't unlock lock1");
        }
        printf("child finish\n");
    }
    else{
        printf("parent unlock lock1\n");
        if((err=pthread_mutex_unlock(&lock1))!=0)
        {
            err_exit(err, "parent can't unlock lock1");
        }
        printf("parent finish\n");
    }
    return (void*)0;
}

int main(void)
{
    int err;
    pthread_t tid;
    printf("parent create pthread\n");
    if((err=pthread_create(&tid, NULL, parent_thr_fn, NULL))!=0)
    {
        printf("parent can't create pthread\n");
    }
    sleep(10); //等待创建的线程完成
    return 0;
}

```

执行结果：

```shell
$ ./pthread_atfork.o 
parent create pthread
parent lock lock1
prepare fn lock lock2
parent fn unlock lock2
parent unlock lock1
parent finish
child fn unlock lock2
child fn unlock lock1
child 2 lock lock1
child 2 lock lock2
child process here
child 2 lock unlock2
child 2 lock unlock1
child 2 lock lock1
child 1 lock lock2
child 1 process here
child 1 lock unlock2
child 1 lock unlock1
child finish

```

解释：程序首先创建一个在线程，在新创建的线程中获取锁`lock1`。注册`fork`处理程序。`prepare`获取锁`lock2`，`parent`释放锁`lock2`，`child`释放锁`lock1`和`lock2`。而后创建新进程，原来的进程释放`lock1`结束。新进程中，由于`child`函数释放了两个锁，所以两个锁都是未锁定状态。先创建一个子线程，在子线程中获取两把锁，处理之后的程序，再释放两把锁，在原来的线程中先等待创建的子线程完成，而后获取两把锁，执行处理程序，而后释放两把锁。

## 线程和I/O

函数`pread`和`pwrite`在多线程中是十分有用的，由于同一进程共享文件描述符，如果偏移量发生变化与读取之间又别的线程更改了偏移量，读取或写就会出错，`pread`和`pwrite`将更改偏移量和读写组成了原子操作，这样保证读写的准确性。

# 第十三章 守护进程

守护进程是生存期长的一种进程。他们常常在系统引导装入时启动，仅在系统关闭时才终止。它们没有控制终端，因此都是在后台运行的，往往用来处理日常事物活动。

## 守护进程特性

`ps`命令打印系统中各个进程的状态。`ps -axj`：选项`-a`显示由其他用户拥有的进程的状态，`-x`显示没有控制终端的进程的状态，`-j`显示与作业有关的信息：会话ID、进程组ID、控制终端以及终端进程组ID。其输出是：

```
$ ps -axj
 PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
    0     1     1     1 ?           -1 Ss       0   0:02 /sbin/init splash
    0     2     0     0 ?           -1 S        0   0:00 [kthreadd]
    2     7     0     0 ?           -1 S        0   0:00 [ksoftirqd/0]
    2    10     0     0 ?           -1 S        0   0:00 [migration/0]
    2    11     0     0 ?           -1 S        0   0:00 [watchdog/0]
    2    87     0     0 ?           -1 I<       0   0:00 [writeback]
    2   300     0     0 ?           -1 I<       0   0:00 [ext4-rsv-conver]
    1  1258  1258  1258 ?           -1 Ss       0   0:00 /sbin/wpa_supplicant -u
    1  1260  1260  1260 ?           -1 Ssl    102   0:00 /usr/sbin/rsyslogd -n
    1  1270  1270  1270 ?           -1 Ssl      0   0:00 /usr/bin/python3 /usr/b
    1  1273  1273  1273 ?           -1 Ss       0   0:00 /lib/systemd/systemd-lo
    1  1277  1277  1277 ?           -1 Ssl      0   0:00 /usr/lib/accountsservic
    1  1284  1284  1284 ?           -1 Ssl      0   0:00 /usr/sbin/ModemManager 
    1  1286  1286  1286 ?           -1 Ssl      0   0:00 /usr/lib/udisks2/udisks
    1  1290  1290  1290 ?           -1 Ssl      0   0:00 /usr/sbin/thermald --no
    1  1293  1293  1293 ?           -1 Ssl      0   0:00 /usr/sbin/irqbalance --
    1  1295  1295  1295 ?           -1 Ss     116   0:00 avahi-daemon: running [
    1  1296  1296  1296 ?           -1 Ssl      0   0:00 /usr/sbin/NetworkManage
    1  1297  1297  1297 ?           -1 Ssl      0   0:01 /usr/lib/snapd/snapd
    1  1298  1298  1298 ?           -1 Ss       0   0:00 /usr/sbin/cron -f
    1  1503  1503  1503 ?           -1 Ssl      0   0:00 /usr/sbin/gdm3
    1  1515  1514  1514 ?           -1 S      108   0:00 /usr/sbin/dnsmasq -x /r
    1  1532  1531  1531 ?           -1 Sl     125   0:00 /usr/sbin/mysqld --daem
 1503  1533  1503  1503 ?           -1 Sl       0   0:00 gdm-session-worker [pam
    2  4652     0     0 ?           -1 I        0   0:00 [kworker/3:1]
    2  4673     0     0 ?           -1 I        0   0:00 [kworker/u24:0]
    2  4674     0     0 ?           -1 I        0   0:00 [kworker/10:0]
 2376  4707  4707  4707 ?           -1 Rsl   1000   0:00 /usr/lib/gnome-terminal
 4707  4718  4718  4718 pts/0     4727 Ss    1000   0:00 bash
 4718  4727  4727  4718 pts/0     4727 R+    1000   0:00 ps -axj

```

系统进程依赖于操作系统的实现。父进程为0的进程通常是内核进程，它们作为系统引导装入过程的一部分而启动。内核进程是特殊的，通常存在于系统的整个生命周期中。以超级用户特权运行，无控制终端，无命令行。

对于需要在进程上下文执行工作但却不被用户层进程上下文调用的每一个内核组件，通常有自己的内核守护进程。例如：

1. `kswapd`守护进程也被称为内存换页守护进程。支持虚拟内存子系统在经过一段时间后将脏页面慢慢写回磁盘来回收这些页面。
2. `flush`守护进程用于内存达到设置的最小阈值时将脏页面冲洗至磁盘。
3. `sync_supers`守护进程定期将文件系统元数据冲洗至磁盘。
4. `jbd`守护进程帮助实现`exit4`文件系统中的日志功能。

大多数守护进程都以超级用户特权运行。所以的守护进程都没有控制终端，其终端名设置为问号。大多数用户层守护进程都是进程组的组长进程以及会话的首地址，而且是这些进程组和会话的唯一进程（`rsyslogd`除外）。用户层守护进程的父进程是`init`进程。

## 编程规则

下面为守护进程的编译一般规则：

1. 首先使用`umask`将文件模式创建屏蔽字设置为一个已知值（通常是0）。由继承得来的文件模式创建屏蔽字可能会被设置为拒绝某种权限。如果守护进程要创建文件，那么它可能要设置特定的权限。另一方面，如果守护进程调用的库函数创建了文件，那么将文件模式创建屏蔽字设置为一个限制性更强的值（如007）可能会更明智，因为库函数可能不允许调用者通过一个显示的函数来设置权限。
2. 调用`fork`，然后使父进程`exit`。这样做实现了下面几点。第一：如果守护进程是作为一条简单的shell命令启动的，那么父进程终止会让shell认为这条命令已近执行完毕。第二：虽然子进程继承了父进程的进程组ID，但获得了一个新的进程ID，这将保证了子进程不是一个进程组的组长进程，这是下面将用进行的`setsid`调用的先决条件。（具体看第九章会话节）。在基于`system v`的系统中，建议再次调用`fork`，终止父进程，继续使用子进程中的守护进程。这就保证了该守护进程不是会话首进程，可以防止其取得控制终端。
3. 调用`setsid`创建一个会话。使调用进程：（a）成为新会话的首进程，（b）成为一个新进场的进程组组长进程，（c）没有控制终端。
4. 将当前工作目录改为根目录。从父进程继承过来的当前目录可能是在一个挂载的文件系统中。因为守护进程通常在系统再引导之前一直存在，所以守护进程的当前工作目录在一个文件系统中，那么该文件系统就不能被正常挂载。
5. 关闭进程不在需要的文件描述符。
6. 某些守护进程打开`/dev/null`使其具有文件描述符0、1和2，这样，任何一个试图读标准输入、写标准输出或者标准错误的库例程都不会产生任何效果。因为守护进程不与终端设备关联，所以其输出无处显示，也无处从交互式用户那里接收输入。

例程：

```c
#include"apue.h"
#include"syslog.h"
#include<fcntl.h>
#include<sys/resource.h>

void daemonize(const char *cmd)
{
    int i, fd0, fd1, fd2;
    pid_t pid;
    rlimit rl;
    struct sigaction sa;

    // clear file creation mask
    umask(0);

    //get maximun number of file description
    if(getrlimit(RLIMIT_NOFILE, &rl)<0)
    {
        err_quit("%s, can't get file limit", cmd);
    }

    // become a session leader to lose controlling TTY
    if((pid = fork())<0)
    {
        err_quit("%s, can't fork", cmd);
    }
    else if(pid != 0)
    {
        exit(0);
    }
    setsid();

    // ensure future opens won't allocate controlling TTYs
    sa.sa_handler = SIG_IGN;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    if(sigaction(SIGHUP, &sa, NULL)<0)
    {
        err_quit("%s, can't ignore sighup", cmd);
    }
    if((pid = fork())<0)
    {
        err_quit("%s, can't fork", cmd);
    }
    else if(pid!=0)
    {
        exit(0);
    }

    // change the current working directory to the root
    if(chdir("/")<0)
    {
        err_quit("%s, can't change directory to /", cmd);
    }

    //close all open file descriptors
    if(rl.rlim_max == RLIM_INFINITY)
    {
        rl.rlim_max = 1024;
    }
    for(i=0;i<rl.rlim_max;i++)
    {
        close(i);
    }

    // attach file descriptors 0,1,2 to /dev/null
    fd0 =open("/dev/null", O_RDWR);
    fd1 = dup(0);
    fd2 = dup(0);

    // initialize the log file
    openlog(cmd, LOG_CONS, LOG_DAEMON);
    if(fd0 !=0 || fd1 != 1 || fd2 != 2)
    {
        syslog(LOG_ERR, "unexpection file descriptors %d,%d,%d", fd0, fd1, fd2);
        exit(1);
    }
}

int main()
{
    daemonize("daemonzie");
}

```

## 出错记录

守护进程没有控制终端，不能简单的写到标准错误上。对于出错记录BSD的`syslog`设施被广泛应用：

![写错误](https://s2.ax1x.com/2019/10/01/uUg538.png)

有以下三种日志生成的方式：

1. 内核例程调用`log`函数。任何一个进程都可以通过打开（`open`）并读取（`read`）`/dev/klog`设备来读取这些消息。
2. 大多数用户进程（守护进程）调用`syslog`函数来产生日志消息。下面将进行详细的解释。这使得消息被发送至UNIX域数据报套接字`/dev/log`。
3. 无论一个用户进程是在此主机上，还是通过`TCP/IP`网络连接到此主机的其他主机上。都可以将日志消息发送到UDP端口514。注意：`syslog`函数不产生这些UDP数据报，它们要求产生此日志消息的进程进行显示的网络编程。

通常`syslogd`守护进程读取所以三种格式的日志消息。此守护进程在启动时读取一个配置文件，其名通常是`/etc/syslog.conf`，该文件决定了不同种类的消息该发送至何处。例如：紧急消息可发送至系统管理员（若已登录），并在控制台上打印，而警告信息则可记录到一个文件中。接口函数为：

```c
#include<syslog.h>
void openlog(const char *ident, int open, int facility);
void syslog(int priority, const char *format, ...);
void closelog(void);
int setlogmask(int maskpir);
//返回值：前日志记录优先级屏蔽字值

```

调用`openlog`是可选的。如果不调用`openlog`，则在第一次调用`syslog`时，自动调用`openlog`。调用`closelog`也是可选的，它只是关闭曾用于与`syslogd`守护进程进行通信的描述符。

调用`opnlog`使我们可以指定一个`ident`，以后此`ident`将被加至每则日志消息中。`ident`一般是程序名字。`option`参数是指定各种选项的位屏蔽。可选下面值：

| `option`     | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| `LOG_CONS`   | 若日志消息不能通过UNIX域数据报送至`syslogd`，则将消息写至控制台。 |
| `LOG_NDELAY` | 立即打开至`syslogd`守护进程的UNIX域数据报套接字，不要等到第一条消息已经被记录时才打开。通常在记录第一条消息之前不打开该套接字。 |
| `LOG_NOWAIT` | 不要等待在将消息记入日志过程中可能已创建的子进程.因为在`syslog`调用`wait`时，应用程序可能已经获得了子进程的状态，这种处理阻止了与捕获`SIGCHLD`信号应用程序之间产生的冲突。（`syslog`调用应该会创建子进程） |
| `LOG_ODELAY` | 在第一条消息被记录之前延迟打开至`syslogd`守护进程的连接。    |
| `LOG_PERROR` | 除将日志消息发送至`syslogd`以外，还将它写至标准错误。        |
| `LOG_PID`    | 记录每条消息都要包含进程ID。此选项可供对每个不同的请求都`fork`一个子进程的守护进程使用。 |

`openlog`的`facility`参数值取自下图：

![facility and level](https://s2.ax1x.com/2019/11/09/Mm9L8I.png)

设置`facility`参数的目的是可以让配置文件说明，来自不同设置的消息将以不同的方式进行处理。

调用`syslog`产生一个日志消息。其`priority`参数是`facility`和`level`的组合。`level`见上图。

将`format`参数以及其他所以参数传至`vsprintf`函数以便进行格式化。在`format`中，每个出现的`%m`字符都将先被代换为与`erron`值对于的出错消息字符串（strerror）。

`setlogmask`函数用来设置进程的记录优先级屏蔽字。它返回调用它之前的屏蔽字。当设置了记录优先级屏蔽字时，各条消息除非已经在记录优先级屏蔽字中进行了设置，否则不会被记录。

实例：

在一个（假定的）行式打印机假脱机守护进程中，可能包含有下面的调用序列：

```
openlog("lpd", LOG_PID, LOG_LPR);
syslog(LOG_ERR, "open error for %s: %m", filename);

```

第一个调用将`ident`字符串设置为程序名，指定该进程ID要始终被打印，并且将系统默认的`facility`设定为行打印机系统。对`syslog`的调用指定一个出错条件和一个消息字符串。如果不调用`openllog`，则第二个调用形式可能是：

```
syslog(LOG_ERR | LOG_LPR, "open error for %s: %m",filename);

```

其中将`priority`参数被指定为level和facility的组合。



## 单实例守护进程

为了正常运作，某些守护进程会实现为，在任一时刻只运行该守护进程的一个副本。例如，这种守护进程可能需要排它的访问一个设备。

如果一个守护进程需要访问一个设备，而该设备驱动程序有时会阻止想要多次打开`/dev`目录下相应设备节点的尝试。这就限制了在一个时刻只能运行守护进程的一个副本。但如果没有终止设备可供使用，那么我们需要自己处理来保证任一时刻只运行该守护进程的一个副本。

文件和记录锁机制为一中方法提供了基础，该方法保证一个守护进程只有一个副本在运行。（具体在下一章讨论）如果每一个守护进程创建一个有固定名字的文件，并在该文件的整体上加一把锁，那么只允许创建一把这样的写锁。在此之后创建写锁的尝试都会失败，这向后续守护进程的副本指明已有一个副本正在运行。

文件和记录锁提供了一种方便的互斥机制。如果守护进程在一个文件的整体上得到一把写锁，那么在该守护进程终止时，这把锁自动删除。

例程：

```c
#include<unistd.h>
#include<stdio.h>
#include<fcntl.h>
#include<syslog.h>
#include<string.h>
#include<errno.h>
#include<stdio.h>
#include<sys/stat.h>
#include<stdlib.h>

#define LOCKFILE "/var/run/daemon.pid"
#define LOCKMDE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

extern int lockfile(int);

int already_running(void)
{
    int fd;
    char buf[16];

    fd = open(LOCKFILE, O_RDWR | O_CREAT, LOCKMDE);
    if(fd<0)
    {
        syslog(LOG_ERR, "can't open %s: %s", LOCKFILE, strerror(errno));
        exit(0);
    }

    if(lockfile(fd)<0)
    {
        if(errno == EACCES || errno == EAGAIN)
        {
            close(fd);
            return 1;
        }
        syslog(LOG_ERR, "can't lock %s: %s", LOCKFILE, strerror(errno));
        exit(1);
    }
    ftruncate(fd, 0);
    sprintf(buf, "%ld", (long)getpid());
    write(fd, buf, strlen(buf)+1);
    return 0;
}

```

守护进程的每个副本试图创建一个文件，并将其进程ID写到该文件中。如果该文件已经加锁，那么`lockfile`函数将失败，`erron`将被设置为`EACESS`或`EAGAIN`，函数返回1，表示该守护进程存在一个副本在运行。否则将文件长度截断为0，将进程ID写入该文件。将文件截断为0的原因是，之前的进程ID可能长于当前的进程ID，如之前是12345，现在是9999，则如果不截断，则会变成99995。

## 守护进程

UNIX中，守护进程遵循下列通用惯例：

1. 若守护进程使用锁文件，那么该文件通常存储在`/var/run`目录中。守护进程需要超级用户权限才能在此文件夹下创建文件。锁文件的名字通常是`name.pid`，其中`name`是该守护进程或者访问的名字。
2. 若守护进程支持配置选项，那么配置文件通常放在`/etc`目录中。配置文件的名字通常是`name.conf`，其中`name`是该守护进程或者访问的名字。
3. 守护进程可用命令行启动，但通常它们是由系统初始化脚本之一（`/etc/re*`或`/etc/init.d/*`)启动的。如果在守护进程终止时，应当自动地重新启动它，则我们可用在`/etc/inittab`中为该进程包括`respawn`记录项，这样`init`就会重新启动该进程。
4. 若一个守护进程有一个配置文件，那么当该守护进程启动时会读该文件，但在此之后一般不会再查看它。若某个管理员更改了配置文件，那么该守护进程可能需要被停止，然后在启动，以使配置文件生效。为了避免这种麻烦，某些文件将捕捉`SIGHUP`信号，当它们接收到信号时，重新读取配置文件。因为守护进程并不与终端相结合，它们或者是无终端的会话首进程，或者是孤儿进程组的成员，所以守护进程没有理由期望接收到`SIGHUP`，因此可以安全地重复使用`SIGHUP`。

例程：

```c
#include"apue.h"
#include<pthread.h>
#include<syslog.h>

sigset_t mask;

extern int already_running(void);

void reread(void)
{
    /*...*/
}

void *thr_fn(void *arg)
{
    int err, signo;
    while(1)
    {
        //wait signal which included mask
        err = sigwait(&mask, &signo);
        if(err!=0)
        {
            syslog(LOG_ERR, "sigwait failed");
            exit(1);
        }
        switch(signo)
        {
            case SIGHUP:
            syslog(LOG_INFO, "Re-reading configuration file");
            reread();
            break;
            case SIGTERM:
            syslog(LOG_INFO, "get SIGTERM;exiting");
            exit(0);
            default:
            syslog(LOG_INFO, "unexpected signal %d\n", signo);
        }
    }
    return 0;
}

int main(int argc, char *argv[])
{
    int err;
    pthread_t tid;
    char *cmd;
    struct sigaction sa;
    if((cmd = strrchr(argv[0],'/')) == NULL)
    {
        cmd = argv[0];
    }
    else{
        cmd++;
    }

    /* become daemon */
    daemonize(cmd);

    /* ensure only one copy of the daemon is running*/

    if(already_running())
    {
        syslog(LOG_ERR, "daemon already running");
        exit(1);
    }

    /* restore SIGHUP default and block all signals*/

    sa.sa_handler = SIG_DFL;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    if(sigaction(SIGHUP, &sa, NULL)<0)
    {
        err_quit("%s: can't restore SIGHUP default");
    }
    sigfillset(&mask);
    if((err = pthread_sigmask(SIG_BLOCK, &mask, NULL))!=0)
    {
        err_exit(err, "SIG_BLOCK error");
    }

    /* create a pthread to handle SIGHUP and SIGTERM*/

    err = pthread_create(&tid,NULL,thr_fn, (void*)0);
    if(err!=0)
    {
        err_exit(err, "can't create thread");
    }
    /* process with the rest of the daemon*/

    exit(0);
}

```

这里使用创建了一个线程专门用来处理信号，当然也可以使用一个单线程守护进程来实现。

## 客户进程-服务进程模型

守护进程通常用服务器进程。用户进程用UNIX域数据报套接字向其发送消息。一般而言，服务器进程等待客户进程与其连续，提出某种类型的服务请求。

在服务器进程中调用`fork`然后`exec`另一个程序来向客户进程提供服务是很常见的。这些服务器进程通常管理者多个文件描述符：通信端点、配置文件、日志文件和类似的问价。最好的情况下，让子进程中的这些文件描述符保持打开状态并无大碍，因为在子进程中很可能用不到。最坏情况下，保持打开可能会导致安全问题：被执行程序可能有一些恶意行为，如更改服务器配置文件或欺骗客户端程序使其认为正在与服务器通信，从而获取未授权的信息。

为了解决此问题的一个简单方式是对所以被执行的文件描述符设置执行时关闭，可以使用如下的程序：

```
#include"apue.h"
#include<fcntl.h>

int set_cloexec(int fd)
{
	int val;
	if((val = fctl(fd, FD_GETFD, 0))<0)
	{
		return -1;
	}
	val |= FD_CLOEXEC;
	return fctl(fd, F_SETFD, val);
}

```



# 第十四章 高级I/O

## 非阻塞I/O

系统调用分为两类：低速系统调用和其他系统调用。低速系统调用是可能会使进程永远阻塞的一类系统调用(第十章)：

1. 如果某些类型文件（如读管道、终端设备和网路设备）的数据不存在，则读操作可能会使调用者永远阻塞。
2. 如果这些数据不能被相同类型的文件立即接受，则写操作可能会使调用者永远阻塞。
3. 在某些条件发生之前打开某些文件，可能会发生阻塞（例如打开一个终端设备，需要先等待与之连接的调制解调器应答）。
4. 对已经加上强制性记录锁的文件进行读写。
5. 某些`ioct1`操作。
6. 某些进程间通信函数。

虽然读写磁盘的操作会暂时阻塞调用者，但不能将与磁盘相关I/O有关的系统调用视为低速。

非阻塞I/O使我们可以发出`open`、`read`和`write`这样的I/O操作，并使这些操作不会永远阻塞。如果这种操作不能完成，则调用者立即出错返回，表示该操作若继续执行将阻塞。

对于一个给定的描述符，有两种方式为其指定非阻塞I/O的方法：

1. 如果调用`open`获得描述符，则可以指定`O_NONBLOCK`标志。
2. 对于已经打开的文件描述符，则可以调用`fcntl`，由该函数打开`O_NONBLOCK`文件状态标志。

例程：

```c
#include"apue.h"
#include<errno.h>
#include<fcntl.h>

char buf[500000];

void set_fl(int fd, int flags)
{
    int val;
    if(val = fcntl(fd, F_GETFD, 0)<0)
    {
        err_sys("fcntl F_GETFD error");
    }
    val |= flags;
    if(fcntl(fd, F_SETFL, val)<0)
    {
        err_sys("fct1 F_SETFL error");
    }

}

void clr_fl(int fd, int flags)
{
    int val;
    if(val = fcntl(fd, F_GETFD, 0)<0)
    {
        err_sys("fcntl F_GETFD error");
    }
    val &= (~flags);
    if(fcntl(fd, F_SETFL, val)<0)
    {
        err_sys("fct1 F_SETFL error");
    }

}

int main(void)
{
    int ntowrite, nwrite;
    char *ptr;
    ntowrite = read(STDIN_FILENO, buf, sizeof(buf));
    fprintf(stderr, "read %d byte\n", ntowrite);

    set_fl(STDOUT_FILENO, O_NONBLOCK);

    ptr = buf;

    while (ntowrite > 0)    
    {
        errno = 0;
        nwrite =write(STDOUT_FILENO, ptr, ntowrite);
        fprintf(stderr, "nwrite = %d, errno=%d\n", nwrite, errno);

        if(nwrite>0)
        {
            ptr += nwrite;
            ntowrite -= nwrite;
        }
    }

    clr_fl(STDOUT_FILENO, O_NONBLOCK);
    exit(0);  
}

```

运行：

```shell
$ ls -l /etc/services 
-rw-r--r-- 1 root root 19183 12月 26  2016 /etc/services

##输出到指定文件
$ ./14_2.o < /etc/services > temp.file
read 19183 byte
nwrite = 19183, errno=0

## 输出到终端
$ ./14_2.o < /etc/services 2>stderr.out
.....
.....
.....
$ cat stderr.out 
read 19183 byte
nwrite = 18667, errno=0
nwrite = -1, errno=11
nwrite = -1, errno=11
.
.
.
.

nwrite = -1, errno=11
nwrite = -1, errno=11
nwrite = 516, errno=0
chst@wyk-GL63:~/study_file/unix编程$

```

在向终端输出的过程中，发出了大量`write`调用，但只有2个产生了真正的输出，其余都返回了错误。这种形式的循环称为轮询，在多用户系统上会浪费CPU时间。



## 记录锁

### fcntl记录锁

记录锁的功能是：当第一个进程正在读或修改文件的某个部分时，使用记录锁可以阻止其他进程修改同一文件区。

`fcntl`记录锁：

```
#include<fcntl.h>
int fcntl(int fd, int cmd, .../*struct flock *flockptr */);
//成功，返回依赖于cmd，否则返回-1

```

对于记录锁，cmd是`F_GETLK`, `F_SETFL`, `F_SETLKW`。第三个参数是一个指向`flock`结构的指针：

```c
struct flock{
	short l_type; /*F_RDLCK, F_WRLCK, or F_UNLCK */
    short l_whence; /* SEEK_SET, SEEK_CUR, or SEEK_END*/
    off_t l_start; /*offset in bytes, relative to l_whence*/
    off_t l_len; /*length , in byte, 0 mean lock to EOF*/
    pid_t l_pid; /*return with F_GETLK*/
}

```

`flock`结构说明如下：

1. 所希望锁类型： `F_RDLCK`(共享读锁)、`F_WRLCK`（独占性写锁）或`F_UNLCK`(解锁一个区域)。
2. 要加锁或解锁区域的起始字节偏移量（`l_start`和`l_whence`）。
3. 区域的字节长度。
4. 进程ID（`l_pid`）持有的锁能阻塞当前进程（仅由`F_GETLK`返回）。

锁可以在当前文件开始或者越过尾端处开始，但不能在文件起始之前开始。如若`l_len`为0，则表示锁的范围可以扩展到最大可能偏移量。这意味着不管向该文件中追加了多少数据，他们都可以处于锁的范围内，而且起始位置可以是文件终端任意位置。

共享读锁和独占写锁基本规则是：任意多个进程在一个给定的字节上可以有一把共享的读锁，但是在一个给定字节上只能有一把独占写锁。这个规则只适用于多个进程，并不适用于单个进程的多个锁请求。<font color = red>如果一个进程对一个文件区域已经有一把锁，后来该进程又企图在同一个文件区域再加一把锁，那么新的锁将会替换已有的锁。</font>

加读锁时，该描述符必须是读打开的。加写锁时，该描述符必须是写打开的。

对于`fcntl`函数的3中命令：

| `F_GETLK`  | 判断由`flockptr`所描述的锁是否会被另外一把锁排斥（阻塞）。如果存在一把锁，它阻止创建由`flockptr`锁描述的锁，则该现有锁的信息将重写`flockptr`指向的信息。如果不存在这种情况，则除了将`l_type`设置为`F_UNLCK`之外，`flockptr`所指向的信息保持不变。 |
| ---------- | ------------------------------------------------------------ |
| `F_SETLK`  | 设置由`flockptr`所描述的锁。如果试图获得一把读锁或写锁，而兼容性规制阻止系统给我们这把锁，那么`fcntl`会立即出错返回，此时`errno`设置为`EACCES`或`EAGAIN`。 |
| `F_SETLKW` | 该命令是`F_SETLK`的阻塞版本。如果加锁不能被授权，那么调用进程会被设置为阻塞。如果请求创建的锁已经可用，或者休眠由信号中断，则该进程被唤醒。 |

`F_GETLK`  和`F_SETLK`之间不是原子操作，因此在执行完第一个查询后不能保证是否有别的进程插入并建立一把相同的锁。如果不希望在等待锁变成可用时产生阻塞，就必须处理由`F_SETLK`返回的可能错误。

实例：请求和释放一把锁

```c
#include"apue.h"
#include<fcntl.h>

int lock_reg(int fd, int cmd, int type, off_t offset, int whenec, off_t len)
{
    flock lock;
    lock.l_len = len;
    lock.l_start = offset;
    lock.l_type = type;
    lock.l_whence = whenec;

    return fcntl(fd, cmd, &lock);
}

```

在`apue.h`中定义了五个宏：

```c
#define	read_lock(fd, offset, whence, len) \
			lock_reg((fd), F_SETLK, F_RDLCK, (offset), (whence), (len))
#define	readw_lock(fd, offset, whence, len) \
			lock_reg((fd), F_SETLKW, F_RDLCK, (offset), (whence), (len))
#define	write_lock(fd, offset, whence, len) \
			lock_reg((fd), F_SETLK, F_WRLCK, (offset), (whence), (len))
#define	writew_lock(fd, offset, whence, len) \
			lock_reg((fd), F_SETLKW, F_WRLCK, (offset), (whence), (len))
#define	un_lock(fd, offset, whence, len) \
			lock_reg((fd), F_SETLK, F_UNLCK, (offset), (whence), (len))

```

实例：测试一把锁

```c
#include"apue.h"
#include<fcntl.h>

pid_t lock_test(int fd, int type, off_t offset, int whence, off_t len)
{
    flock lock;
    lock.l_whence = whence;
    lock.l_type = type;
    lock.l_start = offset;
    lock.l_len = len;

    if(fcntl(fd, F_GETLK, &lock)<0)
    {
        err_sys("fcntl error");
    }

    if(lock.l_type == F_UNLCK)
    {
        return 0; /* false region isn't locked by another proc*/
    }
    return lock.l_pid; /*true, return pid of lock owner*/
}

```

`apue.h`中定义了两个宏：

```c
#define	is_read_lockable(fd, offset, whence, len) \
			(lock_test((fd), F_RDLCK, (offset), (whence), (len)) == 0)
#define	is_write_lockable(fd, offset, whence, len) \
			(lock_test((fd), F_WRLCK, (offset), (whence), (len)) == 0)

```

注意：进程不能使用`lock_test`函数来测试它自己是否在文件的某一部分持有锁。`F_GETLK`定义说明，返回信息指示是否现有的锁会阻止调用进程获取自己的锁。因为`F_SETLK`和`F_SETLKW`命令总是替换调用进程现有的锁，所以调用进程不会阻塞在自己持有的锁上。

实例：死锁。

当两个进程相互等待对方持有并且不释放的资源时，则两个进程进入死锁状态。

```c
#include"apue.h"
#include<fcntl.h>

static void lockabyte(const char *name, int fd, off_t offset)
{
    if(write_lock(fd, offset, SEEK_SET, 1)<0)
    {
        err_sys("%s: write_lock error", name);
    }
    printf("%s: got the lock, byte %lld\n", name, (long long)offset);
}

int main(void)
{
    int fd;
    pid_t pid;

    if((fd = creat("templock", FILE_MODE))<0)
    {
        err_sys("create file error");
    }
    if(write(fd, "ab", 2)!=2)
    {
        err_sys("write to file error");
    }

    TELL_WAIT();
    if((pid = fork())<0)
    {
        err_sys("fork error");
    }
    else{
        if(pid>0)
        {
            lockabyte("parent", fd, 0);
            TELL_CHILD(pid);
            WAIT_CHILD();
            lockabyte("parent", fd, 1);
        }
        else{
            lockabyte("child", fd, 1);
            TELL_PARENT(getppid());
            WAIT_PARENT();
            lockabyte("child", fd, 0);
        }
    }
    exit(0);
}

```

执行结果：

```shell
$ ./14_7.o
parent: got the lock, byte 0
child: got the lock, byte 1
child: write_lock error: Resource temporarily unavailable
parent: write_lock error: Resource temporarily unavailable

```

这里可以看到，父进程和子进程都无法获得锁，陷入死锁状态。

### 锁的隐含继承和释放

锁的自动继承和释放有3条规则：

（1）锁与进程和文件两者相关联。有两层含义：第一个是当进程终止时，其所建立的锁全部释放。第二是无论一个描述符何时关闭，该进程通过这一描述符引用的文件上的任何一把锁都会释放。因此：

```c
fd1 = open(pathname, ...);
read_lock(fd1, ...);
fd2 = dup(fd1);
close(fd2);

```

在`close(fd2)`后在`f1`上设置的锁被释放。将`dup`换成`open`也是一样的。

（2）由fork产生的子进程不继承父进程所设置的锁。

（3）在执行exec后，新程序可以继承原执行的锁。但如果一个文件描述符设置了执行时关闭，那么exec后，会关闭文件同时释放锁。

### FreeBSD实现

考虑进程执行下面语句：

```c
fd1 = open(pathname, ...);
write_lock(fd1, 0, SEEK_SET, 1);
if((pid = fork()) >0)
{
	fd2 = dup(fd1);
	fd3 = open(pathname, ...);
}
else if(pid == 0)
{
	read_lock(fd1, 1, SEEK_SET, 1);
}
pause();

```

下图展示了运行到`pause`时数据结构：

![pause](https://s2.ax1x.com/2019/10/01/uUgLEn.png)

这里在原来图的基础上添加了`lockf`结构，它们由`i`节点结构开始互相连接起来。每个`lockf`结构描述一个给定进程的一个加锁区域（由偏移量和长度决定）。在父进程中，关闭`fd1`、`fd2`或`fd3`中任意一个都会释放由父进程设置的写锁。在关闭这三个其中一个时，内核会从该描述符关联的`i`节点开始，逐个检查`lockf`链表中各项，并释放由该调用进程锁持有的各把锁。

实例：在单实例守护进程中我们使用`lockfile`函数来保证只有该守护进程的唯一副本在运行，下面给出其函数实现：

```c
int lockfile(fd)
{
	flock f1;
	f1.l_type = F_WRLCK;
	f1.l_start = 0;
	f1.l_whence = SEEK_SET;
	f1.l_len = 0;
	return fcntl(fd, F_SETLK, &f1);
}

```

### 在文件末尾加锁

在获取从某个位置到文件末尾的锁时，不能简单的使用`fstat`函数来获取文件长度来进行加锁，因为在`fstat`之后和加锁之前，可能存在别的进程改变该文件。因此一般是指定长度为0，此时可获得到文件末尾的锁。但是考虑如下代码：

```c
writew_lock(fd, 0, SEEK_END, 0);
write(fd, buf, 1);
un_lock(fd, 0, SEEK_END);
write(fd, buf, 1);

```

该代码获取一把写锁，该写锁从当前文件末尾起，包括以后可能追加写到该文件的任何数据。当文件偏移量处于末尾时，执行第一个写，该操作将文件延长1个字节，且被加锁。随后的解锁操作是对以后追加写到文件上的数据不加锁。但之前写的一个字节则保留加锁状态。写第二个字节时，文件末尾又延伸一个字节，但未加锁。想要删除所以锁，应该使用`un_lock(fd, -1, SEEK_END);`。这里-1是相对偏移量，表示相对末尾的前一个字节。

### 建议性锁和强制性锁

强制性锁会让一内核检测每一个`open`、`read`和`write`，验证调用进程是否违背了正在访问的文件上的一把锁。强制性锁有时也被称作强迫方式锁。

这里建议性锁就是我们之前所叙述的锁，我们使用锁来保证读写时不与其他进程冲突。而强制性锁是相对于某个进程不使用锁来保证访问冲突时发生的情况，即，一些进程使用了锁，而另一些进程根本没想过要使用锁而直接对文件进行打开、读取和写的操作时会发生什么情况。对于建议性锁来说，可以正常执行，但可能导致进程间混乱冲突。对于强制性锁，如果有的进程已经又了该锁，而且按照规则当前进程不应该进行相关操作却进行时，会产生错误。这两者之间的比较看该节最后的例程会有更加深入的理解。

对一个特定文件，打开其设置组ID位、关闭其组执行位便开启了对该文件的强制性锁机制。因为当关闭组执行位时，设置组ID位将不在有意义。

当一个进程试图读写一个强制性锁起作用的文件时，下图展示了其可能情况：

![state](https://s2.ax1x.com/2019/11/30/QZpG24.png)

除了对`read`和`write`函数产生影响，另一个进程持有的强制性锁也会对`open`函数产生影响。如果要打开的文件具有强制性记录锁，而且`open`调用的标识是`O_TRUNC`或`O_CREAT`，则不论是否指定`O_NONBLOCK`，`open`都立即出错返回，`error`设置为`EAGAIN`。

`Linux`使用`strace`命令可以得到一个进程的系统调用跟踪信息。`Linux`如果用户想要使用强制性锁，需要在各个文件系统基础上用`mount`命令的`-o mand`选项来打开。

例程：确定一个系统是否支持强制性锁。

```c
#include"apue.h"
#include<errno.h>
#include<fcntl.h>
#include<sys/wait.h>

int main(int argc, char *argv[])
{
    int fd;
    pid_t pid;
    char buf[5];
    struct stat statbuf;
    if(argc != 2)
    {
        fprintf(stderr, "usage:%s filename\n", argv[0]);
        exit(1);
    }

    if((fd = open(argv[1], O_RDWR | O_CREAT |O_TRUNC, FILE_MODE))<0)
    {
        err_sys("open error");
    }

    if(write(fd, "abcdef", 6)!=6)
    {
        err_sys("write error");
    }

    /*turn on set-group-id and turn off group-execute*/
    if(fstat(fd, &statbuf)<0)
    {
        err_sys("fstat error");
    }

    if(fchmod(fd, (statbuf.st_mode & ~S_IXGRP) | S_ISGID)<0)
    {
        err_sys("fchmod error");
    }

    TELL_WAIT();

    if((pid = fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid>0)
    {
        if(write_lock(fd, 0, SEEK_SET, 0)<0)
        {
            err_sys("write_lock error");
        }
        TELL_CHILD(pid);
        if(waitpid(pid, NULL, 0)<0)
        {
            err_sys("waitpid error");
        }
    }
    else{
        WAIT_PARENT();
        set_fl(fd, O_NONBLOCK);

        if(read_lock(fd, 0, SEEK_SET, 0)!=-1)
        {
            err_sys("child: read_lock succeed");
        }
        printf("read_lock of alread-locked region return %d\n", errno);

        if(lseek(fd, 0, SEEK_SET) == -1)
        {
            err_sys("lseek error");
        }
        if(read(fd, buf, 2)<0)
        {
            err_ret("read faild (mandatory locking works)");
        }
        else{
            printf("read ok (no mandatory locking), buf=%2.2s\n", buf);
        }
    }
    exit(0);
}

```

在linux未打开强制性锁机制时：

```
$ ./edit temp.lock
read_lock of alread-locked region return 11
read ok (no mandatory locking), buf=ab

```

目前还没有成功打开强制性锁机制（哭了）。

## I/O多路转接

### 函数select和pselect

`select`函数使我们可以执行I/O多路转接。传递给`select`的参数告诉内核：

1. 我们所关心的描述符。
2. 对于每个描述符我们关系的条件（从其读，向其写，发生异常）。
3. 愿意等待时间。

从`select`返回时，内核告诉我们：

1. 已准备好的描述符数量。
2. 对于读、写或异常这三个条件中的每一个，哪些描述符已经准备好了。

函数原型：

```c
#include<sys/select.h>
int select(int maxfdp1, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict execptfds, struct timeval *restrict tvptr);
// 返回值：准备就绪的数量，若超时返回0，若出错，返回-1

```

`tvptr`指定等待时间。当`tvptr`是NULL时，永远等待。如果捕捉到一个信号则中断此无限等待。当所指定的文件描述符中的一个已经准备好或者捕捉到一个信号则返回。当`tvptr->tv_sec == 0 && tvptr->tv_usec == 0`不等待直接返回。否则等待指定的时间。

中间三个参数`readfds、writefds、execptfds`是指向描述符集的指针。这三个描述符集说明了我们所关心的读写或异常描述符集。每个描述符集存储在一个`fd_set`数据类型中。它可以为每一个可能的描述符保持一位，我们可以认为其是很大的数组。

对于`fd_set`数据类型，唯一可以进行的处理是：分配一个这种类型的变量，将这种类型的变量赋值给另一个变量，或对这种变量使用下面的函数：

```
#include<sys/select.h>
int FD_ISSET(int fd, fd_set *fdset);
//如果fd在描述符集中，返回非0， 否则返回0

void FD_CLR(int fd, fd_set *fdset);
void FD_SET(int fd, fd_set *fdset);
void FD_ZERO(fd_set *fdset);

```

申明一个描述符集时，必须使用`FD_ZERO`函数将其置0，然后设置我们关心的各个描述符位，如：

```c
fd_set rset;
int fd;
FD_ZERO(&rset);
FD_SET(fd, &fset);
FD_SET(STDIN_FILENO, &rset);

```

当从`select`返回时，应该使用`FD_ISSET`测试该集中的一个给定位是否仍处于打开状态：

```
if(FD_ISSET(fd, &rset))
	.....

```

`select`中间的三个参数任意一个都可以是NULL，空表示并没有要关心的描述符。当三个都是NULL时，`select`退化成`sleep`，不过提供了更高的精度。

`select`第一个参数`maxfdp1`意思是“最大文件描述符值加1”。考虑在3个文件描述符集中最大的文件描述符值，然后加1.也可以设置为`FD_SETSIZE`，这是`<sys/select.h>`中一个常值。该参数指定了`select`搜索范围，如果远远大于我们实际使用的文件描述符最大值加1将会造成浪费。因为文件描述符从0开始，因此要加一。第一个参数实际指定了要检测的文件描述符数量。

`select`有三个可能返回值：

（1）返回值-1表示出错。

（2）返回0表示没有描述符准备好。

（3）一个正值表示准备好的描述符数量。

准备好的含义是：

1. 对于读集中的一个描述符进行读操作不会阻塞，表示是准备好的。
2. 对于写集中的一个描述符进行写`操作不会阻塞，表示是准备好的。
3. 对于异常条件集中的一个描述符有一个未决异常条件，则认为此描述符是准备好的。现在，异常条件包括：在网络连接上到达带外的数据，或者处于数据包模式的伪终端上发生了某些条件。
4. 对于读写和异常条件。普通文件的文件描述符总是返回准备好。
5. 一个文件描述符阻塞与否不影响`select`是否阻塞。

如果一个描述符碰到了文件末尾，则`select`会认为该描述符时可读的。然后调用`read`它返回0，这是`UNIX`系统指示到达文件末尾的方式。

`POSIX.1`也定义了一个`select`的变体：

```c
#include<sys/select.h>

int pselect(int maxfdp1, fd_set *restrict readfds, fd_set *restrict writefds, fd_set *restrict execptfds, const struct timespec *retrict tsptr, const sigset_t *restrict sigmask);
// 返回值：准备就绪的数量，若超时返回0，若出错，返回-1

```

与`select`之间区别为下面几点：

1. 超时时间使用类型不一致。
2. `pselect`超时时间设置为`const`保证不会被改变。
3. `pselect`可使用信号屏蔽字。若`sigmask`为NULL，在调用`pselect`与`select`一致，否则，`sigmask`指向的信号屏蔽字将会以原子操作的方式被安装，在返回时，恢复以前的信号屏蔽字。

### 函数poll

函数原型：

```c
#include<poll.h>
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout)
// 返回值：准备就绪的数量，若超时返回0，若出错，返回-1

```

`pollfd`结构指定一个描述符编号以及我们对该描述符感兴趣的条件：

```
struct pollfd{
	int fd;
	short events;
	short reevents;
}

```

`fdarray`数组中的元素数由`nfds`指定。

`events`成员设置为下表中的一个或几个，通过这些值告诉内核我们关系的是描述符的哪些事件。返回时，`revents`成员由内核设置，用于说明每个描述符发生了那些事件。

![(r)events](https://s2.ax1x.com/2019/12/01/QZX20f.png)

前四行测试可读性，后面三行测试可写性，最后三行测试异常条件。最后三行由内核返回时设置。即使`events`未指定这三个值，如果响应条件发生，在`revents`中也会返回它们。

当一个描述符被挂断（POLLHUP）后，不能再写该描述符，但是有可能任然可以从该描述符读取数据。

`poll`最后一个参数指定了我们愿意等待的时间，其为毫秒。-1表示永久等待，0表示不等待，>0表示等待对应毫秒。

文件尾端和挂断区别，如果我们向终端输入数据，并键入文件结束符，那么就会打开`POLLIN`，于是我们可以读文件结束指示（read返回0）。`revents`中的POLLHUP并未被打开。如果正在读调制解调器，并且电话线已挂断，我们将接收到`POLLHUP`。

`select`和`poll`由于信号造成的中断一般不会重启。

## 异步I/O

`POSIX`异步I/O接口为对不同类型的文件进行异步I/O提供了一套一致的方法。这些异步接口使用AIO控制块来描述I/O操作。`aiocb`结构定义了`AIO`控制块。该结构至少包含下面的字段：

```
struct aiocb{
	int aio_filedes; /* file descriptor */
	off_t aio_offset; /* file offset for I/O */
	volatile void *aio_buf; /* buffer for I/O*/
	size_t aio_nbytes; /* number of bytes to transfer */
	int aio_reqprio; /* priority */
	struct sigevent aio_sigevent; /*signal information*/
	int aio_lio_opcode; /*operation for list I/O*/
}

```

`aio_fileds`字段表示被打开用来读写的文件描述符。读写操作从`aio_offset`指定的偏移量开始。对于读操作，数据会复制到缓冲区，该缓冲区从`aio_buf`指定的地址开始。对于写操作，数据从这个缓冲区中复制出来。`aio_nbytes`字段包含了要读写的字节数。

异步I/O必须显示的指定偏移量。异步I/O并不影响由操作系统维护的文件偏移量。不能在同一进程里把异步I/O和传统I/O函数混在一起。如果异步I/O接口向一个以追加模式打开的文件写入数据，AIO控制模块的`aio_offset`字段会自动忽略。

应用程序使用`aio_reqprio`字段为异步I/O请求提示顺序（建议性，非强制）。`aio_lio_opcode`字段只能用于基于列表的异步I/O。`aio_sigevent`控制在I/O完成后，如何通知应用程序。改字段对于结构`sigevent`为：

```c
struct sigevent{
	int sigev_notify; /* notify type*/
	int sigev_signo; /* signal number*/
	union sigval sigev_value; /notify argument*/
	void (*sigev_notify_function)(union sigval); /*notify function*/
	pthread_attr_t *sigev_notify_attributes; /*notify attrs*/
}

```

`sigev_notify`控制通知类型。取值为下面三个中一个。

| `SIGEV_NONE`   | 异步I/O请求完成后，不通知进程。                              |
| -------------- | ------------------------------------------------------------ |
| `SIGEV_SIGNAL` | 异步I/O请求完成后，产生由`sigev_signo`字段指定的信号。如果应用程序已选择捕捉信号，且在建立信号处理程序时指定了`SA_SIGINFO`标志，那么该信号将被入队（如果支持排队信号）。信号处理程序会传送给一个`siginfo`结构，该结构的`si_value`字段被设置为`sigev_value`。 |
| `SIGEV_THREAD` | 异步I/O请求完成时，由`sigev_notify_function`字段指定的函数被调用。`sigev_value`字段被传入作为它的唯一参数。除非`sigev_notify_attributes`字段被设置为`pthread`属性结构的地址，且该结构指定了一个另外的线程属性，否则该函数将在分离状态下的一个单独的线程中执行。 |

下面的函数用来实现异步读写：

```c
#include<aio.h>
int aio_read(struct aiocb *aiocb);
int aio_write(struct aiocb *aiocb);
//成功返回0，否则返回-1

```

当这些函数成功返回时，异步I/O请求便已经被操作系统放入等待队列中。返回值与实际I/O结果没有关系。I/O操作在等待时，必须确保AIO控制块和数据库缓冲区保持稳定，它们下面对应的内存必须是始终合法的，除非I/O操作完成，否则不能复用。

要想强制所以等待中的异步操作不等待而写入持久化的存储中，可以调用`aio_fsync`函数：

```c
#include<aio.h>
int aio_fsync(int op, struct aiocb *aiocb);
//成功返回0，否则返回-1

```

`op`参数设定为`O_DSYNC`，那么操作执行起来就会像调用了`fdatasync`一样，否则，如果指定`op`为`O_SYNC`，那么执行操作就会像调用了`fsync`。

下面的函数可以获得异步读写或者同步操作的完成状态：

```c
#include<aio.h>
int aio_error(const struct aiocb *aiocb);

```

函数返回值为：

| 0             | 异步操作成功完成。需要调用`aio_return`函数获取操作返回值。 |
| ------------- | ---------------------------------------------------------- |
| -1            | 对`aio_error`的调用失败。`erron`会标识原因。               |
| `EINPROGRESS` | 异步读写或同步操作任然在等待。                             |
| 其它          | 异步操作失败返回的错误码。                                 |

异步操作成功调用`aio_return`获得异步操作返回值：

```
#include<aio.h>
ssize_t aio_return(const struct aiocb *aiocb);

```

直到异步操作完成之前，都需要小心不要调用`aio_return`函数。每个异步操作只调用一次`aio_return`。一旦调用了该函数，操作系统就可以释放掉包含I/O操作返回值的记录。

如果`aio_return`函数本身失败，则返回-1，并设置`error`。其他情况下，返回`read`、`write`或`fsync`在被成功调用时可能返回的结果。

如果在完成所以事物时，还有异步操作未完成，可以调用`aio_suspend`函数来阻塞进程，直到操作完成：

```c
#include<aio.h>
int aio_suspend(const struct aiocb *const list[], int nent, const struct timespec *timeout);
//成功返回0，否则返回-1

```

`aio_suspend`可能返回三种情况中的一种。如果被信号中断，返回-1，并将`error`设置为`EINTR`。如果在没有任何I/O操作完成的情况下，阻塞时间超时，返回-1，并将`error`设置为`EAGAIN`。如果有任何I/O操作完成，返回0。如果调用`aio_suspend`时，所以异步I/O操作都完成了，那么直接返回。

`list`参数是一个指向AIO控制块数组的指针，`nent`参数表面数组中条目数。空指针会被跳过。

当还有我们不想再完成的等待中的异步I/O操作时，可以尝试使用`aio_cancel`函数来取消它们：

```
#include<aio.h>
int aio_cancel(int fd, struct aiocb *aiocb);

```

`fd`指定未完成的异步I/O操作的文件描述符。如果`aiocb`参数为NULL，系统将会尝试取消所有该文件描述符上未完成的I/O操作。该函数返回值为下面几个：

| `AIO_ALLDONE`     | 操作在尝试取消之前已经完成。            |
| ----------------- | --------------------------------------- |
| `AIO_CANCELED`    | 所有要求的操作已被取消。                |
| `AIO_NOTCANCELED` | 至少有一个要求的操作未被取消。          |
| `-1`              | 函数调用失败，错误码被存储在`erron`中。 |

`lio_listio`函数即能以同步方式来使用，又能以异步方式来使用：

```c
#include<aio.h>
int lio_listio(int mode, struct aiocb *restrict const list[restrict], int nent, struct sigevent *restrict sigev);
//成功返回0，否则返回-1

```

`mode`参数决定了I/O是否是异步的。如果是`LIO_WAIT`，则函数将在由列表指定的I/O操作完成之后返回，此时`sigev`参数会被忽略。如果`mode`参数设定为`LIO_NOWAIT`，则函数将会在I/O请求入队后立即返回。进程将在所以的操作完成后，按照`sigev`参数指定的，被异步通知。如果不想被通知，则把`sigev`设置为NULL。每个AIO本身有一个各自操作完成后的异步通知。`sigev`参数指定的异步通知是在此之外另加的，且只在所以异步操作完成之后才发送。

在每一个AIO控制块中，`aio_lio_opcode`字段指定了该操作是一个读操作（`LIO_READ`）、写操作（`LIO_WRITE`）还是将被忽略的空操作（`LIO_NOP`）。读操作会将对应的块传递给`aio_read`处理，写操作给`aio_write`处理。

实例：

```c
#include"apue.h"
#include<aio.h>
#include<ctype.h>
#include<fcntl.h>
#include<errno.h>

#define BSZ 4096
#define NBUF 8

enum rwop{
    UNUSED = 0,
    READ_PENDING = 1,
    WRITE_PENDING = 2
};

struct buf{
    enum rwop op;
    int last;
    struct aiocb aiocb;
    unsigned char data[BSZ];
};

struct buf bufs[NBUF];
unsigned char translate(unsigned char c)
{
    if(isalpha(c))
    {
        if(c >= 'n')
        {
            c -= 13;
        }
        else if(c >= 'a')
        {
            c += 13;
        }
        else if(c >= 'N')
        {
            c -= 13;
        }
        else{
            c += 13;
        }
    }
    return c;
}

int main(int argc, char *argv[])
{
    int ifd, ofd, i, j, n, err, numop;
    struct stat sbuf;
    const struct aiocb *aiolist[NBUF];
    off_t off = 0;

    if(argc != 3)
    {
        err_quit("usage: rot13 infile outfile");
    }

    if((ifd = open(argv[1], O_RDONLY))<0)
    {
        err_sys("can't open %s", argv[1]);
    }

    if((ofd = open(argv[2], O_WRONLY | O_CREAT| O_TRUNC, FILE_MODE))<0)
    {
        err_sys("can't open %s", argv[2]);
    }

    if(fstat(ifd, &sbuf)<0)
    {
        err_sys("fstat failed");
    }

    for(i=0;i<NBUF;i++)
    {
        bufs[i].op = UNUSED;
        bufs[i].aiocb.aio_buf = bufs[i].data;
        bufs[i].aiocb.aio_sigevent.sigev_notify = SIGEV_NONE;
        aiolist[i] = NULL;
    }

    numop = 0;

    while(1)
    {
        for(i=0;i<NBUF;i++)
        {
            switch (bufs[i].op){
                case UNUSED:
                /* read from the input file if more data remain unread*/
                if(off<sbuf.st_size)
                {
                    bufs[i].op = READ_PENDING;
                    bufs[i].aiocb.aio_fildes = ifd;
                    bufs[i].aiocb.aio_offset = off;
                    off += BSZ;
                    if(off >= sbuf.st_size)
                    {
                        bufs[i].last = 1;
                    }
                    bufs[i].aiocb.aio_nbytes = BSZ;
                    if(aio_read(&bufs[i].aiocb)<0)
                    {
                        err_sys("aio_read failed");
                    }
                    aiolist[i] = &bufs[i].aiocb;
                    numop++;
                }
                break;
                case READ_PENDING:
                if((err= aio_error(&bufs[i].aiocb)) == EINPROGRESS)
                {
                    continue;
                }
                if(err != 0)
                {
                    if(err == -1)
                    {
                        err_sys("aio_error failed");
                    }
                    else{
                        err_exit(err, "read failed");
                    }
                }
                /* a read is complete; translate the buffer and write it*/
                if((n = aio_return(&bufs[i].aiocb))<0)
                {
                    err_sys("aio_return failed");
                }
                if(n!=BSZ && !bufs[i].last)
                {
                    err_quit("short read(%d/%d)", n, BSZ);
                }
                for(j = 0; j<n;j++)
                {
                    bufs[i].data[j] = translate(bufs[i].data[j]);
                }
                bufs[i].aiocb.aio_nbytes = n;
                bufs[i].aiocb.aio_fildes = ofd;
                bufs[i].op = WRITE_PENDING;
                if(aio_write(&bufs[i].aiocb)<0)
                {
                    err_sys("aio_write failed");
                }
                break;
                case WRITE_PENDING:
                if((err = aio_error(&bufs[i].aiocb)) == EINPROGRESS)
                {
                    continue;
                }
                if(err != 0)
                {
                    if(err != -1)
                    {
                        err_sys("aio_error failed");
                    }
                    else{
                        err_exit(err, "aio_write failed");
                    }
                }
                /* a write is complete , mark the buffer is unused*/
                if((n = aio_return(&bufs[i].aiocb))<0)
                {
                    err_sys("aio_return failed");
                }

                if(n != bufs[i].aiocb.aio_offset)
                {
                    err_quit("short write (%d/%d)", n, BSZ);
                }
                aiolist[i] = NULL;
                bufs[i].op = UNUSED;
                numop--;
                break;
            }
        }
        if(numop == 0)
        {
            if(off >= sbuf.st_size)
            {
                break;
            }
        }
        else{
            if(aio_suspend(aiolist, NBUF, NULL)<0)
            {
                err_sys("aio_suspend failed");
            }
        }
    }

    bufs[0].aiocb.aio_fildes = ofd;
    if(aio_fsync(O_SYNC, &bufs[i].aiocb)<0)
    {
        err_sys("aio_fsycn failed");
    }
    exit(0);

}

```

这里使用了8个缓冲区，因此可以有8个异步I/O请求。将一个文件的内容经过一个变换，存储到另一个文件中去。

## 函数readv和writev

`readv`和`writev`函数用来在一次函数调用中读写多个非连续缓冲区：

```c
#include<sys/uio.h>
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
//返回值：已读或已写的字节数，若出错，返回-1

```

函数的第二个参数是指向`iovec`结构数组的一个指针：

```c
struct iovec{
	void *iov_base; /*starting address of buffer*/
	size_t iov_len; /*size of buffer*/
};

```

`iov`数组中元素数由`iovec`指定。下图显示了这两个函数的参数和`iovec`结构的关系：

![iovec](https://s2.ax1x.com/2019/10/01/uUgX40.png)

`writev`函数从缓冲区中聚集输出数据顺序是：`iov[0], iov[1], ...iov[iovcent-1]`。返回输出的总字节数，通常等于所以缓冲区长度之和。

`readv`函数则将读入的数据按照相同顺序散步到缓冲区。`readv`总是先填满一个再填写下一个。`readv`返回读到的字节总数。如果遇到文件末尾，无数据可读，则返回0。

## 函数readn和writen

管道、FIFO以及某些设备（特别是网络和终端）有下列性质：

1. 一次`read`操作返回的数据可能少于所要求的数据，即使还未达到文件的末尾也可能出现这种情况。这不是错误，应该继续读该设备。
2. 一次`write`操作的返回值也可能少于指定输出的字节数。这也可能是某种原因造成的，例如内核输出缓冲区变满。这也不是错误，应该继续写余下数据。

通常在读写一个管道、网络设备或终端时，需要考虑这些特性。下面的两个函数功能分别是读写指定的N字节数据，并处理返回值小于要求值的情况。这两个函数只是按需要多次调用`read`和`write`直至读写了N字节数据。

```c
#include"apue.h"
ssize_t readn(int fd, void *buf, size_t nbytes);
ssize_t writen(int fd, void *buf, size_t nbytes);
//函数返回值：读写字节数，如果出错返回-1

```

两个函数的实现：

```c
#include"apue.h"
ssize_t readn(int fd, void *ptr, size_t n)
{
    size_t nleft;
    ssize_t nread;
    nleft = n;
    while(nleft > 0)
    {
        if((nread = read(fd, ptr, nleft))<0)
        {
            if(n == nleft) /*error return*/
                return -1;
            else{ /* error return amount read so far*/
                break;
            }
        }
        else{
            if(nread == 0)
            {
                break; /*EOF*/
            }
        }
        nleft -= nread;
        ptr = (void*)((char*)ptr + nread);
    }
    return n-nleft;
}

ssize_t writen(int fd, const void *ptr, size_t n)
{
    size_t nleft;
    ssize_t nwritten;

    nleft = n;
    while(nleft>0)
    {
        if((nwritten = write(fd, ptr, nleft))<0)
        {
            if(n == nleft)
            {
                return -1;
            }
            else{
                break;
            }
        }
        else{
            if(nwritten == 0)
            {
                break;
            }
        }
        ptr = (void *)((char*)ptr+nwritten);
    }
    return n-nleft;
}

```

## 存储映射I/O

存储映射I/O能将一个磁盘文件映射到存储空间的一个缓冲区上，于是当从缓冲区中取数据时，就相当于读文件的相应字节，将数据存入缓冲区时，相应字节就自动写入文件。这样可以在不使用`read`和`write`的情况下执行I/O。

为了使用该功能，应该先告诉内核将一个给定文件映射到一个存储区域中。这由`mmap`函数实现：

```c
#include<sys/mman.h>
void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
//成功返回映射区起始地址，出错返回MAP_FAILED

```

`addr`用于指定映射区域的起始地址。通常将其设置为0，表示由系统选择该存储映射区的起始地址。

`fd`指定要被映射文件的描述符。在文件映射到地址空间之前必须先打开文件。`len`参数是映射的字节数。`off`是要映射字节在文件中的起始偏移量。

`port`参数指定了映射存储区的保护要求：

| `port`       | 说明           |
| ------------ | -------------- |
| `PROT_READ`  | 映射区可读     |
| `PROT_WRITE` | 映射区可写     |
| `PROT_EXEC`  | 映射区可执行   |
| `PROT_NONE`  | 映射区不可访问 |

可将`prot`参数指定为`PROT_NONE`，也可指定为`PROT_READ`  、`PROT_WRITE`和`PROT_EXEC`三者的按位或。对存储映射区的保护要求不能超过文件`open`模式访问权限。

存储映射区的实现细节见下图：

![mmap](https://s2.ax1x.com/2019/10/01/uUgx3T.png)

其中起始地址是`mmap`返回值，映射存储区位于堆和栈之间。

`flag`参数影响存储映射区的多种属性：

| `MAP_FIXED`   | 返回值必须等于`addr`。因为不利于移植，一般不建议使用。       |
| ------------- | ------------------------------------------------------------ |
| `MAP_SHARED`  | 此标志指定存储操作修改映射文件，即存储操作相当于对该文件的`write`。必须指定本标志或下一个标志，但不能同时指定这两个标志。 |
| `MAP_PRIVATE` | 本标志说明，对映射区的存储操作导致创建该文件的一个私有副本，所以后来对该映射区的引用都是引用该副本。 |

`off`的值和`addr`的值通常被要求是系统虚拟存储页长度的倍数。虚拟存储页长可用带参数`_SC_PAGESSIZE`或`_SC_PAGE_SIZE`的`sysconf`函数得到。`off`和`addr`常常指定为0，所以这种要求一般不重要。

当映射区长度不是页长整数倍时，例如当文件长度为12字节，系统页长512字节，则系统通常提供512字节的映射区，其中后500字节被设置为0。可以修改后面这500字节，但不会体现到文件中。不能使用`mmap`将数据添加到文件中，必须先加长文件。`

与映射区相关的信号有`SIGSEGV`和`SIGBUS`。信号`SIGSEGV`通常用于指示进程试图访问对它不可用的存储区。如储存区是只读的，当向其写时，会产生此信号。如果映射区的某个部分在访问时已经不存在，则产生`SIGBUS`信号。如用文件长度映射了一个文件，但在映射前，另一个进程已经将该文件截断，此时如果进程试图访问对应于该文件已截去部分的映射区，将会收到`SIGBUS`信号。

子进程能够通过`fork`继承存储映射区，但是不能通过`exec`继承存储映射区。

`mprotect`函数可以更改一个现有映射的权限：

```c
#include<sys/mman.h>
int mprotect(void *addr, size_t len, int prot);
//成功返回0，出错返回-1

```

`prot`的合法值与`mmap`中一样。如果修改页是通过`MAP_SHARED`标志映射到地址空间的，那么修改不会立即写回到文件。

如果共享映射的页已被修改，那么可以调用`msync`将该页冲洗到被映射的文件中：

```c
#include<sys/mman.h>
int msync(void *addr, size_t len, int flags);
//成功返回0，出错返回-1

```

`flags`参数使我们对如何冲洗存储区有某种程度的控制。

| `MS_ASYNC`      | 简单的调试要写的页                                           |
| --------------- | ------------------------------------------------------------ |
| `MS_SYNC`       | 在返回之前等待写操作完成。与上面的必须指定一个。             |
| `MS_INVALIDATE` | 可选标志，允许我们通知操作系统丢弃那些与底层存储器没有同步的页。 |

当进程终止时，会自动解除存储映射区的映射，或者直接调用`munmap`函数来解除映射区。关闭映射存储区时使用的文件描述符并不解除映射区：

```c
#include<sys/mman.h>
int munmap(void *addr, size_t len);
//成功返回0，出错返回-1

```

`munmap`并不影响被映射的对象，即调用`munmap`并不会使映射区的内容写到磁盘。对于`MAP_SHARED`何时写到磁盘取决于内核调度算法。而对于`MAP_PRIVATE`存储区的修改会被丢弃。

实例：用存储映射I/O实现文件拷贝

```c
#include"apue.h"
#include<fcntl.h>
#include<sys/mman.h>

#define COPYINCR (1024*1024*1024) /* 1GB*/

int main(int argc, char *argv[])
{
    int fdin, fdout;
    void *src, *dst;
    size_t copyze;
    struct stat sbuf;
    off_t fsz = 0;
    if(argc != 3)
    {
        err_quit("usage: %s<fromfile> <tofile>", argv[0]);
    }

    if((fdin = open(argv[1], O_RDONLY))<0)
    {
        err_sys("can't open %s for read",argv[1]);
    }

    if((fdout = open(argv[2], O_RDWR | O_CREAT | O_TRUNC, FILE_MODE))<0)
    {
        err_sys("can't creat %s for writing", argv[2]);
    }

    if(fstat(fdin, &sbuf)<0)
    {
        err_sys("fstat error");
    }

    if(ftruncate(fdout, sbuf.st_size)<0)/*set len of output file*/
    {
        err_sys("ftruncate error");
    }

    while(fsz<sbuf.st_size)
    {
        if((sbuf.st_size-fsz)>COPYINCR)
        {
            copyze = COPYINCR;
        }
        else{
            copyze = sbuf.st_size - fsz;
        }

        if((src = mmap(0, copyze, PROT_READ, MAP_SHARED, fdin, fsz)) == MAP_FAILED)
        {
            err_sys("mmap error for input");
        }

        if((dst = mmap(0, copyze, PROT_READ | PROT_WRITE, MAP_SHARED, fdout, fsz)) == MAP_FAILED)
        {
            err_sys("mmap error for output");
        }

        memcpy(dst, src, copyze);
        munmap(src, copyze);
        munmap(dst, copyze);
        fsz += copyze;
    }
    exit(0);
}

```



# 第十五章 进程间通信（IPC）

## 管道

管道是UNIX最古老的IPC形式。其具有两个局限性：

1. 是半双工的，即数据只能在一个方向上流动。
2. 管道只能在公有祖先的两个进程之间使用。通常一个管道由一个进程创建，在进程调用`fork`之后，这个管道即可在父进程与子进程之间使用。

每当在管道中键入一个命令序列，当`shell`执行时，`shell`都会为每一条命令创建一个进程，然后用管道将前一条命令的标准输出与后一条命令的标准输入相连。

管道由`pipe`函数创建：

```c
#include<unistd.h>
int pipe(int fd[2]);
//成功返回0，出错返回-1

```

`fd`返回两个文件描述符：`fd[0]`为读而打开，`fd[1]`为写而打开。`fd[1]`的输出是`fd[0]`的输入。

下图展示了这种结果：

![pipe](https://s2.ax1x.com/2019/10/01/uUgvCV.png)

`fstat`函数对管道的每一端都返回一个`FIFO`类型文件描述符。可以使用`S_ISFIFO`宏来测试管道。

单个进程中的管道基本没有用，通常是进程调用`pipe`后，接着调用`fork`，从而创建父进程到子进程的管道，如下图显示：

![pipe](https://s2.ax1x.com/2019/10/01/uU2SvF.png)

`fork`之后做什么取决于我们想要的数据流向，对于从父进程到子进程的管道，父进程关闭`fd[0]`，子进程关闭`fd[1]`。于是得到下图结果：

![uU29u4.png](https://s2.ax1x.com/2019/10/01/uU29u4.png)

当管道一端被关闭时，下面两条规则起作用：

1. 当`read`一个写端已经被关闭的管道时，在所有数据被读完后，`read`返回0，表示文件结束。
2. 如果`write`一个读端已经关闭的管道，则产生信号`SIGPIPE`。如果忽略信号或者捕捉该信号并从处理程序返回，则`write`返回-1，`erron`设置为`EPIPE`。

写管道时，`PIPE_BUF`规定了内核管道缓冲区大小。应该保证写的数据小于该值。`pathconf`或`fpathconf`获取该值。

例程：创建从父进程到子进程的管道

```c
#include"apue.h"

int main()
{
    int n;
    int fd[20];
    int err;
    pid_t pid;
    char line[MAXLINE];

    if((err = pipe(fd)) < 0)
    {
        err_sys("pipe error");
    }

    if((pid = fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid == 0)
    {
        close(fd[1]);
        n = read(fd[0], line, MAXLINE);
        write(STDOUT_FILENO, line, n);
    }
    else{
        close(fd[0]);
        write(fd[1], "hello world\n", 12);
    }
    exit(0);
}

```

例程：每次一页的显示已产生的输出，分页功能直接调用已有程序即可，我们只需要向分页程序传递输入数据即可：

```
#include"apue.h"
#include<sys/wait.h>

#define DEF_PAGER "/bin/more" /*default prger program*/

int main(int argc, char *argv[])
{
    int n, fd[2];
    pid_t pid;
    char *pager, *argv0;
    char line[MAXLINE];
    FILE *fp;

    if(argc != 2)
    {
        err_quit("usage: a.out <pathneam>");
    }

    if((fp = fopen(argv[1], "r")) == NULL)
    {
        err_sys("can't open %s", argv[1]);
    }

    if(pipe(fd)<0)
    {
        err_sys("pipe error");
    }

    if((pid=fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid > 0)
    {
        close(fd[0]);

        while(fgets(line, MAXLINE, fp) != NULL)
        {
            n = strlen(line);
            if(write(fd[1], line, n)!=n)
            {
                err_sys("write error to pipe");
            }
        }
        if(ferror(fp))
        {
            err_sys("fgets error");
        }

        close(fd[1]);

        if(waitpid(pid, NULL, 0)<0)
        {
            err_sys("waitpid error");
        }
        exit(0);
    }
    else{
        close(fd[1]);
        if(fd[0] != STDIN_FILENO)
        {
            if(dup2(fd[0], STDIN_FILENO)!=STDIN_FILENO)
            {
                err_sys("dup2 error to stdin");
            }
        }

        /* get arguments for exec*/

        if((pager = getenv("PAGER"))==NULL)
        {
            pager = DEF_PAGER;
        }
        if((argv0 = strrchr(pager, '/')) != NULL)
        {
            argv0++;
        }
        else{
            argv0 = pager;
        }

        if(execl(pager, argv0, (char*)0)<0)
        {
            err_sys("execl error for %s", pager);
        }
    }
    exit(0);
}

```

例程：利用管道实现`TELL_xxx`和`WAIT_xxx`的进程间同步。

```c
#include"apue.h"

static int fd1[2], fd2[2];

void TELL_WAIT()
{
    if(pipe(fd1)<0)
    {
        err_sys("pipe error");
    }
    if(pipe(fd2)<0)
    {
        err_sys("pipe error");
    }
}

void TELL_CHILD(pid_t pid)
{
    if(write(fd1[1], "c", 1)!=1)
    {
        err_sys("write error to pipe");
    }
}

void TELL_PARENT(pid_t pid)
{
    if(write(fd2[1], "c", 1)!=1)
    {
        err_sys("write error to pipe");
    }
}

void WAIT_PARENT()
{
    char c;
    if(read(fd1[0], &c, 1)!=1)
    {
        err_sys("read error form pipe");
    }
    if(c != 'c')
    {
        err_sys("WAIT_PARENT: incorrect data");
    }
}

void WAIT_CHILD()
{
    char c;
    if(read(fd2[0], &c, 1)!=1)
    {
        err_sys("read error from pipe");
    }
    if(c != 'c')
    {
        err_sys("WAIT_CHILD: incorrect data");
    }
}

```



## 函数popen和pclose

```c
#include<stdio.h>
FILE *popen(const char *cmdstring, const char *type);
//若成功，返回文件指针，若出错，返回NULL

int pclose(FILE *fp);
//成功返回cmdstring的终止状态，出错返回-1;

```

函数`popen`先执行`fork`，然后调用`exec`执行`cmdstring`，并返回一个标准I/O文件指针。如果`type`是`”r“`，则文件指针连接到`cmdstring`的标准输出，如果`type`是`”w“`，则文件指针连接到`cmdstring`的标准输入。如下图：

![popen](https://s2.ax1x.com/2019/10/01/uU2Pb9.png)

`popen`与`fopen`可以进行类别。

`pclose`函数关闭标准I/O流，等待命令终止，然后返回`shell`的终止状态，如果`shell`不能被执行，则`pclose`返回的终止状态与`shell`已执行`exit(127)`一致。

例程：利用`popen`实现上一节最后分页的例程。

```c
#include"apue.h"
#include<sys/wait.h>

#define PAGER "${PAGER:-more}" /*environment variable, or default*/

int main(int argc, char *argv[])
{
    char line[MAXLINE];

    FILE *fpin, *fpout;
    if(argc != 2)
    {
        err_quit("usage: a.out <pathneam>");
    }

    if((fpin = fopen(argv[1], "r")) == NULL)
    {
        err_sys("can't open %s", argv[1]);
    }

    if((fpout = popen(PAGER, "w")) == NULL)
    {
        err_sys("popen error");
    }

    /* copy argv[1] to pager*/

    while(fgets(line, MAXLINE, fpin)!=NULL)
    {
        if(fputs(line, fpout) == EOF)
        {
            err_sys("fputs error to pipe");
        }
    }

    if(ferror(fpin))
    {
        err_sys("fgets error");
    }
    if(pclose(fpout) == -1)
    {
        err_sys("pclose error");
    }
    exit(0);
}

```

实例：函数popen和pclose的实现。

```c
#include"apue.h"
#include<errno.h>
#include<fcntl.h>
#include<sys/wait.h>

static pid_t *childpid = nullptr;

static int maxfd;

FILE *popen(const char *cmdstring, const char *type)
{
    int i, pfd[2];
    pid_t pid;
    FILE *fp;

    /* only allow "r" or "w"*/
    if((type[0]!='r' && type[0]!='w')||type[1]!=0)
    {
        errno = EINVAL;
        return nullptr;
    }

    if(childpid == nullptr) /* first time through*/
    {
        /* allocate zeroed out array for child pids*/
        maxfd = open_max();
        if((childpid = (pid_t*)calloc(maxfd, sizeof(pid_t))) == NULL)
        {
            return nullptr;
        }
    }

    if(pipe(pfd)<0)
    {
        return nullptr;
    }

    if(pfd[0] >= maxfd || pfd[1]>=maxfd)
    {
        close(pfd[0]);
        close(pfd[1]);
        errno = EMFILE;
        return nullptr;
    }

    if((pid=fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid == 0)
    {
        if(type[0] == 'r')
        {
            close(pfd[0]);
            if(pfd[1] != STDOUT_FILENO)
            {
                dup2(pfd[1], STDOUT_FILENO);
                close(pfd[1]);
            }
        }
        else{
            close(pfd[1]);
            if(pfd[0]!=STDIN_FILENO)
            {
                dup2(pfd[0], STDIN_FILENO);
                close(pfd[0]);
            }
        }

        /*close all descriptors in childpid[]*/
        for(i = 0; i<maxfd; i++)
        {
            if(childpid[i] > 0)
            {
                close(i);
            }
        }

        execl("/bin/sh", "sh", "-c", cmdstring, (char*)0);
        _exit(127);
    }

    /* parent contine ...*/

    if(type[0] == 'r')
    {
        close(pfd[1]);
        if((fp = fdopen(pfd[0], type)) == NULL)
        {
            return nullptr;
        }
    }
    else{
        close(pfd[0]);
        if((fp = fdopen(pfd[1], type)) == NULL)
        {
            return nullptr;
        }
    }
    childpid[fileno(fp)] = pid;
    return fp;
}

int pclose(FILE *fp)
{
    int fd, stat;
    pid_t pid;

    if(childpid == NULL)
    {
        errno = EINVAL;
        return -1;
    }

    fd = fileno(fp);
    if(fd >= maxfd)
    {
        errno = EINVAL;
        return -1;
    }

    if((pid = childpid[fd]) == 0)
    {
        errno = EINVAL;
        return -1;
    }

    childpid[fd] = 0;

    if(fclose(fp) == EOF)
    {
        return -1;
    }

    while(waitpid(pid, &stat, 0)<0)
    {
        if(errno != EINTR)
        {
            return -1;
        }
    }
    return stat;
}

```

其中`open_max`对于代码为：

```c
#include"apue.h"
#include<errno.h>
#include<limits.h>

#ifdef OPEN_MAX
static long openmax = OPEN_MAX;
#else
static long openmax = 0;
#endif

#define OPEN_MAX_GUESS 256

long open_max()
{
    if(openmax == 0)
    {
        errno = 0;
    }
    if((openmax = sysconf(_SC_OPEN_MAX))<0)
    {
        if(errno == 0)
        {
            openmax = OPEN_MAX_GUESS;
        }
        else{
            err_sys("sysconf error for _SC_OPEN_MAX");
        }
    }
    return openmax;
}

```

`open_max`返回可以打开文件的最大个数的近似值。

`POSIX.1`要求`popen`关闭那些以前调用`popen`打开的、现在仍然在子进程中打开着的I/O流。

实例：向标准输出写一个提示，然后从标准输入读一行。使用`popen`生成新的进程来处理输入，再将结果返回到原进程。如下图：

![popen](https://s2.ax1x.com/2019/10/01/uU2kU1.png)

```c
#include"apue.h"
#include<ctype.h>

int main(void)
{
    int c;
    while((c = getchar()) != EOF)
    {
        if(isupper(c))
        {
            c = tolower(c);
        }
        if(putchar(c) == EOF)
        {
            err_sys("output error");
        }
        if(c = '\n')
        {
            fflush(stdout);
        }
    }
    exit(0);
}

```

输入处理程序，将输入字符全部变成小写字符。编译该文件生成`myuclc`可执行文件。

```c
#include"apue.h"
#include<sys/wait.h>

int main(void)
{
    char line[MAXLINE];
    FILE *fpin;

    if((fpin = popen("./myuclc", "r")) == NULL)
    {
        err_sys("popen error");
    }
    while(1)
    {
        fputs("prompt>", stdout);
        fflush(stdout);

        if(fgets(line, MAXLINE, fpin) == NULL)
        {
            break;
        }

        if(fputs(line, stdout) == EOF)
        {
            err_sys("fputs error to pipe");
        }
        
    }
    if(pclose(fpin) == -1)
    {
        err_sys("pclose error");
    }

    putchar('\n');
    exit(0);
}

```

## 协同进程

UNIX系统过滤程序从标准输入读取数据，向标准输出写数据。几个过滤程序通常在`shell`管道中线性连接。当一个过滤程序即产生某个过滤程序的输入，又读取过滤程序的输出时，它就变成了协同进程。、

实例：输入两个数，传递给协同进程加和，再传回原进程输出：

```c
#include"apue.h"
int main(void)
{
    int a1, a2, n;

    char line[MAXLINE];
    while((n = read(STDIN_FILENO, line, MAXLINE))>0)
    {
        line[n] = 0;
        if(sscanf(line, "%d%d", &a1, &a2) == 2)
        {
            sprintf(line, "%d\n", a1+a2);
            n = strlen(line);
            if(write(STDOUT_FILENO, line, n)!=n)
            {
                err_sys("write error");
            }
        }
        else{
            if(write(STDOUT_FILENO, "invalid args\n", 13)!=13)
            {
                err_sys("printf error");
            }
        }
    }
    exit(0);
}

```

加和程序编译成`add`。

```c
#include"apue.h"

static void sig_pipe(int); /*siganel handler*/

int main(void)
{
    int n, fd1[2], fd2[2];
    pid_t pid;
    char line[MAXLINE];

    if(signal(SIGPIPE, sig_pipe) == SIG_ERR)
    {
        err_sys("signal error");
    }

    if(pipe(fd1)<0 || pipe(fd2)<0)
    {
        err_sys("pipe error");
    }

    if((pid = fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid>0)
    {
        close(fd1[0]);
        close(fd2[1]);

        while(fgets(line, MAXLINE, stdin) != NULL)
        {
            n = strlen(line);
            if(write(fd1[1], line, n)!=n)
            {
                err_sys("write error to pipe");
            }

            if((n = read(fd2[0], line, MAXLINE))<0)
            {
                err_sys("read error from pipe");
            }
            else if(n==0)
            {
                err_msg("child closed pipe");
            }

            line[n] = 0;
            if(fputs(line, stdout) == EOF)
            {
                err_sys("fputs error");
            }
        }
        if(ferror(stdin))
        {
            err_sys("fgets error on stdin");
        }
    }
    else{
        close(fd1[1]);
        close(fd2[0]);

        if(fd1[0] != STDIN_FILENO)
        {
            if(dup2(fd1[0], STDIN_FILENO) != STDIN_FILENO)
            {
                err_sys("dup2 error to stdin");
            }
            close(fd1[0]);
        }

        if(fd2[1] != STDOUT_FILENO)
        {
            if(dup2(fd2[1], STDOUT_FILENO) != STDOUT_FILENO)
            {
                err_sys("dup2 error to stdout");
            }
            close(fd2[1]);
        }

        if(execl("./add", "add", (char*)0)<0)
        {
            err_sys("execl error");
        }
    }
    exit(0);
}

static void sig_pipe(int signo)
{
    printf("SIGPIPE caught\n");
    exit(0);
}

```

这里`add`使用了底层调用，当使用标准`I/O`时，即改成如下形式时：

```c
#include"apue.h"
int main(void)
{
    int a1, a2, n;

    char line[MAXLINE];
    while(fgets(line, MAXLINE, stdin)!=NULL)
    {
        if(sscanf(line, "%d%d", &a1, &a2) == 2)
        {
            if(printf("%d\n", a1+a2) == EOF)
            {
                err_sys("printf error");
            }
        }
        else{
            if(printf("invalid args\n") == EOF)
            {
                err_sys("printf error");
            }
        }
    }
    exit(0);
}

```

上述程序将无法正常执行。这是因为默认缓冲的原因。对于`add`程序，其标准输入输出都是管道，此时，默认是全缓冲，因此`fgets`函数将会发生阻塞，不会为父进程传递数据，将会导致父进程的`read`发生阻塞，于是产生死锁。

## FIFO

`FIFO`被称为命名管道。未命名管道只能在两个相关进程之间使用，而命名管道可以使不相关的进程也能交换数据。

创建FIFO文件：

```c
#include<sys/stat.h>
int mkfifo(const char *path, mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
//成功返回，错误返回-

```

其中`mode`参数与`open`的相同。当创建了一个`FIFO`时，要用`open`来打开它。当`open`一个`FIFO`时，非阻塞标志(`O_NONBLOCK`)将会导致：

1. 一般情况下（不指定`O_NONBLCOK`），只读`open`要阻塞到某个其他进程为写而打开这个`FIFO`为止。同样，只写`open`要阻塞到某个其他进程为读打开为止。
2. 如果指定了`O_NONBLOCK`，则只读`open`立即返回。但如果没有进程为读而打开一个`FIFO`，那么只写`open`将返回-1，并设置`errno`为`ENXIO`。

类似管道，当`write`一个尚无进程为读而打开的`FIFO`，则产生信号`SIGPIPE`。若某个`FIFO`最后一个写进程关闭了该`FIFO`，则为`FIFO`的读进程产生一个文件结束标识符。

`FIFO`有以下两个用途：

1. `shell`命令使用`FIFO`将数据从一条管道传送到另一条，无需创建中间文件。
2. 客户进程-服务器进程应用程序中，`FIFO`用作汇聚点，在客户进程和服务器进程之间传递数据。

实例：用`FIFO`复制输出流。

构建一个如下处理数据结构：

![ex](https://s2.ax1x.com/2019/10/01/uU2A4x.png)

这里，由于`prog`的输出要到两个地方，因此我们可以利用管道和`tee`程序。`tee`程序可以将标准输入同时复制到标注输出和其命令行中命名的文件中，这里我们的命名文件是一个管道。于是构建出下图：

![tee](https://s2.ax1x.com/2019/10/01/uU2VC6.png)

`shell`脚本为：

```shell
mkfifo fifo1
prog3 < fifo1 &
prog1 < infile | tee fifo1 | prog2

```

实例：使用`FIFO`进行客户进程-服务器进程通信

一个服务器，其与横多客户进程相关，每个客户进程都可将其请求写到一个该服务器进程创建的众所周知的`FIFO`中。如下图结果:

![server-client](https://s2.ax1x.com/2019/10/01/uU2Z8K.png)

由于该`FIFO`有多个写进程，所以客户进程发送给服务器进程的请求长度要小于`PIPE_BUF`字节。这种类型的结构存在一个问题是：服务器进程如何将回答发送给各个客户进程。一种解决方案是，每个客户进程都在其请求中包含它们的进程ID。然后服务器为每个客户进程创建一个`FIFO`，所使用的路径名是以客户进程ID为基础的。下图显示了这种安排：

![server-cli](https://s2.ax1x.com/2019/10/01/uU2egO.png)

此时服务器进程不知道客户进程是否崩溃终止，这就使得客户进程`FIFO`专用`FIFO`会遗留到文件系统中。同时必须捕捉`SIGPIPE`信号。同时，按照上图结果，如果服务器以只读打开`FIFO`时，当客户进程从1到0时，服务器进程将读到一个文件结束标志，为了避免处理这种情况，可以以读写的方式打开。

## XSI IPC

有三种称作XSI IPC的IPC：消息队列，信号量以及共享存储器。该节介绍其相同点。

### 标识符和键

每个内核IPC结构都用一个非负整数的标识符加以引用。如向一个消息队列发送消息或者从一个消息队列取消息，只需要知道其队列标识符。IPC标识符不是小的整数。当一个IPC结构被创建，然后又被删除时，于这种结构相关的标识符连续加1，直至达到一个整数的最大正值，然后又会转到0。

每个IPC对象都与一个键（key）相关联，将这个键作为该对象的外部名。无论何时创建IPC结构，都应指定一个键。键的基本数据类型是`key_t`。这个键由内核变换成标识符。

有多种方法使客户进程和服务器进程在同一IPC结构上汇聚：

1. 服务器进程可以指定键`IPC_PRIVATE`创建一个新IPC结构，将返回的标识符存放在某处（如一个文件中）以便客户进程读取。键`IPC_PRIVATE`保证服务器进程创建一个新IPC结构。缺点是文件系统需要服务器进程将整型标识符写到文件中，此后客户端又要读这个文件取次标识符。

2. 可以在一个公用头文件中定义一个客户进程与服务器进程都认可的键。然后服务器进程指定此键创建一个新的IPC结构。该方法的问题是改键可能已经与一个IPC结构相结合，此时`get`函数出错返回。服务器必须处理这一错误，删除已存在的IPC结构，然后试着再创建它。

3. 客户进程和服务器进程认同一个路径名和项目ID（0~255），接着调用`ftok`将两个值变换为一个键。然后在上一个方法中使用该键。`ftok`提供服务是由一个路径名和项目ID产生一个`key`。

   ```c
   #include<sys/ipc.h>
   key_t ftok(const char *path, int id);
   //成功返回键，否则返回（key_t)-1
   
   ```

   

三个`get`函数（`msgget`、`semget`和`shmget`）都有两个类似的参数：一个`key`和一个整型`flag`。在创建新的`IPC`结构时，如果`key`是`IPC_PRIVATE`或者和当前某种类型的IPC结构无关，则需指明`flag`的`IPC_CREAT`标志位。为了引用一个现有IPC，`key`必须等于IPC创建时指明的`key`值。并且`IPC_CREAT`必须不被指明。决不能使用`IPC_PRIVATE`作为键来引用一个现有IPC，因为这个特殊键总是创建一个新IPC。

如果希望创建一个新的IPC结构，而且要保证没有引用具有同一标识符一个现有IPC结构，那么必须在`flag`中同时指定`IPC_CREAT`和`IPC_EXCL`位。这样做后，如果IPC结果已经存在就会造成出错，返回`EEXIST	`。

### 权限结构

`XSI IPC`为每一个IPC结构关联了一个`ipc_perm`结构。该结构规定了权限和所有者：

```c
struct ipc_perm{
	uid_t uid; /*owner's effective user id*/
    gid_t gid; /* owner's effective group id*/
    uid_t cuid; /*creator's effective user id*/
    gid_t cgid; /*creator's effective group id*/
    mode_t mode; /*access modes*/
    ...
}

```

在创建IPC结构时，对所以字段都赋初值。以后可以调用`msgctl`、`semctl`或`shmctl`修改`uid`、`gid`和`mode`字段。为了修改这些值，调用进程必须是IPC结构的创建者或者超级用户。修改这些字段类似于对文件调用`chown`和`chmod`。

`mode`字段值类似于第四章中新文件和目录所有权中的`mode`。当是对于任何IPC结构都不存在执行权限。另外，消息队列和共享存储使用术语“读”和“写”，而信号量则使用术语“读”和“更改”。下表定义了每种IPC权限：

| 权限           | 位   |
| -------------- | ---- |
| 用户读         | 0400 |
| 用户写（更改） | 0200 |
| 组读           | 0040 |
| 组写（更改）   | 0020 |
| 其他读         | 0004 |
| 其他写（更改） | 0002 |



### 优点和缺点

XSI IPC的一个基本问题是： IPC结构是在系统范围内起作用的，没有引用计数。如果进程创建一个消息队列，并且在该队列中放入几则消息，然后终止，那么该消息队列及其内容不会被删除。他们会一直留在系统中直到发生下列动作为止：

1. 由某个进程调用`msgrcv`或`msgctl`读消息或者删除消息队列。
2. 或某个进程执行`ipcrm`命令删除消息队列。
3. 或正在自举的系统删除消息队列。

与管道相比复杂很多。

XSI IPC的另一个问题是：这些IPC结构在文件系统中没有名字。

因为这些形式的IPC不使用文件描述符，所以不能对他们使用多了转接I/O函数，这使得它很难一次使用一个以上IPC结构。

## 消息队列

消息队列是消息链接表，存储在内核中，由消息队列标识符标识。

`msgget`由于创建一个新队列或者打开一个现有队列。`msgsnd`将新消息添加到队列尾端。每个消息包含一个正的长整型类型的资源、一个非负长度以及实际数据字节数（对应长度），所有这些都在将消息添加到队列时传送给`msgsnd`。`msgrcv`用于从队列中取消息。不必按照先进先出取，而是可以按照消息的类型字段取消息。

每个队列都有一个`msqid_ds`结构与其相关联：

```c
struct msqid_ds{
    struct ipc_perm msg_perm;
    msgqnum_t msg_qnum; /*# of message on queue*/
    msglen_t msg_qbytes; /* max # of bytes on queue*/
    pid_t msg_lspid; /* pid of last msgsnd()*/
    pid_t msg_lrpid; /* pid of last msgrcv()*/
    time_t msg_stime; /* last-,sgsnd() time*/
    time_t msg_rtime; /* last-msgrcv() time*/
    time_t msg_ctime; /* last-change time*/
    ...
};

```

此结构定义了队列当前状态。

`msgget`打开一个现有队列或者创建一个新队列：

```c
#include<sys/msg.h>

int msgget(key_t key, int flag);
//返回值，成功返回消息队列ID，出错返回-1

```

上一节说明了如何创建新的队列，在创建新队列时，要初始化`msqid_ds`结构的下列成员：

1. `ipc_perm`结构按照上一节所述初始化。该结构中的`mode`按照`flag`中响应权限位设置。这些权限由上一节最后一个表指定。
2. `msg_qnum`、`msg_lspid`、`msg_lrpid`、`msg_stime`和`msg_rtime`都设置为0。
3. `msg_ctime`设置为当前时间。
4. `msg_qbytes`设置为体统限制值。

`msgctl`函数对队列执行多种操作。它和另外两个与信号量和共享存储有关的函数（`semctl`和`shmctl`）都是XSI IPC的类似于`ioctl`的函数（即垃圾桶函数）。

```c
#include<sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
//成功返回0，出错返回-1

```

`cmd`参数指定对`msqid`指定的队列要执行的命令：

| `IPC_STAT` | 获取次队列的`msqid_ds`结构，将其存储在`buf`中。              |
| ---------- | ------------------------------------------------------------ |
| `IPC_SET`  | 将字段`msg_perm.uid`、`msg_perm.gid`、`msg_perm.mode`和`msg_qbytes`从`buf`指向的结构复制到与这个队列相关的`msqid_ds`结构中。此命令只能由两种进程执行：一种是其有效用户ID等于`msg_perm.cuid`或`msg_perm.uid`，另一种是具有超级用户特权的进程。只有超级用户才能增加`msg_qbytes`的值。 |
| `IPC_RMID` | 从队列中删除该消息队列以及仍在该队列的所有数据。这种删除立即生效。仍在使用该消息队列的其他进程在下一个试图对此队列进行操作时，将得到`EIDRM`错误。此命令只能由两种进程执行：一种是其有效用户ID等于`msg_perm.cuid`或`msg_perm.uid`，另一种是具有超级用户特权的进程。 |

这三条命令也可用于信号量和共享存储。

调用`msgsnd`将数据放到消息队列中：

```c
#include<sys/msg.h>
int msgsnd(int msqid, const void *ptr, size_t nbytes, int flag);
//成功返回0，出错返回-1

```

每个消息都由3部分组成：一个长整型类型的字段、一个非负的长度（`nbytes`）表示实际数据字节。消息总是放到队列尾端。`ptr`参数指向一个长整型数，它包含正的整型消息类型，其后紧接着的是消息数据（若`nbytes`是0，则无消息）。若发送最长消息是512字节的，则可定义下列结构：

```c
struct mymesg{
    long mtype; /* positive message type*/
    char mtext[512]; /* message data, of length nbytes*/
};

```

`ptr`就是一个指向`mymesg`结构的指针。接收者可以使用消息类型以非先进先出的次序取消息。

参数`flag`的值可以指定为`IPC_NOWAIT`。这类似于文件I/O的非阻塞标志。若队列已满，则指定非阻塞时使得`msgsnd`立即出错返回`EAGAIN`。如果未指定非阻塞，则进程会一直阻塞到下面一个条件满足：

1. 有空间可以容纳要发送的消息。
2. 从系统删除了该队列。此时返回`EIDRM`错误（“标识符被删除”）。
3. 捕捉到一个信号，并从信号处理程序中返回。此时返回`EINTR`错误。

当`msgsnd`返回成功时，消息队列相关的`msqid_ds`结构会随之更新，表明调用进程ID、调用时间已经队列中新增的消息（`msg_qnum`)。

`msgrcv`从队列中取用信息：

```c
#include<sys/msg.h>
ssize_t msgrcv(int msqid, void *ptr, size_t nbytes, long type, int flag);
//成功返回消息部分长度，出错返回-1

```

`ptr`指向一个长整型数（其中存储的是返回的消息类型），其后跟随的是存储实际消息的缓冲区。`ptr`应该与`msgsnd`函数传递的类型一致。`nbytes`指定数据缓冲区的长度，若返回消息长度大于`nbytes`，而且在`flag`中设置了`MSG_NOERROR`位则消息会被截断且不会通知我们。如果没有设置该标准，而消息过长，则出错返回`E2BIG`（消息依然留在队列中）。

参数`type`可以指定想要哪种消息：

| `type == 0` | 返回队列第一个消息                                           |
| ----------- | ------------------------------------------------------------ |
| `type >0`   | 返回队列中消息类型为`type`的第一个消息。                     |
| `type<0`    | 返回队列中消息类型值小于等于`type`绝对值的消息，如果这种消息存在若干个，则取消息类型最小的。 |

可以将`flag`指定为`IPC_NOWAIT`，使操作不阻塞，这样，如果没有所指定类型的消息可用，则`msgrcv`返回-1，`errno`设置为`ENOMSG`。如果未指定`IPC_NOWAIT`，则进程会阻塞到有了指定类型的消息可用，或者从系统删除了此队列（返回-1，`errno`设置为`EIDRM`），或者捕捉到一个信号并从信号返回（返回-1，`errno`设置为`EINTR`）。

`msgrcv`成功执行时，内核会更新与该消息队列相关联的`msgid_ds`结构，以指示调用者的进程ID（`msg_lrpid`）和调用时间（`msg_rtime`），并指示队列中消息数量减少了一个（`msg_qnum`）。

## 信号量

信号量是一个计数器，用于为多个进程提供对共享数据对象的访问（类似于多线程中的锁？）。

为了获得共享资源，进程需要执行下列操作：

（1）测试控制该资源的信号量。

（2）若此信号量的值为正，则进程可以使用该资源。在这种情况下，进程将信号量值减一，表示它使用了一个资源单位。

（3）否则，若此信号量值为0，则进程进入休眠状态，直至信号量值大于0.进程被唤醒，返回第一步。

当进程不再使用由一个信号量控制的共享资源时，该信号值增加1.如果有进程正在休眠等待此信号量，则唤醒他们。

为了正确实现信号量，信号量加一和减一应当都是原子操作。因此信号量通常是在内核中实现的。

常用的信号量形式为二元信号量。它控制单个资源，其初始值为1。但是一般而言，信号量的初值可以是任意一个正值，该值表示有多少个共享资源单位可供共享应用。

XSI信号量由于下面三个原因变得十分复杂：

1. 信号量并非是单个非负值，而必须定义为一个或多个信号量的集合。当创建信号量时，要指定信号集中信号量数值的数量。
2. 信号量的创建（`semget`）是独立于它的初始化（`semctl`）的，这是一个致命缺点，不能够原子的创建一个信号量集合并且对该集合的各个信号量赋初值。
3. 即使没有进程正在使用各种形式的XSI IPC，它们任然是存在的。有点程序在终止时并没有释放已分配给它的信号量，所以我们不得不考虑这种情况。

内核为每个信号量集合维护着一个`semid_ds`结构：

```c
struct semid_ds{
	struct ipc_perm sem_perm;
    unsigned short sem_nsems; /* # of semaphores in set */
    time_t sem_otime; /* last-semop() time*/
    time_t sem_ctime; /* last-change time */
    ...
};

```

每个信号量由一个无名结构表示，至少包含下列成员：

```c
struct{
	unsigned short semval; /* semaphore value, always >= 0*/
    pid_t sempid; /* pid for last operation*/
    unsigned short semncnt; /* # processes awaiting semval>curval*/
    unsigned short semzcnt; /* # processes aeaiting semval == 0 */
};

```

`semget`函数来获取一个信号ID：

```c
#include<sys/sem.h>

int semget(key_t key, int nsems, int flag);
//成功返回信号量ID，出错返回-1

```

前面已经讨论过如何创建一个新IPC。创建新集合时，对`semid_ds`结构的下列成员赋初值：

1. 对`ipc_perm`结构初始化，其中`mode`成员被设置为`flag`中响应权限位。
2. `sem_otime`设置为0。
3. `sem_ctime`设置为当前时间。
4. `sem_nsems`设置为`nsems`。

`nsems`是该集合中信号量数，如果是创建新集合，则必须指定`nsems`，如果是引用现有集合，则将`nsems`指定为0。

`semctl`函数包含了多种信号量操作：

```c
#include<sys/sem.h>
int semctl(int semid, int semnum, int cmd, .../* union semun arg */);

```

第四个参数是可选的，是否使用取决于所请求的命令，如果使用该参数，其类型是`semun`，它是多个命令特定参数的联合（`union`）：

```c
union semun{
	int val; /* forSETVAL */
    struct semid_ds *buf; /* for IPC_STAT and IPC_SET */
    unsigned short *array; /* for GETALL and SETALL*/
};

```

`cmd`参数指定下列10种命令中的一种，其中`semnum`指定该信号量中的一个成员。`semnum`值在0和`nsems-1`之间。

| `IPC_STAT` | 对此信号取`semid_ds`结构并存储在`arg.buf`指向的结构中。      |
| ---------- | ------------------------------------------------------------ |
| `IPC_SET`  | 按`arg.buf`指向结构中的值，设置与此结构相关的结构中的`sem_perm.uid`、`sem_perm.gid`和`sem_perm.cuid`字段，此命令只能由两种进程执行：一种是其有效用户ID等于`sem_perm.cuid`或`sem_perm.uid`，另一种是具有超级用户特权的进程。 |
| `IPC_RMID` | 从系统中删除该信号量集合。这种删除立即发生。仍在使用该消息队列的其他进程在下一个试图对此队列进行操作时，将得到`EIDRM`错误。此命令只能由两种进程执行：一种是其有效用户ID等于`sem_perm.cuid`或`sem_perm.uid`，另一种是具有超级用户特权的进程。 |
| `GETVAL`   | 返回成员`semnum`的`semval`值。                               |
| `SETVAL`   | 设置成员`semnum`的`semval`值，该值由`arg.val`指定。          |
| `GETPID`   | 返回成员`semnum`的`sempid`值。                               |
| `GETNCNT`  | 返回成员`semnum`的`semncnt`值。                              |
| `GETZCNT`  | 返回成员`semnum`的`semzcnt`值。                              |
| `GETALL`   | 取该集合中所以的信号量的值。这些值存储在`arg.array`指向的数组中。 |
| `SETALL`   | 将该集合中所以信号量值设置成`arg.array`指向的数组中的值。    |

除了需要返回值的命令，其他命令若成功返回0，出错设置`errno`并返回-1。

函数`semop`自动执行信号量集合上的操作数组：

```c
#include<sys/sem.h>
int semop(int semid, struct sembuf semoparray[], size_t nops);
//成功返回0， 出错返回-1

```

参数`semoparray`是一个指针，指向由`semnuf`结构表示的信号量操作数组：

```c
struct sembuf{
	unsigned short sem_num; /* number # in set (0, 1, ...., nsems-1) */
    short sem_op; /* operation(negtive, 0, or pasitive) */
    short sem_flag; /*IPC_NOWAIT, SEM_UNDO*/
}

```

参数`nops`规定该数组中操作的数量（元素数）。

对集合中每个成员的操作由响应的`sem_op`值规定。此值可以是负值，0或正值。下面讨论的信号量的`undo`标志，此标志对应于`sem_flg`中的`SEM_UNDO`位。（书中对`SEM_UNDO`解释有错误，根据实际程序发现，其实是否指定该标志对信号量值的加减没有影响，只是在某个进程意为退出是恢复而已。）

（1）当`sem_op`值为正时，对应于进程释放的占用的资源数。`sem_op`值会加到信号量的值上。如果指定了`undo`标志，从该进程的次信号量调整值中加上去`sem_op`。

（2）若`sem_op`值为负，表示要获取由该资源控制的资源。如果该信号量的值大于等于`sem_op`的绝对值，则从信号量值中减去`sem_op`的绝对值。保证信号量值大于等于0。如果指定了`undo`标志，该进程的此信号调整值减去`sem_op`的绝对值。如果信号量值小于`sem_op`的绝对值，则下面的规则适用：

1. 若指定了`IPC_NOWAIT`，则`semop`出错返回`EAGAIN`。
2. 若未指定`IPC_NOWAIT`，则该信号的`semncnt`值加1（因为调用进程进入休眠状态），然后调用进程被挂起直至下列之一发生：
   1. 此信号量值变成大于等于`sem_op`的绝对值（即某个进程已释放了某些资源）。此信号量的`semncnt`值减一（因为已结束等待），并且从信号量值中减去`sem_op`的绝对值。如果指定`undo`标志，进程的此信号调整值减去`sem_op`的绝对值。
   2. 从系统中删除了此信号量，在这种情况下，函数出错返回`EIDRM`。
   3. 进程捕捉到一个信号，并从信号处理程序返回，此时，此信号量的`semncnt`值减1（因为调用进程不再等待），并且函数出错返回`EIDRM`。

（3）如果`sem_op`为0，这表示调用进程希望等待到该信号量值变成0。如果此信号值当前是0，此信号函数立即返回。如果信号量值非0，则适用下列条件：

1. 若指定了`IPC_NOWAIT`，则出错返回`EAGAIN`。
2. 若未指定`IPC_NOWAIT`，则该信号的`semzcnt`值加一（调用进程进入休眠），然后调用进程被挂起，直到下列一个事件发生：
   1. 此信号量值变成0。此信号量的`semzcnt`值减一（进程结束等待）。
   2. 从系统中删除了此信号量。此时，函数出错返回`EIDRM`。
   3. 进程捕捉到一个信号，并从信号处理程序返回。此信号量的`semncnt`值减1（因为调用进程不再等待），并且函数出错返回`EIDRM`。

`semop`函数具有原子性。

`exit`时的信号量调整。无论何时，只要为进程量操作指定了`SEM_UNDO`标志，然后分配资源（`sem_op`小于0），那么进程就会记住对于该特定信号量，分配给进程多少资源(`sem_op`的绝对值)。当该进程终止时，不论自愿还是不自愿，内核都会检验该进程是否还有尚未处理的信号调整量，如果有，则按调整值对应信号量值进行处理。如果用带`SETVAL`或`SETALL`命令的`semctl`设置一个信号量的值，则在所有进程中，该信号量的调整值都将设置为0。

## 共享存储

共享存储允许两个或多个进程共享一个给定的存储区。因为数据不需要在客户进程和服务器进程之间复制，所以这是最快的一中IPC。共享存储唯一问题就是访问冲突，一个进程在写时，别的进程不应该去读。

XSI共享存储与内存映射的文件的不同之处在于，前者没有相关文件。XSI共享存储段是内存的匿名段。

内核为每个共享存储段维护者一个结构：

```c
struct shmid_ds{
	struct ipc_perm shm_perm;
    size_t shm_segsz; /* size of segment in bytes */
    pid_t shm_lpid; /* pid of last shmop() */
    pid_t shm_cpid; /* pid of creator */
    shmatt_t shm_nattch; /* number of current attaces */
    time_t shm_atime; /* last-attach time */
    time_t shm_dtime; /* last-detach time */
    time_t shm_ctime; /* last-change time */
    ...
};

```

`shmatt_t`类型定义为无符号整型，它至少与`unsigned short`一样大。

调用的第一个函数通常是`shmget`，它获得一个共享存储标识符：

```c
#include<sys/shm.h>
int shmget(key_t key, size_t size, int flag);
//返回值：成功发货共享存储ID，出错返回-1

```

之前说了将`key`变换为一个标识符的规则和如何创建一个新的共享存储段。当创建一个新段时，初始化`shmid_ds`结构的下列成员：

1. `ipc_perm`结构。该结果的`mode`按`flag`中的响应权限位设置。
2. `shm_lpid`、`shm_nattach`、`shm_atime`和`shmid_dtime`都设置成0。
3. `shm_ctime`设置为当前时间。
4. `shm_segsz`设置为请求的`size`。

参数`size`是共享存储段的长度，以字节为单位。实现通常将其像上取整为系统页长的整数倍。但，若引用所指定的`size`值并非页长整数倍，那么最后一页的余下部分是不可使用的。如果创建一个新段，则必须指定其`size`。

`shmctl`函数对共享存储执行多种操作：

```c
#include<sys/shm.h>
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
//成功返回0，出错返回-1

```

`cmd`是下列5种命令中的一种：

| `IPC_STAT`   | 获取此段的`shmid_ds`结构，存储在`buf`指向结构中。            |
| ------------ | ------------------------------------------------------------ |
| `IPC_SET`    | 按`buf`指向的结构中的值设置与此共享存储段相关的`shmid_ds`结构中的下列字段：`shm_perm.uid`、`shm_perm.gid`和`shm_perm.mode`。此命令只能由下列两种进程执行：一种是其有效用户ID等于`shm_perm.cuid`或`shm_perm.uid`的进程；另一种是超级用户的特权进程。 |
| `IPC_RMID`   | 从系统中删除该共享存储段。由于每个共享存储段维护着一个连接计数（`shmid_ds`结构中的`shm_nattch`字段），所以除非使该字段的最后一个进程终止或与该段分离，否则不会实际上删除存储段。但不管此段是否任在使用，该段标识符都会被立即删除，所以不能再使用`shmat`与该段连接。此命令只能由下列两种进程执行：一种是其有效用户ID等于`shm_perm.cuid`或`shm_perm.uid`的进程；另一种是超级用户的特权进程。 |
| `SHM_LOCK`   | 在内存对共享存储段加锁。此命令只能由超级用户执行。（Linux和Solaris提供） |
| `SHM_UNLOCK` | 解锁共享段，此命令只能由超级用户执行。（Linux和Solaris提供） |

创建一个共享存储段后，进程可以调用`shmat`将其连接到它的地址空间中：

```c
#include<sys/shm.h>
void *shmat(int shmid, const void *addr, int flag);
//成功返回共享存储段的指针，出错返回-1

```

返回的地址位置取决于`addr`和`flag`：

| `addr = 0`   | `-`                   | 由内核选择的第一个可用地址上。（推荐方式）                   |
| ------------ | --------------------- | ------------------------------------------------------------ |
| `addr ！= 0` | `flag`未指定`SHM_RND` | 连接到`addr`指定地址。                                       |
| `addr！=0`   | `flag`指定`SHM_RND`   | 此段连接到`addr-（addr mod SHMLBA）`所表示的地址上。`SHM_RND`意为取整。`SHBLBA`意为低边界地址倍数。 |

如果在`flag`中指定了`SHM_RDONLY`位，则以只读方式连接此段，否则以读写方式连接。

`shmat`返回值是与该段所连接的实际地址，如果出错则返回-1。如果成功，那么内核将使与该共享存储段相关联的`shmid_ds`结构中的`shm_nattch`计数器加一。

当对共享存储段操作完成后，则调用`shmdt`与该段分离：

```c
#include<sys/shm.h>
int shmdt(const void *addr);
//成功返回0， 失败返回-1

```

如果成功`shmdt`将使相关`shmid_ds`结构中的`shm_nattch`计数值减一。

例程：查看特定系统存放各种类型数据的位置信息：

```c
#include"apue.h"
#include<sys/shm.h>

#define ARRAY_SIZE 40000
#define MALLOC_SIZE 100000
#define SHM_SIZE 100000
#define SHM_MODE 0600 /* user read/write */

char array[ARRAY_SIZE]; /* uninitialized data = bss */

int main()
{
    int shmid;
    char *ptr, *shmptr;
    printf("array[] from %p to %p\n", (void *)array, array+ARRAY_SIZE);

    printf("stack around %p\n", (void*)&shmid);

    if((ptr = (char*)malloc(MALLOC_SIZE)) == NULL)
    {
        err_sys("malloc error");
    }

    printf("malloc from %p to %p\n", (void*)ptr, (void*)(ptr+MALLOC_SIZE));

    if((shmid = shmget(IPC_PRIVATE, SHM_SIZE, SHM_MODE)) < 0)
    {
        err_sys("shmget error");
    }

    if((shmptr = (char*)shmat(shmid, 0, 0)) == (char*)-1)
    {
        err_sys("shmat error");
    }

    if(shmctl(shmid, IPC_RMID, 0)<0)
    {
        err_sys("shmctl error");
    }
    exit(0);
}

```

执行结果：

```
$ ./15-31.o
array[] from 0x557b5b113060 to 0x557b5b11cca0
stack around 0x7ffc6136ea64
malloc from 0x557b5b211670 to 0x557b5b229d10

```

下图展示了Linux系统上的存储布局：

![mem](https://s2.ax1x.com/2019/10/01/uU2mvD.png)

## POSIX信号量

POSIX信号量接口意在解决XSI信号量接口的几个缺陷：

1. 相比于XSI接口，POSIX信号量接口考虑到了更高性能的实现。
2. POSIX信号量接口使用更简单：没有信号集，在熟悉的文件系统操作后一些接口被模式化了。
3. POSIX信号量在删除时表现的更完美。

POSIX信号量有两种形式：命名和未命名的。它们的差异在于创建和销毁的形式上，但其他工作一样。未命名信号量只存在于内存中，并要求能使用信号量的进程必须可以访问内存。这意味着它们只能应用在同一进程中的线程，或者不同进程中已经映射相同内存内容到它们的地址空间中的线程。相反，命名信号量可以通过名字访问，因此可以被任何已知它们名字的进程中的线程使用。

`sem_open`函数来创建一个新的命名信号量或者使用一个现有信号量：

```c
#include<semaphore.h>
sem_t *sem_open(const char *name, int oflag, ... /* mode_t mode, unsigned int value */);
//成功返回指向信号量的指针，出错返回SEM_FAILED

```

当使用一个现有命名信号量时，我们只指定两个参数：信号量名字和`oflag`参数的0值。当`oflag`参数有`O_CREAT`标志时，如果命名信号量不存在，则创建一个新的。如果存在，则会被使用，但不会有额外的初始化发生。

当我们指定`O_CREAT`标志时，需要提供两个额外的参数。`mode`参数指定谁可以访问信号量。其取值与打开文件的权限位相同（用户读写执行，组读写执行，其他读写执行）。赋值给信号量的权限可以被调用者的文件创建屏蔽字修改。创建信号量时，`value`指定信号量初始值。

如果想确保创建的是信号量，可以设置`oflag`为`O_CREAT|O_EXCL`。如果信号量已经存在，则会导致函数失败。

为了增加可移植性，信号量命名时应该遵循下列规则：

1. 名字的第一个字符应该是斜杠（`/`）。
2. 名字不应该包含其他斜杠以避免实现定义的行为。
3. 信号量的最大长度是实现定义的。

完成信号量操作后，`sem_close`函数来释放任何信号量相关资源：

```c
#include<semaphore.h>
int sem_close(sem_t *sem);
//成功返回0，出错返回-1

```

如果进程没有首先调用`sem_close`而退出，那么内核将自动关闭任何打开的信号量。注意：这不会影响信号量值的状态——如果已经对它进行了增加1操作这不会仅因为退出而改变（不会自动减一的）。类似的，如果调用`sem_close`，信号量值也不会受到影响。

可以调用`sem_unlink`函数来销毁一个命名信号量：

```c
#include<semaphore.h>
int sem_unlink(const char *name);
//成功返回0，出错返回-1

```

`sem_unlink`函数删除信号量的名字。如果没有打开的信号量引用，则该信号量会被销毁。否则，销毁将延迟到最后一个打开的引用关闭。

下面的函数实现信号量的减一：

```c
#include<semaphore.h>
int sem_trywait(sem_t *sem);
int sem_wait(sem_t *sem);
//成功返回0，否则返回-1；

```

`sem_trywait`避免阻塞。如果信号量是值是0`sem_wait`就会阻塞。还可以选择阻塞一段时间：

```c
#include<semaphore.h>
#include<time.h>
int sem_timedwait(sem_t *restruct sem, const struct timespec *restrict tsptr);
//成功返回0，出错返回-1

```

如果超时到期并且信号量计数还没能减一，函数返回-1，且将`errno`设置为`ETIMEDOUT`。

调用函数`sem_post`函数使信号量值增加1：

```c
#include<semaphore.h>
int sem_post(sem_t *sem);
//成功返回0，出错返回-1

```

如果`sem_post`时，如果在调用`sem_wait`中发生进程阻塞，那么进程会被唤醒并且被`sem_post`增1的信号量计数会再次被`sem_wait`减1。

当在单个进程中使用信号量时，使用未命名的信号量更容易。这仅仅改变创建和销毁信号量的方式。可以调用`sem_init`函数来创建一个未命名信号量：

```c
#include<semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
//成功返回0，出错返回-1

```

`pshared`参数表明是否在多个进程中使用信号量。如果是，将其设置为非0。`value`参数指定信号量初始值。需要申明一个`sem_t`类型变量并将其地址传给`sem_init`来初始化。如果要在两个进程之间使用信号量，需要确保`sem`参数指向两个进程之间共享的内存范围。

对未命名信号量的使用已经完成时，可以调用`sem_destroy`函数丢弃：

```c
#include<semaphore.h>

int sem_destory(sem_t *sem);
//成功返回0，出错返回-1

```

调用`sem_destory`函数后，不能再使用任何带有`sem`的信号量函数，除非通过调用`sem_init`重新初始化它。

`sem_getvalue`函数可以用来检索信号量值：

```c
#include<semaphore.h>
int sem_getvalue(sem_t *restruct sem, int *restrict valp);
//成功返回0，出错返回-1

```

实例：使用信号量来创建锁原语从而提供互斥：

```c
#include<stdlib.h>
#include<stdio.h>
#include<unistd.h>
#include<errno.h>
#include<semaphore.h>
#include<fcntl.h>

struct slock{
    sem_t *semp;
    char name[100];
};

struct slock *s_alloc()
{
    slock *sp;
    static int cnt;

    if((sp = (slock *)malloc(sizeof(slock))) == NULL)
    {
        return NULL;
    }
    do{
        snprintf(sp->name, sizeof(sp->name), "/%ld.%d", (long)getpid(), cnt++);
        sp->semp = sem_open(sp->name, O_CREAT | O_EXCL, S_IRWXU, 1);
    }while((sp->semp == SEM_FAILED) && (errno == EEXIST));

    if(sp->semp == SEM_FAILED)
    {
        free(sp);
        return NULL;
    }
    sem_unlink(sp->name); /* destory the name */
    return sp;
}

void s_free(slock *sp)
{
    sem_close(sp->semp);
    free(sp);
}

int s_lock(slock *sp)
{
    return sem_wait(sp->semp);
}

int s_trylock(slock *sp)
{
    return sem_trywait(sp->semp);
}

int s_unlock(slock *sp)
{
    return sem_post(sp->semp);
}

```

我们在打开一个信号量后断开了它的连接。这销毁了名字，所以导致其他进程不能再次尝试访问它，这简化了进程结束时的清理工作。



## 客户进程-服务器进程属性

客户进程和服务器进程的某些属性受到所使用的各种IPC类型的影响。

## 部分习题代码

15.12：

```c
#include<sys/msg.h>
#include<stdio.h>
#include<sys/ipc.h>
#include<error.h>
#include<errno.h>
#include"apue.h"

struct mymesg{
    long mtype;
    char mtext[128];
};

int main()
{
    int err;
    char key[] = "~/study_file/UNIX-PROGRAM";
    for(int i=0;i<5;i++)
    {
        key_t ki = ftok(key, i);
        err = msgget(ki, IPC_CREAT | IPC_EXCL | 0666);
        if(err == EEXIST)
        {
            err_sys("msg has exist");
        }
        else{
            printf("%d msg fd is %d\n", i, err);
        }
        if(msgctl(err, IPC_RMID, NULL)<0)
        {
            err_sys("delete msg %d error", err);
        }

    }
    char line[128];
    for(int i=0;i<5;i++)
    {
        err = msgget(IPC_PRIVATE, IPC_CREAT | IPC_EXCL | 0666);
        if(err == EEXIST)
        {
            err_sys("msg has exist");
        }
        else{
            printf("%d msg fd is %d\n", i, err);
        }

        mymesg msg;
        msg.mtype = i+1;
        int n;
        if((n = sprintf(msg.mtext, "hello world\n"))<0)
        {
            printf("sprintf error");
        }
        else{
            printf("line length is %d\n", n);
            msg.mtext[n] = 0;
            printf("line is %s\n", msg.mtext);
        }
        if(msgsnd(err, (void *)&msg, strlen(msg.mtext)+1, 0)<0)
        {
            err_sys("msgsnd error in %d msg", err);
        }
        mymesg msgr;
        if((n = msgrcv(err, (void *)&msgr, 128, i+1, 0))<0)
        {
            err_sys("msgrcv error");
        }
        else{
            printf("receive data is %s\n", msgr.mtext);
        }
    }
    exit(0);
}

```

15.15：

```c
#include<sys/shm.h>
#include"apue.h"
#include<stdio.h>

#define NLOOPS 1000

int main()
{
    pid_t pid;
    int i, counter, shmid;
    int *area;
    key_t key = ftok("/study_file/unix", 1);
    TELL_WAIT();
    if((pid = fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid > 0)
    {
        shmid = shmget(key, sizeof(int), IPC_EXCL | IPC_CREAT | 0600);
        if(shmid<0)
        {
            err_sys("shm has exit");
        }
        else{
            area = (int*)shmat(shmid, 0, IPC_NOWAIT);
            *area = 0;
        }
        TELL_CHILD(pid);

        for(i=0;i<NLOOPS; i += 2)
        {
            if((counter = (*area)++) != i)
            {
                err_quit("parent: expected %d, got %d", i, counter);
            }
            else{
                printf("parent counter is %d\n", counter);
            }
            TELL_CHILD(pid);
            WAIT_CHILD();
        }
        sleep(0.1);
        shmctl(shmid, IPC_RMID, NULL);
    }
    else{
        WAIT_PARENT();
        shmid = shmget(key, 0, 0);
        if(shmid <0)
        {
            err_sys("shmget error");
        }
        else{
            area = (int*)shmat(shmid, 0, IPC_NOWAIT);
        }

        for(int i=1; i<NLOOPS; i+= 2)
        {
            WAIT_PARENT();
            if((counter = (*area)++) != i)
            {
                err_quit("child excepted %d get %d", i, counter);
            }
            else{
                printf("child counter is %d\n", counter);
            }
            TELL_PARENT(getppid());
        }
    }
    exit(0);
}

```

15.16

```c
#include <sys/sem.h>
#include "apue.h"
#include <stdio.h>

#define NLOOPS 1000

union semun {
    int val;				// <= value for SETVAL
     struct semid_ds *buf;		// <= buffer for IPC_STAT & IPC_SET
     unsigned short int *array;		// <= array for GETALL & SETALL
     struct seminfo *__buf;		// <= buffer for IPC_INFO
};

int area = 0;

int main()
{
    pid_t pid;
    int semid = semget(IPC_PRIVATE, 1, IPC_CREAT | IPC_EXCL | 0600);
    if (semid < 0)
    {
        err_sys("semget error");
    }
    semun arg;
    arg.val = 1;
    if(semctl(semid, 0, SETVAL, arg)<0)
    {
        err_sys("semctl error");
    }
    TELL_WAIT();
    if((pid=fork())<0)
    {
        err_sys("fork error");
    }
    else if(pid > 0)
    {   sembuf semb[1];
        for(int i=0;i<NLOOPS;i+=2)
        {
            semb[0].sem_flg = 0;
            semb[0].sem_op = -1;
            semb[0].sem_num = 0;
            if(semop(semid, semb, 1)<0)
            {
                err_sys("semop error");
            }
            int n = semctl(semid, 0, GETVAL);
            printf("parent process1 val is %d\n", n);
            fflush(stdout);
            TELL_CHILD(pid);
            area += 2;
            if(area-2 != i)
            {
                err_sys("parent excepted %d got %d", i, area-2);
            }
            else{
                printf("parent counter is %d\n", area-2);
                fflush(stdout);
            }
            semb[0].sem_flg = 0;
            semb[0].sem_op = 1;
            semb[0].sem_num = 0;
            if(semop(semid, semb, 1)<0)
            {
                err_sys("semop error");
            }
            n = semctl(semid, 0, GETVAL);
            printf("parent process1 val is %d\n", n);
            fflush(stdout);
            WAIT_CHILD();
        }
        sleep(0.1);
        semctl(semid, 0, IPC_RMID);
    }
    else{
        area = 1;
        sembuf semb[1];
        for(int i=1; i<NLOOPS; i+=2)
        {
            WAIT_PARENT();
            semb[0].sem_flg = 0;
            semb[0].sem_op = -1;
            semb[0].sem_num = 0;
            if(semop(semid, semb, 1)<0)
            {
                err_sys("semop error");
            }
            int n = semctl(semid, 0, GETVAL);
            printf("child process1 val is %d\n", n);
            fflush(stdout);
            TELL_PARENT(getppid());
            area += 2;
            if(area-2 != i)
            {
                err_sys("child excepted %d got %d", i, area-2);
            }
            else{
                printf("child counter is %d\n", area-2);
                fflush(stdout);
            }
            semb[0].sem_flg = 0;
            semb[0].sem_op = 1;
            semb[0].sem_num = 0;
            if(semop(semid, semb, 1)<0)
            {
                err_sys("semop error");
            }
            n = semctl(semid, 0, GETVAL);
            printf("child process2 val is %d\n", n);
            fflush(stdout);
        }
    }
    exit(0);
}

```

# 第十六章 网络IPC：套接字

## 套接字描述符

套接字是通信端点的抽象。套接字描述符在UNIX系统中被当做一种文件描述符。为创建一个套接字，调用下面的函数：

```c
#include<sys/socket.h>
int socket(int domain,int type, int protocol);
//成功返回套接字描述符，出错返回-1
```

domain（域）确定通信的特征，包括地址格式。各个域使用AF_开头，意指地址在（address family）。下表列出POSIX.1指定的各个域：

| 域（地址族） | 描述         |
| ------------ | ------------ |
| `AF_INER`    | IPv4因特网域 |
| `AF_INET6`   | IPv6因特网域 |
| `AF_UNIX`    | UNIX域       |
| `AF_UPSPEC`  | 未指定       |

type确定套接字类型，进一步确定通信特征。下表总结了POSIX.1定义的套接字类型：

| 类型             | 描述                                             | 详情                                                         |
| ---------------- | ------------------------------------------------ | ------------------------------------------------------------ |
| `SOCK_DGRAW`     | 固定长度的、无连接的、不可靠的报文传输。         | 两个对等进程之间通信不需要逻辑连接。                         |
| `SOCK_RAW`       | IP协议的数据报接口。                             | 提供一个数据报接口，用于直接访问IP层。使用该接口，应用程序否则自己构建自己的协议头部。当创建一个原始套接字时，需要超级用户权限。 |
| `SOCK_SEQPACKET` | 固定长度的、有序的、可靠的、面向连接的报文传递。 | 与SOCK_SEQPACKET类似。不过提供基于报文的服务。               |
| `SOCK_STREAM`    | 有序的、可靠的、双向的、面向连接的字节流。       | 在交换数据之前建立逻辑连接。                                 |

protocol通常是0，表示为给定的域和套接字类型选择默认协议。同一域和套接字类型支持多个协议时，可以使用Protocol选择一个特定协议。在AF_INET通信域中，套接字类型SOCK_STREAM的默认协议是TCP。在AF_INET通信域中，套接字类型SOCK_DGRAM的默认协议是UDP。下表列出了因特网域套接字定义的协议：

| 协议           | 描述               |
| -------------- | ------------------ |
| `IPPROTO_IP`   | IPv4网际协议       |
| `IPPROTO_IPv6` | IPv6网际协议       |
| `IPPROTO_ICMP` | 因特网报文控制协议 |
| `IPPROTO_RAW`  | 原始IP数据包协议   |
| `IPPROTO_TCP`  | 传输控制协议       |
| `IPPROTO_UDP`  | 用户数据报协议     |

soket与open函数类似，都是返回可用于I/O的文件描述符。不再需要时，调用close关闭。下面总结了常用的文件描述符对套接字的支持：

![16-4](https://s2.ax1x.com/2020/02/14/1Xq4VH.png)

套接字通信是双向的。使用实用shutdown函数来禁止一个套接字的I/O：

```c
#include<sys/socket.h>

int shutdown(int sockfd, int how);
//成功返回0，出错返回-1
```

如果how是SHUT_RD(关闭读端)，那么无法从套接字读数据。如果how是SHUT_WR（关闭写端），那么无法向套接字无法写数据。

## 寻址

进程标识由两部分组成：计算机网络地址和计算机端口号（标识进程）。

### 字节序

字节序是处理器架构特性，用于指示像整数这样的大数据类型内部的字节如何排序。下图展示了

位整数中字节如何排序：

![16-5](https://s2.ax1x.com/2020/02/14/1jJOZq.png)

字节序是指索引内部地址时的顺序，不影响实际存储顺序。大端中，索引0在最左边，小端是，索引0在右边。如一个32位整数（0x04030201），大端的索引0是4，小端的索引0是1。其中LSB是最低有效字节（Least Significant Byte，LBS）。MSB是最高有效字节（Most Significant Byte，MSB）。

网络协议为了在异构计算机系统能够交换协议信息而不会被字节序所混淆指定了字节序。TCP/IP协议栈使用了大端字节序。对于TCP/IP应用程序，下面函数用来处理字节序和网络字节序之间转换：

```c
#include<arpa/inet.h>
uint32_t htonl(uint32_t hostint32);
//返回以网络字节序表示的32为整数

uint16_t htons(uint16_t hostint16);
//返回以网络字节序表示的16为整数

uint32_t ntohl(uint32_t netint32);
//返回值为主机字节序表示的32位整数

uint16_t ntohl(uint16_t netint16);
//返回值为主机字节序表示的16位整数
```

h表示主机，n表示网络。l表示长，s表示短。



### 地址格式

为使不同格式地址能够传入到套接字函数，地址会被强制转换成一个通用地址格式的sockaddr：

```c
struct sockaddr{
	sa_family_t sa_family; /* address family */
	char sa_data[]; /* variable-length address */
	.
	.
	.
}
```

Linux中，该结构是：

```c
struct sockaddr{
	sa_family_t sa_family; /* address family */
	char sa_data[14]; /* variable-length address */
}
```

因特网地址定义在<netinet/in.h>头文件中。IPv4因特网域中，套接字地址用sockaddr_in表示：

```c
struct in_addr{
	in_addr_t s_addr; /* IPv4 address */
}

struct sockaddr_in{
	sa_family_t sin_family; /*addresss family*/
	in_port_t sin_port; /* port number */
	struct in_addr sin_addr; /* IPv4 address */
}
```

数据类型`in_port_t`定义成`uint16_t`。数据类型`in_addr_t`定义成`uint32_t`。

IPv6因特网域套接字地址：

```c
struct in6_addr{
	uint8_t s6_addr[16]; /* IPv6 address */
}

struct sockaddr_in6{
	sa_family_t sin6_family; /* address family */
	in_port_t sin6_port; /* port number */
	uint32_t sin6_flowninfo; /* traffic class and flow info */
	struct in6_addr sin6_addr; /* IPv6 address */
	uint32_t sin6_scope_i; /* set of interface for scope */
}
```

Linux中，sockaddr_in定义如下：

```c
struct sockaddr_in{
	sa_family_t sin_family; /* address family */
	in_port_t sin_port; /* port number */
	struct in_addr sin_addr; /* IPv4 address */
	unsigned char sin_zero[8]; /* filler */
}
```

成员sin_zero是填充字段，应该全部设为0。

sockaddr_in和sockaddr_in6结构相差较大，但均被强制转换成sockaddr结构输入到套接字例程中。

下列两个函数用于二进制地址与点分十进制格式进行转换：

```c
#include<arpa/inet.h>
const char *inet_ntop(int domain, const void *restrict addr, char *restrict str, soclen_t size);
//成功返回地址字符串指针，出错返回NULL

int inet_pton(int domain,const char *restrict str, void *restrict addr);
//成功返回1，若格式无效，返回0，出错返回-1
```

函数`inet_ntop`将网络字节序的二进制转换为文本字符串格式。参数domain支持AF_INET和AF_INET6。参数size指定了保存文本的缓冲区大小。可以使用两个常数来简化工作：INET_ADDRSTRLEN和INET6_ADDRSTRLEN分别定了了足够大的空间来保存一个IPv4和IPv6地址的字符串。

### 地址查询

网络配置信息被存放在很多地方。可以存放在静态文件（如/etc/hosts或/etc/services）中，或者DNS中或网络信息服务（NIS）中。无论在何处都可以使用同样的函数访问到。

调用`gethostent`获得给定计算机主机信息：

```c
#include<netdb.h>

struct hostent *gethostent(void);
//成功返回指针，若出错，返回NULL

void sethostent(int stayopen);
void endhostent(void);
```

如果主机数据库文件没有打开，gethostent会打开它。函数gethostent返回文件中的下一个条目。函数sethostend会打开文件，如果文件已经打开，那么将其绕回。当stayopen参数设置成非零，调用gethostent后，文件依然打开。函数endhostent可以关闭文件。

gethostent返回一个指向hostent结构的指针，该结构包含一个静态缓冲区，每次调用gethostent后，缓冲区都会被覆盖。hostent结构如下：

```c
struct hostent{
	char *h_name; /* name of host */
	char **h_aliases; /* points to alternate host name array */
	int h_addrtype; /*address type */
	int h_length; /* length in bytes of address */
	char **h_addr_list; /* pointer to array of network address */
	.
	.
	.
}
```

采用一套相似的接口来获得网络名字和网络编号：

```c
#include<netdb.h>

struct netent *getnetbyaddr(uint32_t net, int type);

struct netent *getnetbyname(const char *name);

struct netent *getnetent(void);
//成功返回指针，出错返回NULL

void setnetent(int stayopen);

void endnetent(void);

struct netent{
    char *n_name; /* network name */
    char **n_aliases; /* alternate network name array pointer */
    int n_addrtype; /*address type */
    uint32_t n_net; /* network number */
    .
    .
    .
}
```

可以使用下面的函数在协议名字和协议编号之间进行映射：

```c
#include<netdb.h>
struct protoent *getprotobyname(const char *name);
struct protoent *getprotobynumber(int proto);
struct protoent *getprotoent(void);
//成功返回指针，出错返回NULL

void setprotoent(int stayopen);
void endprotoent(void);

struct protoent{
	char *p_name; /*protocol name*/
	char **p_aliases; /* pointer to altername protocol name array */
	int p_proto; /* protocol number */
	.
	.
	.
}
```

服务是由地址的端口号部分表示的。每个服务由一个唯一的众所众知的端口号来支持。可以使用函数`getservbyname`将一个服务名映射到一个端口号，使用函数`getservbyport`将一个端口号映射到一个服务名，使用`getservent`顺序扫描服务数据库：

```
#include<netdb.h>

struct servent *getservbyname(const char *name, const char *proto);
struct servent *getservbyport(int port, const char *proto);
struct servent *getservent(void);
//成功返回指针，失败返回NULL

void setservent(int stayopen);
void endservent(void);

struct servent{
	char *s_name; /* service name*/
	char **s_aliase; /* pointer to alternate service name array */
	int s_port; /* port number */
	char *s_proto; /* name of protocol */
}
```



POSIX.1定义了若干新函数，允许应用程序将一个主机名和一个服务器映射到一个地址，或者反之。`getaddrinfo`允许将一个主机名和一个服务名映射到一个地址上：

```c
#include<sys/socket.h>
#include<netdb.h>

int getaddrinfo(const char *restrict host, const char *restrict service, 
				const struct addrinfo *restrict hint, struct addrinfo **restrict res);
//成功返回0，出错返回非0错误码

void freeaddrinfo(struct addrinfo *ai);
```

需要提供主机名、服务名，或者两者都提供。如果只提供一个名字，另外一个必须是一个空指针。主机名可以是一个节点名或点分格式的主机地址。

函数返回一个链表结构`addrinfo`：

```c
struct addrinfo{
	int ai_flags; /* customize behavior */
	int ai_family; /* address family */
	int ai_socktype; /* socket type */
	int ai_protocol; /* protocol */
	socklen_t *ai_addr; /* length in bytes of address */
	struct sockaddr *ai_addr; /* address */
	char *ai_canonname; /* canonical name of host */
	struct addrinfo *ai_next;
	.
	.
	.
}
```

可以提供一个可选的hint来选择符合特定条件的地址。hint是一个用于过滤地址的模板，包括`ai_family`，`ai_protocol`，`ai_flags`和`ai_socktype`字段。剩余字段必须设置成0，指针必须为空。下图总结了`ai_flags`字段中的标志：

| 标志             | 描述                                           |
| ---------------- | ---------------------------------------------- |
| `AI_ADDRCONFIG`  | 查询配置的地址类型（IPv4或IPv6）               |
| `AI_ALL`         | 查找IPv4和IPv6地址（用于`AI_V4MAPPED`）        |
| `AI_CANONNAME`   | 需要一个规范的名字（与别名相对）               |
| `AI_NUMERICHOST` | 以数字格式指定主机地址，不翻译                 |
| `AI_NUMERICSERV` | 将服务指定为数字端口，不翻译                   |
| `AI_PASSIVE`     | 套接字地址用于监听绑定                         |
| `AI_V4MAPPED`    | 如果没有IPv6地址，返回映射到IPv6格式的IPv4地址 |

如果`getaddrinfo`失败，不能使用`perror`或`strerror`来生成错误信息，需要调用`gai_strerror`将返回的错误码转换成错误信息：

```c
#include<netdb.h>

const char *gai_strerror(int error);
//返回指向错误的字符串的指针
```

`getnameinfo`将一个地址转换成一个主机名和一个服务名：

```
#include<sys/socket.h>
#include<netdb.h>

int getnameinfo(const struct sockaddr *restrict addr, socklen_t alen, char *restrict host,socklen_t hostlen, char *restrict service, socklen_t servlen, int flags);
//成功返回0，出错返回非0值
```

flags参数提供一些控制方式：

| 标志              | 描述                                       |
| ----------------- | ------------------------------------------ |
| `NI_DGRAM`        | 服务基于数据报而非基于流                   |
| `NI_NAMEREQD`     | 如果找不到主机名，将其作为一个错误对待     |
| `NI_NOFQDN`       | 对于本地主机，仅返回全限定域名的节点名部分 |
| `NI_NUMERICHOST`  | 返回主机地址的数字形式，而非主机名         |
| `NI_NUMERICSCOPE` | 对于IPv6，返回范围ID的数字形式，而非名字   |
| `NI_NUMERICSERV`  | 返回服务地址的数字形式（端口号），而非名字 |

`getaddrinfo`函数使用方式：

```c
#include"apue.h"
#if defined(SOLARIS)
#include<netinet/in.h>
#endif
#include<netdb.h>
#include<arpa/inet.h>
#if defined(BSD)
#include<sys/socket.h>
#endif

void print_family(struct addrinfo *aip)
{
    printf(" family ");
    switch(aip->ai_family){
        case AF_INET:
        printf("inet");
        break;

        case AF_INET6:
        printf(" inet6 ");
        break;

        case AF_UNIX:
        printf(" unix ");
        break;

        case AF_UNSPEC:
        printf(" unspecified ");
        break;

        default:
        printf(" unknow ");
    }
}

void print_type(struct addrinfo *aip){
    printf(" type ");
    switch (aip->ai_socktype)
    {
    case SOCK_STREAM:
        printf("stream");
        break;

    case SOCK_DGRAM:
        printf("datagram");
        break;

    case SOCK_SEQPACKET:
        printf("seqpacket");
        break;
    
    case SOCK_RAW:
        printf("raw");
        break;
    
    default:
        printf("unknow (%d)", aip->ai_socktype);
    }
}

void print_protocol(struct addrinfo *aip){
    printf(" protocol ");
    switch (aip->ai_protocol)
    {
    case 0:
        printf("default");
        break;
    
    case IPPROTO_TCP:
        printf("TCP");
        break;

    case IPPROTO_UDP:
        printf("UDP");
        break;

    case IPPROTO_RAW:
        printf("raw");
        break;

    default:
        printf("unknow (%d)",aip->ai_protocol);
        break;
    }
}

void print_flags(struct addrinfo *aip)
{
    printf(" flags ");
    if(aip->ai_flags == 0)
    {
        printf("0");
    }
    else{
        if(aip->ai_flags & AI_PASSIVE)
        {
            printf(" passive");
        }
        if(aip->ai_flags & AI_CANONNAME)
        {
            printf(" canon");
        }
        if(aip->ai_flags & AI_NUMERICHOST)
        {
            printf(" numhost");
        }
        if(aip->ai_flags & AI_NUMERICSERV)
        {
            printf(" numserv");
        }
        if(aip->ai_flags & AI_V4MAPPED)
        {
            printf(" v4mapped");
        }
        if(aip->ai_flags & AI_ALL)
        {
            printf(" all");
        }
    }
}

int main(int argc, char *argv[])
{
    struct addrinfo *ailist, *aip;
    struct addrinfo hint;
    struct sockaddr_in *sinp;
    const char *addr;
    int err;
    char abuf[INET_ADDRSTRLEN];

    if(argc != 3)
    {
        err_quit("usage:%s nodename service",argv[0]);
    }

    hint.ai_flags = AI_CANONNAME;
    hint.ai_family = 0;
    hint.ai_socktype = 0;
    hint.ai_protocol = 0;
    hint.ai_canonname = NULL;
    hint.ai_addr = NULL;
    hint.ai_next = NULL;
    if((err = getaddrinfo(argv[1], argv[2], &hint, &ailist))!=0)
    {
        err_quit("getaddrinfo error %s",gai_strerror(err));
    }
    for(aip = ailist; aip!=NULL; aip=aip->ai_next)
    {
        print_flags(aip);
        print_family(aip);
        print_type(aip);
        print_protocol(aip);
        printf("\n\thost %s", aip->ai_canonname ? aip->ai_canonname:"-");
        if(aip->ai_family == AF_INET)
        {
            sinp = (struct sockaddr_in*)aip->ai_addr;
            addr = inet_ntop(AF_INET, &sinp->sin_addr, abuf, INET_ADDRSTRLEN);
            printf(" address %s", addr?addr:"unknow");
            printf(" port %d", ntohs(sinp->sin_port));
        }
        printf("\n");
    }
    exit(0);
}
```

执行：

```shell
$ ./16-9.o www.baidu.com http
 flags  canon family inet type stream protocol TCP
        host www.a.shifen.com address 39.156.66.18 port 80
 flags  canon family inet type stream protocol TCP
        host - address 39.156.66.14 port 80
```

### 将套接字与地址关联

使用bind函数来关联地址和套接字：

```c
#include<sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t len);
//成功返回0，失败返回-1
```

对于地址有如下限制：

1. 在进程正在运行的计算机上，指定的地址必须有效：不能指定一个其他机器上的地址。
2. 地址必须和创建套接字时的地址族所支持的格式相匹配。
3. 地址中的端口号必须小于1024，除非该进程具有相应的特权（超级用户）。
4. 一般只能将一个套接字端点绑定到一个给定的地址上，尽管这些协议允许多重绑定。

对于因特网域，如果指定IP为`INADDR_ANY`（<netinet/in.h>中定义)，套接字端点可以被绑定到所有的网络接口上，即可以接收改系统所安装的任何一个网卡的数据包。如果调用`connect`和`listen`前未绑定地址到套接字上，系统会选择一个地址绑定到套接字上。

调用`getsockname`函数来发现绑定到套接字上的地址：

```c
#include<sys/socket.h>

int getsockname(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict alenp);
//成功返回0，出错返回-1
```

如果套接字已经和对等方连接，可以调用getpeername来找到对方地址：

```c
#include<sys/socket.h>
int getpeername(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict alenp);
//成功返回0，出错返回-1
```



## 建立连接

对于面向连接的网络服务（SOCK_STREAM和SOCK_SEQPACKET），交换数据之前，需要进行连接。使用connect来建立连接：

```c
#include<sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t len);
//成功返回0，出错返回-1
```

connect中指定的地址是我们希望建立连接的地址。如果sockfd没有绑定到一个地址上，connect会给其绑定到一个默认地址。

连接成功的条件是服务器必须运行中，且服务器等待队列要有足够的空间。因此，应用程序必须处理connect返回的错误。

实例：

```c
#include"apue.h"
#include<sys/socket.h>

#define MAXSLEEP 128

int connect_retry(int domain, int type, int protocol, const struct sockaddr *addr, socklen_t alen)
{
    int numsec, fd;

    for(numsec = 1; numsec<=MAXSLEEP; numsec<<1)
    {
        if((fd=socket(domain, type, protocol))<0)
        {
            return -1;
        }
        if(connect(fd, addr, alen) == 0)
        {
            return fd;
        }

        close(fd);

        if(numsec<= MAXSLEEP/2)
        {
            sleep(numsec);
        }
    }
}
```

由于部分系统connect调用失败会使得套接字的状态变成未定义的，因此如果失败了，一个关闭套接字，再次创建（Linux其实不用）。

上述函数也展示出了指数补偿算法。如果套接字描述符处于非阻塞状态，那么当不能立即建立连接时，connect返回-1并且将errno设置为EINPROGRESS。应用程序可以使用poll或select来判断文件描述符何时可写。如果可写，连接完成。

connect函数也可以用于无连接的网络服务（SOCK_DGRAM）。如果SOCK_DGRAM调用connect，传输报文的目的地址会设置成调用中所指定的地址，这样，每次调用不用再指定地址，同时，只能接收从指定地址而来的报文。

服务器调用listen函数来宣告其愿意接收连接：

```c
#include<sys/socket.h>

int listen(int sockfd, int backlog);
//成功返回0，出错返回-1
```

参数backlog提供提示，提示系统该进程所要入队的未完成连接请求数量。其实际值由系统决定，当上限由<sys/socket.h>中的SOMAXCONN指定。

一旦队列满，系统拒绝多余请求，所要backlog的值一个基于服务器期望负载和处理量来选择。

一旦服务器调用了listen，所用的套接字就能够接收连接请求。使用accept函数获得连接请求并建立连接：

```c
#include<sys/socket.h>

int accept(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict len);
//成功返回文件描述符，失败返回-1
```

函数accept所返回的文件描述符是套接字文件描述符，该描述符连接到调用connect的客户端。新的套接字和原始套接字具有相同的套接字类型和地址族。传给accept的原始套接字继续保持可用状态接收其他连接请求。

如果不关心客户端标识，可以将参数addr和len设为NULL。否则可以传递对应参数来获取。如果没有连接请求在等待，accept会阻塞直到一个请求到来。如果sockfd处于非阻塞模式accept会返回-1，并将errno设置为EWOULDBLOCK。

如果服务器调用accept，并且当前没有连接请求，服务器会阻塞直到一个请求的到来。服务器可以使用poll或select来等待一个请求的到来。此时，一个等待连接请求的套接字会以可读的方式出现。

实例：

```c
#include"apue.h"
#include<errno.h>
#include<sys/socket.h>

int initserver(int type, const struct sockaddr *addr, socklen_t alen, int qlen)
{
    int fd;
    int err = 0;
    if((fd = socket(addr->sa_family,type, 0))<0)
    {
        return -1;
    }

    if(bind(fd, addr, alen)<0)
    {
        goto errout;
    }

    if(type == SOCK_STREAM || type == SOCK_SEQPACKET)
    {
        if(listen(fd, qlen)<0)
        {
            goto errout;
        }
    }
    return fd;

errout:
    err = errno;
    close(fd);
    errno = err;
    return -1;
}
```

## 数据传输

套接字描述符上使用read和write是十分有意义的，这意味着可以将套接字描述符传递给那些为原本处理本地文件而设计的函数。而且可以安排将套接字描述符传递给子进程，而该子进程执行的程序并不了解套接字。

send用于发送数据：

```c
#include<sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t nbytes, int flags);
//成功返回发送字节数，出错返回-1
```

类似write，参数buf和nbytes与write中的一致。下表总结了flags：

| 标志            | 描述                                  |
| --------------- | ------------------------------------- |
| `MSG_CONFIRM`   | 提供链路层反馈以保持地址映射有效      |
| `MSG_DONTROUTE` | 勿将数据报路由出本地网络              |
| `MSG_DONTWAIT`  | 允许非阻塞操作                        |
| `MSG_EOF`       | 发送数据后关闭套接字发送端            |
| `MSG_EOR`       | 如果协议支持，标记记录结束            |
| `MSG_MORE`      | 延迟发送数据包允许写更多数据          |
| `MSG_NOSIGNAL`  | 在写无连接的套接字时不产生SIGPIPE信号 |
| `MSG_OOB`       | 如果协议支持，发送外带数据            |

对于支持报文边界的协议，如果尝试发送的单个报文长度超过协议所支持的最大长队，那么send将会失败，并将errno设置为EMSGSIZE。对于字节流协议，send会阻塞到整个数据传输完成。

函数sendto与send类似，不过指定目的传输地址：

```c
#include<sys/socket.h>

ssize_t sendto(int sockfd, const void *buf, size_t nbytes, int flags, const struct sockaddr *destaddr, socklen_t destlen);
//成功返回发送字节数，出错返回-1
```

sendmsg指定多重缓冲区传输数据，这和writev函数类似：

```c
#include<sys/socket.h>

ssize_t sendmsg(int sockfd, const struct msghdr *msg,int flags);
//成功返回发送字节数，出错返回-1

struct msghdr{
    void *msg_name; /* option address */
    socklent_t msg_namelen; /* address size in bytes */
    struct iovec *msg_iov; /* array of I/O buffers*/
    int msg_iovlen; /* number of element in array */
    void *msg_control; /* ancillary data */
    socklen_t msg_controllen; /* number of ancillary bytes */
  	int msg_flags; /* flags for recived message */
    .
    .
    .
}
```

其中iovec结构详见writev函数章节。

recv和read类似：

```c
#include<sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t nbytes, int flags);
//返回数据字节长度，如果无可用数据或对等方已经按序结束，返回0；如出错，返回-1
```

下表总结了flags使用：

| 标志               | 描述                                               |
| ------------------ | -------------------------------------------------- |
| `MSG_CMSG_CLOEXEC` | 为UNIX域套接字上接收到文件描述符设置执行时关闭标志 |
| `MSG_DONTWAIT`     | 启用非阻塞操作                                     |
| `MSG_ERRQUEUE`     | 接收错误信息作为辅助数据                           |
| `MSG_OOB`          | 如果协议支持，获取外带数据                         |
| `MSG_PEEK`         | 返回数据包内容而不真正取走数据包                   |
| `MSG_TRUNC`        | 即使数据包被截断，也返回数据包的实际长度           |
| `MSG_WAITALL`      | 等待直到所有数据可用（仅SOCK_STREAM)               |

当指定`MSG_PEEK`时，可用查看要读取的数据而不真正取走它，在此调用read或一个recv函数时，会返回刚才查看的数据。对于SOCK_STREAM套接字，接收到的数据可用比预期的少。MSG_WAITALL标志会阻止这种行为，直到请求的数据全部返回，recv函数才会返回。对于其他类型套接字，`MSG_PEEK`标志无作用。

如果发送者调用了shutdown来结束传输，或者网络协议支持默认属性关闭并且发送端已经关闭，那么当所以数据接收完毕后，recv返回0。

使用recvfrom来得到来得到数据发送者的源地址：

```c
#include<sys/socket.h>

ssize_t recvfrom(int sockfd, void *restrict buf, size_t len, int flags, struct sockaddr *restrict addr, socklen_t *restrict addrlen);
//返回数据字节长度，如果无可用数据或对等方已经按序结束，返回0；如出错，返回-1
```

为将数据送入多个缓冲区，类似于readv，可以使用recvmsg：

```c
#include<sys/socket.h>

ssize_t recvmsg(int sockfd, struct msghdr *msg, int flag);
//返回数据字节长度，如果无可用数据或对等方已经按序结束，返回0；如出错，返回-1
```

| 标志           | 描述                   |
| -------------- | ---------------------- |
| `MSG_CTRUNC`   | 控制数据被截断         |
| `MSG_EOR`      | 接收记录结束符         |
| `MSG_ERRQUEUE` | 接收错误信息作为辅助数 |
| `MSG_OOB`      | 接收外带数据           |
| `MSG_TRUNC`    | 一般数据被截断         |

这里我自己写了一个用于文件传输的例程：

服务端：

```c
#include<netdb.h>
#include<errno.h>
#include<arpa/inet.h>
#include<sys/socket.h>
#include<stdlib.h>
#include<unistd.h>
#include<stdio.h>
#include<string.h>
#include<poll.h>
#include<fcntl.h>

#define MAXLEN 1000

int initserver(int type, const struct sockaddr *addr, socklen_t alen, int qlen)
{
    int fd;
    int err = 0;
    if((fd = socket(addr->sa_family,type, 0))<0)
    {
        return -1;
    }

    if(bind(fd, addr, alen)<0)
    {
        goto errout;
    }

    if(type == SOCK_STREAM || type == SOCK_SEQPACKET)
    {
        if(listen(fd, qlen)<0)
        {
            goto errout;
        }
    }
    return fd;

errout:
    err = errno;
    close(fd);
    errno = err;
    return -1;
}
char buf[MAXLEN];
int main()
{
    int sockfd;
    struct in_addr w;
    struct sockaddr_in socketaddr;
    int addr;
    inet_pton(AF_INET, "192.168.100.5", &addr); //替换成自己主机IP地址
    w.s_addr = addr;
    socketaddr.sin_addr = w;
    socketaddr.sin_port = 4301;
    socketaddr.sin_family = AF_INET;
    for(int i=0;i<sizeof(socketaddr.sin_zero);i++)
    {
        socketaddr.sin_zero[i] = '0';
    }
    if((sockfd =initserver(SOCK_STREAM, (sockaddr*)&socketaddr, sizeof(socketaddr), 10))<0)
    {
        printf("initserver error\n");
        return -1;
    }
    int tfd;
    if((tfd = accept(sockfd, NULL,NULL))<0)
    {
        printf("accept error\n");
        return 0;
    }

    
    if(recv(tfd,buf, MAXLEN, 0)<0)
    {
        printf("recv error");
        return -1;
    }

    printf("recv buf %s\n", buf);
    int ffd;
    if((ffd = open(buf, O_RDONLY))<0)
    {
        printf("open error\n");
        return -1;
    }
    // char buf1[MAXLEN];
    // read(ffd, buf, MAXLEN);
    // printf("%s", buf);

    pollfd fdarry[1];
    fdarry[0].fd = tfd;
    fdarry[0].events = POLLOUT;
    // char buf[MAXLEN];
    int end = 0;
    int n;
    int sum = 0;
    while(end == 0 && poll(fdarry, 1, 1000)>0)
    {
        if(fdarry[0].revents == POLLOUT)
        {
            if((n = read(ffd, buf, MAXLEN))<0)
            {
                printf("read error\n");
            }
            sum += n;
            printf("sum = %d\n", sum);
            if(n==0)
            {
                end = 1;
                continue;
            }
            // write(STDOUT_FILENO, buf, n);
            if((send(fdarry[0].fd, buf, n,0))!=n)
            {
                printf("send error\n");
                return -1;
            }
        }
        fdarry[0].revents = 0;
    }

    close(fdarry[0].fd);
    close(sockfd);
    close(ffd);
    // free(fdarry);
    return 0;
}
```

客户端：

```c
#include<netdb.h>
#include<error.h>
#include<syslog.h>
#include<sys/socket.h>
#include<arpa/inet.h>
#include<stdlib.h>
#include<stdio.h>
#include<time.h>
#include<string.h>
#include<unistd.h>
#include<fcntl.h>
#include<poll.h>

#define MAXSLEEP 128
#define MAXLEN 1005
#define	FILE_MODE	(S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

int connect_retry(int domain, int type, int protocol, const struct sockaddr *addr, socklen_t alen)
{
    int numsec, fd;

    for(numsec = 1; numsec<=MAXSLEEP; numsec<<1)
    {
        if((fd=socket(domain, type, protocol))<0)
        {
            return -1;
        }
        if(connect(fd, addr, alen) == 0)
        {
            return fd;
        }

        close(fd);

        if(numsec<= MAXSLEEP/2)
        {
            sleep(numsec);
        }
    }
}

int main(int argv, char *argc[])
{
    int addr;
    struct in_addr w;
    struct sockaddr_in sockaddr;
    sockaddr.sin_family =  AF_INET;
    inet_pton(AF_INET, argc[1], &addr);
    w.s_addr = addr;
    sockaddr.sin_addr = w;
    sockaddr.sin_port = atoi(argc[2]);
    for(int i=0;i<sizeof(sockaddr.sin_zero);i++)
    {
        sockaddr.sin_zero[i] = '0';
    }
    int sockfd;
    if((sockfd = connect_retry(AF_INET, SOCK_STREAM, IPPROTO_TCP, (struct sockaddr*)&sockaddr, sizeof(sockaddr)))<0)
    {
        printf("connect error\n");
        return -1;
    }
    
    if(send(sockfd, argc[3], strlen(argc[3]), 0) != strlen(argc[3]))
    {
        printf("send error\n");
        return -1;
    }
    
    int ffd;
    if((ffd = open(argc[4], O_WRONLY | O_CREAT | O_TRUNC, FILE_MODE))<0) //权限为用户和组读写
    {
        printf("open file error\n");
        return -1;
    }

    pollfd fdarry[1];
    fdarry[0].fd = sockfd;
    fdarry[0].events = POLLIN;
    char buf[MAXLEN];
    int end = 0;
    int n;
    while(end == 0 && poll(fdarry, 1, 1000)>=0)
    {
        if(fdarry[0].revents == POLLIN)
        {
            if((n = recv(fdarry[0].fd, buf, MAXLEN,0))<0)
            {
                printf("recv error\n");
                return -1;
            }
            
            if(n==0)
            {
                end =1;
                continue;
            }

            // write(STDOUT_FILENO, buf, n);
            if(write(ffd, buf, n)!=n)
            {
                printf("write error\n");
                return 0;
            }
            
        }
        fdarry[0].revents = 0;
    }

    close(fdarry[0].fd);
    close(ffd);
    close(sockfd);
    // free(fdarry);
    return 0;
}
```

这里客户端参数分别是：服务器IP地址，端口号，要获取的文件，文件存储位置。注意，由于NAT的存在，该程序只能在连接同一路由器的处于同一局域网的两台电脑之间进行文件传输。

下面是书本中的程序：

一个与服务器通信的客户端从（服务器）系统的uptime命令获得输出，我们称之为“远程正常运行时间”（remote uptime，ruptime）。

南向连接的客户端：

```c
#include"apue.h"
#include<netdb.h>
#include<errno.h>
#include<sys/socket.h>

#define BUFLEN 128

extern int connect_retry(int , int, int, const struct sockaddr*, socklen_t);

void print_uptime(int sockfd)


{
    int n;
    char buf[BUFLEN];

    while((n = recv(sockfd, buf, BUFLEN, 0))>0)
    {
        write(STDOUT_FILENO, buf, n);
    }
    if(n<0)
    {
        err_sys("recv error");
    }
}

int main(int argc, char *argv[])
{
    struct addrinfo *ailist, *aip;
    struct addrinfo hint;
    int sockfd, err;

    if(argc != 2)
    {
        err_quit("usage: reptime hostname");
    }

    memset(&hint, 0, sizeof(hint));

    hint.ai_socktype = SOCK_STREAM;
    hint.ai_canonname = NULL;
    hint.ai_addr = NULL;
    hint.ai_next = NULL;
    if((err = getaddrinfo(argv[1], "ruptime",&hint,&ailist))!=0)
    {
        err_quit("getaddrinfo error:%s", gai_strerror(err));
    }
    for(aip = ailist; aip !=NULL; aip = aip->ai_next)
    {
        if((sockfd = connect_retry(aip->ai_family, SOCK_STREAM, 0, aip->ai_addr, aip->ai_addrlen))<0)
        {
            err = errno;
        }
        else{
            printf("connect success\n");
            print_uptime(sockfd);
            exit(0);
        }
    }
    err_exit(err, "can't connect to %s", argv[1]);
}
```

面向连接的服务器：

```
#include "apue.h"
#include <netdb.h>
#include <errno.h>
#include <syslog.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <fcntl.h>

#define BUFLEN 128
#define QLEN 10

#ifndef HOST_NAME_MAX
#define HOST_NAME_MAX 128
#endif

extern int initserver(int, const struct sockaddr *, socklen_t, int);

int set_cloexec(int fd)
{
    int val;
    if ((val = fcntl(fd, F_GETFD, 0)) < 0)
    {
        return -1;
    }
    val |= FD_CLOEXEC;
    return fcntl(fd, F_SETFD, val);
}

void serve(int sockfd)
{
    int clfd;
    FILE *fp;
    char buf[BUFLEN];

    set_cloexec(sockfd);
    while (1)
    {
        if ((clfd = accept(sockfd, NULL, NULL)) < 0)
        {
            syslog(LOG_ERR, "ruptimeed: accept error:%s", strerror(errno));
            exit(0);
        }
        syslog(LOG_INFO, "connect success");
        set_cloexec(clfd);
        if ((fp = popen("/usr/bin/uptime", "r")) == NULL)
        {
            sprintf(buf, "error:%s\n", strerror(errno));
            syslog(LOG_INFO, "send %s", buf);
            send(clfd, buf, strlen(buf), 0);
        }
        else
        {
            syslog(LOG_INFO, "open uptime succeed");
            while (fgets(buf, 20, fp) != NULL)
            {
                syslog(LOG_INFO, "send %s", buf);
                send(clfd, buf, strlen(buf), 0);
            }
            pclose(fp);
        }
        close(clfd);
    }
}
char buf1[20];
int main(int argc, char *argv[])
{
    struct addrinfo *ailist, *aip;
    struct addrinfo hint;
    int sockfd, err, n;
    char *host;

    if (argc != 1)
    {
        err_quit("usage: ruptimed");
    }
    if ((n = sysconf(_SC_HOST_NAME_MAX)) < 0)
    {
        n = HOST_NAME_MAX;
    }

    if ((host = (char *)malloc(n)) == NULL)
    {
        err_sys("malloc error");
    }
    if (gethostname(host, n) < 0)
    {
        err_sys("gethostname error");
    }
    printf("hostname is %s\n", host);

    daemonize("ruptimed"); //将进程转换为守护进程

    memset(&hint, 0, sizeof(hint));

    hint.ai_flags = AI_PASSIVE;
    hint.ai_socktype = SOCK_STREAM;
    hint.ai_canonname = NULL;
    hint.ai_addr = NULL;
    hint.ai_next = NULL;

    if ((err = getaddrinfo(host, "ruptime", &hint, &ailist)) != 0)
    {
        syslog(LOG_ERR, "ruptimed: getaddrinfo error:%s", gai_strerror(err));
        exit(1);
    }

    // struct sockaddr *aip;

    for (aip = ailist; aip != NULL; aip = aip->ai_next)
    {
        struct sockaddr_in *skin = (struct sockaddr_in *)ailist->ai_addr;
        if (inet_ntop(AF_INET, &(skin->sin_addr), buf1, 20) < 0)
        {
            syslog(LOG_INFO, "inet_ntop error\n");
        }
        else
        {
            syslog(LOG_INFO, "addr is %s, port is %d", buf1, ntohs(skin->sin_port));
        }
    }

    for (aip = ailist; aip != NULL; aip = aip->ai_next)
    {
        if ((sockfd = initserver(SOCK_STREAM, aip->ai_addr, aip->ai_addrlen, QLEN)) >= 0)
        {
            serve(sockfd);
            exit(0);
        }
    }
    exit(0);
}
```

这里存在一些细节问题需要注意：

在服务器程序中：unix下是没有ruptime服务的，直接调用`getaddrinfo`是无法成功的，应该先在`/etc/services`中添加ruptime服务，比如我是这么添加的ruptime   40031/tcp。其次，`getaddrinfo`主要目的是实现一次DNS查询，即知道主机名获取对应的IP地址。对于端口号来说，其实并不进行对应查询，它只能查询自己本地存在的服务，因为访问的端口号都是一致的（否则就是错误的服务，无法正常运行），因此不需要查询，只需要查询IP地址。而根据这篇博客http://luodw.cc/2015/12/27/dns02/可知，其进行查询时，首先会查询本地DNS会先到配置文件/etc/resolv.conf文件查到本地的dns服务器ip地址,然后向本地dns服务器建立udp请求信息,获取信息.ubuntu默认的本地dns服务器地址是127.0.1.1,也就是本地的dnsmasq守护进程。127.0.1.1也是一个本地回环地址,而dnsmasq进程监听的正是这个地址以及53端口号.我们也可以配置其他计算机为本地DNS域名服务器。如果dnsmasq没有相关的ip地址,那么dnsmasq会向其他域名服务器查询,最后返回到本地域名服务器缓存中。因此如果直接调用`getaddrinfo`获得的地址将是127.0.1.1，将其转换为当前主机的实际网络IP地址时，这样建立的服务器才能被服务器进程所访问到。最后，这里将服务器进程变成了守护进程，因此输出只能到syslog中，要查询，应该到查询`/var/log/syslog`文件。

在客户程序中：如上面所述的，`getaddrinfo`只支持查询自己主机支持的服务，因此也需要将ruptime加入到`/etc/services`。

另一个面向连接的服务器：

```c
#include "apue.h"
#include <netdb.h>
#include <errno.h>
#include <syslog.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include<sys/wait.h>

#define BUFLEN 128
#define QLEN 10

#ifndef HOST_NAME_MAX
#define HOST_NAME_MAX 128
#endif

extern int initserver(int, const struct sockaddr *, socklen_t, int);

int set_cloexec(int fd)
{
    int val;
    if ((val = fcntl(fd, F_GETFD, 0)) < 0)
    {
        return -1;
    }
    val |= FD_CLOEXEC;
    return fcntl(fd, F_SETFD, val);
}

void serve(int sockfd)
{
    int clfd, status;
    pid_t pid;

    set_cloexec(sockfd);
    while (1)
    {
        if ((clfd = accept(sockfd, NULL, NULL)) < 0)
        {
            syslog(LOG_ERR, "ruptimeed: accept error:%s", strerror(errno));
            exit(0);
        }
        if((pid = fork())<0)
        {
            syslog(LOG_ERR, "ruptined: fork error:%s", strerror(errno));
        }
        else{
            if(pid == 0) /* child */
            {
                if(dup2(clfd, STDOUT_FILENO) != STDOUT_FILENO || dup2(clfd, STDERR_FILENO) != STDERR_FILENO)
                {
                    syslog(LOG_ERR, "ruptime:unexpected error");
                    exit(0);
                }
                close(clfd);
                execl("/usr/bin/uptime", "uptime", (char*)0);
                syslog(LOG_ERR, "ruptime: unexpected return form exec:%s", strerror(errno));
            }
            else{
                 close(clfd);
                 waitpid(pid, &status, 0);
            }
            
        }
    }    
}
char buf1[20];
int main(int argc, char *argv[])
{
    struct addrinfo *ailist, *aip;
    struct addrinfo hint;
    int sockfd, err, n;
    char *host;

    if (argc != 1)
    {
        err_quit("usage: ruptimed");
    }
    if ((n = sysconf(_SC_HOST_NAME_MAX)) < 0)
    {
        n = HOST_NAME_MAX;
    }

    if ((host = (char *)malloc(n)) == NULL)
    {
        err_sys("malloc error");
    }
    if (gethostname(host, n) < 0)
    {
        err_sys("gethostname error");
    }
    printf("hostname is %s\n", host);

    daemonize("ruptimed"); //将进程转换为守护进程

    memset(&hint, 0, sizeof(hint));

    hint.ai_flags = AI_PASSIVE;
    hint.ai_socktype = SOCK_STREAM;
    hint.ai_canonname = NULL;
    hint.ai_addr = NULL;
    hint.ai_next = NULL;

    if ((err = getaddrinfo(host, "ruptime", &hint, &ailist)) != 0)
    {
        syslog(LOG_ERR, "ruptimed: getaddrinfo error:%s", gai_strerror(err));
        exit(1);
    }

    // struct sockaddr *aip;

    for (aip = ailist; aip != NULL; aip = aip->ai_next)
    {
        struct sockaddr_in *skin = (struct sockaddr_in *)ailist->ai_addr;
        if (inet_ntop(AF_INET, &(skin->sin_addr), buf1, 20) < 0)
        {
            syslog(LOG_INFO, "inet_ntop error\n");
        }
        else
        {
            syslog(LOG_INFO, "addr is %s, port is %d", buf1, ntohs(skin->sin_port));
        }
    }

    for (aip = ailist; aip != NULL; aip = aip->ai_next)
    {
        if ((sockfd = initserver(SOCK_STREAM, aip->ai_addr, aip->ai_addrlen, QLEN)) >= 0)
        {
            serve(sockfd);
            exit(0);
        }
    }
    exit(0);
}
```

这里与之前版本的服务器存在两个差异，一个是收到连接请求后，创建一个子进程来处理请求，第二个是不是调用popen来获取程序输出，而是绑定描述符。将子进程的输出和错误描述符绑定到套接字描述符。

无连接的客户端：

```c
#include"apue.h"
#include<netdb.h>
#include<errno.h>
#include<sys/socket.h>

#define BUFLEN 128
#define TIMEOUT 20

extern int connect_retry(int , int, int, const struct sockaddr*, socklen_t);

void sigalrm(int signo)
{

}

void print_uptime(int sockfd, struct addrinfo *aip)
{
    int n;
    char buf[BUFLEN];
    buf[0] = 0;

    if(sendto(sockfd, buf, 1, 0, aip->ai_addr, aip->ai_addrlen)<0)
    {
        err_sys("sendto error");
    }
    alarm(TIMEOUT);
    if((n = recvfrom(sockfd, buf, BUFLEN, 0, NULL, NULL))<0)
    {
        if(errno != EINTR)
        {
            alarm(0);
        }
        err_sys("recv error");
    }
    alarm(0);
    write(STDOUT_FILENO, buf, n);
}

int main(int argc, char *argv[])
{
    struct addrinfo *ailist, *aip;
    struct addrinfo hint;
    int sockfd, err;
    struct sigaction sa;

    if(argc != 2)
    {
        err_quit("usage: reptime hostname");
    }

    sa.sa_handler = sigalrm;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);

    if(sigaction(SIGALRM, &sa, NULL)<0)
    {
        err_sys("sigaction error");
    }

    memset(&hint, 0, sizeof(hint));

    hint.ai_socktype = SOCK_DGRAM;
    hint.ai_canonname = NULL;
    hint.ai_addr = NULL;
    hint.ai_next = NULL;
    if((err = getaddrinfo(argv[1], "ruptime",&hint,&ailist))!=0)
    {
        err_quit("getaddrinfo error:%s", gai_strerror(err));
    }
    for(aip = ailist; aip !=NULL; aip = aip->ai_next)
    {
        if((sockfd = connect_retry(aip->ai_family, SOCK_STREAM, 0, aip->ai_addr, aip->ai_addrlen))<0)
        {
            err = errno;
        }
        else{
            printf("connect success\n");
            print_uptime(sockfd, aip);
            exit(0);
        }
    }
    err_exit(err, "can't connect to %s", argv[1]);
}
```

无连接服务器：

```c
#include "apue.h"
#include <netdb.h>
#include <errno.h>
#include <syslog.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <fcntl.h>

#define BUFLEN 128
#define MAXADDRLEN 256

#ifndef HOST_NAME_MAX
#define HOST_NAME_MAX 128
#endif

extern int initserver(int, const struct sockaddr *, socklen_t, int);

int set_cloexec(int fd)
{
    int val;
    if ((val = fcntl(fd, F_GETFD, 0)) < 0)
    {
        return -1;
    }
    val |= FD_CLOEXEC;
    return fcntl(fd, F_SETFD, val);
}

void serve(int sockfd)
{
    int n;
    socklen_t alen;
    FILE *fp;
    char buf[BUFLEN];
    char abuf[MAXADDRLEN];
    struct sockaddr *addr = (struct sockaddr *)abuf;


    set_cloexec(sockfd);
    while (1)
    {   
        alen = MAXADDRLEN;
        if((n = recvfrom(sockfd, buf, BUFLEN, 0, addr, &alen))<0)
        {
            syslog(LOG_ERR, "ruptimed: recvfrom error:%s", strerror(errno));
            exit(0);
        }
        syslog(LOG_INFO, "connect success");
        if ((fp = popen("/usr/bin/uptime", "r")) == NULL)
        {
            sprintf(buf, "error:%s\n", strerror(errno));
            syslog(LOG_INFO, "send %s", buf);
            sendto(sockfd, buf, strlen(buf), 0, addr, alen);
        }
        else
        {
            syslog(LOG_INFO, "open uptime succeed");
            while (fgets(buf, 20, fp) != NULL)
            {
                syslog(LOG_INFO, "send %s", buf);
                sendto(sockfd, buf, strlen(buf), 0, addr, alen);
            }
            pclose(fp);
        }
    }
}
char buf1[20];
int main(int argc, char *argv[])
{
    struct addrinfo *ailist, *aip;
    struct addrinfo hint;
    int sockfd, err, n;
    char *host;

    if (argc != 1)
    {
        err_quit("usage: ruptimed");
    }
    if ((n = sysconf(_SC_HOST_NAME_MAX)) < 0)
    {
        n = HOST_NAME_MAX;
    }

    if ((host = (char *)malloc(n)) == NULL)
    {
        err_sys("malloc error");
    }
    if (gethostname(host, n) < 0)
    {
        err_sys("gethostname error");
    }
    printf("hostname is %s\n", host);

    daemonize("ruptimed"); //将进程转换为守护进程

    memset(&hint, 0, sizeof(hint));

    hint.ai_flags = AI_PASSIVE;
    hint.ai_socktype = SOCK_DGRAM;
    hint.ai_canonname = NULL;
    hint.ai_addr = NULL;
    hint.ai_next = NULL;

    if ((err = getaddrinfo(host, "ruptime", &hint, &ailist)) != 0)
    {
        syslog(LOG_ERR, "ruptimed: getaddrinfo error:%s", gai_strerror(err));
        exit(1);
    }

    // struct sockaddr *aip;

    for (aip = ailist; aip != NULL; aip = aip->ai_next)
    {
        struct sockaddr_in *skin = (struct sockaddr_in *)ailist->ai_addr;
        if (inet_ntop(AF_INET, &(skin->sin_addr), buf1, 20) < 0)
        {
            syslog(LOG_INFO, "inet_ntop error\n");
        }
        else
        {
            syslog(LOG_INFO, "addr is %s, port is %d", buf1, ntohs(skin->sin_port));
        }
    }

    for (aip = ailist; aip != NULL; aip = aip->ai_next)
    {
        if ((sockfd = initserver(SOCK_STREAM, aip->ai_addr, aip->ai_addrlen, 0)) >= 0)
        {
            serve(sockfd);
            exit(0);
        }
    }
    exit(0);
}
```

这里也存在一些问题：与有链接的服务一样，首先我们一个在`/etc/services`中添加无连接的ruptime服务，如ruptime 40031/udp。其次这里客户端与服务器都存在一些区别，首先，对于无连接的服务器的来说，应该先由客户发送一个请求，服务器通过请求来确定来源，以此返回信息。这里使用客户机先发送一个字节作为请求，服务器接收到该请求，返回uptime程序执行结果。在客户机中使用了时钟信号，用来控制发送请求和接收返回的时间间隔。

## 套接字选项

套接字机制提供了两个套接字选项接口来控制套接字行为。一个用来设置选项，另一个接口可以查询选项的状态，可以获取或设置以下3中选项：

1. 通用选项，工作在所有套接字类型上。
2. 在套接字层次管理的选项，但是依赖于下层协议的支持。
3. 特定于某协议的选项，每个协议独有。

使用setsockopt函数来设置套接字选项：

```c
#include<sys/socket.h>

int setsockopt(int sockfd, int level, int option, const void *val, socklen_t len);
//成功返回0，出错返回-1
```

level标识选项应用的协议。对于通用套接字层次选项，level设置为SOL_SOCKET。否则，level设置成控制这个选项的协议编号。对于TCP选项，level是IPPROTO_TCP，对于IP，level是IPPROTO_IP。下表总结了通用套接字层次选项：

![16-21](https://s2.ax1x.com/2020/02/17/3PBMSe.png)

参数val根据选项的不同指向一个数据结构或者一个整数。一些选项是off/on开关。如果整数非0，则启用选项，如果是0，关闭选项。len指定了val指向对象的大小。

使用getsockopt函数来查看选项的当前值：

```c
#include<sys/socket.h>

int getsockopt(int sockfd, int level, int option, void *restrict val, socklen_t *restrict lenp);
//成功返回0，出错返回-1
```

参数lenp是一个指向整数的指针。调用该函数之前，该整数为缓冲区长度。如果选项从实际长度大于该值，选项会被截断。如果实际长度小于该值，那么返回时该值会被更新为实际长度。

之前的initserver服务器在终止并立即重启会无法正常工作，因为除非超时（一般几分钟），否则TCP的实现不允许绑定到同一地址。然而套接字选择SO_REUSEADDR可以绕过该限制：

新的initserver：

```c
#include"apue.h"
#include<errno.h>
#include<sys/socket.h>

int initserver(int type, const struct sockaddr *addr, socklen_t alen, int qlen)
{
    int fd;
    int err = 0;
    int reuse;
    if((fd = socket(addr->sa_family,type, 0))<0)
    {
        return -1;
    }
    if(setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(int))<0)
    {
        goto errout;
    }
    if(bind(fd, addr, alen)<0)
    {
        goto errout;
    }

    if(type == SOCK_STREAM || type == SOCK_SEQPACKET)
    {
        if(listen(fd, qlen)<0)
        {
            goto errout;
        }
    }
    return fd;

errout:
    err = errno;
    close(fd);
    errno = err;
    return -1;
}
```

## 带外数据

带外数据（out-of-band data）是一些通信协议所支持的可选功能，与普通数据相比，它允许更高优先级的数据传输。外带数据先行传输，即使传输队列已经有数据。TCP支持外带数据。TCP将外带数据称为紧急数据（urgent data）。TCP仅支持一个字节的紧急数据，但是允许紧急数据在普通数据传输机制之外传输。为了产生紧急数据，可以在3个send函数中的任一个里指定MSG_OOB标志。如果带MSG_OOB标志发送的字节数超过一个时，最后一个字节将被视为紧急数据字节。

如果套接字安排了信号的产生，那么当紧急数据被接收时，会发送SIGURG信号。在十四章中，我们学到在fcntl中使用F_SETOWN命令来设置一个套接字所有权。如果第三个参数为正，那么指定的就是进程ID，如果为非-1的负数，那么绝对值代表的就是进程组ID。因此，可以使用下面的函数安排进程接收套接字的信号：

```
fcntl(sockfd, F_SETOWN, pid);
```

F_GETOWN可以用来获取当前套接字所有权，其返回值与F_SETOWN含义一致，因此调用：

```
owner = fcntl(sockfd, F_GETOWN, 0);
```

获得拥有者。

TCP支持紧急标记（urgent mark）概念，即在普通数据流中紧急数据所在位置。如果采用套接字选项SO_OOBINLINE，那么可以在普通数据中接收紧急数据。sockatmark函数用来判断是否已到达紧急标志：

```c
#include<sys/socket.h>

int sockatmark(int sockfd);
//在标记处，返回1，未在标记处，返回-1，出错返回-1
```

带外数据出现在套接字队列时，select函数会返回一个文件描述符并且有一个待处理的异常条件。可以在普通数据流上接收紧急数据，也可以在其中一个recv函数中采用MSG_OOB标志在其他队列数据之前接收紧急数据。如果在接收当前紧急数据之前，又有一个紧急数据到来，则原来的紧急数据将会被丢弃。

## 非阻塞和异步I/O

recv在没有数据时会阻塞，套接字输出队列没有足够空间来发送消息时，send函数会阻塞。在套接字非阻塞模式下，函数不会阻塞而是失败，将errno设置为EWOULDBLOCK或EAGAIN。当这种情况发送时，可以使用poll或select来判断能否接收或者传输数据。

在基于套接字的异步I/O中，当从套接字中读取数据时，或者当套接字写队列中空间变得可用时，可以安排要发送的信号SIGIO。启用异步I/O分为两步：

1. 建立套接字所有权，这样信号可以可以被传递到合适的进程。
2. 通知套接字I/O操作包含阻塞时发信号。

完成第一步有3种方式：

1. 在fcntl中使用F_SETOWN命令。
2. 在ioctl中使用FIOSETOWN。
3. 在ioctl中使用SIOCSPGRP。

完成第二步有两个选择：

1. 在fcntl中使用F_SETFL命令并且启用文件标志O_ASYNC。
2. 在ioctl中使用FIOASYNC命令。

## 习题16.6 

写两个库例程，一个在套接字允许异步I/O，一个不允许异步I/O：

```c
#include<sys/ioctl.h>
#include"apue.h"
#include<errno.h>
#include<fcntl.h>
#include<sys/socket.h>

int setasync(int sockfd)
{
    int n;
    if(fcntl(sockfd, F_SETOWN, getpid())<0)
    {
        return -1;
    }
    n= 1;
    if(ioctl(sockfd, FIOASYNC, &n)<0)
    {
        return -1;
    }
    return 0;
}

int clrasync(int sockfd)
{
    int n;

    n = 0;
    if(ioctl(sockfd, FIOASYNC, &n)<0)
    {
        return -1;
    }
    return 0;

}
```

# 第十七章 高级进程间通信

## UNIX域套接字

UNIX域套接字用于在同一台计算机上运行的进程之间的通信。因特网域套接字也可以实现，但UNIX域套接字的效率更高。UNIX域套接字提供流和数据报两种接口。UNIX域数据报服务是可靠的，即不会丢失报文也不会传输错误。UNIX域套接字像是套接字和管道的混合，可以使用面向网络的域套接字接口或者使用socketpair函数来创建一对无命名的、相互连接的UNIX域套接字：

```c
#include<sys/socket.h>

int socketpair(int domain, int type, int protocol, int sockfd[2]);
//成功返回0，失败返回-1
```

一对相互连接的UNIX域套接字可以起到全双工管道的作用：两端对读和写开发（如下图）。我们称其为fd管道，以便与普通半双工管道区分开来。

![17-1](https://s2.ax1x.com/2020/02/18/3FcYtA.png)

实例：fd_pipe函数：

```c
#include"apue.h"
#include<sys/socket.h>

int fd_pipe(int fd[2])
{
    return(socketpair(AF_UNIX, SOCK_STREAM, 0, fd));
}
```

实例：借助UNIX域套接字轮询XSI消息队列

XSI消息队列的使用存在一个问题，即不能使用poll或者select一起使用，因为它们不能关联描述符。套接字和文件描述符是关联的，消息到达时，可以用套接字来通知。对每个消息队列使用一个线程，每个线程都会在msgrcv调用中阻塞。当消息达到时，线程会把它写入一个UNIX域套接字一端。

```c
#include"apue.h"
#include<poll.h>
#include<pthread.h>
#include<sys/msg.h>
#include<sys/socket.h>

#define NQ 3
#define MAXMSZ 512
#define KEY 0x123

struct threadinfo{
    int qid;
    int fd;
};

struct mymesg{
    long mtype;
    char mtext[MAXMSZ];
};

void *helper(void *arg)
{
    int n;
    struct mymesg m;
    struct threadinfo *tip = (threadinfo *)arg;
    while(1)
    {
        memset(&m, 0,sizeof(m));
        if((n = msgrcv(tip->qid, &m, MAXMSZ, 0, MSG_NOERROR))<0)
        {
            err_sys("msgrcv error");
        }
        if(write(tip->fd, m.mtext, n)<0)
        {
            err_sys("write error");
        }
    }
}

int main()
{
    int i, n, err;
    int fd[2];
    int qid[NQ];
    struct pollfd pfd[NQ];
    struct threadinfo ti[NQ];
    pthread_t tid[NQ];
    char buf[MAXMSZ];

    for(i = 0; i< NQ; i++)
    {
        if((qid[i] = msgget((KEY+i), IPC_CREAT | 0666))<0)
        {
            err_sys("msgget error");
        }

        printf("queue ID %d is %d\n", i, qid[i]);

        if(socketpair(AF_UNIX, SOCK_DGRAM, 0, fd)<0)
        {
            err_sys("socketpair error");
        }

        pfd[i].fd = fd[0];
        pfd[i].events = POLLIN;
        ti[i].qid = qid[i];
        ti[i].fd = fd[1];
        if((err = pthread_create(&tid[i], NULL, helper, &ti[i]))!=0)
        {
            err_exit(err, "pthread_create error");
        }
    }
    while(1)
    {
        if(poll(pfd, NQ, -1)<0)
        {
            err_sys("poll error");
        }

        for(i=0;i<NQ;i++)
        {
            if(pfd[i].revents & POLLIN)
            {
                if((n=read(pfd[i].fd, buf, sizeof(buf)))<0)
                {
                    err_sys("read error");
                }
                buf[n] = 0;
                printf("queue id %d, message %s\n", qid[i], buf);
            }
        }
    }
}
```

使用下面的程序给上面的消息队列发送消息：

```c
#include"apue.h"
#include<sys/msg.h>

#define MAXMSZ 512

struct mymesg{
    long mtype;
    char mtext[MAXMSZ];
};

int main(int argc, char *argv[])
{
    key_t key;
    long qid;
    long pid;
    size_t nbytes;
    struct mymesg m;
    if(argc != 3)
    {
        fprintf(stderr, "usgae: sendmsg KEY message\n");
        exit(1);
    }

    key = strtol(argv[1], NULL, 0);
    if((qid = msgget(key, 0))<0)
    {
        err_sys("can't open queue key %s", argv[1]);
    }

    memset(&m, 0, sizeof(m));
    strncpy(m.mtext, argv[2], MAXMSZ-1);
    nbytes = strlen(m.mtext);
    m.mtype = 1;
    if(msgsnd(qid, &m, nbytes, 0)<0)
    {
        err_sys("can't send messgae");
    }
    exit(0);
}
```

程序执行结果：

```shell
$ ./17-3 &
[2] 19237
[1]   Terminated              ./17-3
queue ID 0 is 0
queue ID 1 is 32769
chst@wyk-GL63:~/study_file/unix编程$ queue ID 2 is 65538

chst@wyk-GL63:~/study_file/unix编程$ ./17-4 0x123 "hello world"
queue id 0, message hello world
chst@wyk-GL63:~/study_file/unix编程$ ./17-4 0x124 "just a test"
queue id 32769, message just a test
chst@wyk-GL63:~/study_file/unix编程$ ./17-4 0x125 "bye"
queue id 65538, message bye
```



<font size=8>命名UNIX域套接字</font>

socketpair函数创建的一对互联的套接字没有名字，这意味着无关进程不能使用它们。与上一章一样，我们可以命名UNIX域套接字，并可将其用于告示服务。UNIX域套接字使用的地址格式为：

```c
#include<sys/un.h>

struct sockaddr_un{
	sa_family_t sun_family; /* AF_UNIX */
	char sun_path[108]; /* pathname */
}
```

sockaddr_un结构的sun_path成员包含一个路径名。当我们将一个地址绑定到一个UNIX域套接字时，系统会用该路径名创建一个S_IFSOCK类型的文件。该文件仅用于向客户进程告示该套接字名字。该文件无法打开，也不能由应用程序用于通信。

如果我们试图绑定同一地址时，该文件已经存在，那么bind会失败。当关闭套接字时，并不自动删除该文件，所以必须确保在应用程序退出之前，对该文件解除链接操作。

实例：

```c
#include"apue.h"
#include<sys/socket.h>
#include<sys/un.h>

int main()
{
    int fd, size;
    struct sockaddr_un un;
    un.sun_family = AF_UNIX;
    strcpy(un.sun_path, "foo.socket");
    if((fd = socket(AF_UNIX, SOCK_STREAM, 0))<0)
    {
        err_sys("socket error");
    }

    size = offsetof(struct sockaddr_un, sun_path) + strlen(un.sun_path);
    if(bind(fd, (struct sockaddr*)&un, size)<0)
    {
        err_sys("bind failed");
    }
    printf("UNIX domain socket bound\n");
    exit(0);
}
```

这里执行：

```
$ ./17-5
UNIX domain socket bound
$ ./17-5
bind failed: Address already in use
chst@wyk-GL63:~/study_file/unix编程$ rm foo.socket
chst@wyk-GL63:~/study_file/unix编程$ ./17-5
UNIX domain socket bound
```

这里由于程序没有对生成的文件进行解除链接，再次执行时，bind将会出错。

确定绑定地址长度的方法是，先计算成员在sockaddr_un结构中的偏移量，然后将结果与路径名长度（不包括终止null字符）相加。其中：

```
#define offsetof(TYPE, MEMBER) ((int)&((TYPE *)0)->MEMBER)
```

## 唯一连接

下图分别展示了建立连接之前和建立之后的情形：

![17-67](https://s2.ax1x.com/2020/02/21/3nHyJH.png)

这里我们类别于因特网域，构建三个函数：

```c
#include"apue.h"

int serv_listen(const char *name);
//成功返回监视的描述符，出错返回负值

int serv_accept(int listenfd, uid_t *uidptr);
//成功返回新的文件描述符，出错返回负值

int cli_conn(const char *name);
//成功返回文件描述符，出错返回负值
```

下面先给出serv_listen函数：

```c
#include"apue.h"
#include<sys/socket.h>
#include<sys/un.h>
#include<errno.h>

#define QLEN 10

int serv_listen(const char *name)
{
    int fd, len, err, rval;
    struct sockaddr_un un;
    if(strlen(name) >= sizeof(un.sun_path))
    {
        errno = ENAMETOOLONG;
        return -1;
    }

    if((fd = socket(AF_UNIX, SOCK_STREAM, 0))<0)
    {
        return -2;
    }

    unlink(name);

    memset(&un, 0, sizeof(un));
    un.sun_family  = AF_UNIX; 
    strcpy(un.sun_path, name);
    len = offsetof(struct sockaddr_un, sun_path) + strlen(name);

    if(bind(fd, (struct sockaddr*)&un, len)<0)
    {
        rval = -3;
        goto errout;
    }

    if(listen(fd, QLEN)<0)
    {
        rval = -4;
        goto errout;
    }
    return fd;

errout:
    err = errno;
    close(fd);
    errno = err;
    return rval;
}
```

serv_accept:

```c
#include"apue.h"
#include<sys/socket.h>
#include<sys/un.h>
#include<time.h>
#include<errno.h>

#define STALE 30 /* client't name can't be older than this (sec) */

int serv_accept(int listenfd, uid_t *uidptr)
{
    int clifd, err, rval;
    socklen_t len;
    time_t staletime;
    struct sockaddr_un un;
    struct stat statbuf;
    char *name;

    if((name = (char*)malloc(sizeof(un.sun_path)+1)) == NULL)
    {
        return -1;
    }
    len = sizeof(un);
    if((clifd = accept(listenfd, (struct sockaddr *)&un, &len))<0)
    {
        free(name);
        return -2;
    }

    len -= offsetof(struct sockaddr_un, sun_path);
    memcpy(name, un.sun_path, len);
    name[len] = 0;
    if(stat(name, &statbuf)<0)
    {
        rval = -3;
        goto errout;
    }

#ifdef S_ISSOCK
    if(S_ISSOCK(statbuf.st_mode) == 0)
    {
        rval = -4;
        goto errout;
    }
#endif

    if((statbuf.st_mode & (S_IRWXG | S_IRWXO)) || (statbuf.st_mode & S_IRWXU)!=S_IRWXU)
    {
        rval = -5;
        goto errout;
    }
    staletime = time(NULL) - STALE;

    if(statbuf.st_atime < staletime || statbuf.st_ctime < staletime || statbuf.st_mtime<staletime)
    {
        rval = -6;
        goto errout;
    } 

    if(uidptr != NULL)
    {
        *uidptr = statbuf.st_uid; /* return uid of caller */
    }

    unlink(name);
    free(name);
    return clifd;

errout:
    err = errno;
    close(clifd);
    free(name);
    errno = err;
    return rval;

}

```

这里我们要使用stat函数验证：该路径确实是一个套接字；其权限仅允许用户读、用户写以及用户执行。还有验证与套接字相关联的3个时间参数不比当前时间早30秒。

客户进程调用cli_conn函数对连接到服务器进程的连接进行初始化：

```c
#include"apue.h"
#include<sys/socket.h>
#include<sys/un.h>
#include<errno.h>

#define CLI_PATH "/var/tmp"
#define CLI_PERM S_IRWXU /* rwx for user only */

int cli_conn(const char *name)
{
    int fd, len, err, rval;
    struct sockaddr_un un,sun;
    int do_unlink = 0;

    if(strlen(name) >= sizeof(un.sun_path))
    {
        errno = ENAMETOOLONG;
        return -1;
    }

    /* create a UNIX domain stream socker */
    if((fd = socket(AF_UNIX, SOCK_STREAM, 0))<0)
    {
        return -1;
    }

    /* fill socket address structure with our address */
    memset(&un, 0, sizeof(un));
    un.sun_family = AF_UNIX;
    sprintf(un.sun_path, "%d%05ld", CLI_PATH, (long)getpid());
    len = offsetof(struct sockaddr_un, sun_path) + strlen(un.sun_path);

    unlink(un.sun_path); /* in case it already exists */

    if(bind(fd, (struct sockaddr *)&un, len)<0)
    {
        rval = -2;
        goto errout;
    }

    if(chmod(un.sun_path, CLI_PERM)<0)
    {
        rval = -3;
        do_unlink = 1;
        goto errout;
    }

    /* fill socket address structure with server's address */
    memset(&sun, 0, sizeof(sun));
    sun.sun_family = AF_UNIX;
    strcpy(sun.sun_path,name);
    len = offsetof(struct sockaddr_un,sun_path)+strlen(name);
    if(connect(fd, (struct sockaddr *)&sun, len)<0)
    {
        rval = -4;
        do_unlink = 1;
        goto errout;
    }
    return fd;

errout:
    err = errno;
    close(fd);
    if(do_unlink)
    {
        unlink(un.sun_path);
    }
    errno = err;
    return rval;
}
```

## 传送文件描述符

一个进程向另一个进程传递文件描述符含义如下图：

![17-11](https://s2.ax1x.com/2020/02/25/3YHjQP.png)

计数上，我们是将指向一个打开文件表项的指针从一个进程发送到另一个进程。该指针被分配在存放在接收进程的第一个第一个可用文件表项中。（发送进程和接收进程的描述符编号没有关系，通常不相同）。两个进程共享同一个打开表项，与fork后的父进程和子进程共享打开文件表项的情况一致。

下面定义本章用以发送和接收文件描述符的3个函数：

```
#include"apue.h"

int send_fd(int fd, int fd_to_send);
int send_err(int fd, int status, const char *errmsg);
//成功返回0，出错返回-1;

int recv_fd(int fd, ssize_t (*userfunc)(int, const void*, size_t));
//成功返回文件描述符，出错返回负值
```

其中send_fd和send_err用来发送文件描述符，recv_fd用于接收描述符。send_fd中参数fd表示与另一个进程通信的文件描述符，而fd_to_send表示要被传输的文件描述符。recv_fd正常情况下返回非负文件描述符，如果发送方发送的是send_err则返回send_err发送的负值。

send_err发送status（-1～-255）的值。另外，如果服务器发出一条出错信息，客户进程调用自己的userfunc函数来处理该消息。

为实现上述三个函数，需要自定义一套协议。对于发送一个文件描述符，send_fd先发送两字节0，然后是实际描述符。为了发送一条出错信息，send_err发送errmsg，然后是1字节0，最后是status的绝对值（1～255）。recv_fd函数读取套接字中所有字节，直到遇到null字符。（对于send_err来说，这部分读取到的是errmsg，对于send_fd来说，为空），因此发送的两个字节，第一个为0，用于分割errmsg。null之前的字符全部传递给调用者的userfunc。recv_fd读取的下一个字节是状态（status）字节，若状态字节为0，则表示一个描述符已经传输过来，否则表示没有描述符可接收。

如下为send_err函数：

```c++
#include"apue.h"

int send_err(int fd, int errcode, const char *msg) {
    int n;
    if((n = strlen(msg))) {
        if(write(fd, msg, n) != n) {
            return -1;
        }
    }

    // 状态码必须为负数
    if(errcode > 0) {
        errcode = -1;
    }

    if(send_fd(fd, errcode) < 0) {
        return -1;
    }

    return 0;
}
```

为了使用unix域套接字传递文件描述符，调用sendmsg和recvmsg函数（16章）。这两个函数都使用一个指向msghdr的指针，该结构包含了所有要发送或要接收的消息的信息。结构定义如下：

```c++
struct msghdr{
    void *msg_name; /* option address */
    socklent_t msg_namelen; /* address size in bytes */
    struct iovec *msg_iov; /* array of I/O buffers*/
    int msg_iovlen; /* number of element in array */
    void *msg_control; /* ancillary data */
    socklen_t msg_controllen; /* number of ancillary bytes */
  	int msg_flags; /* flags for recived message */
    .
    .
    .
}
```

前两个元素通常在网络连接上发送数据报，其中目的地址可以由每个数据报指定。接下来的两个元素使我们可以指定一个由多个缓冲区构成数组（散布读和聚集写）。msg_flags字段包含了描述接收到的消息的标志。

两个元素处理控制信息的传输和接收。msg_control字段指向cmsghdr（控制信息头）结构，msg_controllen字段包含控制信息的字节数。

```c++
struct cmsghdr{
    socklen_t cmsg_len; /* data byte count, include header */
    int cmsg_level; /* originating protocol */
    int cmsg_type; /* protocol-specific type */
    /* followed by the actual control message data（后面更随着赋值数据） */
};
```

为了发送文件描述符，将cmsg_len设置为cmsghdr长度加一个整数的长度（描述符长度），cmg_level字段设置为SOL_SOCKET，cmsg_type字段设置为SCM_RIGHTS，用于表明在传输访问控制权。（SCM是Socket-level Control Message，即套接字级控制消息），访问权只能通过UNIX域套接字传送。描述符紧跟cmsg_type之后存储，用CMSG_DATA宏取得该整型量的指针。

unix定义了三个宏，用于访问控制数据，一个宏用于计算cmsg_len所使用的值。

```c
#include<sys/socket.h>
unsigned char *CMSG_DATA(struct cmsghdr *cp);
/* 返回一个指针，指向与cmsghdr结构相关联的数据。对于发送放来说，使用返回的指针来赋值，对于接收者来说，使用该值获取数据 */

struct cmsghdr *CMSS_FIRSTHDR(struct msghdr *mp);
/* 返回值：返回一个指针，指向与msghdr结构关联的第一个cmsghdr结构，若无这样的结构，返回null。使用msghdr一次可以传输多个数据（套接字）*/

struct cmsghdr *CMSG_NEXTHDR(struct msghdr *mp, struct cmsghdr *cp);
/* 返回值：返回一个指针，指向与msghdr结构相关联的下一个cmsghdr结构，该msghdr结构给出了当前的cmsghdr结构，若当前已近是最后一个，则返回null。对于发送方来说，一次想要传输多个数据时，需要设置msghdr的msg_controllen值为多个cmsghdr的长度之和，然后调用该函数依次为每个cmsghdr进行赋值 */

unsigned int CMSG_LEN(unsigned int nbytes);
/* 返回值：返回与cmsghdr结构相关联的数据（辅助数据）长度为nbytes时，cmsg对象的长度*/
```

CMSG_LEN宏返回存储nbytes长的数据对象所需要的字节数，其先将nbytes加上cmsghdr结构的长度，然后按照处理器体系结构的对齐要求进行调整，最后向上取整。

个人理解：当传递描述符时，对CMSG_DATA对象赋值套接字的fd，这并不是说最终传输的数据就是fd这个值（如果是这样，前文也不会说发送进程和接收进程的描述符编号往往不同，而且也没有必须使用这么麻烦的方式进行发送了），因为需要传递的实际上是指向文件表项的指针。当调用sendmsg时，会根据传参的msg信息对所需传输信息进行进一步处理（获取到fd指向的文件表项指针），同样的，在调用recvmsg时，会依据接收到的数据，对CMSG_DATA指向的地址分配一个未使用的文件描述符编号，并将其与指向文件表的指针相关联。

下面是send_fd的实现：

```c
#include"apue.h"
#include<sys/socket.h>

#define CONTROLLEN CMSG_LEN(sizeof(int))

static struct cmsghdr *cmptr = NULL;

int send_fd(int fd, int fd_to_send) {
    struct iovec iov[1];
    struct msghdr msg;
    char buf[2];

    iov[1].iov_base = buf;
    iov[1].iov_len = 2;
    msg.msg_iov = iov;
    msg.msg_name = NULL;
    msg.msg_namelen = 0;

    if(fd_to_send < 0) {
        msg.msg_control = NULL;
        msg.msg_controllen = 0;
        // buf[1] 非0，表示发送的是错误信息，没有套接字
        buf[1] = -fd_to_send;
        if(buf[1] == 0) {
            buf[1] = -1;
        }
    } else{
        if(cmptr == NULL && (cmptr = (cmsghdr*)malloc(CONTROLLEN)) == NULL) {
            return -1;
        }
        msg.msg_control = cmptr;
        msg.msg_controllen =CONTROLLEN;
        cmptr->cmsg_len = CONTROLLEN;
        cmptr->cmsg_level = SOL_SOCKET;
        cmptr->cmsg_type = SCM_RIGHTS;
        *(int*)CMSG_DATA(cmptr) = fd_to_send;
        // buf[1] = 0 表示正常传输了文件描述符
        buf[1] = 0;
    }

    buf[0] = 0;
    // 这里发送的数据只有msg中的iov数据，对于cmptr属于辅助数据，因此发送成功的返回值为2
    if(sendmsg(fd, &msg, 0) != 2) {
        return -1;
    }
    return 0;
}
```

下面是recv_fd的实现：

```c
#include"apue.h"
#include<sys/socket.h>

#define CONTROLLEN CMSG_LEN(sizeof(int))

static struct cmsghdr *cmptr = NULL;


int recv_fd(int fd, ssize_t (*userfunc)(int, const void *, size_t)) {
    int newfd, nr, status;
    char *ptr;
    char buf[MAXLINE];
    struct iovec iov[1];
    struct msghdr msg;

    status = -1;
    for(;;) {
        iov[0].iov_base = buf;
        iov->iov_len = sizeof(buf);
        msg.msg_iov = iov;
        msg.msg_iovlen = 1;
        msg.msg_name = NULL;
        msg.msg_namelen = 0;
        if(cmptr == NULL && (cmptr = (cmsghdr*)malloc(CONTROLLEN)) == NULL){
            return -1;
        }
        msg.msg_control = cmptr;
        msg.msg_controllen = CONTROLLEN;
        if((nr = recvmsg(fd, &msg, 0)) < 0) {
            err_ret("receiver error");
            return -1;
        } else if(nr == 0) {
            err_ret("connect closed by server");
            return -1;
        }

        for(ptr = buf; ptr < &buf[nr]; ) {
            // 找到第一个为空的字符串，前面为错误信息，后面为传递的fd
            if(*ptr++ == 0) {
                if(*ptr != buf[nr-1]) {
                    err_dump("message format error");
                }
                status = *ptr & 0xFF;
                if(status == 0) {
                    if(msg.msg_controllen < CONTROLLEN) {
                        err_dump("status =0 but no fd");
                    }
                    newfd = *(int*)CMSG_DATA(cmptr);
                } else{
                    newfd = -status;
                }
                nr -= 2;
            }
        }

        // 传输内容存在错误信息时
        if(nr > 0 && (*userfunc)(STDERR_FILENO, buf, nr) != nr) {
            return -1;
        }

        if(status >= 0) {
            return newfd;
        }
    }
}
```

该函数总是准备接收一个描述符，仅当msg_controllen返回非0时，程序才返回。



## open服务

创建一个守护进程，通过unix域套接字与客户进程交换，客户进程传输需要打开的文件和打开模式，守护进程返回打开的文件描述符。

客户进程和服务器进程之间的交互协议为：

1. 服务器通过unix域套接字发送`open <pathname> <openmode>\0`形式请求。openmode是数值，为open函数的第二个参数，请求字符串以null字符终止。
2. 服务器调用send_fd或send_err回送打开的文件描述符或错误。

首先定义头文件open.h

```c
#include"apue.h"
#include<errno.h>

#define CL_OPEN "open"

#define CS_OPEN "/tmp/opend.socket" /*众所众知的命名域套接字*/

int csopen(char *, int mode);
```

客户端请求main函数：

```c
#include"open.h"
#include<fcntl.h>
#include<sys/uio.h>

#define BUFFSIZE  8192

int main(int argc, char *argv[]) {
    int n, fd;
    char buf[BUFFSIZE];
    char line[MAXLINE];

    while (fgets(line, MAXLINE, stdin) != NULL)
    {
        if(line[strlen(line) - 1] == '\n') {
            line[strlen(line) - 1] = 0;
        }

        if((fd = csopen(line, O_RDONLY)) < 0) {
            continue;
        }

        while ((n = read(fd, buf, BUFFSIZE)) > 0)
        {
            if(write(STDOUT_FILENO, buf, n) != n) {
                err_sys("write error");
            }
        }

        if(n < 0) {
            err_sys("read error");
        }
        close(fd);
    }
    exit(0);
}
```

客户端读取每行的数据，即为要读取的文件名，调用csopen函数获得描述符，打印文件内容，下面看csopen的实现：

```c
#include"open.h"
#include<sys/uio.h>

int csopen(char *name, int oflag) {
    int len;
    char buf[12];
    struct iovec iov[3];
    static int csfd = 1;

    if(csfd < 0) {
        if((csfd = cli_conn(CS_OPEN)) < 0) {
            err_ret("cli_conn error");
            return -1;
        }
    }

    /*注意，我们的协议中，每部分由空格分割，因此这里%d前有一个空格，用来在pathname和oflag中插入空格*/
    sprintf(buf, " %d", oflag); 

    /* 这个实际执行为(char*) "open" " " = (char*)"open ",其中空格用于分割open和后面的pathname */
    iov[0].iov_base = (char*)CL_OPEN " ";
    iov[0].iov_len = strlen(CL_OPEN) + 1;
    iov[1].iov_base = name;
    iov[1].iov_len = strlen(name);
    iov[2].iov_base = buf;
    iov[2].iov_len = strlen(buf) + 1; // 传递的终止为空字符。
    len = iov[0].iov_len + iov[1].iov_len + iov[2].iov_len;
    if(writev(csfd, &iov[0], 3) != len) {
        err_ret("write error");
    }

    return recv_fd(csfd, write);
}
```

下面看服务端的实现。其中服务端的open.h如下：

```c
#include"apue.h"
#include<errno.h>

#define CS_OPEN "tmp/opend.socket"
#define CL_OPEN "open"

extern int ddebug; //区分是否要作为守护进程

extern char errmsg[];

extern int oflag;

extern char *pathname;

typedef struct{
    int fd;
    int uid;
} Client;

extern Client *client;
extern int client_size;

int cli_args(int, char **);

int client_add(int, uid_t);

void client_del(int);

void loop();

void handle_request(char*, int, int, uid_t);
```

其中定义了Client结构，用于存储与服务器建立连接的套接字。并包含两个处理函数，add和delete，其实现如下：

```c
#include"serviceOpen.h"

#define NALLOC 10

static void client_alloc(void) {
    int i;
    if(client == NULL) {
        client = (Client*)malloc(NALLOC * sizeof(Client));
    } else {
        client = (Client*)realloc(client, (client_size + NALLOC) * sizeof(Client));
    }

    if(client == NULL) {
        err_sys("can't alloc for client array");
    }

    for(i=client_size; i<client_size+NALLOC; i++) {
        client[i].fd = -1;
    }

    client_size += NALLOC;
}

int client_add(int fd, uid_t uid) {
    int i;
    if(client == NULL) {
        client_alloc();
    }
again:
    for(i=0; i<client_size; i++){
        if(client[i].fd == -1) {
            client[i].fd = fd;
            client[i].uid = uid;
            return i;
        }
    }

    client_alloc();
    goto again;

}

void client_del(int fd) {
    int i;
    for(i =0 ;i<client_size; i++) {
        if(client[i].fd == fd) {
            client[i].fd = -1;
            return;
        }
    }
    log_quit("can't find client entry for fd %d", fd);
}
```

其实现了连接的复用和动态增涨。

为了方便调试，我们可能希望能够通过参数指定服务器进程是守护进程还是非守护进程。Single UNIX Specification包括一系列规范和约定来保证命名语法的一致性。库函数中的getopt函数帮助开发者以一致的方式处理命令行参数。

```c
#include<unistd.h>

int getopt(int argc, char *const argv[], const char *options);

extern int optind, opterr, optopt;

extern char *optarg;

//返回值：当所有选项被处理完时，返回-1，否则返回下一个选项字符
```

参数argc和argv与传入的main函数一致。options参数是一个包含该命令支持的选项字符的字符串。如果一个选项字符后面接一个冒号，则表示该选项需要参数，否则该选项不需要额外参数。举例来说，如果一条命令的用法说明如下：

```
commend [-i] [-u username] [-z] filename
```

可以给getopt传送一个`iu:z`作为options字符串。

getopt包含四个外部变量：

1. optarg：如果一个参数选项需要参数，在处理该选项时，getopt会设置optarg指向该选项的参数字符串。
2. opterr：如果一个选项发生了错误，getopt会默认打印一条出错信息。应用程序可以通过设置opterr为0来禁止该行为。
3. optind：用来存放下一个要处理的字符串在argv数组里面的下标。从1开始，每处理一个参数，getopt都对其增加1.
4. optopt：如果处理选项时发生错误，getopt会设置optopt指向导致出错的选项字符串。

在我们的服务器中，使用-d选项来控制程序是否为守护进程的方式运行。具体如下：

```c
#include"open.h"
#include<syslog.h>

int debug, oflag, client_size, log_to_stderr;
char errmsg[MAXLINE];
char *pathname;
Client *client = NULL;

int main(int argc, char *argv[]) {
    int c;

    log_open("open.serv", LOG_PID, LOG_USER);

    opterr = 0;
    while ((c = getopt(argc, argv, "d")) != EOF)
    {
        switch (c)
        {
        case 'd':
            debug = log_to_stderr = 1;
            break;
        case '?':
            err_quit("unrecognized option: -%c", optopt);
        }
    }

    if(debug == 0) {
        daemonize("opend");
    }

    loop();
    
}
```

loop循环用于接收客户端请求并进行处理，这里展示使用select的处理方式：

```c
void loop() {
    int i, n, maxfd, maxi, listenfd, clifd, nread;
    char buf[MAXLINE];
    uid_t uid;
    fd_set rset, allset;

    FD_ZERO(&allset);

    if((listenfd = serv_listen(CS_OPEN)) < 0) {
        log_sys("service_listen error");
    }

    FD_SET(listenfd, &allset);
    maxfd = listenfd;
    maxi = -1;

    for(;;) {
        rset = allset; //由于每次循环后，需要监听的套接字集合存在变换，因此要重新赋值。

        // 只监听是否可读
        if((n = select(maxfd+1, &rset, NULL, NULL, NULL)) < 0) {
            printf("select error");
            log_sys("select error");
        }

        if(FD_ISSET(listenfd, &rset)) {
            printf("receiver a quest\n");
            if((clifd = serv_accept(listenfd, &uid)) < 0) {
                log_sys("serv_accept error: %d", clifd);
            }
            i = client_add(clifd, uid);
            FD_SET(clifd, &allset);
            if(clifd > maxfd) {
                maxfd = clifd;
            }
            if(i > maxi) {
                maxi = i;
            }
            log_msg("new conncetion: uid %d, fd %d", uid, clifd);
            continue;
        }

        for(i = 0; i<= maxi; i++) {
            if((clifd = client[i].fd) < 0) {
                continue;
            }

            if(FD_ISSET(clifd, &rset)) {
                if((nread = read(clifd, buf, MAXLINE)) < 0) {
                    log_sys("read error on fd %d", client[i].fd);
                } else if(nread == 0){
                    // 可读，但读取长度为0，表示连接关闭
                    log_msg("close: uid %d, fd %d", client[i].uid, client[i].fd);
                    client_del(clifd);
                    FD_CLR(clifd, &allset);
                    close(clifd);
                    
                } else {
                    handle_request(buf, nread, clifd, client[i].uid);
                }
            }
        }
    }
}
```

对于handle_request为:

```c
void handle_request(char *buf, int nread, int clifd, uid_t uid) {
    int newfd;

    if(buf[nread-1] != 0) {
        snprintf(errmsg, MAXLINE-1, "request from uid %d not null terminated: %*.*s\n", uid, nread, nread, buf);
        send_err(clifd, -1, errmsg);
        return;
    }

    log_msg("request: %s, from uid %d", buf, uid);

    if(buf_args(buf, cli_args)<0) {
        send_err(clifd, -1, errmsg);
        log_msg(errmsg);
        return;
    }

    if((newfd = open(pathname, oflag)) < 0) {
        snprintf(errmsg, MAXLINE-1, "can't open %s: %s\n", pathname, strerror(errno));
        send_err(clifd, -1, errmsg);
        log_msg(errmsg);
        return;
    }

    if(send_fd(clifd, newfd) < 0) {
        log_sys("send_fd error");
    }


    log_msg("send fd %d over fd %d for %s", newfd, clifd, pathname);
    close(newfd);
}

#define WHITE " \t\n"
#define MAXARGC 50

int buf_args(char *buf, int (*optfunc)(int, char **)) {
    char *ptr, *argv[MAXARGC];
    int argc;

    if(strtok(buf, WHITE) == NULL) {
        return -1;
    }
    argv[argc=0] = buf;
    while((ptr = strtok(NULL, WHITE)) != NULL) {
        if(++argc > MAXARGC-1) {
            return -1;
        }
        argv[argc] = ptr;
    }
    argv[++argc] = NULL;

    return (*optfunc)(argc, argv);
}

int cli_args(int argc, char **argv) {
    if(argc != 3 || strcmp(argv[0], CL_OPEN) != 0) {
        strcpy(errmsg, "usage: <pathname> <oflag>\n");
        return -1;
    }
    pathname = argv[1];
    oflag = atoi(argv[2]);
    return 0;
}

```

其使用buf_args和cli_args来处理请求，使用strtok函数来将请求转换为程序输入参数的形式，并从中获取打开文件的路径和打开模式。而后打开文件，返回文件的描述符。

分别编译客户端和服务端生成openClient和openService。执行：

```
// 窗口1
$./openService -d

// 窗口2
$./openClient
/home/work/test.txt
hello ...
...
...
```

由于daemonize函数在之前的实现中，会关闭套接字的原因，因此如果不使用-d参数，使用守护进程，客户端会无法进行连接。