# 基础

## 简介

> **基本构建块**: 构建块代表着业务, IT或者架构能力的一个组件,并且可以与其他构建块组合在一起来对各种架构和解决方案进行交付.

Spring 集成框架构建在一些基本构建块之上: 消息, 通道和端点. 消息是数据的容器, 通道是持有这些消息的地址. 端点为连接消费和发布消息的组件.

后续的章节会详细讨论这些基本构建块. 这一节仅仅是简单的触碰. 本章讨论消息, 其他(通道, 端点等) 将会在后续的章节讨论.

## 消息

消息是携带信息的对象, 它由一端构造, 通常是消息应用的生产端. 并在另一端消费和销毁, 通常是消费/订阅端.
消息作为携带业务数据的对象, 比如一个新 Account(账号), 或 Trade(交易,贸易) 信息.
消息的发布者/生产者构造这个对象, 并发布到通道, 连接到相同通道的消费/订阅者接收这些消息, 并重建领域对象,执行业务处理.

### 剖析消息

消息包含两个部分, 消息头(Header)和载荷(Payload). 消息的例子:

- 书信
- 电子邮件
- 快递

他们都有相同的特特性, 对于书信来说, 信封上的发件人和收件人的邮编和地址,收件人发件人的姓名就是消息头,
信件内容就是消息携带的数据, 电子邮件类似, 只是消息头的属性不同, 快递也大致相同, 消息头现在以及很少在使用邮编了,
一般是地址, 手机号码, 收件人姓名, 你买的商品就是携带的数据.

```java
public interface Message<T> {
    T getPayLoad();
    MessageHeaders getHeaders();
}
```

### 消息的一般实现

框架提供默认的 `GenericMessage` 实现, 提供有两个构造函数实例化一个消息. 经管如此,
强烈建议使用 `MessageBuilder` 来构造消息, 它提供了更加方便和安全的方式.

```java
// 创建消息体
Account a = new Account();
// 创建一个Map存储消息头
Map<String, Object> accountProperties = new HashMap<String, Object>();
// 设置消息头属性
accountProperties.put("ACCOUNT_EXPIRY","NEVER");
// 使用 MessageBuilder 类创建消息并设置消息头
Message<Account> m = MessageBuilder.withPayload(a)
    .setHeader("ACCOUNT_EXPIRY", "NEVER")
    .build();
```

MessageBuilder 工具类用于创建消息, 操作消息以及消息头, 它遵循**构建器**(**Builder**)模式.

## 消息通道

消息表示携带数据的容器, 通道表示消息要发送的位置. 通道封装了位置信息(**Where**), 因此只需要把消息扔给通道, 通道知道如何把消息发送给对端.

下图说明了一个消息通道, 在Spring Integration中, 通道以 `MessageChannel` 接口表示. 图中圆角矩形表示消息的端点, 管道形状表示消息通道.

![clipboard.png](https://segmentfault.com/img/bVbiQ45)

### 声明通道

Spring Integration 提供了声明性的模型来创建通道, 因此不必使用Java类来实例化通道. 声明通道简单直接, 如下代码片段所示:

一个JDBC适配器通道例子

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:int="http://www.springframework.org/schema/integration"
       xmlns:int-jdbc="http://www.springframework.org/schema/integration/jdbc"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/integration
        http://www.springframework.org/schema/integration/spring-integration.xsd
        http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- 扫描包 -->
    <context:component-scan base-package="com.example.adapterjdbc"/>
    <!-- Default poller -->
    <int:poller default="true" fixed-rate="1000">
        <int:transactional/>
    </int:poller>
    <int:channel id="channelResultSet"/>
    <int:channel id="channelUpdate"/>
    <!-- JDBC查询通道 -->
    <int-jdbc:inbound-channel-adapter id="jdbcQuery" data-source="dataSource" channel="channelResultSet" query="SELECT * FROM test"/>
    <!-- JDBC更新通道 -->
    <int-jdbc:outbound-channel-adapter id="jdbcUpdate" data-source="dataSource" channel="channelUpdate" query="INSERT INTO test VALUES(NULL, 'name')"/>
</beans>
```

因为通道和其他消息组件定义在集成XML名称空间(**int:**前缀), 需要保证XML文件包含上述名称空间.

框架还提供了一些具体的实现, 比如 `QueueChannel`(队列通道), `PriorityChannel`(优先级通道), `RendezvousChannel`(独占通道).

## 端点

端点是一个基础组件, 用于从输入通道读取消息, 并投递到输出通道. 框架提供了一些直接可用的端点, 比如过滤器, 路由器, 转换器, 分离器等.
它还提供了用于链接外部系统(比如, JMS, FTP, JDBC等)的适配器.

### 服务激活器端点示例

服务激活器端点是一个通用端点, 用于当一个消息到达输入通道时调用 Bean 上的一个方法.  该端点的声明方式如下:

```xml
<int:service-activator
    input-channel="positions-channel"
    ref="newPositionProcessor"
    method="processNewPosition">
</int:service-activator>

<bean id="newPositionProcessor" class="com.madhusudhan.jsi.basics.NewPositionProcessor" />
```

> `<int:service-activator/>` 通过属性 `ref` 引用 `id` 为 `newPositionProcessor` 的 Bean

**service-activator** 端点从 **positions-channel** 通道获取消息并调用 **processNewPosition** 方法.

### 例子

现在以及知道了框架的各种基本构建块, 下面将会展示一个例子. Spring Integration框架关注于业务逻辑的处理,
而让框架本身来帮你处理乏味的其他任务, 可以使用适配器 **inbound-channel-adapter** 从输入JMS队列中获取消息, 适配器的声明给定如下:

```xml
<int:channel id="trades-channel"/>
<jms:inbound-channel-adapter
    id="tradesJmsAdapter"
    connection-factory="connectionFactory"
    destination="tradesQueue"
    channel="trades-channel">
    <int:poller fixed-rate="1000" />
</jms:inbound-channel-adapter>

<bean id="tradesQueue"
    class="org.apache.activemq.command.ActiveMQQueue">
    <constructor-arg value="TRADES_QUEUE" />
</bean>

<bean name="connectionFactory"
    class="org.apache.activemq.ActiveMQConnectionFactory">
    <property name="brokerURL">
        <value>tcp://localhost:61616</value>
    </property>
</bean>
```

**inbound-channel-adapter** 元素的声明帮你处理了所有JMS相关的模板代码, 它从 **positionsQueue** 获取消息, 转换消息为内部格式, 最后发布消息到内部通道 **trades-channel**.

**jms:inbound-channel-adapter** 适配器的 **connection-factory** 属性提供了端点的连接信息, 在此例中, 是运行在 **localhost:61616** 的 ActiveMQ 消息队列服务器.

到此, 对于每一个收到的 **Trade** 消息, 都会调用 **NewTradeProcessor** 的 **processNewTrade** 方法.

接下来我们在XML配置中进行组装:

```xml
<int:service-activator
    input-channel="trades-channel"
    ref="newTradeProcessor"
    method="processNewTrade">
</int:service-activator>

<bean id="newTradeProcessor" class="com.madhusudhan.jsi.basics.NewTradeProcessor" />
```

然后实现这个Java类:

```java
public class NewTradeProcessor {
    public void processNewTrade(Trade t){
        // Process your trade here.
        System.out.println("Message received:"+m.getPayload().toString());
    }
}
```

最后, 编写一个测试来运行它:

```java
public class NewTradeProcessorTest {
    private ApplicationContext ctx = null;
    private MessageChannel channel = null;
    // Constructor which instantiates the endpoints
    public NewTradeProcessorTest() {
        ctx = new ClassPathXmlApplicationContext("basics-example-beans.xml");
        channel = ctx.getBean("trades-channel", MessageChannel.class);
    }
    private Trade createNewTrade() {
        Trade t = new Trade();
        t.setId("1234");
        ...
        return t;
    }
    private void sendTrade() {
        Trade trade = createNewTrade();
        Message<Trade> tradeMsg = MessageBuilder.withPayload(trade).build();
        channel.send(tradeMsg, 10000);
        System.out.println("Trade Message published.");
    }
    public static void main(String[] args) {
        NewTradeProcessorTest test = new NewTradeProcessorTest();
        test.sendTrade();
    }
}
```

## 结语

本章给出了Spring Integration框架的概要, 介绍了框架提供的消息, 通道, 端点等组件. 本章作为一个介绍性的章节, 通过示例提供一些基础性的概念.