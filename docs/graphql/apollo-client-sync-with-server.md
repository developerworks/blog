GraphQL 入门: Apollo Client - 服务器同步

Apollo Client 会缓存查询结果, 并且在有可能的情况下使用缓存解析查询. 当你请求数据的时候, 你可以使用 [Fetch Prolicy](http://dev.apollodata.com/react/api-queries.html#graphql-config-options-fetchPolicy) 来控制这个行为. 但如果缓存中的信息过期了呢? 我们如何保证使用服务器上的更新来更新缓存. 我们的UI如何更新以反映信息的变化? 本章将会回答这些问题.

为了保证Apollo Client和服务器数据的最终一致性, 有几个策略可以考虑.

- 手动刷新
- 轮训
- GraphQL订阅


## Refech (重新获取)

重新获取是强制更新缓存最简单的方法, 以反映服务器上数据的变化. 基本上, 重新获取是`绕过缓存`强制执行一次查询, 查询结果和普通的查询一样, 更新缓存中的数据, 同时更新UI.

例如, 使用`GitHunt`模式, 我们有如下的组件实现:

```
import React, { Component } from 'react';
import { graphql, gql } from 'react-apollo';

class Feed extends Component {
  // ...
  onRefreshClicked() {
    this.props.data.refetch();
  }
  // ...
}

const FeedEntries = gql`
  query FeedEntries($type: FeedType!, $offset: Int, $limit: Int) {
    feed($type: NEW, offset: $offset, limit: $limit) {
      createdAt
      commentCount
      score
      id
      respository {
        # etc.
      }
    }
  }
`;

const FeedWithData = graphql(FeedEntries)(Feed);
```

假设我们在页面上有一个"刷新"按钮, 当点击该按钮是, 组件的`onRefreshClicked`方法被调用. 同时, `this.props.data.refetch`方法允许我们重新执行和组件`FeedCompoment`关联的查询. 这标识, 查询不是从缓存中获取`field`字段的信息, 而是直接从服务器取回最新的数据,并更新缓存, 以反映最新的结果.

如果服务器有更新. Apollo Client 存储会获取更新, 并在需要的时候重新渲染UI.

## 什么时候进行Refetch

为了获取最新的数据, 你需要知道什么时候应该执行一次`refetch`, 这里有几个不同的案例.

- `手动刷新` 多数应用提供了一个让用户手动刷新的按钮以获取最新的数据
- `事件响应` 有时候发生了一个你知道的时间, 并且数据已经失效, 例如, 你可能有一些事件推送, 它高数你什么时候数据发生了变化, 那么, `refetch` 特别适用于这种情况.
- `执行一次Mutation之后`
-
