# 适配器

## 简介

使用消息作为媒介集成外部系统是一个有挑战的任务. 需要考虑各种问题, 包括连接的复杂性, 以及不同系统产生的消息格式的传输问题.

## 文件适配器

## FTP 适配器

## JMS 适配器

## JDBC 适配器

Inbound JDBC Adapters

其责任是读取数据集并转换为消息, **inbound-channel-adapter** 用于创建端点.

```xml
<jdbc:inbound-channel-adapter
    channel="resultset-channel"
    data-source="mySqlDatasource"
    query="SELECT * FROM ACCOUNTS A where A.STATUS='NEW' and POLLED='N'">
    <int:poller fixed-rate="1000"/>
</jdbc:inbound-channel-adapter>
```

该适配器的用途是: 连接到数据库并执行SQL查询, 把查询结果返回的记录集合转换为消息, 并发送到 **resultset-channel** 通道.

整个结果集被转换为单个消息, 消息的内容是一个 List 类型的集合. 该适配器还指定了一个轮询器(poller),
该轮询器以1000毫秒为脉搏执行一次SQL查询,转换为消息,并发送到 **resultset-channel** 通道.

有的时候, 我们需要避免获得相同的记录集. 通过把已经获取过的记录标记为 **POLLED** 可以避免每次获取重复的记录.
这种情况一般用于数据处理. 每次获取要处理的数据, 并标记为 **已处理(POLLED)**, 只拉取未处理的数据.

```xml
<jdbc:inbound-channel-adapter
    channel="resultset-channel"
    data-source="mySqlDatasource"
    query="SELECT * FROM ACCOUNTS A where A.STATUS='NEW' and POLLED='N'"
    update="UPDATE ACCONTS set POLLED='Y' where ACCOUNT_ID in (:ACCOUNT_ID)">
    <int:poller fixed-rate="1000" />
</jdbc:inbound-channel-adapter>
```

其中, 更新语句的参数 **:ACCOUNT_ID** 是从查询语返回的结果集字段.

### Outbound JDBC Adapters

该类适配器的用途是: 监听通道, 并把通道传递过来的消息中包含的值, 构造成SQL查询, 并执行.

```xml
<jdbc:outbound-channel-adapter
    channel="trades-persistence-channel"
    data-source="mySqlDatasource"
    query="insert into TRADE t(ID,ACCOUNT,INSTRUMENT) values(:payload[TRADE_ID], :payload[TRADE_ACCOUNT], :payload[TRADE_INSTRUMENT])">
</jdbc:outbound-channel-adapter>
```

其中 **query** 属性指定的SQL语句中的参数 **:payload** 要求消息的内容是一个Map. 还可也把 **Headers** 作为Map来访问

```xml
qyery="insert into TRADE t(ID,ACCOUNT,INSTRUMENT,EXPIRY) values(:payload[TRADE_ID], :payload[TRADE_ACCOUNT], :payload[TRADE_INSTRUMENT], :headers[EXPIRY])">
```

Map 消息的创建

```java
public Message<Map<String, Object>> createTradeMessage(){
    Map<String, Object> tradeMap = new HashMap<String, Object>();
    tradeMap.put("ID", "1929303d");
    tradeMap.put("ACCOUNT", "ACC12345");
    //..
    // Create a Msg using MessageBuilder
    Message<Map<String, Object>> tradeMsg = MessageBuilder.withPayload(tradeMap).build();
    return tradeMsg;
}
```

## JDBC 适配器示例

- [Github 代码库](https://github.com/developerworks/spring-integration-jdbc-adapter)
