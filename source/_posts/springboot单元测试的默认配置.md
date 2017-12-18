---
title: springboot单元测试的默认配置
tags: [spring-boot,java]
categories: person idea
---
## 前言
最近根据<a href="https://tech.meituan.com/MT_Leaf.html" target="_blank">Leaf——美团点评分布式ID生成系统</a>中Leaf-snowflake方案的思路，实现了一个简单版本的分布式ID生成方法。在这过程中遇到了一些问题，在这里把问题记下来，当个错题集吧，觉得low的看官，请勿拍砖。因为只是一个简单的生成id方法，我并未想着去将之做成单个服务，而是一个工具包的形式，让有需求的应用系统依赖这个包。所以这个时候就要考虑单个系统部署集群的时候在zookeeper里的节点如何配置。于是我能想到了是根据ip和端口号去唯一标示集群里的单个服务。然后我们的问题来了。
## 问题
当我在Springboot的配置文件里配置服务的端口号，然后单元测试的时候，却怎么也获取不到这个端口号。
```java
he application.properties:
server.port = 30008

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(CustomerCenterApplication.class)
@WebAppConfiguration
public class BaseTestService {
@Value("${server.port}")
String serverPort;
@Test
public void test(){
System.out.println(serverPort);
}
}
```
<!--more-->
打印的结果很让人意外，是-1，而不是null之类的。在这个问题是纠结了一天之后，我选择升级springboot的版本，采用1.4.0以上的版本,采用以下的代码实现：
```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
public class BaseTestService {
@Value("${local.server.port}")或者@LocalServerPort
String serverPort;
@Test
public void test(){
System.out.println(serverPort);
}
}
```
## 原因
很神奇，可是为什么呢，还是那句话，源码不会骗人。既然是配置的问题，我们来看看Springboot启动测试的时候是如何加载配置的，我们立马想到@SpringApplicationConfiguration这个注解。
```java
@ContextConfiguration(loader = SpringApplicationContextLoader.class)
@Documented
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface SpringApplicationConfiguration {
    ...
}
```
可以看到通过这个注解，springboot加载了一个SpringApplicationContextLoader来完成上下文的配置加载。我们来看看这个loader做了些什么事。
```java
@Override
	public ApplicationContext loadContext(final MergedContextConfiguration config)
			throws Exception {
		assertValidAnnotations(config.getTestClass());
		SpringApplication application = getSpringApplication();
		application.setMainApplicationClass(config.getTestClass());
		application.setSources(getSources(config));
		ConfigurableEnvironment environment = new StandardEnvironment();
		if (!ObjectUtils.isEmpty(config.getActiveProfiles())) {
			setActiveProfiles(environment, config.getActiveProfiles());
		}
		Map<String, Object> properties = getEnvironmentProperties(config);
		addProperties(environment, properties);
		...
	}
```
我们看到 通过getEnvironmentProperties(config)获取配置文件配置信息，然后放到environment里。
```java
	protected Map<String, Object> getEnvironmentProperties(
			MergedContextConfiguration config) {
		Map<String, Object> properties = new LinkedHashMap<String, Object>();
		// JMX bean names will clash if the same bean is used in multiple contexts
		disableJmx(properties);
		properties.putAll(TestPropertySourceUtils
				.convertInlinedPropertiesToMap(config.getPropertySourceProperties()));
		if (!TestAnnotations.isIntegrationTest(config)) {
			properties.putAll(getDefaultEnvironmentProperties());
		}
		return properties;
	}

    private Map<String, String> getDefaultEnvironmentProperties() {
		return Collections.singletonMap("server.port", "-1");
	}
```
真想是如此简单。在最新的Springboot版本里这个方法改动很大，但依旧默认将端口设置为-1
```java
protected String[] getInlinedProperties(MergedContextConfiguration config) {
        ArrayList<String> properties = new ArrayList();
        this.disableJmx(properties);
        properties.addAll(Arrays.asList(config.getPropertySourceProperties()));
        if (!this.isEmbeddedWebEnvironment(config) && !this.hasCustomServerPort(properties)) {
            properties.add("server.port=-1");
        }

        return (String[])properties.toArray(new String[properties.size()]);
    }
```
想想大概是测试的时候不对外暴露服务吧。
