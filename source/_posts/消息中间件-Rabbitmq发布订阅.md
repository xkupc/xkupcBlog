---
title: 消息中间件-Rabbitmq发布订阅
date: 2017-12-28 20:30:53
tags: [java,Rabbitmq]
categories: Rabbitmq,消息中间件
---
## 前言
上一篇工作队列我们讲到了，当一个消费者接收到消息并成功处理，返回确认以后，RabbitMQ将消息从队列里移除，消息不回再被另外的消费者锁接收处理，但是我们会有这样的一些场景，我们接着上一篇的场景来说，如果客户注册以后，我们不仅需要根据客户注册信息给客户推送合适的优惠券，还需要将客户的注册信息同步到大数据平台进行保存并分析，这就意味着一个客户注册消息可以被多个消费者消费。这种场景就是我们很熟悉的发布订阅的模式了。今天我们来研究一下RabbitMQ的发布订阅模式。
<!--more-->
## Exchanges
前面的几篇中我们讲到消息的发送和接收都是通过队列实现的，这里面我们忽略了一个重要的东西，或者说RabbitMQ给我们提供了默认的实现，让我们不必去在意这个东西，今天是时候来讲一下RabbitMQ完整的消息模型了。
在RabbitMQ的消息模型里，事实上，生产者不会直接将消息发送到一个队列里，可以这样说，生产者是不知道自己的消息是否会被发送到任何队列里的。生产者只会把消息发送到一个交换机上，和网络模型里的交换机一样，它接收生产者发送的消息，然后匹配路由规则，将消息分发到正确的队列里。这就意味着交换机需要知道如何处理它接收到的消息，是将消息推送到一个确定的队列耗时多个队列，还是丢弃。这些规则都取决于交换机的类型，RabbitMQ为我们提供了多种类型以适配多种规则，主要有direct, topic, headers 和 fanout。今天先来了解一下fanout这种类型的特点。在前面的几篇文章里，我们没有定义一个交换机，是因为提供了默认的交换机，我们发送消息时，设置交换机为空串，则RabbitMQ为我们自动适配到默认的交换器上。
## fanout与临时队列
fanout这样类型的交换机很简单，它只是简单的将他所接收到的消息广播到所有它知道的队列里去。我们可以简单的通过这样的代码创建一个交换机：
```java
//创建一个名为registerUser的fanout类型的交换机
channel.exchangeDeclare("registerUser", BuiltinExchangeType.FANOUT);
```
发布消息
```java
//发布消息到确定的队列里，如queuenceName为空，则发布到所有绑定了exchange的队列里
channel.basicPublish("registerUser", queuenceName, null, message.getBytes("UTF-8"));
```
对于我们前言里提到的场景，注册用户信息需要推送到两个不同的服务里，所以在发送消息时，不需指定队列名：
```java
//发布到非持久化的队列里
channel.basicPublish("registerUser", "", null, message.getBytes("UTF-8"));
```
对于两个不同的服务，我们可以各自的服务里自定义一个队列，然后分别与registerUser交换机绑定，但RabbitMQ为我们提供了一种更简单的直接的方式-临时队列。临时队列提供了一种机制：当我们每次连接到Rabbit server的时候，我们都需要一个新的空的名字随机生成的队列，当我们断开连接的时候，队列能自动被删除。我们可以通过代码创建这样一个临时队列：
```java
String queueName = channel.queueDeclare().getQueue();
```
接着我们只需要将这个队列和registerUser交换机绑定：
```java
channel.queueBind(queueName,exchange,"");
```
绑定关系图：
![RabbitMQ消息模型](images/rabbitMQ/message-model.png)
## 源码
生产者：
```java
public void sendMq(String exchange, String queuenceName, String message) {
        try {
            //创建连接
            Connection connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(exchange,BuiltinExchangeType.FANOUT);
            channel.basicPublish(exchange, queuenceName, null, message.getBytes("UTF-8"));
            System.out.println("send message:" + message);
            channel.close();
            connection.close();
        } catch (Exception e) {
        }

    }
```
消费者：
```java
public void receiveMq(String exchange, String queueName，int i) {
        Connection connection = null;
        try {
            connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(exchange, BuiltinExchangeType.FANOUT);
            if (Strings.isNullOrEmpty(queueName)) {
                queueName = channel.queueDeclare().getQueue();
            }
            channel.queueBind(queueName, exchange, "");
            System.out.println("Waiting for message....");
            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
                        throws IOException {
                    String message = new String(body, "UTF-8");
                    System.out.println("receive message:"+ i + message);
                }
            };
            //自动确认
            channel.basicConsume(queueName, true, consumer);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public void receiveMq1(String exchange, String queueName) {
        receiveMq(exchange,queueName,1);
    }

    @Override
    public void receiveMq2(String exchange, String queueName) {
        receiveMq(exchange,queueName,2);
    }
```
测试用例：
```java
    @Test
    public void testPublicMq() throws InterruptedException{
        String exchange = "registerUser";
        receivemqService.receiveMq1(exchange,"");
        receivemqService.receiveMq2(exchange,"");
        sendmqService.sendMq(exchange, "","Hello World!");
    }

```
## 参考资料
http://www.rabbitmq.com/tutorials/tutorial-three-java.html