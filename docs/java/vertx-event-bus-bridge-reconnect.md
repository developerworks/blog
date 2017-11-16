## Vert.x 3: Node和浏览器端事件总线重连

默认初始化的EventBus对象没有开启重连, 如果网络断开后, 再次调用服务代理会导致错误. 因此这里我们需要在 **onopen** 和 **onreconnect** 两个事件中处理一下服务注册的问题.

```js
var EventBus = require("vertx3-eventbus-client");
var address = "http://localhost:8080/eventbus"

var options = {
  vertxbus_reconnect_attempts_max: Infinity,  // Max reconnect attempts
  vertxbus_reconnect_delay_min: 1000,         // Initial delay (in ms) before first reconnect attempt
  vertxbus_reconnect_delay_max: 5000,         // Max delay (in ms) between reconnect attempts
  vertxbus_reconnect_exponent: 2,             // Exponential backoff factor
  vertxbus_randomization_factor: 0.5          // Randomization factor between 0 and 1
};

var eb = new EventBus(`${address}`, options);
eb.enableReconnect(true)


# 服务注册

var QrcodeServiceProxy = require("../lib/qrcode_service-proxy");

eb.onopen = function () {
  logger.info("Opening event bus...")
  # 注册到global对象, 让其在所有模块中可用.
  global.vertx_services = {
    qrcodeService: new QrcodeServiceProxy(eb, "com.totorotec.servicefactory.qrcode-service")
  }
}
eb.onerror = function() {
  logger.error("Eventbus: error ocurred.")
}
eb.onclose = function() {
  logger.error("Eventbus: closed.")
}

eb.onreconnect = function() {
  logger.info("Eventbus: reconnecting...")
  global.vertx_services = {
    qrcodeService: new QrcodeServiceProxy(eb, "com.totorotec.servicefactory.qrcode-service")
  }
}
```

杀掉服务端进程, 观看Node端的输出:

```
[2017-11-16 16:00:01.821] [ERROR] default - Eventbus: closed.
[2017-11-16 16:00:06.837] [ERROR] default - Eventbus: closed.
...
...
...
```

重启服务端 `yarn run_qr8`后观察Node的输出:

```
...
...
...
[2017-11-16 16:00:46.912] [ERROR] default - Eventbus: closed.
[2017-11-16 16:00:52.062] [INFO] default - Opening event bus...
[2017-11-16 16:00:52.062] [INFO] default - Eventbus: reconnecting...
```

我们看到连接恢复了.

![image](https://user-images.githubusercontent.com/725190/32880457-b2b35b4c-cae8-11e7-94b0-833ddd21042d.png)


## 代码库

二维码项目的所有源代码, 包括服务器和客户端在下面的仓库中:

https://github.com/developerworks/vertx-qrcode-service


