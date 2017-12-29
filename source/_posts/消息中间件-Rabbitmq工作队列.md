---
title: 消息中间件-Rabbitmq工作队列
date: 2017-12-28 10:21:44
tags: [java,Rabbitmq]
categories: Rabbitmq,消息中间件
---
## 前言
上一篇通过简单的例子实现了Rabbitmq的最简单的生产消费队列。这种队列的应用场景有很多，如客户在商城里注册以后，我们需要给这个客户发新手优惠券，发送的优惠券是根据用户注册时提交的一些个人爱好相关的产品的券，是个耗时的操作，注册系统不必等待也不关系是否优惠券推送成功，客户注册成功直接返回就是了，只要将客户注册信息推送给优惠券系统就可以了。这里面就会有一个问题，客户知道注册时有优惠券送的，但是当他注册完的时候，优惠券服务给他推送优惠券的时候突然宕机了，这个时候问题就来了，因为Rabbitmq的机制里，消息一旦被接收，消息就会在队列里被删除，这样这个客户注册的消息就丢失了，即便是优惠券服务重启，这个客户也不会收到优惠券了。针对这种情形，Rabbitmq给出消息确认的方案，即Message acknowledgment。
<!--more-->
## Message acknowledgment
前面我们已经介绍了，Message acknowledgment是使得，当消费者接收到消息之后，Rabbitmq很会等待消费者处理完消息之后，发送一个ack信号，确认已处理完毕之后，将消息从队列里移除，这样我们可以一定程度上确保消息不会在异常中丢失。在Message acknowledgment机制里，如果消息在被消费者处理中，消费者宕机或者tcp连接被关闭，Rabbitmq会认为这个消息没有被处理完，它会将这个消息重新放回队列里，被其他消费者消费掉。Rabbitmq并未设置消息超时时间，这意味着，消息可以被处理很久很久，直到最终返回结果。
要实现ack，我们只需要修改一下消费时的参数autoAck设置为false,取消自动ack,有程序控制，手工ack。我们还是使用上一篇的生产者代码，修改一下消费者代码：
消费者手工ack
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
            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body)
                        throws IOException {
                    String message = new String(body, "UTF-8");
                    try {
                        System.out.println("receive message:" + message);
                    } finally {
                        //手工确认消息，确认当前id为deliveryTag的消息
                        channel.basicAck(envelope.getDeliveryTag(),false);
                    }
                }
            };
            //取消自动ack,等待程序处理结果
            boolean autoAck = false;
            channel.basicConsume(queuenceName, autoAck, consumer);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (TimeoutException e) {
            e.printStackTrace();
        }

    }
}
```
测试：
```java
    @Test
    public void testMq() {
        String queuenceName = "hello_test";
        receivemqService.receiveMq(queuenceName);
        for (int i = 0; i < 10; i++)
            sendmqService.sendMq(queuenceName, "Hello World!" + i);
    }
    
```
测试结果是这样：
```js
Waiting for message....
send message:Hello World!0
receive message:Hello World!0
send message:Hello World!1
receive message:Hello World!1
send message:Hello World!2
receive message:Hello World!2
send message:Hello World!3
receive message:Hello World!3
send message:Hello World!4
receive message:Hello World!4
...
```
这里需要说明的是,当我们把autoAck设置为false的时候，记得一定要手工ack,即 channel.basicAck，不然将会有大量的消息堆积在队列里。
当然这里还有一些其他的方法来ack：
channel.basicReject,它和basicAck有同样的效果，即RabbitMQ都会将消息从队列里移除，不同是basicAck是确认该消息已被成功处理，而basicReject则表示该消息未被处理，但仍需在从队列里移除，这种场景也有很多，比如前言里提到的场景，客户注册消息里缺少一些关键参数导致无法判断该发什么类型的优惠券，这个时候我们可以使用basicReject。
channel.basicNack,这个则直接表明消息未被成功处理，我们可以通过传参，告诉RabbitMQ，这个未被成功处理的消息是否该从队列里移除或者重新发送。这种场景也有许多，还是上面那个场景，给客户推送优惠券时，可能涉及远程服务调用，而调用服务异常，这个时候我们可以使用basicNack,并设置消息重发参数，告诉RabbitMQ这个消息可以重发，从而达到远程调用重试的目的。
## 消息的持久化
通过ack机制，我们能确保当消费者挂掉或者连接断开的时候，我们的消息不至于丢失，但是当RabbiMQ server停止或者异常挂掉，那么还处于队列里未被消费的消息将全部丢失。这个时候我们想到是类似redis缓存的持久化机制，我们可以将消息进行持久化处理。RabbitMQ支持消息的持久化。
首先我们需要在声明队列的时候声明为持久化队列:
```java
boolean durable = true;
channel.queueDeclare(queuenceName, durable, false, false, null);
```
然后再对发送的消息进行持久化处理，只需要在对发布消息时，设置消息的MessageProperties为PERSISTENT_TEXT_PLAIN：
```java
channel.basicPublish("", queuenceName, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes("UTF-8"));
```
## Publisher Confirms
前面通过消费者ack和消息持久化，我们能在一定程度上保证我们的消息不会丢失，这里依旧不能绝对的保证我们的消息不会丢失，当RabbitMQ接受了一个消息，还没发送给消费者的时候，它需要仍然需要一段时间将消息写入硬盘，而RabbitMQ server崩溃就可能发生在这段时间。当然在RabbitMQ持久化实现机制中，和redis一样，并没有直接将消息同步的写入硬盘，而只是放入缓存中。这里我们介绍RabbitMQ提供的一个更强大的保障机制-Publisher Confirms。
在讲Publisher Confirms之前，我们先来解释几个acknowledgment的几个细节性的问题：
```java
1.批量确认。我们解释一下 channel.basicAck(envelope.getDeliveryTag(),false)的作用。DeliveryTag是amqp-client为这个消息的生成的唯一单调递增的整型标识，当后面这个参数为false时，RabbitMQ只确认id为DeliveryTag的这个消息，若为true,则将channel里未被确认的消息全部确认掉。也就是说RabbitMQ支持批量确认。不过这个功能好像没啥用。
2.通道限制设置。我们知道消息中间件实现了一种异步流程，这其中消息发送到队列，队列发送到消费者，消费者处理消息，本质上都是异步的，这也意味着，某一时刻队列里有很多个消息待消费，对消费者来说当它在处理一个耗时的任务时，也在不停的接收着队列推过来的消息。这里就会出现大量消息堆积在消费者那里，而造成消费者端缓存过载。于是RabbitMQ为为我们提供一种机制去限制往消费者推送消息的个数。
我们可以通过channel.basicQos设置channel里未确认消息的个数，当未确认消息个数达到设置的个数的时候，RabbitMQ将停止向消费者端推送消息。官网上说预设值在100到300之间的效果是极好的。
3.重复确认与重复推送。前面我们已经知道，当消费者未对消息进行确认的时候，RabbitMQ会将消息重新推送，这个消息会带上redeliver=true的标签，这个时候我们要注意的是需要从业务逻辑层面上判断消息是否已被处理，按上面的例子说，我们要避免出现多次给新客推送优惠券的情况。同时当消费者多次对同一个DeliveryTag的消息确认时，RabbitMQ会抛出异常-类似PRECONDITION_FAILED - unknown delivery tag 100。
4.事务机制。要做到消息不丢失，我们还可以向数据库事务一样处理消息，我们将生产者提交消息到RabbitMQ的过程做成事务，一旦提交失败，我们便可以通过捕获异常进行回滚，amqp-client提供了channel.txSelect()用来将当前channel设置成事务，channel.txCommit()用来提交这个事务，提交成功则消息一旦发送到了RabbitMQ，channel.txRollback()使用于回滚事务。
```
了解这些以后，我们再来解释Publisher Confirms。
上面我们知道使用事务，我们能确保消息一定发送给RabbitMQ，但是带来了性能问题。于是RabbitMQ为我们提供了一个思路：Publisher Confirms模式。从名字就可以看出，Publisher Confirms意味着，RabbitMQ将向生产者发送是否消息被成功的通过exchange分配到所有的队列的确认。同样的它使用basic.ack,这是模仿的消费者向RabbitMQ发送确认。那么和消费者确认类似，每一个发送到该channel的消息都被赋予一个自增的从1开始的序列号。当消息被RabbitMq正确分配之后，它会想生产者发送一个确认并包含那个唯一的序列号。同样的也通过multiple参数来支持批量确认，来表明当前序列号以前的包括当前序列号的消息一全部被分配。当RabbitMq不能成功分配消息时，它将使用nack来向生产者表明，它无法处理消息，这就意味着生产者需要重新发送消息。
使用Confirms的好处是因为对于生产者来说确认时异步的，他不必像事务模式那样等待消息事务提交，而是继续发送下一个消息，提高了RabbitMQ的吞吐量。同时事务模式和Confirms不能兼容，一个队列不可能同时支持两种模式。
对于需持久化的消息，那么确认消息将在消息被持久化到磁盘之后发送，由于RabbitMQ消息持久化到磁盘是批量的，所以强烈建议生产者确认消息也是用批量的方式。
和事务机制类似：RabbitMQ使用channel.confirmSelect()进入Confirms模式，如果没有设置no-wait标志的话，broker会返回confirm.select-ok表示同意发送者将当前channel信道设置为confirm模式。如果调用了channel.confirmSelect方法，默认情况下是直接将no-wait设置成false的，也就是默认情况下RabbitMQ是必须回传confirm.select-ok的。然后通过channel.waitForConfirms()确认消息是否被成功分配。
## 负载均衡
在生产消费模式里，我们考虑这样的一个场景，两个消费者同时消费一个队列里的消息，可能出现分配给一个消费者消息处理特别耗时，而分配给另一个消费者的消息却很简单。这样就会造成一个消费者会特别繁忙，另一个却无事可做。针对这样的场景，我们可用上面提到的通道限制设置，通过在消费者端使用channel.basicQos设置prefetchCount参数为1，即告诉RabbiMQ,在消费者没有确认上一个消息的时候，不允许推送新的消息。通过这种方式我们间接的实现了消息推送的负载均衡。
![RabbitMQ负载均衡](images/rabbitMQ/prefetch-count.png)
当所有的消费者端都忙的时候，队列将会被填满，这个时候就得当心了，或者你该考虑换别的策略了。
## 参考资料
http://www.rabbitmq.com/tutorials/tutorial-two-java.html