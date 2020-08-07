---
title: OpenWRT的feed机制
type: openwrt
description: 在OpenWRT中，一个 "feed"就是一系列在同一位置的包的合集。
date: 2020-07-09 16:23:42
---

官方说明：[https://openwrt.org/docs/guide-developer/feeds](https://openwrt.org/docs/guide-developer/feeds)

> In OpenWrt, a “feed” is a collection of packages which share a common location. Feeds may reside on a remote server, in a version control system, on the local filesystem, or in any other location addressable by a single name (path/URL) over a protocol with a supported feed method. 
>
> 在OpenWrt中，一个“feed”就是一系列在同一位置的包的合集。“feeds”可能位于一个远程服务器、一个版本控制系统、本地文件系统或者其他任何地方，（path/URL）只要能被feed method 支持就可以。

## Feed配置文件

默认采用`<openwrt dir>/feed.conf`文件作为配置，如果不存在则采用默认配置文件 `<openwrt dir>/feeds.conf.default` 。

`<openwrt dir>/feeds.conf.default`文件的格式：

* 每一行为一个 `feeds`配置
* 空白行和以 `#` 开头的行会被忽略
* 每行由三个被空格分隔的部分组成，分别是：`feed method`、`feed name`、`feed source`

默认的配置文件 `<openwrt dir>/feeds.conf.default`，如下：

```
src-git packages https://git.openwrt.org/feed/packages.git
src-git luci https://git.openwrt.org/project/luci.git
src-git routing https://git.openwrt.org/feed/routing.git
src-git telephony https://git.openwrt.org/feed/telephony.git
#src-git video https://github.com/openwrt/video.git
#src-git targets https://github.com/openwrt/targets.git
#src-git management https://github.com/openwrt-management/packages.git
#src-git oldpackages http://git.openwrt.org/packages.git
#src-link custom /usr/src/openwrt/custom-feed
```


**feed method**

* `src-bzr`:	    Data is downloaded from the source path/URL using bzr
* `src-cpy`:	    Data is copied from the source path. The path can be specified as either relative to OpenWrt repository root or absolute.
* `src-darcs`: 	    Data is downloaded from the source path/URL using darcs
* `src-git`: 	    Data is downloaded from the source path/URL using git as a shallow (depth of 1) clone
* `src-git-full`: 	Data is downloaded from the source path/URL using git as a full clone
* `src-gitsvn`: 	    Bidirectional operation between a Subversion repository and git
* `src-hg`: 	        Data is downloaded from the source path/URL using hg
* `src-link`: 	    A symlink to the source path is created. The path must be absolute.
* `src-svn`: 	    Data is downloaded from the source path/URL using svn

**feed name**

`feed name` 是feeds的名称标识

**feed source**

`feed source`指明feeds的路径（path/URL）。根据 `feed method` 设置相应的 `feed source`


## feed使用

feeds命令位于 `<openwrt dir>/scripts/feeds`，命令帮助如下：

```
[root@localhost openwrt]# ./scripts/feeds 
Usage: ./scripts/feeds <command> [options]

Commands:
        list [options]: List feeds, their content and revisions (if installed)
        Options:
            -n :            List of feed names.
            -s :            List of feed names and their URL.
            -r <feedname>:  List packages of specified feed.
            -d <delimiter>: Use specified delimiter to distinguish rows (default: spaces)
            -f :            List feeds in feeds.conf compatible format (when using -s).

        install [options] <package>: Install a package
        Options:
            -a :           Install all packages from all feeds or from the specified feed using the -p option.
            -p <feedname>: Prefer this feed when installing packages.
            -d <y|m|n>:    Set default for newly installed packages.
            -f :           Install will be forced even if the package exists in core OpenWrt (override)

        search [options] <substring>: Search for a package
        Options:
            -r <feedname>: Only search in this feed

        uninstall -a|<package>: Uninstall a package
        Options:
            -a :           Uninstalls all packages.

        update -a|<feedname(s)>: Update packages and lists of feeds in feeds.conf .
        Options:
            -a :           Update all feeds listed within feeds.conf. Otherwise the specified feeds will be updated.
            -i :           Recreate the index only. No feed update from repository is performed.
            -f :           Force updating feeds even if there are changed, uncommitted files.

        clean:             Remove downloaded/generated files.
```

通常在我们首次使用openwrt时，在`git clone` 完代码后，要紧接着执行 `./scripts/feeds update -a` 和 `./scripts/feeds/install -a`，根据上述帮助信息，可知这两条命令的意思是：

`./scripts/feeds update -a`: 根据 feed.conf 的配置，使用 `feed method`到 `feed source`获取相应文件，并保存到 `<openwrt dir>/feeds/<feed name>` 下。（注意：此时仅仅是获取了相应的feeds文件，但是并没有合并到系统构建目录 `<openwrt dir>/packages`）

`./scripts/feeds install -a`: 对所有的feeds合并到 `<openwrt dir>/packages/feeds`，并在目录下创建feed包的软连接，指向feed包

例如：

```
[root@localhost openwrt]# pwd
/root/openwrt
[root@localhost openwrt]# cd package/feeds/
[root@localhost feeds]# ls
freifunk  luci  packages  routing  telephony
[root@localhost feeds]# cd luci/
[root@localhost luci]# ls -l
total 8
lrwxrwxrwx 1 root root 43 Jul 13 16:25 csstidy -> ../../../feeds/luci/contrib/package/csstidy
lrwxrwxrwx 1 root root 36 Jul 13 16:25 luci -> ../../../feeds/luci/collections/luci
lrwxrwxrwx 1 root root 45 Jul 13 16:25 luci-app-acl -> ../../../feeds/luci/applications/luci-app-acl
lrwxrwxrwx 1 root root 46 Jul 13 16:25 luci-app-acme -> ../../../feeds/luci/applications/luci-app-acme
```

feed 其它命令使用举例：

./scripts/feeds install -a 	安装所有包（所有feeds下的包）
./scripts/feeds install luci 	仅安装名为 luci的 包
./scripts/feeds install -a -p luci 	安装名为luci的feeds下的所有包
./scripts/feeds update packages luci  仅更新feeds为luci下的包

其它命令 `search`、`list` 、`uninstall` 可直接看命令帮助。

## 自定义feeds

这里我们使用绝对路径作为 `feed source`

1. 创建文件夹 `~/myopenwrt_pkg`，并创建两个自文件夹`mypkg1` 和 `mypkg2`，目录结构如下：

```
[root@localhost myopenwrt_pkg]# pwd
/root/myopenwrt_pkg
[root@localhost myopenwrt_pkg]# ls
mypkg1  mypkg2
[root@localhost myopenwrt_pkg]# tree 
.
|-- mypkg1
|   `-- Makefile
`-- mypkg2
    `-- Makefile
```

`Makefile`的内容如下：

`mypkg1/Makefile`
```
include $(TOPDIR)/rules.mk

PKG_NAME:=mypkg1
PKG_VERSION:=0.0.1
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
#PKG_SOURCE_URL:=http://
#PKG_HASH:=1ad080674e9f974217b3a703e7356c6c8446dc5e7b2014d0d06e1bfaa11b5041

PKG_MAINTAINER:=Just for test <1029582431@qq.com>
PKG_LICENSE:=GPL-2.0-or-later
PKG_LICENSE_FILES:=LICENSE

PKG_INSTALL:=1
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk

define Package/mypkg1
  #SECTION:=net
  CATEGORY:=Network
  TITLE:=title of mypkg1
  SUBMENU:=New Folder
  DEPENDS:=+libreadline +libpthread
endef


define Package/mypkg1/install
        # do nothing
endef

define Package/mypkg1/install
        # do nothing
endef

$(eval $(call BuildPackage,mypkg1))
```

`mypkg2/Makefile`
```Makefile
include $(TOPDIR)/rules.mk

PKG_NAME:=mypkg2
PKG_VERSION:=0.0.1
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
#PKG_SOURCE_URL:=http://
#PKG_HASH:=1ad080674e9f974217b3a703e7356c6c8446dc5e7b2014d0d06e1bfaa11b5041

PKG_MAINTAINER:=Just for test <1029582431@qq.com>
PKG_LICENSE:=GPL-2.0-or-later
PKG_LICENSE_FILES:=LICENSE

PKG_INSTALL:=1
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk

define Package/mypkg2
  CATEGORY:=Network
  TITLE:=title of mypkg2
  SUBMENU:=New Folder
  DEPENDS:=+libreadline +libpthread
endef



define Package/mypkg2/install
        # do nothing
endef

define Package/mypkg2/install
        # do nothing
endef

$(eval $(call BuildPackage,mypkg2))
```

2. 修改 `<<openwrt dir>/feeds.conf.default>`，追加如下内容

```
src-link myopenwrt_pkg /root/myopenwrt_pkg
```

3. 更新并安装feed

```
cd <openwrt dir>
./scripts/feeds update myopenwrt_pkg 
./scripts/feeds install -a -p myopenwrt_pkg 
```

4. `make menuconfig` 查看

![make menuconfig](/images/menuconfig.png)

