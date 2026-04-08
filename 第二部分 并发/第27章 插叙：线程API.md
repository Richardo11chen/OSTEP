> 本章介绍了主要的线程API。后续章节也会进一步介绍如何使用API。更多的细节可以 参考其他书籍和在线资源。

**27.1 线程创建**
编写多线程程序的第一步就是创建新线程，因此必须存在某种线程创建接口。在POSIX 中，很简单：
```c
#include <pthread.h>
int pthread_create( pthread_t * thread,
					const pthread_attr_t * attr,
					void * (*start_routine)(void*),
					void * arg);
```
该函数有4个参数：`thread、attr、start_routine`和`arg`。
第一个参数`thread`是指 向 `pthread_t`结构类型的指针，我们将利用这个结构与该线程交互，因此需要将它传入 `pthread_create(`)，以便将它初始化。
第二个参数 `attr` 用于指定该线程可能具有的任何属性。一些例子包括**设置栈大小**，或关于该**线程调度优先级**的信息。一个属性通过单独调用`pthread_attr_init()`来初始化。有关详细信 息，请参阅手册。但是，在大多数情况下，默认值就行。在这个例子中，我们只需传入`NULL`。
第三个参数最复杂，但它实际上只是问：这个线程应该在哪个函数中运行？在 C 中， 我们把它称为一个函数指针（function pointer），这个指针告诉我们需要以下内容：一个函数 名称（start_routine），它被传入一个类型为`void *`的参数（start_routine 后面的括号表明了这 一点），并且它返回一个`void *`类型的值（即一个void指针）
最后，第四个参数arg就是要传递给线程开始执行的函数的参数。
> 为什么 我们需要这些void指针？
> 答案很简单：将void指针作为函数的参数start_routine，允 许我们传入任何类型的参数，将它作为返回值，允许线程返回任何类型的结果。

下面来看图27.1 中的例子。这里我们只是创建了一个线程，传入两个参数，它们被打 包成一个我们自己定义的类型（myarg_t）。该线程一旦创建，可以简单地将其参数转换为它 所期望的类型，从而根据需要将参数解包。
```c
#include <pthread.h>

typedef struct myarg_t {
    int a;
    int b;
} myarg_t;

void *mythread(void *arg) {
    myarg_t *m = (myarg_t *) arg;
    printf("%d %d\n", m->a, m->b);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    int rc;

    myarg_t args;
    args.a = 10;
    args.b = 20;
    rc = pthread_create(&p, NULL, mythread, &args);
    ...
}
```

**27.2 线程完成**
上面的例子展示了如何创建一个线程。但是，如果你想等待线程完成，怎么做呢？你必须调用函数pthread_join()。
```c
int pthread_join(pthread_t thread, void **value_ptr);
```
该函数有两个参数。pthread_join允许主线程等待另一个线程结束，并获取其返回值。
第一个是pthread_t 类型，用于指定要等待的线程。这个变量是由 线程创建函数初始化的（当你将一个指针作为参数传递给 pthread_create()时）。
第二个参数是一个指针，指向你希望得到的返回值。因为函数可以返回任何东西，所 以它被定义为返回一个指向void的指针。因为pthread_join()函数改变了传入参数的值，所 以你需要传入一个指向该值的指针，而不只是该值本身。
	形象的讲，线程结束时，`pthread_join`内部会执行类似这样的操作：`*retval = exit_value`, retval 就是传入的 (void**)value_ptr，exit_value是线程函数的(void \*)返回值。
我们来看另一个例子（见图27.2）。在代码中，再次创建单个线程，并通过myarg_t结 构传递一些参数。对于返回值，使用 myret_t 型。当线程完成运行时，主线程已经在 pthread_join()函数内等待了。然后会返回，我们可以访问线程返回的值，即在myret_t中的 内容。
如果不需要返回值，那么pthread_join()调 用也可以传入NULL。
```c
#include <stdio.h>
#include <pthread.h>
#include <assert.h>
#include <stdlib.h>

typedef struct myarg_t {
    int a;
    int b;
} myarg_t;

typedef struct myret_t {
    int x;
    int y;
} myret_t;

void *mythread(void *arg) {
    myarg_t *m = (myarg_t *) arg;
    printf("%d %d\n", m->a, m->b);
    myret_t *r = Malloc(sizeof(myret_t));
    r->x = 1;
    r->y = 2;
    return (void *) r;
}

int main(int argc, char *argv[]) {
    int rc;
    pthread_t p;
    myret_t *m;
    myarg_t args;
	args.a = 10;
	args.b = 20;
	Pthread_create(&p, NULL, mythread, &args);
	Pthread_join(p, (void **) &m);
	printf("returned %d %d\n", m->x, m->y);
	return 0;
}
```
一个更简单的情况：
```c
void *mythread(void *arg) {
    int m = (int) arg;
    printf("%d", m);
    return (void *) (arg + 1);
}

int main(int argc, char *argv[]) {
    pthread_t p;
    int rc, m;
    Pthread_create(&p, NULL, mythread, (void *) 100);
    Pthread_join(p, (void **) &m);
    printf("returned %d", m);
    return 0;
}
```
我们应该注意，必须对线程返回值非常小心。特别是，永远不要返回一个 指针，并让它指向线程调用栈上分配的东西。
> 如果返回值指向线程调用栈上分配的东西，那么函数结束后，栈上的变量会自动释放，因此指针会指向已释放的变量。

最后，你可能会注注到，使用pthread_create()创建线程，然后立即调用pthread_join()， 这是创建线程的一种非常奇怪的方式。事实上，有一个更简单的方法来完成这个任务，它 被称为过程调用（procedure call）。

**27.3 锁**
除了线程创建和join之外，POSIX线程库提供的最有用的函数集，可能是通过锁（lock） 来提供互斥进入临界区的那些函数。这方面最基本的一对函数是：
```c
int pthread_mutex_lock(pthread_mutex_t *mutex); 
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
如果有一段代码是一个临界区，就需要通过锁来保护，你大概可以想象代码的样子：
```c
pthread_mutex_t lock; 
pthread_mutex_lock(&lock); 
x = x + 1; // or whatever your critical section is
pthread_mutex_unlock(&lock);
```
这段代码的意思是：如果在调用 pthread_mutex_lock()时没有其他线程持有锁，线程将 获取该锁并进入临界区。如果另一个线程确实持有该锁，那么尝试获取该锁的线程将不会 从该调用返回，直到获得该锁（注味着持有该锁的线程通过解锁调用释放该锁）。当然，在 给定的时间内，许多线程可能会卡住，在获取锁的函数内部等待。然而，只有获得锁的线 程才应该调用解锁。
遗憾的是，这段代码有两个重要的问题。第一个问题是缺乏正确的初始化（lack of proper initialization）。所有锁必须正确初始化，以确保它们具有正确的值。

对于 POSIX 线程，有两种方法来初始化锁。一种方法是使用 PTHREAD_MUTEX_ INITIALIZER，如下所示：
```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```
这样做会将锁设置为默认值，从而使锁可用。初始化的动态方法（即在运行时）是调 用pthread_mutex_init()，如下所示：
```c
int rc = pthread_mutex_init(&lock, NULL); 
assert(rc == 0); // always check success!
```
此函数的第一个参数是锁本身的地址，而第二个参数是一组可选属性。请你自己去详细了解这些属性。传入 NULL 就是使用默认值。无论哪种方式都有效，但我们通常使用动 态（后者）方法。请注注，当你用完锁时，还应该相应地调用pthread_mutex_destroy()，所 有细节请参阅手册。

上述代码的第二个问题是在调用获取锁和释放锁时没有检查错误代码。就像 UNIX 系 统中调用的任何库函数一样，这些函数也可能会失败！至少要使 用包注的函数，它对函数成功加上断言（见图27.4）。更复杂的（非玩具）程序，在出现问 题时不能简单地退出，应该检查失败并在获取锁或释放锁未成功时执行适当的操作。
```c
// Use this to keep your code clean but check for failures 
// Only use if exiting program is OK upon failure 
void Pthread_mutex_lock(pthread_mutex_t *mutex) { 
	int rc = pthread_mutex_lock(mutex); 
	assert(rc == 0); 
}
```

获取锁和释放锁函数不是pthread与锁进行交互的仅有的函数。特别是，这里有两个你 可能感兴趣的函数：
```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex, 
						struct timespec *abs_timeout);
```
这两个调用用于获取锁。如果锁已被占用，则trylock版本将失败。获取锁的timedlock 定版本会在超时或获取锁后返回，以先发生者为准。因此，具有零超时的timedlock退化为 trylock 的情况。通常应避免使用这两种版本，但有些情况下，避免卡在（可能无限期的） 获取锁的函数中会很有用，我们将在以后的章节中看到（例如，当我们研究死锁时）。

**27.4 条件变量**
所有线程库还有一个主要组件（当然 POSIX 线程也是如此），就是存在一个条件变量 （condition variable）。当线程之间必须发生某种信号时，如果一个线程在等待另一个线程继 续执行某些操作，条件变量就很有用。希望以这种方式进行交互的程序使用两个主要函数：
```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex); 
int pthread_cond_signal(pthread_cond_t *cond);
```
要使用条件变量，必须另外有一个与此条件相关的锁。在调用上述任何一个函数时， 应该持有这个锁。

第一个函数pthread_cond_wait()使调用线程进入休眠状态，因此等待其他线程发出信号。典型的用法如下所示：
```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER; 
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
Pthread_mutex_lock(&lock); 
while (ready == 0) 
	Pthread_cond_wait(&cond, &lock); 
Pthread_mutex_unlock(&lock);
```
在这段代码中，在初始化相关的锁和条件之后，一个线程检查变量ready是否已经被设置 为零以外的值。如果没有，那么线程只是简单地调用等待函数以便休眠，直到其他线程唤醒它。 唤醒线程的代码运行在另外某个线程中，像下面这样：
```c
Pthread_mutex_lock(&lock); 
ready = 1;
Pthread_cond_signal(&cond); 
Pthread_mutex_unlock(&lock);
```
关于这段代码有一些注注事项。首先，在发出信号时（以及修改全局变量ready 时）， 我们始终确保持有锁。这确保我们不会在代码中注外引入竞态条件。
其次，你可能会注注到等待调用将锁作为其第二个参数，而信号调用仅需要一个条件。 造成这种差异的原因在于，等待调用除了使调用线程进入睡眠状态外，还会让调用者睡眠 时释放锁。想象一下，如果不是这样：其他线程如何获得锁并将其唤醒？但是，在被唤醒 之后返回之前，pthread_cond_wait()会重新获取该锁。从而确保线程在运行的任何时间，它持有锁。
最后一点需要注注：等待线程在while循环中重新检查条件，而不是简单的if语句。在 后续章节中研究条件变量时，我们会详细讨论这个问题，但是通常使用while循环是一件简 单而安全的事情。
有一些 pthread 实现可能会错误地唤醒等待的线程。在这种情况下，没有重复检查，等待的线程会继续认 为条件已经改变。

请注注，有时候线程之间不用条件变量和锁，用一个标记变量会看起来很简单，很吸 引人。例如，我们可以重写上面的等待代码，像这样：`while (ready == 0) ; // spin`，相关的发信号代码看起来像这样：`ready = 1;`。
千万不要这么做。首先，多数情况下性能差（长时间的自旋浪费CPU）。其次，容易出 错。最近的研究\[X+10]显示，线程之间通过标志同步（像上面那样），出错的可能性让人吃 惊。在那项研究中，这些不正规的同步方法半数以上都是有问题的。不要偷懒，就算你想 到可以不用条件变量，还是用吧。
如果条件变量听起来让人迷惑，也不要太担心。后面的章节会详细介绍。在此之前， 只要知道它们存在，并对为什么要使用它们有一些概念即可。

**27.5 编译和运行**
本章所有代码很容易运行。代码需要包括头文件pthread.h才能编译。链接时需要pthread 库，增加-pthread标记。 
例如，要编译一个简单的多线程程序，只需像下面这样做：
`prompt> gcc -o main main.c -Wall -pthread`
只要main.c包含pthreads头文件，你就已经成功地编译了一个并发程序。

**27.6 小结**
**补充：线程 API 指导**
当你使用 POSIX 线程库（或者实际上，任何线程库）来构建多线程程序时，需要记住一些小而重要的事情：
- **保持简洁**。最重要的一点，线程之间的锁和信号的代码应该尽可能简洁。复杂的线程交互容易产生缺陷。
- **让线程交互减到最少**。尽量减少线程之间的交互。每次交互都应该想清楚，并用验证过的、正确的方法来实现（很多方法会在后续章节中学习）。
- **初始化锁和条件变量**。未初始化的锁和条件变量有时工作正常，有时失败，会产生奇怪的结果。
- **检查返回值**。当然，任何 C 和 UNIX 的程序，都应该检查返回值，这里也是一样。否则会导致古怪而难以理解的行为，让你尖叫，或者痛苦地揪自己的头发。
- **注意传给线程的参数和返回值**。具体来说，如果传递在栈上分配的变量的引用，可能就是在犯错误。
- **每个线程都有自己的栈**。类似于上一条，记住每一个线程都有自己的栈。因此，线程局部变量应该是线程私有的，其他线程不应该访问。线程之间共享数据，值要在堆（heap）或者其他全局可访问的位置。
- **线程之间总是通过条件变量发送信号**。切记不要用标记变量来同步。
- **多查手册**。尤其是 Linux 的 pthread 手册，有更多的细节、更丰富的内容。请仔细阅读！
