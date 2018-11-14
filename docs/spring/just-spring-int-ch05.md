# 转换器

## 简介

Not all applications understand the data they consume. Sometimes the messages need to be transformed before they can be consumed to achieve a business purpose. For example, a producer uses a Java Object as its payload to produce a message, while a consumer is interested in non-Java Object types like plain XML or name-value pairs.

To help the producer and consumer communicate, transformers are used to transform Java Object to non-Java Objects. The Spring Integration framework provides the trans- former components that do exactly what is required. This chapter looks in detail at the transformation endpoints provided by the framework.

First, we will discuss the transformers like Object-to-String or Object-to-Map trans- formers, which come out of the box from the Spring Integration framework. We then discuss the ways to create our own transformers if the built-in ones are inadequate.

## 内置转换器

Framework provides a couple of built-in transformers so you don’t have to create them for simple cases such as converting an Object to String, Map, or JSON formats. The integration namespace supports these transformers.

### 字符串转换器

Using the Object to String transformer is easy—all we have to do is define one in our bean’s XML using the **object-to-string-transformer** element. The following snippet shows the definition:

```xml
<int:object-to-string-transformer
    input-channel="in-channel"
    output-channel="stdout">
</int:object-to-string-transformer>
<int-stream:stdout-channel-adapter id="stdout"/>
```

So, any POJOs (Plain Old Java Objects) appearing in the trades-in-channel will au- tomatically be converted to string without intervention by custom transformers. Note that we do not provide reference to any transformer in the above config definition. In fact, the object-to-string-transformer element will not take the ref attribute. The payload at the receiver’s end will always be a toString() of the POJO. In the above example, the payload is written to the stdout using the stdout-channel-adapter. So, make sure your published POJO has overridden the toString() method, or else you will see gibberish as your String payload (such as com.madhusud han.jsi.domain.Trade@309ff0a8).

### Map 转换器

If you need to convert the POJO to a name-value pair of Map, you can use the Object to Map transformer. It is represented by the object-to-map-transformer element that takes the payload from the input channel and emits a name-value paired Map object onto the output channel.

```xml
<int:object-to-map-transformer
    input-channel="in-channel"
    output-channel="stdout">
</int:object-to-map-transformer>
<int-stream:stdout-channel-adapter id="stdout"/>
```

The above snippet’s output is printed to the console using the stdout-channel-
adapter as below:

```shell
{direction=BUY, account=B12D45, security=null, status=NEW, quantity=0, id=1234}
```

Conversely, the map-to-object-transformer is used to convert the name-valued pairs
of Map to a Java Object. The use of the element is shown in the snippet below:

```xml
<int:map-to-object-transformer
    input-channel="in-channel"
    output-channel="stdout">
</int:map-to-object-transformer>
```

### 序列化和反序列化转换器

Readers familiar with Java Message Service (JMS) will know that the messages must be serialized and deserialized when sent or received, respectively. The Payload Serializ ing transformer transforms a POJO to a byte array. It is represented below by **payload- serializing-transformer**:

```xml
<int:payload-serializing-transformer
    input-channel="trades-in-channel"
    output-channel="trades-out-channel">
</int:payload-serializing-transformer>
```

When a **SerializableTrade** is published onto the **trades-in-channel**, the transformer picks and converts the payload to bytes. The deserializing transformer is then used to read the bytes back to **SerializableTrade**.

The deserializing transformer works in exactly the opposite manner as its counterpart by deserializing the serialized payload to a POJO message. It is represented by **payload- deserializing-transformer** and reads a byte array. The following snippet demonstrates a deserializing transformer printing out the **toString()** of **SerializableTrade** onto the console by picking up the bytes from the **trades-out-channel**, the output channel of the Serializing Transformer.

```xml
<int:payload-deserializing-transformer
    input-channel="trades-out-channel"
    output-channel="stdout">
</int:payload-deserializing-transformer>
<int-stream:stdout-channel-adapter id="stdout"/>
```

### JSON 转换器

JavaScript Object Notation (JSON) is the lightweight message data exchange format that is completely language independent. It produces human-readable, formatted name values. The Spring Integration framework supports automatic transformations from an Object to JSON representation. As the name suggests, the **object-to-json-trans** former transforms an Object to JSON-formatted payload.

```xml
<int:object-to-json-transformer input-channel="trades-in-channel" output-channel="stdout">
</int:object-to-json-transformer>
```

Using our Trade object, the expected JSON format is printed to the console:

```json
{"id":"1234","direction":"BUY","account":"B12D45","security":null,"status":
"NEW","quantity":0}
```

The **json-to-object-transformer** acts the other way—converting the JSON formatted payload to a Java Object. The type attribute specifies the type of object that the trans- former needs to instantiate and populate with the input JSON data.

```xml
<int:json-to-object-transformer
    input-channel="trades-in-channel"
    output-channel="trades-out-channel"
    type="com.example.domain.Trade">
</int:json-to-object-transformer>
```

### XML 转换器

For those applications which use XML as the message format, the framework provides support in converting a POJO to XML and vice versa automatically. There’s a bit more involved than simply using an XML tag, as we see in the above built-in transformers. If you have already worked with Spring’s Object-to-XML (OXM) framework, this sec- tion will be easy.

Spring uses two classes to marshal and unmarshal the Object into XML and vice versa: the org.springframework.oxm.Marshaller and the org.springframework.oxm.Unmarshal ler. The Marshaller is used to convert an Object to an XML Stream, while the Un- marshaller does the opposite—converting an XML stream to an Object.

You need to access the XML transformers using an XML namespace. The following highlighted code shows the importation of another namespace in our XML file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans" ...
    xmlns:int-xml="http://www.springframework.org/schema/integration/xml"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    ....
    http://www.springframework.org/schema/integration/xml
    http://www.springframework.org/schema/integration/xml/spring-integration-xml-2.1.xsd">
</beans>
```

Once you declare the namespaces, use the **marshalling-transformer** element to read a message off an input channel. The message is formatted into XML and posted back to the output channel. See the wiring of the **marshalling-transformer** below:

```xml
<int-xml:marshalling-transformer
    input-channel="trades-in-channel"
    output-channel="stdout"
    marshaller="marshaller"
    result-type="StringResult">
</int-xml:marshalling-transformer>
<bean id="marshaller" class="org.springframework.oxm.castor.CastorMarshaller" />
```

As expected, the **marshalling-transformer** picks up the messages from an input channel and spits out an XML-formatted message onto a standard output. The noteworthy point is the wiring of the **marshaller** and the **result-type**. The referenced marshaler is a **CastorMarshaller** which is declared as a bean in the same **config** file.

The output of the message is printed below (note that I’ve formatted the output result with new lines for clarity):

```xml
Payload:
<?xml version="1.0" encoding="UTF-8"?>
<trade>
    <status>NEW</status>
    <account>B12D45</account>
    <direction>BUY</direction>
    <id>1234</id>
</trade>
```

The **marshalling-transformer** takes an optional **result-type** which decides the result type. There are two built-in result types—**javax.xml.transform.dom.DOMResult** and **org.springframework.xml.transform.StringResult**. The **DOMResult** is the default one, meaning if you don’t provide the **result-type**, the output message payload will be of the **DOMResult** type.

If you wish to use your own custom **result-type** transformer, you have the option of providing a **result-factory** attribute.

```xml
<int-xml:marshalling-transformer
    input-channel="trades-in-channel-xml"
    output-channel="trades-out-channel-xml"
    marshaller="marshaller"
    result-factory="tradeResultFactory">
</int-xml:marshalling-transformer>
<bean id="tradeResultFactory" class="com.example.transformers.builtin.TradeResultFactory" />
```

The **TradeResultFactory** has one method to implement—createResult, inherited from **ResultFactory**:

```java

public class TradeResultFactory implements ResultFactory {
    public Result createResult(Object payload) {
        System.out.println("Creating result ->"+payload);
        //create your own implementation of Result
        return new TradeResult();
    }
}
```

### XPath 转换器

The xpath-transformer decodes the XML payload using XPath expressions. The trans- former expects an XML payload on an input channel and outputs the result to the output channel after applying the XPath expression. The configuration is simple:

```xml
<int-xml:xpath-transformer
    input-channel="trades-in-channel"
    output-channel="stdout"
    xpath-expression="/trade/@status">
    <int:poller fixed-rate="1000" />
</int-xml:xpath-transformer>
```

Create and publish the XML payload message as shown below onto the trades-in- channel:

```java
private String createNewTradeXml() {
    return "<trade status='NEW' account='B12D45' direction='BUY'/>";
}
```

The input message’s payload will be parsed for a status attribute’s value and will print it to the console:

```shell
//publishes the status onto stdout:

NEW
```

## 自定义转换器

## 使用注解

You can use Framework’s @Transformer annotation to refer to the transformer bean from your config file. The component-scan allows the container to scan for annotated beans in the transformers package, In this case, the AnnotatedTradeMapTransformer class will be instantiated:

```java
@Component
public class AnnotatedTradeMapTransformer {
    @Transformer
    public Map<String, String> transform(Trade t) {
        Map<String,String> tradeNameValuesMap = new HashMap<String,String>();
        ....
        return tradeNameValuesMap;
    }
}
```

The annotated **transform** method is invoked when a message arrives in the **in-channel**. The configuration is similar to the one we have already seen, except the **component-scan** tag is added. This scans for the beans decorated with @Component and creates instances of them once found in the application context container:

```xml
<context:component-scan base-package="com.example.transformer" />
```

## 结语

Transformers play an important role in satisfying different clients’ requirements. They form a vital part of creating seamless integration between the endpoints. In this chapter, we discussed the workings of Transformers in detail. We touched on various aspects of Transformers, including the difference between the custom and built-in transform- ers. Finally, we explored the transformers used for transforming real word objects to XML and vice versa.