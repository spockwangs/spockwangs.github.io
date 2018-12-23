---
layout: post
title: 线程私有存储
date: 2017-12-01 21:08:54 +0800
comments: true
categories:
- computer system
- programming language
- concurrent programming
---

# Introduction

线程私有变量（Thread Local Storage）之于线程相当于静态变量之于进程，与进程变量相比是每个线程都有一份，
也就是所谓的“私有”。也可以把线程私有变量理解为key-value对，其中key是线程ID。它的主要作用是在多线程编程
中避免锁竞争的开销。本文将重点介绍线程私有变量的几种形式、用法及其背后的实现原理。
<!--more-->

# 显示TLS

POSIX线程库提供了如下API管理TLS：

- `pthread_key_create()`
   - 创建一个TLS变量，并设置析构函数
- `pthread_setspecific()`
   - 给TLS变量赋值
- `pthread_getspecific()`
   - 获取TLS变量的当前值
- `pthread_key_delete()`
   - 回收TLS变量，但是注意并不调用TLS的析构函数

下面是一个简单的例子。

~~~c++
#include <pthread.h>
#include <thread>
#include <cassert>
#include <iostream>

using namespace std;

void cleanup(void* p)
{
    cout << "clean" << endl;
    int* pi = reinterpret_cast<int*>(p);
    delete pi;
}

int main()
{
    pthread_key_t tls_key;
    int err = pthread_key_create(&tls_key, cleanup);
    assert(err == 0);
    std::thread([=] {
                    void* p = pthread_getspecific(tls_key);
                    // 还没设置过值，所以是空指针。
                    assert(p == nullptr);
                    p = new int(3);
                    int err = pthread_setspecific(tls_key, p);
                    assert(err == 0);
                    p = pthread_getspecific(tls_key);
                    int a = *reinterpret_cast<int*>(p);
                    assert(a == 3);
                }).join();
    void *p = pthread_getspecific(tls_key);
    // 主线程的TLS与子线程是分开的，所以还是空指针。
    assert(p == nullptr);
    
    // 由于子线程已退出，所以这里可以回收|tls_key|了。
    err = pthread_key_delete(tls_key);
    assert(err == 0);
    return 0;
}
~~~

有几个值得注意的地方：

- `pthread_key_delete()`不会自动调用每个线程TLS变量的析构函数，而且在回收key后线程在退出时也不会再调
用析构函数，所以必须要确定: 1) 所有线程已经析构TLS变量（要么显示析构，要么线程退出，像上面的例子）; 2)而且不再使用
该key时才能回收这个key.  这两个条件一般难以确定，所以实际使用时一般也不会回收key.
- `pthread_setspecific()`在覆盖之前的value时不会主动调用析构函数，调用线程必须自己保证释放资源。
- TLS key的个数是有限的，通过`PTHREAD_KEYS_MAX`(limits.h)获取（在我电脑上是512）。下面我会
讲到如何突破这个限制。

# 显示TLS的实现

在Linux中每个进程有一个全局的数组管理TLS key，定义类似如下：
~~~ cpp
struct pthread_key_struct {
    uintptr_t seq;
    void (*destructor)(void*);
} __pthread_keys[PTHREAD_KEYS_MAX];
~~~

每个元素用于存储key的序列号（用于判断key是否有效，下面解释）和其对应的析构函数。序列号初始值为0.
`pthread_create_key()`会从该数组中找到一个还未分配的元素，将其序列号加1，记上析构函数地址，并将其下标
作为TLS key返回。那么如何判断一个key是否已分配呢？

~~~ C++
#define KEY_UNUSED(seq) (((seq) & 1) == 0)
~~~
若序列号为偶数则表示未分配，分配时将其加1变成奇数即可。这个操作采用原子CAS来完成，以保证线程安全。在
`pthread_key_delete()`时也会将序列号加1，表示可以继续使用，通过序列号机制来保证回收的key不会被复用
（复用的key会导致线程在退出时可能会调用错误的析构函数）。但是一直加1会导致序列号回绕，还是会复用key。
所以创建key时会检查是否有回绕风险，如果有则创建失败。

~~~ C++
#define KEY_USABLE(seq) (((uintptr_t)(seq)) < ((uintptr_t) ((seq) + 2)))
~~~

每个线程都有一个自己的线程控制块TCB（管理寄存器、线程栈等），里面有一个TLS数组(我的电脑上是32个元素）
用于存放线程私有变量。TLS数组的每个元素定义类似如下：

~~~ C++
struct pthread_key_data {
    uintptr_t seq;
    void* data;
};
~~~
上面提到过`pthread_key_create()`返回的TLS key其实是`__pthread_keys`数组的下标。
`pthread_setspecific()`根据TLS key定位TLS数组的一个元素，并设置其序列号seq和数据指针。
如果这个数组不够用，则会再动态分配一个二级数组（在我的电脑上是一个32x32的稀疏数组）。
线程退出时，`pthread_key_data`中的序列号用于判断该key是否仍在使用中（即与`__pthread_keys`中
的同一个下标对应的序列号是否相同），若是则调用TLS的析构函数。

## Boost的TLS实现

为了突破POSIX线程库对TLS变量个数的限制，`boost::thread_specific_ptr`只需要创建一个key，并设置该
key对应的TLS变量指向一个`map`，这样可以在该`map`中存放无数个（仅受内存限制）TLS变量。

另外`boost::thread_specific_ptr`的`reset()`可规避`pthread_setspecific()`忘记释放资源的坑。
不过`boost::thread_specific_ptr`的析构函数的执行时机也有与`pthread_key_delete()`一样的限制。

# 隐式TLS

相比于上面显示TLS需要动态调用API构造线程私有变量，隐式TLS更加方便，采用编程语言提供的关键字声明即可。
下面是采用C++11的关键字`thread_local`重写上面的程序。

~~~ C++
#include <thread>
#include <cassert>
#include <memory>

using namespace std;

int main()
{
    thread_local unique_ptr<int> tls_int_ptr;
    std::thread([] {
                    tls_int_ptr.reset(new int(3));
                    assert(*tls_int_ptr == 3);
                }).join();
    // 主线程的TLS与子线程是分开的，所以还是空指针。
    assert(tls_int_ptr == nullptr);
    return 0;
}
~~~

`thread_local`声明的变量在线程启动时构造，并在线程退出时析构。而且不像POSIX的TLS，没有数量限制。但是
其析构的时机没法动态控制。

## 为什么`thread_local`不能是非静态变量

根据C++11标准，`thread_local`变量必须是静态变量（名字空间变量、函数静态变量或者类的静态成员变量），不
能是类的非静态成员。例如，下面的代码是非法的:

~~~ C++
class C {
    int a;
    thread_local int b;
};
~~~
必须如下声明`b`是静态成员变量：

~~~ C++
class C {
    int a;
    thread_local static int b;
};
~~~
这有时候确实不方便，因为我们希望每个`C`对象都有自己的成员副本`b`，不过这可以用下面的方法规避这个问题：

~~~ C++
class C {
    int GetBPerThread()
    {
        return b_map[this];
    }

    int a;
    thread_local static std::map<void*, int> b_map;
};
~~~

既然可以规避，那为啥标准不直接支持呢？这是因为TLS变量的内存布局与类的非静态成员不兼容导致的。假设
`b`是一个非静态成员，那么其内存位置必然与`a`是相邻的。但是`a`是全局唯一的（对于某个C对象来讲它
是唯一的，不同的C对象有不同的`a`），如果`b`的位置与`a`相邻，则`b`也必然是唯一的，这就与其TLS变量
的身份相冲突了。

# 隐式TLS的实现

下面介绍的是ELF对TLS的实现，其它对象文件格式的支持也是类似的。

## ELF中的TLS段

代码中所有的全局变量都存储在`.data`（静态初始化变量）和`.bss`（未静态初始化的变量）这两个段，相应地，
TLS变量分别存储在`.tdata`（由`SHF_PROGBITS`+`SHF_TLS`标记）和`.tbss`（由`SHF_NOBITS`+
`SHF_TLS`标记）.

与`.data`不一样的是，运行时程序不会直接访问这些段。在程序启动后，动态链接器会对这两个段进行动态初始化
（如果有），之后这两个段不再变化，而是作为TLS的**初始化镜像**保存起来。每次启动一个线程时（包括启动
线程）都会分配TLS块将初始化镜像复制过来。所以每个线程启动时TLS都是相同的。

每个线程的TLS块都是运行时分配的，所以在链接时是不知道其地址的，要访问TLS变量必须借助动态链接器才能计算出
其地址。链接时只能知道TLS变量在TLS段中的偏移（内存中`.tbss`紧跟在`.tdata`后）。

## TLS的运行时分配

根据上面的介绍，要访问TLS变量需要确定两个信息：

1. 定义TLS变量的模块（可执行程序exec或动态共享库DSO）；
2. TLS变量在该模块的TLS段的偏移.

这两个信息在链接时是不知道的，需要动态链接器在运行时计算。然后再根据这两个信息查找这个TLS变量在当前线程
中的地址，这需要借助如下的数据结构来完成。

{% img /images/tls_data_structure.jpg Fig. 1 Thread-Local Storage data structures %}

$tp\_t$表示线程$t$的线程寄存器，指向线程控制块TCB，其在0偏移处指向动态线程向量$dtv_t$. DTV第一个元素
是一个generation counter，用于判断该DTV是否需要更新，下面会进一步解释；其它元素包含了所有
该线程用到的模块的TLS块的地址，用模块ID作为下标访问。有些模块的TLS块跟TCB放在一起，是程序启动时就分配
的（如exec及其依赖的DSO），称为*静态模型*；有些模块是程序运行中动态加载的（通过`dlopen()`动态加载），
TLS块在线程第一次访问时分配，称为*动态模型*。对于静态模型，在程序启动时动态链接器就可以确定其相对于$tp_t$
的偏移值，编译器生成代码时可以直接使用这些偏移值来访问。对于动态模型，由于TLS时延迟分配的，需要调用运行时
系统提供的`__tls_get_addr()`获取其地址。

### 程序启动

程序启动时，动态链接器搜集所有模块的TLS信息，包括以下字段：

- TLS初始化镜像的地址
- TLS初始化镜像的大小
- 模块的偏移值$tlsoffset$
- 是否使用静态模型

动态加载一个模块时也会加入这些信息。线程库将利用这些信息为新建的线程分配TLS块和DTV.  对于使用动态模型和动态
加载的模块，线程库会延迟分配TLS块，直到第一次访问。

### 延迟分配TLS

对于延迟分配的TLS，由于其偏移值在启动时未知，必须借助于`__tls_get_addr()`获取，定义类似如下：

~~~ C
struct tls_index {
    size_t module_id;
    size_t offset;
};

void* __tls_get_addr(struct tls_index* ti)
{
    // Get the DTV of current thread.
    dtv_t* dtv = GET_CURRENT_DTV();

    // Check if the DTV is stale, and if so, update it.
    if (dtv[0].counter != dl_tls_generation) {
        update_dtv();
    }

    // Get the TLS block. If not allocated yet, allocate now.
    char* tls_block = dtv[ti->module_id];
    if (tls_block == UNALLOCATED_TLS_BLOCK) {
        tls_block = dtv[ti->module_id] = allocate_tls(module_id);
    }

    return tls_block + ti->offset;
}
~~~
`module_id`是模块ID，由动态链接器在加载模块时分配，从1开始（exec的模块ID固定是1）。

当动态加载或卸载一个模块时，动态链接器维护的`dl_tls_generation`会加1，表示模块信息有了变化。由于每个线
程的DTV时延迟更新的，所以每个线程的`dtv[0]`也会维护自己的`generation counter`，用于在访问TLS时判断
是否需要更新DTV.

## TLS的访问模式

上面已经提到过TLS的两种访问模式：静态模式和动态模式，这一节针对不同场景细化为4种模式。对于每一种模式，
都要求动态链接器在程序启动或动态加载模块时立即处理TLS相关的重定位。TLS的重定位是为了确定模块ID和在TLS块
中的偏移，然后存储在`GOT`表中。

模块ID和偏移有时候需要运行时才能确定，有时候链接时即可知道，因此组合起来可分为4种访问模式。具体采用哪种模式
由编译器和链接器共同决定。

### General Dynamic

支持访问所有的TLS变量，包括exec和DSO定义的TLS；也支持延迟分配TLS.  这种模式下不需要链接时知道模块ID和
偏移值。程序启动时动态链接器通过重定向确定模块ID和TLS变量的偏移值，存储在`GOT`表中。在访问TLS时调用
`__tls_get_addr()`，传入这两个参数，获取TLS变量的地址。

### Local Dynamic

如果链接编辑器确定访问的TLS变量属于本模块（如文件作用域的TLS变量），则采用此模型。TLS变量的偏移值在链接时
即可确定，只需要调用`__tls_get_addr()`确定TLS块的地址即可。由于TLS块的地址可以在不同的本地TLS变量
访问时复用，所以相比于GD模型编译器可利用此模型生成有效的代码减少对`__tls_get_addr()`的调用次数。例如，在同一个函数
内访问不同的本地TLS变量，只需要调用一次`__tls_get_addr()`即可。

### Initial Exec

如果可以确定访问的TLS变量在程序启动时就已分配好，则采用此模型。TLS变量相对于线程寄存器的偏移量可在
程序启动时由动态链接器计算好存放在GOT表中。访问TLS变量相当于一次间接地址访问，不需要调用`__tls_get_addr()`.

### Local Exec

如果可以确定在exec中访问exec定义的TLS变量，则采用此模型。链接时即可知道TLS变量相对于线程寄存器的偏移量，
计算其地址相当于寄存器加上一个常量，因此访问TLS变量与访问局部变量没有区别。

# Per-thread resource allocation pattern

线程局部变量可以有效避免锁竞争，根据上面介绍的TLS实现来看，访问成本与普通变量也差不多，但是它完全防止了
线程之间的通信，很多时候不满足需求。所以需要在避免竞争与线程通信之间取得平衡，一种常用的模式是每个线
程分配自己的私有资源以满足大部分需求，只有在必要的时候才与其它线程通信。如下的程序实现了并发统计，在
线程结束后由主线程汇总打印。

~~~ C++
#include <iostream>
#include <thread>
#include <ctime>
#include <vector>

size_t kNumThreads = 10;

int main()
{
    uint32_t stats[kNumThreads];
    std::vector<std::thread> threads;
    for (size_t i = 0; i < kNumThreads; ++i) {
        threads.push_back(
                std::thread([&, i] {
                                time_t start = std::time(nullptr);
                                do_something();
                                time_t end = std::time(nullptr);
                                stats[i] = end-start;
                            }));
    }
    for (auto& t : threads) {
        t.join();
    }
    uint32_t total = 0;
    for (auto t : stats) {
        total += t;
    }
    std::cout << "total=" << total << std::endl;
    return 0;
}
~~~

主线程分配了一个数组，每个元素对应一个子线程，防止线程竞争。这样也方便了子线程将数据传回给主线程。
如果采用线程私有变量则比较麻烦，必须借助其它全局变量来通信。所以说为了避免竞争并不一定需要线程私有变量。

Linux内核很多统计即采用了这种模式，每个CPU核维护自己的统计数据，需要报告数据时则遍历所有核的数据合计
输出。著名的内存分配器tcmalloc也采用了这种模式来避免锁竞争，提高并发效率。它针对小内存的分配和释放做
了优化，小内存从本线程维护的内存池分配，大内存才从全局的内存池分配。同时为防止资源浪费，会在线程之间进行
内存资源调度，将本线程长期不用的内存转移给其它线程使用。所以说在大多数情况下避免了资源协调。

# Conclusion

如果线程之间完全不需要通信，推荐采用隐式TLS，使用非常方便，也不用担心资源释放的问题。但是如果要自己
灵活控制TLS资源的释放，则可以考虑采用显示TLS，推荐使用Boost库，可省不少心。

如果线程之间多少是需要通信的，不建议TLS，建议还是用共享数据结构加锁保护。为了提高性能可考虑使用原子变
量或者无锁数据结构。最终极的优化方法还是优化多线程的交互结构，尽量减少线程间通信。

# References

- [ELF Handling of Thread-Local Storage](https://uclibc.org/docs/tls.pdf)
- [Linkers and Libraries Guide](https://docs.oracle.com/cd/E19683-01/817-3677/chapter8-tbl-1/index.html)
- [TLS performance overhead and cost on GNU/Linux](http://david-grs.github.io/tls_performance_overhead_cost_linux/)
- [TCMalloc小记](http://blog.csdn.net/chosen0ne/article/details/9338591)
- [Boost Thread Local Storage](http://www.boost.org/doc/libs/1_65_1/doc/html/thread/thread_local_storage.html)
