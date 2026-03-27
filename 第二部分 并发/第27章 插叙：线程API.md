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
第一个参数`thread`是指 向 `pthread_t `结构类型的指针，我们将利用这个结构与该线程交互，因此需要将它传入 `pthread_create(`)，以便将它初始化。
第二个参数 `attr` 用于指定该线程可能具有的任何属性。一些例子包括**设置栈大小**，或关 于该**线程调度优先级**的信息。一个属性通过单独调用`pthread_attr_init()`来初始化。有关详细信 息，请参阅手册。但是，在大多数情况下，默认值就行。在这个例子中，我们只需传入`NULL`。
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