> 服务代理提供了在事件总线上暴露服务, 并且减少调用服务所要求的代码的一种方式. 服务代理帮助你解析事件总线的消息结构和接口方法的映射关系, 比如方法名称, 参数等等. 本质上是一种RPC. 它通过代码生成的方式创建服务代理.

因此需要实现两个Verticle, 一个为 `QrcodeService`, 提供数据库操作服务. 另一个 Verticle 用于请求数据库操作.

Verticle 运行在一个两节点的集群环境中中

## 提供服务

要提供一个服务, 需要一个服务接口, 一个实现和一个Verticle

- 服务接口: `QrcodeService`, 服务接口, 定义了数据操作接口和代理创建接口.
- 服务实现: `QrcodeServiceImpl`, 服务实现类, 实际的数据库操作在这里实现
- 服务注册: `QrcodeServiceVerticle`, 用于注册服务

服务接口是通过 `@ProxyGen` 标注的接口, 例如:

```
package com.totorotec.service.qrcode;

import io.vertx.codegen.annotations.ProxyGen;
import io.vertx.codegen.annotations.VertxGen;
import io.vertx.core.AsyncResult;
import io.vertx.core.Handler;
import io.vertx.core.Vertx;
import io.vertx.core.json.JsonObject;

import com.totorotec.service.qrcode.impl.QrcodeServiceImpl;

@ProxyGen
@VertxGen
public interface QrcodeService {
  public static final String SERVICE_ADDRESS = "com.totorotec.servicefactory.qrcode-service";

  static QrcodeService create(Vertx vertx, JsonObject config) {
    return new QrcodeServiceImpl(vertx, config);
  }

  static QrcodeService createProxy(Vertx vertx, String address) {
    return new QrcodeServiceVertxEBProxy(vertx, address);
  }

  void getQrcode(String text, int imageSize, String imageType, String outputType, String filePatten, Handler<AsyncResult<JsonObject>> resultHandler);
}

```

> 注意: `java.lang.IllegalStateException: Cannot find proxyClass`, 把``插件的版本升级到`3.7.0`

```
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.7.0</version>
  <configuration>
    <source>1.8</source>
    <target>1.8</target>
  </configuration>
</plugin>
```

> 另外: 需要给插件 `maven-compiler-plugin` 增加符号处理器配置 `annotationProcessor`, 如下

```
<plugin>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.7.0</version>
  <configuration>
    <source>1.8</source>
    <target>1.8</target>
    <annotationProcessors>
      <annotationProcessor>io.vertx.codegen.CodeGenProcessor</annotationProcessor>
    </annotationProcessors>
  </configuration>
</plugin>
```

完整的 pom.xml 项目文件可以参考: https://github.com/developerworks/service_qrcode/blob/master/pom.xml

## 消费服务

实现了一个服务提供者, 下面我们来说明如何消(调)费(用)这个服务.

### 在Vertx JVM中从Java端调用

```java
package com.totorotec.service.qrcode;

import io.vertx.core.AbstractVerticle;
import io.vertx.core.logging.Logger;
import io.vertx.core.logging.LoggerFactory;

/**
 * QrcodeServiceConsumer
 */
public class QrcodeServiceConsumer extends AbstractVerticle {

  private static final Logger logger = LoggerFactory.getLogger(QrcodeServiceConsumer.class);

  @Override
  public void start() throws Exception {
    super.start();

    QrcodeService qrcodeServiceProxy = QrcodeService.createProxy(vertx, QrcodeService.SERVICE_ADDRESS);
    qrcodeServiceProxy.getQrcode("https://www.qq.com", 600, "jpg", "file",
        "/Users/hezhiqiang/totoro/_vertx-projects/service_qrcode/_tmp/%s.%s", ar -> {
          if (ar.succeeded()) {
            logger.info(ar.result().encodePrettily());
          } else {
            logger.error(ar.cause());
          }
        });
  }
}
```

### 在Vertx JVM中从Javascript调用

```js
var service = require("qrcode-service-js/qrcode_service");

console.log("Creating service proxy...");
var proxy = service.createProxy(vertx, "com.totorotec.servicefactory.qrcode-service");

proxy.getQrcode("https://gm.totorotec.com", 380, "png", "dataurl", "/tmp/%s.%s", function (error, data) {
  if(error == null) {
    console.log(data);
  }
  else {
    console.log(error);
  }
});
```

### 在浏览器中调用

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Test Qrcode Service in Browser</title>
  <script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
  <script src="./vertx-eventbus.js"></script>
  <script src="./qrcode_service-proxy.js"></script>
</head>

<body>

  <div id="qrcode"></div>

  <script>

    var qrcodeStr;
    var eb = new EventBus('http://localhost:8080/eventbus');
    eb.onopen = function () {
      var service = new QrcodeService(eb, "com.totorotec.servicefactory.qrcode-service");
      service.getQrcode("https://www.qq.com", 380, "png", "dataurl", "/tmp/%s.%s", function (error, data) {
        if (error == null) {
          console.log(data.data);
          qrcodeStr = data.data
          document.getElementById("qrcode").innerHTML = qrcodeStr;

        }
        else {
          console.log(error);
        }
      });
    };
  </script>

</body>

</html>

```

### 在Node.js环境中调用

```js
var EventBus = require("vertx3-eventbus-client");

var eb = new EventBus("http://localhost:8080/eventbus");

eb.onopen = function () {
  // 导入代理模块
  var QrcodeService = require("../target/classes/qrcode-service-js/qrcode_service-proxy");
  // 实例化服务对象
  var service = new QrcodeService(eb, "com.totorotec.servicefactory.qrcode-service");
  // 调用服务
  service.getQrcode("https://www.qq.com", 380, "png", "dataurl", "/tmp/%s.%s", function (data, error) {
    if(error == null) {
      console.log(data);
    } else {
      console.log(error);
    }
  });
}

```

## 结语

事件总线, 对于异构系统集成来讲是一个很好的工具. 通过定义服务接口, 服务实现, 代理类代码生成, 服务注册, 事件总线桥, 我们可以把异构系统的各个端点连接到事件总线中, 实现分布式的, 异构系统的通信, 事件处理.

异构还特别对团队有用, 大的团队使用不同的开发工具, 语言, 运行时系统等, 都可以很方便的进行集成, 只要你在JVM的生态里面, 不管你使用JVM的什么语言. 即使你不在JVM生态里面, 例如Node.js, 浏览器, 其他编程语言等, Vert.x还提供了一个TCP事件总线桥的方式进行集成.

我们前面只介绍了SockJS这种集成方式, 当前互联网应用程序大部分使用HTTP作为应用层协议, 原生TCP的方式用的比较少, 在这里就不详细说明了, 有需要的可以参考Vertx的文档: http://vertx.io/docs/vertx-tcp-eventbus-bridge/java/

## 项目源码

完整的项目和实现可以参考我的仓库: https://github.com/developerworks/service_qrcode
