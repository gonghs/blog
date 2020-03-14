---
title: redis和redisson分布式锁集成
date: 2019-01-20 16:42:19
categories: 
- kotlin web
tags:
- springboot
- kotlin
- web
description: 此篇讲述使用springboot集成redis和redisson分布式锁
---
# 版本概要

---
> springboot版本2.1.2.RELEASE  
> kotlin版本1.3.11 

# *新建项目请参考{% post_link springboot初探和配置文件映射  %}*
---

# redis集成

- 新增依赖
    ```kotlin
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    ```
- 新增配置项
    ```kotlin
    spring:
      redis:
        host: 127.0.0.1
        port: 6379
        timeout: 1000ms
    ```
- 配置redisTemplate
    ```kotlin
    @Configuration
    class RedisConfig{
    
        /**
         * 将redisTemplate格式化为string,any格式
         *
         * @param factory redis连接工厂
         * @return redisTemplate
         */
        @Bean
        fun redisTemplate(factory: RedisConnectionFactory):RedisTemplate<String,Any> {
            val template = RedisTemplate<String,Any>()
            template.setConnectionFactory(factory)
            val jackson2JsonRedisSerializer = Jackson2JsonRedisSerializer(Any::class.java)
            val om = ObjectMapper()
            om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY)
            om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL)
            jackson2JsonRedisSerializer.setObjectMapper(om)
            val stringRedisSerializer = StringRedisSerializer()
            template.keySerializer = stringRedisSerializer
            template.hashKeySerializer = stringRedisSerializer
            template.valueSerializer = jackson2JsonRedisSerializer
            template.hashValueSerializer = jackson2JsonRedisSerializer
            template.afterPropertiesSet()
            return template
        }
    }
    ```
- 新建测试类,继承生成的测试类,并运行
    ```kotlin
    class RedisTest : KotlinSpringbootApplicationTests() {

        private val log = LoggerFactory.getLogger(this.javaClass)
        private val testKey = this.javaClass.name

        @Autowired
        lateinit var redisTemplate: RedisTemplate<String, Any>

        @Test
        fun testString() {
            log.info("---设置值---")
            redisTemplate.opsForValue().set(testKey, "hello")
            val str = redisTemplate.opsForValue().get(testKey) as? String
            log.info("---打印值:$str---")
            redisTemplate.delete(testKey)
        }
    }
    ```
    ![image.png](https://i.loli.net/2019/09/28/zNA1sD8uUaVt6FY.png)
- 尝试进行对象存储
    ```kotlin
    data class User(val username:String, val sex:String,val phone:Long)
    //新增测试类
    @Test
    fun testAny() {
        val user = User("maple","man",18011111111)
        log.info("---设置对象---")
        redisTemplate.opsForValue().set(testKey, user)
        val user = redisTemplate.opsForValue().get(testKey) as? User
        log.info("---打印值:$user---")
        redisTemplate.delete(testKey)
    }
    ```
    ![image.png](https://i.loli.net/2019/09/28/REJpaqO7G6bdTHQ.png)
    我们会发现当前的序列化工具对于kotlin对象的序列化并不是那么理想,我们需要重写一个序列化工具
## 方式1:使用hessian帮助我们进行序列化
    ```kotlin
    //依赖
    <hessian.vesion>4.0.51</hessian.vesion>
    <dependency>
        <groupId>com.caucho</groupId>
        <artifactId>hessian</artifactId>
        <version>${hessian.vesion}</version>
    </dependency>
    
    //序列化工具包
    object SerializeUtils{
    
        @Throws(IOException::class)
        fun hessianDeserialize(by: ByteArray?): Any {
            by?: throw NullPointerException()
            return hessianDeserialize(ByteArrayInputStream(by))
        }
    
        @Throws(IOException::class)
        fun hessianDeserialize(input:InputStream): Any{
            return HessianInput(input).readObject()
        }
    
        fun hessianSerialize(obj: Any?): ByteArray {
            obj?: throw NullPointerException()
            try {
                val os = ByteArrayOutputStream()
                val ho = HessianOutput(os)
                ho.writeObject(obj)
                return os.toByteArray()
            } catch (e: Exception) {
                throw e
            }
        }
    
        @Throws(IOException::class)
        fun hessianSerialize(obj: Any,out:OutputStream){
            HessianOutput(out).writeObject(obj)
        }
    
        fun javaSerialize(obj: Any?):ByteArray{
            obj?: throw NullPointerException()
            val os = ByteArrayOutputStream()
            val out = ObjectOutputStream(os)
            out.writeObject(obj)
            return os.toByteArray()
        }
    
        fun javaSerialize(by: ByteArray?):Any{
            by?: throw NullPointerException()
            val `is` = ByteArrayInputStream(by)
            val `in` = ObjectInputStream(`is`)
            return `in`.readObject()
        }
    
    }
    ```
- 新建类实现RedisSerializer(对象需要实现序列化)并修改redis配置
    ```kotlin
    class HessianRedisSerializer<T>(private var clazz: Class<T>) : RedisSerializer<T>{
    
        override fun serialize(t: T?): ByteArray? {
            return SerializeUtils.hessianSerialize(t)
        }
    
        override fun deserialize(bt: ByteArray?): T? {
            if(bt == null || bt.isEmpty()) return null
            @Suppress("UNCHECKED_CAST")
            return SerializeUtils.hessianDeserialize(bt) as? T
        }
    }
    //redis配置
    @Bean
    fun redisTemplate(factory: RedisConnectionFactory):RedisTemplate<String,Any> {
        val template = RedisTemplate<String,Any>()
        template.setConnectionFactory(factory)
        val hessianRedisSerializer = HessianRedisSerializer(Any::class.java)
        val stringRedisSerializer = StringRedisSerializer()
        template.keySerializer = stringRedisSerializer
        template.hashKeySerializer = stringRedisSerializer
        template.valueSerializer = hessianRedisSerializer
        template.hashValueSerializer = hessianRedisSerializer
        template.afterPropertiesSet()
        return template
    }
    //修改实体类 让实体类实现序列化接口
    data class User(val username:String, val sex:String,val phone:Long):Serializable
    ```
- 再次测试,存取成功
    ![image.png](https://i.loli.net/2019/09/28/mzYk71wx6yqMgLj.png)
## 方式2,使用java 对象输入输出流实现(对象需要实现序列化) 比较慢 不推荐
    ```kotlin
    class HessianRedisSerializer<T>(private var clazz: Class<T>) : RedisSerializer<T>{
    
        override fun serialize(t: T?): ByteArray? {
            return SerializeUtils.hessianSerialize(t)
        }
    
        override fun deserialize(bt: ByteArray?): T? {
            if(bt == null || bt.isEmpty()) return null
            @Suppress("UNCHECKED_CAST")
            return SerializeUtils.hessianDeserialize(bt) as? T
        }
    }
    ```
## 方式3,使用fastjson,对象可以不用实现序列化
    ```kotlin
    //依赖
    <fast-json.vesion>1.2.54</fast-json.vesion>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>${fast-json.vesion}</version>
    </dependency>
    //序列化类
    class FastJsonRedisSerializer<T>(private val clazz:Class<T> ) : RedisSerializer<T>{
        private val charset = Charset.forName("utf-8")!!
    
        override fun serialize(t: T?): ByteArray? {
            return JSON.toJSONString(t, SerializerFeature.WriteClassName).toByteArray(charset)
        }
    
        override fun deserialize(bt: ByteArray?): T? {
            if(bt == null || bt.isEmpty()) return null
            val str = String(bt, charset)
            return JSON.parseObject<T>(str, clazz)
        }
    }
    //配置
    @Bean
    fun redisTemplate(factory: RedisConnectionFactory):RedisTemplate<String,Any> {
        val template = RedisTemplate<String,Any>()
        template.setConnectionFactory(factory)
        var fastJsonRedisSerializer = FastJsonRedisSerializer(Any::class.java)
        //配置白名单
        ParserConfig.getGlobalInstance().addAccept("com.maple.kotlinspringboot.entity.User")
        //或者直接关闭这个检测
        //ParserConfig.getGlobalInstance().isAutoTypeSupport = true
        val stringRedisSerializer = StringRedisSerializer()
        template.keySerializer = stringRedisSerializer
        template.hashKeySerializer = stringRedisSerializer
        template.valueSerializer = fastJsonRedisSerializer
        template.hashValueSerializer = fastJsonRedisSerializer
        template.afterPropertiesSet()
        return template
    }
    ```
- 做一个简单的redis工具包
    ```kotlin
    @Component
    class RedisUtils {
        @Autowired
        lateinit var redisTemplate: RedisTemplate<String, Any>
    
    
        /**
         * 获取指定缓存对象
         *
         * @param key 键
         * @return any 值对象
         */
        fun <T : Any> getT(key: String):T? {
            @Suppress("UNCHECKED_CAST")
            return  redisTemplate.opsForValue().get(key) as? T
        }
    
        /**
         * 获取long值
         *
         * @param key 键
         * @return any 值对象
         */
        fun getLong(key: String):Long?{
            val value = getAny(key)
            return when(value){
                is Byte -> value.toLong()
                is Int -> value.toLong()
                is Long -> value
                is Float -> value.toLong()
                is Double -> value.toLong()
                is Char -> value.toLong()
                is String -> value.toLongOrNull()
                else -> null
            }
        }
    
        /**
         * 获取int值
         *
         * @param key 键
         * @return any 值对象
         */
        fun getInt(key: String):Int?{
            val value = getAny(key)
            return when(value){
                is Byte -> value.toInt()
                is Int -> value
                is Long -> value.toInt()
                is Float -> value.toInt()
                is Double -> value.toInt()
                is Char -> value.toInt()
                is String -> value.toIntOrNull()
                else -> null
            }
        }
    
        /**
         * 存入普通缓存对象
         *
         * @param key 键
         * @param value 值对象
         * @return Boolean 是否成功
         */
        fun setAny(key:String,value:Any) {
            redisTemplate.opsForValue().set(key,value)
        }
    
        /**
         * 删除普通缓存对象
         *
         * @param key 键
         * @return Boolean 是否成功
         */
        fun delete(key:String):Boolean {
            return redisTemplate.delete(key)
        }
    
        /**
         * 获取普通缓存对象
         *
         * @param key 键
         * @return any 值对象
         */
        private fun getAny(key: String):Any? {
            return redisTemplate.opsForValue().get(key)
        }
    }
    ```

# redisson集成

- 引入依赖
    ```kotlin
    <redisson.version>3.10.0</redisson.version>
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson</artifactId>
        <version>${redisson.version}</version>
    </dependency>
    ```
- 配置bean
    ```kotlin
    @Autowired
    lateinit var redisProperties: RedisProperties
    @Bean
    fun redisClient(): RedissonClient {
        val serverConfig = Config().apply {
            this.useSingleServer()
                    .setAddress("redis://${redisProperties.host}:${redisProperties.port}").timeout = (redisProperties.timeout.seconds * 1000).toInt()
        }
        return Redisson.create(serverConfig)
    }
    ```
- 测试类
    ```kotlin
    class RedisLockTest : BaseTest(){
        private val log = LoggerFactory.getLogger(this.javaClass)
        private val testRedisKey = this.javaClass.name
        private val lockKey = "testLockKey"
        @Autowired
        lateinit var redisUtils: RedisUtils
        @Autowired
        lateinit var redisClient: RedissonClient
        @Before
        fun setUp() {
            log.info("----------before-----------")
        }
    
        @After
        fun tearDown() {
            log.info("----------after-----------")
        }
    
        @Test
        fun setInt(){
            redisUtils.setAny(testRedisKey,100)
            println(redisUtils.getInt(testRedisKey))
        }
    
        @Test
        fun testLock(){
            val lock:RLock = redisClient.getLock(lockKey)
            println("---尝试上锁---")
            lock.tryLock(20, TimeUnit.SECONDS)
            println("---进入循环---")
            for (i in 0..10){
                val stock = redisUtils.getInt(testRedisKey)
                Thread.sleep(1000)
                if(stock!! > 0){
                    redisUtils.setAny(testRedisKey, value = stock-1)
                    println("stock: $stock-1")
                }
            }
            lock.unlock()
        }
    }
    ```
- 先执行一次设值,再同时运行两次测试类,我们希望达到的效果是,当第一个执行完循环时,第二个才开始进入循环,可以看到,最终结果确如我们所愿
    ![redis-lock.gif](https://i.loli.net/2019/09/28/2ALRvEJkh67pPsn.gif)

# 注解获取缓存对象  

- 不论我们使用shiro,还是security或是其他的框架做项目的权限控制,我们可能需要将用户存入缓存中,这里我使用注解获取缓存中的用户对象
- 新建一个注解类
    ```kotlin
    @Target(AnnotationTarget.VALUE_PARAMETER)
    @Retention(AnnotationRetention.RUNTIME)
    annotation class CurrentUser
    ```
- 新建一个方法解析器
    ```kotlin
    @Component
    class CurrentUserMethodArgumentResolver:HandlerMethodArgumentResolver{
        @Autowired
        lateinit var redisUtils: RedisUtils
        
        override fun supportsParameter(parameter: MethodParameter): Boolean {
            return parameter.parameterType.isAssignableFrom(User::class.java)
                    && parameter.hasParameterAnnotation(CurrentUser::class.java)
        }
    
        override fun resolveArgument(parameter: MethodParameter, mavContainer: ModelAndViewContainer?, webRequest: NativeWebRequest, binderFactory: WebDataBinderFactory?): Any? {
            return redisUtils.getT<User>("currentUser")
        }
    }
    ```
- 修改spring配置
    ```kotlin
    @Configuration
    class SpringConfig : WebMvcConfigurer {
        @Autowired
        lateinit var currentUserResolver: CurrentUserMethodArgumentResolver
        override fun addViewControllers(registry: ViewControllerRegistry) {
            registry.addViewController("/").setViewName("forward:/hello")
            registry.setOrder(Ordered.HIGHEST_PRECEDENCE)
        }
    
        override fun addArgumentResolvers(resolvers: MutableList<HandlerMethodArgumentResolver>) {
            super.addArgumentResolvers(resolvers)
            resolvers.add(currentUserResolver)
        }
    }
    ```
- 修改控制层
    ```kotlin
    @RestController
    class HelloController {
        @Autowired
        lateinit var redisUtils: RedisUtils
    
        @GetMapping("/hello")
        fun hello(@CurrentUser user:User): String {
            return user.toString()
        }
    
        @GetMapping("/testLogin")
        fun testLogin(user:User): String {
            redisUtils.setAny("currentUser",user)
            return "success"
        }
    }
    ```
- 依次运行  
  http://localhost:9001/testLogin?username=me&sex=man&phone=1000
  http://localhost:9001/hello  
  可以看到我们成功取到了这个对象
    ![image.png](https://i.loli.net/2019/09/28/UdMj8DkoYxl4fcr.png) 

# [代码链接](https://github.com/gonghs/kotlin-web/tree/master/kotlin-springboot-redis)


[回到顶部](#top)
