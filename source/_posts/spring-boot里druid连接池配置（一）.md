---
title: spring-boot里druid连接池配置（一）
date: 2017-09-18 19:20:18
tags: [druid,spring-boot]
categories: java中间件
---

## 简介
J2EE项目大部分都会涉及到数据库，为了提高操作数据库的性能，提高访问的并发量，大部分的应用会选择使用数据库连接池，目前比较成熟的数据库连接池有许多，像c3p0,dhcp等已经成为ORM中间件的标配。druid作为数据库连接池的后起之秀，号称史上性能最好的数据库连接池，逐步的被广泛应用于互联网公司项目中。今天介绍一下druid在spring-boot里的简单的使用。后续将更新druid的一些配置含义

## 依赖配置
小编使用了druid目前支持的集成spring-boot版本，主要的maven依赖如下：

```js
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.3</version>
</dependency>
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.1.1</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.14</version>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
</dependency>
```
<!--more-->
## 数据库配置
数据库使用mysql数据库
```
####################数据库配置-start#############################
spring.datasource.druid.url=jdbc:mysql://xxxx:3306/dataBaseName?autoReconnect=true&useUnicode=true&characterEncoding=utf8&useSSL=true
spring.datasource.druid.username=root
#spring.datasource.druid.password=123456
# 生成的加密后的密码
spring.datasource.druid.password=HT+5KPmyjBjCFi3+DbO0L8epACBi+m9i6l3R1D6pllgLPPLal7m8p1blvPUijlnx8A9pYZEDmA4Bbr5K1/gNJQ==
# 启用ConfigFilter
spring.datasource.druid.filter.config.enabled=true
spring.datasource.druid.driver-class-name=com.mysql.jdbc.Driver
####################数据库连接池配置-start#######################
spring.datasource.druid.initial-size=1
spring.datasource.druid.max-active=20
spring.datasource.druid.min-idle=1
spring.datasource.druid.max-wait=50000
spring.datasource.druid.pool-prepared-statements=true
spring.datasource.druid.max-pool-prepared-statement-per-connection-size=20
spring.datasource.druid.validation-query=select 1 from user_info
spring.datasource.druid.validation-query-timeout=30000
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.time-between-eviction-runs-millis=60000
spring.datasource.druid.filters=stat,wall,log4j
####################数据库连接池配置-end#########################
```
## spring-boot初始化配置
使用druid的第一个好处是它帮我们实现了简单的数据库监控，我们能监控sql执行的情况。通过配置druid自定义的filter和servlet，我们能通过浏览器查看数据库相关状态和应用sql执行情况，同时还能帮我们统计应用请求情况哦。
这里我们使用java代码的方式配置druid的filter和servlet，使用注解的配置莫名其妙的未生效让人很捉急。
```
/**
 * @author xk
 * @createTime 2017/9/18 0009 上午 10:47
 * @description
 */
@Configuration
public class DruidConfiguration {

    /**
     * 配置StatView的Servlet
     */
    @Bean
    public ServletRegistrationBean DruidStatViewServlet(){
        ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(new StatViewServlet(),"/druid/*");
        //添加初始化参数：initParams
        //白名单：
        servletRegistrationBean.addInitParameter("allow", "127.0.0.1");
        //IP黑名单 (存在共同时，deny优先于allow) : 如果满足deny的话提示:Sorry, you are not permitted to view this page.
        servletRegistrationBean.addInitParameter("deny", "192.x.x.x");
        //登录查看信息的账号密码.
        servletRegistrationBean.addInitParameter("loginUsername", "admin");
        servletRegistrationBean.addInitParameter("loginPassword", "123456");
        //是否能够重置数据.
        servletRegistrationBean.addInitParameter("resetEnable", "false");
        return servletRegistrationBean;
    }

    /**
     * 配置statView的Filter
     * @return
     */
    @Bean
    public FilterRegistrationBean druidStatFilter(){
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean(new WebStatFilter());

        //添加过滤规则.
        filterRegistrationBean.addUrlPatterns("/*");

        //添加不需要忽略的格式信息.
        filterRegistrationBean.addInitParameter("exclusions", "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        //添加数据库sql监控
        filterRegistrationBean.addInitParameter("profileEnable","true");
        return filterRegistrationBean;
    }
}
```
当然大家别忘了数据源的配置和mybatis的配置，这里我不做介绍。

## 运行结果

应用跑起来，浏览器输入localhost:port/druid/,就会进入登录页，输入账号密码，即可进入首页，如下图：
![druid监控页图](/images/druid/druid_index_view.png)
## 数据库安全配置
使用druid的第二个好处就是实现了密文的数据库账号密码配置的加密,废话不多说，我们来看看怎么对我们的密码进行加密。
cmd到druid的jar包目录下，执行：java -cp druid-1.1.3.jar com.alibaba.druid.filter.config.ConfigTools you_password
![druid密码加密](/images/druid/druid_password.png)
这里取public-key作为解密公钥，password即为加密后的密码。
配置文件里修改配置
```
spring.datasource.druid.password=HT+5KPmyjBjCFi3+DbO0L8epACBi+m9i6l3R1D6pllgLPPLal7m8p1blvPUijlnx8A9pYZEDmA4Bbr5K1/gNJQ==
```
添加公钥配置和启用使用加密方法的配置
```
//生成的公钥
public-key=MFwwDQYJKoZIhvcNAQEBBQADSwAwSAJBAMIQ1SdCmGJlswLkNvh5rpySGPxZXr9bfFei5J4h7/q9XVlqePLcOTVkyQ0je4Gnnp2ZQPlCBsAo5ZPEXJShgRUCAwEAAQ==
//配置 connection-properties，启用加密，配置公钥。
spring.datasource.druid.connection-properties=config.decrypt=true;config.decrypt.key=${public-key}
```
启用configfilter，使加密生效
```
spring.datasource.druid.filter.config.enabled=true
```
详细代码：https://github.com/xkupc/druid-spring-boot
参考内容：https://github.com/alibaba/druid  https://github.com/alibaba/druid/tree/master/druid-spring-boot-starter

