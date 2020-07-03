---
title: VPP编译错误'package python36-devel is not installedpackage'
type: vpp
description: 编译错误处理"Please install missing RPM,spackage python36-devel is not installedpackage python36-pip is not installed"
date: 2020-07-03 19:10:02
---

在CentOS-7.5.1804环境编译VPP（version=tag19.08）时，系统已经安装了`python3-devel`，但是编译却依赖`python36-devel`，尝试安装`python36-devel`rpm包，但是通过
`yum`安装时却一直提示 "Package python36-devel is obsoleted by python3-devel, trying to install python3-devel-3.6.8-13.el7.x86_64 instead"。也就是说
只能安装 `python3-devel`。经过尝试终于找到解决办法，这里记录一下。

## 通过指定仓库来安装

第一步，要先安装python的官方yum仓库：

```
rpm -Uvh http://mirrors.gbcom.com.cn/packages/ius-release-el7.rpm
```

第二步，卸载`python3-devel python3-pip`

```
yum remove -y python3-devel python3-pip
```

第三步，指定仓库安装`python36-devel python36-pip`

```
yum install python36-devel python36-pip  --disablerepo=* --enablerepo=ius
```

至此问题解决，编译VPP通过。