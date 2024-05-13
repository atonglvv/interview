自动装配的基础，是 Spring 从 4.x 版本开始支持 JavaConfig，让开发者可以免去繁琐的 xml 配置形式，而是使用熟悉的 Java 代码加注解，通过 @Configuration、@Bean 等注解可以直接向 Spring 容器注入 Bean 信息。

那么就有种设想，如果我把一些必须的 Bean 以 Java 代码方式准备好呢，只需要引入对应的配置类，相应的 Bean 就会被加载到 Spring 容器中。所以，有了这个基础 Spring Boot 就有了实现自动装配的可能。

先看一个springboot的启动类：

```java
@SpringBootApplication
public class WorkAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(WorkAdminApplication.class, args);
    }
}
```

在每个 Spring Boot 的启动类上，都会有这样一个复合注解 **@SpringBootApplication**，而它的内部是这样的。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
}
```

上面四个都是一些通用注解（元注解），至于 @SpringBootConfiguration，这个注解包含了@Configuration，@Configuration里面又包含了一个@Component注解，也就是说，**这个注解标注在哪个类上，就表示当前这个类是一个配置类，而配置类也是spring容器中的组件**

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@Indexed
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```

关键在于下面的 `@EnableAutoConfiguration`注解，从名字就可以看出来，它可以开启自动配置。在 EnableAutoConfiguration 内部，它使用 @Import 标注了AutoConfigurationImportSelector。通过 `@Import` 注解我们可以导入一些自定义的 BeanDefination 信息，或者导入一些配置类。如下关于EnableAutoConfiguration：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```

**@AutoConfigurationPackage**

这个注解的作用说白了就是将主配置类@SpringBootApplication标注的类所在包以及子包里面的所有组件扫描并加载到spring的容器中，这也就是为什么我们在利用springboot进行开发的时候，无论是Controller还是Service的路径都是与主配置类同级或者次级的原因。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({Registrar.class})
public @interface AutoConfigurationPackage {
    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};
}
```

**@Import({AutoConfigurationImportSelector.class})**

瞅一眼 AutoConfigurationImportSelector.class 的 uml类图：

![AutoConfigurationImportSelector](img\AutoConfigurationImportSelector.png)可以看到，它实现了 ImportSelector 接口。

让我们来看一下**AutoConfigurationImportSelector** 里 **selectImports** 方法。

```java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
            AutoConfigurationImportSelector.AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
            return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
        }
}
```

**getAutoConfigurationEntry**

```java
protected AutoConfigurationImportSelector.AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            AnnotationAttributes attributes = getAttributes(annotationMetadata);
            List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
            configurations = removeDuplicates(configurations);
            Set<String> exclusions = getExclusions(annotationMetadata, attributes);
            checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = getConfigurationClassFilter().filter(configurations);
            fireAutoConfigurationImportEvents(configurations, exclusions);
            return new AutoConfigurationImportSelector.AutoConfigurationEntry(configurations, exclusions);
        }
}
```

**getCandidateConfigurations**

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
}
```

**loadFactoryNames**

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
        ClassLoader classLoaderToUse = classLoader;
        if (classLoader == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }

        String factoryTypeName = factoryType.getName();
        return (List)loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

**loadSpringFactories**

```java
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = (Map)cache.get(classLoader);
        if (result != null) {
            return result;
        } else {
            HashMap result = new HashMap();

            try {
                Enumeration urls = classLoader.getResources("META-INF/spring.factories");

                while(urls.hasMoreElements()) {
                    URL url = (URL)urls.nextElement();
                    UrlResource resource = new UrlResource(url);
                    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                    Iterator var6 = properties.entrySet().iterator();

                    while(var6.hasNext()) {
                        Entry<?, ?> entry = (Entry)var6.next();
                        String factoryTypeName = ((String)entry.getKey()).trim();
                        String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                        String[] var10 = factoryImplementationNames;
                        int var11 = factoryImplementationNames.length;

                        for(int var12 = 0; var12 < var11; ++var12) {
                            String factoryImplementationName = var10[var12];
                            ((List)result.computeIfAbsent(factoryTypeName, (key) -> {
                                return new ArrayList();
                            })).add(factoryImplementationName.trim());
                        }
                    }
                }

                result.replaceAll((factoryType, implementations) -> {
                    return (List)implementations.stream().distinct().collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
                });
                cache.put(classLoader, result);
                return result;
            } catch (IOException var14) {
                throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var14);
            }
        }
}
```

