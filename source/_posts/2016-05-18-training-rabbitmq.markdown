---
layout: post
title: "US专题(一)-RabbitMQ"
date: 2016-05-18 23:12:23 +0800
comments: true
categories: 
---
> 说明：RabbitMQ是一种消息中间件，并以其高效的集群部署和HA得到广泛的应用

###**有关queue_declare的总结**


在publish/subscribe这种方式中，如果将queue中定义的prefetch_count设置为1，那么创建的queue就会认为只会提取一个消息。这种场景适合于希望一段时间消息被固定在一个消费者上。如果不设置这个参数，可能在product产生多个消息发出后，一个消费者就可能从RabbitMQ中取所有的消息，而不让其他消费者对消息进行消费。

> 这里需要注意的一个问题就是，一个消费者只能绑定一个queue，一个exchange可以定义多个queue。而多个消费者可能就是在轮询的接受信息。


###**RabbitMQ的HA**
这个HA的特性是RabbitMQ的特点之一，所以对开发者来说就显得比较重要。
主要包括两部分：

- 服务可靠：这里的服务可靠性主要是指RabbitMQ的集群，保证元数据的HA

> rabbitmq cluster 实现vhost&exchange&queue&binding等元数据的同步

- 数据可靠：这里的数据可靠性主要是针对mirror queue，镜像队列可以将消息复制到集群的多个节点上去，这样如果出现某一个节点down的情况，也可以保证整个集群式高可用的。

> mirror queue 实现队列中消息的同步


####**如何实现RabbitMQ的高可用性**

#####可靠的消息传递

 - **Pubilsher confirm**

 - **Delivery Acknowledge**

 - **故障时消息不会消失：多个副本；写磁盘**




##### 消息队列服务HA

 - **避免单点故障**

 - **可扩展的消息集群**

#####实现高可用的集群

 - **rabbitmq cluster**

 - **mirror queue**

 - **共享存储**

 - **pacemaker：实现服务监控和故障转移**



####比较好的参考资料：

https://www.rabbitmq.com/ha.html

https://www.rabbitmq.com/clustaering.html

http://www.rabbitmq.com/pacemaker.html

http://pdf.th7.cn/down/files/1312/RabbitMQ%20in%20Action.pdf

http://my.oschina.net/moooofly/blog/284542

http://my.oschina.net/hncscwc/blog/186350


###**RabbitMQ troubleshooting介绍**

####**排错工具**

#####通过监控查看rabbitmq的状态是否正常
 - **rabbitmqctl & rabbitmqadmin**

```
启动或终止rabbitmq
用户、权限、集群、policy管理
查看队列、exchange、queue、channel、consumer、状态...
```
 - **rabbitmq manangement plugin**
#####查看日志，确定具体的错误
 - **启动日志：/var/log/rabbitmq/startup_{log,err} **
 - **关闭日志：/var/log/rabbitmq/shutdown_{log,err} **
 - **运行日志：/var/log/rabbitmq/rabbit@<host>.log**

####**排错思路**

- rabbitmq server 本身的问题
- 应用程序的问题

####**常见故障检测**

- rabbitmq server启动失败
- rabbitmq 网络分区
- rabbitmq 不接受消息
- rabbitmq 消息堆积

####**rabbitmq server启动失败**

- cookie不一致
- hostname发生了变化
- 集群：连接不上其他节点

####**网络分区**

- 出现网络分区的情况下，各自独立提供服务，但是不再同步消息
- net_ticktime：默认60-120秒；这个时间段内节点之间会发送4个tick包
- mnesia不一致：网络中断，不能通讯

####**rabbitmq不接受消息**


#####rabbitmq内部发生了阻塞
 - **flow control:内存、磁盘**
 - **消息堆积**
#####连接异常导致空等

####**rabbitmq消息堆积**

在出现网络连接异常时，client端没有重新建立连接，一直等待，不消费消息
####**不能检测网络故障并自动重连**

- RabbitMQ heartbeat
- tcp keep alive
- net.ipv4.tcp_retries2

####**参考资料**
https://www.rabbitmq.com/management.html

https://www.rabbitmq.com/nettick.html

http://stackoverflow.com/questions/5907527/application-control-of-tcp-retransmission-on-linux



###**performance tuning for RabbitMQ**

####**trade-off**

- 对性能的需求：1k/s,10k/s,100k/s

- 对消息持久性的需求

- 通信模式：pub-sub,fanout,topic

- 集群的规模

####**性能调优**

- 放开文件描述符的限制

- 提高内存的占用量

- 提高磁盘读写性能




