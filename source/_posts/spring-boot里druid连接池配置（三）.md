---
title: spring-boot里druid连接池配置（三）
date: 2017-09-21 09:25:23
tags: [druid,spring-boot]
categories: java中间件
---
### 简介
上一篇我们知道druid实现监控的大致原理，今天我们学习一下，druid数据库连接池的实现。
### 数据库连接池配置
首先我们来看一下数据库数据源的初始化配置：
```
    /**
     * 主数据源
     */

    @Bean(name = "primaryDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.druid")
    public DataSource primaryDataSource() {
        return DruidDataSourceBuilder.create().build();
    }
```
获取spring.datasource.druid前缀的相关配置属性，初始化数据源。我们来看看druid为适配springboot开发的druid-spring-boot-starter都做了哪些事。
![druid-spring-boot-starter代码结构](/images/druid/druid-spring-boot-starter.png)
从图中我们看到，这个包很简单，主要做了两个事，一个是通过springboot的数据源初始化机制初始化自己的数据源，通过DruidDataSourceWrapper兼容springboot的数据源配置，第二个是通过springboot的注解机制，通过配置初始化自定义的系列Filter。
是的，我们也可以通过配置文件配置我们上篇文章提到的statViewServlet和webStatFilter。
```
//初始化statViewServlet
spring.datasource.druid.stat-view-servlet.enabled
//初始化webStatFilter
spring.datasource.druid.web-stat-filter.enabled
```
<!--more-->
看这里，这个类完成了所有配置的初始化。
![druid初始化配置](images/druid/druid_starter.png)
DruidStatProperties这个类加载了statViewServlet和webStatFilter的配置属性，然后分别通过DruidStatViewServletConfiguration和DruidWebStatFilterConfiguration以spring注册bean的方式完成注册，与我们直接通过代码的方式注册一样。
DataSourceProperties这个则加载了数据源的配置属性，然后通过DataSourceAutoConfiguration完成数据源初始化。
DruidDataSourceWrapper则完成了数据库连接池相关的配置，我们看到它继承了DruidDataSource。
### 数据库连接池初始化
druid通过springboot加载完配置后，连接池是如何初始化的呢？
```
   public DruidDataSource(boolean fairLock){
        super(fairLock);

        configFromPropety(System.getProperties());
    }
```
看这里，有一个公平锁的概念，小编马上先谷歌了一下公平锁的概念，公平锁是为了解决cpu调度线程的时候随机选择线程，无法保证线程先到先得，从而导致某些优先级低的线程永远也无法获得cpu资源的问题。公平锁能够保证线程按照时间的先后顺序执行。既然要按顺序执行，那么就要维护一个有序的队列，这个开销就有点大了，果然公平是不容易实现的。druid默认是非公平锁。
