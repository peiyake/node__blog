---
title: VPP 二层桥接测试
type: vpp
description: CentOS Linux release 7.5.1804 (Core),VPP-tag-19.089
date: 2020-06-22 19:10:42
---

# VPP 二层桥接测试

## 系统环境

* CentOS Linux release 7.5.1804 (Core)
* VPP-stable/1908
* 工作路径 /root/vpp

## 组网

![二层桥接组网](/images/l2_bridge.png)

## 运行环境准备

1. 启动VPP

```
[root@localhost vpp]#./build-root/build-vpp_debug-native/vpp/bin/vpp -c /root/vpp/startup.conf
```

2. 启动vppctl

```
[root@localhost vpp]# ./build-root/build-vpp_debug-native/vpp/bin/vppctl
```

## 测试过程

1. vppctl 配置

```
DBGvpp# show interface 
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1     down         9000/0/0/0     
vpp1                              2     down         9000/0/0/0   
DBGvpp# set interface state vpp0 up
DBGvpp# set interface state vpp1 up
DBGvpp# show interface 
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1      up          9000/0/0/0     
vpp1                              2      up          9000/0/0/0
DBGvpp# set interface l2 bridge vpp0 1
DBGvpp# set interface l2 bridge vpp1 1
DBGvpp# show bridge-domain 1
  BD-ID   Index   BSN  Age(min)  Learning  U-Forwrd   UU-Flood   Flooding  ARP-Term  arp-ufwd   BVI-Intf 
    1       1      0     off        on        on       flood        on       off       off        N/A    
DBGvpp# show bridge-domain 1 detail
  BD-ID   Index   BSN  Age(min)  Learning  U-Forwrd   UU-Flood   Flooding  ARP-Term  arp-ufwd   BVI-Intf 
    1       1      0     off        on        on       flood        on       off       off        N/A    

           Interface           If-idx ISN  SHG  BVI  TxFlood        VLAN-Tag-Rewrite       
             vpp0                1     1    0    -      *                 none             
             vpp1                2     1    0    -      *                 none 
```

2. vpp0 连接PC1，PC1配置静态IP `172.16.1.11/16`
3. vpp1 连接PC2，PC2配置静态IP `172.16.1.12/16`
4. PC1 ping  PC2 
```
[root@localhost ~]# ping 172.16.1.12
PING 172.16.1.12 (172.16.1.12) 56(84) bytes of data.
64 bytes from 172.16.1.12: icmp_seq=53 ttl=64 time=0.513 ms
64 bytes from 172.16.1.12: icmp_seq=54 ttl=64 time=0.850 ms
64 bytes from 172.16.1.12: icmp_seq=55 ttl=64 time=0.849 ms
64 bytes from 172.16.1.12: icmp_seq=56 ttl=64 time=0.609 ms
64 bytes from 172.16.1.12: icmp_seq=57 ttl=64 time=0.809 ms
```

5. 再次查看vppctl
```
DBGvpp# show interface 
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1      up          9000/0/0/0     rx packets                   121
                                                                    rx bytes                   10944
                                                                    tx packets                   344
                                                                    tx bytes                   33906
                                                                    tx-error                      46
vpp1                              2      up          9000/0/0/0     rx packets                   390
                                                                    rx bytes                   37748
                                                                    tx packets                   121
                                                                    tx bytes                   10944
                                                                    drops                         46
```