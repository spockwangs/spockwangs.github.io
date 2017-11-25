---
layout: post
title: 在Windows上配置Emacs 23
categories:
- administration
---

## Emacs的配置文件

在Windows下，Emacs的配置文件存放在被Emacs称为HOME的目录下。默认情况下该目录
是`C:/Users/<login name>/AppData/Roaming/`（Windows 7）。这个目录可以通
过环境变量`HOME`来设置。

## 启用Emacs的服务器模式

Emacs有一个服务器模式，就是先启动Emacs服务器，然后每当编辑文件时启动Emacs客
户端。这个客户端会与这个服务器联系，让其来完成各种编辑命令。这样就不用重新再
启动另一个Emacs实例来编辑文件，不管编辑多少个文件都只要一个Emacs实例（即进程）
就可以了。

为了达到这个目的，需要做以下几步设置：

1.  在`~/.emacs`中加入`(server-start)`. 这样Emacs第一次启动时会启动服务器。
2.  设置环境变量`EMACS_SERVER_FILE`为`%HOME%/.emacs.d/server/server`。这是
Emacs客户端与服务器用来通信的。每当服务器启动时都会建立这个文件，退出时删除
这个文件。这个环境变量告诉Emacs客户端服务器建立的文件在哪里。

## 将Emacs加入右键菜单

将Emacs加入资源管理器的右键菜单可以大大方便用Emacs编辑文件。而这个可以通过编辑注册表来实现。

在DOS命令行中用regedit命令打开注册表编辑器。在“HKEY_CLASSES_ROOT/*/Shell”下
新建一名为“Edit with Emacs”（可自由命名）的项，然后在这个项下再建一个名叫
“command”（必须如此命名）的项，将其值设为
`<emacs-install-dir>/bin/emacsclientw.exe
-a <emacs-install-dir>/bin/runemacs.exe -n %1`。这样就可以了。其中
`<emacs-install-dir>`表示Emacs的安装目录。emacsclientw.exe是Emacs客户端，
`-a`和`-n`都是它的选项参数，可通过`emacsclientw.exe –help`查看。`-a`表示若
Emacs服务器没有启动则启动该选项的值所表示的编辑器。
