# 消息通道

## 简介

## 消息通道接口

消息通道接口的定义:

```java
boolean send(Message message);
boolean send(Message message, long timeout)
```

它包含两个接口方法, 定义了消息通道有发送消息的行为. 第一个方法会等待消息成功发出位置, 第二个方法在指定的时间内, 如果消息没有发送成功, 就抛出异常. timeout 的值可以是负数, 0, 或正整数. 如果 timeout 为负数, 线程会一直等待, 知道消息发送成功为止, 如果是0, 发送操作将会立即返回, 而不管是否发送成功, 如果为正整数, 发送线程会等待 timeout 毫秒, 如果期间发送操作没有返回, 抛出异常.

> **send** 方法的返回值布尔类型说明了消息是否发送成功.

这就是消息通道的基本定义.

### 接收消息

Spring Integration 框架定义了两个接口类分别表示两种投递模式: **PollableChannel** 和 SubscribableChannel.

接收消息有两种方式, 下面一一列举

#### 点对点模式

下面是点对点通道的定义

```java
public interface PollableChannel extends MessageChannel {
    // 接受者线程一直等待
    Message<?> receive();
    // 如果 timeout 时间内没有消息, 接受者线程退出
    Message<?> receive(long timeout);
}
```

> 为了节约CPU时间, 建议使用第二种, 但是业务代码需要做额外的处理.

框架提供的实现包括: **QueueChannel**, **PriorityChannel**,和 **RendezvousChannel**

#### 点对点例子

```java
public class QueueChannelTest {
    private ApplicationContext ctx = null;
    private MessageChannel qChannel = null;
    public QueueChannelTest() {
        ctx = new ClassPathXmlApplicationContext("channels-beans.xml");
        qChannel = ctx.getBean("q-channel", MessageChannel.class);
    }
    public void receive() {
        // This method receives a message, however it blocks
        // indefinitely until it finds a message
        // Message m = ((QueueChannel) qChannel).receive();
        // This method receives a message, however it exists
        // within the 10 seconds even if doesn't find a message
        Message m = ((QueueChannel) qChannel).receive(10000);
        System.out.println("Payload: " + m.getPayload());
    }
}
```

#### 发布/订阅模式

下面是发布订阅模式的接口定义

```java
public interface SubscribableChannel extends MessageChannel {
    // 订阅
    boolean subscribe(MessageHandler handler);
    // 退订
    boolean unsubscribe(MessageHandler handler);
}

public interface MessageHandler{
    // this method is invoked when a fresh message appears on the channel
    void handleMessage(Message<?> message) throws MessagingException;
}
```

#### 发布/定于例子

```java
public class ChannelTest {
    private ApplicationContext ctx = null;
    private MessageChannel pubSubChannel = null;
    public ChannelTest() {
        ctx = new ClassPathXmlApplicationContext("channels-beans.xml");
        pubSubChannel = ctx.getBean("pubsub-channel", MessageChannel.class);
    }
    public void subscribe() {
        ((PublishSubscribeChannel)pubSubChannel).subscribe(new TradeMessageHandler());
    }
    class TradeMessageHandler implements MessageHandler {
        public void handleMessage(Message<?> message) throws MessagingException {
            System.out.println("Handling Message:" + message);
        }
    }
}
```

### 队列通道

队列通道的特点是, 消息是已先进先出(FIFO)的顺序进出通道.

### 优先级通道

优先级通道的特点是消息是以优先级顺序确定消息出通道的顺序, 而不管消息进入通道的顺序.

### 独占通道

消息通道中有且仅能存在一个消息, 通道两端都是阻塞的方式. 两端一直等待, 直到有消息为止.

### 发布订阅通道

可以有多个订阅者, 进入通道中的每一个消息会广播给所有订阅者.

### 点对点通道

点对点通道可以有1..N个订阅者/消费者, 但是通道中的消息一次只会发给其中一个订阅者,
在有多个订阅者的情况下, 消息是一轮询的方式发给每一个订阅者的. 这种通道适合于做多任务处理.

### 执行器通道

### 空通道

## 结语
