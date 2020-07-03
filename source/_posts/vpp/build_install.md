---
title: VPP 编译、安装、运行
type: vpp
description: VPP 编译、安装、运行总结
date: 2020-06-22 19:10:42
---

## 下载源码

[参考官方文档](https://wiki.fd.io/view/VPP/Pulling,_Building,_Running,_Hacking_and_Pushing_VPP_Code)

```
[root@localhost ~]git clone https://gerrit.fd.io/r/vpp
[root@localhost ~]cd vpp
[root@localhost ~]git branch -a                   #查看分支
[root@localhost ~]git tag                         #查看标签
[root@localhost ~]# git checkout -b v19.08 v19.08 #从tag  v19.08 检出分支 v19.08
```

## 开启DPDK的`igb_uio.ko`编译

```
sed -i 's/RTE_EAL_IGB_UIO,n/RTE_EAL_IGB_UIO,y/' build/external/packages/dpdk.mk   
```

## 编译

```
# if vpp<08.10

make install-dep
make bootstrap
make build        # or `make build-release`

# vpp 08.10+ (cmake)
make install-dep
make install-ext-deps
make build        # or `make build-release`
```

其中`make install-ext-deps`,编译完成会生成一个rpm包`build/external/vpp-ext-deps-19.08-13.x86_64.rpm`，这个rpm包会被自动安装，安装后的文件位于 `/opt/vpp/` 目录下
里面包括 `dpdk`相关的库、模块和一些可执行文件。

_注意：make build编译debug版本，只能用于调试，性能很差的_

## 配置大页内存

```
~# [ -d /mnt/huge ] || mkdir /mnt/huge
~# mount -t hugetlbfs nodev /mnt/huge
~# echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
挂载hugetlbfs还可以这样 

```
echo "nodev                   /mnt/huge               hugetlbfs rw,relatime" >> /etc/fstab
```

如果设备支持1G大页内存，通过 `bootcmd` 来启用，修改启动 `grub.cfg` 找到启动命令行，添加如下内容：

```
default_hugepagesz=1G hugepagesz=1G hugepages=4
```

## 屏蔽CPU

修改 `grub.cfg` 在启动命令行添加
```
isolcpus=1,2,3,4
```
如上，系统启动后，CPU核心1/2/3/4，将不会被系统使用，后面可以配置成VPP运行专用CPU核心。

## 加载UIO驱动

```
~# modprobe uio
~# insmod /opt/vpp/external/`uname -m`/lib/modules/`uname -r`/extra/dpdk/igb_uio.ko
```

### 构造配置文件 startup.conf

查看设备网口信息，如下，我的测试设备有6个千兆口，还有4个扩展千兆口，我已经配置DPDK接管了其中PCI编号 `0000:07:00.0`、`0000:08:00.1`、`0000:09:00.0`、`0000:0a:00.1`四个网口。
   
```
[root@localhost vpp]# /opt/vpp/external/x86_64/sbin/dpdk-devbind --status

Network devices using DPDK-compatible driver
============================================
0000:03:00.0 '82580 Gigabit Fiber Network Connection' drv=uio_pci_generic unused=igb
0000:03:00.1 '82580 Gigabit Fiber Network Connection' drv=uio_pci_generic unused=igb

Network devices using kernel driver
===================================
0000:03:00.2 '82580 Gigabit Fiber Network Connection' if=eth8 drv=igb unused=uio_pci_generic 
0000:03:00.3 '82580 Gigabit Fiber Network Connection' if=eth9 drv=igb unused=uio_pci_generic 
0000:05:00.0 '82583V Gigabit Network Connection' if=eth0 drv=e1000e unused=uio_pci_generic 
0000:06:00.0 '82583V Gigabit Network Connection' if=eth1 drv=e1000e unused=uio_pci_generic 
0000:07:00.0 '82583V Gigabit Network Connection' if=eth2 drv=e1000e unused=uio_pci_generic 
0000:08:00.0 '82583V Gigabit Network Connection' if=eth3 drv=e1000e unused=uio_pci_generic 
0000:09:00.0 '82583V Gigabit Network Connection' if=eth4 drv=e1000e unused=uio_pci_generic 
0000:0a:00.0 '82583V Gigabit Network Connection' if=eth5 drv=e1000e unused=uio_pci_generic 
...
...
```

构造配置文件

```
~# mkdir -p /etc/vpp
~# cp build-root/install-vpp_debug-native/vpp/etc/vpp/startup.conf /etc/vpp/
```

最终的配置文件

```
unix {
  log /var/log/vpp.log
  full-coredump
  cli-listen /run/vpp/cli.sock
}
api-trace {
  on
}
socksvr {
  default
}
cpu {
        main-core 1
        corelist-workers 4-7
}
dpdk {
        dev 0000:07:00.0 {
                name vpp0
        }
        dev 0000:08:00.0 {
                name vpp1
        }
        dev 0000:09:00.0 {
                name vpp2
        }
        dev 0000:0a:00.0 {
                name vpp3
        }
}
```

### 运行VPP

```
[root@localhost vpp]# ./build-root/install-vpp_debug-native/vpp/bin/vpp -c /etc/vpp/startup.conf
vlib_plugin_early_init:361: plugin path /root/vpp/build-root/build-vpp_debug-native/vpp/lib/x86_64-linux-gnu/vpp_plugins:/root/vpp/build-root/build-vpp_debug-native/vpp/lib/vpp_plugins
load_one_plugin:189: Loaded plugin: abf_plugin.so (Access Control List (ACL) Based Forwarding)
load_one_plugin:189: Loaded plugin: acl_plugin.so (Access Control Lists (ACL))
load_one_plugin:189: Loaded plugin: avf_plugin.so (Intel Adaptive Virtual Function (AVF) Device Driver)
load_one_plugin:189: Loaded plugin: cdp_plugin.so (Cisco Discovery Protocol (CDP))
load_one_plugin:189: Loaded plugin: crypto_ia32_plugin.so (Intel IA32 Software Crypto Engine)
...

```

## 查看 

```
[root@localhost vpp]# ./build-root/install-vpp_debug-native/vpp/bin/vppctl 
    _______    _        _   _____  ___ 
 __/ __/ _ \  (_)__    | | / / _ \/ _ \
 _/ _// // / / / _ \   | |/ / ___/ ___/
 /_/ /____(_)_/\___/   |___/_/  /_/    

DBGvpp# 
DBGvpp# 
DBGvpp# show interface 
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
vpp0                              1     down         9000/0/0/0     
vpp1                              2     down         9000/0/0/0     
vpp2                              3     down         9000/0/0/0     
vpp3                              4     down         9000/0/0/0     
DBGvpp# 
DBGvpp# 
```

### 错误处理

安装上面运行，可能会出现如下错误

1. **api_segment_config: group vpp does not exist Aborted (core dumped)**

这种错误是由于配置文件 `startup.conf` 中 配置了 `gid vpp`导致的。 错误提示 **组vpp不存子**，那么我们只需要创建一下就可以了

```
[root@localhost vpp]# groupadd -r vpp
```


