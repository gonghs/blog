---
title: Idea插件Custom Postfix Template，代码补全的一大利器
date: 2020-07-06 18:16:22
categories: 
- tool
tags:
- idea
description: 之前介绍过Idea后缀补全（Postfix Completion）和环绕模板（Live Templates），后缀补全相对而言调用比较顺手，只需要用.符号就可以触发，但由于官方目前并未告知如何定义多任务光标的补全模板，自带的有类似功能的模板也无法编辑模仿，因此功能上来说还是环绕模板更加全面，使用插件Custom Postfix Templates可以补全这个缺憾，功能上完全可以取代环绕模板，也增强了便利性。
---

Custom Postfix Templates的存在本身是由于旧版本Idea官方不支持自定义后缀补全，而在官方支持自定义模板的今天，我们仍可以为了更大更全面的模板定义而使用它。

## 安装

在Setting -> Plugins -> Marketplace中找到Custom Postfix Templates并安装重启。

![image-20200706150646081](https://gitee.com/gonghs/image/raw/master/img/20200706150646.png)

若安装顺利将会在Setting -> Custom Postfix Templates 看到一系列自带的模板配置，并在此可以选择自动联网更新。

![image-20200706150912746](https://gitee.com/gonghs/image/raw/master/img/20200706150912.png)

如果在配置中找不到任何模板文件，可能是因为raw.githubusercontent.com地址无法访问，需要自行在hosts文件增加DNS解析，插件自带的由各个作者提供的模板文件地址维护在官方插件github地址下的templates文件夹下，在这里可以看到，所有的模板文件都在这个域名下。

![image-20200706151339417](https://gitee.com/gonghs/image/raw/master/img/20200706151339.png)

## 使用

插件自身已经加载了上步骤中的文件配置，直接使用**.**符号即可触发选择，也可以使用Ctrl + Space或Ctrl + Alt + Space主动触发选择，使用Tab或Enter键确定选择。

![使用](https://gitee.com/gonghs/image/raw/master/img/20200706152538.gif)

在选择时按Alt + Enter可以进入当前模板的定义文件。

![进入定义文件](https://gitee.com/gonghs/image/raw/master/img/20200706154304.gif)

快捷键Alt + Shift + P可以编辑用户设置，或查看插件配置。

![编辑用户或查看插件配置](https://gitee.com/gonghs/image/raw/master/img/20200706154638.gif)

## 模板配置

### 配置综述

插件模板的自定义基本和Live Templates是一致的，并且支持所有Live Templates支持的变量表达式。

Live Templates定义：

![配置图解](https://gitee.com/gonghs/image/raw/master/img/20200605101246.png)

插件模板定义：

![插件模板定义](https://gitee.com/gonghs/image/raw/master/img/20200706165349.png)

除了原有的Live Templates和后缀补全配置（部分类型配置）之外，插件还支持了一些诸如：指定光标次序，条件启用等配置。

演示：

![演示](https://gitee.com/gonghs/image/raw/master/img/20200706162122.gif)

### 类型匹配

可以为不同的变量类型配备不同的表达式（按顺序匹配 有匹配到的则直接采用）：

```
.optmor : optional map orElse
    java.lang.String → Optional.ofNullable($expr$).map($map:typeOfVariable(expr):"item ->"$$method$).orElse("$else$");
    java.lang.Long → Optional.ofNullable($expr$).map($map:typeOfVariable(expr):"item ->"$$method$).orElse($else$);
```

演示：

![类型配置](https://gitee.com/gonghs/image/raw/master/img/20200706171117.gif)

以下为官方支持的Java所有匹配类型：

| 类型名            | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| ANY               | 任意表达式                                                   |
| VOID              | 任意空表达式（暂时没有测试出适用于什么场景，有知道的同学麻拜托评论指点一下） |
| NON_VOID          | 任意非空表达式                                               |
| ARRAY             | 任意Java数组                                                 |
| BOOLEAN           | boolean基本类型及其包装类型                                  |
| ITERABLE_OR_ARRAY | 任意迭代器类型或数组                                         |
| NOT_PRIMITIVE     | 任意非基本类型                                               |
| NUMBER            | 任意数字基本类型及其包装类型                                 |
| BYTE              | byte基本类型及其包装类型                                     |
| SHORT             | short基本类型及其包装类型                                    |
| CHAR              | char基本类型及其包装类型                                     |
| INT               | int基本类型及其包装类型                                      |
| LONG              | long基本类型及其包装类型                                     |
| FLOAT             | float基本类型及其包装类型                                    |
| DOUBLE            | double基本类型及其包装类型                                   |
| NUMBER_LITERAL    | 数字常量                                                     |
| BYTE_LITERAL      | byte常量                                                     |
| SHORT_LITERAL     | short常量                                                    |
| CHAR_LITERAL      | char常量                                                     |
| INT_LITERAL       | int常量                                                      |
| LONG_LITERAL      | long常量                                                     |
| FLOAT_LITERAL     | float常量                                                    |
| DOUBLE_LITERAL    | double常量                                                   |
| STRING_LITERAL    | 字符串常量                                                   |
| CLASS             | 任意类引用类型                                               |

### 条件启用

类型匹配后跟随的[any class]配置，意味着只有该类存在，模板配置才会生效。

此配置便于我们定义第三方类库的模板且不需要担忧未引入对应第三方依赖时有多余的模板触发，干扰选择。

例如此项配置在未引入对应依赖时将不被启用：

```
.isBlank : isBlank
    java.lang.String [org.apache.commons.lang3.StringUtils] -> StringUtils.isBlank($expr$)
```

当->右侧表达式为**仅有**[SKIP]（若非仅有将会被当做字符串处理）时，该类型将被忽略。

```
.isBlank : isBlank
    java.lang.String [org.apache.commons.lang3.StringUtils] -> [SKIP]
```

### 静态导入

在->右侧表达式末尾加入[USE_STATIC_IMPORTS]时，将使用静态导入的方式导入依赖，要使用静态导入时必须指定全限定类名：

```
.isBlank : isBlank
    java.lang.String [org.apache.commons.lang3.StringUtils] -> org.apache.commons.lang3.StringUtils.isBlank($expr$) [USE_STATIC_IMPORTS]
```

![静态导入](https://gitee.com/gonghs/image/raw/master/img/20200706180633.gif)