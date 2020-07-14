---
title: WSL2 Ubuntu2020.04子系统Docker安装
date: 2020-07-14 18:24:22
categories: 
- windows
- ubuntu
tags:
- ubuntu
- docker
description: 使用Windows10 Ubuntu子系统后，筹划将原有的部分工具移动到子系统使用docker管理，顺便踩踩坑积累经验。
---
由于默认非root账号登录，大部分指令都需要加上 sudo，若权限不足加入sudo执行即可。

如果嫌麻烦可以启用并切换至root用户：

```
# 设置root用户密码
sudo passwd root
# 切换root用户
su root
```

## 更换镜像源

备份默认源：

```
cp /etc/apt/sources.list /etc/apt/sourses.list.bak
```

编辑文件：

```
vim /etc/apt/sources.list
```

删除原有内容并替换为：

```
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

更新源：

```
apt-get update
apt-get upgrade
```

## Doker安装

卸载旧版本（如果有）：

```
apt-get remove docker docker-engine docker.io containerd runc
```

设置存储库：

```
# 安装软件包以允许 apt 通过 HTTPS 使用存储库
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
# 添加 Docker 的官方 GPG 密钥
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# 设置稳定的存储库
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
# 更新 apt 包索引
apt-get update
```

若秘钥未配置更新索引时可能会出现异常：

![秘钥未配置](https://gitee.com/gonghs/image/raw/master/img/20200709154912.png)

安装最新版本的 Docker Engine-Community 和 containerd

```
apt-get install docker-ce docker-ce-cli containerd.io
```

查看是否安装成功：

```
docker --version
# 启动docker
service docker start
```

![版本号](https://gitee.com/gonghs/image/raw/master/img/20200709155621.png)

配置镜像源：

```
# 创建文件夹 不先创建保存文件可能提示权限不足(root用户忽略)
mkdir /etc/docker 
# 编辑文件
vim /etc/docker/daemon.json
# 文件内容
{
  "registry-mirrors": ["https://hub-mirror.c.163.com"]
}
```

重启docker：

```
service docker restart 
```

## 在Idea中访问

开启远程访问端口：

```
 # 编辑配置
 vim /etc/default/docker
 # 配置内容
 DOCKER_OPTS="-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock"
```

![image-20200713145647547](https://gitee.com/gonghs/image/raw/master/img/20200713145648.png)

重启服务：

```
service docker restart 
```

确认端口状态：

```
netstat -tunlp
```

![端口运行状态](https://gitee.com/gonghs/image/raw/master/img/20200713150016.png)

由于目前外部程序连接无法直接通过localhost访问（浏览器似乎可以[localhost:2375/info](http://localhost:2375/info)），需要使用子系统ip访问。

![localhost访问](https://gitee.com/gonghs/image/raw/master/img/20200714163959.png)

查询子系统ip：

```
ip addr
```

![查询ip](https://gitee.com/gonghs/image/raw/master/img/20200713171043.png)

Idea setting -> Build,Execution,Deployment -> Docker新增配置

![Idea docker配置](https://gitee.com/gonghs/image/raw/master/img/20200713171142.png)

配置完成可以在管理窗口看到容器与镜像信息。

![docker管理窗口](https://gitee.com/gonghs/image/raw/master/img/20200713171312.png)

## 处理IP变化问题

由于子系统IP总是变化，外部访问时如果用IP就会很麻烦，可以借助[wsl2host](https://github.com/shayne/go-wsl2-host/releases)配置host改善。

下载：

![下载](https://gitee.com/gonghs/image/raw/master/img/20200714164651.png)

下载完毕后使用Cmd或PowerShell运行：

```
.\wsl2host.exe install
```

输入系统用户名密码，无异常则安装成功。

![wsl2host安装](https://gitee.com/gonghs/image/raw/master/img/20200714165909.png)

安装成功到计算机->管理->服务和应用程序->服务 在服务列表中找到WSL2 Host并启动。

![WSL2 Host服务](https://gitee.com/gonghs/image/raw/master/img/20200714181005.png)

若启动失败并提示登录失败，右键点击服务->属性 在登录页签变更用户密码。

![登录失败](https://gitee.com/gonghs/image/raw/master/img/20200714181132.png)

![修改密码](https://gitee.com/gonghs/image/raw/master/img/20200714181315.png)

服务正常启动，将在host文件写入一行。

```
[WSL2 ip] ubuntu2004.wsl
```

之后我们都使用ubuntu2004.wsl访问服务即可。

![访问服务](https://gitee.com/gonghs/image/raw/master/img/20200714181609.png)

---

参考资料:

[解决Win10 WSL2 IP地址经常变动导致docker容器无法正常访问](https://blog.csdn.net/kfeng632/article/details/105979109/)