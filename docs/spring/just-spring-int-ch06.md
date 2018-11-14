# 流组件

## 过滤器

> 过滤器的特点: **要么接受, 要么丢弃**

```xml
<!-- 独立声明, 可重用 -->
<bean id="newTradeFilter" class="com.madhusudhan.jsi.flow.ex1.NewTradeFilter" />

<int:filter
    input-channel="in-channel"
    output-channel="out-channel"
    ref="newTradeFilter"
    method="isNewTrade">
</int:filter>

<int:filter input-channel="in-channel" output-channel="out-channel">
    <!-- 过滤器私有Bean, 只能该过滤器使用 -->
    <bean class="com.madhusudhan.jsi.flow.filters.NewTradeFilter" />
</int:filter>
```

> 消息通过 **input-channel** 进入, 通过 **testFilter** 过滤器, 如果过滤器接收该消息,
> 则过滤器方法 **isNewTrade** 返回 true, 并且消息通过 **output-channel** 发出去. 否则丢弃该消息.

```java
public class MyFilter {
    public boolean isOK(Message<?> message) {
        Trade t = (Trade)message.getPayload();
        return (t.getStatus().equalsIgnoreCase("new"));
    }
}
```

> 如果 **MyFilter** 过滤器类只有一个方法, 那么: **<int:filter/>** 标签的 **method** 属性可以省略.
> 如果包含多个方法, **method** 则必须要指定. 可以把不同的过滤规则写在一个过滤器类中.

### 使用 MessageSelector

和上述自定义过滤器不同, Spring Integration 框架提供了 MessageSelector, 用于有选择的过滤消息.

```java
public class CancelTradeFilter implements MessageSelector {
    public boolean accept(Message<?> message) {
        Trade t = (Trade)message.getPayload();
        return (t.getStatus().equalsIgnoreCase("cancel"));
    }
}
```

### 使用注解的方式定义过滤器

```java
public class AnnotatedNewTradeFilter {
    @Filter
    public boolean isTradeCancelled(Message<?> message) {
        Trade t = (Trade)message.getPayload();
        return (t.getStatus().equalsIgnoreCase("cancel"));
    }
}
```

### 关于丢弃的消息

消息丢弃了就丢弃了, 如果需要知道哪些消息被丢弃(问题分析), Spring Integration提供了处理丢弃消息的两种方式:

- **抛出异常**
- **转发到其他通道**

```xml
<!-- 抛出异常 -->
<int:filter
    input-channel="all-trades-in-channel"
    output-channel="cancel-trades-out-channel"
    ref="cancelTradeFilter"
    throw-exception-on-rejection="true">
</int:filter>

<!-- 转发到其他通道 -->
<int:filter
    input-channel="all-trades-in-channel"
    output-channel="cancel-trades-out-channel"
    ref="cancelTradeFilter"
    discard-channel="non-cancel-trades-hospital-channel">
</int:filter>
```

## 路由器

> 根据条件, 把消息分发到多个目标. 路由器从通道中获取消息, 依据路由规则(路由表), 对消息进行重新投递, 它的工作机制和网络路由器的工作机制是相同的.
> 路由器会依据消息头, 和消息内容进行判断, 应该把消息重新投递到哪个目标通道中去.

### PayloadTypeRouter

> 依据消息的内容进行路由

### HeaderValueRouter

> 依据消息头的内容进行路由

### Custom Routers

> 自定义路由

### Recipient List Router

> 接收人列表类型的路由器, 会把消息通知给列表中的所有目标. 相当于**广播**

```xml
<int:recipient-list-router input-channel="all-in-channel">
    <int:recipient channel="persistor-channel"/>
    <int:recipient channel="trades-channel"/>
    <int:recipient channel="audit-channel"/>
</int:recipient-list-router>
```

### Unqualified Messages

> 这种消息没有匹配任何路由规则, 被投递到**默认输出通道**

```xml
<int:payload-type-router
    input-channel="all-in-channel" default-output-channel="non-matches-channel">
    ...
</int:payload-type-router>
```

### 使用注解定义路由器

```java
public class AnnotatedBigTradeRouter {
    @Router
    public String bigTrade(Message<Trade> message) {
        Trade t = message.getPayload();
        if (t.getQuantity() > 10000)
            return "big-trades-channel";
        return "trades-stdout"; // 目标通道
    }
}
```

XML装配

```xml
<context:component-scan
    base-package="com.madhusudhan.jsi.flow.router" />
<int:router
    id="annonatedRouter" input-channel="in-channel"
    default-output-channel="no-matches-channel"
    ref="annotatedBigTradeRouter">
</int:router>
```

> 路由器的路由方法实现应该返回一个字符串, 返回的字符串就是目标通道的名字.

## 分离器

> 分离器的作用是把一个消息切分为多个消息片段, 进行并发处理, 把处理结果再聚合为一个消息. 它的概念和大数据处理的 **MapReduce** 的作用相似.

## 聚合器

聚合器的工作是组装多个消息分片为一个完整的消息. 它的工作机制类似于TCP协议的网络包的分片和组装. 在所有分片到达前, 消息需要存储, 这就引入了一个问题: 消息存储需要占用存储资源, 当所有消息到达并完成组装后, 需要释放消息临时存储占用的存储空间.

因为所有的消息分片到达后, 聚合器才开始工作, 在开始一个复杂的工作前, 下面通过一个简单的例子来展示聚合器是如何工作的, 例如, 我们有一个处理交易订单的聚合器, 它的作用是把多个订单作为整体进行处理, 一个现实的使用场景是微信的提现, 微信商户以每周为一个提现周期, 自动打款到设置的银行账号上.

这里我们就可以通过聚合器来批量的处理订单的对账问题. 微信提供单一订单号的查询, 当然假设有100个订单需要对账, 那么我们就需要调用100次查询接口, 当所有查询结果(分片)达到后, 开始执行聚合.

```java
public class OrderAggregator {
    public Order aggregateOrder(List<Order> childOrders) {
        ...
    }
}
```

聚合器的声明如下:

```xml
<!-- 聚合器声明 -->

<int:aggregator
    input-channel="in-channel"
    output-channel="aggregate-channel"
    ref="orderAggregator"
    method="aggregateOrder">
</int:aggregator>

<!-- 聚合器实现类 -->

<bean id="orderAggregator" class="com.example.aggregator.OrderAggregator" />

<!-- 分离器 -->

<int:splitter
    input-channel="in-channel"
    ref="customSplitter"
    output-channel="out-channel">
</int:splitter>

<!-- 自定义分离器实现类 -->

<bean id="customSplitter" class="com.example.splitter.OrderSplitter" />
```

## 重排序器

消息的传输是无序的, 要把消息分片组装为单个消息, 需要对象消息分片进行重新排序, 还原原始的消息分片顺序.

### 策略

> 策略定义了消息分片的处理和释放逻辑和算法.

### 关联策略

该策略检查每个消息的 **CORRELATION_ID** 属性, 并把 **CORRELATION_ID** 属性值相同的消息放到相同的详细桶中等待聚合.

框架提供了一个 **HeaderAttributeCorrelationStrategy**, 也可以实现自定义策略, 自定义策略可以以POJO的方式实现, 也可以实现 **CorrelationStrategy** 接口, 前者没有依赖框架本身, 一直性强, 后缀依赖Spring Integration. 因此如果要编写适用于不同框架的业务代码, 建议使用POJO的方式.

下面以一个实际的例子来说明:

实现自定义策略

```java
public class MyCorrelationStrategy implements CorrelationStrategy {
    public Object getCorrelationKey(Message<?> message) {
        // implement your own correlation key here
        // return ..
    }
}
```

组装

```xml
<int:aggregator
    input-channel="all-trades-out-channel"
    output-channel="aggregate-channel"
    ref="orderAggregator"
    method="aggregateOrder"
    correlation-strategy="myCorrelationStrategy">
</int:aggregator>

<bean id="myCorrelationStrategy" class="com.example.aggregator.MyCorrelationStrategy" />
```

### 释放策略

> 消息存储是用临时存储方式, 聚合器在完成消息的组装后, 需要释放这部分临时占用的存储空间.

基于分片数量的释放策略

框架提供的默认释放策略是 **SequenceSizeReleaseStrategy**, 它实现了 **ReleaseStrategy** 接口. 它以分片的数量为依据来判断释放的条件. 例如, 一个消息被切分为10个分片, 当消息分片数量达到10收, 触发消息释放.

你也可以实现自己的自定义策略

```xml
<int:aggregator
    input-channel="in-channel"
    output-channel="aggregate-channel"
    ref="tradeAggregator"
    method="aggregateTrade"
    correlation-strategy="myCorrelationStrategy"
    correlation-strategy-method="fetchCorrelationKey"
    release-strategy="myReleaseStrategy">
</int:aggregator>

<bean id="myReleaseStrategy" class="com.example.aggregator.MyReleaseStrategy" />
```

如果要事先自定义释放策略, 策略方法实现的参数签名为: 接收一个 `List<Object>` 参数, 返回 **boolean**. 如下:

```java
public class MyReleaseStrategy {
    public boolean canRelease(List<Object> messages) {
        if (messages.size() == 10) {
            return true;
        }
        return false;
    }
}
```

自定义策略需要同时制定 **release-strategy** and **release-strategy-method** 属性:

```xml
<int:aggregator
    input-channel="in-channel"
    ...
    release-strategy="myReleaseStrategy"
    release-strategy-method="canRelease">
</int:aggregator>
```

### 消息存储

被拆分为片段的消息一直存储在聚合器当中, 直到消息的最后一个分片达到聚合器, 并完成组装.
在所有消息片段到达之前, 这些消息片段是不能释放的. 这意味着, 聚合器必须有一个存储消息的地方, 这就是 **MessageStore**.

Spring Integration 提供了一个选项来设置聚合器的消息存储.

> 聚合器的消息存储支持两种策略: **内存** 和 **外部存储**
> 内存是默认的存储方式, 使用 **java.util.Map** 作为数据结构存储消息分片. 外部存储目前支持下面几种:
> - **JDBC**
> - **Redis**
> - **MongoDB**
> - **Gemfire**

下面的XML配置使用了JDBC作为外部存储. **message-store** 属性, 引用一个 **JdbcMessageStore** 实例.

```xml
<int:aggregator
    input-channel="all-trades-out-channel"
    output-channel="aggregate-channel"
    message-store="mySqlStore">
</int:aggregator>

<bean id="mySqlStore" class="org.springframework.integration.jdbc.JdbcMessageStore">
    <property name="dataSource" ref="mySqlDataSource"/>
</bean>
```

数据库存储的DDL语句位于 **spring-integration-jdbc** 工件的 **org.springframework.integration.jdbc** 包之中

![clipboard.png](/img/bVbjq2Y)

### 元数据存储
