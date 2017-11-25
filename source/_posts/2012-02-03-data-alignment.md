---
layout: post
title: 数据对齐
categories:
- computer system
---

## 为什么要数据对齐？

所谓数据对齐是指访问数据的地址要满足一定的条件，能被这个数据的长度所整除。例
如，1字节数据已经是对齐的，2字节的数据的地址要被2整除，4字节的数据地址要
被4整除。

但为什么要数据对齐呢？简单地说，数据对齐是为了读取数据的效率。假如说每一次读
取数据时都是一个字节一个字节读取，那就不需要对齐了，这跟读一个字节没有什么区
别，就是多读几次。但是这样读取数据效率不高。为了提高读取数据的带宽，现代存储
系统都采用许多并行的存储芯片来提高读取效率。以自然字长为4个字节的机器为例。
```
memory chip     0       1       2       3
offset

  0             0       1       2       3
  1             4       5       6       7
  2             8       9      10      11
  N            4N    4N+1    4N+2    4N+3
```
如上图所示，地址0-3的4字节数据都存储在4个不同的芯片上。CPU的bits 0-7连上芯片
0，bits 8-15连上芯片1，bits 16-23连上芯片2，bits 24-31连上芯片4。

如果从地址0读取4字节数据，4个存储芯片都从各自的地址0读出数据，而且位置也是正
确的。这4个字节的读取可以并行地进行，因此带宽是一次读取一个字节的4倍。

假如要从地址1读取4字节数据会发生什么呢？首先，读取的数据位置不正确。从芯片1
读取的数据应该放在bits 0-7，它却对应CPU的bits 8-15。从芯片2，3，0读取的数据
也是一样。这并不是什么大问题，CPU可以把读入的数据循环左移8比特来把它们放置在
正确的位置上。问题在于，这一次发给4个芯片的地址不一样，给芯片1，2和3的地址是
0，给芯片0的地址是1。也就是说给每一个芯片的地址都不一样，这就需要多根地址总
线。32位的CPU就需要4个地址总线，每个地址总线需要用到CPU的至少32个引脚，也就
是必须要100多根引脚。而通常32位的CPU也就400多个引脚。这当然会增加CPU与内存接
口的复杂性（也可能会有其它的方案解决这个问题，但最终都会使接口变得更复杂，还
可能会降低效率）。但是增加这个复杂性是不值得。由于大多数数据访问都可以做成对
齐的（编译器通常会使数据对齐），因此为了使极少数的不对奇访问速度更快而使CPU
与内存的接口变的更复杂，是不划算的。

在这样的设计下，访问没有对齐的数据需要2次访问。比如，访问地址1, 2, 3, 4的4字
节数据，先读取地址0-3的数据，再读取地址4-7的数据，再把地址1-5的数据取出来放
到目的寄存器中。相应地，这增加了CPU内部的复杂性。这就解释了为什么不对齐的数
据要多次访问内存，速度不如访问对齐的数据快。而且也解释了为什么原子类型的数据
必须要对齐。有的CPU，例如Motorala生产的一些CPU，还有IBM的PowerPC，为了消除这
个复杂性，干脆禁止访问不对奇地址（若访问不对奇地址时将产生异常）。

总结一下为什么需要数据对齐。为了增加CPU读取数据的带宽，内存系统通常都采用并
行结构使得可以并行传输数据。这样的并行结构使得访问对齐的数据速度快，但是若要
使访问不对奇的数据也一样快会使CPU与内存系统的接口变得更复杂，而这是划不来的。
经过权衡之后，最终的结果是：访问对齐的数据速度快，访问不对奇的数据速度慢（需
要2次访问）或干脆禁止访问不对奇数据。

关于访问对齐数据和不对齐数据的速度差异请看[这里](http://www.ibm.com/developerworks/library/pa-dalign/)。关于为什么使访问不对
齐数据需要多次访问请看[这里](http://stackoverflow.com/q/3903164/471846)。关于哪些CPU禁止访问不对奇数据以及在这些CPU上如何解
决访问不对齐数据的问题请看[这里](http://en.wikipedia.org/wiki/Data_structure_alignment).

## Visual C++的结构体对齐规则

MSDN对Visual C++的结构体对齐规则有[简单介绍](http://msdn.microsoft.com/zh-cn/library/hx1b6kkd.aspx)，现摘录如下：

> Structure members are stored sequentially in the order in which they are
> declared: the first member has the lowest memory address and the last
> member the highest.
>
> Every data object has an alignment-requirement. The alignment-requirement
> for all data except structures, unions, and arrays is either the size of
> the object or the current packing size (specified with either /Zp or the
> pack pragma, whichever is less). For structures, unions, and arrays, the
> alignment-requirement is the largest alignment-requirement of its
> members. Every object is allocated an offset so that
>
>     offset %  alignment-requirement == 0
>
> Adjacent bit fields are packed into the same 1-, 2-, or 4-byte allocation
> unit if the integral types are the same size and if the next bit field
> fits into the current allocation unit without crossing the boundary
> imposed by the common alignment requirements of the bit fields.

每一个数据对象都有一个对齐限制，通常称为<strong>对齐模数</strong>。对于基本
类型（不包括结构体，联合体和数组）不同的数据类型的对齐模数通常等于这个数据类
型的大小，比如`char`的对齐模数为1，`short`的对齐模数为2，`int`的对齐模数为4，
`float`的对齐模数是4，`double`的对齐模数为8。当然，这个对齐模数每一个编译器
可能有差异，并且可以设置。在Visual C++中可以通过<tt>/Zp</tt>选项或
`#pragma`设置。综合起来，基本类型的对齐模数是这个类型的大小和设置的对齐限制
的较小者。对于结构体，联合体和数组，对齐模数是它的成员的对齐模数的最大值。

现举几个例子说明一下。（下面例子都在Visual C++ 2008 Express Edition上测试
过。）
``` c
struct A {
    char c;
    int i;
};
```
`c`的对齐模数是1，`i`的对齐模数是4，因此结构体`A`的对齐模数为4。`c`本身占用
1个字节，由于`i`的对齐模数为4，因此`c`后面填充3个字节以使`i`的起始地址对齐。
这样加起来是8个字节，`A`已经对齐了, 后面不用再填充了，因此`A`占用8个字节。

另一个例子。
``` c
struct A {
    char c;
    int i;
    char d;
};
```
跟上面一样，`c`的对齐模数是1，`i`的对齐模数是4，
`d`的对齐模数是1，因此结构体`A`的对齐模数为4。
`c`占用1字节，为使`i`对齐，`c`后面要填充3个
字节，因此`c`和`i`占用8个字节。`d`占用1个字节，
这样一共占用9个字节，但是`A`的对齐模数是4，因此`d`后面
要填充4字节。因此`A`占用12字节。

对于比特域来讲，上面的规则要稍稍修改一下。对于连续的声明类型大小相同的比特域
来讲，如果它们申请的宽度加起来不超过它们声明的类型大小，那么它们可以压缩在一
个分配单元中。对于声明类型大小不同的比特与来讲，它们分配在不同的分配单元中。

    struct A {
        char a : 3;
        char b : 5;
    };

`a`和`b`的类型长度都为1字节，因此可以考虑合并。它们加起来为8比特，
可以放在一个字节中，由于`A`的对齐模数为1，因此不需在后面再填充数
据，所以`A`的大小为1字节。

另一个例子.

    struct A {
        char a : 4;
        char b : 5;
    }

`a`和`b`的类型长度都为1字节，因此可以考虑合并，但跟上
例不同的是，它们无法放入一个字节中（加起来为9比特），因此它们要放在各自的分
配单元中，`a`和`b`之间有4比特的空隙。`A`的
大小为2字节。

再考虑如下例子.

    struct A {
        char a : 3;
        int b : 2;
        char c : 2;
    };

`a`和`b`的类型长度不相等（`a`的为1字节，
`b`的为4字节），因此不能合并，同样`b`和`c`
也不能合并。因此，`a`单独占用1字节，后面填充3字节以使
`b`对齐，`b`占用4字节，`c`占用1字节。由于
`A`的对齐模数是4字节，所以`c`后面要填充3字节，
`A`的大小为12字节。

下面再考虑匿名比特域的例子。匿名比特域主要目的是在两个比特域之间加入一段空隙。

    struct A {
        char a : 2;
        char   : 2;
        char b : 2;
    };

匿名比特域对对齐规则没有影响，它唯一的不同是无法引用这个域。由于这3个域的类
型大小是一样的，且可以放在一个字节之中，因此可以合并，`A`的大小为
1字节。

下面再考虑匿名比特域宽度为0的，它的作用是使它后面的比特域对齐到下个比特域的对齐模数上，也就是说，不允许它跟上一个比特域合并。

    struct A {
        char a : 2;
        char   : 0;
        char b : 2;
    };

尽管这三个比特域的类型一样，但是由于`a`和`b`之间有一个
宽度为0的匿名比特域，因此 它们不能合并。`a`和`b`个占用
1个字节，所以`A`的大小为2字节。

最后，对于具有普通成员和比特域的结构体，它们的对齐规则跟上面的一样。注意，普
通成员与比特域不能合并，只有连续的比特域之间可以考虑合并，每一个成员都要对
齐到自己的对齐模数上。

## 如何实现数据对齐

通过调用`malloc()`或者`new`运算符动态分配的内存已经是对任何类型都对齐了（见[这
里
](http://stackoverflow.com/questions/8752546/how-does-malloc-understand-alignment/18479609#18479609)
）。如果自己动手写一个内存分配器也要注意分配对齐的内存。通常编译器会提供工具来帮助对齐，例如，GCC提
供了[`__alignof__()`](https://gcc.gnu.org/onlinedocs/gcc/Alignment.html)来获得数据结构的对齐要求。
C++11也提供了类似的运算符[`alignof`](http://en.cppreference.com/w/cpp/language/alignof)。

在静态存储区或者栈上定义的对象由编译器分配内存并保证对齐要求。如果是用户自己在静态存储区或者栈上分配
内存可以使用编译器或者编程语言提供的工具实现对齐，如GCC提供
[`aligned`](https://gcc.gnu.org/onlinedocs/gcc/Common-Variable-Attributes.html#Common-Variable-Attributes)
属性实现对齐：

    int a __attribute ((aligned(__alignof(std::string)));

表示`a`的对齐要求与`std::string`一样。C++11提供了运算符
[`alignas`](http://en.cppreference.com/w/cpp/language/alignas)，上述的例子改写如下：

    alignas(alignof(std::string)) int a;
