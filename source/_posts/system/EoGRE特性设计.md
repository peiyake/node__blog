---
title: EoGRE特性设计
type: system
description: 
date: 2020-11-09 09:10:42
---

Linux内核已经支持了EoGRE特性，根据前文讲述[GRE协议--EoGRE](https://peiyake.com/2020/09/27/net/gre%E5%8D%8F%E8%AE%AE--eogre/)，实现Eogre功能通常是将创建的虚接口加入到bridge下，如此在隧道两端的虚接口都加入桥下时，可以实现两端的二层桥接；但是有个问题，如果桥下收到的报文的目的IP跟桥接口不在同一网段时，那么该报文不会在桥下扩散，也就是无法进行隧道转发。  目前有个需求：

1. 对于所有报文无差别的通过EoGRE隧道转发出去
2. 本机报文可接收

经过深入研究，发现只需要对linux桥模块稍加修改，就可以完整实现该功能，并且可以实现报文携带8021Q。

## 基础环境

* Linux-4.4

## 设计说明

仔细阅读linux桥的代码发现，在bridge对报文进行桥内转发时，函数br_handle_frame_finish会调用一个预先插入的HOOK --**br_get_dst_hook**，这个hook默认是NULL，根据逻辑，桥转发会将报文转发给该hook的返回值。代码如下：

![/images/br_dst_hook.png]

那么，我们只要写个内核模块实现这个HOOK，并且将我们想要转发的接口从FDB表中查找出来，作为HOOK的返回值，那么就可以实现桥下所有报文都通过这个接口转发出去了（我们这里的目标是EoGRE隧道接口）。当然，我们也可以把隧道接口的VLAN子接口加入到桥下，报文通过隧道VLAN子接口转发的话，就可以携带VLAN标签了。  

经过测试，这个方案是可行的 。

## 模块代码

```
#include <linux/module.h>
#include <linux/kernel.h>
#include <net/sock.h>
#include <linux/in.h>
#include <linux/if_bridge.h>
#include <linux/netdevice.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include "br_private.h"

#define MAC_FMT     "%02X:%02X:%02X:%02X:%02X:%02X"
#define MAC_ARG(x)  ((unsigned char*)(x))[0],((unsigned char*)(x))[1],((unsigned char*)(x))[2], \
                    ((unsigned char*)(x))[3],((unsigned char*)(x))[4],((unsigned char*)(x))[5]
static unsigned long  tunnel_vlan = 1;
static u32 debug = 0;
struct proc_dir_entry *root_entry = NULL;
struct proc_dir_entry *vlan_entry = NULL;

#define pr_log(fmt,arg...)  do {  \
    if(debug) printk(fmt,##arg);  \
  }while(0)


static ssize_t proc_proc_write(struct file *f, const char __user *buf, size_t s, loff_t *ppos)
{
  char line[32] = {0};
  char *ptr;
  int len = 0;
  unsigned long vlan;
  
  copy_from_user(line,buf,sizeof(line));

	len = strlen(line);
	ptr = line + len - 1;
	if (len && *ptr == '\n')
		*ptr = '\0';

  if(0 == strncmp(line,"debug_on",strlen("debug_on"))){
    debug = 1;
  }else if(0 == strncmp(line,"debug_off",strlen("debug_off"))){
    debug = 0;
  }else{
    vlan = simple_strtoul(line, &ptr, 0);
    if((vlan >= 1) && (vlan <= 4096))
      tunnel_vlan = vlan;
  }
  return s;
}

static int proc_proc_show(struct seq_file *seq, void *v)
{
  seq_printf(seq,"%lu\n",tunnel_vlan);
  return 0;        
}

static int proc_proc_open(struct inode *inode, struct file *file)
{
  return single_open(file, proc_proc_show,NULL);
}


static struct file_operations vlan_fops = {
  .open  = proc_proc_open,
  .read  = seq_read,
  .write  = proc_proc_write,
  .llseek  = seq_lseek,
  .release = single_release,
};


void proc_node_init(void)
{
  root_entry = proc_mkdir("br_hook",NULL);
  if(NULL == root_entry){
    pr_log("proc mkdir 'br_hook' fail");
    return ;
  }
  /* create /proc/br_hook/vlan */
  vlan_entry = proc_create("vlan", 0x644, root_entry,&vlan_fops);
  if(NULL == vlan_entry){
    pr_log("proc mkdir 'br_hook/vlan' fail");
    return ;
  }
  return ;
}
struct net_bridge_port* br_external_forward(
          const struct net_bridge_port *src,
          struct sk_buff **pskb)
{
  struct net_bridge_fdb_entry *dest_fdb = NULL, *skb_dest_fdb = NULL;
  struct net_bridge_port *dst= NULL;
  struct net_bridge_port *p = NULL;
	struct net_bridge *br = NULL;
  struct net_device *dev = NULL;
  struct sk_buff *skb = *pskb;
	const unsigned char *dest;
	const unsigned char *skb_dest;
	struct ethhdr *l2h = NULL;
  u16 vid = (u16)tunnel_vlan;
  char ifname[32] = {0};
  
  if((NULL == src) || (NULL == skb)){
     goto end;
  }

  if(vid == 1){
    sprintf(ifname,"tunnel");
  }else{
    sprintf(ifname,"tunnel.%u",vid);
  }
  dev = dev_get_by_name(&init_net,ifname);
  if(!dev){
    pr_log("Error,get dev '%s' fail\n",ifname);
    goto end;
  }

  l2h = eth_hdr(skb);

	p = br_port_get_rcu(dev);
	if(!p){
	  pr_log("Error,br_port_get_rcu fail\n");
	  goto end;
	}
	  
	br = p->br;
	if(!br || (src->br != br)){
    pr_log("Error,br is %s src_br:%s,br:%s\n",
            !skb ? "NULL," : "not NULL, but",
            src->br->dev->name,
            br->dev->name);
	}
	dest = dev->dev_addr;
	skb_dest = eth_hdr(skb)->h_dest;

  /*
    Find fdb info by dest_mac in skb.
    1. if find, just return the dest fdb port and forward ,but 'is_local' port
    2  if not,find fdb again by the tunnel mac if src dev is not tunnel dev (it's means 
        skb is send from sta , we just forward the package to tunnel dev). othwise, tunnel
        fdb can't find in fdb table, make dst = NULL, means this package maybe  invaild and
        send back to the bridge .
  */
  skb_dest_fdb = __br_fdb_get(br,skb_dest,vid);

  if(skb_dest_fdb){
    if (skb_dest_fdb->is_local){
      pr_log("skb_dest_fdb is_local,dest["MAC_FMT"]",MAC_ARG(skb_dest));
      dst = NULL;
    }else{
      dst = skb_dest_fdb->dst;
      pr_log("skb_dest_fdb is not local,dest["MAC_FMT"]",MAC_ARG(skb_dest));
    }
  }else{
    if(src->dev != dev){
      dest_fdb = __br_fdb_get(br,dest,vid);
      dst = dest_fdb ? dest_fdb->dst : NULL;
      pr_log("dest fdb is %s",dest_fdb ? "not NULL" : "NULL");
    }else{
      pr_log("src->dev[%s],same with dev[%s]",src->dev->name,dev->name);
      dst = NULL;
    }
  }
end:
  if(dev)
    dev_put(dev);
  return dst;
}

static int __init dst_hook_init(void)
{
  proc_node_init();
  br_get_dst_hook = br_external_forward;
  printk("dst_hook_init ok\n");
  return 0;
}

static void __exit dst_hook_exit(void)
{
  br_get_dst_hook = NULL;
  if(NULL != vlan_entry) 
    proc_remove(vlan_entry);
  if(NULL != root_entry) 
    proc_remove(root_entry);
  printk("dst_hook_exit ok\n");
  return ;
}

module_init(dst_hook_init);
module_exit(dst_hook_exit);
//module_param(ifname,charp,0644);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Mr.Piak,<1029582431@qq.com>");
MODULE_DESCRIPTION("Hook for external bridge forwarding logic");

```