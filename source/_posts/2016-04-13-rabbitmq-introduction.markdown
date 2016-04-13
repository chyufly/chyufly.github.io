---
layout: post
title: "RabbitMQ概念和应用详解"
date: 2016-04-13 10:46:01 +0800
comments: true
categories: 
---



-------------------------------

## RabbitMQ概述

###RabbitMQ可以做什么？
RabbitMQ是实现AMQP(高级消息队列协议)的消息中间件的一种，可用于在分布式系统中存储转发消息，主要有以下的技术亮点：

- 可靠性
- 灵活的路由
- 集群部署
- 高可用的队列消息
- 可视化的管理工具

RabbitMQ主要用于系统间的双向解耦，当生产者（productor）产生大量的数据时，消费者（consumer）无法快速的消费信息，那么就需要一个类似于中间件的代理服务器，用来处理和保存这些数据，RabbitMQ就扮演了这个角色。
###如何使用RabbitMQ
- Erlang语言包
- RabbitMQ安装包

###基本概念

####1.Broker
 用来处理数据的消息队列服务器实体
####2.虚拟主机(vhost)
 由RabbitMQ服务器创建的虚拟消息主机，拥有自己的权限机制，一个broker里可以开设多个vhost，用于不同用户的权限隔离，vhost之间是也完全隔离的。
####3.生产者(productor)
 产生用于消息通信的数据
####4.信道(channel)
消息通道，在AMQP中可以建立多个channel，每个channel代表一个会话任务。
####5.交换机(exchange)
(1)接受消息，转发消息到绑定的队列，总共有四种类型的交换器：direct，fanout，topic，headers。

- **direct：**转发消息到routing-key指定的队列


{% img /images/direct-exchange.png %}


- **fanout：**转发消息到所有绑定的队列，类似于一种广播发送的方式。


- **topic：**按照规则转发消息，这种规则多为模式匹配，也显得更加灵活

{% img /images/topic-exchange.png %}



(2).交换器在RabbitMQ中是一个存在的实体，不能改变，如有需要只能删除重建。

(3).topic类型的交换器利用匹配规则分析消息的routing-key属性。

(4).属性

- **持久性：**声明时durable属性为true
- **自动删除：**绑定的queue删除也跟着删除
- **惰性：**不会自动创建

####6.队列(queue)

(1).队列是RabbitMQ的内部对象，存储消息

(2).可以动态的增加消费者，队列将接受到的消息以轮询(round-robin)的方式均匀的分配给多个消费者

(3).队列的属性

- **持久性：**如果启用，队列将会在server重启之前有效
- **自动删除：**消费者停止使用之后就会自动删除
- **惰性：**不会自动创建
- **排他性：**如果启用，队列只能被声明它的消费者使用。

####7.两个key

- **routing-key:**消息不能直接发到queues，需要先发送到exchanges，routing-key指定queues名称，exchanges通过routing-key来识别与之绑定的queues

``` python
channel.queue_publish(exchange=exchange_name,
                     routing-key="rabbitmq",
                     body="openstack")
```

- **binding-key:**主要是用来表示exchanges和queues之间的关系，为了区别queue_publish的routing-key，就称作binding-key。
``` python
channel.queue_bind(exchange=exchange_name,
                  queue=queue_name,
                  routing-key="rabbitmq")
```

####8.绑定(binding)

表示交换机和队列之间的关系，在进行绑定时，带有一个额外的参数*binding-key*，来和*routing-key*相匹配。

####9.消费者(consumer)

监听消息队列来进行消息数据的读取

####10.高可用性(HA)

(1).在consumer处理完消息后，会发送消息ACK，通知通知RabbitMQ消息已被处理，可以从内存删除。如果消费者因宕机或链接失败等原因没有发送ACK，则RabbitMQ会将消息重新发送给其他监听在队列的下一个消费者。

``` python
channel.basicConsume(queuename, noAck=false, consumer);
```
(2).消息和队列的持久化

(3).镜像队列，实现不同节点之间的元数据和消息同步


##RabbitMQ在OpenStack中的应用


###**RPC之neutron专题**


基于RabbitMQ的RPC消息通信是neutron中跨模块进行方法调用的很重要的一种方式，根据上面的描述，要组成一个完整的RPC通信结构，需要信息的生产者和消费者。

- **client端：**用于产生rpc消息。
- **server端：**用于监听消息数据并进行相应的处理。

####1.neutron-agent中的RPC

在dhcp_agent、l3_agent、metadata_agent，metering_agent的main函数中都存在一段创建一个rpc服务端的代码，下面以dhcp_agent为例。

``` python
def main():
    register_options(cfg.CONF)
    common_config.init(sys.argv[1:])
    config.setup_logging()
    server = neutron_service.Service.create(
        binary='neutron-dhcp-agent',
        topic=topics.DHCP_AGENT,
        report_interval=cfg.CONF.AGENT.report_interval,
        manager='neutron.agent.dhcp.agent.DhcpAgentWithStateReport')
    service.launch(cfg.CONF, server).wait()
```



最核心的，也是跟rpc相关的部分包括两部分，首先是创建rpc服务端。

``` python
    server = neutron_service.Service.create(
        binary='neutron-dhcp-agent',
        topic=topics.DHCP_AGENT,
        report_interval=cfg.CONF.AGENT.report_interval,
        manager='neutron.agent.dhcp.agent.DhcpAgentWithStateReport')
```


该代码实际上创建了一个rpc服务端，监听指定的topic并定期的运行manager上的定期任务。

create()方法返回一个neutron.service.Service对象，neutron.service.Service继承自neutron.common.rpc.Service类。

首先看neutron.common.rpc.Service类，该类定义了start方法，该方法主要完成两件事情：一件事情是将manager添加到endpoints中；一件是创建了三个rpc的consumer，分别监听topic、topic.host和fanout的队列消息。

而在neutron.service.Service类中，初始化中生成了一个manager实例（即neutron.agent.dhcp_agent.DhcpAgentWithStateReport）；并为start方法添加了周期性执行report_state方法和periodic_tasks方法。report_state方法实际上什么都没做，periodic_tasks方法则调用manager的periodic_tasks方法。

manager实例（即neutron.agent.dhcp_agent.DhcpAgentWithStateReport）在初始化的时候首先创建一个rpc的client端，通过代码

``` python
 self.state_rpc = agent_rpc.PluginReportStateAPI(topics.PLUGIN)
```

该client端实际上定义了report_state方法，可以状态以rpc消息的方式发送给plugin。

manager在初始化后，还会指定周期性运行_report_state方法，实际上就是调用client端的report_state方法。

至此，对rpc服务端的创建算是完成了，之后执行代码。

``` python
service.launch(server).wait()
```


service.launch(server)方法首先会将server放到协程组中，并调用server的start方法来启动server。

####2.neutron-plugin中的RPC

以openvswitch的plugin为例进行分析。

neutron.plugin.openvswitch.ovs_neutron_plugin.OVSNeutronPluginV2类在初始化的时候调用了self.setup_rpc方法。


其代码为

``` python
def setup_rpc(self):
        # RPC support
        self.service_topics = {svc_constants.CORE: topics.PLUGIN,
                               svc_constants.L3_ROUTER_NAT: topics.L3PLUGIN}
        self.conn = n_rpc.create_connection(new=True)
        self.notifier = AgentNotifierApi(topics.AGENT)
        self.agent_notifiers[q_const.AGENT_TYPE_DHCP] = (
            dhcp_rpc_agent_api.DhcpAgentNotifyAPI()
        )
        self.agent_notifiers[q_const.AGENT_TYPE_L3] = (
            l3_rpc_agent_api.L3AgentNotifyAPI()
        )
        self.endpoints = [OVSRpcCallbacks(self.notifier, self.tunnel_type),
                          agents_db.AgentExtRpcCallback()]
        for svc_topic in self.service_topics.values():
            self.conn.create_consumer(svc_topic, self.endpoints, fanout=False)
        # Consume from all consumers in threads
        self.conn.consume_in_threads()
```



创建一个通知rpc的客户端，用于向l2的agent发出通知。所有plugin都需要有这样一个发出通知消息的客户端。

分别创建了一个dhcp agent和l3 agent的通知rpc客户端。

之后，创建两个跟service agent相关的consumer，分别监听topics.PLUGIN和topics.L3PLUGIN两个主题。

 {% img /images/RPC_Neutron.PNG %}




####3.neutron-server中的RPC
这个rpc服务端主要通过neutron.server中主函数中代码执行

``` python
neutron_rpc = service.serve_rpc()
```

方法的实现代码(目录：neutron/neutron/service.py)如下

``` python
def serve_rpc():
    plugin = manager.NeutronManager.get_plugin()
    service_plugins = (
        manager.NeutronManager.get_service_plugins().values())

    if cfg.CONF.rpc_workers < 1:
        cfg.CONF.set_override('rpc_workers', 1)
    if not plugin.rpc_workers_supported():
        LOG.debug("Active plugin doesn't implement start_rpc_listeners")
        if 0 < cfg.CONF.rpc_workers:
            LOG.error(_LE("'rpc_workers = %d' ignored because "
                          "start_rpc_listeners is not implemented."),
                      cfg.CONF.rpc_workers)
        raise NotImplementedError()

    try:
        rpc = RpcWorker(service_plugins)
        LOG.debug('using launcher for rpc, workers=%s', cfg.CONF.rpc_workers)
        session.dispose()
        launcher = common_service.ProcessLauncher(cfg.CONF, wait_interval=1.0)
        launcher.launch_service(rpc, workers=cfg.CONF.rpc_workers)
        if (cfg.CONF.rpc_state_report_workers > 0 and
            plugin.rpc_state_report_workers_supported()):
            rpc_state_rep = RpcReportsWorker([plugin])
            LOG.debug('using launcher for state reports rpc, workers=%s',
                      cfg.CONF.rpc_state_report_workers)
            launcher.launch_service(
                rpc_state_rep, workers=cfg.CONF.rpc_state_report_workers)

        return launcher
    except Exception:
        with excutils.save_and_reraise_exception():
            LOG.exception(_LE('Unrecoverable error: please check log for '
                              'details.'))
```



其中，RpcWorker(plugin)主要通过调用plugin的方法来创建rpc服务端。

``` python
self._servers = self._plugin.start_rpc_listeners()
```
该方法在大多数plugin中并未被实现，目前ml2支持该方法。

在neutron.plugin.ml2.plugin.ML2Plugin类中，该方法创建了一个topic为topics.PLUGIN的消费rpc。

``` python
def start_rpc_listeners(self):
        self.endpoints = [rpc.RpcCallbacks(self.notifier, self.type_manager),
                          agents_db.AgentExtRpcCallback()]
        self.topic = topics.PLUGIN
        self.conn = n_rpc.create_connection(new=True)
        self.conn.create_consumer(self.topic, self.endpoints,
                                  fanout=False)
        return self.conn.consume_in_threads()
```

###**RPC之nova专题**

在Openstack中，每一个Nova服务初始化时会创建两个队列，一个名为“NODE-TYPE.NODE-ID”，另一个名为“NODE-TYPE”，NODE-TYPE是指服务的类型，NODE-ID指节点名称。

{% img /images/RPC_Nova.png %}

####1.nova中实现exchange的种类

- **direct:**初始化中，各个模块对每一条系统消息自动生成多个队列放入RabbitMQ服务器中，队列中绑定的binding-key要与routing-key匹配
- **topic:**各个模块也会自动生成两个队列放入RabbitMQ服务器中。

####2.nova中调用RPC的方式
- **RPC.CALL:**用于请求和响应方式
- **RPC.CAST:**只是提供单向请求
 
####3.nova中模块的逻辑功能
- **Invoker:**向消息队列中发送系统请求信息，如Nova-API和Nova-Scheduler，通过RPC.CALL和RPC.CAST两个进程发送系统请求消息。
- **Worker:**从消息队列中获取Invoker模块发送的系统请求消息以及向Invoker模块回复系统响应消息，如Nova-Compute、Nova-Volume和Nova-Network，对RPC.CALL做出响应。

####4.nova中的exchange domain

- **direct exchange domain:** Topic消息生产者（Nova-API或者Nova-Scheduler）与Topic交换器生成逻辑连接，通过PRC.CALL或者RPC.CAST进程将系统请求消息发往Topic交换器。交换器根据不同的routing-key将系统请求消息转发到不同的类型的消息队列。Topic消息消费者探测到新消息已进入响应队列，立即从队列中接收消息并调用执行系统消息所请求的应用程序。

    - *点到点消息队列*：Topic消息消费者应用程序接收RPC.CALL的远程调用请求，并在执行相关计算任务之后将结果以系统响应消息的方式通过Direct交换器反馈给Direct消息消费者。

    - *共享消息队列*：Topic消息消费者应用程序只是接收RPC.CAST的远程调用请求来执行相关的计算任务，并没有响应消息反馈。

- **topic exchange domain:** Direct交换域并不是独立运作，而是受限于Topic交换域中RPC.CALL的远程调用流程与结果，每一个RPC.CALL激活一次Direct消息交换的运作。


{% img /images/RPC_Nova_domain.PNG %}


*以nova启动虚拟机的过程为例，详细介绍RPC通信过程。*


{% img /images/RPC_Nova_rpcCALL.PNG %}  


   RPC.CAST缺少了系统消息响应流程。一个Topic消息生产者发送系统请求消息到Topic交换器，Topic交换器根据消息的Routing Key将消息转发至共享消息队列，与共享消息队列相连的所有Topic消费者接收该系统请求消息，并把它传递给响应的Worker进行处理，其调用流程如图所示：

{% img /images/RPC_Nova_rpcCAST.PNG %}


