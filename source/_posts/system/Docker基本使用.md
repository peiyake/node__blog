---
title: Docker 基本使用
type: system
description: 最近玩了一下Docker，发现确实是个好东西。由于工作环境并不用docker环境，所以自己只是简单使用了一下，这里总结一下基本使用方法，供自己查阅。
date: 2020-07-23 09:59:22
---

知乎上有一片文章介绍[docker容器与虚拟机有什么区别？](https://www.zhihu.com/question/48174633)，**相比于虚拟机，Docker要简洁很多,因为我们不需要运行一个臃肿的客户机操作系统**。但是本人目前对docker的使用就是想运行一个**臃肿的操作系统**，因为我想在一台机器上随意切换不同的系统环境，目的是用来开发或者编译。

### 安装

宿主机
* CentOS Linux release 7.5.1804 (Core)
* Intel(R) Core(TM) i7-2600 CPU @ 3.40GHz
* MemTotal:        3823684 kB

安装docker仓库，三选一

```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo                      # 官方
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo              # 阿里云
sudo yum-config-manager --add-repo https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo   # 清华大学
```

安装docker

```
sudo yum install docker-ce docker-ce-cli containerd.io
```

安装指定版本docker

```
yum list docker-ce --showduplicates | sort -r
sudo yum install docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```

启动docker

```
sudo systemctl start docker
```

测试docker

```
sudo docker run hello-world
```

### Dockerfile

这里通过Dockerfile制作一个 ubuntu16.04的环境

```
cd ~
mkdir doc && cd doc
touch Dockerfile
```

Dockerfile
```
FROM ubuntu:16.04

RUN apt-get update \
        && apt-get install -y \
        inetutils-ping \
        net-tools \
        openssh-server  \
        gcc \
        g++ \
        binutils \
        patch \
        bzip2 \
        flex \
        make \
        gettext \ 
        pkg-config \
        unzip \
        zlib1g-dev \ 
        libc6-dev \
        subversion \
        libncurses5-dev \
        gawk sharutils \
        curl \
        libxml-parser-perl \
        python-yaml \
        ocaml-nox \
        ocaml \
        ocaml-findlib \ 
        libssl-dev \
        libfdt-dev \
        dpkg \
        xz-utils \
        git \
        vim \
        sed \
        python \
        perl \
        bzip2 \
        device-tree-compiler \
        u-boot-tools \
        device-tree-compiler \
        libfdt-dev \
        lib32ncurses5  \
        lib32z1 \
        tree 

RUN sed -i 's/UsePAM yes/#UsePAM yes/' /etc/ssh/sshd_config \
        && sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config \
        && sed -i 's/PermitEmptyPasswords no/PermitEmptyPasswords no/' /etc/ssh/sshd_config \
        && update-rc.d ssh enable \
        && touch /etc/start.sh \
        && echo "#!/bin/bash" > /etc/start.sh \
        && echo "passwd root << EOF" >> /etc/start.sh \
        && echo "123456" >> /etc/start.sh \
        && echo "123456" >> /etc/start.sh \
        && echo "EOF" >> /etc/start.sh \
        && echo "/etc/init.d/ssh start" >> /etc/start.sh \
        && echo "/bin/bash" >> /etc/start.sh \
        && chmod a+x /etc/start.sh
```

制作容器

```
cd ~/doc/
docker build -t ubuntu:16.04.ssh .
```

运行容器

```
sudo docker run -dit -v /home/peiyake:/root -p 6022:22 --restart=always  ubuntu:16.04.ssh /etc/start.sh
```

* **ubuntu:16.04.ssh /etc/start.sh**：启动容器 ubuntu:16.04.ssh，并在启动后执行脚本 `/etc/start.sh`
* **-p 6022:22**：宿主机端口6022映射到容器22端口，用于ssh连接
* **-v /home/peiyake:/root**：宿主机目录`/home/peiyake` ，挂载到容器 `/root` 目录下，这样ssh登录容器后，操作的目录其实就是宿主机的目录文件
* **-dit**：后台、交互式启动
* **--restart=always**：开机自启动、意外退出自启动

### 结束语

以上是docker的基本使用，以后如果工作需要再做更深入了解





