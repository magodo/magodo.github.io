---
layout: "post"
title: "C++11 thread, mutex, condition variable"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1. 线程

`std::thread` 是一个类，代表一个线程的执行。

## 1.1 创建线程

通常，它在构造的时候开始执行；也可以在构造的时候不传入参数，得到一个线程对象(但是处于一个特殊的状态，此时它不代表一个线程)，在想要执行的点再把它幅值为另一个线程：

{% highlight CPP linenos %}
    #include <thread>
    #include <mutex>
    #include <iostream>

    std::mutex g_lock;

    void thread_func()
    {
        g_lock.lock();
        std::cout << "TID: " << std::this_thread::get_id() << std::endl;
        g_lock.unlock();
    }

    int main()
    {
        std::thread t1(thread_func);        // 立即启动线程1
        std::thread t2;                     // 延迟启动线程2
        std::cout << "T2: " << t2.get_id() << std::endl;
        t2 = std::thread{&thread_func};     // 线程2此时启动
        std::cout << "T2: " << t2.get_id() << std::endl;

        t1.join();
        t2.join();
    }
{% endhighlight %}

输出：

{% highlight CPP linenos %}
    T2: thread::id of a non-executing thread
    TID: 140350045726464
    T2: 140350037333760
    TID: 140350037333760
{% endhighlight %}

此外，创建线程的时候传递的第一个参数可以是函数以外的可调用对象，包括：

* lambda
* callable class

## 1.2 线程的结束

可以在其他线程中调用`join`来等待指定的线程结束，这个操作会堵塞当前线程。注意，`join`的返回是空，也就意味着线程不会向其他线程返回任何值。

也可以用`detach`使指定线程脱离管理。

## 1.3 在线程中访问当前线程

`std::this_thread` 这个命名空间定义了一组用于访问当前线程的函数，包括：

* `get_id`: 得到当前线程的ID (注意，这个id并不是实现中定义的线程ID，需要对线程实例调用`native_handle`)
* `yield`: 主动放弃当前时间片，允许操作系统调度其他线程
* `sleep_until`/`sleep_for`: 使当前线程堵塞指定时间或者堵塞到指定时间。由于多线程管理操作，这个时间会有一定误差。

## 1.4 传递参数给线程函数

传递的参数一般都是以值传递的方式传给线程函数。如果想要传递引用，必须适用`std::ref` 或者 `std::cref`.

## 1.5 其他

线程还提供了`swap`操作，用于交换两个线程对象的底层handle。

此外，线程中抛出的异常是无法在主线程中被catch，只能在该线程内部被catch。关于这部分，可以参考[这里](https://www.codeproject.com/articles/598695/cplusplus-threads-locks-and-condition-variables)获得更多信息。

# 2. 互斥锁

互斥锁(mutex)，提供了对共享资源的保护。

## 2.1 基本互斥锁

C++11中定义了如下基本互斥锁：

1. `mutex`: 提供了基本的`lock()`,`unlock()`和`try_lock()`函数. 在pthread的实现中，对于已经unlock的mutex如果再次调用unlock, 结果是未定义的。
2. `recursive_mutex`: 允许同一个线程多次对其上锁。
3. `timed_mutex`: 类似`mutex`,但是多了`try_lock_for()`和`try_lock_until`两个操作。
4. `recursive_timed_mutex`: timed_mutex和recursive_mutex的结合。

## 2.2 RAII-style 互斥锁

由于基本的锁在使用时，必须要保证lock和unlock的成对使用，稍不注意就容易导致死锁等问题（例如忘记解锁，或者不正确的上锁顺序等）。因此，C++标准为用户提供了更高级的RAII-style的锁的封装类。

(*RAII-style*(Resource Acquisition Is Initialization) 表示一个对象所使用的资源需要在构造之后就已经就绪，在析构的时候自动释放)

这些封装类能够根据作用域，自动上锁和解锁。其中包括：

* `lock_guard<class Mutex>`: 当对象构造之后，获得锁；当对象析构的时候，自动释放锁。是一个non-copyable 的类
* `unique_lock<class Mutex>`: 和`lock_guard`类似提供了RAII,但是给予更多构造函数，更灵活。

以上两个封装互斥锁类的构造函数的输入参数中，除了第一个输入的基本互斥锁（2.1中提到的4个）以外，还接受第二个所谓的strategy参数：

* `adopt_lock`:假定当前线程已经hold该互斥锁
* `defer_lock`(unique_lock only): 推迟wrapper对锁的上锁操作
* `try_to_lock`(unique_lock only): 尝试去上锁

举个官网的例子：

{% highlight CPP linenos %}

	#include <mutex>
	#include <thread>
	 
	struct bank_account {
		explicit bank_account(int balance) : balance(balance) {}
		int balance;
		std::mutex m;
	};
	 
	void transfer(bank_account &from, bank_account &to, int amount)
	{
		// lock both mutexes without deadlock
		std::lock(from.m, to.m);
		// make sure both already-locked mutexes are unlocked at the end of scope
		std::lock_guard<std::mutex> lock1(from.m, std::adopt_lock);
		std::lock_guard<std::mutex> lock2(to.m, std::adopt_lock);
	 
	// equivalent approach:
	//    std::unique_lock<std::mutex> lock1(from.m, std::defer_lock);
	//    std::unique_lock<std::mutex> lock2(to.m, std::defer_lock);
	//    std::lock(lock1, lock2);
	 
		from.balance -= amount;
		to.balance += amount;
	}
	 
	int main()
	{
		bank_account my_account(100);
		bank_account your_account(50);
	 
		std::thread t1(transfer, std::ref(my_account), std::ref(your_account), 10);
		std::thread t2(transfer, std::ref(your_account), std::ref(my_account), 5);
	 
		t1.join();
		t2.join();
	}

{% endhighlight %}

注意上面用到的`std::lock`, 这不同于对基本互斥类的`lock`操作。前者可以避免死锁(它的内部使用了`lock`, `try_lock`和`unlock`来实现的)。

# 3. 条件变量

条件变量是C++11中提供的另一种同步原语。它可以是一个或多个线程被阻塞，直到：

1. 收到其他线程的通知(notification)
2. 或者 超时
3. 或者 发生了伪唤醒(spurious wake-up)

在`<condition_variable>`头文件中，它有两种实现：

1. `condition_variable`: 要求任何想要被阻塞的线程都必须持有`std::unique_lock`
2. `condition_variable_any`: 一种更通用（灵活）的实现，允许任何基本互斥锁。不过这种条件变量的性能较差

## 3.1 基本流程

* 必须有至少一个线程在等待条件变量满足。这个等待线程要先持有一个`unique_lock`,这个互斥锁被传入`wait()`函数。该函数释放这个持有的互斥锁，并且挂起线程。直到条件变量满足(signaled)，于是这个线程被唤醒，并且继续持有互斥锁。
* 必须有至少一个线程signal互斥变量。这既可以通过`notify_one()`来使其中一个waiting的线程被唤醒，也可以通过`notify_all()`来使所有正在等待该条件变量的线程被唤醒。
* 有时候在多核系统中，由于某种原因会发生伪唤醒(spurious wake-up).它会使waiting的线程被唤醒，即使实际上没有线程signal条件变量。这意味着条件变量实际上还没有被更改。因此，对于这些waiting的线程必须要在wake之后检查条件变量是否被改变了，如果没有则继续wait。

    这一般利用while循环来写:
   
        std::mutex              g_lockqueue;
        std::condition_variable g_queuecheck;

        ...

        std::unique_lock<std::mutex> locker(g_lockqueue);

        while(!g_notified) // used to avoid spurious wakeups 
        {
            g_queuecheck.wait(locker);
        }

    在C++11中可以对`wait()`函数传入一个lambda表达式来进行判断:

        std::mutex              g_lockqueue;
        std::condition_variable g_queuecheck;

        ...

        std::unique_lock<std::mutex> locker(g_lockqueue);
        g_queuecheck.wait(locker, [&]{return g_notified;});


# 引用

[1] [C++11 thread, locks and condition variables](https://www.codeproject.com/articles/598695/cplusplus-threads-locks-and-condition-variables)
