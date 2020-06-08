---
title: Idea Live Templates配置详解及演示
date: 2020-06-07 00:00:00
categories: 
- tool
tags:
- idea
description: Live Templates是一个便捷的定义代码模板的方式，其中提供了种类繁多的预定义模板，便于方便开发者使用。在官方预定义模板之外Idea亦拥有足够全能的变量定义表达式让开发者定义心仪的代码模板。
---
## 模板定义

### 活动模板

活动模板不依托于某个代码块，直接输入即可以触发，或者使用快捷键*Ctrl+J*主动触发。

在*Setting -> Editor -> Live Templates*中可以自定义活动模板。

![自定义模板](https://gitee.com/gonghs/image/raw/master/img/20200605101236.png)

活动模板的自定义支持多任务光标，一个变量即是一个任务光标，可用$END$标记结束光标。

![配置图解](https://gitee.com/gonghs/image/raw/master/img/20200605101246.png)

在底部可以修改适用范围，勾选Java -> statement就可以适用于*Ctrl+J*或空白输入。

![修改适用范围](https://gitee.com/gonghs/image/raw/master/img/20200606213520.png)

执行后效果如下：

![活动模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606213032.gif)

### 环绕模板

环绕模板依托于某个代码块，使用快捷键*Ctrl+Alt+J*触发。

定义时使用$SELECTION$标记被环绕的代码块。

![环绕模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606214320.png)

使用时若光标无选中代码块，则以当前行为环绕对象，否则以选中代码块为环绕对象。

演示：

![环绕模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606214736.gif)

## 模板变量表达式含义表

以下是定义变量时支持的表达式基本含义（当匹配多个结果时结果将出现在待选列表中）：

| 表达式                                                |                             含义                             |
| :---------------------------------------------------- | :----------------------------------------------------------: |
| annotated(\<annotation\>)                               |              返回具有指定注解的类，方法或字段名              |
| arrayVariable()                                       |            返回当前范围内数组变量，最近的优先展示            |
| lineCommentStart()                                    |               返回当前语言指示行注释开始的字符               |
| blockCommentStart()                                   |               返回当前语言指示块注释开始的字符               |
| blockCommentEnd()                                     |               返回当前语言指示块注释结束的字符               |
| commentStart()                                        |  返回当前语言指示注释开始的字符，对有行注释的返回行注释开头  |
| commentEnd()                                          | 返回当前语言指示注释结束的字符，对有行注释的返回空（行注释通常没有结束字符） |
| camelCase(\<String\>)                                   |                    将字符串转换为驼峰形式                    |
| snakeCase(\<String\>)                                   |                 将字符串转换为下划线分割形式                 |
| spaceSeparated(\<String\>)                              |                  将字符串转换为空格分开形式                  |
| spacesToUnderscores(\<String\>)                         |                  将字符串的空格替换为下划线                  |
| capitalize(\<String\>)                                  |                    将字符串首字母设为大写                    |
| capitalizeAndUnderscore(\<String\>)                     |               将字符串转换为大写并用下划线隔开               |
| decapitalize(\<String\>)                                |                    将字符串首字母设为小写                    |
| underscoresToCamelCase(\<String\>)                      |               将下划线形式字符串转换为驼峰形式               |
| underscoresToSpaces(\<String\>)                         |             将下划线形式字符串转换为空格隔开形式             |
| lowercaseAndDash(\<String\>)                            |               将字符串转为小写并使用中划线分割               |
| escapeString(\<String\>)                                |     将字符串中的特殊符号进行转义，便于在java字符串中使用     |
| substringBefore(\<String\>, \<Delimeter\>)                |              截取字符串在\<Delimeter\>之前的部分               |
| firstWord(\<String\>)                                   |                    返回字符串中的首个单词                    |
| castToLeftSideType()                                  |              获取左侧变量的类型判断是否需要强转              |
| rightSideType()                                       |                   获取右侧表达式的变量类型                   |
| className()                                           |          返回当前所在类（在内部类则返回内部类）类名          |
| currentPackage()                                      |                       返回当前所在包名                       |
| qualifiedClassName()                                  | 返回当前所在类（在内部类则返回内部类）的全限定类名（包+类名） |
| classNameComplete()                                   |                    触发类名相关的代码补全                    |
| clipboard()                                           |                     返回系统剪贴板的内容                     |
| complete()                                            |         调用一次代码补全，相当于调用一次*Ctrl+Space*         |
| completeSmart()                                       |     调用一次智能代码补全，相当于调用一次*Ctrl+Alt+Space*     |
| componentTypeOf(\<array\>)                              |                         返回数组类型                         |
| concat(\<String\>, ...)                                 |                          拼接字符串                          |
| date([format])                                        | 指定格式化方式返回当前系统时间字符串（根据*SimpleDateFormat*格式） |
| time([format])                                        | 指定格式化方式返回当前系统时间字符串（无日期，根据*SimpleDateFormat*格式） |
| descendantClassesEnum(\<String\>)                       |                       返回指定类的子类                       |
| lineNumber()                                          |                        返回当前行行号                        |
| enum(\<String\>, ...)                                   |                     返回建议的字符串列表                     |
| expectedType()                                        | 自动识别并返回期望的类型，一般用于赋值，方法参数，返回语句处。 |
| fileName()                                            |                  返回当前文件名（带拓展名）                  |
| fileNameWithoutExtension()                            |                 返回当前文件名（不带拓展名）                 |
| filePath()                                            |                 返回当前文件路径（带拓展名）                 |
| fileRelativePath()                                    |          返回当前文件相对当前项目的路径（带拓展名）          |
| groovyScript(\<String\>, [arg, ...])                    |             执行作为字符串形式传递的*groovy*脚本             |
| guessElementType(\<Collection\>)                        |                     返回集合中元素的类型                     |
| iterableComponentType(\<Iterable\>)                     |                     返回可迭代对象的类型                     |
| iterableVariable()                                    |         返回当前范围内可迭代类型对象，最近的优先展示         |
| methodName()                                          |                      返回当前所在方法名                      |
| methodParameters()                                    |                 返回当前所在方法的所有参数名                 |
| methodReturnType()                                    |                  返回当前所在方法的返回类型                  |
| regularExpression(\<String\>, \<Pattern\>, \<Replacement\>) |   查找字符串中满足\<Pattern\>的所有部分并替换为\<Replacement\>   |
| typeOfVariable(\<String\>)                              |                        返回变量的类型                        |
| variableOfType(\<String\>)                              |       返回当前范围内满足类型条件的变量，最近的优先展示       |
| suggestFirstVariableName(\<String\>)                    | 返回当前范围内满足类型条件的部分变量，最近的优先展示和*variableOfType*类似但不推荐true，false，this，和super |
| subtypes(\<String\>)                                    |                     返回指定类型的子类型                     |
| suggestIndexName()                                    | 返回当前范围中未使用的第一个常用迭代下标变量名（i，j，k等）  |
| suggestVariableName()                                 |        根据变量命名规则的代码风格设置返回建议的变量名        |
| suggestShortVariableName()                            |                      建议的变量名精简版                      |
| user()                                                |                    返回当前系统的用户名称                    |



## 变量表达式定义和演示

### annotated(\<annotation\>)

定义时在括号内传入注解的全限定类名：

![annotated模板配置](https://gitee.com/gonghs/image/raw/master/img/20200605101311.png)

演示：

![annotated模板演示](https://gitee.com/gonghs/image/raw/master/img/20200605101320.gif)

### arrayVariable

返回类字段，或方法变量中的数组类型变量名称。**离得近的将被优先推荐**。

演示：

![arrayVariable模板配置](https://gitee.com/gonghs/image/raw/master/img/20200605101331.gif)

### lineCommentStart~commentEnd

*lineCommentStart*，*blockCommentStart*，*blockCommentEnd*，*commentStart*和*commentEnd*在不同的语言环境中表现是不一致的。

*lineCommentStart*返回当前语言中指示行注释开始的字符。

*blockCommentStart*和*blockCommentEnd*则返回当前语言中指示块注释开始，结束的字符。

*commentStart*和*commentEnd*视情况而定，若当前语言有行注释则与lineComment表现一致（行注释通常没结束标记*commentEnd*为空），若没有行注释则与*blockComment*表现一致。

![comment模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606010034.png)

演示：

![comment模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606010605.gif)



### camelCase(\<String\>)~firstWord(\<String\>)

#### camelCase(\<String\>)

将**参数内容**转换为驼峰形式。可以转换空格，下划线，中划线分割的字符串（*之后的一些表达式也都是类似机制，因此不再单独录制演示*）。

![camelCase模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606184836.png)

演示：

![camelCase模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606185330.gif)

#### snakeCase(\<String\>)

将参数内容字符串转换为下划线分割形式，例如将user name，userName，user-name转换为user_name

#### spaceSeparated(\<String\>)

将字符串转换为空格分开形式（不会改变原来的大小写状态），例如将*userName*，*user Name*和*user-Name*转换为*user Name*

#### spacesToUnderscores(\<String\>)

将字符串的空格替换为下划线，例如将user name转换为user_name，将user  name（两个空格）替换为user__name

#### capitalize(\<String\>)

将字符串首字母设为大写，例如将username转换为Username

#### capitalizeAndUnderscore(\<String\>)

将字符串转换为大写并用下划线隔开 ，例如将*UserName*，*user name*和*user-name*转换为*USER_NAME*

#### decapitalize(\<String\>)

将字符串首字母设为小写，例如将Username转换为username

#### underscoresToCamelCase(\<String\>)

将字符串下划线形式转换为驼峰形式，例如将user_name转换为userName，将user_NAME转换为userName，将USERNAME转换为username。

#### underscoresToSpaces(\<String\>)

将字符串下划线替换为空格，例如将user_name转换为user name。

#### lowercaseAndDash(\<String\>)  

将字符串转换为小写并用中划线隔开 ，例如将*UserName*，*user name*和*user_name*转换为*user-name*。

#### escapeString(\<String\>)

对字符串中的特殊字符进行转义，以便在java字符串中进行使用。例如将"转换为\\"。

#### substringBefore(\<String\>, \<Delimeter\>)  

截取字符串在\<Delimeter\>之前的部分 ，例如substringBefore("fileName.zip",".")返回fileName。

#### firstWord(\<String\>)

返回字符串中的第一个单词。例如user name返回user

### castToLeftSideType与rightSideType

由于*castToLeftSideType*需要比对左右侧变量类型，左侧类型可以等待任务光标完成编辑，右侧却不行，因此任务光标到达*castToLeftSideType*变量处时，右侧变量需要是已知类型。

![castToLeftSideType模板配置](https://gitee.com/gonghs/image/raw/master/img/20200605233922.png)

演示：

![castToLeftSideType模板演示](https://gitee.com/gonghs/image/raw/master/img/20200605234456.gif)

*rightSideType*可以获取右侧类型作为默认值，因此任务光标到达时右侧变量也需要是已知类型。**需要注意的是*rightSideType*似乎必须定义一个默认值，否则将获取不到任何提示**。

![rightSideType模板配置](https://gitee.com/gonghs/image/raw/master/img/20200605235759.png)演示：

![rightSideType模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606001000.gif)

### className~qualifiedClassName

*className*返回当前类名，可用作构造函数预定义构造函数，日志对象之类的模板。

*currentPackage*返回当前包名。

*qualifiedClassName*则是二者的拼接。

![class模板配置](https://gitee.com/gonghs/image/raw/master/img/20200607031712.png)

演示：

![class模板演示](https://gitee.com/gonghs/image/raw/master/img/20200607031858.gif)

### clipboard

返回剪贴板内容。

演示：

![clipboard模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606002415.gif)

### componentTypeOf(\<array\>)  

返回参数的数组类型 。

![componentTypeOf模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606194718.png)

演示：

![componentTypeOf模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606195139.gif)

### concat(\<String\>, ...)  

拼接参数中的所有字符串。

![concat模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606195705.png)

演示：

![concat模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606195914.gif)

### date([format])

指定格式化方式返回当前系统时间字符串，格式化字符串遵循*SimpleDateFormat*格式。

![date模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606200727.png)

演示：

![date模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606201051.gif)

### descendantClassesEnum(\<String\>)

返回指定类的子类。

![descendantClassesEnum模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606202904.png)

演示：

![descendantClassesEnum模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606203009.gif)

### enum(\<String\>, ...)

自行指定返回的字符串列表。

![enum模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606203238.png)

演示：

![enum模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606203340.gif)

### expectedType

自动识别并返回期望的类型，可以用于赋值，方法参数，返回语句处。

![expectedType模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606204639.png)

演示：

![expectedType模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606204911.gif)

### fileName~fileRelativePath

*fileName*返回当前文件名（带拓展名）。

*fileNameWithoutExtension*返回当前文件名（不带拓展名）。

*filePath*返回文件全路径（带拓展名）。

*fileRelativePath*返回文件相对当前项目的路径（带拓展名）。

![fileName模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606205424.png)

演示：

![fileName模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606210337.gif)

### groovyScript(\<String\>, [arg, ...])

执行作为字符串形式传递的*groovy*脚本，第一个参数为脚本内容或脚本文件路径 ，之后的参数都为可选参数。

如果要在脚本中调用可选参数可以使用*\_1，\_2，\_3*以此类推，要访问当前编辑器可以使用*\_editor*变量。

此段脚本为两个变量做一个简单的拼接：

```
groovyScript("return \"${_1}\" + \"${_2}\"",var1,var2)
```

![groovyScript模板配置](https://gitee.com/gonghs/image/raw/master/img/20200607003349.png)

演示：

![groovyScript模板演示](https://gitee.com/gonghs/image/raw/master/img/20200607003539.gif)

### guessElementType(\<Collection\>)

返回集合中的泛型类型。

![guessElementType模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606215632.png)

演示：

![guessElementType模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606220047.gif)

### iterableComponentType(\<Iterable\>)

返回可迭代对象中的泛型类型，使用于数组，对象及其他任意实现Iterable接口的对象。

![iterableComponentType模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606220543.png)

演示：

![iterableComponentType模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606220732.gif)

### classNameComplete

触发一次类名相关的补全提示。

演示：

![classNameComplete模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606003545.gif)



### methodParameters

获取所有的参数名，返回时在外面拼接[]。

演示：

![methodParameters模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606012053.gif)

### regularExpression(\<String\>, \<Pattern\>, \<Replacement\>)

查找字符串中满足\<Pattern\>的所有部分并替换为\<Replacement\>  ，支持所有标准正则表达式。

![regularExpression模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606221424.png)

演示：

![regularExpression模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606222357.gif)

### typeOfVariable(\<String\>)

返回变量的类型。

![typeOfVariable模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606230055.png)

演示：

![typeOfVariable模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606230237.gif)

### variableOfType(\<String\>)~ subtypes(\<String\>) 

*variableOfType*返回所有满足类型条件的变量，如果传入""则会返回所有的可用变量，距离较近的变量优先展示。

*suggestFirstVariableName*和*variableOfType*类似，但不会推荐true，false，this，和super。

![variableOfType和suggestFirstVariableName对比](https://gitee.com/gonghs/image/raw/master/img/20200607001701.png)

*subtypes*返回指定类型的子类型。

![variableOfType与subtypes模板配置](https://gitee.com/gonghs/image/raw/master/img/20200606232013.png)

演示：

![variableOfType与subtypes模板演示](https://gitee.com/gonghs/image/raw/master/img/20200606232351.gif)