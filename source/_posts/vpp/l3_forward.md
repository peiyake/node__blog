---
title: PP三层转发测试
type: vpp
description: CentOS Linux release 7.5.1804 (Core),VPP-tag-19.089
date: 2020-06-22 19:10:42
---

## 系统环境

* CentOS Linux release 7.5.1804 (Core)
* VPP-stable/1908
* 工作路径 /root/vpp

## 组网

![三层转发组网](/images/l3_forward.png)

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
DBGvpp# show interface                              // 查看接口                          
            Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1     down         9000/0/0/0     
vpp1                              2     down         9000/0/0/0

DBGvpp# set interface state vpp0 up                 // 启动接口
DBGvpp# set interface state vpp1 up
DBGvpp# show interface             
            Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1      up          9000/0/0/0     
vpp1                              2      up          9000/0/0/0 

DBGvpp# set interface ip address vpp0 172.16.1.1/16 // 接口配置IP
DBGvpp# set interface ip address vpp1 172.17.1.1/16
DBGvpp# show interfaces addr                        // 查看接口IP
local0 (dn):
vpp0 (up):
L3 172.16.1.1/16
vpp1 (up):
L3 172.17.1.1/16

```

2. vpp0 连接PC1，PC1配置静态IP `172.16.1.11/16`
3. vpp1 连接PC2，PC2配置静态IP `172.17.1.11/16`
4. PC1 ping  PC2
5. 注意事项
   * 由于我的PC1有多个接口，我这里需要配置一个静态路由指向 `172.17.0.0/16` 网段从 eth3 出去。
   * PC1 和 PC2 的防火墙记得关闭

```
[root@localhost ~]route add -net 172.17.0.0 netmask 255.255.0.0 gw 172.16.1.1
[root@localhost ~]# ifconfig eth3
eth3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.1.11  netmask 255.255.0.0  broadcast 172.16.255.255
        inet6 fe80::2a51:32ff:fe0e:4e8e  prefixlen 64  scopeid 0x20<link>
        ether 28:51:32:0e:4e:8e  txqueuelen 1000  (Ethernet)
        RX packets 11225  bytes 1132564 (1.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11748  bytes 1185688 (1.1 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 19  memory 0xdfb00000-dfb20000  

[root@localhost ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.60.1.254     0.0.0.0         UG    0      0        0 eth0
10.60.0.0       0.0.0.0         255.255.0.0     U     0      0        0 eth0
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth3
172.17.0.0      172.16.1.1      255.255.0.0     UG    0      0        0 eth3
[root@localhost ~]# ping 172.17.1.11
PING 172.17.1.11 (172.17.1.11) 56(84) bytes of data.
64 bytes from 172.17.1.11: icmp_seq=1 ttl=63 time=0.641 ms
64 bytes from 172.17.1.11: icmp_seq=2 ttl=63 time=0.623 ms
64 bytes from 172.17.1.11: icmp_seq=3 ttl=63 time=0.639 ms
64 bytes from 172.17.1.11: icmp_seq=4 ttl=63 time=0.605 ms
64 bytes from 172.17.1.11: icmp_seq=5 ttl=63 time=0.589 ms
^C
--- 172.17.1.11 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4000ms
rtt min/avg/max/mdev = 0.589/0.619/0.641/0.029 ms
```

