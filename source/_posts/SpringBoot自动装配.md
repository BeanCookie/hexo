---
title: SpringBoot自动装配
date: 2020-10-29 16:05:46
tags:
---

#### 自动装配原理

相较于传统的Spring项目，SpringBoot的项目配置称得上极简主义，整个项目没有一个xml配置，就连唯一application.yml里面也是空无一物好像只是象征性的摆设。那么SpringBoot究竟是用了怎样的魔法对一向臃肿的JavaEE项目进行大瘦身的呢，这里的关键就是自动装配。

明确了研究主题之后我们就先从唯一个有疑点的类SpringBootDemoApplication开始，这是由SpringBoot自动生成的类，打开之后会发现一个最显目的@SpringBootApplication注解，打开源码会发现这个类相关复杂，接着看@EnableAutoConfiguration注解点开源码会发现这里Impoert了AutoConfigurationImportSelector，这个类就是用户导入自动装配的项目Bean的。

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    // 判断是否开启了自动配置
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    // 获取所有需要被注入到Ioc容器中的配置类
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

```java
/**
* Return the {@link AutoConfigurationEntry} based on the {@link AnnotationMetadata}
* of the importing {@link Configuration @Configuration} class.
* @param annotationMetadata the annotation metadata of the configuration class
* @return the auto-configurations that should be imported
*/
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 获取所有自动配置的类名
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);
    // 去除在@EnableAutoConfiguration配置的需要被排除的配置类
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = getConfigurationClassFilter().filter(configurations);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

```java
/**
* Return the auto-configuration class names that should be considered. By default
* this method will load candidates using {@link SpringFactoriesLoader} with
* {@link #getSpringFactoriesLoaderFactoryClass()}.
* @param metadata the source metadata
* @param attributes the {@link #getAttributes(AnnotationMetadata) annotation
* attributes}
* @return a list of candidate configurations
*/
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 获取所有key为org.springframework.boot.autoconfigure.EnableAutoConfiguration的配置项
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

```java
// 
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
	return EnableAutoConfiguration.class;
}
```

```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        // 加载所有依赖META-INF目录下spring.factories中配置的自动装配配置类的信息
        Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION)); // META-INF/spring.factories
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

#### 自定义starter项目

```shell
mvn archetype:generate -DgroupId=com.zeaho.spring -DartifactId=spring-boot-youdao-starter -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

```java
@ConfigurationProperties("youdao")
public class YouDaoTranslateProperties {
    private String appKey;
    private String appSecret;
    private String youdaoUrl;

    public String getAppKey() {
        return appKey;
    }

    public void setAppKey(String appKey) {
        this.appKey = appKey;
    }

    public String getAppSecret() {
        return appSecret;
    }

    public void setAppSecret(String appSecret) {
        this.appSecret = appSecret;
    }

    public String getYoudaoUrl() {
        return youdaoUrl;
    }

    public void setYoudaoUrl(String youdaoUrl) {
        this.youdaoUrl = youdaoUrl;
    }
}
```

```java
public class TranslateServiceImpl implements TranslateService {
    private final Logger logger = LoggerFactory.getLogger(TranslateServiceImpl.class);
    private final YouDaoTranslateProperties youDaoTranslateProperties;

    public TranslateServiceImpl(YouDaoTranslateProperties youDaoTranslateProperties) {
        this.youDaoTranslateProperties = youDaoTranslateProperties;
    }
    @Override
    public String translate(String text) {
        logger.info("{} {} {}", youDaoTranslateProperties.getAppKey(), youDaoTranslateProperties.getAppSecret(), youDaoTranslateProperties.getYoudaoUrl());
        return String.format("translate: %s", text);
    }
}
```

```java
@Configuration
@ConditionalOnClass(TranslateService.class)
@EnableConfigurationProperties(YouDaoTranslateProperties.class)
public class YouDaoTranslateAutoConfiguration {
    @Resource
    private YouDaoTranslateProperties youDaoTranslateProperties;

    @Bean
    @ConditionalOnMissingBean(TranslateService.class)
    public TranslateService translateService() {
        return new TranslateServiceImpl(youDaoTranslateProperties);
    }
}
```

添加META-INF/spring.factories

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.zeaho.spring.config.YouDaoTranslateAutoConfiguration
```

