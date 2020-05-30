---
title: 敏捷开发，代码重构。Idea不得不提的三大技巧
date: 2020-05-30 18:51:32
categories: 
- tool
tags:
- idea
description: 优秀的IDE让你的开发如丝绸般柔顺。
---
## 批量操作

多光标操作，是一款优秀的编辑器的基本职能。

在idea中利用*Alt*键可以轻松的一次性拉取多行进行多点编辑操作：

![Alt多点编辑](https://gitee.com/gonghs/image/raw/master/img/20200530000428.gif)

如果需要按点选择，则可以按*Alt*+*Shift*选择编辑：

![多点光标操作](https://gitee.com/gonghs/image/raw/master/img/20200530001522.gif)

*Ctrl*+*Alt*+*Shift*则包揽了行选择和点选择的功能：

![点选择和行选择](https://gitee.com/gonghs/image/raw/master/img/20200530011728.gif)

组合其他快捷键使用可以完成相对复杂的操作，例如将原有类变量的行注释，变更为段注释。

这里用到了*Ctrl* + *Alt*  +*Shift* + *J* 和 *Ctrl*+*W*。

前者可以选中当前文件中所有相同的内容，如果光标没有选择范围则默认为附近的一个词。

而后者则可以选中当前光标附近的词。

![行注释变更](https://gitee.com/gonghs/image/raw/master/img/20200530003612.gif)

利用*Ctrl*+*W*可以轻易找到多个编辑点内容开头结尾

![多光标不同长度末尾编辑](https://gitee.com/gonghs/image/raw/master/img/20200530005717.gif)

## 正则替换

idea文本替换功能支持所有的标准正则表达式，利用正则表达式可以完成很多复杂的功能。

使用正向和反向预查正则可以仅替换满足指定条件的文本，这个表达式可以替换被

```
<li class="nav-li"></li>
```

包裹的中文：

```reg
(?<=<li class="nav-li">)[\u4e00-\u9fa5]*(?=</li>)
```

![预查替换](https://gitee.com/gonghs/image/raw/master/img/20200530021213.gif)

idea支持使用小括号对正则进行分组，然后进行取值替换，上面将行注释变更的功能使用分组替换也可以完成：

```reg
\/\/ ([\u4e00-\u9fa5]*)
```

![行注释正则替换](https://gitee.com/gonghs/image/raw/master/img/20200530022122.gif)

利用复杂的正则表达式可以简便的完成数据库方言的替换。

例如时间格式化字符串在Oracle中是*yyyymmddHH24miss*而在mysql中却是*%Y%m%d%H%i%s*。

替换正则：

```
(yyyy)(-|/)*?(mm)(-|/)*?(dd)(\s)*?(HH24)(:*?)(mi)(:*?)(ss)
```

替换为:

```
%Y$2%m$4%d$6%H$8%i$10%s
```

![数据库方言替换](https://gitee.com/gonghs/image/raw/master/img/20200530024154.gif)

用好正则替换功能将会给代码重构带来极大的便利。

除了标准的正则表达式外，idea还支持在替换时修改替换文本的大小写属性（该功能不支持全局替换）。

| 表达式 |                  含义                  |
| :----: | :------------------------------------: |
|   \l   |       将替换文本的首字母置为小写       |
|   \u   |       将替换文本的首字母置为大写       |
|   \L   | 将替换文本置为小写，可以使用\E标记停止 |
|   \U   | 将替换文本置为大写，可以使用\E标记停止 |

![大小写转置](https://gitee.com/gonghs/image/raw/master/img/20200530025950.gif)

## 后缀补全

对对象使用调用符*.*时idea除了会弹出调用方法的提示，还会弹出一些预定义的后缀模板，利用这些后缀模板可以大幅减少移动光标的次数：

![后缀补全](https://gitee.com/gonghs/image/raw/master/img/20200530160430.gif)

官网预定义的模板可以满足大部分场景，但有些调用Api并不满足需要，在*2018.01*版本后的idea允许自定义后缀模板（旧版本可以使用插件*Custom Postfix Templates*）。

按*Ctrl*+*Shift*+*A*搜索postfix completion或者Setting=>General=>PostFix Completion打开postfix completion编辑页面。

这里允许自定义后缀模板：

![image-20200530175945024](https://gitee.com/gonghs/image/raw/master/img/20200530175952.png)

定义的模板中可以配置关键词，语言版本，适用的变量类型等，这里为任意对象定义一个Optional.orElse的模板：

![image-20200530180831492](https://gitee.com/gonghs/image/raw/master/img/20200530180831.png)

*$EXPR$*模板生效后变量将处在的位置，*$END$*则标记了光标最后处在的位置：

![自定义表达式](https://gitee.com/gonghs/image/raw/master/img/20200530181308.gif)

唯一遗憾的一点是官方文档中并未说明多任务光标要如何定义，期待官方后续版本能够开发这个设置。

![多任务光标](https://gitee.com/gonghs/image/raw/master/img/20200530181955.gif)





