---
layout: post
title: "Concurrent Porgramming"
categories: blog
tags: ['program']
published: true
comments: true 
excerpted: |
    You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site ...

script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

Mordern OS provide 3 basic approach for building concurrent programs

1. Process
  * each logical control flow is a process
  * processes are scheduled and maintained by kernel
  * use explicit _interprocess communication(IPC)_ mechanism for communication

2. I/O multiplexing
  * application explicitly schedule their own logical flow in context of a process
  * logical flows are modeled as state machines that the main program explicitly transitions from state to state as a result of data arriving on file descriptors
  * since the program is a single process, all flows share the same address space

3. Threads
  * think of threads as a hybrid of the other two approcheas: scheduled by kernel, sharing the same virtual address space

# Concurrent programming with __Process__

__NOTE__

* Once a child process is forked, for example, in an server-client model, remember to release unused resource in both parent process and child process, for example, connection fd in parent process and listening fd in child process since they don't need it

__Pros & Cons__

* Pros:
    - Have separate address space, which eliminates accidentally overwrite the VM of another process
    - Processes are scheduled automatically by the kernel

* Cons:
    - Have separate address space, which is more difficult to share state information. To share information, they must use explicit IPC mechanisms
    - Tends to be slower because the overhead for process control and IPC is high

# Concurrent programming with __I/O Multiplexing__

The basic idea is to use the `select` function to ask the kernel to suspend the process, returing control to the application only after one or more I/O events have occured, as in the following examples:

* Return when any descriptor in the set {0, 4} is ready for reading.
* Return when any descriptor in the set {1, 2, 7} is ready for writing.
* Timeout if 152.13 seconds have elapsed waiting for an I/O event to occur.

`select` prototype:

    int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);

In simple case, `select` takes two inputs:

* a descriptor set called the _readfds_
* cardinality _n_ of the readfds

`select` blocks until at least one descriptor in the readfds is ready for reading. A descriptor _k_ is ready for reading if and only if a request to read 1 byte from that descriptor would not block.

As a side effect, `select` modifies the fd\_set pointed to by argment _readfds_ to indicate a subset of the _readfds_ called the _ready set_, consiting of the descriptors in the readfds that are ready for reading.

The value returned indicates the cardinality of the _ready set_

__Pros & Cons__

* Pros: 
    - Give programmer more control over the behavior for their programs. E.g we can imagine writing an event-driven concurrent server that gives preferred service to some clients
    - Since I/O multiplexing runs in the context of a single process, thus every logical flow has access to the entire address space of the process. This makes it easy to share data between flows
    - Is able to be debugged as any sequential program, using a familiar debugging tool as GDB
    - More efficient

* Cons:
    - Coding complexity


# Concurrent programming with __Threads__

## Introducing thread



### General Introduction

* The threads are scheduled by kernel
* Each thread has its own _thread context_, including a unique integer _thread ID_(TID), stack, stack pointer, program counter, general-purpose registers and condition codes
* All threads share the entire VM space of one process, including its code, data, heap, shared libraries and open files
* _main thread_ is the first thread to run in the process,  _peer threads_  are the following created threads. Together they form a _pool_ of peers, independent of which threads were created by which other threads
* The code and local data for a thread is encapsulated in a _thread routine_, it takes a generic pointer as input and returns a generic pointer


### Thread creation

New thread is created via calling `pthread_create`:

    #include <pthread.h>

    typedef void *(func)(void *);
    
    int pthread_create(pthread_t *tid, pthread_attr_t *attr,
                       func *f, void *arg);

When `pthread_create` returns, argument _tid_ contains the ID of the newly created thread.

The new thread can determine its own thread ID by calling `pthread_self`:

    #include <pthread.h>

    pthread_t pthread_self(void);

Take note to _arg_ argument in `pthread_create`. When _arg_ points to a local variable in creating thread, it might raise __race__ condition against following thread creation.

For example, imagine there is a continuous creation of threads:

{% highlight C linenos %}
void main(){
    int connfd;

    ...

    while(1){
        connfd = accpet(listenfd, (SA *)&clientaddr, &clientlen);
        pthread_create(&tid, NULL, thread, &connfd);
    }
}

void *thread(void *vargp){
    int connfd = *((int*)vargp);
    ...
}
{% endhighlight %}

In this case, there is a __race__ condition between line 13 and line 7. When line 13 is executed after line 7, then it will cause two thread using identical connection file descriptor.

In order to avoid potentially dadly __race__, we should assign each _arg_ to its own dynamically allocated memory block, which is tended to be used exclusively by the created thread. Therefore, the changed code should look like this:

{% highlight C linenos %}
void main(){
    int *connfdp;            // define a pointer

    ...

    while(1){
        connfdp = (int*)malloc(sizeof(int));  // dynamically allocate thread-specific argument
        *connfdp = accpet(listenfd, (SA *)&clientaddr, &clientlen);
        pthread_create(&tid, NULL, thread, connfd);
    }
}

void *thread(void *vargp){
    int connfd = *((int*)vargp);
    free(vargp)               // thread is responsible to free the allocated memory
    ...
}

{% endhighlight %}


### Thread termiation

A thread terminates in one of the following ways:

* The thread terminates _implicitly_ when its top-level thread routine returns
* The thread terminates _explicitly_ by calling the `pthread_exit` function. If the _main thread_ calls `pthread_exit`, it waits for all other _peer threads_ to terminate, and then terminates the main thread and the entire process with a return value of `thread_return`

<b></b>

    #include <pthread.h>

    void pthread_exit(void *thread_return);

* _main thread_ or _peer thread_ calls `exit` function, which terminates the process and all threads assciated
* Some other _peer thread_ terminates the current thread by calling the `pthread_cancel` function with the TID of the current thread

<b></b>

    #include <pthread.h>

    int pthread_cancel(pthread_t tid);

### Thread repeatedly termination

Threads wait for other threads to terminate by calling the `pthread_join` function:

    #include <pthread.h>

    int pthread_join(pthread_t tid, void **thread_return);

The `thread_join` function blocks until thread _tid_ terminates, assigns the generic (void \*) pointer returned by the thread routine to the location pointed to by *thread_return* , and then _reaps_ any memory resources held by the terminated thread.

Note: unlike Unix `wait` function, the `pthread_join` function can only wait for a _specific_ thread to terminiate. There is no way to instruct `pthread_wait` to wait for an _arbitrary_ thread to terminate.


### Thread detachment

At any point in time, a thread is _joinable_ or _detached_.

* A _joinable_ thread can be reaped and killed by other threads. Its memory resrouces are not freed until it is reaped by another thread;
* A _detached_ thread cannot be reaped or killed by other threads. Its memory resources are freed automatically by the system when it terminates.

By default, threads are created _joinable_. In order to avoid memory leaks, each joinable thread should:
* be explicitly reaped by another thread;
or
* be detached by a call to `pthread_detach` function:

<b></b>

    #include <pthread.h>

    int pthread_detach(pthread_t tid);


### Thread initialization

The `pthread_once` function allows you to initialize the state associated with a thread routine.

    #include <pthread.h>

    pthread_once_t once_control = PTHREAD_ONCE_INIT;

    int pthread_once(pthread_once_t *once_control,
                     void (*init_routine)(void));

The *once_control* is a global or static variable that is always initialized to `PTHREAD_ONCE_INIT`. 

The first time any thread in a process call `pthread_once` with an argument of *once_control*, it invokes `init_routine`, which is a function with _no_ input arguments that returning nothing(it is always used to define some global variables).

Subsequent calls from any other threads inside this process to `pthread_once` with the same *once_control* variable do nothing. 


## Share variables in thread programs

The variable is _shared_ if and only if multiple threads reference some instance of the variable.

Each thread has its own separate _thread context_, which includes:

  * thread ID (kernel space)
  * stack (mem)
  * stack pointer (reg)
  * program counter (reg)
  * condition codes (reg)
  * general-purpose register (reg)

Each thread shares the rest of the process context with the other threads, which includes:

  * entire user virtual address space, includes:
      * read-only text (code)
      * r/w data
      * heap
      * shared library code and data
  * same set of open files

In operational sense, it is impossible for one thread to read or write the register values of another thread; On the other hand, any thread can access any location in the shared VM (including other threads' stack). Thus, registers are never shared, whereas VM is always shared(stack is not clean).


### mapping variables to memory

* _Global variables_: At run time, the r/w area of VM contains exactly one instance of each global variables that can be referenced by any thread.
* _Local automatic variables_: At run time, each thread's stack contain its own instance of any local automatic variables.
* _Local static variables_: Same as _Global variables_.



## synchronizing threads with semaphores


### machine instrcution interleave

When peer threads run concurrently on a uniprocessor, the _machine instructions_ are completed one after the other in some order. This will cause nasty synchronizing errors.

For example:

{% highlight C linenos %}
volatile int cnt = 0;    // "volatile" is necessary for global variables shared by threads

void main(){
    int niters = 100;
    ...
    pthread_create(&tid1, NULL, thread, &niters);
    pthread_create(&tid2, NULL, thread, &niters);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    ...
}

void *thread(void *vargp){
    int i, niters = *((int*)vargp);

    for (i = 0; i < niters; i++)
        cnt++;

    return NULL;
}
{% endhighlight %}

In this case, the resulting _cnt_ is not ensured to be `2 * niters`. This is because, the for loop body in line 17 consists of 3 _machine instructions_ (load, update, store). _In general, there is no way for you to predict wether the operating system will choose a correct ordering for your threds._


### progress graph

A _progress graph_ models the execution of _n_ concurrent threads as a trajectory through an _n_-dimensional Cartesian space. Each axis _k_ corresponds to the progress of thread _k_. Each point represents the state where the corresponding thread has completed last instruction. The origin of the graph corresponds to the _initial state_ where none of the threads has yet completed an instruction.

A progress graph models instruction execution as a _transition_ from one state to another. A transition is represented as a directed edge from one point to an adjacent point. Legal transitions move to the right or up. The execution history of a program is modeled as a _trajectory_ through the state space.

Following is an example _trajectory_ of code above:

![trajectory1](/blog/images/concurrent-programming/trajectory1.png)

In this case, for thread _i_, the instructions (L<sub>i</sub>, U<sub>i</sub>, S<sub>i</sub>) that manipulate the contents of the shared variable _cnt_ consitute a _critical section_ that should not be interleaved with the critical section of the other thread. The phenomenon in general is known as _mutual exclusion_.

![trajectory2](/blog/images/concurrent-programming/trajectory2.png)

In order to guarantee correct execution of concurrent program that shares global data structures, we must _synchronize_ the threads so that they always have a safe trajectory.


### Semaphore Introduction

A semaphore, _s_, is a global variable with a _nonnegative_ integer value that can only be manipulated by two special operations, called _P_ and _V_:

* _P_(s): 
    * If _s_ is nonzero, _P_ decrements _s_ and returns immediately;
    * If _s_ is zero, then suspend the thread until _s_ becomes nonzero and the process is restarted by a _V_ operation. After restarting, the _P_ operation decrements _s_ and returns.

* _V_(s): increment _s_ by 1

NOTE:

  * The test and decrement operations in _P_ occur indivisibly;
  * The increment operation in_V_ occurs indivisibly;
  * _V does not_ define the order in wihch waiting threads are restarted. Thus, when several threads are waiting at a semaphore, you cannot predict which one will be restarted as a result of the V.

The semaphore ensures a running program can never enter a state where a properly initialized semaphore has a negative value. This property, known as the _semaphore invariant_, provides a powerful tool for controlling the trajectories of concurrent programs.

The Posix standard defines a variety of functions for semaphores:

    # include <semaphore.h>

    int sem_init(sem_t *sem, 0, unsigned int value);
    int sem_wait(sem_t *s);    /* P(s) */
    int sem_post(sem_t *s);    /* V(s) */


### Use semaphore for mutual exclusion

The basic idea is to associate a semaphore _s_, initially 1, with each shared variable (or related set of shared variables) and then surround the correspoding critical section with _P_(s) and _V_(s) operations.

A semaphore that is used in this way to protect shared variables is called a <a name="bin_sem"></a>__binary semaphore__ because its value is always 0 or 1. Binary semaphores whose purpose is to provide mutual exclusion, AKA _mutexes_.

  * Performing a _P_ operation on a mutex is called _locking_ the mutex
  * Performing a _V_ operation on a mutex is called _unlocking_ the mutex
  * A semaphore that is used as a counter for a set of available resources is called a _counting semaphore_

Use this method, we could ensure the _cnt example_ above to make the critical region as a forbidden area on progress graph, which ensures it is impossible for multiple threads to be executing instructions in the enclosed critical region at any point in time. In other words, the semaphore operations ensure mutually exclusive access to the shared resources. 

The code should be changed to:

{% highlight C linenos %}
volatile int cnt = 0;    // "volatile" is necessary for global variables shared by threads
sem_t mutex_cnt;         // 1. Semaphore that protects counter

void main(){
    int niters = 100;
    ...
    sem_init(&mutex_cnt, 0, 1);   // 2. mutext_cnt = 1
    ...
    pthread_create(&tid1, NULL, thread, &niters);
    pthread_create(&tid2, NULL, thread, &niters);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);
    ...
}

void *thread(void *vargp){
    int i, niters = *((int*)vargp);

    for (i = 0; i < niters; i++){
        sem_wait(&mutex_cnt);     // 3. P
        cnt++;
        sem_post(&mutext_cnt);    // 4. V
    }
    return NULL;
}
{% endhighlight %}

With changes above, we could get following _progress graph_ with _forbidden region_ that surrounds the unsafe region:

![trajectory3](/blog/images/concurrent-programming/trajectory3.png)



### Use semaphore to schedule shared resources

A thread could use semaphore operation to notify another thread that some condition in the program state has become true.


## using threads for parallelism

The set of all programs can be partitioned into the disjoint set of _sequential_ and _concurrent_ programs.

* A sequential program is of a single logical flow
* A concurrent program is of multiple concurrent flows. A parallel program is a concurrent program running on multiple processors

# Concurrency Issues

## Thread safety

A function is said to be _thead-safe_ if and only if it will always produce correct results when called repeatedly from one or multiple concurrent threads

Normally, there are four classes of _thread-unsafe_ functions:

* __Class 1__: Functions that do not protect shared variables

  **Fix**

  - Option1: Rewrite the **callee** function with a mutex to protect the shared variables with synchronization operations such as _P_ and _V_

  - Option2: Rewrite the **caller** function to associate a mutex with the callee function to protect the shared variables

<b></b>

* __Class 2__: Functions that keep state in some static variables(e.g. variables in .data section) across multiple invocations

  **Fix**

  - Rewrite the **callee** and **caller** function so that **callee** function don't use any shared data, relying instead on the **caller** to pass state information in arguments. (in this case, the callee is a [_reentrant function_ ](#reentrancy)

<b></b>

* __Class 3__: Functions that return a pointer to a static variable

  **Fix**

  - Option1: Rewrite the **callee** and **caller** function so that **caller** function pass the address of the variable to **callee** in which to store the results.

  - Option2: Write a thread-safe wrapper function using  _lock-and-copy_ technique and replace all calls in **caller** function to the thread-unsafe function with calls to the wrapper. The basic idea of _lock-and-copy_ technique is to associate a mutex with the thread-unsafe function. At each call, lock the mutex, call the thread-unsafe function, copy the result returned by function to a thread-private memory location, and then unlock the mutex. 

<b></b>

* __Class 4__: Functions that call thread-unsafe functions

  If a function _f_ calls a thread-unsafe function _g_, it depends:

  - _g_ is _Class 1_ or _Class 3_

    _f_ could use mutex for invoking _g_ to make _f_ itself to be thread-safe

  - _g_ is _Class 2_

    Only when _g_ is rewritten to be thread-safe, otherwise, _f_ is thread-unsafe

## Reentrancy

_Reentrant function_ is characterized by the property that it doesn't reference _any_ shared data when they are called by multiple threads. It is one class of _thread-safe_ function.

There are two favors of _reentrant function_:

* _explicitly reentrant function_: If all function arguments are passed by value(i.e. no pointers) and all data references are to local automatic stack variables(i.e. no references to static or global variables), then the function is _explicitly reentrant function_. We can assert its _reentrancy_ regardless of how it is called
* _implicitly reentrant function_: If some arguments are pointers, than we have an _implicitly reentrant function_. We can't tell if its _reentrancy_ in this case. It is reentrant only if the pointers point to nonshared data.

## Use existing library functions in thread programs

Most Unix functions, including functions defined in standard C library, are _thread-safe_, with only a few exceptions.

Here lists the common exceptions:

|------------------------+-------------------+-------------------------|
| Thread-unsafe function |Thread-unsafe class| Unix thread-safe version|
|:-----------------------|:------------------|:------------------------|
| rand                   | 2                 | rand_r                  |
|------------------------|-------------------|-------------------------|
| strtok                 | 2                 | strtok_r                |
|------------------------|-------------------|-------------------------|
| asctime                | 3                 | asctime_r               |
|------------------------|-------------------|-------------------------|
| ctime                  | 3                 | ctime_r                 |
|------------------------|-------------------|-------------------------|
| gethostbyaddr          | 3                 | gethostbyaddr_r         |
|------------------------|-------------------|-------------------------|
| gethostbyname          | 3                 | gethostbyname_r         |
|------------------------|-------------------|-------------------------|
| inet_ntoa              | 3                 | (none)                  |
|------------------------|-------------------|-------------------------|
| localtime              | 3                 | localtime_r             |
|------------------------|-------------------|-------------------------|

## Races

A _race_ occurs when the correctness of a program depends on one thread reaching point _x_ in its control flow before another thread reaches point _y_.

Take following example to illustrate:

{% highlight C linenos %}
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define N 4

void *thread(void *vargp);

int main(){
    pthread_t tid[N];
    int i;

    for (i = 0; i < N; i++)
        pthread_create(&tid[i], NULL, thread, &i);
    for (i = 0; i < N; i++)
        pthread_join(tid[i], NULL);
    exit(0);
}
void *thread(void *vargp){
    int myid = *((int*)vargp);
    printf("Hello from thread %d\n", myid);
    return NULL;
}

{% endhighlight %}

The output is:

    Hello from thread 3
    Hello from thread 0
    Hello from thread 0
    Hello from thread 0

This incorrect result is because of _race_ between line 13 and line 20.

To eliminate the race, we can use one of following 2 approaches:

1. Dynamically allocate a separate block for each integer ID, and pass the thread routine a pointer to this block. Notice that the thread is responsible to free the block in order to avoid a memory leak;
2. Schedule access to integer ID via semaphore.

{% highlight C linenos %}
/* Dynamically allocate thread-specific block for each ID */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

#define N 4

void *thread(void *vargp);

int main(){
    pthread_t tid[N];
    int i, *id;

    for (i = 0; i < N; i++){
        id = malloc(sizeof(int));              // allocate thread-specific block
        *id = i;
        pthread_create(&tid[i], NULL, thread, id);
    }
    for (i = 0; i < N; i++)
        pthread_join(tid[i], NULL);
    exit(0);
}

void *thread(void *vargp){
    int myid = *((int*)vargp);
    free(vargp);                               // free thread-specific block
    printf("Hello from thread %d\n", myid);
    return NULL;
}

{% endhighlight %}

{% highlight C linenos %}
/* Use semaphore to schedule resource access */

#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <semaphore.h>

#define N 4

void *thread(void *vargp);

sem_t sem;                    // define a semaphore to make "i" mutual exclusive

int main(){
    pthread_t tid[N];
    int i;

    sem_init(&sem, 0, 0);                    // init sem with value 0
    for (i = 0; i < N; i++){
        pthread_create(&tid[i], NULL, thread, &i);
        sem_wait(&sem);                      // P, wait until last thread finish store i
    }
    for (i = 0; i < N; i++)
        pthread_join(tid[i], NULL);
    exit(0);
}

void *thread(void *vargp){
    int myid = *((int*)vargp);
    sem_post(&sem);                          // V, after finish store i, increase sem
    printf("Hello from thread %d\n", myid);
    return NULL;
}

{% endhighlight %}

## Deadlocks

Semaphores introduce the potential for a nasty kind of run-time error, called _deadlock_. It means _a collection of threads are blocked, waiting for a condition that will never be true_.

The progress graph is like below:

![trajectory_deadlock](/blog/images/concurrent-programming/trajectory_deadlock.png)

From this graph, we can glean some important insights about deadlock:

* The programmer has incorrectly ordered the P and V operations such that the forbidden regions for the two semaphores create a closed, left-bottom corner _deadlock region_. If a trajectory happends to touch a state in the deadlock region, then deadlock is inevitable. Trajectories can enter deadlock regions, but they can never leave!
* Deadlock is not predictable.

Programs deadlock for many reasons and avoiding them is a difficult problem in general. However, when [_binary semaphores_](#use-semaphore-for-mutual-exclusion)) are used for mutual exclusion, then you can apply the following simple and effective rule to avoid deadlocks:

  __Mutex lock ordering rule__: A program is deadlock-free if, for each pair of mutexes ( _s_, _t_, _k_, ...) in the program, each thread that holds _s_, _t_ and _k_ simultaneously, locks these semaphores in the same order.

For example, we can fix the deadlock above by locking _s_ first, then _t_ in each thread:

![trajectory_deadlock_free](/blog/images/concurrent-programming/trajectory_deadlock_free.png)
