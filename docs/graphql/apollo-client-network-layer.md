> Apollo client 有一个可插拔的网络接口层, 它让你能够配置查询如何通过HTTP进行发送, 或者通过Websocket, 或者使用客户端模拟数据.

## 配置网络接口

要创建一个网络接口使用 [createNetworkInterface][1]

下面的示例用自定义端点初始化一个新的Client:

```
import ApolloClient, { createNetworkInterface } from 'apollo-client';

const networkInterface = createNetworkInterface({ uri: 'https://example.com/graphql' });

const client = new ApolloClient({
  networkInterface,
});
```

如果需要, 还可以传递额外的选项给[fetch][2]:

```
import ApolloClient, { createNetworkInterface } from 'apollo-client';

const networkInterface = createNetworkInterface({
  uri: 'https://example.com/graphql',
  opts: {
    // Additional fetch options like `credentials` or `headers`
    credentials: 'same-origin',
  }
});

const client = new ApolloClient({
  networkInterface,
});
```

## 中间件(Middleware)

> **注:** 中间件的概念和Windows编程中的`钩子`概念类似.

`createNetworkInterface`创建的网络接口还可以和中间件一起使用, 中间件用于在请求发送到服务器之前修改请求. 要使用中间件, 需要传递一个对象数组给 networkInterface 的 `use` 函数, 每个对象必须包含有如下参数的 `applyMiddleware`方法:

- `req`: `object`
    由中间件处理的HTTP请求对象.
- `next`: `function`
    该函数把请求对象往下传递(传递到下一个中间件, 如果有多个中间件).

下面的例子, 显示了如何传递一个身份验证Token给HTTP请求的Header.

```
import ApolloClient, { createNetworkInterface } from 'apollo-client';

const networkInterface = createNetworkInterface({ uri: '/graphql' });

networkInterface.use([{
  applyMiddleware(req, next) {
    if (!req.options.headers) {
      req.options.headers = {};  // 如果需要, 创建 header 对象.
    }
    req.options.headers['authorization'] = localStorage.getItem('token') ? localStorage.getItem('token') : null;
    next();
  }
}]);

const client = new ApolloClient({
  networkInterface,
});
```

下面的例子显示如何使用多个中间件:

```
import ApolloClient, { createNetworkInterface } from 'apollo-client';

const networkInterface = createNetworkInterface({ uri: '/graphql' });
# 用于请求GraphQL服务器的身份标识
const token = 'first-token-value';
const token2 = 'second-token-value';

# 第一个中间件
const exampleWare1 = {
  applyMiddleware(req, next) {
    if (!req.options.headers) {
      req.options.headers = {};  // Create the headers object if needed.
    }
    req.options.headers['authorization'] = token;
    next();
  }
}

# 第二个中间件
const exampleWare2 = {
  applyMiddleware(req, next) {
    if (!req.options.headers) {
      req.options.headers = {};  // Create the headers object if needed.
    }
    req.options.headers['authorization'] = token2;
    next();
  }
}

# 以对象数组的方式传递
networkInterface.use([exampleWare1, exampleWare2]);

# 初始化客户端
const client = new ApolloClient({
  networkInterface,
});
```

## Afterware

Afterware 用户请求发送后, 并且HTTP响应从服务器返回的时候, 一般用于处理HTTP请求的响应. 与中间件不同的是, `applyAfterware` 方法的参数不同:

- `{ response }: object`
    一个HTTP响应对象.
- `next: function`
     传递HTTP响应到下一个 `afterware`.

> **注:** 这个我觉得应该叫做 `ResponseHandlers` 好像更容易理解. 同理之前的中间件可以称作 `RequestHandlers`

下面是一个 Afterware 的例子:

```
import ApolloClient, { createNetworkInterface } from 'apollo-client';
import {logout} from './logout';

const networkInterface = createNetworkInterface({ uri: '/graphql' });

# 这里使用的是 useAfter
networkInterface.useAfter([{
  applyAfterware({ response }, next) {
    if (response.status === 401) {
      logout();
    }
    next();
  }
}]);

const client = new ApolloClient({
  networkInterface,
});
```

多个Afterware的例子:

```
import ApolloClient, { createNetworkInterface } from 'apollo-client';
import {redirectTo} from './redirect';

const networkInterface = createNetworkInterface({ uri: '/graphql' });

const exampleWare1 = {
  applyAfterware({ response }, next) {
    if (response.status === 500) {
      console.error('Server returned an error');
    }
    next();
  }
}

const exampleWare2 = {
  applyAfterware({ response }, next) {
    if (response.status === 200) {
      redirectTo('/');
    }
    next();
  }
}

networkInterface.useAfter([exampleWare1, exampleWare2]);

const client = new ApolloClient({
  networkInterface,
});
```

## Chaining (链起来)


```
networkInterface.use([exampleWare1])
  .use([exampleWare2])
  .useAfter([exampleWare3])
  .useAfter([exampleWare4])
  .use([exampleWare5]);
```

## 自定义网络接口

可以自定义以不同的方式向GraphQL服务器发送查询或数据更新请求, 这样做有几个原因:

- 替换传输层, 比如用Websocket替换默认的HTTP, 降低Web应用的延迟. 增强Web应用的流程度, 提升用户体验等.
- 请求发送前需要修改查询和变量
- 开发阶段的纯客户端模式, 模拟服务器的响应

## Query batching

Query batching 是这样一个概念, 当多个请求在一个特定的时间间隔内产生时(比如: 100毫秒内), Apollo 会把多个查询组合成一个请求, 比如在渲染一个包含导航条, 边栏, 内容等带有GraphQL查询的组件时, Query batching 要求服务支持 `Query batching`(比如 [graphql-server][3]).


使用Query batching, 要传递`BatchedNetworkInterface`给`ApolloClient`构造函数, 例如:

```
import ApolloClient, { createBatchingNetworkInterface } from 'apollo-client';

const batchingNetworkInterface = createBatchingNetworkInterface({
  uri: 'localhost:3000',
  batchInterval: 10,
  opts: {
    // 传递给`fetch`的选项
  }
});

const apolloClient = new ApolloClient({
  networkInterface: batchingNetworkInterface,
});

apolloClient.query({ query: firstQuery });
apolloClient.query({ query: secondQuery });
```

## Query batching 是怎么工作的

Query batching 是传输层机制, 需要服务器的支持(比如 [graphql-server][4], 如果服务器不支持, 请求会失败. 如果服务器打开了 Query batching, 多个请求会组合到一个数组中:

```
[{
   query: `query Query1 { someField }`,
   variables: {},
   operationName: 'Query1',
 },
 {
   query: `query Query2 ($num: Int){ plusOne(num: $num) }`,
   variables: { num: 3 },
   operationName: 'Query2',
 }]
```

## 查询去除重复(Query deduplication)

查询去重可以减少发送到服务器的查询数量, 默认关闭, 可通过`queryDeduplication`选项传递给`ApolloClient`构造函数开启, 查询去重发送在请求到达网络层之前.

```
const apolloClient = new ApolloClient({
  networkInterface: batchingNetworkInterface,
  queryDeduplication: true,
});
```

查询去重在多个组件显示相同数据的时候非常有用. 避免从服务器多次获取相同的数据.

## 接口

这里有几个和网络请求相关的接口, 通过这些接口, 你可以自定义如何处理网络请求和响应

http://dev.apollodata.com/core/network.html#NetworkInterface


  [1]: http://dev.apollodata.com/core/apollo-client-api.html#createNetworkInterface
  [2]: https://github.github.io/fetch/
  [3]: https://github.com/apollostack/graphql-server
  [4]: https://github.com/apollostack/graphql-server
