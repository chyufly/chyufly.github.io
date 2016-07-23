---
layout: post
title: "多机环境下虚机网络实现机制（四）"
date: 2016-07-23 09:52:22 +0800
comments: true
categories: 
---

# ---  ***SDN融合方案中，neutron底层网络运行机制***
----------------------------------------------

# **1、服务部署情况**

一般而言，neutron-server和各neutron-plugin部署在控制节点或者网络节点上，而neutron agent则部署在网络节点上和计算节点上。我们先来简单地分析控制端neutron-server和neutron-plugin的工作，然后再分析设备端neutron-agent的工作。


## **1.1 compute节点的部署情况**

- keystone认证服务
- nova-comput服务
- OVS service和OVS-agent


## **1.2 network节点的部署情况**

- OVS service
- OVS agent
- L3 agent
- DHCP agent
- metadata agent
- keystone服务

## **1.3 controller节点的部署情况**

- SQL server
- message queue
- keystone service
- neutron server
- ml2 plugin




{% img /images/understanding_neutron_compute_controller_network_services.png %}


# **2、neutron底层实现机制**

## **2.1 总体介绍**

在neutron和SDN方案中，neutron底层网络的实现包括：

- agent是处理数据流的网络设备（OVS，Router，负载均衡等）
- agent是SDN controller（ODL，ONOS等）

具体关系包括两种类型，如图所示：

### **第一种：neutron作为SDN controller**

{% img /images/understanding_neutron_sdn_only_neutron.PNG %}

### **第二种：neutron作为application**

{% img /images/understanding_neutron_sdn_notonly_neutron.PNG %}

 第一种类型：neutron相当于SDN控制器
 第二种类型：neutron作为SDN的应用，将业务告知给SDN controller，neutron的角色更多的是super controller或者网络编排器，来完成openstack中有关于网络业务的集中分发，将应用的RESTful API分发给相应的SDN进行处理。具体实现方法包括：
 
 - 特定的SDN controller plugin
 - ML2 plugin和定制的mechanism driver
具体实现如下图所示：


{% img /images/understanding_neutron_sdn_opendaylight_neutron.PNG %}


## **2.2 应用举例**

openstack和SDN集成中，以openDayLight为例，可以通过实现mechanism driver来与neutron对接，利用ml2 plugin的北向 RESTful API实现功能。

通过一个openstack用户应用的调用为例进行说明（ml2配置的ODL的mechanism driver，以及一个VXLAN的driver）：

（1）用户在horizon界面发从对网络操作的一个请求，包装成RESTful API，并送给neutron server，这可以作为一个北向RESTful API；
（2）neutron server将RESTful API给到ml2 plugin，并会将配置信息同步给 neutron database（注：由于ODL driver并不实现pre commit,只实现post commit，会出现openstack的数据库和ODL数据库的不一致性）；
（3）ml2 plugin调用RESTFul API给到ODL controller；
（4）ODL接收到请求后，就会利用南向的plugins/网络协议（openFlow,OVSDB等）来对网络进行相应的操作。
 
 

{% img /images/understanding_neutron_sdn_openstack.PNG %}


