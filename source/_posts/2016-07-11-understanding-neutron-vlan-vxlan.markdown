---
layout: post
title: "多机环境下虚机网络实现机制（二）"
date: 2016-07-11 22:47:53 +0800
comments: true
categories: 
---

# ---  ***VLAN和VXLAN流表实现机制***
----------------------------------------------

{% img /images/understanding_neutron_horizon.PNG %}

## 1、neutron中的网络概念

### 1.1 网络的命名空间
   在 neutron中，网络名字空间可以被认为是隔离的拥有单独网络栈（网卡、路由转发表、iptables）的环境，只有拥有同样网络名字空间的设备,才能看到彼此。namespace的好处就是可以隔离网络设备和服务，在neutron中，使用ip netns来列举出所有的命名空间。

```
[root@host41 ~]# ip netns
qrouter-ec5e63fc-c5a4-4925-9767-154583432d21
qdhcp-ce0869f1-b055-4914-85f9-9398bad6de7c
qdhcp-3a14c59d-37f1-42a7-a135-29466583d3e2

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


{% img /images/understanding_neutron_security_group.PNG %}


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

OUTPUT和FORWARD的规则查看方法和INPUT的类似，可以依据上述方法进行安全组的学习

## 2、计算节点的流量转发机制

### 2.1 网桥介绍




在多机环境下，分别在两个计算节点上部署虚机，网桥整体组成如下所示：


{% img /images/understanding_neutron_compute_node_bridge.PNG %}


在compute节点上主要有两种网桥，一种是linux bridge，另外一种就是OVS网桥。

（1）**linux网桥**
主要用来绑定security group的iptables规则，与实际的转发并没有直接的关系，具体查看语法如下：

```
[root@host42 ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
qbr0920e155-86          8000.0a6f338bd56b       no              qvb0920e155-86
                                                        tap0920e155-86
qbr101e9de7-be          8000.ba2a0fd52af0       no              qvb101e9de7-be
                                                        tap101e9de7-be
qbr81b5c22c-86          8000.166b1b7bbfb7       no              qvb81b5c22c-86
                                                        tap81b5c22c-86
qbra1c05cb3-90          8000.7aeb0ca3b0bc       no              qvba1c05cb3-90
                                                        tapa1c05cb3-90
virbr0          8000.525400e1a3da       yes             virbr0-nic
```


（2）**OVS网桥**
在compute节点上主要包括两种，br-int和br-tun，br-int主要进行vlan标签的设置和转发，而br-tun主要用于vxlan标签的设置和转发流量。

```
[root@host42 ~]# ovs-vsctl show
eff53d4c-55b1-4a65-9889-c4480580d456
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-ac150b2b"
            Interface "vxlan-ac150b2b"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.42", out_key=flow, remote_ip="172.21.11.43"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-ac150b2c"
            Interface "vxlan-ac150b2c"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.42", out_key=flow, remote_ip="172.21.11.44"}
        Port "vxlan-ac150b29"
            Interface "vxlan-ac150b29"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.42", out_key=flow, remote_ip="172.21.11.41"}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-int
        fail_mode: secure
        Port "qvoa1c05cb3-90"
            tag: 6
            Interface "qvoa1c05cb3-90"
        Port "qvo81b5c22c-86"
            tag: 8
            Interface "qvo81b5c22c-86"
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qvo0920e155-86"
            tag: 2
            Interface "qvo0920e155-86"
        Port "qvo101e9de7-be"
            tag: 2
            Interface "qvo101e9de7-be"
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.4.0"



```


### 2.2 Vlan标签设置以及流表转发机制

在ovs中，通过网桥br-int来对vlan和mac进行转发，主要是作为一个二层交换机使用，所包含的接口主要包括两类：
- linux bridge过来的qvo-xxx
- 往外的 patch-tun 接口，连接到 br-tun 网桥

这样就可以通过qvo-xxx 接口上为每个经过的网络分配一个内部 vlan的tag，如果在同一个neutron网络里启动了多台虚机，那么它们的tag都是一样的，如果是在不同的网络，那么vlan tag就会不一样。在多机环境中，分别在网络10.0.1.0/24和10.1.0.0/24里创建了两台虚机host01和host03，那么对应的tag分别为6和8。



```
    Bridge br-int
        fail_mode: secure
        Port "qvoa1c05cb3-90"      //host01所在网络对应的端口，vlan号为6
            tag: 6
            Interface "qvoa1c05cb3-90"
        Port "qvo81b5c22c-86"      //host03所在网络对应的端口，vlan号为8
            tag: 8
            Interface "qvo81b5c22c-86"
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qvo0920e155-86"
            tag: 2
            Interface "qvo0920e155-86"
        Port "qvo101e9de7-be"
            tag: 2
            Interface "qvo101e9de7-be"
        Port br-int
            Interface br-int
                type: internal
```
接着查看两台虚机所在网络的端口号，可以看到对应的port number分别为15和17，查看端口号主要为了方便读取流表的转换规则。
```
[root@host42 ~]# ovs-ofctl show br-int
OFPT_FEATURES_REPLY (xid=0x2): dpid:0000fa2a90a1c942
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 3(patch-tun): addr:c6:0c:c4:5b:c9:bc
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 5(qvo101e9de7-be): addr:be:27:b5:da:bd:7a
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 11(qvo0920e155-86): addr:0e:ea:1b:6d:d9:91
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 15(qvoa1c05cb3-90): addr:4e:b1:8d:af:52:8f
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 17(qvo81b5c22c-86): addr:ca:ef:7d:cb:50:ee
     config:     0
     state:      0
     current:    10GB-FD COPPER
     speed: 10000 Mbps now, 0 Mbps max
 LOCAL(br-int): addr:fa:2a:90:a1:c9:42
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
```



知道各自的port number后，对br-int的流表进行查看，找到port=15和port=17的流表规则如下所示：

```
[root@host42 ~]# ovs-ofctl dump-flows br-int
 cookie=0xb05867d95f1c0bc0, duration=7330.618s, table=0, n_packets=0, n_bytes=0, idle_age=7330, priority=10,icmp6,in_port=15,icmp_type=136 actions=resubmit(,24)
 cookie=0xb05867d95f1c0bc0, duration=7330.581s, table=0, n_packets=0, n_bytes=0, idle_age=7330, priority=10,icmp6,in_port=17,icmp_type=136 actions=resubmit(,24)
  cookie=0xb05867d95f1c0bc0, duration=7330.603s, table=0, n_packets=6, n_bytes=252, idle_age=5203, priority=10,arp,in_port=15 actions=resubmit(,24)
 cookie=0xb05867d95f1c0bc0, duration=7330.568s, table=0, n_packets=4, n_bytes=168, idle_age=5202, priority=10,arp,in_port=17 actions=resubmit(,24)
  cookie=0xb05867d95f1c0bc0, duration=7330.627s, table=24, n_packets=0, n_bytes=0, idle_age=7330, priority=2,icmp6,in_port=15,icmp_type=136,nd_target=fe80::f816:3eff:fe54:74c7 actions=NORMAL
 cookie=0xb05867d95f1c0bc0, duration=7330.588s, table=24, n_packets=0, n_bytes=0, idle_age=7330, priority=2,icmp6,in_port=17,icmp_type=136,nd_target=fe80::f816:3eff:fe1e:125e actions=NORMAL
  cookie=0xb05867d95f1c0bc0, duration=7330.610s, table=24, n_packets=6, n_bytes=252, idle_age=5203, priority=2,arp,in_port=15,arp_spa=10.0.1.3 actions=NORMAL
 cookie=0xb05867d95f1c0bc0, duration=7330.574s, table=24, n_packets=4, n_bytes=168, idle_age=5202, priority=2,arp,in_port=17,arp_spa=10.1.0.4 actions=NORMAL
```
先根据priority的优先级找到table=0的flow规则，不管是ARP还是icmp都交给table=24来进行处理。




- **vm发出ARP报文** ：
以host01的虚拟网络为例，flow entry遇到从15口进入的arp，并且arp来源（arp_spa）是10.0.1.3的，让数据包通过（normal）。
```
  cookie=0xb05867d95f1c0bc0, duration=7330.610s, table=24, n_packets=6, n_bytes=252, idle_age=5203, priority=2,arp,in_port=15,arp_spa=10.0.1.3 actions=NORMAL
 cookie=0xb05867d95f1c0bc0, duration=7330.574s, table=24, n_packets=4, n_bytes=168, idle_age=5202, priority=2,arp,in_port=17,arp_spa=10.1.0.4 actions=NORMAL
```
- **vm发出ICMP报文**
以host01的虚拟网络为例，flow entry遇到从15口进入的，目标地址为fe80::f816:3eff:fe1e:125e的icmp包就可以通过。

```
  cookie=0xb05867d95f1c0bc0, duration=7330.627s, table=24, n_packets=0, n_bytes=0, idle_age=7330, priority=2,icmp6,in_port=15,icmp_type=136,nd_target=fe80::f816:3eff:fe54:74c7 actions=NORMAL
 cookie=0xb05867d95f1c0bc0, duration=7330.588s, table=24, n_packets=0, n_bytes=0, idle_age=7330, priority=2,icmp6,in_port=17,icmp_type=136,nd_target=fe80::f816:3eff:fe1e:125e actions=NORMAL
```


### 2.2 Vxlan标签设置以及流表转发机制

neutron中的vxlan的设置规则相对来说就更加复杂一些，主要利用br-tun网桥来实现，该网桥主要根据自身的规则将合适的网包经过 VXLAN 隧道送出去，可以从两个维度来进行思考：

- 从vm内部过来的数据包进行规则的设置，数据包带着正确的vlan tag过来，从正确的tunnel扔出去；
- 数据包从外面public network带着正确的vxlan tunnel号过来，要修改到对应的内部的vlan tag扔到里面去。

以该多机环境为例，考虑创建的虚拟机在不同的计算节点，由于host01和host02分别创建在节点172.21.11.42和节点172.21.11.43上，所以会有不同的vxlan tunnelID，以host01所在的172.21.11.42的compute节点为例，查看br-tun网桥的信息。



```
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-ac150b2b"
            Interface "vxlan-ac150b2b"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.42", out_key=flow, remote_ip="172.21.11.43"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "vxlan-ac150b2c"
            Interface "vxlan-ac150b2c"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.42", out_key=flow, remote_ip="172.21.11.44"}
        Port "vxlan-ac150b29"
            Interface "vxlan-ac150b29"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.42", out_key=flow, remote_ip="172.21.11.41"}
        Port br-tun
            Interface br-tun
                type: internal
```
vxlan-ac150b29就是计算节点（42）和控制节点（41）在发送数据包时候的vxlan tunnel端口，patch-int端口主要连通br-int上的patch-tun端口。
下面来具体查看br-tun的流表规则：



```
[root@host42 ~]# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
 cookie=0xb05867d95f1c0bc0, duration=1195937.201s, table=0, n_packets=14103, n_bytes=1150634, idle_age=194, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0xb05867d95f1c0bc0, duration=1195936.854s, table=0, n_packets=15517, n_bytes=1237168, idle_age=194, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)
 cookie=0xb05867d95f1c0bc0, duration=1195918.788s, table=0, n_packets=16807, n_bytes=1494619, idle_age=173, hard_age=65534, priority=1,in_port=3 actions=resubmit(,4)
 cookie=0xb05867d95f1c0bc0, duration=1195913.601s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,in_port=4 actions=resubmit(,4)
 cookie=0xb05867d95f1c0bc0, duration=1195937.201s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xb05867d95f1c0bc0, duration=1195937.200s, table=2, n_packets=12884, n_bytes=1078184, idle_age=194, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0xb05867d95f1c0bc0, duration=1195937.200s, table=2, n_packets=1219, n_bytes=72450, idle_age=199, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0xb05867d95f1c0bc0, duration=1195937.199s, table=3, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xb05867d95f1c0bc0, duration=1151786.751s, table=4, n_packets=567, n_bytes=70654, idle_age=18902, hard_age=65534, priority=1,tun_id=0x1003a actions=mod_vlan_vid:2,resubmit(,10)
 cookie=0xb05867d95f1c0bc0, duration=86611.109s, table=4, n_packets=71, n_bytes=6213, idle_age=173, hard_age=65534, priority=1,tun_id=0x10049 actions=mod_vlan_vid:6,resubmit(,10)
 cookie=0xb05867d95f1c0bc0, duration=77340.369s, table=4, n_packets=58, n_bytes=4496, idle_age=10412, hard_age=65534, priority=1,tun_id=0x1003d actions=mod_vlan_vid:8,resubmit(,10)
 cookie=0xb05867d95f1c0bc0, duration=1195937.198s, table=4, n_packets=18446, n_bytes=1576993, idle_age=471, hard_age=65534, priority=0 actions=drop
 cookie=0xb05867d95f1c0bc0, duration=1195937.198s, table=6, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xb05867d95f1c0bc0, duration=1195937.197s, table=10, n_packets=13878, n_bytes=1154794, idle_age=173, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0xb05867d95f1c0bc0,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
 cookie=0xb05867d95f1c0bc0, duration=199.589s, table=20, n_packets=2, n_bytes=374, hard_timeout=300, idle_age=194, hard_age=194, priority=1,vlan_tci=0x0006/0x0fff,dl_dst=fa:16:3e:fd:2e:d2 actions=load:0->NXM_OF_VLAN_TCI[],load:0x10049->NXM_NX_TUN_ID[],output:2
 cookie=0xb05867d95f1c0bc0, duration=173.176s, table=20, n_packets=0, n_bytes=0, hard_timeout=300, idle_age=173, priority=1,vlan_tci=0x0006/0x0fff,dl_dst=fa:16:3e:11:8d:9d actions=load:0->NXM_OF_VLAN_TCI[],load:0x10049->NXM_NX_TUN_ID[],output:3
 cookie=0xb05867d95f1c0bc0, duration=1195937.197s, table=20, n_packets=1, n_bytes=98, idle_age=10418, hard_age=65534, priority=0 actions=resubmit(,22)
 cookie=0xb05867d95f1c0bc0, duration=1151786.757s, table=22, n_packets=136, n_bytes=10890, idle_age=19085, hard_age=65534, dl_vlan=2 actions=strip_vlan,set_tunnel:0x1003a,output:2,output:3,output:4
 cookie=0xb05867d95f1c0bc0, duration=86611.115s, table=22, n_packets=19, n_bytes=1822, idle_age=199, hard_age=65534, dl_vlan=6 actions=strip_vlan,set_tunnel:0x10049,output:2,output:3,output:4
 cookie=0xb05867d95f1c0bc0, duration=77340.375s, table=22, n_packets=16, n_bytes=1592, idle_age=10418, hard_age=65534, dl_vlan=8 actions=strip_vlan,set_tunnel:0x1003d,output:2,output:3,output:4
 cookie=0xb05867d95f1c0bc0, duration=1195937.185s, table=22, n_packets=91, n_bytes=7490, idle_age=65534, hard_age=65534, priority=0 actions=drop
```


虽然表面上比较复杂，先要明确数据包出口的方向，由于在br-tun上存在两个方向的数据包转发，从内部vm过来的还有外部过来的，在该compute节点（172.21.11.42）上，br-tun的port值为1的端口表示从内部过来的数据包转发，port值为2表示数据包从外部进来的，port值为3表示数据包是从另外一个compute节点上的vm过来的。

上述的转发逻辑可以归结如下所示：


{% img /images/understanding_neutron_compute_node_tables.PNG %}


### **table=0**
主要关注in_port=1，2和3的流表规则，从 1 端口（patch-int）进来的内部vm的网络包，扔给表 2 处理，从 2 端口（vxlan-ac150b29）外面进来的网包，扔给表 4 处理，从3端口（vxlan-ac150b2b）另外一个compute节点进来的数据包，扔给表4处理，其他的做drop处理。


```
 cookie=0xb05867d95f1c0bc0, duration=1198890.122s, table=0, n_packets=14103, n_bytes=1150634, idle_age=3147, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0xb05867d95f1c0bc0, duration=1198889.775s, table=0, n_packets=15519, n_bytes=1237300, idle_age=347, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)
 cookie=0xb05867d95f1c0bc0, duration=1198871.709s, table=0, n_packets=16807, n_bytes=1494619, idle_age=3126, hard_age=65534, priority=1,in_port=3 actions=resubmit(,4)
 cookie=0xb05867d95f1c0bc0, duration=1198866.522s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,in_port=4 actions=resubmit(,4)
 cookie=0xb05867d95f1c0bc0, duration=1198890.122s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
```
### **table=2**
从里面vm出来的单播数据包扔给表20处理，多播和广播的数据包交给表22处理。
```
 cookie=0xb05867d95f1c0bc0, duration=1198890.121s, table=2, n_packets=12884, n_bytes=1078184, idle_age=3147, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0xb05867d95f1c0bc0, duration=1198890.121s, table=2, n_packets=1219, n_bytes=72450, idle_age=3152, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
```

### **table=3**
经过表3处理的数据包全部扔掉
```
cookie=0xb05867d95f1c0bc0, duration=1198890.120s, table=3, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
```

### **table=4**
主要处理外部过来的数据包，这里包括网络节点来的数据包以及其他compute节点过来的数据包。

- 匹配vxlan tunnel号为0x1003a的数据包，添加vlan=2的tag，扔给表10进行学习，再扔给br-int
- 匹配vxlan tunnel号为0x10049的数据包，添加vlan=6的tag，扔给表10进行学习，再扔给br-int
- 匹配vxlan tunnel号为0x1003d的数据包，添加vlan=8的tag，扔给表10进行学习，再扔给br-int
- 其它不匹配的直接扔掉
```
 cookie=0xb05867d95f1c0bc0, duration=1154739.672s, table=4, n_packets=567, n_bytes=70654, idle_age=21855, hard_age=65534, priority=1,tun_id=0x1003a actions=mod_vlan_vid:2,resubmit(,10)
 cookie=0xb05867d95f1c0bc0, duration=89564.030s, table=4, n_packets=71, n_bytes=6213, idle_age=3126, hard_age=65534, priority=1,tun_id=0x10049 actions=mod_vlan_vid:6,resubmit(,10)
 cookie=0xb05867d95f1c0bc0, duration=80293.290s, table=4, n_packets=58, n_bytes=4496, idle_age=13365, hard_age=65534, priority=1,tun_id=0x1003d actions=mod_vlan_vid:8,resubmit(,10)
 cookie=0xb05867d95f1c0bc0, duration=1198890.119s, table=4, n_packets=18448, n_bytes=1577125, idle_age=347, hard_age=65534, priority=0 actions=drop
```
### **table=6**
经过表6的数据包直接扔掉处理。
```
 cookie=0xb05867d95f1c0bc0, duration=1198890.119s, table=6, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
```

### **table=10**
表10的作用是用来学习network节点或者其它compute节点过来的外部数据包，主要是vxlan tunnel进来的包，学习结束后往表20中添加对返程包的正常转发规则，然后扔给br-int。

table=20 说明是修改表 20 中的规则，后面是添加的规则内容；

- NXM_OF_VLAN_TCI[0..11]，匹配跟当前流同样的 VLAN 头，其中 NXM 是 Nicira Extensible Match 的缩写；
- NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[]，包的目的 mac 跟当前流的源 mac 匹配；
- load:0->NXM_OF_VLAN_TCI[]，将 vlan 号改为 0；
- load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[]，将 tunnel 号修改为当前的 tunnel 号；
- output:NXM_OF_IN_PORT[]，从当前入口发出。

```
 cookie=0xb05867d95f1c0bc0, duration=1198890.118s, table=10, n_packets=13878, n_bytes=1154794, idle_age=3126, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0xb05867d95f1c0bc0,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
```


### **table=20**

将数据包直接交给表22处理

```
 cookie=0xb05867d95f1c0bc0, duration=1198890.118s, table=20, n_packets=1, n_bytes=98, idle_age=13371, hard_age=65534, priority=0 actions=resubmit(,22)
```

### **table=22**

- 对于vlan tag=2的数据包，去掉valn tag，添加vxlan tunnel号为0x1003a，扔给端口2,3,和4
- 对于vlan tag=6的数据包，去掉valn tag，添加vxlan tunnel号为0x10049，扔给端口2,3,和4
- 对于vlan tag=8的数据包，去掉valn tag，添加vxlan tunnel号为0x1003d，扔给端口2,3,和4
- 其它不匹配的就直接丢掉

```
 cookie=0xb05867d95f1c0bc0, duration=1154739.678s, table=22, n_packets=136, n_bytes=10890, idle_age=22038, hard_age=65534, dl_vlan=2 actions=strip_vlan,set_tunnel:0x1003a,output:2,output:3,output:4
 cookie=0xb05867d95f1c0bc0, duration=89564.036s, table=22, n_packets=19, n_bytes=1822, idle_age=3152, hard_age=65534, dl_vlan=6 actions=strip_vlan,set_tunnel:0x10049,output:2,output:3,output:4
 cookie=0xb05867d95f1c0bc0, duration=80293.296s, table=22, n_packets=16, n_bytes=1592, idle_age=13371, hard_age=65534, dl_vlan=8 actions=strip_vlan,set_tunnel:0x1003d,output:2,output:3,output:4
 cookie=0xb05867d95f1c0bc0, duration=1198890.106s, table=22, n_packets=91, n_bytes=7490, idle_age=65534, hard_age=65534, priority=0 actions=drop
```

## 3、控制节点的Vxlan流量转发机制

多机环境下，控制节点的网桥整体结构如下所示：

{% img /images/understanding_neutron_controller_node_bridge.PNG %}

在该多机环境下，网络节点和控制节点部署在一起，所以这里所说的控制节点的neutron网络服务实际上指的是网络节点所部署的neutron服务，包括DHCP服务和路由服务等。
该controller节点主要包括三种类型的网桥，br-int,br-tun,br-ex。

```
[root@host41 ~]# ovs-vsctl show
304203dc-5cc9-4998-9908-2077e4d0782e
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-ac150b2c"
            Interface "vxlan-ac150b2c"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.41", out_key=flow, remote_ip="172.21.11.44"}
        Port "vxlan-ac150b2a"
            Interface "vxlan-ac150b2a"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.41", out_key=flow, remote_ip="172.21.11.42"}
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-ac150b2b"
            Interface "vxlan-ac150b2b"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.41", out_key=flow, remote_ip="172.21.11.43"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    Bridge br-int
        fail_mode: secure
        Port "qr-99977504-99"
            tag: 5
            Interface "qr-99977504-99"
                type: internal
        Port "qr-ed283f80-4c"
            tag: 8
            Interface "qr-ed283f80-4c"
                type: internal
        Port int-br-vlan
            Interface int-br-vlan
                type: patch
                options: {peer=phy-br-vlan}
        Port "tapdba45207-16"
            tag: 2
            Interface "tapdba45207-16"
                type: internal
        Port "tap003dc11c-ff"
            tag: 8
            Interface "tap003dc11c-ff"
                type: internal
        Port "qr-3eac92eb-2c"
            tag: 3
            Interface "qr-3eac92eb-2c"
                type: internal
        Port "tap84987a29-8c"
            tag: 6
            Interface "tap84987a29-8c"
                type: internal
        Port "qr-86e75355-42"
            tag: 2
            Interface "qr-86e75355-42"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tapb82869ee-b7"
            tag: 14
            Interface "tapb82869ee-b7"
                type: internal
        Port "tap5b3887ae-96"
            tag: 5
            Interface "tap5b3887ae-96"
                type: internal
        Port "tapaeef9fcb-b4"
            tag: 3
            Interface "tapaeef9fcb-b4"
                type: internal
        Port "qr-c4a064a6-c5"
            tag: 15
            Interface "qr-c4a064a6-c5"
                type: internal
        Port "qr-4d5301c0-7c"
            tag: 14
            Interface "qr-4d5301c0-7c"
                type: internal
        Port "tap14a62a38-17"
            tag: 4
            Interface "tap14a62a38-17"
                type: internal
        Port "qr-a0ad0357-0e"
            tag: 2
            Interface "qr-a0ad0357-0e"
                type: internal
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port "qr-5726f55a-28"
            tag: 1
            Interface "qr-5726f55a-28"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "tap01676ab5-12"
            tag: 1
            Interface "tap01676ab5-12"
                type: internal
        Port "tap3dbb06d6-c8"
            tag: 15
            Interface "tap3dbb06d6-c8"
                type: internal
    Bridge br-ex
        Port "qg-69307f08-2f"
            Interface "qg-69307f08-2f"
                type: internal
        Port br-ex
            Interface br-ex
                type: internal
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port "qg-4306bf11-af"
            Interface "qg-4306bf11-af"
                type: internal
        Port "ens160"
            Interface "ens160"
        Port "qg-7a817caf-50"
            Interface "qg-7a817caf-50"
                type: internal
    ovs_version: "2.4.0"
```

### 3.1 vxlan标签设置以及流表转发机制



类似于计算节点的VXLAN tunnel，controller节点的VXALN规则也是通过br-tun网桥来实现的，该网桥主要根据自身的规则将合适的网包经过 VXLAN 隧道送出去，可以从两个维度来进行思考：

- 从vm内部过来的数据包进行规则的设置，数据包带着正确的vlan tag过来，从正确的tunnel扔出去；
- 数据包从外面public network带着正确的vxlan tunnel号过来，要修改到对应的内部的vlan tag扔到里面去。

以多机环境中172.21.11.41上的controller节点为例，查看br-tun网桥的信息。可以看到不同的VXLAN tunnel号对应不同的网络连接。

```
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-ac150b2c"
            Interface "vxlan-ac150b2c"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.41", out_key=flow, remote_ip="172.21.11.44"}
        Port "vxlan-ac150b2a"
            Interface "vxlan-ac150b2a"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.41", out_key=flow, remote_ip="172.21.11.42"}
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-ac150b2b"
            Interface "vxlan-ac150b2b"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="172.21.11.41", out_key=flow, remote_ip="172.21.11.43"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
```
下面来具体分析br-tun网桥上的流表信息，分析方法和之前计算节点上的br-tun网桥基本一致，具体流表信息如下所示：

```
[root@host41 ~]# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
 cookie=0xa08ff20bf20b5fae, duration=1640812.296s, table=0, n_packets=557112, n_bytes=280383648, idle_age=20, hard_age=65534, priority=1,in_port=1 actions=resubmit(,2)
 cookie=0xa08ff20bf20b5fae, duration=1204517.666s, table=0, n_packets=14004, n_bytes=1142528, idle_age=8775, hard_age=65534, priority=1,in_port=2 actions=resubmit(,4)
 cookie=0xa08ff20bf20b5fae, duration=1204499.592s, table=0, n_packets=502898, n_bytes=59684255, idle_age=20, hard_age=65534, priority=1,in_port=3 actions=resubmit(,4)
 cookie=0xa08ff20bf20b5fae, duration=1204494.411s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,in_port=4 actions=resubmit(,4)
 cookie=0xa08ff20bf20b5fae, duration=1640812.286s, table=0, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa08ff20bf20b5fae, duration=1640812.282s, table=2, n_packets=552989, n_bytes=280198986, idle_age=20, hard_age=65534, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0xa08ff20bf20b5fae, duration=1640812.282s, table=2, n_packets=4123, n_bytes=184662, idle_age=19451, hard_age=65534, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)
 cookie=0xa08ff20bf20b5fae, duration=1640812.282s, table=3, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa08ff20bf20b5fae, duration=1640748.151s, table=4, n_packets=747602, n_bytes=98745467, idle_age=20, hard_age=65534, priority=1,tun_id=0x10059 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1640748.106s, table=4, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,tun_id=0x10052 actions=mod_vlan_vid:2,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1640748.081s, table=4, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,tun_id=0x1005d actions=mod_vlan_vid:3,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1640748.055s, table=4, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,tun_id=0x1000c actions=mod_vlan_vid:4,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1640748.007s, table=4, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,tun_id=0x1001d actions=mod_vlan_vid:5,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1640747.977s, table=4, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=1,tun_id=0x10068 actions=mod_vlan_vid:6,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1213356.221s, table=4, n_packets=937, n_bytes=99354, idle_age=27478, hard_age=65534, priority=1,tun_id=0x1003a actions=mod_vlan_vid:8,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=95292.240s, table=4, n_packets=119, n_bytes=10570, idle_age=8748, hard_age=65534, priority=1,tun_id=0x10049 actions=mod_vlan_vid:14,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=95266.306s, table=4, n_packets=106, n_bytes=8850, idle_age=18993, hard_age=65534, priority=1,tun_id=0x1003d actions=mod_vlan_vid:15,resubmit(,10)
 cookie=0xa08ff20bf20b5fae, duration=1640812.281s, table=4, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa08ff20bf20b5fae, duration=1640812.281s, table=6, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa08ff20bf20b5fae, duration=1640812.281s, table=10, n_packets=764906, n_bytes=100191850, idle_age=20, hard_age=65534, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,cookie=0xa08ff20bf20b5fae,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
 cookie=0xa08ff20bf20b5fae, duration=1883.824s, table=20, n_packets=551, n_bytes=252408, hard_timeout=300, idle_age=20, hard_age=19, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:b4:00:8c actions=load:0->NXM_OF_VLAN_TCI[],load:0x10059->NXM_NX_TUN_ID[],output:3
 cookie=0xa08ff20bf20b5fae, duration=1640812.281s, table=20, n_packets=1968, n_bytes=161462, idle_age=2375, hard_age=65534, priority=0 actions=resubmit(,22)
 cookie=0xa08ff20bf20b5fae, duration=1213356.228s, table=22, n_packets=9, n_bytes=378, idle_age=65534, hard_age=65534, dl_vlan=8 actions=strip_vlan,set_tunnel:0x1003a,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640746.358s, table=22, n_packets=30, n_bytes=1316, idle_age=65534, hard_age=65534, dl_vlan=2 actions=strip_vlan,set_tunnel:0x10052,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640746.351s, table=22, n_packets=15, n_bytes=686, idle_age=65534, hard_age=65534, dl_vlan=3 actions=strip_vlan,set_tunnel:0x1005d,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640746.344s, table=22, n_packets=2, n_bytes=140, idle_age=65534, hard_age=65534, dl_vlan=4 actions=strip_vlan,set_tunnel:0x1000c,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640746.337s, table=22, n_packets=3186, n_bytes=161198, idle_age=2375, hard_age=65534, dl_vlan=1 actions=strip_vlan,set_tunnel:0x10059,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640746.331s, table=22, n_packets=15, n_bytes=686, idle_age=65534, hard_age=65534, dl_vlan=5 actions=strip_vlan,set_tunnel:0x1001d,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640746.324s, table=22, n_packets=2, n_bytes=140, idle_age=65534, hard_age=65534, dl_vlan=6 actions=strip_vlan,set_tunnel:0x10068,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=95292.249s, table=22, n_packets=16, n_bytes=1208, idle_age=65534, hard_age=65534, dl_vlan=14 actions=strip_vlan,set_tunnel:0x10049,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=95266.313s, table=22, n_packets=30, n_bytes=2200, idle_age=18999, hard_age=65534, dl_vlan=15 actions=strip_vlan,set_tunnel:0x1003d,output:3,output:2,output:4
 cookie=0xa08ff20bf20b5fae, duration=1640812.267s, table=22, n_packets=173, n_bytes=14398, idle_age=65534, hard_age=65534, priority=0 actions=drop
```
将上述的流表规则进行整理可以得到：


{% img /images/understanding_neutron_controller_node_tables.PNG %}



### 3.2 VLAN标签设置以及流表转发机制

在controller节点上，vlan tag的设置主要在br-int网桥上进行，作为一个正常的二层交换设备进行使用，只是根据vlan和mac进行数据包的转发。接口类型包括：

- tap-xxx，连接到网络 DHCP 服务的命名空间；
- qr-xxx，连接到路由服务的命名空间；
- patch-tun 接口，连接到 br-tun 网桥。


```
    Bridge br-int
        fail_mode: secure
        Port "qr-99977504-99"
            tag: 5
            Interface "qr-99977504-99"
                type: internal
        Port "qr-ed283f80-4c"
            tag: 8
            Interface "qr-ed283f80-4c"
                type: internal
        Port int-br-vlan
            Interface int-br-vlan
                type: patch
                options: {peer=phy-br-vlan}
        Port "tapdba45207-16"
            tag: 2
            Interface "tapdba45207-16"
                type: internal
        Port "tap003dc11c-ff"
            tag: 8
            Interface "tap003dc11c-ff"
                type: internal
        Port "qr-3eac92eb-2c"
            tag: 3
            Interface "qr-3eac92eb-2c"
                type: internal
        Port "tap84987a29-8c"
            tag: 6
            Interface "tap84987a29-8c"
                type: internal
        Port "qr-86e75355-42"
            tag: 2
            Interface "qr-86e75355-42"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tapb82869ee-b7"
            tag: 14
            Interface "tapb82869ee-b7"
                type: internal
        Port "tap5b3887ae-96"
            tag: 5
            Interface "tap5b3887ae-96"
                type: internal
        Port "tapaeef9fcb-b4"
            tag: 3
            Interface "tapaeef9fcb-b4"
                type: internal
        Port "qr-c4a064a6-c5"
            tag: 15
            Interface "qr-c4a064a6-c5"
                type: internal
        Port "qr-4d5301c0-7c"
            tag: 14
            Interface "qr-4d5301c0-7c"
                type: internal
        Port "tap14a62a38-17"
            tag: 4
            Interface "tap14a62a38-17"
                type: internal
        Port "qr-a0ad0357-0e"
            tag: 2
            Interface "qr-a0ad0357-0e"
                type: internal
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
        Port "qr-5726f55a-28"
            tag: 1
            Interface "qr-5726f55a-28"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "tap01676ab5-12"
            tag: 1
            Interface "tap01676ab5-12"
                type: internal
        Port "tap3dbb06d6-c8"
            tag: 15
            Interface "tap3dbb06d6-c8"
                type: internal
```
转发流表也比较简单，表0中进行正常的转发，其他的就直接丢弃。

```
[root@host41 ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0xa08ff20bf20b5fae, duration=1641332.633s, table=0, n_packets=628544, n_bytes=39425496, idle_age=0, hard_age=65534, priority=2,in_port=19 actions=drop
 cookie=0xa08ff20bf20b5fae, duration=1641332.769s, table=0, n_packets=1322424, n_bytes=380681779, idle_age=10, hard_age=65534, priority=0 actions=NORMAL
 cookie=0xa08ff20bf20b5fae, duration=1641332.762s, table=23, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
 cookie=0xa08ff20bf20b5fae, duration=1641332.755s, table=24, n_packets=0, n_bytes=0, idle_age=65534, hard_age=65534, priority=0 actions=drop
```
### 3.3 和外部网络通信实现机制




主要通过br-ex网桥和public network进行通信，
一个是挂载的物理接口上，如 ens160，网包将从这个接口发送到外部网络上。
另外一个是 qg-xxx 这样的接口，是连接到 router 服务的网络名字空间中，里面绑定一个路由器的外部 IP，作为 nAT 时候的地址，另外，网络中的 floating IP 也放在这个网络名字空间中。
```
    Bridge br-ex
        Port "qg-69307f08-2f"
            Interface "qg-69307f08-2f"
                type: internal
        Port br-ex
            Interface br-ex
                type: internal
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
        Port "qg-4306bf11-af"
            Interface "qg-4306bf11-af"
                type: internal
        Port "ens160"
            Interface "ens160"
        Port "qg-7a817caf-50"
            Interface "qg-7a817caf-50"
                type: internal
```
查看流表规则，基本就是正常的转发动作。

```
[root@host41 ~]# ovs-ofctl dump-flows br-ex
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=1641423.664s, table=0, n_packets=8606, n_bytes=541782, idle_age=120, hard_age=65534, priority=2,in_port=2 actions=drop
 cookie=0x0, duration=1641423.740s, table=0, n_packets=47181071, n_bytes=65307678044, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
```


### 3.4 DHCP服务实现机制

dhcp服务是通过dnsmasq进程（轻量级服务器，可以提供dns、dhcp、tftp等服务）来实现的，该进程绑定到dhcp名字空间中的br-int的接口上。可以查看相关的进程。



### 3.5 路由服务实现机制

neutron中的路由服务主要是提供跨子网间的网络通信，包括虚拟想访问外部网络等。路由服务主要利用namespace实现不同网络之间的隔离性。另外，router还可以实现tenant network和external network之间的网络连接，通过SNAT实现tenant network往external network的网络连通性（fixed IP），通过DNAT实现external network往tenant network的网络连通性（floating IP），

```
[root@host41 tmp]# ip netns
qrouter-ec5e63fc-c5a4-4925-9767-154583432d21
qdhcp-ce0869f1-b055-4914-85f9-9398bad6de7c
qdhcp-3a14c59d-37f1-42a7-a135-29466583d3e2
```

在该控制节点上创建的路由服务是qrouter-ec5e63fc-c5a4-4925-9767-154583432d21，于是可以进一步查看namespace中的信息。

```
[root@host41 tmp]# ip netns exec qrouter-ec5e63fc-c5a4-4925-9767-154583432d21 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
41: qr-4d5301c0-7c: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:12:54:fb brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.1/24 brd 10.0.1.255 scope global qr-4d5301c0-7c
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe12:54fb/64 scope link 
       valid_lft forever preferred_lft forever
42: qg-4306bf11-af: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:d3:a5:32 brd ff:ff:ff:ff:ff:ff
    inet 172.21.11.234/24 brd 172.21.11.255 scope global qg-4306bf11-af
       valid_lft forever preferred_lft forever
    inet 172.21.11.209/32 brd 172.21.11.209 scope global qg-4306bf11-af
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fed3:a532/64 scope link 
       valid_lft forever preferred_lft forever
43: qr-c4a064a6-c5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:fc:33:6f brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.1/24 brd 10.1.0.255 scope global qr-c4a064a6-c5
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fefc:336f/64 scope link 
       valid_lft forever preferred_lft forever
47: qr-622bfadb-9b: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:e7:8d:5d brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/24 brd 192.168.0.255 scope global qr-622bfadb-9b
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fee7:8d5d/64 scope link 
       valid_lft forever preferred_lft forever
```


其中，qg-4306bf11-af是连接br-ex网桥的端口，负责外面的数据包的转发和处理，qr-XXX用于连接br-int网桥，处理带有VLAN tag的网络包。

```
[root@host41 ~]# ip netns exec qrouter-ec5e63fc-c5a4-4925-9767-154583432d21 ip route
default via 172.21.11.1 dev qg-4306bf11-af 
10.0.1.0/24 dev qr-4d5301c0-7c  proto kernel  scope link  src 10.0.1.1 
10.1.0.0/24 dev qr-c4a064a6-c5  proto kernel  scope link  src 10.1.0.1 
172.21.11.0/24 dev qg-4306bf11-af  proto kernel  scope link  src 172.21.11.234 
192.168.0.0/24 dev qr-622bfadb-9b  proto kernel  scope link  src 192.168.0.1 
```

从上面规则中可以看出，从三个不同的qr-XXX端口进来的数据包都会通过qg-4306bf11-af发送到br-ex中去，从而到达外网。




综上所述，在多机环境下，计算节点和网络节点的整体网桥连接以及VLAN和VXLAN实现原理如下所示：


{% img /images/understanding_neutron_compute_node_and_controller_node_bridge.PNG %}














