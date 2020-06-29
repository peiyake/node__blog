---
title: iptables 命令详解
type: shell
description: iptable命令常用功能使用方法总结
date: 2020-06-22 19:10:42
---

iptables是linux下的防火墙管理工具,它是linux系统下和NETFILETER架构模块交互的用户态命令行工具,通过iptable可以对linux的NETFILTER配置各种规则,来达到对网络防火墙进行管理的目的. NETFILTER本身就是非常复杂的一套东西, 本文这里主要介绍iptables的使用方法,有关NETFILTER架构的有关知识可以查看我的另一篇文章 `NETFILTER专题学习`。

## iptables 命令学习总结

### iptables规则基本要素

一条iptables规则由以下几种基本要素构成：

* 规则适配哪张表
* 规则的动作(增、删、改)
* 规则适配该表的哪条链
* 规则如何对报文进行匹配、过滤
* 规则匹配到报文之后如何处理

### 表 和 链

本文主要介绍iptables中常用的三张表 **mangle**、**nat**、**filter**

|表名|链名|
|----|----|
|mangle|PREROUTING
||INPUT|
||FORWARD|
||POSTROUTING|
||OUTPUT|
|nat|PREROUTING|
||OUTPUT|
||POSTROUTING|
|filter|INPUT|
||FORWORD|
||OUTPUT|

知道了iptables 有哪些表以及每张表有哪些链之后，那么对应 **iptables规则基本要素** 此时我们就可以来确定前三个基本要素了

---

### 要素1 : **规则适配哪张表**

* `-t` 参数指定规则要适配的表名称，例如：

```shell
iptables -t nat         //匹配nat表
iptables -t filter      //匹配filter表
iptables -t mangle      //匹配mangle表
```
注意：如果省略 `-t` 参数,默认操作的是 `filter` 表

---

### 要素2、3 : **规则的动作 和 指定链名称**

该要素大致分为2类，一是 **查看动作** ，另一个是 **设置动作** , 规则的动作一般都是针对 **链** 的，所以一般都要指定链的名称， __某张表有哪些链在上面表格已经说明__

* **查看动作**
    - `-L | --list` 列出此规则链中的所有规则,如果没有指定链名,则列出该表的所有规则
    - `-S | --list-rules` 以iptablses save格式打印特定规则链中的规则
    - `-n | --numeric` 以数字形式显示IP和端口
    - `-v | --verbose` 列出每条规则额外信息，如：数据包个数、规则选项及相关网络接口
    - `-L --line-numbers`  列出规则的同时,列出每条规则的序号

例如：

```shell
[root@localhost ~]# iptables -nvL
Chain INPUT (policy ACCEPT 7105K packets, 5036M bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     udp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
    0     0 ACCEPT     tcp  --  virbr0 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:53

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  *      virbr0  0.0.0.0/0            192.168.122.0/24     ctstate RELATED,ESTABLISHED
    0     0 ACCEPT     all  --  virbr0 *       192.168.122.0/24     0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 6119K packets, 2825M bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     udp  --  *      virbr0  0.0.0.0/0            0.0.0.0/0            udp dpt:68
[root@localhost ~]# iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A INPUT -i virbr0 -p udp -m udp --dport 53 -j ACCEPT
-A INPUT -i virbr0 -p tcp -m tcp --dport 53 -j ACCEPT
-A FORWARD -d 192.168.122.0/24 -o virbr0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A OUTPUT -o virbr0 -p udp -m udp --dport 68 -j ACCEPTrbr0 -j REJECT --reject-with icmp-port-unreachable
```

* **设置动作**
    - `-F | --flush` 清空指定规则链的所有规则，没有指定链名的话则清空该表的所有规则
    - `-Z | --zero` 重置与每个规则链相关的数据包和字节统计
    - `-P | --policy <chain> <policy>`  为链设置默认规则, `chain` 表示链名，必须指定, `policy` 是默认规则,可以是 `ACCEPT` 和 `DROP`
    - `-A | --append` 在指定链所有规则后面追加一条规则 
    - `-I | --insert [<rule number>]` 在指定 `rule number` 开始处插入一条规则. `rule number` 可以通过 `iptable -L --line-numbers` 查看
    - `-R | --replace [<rule number>]` 替换指定 `rule number` 的规则
    - `-D | --delete [<rule number> | 规则详情]`  删除某条链上指定 `规则详情` 或者 `rule number` 的规则
    - `-C | --check [<规则详情>]`  检查链上是否已存在某条规则

>上面我们已经知道了构成一条iptables命令的前三个要素，那么当我们写一条iptables命令的时候已经知道前半部分怎么写了。 后半部分则是相对复杂一点的，这里我们
跳过 **要素4** 先介绍一下 **要素5**

---

### **要素5 : 匹配后如何处理？**

在继续说明之前，这里要回顾一下 **要素2 的 设置动作** 中的一个选项 `-P` ,例如：

```shell
[root@localhost ~]# iptables -F                     // 清空默认表 filter 的所有规则
[root@localhost ~]# iptables -L FORWARD             // 查看 filter表 FORWARD 链的规则
Chain FORWARD (policy ACCEPT)   //  policy ACCEPT  表示，进入这条链的所有报文都会被 ACCEPT （接受），总是允许转发
target     prot opt source               destination         
[root@localhost ~]# iptables -P FORWARD DROP        // 修改 filter 表 FORWARD 链的默认规则是 DROP
[root@localhost ~]# iptables -L FORWARD             // 再次查看
Chain FORWARD (policy DROP)     //  policy DROP  表示，进入这条链的所有报文都会被 DROP （丢弃），也就是禁止转发
target     prot opt source               destination
```

    在我们决定某条规则匹配后如何处理时，首先我们要看该链的默认规则是什么

* 如果默认规则是 **ACCEPT**， 那么我们规则的处理动作一般是 将匹配规则的报文 DROP 或者进行其处理. 意思是： 正常情况下报文都接收，但是只有匹配我们设置的规则的报文拒绝接收
* 如果默认规则是 **DROP**， 那么我们规则的处理动作一般是 将匹配规则的报文 ACCEPT 或者进行其处理. 意思是： 正常情况下报文都拒绝，但是只有匹配我们设置的规则的报文才允许通过

>理解默认 `-P` 选项指定的默认规则后，下面我们再来说明 iptables 命令如何指定 匹配后如何处理，这个由 `-j` 选项来指明，如下：


>下面正式对 **要素5 匹配后如何处理** 进行讲解，如何处理由参数 `-j` 指定，如下说明：

* `-j  < ACCEPT | DROP | LOG >`
    - `-j ACCEPT`  意思是接受匹配到的报文
    - `-j DROP`  意思是丢弃匹配到的报文
    - `-j LOG`  意思是当匹配到指定条件报文是记录日志,这个日志是记录在 syslog 指定的 kernel日志目录,当然 `dmesg` 命令也能看到



### **要素4 -- 规则如何对报文进行匹配、过滤**

    报文的匹配规则比较多,也比较复杂，这些匹配规则指明了：这条iptables规则是针对哪些报文生效的？例如说：针对某个接口、针对某种协议、或者端口号等。为了更好的记忆和理解
我对这些匹配规则进行简单分类为：基本匹配规则 和 扩展匹配规则。 下面对其一一进行说明。

### **基本匹配规则**

* `-i | --in-interface [!] [<interface>]`  匹配从指定物理接口 或者 非指定接口 上来的报文, 用于 **PREROUTING / INPUT / FORWARD** 链
* `-o | --out-interface [!] [<interface>]` 匹配从指定物理接口 或者 非指定接口 发出的报文, 用于 **FORWORD / OUTPUT / FORWARD** 链
* `-p | --protocol [!] [<protocol>]`  匹配指定协议 或者 非指定协议 的报文 ， 例如 `-p tcp` ,`-p udp` ,`-p ! tcp`, `-p ! udp`
* `-s | --source | --src [!] <address>[</mask>]` 匹配指定源IP 或者 非指定源IP 的报文, 也可以使用掩码表示地址段
* `-d | --destination | --dst [!] <address>[</mask>]`  匹配指定目的IP 或者 非指定目的IP 的报文

使用举例：

`-i` 选项使用 
* `iptable -A INPUT -i eth0 -j ACCEPT`      filter表INPUT链上接受eth0端口的所有报文
* `iptable -A INPUT ! -i eth0 -j ACCEPT`     filter表INPUT链上接受除了eth0外其它端口的所有报文
. `-o` 选项使用
* `iptable -A OUTPUT -o eth0 -j DROP`      filter表OUTPUT链上丢弃发出的报文
* `iptable -A OUTPUT ! -i eth0 -j DROP`     filter表OUTPUT链上丢弃除了eth0外其它端口的所有报文
. `-p` 选项使用
* `iptables -A INPUT -p tcp -j ACCEPT`  filter表INPUT链接受所有tcp协议报文
* `iptables -A INPUT ! -p tcp -j ACCEPT` filter表INPUT链接受所有非 tcp协议的报文
. `-s`, `-d` 选项使用
* `iptables -A INPUT --src 10.60.200.2 -j ACCEPT`  filter 表INPUT链接受所有源IP是10.60.200.2的报文
* `iptables -A INPUT --src 10.60.0.0/16 -j ACCEPT`
* `iptables -A INPUT ! --src 10.60.0.0/16 -j ACCEPT`
* `iptables -A INPUT --dst 10.60.200.2 -j ACCEPT`
* `iptables -A INPUT --dst 10.60.0.0/16 -j ACCEPT`
* `iptables -A INPUT ! --dst 10.60.0.0/16 -j ACCEPT`

>以上就是 **要素5的基本匹配规则**，上面列列举的例子只是做语法的展示，并没有实际的用途，后面我会展示一些常用的规则配置实例。

#### **扩展匹配规则**

1. 按协议扩展匹配规则
 1. `-p tcp` 协议扩展匹配规则
   * `--source-port | --sport [[!] <port>[:<port>]]`   用于指定tcp协议的源端口 或者非指定端口,也可以指定端口的范围
   * `--destination-port | --dport <port>[:<port>]`    指定tcp协议目的端口 或者 端口范围
   * `--tcp-flags <which-mask> <macth-mask>`   指定匹配tcp连接标志位的哪些位进行匹配 （这个知识点有点广泛,后面我会总结一下，也可以参考 link:http://www.zsythink.net/archives/1578[这个博客]）
   * `[!]--syn` 这个相当于上面的 `-p tcp --tcp-flags ALL SYN`, 如果指定了 **！** 则是除了 **SYN** 的情况
 2. `-p udp` 协议扩展匹配规则
   * `--source-port | --sport [!] <port>[:<port>]`     用于匹配指定源端口 或者 非指定源端口的UDP报文
   * `--destination-port | --dport <port>[:<port>]`    用于匹配指定目的端口 或者 非指定目的端口的UDP报文 
 3. `--icmp-type [!] <type>`     匹配指定类型名或者类型号 或者 非指定类型名和类型号 ICMP协议报文

    * **type** 的值有以下这些
      * 类型号 `0` : 类型名 `echo-reply`      ping 应答
      * 类型号 `8` : 类型名 `echo-request`      ping 请求
      * 类型号 `3` : 类型名 `destination-unreachable`     目标不可达
      * `...` 更多类型可以参考 <TCP/IP详解卷1>
