---
title: SpringBoot源码系列（一）
date: 2021-01-19 00:39:10
tags:
    - Spring
    - 源码
categories:    
    - Spring
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/springboot.png
---

# Spring项目架构预览
![Spring项目架构](/img/spring-boot-project.png)

> 推荐使用idea进行代码阅读

## 核心
`预定大于配置`，这是SpringBoot的一个原则。它的核心就是配置自动加载。
核心部分:
1. 注解: @EnableAutoConfiguration
2. 配置注册内容: spring.factories
3. 自动配置类: xxxAutoConfiguration
4. 前置条件注解: @Conditional
5. Starters: 三方组件

### 自动装配

#### @EnableAutoConfiguration注解
```
/**
  * 启动应用上下文自动配置，并配置你需要的bean。
  */
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	// 是否启动自动配置环境变量
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	// 自启动需要排除的类集合
	Class<?>[] exclude() default {};
}
```

**核心注解**: `@Import()`
```
/**
  * 和xml文件中的import作用一样，可以导入@Configuration注解类,ImportSelector实现类,ImportBeanDefinitionRegistrar实现类或者普通的pojo
  */
public @interface Import {
	Class<?>[] value();
}
```
**核心接口类**: ImportSelector
```
/**
  * 可以导入@Configuration注解类。通过下面的接口方法，放回一个string类型的集合，集合内是全类名。
  */
public interface ImportSelector {
    String[] selectImports(AnnotationMetadata importingClassMetadata);
}
```
**ImportSelector实现类**: **AutoConfigurationImportSelector**，也是自动配置中导入的类。
```
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
}
```
- 可以看出它实现了`DeferredImportSelector`类，`DeferredImportSelector`类是`ImportSelector`的子类。这个子类会在所有的`@Configuration`bean加载之后运行，返回配置类集合。
- 同时它还实现了BeanClassLoaderAware,ResourceLoaderAware,BeanFactoryAware,EnvironmentAware这四个接口，那么Spring就会在调用ImportSelector之前调用Aware接口的方法。
- 同时，它还实现了Order接口，可以通过@Order注解来自定义顺序。
- 加载配置的部分，后面详细分析

**EnableAutoConfiguration总结**: 
1. 通过`@Import`注解开启加载功能
2. 检查是否开启自动配置的环境变量
3. 排除不需要加载的类或者类名
4. 通过`ImportSelector`的`selectImports()`方法，加载引入了注解`@Congifuration`的类的全类名集合
以上就是自动装配的几个核心步骤。

### selectImports的具体过程
核心步骤之后就是具体流程了，selectImports方法几乎涵盖了组件自动装配的所有处理逻辑。以下类是`@Import`住家加载的类，可以查看详细的流程。

#### AutoConfigurationImportSelector
> 具体自动配置加载流程。
```
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
		
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // 判断是否开启了自动装配
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        // 通过注解元信息，获取对应包装之后的注解类信息
        AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
        // 将获取的配置信息转换为对应的字符串集合
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }
    
    protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        // 再次判断是否开启注解
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        }
        // 获取注解属性
        AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // 加载被认为是自动配置的类名。重点！！！
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        // 去重
        configurations = removeDuplicates(configurations);
        // 根据注解拿取需要排除的类的信息
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        // 排除类检查，确定当前类包含那些排除类
        checkExcludedClasses(configurations, exclusions);
        // 正式去除排除类
        configurations.removeAll(exclusions);
        // 获取过滤配置（将源码处拆分为两部，方便查看过程）
        ConfigurationClassFilter configurationClassFilter = getConfigurationClassFilter();
        // 对自动加载的配置进行过滤。把不匹配的过滤。重点！！！
        configurations = configurationClassFilter.filter(configurations);
        // 获取监听器配置，并进行事件广播
        fireAutoConfigurationImportEvents(configurations, exclusions);
        return new AutoConfigurationEntry(configurations, exclusions);
    }
    
    
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        // 使用指定的加载器加载对应工厂类型的所有类的全类名。
        List<String> list = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader())
        List<String> configurations = new ArrayList<>(list);
        // 在新版的springboot中，自动配置类已经不在META-INF/spring。factories中了。
        // 将其拆封开了，放入了META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports中
        ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);
        Assert.notEmpty(configurations,
                "No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
                        + "are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
    
    // 指定工厂类的类型为EnableAutoConfiguration，所以之后只会加载相关自动配置的类
    protected Class<?> getSpringFactoriesLoaderFactoryClass() {
        return EnableAutoConfiguration.class;
    }
    
    // 获取需要排除的类
    protected Set<String> getExclusions(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		Set<String> excluded = new LinkedHashSet<>();
		// 拿取注解上的配置
		excluded.addAll(asList(attributes, "exclude"));
		excluded.addAll(asList(attributes, "excludeName"));
		// 拿取xml的配置: spring.autoconfigure.exclude
		excluded.addAll(getExcludeAutoConfigurationsProperty());
		return excluded;
	}
	
	private ConfigurationClassFilter getConfigurationClassFilter() {
		if (this.configurationClassFilter == null) {
		    // 没有默认配置就从自动加载的配置里面找
			List<AutoConfigurationImportFilter> filters = getAutoConfigurationImportFilters();
			for (AutoConfigurationImportFilter filter : filters) {
			    // 循环过滤器，如果过滤器是Aware的实现类，且是BeanClassLoaderAware，BeanFactoryAware，EnvironmentAware，ResourceLoaderAware的实现类，
			    // 将当前对应的配置设置到对应的Aware实现类中
				invokeAwareMethods(filter);
			}
			// 实例化
			this.configurationClassFilter = new ConfigurationClassFilter(this.beanClassLoader, filters);
		}
		return this.configurationClassFilter;
	}
	
	// 获取自动配置中的监听配置集合，并进行封装
	private void fireAutoConfigurationImportEvents(List<String> configurations, Set<String> exclusions) {
		List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
		if (!listeners.isEmpty()) {
			AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this, configurations, exclusions);
			// 循环监听器并调用onAutoConfigurationImportEvent()方法进行广播
			for (AutoConfigurationImportListener listener : listeners) {
				invokeAwareMethods(listener);
				// 进行事件广播
				listener.onAutoConfigurationImportEvent(event);
			}
		}
	}
	
	// 获取自动配置过滤器
	protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
		return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
	}
	
	private void invokeAwareMethods(Object instance) {
		if (instance instanceof Aware) {
			if (instance instanceof BeanClassLoaderAware) {
				((BeanClassLoaderAware) instance).setBeanClassLoader(this.beanClassLoader);
			}
			if (instance instanceof BeanFactoryAware) {
				((BeanFactoryAware) instance).setBeanFactory(this.beanFactory);
			}
			if (instance instanceof EnvironmentAware) {
				((EnvironmentAware) instance).setEnvironment(this.environment);
			}
			if (instance instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) instance).setResourceLoader(this.resourceLoader);
			}
		}
	}

    // 内部类，用于实现过滤匹配	
	private static class ConfigurationClassFilter {
	    
		private final AutoConfigurationMetadata autoConfigurationMetadata;

		private final List<AutoConfigurationImportFilter> filters;

        // 加载 META-INF/spring-autoconfigure-metadata.properties 元数据
		ConfigurationClassFilter(ClassLoader classLoader, List<AutoConfigurationImportFilter> filters) {
			this.autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(classLoader);
			this.filters = filters;
		}
	    
	    List<String> filter(List<String> configurations) {
			long startTime = System.nanoTime();
			String[] candidates = StringUtils.toStringArray(configurations);
			boolean skipped = false;
			for (AutoConfigurationImportFilter filter : this.filters) {
			    // 过滤器匹配处理后的自动配置类与元文件
				boolean[] match = filter.match(candidates, this.autoConfigurationMetadata);
				for (int i = 0; i < match.length; i++) {
					if (!match[i]) {
						candidates[i] = null;
						skipped = true;
					}
				}
			}
			// 如果都不匹配则跳过，返回配置类
			if (!skipped) {
				return configurations;
			}
			List<String> result = new ArrayList<>(candidates.length);
			// 构建新的配置类集合
			for (String candidate : candidates) {
				if (candidate != null) {
					result.add(candidate);
				}
			}
			if (logger.isTraceEnabled()) {
				int numberFiltered = configurations.size() - result.size();
				logger.trace("Filtered " + numberFiltered + " auto configuration class in "
						+ TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime) + " ms");
			}
			return result;
		}
	}
}
```

#### ImportCandidates
    新版本的自动装配文件加载过程
```
// 包含@Configuration注解导入的后续配置，通常都是自动配置
public final class ImportCandidates implements Iterable<String> {

    private static final String LOCATION = "META-INF/spring/%s.imports";
    
	/**
	 * 加载classpath路径下的额外配置，文件名为META-INF/spring/full-qualified-annotation-name.imports 
	 * 每一行都是它的全类名，使用#标记注释 
	 */
	public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {
		Assert.notNull(annotation, "'annotation' must not be null");
		// 获取类加载器
		ClassLoader classLoaderToUse = decideClassloader(classLoader);
		// 根据传入的类名AutoConfiguration，转换路径；
		// META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
		String location = String.format(LOCATION, annotation.getName());
		// 构建为url的枚举对象，方面后续遍历
		Enumeration<URL> urls = findUrlsInClasspath(classLoaderToUse, location);
		List<String> importCandidates = new ArrayList<>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			// 根据枚举的url查找对应的自动配置文件，并返回类名的集合
			importCandidates.addAll(readCandidateConfigurations(url));
		}
		// 封装自动装配类名的集合并返回
		return new ImportCandidates(importCandidates);
	}
}
```


#### SpringFactoriesLoader
    spring.factories配置的加载
```
public final class SpringFactoriesLoader {
    
    // facotries的加载路径配置
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    // 加载目录"META-INF/spring.fatories"下的工厂实现类全类名 
    public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
        // 赋值类加载器，如果为null，默认为SpringFactoriesLoader这个类加载器
        ClassLoader classLoaderToUse = classLoader;
        if (classLoaderToUse == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }
        // 获取工厂类型的名称
        String factoryTypeName = factoryType.getName();
        // 获取到配置列表，再获取符合传入工厂类名的集合，默认为空集合
        return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
    }

    // 加载所有的META-INF/spring.factories文件，封装为Map
    private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        // 预先从缓存中获取
        Map<String, List<String>> result = cache.get(classLoader);
        if (result != null) {
            return result;
        }
    
        result = new HashMap<>();
        try {
            // 通过类加载器加载对应路径的资源
            Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                UrlResource resource = new UrlResource(url);
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    String factoryTypeName = ((String) entry.getKey()).trim();
                    String[] factoryImplementationNames =
                            StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                    for (String factoryImplementationName : factoryImplementationNames) {
                        result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                                .add(factoryImplementationName.trim());
                    }
                }
            }
    
            // 去重
            result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
                    .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
            cache.put(classLoader, result);
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Unable to load factories from location [" +
                    FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
        return result;
    }
}
```

#### AutoConfigurationImportFilter
    自动配置的过滤器
```
/** 注册在spring.factories中的过滤器，用来快速过滤自动配置，在读取字节码之前。
  * 当它的实现类，也是如下四个Aware接口的实现类时，会优先调用它们的方法，在调用filter
  * EnvironmentAware
  * BeanFactoryAware
  * BeanClassLoaderAware
  * ResourceLoaderAware
  */
public interface AutoConfigurationImportFilter {
  // 筛选对应的自动配置。返回的组合和传入的自动配置数组长度相同
  boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata);
}
```

#### FilteringSpringBootCondition 
    实现了AutoConfigurationImportFilter接口的match方法
> OnBeanCondition，OnClassCondition，OnWebApplicationCondition的父类抽象类，这三个子类是自动配置AutoConfigurationImportFilter的过滤器
```
abstract class FilteringSpringBootCondition extends SpringBootCondition
		implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {
  
    // 实现了过滤方法
    public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
		ConditionEvaluationReport report = ConditionEvaluationReport.find(this.beanFactory);
		// 抽象方法，具体是三个子类实现
		ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses, autoConfigurationMetadata);
		boolean[] match = new boolean[outcomes.length];
		for (int i = 0; i < outcomes.length; i++) {
		    // 返回结果判断，为null或者结果为true，最后结果就是true，否则为false
			match[i] = (outcomes[i] == null || outcomes[i].isMatch());
			if (!match[i] && outcomes[i] != null) {
			    // 打印错误日志
				logOutcome(autoConfigurationClasses[i], outcomes[i]);
				if (report != null) {
				    // 进行记录
					report.recordConditionEvaluation(autoConfigurationClasses[i], this, outcomes[i]);
				}
			}
		}
		return match;
	}
  
    // 类加载操作
	protected static Class<?> resolve(String className, ClassLoader classLoader) throws ClassNotFoundException {
		if (classLoader != null) {
			return Class.forName(className, false, classLoader);
		}
		return Class.forName(className);
	}
	
	protected enum ClassNameFilter {

		PRESENT {

			@Override
			public boolean matches(String className, ClassLoader classLoader) {
				return isPresent(className, classLoader);
			}

		},

		MISSING {

			@Override
			public boolean matches(String className, ClassLoader classLoader) {
				return !isPresent(className, classLoader);
			}

		};

		abstract boolean matches(String className, ClassLoader classLoader);

        // 通过类加载器来判断是否该类存在
		static boolean isPresent(String className, ClassLoader classLoader) {
			if (classLoader == null) {
				classLoader = ClassUtils.getDefaultClassLoader();
			}
			try {
				resolve(className, classLoader);
				return true;
			}
			catch (Throwable ex) {
				return false;
			}
		}

	}
}
```
#### OnClassCondition: 
```
@Order(Ordered.HIGHEST_PRECEDENCE)
class OnClassCondition extends FilteringSpringBootCondition {

    @Override
	protected final ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		// 有多个配置和多个后台线程时，使用一个额外的线程可以有更好的性能，更多则不是很理想
		if (autoConfigurationClasses.length > 1 && Runtime.getRuntime().availableProcessors() > 1) {
			return resolveOutcomesThreaded(autoConfigurationClasses, autoConfigurationMetadata);
		}
		else {
		    // 单个情况下处理
			OutcomesResolver outcomesResolver = new StandardOutcomesResolver(autoConfigurationClasses, 0,
					autoConfigurationClasses.length, autoConfigurationMetadata, getBeanClassLoader());
			return outcomesResolver.resolveOutcomes();
		}
	}
	
	private ConditionOutcome[] resolveOutcomesThreaded(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
	    // 获取配置的中间树
		int split = autoConfigurationClasses.length / 2;
		// 第一部分放入一个线程
		OutcomesResolver firstHalfResolver = createOutcomesResolver(autoConfigurationClasses, 0, split,
				autoConfigurationMetadata);
		// 第二部分正常执行
		OutcomesResolver secondHalfResolver = new StandardOutcomesResolver(autoConfigurationClasses, split,
				autoConfigurationClasses.length, autoConfigurationMetadata, getBeanClassLoader());
		ConditionOutcome[] secondHalf = secondHalfResolver.resolveOutcomes();
		ConditionOutcome[] firstHalf = firstHalfResolver.resolveOutcomes();
		ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
		// 合并返回结果
		System.arraycopy(firstHalf, 0, outcomes, 0, firstHalf.length);
		System.arraycopy(secondHalf, 0, outcomes, split, secondHalf.length);
		return outcomes;
	}
	
	// 内部类
	private static final class StandardOutcomesResolver implements OutcomesResolver {
	    
	    @Override
		public ConditionOutcome[] resolveOutcomes() {
			return getOutcomes(this.autoConfigurationClasses, this.start, this.end, this.autoConfigurationMetadata);
		}
        
        // 最后结果都会调用这个方法。间接调用getOutcome()方法
		private ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses, int start, int end,
				AutoConfigurationMetadata autoConfigurationMetadata) {
			// 构建新的集合
			ConditionOutcome[] outcomes = new ConditionOutcome[end - start];
			// 通过开始和结束的位置进行遍历
			for (int i = start; i < end; i++) {
			    // 活动对应自动配置的全类名
				String autoConfigurationClass = autoConfigurationClasses[i];
				if (autoConfigurationClass != null) {
				    // 非空情况下，从元数据中获取对应的配置类的ConditionalOnClass
					String candidates = autoConfigurationMetadata.get(autoConfigurationClass, "ConditionalOnClass");
					if (candidates != null) {
					    // 调用方法后，把结果放入对应位置
						outcomes[i - start] = getOutcome(candidates);
					}
				}
			}
			return outcomes;
		}
		
		private ConditionOutcome getOutcome(String candidates) {
			try {
                // 判断是否有多个，没有则继续往下走
				if (!candidates.contains(",")) {
					return getOutcome(candidates, this.beanClassLoader);
				}
				for (String candidate : StringUtils.commaDelimitedListToStringArray(candidates)) {
				    // 把对应的元数据中的配置
					ConditionOutcome outcome = getOutcome(candidate, this.beanClassLoader);
					if (outcome != null) {
						return outcome;
					}
				}
			}
			catch (Exception ex) {
				// We'll get another chance later
			}
			return null;
		}
		
		// 判断类是否符合条件
		private ConditionOutcome getOutcome(String className, ClassLoader classLoader) {
		    // 调用FilteringSpringBootCondition内部类ClassNameFilter来判断类加载情况
		    // 如果没有匹配到，就会将结果false进行封装，再进行返回
			if (ClassNameFilter.MISSING.matches(className, classLoader)) {
				return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
						.didNotFind("required class").items(Style.QUOTE, className));
			}
			return null;
		}
	}
}
```

#### 过滤器流程总结
- 从自动装配的配置中通过`AutoConfigurationImportFilter`类型获取过滤器集合
- 遍历过滤器，并把其中比较特殊的几种过滤器进行操作，然后返回对应的封装类
- 再次遍历过滤器，依次执行过滤器的`match()`方法
- 拼装自动配置类名和特定类名
- 通过类加载的方式校验是否存在
- 加载成功则匹配成功，抛出异常则失败
- 将匹配结果以特定长度的布尔值返回
- 根据返回值对自动配置类集合进行重新筛选

#### @Conditional注解
spring4.0引入的新特性，可根据是否满足指定条件来决定是否进行Bean的实例化及其装配。
```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
	// 所有的condition的类都必须满足match()方法才能够被注册
	Class<? extends Condition>[] value();
}
```
```
@FunctionalInterface
public interface Condition {
  // context: spring应用的上下文
  // metadata: 访问特定类的注解功能
  boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
拥有众多衍生注解，它们都组合了`@Conditional`注解:
- @ConditionalOnBean: 容器有指定的bean
- @ConditionalOnClass: 在classpath类路径下有指定类的条件下
- @ConditionalOnExpression: 基于SpEL表达式判断条件
- @ConditionalOnJava: 基于JVM版本作为判断条件
- @ConditionalOnResource: 类路径是否有指定的值
- @ConditionalOnWebApplication: 在项目是一个Web项目的条件下

```
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
// 组合@Conditional注解，并指定了对应的Condition类
@Conditional(OnWebApplicationCondition.class)
public @interface ConditionalOnWebApplication {

    	Type type() default Type.ANY;

	// web应用类型的枚举
	enum Type {
		ANY,
		SERVLET,
		REACTIVE
	}
}
```
`OnWebApplicationCondition`类也同样实现了`FilteringSpringBootCondition`类，其中还继承了`SpringBootCondition`
```
// 实现了Condition接口的match()方法，并定义了接口getMatchOutcome()
public abstract class SpringBootCondition implements Condition {
    
    public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 内部类，获取类名或者方法名
        String classOrMethodName = getClassOrMethodName(metadata);
		try {
		    // 抽象类，子类实现的
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			// 记录日志
			logOutcome(classOrMethodName, outcome);
			// 记录
			recordEvaluation(context, classOrMethodName, outcome);
			// 返回是否匹配
			return outcome.isMatch();
		}
		catch (NoClassDefFoundError ex) {
		    ...
		}
		catch (RuntimeException ex) {
			throw new IllegalStateException("Error processing condition on " + getName(metadata), ex);
		}
    }
    
    public abstract ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
子类`OnWebApplicationCondition`类实现`getMatchOutcome()`方法
```
@Order(Ordered.HIGHEST_PRECEDENCE + 20)
class OnWebApplicationCondition extends FilteringSpringBootCondition {

    @Override
	public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
	    // 判断是否包含注解
		boolean required = metadata.isAnnotated(ConditionalOnWebApplication.class.getName());
		// 判断是否为web应用
		ConditionOutcome outcome = isWebApplication(context, metadata, required);
		// 包含注解，不匹配，返回不匹配的错误信息
		if (required && !outcome.isMatch()) {
			return ConditionOutcome.noMatch(outcome.getConditionMessage());
		}
		// 不包含注解，但是匹配，返回不匹配的错误信息
		if (!required && outcome.isMatch()) {
			return ConditionOutcome.noMatch(outcome.getConditionMessage());
		}
		// 包含注解，并且匹配。返回正确的匹配信息
		return ConditionOutcome.match(outcome.getConditionMessage());
	}

    // 判断web应用
    private ConditionOutcome isWebApplication(ConditionContext context, AnnotatedTypeMetadata metadata,
			boolean required) {
		switch (deduceType(metadata)) {
			case SERVLET:
				return isServletWebApplication(context);
			case REACTIVE:
				return isReactiveWebApplication(context);
			default:
				return isAnyWebApplication(context, required);
		}
	}
	
	// 判断是否为servlet应用的四个点
	private ConditionOutcome isServletWebApplication(ConditionContext context) {
		ConditionMessage.Builder message = ConditionMessage.forCondition("");
		// 1. 尝试使用类加载方式加载对应的web应用类
		if (!ClassNameFilter.isPresent(SERVLET_WEB_APPLICATION_CLASS, context.getClassLoader())) {
			return ConditionOutcome.noMatch(message.didNotFind("servlet web application classes").atAll());
		}
		// 2. bean工厂非null，获取注册的所以作用域，并校验是否拥有session，如果有，则是web应用s
		if (context.getBeanFactory() != null) {
			String[] scopes = context.getBeanFactory().getRegisteredScopeNames();
			if (ObjectUtils.containsElement(scopes, "session")) {
				return ConditionOutcome.match(message.foundExactly("'session' scope"));
			}
		}
		// 3. 判断Environment是否为ConfigurableWebEnvironment
		if (context.getEnvironment() instanceof ConfigurableWebEnvironment) {
			return ConditionOutcome.match(message.foundExactly("ConfigurableWebEnvironment"));
		}
		// 4. 判断ResourceLoader是否为WebApplicationContext
		if (context.getResourceLoader() instanceof WebApplicationContext) {
			return ConditionOutcome.match(message.foundExactly("WebApplicationContext"));
		}
		return ConditionOutcome.noMatch(message.because("not a servlet web application"));
	}
}

```
