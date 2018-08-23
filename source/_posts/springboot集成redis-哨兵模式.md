---
title: 'springboot集成redis-哨兵模式'
date: 2018-08-23 19:20:48
tags: [spring,spring-boot,redis]
categories: java framework
---
## 前言
我们都知道redis的单机模式，但是单机模式在复杂的生产环境下是面临极大的风险的，所以我们往往选择通过多台redis组成主备的模式，当master宕掉以后，slave将成为新的master继续工作，对于主备的自动切换，就是redis提供的一种模式-哨兵模式(redis-sentinel),通过至少三台redis服务组成一个选举团队，从配置的redis服务组里选举出master,当master宕机时，重新进行选举并进行主从切换，这个选举团队同时也可以是担任业务服务的主从redis服务。在主备未发生切换的时候，实际上应用连接的还是一种单机模式，所以当运维给我们redis的哨兵模式的配置时，我们同样可以当作单机模式配置到spring-boot里，但是一旦发生主从切换，那么我们的配置就会发生问题。正确的哨兵模式配置应该是让应用连接那个选举团队(redis-sentinel),让redis-sentinel告诉应用哪个是主，该去连接哪个，发生主从切换后，应用通过心跳，从redis-sentinel获取新的master连接。spring已经为我们提供了redis-sentinel模式的配置服务，下面是相关配置。
## pom依赖
```java
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```
小白我使用的1.5.6的版本
## 配置
```yml
  redis:
      #host: 10.14.205.160
      #port: 6379
      password: 123456
      # 连接超时时间（毫秒）
      timeout: 20000
      pool:
        # 连接池中的最大空闲连接
        max-idle: 100
        # 连接池中的最小空闲连接
        min-idle: 10
        # 连接池最大连接数（使用负值表示没有限制）
        max-active: 100
        # 连接池最大阻塞等待时间（使用负值表示没有限制）
        max-wait: -1
      sentinel:
        master: master_1
        nodes: 127.0.0.1:6410,127.0.0.1:6410,127.0.0.1:6410
```
这里没有指定master，启动时sentinel会返回master信息。
## 初始化config
```java
@Configuration
public class RedisConfig {
    
    //spring会自动将redis配置装载到这个Properties里，所以就不用一个一个通过@Value装载了
    //当然通过@ConfigurationProperties(prefix = "spring.redis.pool")同样可以避免一个一个@Value
    @Autowired
    RedisProperties redisProperties;

    private static final Logger logger = LoggerFactory.getLogger(RedisConfig.class);

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.pool")
    public JedisPoolConfig jedisPoolConfig() {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxIdle(redisProperties.getPool().getMaxIdle());
        config.setMinIdle(redisProperties.getPool().getMinIdle());
        config.setMaxWaitMillis(redisProperties.getPool().getMaxWait());
        config.setTestOnBorrow(false);
        config.setTestWhileIdle(false);
        return config;
    }

    @Bean
    public RedisSentinelConfiguration sentinelConfiguration() {
        RedisSentinelConfiguration sentinelConfiguration = new RedisSentinelConfiguration();
        sentinelConfiguration.setMaster(redisProperties.getSentinel().getMaster());
        sentinelConfiguration.setSentinels(getRedisNodeList(redisProperties.getSentinel().getNodes()));
        return sentinelConfiguration;
    }

    @Bean
    public JedisConnectionFactory redisConnectionFactory() {
        JedisConnectionFactory factory = new JedisConnectionFactory(sentinelConfiguration());
        factory.setPoolConfig(jedisPoolConfig());
        factory.setTimeout(redisProperties.getTimeout());
        factory.setPassword(redisProperties.getPassword());
        return factory;
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> template = new RedisTemplate<String, String>();
        template.setConnectionFactory(redisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new StringRedisSerializer());
        return template;
    }

    @Bean
    public CacheService cacheService() {
        return new RedisCacheServiceImpl();
    }
     /**
     * 这里我没有找到直接传入string的方法，所以这里解析了一下sentinel的节点信息
     * @param nodes
     * @return
     */
    private Set<RedisNode> getRedisNodeList(String nodes) {
        Set<RedisNode> sentinelSet = new LinkedHashSet();
        Set<String> sentinelStr = StringUtils.commaDelimitedListToSet(nodes);
        Iterator sentinels = sentinelStr.iterator();
        while (sentinels.hasNext()) {
            String[] args = StringUtils.split((String) sentinels.next(), ":");
            sentinelSet.add(new RedisNode(args[0], Integer.valueOf(args[1])));
        }
        return sentinelSet;
    }
}
```