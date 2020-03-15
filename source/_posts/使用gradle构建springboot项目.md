---
title: 使用gradle构建springboot项目
date: 2019-03-03 16:17:47
categories: 
- kotlin web
tags:
- springboot
- kotlin
- web
description: 此篇讲述使用idea构建kotlin gradle springboot项目,和基本的部署操作
---
# 版本概要

--- 
> springboot版本2.1.3.RELEASE  
> kotlin版本1.3.21   
> gradle版本5.2.1  
> idea版本2018.2.6 ultimate edition
---

# 新建项目

- 点击file -> new project -> 选择新建gradle项目
    ![image.png](https://i.loli.net/2019/09/28/Gi1Rs6D9ILOaTon.png)
- 输入groupId和artifactId 进入下一步
- 勾选使用本地gradle路径,选择gradle所在根路径(即bin的上层路径) 进入下一步
    ![image.png](https://i.loli.net/2019/09/28/LETbvJpMadqXxOe.png)
- 选择项目路径 点击finish等待项目构建完成

# 引入springboot

- 修改maven依赖访问地址,使用国内镜像
    - 在build.gradle中加入
        ```
        repositories {
            maven{
                url 'http://maven.aliyun.com/nexus/content/groups/public/'
            }
        }
        ```
- 引入springboot
    - 在plugins节点中加入
        ```
        id 'org.springframework.boot' version '2.1.3.RELEASE'
        ```
    - 加入根节点 使用spingboot插件(即最顶层)
        ```
        apply plugin: 'io.spring.dependency-management'

        ```
    - 引入springboot web和test依赖 在dependencies节点加入
        ```
        implementation 'org.springframework.boot:spring-boot-starter-web'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        ```
- 新建启动类并测试
    - 在java文件夹或kotlin文件夹下创建springboot启动类
        ```kotlin
        @SpringBootApplication
        open class SpringbootRun
        
        @RestController
        class HelloController {
            @GetMapping
            fun hello(): String {
                return "hello"
            }
        }
        
        fun main() {
            runApplication<SpringbootRun>()
        }
        ```
    - 点击右侧bootRun尝试启动(注意这里bootRun会自动扫描main方法,如果存在多个main方法只会选择其中一个),或者使用传统方式启动
        ![image.png](https://i.loli.net/2019/09/28/zZRoTUX36k4GOtV.png)
    - 访问localhost:8080查看结果
        ![image.png](https://i.loli.net/2019/09/28/CjK4qsF6AI93bdW.png)
- 打成jar包并运行
    - 点击右侧build下bootJar
        ![image.png](https://i.loli.net/2019/09/28/ihwdZ64DHVLtAO1.png)
    - 项目下build/libs/将会生成一个jar包
        ![image.png](https://i.loli.net/2019/09/28/ZceH7LAsOfXpiNj.png)
    - 使用命令行运行,并访问
        ![image.png](https://i.loli.net/2019/09/28/IKtVEFskcBrZLou.png)
- 打成war包并运行
    - 修改build.gradle 加入根节点
        ```
        apply plugin: 'war'
        ```
    - 修改启动类使其继承SpringBootServletInitializer
        ```kotlin
        @SpringBootApplication
        open class SpringbootRun : SpringBootServletInitializer()
        ```
    - 点击右侧bootWar
    - 拷贝war包至tomcat安装路径webapps下
    - 运行bin/startup.bat 启动tomcat并尝试访问
    - 访问结果
        ![image.png](https://i.loli.net/2019/09/28/yWQzl7G5ikmPTfF.png)
- 打成war包使用jetty运行
    - 修改build.gradle 加入根节点
        ```kotlin
        configurations {
            compile.exclude module: "spring-boot-starter-tomcat"
        }
        ```
    - 加入依赖
        ```
        implementation 'org.springframework.boot:spring-boot-starter-jetty'
        ```
    - 点击右侧bootWar
    - 拷贝war包至jetty安装路径webapps下
    - 运行,并访问
        ```
        java -jar start.jar
        ```
    - 访问结果:
        ![image.png](https://i.loli.net/2019/09/28/fTC3hg4ajxNXQqr.png)
        ![image.png](https://i.loli.net/2019/09/28/FLZB4ayjwv2t8cE.png)
    
# [代码链接](https://github.com/gonghs/kotlin-web/tree/master/springboot-gradle)