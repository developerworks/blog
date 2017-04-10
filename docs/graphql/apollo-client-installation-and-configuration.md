## 安装

要在React中使用Apollo, 需要按照`apollo-client`包, 以及`react-apollo`集成包, 以及`graphql-tag`库用于构造查询文档.

```
npm install react-apollo --save
```

如果你在一个没有全局fetch实现的环境中(浏览器不支持Fetch API), 请确保安装 [whatwg-fetch](https://www.npmjs.com/package/whatwg-fetch) 或则

> Polyfill是一个英国产品,在美国称之为Spackling Paste(译者注:刮墙的,在中国称为腻子).记住这一点就行:把旧的浏览器想象成为一面有了裂缝的墙.这些[polyfills]会帮助我们把这面墙的裂缝抹平,还我们一个更好的光滑的墙壁(浏览器)

> Fetch API 用于代替 XMLHttpRequest
> Fetch API 请参考 https://github.github.io/fetch/

## 初始化

要在React中使用Apollo, 需要创建一个 `ApolloClient`和一个`ApolloProvider`.

`ApolloClient` 用户管理作为缓存的查询存储, 以及分发查询结果. `ApolloProvider` 用于关联`ApolloClient`到React组件.

### 创建一个客户端

创建一个客户端示例, 并且指向GraphQL服务器:

```
import { ApolloClient } from 'react-apollo';
// 默认情况客户端会发送到相同主机名(域名)下的/graphql端点
const client = new ApolloClient();
```

客户端可以携带各种[选项](http://dev.apollodata.com/core/apollo-client-api.html#constructor), 在特殊情况下, 如果你想修改GraphQL服务器的URL, 可以创建一个自定义的`NetworkInterface`:

```
import { ApolloClient, createNetworkInterface } from 'react-apollo';
const client = new ApolloClient({
  networkInterface: createNetworkInterface({ uri: 'http://my-api.graphql.com' }),
});
```

`ApolloClient` 还有一些控制客户端行为的选项, 在 [Use GraphQL with React](http://dev.apollodata.com/react/index.html) 文档中可以找到.

### 创建一个Provider

要连接客户端实例到React组件树, 需要用到`ApolloProvider`组件. 你需要确保`ApolloProvider`作为一个容器去包裹其他的需要访问GraphQL服务器数据的React组件.

```
import { ApolloClient, ApolloProvider } from 'react-apollo';
const client = new ApolloClient();
ReactDOM.render(
  <ApolloProvider client={client}>
    <MyRootComponent />
  </ApolloProvider>,
  domContainerNode
)
```
## 参考资料

- http://dev.apollodata.com/react/index.html
- [Apollo的交互式学习项目](https://github.com/learnapollo/learnapollo)


