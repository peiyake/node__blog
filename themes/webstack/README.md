# hexo-theme-webstack

本博客是基于hexo + github page来搭建的，博主对所用主题进行了一些改动后才呈现出当前样子。由于博主非前端出身，本博客的一些实现只求
实现功能，不求代码细节。

### 快速搭建Hexo-Demo站

安装 [Node.js](https://nodejs.org/en/)

安装 [Hexo](https://hexo.io/zh-cn/)

安装git工具，Window用户可安装[Git for windows](https://gitforwindows.org/)
   
    $ npm install hexo-cli -g

快速开始 `hexo 静态页面`

```
$ cd e:
$ mkdir hexo_demo && cd hexo_demo     //创建一个空白目录，并切换到空白目录hexo_demo
$ hexo init
$ npm install
$ hexo server
```

完成上述步骤，此时打开本地web页面 http://localhost:4000，此时hexo默认的静态站点就建好了。如图：

<img src="hexo_demo.png">

### 搭建本站

本站主要是采用了Hexo官方[主题](https://hexo.io/themes/)中的`webstack`，由于`webstack`仅仅是一个网站导航的样式，没有连接文章，但是博主比较喜欢该主题
的布局风格，所以自己对该主题改造成了适合作为博文站点的主题[hexo-theme-webstack](git@github.com:peiyake/hexo-theme-webstack.git)。 下面就关于本站的
这个主题使用进行说明.

采用 `hexo-theme-webstack`主题：

```
$ cd hexo_demo/themes
$ git clone git@github.com:peiyake/hexo-theme-webstack.git
```

修改 `hexo_demo/_config.yml`

```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
#theme: landscape               # 注释该行
theme: hexo-theme-webstack      # 把主题修改
```

重启Hexo Server

    $ hexo generate
    $ hexo server

再次打开本地web页面 http://localhost:4000，如图：

<img src="hexo_piak.png">

### hexo-theme-webstack 使用说明

* 新增或删除左侧导航目录

根据你自己的需求，修改 `_config.yml`中 menu的配置

```
menu:
  - name: 建站指南
    englishName: guide      # 英文名，用来给文章分类（中文的分类名称在程序中存在编码问题，所以这里都设置一个英文名称）
    icon: fas fa-book
    config: devTools
  - name: 一级目录
    icon: fas fa-th-list
    submenu:
      - name: 二级目录
        icon: fas fa-folder
        englishName: linuxc
        config: devTools
      - name: 二级目录
        icon: fas fa-folder
        englishName: linuxc
        config: devTools
      - name: 二级目录
        icon: fas fa-folder
        englishName: linuxc
        config: devTools
  - name: 有趣的灵魂
    englishName: life
    icon: fas fa-book
    config: devTools
```

* 新增文章 

修改文章模板格式 `hexo_demo/scaffolds/post.md`

```
---
title: {{ title }}
date: {{ date }}
type: <请选择分类：guide、linuxc、net、vpp、sql、shell、kernel、system、software、manage、life ...>
description: <请输入文章概述...>
---
```

然后，执行下面命令新建一片文章

```
$ hexo new hello
```

执行后，会生成 `hexo_demo/source/_posts/hello.md` ，示例如下：

```
$ cat source/_posts/hello.md
---
title: hello
type: <请选择分类：guide、linuxc、net、vpp、sql、shell、kernel、system、software、manage、life ...>
description: <请输入文章概述...>
date: 2020-06-28 17:58:11
---
```

  * `title` : 就是将来文章列表中显示的文章标题
  * `type` ： 是文章的分类
  * `description` ： 文章的简介描述，将来会显示在文章列表中每篇文章的一个简单介绍

_当然，你也可以直接在 `hexo_demo/source/` 目录下直接新建 `xxx.md`的方式新增文章，而且可以设置目录，把统一类别的文章放在同一个文件夹下，`hexo new` 的方法不能指定文章类别.通常建议自己手动新建文章_
