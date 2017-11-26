---
layout: post
title: Typecast in C++
date: 2011-8-22
comments: true
categories:
- programming language
---

There are four types of type-cast in C++, which are preferred than the C-style cast.
<!--more-->
## static_cast

`static_cast` has following uses.

1.  Cast between arithmetic types, such as casting from `int` to `char`.
2.  Cast a value of integral type to an enumeration type.  In C++ a value of integral type can not be implicitly converted to an enumeration type, which is different from C.  The integral value must be in the range of the enumeration values.  (See clause 7, 5.2.9 of [C++ standard 98])
3.  Cast a value of type "pointer to `void`" to a pointer to object type.  You must ensure that the original pointer is really pointing to the destination object type.
4.  Cast any type to `void`.
5.  Down-cast in case of pointer.  Convert a value of type "pointer to `B`" to a value of type "pointer to `D`", where `B` is a non-virtual base class of `D`.  You must make sure that the original pointer is really pointing to the sub-object of `D`. (See clause 8, 5.2.9 of [C++ standard 98])
6.  Dow-cast in case of reference.  Convert a value of type "reference to `B`" to a value of type "reference to `D`", where `B` is a non-virtual base class of `D`.  You must make sure that the original reference is really referencing the sub-object of `D`. (See clause 5, 5.2.9 of [C++ standard 98])

Note that you cannot use `static_cast` to convert a value of type "pointer
to `int`" to a pointer to `char`.  This is a frequent misuse of
`static_cast`.  You have to use `reinterpret_cast` to do that.

All the conversions that `static_cast` does is done at compile-time, so
what it can do is limited.  Due to lacking run-time info it just does basic
type checking and can not make sure what you do is valid.  For example you
may convert a pointer to a wrong type but `static_cast` can not figure it
out.

## dynamic_cast

Dynamic cast is usually used to convert a pointer to a base class to a pointer to a derived class.

Consider the following dynamic cast expression:

    dynamic_cast<T>(v)

we have following situations

1.  If `v` is a null pointer the result is the null pointer in `T` type. (See clause 4, 5.2.7 of [C++ standard 98])
2.  Up-cast in case of pointer.  If `v` is a pointer to `D` and `T` is "pointer to `B`" where `B` is a base class of `D`, the result is a pointer to the unique `B` sub-object of `D`. (See clause 5, 5.2.7 of [C++ standard 98])
3.  Up-cast in case of reference.  If `v` has type `D` and `T` is "reference to `B`" where `B` is a base class of `D`, the result is a reference to the unique `B` sub-object of `D`. (See clause 5, 5.2.7 of [C++ standard 98])
4.  Down-cast and cross-cast.  If `T` is "pointer to `void`", the result is a pointer to the most derived object pointed to by `v`.  Otherwise, a run-time check is applied to see if the conversion is valid.  The value of a failed cast to pointer type is a null pointer.  A failed cast to reference type throws `bad_cast`.  (See clause 8, 5.2.7 of [C++ standard 98])

## reinterpret_cast

This is the most dangerous type cast of all because the compiler and
run-time will not do any check.  Just like what the name means it is a
re-interpretation of the bit pattern of the original object.  The C++
standard does not guarantee that the bit pattern be not modified (See
clause 3, 5.2.10 of [C++ standard 98]).  Use it judiciously.

1.  Conversion between integral type and pointer type. 
2.  Conversion between different types of pointers. 
3.  Conversion between references which refer to different types. 

## const_cast

It can only be used to cast away cv-qualifiers and the original type must
be a pointer or reference.  If the result of a `const_cast` that casts away
const-qualifier is pointing to or referring to a object which is declared
with const-qualifier, you can not use it to modify the underlying object
(see clause 7, 5.2.11 and clause 7, 7.1.5.1 of [C++ standard 98]).

## References

*  [C++ standard 98] _The C++ standard_, 1998.
