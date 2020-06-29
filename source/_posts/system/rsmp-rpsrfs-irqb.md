---
title: Linux性能调节之IRQ/RPS
type: system
description: 使用irqbalance/smp irq affinity/rps,rfs 技术进行网络性能调节
date: 2020-06-22 19:10:42
---

irqbalance 是Linux内核自带的一个中断调节服务，它根据网络负载情况自动将网卡中断和CPU进行亲和请设置。事实上它就是 **自动的 smp irq affinity** 它有很好的动态调节效果，但是对于大量小包的网络环境，irqbalance几乎是无效的。

## 功能说明

### irqbalance

#### 功能

 引用 [Linux Man Page](https://linux.die.net/man/1/irqbalance)上的解释：
>irqbalance - distribute（把...分散开） hardware interrupts across processors on a multiprocessor system.
>The purpose of irqbalance is distribute hardware interrupts across processors on a multiprocessor system in order to increase performance.

    irqbalance 是Linux内核自带的一个中断调节服务，它根据网络负载情况自动将网卡中断和CPU进行亲和请设置。事实上它就是 **自动的 smp irq affinity** 它有很好的动态调节效果，但是对于大量小包的网络环境，irqbalance几乎是无效的。

#### 使用方式

1. 开启

    systemctl start irqbalance.service

2. 关闭

    systemctl stop irqbalance.service

#### 特点

    在linux系统网络设备上，使用irqbalance满足普通场景，但对于高负载网络环境，有时它并不能完全利用CPU的处理能力。

### smp irq affinity

#### 功能

    多处理器系统上，配置中断和CPU的亲和性。[更多官方解释](https://cs.uwaterloo.ca/~brecht/servers/apic/SMP-affinity.txt)

#### 使用方式

1. 找到每个NIC对应的中断号(**IrqNO**) `cat /proc/interrupts`

```
    [root@localhost~]# cat /proc/interrupts 
            CPU0       CPU1       CPU2       CPU3       
    0:        164          0          0          0   IO-APIC-edge      timer
    1:          1          1          0          0   IO-APIC-edge      i8042
    4:        277        284        296        289   IO-APIC-edge      serial
->  28:         40         37         33   60102094   PCI-MSI-edge      eth0
->  29:          4    4524023          2          3   PCI-MSI-edge      eth1
    30:    5533061       1759       1985       1631   PCI-MSI-edge      0000:00:1f.2
    31:          0          0          1          1   PCI-MSI-edge      gma500
->  32:          2          3     834989          0   PCI-MSI-edge      eth2
->  33:       3347         47     644476     187124   PCI-MSI-edge      eth3
    NMI:      74720     123133      42419     113126   Non-maskable interrupts
    LOC:  488678875  312854283  356776267  309250165   Local timer interrupts
    SPU:          0          0          0          0   Spurious interrupts
    PMI:      74720     123133      42419     113126   Performance monitoring interrupts
    IWI:    2875727    7429396    4221092    9896317   IRQ work interrupts
    RTR:          0          0          0          0   APIC ICR read retries
    RES:    9273280    7005313    7621455    6550609   Rescheduling interrupts
    CAL: 4294919647 4294931395     393526     421503   Function call interrupts
    TLB:    4806826   13109838    5281202   10316715   TLB shootdowns
    TRM:          0          0          0          0   Thermal event interrupts
    THR:          0          0          0          0   Threshold APIC interrupts
    DFR:          0          0          0          0   Deferred Error APIC interrupts
    MCE:          0          0          0          0   Machine check exceptions
    MCP:       5567       5567       5567       5567   Machine check polls
    ...
```
2. 每个中断号都有如下配置项
   
   /proc/irq/**IrqNo**/smp_affinity   （16进制，亲和CPU）

   /proc/irq/**IrqNo**/smp_affinity_list （10进制，亲和CPU）

3. CPU位图配置方法

    例如：在4核、4NIC设备上

    * 某一个NIC 和 某个CPU设置亲和性
```
                Binary       Hex(smp_affinity) 
        CPU 0    0001         1 
        CPU 1    0010         2
        CPU 2    0100         4
        CPU 3    1000         8
```
    * 某一个NIC 和 两个CPU设置亲和性
```
                Binary       Hex 
        CPU 0    0001         1 
    +   CPU 2    0100         4
        -----------------------
        both     0101         5
```
    * 某一个NIC 和 全部4个CPU设置亲和性
```
                Binary       Hex 
        CPU 0    0001         1 
        CPU 1    0010         2
        CPU 2    0100         4
    +   CPU 3    1000         8
        -----------------------
        both     1111         f
```

4. 配置示例如下：

    * eth0 和 CPU0这只亲和性

        echo 1 > /proc/irq/28/smp_affinity  或者 echo 0 > /proc/irq/28/smp_affinity_list

    * eth0 和 CPU0/CPU1设置亲和性（对多队列网卡而言）

        echo 3 > /proc/irq/28/smp_affinity  或者 echo '0,1' > /proc/irq/28/smp_affinity_list

    注意： smp_affinity_list 中写入的是CPU号，例如4核CPU，那么编号是 0/1/2/3。另外，只需要配置上面两者其中一个就可以了，另一个会随之改变。

#### 特点

1. SMP affinity 是将硬中断 和 CPU直接配置亲和处理
2. 对单队列网卡，将一个NIC和多个CPU设置亲和性是不生效的，单队列网卡只能绑定一个CPU。

问题来了：如果一个网络设备有8个CPU核，但是只有4个单队列NIC，怎么才能把所有CPU都利用上呢？ 答案就是下面介绍的 **RPS/RFS**

### rps/rfs

#### 功能

1. rps (Receive Side Scaling)

    把收到的报文按照hash方式负载均衡到所配置的CPU核心上。

2. rfs

    rfs的实施以rps为前提，因为rps虽然可以将包负载均衡到各个CPU上，但是如果侦听该包的应用程序工作的CPU和rps将包“转换”到的CPU，不相同的话，就会降低cache命中率，性能反而
    并不好。 rfs的作用就是，在rps的前提下，依据系统流的信息来确定报文被分发到哪个CPU。 尽可能的将包分发到上次处理的CPU或者socket监听的CPU。这样就会增加cache命中率，提高
    性能。

#### 使用方法

和 SMP一样，rps的生效也是通过配置sys文件系统节点来动态生效的，而且配置值也是CPU位图值。

1. rps配置文件

```
/sys/class/net/<dev>/queues/rx-<n>/rps_cpus
```
2. rfs配置文件
```
/proc/sys/net/core/rps_sock_flow_entries
/sys/class/net/<dev>/queues/rx-<n>/rps_flow_cnt
```
3. rfs官方配置建议：

```
    Both of these need to be set before RFS is enabled for a receive queue.309 Values for both are rounded up to the nearest power of two. The310 suggested flow count depends on the expected number of active connections 311 at any given time, which may be significantly less than the number of open 312 connections. We have found that a value of 32768 for rps_sock_flow_entries 313 works fairly well on a moderately loaded server.

    For a single queue device, the rps_flow_cnt value for the single queue316 would normally be configured to the same value as rps_sock_flow_entries. 317 For a multi-queue device, the rps_flow_cnt for each queue might be318 configured as rps_sock_flow_entries / N, where N is the number of319 queues. So for instance, if rps_sock_flow_entries is set to 32768 and there 320 are 16 configured receive queues, rps_flow_cnt for each queue might be321 configured as 2048.
```

```
    测试总结，当rps_sock_flow_entries 的值设置为 32768效果最好。
    对于单队列网卡，rps_flow_cnt的值最好配置成和rps_sock_flow_entries的值一样。
对于多队列网卡，假如队列数为N， rps_flow_cnt的值最好满足如下公式：rps_flow_cnt = rps_sock_flow_entries / N
```

#### 特点

1. rps/rfs 是通过软件的方式实现包 和 CPU的绑定的。
2. rps/rfs 并不能降低报文硬中断的次数，反而会增加软中断的次数。
3. rps/rfs 最好用在单队列网卡，多CPU的环境，这样就能提供CPU的使用率。

## 自动化配置脚本

```shell
#!/bin/bash

#**********************************************************************************#
#* 1. auto set SMP Irq Affinity                                                    #
#*    1.1  配置文件                                                                 # 
#*         /proc/irq/X/smp_affinity                                                #
#*    1.2  配置参数                                                                 #
#*         CPU位图值                                                                #
#*                                                                                 #
#* 2. auto set RPS/RFS                                                             #
#*    2.1 配置文件                                                                  #
#*         /sys/class/net/<dev>/queues/rx-<n>/rps_cpus                             #
#*         /sys/class/net/<dev>/queues/rx-<n>/rps_flow_cnt                         #
#*         /proc/sys/net/core/rps_sock_flow_entries                                #
#*    2.2 配置参数                                                                  #
#*         rps_cpus = CPU位图值                                                     #
#*         rps_sock_flow_entries = 32768                                           #
#*         单队列网卡：rps_flow_cnt = rps_sock_flow_entries                          #
#*         多队列网卡：rps_flow_cnt = rps_sock_flow_entries / N  (N等于网卡队列数)    #
#* 3. irqbalance                                                                    #
#*         启动：systemctl start irqbalance                                          #
#*         停止：systemctl stop irqbalance                                           #
#***********************************************************************************#
interface=""
if [ $# -gt 1 ];then
    for arg in $@
    do
        if `echo ${arg} | grep -qE '^eth[0-9]+-[0-9]+$'`
        then
            interface="${interface} ${arg%-*}"
        fi
    done
fi
interface=`ls /sys/class/net | egrep "^eth[0-9]+$"` 
cpunum=`head -n1 /proc/interrupts|awk '{print NF}'`
tmp_smpfile="/tmp/smpfile"

# 方案1： 仅设置网卡中断亲和性，NIC中断和CPU核绑定
function onlysetup_smp_affinity()
{
    clean_all
    echo ""
    echo "startting setup balance rule 2: use only smp affinity"

    [ -f ${tmp_smpfile} ] && rm -f ${tmp_smpfile}

    for ifname in ${interface}
    do
        cat /proc/interrupts | grep ${ifname} | while read line
        do
            irqnum=`echo ${line} | awk -F ':' '{print $1}' | awk '{print $1}'`
            ifinfo=`echo ${line} | awk '{print $NF}'`
            echo "${ifinfo}:${irqnum}:/proc/irq/${irqnum}/smp_affinity_list" >> ${tmp_smpfile}
        done
    done

    declare -i cpu
    declare -i count=0
    cat ${tmp_smpfile} | while read line
    do
        ifinfo=`echo ${line} | awk -F ':' '{print $1}'`
        irqnum=`echo ${line} | awk -F ':' '{print $2}'`
        setfile=`echo ${line} | awk -F ':' '{print $3}'`
        cpu=$[${count}%${cpunum}]
        echo ${cpu} > ${setfile}
        echo "SMP Affinity: bind ${ifinfo} irq[${irqnum}] on CPU${cpu}"
        count=$[${count} + 1]
    done
    [ -f ${tmp_smpfile} ] && rm -f ${tmp_smpfile}
}

# 方案2：仅设置RPS，通过软件方式实现CPU负载均衡
function onlysetup_rps_rfs()
{
    clean_all
    echo ""
    echo "startting setup balance rule 3: use only rps/rfs"
	declare -i rfc_flow_cnt=0
	declare -i rps_sock_flow_entries=32768
    rps_cpus=`printf "%x" $((2**$cpunum-1))`

	for ifname in ${interface}
	do
		queuenum=`ls /sys/class/net/"${ifname}"/queues/ | grep rx | wc -l`
		if test ${queuenum} -eq 1 
		then 
			rfc_flow_cnt=${rps_sock_flow_entries}
		else
			rfc_flow_cnt=$[${rps_sock_flow_entries}/${queuenum}]	
		fi
		for rxdir in /sys/class/net/"${ifname}"/queues/rx-*
		do
			echo ${rps_cpus} >${rxdir}/rps_cpus
			echo ${rfc_flow_cnt} >${rxdir}/rps_flow_cnt
		done
        echo "RPS/RFS: set ${ifname} rps_cpus = ${rps_cpus}, rps_flow_cnt = ${rfc_flow_cnt}"
	done
    echo ${rps_sock_flow_entries} > /proc/sys/net/core/rps_sock_flow_entries 
    echo "RPS/RFS: set rps_sock_flow_entries = ${rps_sock_flow_entries}"
}

#方案3：仅启用irqbalance，系统自动调节NIC负载均衡
function only_irqbalance()
{
    clean_all
    echo ""
    echo "startting setup balance rule 4 : start service irqbalance"
    systemctl start irqbalance
}

#方案3：SMP中断亲和性与RPS配合使用
#方案说明：先执行方案1，使NIC硬中断大致在CPU间均衡；再针对单队列网卡，设置RPS负载均衡所有CPU。（多队列网卡，不启用RPS）
function smp_affinity_work_with_rps_rfs()
{
    echo ""
    echo "startting setup balance rule 1 : use both smp and rps"

	declare -i rfc_flow_cnt=32768
	declare -i rps_sock_flow_entries=32768
    declare -i cpu
    declare -i count=0

    rps_cpus=`printf "%x" $((2**$cpunum-1))`

    [ -f ${tmp_smpfile} ] && rm -f ${tmp_smpfile}
    for ifname in ${interface}
    do
        cat /proc/interrupts | grep ${ifname} | while read line
        do
            irqnum=`echo ${line} | awk -F ':' '{print $1}' | awk '{print $1}'`
            ifinfo=`echo ${line} | awk '{print $NF}'`
            echo "${ifinfo}:${irqnum}:/proc/irq/${irqnum}/smp_affinity_list" >> ${tmp_smpfile}
        done
    done
    cat ${tmp_smpfile} | while read line
    do
        ifinfo=`echo ${line} | awk -F ':' '{print $1}'`
        irqnum=`echo ${line} | awk -F ':' '{print $2}'`
        setfile=`echo ${line} | awk -F ':' '{print $3}'`
        cpu=$[${count}%${cpunum}]
        echo ${cpu} > ${setfile}
        echo "SMP Affinity: bind ${ifinfo} irq[${irqnum}] on CPU${cpu}"
        count=$[${count} + 1]
    done
    [ -f ${tmp_smpfile} ] && rm -f ${tmp_smpfile}

	for ifname in ${interface}
	do
		queuenum=`ls /sys/class/net/"${ifname}"/queues/ | grep rx | wc -l`
		if test ${queuenum} -ne 1 
		then 
            echo "RPS/RFS: ${ifname} is ${queuenum}-queue device , RPS disable"
            continue
		fi
		for rxdir in /sys/class/net/"${ifname}"/queues/rx-*
		do
			echo ${rps_cpus} >${rxdir}/rps_cpus
			echo ${rfc_flow_cnt} >${rxdir}/rps_flow_cnt
		done
        echo "RPS/RFS: set ${ifname} rps_cpus = ${rps_cpus}, rps_flow_cnt = ${rfc_flow_cnt}"
	done
    echo ${rps_sock_flow_entries} > /proc/sys/net/core/rps_sock_flow_entries 
    echo "RPS/RFS: set rps_sock_flow_entries = ${rps_sock_flow_entries}"
}

function clean_smp_affinity()
{
    [ -f ${tmp_smpfile} ] && rm -f ${tmp_smpfile}

    for ifname in ${interface}
    do
        cat /proc/interrupts | grep ${ifname} | while read line
        do
            irqnum=`echo ${line} | awk -F ':' '{print $1}' | awk '{print $1}'`
            ifinfo=`echo ${line} | awk '{print $NF}'`
            echo "${ifinfo}:/proc/irq/${irqnum}/smp_affinity_list" >> ${tmp_smpfile}
        done
    done
    cat ${tmp_smpfile} | while read line
    do
        setfile=`echo ${line} | awk -F ':' '{print $2}'`
        ifinfo=`echo ${line} | awk -F ':' '{print $1}'`
        echo 0 > ${setfile}
    done
    [ -f ${tmp_smpfile} ] && rm -f ${tmp_smpfile}

    echo "clean rule 2 : smp affinity over"
}


function clean_rps_rfs()
{
	for ifname in ${interface}
	do
		for rxdir in /sys/class/net/"${ifname}"/queues/rx-*
		do
			echo 0 >${rxdir}/rps_cpus
			echo 0 >${rxdir}/rps_flow_cnt
		done
	done
    echo 0 > /proc/sys/net/core/rps_sock_flow_entries 
    echo "clean rule 3 : rps/rfs over"
}

function clean_irqbalance()
{
    systemctl stop irqbalance
    echo "clean rule 4 : stop irqbalance over"
}

function clean_all()
{
    echo "cleanning all rules ..."
    clean_irqbalance
    clean_rps_rfs
    clean_smp_affinity
}
function help()
{
    echo "Usage: $0 <auto|smp|rps|irqbalance|cleanall>"
    echo "Setup  balance between NIC and CPU on multi-nic multi-cpu system"
    echo ""
    echo "   auto           rule 1: use both rps and smp affinity"
    echo "   smp            rule 2: only use smp affinity to bind NIC irq on CPU"
    echo "   rps            rule 3: only use rps to bind skb to CPU by soft way"
    echo "   irqbalance     rule 4: only use system service irqbalance by system self"
    echo "   cleanall       clean all rules"
}
case $1 in
    smp)
        onlysetup_smp_affinity;;
    rps)
        onlysetup_rps_rfs;;
    irqbalance)
        only_irqbalance;;
    auto)
        smp_affinity_work_with_rps_rfs;;
    cleanall)
        clean_all;;
    *)
        help;;
esac
```





