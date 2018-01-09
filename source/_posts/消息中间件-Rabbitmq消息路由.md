---
title: 消息中间件-Rabbitmq消息路由
date: 2018-01-02 14:01:57
tags: [java,Rabbitmq]
categories: Rabbitmq,消息中间件
---
## 前言
上一篇我们解决了一个消息通过fanout Exchange同时分发到多个队列进行处理，今天我们来介绍另一种类型的Exchange-direct。我们还是拿注册用户的场景来说，前面我们谈到注册成功的用户信息将同时推送给优惠券服务和大数据服务进行处理。现在有这样一个需求，注册信息中有用户实名信息的大数据服务才进行处理和分析，也就说实际上推给大数据服务的消息进行了过滤处理，当然我们完全可以通过程序来处理，接收到消息，校验是否有实名信息，没有的话直接丢弃，今天还是来讲一下将这个校验逻辑交给RabbitMQ来处理。
## Direct exchange
前面我们说fanout这种类型的交换机只是单纯的和队列进行绑定，无脑的往队列里推送它接收到的消息。而direct的实现则要智能的多，direct允许和队列进行绑定时，指定一个或者多个参数bind key，只有当消息匹配这些个参数中的一个时，消息才会被推送到该绑定的队列。请注意是匹配多个参数中的一个时，就会被推送。
<!--more-->
![RabbitMQ-direct](images/rabbitMQ/direct-exchange.png)
上图中，我们可以看到类型为direct的交换机，绑定了两个队列，一个队列通过bing key :orange绑定，另一个则绑定了black和green两个key，这就意味着当消息匹配orange时，消息将被推送到上面的队列，当消息匹配到black或者green时，消息将被推送到下面的队列。
当两个队列都通过同样的一个key进行绑定时，direct就和fanout一样了，它会将消息同时推给这两个队列。
我们首先来声明一个direct exchange：
```java
 //创建一个名为registerUser的类型为direct的交换机
 channel.exchangeDeclare("registerUser",BuiltinExchangeType.DIRECT);
```
```java
 //发布一个bind key 为hadIdCardNo的消息
 channel.basicPublish("registerUser", "hadIdCardNo", null, message.getBytes("UTF-8"));
```
消费者队列绑定：
```java
 String queueName = channel.queueDeclare().getQueue();
 //通过bind key hadIdcardNo绑定到registerUser.
 channel.queueBind(queueName,"registerUser","hadIdcardNo");
```
## 源码
生产者：
```java
    public void sendDirectMq(String exchange,String key,String message){
        try {
            //创建连接
            Connection connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(exchange,BuiltinExchangeType.DIRECT);
            channel.basicPublish(exchange, key, null, message.getBytes("UTF-8"));
            System.out.println("send message:" + message);
            channel.close();
            connection.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
消费者：
```java
public void receiveDirectMq(String exchange, int i,String... keys) {
        Connection connection = null;
        try {
            connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(exchange, BuiltinExchangeType.DIRECT);
            String queueName = channel.queueDeclare().getQueue();
            for (String key : keys) {
                channel.queueBind(queueName, exchange, key);
            }
            System.out.println("Waiting for message....");
            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
                        throws IOException {
                    String message = new String(body, "UTF-8");
                    System.out.println("receive message:" + i + message);
                }
            };
            channel.basicConsume(queueName, true, consumer);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

     @Override
    public void receiveMq1(String exchange, String... key) {
        receiveDirectMq(exchange, 1,key);
    }

    @Override
    public void receiveMq2(String exchange, String... key) {
        receiveDirectMq(exchange,2, key);
    }
```
测试用例：
```java
    @Test
    public void testDirectMq() throws InterruptedException{
        String exchange = "registerUser";
        receivemqService.receiveMq1(exchange,"hadIdcardNo");
        receivemqService.receiveMq2(exchange,new String[]{"hadIdcardNo","noIdcardNo"});
        sendmqService.sendDirectMq(exchange, "hadIdcardNo","Hello World!");
        sendmqService.sendDirectMq(exchange, "noIdcardNo","Hello World!");
    }
```
## 参考资料
http://www.rabbitmq.com/tutorials/tutorial-four-java.html