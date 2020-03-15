---
title: springboot初探和配置文件映射
date: 2019-01-15 16:17:47
categories: 
- kotlin web
tags:
- springboot
- kotlin
- web
description: 此篇讲述使用idea构建kotlin springboot项目,和基本的配置文件映射操作
---
# 版本概要

---
> springboot版本2.1.2.RELEASE  
> kotlin版本1.3.11
---

# 新建项目
- 选择Spring Initializr
- 在metadata页面中选择kotlin作为语言
    ![image.png](https://i.loli.net/2019/09/28/fR5BJe9FXrNvMhb.png)
- 依赖勾选web
    ![image.png](https://i.loli.net/2019/09/28/vObIGPTYgKLFSfh.png)
- 将kotlin文件夹设为项目资源文件夹,并等待项目依赖下载完毕
    ![image.png](https://i.loli.net/2019/09/28/3X7mJgEG4kpZLhb.png)
- 新建测试控制器并设置/hello为默认映射路径
    ```kotlin
    @Configuration
    class SpringConfig : WebMvcConfigurer {
        override fun addViewControllers(registry: ViewControllerRegistry) {
            registry.addViewController("/").setViewName("forward:/hello")
            registry.setOrder(Ordered.HIGHEST_PRECEDENCE)
        }
    }
    
    @RestController
    class HelloController {
        @GetMapping("/hello")
        fun hello(): String {
            return "hello"
        }
    }
    ```
- 启动项目访问页面  
    ![image.png](https://i.loli.net/2019/09/28/m1hQnBGlIpAaYi6.png)


# 读取配置文件
- 新建配置文件test.yml,并添加几个值(如果需要yml配置文件的提示可以安装spring assistant插件,或者点项目配置将其加入配置列表)
    ![image.png](https://i.loli.net/2019/09/28/WLBMZfFOdqnXYeK.png)
    ```kotlin
    person:
      name: maple
      sex: man
      phone: 18111111111
      children: {name: merry,sex: woman}
      lists:
        - 1
        - 2
    
    // 如果需要使用传统properties配置需要引入依赖
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <optional>true</optional>
    </dependency>
    ```
- 修改控制器为属性注入值,注意由于kotlin字符串中的$本身有其含义,因此需要加反斜杠转义
    ```kotlin
    @Value("\${person.name}")
    private lateinit var username:String
    
    @GetMapping("/hello")
    fun hello(): String {
        return "hello $username"
    }
    ```
- 访问项目
    ![image.png](https://i.loli.net/2019/09/28/flnDexAr8PJUNoi.png)
- 实体映射,由于默认的工厂对自定义yml解析有问题,新建映射工厂类解析yml,新建实体并修改控制器(如果需要使用kotlin中的data类型 需要手动注bean)
    ```kotlin
    //解析工厂
    class YamlPropertySourceFactory : PropertySourceFactory {
        @Throws(IOException::class)
        override fun createPropertySource(name: String?, resource: EncodedResource): PropertySource<*> {
            return if (name != null)
                YamlPropertySourceLoader().load(name, resource.resource)[0]
            else
                YamlPropertySourceLoader().load(
                    getNameForResource(resource.resource), resource.resource)[0]
        }
        private fun getNameForResource(resource: Resource): String {
            var name = resource.getDescription()
            if (!StringUtils.hasText(name)) {
                name = resource::class.java.getSimpleName() + "@" + System.identityHashCode(resource)
            }
            return name
        }
    }
    //映射类
    @Configuration
    @ConfigurationProperties(prefix = "person")
    @PropertySource("classpath:/test.yml",factory = YamlPropertySourceFactory::class)
    class UserProperties {
        lateinit var name: String
        lateinit var sex: String
        var phone = 0L
        lateinit var children: Map<String,String>
        lateinit var lists: List<String>
        override fun toString():String{
            return "name:$name,sex:$sex,phone:$phone,children:$children,lists:$lists"
        }
    }
    //修改 controller
    @Autowired
    private lateinit var userProperties:UserProperties
    @GetMapping("/hello")
    fun hello(): String {
        return userProperties.toString()
    }
    ```
- 启动测试
    ![image.png](https://i.loli.net/2019/09/28/3oMwdXUGt5B1gPZ.png)
- 另:使用kotlin data class类型注入(对于map类型的属性暂时没有找到特别好的办法解决)
    ```kotlin
    //修改映射类
    @Configuration
    @PropertySource("classpath:/test.yml",factory = YamlPropertySourceFactory::class)
    data class UserProperties(@Value("\${person.name}")val name:String, @Value("\${person.sex}")val sex:String, @Value("\{person.phone}")val phone:Long)
    
    //注,由于@value注解本身对于list map的支持并非很友好 因此并不推荐
    //配置list
    list: 1,2
    //取值注入
    @Value("#{'\${person.lists}'.split(',')}")
    private lateinit var lists:List<String>
    //或者激活支持转换String为Collection类型的新配置服务
    @Bean
    fun conversionService(): ConversionService{
        return DefaultConversionService()
    }
    //这时list将不用再特殊处理 但配置仍旧只能保持字符串形式
    @Value("\${person.lists}")
    private lateinit var lists:List<String>
    ```

# [代码链接](https://github.com/gonghs/kotlin-web)  
        