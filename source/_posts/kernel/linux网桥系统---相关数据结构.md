---
title: linux网桥系统---相关数据结构
type: kernel
description: linux网桥是一个虚拟网络设备，网桥下可以挂接实接口、vlan虚接口，网桥可实现实现二层转发功能
date: 2020-09-23 10:47:00
---

linux网桥（bridge）相当于一个虚拟二层交换机。 当创建一个bridge时，linux创建一个虚拟接口（net_device），这个虚拟bridge下可以挂接实接口、VLAN虚接口，形成一个多接口的二层交换系统。其内部维护一个高速fdb表（端口、mac、vlan关系表），从而实现完整的二层交换功能，


## bridge系统相关数据结构

* `struct net_device`   内核中每个NIC接口（包括虚接口）都对应一个net_device结构，并组成一个list
* `struct net_bridge`   如果net_device是一个网桥虚接口，那么在创建它的net_device结构时，会在结构尾部追加一块数据区域，用来存放net_bridge结构，用来描述一个网桥设备的相关信息。
* `struct net_bridge_port`  每个net_bridge可以添加多个接口，这些接口使用net_bridge_port描述，在net_bridge 中port_list用来指向该bridge下net_bridge_port组成的链表
* `struct net_bridge_fdb_entry` 每个net_bridge都维护一个二层转发表，这是一个HASH表，net_bridge的成员 hlist_head 指向这个这个HASH表

## 网桥数据结构关系图


![网桥系统数据结构关系图](/images/bridge_struct.png)

