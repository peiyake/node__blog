---
title: RAW socket使用方法
type: linuxc
description: raw socket，即原始套接字，可以接收本机网卡上的数据帧或者数据包，对于监听网络的流量和分析是很有作用的
date: 2020-12-15 19:10:42
---

## RAW socket

>原始套接字（raw socket）是一种网络套接字，允许直接发送/接收IP协议数据包而不需要任何传输层协议格式。对于标准的套接字，通常数据按照选定的传输层协议(例如TCP、UDP)自动封装，socket用户并不知道在网络介质上广播的数据包含了这种协议包头。从原始套接字读取数据包含了传输层协议包头。用原始套接字发送数据，是否自动增加传输层协议包头是可选的。原始套接字用于安全相关的应用程序，如nmap。原始套接字一种可能的用例是在用户空间实现新的传输层协议。 原始套接字常在网络设备上用于路由协议，例如IGMPv4、开放式最短路径优先协议 (OSPF)、互联网控制消息协议 (ICMP)。Ping就是发送一个ICMP响应请求包然后接收ICMP响应回复.

## RAW socket 可以做什么？

1. 收包。
   1. 携带IP头的包
   2. 携带以太网头的包
2. 发包
   1. 发送报文时，自己封装IP头
   2. 发送所有IPV4协议栈支持的协议报文 ICMP/UDP/TCP .. （普通套接字我们只是常用于TCP/UDP）
   3. 发送报文时，自己封装以太网头

## 使用 RAW socket

1. 创建raw socket

```c
    #include <sys/socket.h>
    #include <netinet/in.h>
    raw_socket = socket(int family, int socket_type, int protocol);
```

* _family_:
  * AF_INET : 该类型套接字发送或接受的包，不包括以太网头
  * AF_PACKET : 收到的包 根据 _socket\_type_ 的值，可以包含以太网头，也可以不包含以太网头

* _socket\_type_ 和 _protocol_ 的值,根据 _family_ 来定，具体参考官网手册最可靠
  * AF_INET : 参考 [man手册raw(7)](https://man7.org/linux/man-pages/man7/raw.7.html)
  * AF_PACKET ： 参考 [man手册packet(7)](https://man7.org/linux/man-pages/man7/packet.7.html)

2. 例子程序

本人写了raw socket一些常见的收发包程序,请参考 [raw socket例子程序](https://peiyake.com/manpage/html/l2__recive_8c_source.html)

## 知识点

* 二层收发包，使用 `struct sockaddr_ll` 结构来描述接收、发送的源/目的地址（物理接口级别的）
* 三层收发包，使用 `struct sockaddr_in` 结构来描述接收、发送的源/目的地址（IP地址级别的）
* 二层收包, 可以绑定物理接口,如果需要接收非本机mac的报文，需要设置混杂模式（在例子程序l2_recive.c）中都有体现
* 三层发包,如果只是TCP/UDP的包,其实用 普通套接字就可以了
* 二层发包,需要指定发包的物理接口, 这个使用 `struct sockaddr_ll` 结构 设置 **sll_ifindex**为发包接口，传给 `sendto` 第5个参数

`struct sockaddr_ll` 结构使用介绍：

在 [man手册packet(7)](https://man7.org/linux/man-pages/man7/packet.7.html) 有很详细的解释：

```
The sockaddr_ll structure is a device-independent physical-layer
       address.

           struct sockaddr_ll {
               unsigned short sll_family;   /* Always AF_PACKET */
               unsigned short sll_protocol; /* Physical-layer protocol */
               int            sll_ifindex;  /* Interface number */
               unsigned short sll_hatype;   /* ARP hardware type */
               unsigned char  sll_pkttype;  /* Packet type */
               unsigned char  sll_halen;    /* Length of address */
               unsigned char  sll_addr[8];  /* Physical-layer address */
           };

       The fields of this structure are as follows:

       *  sll_protocol is the standard ethernet protocol type in network
          byte order as defined in the <linux/if_ether.h> include file.  It
          defaults to the socket's protocol.

       *  sll_ifindex is the interface index of the interface (see
          netdevice(7)); 0 matches any interface (only permitted for bind‐
          ing).  sll_hatype is an ARP type as defined in the
          <linux/if_arp.h> include file.

       *  sll_pkttype contains the packet type.  Valid types are PACKET_HOST
          for a packet addressed to the local host, PACKET_BROADCAST for a
          physical-layer broadcast packet, PACKET_MULTICAST for a packet
          sent to a physical-layer multicast address, PACKET_OTHERHOST for a
          packet to some other host that has been caught by a device driver
          in promiscuous mode, and PACKET_OUTGOING for a packet originating
          from the local host that is looped back to a packet socket.  These
          types make sense only for receiving.

       *  sll_addr and sll_halen contain the physical-layer (e.g., IEEE
          802.3) address and its length.  The exact interpretation depends
          on the device.

       When you send packets, it is enough to specify sll_family, sll_addr,
       sll_halen, sll_ifindex, and sll_protocol.  The other fields should be
       0.  sll_hatype and sll_pkttype are set on received packets for your
       information.
```

