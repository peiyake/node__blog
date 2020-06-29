---
title: getopt_long用法
type: linuxc
description: 可变参数处理函数getopt_long用法解析
date: 2020-06-22 19:10:42
---

`getopt_long`是命令行参数处理函数，使用该接口函数可以实现通用的参数写法。

#### 函数原型

```
#include <getopt.h>

int getopt_long(int argc, char * const argv[],
        const char *optstring,
        const struct option *longopts, int *longindex);
```

#### `struct option`

```
struct option {
    const char *name;
    int         has_arg;
    int        *flag;
    int         val;
};
```

* `name` ：长选项参数名称
* `has_arg` ：
    * `no_argument` 或者 `0` ：表示这个选项不需要参数
    * `required_argument ` 或者 `1` ：表示这个选项需要额外的参数
    * `optional_argument ` 或者 `2` ： 表示这个选项的参数是可选的
* `flag` ：如果 `flag=NULL` 这种情况，`getopt_long` 的返回值是 `val` （这种情况，通常返回值设置为短选项）；否则，`flag` 应该指向一个可用的地址，`getopt_long`将返回0，同时将 `val`的值拷贝到 `flag`指向的地址。
* `val` ： 根据 `flag`设置的值不同，这个 `val`的使用方式不同

#### 全局变量

* `optarg` ： 表示对应选项的入参
* `optind` ： 表示下一个将要被处理的参数在argv中的下标值
* `opterr` ： 如果opterr = 0，在getopt、getopt_long、getopt_long_only遇到错误将不会输出错误信息到标准输出流。opterr在非0时，向屏幕输出错误
* `optopt` ： 表示没有被标识的选项

#### 函数入参

* `argc` `argv` ： 这个直接使用main函数的入参
* `optstring` ： 这个用来设置短选项。例如：`"abc:"` 一个字符代表一个选项，在字符后面加一个':'表示该选项带一个参数，字符后带两个':'表示该选项带可选参数(参数可有可无)，若有参数，optarg指向该该参数，否则optarg为0
* `longopts` ： 通常设置一个全局变量，请参考 `struct option`
* `longindex` ： longindex非空，它指向的变量将记录当前找到参数符合longopts里的第几个元素的描述，即是longopts的下标值。

#### 返回值

如果入参是短选项，那么 `getopt_long` 返回该短选项字符；如果入参是长选项，那么根据 `flag` 的设置，`getopt_long`返回0或者`val`；如果入参无法识别，`getopt_long` 返回 `?`；如果参数
解析完毕，`getopt_long` 返回 `-1`；如果选项需要额外参数，但是实际没有传入额外参数，那么 `getopt_long` 返回值根据 `optstring` 的值，如果`optstring`第一个字符是`:`，那么此时`getopt_long`返回 `:`，否则返回 `?`。

## 例子程序

```
#include <stdio.h>
#include <getopt.h>
#include <string.h>

#define RUN_MODE_SERVER     1
#define RUN_MODE_CLIENT     2
char ipaddrs[256];
int run_mode = -1;
static struct option longopts[] = {
    {"server",      no_argument,        &run_mode,   RUN_MODE_SERVER},
    {"client",      no_argument,        &run_mode,   RUN_MODE_CLIENT},
    {"maddr",       required_argument,  NULL,   'a'},
    {"help",        no_argument,        NULL,   'h'},
    {0,0,0,0}
};
void help(char *name)
{
    printf("Usage: %s [option]\n",name);
    printf(" option\n");
    printf("    -s|--server         Test the mylibc api function\n");
    printf("    -c|--client         \n");
    printf("    -a|--maddr  <[x.x.x.x][,x.x.x.x]>   x.x.x.x is ipv4 mulicast addr,multiple addrs should split by ','\n");
    printf("    -h|--help           \n");
    printf("\n");
}
int main(int argc,const char **argv)
{
    char c;
    memset(ipaddrs,0,sizeof(ipaddrs));
    while(1) 
    {
        int option_index = 0;

        c = getopt_long(argc, argv, ":scha:",longopts, &option_index);
        if (c == -1)
            break;
        switch (c)
        {
        case 0:
            break;

        case 's':
            run_mode = RUN_MODE_SERVER;
            break;

        case 'c':
            run_mode = RUN_MODE_CLIENT;
            break;
        case 'a':
            strcpy(ipaddrs,optarg);
            break;
        case 'h':
            help(argv[0]);
            break;
        case ':':
            fprintf(stderr,"option '%c' need an argument.\n",optopt);
            help(argv[0]);
        case '?':
            fprintf(stderr,"unknow input param '%s'\n",argv[optind - 1]);
            help(argv[0]);
            break;

        default:
            printf("?? getopt returned character code 0%o ??\n", c);
            help(argv[0]);
        }
    }
    printf("run_mode[%d]:%s, ipaddrs:%s\n",run_mode,(run_mode == 1) ? "server": "client",ipaddrs);
    return 0;
}
```