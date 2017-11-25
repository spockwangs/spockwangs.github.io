---
layout: post
title: 用MySql实现事务型消息队列
categories:
- programming practice
- database
- distributed system
---

## Introduction

在离线数据处理系统中，为了解除模块之间的耦合关系，往往需要消息队列来实现模块之间的通信。对于离线系统来讲，消息队列要满足以下要求：

* 消息不能丢失，即使在系统失败的情况下。消息一旦被插入就一定会被至少处理一次（只被处理一次是最好的，但是实现起来有难度，所以只要求at-least-once semantic）；
* FIFO顺序；
* 支持多生产者；
* 支持多消费者。每个消息只能被其中一个消费者处理。

## 第一次尝试

为了满足事务特性，最简单的做法就是用MySql数据库来实现这个队列。为了支持FIFO顺序，可以用MySql的自增ID来排序，先进入的消息ID要小些。MySql数据库是支持并发操作的，这就自动支持了多生产者的情况。为了支持多消费者并且每个消息只被一个消费者处理，可以为每个消费者分配一个ID，当某个消息正在被消费者处理时，这个消费者就被认为是这个消息的owner。

根据以上思路，构造表如下：

    CREATE TABLE message_queue (
      id BIGINT PRIMARY KEY AUTO_INCREMENT,
      
      -- owner为NULL时表示消息未被处理；否则正在被某个消费者处理，其值就是这个消费者的ID
      owner VARBINARY(255),
      message VARBINARY(255) NOT NULL
    );
    
其中`message`是跟业务相关的消息数据，可以是一个字段也可以是多个字段。那么enqueue操作如下：

    insert into message_queue (message) values ('a message');
    
dequeue操作有点复杂，分3步：

1. 获取未被处理的消息。先占有这些消息，设置owner。为了加快速度减少访问数据库的频率，可以批量操作；

        update message_queue set owner='me' where owner is null limit 10;

    根据[`mysql_affected_rows()`](http://dev.mysql.com/doc/refman/5.0/en/mysql-affected-rows.html)的返回结果可以判断是否有未被处理的消息。若有，则取出消息：

        select message from message_queue where owner='me';

    注意，`update`操作是根据自增ID的顺序操作的，这就实现了FIFO。若把这两步合并成一个事务执行会造成对表的锁占用过多时间（多了一个请求响应来回时间）。
2. 处理消息，此时表不会被锁住，其它消费者也可以获取消息；
3. 处理完毕后从数据库中删除。对于处理出错的消息可以根据业务要求重试、忽略或者插入消息队列待下次再处理。

        delete from message_queue where owner='me'

上面没有考虑失败的情况。如果某个消费者在第1步和第2步之间失败的话，一个被占用的消息可能永远不会再被处理，即使此时有其它消费者还在运行，这就不满足at-least-once的要求了。

## 考虑消费者失败的情况

为了解决考虑消费者失败的情况，一个比较简单的方法就是对消费者进行监控（比如采用心跳），一旦失败就将其占有的但是还未删除的消息的`owner`设置成`null`。但是由谁来监控呢？容易想到的方法可能是单独建立一个服务对每个消费者进行监控，不过对于我们这里的简单问题不需要这么复杂，而且我本人不喜欢这种主从不对称的解决方案（毕竟谁来监控这个监控服务呢？）。我们可以让消费者相互监控。

每当消费者取消息时，如果发现有消息虽然被占用但是却长时间没有处理完毕，那么就认为该消息的`owner`失败了，这个消息就可以被占用。为了判断消息处理是否超时，增加一个字段表示该消息被处理的时间戳：

    CREATE TABLE message_queue (
      id BIGINT PRIMARY KEY AUTO_INCREMENT,
      
      -- owner为NULL时表示消息未被处理；否则正在被某个消费者处理，其值就是这个消费者的ID
      owner VARBINARY(255),
      
      -- 最近一次被owner取出的时间
      dequeued_time DATETIME,
      message VARBINARY(255) NOT NULL
    );
    
`dequeue`的第1步的update操作改为如下（假设消息处理的耗时不超过60s）：

    update message_queue set owner='me',dequeued_time=NOW() 
        where owner is null or dequeued_time < SUBTIME(NOW(), SEC_TO_TIME(60)) limit 10;
    
这样即使消费者A在第1步和第2步之间失败了，消费者B还可以重新取出该消息重新处理。所以只要还有消费者在，消息至少会被处理一次。

## 如何实现只处理一次呢？

如果消费者在第2步和第3步之间失败的话，这个消息就会被再次取出来处理一次，这样就一共处理了2次。要保证每个消息只被处理一次，也就是要保证第2步和第3步是一个原子操作，要么都做，要么都不做。换句话说，第2步和第3步要合并成一个事务操作，但是这两步涉及到的是两个不同的实体（一个视业务逻辑而定，一个是MySql数据库），用事务处理的概念来讲就是两个不同的resource manager，必须采用分布式事务协议（比如两步提交协议）才能实现事务语义，这已远超出本文的范围了，不在这里赘述。

## See also

* [5-subtle-ways-youre-using-mysql-as-a-queue-and-why-itll-bite-you](https://blog.engineyard.com/2011/5-subtle-ways-youre-using-mysql-as-a-queue-and-why-itll-bite-you/)
* [RabbitMQ](https://www.rabbitmq.com/)是一个消息队列解决方案，毕竟用数据库实现只是权宜之计
