---
title: SDK编译整理
type: wifi
description: 基于csu2，的release.note整理编译过程
date: 2020-07-09 16:23:42
---

根据SDK `csu2`的release_notes.pdf 进行环境编译时，发现存在诸多问题，这里把自己整个`跳坑、填坑`过程整理一下，方便自己查阅。

## SDK编译过程

本文编译环境

* DISTRIB_ID=Ubuntu
* DISTRIB_RELEASE=16.04
* DISTRIB_CODENAME=xenial
* DISTRIB_DESCRIPTION="Ubuntu 16.04.6 LTS"

### 获取SDK压缩包并解压

```
cd ~
xz -d r11.1_xxxxxx.tar.xz
mkdir abc
tar -xf r11.1_xxxxxx.tar.xz -C abc/
cd abc
git checkout .
```

_注意：在后面部分**\<Chipcode directory\>**就代表`~/abc/`_

### copying the files to the appropriate directories

方法1 : 获取脚本执行 [make_skd1.sh](/src/make_sdk1.sh)

```
cd <Chipcode directory>
wget https://peiyake.com/src/make_sdk1.sh
chmod a+x make_sdk1.sh
sh make_sdk1.sh
```
---

### 安装`repo`

这里使用的清华大学的镜像站，repo的官方镜像站需要翻墙

```
apt-get install git #如果git没有安装需要先安装
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod +x repo
mv repo /usr/bin/
```

设置`git-repo`仓库的URL为清华大学镜像站

```
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
```

> 因为repo的运行过程中会尝试访问官方的git源更新自己，如果不能翻墙，那么接下来使用repo命令就会卡住

---

### generating the QSDK framework

```
cd <Chipcode directory>
rm -rf BOOT.AK.1.0 BOOT.BF.3.* IPQ8064.ILQ.11.1 IPQ4019.ILQ.11.1 IPQ8074.ILQ.11.1 RPM.AK.1.0 TZ.AK.1.0 TZ.BF.2.7 TZ.BF.4.0.8 WIGIG.TLN*
cp -rf */* .
```

以下两步耗时比较久

```
repo init -u git://codeaurora.org/quic/qsdk/releases/manifest/qstak -b release -m caf_AU_LINUX_QSDK_NHSS.QSDK.11.1_TARGET_ALL.12.0.4170.001.4548.xml
repo sync -j8 --no-tags -qc    #这里大概要耗1小时
```

文件拷贝

方法1 : 获取脚本执行 [make_skd2.sh](/src/make_sdk2.sh)

```
cd <Chipcode directory>
wget https://peiyake.com/src/make_sdk2.sh
chmod a+x make_sdk2.sh
sh make_sdk2.sh
```

### Enterprise customers:

```
cp apss_proc/out/proprietary/RBIN-NSS-ENTERPRISE/BIN-NSS.CP* qsdk/dl/
```

### Customers with Qualcomm HY-FI, WHC, WAPid, WiGig, or Sigma DUT packages

这一步，可忽略

### Create the 32-bit QSDK build

#### install depens packages 

```
apt-get install gcc g++ binutils patch bzip2 flex make gettext pkg-config unzip zlib1g-dev libc6-dev subversion libncurses5-dev gawk sharutils curl libxml-parser-perl python-yaml ocaml-nox ocaml ocaml-findlib libssl-dev libfdt-dev

apt-get install device-tree-compiler u-boot-tools
```

#### Install the different feeds in the build framework:

```
cd <Chipcode directory>
cd qsdk
./scripts/feeds update -a
./scripts/feeds install -a -f
```

### Copy the base configuration to use for the build

### Enterprise32-bit

```
cp qca/configs/qsdk/ipq_enterprise.config .config
sed -i "s/TARGET_ipq_ipq806x/TARGET_ipq_ipq60xx/g" .config
```

### Regenerate a complete configuration file and start the build:

```
make defconfig
sed -i -e "/CONFIG_PACKAGE_qca-wifi-fw-hw5-10.4-asic/d" .config
make V=s    
```

> 根据PC的配置，`make V=s` 编译过程持续1 ~ 4 小时，耐心等待 ~~~ 可以先干点别的事情了

### Generate a complete firmware image

#### install tools required

```
apt-get install uboot-mkimage
apt-get install device-tree-compiler
apt-get install libfdt-dev
```
#### Copy the flash config files to common/build/ipq

方法1 : 获取脚本执行 [make_skd3.sh](/src/make_sdk3.sh)

```
cd <Chipcode directory>
wget https://peiyake.com/src/make_sdk3.sh
chmod a+x make_sdk3.sh
sh make_sdk3.sh
```

> 注意： \<profile-name\> 是根据前面 [Enterprise32-bit](#Enterprise32-bit)的选择决定的。上面我们选择了 Enterprise 版本，那么 `profile-name 就应该等于E`
>
> E --> Enterprise
>
> P --> Premium 
>
> LM256 --> LM256 
>
> LM512 --> LM512

```
export BLD_ENV_BUILD_ID=<profile-name>
python update_common_info.py
```