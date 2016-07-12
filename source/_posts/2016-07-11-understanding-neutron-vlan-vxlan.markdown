---
layout: post
title: "多机环境下虚机网络实现机制（二）"
date: 2016-07-11 22:47:53 +0800
comments: true
categories: 
---

# ---  ***VLAN和VXLAN流表实现机制***
----------------------------------------------

## 1、neutron中的网络概念

### 1.1 网络的命名空间
   在 neutron中，网络名字空间可以被认为是隔离的拥有单独网络栈（网卡、路由转发表、iptables）的环境，只有拥有同样网络名字空间的设备,才能看到彼此。namespace的好处就是可以隔离网络设备和服务，在neutron中，使用ip netns来列举出所有的命名空间。

```
[root@host41 ~]# ip netns
qrouter-ec5e63fc-c5a4-4925-9767-154583432d21
qdhcp-ce0869f1-b055-4914-85f9-9398bad6de7c
qdhcp-3a14c59d-37f1-42a7-a135-29466583d3e2
qrouter-af687a92-d701-4372-8bb2-b922aee494da
qdhcp-6fc3d619-81d9-4c1a-9080-16fd12321735
qdhcp-8c4abc35-ac62-480d-92ce-ec056b4af3d0
qdhcp-8afb20c6-8c55-41fd-9b9d-bce61aaa8bd7
qdhcp-2a70f3da-d722-4cc4-85e8-9b4e19366106
qdhcp-cdbead44-deb9-42a1-901e-53db3bb6e80f
qdhcp-cf4fee75-358b-46ba-bb88-525aa0be2e36
qdhcp-2c9471bf-e5f7-447e-82d4-55fa29fe9058
qrouter-965d88e2-6d5a-4be3-b8d7-32c0a247245f
qrouter-54aa7fd1-f3d5-4f0e-9bbd-2e8be54b7228
qrouter-03347f01-d551-46ac-944c-2ed83e79362f

```
以qdhcp开头的是DHCP服务的命名空间，以qrouter开头的是路由器服务的命名空间。
为了方便的使用namespace，可以使用*ip netns exec namespaceID commond*来对命名空间进行操作，如下所示：


```
[root@host41 ~]# ip netns exec qrouter-ec5e63fc-c5a4-4925-9767-154583432d21 route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.21.11.1     0.0.0.0         UG    0      0        0 qg-4306bf11-af
10.0.1.0        0.0.0.0         255.255.255.0   U     0      0        0 qr-4d5301c0-7c
10.1.0.0        0.0.0.0         255.255.255.0   U     0      0        0 qr-c4a064a6-c5
172.21.11.0     0.0.0.0         255.255.255.0   U     0      0        0 qg-4306bf11-af
```

### 1.2 Security Group的概念

Security group是一些规则的集合，用来对虚拟机的访问流量进行控制，起到虚拟防火墙的作用，在启动虚拟机的实例时，可以将一个或者多个安全组与该实例关联，在nova中默认的存在一个default的安全组，可以在里面添加相应的规则。

> 在具体的实现上，底层使用的iptables，由于OVS并不支持iptables规则的tap设备，所以在compute节点使用Linux bridge进行实现。


要查看相应的安全组的规则，需要在compute节点上进行查看有三种方式，INPUT（进入），OUTPUT（出去）和FORWARD（转发）。
以INPUT为例，安全组是对应的bridge来说的，为了区分不同的安全组，可以通过查找vm上的tap设备来找到对应的安全组规则，这样就可以实现不同虚机和不同规则的安全组连接。
下面来查找INPUT的iptables规则，可以看到和安全组相关的规则都被重定向到neutron-openvswi-INPUT。


```
[root@host42 ~]# iptables --line-numbers -vnL INPUT
Chain INPUT (policy ACCEPT 18860 packets, 26M bytes)
num   pkts bytes target     prot opt in     out     source               destination         
1    8542K 4539M neutron-openvswi-INPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
2        0     0 ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
3        0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53
4        0     0 ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:67
5        0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:67
```

继续查看neutron-openvswi-INPUT的iptables规则表，可以看到几个不同的tap设备对应着不同的neutron-openvswi规则，这主要是因为在该计算节点已经创建了几台不同的虚拟，而这tap设备就是与虚机的bridge所对应。

```
[root@host42 ~]# iptables --line-numbers -vnL neutron-openvswi-INPUT
Chain neutron-openvswi-INPUT (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 neutron-openvswi-o0920e155-8  all  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in tap0920e155-86 --physdev-is-bridged /* Direct incoming traffic from VM to the security group chain. */
2        0     0 neutron-openvswi-o101e9de7-b  all  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in tap101e9de7-be --physdev-is-bridged /* Direct incoming traffic from VM to the security group chain. */
3        0     0 neutron-openvswi-o81b5c22c-8  all  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in tap81b5c22c-86 --physdev-is-bridged /* Direct incoming traffic from VM to the security group chain. */
4        0     0 neutron-openvswi-oa1c05cb3-9  all  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in tapa1c05cb3-90 --physdev-is-bridged /* Direct incoming traffic from VM to the security group chain. */
```

在network01网络创建的host01的hostnameID为11f1d000-615b-4cee-b760-baef886b6637，那么找到所对应的tap设备后，找出其iptables的规则，如下所示

```
 [root@host42 11f1d000-615b-4cee-b760-baef886b6637]# iptables --line-numbers -vnL neutron-openvswi-INPUT|grep tapa1c05cb3-90
4        0     0 neutron-openvswi-oa1c05cb3-9  all  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in tapa1c05cb3-90 --physdev-is-bridged /* Direct incoming traffic from VM to the security group chain. */
```

可以看出规则被重定向都neutron-openvswi-oa1c05cb3-9，接着查看neutron-openvswi-oa1c05cb3-9的iptables规则，可以看到从虚机发送到DHCP的请求直接通过，而其他的请求则会转到neutron-openvswi-sa1c05cb3-9。

```
[root@host42 11f1d000-615b-4cee-b760-baef886b6637]# iptables --line-numbers -vnL neutron-openvswi-oa1c05cb3-9
Chain neutron-openvswi-oa1c05cb3-9 (2 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        2   648 RETURN     udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp spt:68 udp dpt:67 /* Allow DHCP client traffic. */
2       27  1710 neutron-openvswi-sa1c05cb3-9  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
3        0     0 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            udp spt:67 udp dpt:68 /* Prevent DHCP Spoofing by VM. */
4        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED /* Direct packets associated with a known session to the RETURN chain. */
5       27  1710 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           
6        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            state INVALID /* Drop packets that appear related to an existing connection (e.g. TCP ACK/FIN) but do not have an entry in conntrack. */
7        0     0 neutron-openvswi-sg-fallback  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* Send unmatched traffic to the fallback chain. */
```

接着查看neutron-openvswi-sa1c05cb3-9的规则，这条chain主要检查从vm发出来的网包，是否是openstack所分配的IP和MAC，如果不匹配，则禁止通过。这将防止利用vm上进行一些伪装地址的攻击。

?????????????

OUTPUT和FORWARD的规则查看方法和INPUT的类似，可以依据上述方法进行安全组的学习

## 2、计算节点的流量转发机制

## 3、控制节点的流量转发机制






