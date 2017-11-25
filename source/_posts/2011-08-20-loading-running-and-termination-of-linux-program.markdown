---
layout: post
title: Linux程序的加载、运行和终止
date: 2011-08-20
comments: true
categories:
- operating system
---

## 简介

用户在编写程序时都要定义一个`main()`函数作为程序运行的入口。程序开始
执行时就从这个函数开始。当这个函数返回时就表明程序运行结束了。可是用户编写的
程序要能正确运行远不是这么简单。比如，我们不禁要问`main()`是由谁调用
的呢？当从`main()`返回后又运行到哪里去了呢？C++程序中定义的全局对象是
如何构造的呢？又是如何析构的呢？如果程序是动态链接的，它所依赖的共享库是如何
加载进内存的？更复杂的是，共享对象中的全局对象是如何构造的和析构的呢？要回答
这些问题，就不得不弄清程序加载、运行和终止的整个流程，从中也可以知道系统软件
（包括操作系统、动态链接器、链接编辑器和编译器）为了支持用户程序的正确运行做
了多么复杂的工作。

为了支持用户程序的正确运行需要解决以下几个重要问题：

*  加载用户程序以及它所依赖的所有共享对象；
*  对用户程序和共享对象进行**符号解析**和**重定位**；
*  向用户程序传递环境变量和命令行参数。
*  根据C++标准的规定，全局对象（包括用户程序和共享库中定义的）必须  在`main()`执行前初始化，并在程序结束时以相反的顺序析构。

为了理清这些问题，下面我们来分析Linux系统下程序的运行流程。

## 术语

### 程序头(Program Header) 

程序头在[gabi]的"Program Header"一节中定义，是ELF文件执行视图的重要部
分。它规定了ELF文件中的哪些部分段需要加载以及加载的地址以及是否需要动  态链
接器等信息。若需要动态链接器，程序头中的`PT_INTERP`指定了动态链接器
的路径

### 初始化代码和终止代码(Initialization and Termination code)

每个可执行文件和共享对象都有初始化代码和终止代码。初始化代码在用户程  序开
始执行前执行。所有的共享对象的初始化代码在可执行文件获得控制权之前执  行。
终止代码则在进程退出时执行，顺序与初始化代码执行的顺序相反。共享对象  的初
始化代码和终止代码由动态连接器负责执行。("Initialization and  Termination
Functions", [gabi])

### 加载时重定位（Load-time Relocation）和运行时重定位（Run-time Relocation）

加载时重定位指在动态链接器加载对象文件后就进行的重定位，而运行时重定位  是
指在用户程序已开始运行后在需要的情况下进行的重定位。PLT表的重定位就属于  运
行时重定位。在PLT表的帮助下，当第一次调用一个函数时进行重定位，以后再调  用
时就不用重定位了。若这个函数不被调用则不需要重定位，这可省去加载时重定  位
的时间。详见[abi386-4]的"Procedure Linkage Table"一节。

## 程序运行的基本流程

首先给出一个大致的流程。

1.  操作系统运行用户程序时将其映射到内存中；

2.  当它看到可执行文件中的`PT_INERP`时，操作系统 将`PT_INTERP`指定的动态链接
器映射进内存，并通过栈向其传递它所需要 的参数，并跳到动态链接器的入口处开始
执行；

3.  动态链接器开始自举（Bootstrap），对自己进行重定位，并开始构造符号表；

4.  自举完成后，动态链接器根据可执行文件.dynamic段中的DT_NEEDED元素开始加载
依赖的共享对象，并加入它的符号表。如果这个共享对象依赖其它的共享对象， 动态
链接器也会加载它们。当这个过程结束时，所有需要的共享对象都已加载进内 存，动
态链接器也具有了程序和所有共享库的符号表。

5.  这时，动态链接器重新遍历共享库，并进行加载时重定位（注意加载时重定位采
用依赖图的后序遍历顺序进行。也就是说如果A对象依赖B对象，则先处理B对象再 处理
A对象）。加载时重定位包括：
	*  对数据的引用，在`.rel.dyn`段中，需要初始化一个GOT（在`.got`中）项为一个全局符号的地址；
	*  对代码的引用（在`.rel.plt`段中），需要初始化一个GOT（在`.got.plt`)项为
    PLT表中第二条指令的地址（''Procedure Linkage Table'', [abi386-4]）。

6.  如果共享对象有初始化代码（在`.init`中，全局对象的初始化就是这样实
现的），动态链接器会执行它，并将终止代码（在`.fini`中，全局对象的 析
构就是这样实现的）记录下来以便退出时执行。动态链接器不会执行用户程序的 初始
化代码，它由用户程序的启动代码自己执行。这个过程完成后，所有的共享对 象都已
重定位并初始化，动态链接器跳到用户程序的入口处开始执行。注意，为了 能在程序
退出时让动态链接器有机会调用共享对象的终止代码，动态链接器会传递 一个终止函
数（用以调用共享对象的终止代码）给用户程序。

7.  用户程序开始执行。首先它注册动态链接器的终止函数和它自己的终止函数， 然
后调用用户程序的初始化代码，然后调用用户定义的`main()`函数。`main()`函数返
回后，以注册的相反顺序调用终止函数（也就是说先调用 用户程序的终止函数，再调
用动态链接器的终止函数），最后调用`_exit()`退出进程。

详见[Levine]第10章。

下面结合实际代码给出Linux下详细的运行流程。

### 程序的加载

程序的加载是通过执行`exec(3)`系统调用实现的。当在命令行上执行一个程序或在图
形界面系统中双击一个可执行文件时最终都是通过这个系统调用来执行程序的。执行这
个系统调用后，陷入操作系统内核，由操作系统负责加载该程序文件。在操作系统确认
相关参数后，然后通过内存映射方式加载进内存。若该ELF文件是动态链接的可执行文
件（程序头中存在`PT_INTERP`）需要动态连接器的支持，操作系统则将该动态连接器
映射进内存，并准备好相应的环境，将控制权转移给动态连接器。若ELF文件是静态链
接的，则操作系统准备好环境后直接转移到ELF文件的入口点开始执行。详细过程如下：

1.  执行`exec(3)`调用后陷入操作系统内核，检查参数，并判断可执行文件的类型。
因为Linux支持的可执行文件不止一种类型，加载不同类型的文件方法不一样。下面假
设文件类型为ELF。

2.  检查ELF文件格式的有效性，读入程序头（Program Header），并检查是否存在
`PT_INTERP`项。存在的话说明该文件是动态链接的可执行文件，需要动态连接器的支
持。

3.  根据ELF文件程序头的信息对ELF文件进行映射，通常包括两个段：代码段和数据段。

4.  初始化进程运行的堆栈环境，在栈中存储环境变量、命令行参数以及需要传给动态
连接器的一些附加参数（Auxiliary Vector）。（见[abi386-4]的图3-31）

5.  若ELF文件是静态链接的可执行文件，跳转到用户程序入口点（由其程序头定义）
开始执行；若ELF文件是动态链接的可执行文件，映射动态连接器，并跳转到动态连接
器的入口处开始执行。

另见[abi386-4]的第5节，[gabi]的"Dynamic Linking".

### 运行动态连接器

对于动态链接的可执行文件，还需要动态连接器为其加载可执行文件依赖的共享对象文
件并进行符号重定位才可以执行。动态连接器的位置存储在可执行文件程序头的
`PT_INTERP`元素中（见[gabi]"Program Header"一节）。动态连接器的运行过程如
下：

1.  动态连接器的入口是`_start`，在"glibc/sysdeps/i386/dl-machine.h"中的
`RTLD_START`宏中定义。它首先调用`_dl_start()`(glibc/elf/rtld.c)。

2.  `_dl_start()`首先对动态连接器自己进行重定位，最后调用`_dl_start_final()
`(glibc/elf/rtld.c)收集一些基本的运行时信息后调用`_dl_sysdep_start()`
(glibc/elf/dl-sysdep.c).

3.  `_dl_sysdep_start()`首先处理由操作系统建立的环境信息（Figure 3-31,
p.28, [abi386-4]），设置相关参数(`_dl_argc:` 命令行参数的个数，`_dl_argv:`
命令行参数数组，`_environ:` 环境数组，`_dl_auxv:` 传递给动态连接器的附加参数
数组)，在读入`_dl_auxv`数组存储的信息，最后调用`_dl_main()`
(glibc/elf/rtld.c)进行动态连接器的主要任务。

4.  `_dl_main()`非常长，主要工作是加载可执行文件依赖的所有共享对象，构造符号
表，并进行**加载时重定位**（有些重定位可以延迟到需要时再进行，称为**运行时重
定位**）。考虑到`R_386_COPY`（见[abi386-4]的78页）重定位类型，要特别加载时重
定位的顺序（因为需要copy relocation的变量的初始值可能是另一个全局变量的地址，
所以需要先重定位这个变量）。下面是摘自`_dl_main()`中的一段注释。

    > /\* Now we have all the objects loaded.  Relocate them all except for
the dynamic linker itself.  We do this in reverse order so that copy relocs
of earlier objects overwrite the data written by later objects.  We do not
re-relocate the dynamic linker itself in this loop because that could
result in the GOT entries for functions we call being changed, and that
would break us.  It is safe to relocate the dynamic linker out of order
because it has no copy relocs (we know that because it is
self-contained). \*/

    简单地说，先重定位一个对象文件所依赖的所有对象文件再重定位这个对象文件。
重定位完成后返回到`_dl_sysdep_start()`，然后返回到`_dl_start_final()`，然后
再返回到`_dl_start()`，继续返回到`_start`。

5.  `_start`调用动态连接器的初始化函数（以调用每个共享对象的初始化 代码），
并把动态连接器的终止函数（以调用每个共享对象的终止代码）地址存入`EDX`寄存器
以传给可执行文件，然后跳转到可执行文件的入口处开始执行。

动态连接器任务完成后将控制权转移给用户程序，此时用户程序才正是开始执行。

### 用户程序的执行

不管用户程序是静态的还是动态的可执行文件，它们的入口处都在`_start`
(glibc/sysdeps/i386/elf/Start.S)。它首先设置好一些寄存器后调用
`__libc_start_main()` (glibc/csu/libc-start.c)。`__libc_start_main()`主要进
行以下工作:

1.  调用`__cxa_atexit()` (glibc/stdlib/cxa_atexit.c)注册动态连接器通过`EDX`寄存器传过来的终止函数。
2.  调用`__cxa_atexit()`注册用户程序的终止函数
3.  调用用户程序的初始化函数
4.  调用用户提供的`main()`函数
5.  `main()`返回后调用`exit()` (glibc/stdlib/exit.c). `exit()`以注册的相反
顺序调用`atexit()` (glibc/stdlib/atexit.c)和`__cxa_atexit()`注册的函数，然后
调用`_exit()`结束进程。

## 参考文献

*  [abi386-4] _System V Application Binary Interface: Intel386  Architecture Processor Supplement_, Fourth Edition.
*  [gabi4] _System V Application Binary Interface_, 2001.
*  [Levine] John R. Levine, _[Linkers and Loaders](http://www.iecc.com/linker/)_.
*  [Glibc源代码](http://ftp.gnu.org/gnu/glibc/)
