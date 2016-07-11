---
layout: post
title: "多机环境下虚机网络实现机制（一）"
date: 2016-07-11 22:47:34 +0800
comments: 
categories:   
---
## *** 虚机的网络配置详解***
该部分主要介绍在多机环境下，虚拟机的网络是如何创建和配置的，具体来说就是各个节点之间的相互配合，包括compute node的nova compute的port绑定，以及controller node的neutron server的调用以及实现过程。

## 

