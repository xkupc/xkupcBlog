---
title: spring-boot与RabbitMQ整合
date: 2018-01-04 11:08:22
tags: [Rabbitmq,spring-boot]
categories: 消息中间件,Rabbitmq
---
## 前言
前面几篇我们了解了RabbitMQ的消息模型和一些工作机制，今天通过spring-boot对其进行一下整合。我们知道了RabbitMQ通过Exchange对消息进行分发，Exchange有fanout,topic,direct和默认等几种类型。下面来分别对这几种类型进行spring-boot的整合。关于springboot与RabbitMQ的整合，在网上可以找到很多的demo,单独跑是没有问题的，但是如果想要把它抽象到框架上的东西，供业务层调用就比较难了，毕竟是要做架构师的男人，就该勇敢的尝试一下。
## RabbitMQ配置
AMQP依赖：
```js
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
        <version>1.4.5.RELEASE</version>
    </dependency>
```
<!--more-->
配置文件配置：
```java
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=123456
//指定broker
spring.rabbitmq.virtual-host=/test
//publisher-confirm
spring.rabbitmq.publisher-confirms=true
spring.rabbitmq.publisher-returns=true
```
### 初始化配置
通过@Configuration初始化RabbitMQ相关配置。
```java
@Configuration
public class RabbitConfig {
    @Value("${spring.rabbitmq.host}")
    private String host;

    @Value("${spring.rabbitmq.port}")
    private String port;

    @Value("${spring.rabbitmq.username}")
    private String username;

    @Value("${spring.rabbitmq.password}")
    private String password;

    @Value("${spring.rabbitmq.virtual-host}")
    private String vhost;

    @Value("${spring.rabbitmq.publisher-confirms}")
    private boolean publisherConfirms;

    @Value("$spring.rabbitmq.publisher-returns}")
    private boolean publisherReturns;

    @Bean
    ConnectionFactory connectionFactory() {
        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setAddresses(host + ":" + port);
        cachingConnectionFactory.setUsername(username);
        cachingConnectionFactory.setPassword(password);
        cachingConnectionFactory.setVirtualHost(vhost);
        cachingConnectionFactory.setPublisherConfirms(publisherConfirms);
        cachingConnectionFactory.setPublisherReturns(publisherReturns);
        return cachingConnectionFactory;
    }   
    @Bean
    RabbitTemplate rabbitTemplate() {
        RabbitTemplate template = new RabbitTemplate(connectionFactory());
        return template;
    }
    @Bean
    RabbitAdmin rabbitAdmin() {
        RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory());
        return rabbitAdmin;
    }
    @Bean
    public MessageConverter messageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
监听器容器工厂基础配置,所有的业务层消费者配置继承该配置，采用手工确认的方式，由业务层控制消息销毁：
@EnableRabbit
public abstract class RabbitListenerConfig {
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactoryWithManual(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        //最少消费者数量
        factory.setConcurrentConsumers(3);
        //最大消费数量
        factory.setMaxConcurrentConsumers(10);
        //队列里最大的未确认消息个数
        factory.setPrefetchCount(100);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }
}
```
## 基础客户端配置
生产者：
```java
public class Sender {
    private static Logger logger = LoggerFactory.getLogger(Sender.class);

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送默认Exchange消息
     *
     * @param queueName
     * @param message
     */
    public void send(String queueName, String message) {
        send("", queueName, message);
    }

    /**
     * 发送json格式消息
     *
     * @param exchangeName
     * @param routingKey
     * @param message
     */
    public void send(String exchangeName, String routingKey, String message) {
        //生成消息唯一标识，回调确认使用
        CorrelationData correlationId = new CorrelationData(UUID.randomUUID().toString());
        //设置消息体属性参数messageProperties
        MessagePostProcessor messagePostProcessor = messagePostProcessor(correlationId.getId());
        logger.info("消息routing key：{}，消息id：{}，消息内容：{}", routingKey, correlationId.getId(), message);
        rabbitTemplate.convertAndSend(exchangeName, routingKey, message, messagePostProcessor, correlationId);
    }

    MessagePostProcessor messagePostProcessor(String uuid) {
        return message ->
        {
            MessageProperties messageProperties = message.getMessageProperties();
            if (null != messageProperties) {
                messageProperties.setHeader("id", uuid);
                messageProperties.setHeader("timestamp", System.currentTimeMillis() / 1000);
            }
            return message;
        };
    }
}
生产者基础配置类，所有业务层生产者配置继承该类，
public abstract class PublisherConfig {
    @Bean
    public Sender sender() {
        return new Sender();
    }
}
消费者处理类，所有业务层消费者继承该类，实现具体的消息处理方法handleMessage(message)，并确认消息：
public abstract class ReceiverHandler implements ChannelAwareMessageListener {

    private static final Logger logger = LoggerFactory.getLogger(ReceiverHandler.class);

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        logger.info("receive message:{}", message);
        MqHandlerResult result = MqHandlerResult.REDO;
        try {
            result = handleMessage(message);
        } catch (Exception e) {
            logger.error("message handle error:{}", e);
            //抛出自定义异常,供控制层捕获并统计
            throw new RabbitMQException();
        } finally {
            switch (result) {
                case SUCCESS:
                    logger.info("message success handle:{}", message);
                    //只确认当前消息
                    channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
                    break;
                case REDO:
                    logger.info("message should redo:{}", message);
                    //是否重新推送标示
                    boolean requeue = true;
                    channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, requeue);
                    break;
                case REJECT:
                    logger.info("message should reject:{}", message);
                    channel.basicReject(message.getMessageProperties().getDeliveryTag(), false);
                    break;
                default:
                    logger.info("message handle return error");
                    throw new RabbitMQException();
            }
        }
    }
    public abstract MqHandlerResult handleMessage(Message message);
}

```
## 简单工作队列（默认Exchange）
默认的Exchange实际上是一种点对点的通信方式，一个消息只会被一个客户端消费，Exchange默认为空，所以我们直接在配置里初始化队列。
```java
public class SimpleWorkQueueConfig{

    private static MqQueueName[] mqQueueName;
    //业务层通过该方法设置队列名
    public static void setMqQueueName(MqQueueName[] mqQueueName) {
        SimpleWorkQueueConfig.mqQueueName = mqQueueName;
    }

    @Autowired
    RabbitAdmin rabbitAdmin;
    //通过工厂类初始化队列
    @Bean
    public QueueFactory queueFactory() {
        return new QueueFactory(rabbitAdmin, mqQueueName);
    }
}
```
### 简单队列生产者配置：
```java
public class SimplePublisherConfig extends PublisherConfig {
}
```
### 简单队列消费者配置：
```java
public class SimpleReceiverConfig extends RabbitListenerConfig {
}
```
### 业务层实现：
初始化相关配置
```java
@AutoConfigureAfter(value = RabbitConfig.class)
@Configuration
@Import({SimpleWorkQueueConfig.class, SimpleReceiverConfig.class, SimplePublisherConfig.class})
public class SimpleTestMqConfig {

    public SimpleTestMqConfig() {
        TestMqQueueName[] testMqQueueNames = {TestMqQueueName.TEST_MQ_QUEUE_NAME};
        SimpleWorkQueueConfig.setMqQueueName(testMqQueueNames);
    }
}
```
生产者：
```java
@Component
public class TestPublisher extends PublisherConfirmHandler{
    private static Logger logger = LoggerFactory.getLogger(TestPublisher.class);
    @Autowired
    Sender sender;
    @Autowired
    RabbitTemplate rabbitTemplate;

    public void testSimpleMq(String message) {
        rabbitTemplate.setConfirmCallback(this);
        sender.send(TestMqQueueName.TEST_MQ_QUEUE_NAME.getQueueName(), message);
        logger.info("send message:{}" , message);
    }

    @Override
    public void ackFail() {
       logger.warn("message did not had dispatch");
    }

    @Override
    public void ackSuccess() {
       logger.info("message had dispatch");
    }
}
```
消费者：
```java
@Component
public class TestReceiver extends ReceiverHandler {
    private static Logger logger = LoggerFactory.getLogger(TestReceiver.class);
    //监听指定队列，并指定监听容器工厂
    @RabbitListener(queues = "#{T(com.xkupc.framework.test.rabbitmq.common.TestMqQueueName).TEST_MQ_QUEUE_NAME.queueName}",
            containerFactory = "rabbitListenerContainerFactoryWithManual")
    public void doMessage(Message message, @Header(org.springframework.amqp.support.AmqpHeaders.CHANNEL) Channel channel) {
        try {
            onMessage(message, channel);
        } catch (Exception e) {
            logger.error("handle message error:{}", e);
        }
    }
    //实际业务处理
    @Override
    public MqHandlerResult handleMessage(Message message) {
        logger.info("receive the message:" + message);
        //redo something
        return MqHandlerResult.SUCCESS;
    }
}
```
测试类：
```java
    @Autowired
    TestPublisher testPublisher;

    @Test
    public void testMQ() {
        testPublisher.testSimpleMq("hello world!");
    }
```
这里只是简单介绍，源码可以到我的github上看看。https://github.com/xkupc/framework-parent