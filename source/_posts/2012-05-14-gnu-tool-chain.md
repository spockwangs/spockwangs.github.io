---
layout: post
title: GNU Tool Chain
comments: true
categories:
- programming language
- operating system
---

## Create Static Libraries

To compile source files

    $ gcc -c file1.c file2.c ...

To create archives

    $ ar r lib<sth>.a file1.o file2.o ...

See `ar(1)`.
<!--more-->

## Link with Static Libraries

    $ gcc -static -o outputfile file1.c file2.c -L<search path> -l<archive name>

Use option `-static` to avoid linking against shared libraries.  Use option
`-l` to specify which libraries are required and `-L` to specify where to find
these libraries.  If the required libraries are in system paths (i.e. `/lib`
and `/usr/lib`) `-L` is not required because the linker will search the system
path by default.  The paths are searched in the following order:

1. The paths specified by `-L` and by `SEARCH_DIR` command in the link script (not the default link
   script, but the one specified by `-T`) in the command line from left to right;
2. The paths specified by `SEARCH_DIR` command in the default link script if it is not yet replaced
   by the `-T` option.

The contents of the default link script can be printed by using `--verbose` option to `ld`.

See `-L`, `-T` and `-dT` in `ld(1)`.

## How Linkers Use Static Libraries to Resolve References

See Section 7.6.3 of Computer Systems: A Programmer's Perspective.

Watch out for mutual dependences and cyclic dependences of the static libraries.  See option `--start-group` of `ld(1)`.

## Create Shared Libraries

    $ gcc -shared -fpic -o somename.so file1.c file2.c ... dependent_shared_libraries

The dependent shared libraries will be stored in the .dynamic section (its
type is `DT_NEED` and its value is whatever you put on the command line
(absolute or relative path), you can check it using `readelf -d`) of the
generated shared object.  It tells the dynamic linker which libraries are
needed by this shared object.

## Link with shared libraries

    $ gcc -o outputfile file1.c file2.c ... -L<search path> -l<library name> ... -Wl,-rpath,<path> ...

Like linking with static objects, when linking with shared objects the
linker needs to know which objects are required and where to find them.  It
uses the same mechanisms as linking with static objects to specify these
info.  See above "Link with Static Libraries".

If the shared objects on the command line require other shared objects (you
can check this by see the entry type `DT_NEED` of `.dynamic` section), the
linker editor needs to know where to find the required shared objects.
This info can be specified with the option `-rpath-link` or
`-rpath` of ld or by other ways.  See `ld(1)` about options
`-rpath-link` and `-rpath`.

For example:

    $ gcc main.c -L. -lfun -Wl,-rpath,pos

where `main.c` depends libfun.so which depends on some shared objects
resides in `pos`.  Option `-Wl,` is a way for gcc to pass options to `ld`.

There are other ways to specify the link-time shared objects search path
other than the above ways and they are searched in the following order:

1. -rpath-link
2. -rpath
3. Environment variable `LD_LIBRARY_PATH`
4. system paths `/lib` and `/usr/lib`
5. the directories specified in `/etc/ld.so.conf`

This is incomplete.  See option `-rpath-link` of `ld(1)` for more details.

## Runtime shared objects search path

When a executive which depends on some shared objects runs the dynamic
linker needs to load those shared objects.  So it needs to know two things:
which shared objects and where to find them.

The required shared objects are stored in the `DT_NEED` entry (shown as
"(NEEDED)" in output of `readelf -d`, may not exist if it does not depend
on anything) of `.dynamic` section in the object, which is specified on the
command line when generating this object file.

If the value of `DT_NEED` entry is a path (absolute or relative, i.e. has a
slash) the dynamic linker will try to find it there, and if not found the
executable can not run.  If it is just a file name the dynamic linker will
search for it in a series of paths in the following order:

1. The paths (separated by colons) in the entry `DT_RPATH` of `.dynamic`
   section in the executable (these paths are specified by the option `-rpath`
   of `ld` or `-Wl,-rpath` of gcc);
2. The paths in the environment variable `LD_LIBRARY_PATH` (separated
   by colons);
3. The paths specified in `/etc/ld.so.conf` (actually `/etc/ld.so.cache`,
   so if you edit `/etc/ld.so.conf` you should run `ldconfig` to update the
   cache and the shared objects should be named like libxxx.so);
4. `/lib`
5. `/usr/lib`

Note: all the paths mentioned above is relative to the current working
directory if it is a relative path.

You can see the searching process by setting the environment variable
`LD_DEBUG` to `libs` when running the executable.  See `ld.so(8)`.

## Loading of dependent shared objects and symbol resolving

Often the executive depends on some shared objects, which depends on some
other objects, which depends on some others, and so on.  So the depend
relation is like a graph with the executive being the root.  The Linux
dynamic linker is responsible for loading the dependent shared objects and
loads them in breadth first search (BFS) order.  The dynamic liner has a
global symbol table which includes all symbols it knows so far.  The global
symbol table is used to resolve symbols in the executive when running.  It
merges the symbol table of the shared object with global symbol table when
loaded.  If the symbol table has some symbol with the same name as a symbol
in the global symbol table, it will be ignored.  This will lead to global
symbol interposition.  So if we want to avoid this problem we must make
sure the global variables and functions in our programs have unique names.

The phase of symbol resolving begins after all dependent shared objects are
loaded and global symbol table is constructed.

The procedure of loading and symbol resolving can be checked using the env
variable `LD_DEBUG`.  See `ld.so(8)`.

## References

* `ld(1)` options `-shared`, `-L`, `-l`, `-rpath`, `-rpath-link`
* [`ld.so(8)`](http://man7.org/linux/man-pages/man8/ld.so.8.html)
* gcc(1) options `-shared`, `-static`, and `-Wl,`
* See [here](http://refspecs.linuxbase.org/elf/gabi4+/ch5.dynamic.html#shobj_dependencies) for `DT_RPATH` and `DT_RUNPATH`.
* 程序员的自我修养，Sections 7.6, 8.4 and 8.5
* Computer System: A Programmer's Perspective, Chapter 7.
* See [here](https://wiki.debian.org/Multiarch/LibraryPathOverview) for multi-arch support on toolchain implications.
