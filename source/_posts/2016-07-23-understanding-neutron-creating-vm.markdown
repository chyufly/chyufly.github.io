---
layout: post
title: "多机环境下虚机网络实现机制（一）"
date: 2016-07-23 09:51:22 +0800
comments: true
categories: 
---

# ---  ***创建虚机时的网络配置***
----------------------------------------------

# **1、neutron服务回顾**

## **1.1、neutron server**

neutron server: 对收到的 rest api 请求进行解析，并最终转换成对该 plugin(core or service) 中相应方法的调用，启动后，根据配置文件动态加载对应的core plugin 和 service plugin。

> Neutron-server可以理解为一个专门用来接收Neutron REST API调用的服务器，然后负责将不同的rest api分发到不同的neutron-plugin上。



## **1.2、plugins**

plugin主要用来帮助实现neutron server发来的API请求，主要分为core plugin和service plugin。

### **core plugin**

core plugin实现了对标准API的支持（network，subnet，port），这里主要指ml2 plugin。

  - network:代表一个隔离的二层网段，是为创建它的租户而保留的一个广播域。subnet和port始终被分配给某个特定的network。Network的类型包括Flat，VLAN，VxLAN，GRE等等。
  - subnet:代表一个IPv4/v6的CIDR地址池，以及与其相关的配置，如网关、DNS等等，该subnet中的 VM 实例随后会自动继承该配置。Sunbet必须关联一个network。
  - port: 代表虚拟交换机上的一个虚机交换端口。VM的网卡VIF连接 port 后，就会拥有 MAC 地址和 IP 地址。Port 的 IP 地址是从 subnet 地址池中分配的。




ML2 plugin作为L2的总控，其实现包括Type和Mechanism两部分，每部分又分为Manager和Driver。Type指的是L2网络的类型（如Flat、VLAN、VxLAN等），与厂家实现无关。Mechanism则是各个厂家自己设备机制的实现，如下图所示。当然有ML2，对应的就可以有ML3，不过在Neutron中L3的实现只负责路由的功能，传统路由器中的其他功能（如Firewalls、LB、VPN）都被独立出来实现了，因此暂时还没有看到对ML3的实际需求。


{% img /images/understanding_neutron_ml2.PNG %}

### **service plugin**

- service plugin：用来实现extension API，包括路由服务，防火墙，VPN等,Service-plugin，即为除core-plugin以外其它的plugin，包括l3 router、firewall、loadbalancer、VPN、metering等等，主要实现L3-L7的网络服务。这些plugin要操作的资源比较丰富，对这些资源进行操作的REST API被neutron-server看作Extension API，需要厂家自行进行扩展。



> neutron server启动的时候会启动一个core plugin和多个service plugin
> Neutron-plugin可以理解为不同网络功能实现的入口，各个厂商可以开发自己的plugin。Neutron-plugin接收neutron-server分发过来的REST API，向neutron database完成一些信息的注册，然后将具体要执行的业务操作和参数通知给自身对应的neutron agent。


## **1.3、agent**

neutron agent主要是做为plugin的后端，通过RPC远端接受plugin的命令来对本地网络资源的操作，并将结果通过call back反馈给plugin。功能分为两大类：其一就是和plugin之间的rpc消息处理，用于和plugin进行交互，接受plugin命令等，调用plugin方法并汇报状态信息等；其二就是对本地资源进行操作，包括iptables的表项，配置dhcp服务等。

- ml2 core agent：提供l2功能，该agent可以装在计算节点和网络节点上
- dhcp agent：提供对虚机的dhcp管理和分配，可以安装在任意的节点上
- l3 agent: 提供路由服务（iptables规则的配置），可以安装在任意的节点上
- other agent：提供其他的网络服务，比如metadata、metering等。



## **1.4、drivers**

通过被agent的调用，来具体实施网络操作。


# **2、vm启动时网络配置过程**

## **2.1、openstack组件之间的协作**

**大致描述**

- Nova-compute向neutron-server请求虚拟机对应的Port资源
- Neutron-server根据neutron-database生成Port资源。
- Neutron-server通知Dhcp agent虚拟机信息。
- Dhcp agent将虚拟机信息通知给dhcp server。
- 虚拟机接入并启动。
- 虚拟机从dhcp server处获得IP地址。

{% img /images/understanding_neutron_creating_vm_network.PNG %}


**主要进程**

- Nova-compute向neutron-server发送create_port的REST API请求，生成新的Port资源。
- Neutron-server收到该REST请求，通过APIRouter路由到ML2的create_port方法。该方法中，获得了neutron-database新生成的Port，并通知ML2 Mechanism Driver该Port的生成。
- Nova-compute向neutron发送update_port的REST API请求
- Neutron-server收到该REST请求，通过APIRouter路由到ML2的update_port方法。该方法中，在neutron-database更新该Port的状态，并根据ML2 Mechanism Driver的不同，决定后续的处理：若Mechanism Driver为hyperv/linuxbridge/ofagent/openvswitch，则需要通过ML2的update_port方法中执行rpc远程调用update_port；对于其余的Mechanism Driver，ML2的update_port方法调用其的update_port_postcommit方法进行处理，这些Mechanism Driver可能使用非rpc方式与自身的agent通信（如REST API、Netconf等）。
- ML2执行完update_port方法后，Port资源在wsgi中对应的Controller实例通过DhcpAgentNotifyAPI实例rpc通知给网络节点上的dhcp agent（也可能通过一些调度机制通知给分布在计算节点上的dhcp agent）。
- Dhcp agent收到该rpc通知，通过call_driver方法将虚拟机MAC与IP的绑定关系传递给本地的DHCP守护进程Dnsmaq。
- Nova-compute通过libvirt driver的spawn方法将虚拟机接入网络，然后启动虚拟机。
到这里，虚拟机启动过程中的网络处理就都结束了，虚拟机间就可以开始通信了。




