---
title: DHCP 协议
type: net
description: DHCP协议报文格式详解
date: 2020-06-22 19:10:42
---

# DHCP协议详解

这里引用[百度百科-DHCP](https://baike.baidu.com/item/DHCP/218195?fromtitle=DHCP%E5%8D%8F%E8%AE%AE&fromid=1989741)上的一段解释

>动态主机设置协议（英语：Dynamic Host Configuration Protocol，DHCP）是一个局域网的网络协议，使用UDP协议工作，主要有两个用途：用于内部网或网络服务供应商自动分配IP地址；给用户用于内部网管理员作为对所有计算机作中央管理的手段。

## 协议报文格式定义

1. dhcp协议的核心特性在 **RFC2131** 和 **RFC2132** 中定义

+ [RFC2131](http://www.rfc-editor.org/rfc/rfc2131.txt)

    RFC2131 定义了dhcp协议的格式和规程

+ [RFC2132](http://www.rfc-editor.org/rfc/rfc2132.txt)

    RFC2132 定义了dhcp协议的 option 字段的规定，及使用说明。

2. 协议格式

```
   0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |     op (1)    |   htype (1)   |   hlen (1)    |   hops (1)    |
   +---------------+---------------+---------------+---------------+
   |                            xid (4)                            |
   +-------------------------------+-------------------------------+
   |           secs (2)            |           flags (2)           |
   +-------------------------------+-------------------------------+
   |                          ciaddr  (4)                          |
   +---------------------------------------------------------------+
   |                          yiaddr  (4)                          |
   +---------------------------------------------------------------+
   |                          siaddr  (4)                          |
   +---------------------------------------------------------------+
   |                          giaddr  (4)                          |
   +---------------------------------------------------------------+
   |                                                               |
   |                          chaddr  (16)                         |
   |                                                               |
   |                                                               |
   +---------------------------------------------------------------+
   |                                                               |
   |                          sname   (64)                         |
   +---------------------------------------------------------------+
   |                                                               |
   |                          file    (128)                        |
   +---------------------------------------------------------------+
   |                                                               |
   |                          options (variable)                   |
   +---------------------------------------------------------------+
```

3. 字段详解

下面示意内容，基本来自[百度百科-DHCP协议](https://baike.baidu.com/item/DHCP/218195?fromtitle=DHCP%E5%8D%8F%E8%AE%AE&fromid=1989741)
+ **op** 消息类型，1字节
    + 1 ：BOOTREQUEST
    + 2 ：BOOTREPLY
+ **htype** 硬件地址类型，1字节
    + 1 ：以太网地址类型，也就是我们常用的mac地址
+ **hlen** 硬件地址长度，1字节
+ **hops** 1字节，若封包需经过 router 传送，每站加 1 ，若在同一网内，为 0。
+ **xid** 4 字节，DHCP REQUEST 时产生的数值，以作 DHCPREPLY 时的依据
+ **secs** 2 字节，Client 端启动时间（秒）
+ **flags** 2 字节，从 0 到 15 共 16 bits ，最左一 bit 为 1 时表示 server 将以广播方式传送封包给 client ，其余尚未使用
+ **ciaddr** 4 字节，要是 client 端想继续使用之前取得之 IP 地址，则列于这里
+ **yiaddr** 4 字节，从 server 送回 client 之 DHCP OFFER 与 DHCPACK封包中，此栏填写分配给 client 的 IP 地址。
+ **siaddr** 4 字节，若 client 需要透过网络开机，从 server 送出之 DHCP OFFER、DHCPACK、DHCPNACK封包中，此栏填写开机程序代码所在 server 之地址。
+ **giaddr** 4 字节，若需跨网域进行 DHCP 发放，此栏为 relay agent 的地址，否则为 0
+ **chaddr** 16 字节，Client 之硬件地址
+ **sname** 64 字节，Server 之名称字符串，以 0x00 结尾
+ **file** 4 字节，若 client 需要透过网络开机，此栏将指出开机程序名称，稍后以 TFTP 传送
+ **options** TLV格式数据，T:1byte  L:1byte, 允许厂商定议选项（Vendor-Specific Area)，以提供更多的设定信息（如：Netmask、Gateway、DNS、等等）。其长度可变，同时可携带多个选项，每一选项之第一个 byte 为信息代码，其后一个 byte 为该项数据长度，最后为项目内容

4. option概述
+ option 格式
```
    Code   Len  Type
   +------+------+------+
   | CODE |  LEN | TYPE |
   +------+------+------+
```
+ option 255 用来作为终止符，当CODE=255 意味着dhcp报文的options部分到最后了。 每个DHCP报文都应该以此option结束
  + CODE = 255
  + LEN = 1
  + TYPE = padding 数据

+ option 53
  用来指明DHCP报文的类型，DISCOVERY/OFFER/REQUEST...
  + CODE = 53
  + LEN = 1
  + TYPE = 1-9
```
        Value   Message Type
        -----   ------------
            1     DHCPDISCOVER
            2     DHCPOFFER
            3     DHCPREQUEST
            4     DHCPDECLINE
            5     DHCPACK
            6     DHCPNAK
            7     DHCPRELEASE
            8     DHCPINFORM
```
+ option 54 
  用来指明 dhcp server标识，主要用在 DHCPOFFER DHCPREQUEST 报文中。用在 DHCPOFFER 中时，如果客户端同时受到多个 OFFER,那么依据此值来区分服务器端；
  用在 DHCPREQUEST 时，用来告知服务器“我接受了哪个server分配的IP地址，不是这个IP的server就不要再给我分配地址了”
  + CODE = 54
  + LEN = 4
  + TYPE = ipaddr
+ option 51
  用来指定分配的IP地址的租期
  + CODE = 51
  + LEN = 4
  + TYPE = 租期（秒）
+ option 1
  用来指定分配的IP地址的掩码，subnet
  + CODE = 1
  + LEN = 4
  + TYPE = ipaddr of subnet
+ option 4
  用来指明路由IP，
  + CODE = 4
  + LEN =  应当是4的倍数，最小为4
  + TYPE = 是一个ip list，当LEN > 4 时，表示TYPE 有多个IP
+ option 6
  指明DNS
  + CODE = 6
  + LEN = 应当是4的倍数，最小为4
  + TYPE = 是一个dns ip list，当LEN > 4 时，表示TYPE 有多个dns IP

  ... 等等等等， 还有很多option，具体含义可以参考 [RFC2132](http://www.rfc-editor.org/rfc/rfc2132.txt)， 这里不再列举

## DHCP协议流程

### 图例

```
                Server          Client          Server
            (not selected)                    (selected)

                  v               v               v
                  |               |               |
                  |     Begins initialization     |
                  |               |               |
                  | _____________/|\____________  |
                  |/DHCPDISCOVER | DHCPDISCOVER  \|
                  |               |               |
              Determines          |          Determines
             configuration        |         configuration
                  |               |               |
                  |\             |  ____________/ |
                  | \________    | /DHCPOFFER     |
                  | DHCPOFFER\   |/               |
                  |           \  |                |
                  |       Collects replies        |
                  |             \|                |
                  |     Selects configuration     |
                  |               |               |
                  | _____________/|\____________  |
                  |/ DHCPREQUEST  |  DHCPREQUEST\ |
                  |               |               |
                  |               |     Commits configuration
                  |               |               |
                  |               | _____________/|
                  |               |/ DHCPACK      |
                  |               |               |
                  |    Initialization complete    |
                  |               |               |
                  .               .               .
                  .               .               .
                  |               |               |
                  |      Graceful shutdown        |
                  |               |               |
                  |               |\ ____________ |
                  |               | DHCPRELEASE  \|
                  |               |               |
                  |               |        Discards lease
                  |               |               |
                  v               v               v
```
### 各消息类型含义

```
   Message         Use
   -------         ---

   DHCPDISCOVER -  Client broadcast to locate available servers.
                   客户端广播请求，寻找可用的DHCP服务器

   DHCPOFFER    -  Server to client in response to DHCPDISCOVER with
                   offer of configuration parameters.
                   服务端返回给客户端的IP地址配置参数

   DHCPREQUEST  -  Client message to servers either (a) requesting
                   offered parameters from one server and implicitly
                   declining offers from all others, (b) confirming
                   correctness of previously allocated address after,
                   e.g., system reboot, or (c) extending the lease on a
                   particular network address.
                   客户端向服务端发送的消息：
                   (a) 表面上是向某一个服务器回复消息进行IP地址确认，暗含的意思是也告诉其他服务器---我已经选择了其中一个服务器分配的IP （此时报文是广播包）
                   (b) 例如系统重启后，发送REQUESET 用来对自己持有的IP配置向服务器进行确认
                   (c) 

   DHCPACK      -  Server to client with configuration parameters,
                   including committed network address.
                   服务端向客户端发送的配置参数确认消息

   DHCPNAK      -  Server to client indicating client's notion of network
                   address is incorrect (e.g., client has moved to new
                   subnet) or client's lease as expired

   DHCPDECLINE  -  Client to server indicating network address is already
                   in use.

   DHCPRELEASE  -  Client to server relinquishing network address and
                   cancelling remaining lease.

   DHCPINFORM   -  Client to server, asking only for local configuration
                   parameters; client already has externally configured
                   network address.
```

## DHCP报文举例

1. 一个自动获取DHCP的完整流程

   ![DHCP Discover](/images/dhcp_auto.png)
   
2. DHCP Discover
   
   ![DHCP Discover](/images/dhcp_discover.png)

3. DHCP Offer
  
   ![DHCP Offer](/images/dhcp_offer.png)