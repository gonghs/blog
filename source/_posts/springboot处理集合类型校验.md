---
title: springboot处理集合类型校验
date: 2019-07-25 14:54:32
categories: 
- kotlin web
tags:
- springboot
- gradle
- kotlin
- web
description: 使用springboot的默认校验框架时,我们发现spring对集合类型和集合类型对象是不作为的,此篇即针对这个问题做出的处理
---
# 版本概要

---
> springboot版本2.1.6.RELEASE 
> kotlin版本1.2.71   
> gradle版本5.2.1  
> idea版本2019.1.2 ultimate edition
---

# 新建项目

- 点击file -> new project ->选择spring initializrd点击下一步
- 选择语言,选择项目管理工具
    ![image.png](https://i.loli.net/2019/09/28/dufNkUJ6axel3O9.png)
- 此篇我们讨论只涉及校验,只引入springWebStarter即可
- 选择gradle路径(或者使用默认的),这里我选择本地路径
    ![image.png](https://i.loli.net/2019/09/28/1dLSl6iOIhGC74W.png)
- 输入项目路径,点击finish,查看项目结构,我们可以看到生成的gradle依赖文件变成了kts后缀的文件,和之前比起来,配置会略有不同
- 增加国内镜像地址  
    追加根节点
    ```kotlin
    repositories {
        maven (url = "http://maven.aliyun.com/nexus/content/groups/public/")
        jcenter()
    } 
    ```
- 重新导入等待编译完成

# 重现无法校验场景

- 增加测试控制器和测试类,这里我直接加在了启动类上
```kotlin
@SpringBootApplication
@RestController
class DemoApplication {
    @PostMapping("/demo")
    fun demo(@RequestBody @Validated user: User): User {
        return user
    }

    @PostMapping("/demoList")
    fun demoList(@RequestBody @Validated user: List<User>): List<User> {
        return user
    }
}

class User {
    @NotBlank(message = "用户名不能为空")
    val username: String? = null
    val password: String? = null
}

fun main(args: Array<String>) {
    runApplication<DemoApplication>(*args)
}
```
- 增加测试类
```kotlin
@RunWith(SpringRunner::class)
@WebMvcTest(DemoApplication::class)
class WebMvcTest {
    @Autowired
    lateinit var mockMvc: MockMvc

    @Test
    fun testDemo() {
        val example = "{\"username\":\"\", \"password\":\"111\"}"
        mockMvc.perform(MockMvcRequestBuilders.post("/demo")
                .contentType(MediaType.APPLICATION_JSON_VALUE).content(example))
                .andExpect(status().isOk).andDo { println(it.response.contentAsString) }.andReturn()
    }

    @Test
    fun testDemoList() {
        val example = "[{\"username\":\"\", \"password\":\"111\"}]"
        mockMvc.perform(MockMvcRequestBuilders.post("/demoList")
                .contentType(MediaType.APPLICATION_JSON_VALUE).content(example))
                .andExpect(status().isOk).andDo { println(it.response.contentAsString) }.andReturn()
    }
}
```
- 我们先执行能够正常执行校验的测试方法testDemo,执行后发现控制台报错,说明参数被校验了
    ![image.png](https://i.loli.net/2019/09/28/sUyxuClac4gPQIV.png)
- 但执行List时参数却未被校验
    ![image.png](https://i.loli.net/2019/09/28/KDjE8gNXFTesdLv.png)
    

# 处理方案

## 方案1:
- 新建类包装List,在list上加上@Valid(javax包中)(由于@Validated不支持放在字段上,所以无法使用)注解  
    - 控制器方法
    ```kotlin
    @PostMapping("/demoValidList")
    fun demoValidList(@RequestBody @Validated user: ValidList<User>): List<User>? {
        return user.list
    }
    ```
    - 包装类
    ```kotlin
    class ValidList<T> {
        @Valid
        val list: List<T>? = null
    }
    ```
    - 测试方法
    ```kotlin
    @Test
    fun testDemoValidList() {
        val example = "{\"list\":[{\"username\":\"\", \"password\":\"111\"}]}"
        mockMvc.perform(MockMvcRequestBuilders.post("/demoValidList")
                .contentType(MediaType.APPLICATION_JSON_VALUE).content(example))
                .andExpect(status().isOk).andDo { println(it.response.contentAsString) }.andReturn()
    }
    ```
    **可以看到此方案需要我们将传输的参数变为对象,多了一层无用的嵌套,并且由于@Valid注解的缺陷,无法使用分组**
## 方案2:
- 我们可以采用实现list接口并转接方法的方式,去掉这层无用的嵌套
    ```kotlin
    class ValidList1<T> : MutableList<T> {
        override fun iterator(): MutableIterator<T> {
            return list.iterator()
        }
    
        override fun listIterator(): MutableListIterator<T> {
            return list.listIterator()
        }
    
        override fun listIterator(index: Int): MutableListIterator<T> {
            return list.listIterator(index)
        }
    
        override fun subList(fromIndex: Int, toIndex:Int): MutableList<T> {
            return list.subList(fromIndex, toIndex)
        }
    
        override fun add(element: T): Boolean {
            return list.add(element)
        }
    
        override fun add(index: Int, element: T) {
            return list.add(index, element)
        }
    
        override fun addAll(index: Int, elements: Collection<T>): Boolean {
            return list.addAll(index, elements)
        }
    
        override fun addAll(elements: Collection<T>): Boolean {
            return list.addAll(elements)
        }
    
        override fun clear() {
            list.clear()
        }
    
        override fun remove(element: T): Boolean {
            return list.remove(element)
        }
    
        override fun removeAll(elements: Collection<T>): Boolean {
            return list.removeAll(elements)
        }
    
        override fun removeAt(index: Int): T {
            return list.removeAt(index)
        }
    
        override fun retainAll(elements: Collection<T>): Boolean {
            return list.retainAll(elements)
        }
    
        override fun set(index: Int, element: T): T {
            return list.set(index, element)
        }
    
        override val size: Int
            get() = list.size
    
        override fun contains(element: T): Boolean {
            return list.contains(element)
        }
    
        override fun containsAll(elements: Collection<T>): Boolean {
            return list.containsAll(elements)
        }
    
        override fun get(index: Int): T {
            return list[index]
        }
    
        override fun indexOf(element: T): Int {
            return list.indexOf(element)
        }
    
        override fun isEmpty(): Boolean {
            return list.isEmpty()
        }
    
        override fun lastIndexOf(element: T): Int {
            return list.lastIndexOf(element)
        }
    
        @Valid
        val list: MutableList<T> = mutableListOf()
    }
    ```
    - 测试控制器
    ```kotlin
    @PostMapping("/demoValidList1")
    fun demoValidList(@RequestBody @Validated user: ValidList1<User>): List<User>? {
        return user.list
    }
    ```
    - 测试方法
    ```kotlin
    @Test
    fun testDemoValidList1() {
        val example = "[{\"username\":\"\", \"password\":\"111\"}]"
        mockMvc.perform(MockMvcRequestBuilders.post("/demoValidList1")
                .contentType(MediaType.APPLICATION_JSON_VALUE).content(example))
                .andExpect(status().isOk).andDo { println(it.response.contentAsString) }.andReturn()
    }
    ```
- 使用kotlin委托我们可以节省部分代码(注意直接在list上增加@Valid是无效的)
    - 委托器
    ```kotlin
    class ValidList2<T>(val list: MutableList<T>) : MutableList<T> by list {
        @Valid
        var mlist: MutableList<T> = list
        constructor() : this(mutableListOf())
    }
    ```
    - 控制器
    ```kotlin
    @PostMapping("/demoValidList2")
    fun demoValidList2(@RequestBody @Validated user: ValidList2<User>): List<User>? {
        return user.list
    }
    ```
    - 测试类
    ```kotlin
    @Test
    fun testDemoValidList2() {
        val example = "[{\"username\":\"\", \"password\":\"111\"}]"
        mockMvc.perform(MockMvcRequestBuilders.post("/demoValidList2")
                .contentType(MediaType.APPLICATION_JSON_VALUE).content(example))
                .andExpect(status().isOk).andDo { println(it.response.contentAsString) }.andReturn()
    }
    ```
- **如果使用的是java我们也可以利用lombok替我们节省部分代码**
    ```java
    @Data
    public class ValidList<T> implements List<T>{
        @Delegate
        List<T> list = new ArrayList<>();
    }
    ```

**由于未做异常拦截,以上方案 正常校验,验证器将抛出异常**
![image.png](https://i.loli.net/2019/09/28/5ScyJOqR3BCmnVD.png)
**方案1和方案2本质上是一致的,缺点在于对于控制器代码的侵入性较大(意味着所有需要校验list的控制器方法都需要修改类为新的包装类)**
## 方案3
- 我们可以自定义验证器并配置@ControllerAdvice统一为集合增加验证器
    - 自定义验证器
    ```kotlin
    class CollectionValidator(private val validatorFactory: LocalValidatorFactoryBean) : Validator {
    
        override fun supports(clazz: Class<*>): Boolean {
            return Collection::class.java.isAssignableFrom(clazz)
        }
    
        /**
         * 校验集合 遇到失败即退出
         *
         * @param target 受校验对象
         * @param errors 错误结果
         */
        override fun validate(target: Any, errors: Errors) {
            val collection = target as Collection<*>
            for (`object` in collection) {
                `object`?.let { ValidationUtils.invokeValidator(validatorFactory, `object`, errors) }
                // 存在错误即退出校验
                if (errors.hasErrors()) {
                    break
                }
            }
        }
    }
    ```
    - 控制器拦截
    ```kotlin
    @ControllerAdvice
    class CollectionValidatorAdvice(private val collectionValidator: CollectionValidator) {
        /**
         * 在initBinder阶段修改集合类型的校验器
         */
        @InitBinder
        fun initBinder(binder: WebDataBinder) {
            // 这里必须判断 否则会影响非集合类型校验
            if (binder.target !is Collection<*>) {
                return
            }
            binder.addValidators(collectionValidator)
        }
    }
    ```
    - 配置类
    ```kotlin
    @Configuration
    class CollectionValidatorConfig {
        @Autowired
        lateinit var localValidatorFactoryBean: LocalValidatorFactoryBean
    
        @Bean
        fun collectionValidator(): CollectionValidator {
            return CollectionValidator(localValidatorFactoryBean)
        }
    }
    ```
    - 修改测试类,导入配置文件
    ```kotlin
    @RunWith(SpringRunner::class)
    @WebMvcTest(DemoApplication::class)
    @Import(CollectionValidatorConfig::class)
    class WebMvcTest
    ```
    - 再次运行testDemoList此时此方法将受校验  
**同样由于未做异常拦截,以上方案,验证器将抛出异常**
    ![image.png](https://i.loli.net/2019/09/28/gI4nPkjoG5Ja9wW.png)
**方案3允许使用分组,对参数不需要做变更,而其缺陷在于如果想要知道是集合的哪条数据出现问题相对而言不太容易**


# [代码链接](https://github.com/gonghs/kotlin-web/tree/master/kotlin-springboot-collection-validation)