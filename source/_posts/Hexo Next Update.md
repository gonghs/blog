---
title: Hexo 和Next主题升级
date: 2020-07-21 00:57:16
categories: 
- blog
tags:
- hexo
- next
description: 之前搭建博客的时候为了上传到github，简单粗暴地将Next主题克隆并删除了.git文件夹，因此不方便对主题进行更新，如今Next主题已经更新迭代了很多版本，由于希望得到一些新功能的更新，决定将主题进行升级。
---
本文相关版本：

>Hexo原始版本： 3.9.0
>Hexo升级版本： 4.2.1
>Next原始版本： 5.1.4
>Next升级版本： 7.8.0

[官方给了我们一些更新建议](https://github.com/theme-next/hexo-theme-next/blob/master/docs/UPDATE-FROM-5.1.X.md)。

## 版本更新

首先升级过时的依赖，使用指令查询过时依赖：

```bash
npm outdated
```

![过时依赖](https://gitee.com/gonghs/image/raw/master/img/20200716164904.png)

使用指令更新，将更新Wanted栏与Current栏不一致的依赖。

```bash
npm update
```

也可以使用npm-check-updates更新所有依赖：

```bash
# 安装
npm install -g npm-check-updates
# 查看要更新依赖
ncu
# 更新依赖 这里只更新package.json文件至最新版本
ncu -u
# 安装依赖
npm i
```

这里我选择更新所有（>-<踩坑就踩到地狱）。

便于后续更新，将Next主题仓库代码fork一份到自己的仓库，并克隆至hexo主题文件夹下。

```bash
git clone https://github.com/gonghs/hexo-theme-next themes\next7.8.0
```

利用Hexo3的[Data Files特性](https://hexo.io/docs/data-files)，在source文件夹下\_data文件夹新建next.yml并拷贝旧版本next/\_config.yml配置所有内容，以保留自定义主题配置且防止更新时与官方冲突，但需要注意主题配置与\_data下next.yml中的override配置都必须为false（默认false，因此删除此配置项也都可以），建议**将新版本的配置修改至此处，而旧版本配置放在原文件夹下保留一份可用的**。

![override配置](https://gitee.com/gonghs/image/raw/master/img/20200716164030.png)

原有next文件夹可以不删除，以便出了问题可以切换回旧版本。

## 旧版本问题处理

### 首页菜单栏404

更新所有依赖后菜单栏点击可能会报错404：

![404](https://gitee.com/gonghs/image/raw/master/img/20200716173301.png)



需要到主题文件夹下\_config.yml（\_data/next.yml自定义配置中也可以），将menu配置项||之前的空格去掉

```yml
menu:
  home: / || home
# 变更为
menu:
  home: /|| home
```

除此之外目前看旧版本的表现是正常的，可以用于出错时的备份版使用。

## 新版本问题处理

新版本主题配置如未特殊说明都修改在自定义配置中。

要使用新版本，需要将hexo主题配置修改为hexo新版本所在的文件夹名，这里是：

```yml
theme: next7.8.0
```

![新版本](https://gitee.com/gonghs/image/raw/master/img/20200716173953.png)

启动新版本，看起来有比较多的问题，一点一点处理。

### 语言问题

由于官方将中文的语言命名由zh-Hans改为了zh-CN，因此要将hexo配置文件中的语言进行对应修改。

```yaml
language: zh-CN
```

###  图标显示异常

为了方便自定义图标库，对比新旧配置可以发现原先图标的配置有了变更，因此需要搜索类似的地方进行修改：

```yaml
# 新配置 完整样式名
fa fa-archive
# 旧配置 Awesome图标名
archive
```

涉及修改的参数名前缀大致包含：

```yml
# 页脚配置
footer:
  icon:
# 主菜单配置
menu:
# 访问统计相关
busuanzi_count:
```

图标修改完毕后需要如果还是显示异常可能需要进行一次清空操作再运行：

```bash
hexo clean
hexo s
```

### 首页自定义logo异常

原有的自定义logo配置为对象配置，新版本改为了只配置图片路径，若新旧都为默认配置在新版本渲染中将会异常展示。

```
# 旧版本
custom_logo:
  enabled: false
  image:
# 新版本
custom_logo:
```

### 文章阅读次数和评论展示异常

![阅读次数异常](https://gitee.com/gonghs/image/raw/master/img/20200720160314.png)

valine下的配置有部分配置追加，按需拷贝至新配置文件即可。

```yaml
# 新年增部分
valine:
  language: zh-cn # Language, available values: en, zh-cn
  # 是否文章头部浏览量（不影响首页）
  visitor: false # Article reading statistic
  comment_count: true # If false, comment count will only be displayed in post page, not in home page
  recordIP: false # Whether to record the commenter IP
  serverURLs: # When the custom domain name is enabled, fill it in here (it will be detected 	automatically by default, no need to fill in)
  #post_meta_order: 0
```

### 页面特效

由于6.0.0版本移除了原有的部分自带依赖（可比对next/soure/lib下文件），旧版本的部分配置例如canvas_nest将无法直接生效。

```yml
# Canvas-nest
canvas_nest: true
```

![依赖移除](https://gitee.com/gonghs/image/raw/master/img/20200720221549.png)

如果需要安装可以在[theme-next](https://github.com/theme-next)仓库中搜索并按文档安装。

![搜索canvas-nest](https://gitee.com/gonghs/image/raw/master/img/20200720223635.png)

这里以canvas_nest为例：

在hexo/source/_data下创建footer.swig文件（如果有则追加内容）:

```bash
<script color="0,0,255" opacity="0.5" zIndex="-1" count="99" src="https://cdn.jsdelivr.net/npm/canvas-nest.js@1/dist/canvas-nest.js"></script>
```

修改主题配置（已有则忽略）：

```
# Canvas-nest
canvas_nest: true
```

### 微信二维码

原有的wechat_subscriber配置变更并删除了部分内容，修改了一些样式：

```yaml
# 旧配置
wechat_subscriber:
  enabled: true
  qcode: /uploads/WechatIMG.jpg
  description: 欢迎关注猿枫小店公众号，逗比程序员的代码宅基地。
# 新配置
follow_me:
  WeChat: /images/WechatIMG.jpg || fab fa-weixin
```

旧样式：

![旧样式](https://gitee.com/gonghs/image/raw/master/img/20200720232208.png)

新样式：

![新样式](https://gitee.com/gonghs/image/raw/master/img/20200720232518.gif)

###  版权文字

原有的post_copyright配置进行了变更，并新增了侧边栏版权说明：

```yaml
# 旧配置
post_copyright:
  enable: true
  license: CC BY-NC-SA 3.0
  license_url: https://creativecommons.org/licenses/by-nc-sa/3.0/
# 新配置
creative_commons:
  license: by-nc-sa
  # 侧边栏说明
  sidebar: true
  # 文章底部说明
  post: true
  language: zh-cn
```

![侧边栏版权说明](https://gitee.com/gonghs/image/raw/master/img/20200720233329.png)

### 统计异常

在旧版本中的post_wordcount配置似乎已经不可用，因此需要使用symbols_count_time替代。

```bash
# 卸载原有依赖 不卸载对新插件有影响
npm uninstall hexo-wordcount
# 安装新依赖
npm i hexo-symbols-count-time
```

新的配置在symbols_count_time下：

```yaml
symbols_count_time:
  # 换行显示字数统计和阅读市场
  separated_meta: true
  # 文章底部显示
  item_text_post: true
  # 博客底部显示 默认为false
  item_text_total: true
```

在hexo配置（非主题配置）下可以设置显示哪些内容和一些统计维度：

```yaml
symbols_count_time:
  # 文章字数
  symbols: true
  # 阅读时长
  time: true
  # 总文章字数
  total_symbols: true
  # 阅读总时长
  total_time: true
  # 是否排除代码统计
  exclude_codeblock: false
  # 平均字长 即将多少个字符统计为1个字数
  awl: 4
  # 每分钟的字数 阅读速度
  wpm: 275
  # 统计单位 这里是分钟
  suffix: "mins."
```

### 字体大小

更新至新版以后字体默认大小比原先大了些，需要调整可以在font配置下进行变更。

```yaml
font:
  enable: true
  # 字体地址
  host:
  # 正文字体将根据这个配置生效
  global:
  	# 拓展字体库 设置为true将从host加载字体
    external: true
    # 字体族
    family: Lato
    # 大小  size=1为默认大小 0.8将缩小为80%
    size: 0.8
  # 正文标题字体大小  
  headings:
  	# 拓展字体库 设置为true将从host加载字体
    external: true
    # 字体族
    family: Lato
    # 大小
    size: 0.8
  # 注意这里没有size配置
  posts:
  ...
```



---

以上将此次升级中原自带配置的部分进行了修正，大致拥有了旧版的功能，之后将单独开文记录其他特别的功能在新版本的修改。

参考文档：

[Hexo-NexT配置超炫网页效果](https://www.jianshu.com/p/9f0e90cc32c2)