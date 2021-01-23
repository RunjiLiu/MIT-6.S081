6.S081目标：

1. os的design和implementation,design即higgh-level structure implementation即代码实现
2. 实现一个xv6系统，以及修改现有的系统， 在上面编写软件

## OS的需求目的：

1. 对硬件进行抽象，cpu，内存等对于应用程序太low-level，由os提供接口给应用程序
2. multiplex hardware，硬件复用技术，通过os实现多个程序同时执行不相互干扰
3. isolation，提供给程序相互独立的空间，防止bug
4. sharing，当需要的时候多个程序的相互交互
5. security，当不需要share的时候控制访问权限
6. performace，保证os不妨碍到应用的性能
7. 兼容多种应用

## OS overview:

applications: gcc vi ...

kernel sercices:

* process (a running program) 进程

* memory allocation 内存分配（给进程）
* file system 文件系统（包括 file contents 文件在磁盘中的内容，file namespace包括file name和directories路径 ）
* access control (security) 权限控制
* many others: users（多用户）, IPC, network（网络访问）, time, terminals ,inner process communication（进程间通信）

 hardware: CPU, RAM, disk, net, &c

## api

 内核提供给程序的接口，又称系统调用

```c
fd = open("out", 1);
```

访问文件，返回文件标识符

```c
write(fd, "hello\n", 6);
```

write()写入文件

```c
pid = fork();
```

创建一个新进程，继承当前进程的信息例如文件描述符等

## Why is O/S design+implementation hard and interesting?

* unforgiving environment: quirky h/w, hard to debug  开发环境比较复杂
* many design tensions: 需要进行权衡取舍
  - efficient vs abstract/portable/general-purpose low-level的api与硬件交互效率高，high-level的api可以带来抽象性
  - powerful vs simple interfaces  性能与易用间的平衡
  - flexible vs secure 灵活性可以方便编写程序，但是会对安全性有所影响（权限控制，程序可能干扰其他程序甚至O/S本身的运行）
* features interact: `fd = open(); fork()` 系统调用间的联系（新进程可以继承父进程打开的文件描述符）
* uses are varied: laptops, smart-phones, cloud, virtual machines, embedded 多用途
* evolving hardware: NVRAM, multi-core, fast networks 系统必须随着硬件的发展而发展

### high-level 语言与系统调用

<u>high-level语言例如python与系统调用是不相关的，因为high-level语言通常需要确保可移植性所以一般不会使用系统调用（当前系统的api可能与另外系统不同），但是python的内部最终是通过系统调用来完成操作的（转换为低级与语言 ）</u>

## system call

```c
// copy.c: copy input to output.

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
  char buf[64];

  while(1){
    int n = read(0, buf, sizeof(buf));
    if(n <= 0)
      break;
    write(1, buf, n);
  }

  exit(0);
}
```



此处read的第一个参数是一个已经打开的文件描述符号，此处0是unix中代表标准输入流的文件描述符，write的1代表是标准输出流，buf是暂存缓冲区

read()返回的是读取的字节数，如果直接读到文件尾，则返回0，如果文件描述符不存在会返回负数

如果读的buffer长度大于实际给定字符串数组长度，则可能导致溢出，程序崩溃，或者其他未定义的情况

该程序不关注输入输出内容是什么，对于O/S来说，只是8字节的数据流而已



```c
// open.c: create a file, write to it.

#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int
main()
{
  int fd = open("output.txt", O_WRONLY | O_CREATE);
  write(fd, "ooo\n", 4);

  exit(0);
}
```

kernel会维护一个表来记录每个进程所打开的文件描述符，每个进程都有一个空间来记录文件描述符，两个独立的进程如果同时打开一个文件，可能会得到一个相同的文件描述符，但是由于每个进程都是独立记录文件描述符的，因此一个相同的文件描述符不一定代表两个进程打开同一个文件。

<u>51：49处</u> 有一个question，是“how does a compiler handle system calls does assembly generated make a procedure call to some code segment undefined by the operating system”大致是问编译器怎么处理系统调用以及生成的汇编程序是否为调用O/S中未定义的代码段

answer大致是说risv-5中有一个特殊指是将控制权给kernel，open是一个c libary中定义的函数，但是内部使用的是汇编语言进行实现，由许多特殊指令组成(e-call)，由kernel来从主存和寄存器中操作

 

```c
// fork.c: create a new process

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
  int pid;

  pid = fork();

  printf("fork() returned %d\n", pid);

  if(pid == 0){
    printf("child\n");
  } else {
    printf("parent\n");
  }
  
  exit(0);
}
```

fork产生进程会复制当前进程的指令和数据，对于父进程fork返回子进程的pid（也是该子进程是系统启动后第x个进程），对于子进程返回0

在演示中，由于qemu模拟了一个多核处理器，所以会出现父进程和子进程输出相互交叉的情况

在xv6中，父进程与子进程是相同的，指令，数据，运行栈相同，都有自己的虚拟内存空间，空间内不同

一些复杂系统中可能会有一些detail导致父子进程不同



```c
// exec.c: replace a process with an executable file

#include "kernel/types.h"
#include "user/user.h"

int
main()
{
  char *argv[] = { "echo", "this", "is", "echo", 0 };

  exec("echo", argv);

  printf("exec failed!\n");

  exit(0);
}
```

shell在执行命令的时候，会创建出一个进程来读取输入对应的可执行程序，并执行

exec.c中exec()会切换进程（不会再切回来），丢弃当前进程的上下文除了文件描述符，加载对应可执行文件并执，行除非exec执行错误，例如找不到对应的程序文件，exec才会返回一个数，正常情况下不会返回值

声明char **时，最后一个元素0是由于无法程序确定数组长度，所以加上一个值为0的指针（所指位置为0）即null指针，



```c
#include "kernel/types.h"
#include "user/user.h"

// forkexec.c: fork then exec

int
main()
{
  int pid, status;

  pid = fork();
  if(pid == 0){
    char *argv[] = { "echo", "THIS", "IS", "ECHO", 0 };
    exec("echo", argv);
    printf("exec failed!\n");
    exit(1);
  } else {
    printf("parent waiting\n");
    wait(&status);
    printf("the child exited with status %d\n", status);
  }

  exit(0);
}
```

wait是用来等待子进程执行完成，返回wait的进程号

unix中通常用0表示程序执行成功，1表示出现错误

此处fork()会拷贝内存，但是在子进程又会使用exec丢弃内存，浪费资源，在之后的lab会进行优化，

对于输出的顺序，演示的顺序是先执行父进程，一个可能的推测是由于子进程中的exec需要访问文件系统，从文件中读指令进内存

如果在一个没有子进程的进程中使用wait，会得到一个-1

qs:如果说子进程会拷贝父进程的内存，那么子进程是不是进行了重定义变量

ans: 程序经过编译后得到存储在主存中的指令，只是一个个字节 ，拷贝可以令父子进程的虚拟内存映射一致，对于c程序来说，拷贝的对象只是机器指令

对于多进程来说，一个wait只会等待一个最早结束的进程，可以通过wait的返回值来判断是哪一个进程结束

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

// redirect.c: run a command with output redirected

int
main()
{
  int pid;

  pid = fork();
  if(pid == 0){
    close(1);
    open("output.txt", O_WRONLY|O_CREATE);

    char *argv[] = { "echo", "this", "is", "redirected", "echo", 0 };
    exec("echo", argv);
    printf("exec failed!\n");
    exit(1);
  } else {
    wait((int *) 0);
  }

  exit(0);
}
```

在子进程中，关闭了文件描述符1，本来1是标准输出的，之后open申请得到一个最低的文件描述符，此时1就指向了output.txt，echo的默认输出就是1,因此会输出到output.txt中