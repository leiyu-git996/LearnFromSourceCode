# 说一说springboot启动流程？

## springboot启动类上的注解

springboot启动类上有一个核心注解@SpringBootApplication，它是一个组合注解，它实际上结合了以下三个注解的功能：

- **@Configuration**: 表示当前类是一个 Spring 配置类，能替代 XML 配置文件。

- **@EnableAutoConfiguration**: 告诉 Spring Boot 根据项目的依赖来自动配置 Spring 应用程序，比如获取所需的 Bean。

  - **@Import({AutoConfigurationImportSelector.class})**：这个注解是 @EnableAutoConfiguration 的一部分。AutoConfigurationImportSelector是一个条件导入选择器，它会根据 META-INF/spring.factories 文件中的配置来查找和引入合适的自动配置类。

    在加载时，它会根据当前类路径和其他上下文信息，选择合适的自动配置类并将它们导入到 Spring 上下文中。这一过程是通过 `@Conditional` 注解来判断是否满足某些条件（例如某个库的存在等）。

- **@ComponentScan**: 启用组件扫描，默认扫描与该类相同的包及其子包中的组件（如 @Component、@Service、@Repository、@Controller 注解的类）。

**总结**：springboot启动类上有一个核心注解@SpringBootApplication，其中包括@EnableAutoConfiguration和@Import({AutoConfigurationImportSelector.class})，其中的@Import注解会扫描META-INF/spring.factories 文件中的配置来查找和引入合适的自动配置类。

## springboot如何启动的（核心代码）

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

上面是一个springboot启动类的入口，这里主要看SpringApplication.run(Application.class, args);方法。

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class[]{primarySource}, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return (new SpringApplication(primarySources)).run(args);
}
```

这里就主要分为new SpringApplication的初始化阶段和SpringApplication.run的启动过程了

### new SpringApplication()

SpringApplication的构造参数如下。注释了几个重要的初始化方法。

```java
public SpringApplication(Class<?>... primarySources) {
    this((ResourceLoader)null, primarySources);
}

public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = Collections.emptySet();
    this.isCustomEnvironment = false;
    this.lazyInitialization = false;
    this.applicationContextFactory = ApplicationContextFactory.DEFAULT;
    this.applicationStartup = ApplicationStartup.DEFAULT;
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
  	// 添加源
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
  	// 设置web环境
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new 				 ArrayList(this.getSpringFactoriesInstances(BootstrapRegistryInitializer.class)); 		
		// 加载器初始化：设置ApplicationContext的初始化器，从"spring.factories"文件中加载ApplicationContenxtInitializer实现
	this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
  	// 设置监听器，从"spring.factories"文件中加载ApplicationListener实现
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
  	// 确定主应用类
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

1. 添加源：将提供的源（通常是配置类）添加到应用的源列表中。
2. 设置Web环境：判断应用是否应该运行在web环境中，会影响后续的Web相关配置。
3. 加载初始化期：从"spring.factories"文件中加载ApplicationContenxtInitializer实现，并将他们设置到SpringApplication实例中，以便在应用上下文的初始化阶段执行他们。
4. 设置监听器：加载和设置ApplicationListener实例，以便应用能够响应不同的事件。
5. 确定主应用类：确定主应用类，这个主应用程序通常是包含public static void main(String[] args)方法的类，是启动整个SpringBoot应用的入口。

**这里面的第三步，是SpringBoot自动配置的核心**，因为他会从"spring.factories"文件中加载并实例化指定类型的类。下面是具体实现：

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return this.getSpringFactoriesInstances(type, new Class[0]);
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = this.getClassLoader();
  	// 从"spring.factories"文件中加载指定的工厂名称
    Set<String> names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
  	// 通过名称来创建指定类型的实例。这里利用的是反射，并传入了必要的参数
    List<T> instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

总结：new SpringApplication()会读取"spring.factories"文件中并加载指定的工厂名称，用于后续run方法中实例化bean。

![image-20240909172130333](/Users/leiyu/Library/Application Support/typora-user-images/image-20240909172130333.png)



### SpringApplication.run

先用一张图总结run方法都干了啥：

![image-20240909172235736](/Users/leiyu/Library/Application Support/typora-user-images/image-20240909172235736.png)

来看看具体代码

```java
public ConfigurableApplicationContext run(String... args) {
    long startTime = System.nanoTime();
    DefaultBootstrapContext bootstrapContext = this.createBootstrapContext();
    ConfigurableApplicationContext context = null;
    this.configureHeadlessProperty();
  	// 应用运行监听器，并触发开始事件。
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
      	// 创建应用参数对象
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      	// 环境配置，包括配置文件和属性源
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        this.configureIgnoreBeanInfo(environment);
      	// 打印应用的banner，就是启动时的一个大SPRING标志(可以自定义的)
        Banner printedBanner = this.printBanner(environment);
      	// 创建应用上下文
        context = this.createApplicationContext();
        context.setApplicationStartup(this.applicationStartup);
      	// 准备上下文，包括加载bean定义
        this.prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
      	// 刷新上下文，完成bean的创建和初始化
        this.refreshContext(context);
      	// 后置处理
        this.afterRefresh(context, applicationArguments);
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), timeTakenToStartup);
        }

        listeners.started(context, timeTakenToStartup);
        this.callRunners(context, applicationArguments);
    } catch (Throwable var12) {
        this.handleRunFailure(context, var12, listeners);
        throw new IllegalStateException(var12);
    }

    try {
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        listeners.ready(context, timeTakenToReady);
        return context;
    } catch (Throwable var11) {
        this.handleRunFailure(context, var11, (SpringApplicationRunListeners)null);
        throw new IllegalStateException(var11);
    }
}
```

这里面核心主要是：

1. 创建上下文，就是创建一个ConfigurableApplicationContext context对象

2. 准备上下文：这里就是对ConfigurableApplicationContext context对象做一系列的初始化，后续会用到这个context来创建和初始化bean。

3. 刷新上下文：这里就是spring的核心启动步骤了，这一步包括了实例化所有Bean、设置他们的依赖关系以及执行他们的初始化任务。

   ```java
   public void refresh() throws BeansException, IllegalStateException {
       synchronized(this.startupShutdownMonitor) {
           StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
         	// 准备工作
           this.prepareRefresh();
         	// 刷新内部bean工厂
           ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
         	// bean工厂的准备工作
           this.prepareBeanFactory(beanFactory);
   
           try {
             	// 允许在上下文子类中对bean工厂进行后置处理
               this.postProcessBeanFactory(beanFactory);
               StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
             	// 调用在上下文中注册为bean的工厂处理器
               this.invokeBeanFactoryPostProcessors(beanFactory);
             	// 注册拦截bean创建的bean处理器
               this.registerBeanPostProcessors(beanFactory);
               beanPostProcess.end();
             	// 初始化此上下文的消息源
               this.initMessageSource();
             	// 初始化此上下文的事件多播器
               this.initApplicationEventMulticaster();
             	// 在特定上下文子类中初始化其他特殊bean，并在onRefresh方法中会创建TomcatServer容器并启动
               this.onRefresh();
             	// 检查监听器bean并注册
               this.registerListeners();
             	// 实例化所有剩余的(非懒加载)单例
               this.finishBeanFactoryInitialization(beanFactory);
             	// 最后：发布相应事件
               this.finishRefresh();
           } catch (BeansException var10) {
               if (this.logger.isWarnEnabled()) {
                   this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var10);
               }
   
               this.destroyBeans();
               this.cancelRefresh(var10);
               throw var10;
           } finally {
               this.resetCommonCaches();
               contextRefresh.end();
           }
   
       }
   }
   ```

最后springboot主要启动流程如下	![image-20240909175249031](/Users/leiyu/Library/Application Support/typora-user-images/image-20240909175249031.png)

## springboot启动时，bean是如何加载的

我们都知道spring bean在创建的时候是通过AbstractAutowireCapableBeanFactory`的doCreateBean方法实现的，那么在springboot启动过程中是在哪一步调用的doCreateBean方法呢？答案是在最后一步finishRefresh();时。

在springboot初始化过程中，所有的 Bean 定义会被注册到 BeanFactory 中。然后在finishRefresh() 方法时是通过 DefaultListableBeanFactory 的初始化过程来实现。（finishRefresh方法并不直接调用 doCreateBean）；

