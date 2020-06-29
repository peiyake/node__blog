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

## 运行

### 构造配置文件 startup.conf

1. 查看设备端口
   
    如下，我的测试设备有6个千兆口，还有4个扩展千兆口，我已经配置DPDK接管了其中PCI编号 `0000:03:00.0` 和 `0000:03:00.1` 两个网口。
对应的 `startup.conf` 配置文件如下：
   
```
[root@localhost vpp]# ./extras/vpp_config/scripts/dpdk-devbind.py --status

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

Other network devices
=====================
<none>

Crypto devices using DPDK-compatible driver
===========================================
<none>

Crypto devices using kernel driver
==================================
<none>

Other crypto devices
====================
<none>
```

1. 修改配置文件

    配置中,除了**dpdk**部分是自己设置的，其它都是默认配置。（默认配置文件在**build-root/install-vpp_debug-native/vpp/etc/vpp/startup.conf**），如下：

```
unix {
  nodaemon
  log /var/log/vpp/vpp.log
  full-coredump
  cli-listen /run/vpp/cli.sock
  gid vpp
}
api-trace {
  on
}
api-segment {
  gid vpp
}
socksvr {
  default
}
cpu {
}
 dpdk {
         dev 0000:03:00.0 {
                name vpp0
        } 
         dev 0000:03:00.1 {
                name vpp1
        } 
 }
```

### 运行VPP

注意： 这里我把构造好的配置文件，拷贝到了VPP主目录下面了一份

```
[root@localhost vpp]# ./build-root/build-vpp_debug-native/vpp/bin/vpp -c /root/vpp/startup.conf
vlib_plugin_early_init:361: plugin path /root/vpp/build-root/build-vpp_debug-native/vpp/lib/x86_64-linux-gnu/vpp_plugins:/root/vpp/build-root/build-vpp_debug-native/vpp/lib/vpp_plugins
load_one_plugin:189: Loaded plugin: abf_plugin.so (Access Control List (ACL) Based Forwarding)
load_one_plugin:189: Loaded plugin: acl_plugin.so (Access Control Lists (ACL))
load_one_plugin:189: Loaded plugin: avf_plugin.so (Intel Adaptive Virtual Function (AVF) Device Driver)
load_one_plugin:189: Loaded plugin: cdp_plugin.so (Cisco Discovery Protocol (CDP))
load_one_plugin:189: Loaded plugin: crypto_ia32_plugin.so (Intel IA32 Software Crypto Engine)
...

```

### 错误处理

安装上面运行，可能会出现如下错误

1. **api_segment_config: group vpp does not exist Aborted (core dumped)**

    这种错误是由于配置文件 `startup.conf` 中 配置了 `gid vpp`导致的。 错误提示 **组vpp不存子**，那么我们只需要创建一下就可以了

    ```
    [root@localhost vpp]# groupadd -r vpp
    ```


