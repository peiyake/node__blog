---
title: gre协议--eogre配置使用
type: net
description: 
date: 2020-09-27 09:10:42
---

EoGRE:Ethernet over GRE（[Generic Routing Encapsulation](https://tools.ietf.org/html/rfc1701)）。

## 报文格式

1. 原始报文

![原始报文](../../images/original.png)

2. EoGRE报文

![eogre报文](../../images/eogre.png)


## 配置举例

### 示意图

如图所示：

![示意图](../../images/eogre-tu.png)

* PC1 挂在 AP1的LAN侧，地址 172.16.0.0/16
* PC2 挂在 AP2的LAN侧，地址 192.168.1.0/24
* AP1和AP2的 WAN在同一网段内，地址是 10.60.0.0/16

如果PC1和PC2 通过EoGRE隧道通信，该如何实现？

### 配置

1. AP1上如下配置

```
~# ip link add tunnel1 type gretap remote 10.60.200.168 local 10.60.200.167 dev eth0
~# ip link set tunnel1 up
~# ip addr add 172.16.1.67/16 dev tunnel1
```

增加一条路由，使通过AP1发出的目的192.168.1.0/24的报文通过 tunnel1接口发送出去

```
~# route add -net 192.168.1.0 netmask 255.255.255.0 dev tunnel1

~# route -n
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.60.1.254     0.0.0.0         UG    0      0        0 eth0
10.60.0.0       0.0.0.0         255.255.0.0     U     0      0        0 eth0
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-lan
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 tunnel1
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 tunnel1
```

关闭防火墙 

```
~# /etc/init.d/firewall stop
```

2. AP2上如下配置

```
~# ip link add tunnel2 type gretap remote 10.60.200.167 local 10.60.200.168 dev eth0
~# ip link set tunnel2 up
~# ip addr add 192.168.1.68/24 dev tunnel2
```

增加一条路由，使通过AP2发出的目的172.16.0.0/16的报文通过 tunnel2接口发送出去

```
~# route add -net 172.16.0.0 netmask 255.255.0.0 dev tunnel2

~# route -n
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.60.1.254     0.0.0.0         UG    0      0        0 eth0
10.60.0.0       0.0.0.0         255.255.0.0     U     0      0        0 eth0
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 tunnel2
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 br-lan
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 tunnel2
```

关闭防火墙 

```
~# /etc/init.d/firewall stop
```

### 测试

1. 在AP1上 ping AP2的 192.168.1.68

```
~# ping 192.168.1.68
PING 192.168.1.68 (192.168.1.68): 56 data bytes
64 bytes from 192.168.1.68: seq=0 ttl=64 time=1.735 ms
64 bytes from 192.168.1.68: seq=1 ttl=64 time=1.517 ms
64 bytes from 192.168.1.68: seq=2 ttl=64 time=1.290 ms
```

2. 在AP2上 ping AP1的 172.16.1.67

```
~# ping 172.16.1.67
PING 172.16.1.67 (172.16.1.67): 56 data bytes
64 bytes from 172.16.1.67: seq=0 ttl=64 time=1.476 ms
64 bytes from 172.16.1.67: seq=1 ttl=64 time=1.944 ms
64 bytes from 172.16.1.67: seq=2 ttl=64 time=1.713 ms
64 bytes from 172.16.1.67: seq=3 ttl=64 time=1.492 ms
```








