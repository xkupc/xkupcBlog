---
title: 分布式ID生成方法
date: 2017-10-12 16:01:44
tags: [spring-boot,zookeeper,java]
categories: person idea
---
## 简介
最近小编在为雪花算法在应用部署集群时候，需要修改配置文件workerId，然后重新打包部署而苦恼。在网上看到了美团的Leaf-snowflake的方案<a href="https://tech.meituan.com/MT_Leaf.html" target="_blank">Leaf——美团点评分布式ID生成系统</a>，觉得很不错，可惜没有开源，于是自己尝试着按照其思路写一个简单的版本的Leaf-snowflake实现来生成分布式的全局ID.
## 思路
关于雪花算法，我在这里就不做过多的介绍了，由41位的时间戳，10位的workerId和12位的序列号组成一个64位（首位不用）的long型数。
![snowflake原理](/images/snowflake/snowflake.png)
在一些优化版本里，将十位的workerId拆分为五位的workerId和五位的datacenterId，workerId通过系统配置文件配置，datacenterId可以是当前运行线程的id，或者服务器ip取hash生成。
而序列号则在一毫秒里从0递增，一毫秒能产生的最大序列号2047，即一毫秒可以产生的Id为2047个。
通过配置文件配置workerId就会出现小编说的问题。一个应用在多台服务器上部署，那么为了是ID不冲突，workerId须配置不一样，这样一个应用要打多次包，很痛苦。
所以Leaf-snowflake方案使用Zookeeper持久顺序节点的特性自动对snowflake节点配置wokerId。这样就不用手动配置workerId了。
<!--more-->
现在我们思路很清晰，通过snowflake生成id之前，我们通过zookeeper获取workerId，为了对zookeeper弱依赖，提高性能，我们对workerId进行本地缓存，也就是说我们只在第一次请求的时候通过zookeeper初始化workerId。接下来就只剩一个问题了，如何唯一标识一个节点，即确保一个应用集群里的多个节点分配到的workerId不同。
小编在这里使用了ip加端口的方法去标识唯一节点。主要的流程如下:
![workerId生成流程](/images/snowflake/workerId.png)
每一个应用通过应用名和环境配置注册一个父节点，而应用集群里的多个节点则通过ip和端口注册为多个子节点。这样多个子节点的workerId就随注册的先后顺序而递增。
## 方案
对于zookeeper的连接和使用采用CuratorFramework，pom依赖：
```js
    <dependency>
        <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.11.1</version>
                <exclusions>
                    <exclusion>
                        <artifactId>log4j</artifactId>
                        <groupId>log4j</groupId>
                    </exclusion>
                </exclusions>
    </dependency>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>2.11.1</version>
    </dependency>
```
zookeeper连接实体：
```java
public class ZookeeperProfile {
    /**
     * 应用的ip
     */
    private String ipAddress;
    /**
     * 应用的端口
     */
    private String serverPort;
    /**
     * 配置的根目录
     */
    private String rootNode = "id_sequence";

    /**
     * 环境设置, 如dev, test. prod
     */
    private String environment;
    /**
     * zookeeper的连接地址; 如127.0.0.1:2181;
     * 多个时用逗号分开，如: 192.168.7.52:2181, 192.168.7.53:2181
     */
    private String connectString;

    /**
     * 序列号的空间， 一般使用当前的application名。
     * 一个空间分配的id是唯一的
     */
    private String sequenceSpaceName;

    /**
     * 重试的策略
     */
    private RetryPolicy retryPolicy;
    ...
}
```
获取workerId主要方法：
```java
 @Override
    public Long getWorkId(String name, String ipAndPort) throws Exception {
        if (Strings.isNullOrEmpty(name)) {
            throw new WorkIdException("workId name is null");
        }

        if (Strings.isNullOrEmpty(ipAndPort)) {
            throw new WorkIdException("workId ipAndPort is null");
        }
        String zookeeperIdNodePath = ZKPaths.makePath(zookeeperProfile.getBaseNode(), name);
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(50, 29, 500);
        RetryPolicy lock_RetryPolicy = new ExponentialBackoffRetry(50, 10, 500);
        String lockPath = zookeeperIdNodePath + "_$lock";
        PromotedToLock.Builder builder = PromotedToLock.builder().lockPath(lockPath).timeout(100, TimeUnit.MILLISECONDS).retryPolicy(lock_RetryPolicy);
        DistributedAtomicLong dal = new DistributedAtomicLong(client, zookeeperIdNodePath, retryPolicy, builder.build());
        String zookeeperIdNodechildPath = ZKPaths.makePath(zookeeperIdNodePath, ipAndPort);
        if (null == client.checkExists().forPath(zookeeperIdNodechildPath)) {
            //不存在当前子节点,父节点自增，并将值写入子节点
            AtomicValue<Long> value = dal.increment();
            if (value.succeeded()) {
                lockPath = zookeeperIdNodechildPath + "_$lock";
                builder = PromotedToLock.builder().lockPath(lockPath).timeout(100, TimeUnit.MILLISECONDS).retryPolicy(lock_RetryPolicy);
                dal = new DistributedAtomicLong(client, zookeeperIdNodechildPath, retryPolicy, builder.build());
                AtomicValue<Long> childValue = dal.trySet(value.postValue());
                if (childValue.succeeded()) {
                    return childValue.postValue();
                } else {
                    throw new WorkIdException("child DistributedAtomicLong inclement fail.");
                }
            } else {
                throw new WorkIdException("DistributedAtomicLong inclement fail.");
            }
        }
        lockPath = zookeeperIdNodechildPath + "_$lock";
        builder = PromotedToLock.builder().lockPath(lockPath).timeout(100, TimeUnit.MILLISECONDS).retryPolicy(lock_RetryPolicy);
        dal = new DistributedAtomicLong(client, zookeeperIdNodechildPath, retryPolicy, builder.build());
        AtomicValue<Long> value = dal.get();
        if (value.succeeded()) {
            return value.postValue();
        } else {
            throw new WorkIdException("child DistributedAtomicLong get fail.");
        }
    }
```