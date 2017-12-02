---
layout: post
title: UNIX信号
date: 2017-12-02 20:27:41 +0800
comments: true
categories:
- operating system
- programming language
- programming practice
---

# Introduction

UNIX信号在编程中用的不多，但是对于长时间运行的程序还是很有用的，比如通知程序重新加载配置文件、提早退
出打印一些信息、监控子进程是否还活着。

本文将澄清UNIX信号的概念，使用信号的注意事项及常用的编程模式。
<!--more-->

# UNIX信号的概念

当触发信号产生的事件发生时（如硬件异常、定时器过期、终端命令或者调用`kill()`发送信号），信号将发送给
相关进程或线程。当产生信号时将决定该信号时发送给进程或进程中的某个线程。如果信号的产生原因与线程有关
（如该线程产生了内存访问异常），则发送给该线程；否则，发送给该进程（例如，终端活动）。

信号产生后，如果没有被阻塞，则投递（deliver）给相关进程或线程执行对应的动作，或者被接受（accept，调用`sigwait()`系列函数）。

在线程产生到投递之间，信号处于未决状态（pending）。信号在投递前也可以被阻塞。发送给线程的信号，如果被
阻塞，且该信号的相关动作不是忽略，则该信号将一直处于未决状态，直到：1）解除阻塞；2）被接受；3）其相关
动作设置为忽略。发送给进程的信号，将发送给愿意接受该信号或没有阻塞该信号的线程处理；如果没有这样的线程
则一直处于阻塞状态，直到被某个线程：1）解除阻塞；2）接受；3）设置为忽略该信号。如果发送给进程的信号被
阻塞且其相关动作是忽略时，该信号是否直接丢弃还是处于未决状态是不确定的。

POSIX标准并没有要求信号可以排队，所以一个未决信号重复产生时可能只会投递或接受一次。另外，多个未决信号
在投递或接受时的顺序也是不确定的。

每个信号都有一个与之关联的动作，如下3种：

- 默认
  - 默认动作由系统在程序启动时设置好。注意：有些信号的默认动作是调用`_exit()`终止程序。
- 忽略
  - 注意：`SIGKILL`和`SIGSTOP`是不能设置为忽略的。
- 用户自定义函数
  - 用户可设置自定义函数捕获一个信号，但是`SIGKILL`和`SIGSTOP`是无法捕获的。另外，`SIGSEGV`，`SIGILL`，
  `SIGBUS`和`SIGFPE`的捕获函数必须终止程序而不能正常返回。
  - 捕获函数的实现必须保证信号安全。

# 信号安全

POSIX要求信号的捕获函数必须满足以下要求：

- 只能访问`errno`、`volatile sig_atomic_t`类型的全局变量或者原子类型的全局变量。
- 只能调用规定的信号安全函数。
- 注意保存和恢复`errno`.

所以说信号的捕获函数很难实现复杂的逻辑，甚至打印消息也不行（`printf()`不是信号安全的）。只能通过
原子变量传递一些简单的信息，线程条件变量和锁都不能用（使用线程锁可能导致死锁）。

# 如何注册信号处理函数

系统提供了两个API注册信号处理函数，简单的版本定义如下：

``` c
typedef void (*sig_t) (int);

sig_t signal(int sig, sig_t func);
```
注册一个函数，并返回之前注册的函数。当信号投递后信号处理函数执行期间，该信号将被阻塞直到执行完毕，
防止信号重入。

复杂的版本定义如下，可设置信号处理有关的更多属性。

``` c
struct sigaction {
    union __sigaction_u __sigaction_u;  /* signal handler */
    sigset_t sa_mask;               /* signal mask to apply */
    int     sa_flags;               /* see signal options below */
};

union __sigaction_u {
     void (*__sa_handler)(int);
     void (*__sa_sigaction)(int, siginfo_t *, void *);
};

#define sa_handler      __sigaction_u.__sa_handler
#define sa_sigaction    __sigaction_u.__sa_sigaction

int sigaction(int sig, const struct sigaction *restrict act, struct sigaction *restrict oact);
```
这个API相比于上面的简单版本可以设置信号处理函数执行期间是否阻塞其它信号，或者当信号中断一个慢速的IO
时是否自动重新执行IO，细节可参考`sigaction(2)`.
     
# 信号处理的编程模式

信号处理函数的要求非常严格，除了设置原子变量外也做不了其它事情，下面举一个简单的例子用终端产生信号让
程序提前结束。

``` c++
#include <unistd.h>
#include <signal.h>
#include <thread>

volatile sig_atomic_t g_quit = 0;

void stop(int)
{
    g_quit = 1;
}

int main()
{
    signal(SIGINT, stop);
    std::thread([]() {
                    while (!g_quit) {
                        sleep(1);
                    }
                }).join();
    return 0;
}
```
通知线程退出的前提是该线程周期性检测是否需要退出。但是在比较复杂的场景下，周期检测并不可取，
需要采取更复杂的通信机制来通知线程退出，比如条件变量，但是信号处理函数是不能操作条件变量的，
怎么通知呢？

回忆一下，信号除了投递调用信号处理函数外还可以被线程主动接受，这样我们可以实现任意复杂的逻辑。
创建一个线程专门接受信号，而其它所有线程都阻塞这些信号，由这个专用线程来实现通知逻辑。

```c++
#include <thread>
#include <signal.h>
#include <mutex>
#include <condition_variable>

int main()
{
    // Block SIGINT for all threads.  The subsequently created threads
    // will inherit this mask.
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGINT);
    pthread_sigmask(SIG_BLOCK, &set, NULL);

    std::condition_variable cv;

    // Start a thread dedicated to process signals.
    std::thread([&] {
                    int sig;
                    sigwait(&set, &sig);
                    if (sig == SIGINT) {
                        cv.notify_one();
                    }
                }).detach();

    // The worker thread.
    std::thread([&] {
                    std::mutex mutex;
                    std::unique_lock<std::mutex> lock(mutex);
                    cv.wait(lock);
                }).join();
    return 0;
}
```

# Conclusion

在处理信号时，如果只需要周期性检查信号是否已经发生，则可注册一个信号处理函数即可搞定。但是如果需要
处理复杂逻辑，则最好创建一个专用线程来同步处理信号。

# References

- [POSIX Spec](http://pubs.opengroup.org/onlinepubs/9699919799/), System Interfaces, Singal Concepts