---
title: Shiro权限注解与Aop冲突的前世今生
date: 2020-05-23 21:46:32
categories: 
- tool
tags:
- shiro
- springboot
- aop
- java
description: Shiro权限注解与Aop冲突的前世今生
---
在做springboot和shiro集成时，在度娘的多篇博文上相关**DefaultAdvisorAutoProxyCreator**有下图的描述，但实际测试时将usePrefix和proxyTargetClass二者任意一值设为true都可以解决无法映射请求的问题，此文即是基于此的拓展，有兴趣的童鞋可以在[测试项目](https://gitee.com/gonghs/jianshu/tree/master/shiro-permissions-aop)中进行测试。

![image-20200516234105315](https://gitee.com/gonghs/image/raw/master/img/20200516234106.png)

## 猜测

要搞明白为这两个配置能够解决这个问题，可以先从**DefaultAdvisorAutoProxyCreator**开始看起，从类名上我们很容易得知这是spring自动代理创建器之一。该类可以对通知器进行过滤。

源码注释中表明，通过设置usePrefix值为true，类将仅在通知器bean名称为advisorBeanNamePrefix+.时生效。

![image-20200517001821588](https://gitee.com/gonghs/image/raw/master/img/20200517001822.png)

而proxyTargetClass则配置了代理时是否时基于类代理。

由前面两点可以猜测

- usePrefix能够解决这个问题，是由于具体问题的发生场景和通知器bean名称有关，由于usePrefix使某个通知器被过滤使问题被解决。
- proxyTargetClass能够解决这个问题，是因为此配置修改了代理机制，使代理的冲突消失了。

猜测不一定是正确的，并且由于二者都可以解决这个问题，还需要探求二者之间的关联。

## 由请求开始

当aop与权限注解共存时，不对**DefaultAdvisorAutoProxyCreator**进行任何配置并跟踪请求，部分执行链如下：

```
// {}表示同上
.
├── DispatcherServlet.doService
│   ├── DispatcherServlet.doDispatch
│   	├── DispatcherServlet.getHandler
│   		├── AbstractHandlerMapping.getHandler
│   			├── RequestMappingInfoHandlerMapping.getHandlerInternal
│   				└── AbstractHandlerMethodMapping.lookupHandlerMethod
```

之所以跟踪这个请求，是由于在*doDispatch*方法中的**HandlerExecutionChain**类型变量是spring由容器中取得的实际处理请求的对象。

![image-20200517004212046](https://gitee.com/gonghs/image/raw/master/img/20200517004213.png)

以下是测试控制器的部分代码

```java
@RestController
public class IndexController {
    @GetMapping
    @RequiresPermissions("admin")
    public String empty() {
        return "hello";
    }
}
```

在**AbstractHandlerMethodMapping**.*lookupHandlerMethod*中，可以看到此处从一个映射注册表中的两个hashMap对象中为请求获取最合适的处理方法，而在映射注册表中根本不存在我们的自定义控制器，由此得知问题的根源在于映射注册表中控制器的缺失。

![image-20200517010753991](https://gitee.com/gonghs/image/raw/master/img/20200517010755.png)

![异常与错误图对比](https://gitee.com/gonghs/image/raw/master/img/20200517012547.png)

## 映射注册表初始化

在**AbstractHandlerMethodMapping**类中对类变量mappingRegistry进行查找，可以找到*registerHandlerMethod*方法，用于向注册表中注册处理方法。

![image-20200517123702327](https://gitee.com/gonghs/image/raw/master/img/20200517123703.png)

而该方法的调用在*detectHandlerMethods*方法中，通过对该方法断点调试(启动时)并向上查找堆栈信息，我们可以得到这样一条调用链：

```
.
├── AbstractAutowireCapableBeanFactory.invokeInitMethods
│   ├── RequestMappingHandlerMapping.afterPropertiesSet
│		├── AbstractHandlerMethodMapping.afterPropertiesSet
│   		├── AbstractHandlerMethodMapping.initHandlerMethods
│   			├── AbstractHandlerMethodMapping.processCandidateBean
│   				└── AbstractHandlerMethodMapping.detectHandlerMethods
```

**RequestMappingHandlerMapping**是springboot WebMvcAutoConfiguration中预定义的，自动初始化的bean，用于处理被*RequestMapping*注解标记的方法。

![image-20200517151500795](https://gitee.com/gonghs/image/raw/master/img/20200517151501.png)

而由于该类父类实现了InitializingBean接口，bean初始化完毕时会调用*afterPropertiesSet*方法，由调用链可以看出，映射注册表的初始化即是由这个逻辑促成的。跟踪正常映射的请求可以验证这一点：

![image-20200517152329552](https://gitee.com/gonghs/image/raw/master/img/20200517152330.png)

**AbstractHandlerMethodMapping**.*processCandidateBean*方法中可以找到类能否被注册进映射注册表的逻辑：

![image-20200517153006969](https://gitee.com/gonghs/image/raw/master/img/20200517153008.png)

spring从应用上下文中获取根据名称获取对应bean的类对象并根据类对象中是否含有Controller和RequestMapping注解来判断是否注册。

![image-20200517153216257](https://gitee.com/gonghs/image/raw/master/img/20200517153217.png)

通过调试可以看到加入权限异常映射的bean，在容器中取到的类对象是代理对象，下面是几种情况取到的类对象：

![获取bean结果比对](https://gitee.com/gonghs/image/raw/master/img/20200517184949.png)

可以看到前者取到的对象是由jdk动态代理的，而后两者取到的对象都是由cglib代理的。

除此之外当方法只被aop注解标记时也是获得的类对象也是由cglib代理的，当类路径中不存在Advice类时bean是原始类对象，此时权限注解不生效。

![image-20200517233933042](https://gitee.com/gonghs/image/raw/master/img/20200517233935.png)

由以上情况又引申出了三个问题：

- 为什么在jdk动态代理的情况下认为不存在注解
- 使用哪种代理方式是如何判断的
- 为什么设置usePrefix时需要使用cglib代理

## 如何判断注解的存在

这里涉及到调用链：

```
.
├── AnnotatedElementUtils.hasAnnotation
│   ├── TypeMappedAnnotations.isPresent
│		├── TypeMappedAnnotations.scan
│   		├── AnnotationsScanner.scan
│   			├── AnnotationsScanner.scan
│   				├── AnnotationsScanner.process
│   					├── AnnotationsScanner.processClass
| 							└── AnnotationsScanner.processClassHierarchy
```

在**AnnotationsScanner**.*processClassHierarchy*中可以看到spring首先判断类对象上是否存在注解，如果不存在则递归查找该类对象父类，所有的接口和内部类。

```java
/**
 * 按层次执行类查找
 *
 * @param context 需要寻找的注解
 * @param aggregateIndex
 * @param source 类对象
 * @param processor 注解处理器
 * @param classFilter 类过滤(判断是否要忽略某些类查找)
 * @param includeInterfaces 是否包含接口
 * @param includeEnclosing 是否包含匿名内部类
 * @param <R> 此处为Boolean
 * @return 匹配成功返回true 否则返回false或者null
 */
private static <C, R> R processClassHierarchy(C context, int[] aggregateIndex, Class<?> source, AnnotationsProcessor<C, R> processor, @Nullable BiPredicate<C, Class<?>> classFilter, boolean includeInterfaces, boolean includeEnclosing) {
    try {
        R result = processor.doWithAggregate(context, aggregateIndex[0]);
        if (result != null) {
            return result;
        }
        // 是否仅有普通的java注解
        if (hasPlainJavaAnnotationsOnly(source)) {
            return null;
        }
        // 判断类本身是否存在注解
        Annotation[] annotations = getDeclaredAnnotations(context, source, classFilter, false);
        result = processor.doWithAnnotations(context, aggregateIndex[0], source, annotations);
        if (result != null) {
            return result;
        }
        aggregateIndex[0]++;
        // 如果存在实现接口则遍历接口执行查找 判断接口上是否存在该注解
        if (includeInterfaces) {
            for (Class<?> interfaceType : source.getInterfaces()) {
                R interfacesResult = processClassHierarchy(context, aggregateIndex,
                        interfaceType, processor, classFilter, true, includeEnclosing);
                if (interfacesResult != null) {
                    return interfacesResult;
                }
            }
        }
        // 向上查找所有父类
        Class<?> superclass = source.getSuperclass();
        if (superclass != Object.class && superclass != null) {
            R superclassResult = processClassHierarchy(context, aggregateIndex,
                    superclass, processor, classFilter, includeInterfaces, includeEnclosing);
            if (superclassResult != null) {
                return superclassResult;
            }
        }
        // 查找内部类
        if (includeEnclosing) {
            // Since merely attempting to load the enclosing class may result in
            // automatic loading of sibling nested classes that in turn results
            // in an exception such as NoClassDefFoundError, we wrap the following
            // in its own dedicated try-catch block in order not to preemptively
            // halt the annotation scanning process.
            try {
                Class<?> enclosingClass = source.getEnclosingClass();
                if (enclosingClass != null) {
                    R enclosingResult = processClassHierarchy(context, aggregateIndex, enclosingClass, processor, classFilter, includeInterfaces, true);
                    if (enclosingResult != null) {
                        return enclosingResult;
                    }
                }
            }
            catch (Throwable ex) {
                AnnotationUtils.handleIntrospectionFailure(source, ex);
            }
        }
    }
    catch (Throwable ex) {
        AnnotationUtils.handleIntrospectionFailure(source, ex);
    }
    return null;
}
```

在使用cglib代理的情况下，可以在父类中查找到原始类对象，并获取到注解。

![image-20200518170620227](https://gitee.com/gonghs/image/raw/master/img/20200518170628.png)

而在jdk动态代理情况下获得的对象中无法从类的父子关系，接口实现，内部类中得到原始类对象，也就无法获得原始注解。

以上可以得知代理方式的变更会导致类无法注册入映射注册表，导致bean无法映射请求。

在上面的探究中对比两种代理类的实现接口还发现，获取到jdk动态代理类对象时似乎被二次代理了，其中的原因我们放到后面再探究。

![实现接口比对](https://gitee.com/gonghs/image/raw/master/img/20200518174546.png)

## 代理方式的选择

### bean的后置处理

要了解这个问题，需要对bean的创建过程进行跟踪：

```
.
├── AbstractAutowireCapableBeanFactory.doCreateBean
│   ├── {}.initializeBean
│		├── {}.applyBeanPostProcessorsAfterInitialization
│   		├── AbstractAutoProxyCreator.postProcessAfterInitialization
│   			├── AbstractAutoProxyCreator.wrapIfNecessary
│   				├── {}.createProxy
│   					├── AnnotationsScanner.processClass
| 							└── AnnotationsScanner.processClassHierarchy
```

在bean初始化完成后将在**AbstractAutowireCapableBeanFactory**.*applyBeanPostProcessorsAfterInitialization*方法进行后置处理，spring从容器中获取所有的后置处理器即所有实现BeanPostProcessor接口的bean对bean进行后置处理，在这里可以找到**DefaultAdvisorAutoProxyCreator**

![image-20200522230758590](https://gitee.com/gonghs/image/raw/master/img/20200522230800.png)

在跟踪**DefaultAdvisorAutoProxyCreator**之前，先跟踪**AnnotationAwareAspectJAutoProxyCreator**，以了解一个正常执行的代理流程。

###  **AnnotationAwareAspectJAutoProxyCreator**的加载

**AnnotationAwareAspectJAutoProxyCreator**是spring aop工作的重要构建器，上文我们猜测异常bean可能被二次代理了，第一次代理便是来自spring aop的代理。

在AopAutoConfiguration可以看到在默认配置（且存在Advice类）下使用注解开启了Aspect J自动代理

![image-20200521224957601](https://gitee.com/gonghs/image/raw/master/img/20200521225006.png)

在此注解存在的情况下**AspectJAutoProxyRegistrar**将工作，这里将进行**AnnotationAwareAspectJAutoProxyCreator**的注册工作：

```
.
├── AspectJAutoProxyRegistrar.registerBeanDefinitions
│   ├── AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary
│		└── {}.registerOrEscalateApcAsRequired
```

![image-20200521225632059](https://gitee.com/gonghs/image/raw/master/img/20200521225633.png)

可以看到此处为bean定义在注册表中指定了名称

![image-20200521230100754](https://gitee.com/gonghs/image/raw/master/img/20200521230101.png)

并且在另一处自动配置中将proxyTargetClass值设为了true：

![image-20200521230329847](https://gitee.com/gonghs/image/raw/master/img/20200521230331.png)

![image-20200521230255707](https://gitee.com/gonghs/image/raw/master/img/20200521230256.png)

**AbstractAutoProxyCreator**.*wrapIfNecessary*可以跟踪到**AnnotationAwareAspectJAutoProxyCreator**的创建代理的工作流程：

```java
/**
 * 按需包装(代理)bean
 *
 * @param bean bean
 * @param beanName bean名称
 * @param cacheKey 缓存key 用以标记该bean是否需要被通知
 * @return 包装(代理)后的bean
 */
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    // this.advisedBeans在postProcessBeforeInstantiation即前置操作中初始化过一次 
	// 这里是跳过已经预先知道不需要被通知的bean
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    // 在bean的前置处理时其实已经做过类似的判断
	// isInfrastructureClass是判断是否为aop的基础设施类 诸如Advice，Pointcut，Advisor等
	// shouldSkip是由bean名称判断是否为原始示例 如果为是（名称为bean类名+.ORIGINAL）则跳过
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
    // Create proxy if we have advice.
    // 获取所有的通知bean 存在则进行代理操作
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 创建代理
        Object proxy = createProxy(
                bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }
    // 不存在通知类 则标记为不需要通知
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```

创建代理时用到了创建器本身的proxyTargetClass属性用以判断是否基于类代理，这里前面提到，在自动配置时就已经将该bean proxyTargetClass属性默认设置为true，因此由该类创建的代理对象默认都是基于cglib代理的：

![image-20200523190042803](https://gitee.com/gonghs/image/raw/master/img/20200523190044.png)

如果在配置中配置默认代理方式不基于类（spring.aop.proxy-target-class =false），但类不存在合适的代理接口，spring仍可能选择cglib代理：

```java
/**
 * 评估接口是否适合代理
 *
 * @param beanClass bean类对象
 * @param proxyFactory 代理工厂
 */
protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
    // 获取所有实现接口
    Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
    boolean hasReasonableProxyInterface = false;
    for (Class<?> ifc : targetInterfaces) {
        // isConfigurationCallbackInterface判断接口是否配置回调接口 例如：InitializingBean，Closeable等
        // isInternalLanguageInterface判断接口是否内部语言接口 例如：GroovyObject，cglib代理工厂接口等
        if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
                ifc.getMethods().length > 0) {
            hasReasonableProxyInterface = true;
            break;
        }
    }
    if (hasReasonableProxyInterface) {
        // Must allow for introductions; can't just set interfaces to the target's interfaces only.
        for (Class<?> ifc : targetInterfaces) {
            proxyFactory.addInterface(ifc);
        }
    } else {
        // 没有合适的实现接口则也设置为true
        proxyFactory.setProxyTargetClass(true);
    }
}
```

这一点也解释了为何不引入spring aop的情况下权限注解可用，有兴趣的童鞋可以尝试下载代码进行设置跟踪。这里只讲结论：

- 不引入spring aop只有**DefaultAdvisorAutoProxyCreator**生效的情况下，bean实例仍是cglib代理的。
- 引入了spring aop时**DefaultAdvisorAutoProxyCreator**获得的类是已代理的，有了在此判断外的实现接口，被判断为适合接口代理，因此bean实例最终使用了jdk动态代理。

### DefaultAdvisorAutoProxyCreator的工作

**DefaultAdvisorAutoProxyCreator**和**AnnotationAwareAspectJAutoProxyCreator**的工作过程基本类似。从以上工作过程中其实可以大概了解（实际也是如此），**DefaultAdvisorAutoProxyCreator**异常工作时，对bean再次进行了代理行为，由于该类的proxyTargetClass的默认值为false，且获取的代理对象被判断为适合接口代理，因此采用了jdk动态代理，从bean实例上来看就是被二次代理了。

![image-20200523141226334](https://gitee.com/gonghs/image/raw/master/img/20200523141227.png)

另外需要提及的是当仅proxyTargetClass为true时虽然进行了两次代理，代理类上获取的直接父类还是原始类（而不是父类的父类才是原始类）。详见：

```
.
├── AbstractAutoProxyCreator.wrapIfNecessary
│   ├── {}.createProxy
│		├── ProxyFactory.getProxy
|			└──	CglibAopProxy.getProxy
```

![image-20200523170810525](https://gitee.com/gonghs/image/raw/master/img/20200523170811.png)

## 为什么设置usePrefix时需要使用cglib代理

这里的问题其实应该是，为什么设置usePrefix为true时不进行代理，事实上usePrefix为true时，**DefaultAdvisorAutoProxyCreator**是不工作的。

这里需要从*getAdvicesAndAdvisorsForBean*方法中进行跟踪，因为当设置usePrefix为true时，该方法取到的通知器是空的。

```
.
├── AbstractAutoProxyCreator.wrapIfNecessary
│   ├── {}.getAdvicesAndAdvisorsForBean
│		├── AbstractAdvisorAutoProxyCreator.findEligibleAdvisors
│   		├── {}.findCandidateAdvisors
│   			└── BeanFactoryAdvisorRetrievalHelper.findAdvisorBeans
```

在**BeanFactoryAdvisorRetrievalHelper**.*findAdvisorBeans*中，从容器中取得的通知器是否要最终返回用于代理，存在*isEligibleBean*判断。

![image-20200523143345185](https://gitee.com/gonghs/image/raw/master/img/20200523143346.png)

前面都是和构建器父层抽象类打交道，到这里终于和**DefaultAdvisorAutoProxyCreator**本身打上了交道。

这个判断最终使用了**DefaultAdvisorAutoProxyCreator**.*isEligibleAdvisorBean*的返回结果。

![image-20200523144138787](https://gitee.com/gonghs/image/raw/master/img/20200523144139.png)

在**DefaultAdvisorAutoProxyCreator**.*isEligibleAdvisorBean*中得知，当usePrefix为false时该方法总是返回true，而当usePrefix为true时需要判断bean名称是否以类属性advisorBeanNamePrefix+.开头判断如何返回。

![image-20200523144605015](https://gitee.com/gonghs/image/raw/master/img/20200523144605.png)

当只定义了usePrefix为true而未定义advisorBeanNamePrefix时，大部分情况下bean名称是无法匹配的，因此通知器无法返回，也就不会执行具体的代理行为。而定义usePrefix为false时代理行为总是执行的。

## 总结

再次提出之前的两条猜测：

- usePrefix能够解决这个问题，是由于具体问题的发生场景和通知器bean名称有关，由于usePrefix使某个通知器被过滤使问题被解决。
- proxyTargetClass能够解决这个问题，是因为此配置修改了代理机制，使代理的冲突消失了。

现在看，两个猜测都是部分正确的，并且以上的探究对其进行了拓展，在探究的过程中我们得知，冲突的本质在于是否使用jdk动态代理，只要bean没有被jdk动态代理，这个映射问题就不会存在。下面是各种场景下的运行情况：

| 开启AOP | usePrefix | proxyTargetClass |  代理情况   | 运行情况 |
| :-----: | :-------: | :--------------: | :---------: | :------: |
|   是    |   false   |      false       |  jdk+cglib  |   404    |
|   是    |   false   |       true       | cglib+cglib | 权限生效 |
|   是    |   true    |      false       |    cglib    | 权限生效 |
|   是    |   true    |       true       |    cglib    | 权限生效 |
|   否    |   false   |      false       |    cglib    | 权限生效 |
|   否    |   false   |       true       |    cglib    | 权限生效 |
|   否    |   true    |      false       |     无      | 权限无效 |
|   否    |   true    |       true       |     无      | 权限无效 |

最后总结usePrefix，proxyTargetClass解决的问题：

- userPrefix属性设为true，阻止了**DefaultAdvisorAutoProxyCreator**代理创建行为。

- proxyTargetClass为true，使类对象在被二次代理时，仍旧能够找到原始类对象，并且被成功放入映射注册表。