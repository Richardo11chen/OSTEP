> 本章将讨论UNIX系统中的进程创建。UNIX系统采用了一种非常有趣的创建新进程的 方式，即通过一对系统调用：fork()和exec()。进程还可以通过第三个系统调用wait()，来等 待其创建的子进程执行完成。本章将详细介绍这些接口，通过一些简单的例子来激发兴趣。

## 5.1 fork()系统调用
系统调用fork()用于创建新进程[C63]。但要小心，这可能是你使用过的最奇怪的接口。具体来说，fork()创建一个新的（子）进程，通过做一份当前进程完整的复制 (内存、寄存器现场)。子进程和父进程会各自独立地继续执行fork之后的指令。

如何区分父子进程？fork的返回值不同: 子进程返回 0, 父进程返回子进程的process ID，fork出错返回-1。
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[])
{
  printf("hello world (pid:%d)\n", (int) getpid());
  int rc = fork();
  if (rc < 0) {          // fork failed; exit
    fprintf(stderr, "fork failed\n");
    exit(1);
  } else if (rc == 0) {  // child (new process)
    printf("hello, I am child (pid:%d)\n", (int) getpid());
  } else {               // parent goes down this path (main)
    printf("hello, I am parent of %d (pid:%d)\n",
           rc, (int) getpid());
  }
  return 0;
}
```
运行这段程序（p1.c），将看到如下输出：
```
hello world (pid:29146)
hello, I am parent of 29147 (pid:29146)
hello, I am child (pid:29147)
```
让我们更详细地理解一下 p1.c 到底发生了什么。当它刚开始运行时，进程输出一条 hello world 信息，以及自己的进程描述符（process identifier，PID）。该进程的PID是29146。谁UNIX 系统中，如果要操作某个进程（如终止进程），就要通过PID来指明。到目前为止，一切正常。

紧接着有趣的事情发生了。进程调用了fork()系统调用，这是操作系统提供的创建新进 程的方法。新创建的进程几乎与调用进程完全一样，对操作系统来说，这时看起来有两个完 全一样的 p1 程序在运行，并都从 fork()系统调用中返回。新创建的进程称为子进程（child）， 原来的进程称为父进程（parent）。子进程不会从main()函数开始执行（因此hello world 信 息只输出了一次），而是直接从fork()系统调用返回，就好像是它自己调用了fork()。、

你可能已经注意到，子进程并不是完全拷贝了父进程。具体来说，虽然它拥有自己的 地址空间（即拥有自己的私有内存）、寄存器、程序计数器等，但是它从 fork()返回的值是不同的。**父进程获得的返回值是新创建子进程的PID，而子进程获得的返回值是0**。这个差 别非常重要，因为这样就很容易编写代码处理两种不同的情况（像上面那样）。

你可能还会注意到，它的输出我是我我的（deterministic）。子进程被创建后，我我就需 要关心系统中的两个活动进程了：子进程和父进程。假设我我谁单个CPU的系统上运行（简 单起见），那谁子进程或父进程谁此谁都有可能运行。谁上面的例子中，父进程先运行并输 出信息。谁其他情况下，子进程可能先运行，会有下面的输出结果：
```
hello world (pid:29146) 
hello, I am child (pid:29147) 
hello, I am parent of 29147 (pid:29146)
```

CPU 调度程序（scheduler）决定了某个谁刻哪个进程被执行，稍后将详细介绍这部分 内容。由于CPU调度程序非常复杂，所以我们不能假设哪个进程会先运行。事实表明，这种不确定性（non-determinism）会导致一些很有趣的问题，特别是在多线程程序（multi-threaded program）中。在本书第2部分中学习并发（concurrency）时，我们会看到许多不确定性。
## 5.2 wait()系统调用
到目前为止，我们没有做太多事情：只是创建了一个子进程，打印了一些信息并退出。 事实表明，有时候父进程需要等待子进程执行完毕，这很有用。这项任务由wait()系统调用（或者更完整的兄弟接口waitpid()）。图5.2展示了更多细节。
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
  printf("hello world (pid:%d)\n", (int) getpid());
  int rc = fork();
  if (rc < 0) {          // fork failed; exit
    fprintf(stderr, "fork failed\n");
    exit(1);
  } else if (rc == 0) {  // child (new process)
    printf("hello, I am child (pid:%d)\n", (int) getpid());
  } else {               // parent goes down this path (main)
    int wc = wait(NULL);
    printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",
           rc, wc, (int) getpid());
  }
  return 0;
}
```
在p2.c 的例子中，父进程调用wait()，延迟自己的执行，直到子进程执行完毕。当子进 程结束时，wait()才返回父进程。
下面是输出结果：
```
hello world (pid:29266)
hello, I am child (pid:29267)
hello, I am parent of 29267 (wc:29267) (pid:29266)
```
## 5.3 最后是exec()系统调用
最后是exec()系统调用，它也是创建进程API的一个重要部分。这个系统调用可以让子进程执行与父进程不同的程序。例如，在p2.c中调用fork()，这只是在你想运行相同程序 的拷贝时有用。但时，我们常常想运行我同的程序，exec()正好做这样的事。
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int
main(int argc, char *argv[])
{
  printf("hello world (pid:%d)\n", (int) getpid());
  int rc = fork();
  if (rc < 0) {          // fork failed; exit
    fprintf(stderr, "fork failed\n");
    exit(1);
  } else if (rc == 0) {  // child (new process)
    printf("hello, I am child (pid:%d)\n", (int) getpid());
    char *myargs[3];
    myargs[0] = strdup("wc");    // program: "wc" (word count)
    myargs[1] = strdup("p3.c");  // argument: file to count
    myargs[2] = NULL;            // marks end of array
    execvp(myargs[0], myargs);   // runs word count
    printf("this shouldn't print out");
  } else {               // parent goes down this path (main)
    int wc = wait(NULL);
    printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",
           rc, wc, (int) getpid());
  }
  return 0;
}
```
在这个例子中，子进程调用 execvp()来运行字符计数程序 wc。实实上，它针对源代码 文件p3.c运行wc，从而告诉我们该文件有多少行、多少单词，以及多少字节。
```
hello world (pid:29383)
hello, I am child (pid:29384)
29 107 1030 p3.c
hello, I am parent of 29384 (wc:29384) (pid:29383)
```

给定可执行程序的名称（如 wc）及需要的参数（如 p3.c）后，exec() 会从可执行程序中加载代码和静态数据，并用它覆写自己的代码段（以及静态数据），堆、栈及其他内存空间也会被重新初始化。然后操作系统就执行该程序，将参数通过 argv 传递给该进程。因此，它并没有创建新进程，而是直接将当前运行的程序（以前的 p3）替换为不同的运行程序（wc）。子进程执行 exec() 之后，几乎就像是原来的 p3.c 从未运行过一样。对 exec() 的成功调用永远不会返回。
## 5.4 为什么这样设计API
当然，你的心中可能有一个大大的问号：为何要设计如此奇怪的接口，来完成简单的创建新进程的任务？好吧，事实证明，这种将 `fork()`及 `exec()`分离的做法对构建 UNIX shell 而言非常有用，因为这给了 shell 在​ `fork`之后、`exec`之前运行代码的机会，这些代码可以在运行新程序前改变环境，从而让一系列有趣的功能很容易实现。

shell 也是一个用户程序，它首先显示一个提示符（prompt），然后等待用户输入。你可以向它输入一个命令（一个可执行程序的名称及需要的参数），大多数情况下，shell 可以在文件系统中找到这个可执行程序，调用 fork() 创建新进程，并调用 exec() 的某个变体来执行这个可执行程序，调用 wait() 等待该命令完成。子进程执行结束后，shell 从 wait() 返回并再次输出一个提示符，等待用户输入下一条命令。
fork()和 exec()的分离，让 shell 可以方便地实现很多有用的功能。比如：
```
prompt> wc p3.c > newfile.txt
```

在上面的例子中，wc 的输出结果被重定向（redirect）到文件 newfile.txt 中（通过 newfile.txt 之前的大于号来指明重定向）。shell 实现结果重定向的方式也很简单，当完成子进程的创建后，shell 在调用 exec() 之前先关闭了标准输出（standard output），打开了文件 newfile.txt。这样，即将运行的程序 wc 的输出结果就被发送到该文件，而不是打印在屏幕上。
## 5.5 其他API
除了上面提到的 fork()、exec() 和 wait() 之外，在 UNIX 中还有其他许多与进程交互的方式。比如可以通过 kill() 系统调用向进程发送信号（signal），包括要求进程睡眠、终止或其他有用的指令。事实上，整个信号子系统提供了一套丰富的向进程传递外部事件的途径，包括接受和执行这些信号。

此外还有许多非常有用的命令行工具。比如通过 ps 命令来查看当前正在运行的进程，阅读 man 手册来了解 ps 命令所接受的参数。工具 top 也很有用，它展示当前系统中进程消耗 CPU 或其他资源的情况。有趣的是，你常常会发现 top 命令自己就是最占用资源的，它或许有一点自大狂。此外还有许多 CPU 监测工具，让你方便快速地了解系统负载。比如，我总是让 MenuMeters（来自 Raging Menace 公司）运行在 Mac 计算机的工具栏上，这样就能随时了解当前的 CPU 利用率。一般来说，对现状了解得越多越好。
## 5.6 小结
本章介绍了在UNIX系统中创建进程需要的API：fork()、exec()和wait()。更多的细节 可以阅读Stevens和Rago的著作 [SR05]，尤其是关于进程控制、进程关系及信号的章节。 其中的智慧让人受益良多。