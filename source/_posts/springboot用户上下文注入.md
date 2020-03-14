---
title: springboot用户上下文注入
date: 2019-08-20 14:54:32
categories: 
- kotlin web
tags:
- springboot
- gradle
- kotlin
- web
description: 在分布式系统中通常权限相关的系统是和业务系统分开的,在权限系统验证完毕后,随后的业务请求通常将包含用户信息的相关值放到请求头部(可能是一个redis缓存key,或者其他某种存储途径的标识,也可能直接将json加密之后放在头部),此篇所谈的即是在这种情况下,封装一个通用的用户上下文让业务系统使用起来尽可能的简洁
---
# 版本概要

--- 
> springboot版本2.1.7.RELEASE 
> kotlin版本1.2.71   
> gradle版本5.2.1  
> idea版本2019.1.2 ultimate edition
---

# 新建项目

- 点击file -> new project ->选择spring initializrd点击下一步
- 选择语言,选择项目管理工具
    ![image.png](https://i.loli.net/2019/09/27/hcpGX7TLjZ5kUsb.png)
- 此篇讨论我们只进行数据模拟,不涉及实际数据,只引入springWebStarter进行请求测试即可
- 选择gradle路径(或者使用默认的),这里我选择本地路径
    ![image.png](https://i.loli.net/2019/09/27/9cBA3UaqRNFPpfH.png)
- 增加国内镜像地址  
    追加根节点
    ```kotlin
    repositories {
        maven (url = "http://maven.aliyun.com/nexus/content/groups/public/")
        jcenter()
    } 
    ```
- 增加fastJson依赖用以序列化
    在dependencies中追加
    ```kotlin
    implementation("com.alibaba:fastjson:1.2.59")
    ```
- 重新导入等待编译完成
    
# 控制器注入

使用方法解析器,我们能够在控制器中有选择的解析并注入参数
- 用户上下文对象和标记注解:
```kotlin
data class UserContext(val userId: String, val username: String)

@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class CurrentUser
```
- 配置方法解析器,给加上标记注解的UserContext对象自动解析请求头中的json信息并注入
```kotlin
class CurrentUserMethodArgumentResolver : HandlerMethodArgumentResolver {
    /**
     * 符合条件才进入此参数解析器
     */
    override fun supportsParameter(parameter: MethodParameter): Boolean {
        return parameter.parameterType.isAssignableFrom(UserContext::class.java)
                && parameter.hasParameterAnnotation(CurrentUser::class.java)
    }

    /**
     * 参数解析并注入对象
     */
    override fun resolveArgument(parameter: MethodParameter, mavContainer: ModelAndViewContainer?, webRequest: NativeWebRequest, binderFactory: WebDataBinderFactory?): Any? {
        val userJson = webRequest.getHeader("user-test")
        return JSON.parseObject(userJson, UserContext::class.java)
    }
}
```
- 启用方法解析器
```kotlin
@Configuration
class SpringConfig : WebMvcConfigurer {
    /**
     * 加入解析器列表
     */
    override fun addArgumentResolvers(resolvers: MutableList<HandlerMethodArgumentResolver>) {
        super.addArgumentResolvers(resolvers)
        resolvers.add(CurrentUserMethodArgumentResolver())
    }
}
```
- 测试控制器
```kotlin
@SpringBootApplication
@RestController
class DemoApplication {
    @GetMapping("/getArgument")
    fun getArgument(@CurrentUser userContext: UserContext):UserContext {
        return userContext
    }
}
```
- 编写WebMvc测试类测试结果
```kotlin
@RunWith(SpringRunner::class)
@WebMvcTest(DemoApplication::class)
class WebMvcTest {
    private val log = LoggerFactory.getLogger(this.javaClass)
    @Autowired
    lateinit var mockMvc: MockMvc

    @Test
    fun testGetArgument() {
        val json = "{\"username\":\"测试\", \"userId\":\"测试\"}"
        mockMvc.perform(MockMvcRequestBuilders.get("/getArgument")
                .header("user-test", json))
                .andExpect(status().isOk).andDo { log.info("返回结果 ${it.response.contentAsString}") }.andReturn()
    }
}
```
- 测试结果
![image.png](https://i.loli.net/2019/09/27/P7qCouQgXO25IkU.png)
- 使用此方式我们可以很方便在需要时将用户上下文注入控制器中,并且只有需要时,才会进行参数解析

# 静态方法获取

使用构造器注入的方式,不方便之处在于当我们在service层需要使用时,只能一层一层的向内传,对我们的方法参数造成的一定程度上的污染,我们可以利用线程安全的ThreadLocal对象在每次请求时存储用户上下文
- RequestContext对象
```kotlin
object RequestContext {
    /**
     * 用户上下文
     */
    private val userContextThreadLocal: ThreadLocal<UserContext> = ThreadLocal()

    fun setUserContext(userContext: UserContext) {
        userContextThreadLocal.set(userContext)
    }

    fun getUserContext(): UserContext {
        return userContextThreadLocal.get()
    }

    fun removeUserContext() {
        userContextThreadLocal.remove()
    }
}
```
- 使用spring提供的HandlerInterceptor接口我们可以跟踪请求，解析参数，并及时释放本地线程中的对象
```kotlin
class RequestInterceptor : HandlerInterceptor {
    /**
     * 如果让请求继续执行则返回true
     */
    override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
        val userJson = request.getHeader("user-test")
        if (userJson.isNullOrBlank()) {
            return true
        }

        RequestContext.setUserContext(JSON.parseObject(userJson, UserContext::class.java))
        return super.preHandle(request, response, handler)
    }

    /**
     * 请求结束时移除上下文，抛出异常也会执行
     */
    override fun afterCompletion(request: HttpServletRequest, response: HttpServletResponse, handler: Any, ex: Exception?) {
        RequestContext.removeUserContext()
    }
}
```
- 在spring中配置此拦截器
```kotlin
/**
 * 加入拦截器列表
 */
override fun addInterceptors(registry: InterceptorRegistry) {
    super.addInterceptors(registry)
    registry.addInterceptor(RequestInterceptor())
}
```
- 测试控制器
```kotlin
@GetMapping("/getStatic")
fun getStatic(): UserContext {
//        throw RuntimeException("啊偶 出错了")
    return RequestContext.getUserContext()
}
```
- 测试方法
```kotlin
@Test
fun testGetStatic() {
    val json = "{\"username\":\"测试1\", \"userId\":\"测试1\"}"
    mockMvc.perform(MockMvcRequestBuilders.get("/getStatic")
            .header("user-test", json))
            .andExpect(status().isOk).andDo { log.info("返回结果 ${it.response.contentAsString}") }.andReturn()
}
```
- 测试结果
![image.png](https://i.loli.net/2019/09/27/9pH5MPTWqbS3nL6.png)
- 由于提供的都是静态方法，使用此方式我们就可以在任何地方使用用户上下文对象(注意避免空指针)，例如，我们就可以使用mybatis拦截器替我们完成userId等属性的注入。

# bean获取

由于静态类不由spring管理，业务类使用时不免使代码的耦合性变强，当我们需要变更方案时将会比较麻烦，因此我们希望将类委托spring进行管理
## 方案1(单例bean)
- 我们提供一个接口，向外暴露一个getter方法，在getter方法中调用静态方法获取
```kotlin
interface UserContextManage {
    fun getUserContext(): UserContext {
        return RequestContext.getUserContext()
    }
}
```
- 配置bean
```kotlin
@Bean
fun userContextManage(): UserContextManage {
    return object : UserContextManage {}
}
```
- 控制器
```kotlin
@Autowired
lateinit var userContextManage: UserContextManage

@GetMapping("/getSingletonBean")
fun getSingletonBean(): UserContext {
    return userContextManage.getUserContext()
}
```
- 测试方法
```kotlin
fun testGetSingletonBean() {
    val json = "{\"username\":\"测试3\", \"userId\":\"测试3\"}"
    mockMvc.perform(MockMvcRequestBuilders.get("/getSingletonBean")
            .header("user-test", json))
            .andExpect(status().isOk).andDo { log.info("返回结果 ${it.response.contentAsString}") }.andReturn()
}
```
- 测试结果
![image.png](https://i.loli.net/2019/09/28/GBvE9C3Rb5f8MO1.png)
- 需要额外提及的是，当我们在业务层依赖此对象时，单元测试由于不涉及请求导致用户上下文为空，这里推荐使用mockBean进行测试
```kotlin
@MockBean
lateinit var userContextManage: UserContextManage
@Before
fun before() {
    Mockito.`when`(userContextManage.getUserContext()).thenReturn(UserContext("mock测试","mock测试"))
}
@Test
fun testMockSingletonBean() {
    log.info(userContextManage.getUserContext().toString())
}
```
- mock结果
![image.png](https://i.loli.net/2019/09/28/evKwuSRcCykhZIH.png)

## 方案2(请求bean)
spring为我们提供了scope为request的bean，例如httpServletRequest就是一个这种bean，这种类型的bean的生命周期和请求是息息相关的，伴随的请求开始和结束进行创建和销毁
- 声明bean
```kotlin
@Bean
@Scope(WebApplicationContext.SCOPE_REQUEST)
fun userContext(): UserContext {
    return RequestContext.getUserContext()
}
```
- 控制器
```kotlin
@Autowired(required = false)
var userContext: UserContext? = null
@GetMapping("/getRequestBean")
fun getRequestBean(): UserContext {
    return userContext!!
}
```
- 测试方法
```kotlin
@Test
fun testGetRequestBean() {
    val json = "{\"username\":\"测试4\", \"userId\":\"测试4\"}"
    mockMvc.perform(MockMvcRequestBuilders.get("/getRequestBean")
            .header("user-test", json))
            .andExpect(status().isOk).andDo { log.info("返回结果 ${it.response.contentAsString}") }.andReturn()
}
```
- 在测试时我们会发现，哪怕我们不要求spring为我们一定要注入这个bean，spring还是会尝试注入并报错
![image.png](https://i.loli.net/2019/09/28/SVj76KGrXnucZ1l.png)
- 这里有几种方案处理这种异常
    - 方式1 引入javax inject依赖
    ```xml
    implementation("javax.inject:javax.inject:1")
    ```
    - 控制器中注入对象使用Provider包装
    ```kotlin
    @Autowired
    lateinit var userContext: Provider<UserContext>
    @GetMapping("/getRequestBean")
    fun getRequestBean(): UserContext {
        return userContext.get()
    }
    ```
    - 方式2 为bean使用代理 注意如果使用kotlin不要使用类代理，否则会丢失字段值
    ```kotlin
    // 使用kotlin需要配置消息转换器(java是否需要还未测试)
    @Bean
    fun httpMessageConverter(): HttpMessageConverter<*> {
        //创建fastJson消息转换器
        val fastConverter = FastJsonHttpMessageConverter()

        //升级最新版本需加=============================================================
        val supportedMediaTypes = ArrayList<MediaType>()
        supportedMediaTypes.add(MediaType.APPLICATION_JSON)
        supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8)
        supportedMediaTypes.add(MediaType.APPLICATION_ATOM_XML)
        supportedMediaTypes.add(MediaType.APPLICATION_FORM_URLENCODED)
        supportedMediaTypes.add(MediaType.APPLICATION_OCTET_STREAM)
        supportedMediaTypes.add(MediaType.APPLICATION_PDF)
        supportedMediaTypes.add(MediaType.APPLICATION_RSS_XML)
        supportedMediaTypes.add(MediaType.APPLICATION_XHTML_XML)
        supportedMediaTypes.add(MediaType.APPLICATION_XML)
        supportedMediaTypes.add(MediaType.MULTIPART_FORM_DATA)
        supportedMediaTypes.add(MediaType.IMAGE_GIF)
        supportedMediaTypes.add(MediaType.IMAGE_JPEG)
        supportedMediaTypes.add(MediaType.IMAGE_PNG)
        supportedMediaTypes.add(MediaType.TEXT_EVENT_STREAM)
        supportedMediaTypes.add(MediaType.TEXT_HTML)
        supportedMediaTypes.add(MediaType.TEXT_MARKDOWN)
        supportedMediaTypes.add(MediaType.TEXT_PLAIN)
        supportedMediaTypes.add(MediaType.TEXT_XML)
        fastConverter.supportedMediaTypes = supportedMediaTypes

        //创建配置类
        val fastJsonConfig = FastJsonConfig()
        //修改配置返回内容的过滤
        //WriteNullListAsEmpty  ：List字段如果为null,输出为[],而非null
        //WriteNullStringAsEmpty ： 字符类型字段如果为null,输出为"",而非null
        //DisableCircularReferenceDetect ：消除对同一对象循环引用的问题，默认为false（如果不配置有可能会进入死循环）
        //WriteNullBooleanAsFalse：Boolean字段如果为null,输出为false,而非null
        //WriteMapNullValue：是否输出值为null的字段,默认为false
        fastJsonConfig.setSerializerFeatures(
                SerializerFeature.DisableCircularReferenceDetect,
                SerializerFeature.WriteMapNullValue,
                SerializerFeature.WriteNullStringAsEmpty,
                SerializerFeature.WriteMapNullValue
        )
        fastConverter.fastJsonConfig = fastJsonConfig

        return fastConverter
    }
    ```
    - 修改用户上下文对象
    ```kotlin
    open class UserContext(override val userId: String, override val username: String) : IUserContext
    
    interface IUserContext {
        val userId: String
        val username: String
    }
    ```
    - 接口代理
    ```kotlin
    @Bean
    @Scope(WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.INTERFACES)
    fun iUserContext(): IUserContext {
        return RequestContext.getUserContext()
    }
    ```
    - 控制器注入
    ```kotlin
    @Autowired
    lateinit var userContext: IUserContext
    @GetMapping("/getRequestBean")
    fun getRequestBean(): IUserContext {
        return userContext
    }
    ```
    
- 处理之后测试运行结果
![image.png](https://i.loli.net/2019/09/28/QxM3EBGcDRe9Jz7.png)
- 此方式使用将bean委托spring管理，耦合性较低，并且如果使用代理的方式用起来会更加方便，但需要注意的是，由于使用了代理，在请求不涉及用户上下文(即获取用户上下文为空)的情况下调用代理对象将直接抛异常(无法使用==null做空判断)

# [代码链接](https://github.com/gonghs/kotlin-web/tree/master/kotlin-springboot-user-context-inject)
