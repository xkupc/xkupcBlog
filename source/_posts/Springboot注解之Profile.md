---
title: Springboot注解之Profile
date: 2018-01-31 15:57:46
tags: [spring,spring-boot]
categories: java framework
---
## 简介
荒废一周研究activiti无果，研究一下有意思的Springboot注解。我们都知道springboot提供spring.profiles.active参数配置让开发者配置不同的运行环境的配置。今天来介绍一个关于这个配置的另一个用法。还是来说一个业务场景，有一个需要发送手机验证码场景，我们都知道第三方的手机短信推送服务是按次数计费的，考虑到成本，我们在测试环境测试的时候可以不真实的将短信发送，而是保存短信记录，在后台管理页查看短信发送记录即可获取验证码。这个时候我们可能想到测试和生产环境要切换接口，那么如何优雅的切换接口呢？
## Profile注解
使用profile注解，我们可以将当前环境对应的配置bean或者service通过spirng的DI进行依赖注入。也就说一个短信发送服务接口，我有两种实现，一种实现测试环境不真实发送，一种实现生产环境真实发送，然后通过在两个实现上设置profile注解的环境值，spring会根据spring.profiles.active设置值来注入相应环境下的服务。
<!--more-->
## demo
配置文件配置：
spring.profiles.active=test
```java
public interface SMSSendService {
    /**
     * 发送文本验证码
     *
     * @param phone
     * @param content
     */
    public void sendSms(String phone, String content);

    /**
     * 发送语音验证码
     *
     * @param phone
     * @param content
     */
    public void sendVoiceSms(String phone, String content);
}
生产配置服务：
@Profile("prod")
@Service
public class SMSSendServiceImpl implements SMSSendService {
    .....
}
测试配置服务：
@Profile("test")
@Service
public class TestSMSSendServiceImpl implements SMSSendService {
    ....
}
使用：
    @Autowired
    private SMSSendService smsSendService;

    @Test
    public void sendSms() throws Exception{
       smsSendService.sendSms("132xxxxxxxx", "测试测试一下");
    }
```

