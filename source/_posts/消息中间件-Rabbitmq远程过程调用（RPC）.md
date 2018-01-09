---
title: 消息中间件-Rabbitmq远程过程调用（RPC）
date: 2018-01-02 16:48:43
tags: [java,Rabbitmq]
categories: Rabbitmq,消息中间件
---
## 前言
<a href=http://xkupc.top/%E6%B6%88%E6%81%AF%E4%B8%AD%E9%97%B4%E4%BB%B6-Rabbitmq%E5%B7%A5%E4%BD%9C%E9%98%9F%E5%88%97.html target="_blank">消息中间件第二篇</a>介绍了通过RabbitMQ的工作队列，我们将耗时的任务分发给多个消费者去处理，现在我们有这样一个需求，生产者需要获取消费者处理后的结果，还是用户注册的场景，注册信息被分发给大数据服务后，大数据服务分析处理用户信息后，计算出该用户的基础信用积分，并需将结果回传给账户中心保存。是的，这就是我们熟知的远程过程调用模式。如果我们不借助RabbitMQ的RPC模式，我们也可以将大数据服务也当生产者，当计算出用户信息积分后，将结果丢到一个队列供账户中心去消费处理保存。今天来研究一下RabbitMQ的RPC是怎么玩的。
<!--more-->
## RabbitMQ RPC
直接看RabbitMQ的RPC模型：
![RabbitMQ-direct](images/rabbitMQ/rpc-model.png)
我们可以看到，实际上，RabbitMQ还是使用两个队列来实现RPC,只不过它将两个队列进行了绑定。我们来看看它是怎么绑定两个队列的。
首先我们来看如何声明一个RPC的队列：
```java
String callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());
```
终于用到了BasicProperties了。BasicProperties提供了许多消息属性的配置方法。我们使用replyTo声明一个队列的回调队列，即实现两个队列的绑定。
另一个问题是当生产者将消息推到RPC队列进行处理后，它从回调队列里收到了返回结果消息，如何区分出这个结果是它刚刚发出的那个消息的返回结果呢？即我们如何将请求和结果进行匹配，如果我们不使用RabbitMQ的RPC模式，我们可能使用消息内容里的一个唯一键，比如用户id，在RabbitMQ RPC模式里，通过的设置BasicProperties里的correlationId方法，我们可以设置唯一的一个请求。
```java
AMQP.BasicProperties replyProps = new AMQP.BasicProperties
                            .Builder()
                            .correlationId(properties.getCorrelationId())
                            .build();

```
## 源码
生产者：
```java
public void sendRPCMq(String rpcQuenceName, String callBackQuenceName, String s1) {
        try {
            String corrId = UUID.randomUUID().toString();
            Connection connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder().correlationId(corrId).replyTo(callBackQuenceName).build();
            channel.queueDeclare(rpcQuenceName, false, false, false, null);
            channel.basicPublish("",rpcQuenceName,properties,s1.getBytes("UTF-8"));
            System.out.println("send message:" + s1);
            //新建阻塞队列，等待返回结果
            BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<String>(1);
            channel.basicConsume(callBackQuenceName, new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    if (properties.getCorrelationId().equals(corrId)) {
                        blockingQueue.offer(new String(body, "UTF-8"));
                    }
                }
            });
            System.out.println("result:" + blockingQueue.take());
            channel.close();
            connection.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```
消费者：
```java
 public void receiveRPCMq(String rpcQueueName,String callBackQueueName){
        Connection connection = null;
        try {
            connection = connectionFactory.newConnection();
            Channel channel = connection.createChannel();
            channel.queueDeclare(rpcQueueName, false, false, false, null);
            System.out.println("Waiting for message....");
            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
                        throws IOException {
                    String message = new String(body, "UTF-8");
                    AMQP.BasicProperties replyProps = null;
                    try {
                        System.out.println("receive message:" + message);
                        replyProps = new AMQP.BasicProperties.Builder().correlationId(properties.getCorrelationId()).build();
                    } finally {
                        channel.basicPublish("",properties.getReplyTo(),replyProps,"Hello world too".getBytes("UTF-8"));
                        //手工确认消息，确认当前的deliveryTag的消息
                        channel.basicAck(envelope.getDeliveryTag(), false);
                    }
                }
            };
            //取消自动ack,等待程序处理结果
            boolean autoAck = false;
            channel.basicConsume(rpcQueueName, autoAck, consumer);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }
    }
```
测试用例：
```java
    @Test
    public void testRPCMq() throws InterruptedException{
        String rpcQueueName = "RPC_QUEUE";
        String callBackName = "REPLY_QUEUE";
        receivemqService.receiveRPCMq(rpcQueueName,callBackName);
        sendmqService.sendRPCMq(rpcQueueName, callBackName,"Hello World!");
    }
```
这里有个比较尴尬的问题是rpcQueueName没法使用RabbitMQ提供的默认队列的方式创建，小编我提前新建了一个永久的队列。
## 参考资料
http://www.rabbitmq.com/tutorials/tutorial-six-java.html