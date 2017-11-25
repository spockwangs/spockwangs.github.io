---
layout: post
title: C++的new和delete运算符与内存分配函数和释放函数
categories:
- programming language
---

我曾经碰到下面一段代码：
``` c
int *p = new (sizeof(int));
```
编译通不过，提示说是`new`运算符语法错误。但若换成如下：
``` c
int *p = operator new (sizeof(int));
```
编译顺利通过。一时不得其解。前一句代码显然是调用new运算符；后一句看起来也像
是调用`new`运算符，毕竟其它运算符可以这样调用。

在仔细查看了C++标准后，我才发现这两句代码实际上是不一样的。前一句是调用`new`运
算符，但后一句是直接调用内存分配函数。本文就是要澄清这两者之间的区别。

`new`运算符有两步操作：

1.  调用适当的内存分配函数分配内存`opeator new`或`operator new[]`;
2.  调用适当的构造函数。

同样地，`delete`运算符也有两步操作：

1.  调用适当的析构函数；
2.  调用适当的内存释放函数`operator delete`或`operator delete[]`.

C++标准库在头文件`<new>`中提供了一些默认的内存分配函数和释放函数，包括会抛出
异常的和不抛出异常的版本（见[ISO C++ 1998]的18.4）。严格地来讲，`new`和
`delete`运算符是不能被重载的，重载的只是内存分配函数和释放函数。因此，用户可
以自定义内存分配函数和释放函数。注意，内存分配函数和释放函数必须是全局函数
（非静态的）或类的成员函数（3.7.3.1, [ISO C++ 1998])。

## 内存分配释放函数

内存分配函数的返回值类型为`void *`，第一个参数的类型为`size_t`，可以有多个参
数。内存分配函数分通常(usual)形式和放置(placement)形式。只有一个参数的分配函
数为通常形式，有多个参数的为放置形式。

内存释放函数的返回值类型为`void`，第一个参数的类型为`void *`，可以有多个参数。
内存释放函数也分通常形式和放置形式。对于全局的内存释放函数，只有一个参数的内
存释放函数为通常形式，有多个参数的为放置形式。对于类的成员内存释放函数，若声
明了一个单参数的内存释放函数，则它是通常形式，其它的都为放置形式；若没有声明
这样一个函数，但是声明了一个两个参数的内存释放函数且第二个参数类型为
`size_t`，则它可以作为通常形式的替代品，但它仍是放置形式。(clause 2 of
3.7.3.2, [ISO C++ 1998])

C++标准库提供了8个内存分配和释放函数，我们也可以用同样的函数签名定义这些函数以替代它们。(clause 2 of 17.4.3.4, [ISO C++ 1998])

## new和delete运算符及对应的内存分配和释放函数

对于每一个`new`运算符，编译器都要找到并调用相应的内存分配函数。若`new`前面有
全局作用域解析符或要分配的类型不是类类型，则仅在全局作用域查找内存分配函数；
否则，先在要分配的类型的类作用域里查找，若找不到再在全局作用域里查找。
(clause 9 of 5.3.4, [ISO C++ 1998])

非放置形式的`new`表达式调用通常形式的内存分配函数，放置形式的`new`表达式调用
放置形式的内存分配函数。对于非放置形式的`new`表达式，调用的内存分配函数的第
一个参数是要分配内存的大小。若是放置形式(placement forms)的`new`表达式，
`new`运算符的放置部分将作为内存分配函数的第二个及后续参数。
``` c++
T *p = new T;        // call operator new(sizeof(T)) or T::operator new(sizeof(T))
T *p = new T[5];     // call operator new[](sizeof(T)+delta) or T::operator new[](sizeof(T)+delta)
T *p = new (2, f) T; // call operator new(sizeof(T), 2, f) or T::operator new(sizeof(T), 2, f)
```
对于每一个`delete`表达式，编译器都要找到并调用相应的内存释放函数。若
`delete`前面有全局作用域解析符(`::`)或要释放的类型不是类类型，则仅在全局作用
域查找内存释放函数；否则，先在要释放的类型（注意是其动态类型，而不是静态类型）
的类作用域里查找，若找不到再在全局作用域里查找。(clause 9 of 5.3.4, [ISO
C++ 1998])

非放置形式的`delete`表达式调用非放置形式的内存释放函数。`delete`表达式没有放
置形式。`delete`运算符调用的内存释放函数传给内存释放函数的第一个参数是将要释
放的内存指针。若调用的内存释放函数是类的成员函数且是两参数风格（有两个参数，
第一个参数是`void *`，第二个参数类型是`size_t`，没有声明单参数的内存释放函数，
所以这个函数是通常形式），则内存的大小作为第二个参数(clause 5 of 12.5, [ISO
C++ 1998])。
``` c++
// call operator delete(p) or T::operator delete(p) or 
// T::operator delete(p, sizeof(*p))
delete p;    

// call operator delete[](p) or T::operator delete[](p) or
// T::operator delete[](p, sizeof(*p)*s+delta)
delete[] p;  
```
若释放的对象不是数组，则`delete`的操作数的动态类型与静态类型要么一致，要么其
析构函数是虚拟的；若释放的对象是数组，则`delete[]`操作数的动态类型与静态类型必
须一致。（clause 3 of 5.3.5, [ISO C++ 1998])
    
内存释放函数虽然是静态的，不能动态绑定，但是若类T的析构函数是虚拟的则编译器
将根据指针`T *p`的动态类型来决定调用合适的内存释放函数(clause 7 of 12.5, [ISO
C++ 1998])。例如：
    
    struct B {
        virtual ~B();
        void operator delete(void *);
    };
    struct D : B {
        void operator delete(void *);
    };
    void f()
    {
        B *bp = new D;
        delete bp;         // use D::operator delete(void *), not B::operator delete(void *)
    }

若`new`运算符在初始化对象时抛出异常并且存在与`operator new`匹配的内存释放函
数，那么这个内存释放函数将被调用以释放内存，异常在`new`表达式的环境下继续传
播。非放置形式的内存释放函数与非放置形式的内存分配函数是匹配的；放置形式的内
存释放函数与放置形式的内存分配函数，若它们的第二个及后续参数一致，则是匹配的。
在调用内存释放函数时，内存分配函数的返回值传给内存释放函数的第一个参数；若调
用的是放置形式的内存释放函数，则将调用放置形式的内存分配函数时传递的额外参数
传递给内存释放函数。
    
    class A {
    public:
        A()
        {
            throw runtime_error("error");
        }

        // 放置形式的内存释放函数，也可作为通常形式的替代品，因为单参数的内存释放函数不存在。
        void operator delete(void *p, size_t s) 
        {
            cout << "A::operator delete/n";
        }
    };
    
    int main()
    {
        try {
            A *p = new A;         // 抛出异常！调用A::operator delete()
        } catch (const exception& e) {
            cout << e.what() << endl;
        }
        return 0;
    }

由于A的构造函数会抛出异常，而且A定义了一个与`operator new(size_t)`匹配的内存释放函数，因此编译器将调用那个内存释放函数然后抛出异常。

## References
* [ISO C++ 1998] ISO C++ Standard, 1998.
