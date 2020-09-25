---
title: 内核链表2---hash链表
type: kernel
description: 内核开发者们设计了一套高效的链表操作方法，高效而简洁，直接拿来使用，不要重复造轮子
date: 2020-09-18 14:19:00
---

内核链表支持双向单链表和哈希链表，其设计简洁、高效，使用简单，简直是开发者的福音。这一套操作方法，不仅使用于内核态，稍加修改后也可以在应用层使用。这里对内核链表的使用方法简单做个总结记录。

修改后的list.h下载，不依赖任何头文件，[请点击这里下载](https://peiyake.com/src/list.h)


## HASH 链表使用

HASH链表结构示意图

![hash表示意图](/images/list_hash.png)


HASH链表和之前的内核双向链表不同在于，HASH的结构像一张`N x M`(N行,M列) 的表格。其中呢，对于链表头的使用 通常是这样定义的 `struct hlist_head heads[N]`。那当我们拿到一个数据时我们应该怎么插入呢？ 分两步:

* 计算数据插入行：根据数据某个key计算HASH，这个HASH值通常是 `hash = key % N`，获取到hash后，那么这个数据应该插入到表格的第**hash**行，其hlist_head 就是 `heads[hash]`
* 插入数据：使用接口宏 `hlist_add_head` 插入数据（只能从该行的首部插入）

## 接口函数（宏）

1. 数据结构

```
struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

2. `INIT_HLIST_NODE(struct hlist_node *h)`初始化hash头 
3. ` hlist_add_head(struct hlist_node *n, struct hlist_head *h)` 向hash表的某行插入数据（行内头插）
4. `void hlist_del(struct hlist_node *n)`删除指定节点
5. `hlist_for_each_entry(pos, head, member)` 遍历hash表某行
   * pos,数据结构类型指针
   * head,指向改行的hlist_head指针
   * member,hlist_node在数据结构中的成员名称
6. `hlist_for_each_entry_safe(pos, n, head, member)` 遍历hash表某行，安全方法(**推荐**)
   * pos,数据结构类型指针
   * n,指向hlist_node的指针（定义一个 hlist_node指针临时变量而已）
   * head,指向改行的hlist_head指针
   * member,hlist_node在数据结构中的成员名称

## 示例代码

* 创建一个HASH表，行数是 HASH_BUCKET = 32
* 向表中插入 HASH_BUCKET * 10 个数据
* 删除某些行的数据（练习）
* 使用安全遍历方法遍历数据并打印输出
* 使用非安全遍历方法遍历数据并打印输出
* 删除hash表所有数据，并释放内存
* 检测hash表每行的数据是否已经被清空，并打印输出

```
#include<stdio.h>
#include<malloc.h>
#include<string.h>
#include "list.h"

#define HASH_BUCKET     32

struct hlist_head heads[HASH_BUCKET];

struct hlist_data{
        int num;
        struct hlist_node list;
};

int hash_func(int num)
{
        return num % HASH_BUCKET;
}
void hash_head_init(struct hlist_head *head,int size)
{
        int i;
        for(i = 0;i < size;i ++){
                INIT_HLIST_HEAD(head++);
        }

        return ;
}
int main(int argc,char **argv)
{
        int loop,hash;
        struct hlist_data *ph;
        struct hlist_node *pn;

        hash_head_init(heads,HASH_BUCKET);

        /* 构造数据 */
        for(loop = 0; loop < HASH_BUCKET * 10; loop ++){
                ph = (struct hlist_data *)malloc(sizeof(struct hlist_data));
                ph->num = loop;
                hash = hash_func(ph->num);
                hlist_add_head(&ph->list,&heads[hash]);
        }
        printf("%7s\t  |\tdata\n","index");
        for(loop = 0; loop < HASH_BUCKET;loop ++){
                if(loop%5 == 1){
                        hlist_for_each_entry_safe(ph,pn,&heads[loop],list){
                                hlist_del(&ph->list);
                        }
                }
                printf("%7d\t-->>\t",loop);
                if(loop < HASH_BUCKET/2){
                        /* 非安全遍历方法 */
                        hlist_for_each_entry(ph,&heads[loop],list){
                                printf("%-5d",ph->num);
                        }
                        if(hlist_empty(&heads[loop])){
                                printf("刚才删除了");
                        }
                }else{
                        /* 安全遍历方法 */
                        hlist_for_each_entry_safe(ph,pn,&heads[loop],list){
                                printf("%-5d",ph->num);
                        }
                        if(hlist_empty(&heads[loop])){
                                printf("刚才删除了");
                        }
                }
                printf("\n");
        }

        printf("\n\n删除所有数据...\n");
        for(loop = 0; loop < HASH_BUCKET;loop ++){
                hlist_for_each_entry_safe(ph,pn,&heads[loop],list){
                        hlist_del(&ph->list);
                        free(ph);
                }
        }
        printf("\n\n再次检查hash表...\n\n");
        for(loop = 0; loop < HASH_BUCKET;loop ++){
                printf("HASH[%2d]是否为空? %s\n",loop,hlist_empty(&heads[loop]) ? "yes": "no");
        }

        return 0;
}
```
输出结果

```
[piak@localhost ~]$ ./a.out 
  index   |     data
      0 -->>    288  256  224  192  160  128  96   64   32   0    
      1 -->>    刚才删除了
      2 -->>    290  258  226  194  162  130  98   66   34   2    
      3 -->>    291  259  227  195  163  131  99   67   35   3    
      4 -->>    292  260  228  196  164  132  100  68   36   4    
      5 -->>    293  261  229  197  165  133  101  69   37   5    
      6 -->>    刚才删除了
      7 -->>    295  263  231  199  167  135  103  71   39   7    
      8 -->>    296  264  232  200  168  136  104  72   40   8    
      9 -->>    297  265  233  201  169  137  105  73   41   9    
     10 -->>    298  266  234  202  170  138  106  74   42   10   
     11 -->>    刚才删除了
     12 -->>    300  268  236  204  172  140  108  76   44   12   
     13 -->>    301  269  237  205  173  141  109  77   45   13   
     14 -->>    302  270  238  206  174  142  110  78   46   14   
     15 -->>    303  271  239  207  175  143  111  79   47   15   
     16 -->>    刚才删除了
     17 -->>    305  273  241  209  177  145  113  81   49   17   
     18 -->>    306  274  242  210  178  146  114  82   50   18   
     19 -->>    307  275  243  211  179  147  115  83   51   19   
     20 -->>    308  276  244  212  180  148  116  84   52   20   
     21 -->>    刚才删除了
     22 -->>    310  278  246  214  182  150  118  86   54   22   
     23 -->>    311  279  247  215  183  151  119  87   55   23   
     24 -->>    312  280  248  216  184  152  120  88   56   24   
     25 -->>    313  281  249  217  185  153  121  89   57   25   
     26 -->>    刚才删除了
     27 -->>    315  283  251  219  187  155  123  91   59   27   
     28 -->>    316  284  252  220  188  156  124  92   60   28   
     29 -->>    317  285  253  221  189  157  125  93   61   29   
     30 -->>    318  286  254  222  190  158  126  94   62   30   
     31 -->>    刚才删除了


删除所有数据...


再次检查hash表...

HASH[ 0]是否为空? yes
HASH[ 1]是否为空? yes
HASH[ 2]是否为空? yes
HASH[ 3]是否为空? yes
HASH[ 4]是否为空? yes
HASH[ 5]是否为空? yes
HASH[ 6]是否为空? yes
HASH[ 7]是否为空? yes
HASH[ 8]是否为空? yes
HASH[ 9]是否为空? yes
HASH[10]是否为空? yes
HASH[11]是否为空? yes
HASH[12]是否为空? yes
HASH[13]是否为空? yes
HASH[14]是否为空? yes
HASH[15]是否为空? yes
HASH[16]是否为空? yes
HASH[17]是否为空? yes
HASH[18]是否为空? yes
HASH[19]是否为空? yes
HASH[20]是否为空? yes
HASH[21]是否为空? yes
HASH[22]是否为空? yes
HASH[23]是否为空? yes
HASH[24]是否为空? yes
HASH[25]是否为空? yes
HASH[26]是否为空? yes
HASH[27]是否为空? yes
HASH[28]是否为空? yes
HASH[29]是否为空? yes
HASH[30]是否为空? yes
HASH[31]是否为空? yes
```