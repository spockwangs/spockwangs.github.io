---
layout: post
title: 生成可执行的共享库
categories:
- programming language
---

Linux系统下共享对象文件和可执行文件的格式都是ELF格式，它们并没有什么本质 上
的区别。共享对象文件也是可以执行的。例如Linux下动态链接器ld-linux.so就 是共
享对象文件，它也可以像可执行文件一样执行；Glibc库函数也是一样。

共享对象文件要执行有2个问题要解决：

1.  共享对象文件的**加载地址**(loading address)是随机的（由操作系统决定恰当的
地址），不像可执行文件，每次都加载到相同的固定地址（在Linux下是0x8048000）。
因此，共享对象要能正确执行，必须编译成**位置无关代码**（Position Independent
Code, PIC）。这可以通过gcc的-fpic选项得到。
2.  共享对象也会用到其他的共享库函数（如C语言库函数），引用的外部符号都必须
被**解析和重定位**，而这些必须在运行时完成。这就需要动态链接器的帮助。所以，共
享对象必须编译成依赖动态链接器，也就是说共享对象文件要有**.interp段**。若没有
这个段，操作系统在执行时会认为这个文件不需要动态链接器的支持，因而会直接执行
这个文件，最后由于对外部符号的访问是错误的（对外部符号的引用没有正确定位）导
致segmentation fault。通常情况下，共享对象文件编译时是不会生成.interp段的，
只有可执行文件在使用了共享库的情况下才会生成这个段。下面我就来讲讲如何在共享
对象文件中生成这个段。

最好用一个简单的例子来讲解这2个问题如何解决。示例代码如下：
``` c
#include <stdio.h>   // for printf()
#include <stdlib.h>  // for exit()
void fun()
{
    printf("This is fun.\n");
    exit(0);
}
```
该代码保存在`fun.c`文件中。

第一个问题容易解决。要编译成PIC代码，只需要传递`gcc`选项`-fpic`即可：
```
$ gcc -fpic -shared -Wl,-e,fun -o fun.so fun.c
```
其中`-Wl,-e,fun`是通知链接器生成的对象文件的入口地址是`fun`。执行文件`fun.so`会得到如下结果：
```
$ ./fun.so
Segmentation fault (core dumped)
```
之所以这样是因为在`fun.c`中调用了C语言库函数`printf()`，但没有对这个外部符号
正确地重定位（由于没有.interp段，操作系统认为这个文件不需要动态链接库重定
位），因而引用了非法地址，所以产生了段错误。通过下面的命令可以查看`fun.so`文
件没有.interp段。
```
$ readelf -l ./fun.so | grep interp
$
```
结果没有输出，说明`fun.so`文件中确实不存在.interp段。

要生成.interp段，可以在某个源文件中加入下面一句：
``` c
const char __invoke_dynamic_linker__[] __attribute__ ((section (".interp"))) 
    = RUNTIME_LINKER;
```
这正是`glibc`的做法。这句代码在`glibc`的`elf/interp.c`
文件中。这也是为什么`glibc`库函数有.interp段的原因。RUNTIME_LINKER就
是动态链接库的名字，取决于目标机器。我们也可以用这种方式来解决这个问题，代码
如下：
``` c
#include <stdio.h>    // for printf()
#include <stdlib.h>   // for exit()
const char __invoke_dynamic_linker[] __attribute__ ((section (".interp"))) 
    = "/lib/ld-linux.so.2";
void fun()
{
    printf("This is fun./n");
    exit(0);
}
```
注意我用`/lib/ld-linux.so.2`替代了`RUNTIME_LINKER1`，因为它就是linux上的动态链接器，在`/lib`目录下。

用如下方式编译并执行：
```
$ gcc -fpic -shared -o fun.so -Wl,-e,fun fun.c
$ ./fun.so
```
终于大功告成。

## 参考文献

本文方法来自[这个帖子的](http://sourceware.org/ml/binutils/2008-06/msg00161.html)回帖。
