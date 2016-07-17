---
layout: post
title: "多机环境下虚拟网络实现机制（二）"
date: 2016-07-11 22:48:58 +0800
comments: true
categories: 
---

# ---  ***南北、东西流量网络实现原理***
----------------------------------------------

本部分主要讨论vm的南北流量和东西流量之间的数据包转发机制，原理上使用neutron的ml2 plugin的实现方法，属于纯neutron方案。

（1）南北流向：主要是指vm（Project network）和外面网络（external network）之间的数据包流向，这里存在两种不同的情况：

- vm利用fixed IP地址向外网发送数据包；
- 外界通过floating IP向内部虚拟机发送网络数据包

（2）东西流向：主要是vm之间的网络数据包的流向和转发，也存在两种不同的情况：

- 在同一个子网中的不同虚机之间进行网络包的传输和转发
- 在不同子网中的不同虚机之间进行网络包的传输和转发

# 1、**南北流量实现机制**



## **1.1 Project network向External network发送网络包**

在多机环境下，在compute node上创建虚机后会分配一个fixed IP来进行通信，以172.21.11.42上的host01为例，对Project network向External network的网络流量进行分析。

### **环境准备**

- External network
  - Network：172.21.11.0/24
  - Project network 路由外部网关：172.21.11.234

- Project network
  - Network: 10.0.1.0/24
  - Gateway :10.0.1.1

- Compute node （host01）
  - IP address：10.0.1.3

    
### **过程分析**

#### ***计算节点的网络包转发过程***

- （1）首先，创建的虚拟机host01将数据包通过tap设备发送给Linux Bridge，该数据包包含着Project network的目的MAC地址10.0.1.1;
- （2）接着，Linux Bridge根据安全组规则对数据包进行处理，如果符合规则就通过端口qvb发送给OVS的网桥br-int的端口qvo;
- （3）然后，br-int在qvo端口处对数据包添加相应的vlan tag，并将带有vlan tag的数据包发送给br-tun网桥;
- （4）最后，br-tun网桥收到带有vlan tag的数据包后，按照流表规则，将vlan tag匹配的数据包去掉tag，添加对应的vxlan tunnel号，通过接口发送到VXLAN隧道中去.

#### ***网络节点的网络包转发过程***

- （1）首先，网络节点的OVS网桥br-tun收到计算节点发过来的带有vxlan tunnel号的网络包，按照流表规则进行处理，遇到vxlan tunnel号匹配的网络包就会剥掉vxlan tunnel号，并添加对应的vlan tag，发送给br-int；
- （2）接着，br-int收到带有vlan tag的数据包后，通过端口qr发送给路由器的qr端口，路由器的qr端口存放着该project network的网关IP地址（10.0.1.1）；
- （3）然后，在路由服务的namespace中，由于路由器的qg端口存放着project network路由器接口地址（172.21.11.234），就可以利用iptables将该地址作为源地址，进行SNAT转换；
- （4）最后，路由器namespace将SNAT转换后的数据包通过qg端口发送给br-int网桥，br-int网桥再转发给br-ex网桥，br-ex通过外部网卡将数据包发送给外部的网络，这样数据包就从虚拟机发送到外部的网络中去了。


通过namespace查看路由服务内的信息如下所示：

```
[root@host41 ~]# ip netns exec qrouter-ec5e63fc-c5a4-4925-9767-154583432d21 ip route
default via 172.21.11.1 dev qg-4306bf11-af 
10.0.1.0/24 dev qr-4d5301c0-7c  proto kernel  scope link  src 10.0.1.1 
10.1.0.0/24 dev qr-c4a064a6-c5  proto kernel  scope link  src 10.1.0.1 
172.21.11.0/24 dev qg-4306bf11-af  proto kernel  scope link  src 172.21.11.234 
192.168.0.0/24 dev qr-622bfadb-9b  proto kernel  scope link  src 192.168.0.1 
```

{% img /images/understanding_neutron_inter_to_external.PNG %}

## **1.2 External network向Project network发送网络包**

在多机环境下，如果External network能够连接到内部的虚拟机，那么需要给虚拟机分配一个floating ip，还是以创建的虚机host01为例，给它分配一个floating ip（172.21.11.209），现在分析具体的流量转发实现。

### **环境准备**

- External network
  - Network：172.21.11.0/24
  - Project network 路由外部网关：172.21.11.234

- Project network
  - Network: 10.0.1.0/24
  - Gateway :10.0.1.1

- Compute node （host01）
  - IP address：10.0.1.3
  - floating IP：172.21.11.209

    
### **过程分析**


#### ***network node的网络包转发过程***

- （1）首先，外部external network发送网络包到OVS的网桥br-ex，接着br-ex会将该网络包发送给br-int进行处理；
- （2）接着，br-int将网络包通过qg接口送给路由服务qrouter，在路由namespace中的qg端口上
包含有host01的floating IP（172.21.11.209），在qr端口上包含有路由的外部网关地址（172.21.11.234）。在qrouter中，iptables services会将qr端口中的网关地址（172.21.11.234）作为源地址进行DNAT转换；
- （3）然后，qrouter将网络包发送给br-int，br-int将网络包添加vlan tag发送给br-tun；
- （4）最后，br-tun根据流表规则剥掉匹配的网络包的vlan tag，添加相应的vxlan tunnel号，扔到vxlan隧道中去。


#### ***compute node的网络包转发过程***

- （1）首先，计算节点上的br-tun接受vxlan隧道中的带有vxlan tunnel号的网络包，根据流表剥掉匹配的vxlan tunnel号，并添加对应的vlan tag，发送给br-int；
- （2）接着，br-int将数据包发送给Linux bridge进行处理；
- （3）然后，Linux Bridge根据安全组规则对数据包进行处理；
- （4）最后，linux bridge将处理后的数据包通过tap设备发送给虚机host01。


```
[root@host41 ~]# ip netns exec qrouter-ec5e63fc-c5a4-4925-9767-154583432d21 iptables -t nat -S 
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N neutron-l3-agent-OUTPUT
-N neutron-l3-agent-POSTROUTING
-N neutron-l3-agent-PREROUTING
-N neutron-l3-agent-float-snat
-N neutron-l3-agent-snat
-N neutron-postrouting-bottom
-N neutron-vpn-agen-OUTPUT
-N neutron-vpn-agen-POSTROUTING
-N neutron-vpn-agen-PREROUTING
-N neutron-vpn-agen-float-snat
-N neutron-vpn-agen-snat
-A PREROUTING -j neutron-vpn-agen-PREROUTING
-A PREROUTING -j neutron-l3-agent-PREROUTING
-A OUTPUT -j neutron-vpn-agen-OUTPUT
-A OUTPUT -j neutron-l3-agent-OUTPUT
-A POSTROUTING -j neutron-vpn-agen-POSTROUTING
-A POSTROUTING -j neutron-postrouting-bottom
-A POSTROUTING -j neutron-l3-agent-POSTROUTING
-A neutron-l3-agent-OUTPUT -d 172.21.11.209/32 -j DNAT --to-destination 10.0.1.3
-A neutron-l3-agent-POSTROUTING ! -i qg-4306bf11-af ! -o qg-4306bf11-af -m conntrack ! --ctstate DNAT -j ACCEPT
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
-A neutron-l3-agent-PREROUTING -d 172.21.11.209/32 -j DNAT --to-destination 10.0.1.3
-A neutron-l3-agent-float-snat -s 10.0.1.3/32 -j SNAT --to-source 172.21.11.209
-A neutron-l3-agent-snat -j neutron-l3-agent-float-snat
-A neutron-l3-agent-snat -o qg-4306bf11-af -j SNAT --to-source 172.21.11.234
-A neutron-l3-agent-snat -m mark ! --mark 0x2/0xffff -m conntrack --ctstate DNAT -j SNAT --to-source 172.21.11.234
```


{% img /images/understanding_neutron_external_to_inter.PNG %}

# 2、东西流量实现机制

## **2.1 同一子网中的不同虚机转发网络包**

### **环境描述**

- Project network
  - Network: 10.0.1.0/24

- Compute node 1:
  - host01:10.0.1.3

- Compute node 2:
  - host02:10.0.1.4

- host01和host02属于同一project network（10.0.1.0/24），创建的时候分别部署在不同的compute node上

- host01向host02发送网络包


  
    
### **过程分析**

#### ***compute node1的网络包转发过程***

- （1）首先，host01先将带有目的地址10.0.1.4的network-10.0.1.3的数据包发送给linux bridge；
- （2）接着，Linux Bridge根据安全组规则对network-10.0.1.3的数据包进行处理，规则匹配后将数据包发送到br-int；
- （3）然后，br-int给network-10.0.1.3的网络添加vlan tag，并扔给br-tun，br-tun收到网络包后，根据流表规则匹配vlan tag，符合的对vlan tag进行剥离，添加相应的vxlan tunnel号扔给VXLAN隧道；

#### ***compute node2的网络包转发过程***

- （1）首先，br-tun接收到VXLAN隧道内的网络数据包，将匹配的vxlan tunnel号的网络包剥离vxlan号，添加network-10.0.1.4的vlan tag，并将处理后的数据包扔给br-int；
- （2）接着，br-int将数据包交给linux bridge；
- （3）然后，Linux Bridge根据安全组规则对network-10.0.1.4的数据包进行处理，将处理后的数据包通过tap设备交给host02。



{% img /images/understanding_neutron_same_subnet.PNG %}



## **2.2 不同子网中的不同虚机转发网络包**

### **环境描述**

- Project network1
  - Network: 10.0.1.0/24

- Project network2
  - Network: 10.1.0.0/24
  
- Compute node 1:
  - host03:10.1.0.4

- Compute node 2:
  - host02:10.0.1.4

- 两个Project network同属于一个路由namespace

- host03向host02发送网络包
    
### **过程分析**

#### ***compute node1的网络包转发过程***

- （1）首先，host01先将带有目的网关地址10.1.0.1的network-10.1.0.4的数据包发送给linux bridge；
- （2）接着，Linux Bridge根据安全组规则对network-10.1.0.4的数据包进行处理，规则匹配后将数据包发送到br-int；
- （3）然后，br-int给network-10.1.0.4的网络添加vlan tag，并扔给br-tun，br-tun收到网络包后，根据流表规则匹配vlan tag，符合的对vlan tag进行剥离，添加相应的vxlan tunnel号扔给VXLAN隧道；


#### ***network node的网络包转发过程***

- （1）首先，br-tun收到VXLAN隧道内的网络包，匹配规则剥掉vxlan tunnel号，添加network-10.1.0.4的vlan tag，将数据包扔给br-int；
- （2）接着，br-int将带有network-10.1.0.4的vlan tag的数据包扔给路由服务的qr-1端口，qr-1端口包含network-10.1.0.4的网关地址10.1.0.1，然后路由器将数据包广播路由给qr-2端口，qr-2端口包含network-10.0.1.4的网关地址10.0.1.1；
- （3）然后，路由服务qrouter将数据包交给br-int，br-int将数据包添加上network-10.0.1.4的vlan tag；
- （4）最后，br-int将数据包扔给br-tun，br-tun匹配vlan tag后，剥离vlan tag，并添加可以识别network-10.0.1.4的vxlan tunnel号，然后扔到VXLAN隧道中去。



#### ***compute node2的网络包转发过程***

- （1）首先，br-tun接收到VXLAN隧道内的网络数据包，将匹配的vxlan tunnel号的网络包剥离vxlan号，添加network-10.0.1.4的vlan tag，并将处理后的数据包扔给br-int；
- （2）接着，br-int将数据包交给linux bridge；
- （3）然后，Linux Bridge根据安全组规则对network-10.0.1.4的数据包进行处理，将处理后的数据包通过tap设备交给host02。


{% img /images/understanding_neutron_different_subnet.PNG %}
