现在我们已经创建了一个`ApolloClient`实例并且使用`ApolloProvider`附加到了UI组件树, 我们可以开始使用`react-apollo`的主要功能: 添加GraphQL功能到我们的UI组件当中.

## GraphQL

`graphql` 容器是用于获取或修改数据推荐的方法. 他是一个[高阶组件][1], 用于把Apollo的数据提供给组件.

`graphql` 的基本使用方法如下:

```
import React, { Component } from 'react';
import { graphql } from 'react-apollo';
import gql from 'graphql-tag';

// 定义一个普通的React组件
class MyComponent extends Component {
  render() {
    return <div>...</div>;
  }
}

// 使用`gql`标签初始化GraphQL查询和数据变更
const MyQuery = gql`query MyQuery { todos { text } }`;
const MyMutation = gql`mutation MyMutation { addTodo(text: "Test 123") { id } }`;

// 数据绑定传入MyQuery和MyComponent, 返回一个包含数据的React组件
const MyComponentWithData = graphql(MyQuery)(MyComponent);

// 返回一个包含数据更新的React组件
const MyComponentWithMutation = graphql(MyMutation)(MyComponent);
```

如果使用 [ES2016 decorators][2], 可以这样写:

```
import React, { Component } from 'react';
import { graphql } from 'react-apollo';

@graphql(MyQuery)
@graphql(MyMutation)
class MyComponent extends Component {
  render() {
    return <div>...</div>;
  }
}
```

`graphql` 函数接受2个参数:

- `query`: 必须, 一个使用`gql` tag解析的GraphQL文档
- `config`: 可选, 一个配置对象, 详细的描述如下

该配置对线可以包含一个或多个下面的选项:

- `name`: Rename the prop the higher-order-component passes down to something else
- `options`: Pass options about the query or mutation, documented in the queries and mutations guides
- `props`: 在传递给子组件前修改属性.
- `withRef`: Add a method to access the child component to the container, read more below
- `shouldResubscribe`: A function which gets called with current props and next props when props change. The function should return true if the change requires the component to resubscribe.

`graphql` 函数返回另一个函数, 他接受任何React组件并且返回一个特定查询包裹的新React组件类. 这和Redux中的`connect`是类似的.

`graphql` 函数的详细说明可参考 [queries][3] 和 [mutations][4]

## withApollo

withApollo 让 我们把`ApolloClient` 作为组件的属性直接访问:

```
import React, { Component } from 'react';
import { withApollo } from 'react-apollo';
import ApolloClient from 'apollo-client';
# 创建一个普通的React组件
class MyComponent extends Component { ... }

MyComponent.propTypes = {
  client: React.PropTypes.instanceOf(ApolloClient).isRequired,
}
const MyComponentWithApollo = withApollo(MyComponent);
// or using ES2016 decorators:
@withApollo
class MyComponent extends Component { ... }
```

然后我们可通过 `MyComponent.client` 访问 `ApolloClient` 实例.

> 注: 关于`propTypes`的用途, 参考
> - [PropTypes](http://www.wisestudy.cn/opentech/React-data-propTypes.html)
> - [属性校验](https://segmentfault.com/a/1190000002670661)
> - [React 之 propTypes](http://lib.csdn.net/article/react/13332)

## withRef

如果需要访问被包裹的组件, 可以在选项中使用`withRef`. 可以通过调用返回组件的`getWrappedInstance`获取被包裹的实例.

```
import React, { Component } from 'react';
import { graphql } from 'react-apollo';

# 一个普通的React组件
class MyComponent extends Component { ... }
# 通过graphql函数返回的组件
const MyComponentWithUpvote = graphql(Upvote, {
  withRef: true,
})(MyComponent);

// 调用返回组件的getWrappedInstance方法可得到MyComponent
// MyComponentWithUpvote.getWrappedInstance() returns MyComponent instance
```

## compose

`react-apollo` 导出了一个`compose`函数. 用于减少书写代码的量.

```
import { graphql, compose } from 'react-apollo';
import { connect } from 'react-redux';

export default compose(
  graphql(query, queryOptions),
  graphql(mutation, mutationOptions),
  connect(mapStateToProps, mapDispatchToProps)
)(Component);
```

  [1]: https://facebook.github.io/react/blog/2016/07/13/mixins-considered-harmful.html#subscriptions-and-side-effects
  [2]: https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841#.nn723s5u2
  [3]: https://github.com/apollographql/react-docs/blob/master/react/queries.html
  [4]: https://github.com/apollographql/react-docs/blob/master/react/mutations.html
