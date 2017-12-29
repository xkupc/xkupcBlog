---
title: 消息中间件-Rabbitmq简介
date: 2017-12-27 15:39:15
tags: [java,Rabbitmq]
categories: Rabbitmq,消息中间件
---
## 前言
最近看了redis的发布订阅的实现，觉得还是蛮鸡肋的，频道的定义和缓存数据需要程序去做绑定，消息的发送也是需要手工发送，而不是键值对更新时自动触发。于是研究了一下消息中间件的发布订阅方式，因为公司使用的Rabbitmq,顺道了解一下这个消息中间件。
## Rabbitmq简介
对生产者消费者模型熟悉的同学知道生产者消费者的概念
生产者（producer）：发送消息的程序
消费者（consumer）：接收消息的程序
队列（queue ）：存储并传递消息的容器
了解这些之后，我们可以抽象的描述一下Rabbitmq的作用了，生产者将一个消息对到队列里，消费者监听这个队列后获取到这个消息进行处理。在实际的业务运用中，可能我们会这样使用，当一个用户在统一用户中心登陆成功之后，用户中心异步推送用户登录成功的消息给广告子系统，广告子系统接收到消息后，从消息中获取中户登陆信息，根据信息给用户发送热点资讯。
<!--more-->
## Rabbitmq安装
安装的问题，我就不在这里说了。照着教程就可以了。反正window下需要安装Erlang.
## Hello World
今天我们运用Rabbitmq实现简单的生产消费模式，依赖引入：
```js
    <dependency>
        <groupId>com.rabbitmq</groupId>
        <artifactId>amqp-client</artifactId>
        <version>5.1.1</version>
    </dependency>
```
配置连接RabbitMQ连接工厂：
```java
@Configuration
@EnableConfigurationProperties(RabbitMQConf.class)
public class RabbitmqConfig {

    @Autowired
    private RabbitMQConf rabbitMQConf;

    @Bean
    public ConnectionFactory createConnectionFactory() {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(rabbitMQConf.getHost());
        factory.setVirtualHost(rabbitMQConf.getVirtualHost());
        factory.setPort(rabbitMQConf.getPort());
        factory.setUsername(rabbitMQConf.getUserName());
        factory.setPassword(rabbitMQConf.getPassword());
        return factory;
    }
}
```
sendService:
```java
@Service
public class SendmqServiceImpl implements SendmqService {

    @Autowired
    ConnectionFactory connectionFactory;

    @Override
    public void sendMq(String queuenceName, String message) {
        try {
            //创建连接
            Connection connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            //声明一个队列,非持久的，非排他的，非自动删除的队列
            channel.queueDeclare(queuenceName, false, false, false, null);
            channel.basicPublish("", queuenceName, null, message.getBytes("UTF-8"));
            System.out.println("send message:" + message);
            channel.close();
            connection.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```
receiveService:
```java
@Service
public class ReceivemqServiceImpl implements ReceivemqService {
    @Autowired
    ConnectionFactory connectionFactory;

    @Override
    public void receiveMq(String queuenceName) {
        Connection connection = null;
        try {
            connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(queuenceName, false, false, false, null);
            System.out.println("Waiting for message....");
            //接收消息缓冲处理
            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
                        throws IOException {
                    String message = new String(body, "UTF-8");
                    System.out.println("receive message:" + message);
                }
            };
            channel.basicConsume(queuenceName, true, consumer);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }

    }
}

```
测试类：
```java
    @Autowired
    SendmqService sendmqService;
    @Autowired
    ReceivemqService receivemqService;

    @Test
    public void testMq(){
        String queuenceName = "hello_test";
        receivemqService.receiveMq(queuenceName);
        sendmqService.sendMq(queuenceName,"Hello World!");
    }
```
这样一个简单的生产消费队列就建立了
## 参考资料
http://www.rabbitmq.com/tutorials/tutorial-one-java.html