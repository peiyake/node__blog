---
title: GCC之'__attribute__'用法
type: shell
description: GCC关键字 __attribute__ 用来对变量、函数、结构体、联合体和C++类成员进行一些特殊属性设置
date: 2020-06-22 19:10:42
---

# GCC之 '_\_\_attribute\_\__'用法

The keyword \_\_attribute\_\_ allows you to specify special properties of variables, function parameters, or structure, union, and, in C++, class members. 

GCC关键字 \_\_attribute\_\_ 用来对变量、函数、结构体、联合体和C++类成员进行一些特殊属性设置

## _\_\_attribute\_\__ 语法

An attribute specifier is of the form **\_\_attribute\_\_ ((_attribute-list_))**. An attribute list is a possibly empty comma-separated sequence of attributes, where each attribute is one of the following:

* Empty. Empty attributes are ignored.
* An attribute name (which may be an identifier such as unused, or a reserved word such as const).
* An attribute name followed by a parenthesized list of parameters for the attribute. These parameters take one of the following forms:
    - An identifier. For example, mode attributes use this form.
    - An identifier followed by a comma and a non-empty comma-separated list of expressions. For example, format attributes use this form.
    - A possibly empty comma-separated list of expressions. For example, format_arg attributes use this form with the list being a single integer constant expression, and alias attributes use this form with the list being a single string constant. 

attribute语法格式是 `__attribute__ ((attribute-list))`，其中 **attribute-list**是一个由空格 或者 逗号分隔的属性序列，这些序列格式如下：

* 空白。空白属性会被忽略
* 一个属性名称（可能是一个标识符，例如 **unused**，或者是保留字，例如**const**）
* 一个属性名称后面紧跟着一对括号，括号内是属性参数，这些参数符合下面其中一种格式：
  - 一个标识符。例如：**mode**属性
  - 一个标识符，后面紧跟着一个由逗号分隔的非空表达式组成的一个小节，例如：属性**format**
  - 或者是一个 逗号分隔的非空表达式序列。例如：**format_arg**

`__attribute__ ((attribute-list))`举例：


`attribute mode`

例如，我们定义一些数据类型宏，使得用这些宏的定义的变量无论在哪种平台上数据长度都是一样的。
```c
#include <stdio.h>

typedef unsigned int u_int8_t __attribute__ ((__mode__ (__QI__)));
typedef unsigned int u_int16_t __attribute__ ((__mode__ (__HI__)));
typedef unsigned int u_int32_t __attribute__ ((__mode__ (__SI__)));
typedef unsigned int u_int64_t __attribute__ ((__mode__ (__DI__))); 

int main(int argc,char **argv)
{
        printf("sizeof(u_int8_t)  = %d\n",sizeof(u_int8_t));
        printf("sizeof(u_int16_t) = %d\n",sizeof(u_int16_t));
        printf("sizeof(u_int32_t) = %d\n",sizeof(u_int32_t));
        printf("sizeof(u_int64_t) = %d\n",sizeof(u_int64_t));

        return 0;
}
```
编译运行：

```
[root@localhost ~]# gcc attribute.c -o attr
[root@localhost ~]# ./attr 
sizeof(u_int8_t)  = 1
sizeof(u_int16_t) = 2
sizeof(u_int32_t) = 4
sizeof(u_int64_t) = 8
```

* BI：“Bit” mode represents a single bit, for predicate registers.只占用 1bit
* QI：“Quarter-Integer” mode represents a single byte treated as an integer. 占用 1 byte（1/4 整型）
* HI：“Half-Integer” mode represents a two-byte integer. 占用2byte
* SI：“Single Integer” mode represents a four-byte integer. 占用4byte
* DI：“Double Integer” mode represents an eight-byte integer. 占用8byte
* [更多mode类型...](https://gcc.gnu.org/onlinedocs/gccint/Machine-Modes.html#Machine-Modes)