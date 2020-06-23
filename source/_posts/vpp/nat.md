---
title: VPP之NAT功能测试
type: vpp
description: CentOS Linux release 7.5.1804 (Core),VPP-tag-19.089
date: 2020-06-22 19:10:42
---

# VPP之NAT功能测试

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

按照linux系统上iptables中的NAT特性来对比测试


### DNAT
---

1. 端口映射
   * 测试方案
     * PC1 在8099端口开启web服务器
     * 从PC2（172.17.1.11）访问 vpp0(172.16.1.1)的80端口
     * 映射到PC1的172.16.1.11:8099端口
     * 查看PC1的web服务是否能访问

   * vppctl 配置

```
[root@localhost vpp]# ./build-root/build-vpp_debug-native/vpp/bin/vppctl
    _______    _        _   _____  ___ 
 __/ __/ _ \  (_)__    | | / / _ \/ _ \
 _/ _// // / / / _ \   | |/ / ___/ ___/
 /_/ /____(_)_/\___/   |___/_/  /_/    

DBGvpp# 
DBGvpp# show interface 
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1     down         9000/0/0/0     
vpp1                              2     down         9000/0/0/0     
DBGvpp# show interface addr
local0 (dn):
vpp0 (dn):
vpp1 (dn):
DBGvpp# 
DBGvpp# set interface state vpp0 up
DBGvpp# set interface state vpp1 up
DBGvpp# 
DBGvpp# set interface ip address vpp0 172.16.1.1/16 
DBGvpp# set interface ip address vpp1 172.17.1.1/16
DBGvpp# 
DBGvpp# show interface 
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1      up          9000/0/0/0     
vpp1                              2      up          9000/0/0/0     rx packets                    98
                                                                    rx bytes                   10441
                                                                    drops                         98
                                                                    ip4                           49
                                                                    ip6                           25
DBGvpp# show interface addr
local0 (dn):
vpp0 (up):
  L3 172.16.1.1/16
vpp1 (up):
  L3 172.17.1.1/16
DBGvpp# 
DBGvpp# nat44 add interface address vpp0
DBGvpp# nat44 add interface address vpp1
DBGvpp# set interface nat44 in vpp0
DBGvpp# set interface nat44 out vpp1
DBGvpp# nat44 add static mapping tcp local 172.16.1.11 8099 external  172.16.1.1 80
DBGvpp# show nat44 static mappings                                                 
NAT44 static mappings:
 tcp local 172.16.1.11:8099 external 172.17.1.1:80 vrf 0 
```

  完成上述配置后，从PC2访问 http://172.17.1.1  就可以访问到 PC1的8099端口的wen服务了

1. 重定向
2. 

