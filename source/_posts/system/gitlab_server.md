---
title: 搭建gitlab server
type: system
description: 搭建gitlab server 服务器
date: 2020-06-22 19:10:42
---

gitlab作为一个开源git仓库集成环境很受欢迎，自己工作中也对相关问题进行过处理，这里简单总结记录一下。

## 系统环境

* 操作系统： **CentOS Linux release 7.7.1908 (Core)**
* 内核版本： **3.10.0-1062.el7.x86_64**

## 安装

* 安装 `gitlab-ce`
1. `curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash`
2. `yum install gitlab-ce-12.5.7-ce.0.el7.x86_64`

* 初始化配置
1. `gitlab-ctl reconfigure` ，执行后会自动安装很多东西，成功后如下如所示：

![配置成功截图](/images/gitlab-config.png)

* 启动gitlab
1. `gitlab-ctl restart`  执行后，如下打印说明成功了

![启动成功截图](/images/gitlab-restart.png)

* 启动后就可以通过浏览器访问gitlab了
* 初次访问，需要设置root账户密码，这是管理员权限密码
* 如果不能访问页面，关闭SElinux 再试试 `setenforce 0`

## gitlab-runner 安装

1. `gitlab-ce` 和 `gitlab-runner` 版本应该匹配，否则会注册失败
2. [gitlab-runner安装官方参考](https://docs.gitlab.com/runner/install/)

## gitlab-runner 常见问题处理

1. `Build log exceeded limit of 4194304 bytes.`

* 这个原因是设置的编译日志大小满了，默认是4M。 可以通过在 `/etc/gitlab-runner/config.toml` 添加 `output_limit = 40960` 来修改日志大小
* 参考资料 [runner配置官方说明](https://gitlab.com/gitlab-org/gitlab-runner/blob/master/docs/configuration/advanced-configuration.md#the-runners-section)





