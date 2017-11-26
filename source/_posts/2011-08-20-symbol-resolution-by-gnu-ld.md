---
layout: post
title: Symbol resolution by GNU ld
date: 2011-08-20
comments: true
categories:
- computer system
- programming language
---

## Symbol Resolution

When a symbol appeared multiple times in object files being combined
asymbole resolution process is called by the link editor to determine
whichsymbol is taken. The resolution of two symbols with the same
namedepends on the symbol's attributes (its binding, defineness and
size). See page 67 of [<a href="#gabi41">Gabi4</a>].
<!--more-->

*  **Global:** Global symbols are those whose binding is
`STB_GLOBAL`. They are visible to all object files being
combined. 
*  **Weak:** Weak symbols are binding is `STB_WEAK`. They resemble
global symbols, but their definitions have lower precedence. It can be
further categorized into two kinds: undefined weak symbols and defined weak
symbols.
*  **Common:** Common symbols are those whose section index is
`STN_COMMON`. They are defined but have not been allocated. Its
final size are not determined until the symbol resolution process is
complete.
*  **Undefined:** Undefined symbols are those whose section index is
`STN_UNDEF`. 

The rules for symbol resolution is as follows.

1.  If a global symbol exists it can only appeared once in object files
being combined. Multipe definitions of global symbols with the same name
will cause an error. On the other hand, if a definition of a global symbol
eixists, the appearence of weak symbols and/or common symbols with the same
name will not cause an error. The link editor honors the global definition
and ignores the weak and/or common ones.

2.  Otherwise, if a common symbol exists, the appearence of weak symbols
with the same name will not cause an error. The link editor honors the
common definition and ignores the weak ones. If muliple common symbols with
the same name exists, the link editor honors the common definition with the
biggest size.

3.  Otherwise, multiple appearences of weak symbols with the same name do not cause an error.
    1.  If some of the weak symbols are defined (the section index is a
    positive integer), the link editor will honor the first found defined
    symbol and inogre the others.
	2.  Otherwise, if all the weak symbols are undefined, the symbol will
	be left as an undefined weak symbol in the output file no matter what
	type of output file is being generated. In addition, if a executable is
	being generated, all the reference to the symbol will be assigned a
	value of zero. In the case of dynamic shared object, during process
	execution, the dynamic linker searches for this symbol. If the dynamic
	linker does not find a match, it binds a reference to a address of zero
	instead of generating a fatal runtime relocation error.

## Testing Existence of Functionality

Undefined weak refenrenced symbols may provide a useful mechanism
fortesting the existence of functionality. For example, the following C
codefragment might have been used in the shared object`libfoo.so.1`:

    #pragma weak foo

    extern void foo(char *);

    void bar(char *path)
    {
        void (*fptr)(char *);
        if ((fptr = foo) != 0)
            (*fptr)(path);
    }

When application is built against `libfoo.so.1`, the link editor
willcomplete successfully regardless of whether a definition for the
symbol`foo` is found. If during execution of the application the
functionaddress tests nonzero, the function is called. However, if the
symboldeﬁnition is not found, the function address tests zero and so it is
notcalled. See page 45 of [<a href="symbols.html#sun04">Sun04</a>].

## Refenrences

*  [gabi4] [System V ABI Edition 4.1, 1997](http://www.sco.com/developers/devspecs/gabi41.pdf)
*  [sun04] Sun Microsystems, Inc., [_Linker and Libraries Guide_](http://download.oracle.com/docs/cd/E19683-01/817-3677/817-3677.pdf), 2004.
