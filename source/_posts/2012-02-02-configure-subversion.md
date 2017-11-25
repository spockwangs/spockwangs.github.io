---
layout: post
title: 配置Subversion
categories:
- Administration
---

## 配置Subversion服务器使用xinetd启动

1.  建立/etc/xinetd.d/svn，内容如下：
```
service svn
{
    disable = no
    id = svn-stream
    socket_type = stream
    protocol = tcp
    user = xxx   # 运行svnserve的用户名
    wait = no
    server = /usr/bin/svnserve
    server_args = -i -r /path/to/repository
}

service svn
{
    disable = no
    id = svn-dgram
    socket_type = dgram
    protocol = udp
    user = xxx   # 运行svnserve的用户名
    wait = yes
    server = /usr/bin/svnserve
    server_args = -i -r /path/to/repository
}
```
2.  重启xinetd服务器。
```
$ sudo invoke-rc.d xinetd restart
```
接下来我们就可以采用svn协议访版本库中的数据了。

## 建立SVN仓库

1.  确定仓库的地址，假设为`/home/svn`.
2.  创建一个仓库，名为`repos`:
```
$ svnadmin create /home/svn/repos
```
3.  配置仓库访问控制。编辑`/home/svn/repos/conf/svnserve`：
```
[general]
# ...
anon-access = read
auth-access = write
# ...
password-db = passwd
...
```
    在`[general]`下有几个属性控制访问权限。`anon-access`控制匿名用户的访问权限，
    `auth-access`控制授权用户的访问权限，它们可以为`read`, `write`, `none`。权限
    控制有几种方式，最简单的一种是通过密码文件来授权。密码文件的位置由
    `password-db`指定,一般就放在与`svnserve`同一目录下，叫`passwd`，格式如下：
```
        name = password
```
    等号左边是用户名，右边是密码。如果不允许匿名操作，则需要输入密码。
4.  导入需要版本控制的数据。假设数据在`tree`目录下，要导入到刚刚建立的仓库中：
```
$ svn import tree svn://svnhost/repos/project
```
