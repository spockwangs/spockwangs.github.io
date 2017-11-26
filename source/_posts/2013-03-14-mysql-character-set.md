---
layout: post
title: 关于Mysql客户端、连接和服务器字符集
comments: true
categories:
- mysql
- database
---

## Introduction

Mysql在客户端与服务器交互时涉及到几个字符集和collation的转换问题：

*  客户端发给服务器的查询语句的字符集是什么？
   
   客户端发给服务器的语句的字符集由系统变量[character_set_client][character_set_client]定义。
<!--more-->

*  服务器收到查询语句后会转换成什么字符集？

   服务器将收到的查询语句从[character_set_client][character_set_client]转换为
   [character_set_connection][character_set_connection]，除非字符串常量显式指定了字符集。系统变量
   [collation_connection][collation_connection]定义了字符串常量的collation.

*  服务器返回给客户端的数据的字符集 是什么？
   
   服务器返回给客户端的数据字符集由系统变量[character_set_results][character_set_results]定义。

*  服务器采用什么字符集存储数据？

   Mysql中[字符串类型](http://dev.mysql.com/doc/refman/5.5/en/string-types.html)有两种。对于非二进制字符串类型（`char`,
   `varchar`和`text`）采用建表时指定的字符集存储，对于二进制类型的数据（`binary`, `varbinary`, `blob`, `tinyblob`,
   `mediumblob`等类型的数据）采用[character_set_connection][character_set_connection]定义的字符集所映射的二进制值存储，注
   意这些类型本身是没有字符集的。
   
上面这些系统变量均是[会话变量](http://dev.mysql.com/doc/refman/5.0/en/using-system-variables.html)，改变它们的值不会影响其它客户端与服务器的连接。

## 如何设置字符集

我们通常用`set names`语句设置连接相关的字符集。
```
mysql> set names 'charset_name' [collate 'collation_name']
```
等价于
```
mysql> set character_set_client = 'charset_name'
mysql> set character_set_results = 'charset_name'
mysql> set character_set_connection = 'charset_name'
```
## 如何查看设置的字符集

可以这样查看设置的系统变量
```
mysql> show variables like 'character_set%';
+--------------------------+---------------------------------------+
| Variable_name            | Value                                 |
+--------------------------+---------------------------------------+
| character_set_client     | latin1                                | 
| character_set_connection | latin1                                | 
| character_set_database   | latin1                                | 
| character_set_filesystem | binary                                | 
| character_set_results    | latin1                                | 
| character_set_server     | latin1                                | 
| character_set_system     | utf8                                  | 
| character_sets_dir       | /data/mysql_root/base/share/charsets/ | 
+--------------------------+---------------------------------------+
8 rows in set (0.00 sec)
```
## References

*  [关于连接字符集](http://dev.mysql.com/doc/refman/5.5/en/charset-connection.html).
*  [关于Mysql的两种系统变量](http://dev.mysql.com/doc/refman/5.0/en/using-system-variables.html).

[character_set_client]: http://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_client
[character_set_connection]: http://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_connection
[character_set_server]: http://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_server
[collation_connection]: http://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_connection
