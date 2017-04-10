> 当打开一个客户端页面的时候, 客户端可能会触发多个请求以完成数据的加载. 有可能会超出浏览器并发连接的数目, 为了解决这个问题,我们通过`Query batching`一次的请求完成页面需要的所有数据.


Query batching 就是在单个请求中包含多个查询的技术, 它有几个极其显著的优点:

- 减少请求数量
- 当多个查询包含在一个请求中时, 可以使用 [DataLoader](https://github.com/facebook/dataloader) 批量的从后端API或服务器获取数据.


## 使用

假设一个场景, 多个UI组件都是用了GraphQL查询从服务器获取数据. 下面两个查询可能是由两个不同的UI组件生产的, 会产生两次客户端和服务器之间的数据往返.

```
client.query({ query: query1 })
client.query({ query: query2 })
```

在Apollo Client中, 查询批处理默认是关闭的. 要打开查询批处理, 需要在初始化 Apollo Client的时候打开 `shouldBatch` 选项:

```
const client = new ApolloClient({
    // ... 其他选项
    shouldBatch: true
});
```

## 时间间隔

时间间隔标识在一个特定的时间段内(比如100ms), 如果客户端产生了多个查询, 那么Apollo Client会自动把多个查询合并为一个.


![clipboard.png](/img/bVLGV7)


## 查询合并

查询合并是由 `BatchedNetworkInterface` 进行合并的, 下面举例说明, 现在要执行两个GraphQL查询:

```
query firstQuery {
   author {
    firstName
    lastName
  }
}
query secondQuery {
  fortuneCookie
}
```

我们想象一下合并的问题, 假设上面的两个查询会合并成如下的样子:

```
query ___composed {
  author {
    firstName
    lastName
  }
  fortuneCookie
}
```

但是, 上述合并后的查询有几个问题

- 依据GraphQL规范, 查询字段的名称是不能相同的, 否则会产生命名冲突, 解决这个问题的办法是使用 `alias(别名)` 来保证可以合并任意查询同时不会让字段名字发生名称冲突.
    ```
    query ___composed {
      aliasName: author {
        firstName
        lastName
      }
    }
    ```
- 调试的时候服务器应答的字段名称和客户端的原始名称不一致, 给调试带来麻烦, 解决办法是在`网络层`进行查询的合并. 当前大多数GraphQL客户端实现中, 请求时以如下格式发送到服务器的:

    ```
    {
      "query": “< query string goes here >”,
      "variables": {
        <variable values go here>
      }
    }
    ```

## 网络层的查询合并

网络层的查询合并, 只需要用`createBatchingNetworkInterface` 替换 `createNetworkInterface` 即可. 例如:

```
import ApolloClient, { createBatchingNetworkInterface } from 'apollo-client';
const batchingNetworkInterface = createBatchingNetworkInterface({
  uri: 'localhost:3000',
  batchInterval: 100,
  opts: {
    // Options to pass along to `fetch`
  }
});
const apolloClient = new ApolloClient({
  networkInterface: batchingNetworkInterface,
});
// These two queries happen in quick succession, possibly in totally different
// places within your UI.
apolloClient.query({ query: firstQuery });
apolloClient.query({ query: secondQuery });
```

## 结语

上述Batching查询是正对HTTP这种无状态协议的, 目的是为了减少重复建立新的TCP连接消耗的时间, 如果采用Websocket这种有状态协议, 就没有必要在使用`Query Batching`, 采用`GraphQL Subscriptions`.

## 参考资料

- [Query batching](http://dev.apollodata.com/core/network.html#query-batching)
- [Query batching in Apollo](https://dev-blog.apollodata.com/query-batching-in-apollo-63acfd859862)
- [Improving Performance with Apollo Query Batching](https://www.graph.cool/blog/improving-performance-with-apollo-query-batching-ligh7fmn38)


  [1]: https://segmentfault.com/img/bVLGV7
