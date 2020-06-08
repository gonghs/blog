---
title: Idea减少鼠标操作的几个技巧
date: 2020-06-09 01:00:00
categories: 
- tool
tags:
- idea
description: 远离鼠标不是我们的目的，我们只是希望能够物尽其用，快乐coding。
---
## 演出模式

*View* => *Appearance* => *Enter Presentation Mode* （旧版本为 *View* => *Enter Presentation Mode*）开启演出模式，演出模式让我们能够专注于代码的编辑窗口，隐藏不必要的界面细节。

![演出模式](https://gitee.com/gonghs/image/raw/master/img/20200605101514.png)

如果需要的话，可以在*Setting* => *Keymap* 为演出模式设置快捷键。

![设置快捷键](https://gitee.com/gonghs/image/raw/master/img/20200605101601.png)

如果需要呼出其他窗口可以通过*Alt* + 数字键。可以自行修改使用比较多的页面。

![呼出窗口](https://i.loli.net/2020/06/08/XFdmzsfRGneVwSi.gif)

对Idea足够熟悉的话，在演出模式下完全满足日常开发需求，并且拥有更好的视觉体验。

## 新建类

由于新建类需要使用*File* => *new*或者使用快捷键*Alt* + *insert*，要么离双手覆盖范围太远，或者操作起来较为复杂。

使用*Ctrl*+*Shift*+*A*可以较近距离的完成这个操作。

![新建类](https://i.loli.net/2020/06/05/b6yKnJRzBZljkFV.gif)

如果需要创建指定目录下的文件则需要多一步搜索操作。

![指定目录创建](https://i.loli.net/2020/06/05/lP3Kpifqyuwr4HI.gif)

## 搜索

| 快捷键                 | 用途                                                         |
| ---------------------- | ------------------------------------------------------------ |
| Ctrl + N               | 搜索类                                                       |
| Ctrl + Shift + N       | 搜索文件(加上/可以搜索目录，末尾加斜杆只搜索**目录及目录下文件**或**包**) |
| Ctrl + Shift + Alt + N | 搜索方法,变量,表名,css样式名等                               |
| Ctrl + E               | 查看最近打开文件(允许勾选仅显示变更文件)                     |
| Ctrl + F12             | 查找类的继承关系(可以看到类的所有可调用方法并查找来源,仅查询方法可以使用Alt+7 Structure窗口代替) |
| Ctrl + F/R             | 当前类查找/替换                                              |
| Ctrl + Shift + F/R     | 全局查找/替换                                                |
| Ctrl + Alt + B         | 查找类实现,变量声明等                                        |

以上是较为常用的搜索快捷键，因此这里不做更多的说明。

*Ctrl + Shift + A* 查找操作，这个快捷键可以搜索到几乎所有的Idea操作。如果有什么操作，*配置*（下文中介绍的配置都可以使用此快捷键搜索）不记得，都可以尝试在这里搜索。

![功能查找](https://i.loli.net/2020/06/05/9Qrs2HqflDEJpwR.gif)

*Ctrl + Shift + Alt + J* 和*Alt + J*功能类似，用于搜索并选中相同的词。

![选中词](C:/Users/maple/Pictures/博客/1.gif)

## 编辑

在编辑代码时可能会频繁的用到光标移动，下面一些配置或快捷键将会有效的帮助我们增强效率。

### 代码补全

*Ctrl + Shift + enter*代码补全，这几乎是Idea中最好用的快捷键了，它可以补全大部分的括号，大括号，引号，冒号，分号等，帮助我们做一个漂亮的收尾。

![1](C:/Users/maple/Pictures/博客/1.gif)

### 代码模板

#### *Ctrl+J*

活动模板不依托于某个代码块，除了使用快捷键触发外直接输入亦可以触发。

在*Setting -> Editor -> Live Templates*中可以自定义活动模板。

![自定义模板](https://gitee.com/gonghs/image/raw/master/img/20200605101236.png)

活动模板的自定义支持多任务光标，一个变量即是一个任务光标，可用$END$标记结束光标。

![配置图解](https://gitee.com/gonghs/image/raw/master/img/20200605101246.png)

在底部可以修改适用范围，勾选Java -> statement就可以适用于*Ctrl+J*或空白输入。

![修改适用范围](https://gitee.com/gonghs/image/raw/master/img/20200606213520.png)

执行后效果如下：

![活动模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606213032.gif)

#### *Ctrl+Alt+J*

环绕模板，它与后缀模板更为相似，且定义更加灵活。

定义时使用$SELECTION$标记被环绕的代码块。

![环绕模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606214320.png)

使用时若光标无选中代码块，则以当前行为环绕对象，否则以选中代码块为环绕对象。

演示：

![环绕模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606214736.gif)

关于活动模板和环绕模板的详细变量表达式定义参照[Idea Live Templates配置详解及演示]()

### 跳出引号，括号

#### tab跳出

确保*Settings->Editor->General->Smart Keys*中*Jump outside closing bracket/quote with Tab when typing*为勾选状态。

此时在字符串，括号的编辑区末尾时按*Tab*键将跳出（注意只有首次在编辑区时有效，之后再次进入就无效了）。

![跳出引号括号](https://gitee.com/gonghs/image/raw/master/img/20200608230600.gif)

#### 大括号导航

*Ctrl+Shift+M*可以导航到当前代码块括号上下，*Ctrl+[或]*也有类似的效果，不同的是多次使用可以向更外层跳跃。如果使用*Ctrl+Shift+[]*则会选中经过的代码块。

![大括号导航](https://gitee.com/gonghs/image/raw/master/img/20200608233104.gif)

配合代码补全的自动换行，将可以很轻易方便的在代码块之间穿梭。

### 自动导入和清理包

确保*Settings->Editor->General->Auto Import*中*Add unambiguous imports on the fly*和*Optimize imports on the fly (for current project)*处于选中状态。

编辑器将在需要时自动导入和清理依赖包。

勾选前：

![勾选前](C:/Users/maple/Pictures/博客/1.gif)

勾选后：

![勾选后](https://gitee.com/gonghs/image/raw/master/img/20200609002051.gif)