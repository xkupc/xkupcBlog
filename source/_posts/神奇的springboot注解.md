---
title: 神奇的springboot注解
date: 2017-09-21 14:28:42
tags: [spring,spring-boot]
categories: springboot
---
## 简介
小编接触springboot的时间很短，平时也就拿着已经成熟的框架，写着无关痛痒的业务代码，在看druid的源码的时候，涉猎了很多注解，发现通过简单的一个注解，使大量的配置属性初始化到bean里，对我这种还停留在@value级别的小白来说简直大开眼界了。今天想着介绍一下springboot里我刚刚学到的几个厉害的注解。
## Configuration
在介绍厉害的之前我们先来入个门，这个注解在spring里就已经存在了，是spring用来替代通过xml配置bean的繁琐方式的。在spring里使用这个注解，我们需要在xml里指定扫描的路径，而在springboot里我们就完全不用指定，只要保证springboot启动入口的路径包含了注解所在的包就可以了。在springboot里这个注解一般用来在系统启动时初始化系统数据源，数据库中间件等一些系统配置项。
下面我们通过一个例子来看看：
<!--more-->
```
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
## ConfigurationProperties、EnableConfigurationProperties
有了上面Configuration的基础，我们来看看ConfigurationProperties是做了些啥呢。
修改一下baseModel实体类和UserConfig配置类
```
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
当我把baseModel里的get、set方法删除时，就初始化不了，很明显，这里依旧还是反射，看官们可能觉得你也可以这么做，容我贴一下我的配置文件：
```
user.user-name = test
user.user-age = 12
user.cityId = 330600
user.proviceId =331000
```
就问你惊不惊喜，意不意外。属性的key与baseModel里的属性名没有直接映射。如果我在baseModel里增加另一个类的依赖呢。
```

  //新增实体类
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
## ConditionalOnProperty
## AutoConfigureBefore 

