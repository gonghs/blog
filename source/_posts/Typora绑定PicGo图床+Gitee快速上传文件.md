---
title: Typora绑定PicGo图床+Gitee快速上传文件
date: 2020-05-05 22:24:32
categories: 
- tool
tags:
- typora
- picGo
- markdown
description: Typora是个人比较喜欢的Markdown编辑工具，它可以用来绑定PicGo图床实现粘贴文件自动上传到图床的功能，便于我们编写完文章后发表到其他网址
---
# 下载Typora

直接在[typora官网](https://www.typora.io/)选择Download页签下载即可

# PicGo准备

## 下载

在[PicGo官方的github地址](https://github.com/Molunerfinn/PicGo/releases)下载即可，建议选择2.2.0以上版本，支持PicGo-Server配置自定义端口

## Gitee插件

在插件设置中查找gitee(搜索栏区分大小写)，并安装gitee 搜索到的两个插件

![image-20200505214615638](https://gitee.com/gonghs/image/raw/master/img/20200505214620.png)

安装完毕重新启动，左侧图床设置中将会加入Gitee图床

# 码云仓库准备

登录码云，新建仓库，如果要于外网访问则选择公开仓库(如果选择私有仓库则在typora软件内部也无法预览)

![image-20200505215209562](https://gitee.com/gonghs/image/raw/master/img/20200505215211.png)

点击右上角头像选择设置

![image-20200505215327238](https://gitee.com/gonghs/image/raw/master/img/20200505215328.png)

在安全设置中找到私人令牌，选择生成新令牌，权限勾选projects即可

![image-20200505215509173](https://gitee.com/gonghs/image/raw/master/img/20200505215511.png)

# PicGo Gitee配置

在PicGo Gitee图床设置中填写相关内容

![image-20200505215918872](https://gitee.com/gonghs/image/raw/master/img/20200505215919.png)

# Typora图床配置

在Typora中点击文件->偏好设置 选择图片页签(优先使用相对路径的配置比较奇怪，如果粘贴进去的图片是以相对路径展示的并且查找不到，则勾选/取消勾选一下该配置试一试)

![image-20200505220738573](https://gitee.com/gonghs/image/raw/master/img/20200505223808.png)

点击验证图片上传选项如果成功则代表配置完成

![image-20200505221054787](https://gitee.com/gonghs/image/raw/master/img/20200505221057.png)

# 配置结果

![1](https://gitee.com/gonghs/image/raw/master/img/20200505221358.gif)

# 异常处理

## 测试上传时报错

![image-20200505221626174](https://gitee.com/gonghs/image/raw/master/img/20200505221628.png)

确认PicGo-Server开启且监听端口为36677

PicGo设置中选择PicGo-Server(2.2.0版本以上才有)

![image-20200505221851568](https://gitee.com/gonghs/image/raw/master/img/20200505221852.png)

## 成功连接PicGo但返回失败

![image-20200505222002093](https://gitee.com/gonghs/image/raw/master/img/20200505222002.png)

这是由于文件名重复，在PicGo设置中开启时间戳重命名功能即可

![image-20200505222108803](https://gitee.com/gonghs/image/raw/master/img/20200505222109.png)
