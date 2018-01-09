---
title: 消息中间件-Rabbitmq消息主题
date: 2018-01-02 15:36:40
tags: [java,Rabbitmq]
categories: Rabbitmq,消息中间件
---
## 前言
上一篇我们通过direct交换机，对消息进行过滤后推送给指定队列，匹配的规则还是不够灵活，因为对于一个消费者来说，它所处理的消息可能存在多重的标准，那我们注册用户的场景来说，推送给大数据服务的注册信息要有实名认证信息，对于优惠券来说，推送的优惠券包含实体店优惠券，于是需要注册信息里包含用户的地址信息，这样显然direct已经满足不了我们的需求。今天我们来学习一个满足需求的交换机类型-topic。
## Topic Exchange
和direct一样，topic和队列通过bind key进行绑定进行绑定，不同的是key不再是随意的单词，它必须是一个通过点连接的多个单词列表，如：stock.usd.nyse，quick.orange.rabbit，最长为255个字节。
同时bind key支持模糊匹配，topic通过*和#支持模糊匹配。
<!--more-->
![RabbitMQ-direct](images/rabbitMQ/topic-exchange.png)
需要注意的是：
*只能替代一个单词，而#可以替代零个或者多个单词。
这就意味着lazy.orange.male.rabbit既不匹配 *.orange.*也不会匹配到*.*.rabbit，二回匹配到lazy.#，当一个队列通过#绑定时，则和fanout类似，它将接收到所有消息，当bind key中不包含*和#时，则和direct一样工作。
声明一个topic exchange:
```java
 channel.exchangeDeclare("registerUser",BuiltinExchangeType.TOPIC);
```
发布一个消息：
```java
 channel.basicPublish("registerUser", "hadIdCardNo.hadAddress.userInfo", null, message.getBytes("UTF-8"));
```
消费者绑定队列：
```java
 String queueName = channel.queueDeclare().getQueue();
 //通过bind key hadIdcardNo.#绑定到registerUser,即接收有实名信息的客户注册信息
 channel.queueBind(queueName,"hadIdCardNo.#","hadIdcardNo");
```
## 源码
生产者：
```java
public void sendTopicMq(String exchange,String key,String message){
        try {
            //创建连接
            Connection connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(exchange,BuiltinExchangeType.TOPIC);
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
  public void receiveTopicMq(String exchange, int i,String... keys) {
        Connection connection = null;
        try {
            connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.exchangeDeclare(exchange, BuiltinExchangeType.TOPIC);
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
    public void receiveMq1(String exchange, String... keys) {
        receiveTopicMq(exchange,1,keys);
    }

    @Override
    public void receiveMq2(String exchange, String... keys) {
        receiveTopicMq(exchange,2,keys);
    }
```
测试用例：
```java
    @Test
    public void testTopicMq() throws InterruptedException{
        String exchange = "registerUser";
        receivemqService.receiveMq1(exchange,"hadIdcardNo.#");
        receivemqService.receiveMq2(exchange,"*.hadAddress.*");
        sendmqService.sendTopicMq(exchange, "hadIdcardNo.hadAddress.userInfo","Hello World!");
        sendmqService.sendTopicMq(exchange, "noIdcardNo.hadAddress.userInfo","Hello World haha!");
    }
```
## 参考资料
http://www.rabbitmq.com/tutorials/tutorial-five-java.html