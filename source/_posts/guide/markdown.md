---
title: Markdown语法介绍
type: guide
description: github风格的markdown语法介绍
date: 2020-06-30 17:08:46
---

Hexo采用github风格的[Markdown](https://daringfireball.net/projects/markdown/syntax)语法，本文简单介绍一下使用方法，免得自己忘记。

## 标题

```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题，最多6级标题
```
效果

#### 二级标题
##### 三级标题
###### 四级标题，最多6级标题


## 引用

```
>这是第一段引用，开启另一段引用的话，中间空行

>这是第二段引用
```

>这是第一段引用，开启另一段引用的话，中间空行

>这是第二段引用

```
>这是第一行，引用内部如果需要换行的话，加一个空行的 '<'
>
>换行后,这是引用中的第二行,
>这还是第二行
>
>再次换行，这是第三行
```
>这是第一行，引用内部如果需要换行的话，加一个空行的 '<'
>
>换行后,这是引用中的第二行
>这还是第二行
>
>再次换行，这是第三行


## 无序列表

```
* 枯藤
* 老树
* 昏鸦
```
* 枯藤
* 老树
* 昏鸦

## 有序列表

```
+ 小桥
+ 流水
+ 人家
  - 古道
  - 西风
  - 瘦马
```
+ 小桥
+ 流水
+ 人家
  - 古道
  - 西风
  - 瘦马

## 代码块

```
行内嵌入代码，`print hello`
```
行内嵌入代码，`print hello`

````
一段代码，这样写
```
#include <stdio.h>
int main(int argc,const char **argv)
{
    printf("Hello world!\n");
    return 0;
}
```
````

一段代码，这样写
```
#include <stdio.h>
int main(int argc,const char **argv)
{
    printf("Hello world!\n");
    return 0;
}
```

## 图片

```
使用相对路径
[github的logo](/images/github_logo.png)
使用网址
[github的logo](https://peiyake.com/images/github_logo.png)
使用img标签
<img src="https://peiyake.com/images/github_logo.png">github的logo</img>
```
使用相对路径

![github的logo](/images/github_logo.png)

使用网址

![github的logo](https://peiyake.com/images/github_logo.png)

使用img标签

<img src="https://peiyake.com/images/github_logo.png">github的logo</img>

## 链接

```
[百度一下](https://www.baidu.com/)
[Mr.Piak的博客](https://peiyake.com/)
```

[百度一下](https://www.baidu.com/)

[Mr.Piak的博客](https://peiyake.com/)

## 水平线

```
三个或更多*、- 就会有水平线
***
---
```
三个或更多*、- 就会有水平线

***

---

## 页内跳转

```
先设置锚点，markdown的锚点一般都是标题，这里我们跳到本文的开头处 “标题”

[跳到标题](#标题)
```
[跳到标题](#标题)

## 表格

```
|说明|程序名称|功能|rpm包|
|-------------------|-------------------|---------------------------|-------|
| 主服务器进程| abrtd| ABRT工具集主进程，守护进程，用来收集crash信息 |abrt-2.1.11-50.el7.centos.x86_64 |
| core-dump插件|abrt-hook-ccpp | abrt core-dump 信息收集插件，收集完成会发送给abrtd| abrt-addon-ccpp-2.1.11-50.el7.centos.x86_64|
| oops插件|abrt-dump-oops|abrt内核oops监控插件|abrt-addon-kerneloops-2.1.11-50.el7.centos.x86_64|
| kernel-panic插件|abrt-harvest-vmcore|abrt内核panic监控插件 |abrt-addon-vmcore-2.1.11-50.el7.centos.x86_64|
```

|说明| 程序名称 | 功能 | rpm包 |
|------|---------|-------|-------|
| 主服务器进程 | abrtd| ABRT工具集主进程，守护进程，用来收集crash信息 |abrt-2.1.11-50.el7.centos.x86_64 |
| core-dump插件 |abrt-hook-ccpp | abrt core-dump 信息收集插件，收集完成会发送给abrtd| abrt-addon-ccpp-2.1.11-50.el7.centos.x86_64|
| oops插件 |abrt-dump-oops|abrt内核oops监控插件|abrt-addon-kerneloops-2.1.11-50.el7.centos.x86_64|
| kernel-panic插件 |abrt-harvest-vmcore|abrt内核panic监控插件 |abrt-addon-vmcore-2.1.11-50.el7.centos.x86_64|