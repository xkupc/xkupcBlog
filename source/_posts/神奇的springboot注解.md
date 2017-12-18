---
title: 神奇的springboot注解
date: 2017-09-21 14:28:42
tags: [spring,spring-boot]
categories: java framework
---
## 简介
小编接触springboot的时间很短，平时也就拿着已经成熟的框架，写着无关痛痒的业务代码，在看druid的源码的时候，涉猎了很多注解，发现通过简单的一个注解，使大量的配置属性初始化到bean里，对我这种还停留在@value级别的小白来说简直大开眼界了。今天想着介绍一下springboot里我刚刚学到的几个厉害的注解。
## Configuration
在介绍厉害的之前我们先来入个门，这个注解在spring里就已经存在了，是spring用来替代通过xml配置bean的繁琐方式的。在spring里使用这个注解，我们需要在xml里指定扫描的路径，而在springboot里我们就完全不用指定，只要保证springboot启动入口的路径包含了注解所在的包就可以了。在springboot里这个注解一般用来在系统启动时初始化系统数据源，数据库中间件等一些系统配置项。
下面我们通过一个例子来看看：
<!--more-->
```java
@Configuration
public class UserConfig {
    @Value("${user.name}")
    private String userName;
    @Value("${user.age}")
    private int userAge;
    @Value("${user.cityId}")
    private String cityId;
    @Value("${user.proviceId}")
    private String proviceId;

    @Bean
    public BaseModel baseModel(){
        BaseModel baseModel = new BaseModel();
        baseModel.setUserName(userName);
        baseModel.setUserAge(userAge);
        baseModel.setCityId(cityId);
        baseModel.setProviceId(proviceId);
        return baseModel;
    }
}
    //测试
    @Autowired
    BaseModel baseModel;

    @Test
    public void getBaseModel(){
        System.out.println(baseModel.toString);
    }
```
很简单的demo，结果是baseModel按配置文件初始化了。
## ConfigurationProperties、EnableConfigurationProperties
有了上面Configuration的基础，我们来看看ConfigurationProperties是做了些啥呢。
修改一下baseModel实体类和UserConfig配置类
```java
//使用ConfigurationProperties注解，设置配置文件里关联属性的key的前缀
@ConfigurationProperties(prefix = "user")
public class BaseModel{
    private String userName;
    private int userAge;
    private String cityId;
    private String proviceId;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public int getUserAge() {
        return userAge;
    }

    public void setUserAge(int userAge) {
        this.userAge = userAge;
    }

    public String getCityId() {
        return cityId;
    }

    public void setCityId(String cityId) {
        this.cityId = cityId;
    }

    public String getProviceId() {
        return proviceId;
    }

    public void setProviceId(String proviceId) {
        this.proviceId = proviceId;
    }

    @Override
    public String toString() {
        return "BaseModel{" +
                "userName='" + userName + '\'' +
                ", userAge=" + userAge +
                ", cityId='" + cityId + '\'' +
                ", proviceId='" + proviceId + '\'' +
                '}';
    }
}
//UserConfig配置类
//这里还是使用Configuration初始化配置
@Configuration
//对BaseModel启用ConfigurationProperties，如果没有这个，默认不开启，那么在测试类里用Autowired就会报错
@EnableConfigurationProperties(BaseModel.class)
public class UserConfig {
}
```
测试结果依旧是baseModel按配置初始化成功
当我把baseModel里的get、set方法删除时，就初始化不了，很明显，这里依旧还是反射，看官们可能觉得你也可以这么做，容我贴一下我的配置文件：
```java
user.user-name = test
user.user-age = 12
user.cityId = 330600
user.proviceId =331000
```
就问你惊不惊喜，意不意外。属性的key与baseModel里的属性名没有直接映射，springboot支持“-”别名映射。如果我在baseModel里增加另一个类的依赖呢。
```java

  //在BaseModel里新增一个实体类属性
  private UserCompany workCompany;

  //配置文件新增属性
  user.work-company.company-name = 凯歌集团
  user.work-company.companyPhone = 123456
  
  //测试用例修改
   @Autowired
    BaseModel baseModel;

    @Test
    public void getBaseModel(){
        System.out.println(baseModel.toString());
        System.out.println(baseModel.getWorkCompany().toString());
    }
```
依旧初始化成功。是时候看一波源码了。
打开EnableConfigurationProperties，我可以看到他引入一个EnableConfigurationPropertiesImportSelector,接下来我们看看这个Selector都做了些啥
```java
public String[] selectImports(AnnotationMetadata metadata) {
		MultiValueMap<String, Object> attributes = metadata.getAllAnnotationAttributes(
				EnableConfigurationProperties.class.getName(), false);
        // 获取注解EnableConfigurationProperties里配置的class
		Object[] type = attributes == null ? null
				: (Object[]) attributes.getFirst("value");
		if (type == null || type.length == 0) {
			return new String[] {
					ConfigurationPropertiesBindingPostProcessorRegistrar.class
							.getName() };
		}
        //按demo里的例子，我们配置了@EnableConfigurationProperties(BaseModel.class)
        //则将ConfigurationPropertiesBeanRegistrar和ConfigurationPropertiesBindingPostProcessorRegistrar
        //完成BaseModel与配置属性的映射
		return new String[] { ConfigurationPropertiesBeanRegistrar.class.getName(),
				ConfigurationPropertiesBindingPostProcessorRegistrar.class.getName() };
	}
```
我们看ConfigurationPropertiesBeanRegistrar里的方法
```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
				BeanDefinitionRegistry registry) {
			MultiValueMap<String, Object> attributes = metadata
					.getAllAnnotationAttributes(
							EnableConfigurationProperties.class.getName(), false);
			List<Class<?>> types = collectClasses(attributes.get("value"));
			for (Class<?> type : types) {
				String prefix = extractPrefix(type);
				String name = (StringUtils.hasText(prefix) ? prefix + "-" + type.getName()
						: type.getName());
				if (!containsBeanDefinition((ConfigurableListableBeanFactory) registry,
						name)) {
                    //实际上只做了一件事就是如果配置类（demo里的BaseModel）未被注入容器，则完成配置类的注入
					registerBeanDefinition(registry, type, name);
				}
			}
		}
```
ConfigurationPropertiesBindingPostProcessorRegistrar里的方法
```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
			BeanDefinitionRegistry registry) {
		if (!registry.containsBeanDefinition(BINDER_BEAN_NAME)) {
			BeanDefinitionBuilder meta = BeanDefinitionBuilder
					.genericBeanDefinition(ConfigurationBeanFactoryMetaData.class);
            //这里注入了ConfigurationPropertiesBindingPostProcessor完成属性与配置的绑定
			BeanDefinitionBuilder bean = BeanDefinitionBuilder.genericBeanDefinition(
					ConfigurationPropertiesBindingPostProcessor.class);
			bean.addPropertyReference("beanMetaDataStore", METADATA_BEAN_NAME);
			registry.registerBeanDefinition(BINDER_BEAN_NAME, bean.getBeanDefinition());
			registry.registerBeanDefinition(METADATA_BEAN_NAME, meta.getBeanDefinition());
		}
	}
```
接下来我们看看ConfigurationPropertiesBindingPostProcessor是怎么实现绑定的。查看了<a href="https://docs.spring.io/spring-boot/docs/1.4.7.RELEASE/api/" target="_blank">springboot的官方api</a>,我们可以看到这样一句话：
```java
BeanPostProcessor to bind PropertySources to beans annotated with ConfigurationProperties.
```
通过BeanPostProcessor接口实现配置资源和实体类的绑定。在ConfigurationPropertiesBindingPostProcessor的方法中，我们只要找到实现了BeanPostProcessor的方法即可找到绑定的实现。ok，我们看到BeanPostProcessor有postProcessBeforeInitialization和postProcessAfterInitialization方法，看方法名我们大概有个模糊的认识，应该是在bean初始化前后的操作。我们看postProcessBeforeInitialization这个方法。
```java
private void postProcessBeforeInitialization(Object bean, String beanName, ConfigurationProperties annotation) {
        PropertiesConfigurationFactory<Object> factory = new PropertiesConfigurationFactory(bean);
        factory.setPropertySources(this.propertySources);
        factory.setValidator(this.determineValidator(bean));
        factory.setConversionService(this.conversionService == null ? this.getDefaultConversionService() : this.conversionService);
        if (annotation != null) {
            factory.setIgnoreInvalidFields(annotation.ignoreInvalidFields());
            factory.setIgnoreUnknownFields(annotation.ignoreUnknownFields());
            factory.setExceptionIfInvalid(annotation.exceptionIfInvalid());
            factory.setIgnoreNestedProperties(annotation.ignoreNestedProperties());
            if (StringUtils.hasLength(annotation.prefix())) {
                factory.setTargetName(annotation.prefix());
            }
        }

        try {
            //很明显，经过一系列校验后，通过加载配置的工厂类完成配置和属性的绑定
            factory.bindPropertiesToTarget();
        } catch (Exception var8) {
            String targetClass = ClassUtils.getShortName(bean.getClass());
            throw new BeanCreationException(beanName, "Could not bind properties to " + targetClass + " (" + this.getAnnotationDetails(annotation) + ")", var8);
        }
    }
```
好了，ConfigurationProperties就介绍到这里了，中间还有许多不明白的，源码看的很吃力，还需慢慢努力。属性和配置名不一致也能绑定成功，估计是反射的时候，对配置名做了处理。