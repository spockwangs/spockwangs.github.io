---
layout: post
title: Java Gotchas
categories:
- programming language
---

## Primitive types, Operators, and Conversions

`==` tests object identity (whether the variable references the
same object), not content identity (consider `equals()`
instead).
<!--more-->

Assignment semantics in Java: copy reference for object variables(cf clone), and copy value for basic types.

`>>` do arithmetic right shift (shift in 0 for unsigned types
and 1 for signed types), while `>>>` do logical right shift
(always shift in 0 regardless of sign).

Java command line args (`String[]` args in main) do not take
into account the name of the program, i.e. "java prog arg1 arg2" has 2
arguments (arg1 and arg2), it does not count prog.

In Java all number types are signed except char.

No conversions between boolean and any other type are applied.

All narrower than int types are converted to int before doing arithmetic,
so byte b = 0xFF; b >>> 1 may not be what you want.

Watch out for autobox and unbox, i.e. conversions between basic types and
their wrapper types.  Sometimes it does not work.  For example, what does
this print?

    public class box {
        public static void main(String[] args)  throws IOException  {
            cmp(new Integer(42), new Integer(42));
        }

        static void cmp(Integer first, Integer second)  {
            if (first < second)
                System.out.printf("%d < %d\n", first, second);
            else if (first == second)
                System.out.printf("%d == %d\n", first, second);
            else if (first > second)
                System.out.printf("%d > %d\n", first, second);
        }
    }

## Class

You can not override the same method of base class with a weaker access
privilege.  For example, you can not override a public method with a
protected method.  This is different from C++.

Due to reference assignment semantics in Java, there is no built-in
mechanisms to do copy (to get a duplicate object exactly the same as the
original one).  There are two methods to achieve this: override the clone
method publicly or provide a copy constructor (or a static factory).  See
item 10 of "Effective Java".

An inner class (non-static class declared in another class, which is called
its enclosing class) can access any member of its enclosing class.  And its
enclosing class can also access any member of its inner class.

In Java a method of a class can overload with the method of its super class
with the same name.  To the contrary in C++ this situation is not called
overloading because the two methods are not declared in the same scope.
