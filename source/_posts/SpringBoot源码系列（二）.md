---
title: SpringBoot源码系列（二）
date: 2023-03-03 15:19:13
tags:
    - Spring
    - 源码
categories:    
    - Spring
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---
## Spring启动构造方法流程

> main方法启动
```
@SpringBootApplication
public class ApplicationStart {
    public static void main(String[] args) {
        // 基础也是最简单的启动方法。通过静态方法调用。
        SpringApplication.run(ApplicationStart.class, args);
        
        // 使用构建方法启动
        new SpringApplication(ApplicationStart.class).run(args);
    }
}
```
> 具体过程
```
public class SpringApplication {
    
    public SpringApplication(Class<?>... primarySources) {
		this(null, primarySources);
	}
    
    // resourceLoader: 资源加载接口，默认实现类DefaultResourceLoader。
    // primarySources: 可变参数，默认传入入口类，但必须带有自动配置注解
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        // 赋值成员变量资源加载器，banner等
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		// 赋值成员变量
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		// 推断web应用类型
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		// 获取自动加载配置BootstrapRegistryInitializer，实例化后赋值成员变量
		this.bootstrapRegistryInitializers = new ArrayList<>(
				getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
		// 获取自动加载配置ApplicationContextInitializer，实例化后赋值成员变量
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		// 获取自动加载配置ApplicationListener，实例化后赋值成员变量
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		推断main方法class
		this.mainApplicationClass = deduceMainApplicationClass();
	}
	
	// 获取实例
	private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
	}
	
	private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
		// 依旧是通过META-INF/spring.factories配置，获取对应类型的全部配置
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		// 创建实例
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		// 排序
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
	
	private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
			ClassLoader classLoader, Object[] args, Set<String> names) {
		List<T> instances = new ArrayList<>(names.size());
		// 循环实例名
		for (String name : names) {
			try {
			    // 通过反射获取类
				Class<?> instanceClass = ClassUtils.forName(name, classLoader);
				Assert.isAssignable(type, instanceClass);
				// 获取构建方法
				Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
				// 实例化
				T instance = (T) BeanUtils.instantiateClass(constructor, args);
				instances.add(instance);
			}
			catch (Throwable ex) {
				throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
			}
		}
		return instances;
	}
	
	private Class<?> deduceMainApplicationClass() {
		try {
		    // 获取栈元素
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {
			    // 遍历获取第一个带有main方法的类，并返回
				if ("main".equals(stackTraceElement.getMethodName())) {
					return Class.forName(stackTraceElement.getClassName());
				}
			}
		}
		catch (ClassNotFoundException ex) {
			// Swallow and continue
		}
		return null;
	}
}
```

### WebApplicationType: web类型推断
    枚举类，进行web类型推断
```
public enum WebApplicationType {
    
    NONE,
	SERVLET,
	REACTIVE;
    
    private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
    
	private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";
	private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";
	private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

    static WebApplicationType deduceFromClasspath() {
        // 很明显，都是尝试加载类来判断
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		// 同上
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
		// 默认为servlet应用
		return WebApplicationType.SERVLET;
	}
}
```
### 注入成员变量分析
- BootstrapRegistryInitializer: 回调接口，在初始化BootstrapRegistry之前做操作
    - BootstrapRegistry: 该接口是一个简单的对象注册表，在启动和 Environment 后处理期间可用，直到准备好ApplicationContext为止。可以用于注册创建成本较高或需要在ApplicationContext可用之前共享的实例。注册表使用Class作为键，这意味着只能存储给定类型的单个实例。
- ApplicationContextInitializer: 回调接口，用于在初始化ConfigurableApplicationContext类型的spring上下文做刷新之前，对ConfigurableApplicationContext做进一步设置
- ApplicationListener: 时间监听器，容器启动过程中会定义一些事件，事件发布之后就，监听器就会作出一些操作

```
// spring boot 2.4.5 版本之后才有的方法
@FunctionalInterface
public interface BootstrapRegistryInitializer {
	// BootstrapRegistry: 
	void initialize(BootstrapRegistry registry);
}

```

## Spring启动运行流程
上面是spring构建的过程的，包含一些赋值，推断和判断，现在我们可以了解下`run()`方法里面做了那些操作

### Spring的run()方法

```
public class SpringApplication {
    // 启动spring应用，构建和刷新一个新的应用上下文
    public ConfigurableApplicationContext run(String... args) {
        // jvm的当前时间
        long startTime = System.nanoTime();
        // 创建默认的上下文
        DefaultBootstrapContext bootstrapContext = createBootstrapContext();
        ConfigurableApplicationContext context = null;
        // 设置默认的java.awt.headless的环境配置
        configureHeadlessProperty();
        // 通过传入参数，获取SpringApplicationRunListener的数组
        SpringApplicationRunListeners listeners = getRunListeners(args);
        // 启动监听器，启动类是在构造方法中进行确认的
        listeners.starting(bootstrapContext, this.mainApplicationClass);
        try {
            // 根据参数创建ApplicationArguments对象
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            // 加载属性配置
            ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            configureIgnoreBeanInfo(environment);
            // 打印banner
            Banner printedBanner = printBanner(environment);
            // 根据web类型创建容器
            context = createApplicationContext();
            // 设置启动检测
            context.setApplicationStartup(this.applicationStartup);
            // 准备容器，组件对象之间进行关联
            prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            // 初始化容器
            refreshContext(context);
            // 初始化之后执行，空方法，可自行实现
            afterRefresh(context, applicationArguments);
            // 获取启动时长
            Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
            // 打印启动日志
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
            }
            // 通知监听器，容器启动完成
            listeners.started(context, timeTakenToStartup);
            // 调用ApplicationRunner和CommandLineRunner
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, listeners);
            throw new IllegalStateException(ex);
        }
        try {
            Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
            listeners.ready(context, timeTakenToReady);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }
    
    // 创建并初始化默认的ConfigurableBootstrapContext
    private DefaultBootstrapContext createBootstrapContext() {
		DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
		this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
		return bootstrapContext;
	}
}   

```
以上步骤的核心流程为:
- 获取监听器及其参数配置
- 打印banner
- 创建及其初始化容器
- 监听器发送通知
后续我们再来看详细内容。

#### SpringApplicationRunListener
##### 获取和加载监听器
```
public class SpringApplication {
    // 获取监听器
    private SpringApplicationRunListeners getRunListeners(String[] args) {
        // 构建一个class类型的数组
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		// 通过构建方法，把日志，监听器数组和应用启动步骤器
		return new SpringApplicationRunListeners(logger,
		        // 之前介绍过，通过spring.factories配置，获取SpringApplicationRunListener对应配置的实现：EventPublishingRunListener并进行初始化
		        // 可以看到 
				getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args),
				this.applicationStartup);
	}
	
	private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
	    ClassLoader classLoader = getClassLoader();
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		// 实例化，传入上面方法传过来的this和args作为构造器的参数
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
}
```
##### SpringApplicationRunListener方法

```
public interface SpringApplicationRunListener {
    // run方法第一次执行，会被立即调用，可用于非常早起的初始化功罪
    default void starting(ConfigurableBootstrapContext bootstrapContext) {}
    // 环境变量准备完成，容器创建之前
    default void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
			ConfigurableEnvironment environment) {}
	// 容器第一次创建，但在资源加载之前
	default void contextPrepared(ConfigurableApplicationContext context) {}
	// 容器第一次加载完成，未被刷新之前
	default void contextLoaded(ConfigurableApplicationContext context) {}
	// 容器刷新和容器已经启动，但是ApplicationRunner和CommandLineRunner未被调用前
	default void started(ConfigurableApplicationContext context, Duration timeTaken) {
		started(context);
	}
	// 已废弃，使用上面的方法进行替代
	@Deprecated
	default void started(ConfigurableApplicationContext context) {}
	// run方法调用完成之后，当容器已经刷新和ApplicationRunner和CommandLineRunner被调用后，2.6.0版本使用这个方法代替以前的
	default void ready(ConfigurableApplicationContext context, Duration timeTaken) {
		running(context);
	}
	// 已废弃
	@Deprecated
	default void running(ConfigurableApplicationContext context) {}
	// 程序出现错误
	default void failed(ConfigurableApplicationContext context, Throwable exception) {}
}
```
对各个阶段进行监听，通过实现该接口可以实现不同的阶段的功能，下图展示流程中的各个监听器方法的使用时机
![SpringListenter](/img/SpringApplicationRunListener.png)

##### EventPublishingRunListener
SpringApplicationRunListener在配置文件中的实现类。
```
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    
    public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		// 创建默认的广播事件
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<?> listener : application.getListeners()) {
		    // 将applicationListener与当前监听器进行关联
			this.initialMulticaster.addApplicationListener(listener);
		}
	}
	
	@Override
	public void starting(ConfigurableBootstrapContext bootstrapContext) {
		this.initialMulticaster
				.multicastEvent(new ApplicationStartingEvent(bootstrapContext, this.application, this.args));
	}
	
	@Override
	public void contextLoaded(ConfigurableApplicationContext context) {
		for (ApplicationListener<?> listener : this.application.getListeners()) {
			if (listener instanceof ApplicationContextAware) {
				((ApplicationContextAware) listener).setApplicationContext(context);
			}
			context.addApplicationListener(listener);
		}
		this.initialMulticaster.multicastEvent(new ApplicationPreparedEvent(this.application, this.args, context));
	}
}
```
EventPublishingRunListener针对不同的事件提供了不同的处理方法，但大致都相同，大致为三个流程:
- 事件方法被调用
- 封装application和args到对应事件内
- 通过multicastEvent或者publishEvent发布事件
  
contextLoaded()方法比较特殊，它不仅需要发布事件，还需要将监听器设置到应用上下文中，入股监听器实现了ApplicationContextAware接口，还设置上下文到监听器中。

contextLoaded()方法完成，上下文加载完成，我们就可以通过上下问的publishEvent来进行事件的发布，spring文档上的事件发布就是这个方法。

#### prepareEnvironment
    加载配置属性
```
    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
		// 获取或创建环境变量
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		// 配置环境参数，主要为PropertySources和
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		// 将ConfigurationPropertySource附加在指定环境的第一位，而且动态跟踪环境的添加和删除操作
		ConfigurationPropertySources.attach(environment);
		// 监听器启动
		listeners.environmentPrepared(bootstrapContext, environment);
		// 将defaultProperties配置移动到最后
		DefaultPropertiesPropertySource.moveToEnd(environment);
		// 判断环境配置中不包含environment-prefix
		Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
				"Environment prefix cannot be set via properties.");
        // 将环境绑定到Application
		bindToSpringApplication(environment);
		// 判断是否为定制环境，如果不是就则将环境转换为对应环境类型
		if (!this.isCustomEnvironment) {
			EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
			environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
		}
		// 将ConfigurationPropertySource附加在指定环境的第一位，而且动态跟踪环境的添加和删除操作 
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
	// 有默认的工厂构造器
	private ApplicationContextFactory applicationContextFactory = ApplicationContextFactory.DEFAULT;
	
	private ConfigurableEnvironment getOrCreateEnvironment() {
	    // 判断是否为null
		if (this.environment != null) {
			return this.environment;
		}
		// 在初始化构造方法时，获取到的web类型为servlet，所以结果为ApplicationServletEnvironment
		ConfigurableEnvironment environment = this.applicationContextFactory.createEnvironment(this.webApplicationType);
		if (environment == null && this.applicationContextFactory != ApplicationContextFactory.DEFAULT) {
			environment = ApplicationContextFactory.DEFAULT.createEnvironment(this.webApplicationType);
		}
		return (environment != null) ? environment : new ApplicationEnvironment();
	}
	
	// 配置环境属性
	protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
	    // 属性需要类型转换进行设置
		if (this.addConversionService) {
		    // 使用默认的转换服务ApplicationConversionService
			environment.setConversionService(new ApplicationConversionService());
		}
		// 配置PropertySources
		configurePropertySources(environment, args);
		// 配置profiles，其中没有相关逻辑代码
		configureProfiles(environment, args);
	}
	
	// 存在默认属性时，它的优先级是最低的
	// 如果存在命令行参数是，不包含在默认的资源属性中，它的优先级最高，包含的话，则会经过CompositePropertySource类进行去重处理
	protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
	    // 获取资源属性集合
		MutablePropertySources sources = environment.getPropertySources();
		// 如果默认配置非空，先对比，如果存在则替换，不存在则放入资源属性的最后位置
		if (!CollectionUtils.isEmpty(this.defaultProperties)) {
			DefaultPropertiesPropertySource.addOrMerge(this.defaultProperties, sources);
		}
		// 如果命令行属性存在
		if (this.addCommandLineProperties && args.length > 0) {
		    // 如果默认属性资源中不包含该命令，则将命令放在第一个位置，如果包含，则通过CompositePropertySource处理
			String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
			if (sources.contains(name)) {
				PropertySource<?> source = sources.get(name);
				CompositePropertySource composite = new CompositePropertySource(name);
				composite.addPropertySource(
						new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
				composite.addPropertySource(source);
				sources.replace(name, composite);
			}
			else {
				sources.addFirst(new SimpleCommandLinePropertySource(args));
			}
		}
	}
```
##### 初始化ConfigurableEnvironment
ConfigurableEnvironment接口提供了当前运行环境的公开接口，比如配置文件profiles各类系统属性和变量的设置、添加、读取、合并等功能
```
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {
    // 设置激活的组集合    
    void setActiveProfiles(String... profiles);
    // 向当前激活的组集合中添加一个profiles
    void addActiveProfile(String profile);
    // 设置默认的激活的组集合
    void setDefaultProfiles(String... profiles);
    // 获取当前环境对象中的属性源集合，也就是应用环境变量
    MutablePropertySources getPropertySources();
    // 获取虚拟机环境变量，该方法提供了直接配置虚拟机环境变量的入口
    Map<String, Object> getSystemProperties();
    // 获取操作系统环境变量
    // 该方法提供了直接配置环境变量的入口
    Map<String, Object> getSystemEnvironment();
    // 合并指定环境中的配置到当前环境中
    void merge(ConfigurableEnvironment parent);
}
```
#### configureIgnoreBeanInfo: 忽略配置beanInfo
```
// 这个配置决定是否跳过BeanInfo类的扫描
private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {
    // 如果系统配置中不包含 spring.beaninfo.ignore 配置，则将其默认设置为ture
    if (System.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {
        Boolean ignore = environment.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME,
                Boolean.class, Boolean.TRUE);
        System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME, ignore.toString());
    }
}
```
#### 打印Banner
```
// 初始化SpringApplicationBannerPrinter类及其打印
private Banner printBanner(ConfigurableEnvironment environment) {
    // 检查是否开启打印配置    
    if (this.bannerMode == Banner.Mode.OFF) {
        return null;
    }
    // 获取资源加载器
    ResourceLoader resourceLoader = (this.resourceLoader != null) ? this.resourceLoader
            : new DefaultResourceLoader(null);
    // 构建SpringApplicationBannerPrinter
    SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(resourceLoader, this.banner);
    // 打印到日志中
    if (this.bannerMode == Mode.LOG) {
        return bannerPrinter.print(environment, this.mainApplicationClass, logger);
    }
    // 打印到控制台
    return bannerPrinter.print(environment, this.mainApplicationClass, System.out);
}
```
#### prepareContext: 上下文准备
```
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
	// 第一阶段: 准备阶段		
    
    // 上下文设置环境配置
    context.setEnvironment(environment);
    // 后置处理
    postProcessApplicationContext(context);
    // context刷新之前，使用ApplicationContextInitializer进行初始化。
    applyInitializers(context);
    
    // 第一阶段准备操作完成
    
    // 第二阶段: 应用上下文加载
    
    // 监听器处理，容器第一次加载
    listeners.contextPrepared(context);
    // 上下文已经准备好，BootstrapContext关闭
    bootstrapContext.close(context);
    // 记录start日志
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // 获取ConfigurableListableBeanFactory并注册单例对象
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    // 如果是AbstractAutowireCapableBeanFactory的实现，则设置循环引用，默认为true。
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
        ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
        // 设置是否覆盖注册
        if (beanFactory instanceof DefaultListableBeanFactory) {
            ((DefaultListableBeanFactory) beanFactory)
                    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
    }
    // 添加惰性初始化的bean工厂到上下文内部工厂列表
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // 设置bean工厂
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    // 加载全部配置源
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 将beans加载进应用上下文
    load(context, sources.toArray(new Object[0]));
    // 监听器: 通知监听器上下文加载完成
    listeners.contextLoaded(context);
    
    // 第二阶段加载完成
}
```

##### postProcessApplicationContext
    后置处理
```
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    // 如果beanNameGenerator不为null，则按照默认名字注册单例bean
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                this.beanNameGenerator);
    }
    // resourceLoader不为null，根据它的不同类型，设置资源加载器和类加载器
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
    // 如果有转换服务，则在上下文环境配置中设置为true
    if (this.addConversionService) {
        context.getBeanFactory().setConversionService(context.getEnvironment().getConversionService());
    }
}
```
##### applyInitializers
    初始化
```
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 获取ApplicationContextInitializer集合并遍历
    for (ApplicationContextInitializer initializer : getInitializers()) {
        // 针对给定目标类解析给定泛型接口的单个类型参数，该目标类被假定为实现泛型接口，并可能为其类型变量声明具体类型
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                ApplicationContextInitializer.class);
        // 判断类型是否匹配
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        // 上下文进行初始化
        initializer.initialize(context);
    }
}
```
#### ConfigurableApplicationContext
```
// spi接口
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {
    // 在任何bean定义之前，添加一个bean工厂到上下文内部的bean工厂列表中，并在刷新上下文时调用
    // 该方法在配置上下文时调用
    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);
}
```
##### ConfigurableListableBeanFactory
```
// 提供了更多的bean工厂接口，除了ConfigurableBeanFactory接口提供的外，还包含了分析，修改bean定义，以及预实例化单例的工具
public interface ConfigurableListableBeanFactory
		extends ListableBeanFactory, AutowireCapableBeanFactory, ConfigurableBeanFactory {
		
}
```
目前它只有一个实现类: `DefaultListableBeanFactory`，这个类也是`AnnotationConfigServletWebServerApplicationContext`中`beanFactory`的默认实现。
`AnnotationConfigServletWebServerApplicationContext`继承于`GenericApplicationContext`
```
public class GenericApplicationContext {
    public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
}
```
##### getAllSources
    获取所有资源配置
```
public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet<>();
    // 存在primarySources和sources时，进行设置
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        allSources.addAll(this.primarySources);
    }
    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }
    // 并设置为不可修改
    return Collections.unmodifiableSet(allSources);
}
```
##### load()
    加载配置源到上下文
```
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    // 创建loader
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    // 加载其它的配置
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}
protected BeanDefinitionLoader createBeanDefinitionLoader(BeanDefinitionRegistry registry, Object[] sources) {
    return new BeanDefinitionLoader(registry, sources);
}
```
##### BeanDefinitionLoader
    从底层源加载bean定义，包括XML和JavaConfig
```
class BeanDefinitionLoader {
    // 支持三种类型的加载方式
    BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
		Assert.notNull(registry, "Registry must not be null");
		Assert.notEmpty(sources, "Sources must not be empty");
		this.sources = sources;
		this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
		this.xmlReader = (XML_ENABLED ? new XmlBeanDefinitionReader(registry) : null);
		this.groovyReader = (isGroovyPresent() ? new GroovyBeanDefinitionReader(registry) : null);
		this.scanner = new ClassPathBeanDefinitionScanner(registry);
		this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
	}
    void load() {
		for (Object source : this.sources) {
			load(source);
		}
	}
	// 加载资源，支持加载四种类型
	private void load(Object source) {
		Assert.notNull(source, "Source must not be null");
		if (source instanceof Class<?>) {
			load((Class<?>) source);
			return;
		}
		if (source instanceof Resource) {
			load((Resource) source);
			return;
		}
		if (source instanceof Package) {
			load((Package) source);
			return;
		}
		if (source instanceof CharSequence) {
			load((CharSequence) source);
			return;
		}
		throw new IllegalArgumentException("Invalid source type " + source.getClass());
	}
}
```
####  refreshContext: 初始化上下文
```
private void refreshContext(ConfigurableApplicationContext context) {
    if (this.registerShutdownHook) {
        // 注册hooks
        shutdownHook.registerApplicationContext(context);
    }
    refresh(context);
}

protected void refresh(ConfigurableApplicationContext applicationContext) {
    applicationContext.refresh();
}
```
##### AbstractApplicationContent的refresh()方法
    spring-content中的内容，每一步其实都有官方的注释。这之后，Spring上下文正式开启，SpringBoot的核心特性也随之启动。
```
@Override
public void refresh() throws BeansException, IllegalStateException {
    // 同步处理
    synchronized (this.startupShutdownMonitor) {
        // 启动标识
        StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

        // 准备刷新工作
        prepareRefresh();

        // 通知子类刷新内部bean工厂
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 为当前context准备bean工厂
        prepareBeanFactory(beanFactory);

        try {
            // 运行context的子类对bean工厂进行后置处理
            postProcessBeanFactory(beanFactory);
            // 标识后置处理
            StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // 调用context在中注册为bean的工厂处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // 注册bean处理器
            registerBeanPostProcessors(beanFactory);
            // 后置处理结束
            beanPostProcess.end();

            // 初始化context的信息源，和国际化有关
            initMessageSource();

            // 初始化context中事件广播
            initApplicationEventMulticaster();

            // 初始化其它特殊子类的bean
            onRefresh();

            // 检查并注册事件监听器
            registerListeners();

            // 实例化所有非懒加载的单例bean
            finishBeanFactoryInitialization(beanFactory);

            // 最后一步: 发布对应事件
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // 删除以及初始化的单例bean
            destroyBeans();

            // active复位
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // 重置Spring核心中的公共内省缓存
            resetCommonCaches();
            // 流程结束标识
            contextRefresh.end();
        }
    }
}
```
#### callRunners
    通过spring的callRunner运行
```
private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<>();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    for (Object runner : new LinkedHashSet<>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}

private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
    try {
        // 直接使用
        (runner).run(args);
    }
    catch (Exception ex) {
        throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
    }
}

private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
    try {
        // 获取string参数数组
        (runner).run(args.getSourceArgs());
    }
    catch (Exception ex) {
        throw new IllegalStateException("Failed to execute CommandLineRunner", ex);
    }
}
```
