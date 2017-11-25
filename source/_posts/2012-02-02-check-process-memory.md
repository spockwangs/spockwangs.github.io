---
layout: post
title: 查看进程的内存使用情况
categories:
- computer system
- operating system
---

## 用ps命令查看进程的内存

`ps`命令是Linux下常见的查看进程状况的程序，它有几个字段可以用来查看进程内存使用情况：sz，rss，vsz。分别说明如下：

*  sz：进程映像所占用的物理页面数量，也就是以物理页面为单位表示的虚拟内存大小；
*  rss：进程当前所占用的物理内存大小，单位为kB；
*  vsz：进程的虚拟内存大小，单位为kB，它等于sz乘于物理页面大小（x86平台通常为4kB）。

假如我要查看程序`a.out`的内存使用情况，操作如下：
```
$ ./a.out &
[1] 10069
$ ps -O sz,rsz,vsz
PID    SZ   RSS    VSZ S TTY          TIME COMMAND
 6793  1545  3648   6180 S pts/2    00:00:00 /bin/bash
10069   404   304   1616 S pts/2    00:00:00 ./a.out
10070   626   876   2504 R pts/2    00:00:00 ps -O sz,rss,vsz
```
上面`ps`命令的输出的第3行就是`./a.out`自行后的相关情况。我们可以看出，它的虚拟
内存大小为1616kB，当前占用的物理内存为304kB（其它数据在磁盘上或交换分区），
虚拟内存占用404个物理页面。由于我的机器的物理页面大小为4kB，可以验证404 x
4kB等于1616kB。

## 用/proc文件系统查看进程的内存使用情况

`ps`命令的输出关于内存的情况不是很详细，尤其是进程所使用的内存中有很大一部分
是共享库函数使用的，因此通过`ps`命令的输出看不到进程自己使用了多少内
存。为了查看更详细的信息，可以借助于`/proc`文件系统。这个文件系统并存
在于磁盘上，但是可以象操作其它普通文件一样操作它。它是Linux提供给用户查看进
程相关信息的接口。在`/proc`下有2个文件和进程内存有关：
`/proc/<pid>/status`和`/proc/<pid>/smaps`。

通过`/proc/<pid>/status`可以查看进程的内存使用情况，包括虚拟内存大小
（VmSize），物理内存大小（VmRSS），数据段大小（VmData），栈的大小（VmStk），
代码段的大小（VmExe），共享库的代码段大小（VmLib）等等。
```
$ cat /proc/10069/status
Name:   a.out
State:  S (sleeping)
Tgid:   10069
Pid:    10069
PPid:   6793
TracerPid:      0
Uid:    1001    1001    1001    1001
Gid:    1001    1001    1001    1001
FDSize: 256
Groups: 1000 1001 
VmPeak:     1692 kB
VmSize:     1616 kB
VmLck:         0 kB
VmHWM:       304 kB
VmRSS:       304 kB
VmData:       28 kB
VmStk:        88 kB
VmExe:         4 kB
VmLib:      1464 kB
VmPTE:        20 kB
Threads:        1
SigQ:   0/16382
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000000000
SigIgn: 0000000000000000
SigCgt: 0000000000000000
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: ffffffffffffffff
Cpus_allowed:   f
Cpus_allowed_list:      0-3
Mems_allowed:   1
Mems_allowed_list:      0
voluntary_ctxt_switches:        1
nonvoluntary_ctxt_switches:     1
```
注意，VmData，VmStk，VmExe和VmLib之和并不等于VmSize。这是因为共享库函数的数
据段没有计算进去（VmData仅包含a.out程序的数据段，不包括共享库函数的数据段，
也不包括通过mmap映射的区域。VmLib仅包括共享库的代码段，不包括共享库的数据
段）。

通过`/proc/<pid>/smaps`可以查看进程整个虚拟地址空间的映射情况，它的输
出从低地址到高地址按顺序输出每一个映射区域的相关信息，如下所示：
```
$ cat /proc/10069/smaps
00110000-00263000 r-xp 00000000 08:07 128311     /lib/tls/i686/cmov/libc-2.11.1.so
Size:               1356 kB
Rss:                 148 kB
Pss:                   8 kB
Shared_Clean:        148 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:          148 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
......
......
bfd7f000-bfd94000 rw-p 00000000 00:00 0          [stack]
Size:                 88 kB
Rss:                   8 kB
Pss:                   8 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         8 kB
Referenced:            8 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
```
注意：`rwxp`中，`p`表示私有映射（采用Copy-On-Write技术）。`Size`字段就是该区域的大小。

## 参考文献

*  `ps(1)`的`-O`选项。
*  `proc(5)`的`/proc/[pid]/status`和`/proc/[pid]/smaps`条目。
