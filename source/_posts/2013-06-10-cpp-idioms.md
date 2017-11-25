---
layout: post
title: C++ Idioms
categories:
- programming practice
status: publish
type: post
published: true
meta:
  _edit_last: '1'
  _edit_lock: '1353842551'
  _pingme: '1'
  _encloseme: '1'
  _wp_old_slug: ''
---

## Pimpl idiom

**Declare the constructor and destructor in the header file and define them
  in the source file when using Pimpl idiom, even if they are empty.**

Consider the following code.

    // pimpl.h
    class Impl;    // forward declaration
    class Pimpl {
    public:
       Pimpl();
       
       // The destructor is not declared, so the compiler will generate one.

    private:
        boost::scoped_ptr<Impl> m_impl;
    };

    // pimpl.cc
    class Impl {
         // ...
    };

    Pimpl::Pimpl()
        : m_impl(new Impl)
    {}

If you do not declare the destructor the compiler will generate one in
every translation unit that includes `impl.h`, which will call
the destructor of member variables, that is, the destructor
of `m_impl` which requires the complete definition
of `Impl`. But the whole purpose of Pimpl idiom is to hide the
definition of `Impl`. To solve this problem you should declare
the destructor in the header file to prevent the compiler from generating
one, and define it in the source file. Then only the `impl.cc` requires the
complete definition of `Impl`. Other translation units just
call `Pimpl`'s destructor as an external function, so they don't
need to generate it.

## Prefer anonymous namespace functions to class static functions.

When you are implementing a class's interface, you may need some helper
functions which has no relation with the private (member or static)
data. You can either declare the helper functions as the private static
functions in the class header file, or as the anonymous namespace functions
in the class source file.

Since it does not need to access the private data, the helper function
should be kept out of the class definition to make them loosely
coupled. And it is the implementation's details, which we don't want the
client of the class see it. So if we can put it outside the header file, we
should do it. If we put it in the header file, every time the
implementation changes the client using this class has to be
re-compiled. So we should put the implementation details outside of the
class header file as much as possible.

## Pass read-only arguments by const reference, and read-write arguments by value

If the argument is intended to be read only in the function body, and

1. if its size is bigger than the pointer type pass it by const-reference,
2. otherwise, pass it by value

to avoid unnecessary copy.  That means values of class objec type should be
passed by const-reference, and values of the small types like the basic
types should be passed by value.  If the smaller ones were passed by const
reference another pointer indirection cost would have been incurred in
addition to the pointer copying cost when reading their values.

If the argument is intended to be modifed in the function body, it is
recommended to pass it by value instead of passing it by reference and
making a copy in the function body.

Consider the following code.

    std::vector<std::string> 
    sorted(std::vector<std::string> names)
    {
        std::sort(names);
        return names;
    }
 
    // names is an lvalue; a copy is required so we don't modify names
    std::vector<std::string> sorted_names1 = sorted( names );
 
    // get_names() is an rvalue expression; we can omit the copy!
    std::vector<std::string> sorted_names2 = sorted( get_names() );

If the argument passed is an lvalue a copy is required.  But if the
argument is an rvalue the copy can be optimized out by the compiler.

See ["Want Speed? Pass by
Value"](http://cpp-next.com/archive/2009/08/want-speed-pass-by-value/) for
more details.

## Don't worry about returnning by value

Many modern C++ compilers provide the Return Value Optimization to elide
the copy when returnning value. 

Consider the following code.

    std::string getName()
    {
        std::string name;
        
        // do stuff to `name`
        
        return name;
    }

    std::string s = getName();

Actually the signature of `getName()` is translated by the compiler to

    void getName(std::string *p)
    {
        // do stuff to `*p`
        // not necessary to copy when returnning
    }

The caller allocates space for the return value on the stack, and pass the
address of the space to the callee. Then the callee construct the return
value directly in that space, which elimiates a copy from inside to
outside. So we should no worry about the copy cost when returnning a big
object from a function, and the signature is more satisfactory.
