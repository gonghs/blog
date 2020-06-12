---
title: 如何给网站增加CDN缓存
date: 2020-06-12 18:32:22
categories: 
- website
tags:
- cdn
description: 近期由于博客访问速度过慢，[测速](http://tool.chinaz.com/speedtest)时发现国内各地的下载速度都极慢，因此决定为域名增加CDN缓存。
---
增加CDN前测速：

![全国各地访问延时](https://gitee.com/gonghs/image/raw/master/img/20200612170256.png)

这里选择用[七牛云](https://portal.qiniu.com/cdn)为我们提供CDN服务。使用之前注册并进行实名认证。

## CDN域名创建

进入CDN模块控制台选择*域名管理 -> 添加域名*：

![添加域名](https://gitee.com/gonghs/image/raw/master/img/20200612171734.png)

加速域名输入需要加速的域名（已备案的域名），使用场景根据自身网站进行选择（一般默认即可）。

![域名配置](https://gitee.com/gonghs/image/raw/master/img/20200612172148.png)

源站配置选择源站域名，配置任意与加速域名不同的子域名即可。

回源host根据自身服务器IP是否会变动指定，一般默认即可。

![回源host](https://gitee.com/gonghs/image/raw/master/img/20200612172758.png)

源站测试中填写一个可访问的页面并点击下方测试按钮，测试通过后才可以创建。

![源站配置](https://gitee.com/gonghs/image/raw/master/img/20200612173914.png)

配置完毕后点击下方创建。七牛云提示需要配置CNAME。

![创建完成](https://gitee.com/gonghs/image/raw/master/img/20200612174028.png)

## 配置CNAME

这里以腾讯云为例，进入腾讯云控制台再在域名管理控制台，*我的域名*窗口选择进入*域名解析列表*。

![我的域名](https://gitee.com/gonghs/image/raw/master/img/20200612174553.png)

点击列表域名进入添加记录列表。

![域名解析列表](https://gitee.com/gonghs/image/raw/master/img/20200612174826.png)

点击添加记录指定记录类型为CNAME，主机记录根据提示填写加速域名对应的主机记录，记录值填入七牛云创建的CNAME。

**如果提示冲突，检查是否同一主机记录有一条A记录数据。**

![配置解析](https://gitee.com/gonghs/image/raw/master/img/20200612175513.png)

## 判断是否配置完成

linux/mac系统通过指令：

```
dig [加速域名]
```

windows通过指令：

```
nslookup [加速域名]
```

如果出现了刚刚配置的CNAME则代表成功。

![nslookup](https://gitee.com/gonghs/image/raw/master/img/20200612180232.png)

## 加速结果

启用CDN后大部分地区的访问速度都有所提升（注意加速只针对配置的加速域名，例如加速了www.ice-maple.com，ice-maple.com由于未加速还是原来的速度）。

![加速后访问速度](https://gitee.com/gonghs/image/raw/master/img/20200612182727.png)