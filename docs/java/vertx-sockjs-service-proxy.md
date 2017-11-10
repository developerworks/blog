## SockJS 服务代理

> 通过SockJS服务代理, 我们可以从浏览器或Node.js来调用事件总线上的服务.

我们已经演示过如何在Java中实现服务代理来调用服务, 这篇文章阐述如何在浏览器或Node.js中来调用服务. 这对异构系统的集成来讲是非常方便的. 比如你有一个Java编写的在事件总线上的二维码服务(传递参数生成二维码), 我们需要在Node.js端或浏览器端生成一个二维码. 那么这时, SockJS服务代理就非常有用了, 你不需要手工去解析消息, SockJS服务代理自动为你解析消息并调用远程服务, 就像调用本地函数一样.

为了让代理和服务进行通信,首先我们需要配置SockJS桥.


```java
package com.totorotec.service.qrcode;

import io.vertx.core.AbstractVerticle;
import io.vertx.ext.bridge.PermittedOptions;
import io.vertx.ext.web.Router;
import io.vertx.ext.web.handler.sockjs.BridgeOptions;
import io.vertx.ext.web.handler.sockjs.SockJSHandler;
import java.util.List;

/**
 * Vert.x-Web 包含内置的SockJS套接字处理器, 被称为事件总线桥, 它有效地把服务器端Vert.x事件总线
 * 扩展到了客户端Javascript
 */
public class QrcodeServiceBridge extends AbstractVerticle {
  @Override
  public void start() throws Exception {
    super.start();
    // 初始化路由器
    Router router = Router.router(vertx);
    // 套接字处理
    SockJSHandler sockJSHandler = SockJSHandler.create(vertx);
    // 桥选项
    BridgeOptions options = new BridgeOptions();
    options.addInboundPermitted(new PermittedOptions().setAddress(QrcodeService.SERVICE_ADDRESS));
    options.addOutboundPermitted(new PermittedOptions().setAddress(QrcodeService.SERVICE_ADDRESS));
    // 桥接
    sockJSHandler.bridge(options);
    // 设置路由
    router.route("/eventbus/*").handler(sockJSHandler);
    // 监听 8080 端口
    vertx.createHttpServer().requestHandler(router::accept).listen(8080);
  }
}
```

一旦完成SockJS桥的设置, 我们就可以在浏览器或Node.js中与后端服务通信了. 在服务编译阶段, 编译过程会自动生成JS代理模块. 生成的JS代理模块命名规则为: `module_name-js/server-interface_simple_name + -proxy.js`. 例如如果你的服务接口名称为 **QrcodeService**, 那么生成的代理模块名称为 **qrcode_service-proxy.js**. 然后在浏览器或Node.js中引入(**require**)即可.

生成的模块是兼容 CommonJS, AMD 和 Webpack 的. 下面是一个浏览器端的例子:

```js
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

下面的例子是说明了如何在Node.js环境中调用服务:

```
var EventBus = require("vertx3-eventbus-client");
var eb = new EventBus("http://localhost:8080/eventbus");
eb.onopen = function () {
  // 导入代理模块
  var QrcodeService = require("./target/classes/qrcode-service-js/qrcode_service-proxy");
  // 实例化服务对象
  var service = new QrcodeService(eb, "com.totorotec.servicefactory.qrcode-service");
  // 调用服务
  service.getQrcode("https://gm.totorotec.com", 380, "png", "dataurl", "/tmp/%s.%s", function (data, error) {
    console.log(data);
    console.log(error);
  });
}
```

对于SockJS代理模块的生成, 需要在Maven的POM中增加如下依赖, 然后执行 mvn compile 生成的SockJS代理模块默认会放在 ./target/classes 目录下. 名称遵循前面说过的命名规则. 对于二维码服务这个例子, 生成的SockJS代理的位置为: **target/classes/qrcode-service-js/qrcode_service-proxy.js**:

```xml
<dependency>
  <groupId>io.vertx</groupId>
  <artifactId>vertx-sockjs-service-proxy</artifactId>
  <version>3.5.0</version>
</dependency>
```

完整的例子查看仓库: https://github.com/developerworks/service_qrcode.git

## 演示过程

启动服务

```
./redeploy.sh
```

### 测试浏览器端实现

在项目根目录下运行 `python -m SimpleHTTPServer 9999`, 浏览器地址输入: http://localhost:9999, 打开文件 test-qrcode-service-browser.html 测试浏览器端获取二维码的dataURL.


### 测试Node.js实现

同样在根目录下执行

```
node tests/test-qrcode-service.js
```

另外还有一个ES的 `async/await` 实现, 需要安装 `babel-cli`:

```
yarn add --dev babel-cli
./node_modules/.bin/babel-node ./tests/test-qrcode-service-promise.js
```

