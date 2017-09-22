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
未完待续。。。容我把注解写完。