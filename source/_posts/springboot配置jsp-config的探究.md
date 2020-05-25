---
title: springboot配置jsp-config的探究
date: 2019-12-06 16:17:47
categories:
  - java web
tags:
  - springboot
  - java
  - web
  - jsp
description: 探究在springboot中寻找jsp-config的替代方案
---

# 版本概要

---

> springboot 版本 2.2.2.RELEASE
> jdk 版本 1.8  
> maven 版本 3.6.0  
> idea 版本 2019.1.2 ultimate edition

---

# 项目准备

- 新建 spring Initializr 项目选择一些基本的依赖即可
  ![image.png](https://i.loli.net/2019/12/05/9NjezIZxn783urs.png)
- 在 pom 中追加 jsp 需要的依赖和资源文件扫描
  ```xml
  <dependency>
      <groupId>org.apache.tomcat.embed</groupId>
      <artifactId>tomcat-embed-jasper</artifactId>
  </dependency>
  <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>javax.servlet-api</artifactId>
      <scope>provided</scope>
  </dependency>
  <dependency>
      <groupId>javax.servlet</groupId>
      <artifactId>jstl</artifactId>
  </dependency>
  <build>
    <resources>
      <resource>
        <directory>src/main/webapp/</directory>
        <!--注意必须要放在此目录下才能被访问到 -->
        <targetPath>META-INF/resources</targetPath>
        <includes>
            <include>**/**</include>
        </includes>
      </resource>
    </resources>
  </build>
  ```
- 新建文件夹 webapp 与 resources 文件夹平级,并新建 index.jsp
  - 目录结构
    ```text
    .
    ├── resources
    ├── webapp
    │   └── WEB-INF
    │       ├── index.jsp
    ```
  - 内容
    ```html
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8" />
        <title>index</title>
      </head>
      <body>
        <c:if test="${bool}">测试c if</c:if>
        hello world 哈哈哈哈哈哈
      </body>
    </html>
    ```
- 在 yml 文件中配置访问文件夹
  ```yml
  spring:
  mvc:
    view:
      prefix: /WEB-INF/jsp/
      suffix: .jsp
  ```
- 新建访问控制器
  ```java
  @Controller
  public class IndexController {
      @GetMapping
      public String index() {
          model.addAttribute("bool", true);
          return "index";
      }
  }
  ```

# web.xml中的jsp-config标签

准备工作完成后,项目事实上已经可以成功访问 jsp 页面了,但我们访问时会发现 jsp 页面的中文乱码了,并且c标签也并未生效
![image.png](https://i.loli.net/2019/12/05/od1sm6wkyTZEvKY.png) 
在老项目中我们发现 jsp 在 web.xml 文件中存在这么个配置
```xml
<jsp-config>
    <jsp-property-group>
        <el-ignored>false</el-ignored>
        <page-encoding>UTF-8</page-encoding>
        <include-prelude>/WEB-INF/tags/taglibs.jspf<include-prelude>
    </jsp-property-group>
</jsp-config>
```
taglibs.jspf文件中包含了一些通用的标签引入
```jsp
<%@taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>
<%@taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
```
这个配置的作用在于为所有的 jsp 页面增加统一的编码,和统一的头部信息,这些配置相当于在 jsp 页面头部加入:

```jsp
<%@ include file="/WEB-INF/tags/taglibs.jspf" %>
<%@ page pageEncoding="UTF-8" isELIgnored="false"  %>
```
我们在index.jsp头部中加入上面那段代码,可以看到乱码解决了标签也生效了
![image.png](https://i.loli.net/2019/12/05/HYBniuxyljbJTGd.png)
在旧项目的改造中,如果我们需要在所有页面追加这个头部,那么工作量就太大了,我们需要找到一种方式使jsp-config的api在springboot中同样生效

# 在springboot中配置jsp-config

## 外部tomcat启动配置

springboot本身是支持web.xml的,因此我们可以在WEB-INF下直接放入web.xml其中仅配置jsp-config
  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <web-app version="4.0"
           xmlns="http://xmlns.jcp.org/xml/ns/javaee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
          	 http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd">
      <jsp-config>
          <jsp-property-group>
              <url-pattern>*.jsp</url-pattern>
              <el-ignored>false</el-ignored>
              <page-encoding>UTF-8</page-encoding>
              <include-prelude>/WEB-INF/tags/taglibs.jspf</include-prelude>
          </jsp-property-group>
      </jsp-config>
  </web-app>
  ```
  我们发现当springboot使用嵌入式容器启动时是忽略web.xml的,因此我们需要配置使用外部tomcat启动

### 在idea中配置嵌入式容器启动

- 修改pom 打包方式改为war包
  ```xml
  <packaging>war</packaging>
  ```
- 修改启动类
  ```java
  @SpringBootApplication
  public class BootJspApplication extends SpringBootServletInitializer {
      public static void main(String[] args) {
          SpringApplication.run(BootJspApplication.class, args);
      }

      @Override
      protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
          return builder.sources(BootJspApplication.class);
      }
  }
  ```
- 配置tomcat启动
    ![image.png](https://i.loli.net/2019/12/06/1zefpF4giRJoskQ.png)
- 使用tomcat启动web.xml中的内容便生效了

## 嵌入式容器配置(java config配置)
- 定制嵌入式容器,在tomcat的容器上下文中我们可以找到Jsp相关的配置类并配置它
  ```java
  @Bean
  public ConfigurableServletWebServerFactory configurableServletWebServerFactory() {
      return new TomcatServletWebServerFactory() {
          @Override
          protected void postProcessContext(Context context) {
              super.postProcessContext(context);
              JspPropertyGroup jspPropertyGroup = new JspPropertyGroup();
              jspPropertyGroup.setElIgnored("false");
              jspPropertyGroup.addUrlPattern("*.jsp");
              jspPropertyGroup.setPageEncoding("UTF-8");
              jspPropertyGroup.addIncludePrelude("/WEB-INF/tags/taglibs.jspf");
              JspPropertyGroupDescriptorImpl jspPropertyGroupDescriptor =
                      new JspPropertyGroupDescriptorImpl(jspPropertyGroup);
              // jsp-property-group列表和taglib列表
              context.setJspConfigDescriptor(new JspConfigDescriptorImpl(Collections.singletonList(jspPropertyGroupDescriptor),
                      Collections.emptyList()));
          }
      };
  }
  ```
- 加入这个bean以后使用jar包方式重启项目,可以看到这个配置也生效了  

**注:除了以上方式外,目前还没有找到更好的方式使用java config使该配置生效,因此使用该配置相当于与tomcat嵌入式容器绑定了(ConfigurableServletWebServerFactory这个bean唯有在tomcat中有jsp的对应配置实现)**

## 可能的其他方式
- 在javax.servlet.ServletContext接口中有一个getJspConfigDescriptor方法,这给我们提供了一个修改jsp配置的可能,我们有几种可能的方式获取到ServletContext
  ```java
  // 可能修改的方法
  servletContext.getJspConfigDescriptor().getJspPropertyGroups().add(new JspPropertyGroupDescriptorImpl());
  // 实现ServletContextListener接口
  @Component
  public class JspConfig implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        sce.getServletContext();
    }
  }
  // 实现ServletContextInitializer接口
  @Component
  public class JspConfig implements ServletContextInitializer {
      @Override
      public void onStartup(ServletContext servletContext) throws ServletException {
          servletContext.getJspConfigDescriptor();
      }
  }
  ```
  这两种方式可以获取到servletContext,但在未读取web.xml的情况下,后者获取到的Jsp config描述器为null,而前者则在调用时直接抛出一个异常
  ```
  // 实现ServletContainerInitializer接口
  @Component
  public class JspConfig implements ServletContainerInitializer {
      @Override
      public void onStartup(Set<Class<?>> set, ServletContext servletContext) throws ServletException {
          servletContext.getJspConfigDescriptor();
      }
  }
  ```
  此方式与前面的第二种方式类似,不同的是此方式仅在程序使用外部tomcat容器启动时才会执行

# [代码链接](https://github.com/gonghs/java-web/tree/master/springboot-jsp)