`Apollo Client` 使用四个方法来控制存储:`readQuery`, `readFragment`, `writeQuery`, `writeFragment`. Apollo Client 的这些实例方法可以让你直接读写缓存.

用这四个方法能够让你自定义Apollo Client的行为. 下面我们介绍每个方法的细节.

## 从存储读取查询

第一个和存储交互的方法是`readQuery`. 该方法用于从根查询开始读取缓存数据, 下面是读取数据的例子:

```
const data = client.readQuery({
  query: gql`
  {
    todo(id: 1) {
      id
      text
      completed
    }

  }
  `
});
```

如果数据在缓存中已经存储, 它将会直接返回并存储到组件的`data`属性, 而不需要想服务器请求, 如果在缓存中找不到查询对应的数据, 将会抛出一个错误.

你可以在应用中任何地方使用一个查询, 还可以传入变量:

```
import { TodoQuery } form './TodoGraphQ';

const data = client.readQuery({
  query: TodoQuery,
  variables: {
    id: 5
  }
});
```

`readQuery` 和 `query` 类似, 但是`readQuery` 不会发送请求到服务器, 它总是期望从缓存中查询数据, 如果缓存中找不到对应的数据, 将会抛出一个错误.

## 从存储读取片段

有时候你希望重缓存中读取任意的数据, 而不仅仅是根查询类型. 对此有一个 `readFragment()` 方法用于这类用途. 该方法接受 GraphQL 片段和一个 ID, 并返回 ID 对应的数据片段.

```
client.readFragment({
  id: '5',
  fragment: gql`
    fragment todo on Todo {
      id
      text
      completed
    }
  `
});
```

`id` 应该是一个由 `dataIdFromObject` 函数返回的字符串, 这个字符串是在 Apollo Client 初始化的时候通过 `dataIdFromObject`.

```
const client = new ApolloClient({
  dataIdFromObject: o => {
    if(o.__typename != null && o.id != null) {
      return `${o.__typename}-${o.id}`;
    }
  }
});
```

因为在`id`前面添加了变量`__typename`, 然后`id`的值变为`Todo5`.

## 向存储写入查询和片段

和读取查询和片段对应的还有 `writeQuery()` 和 `writeFragment()` 方法. 这两个方法让你能够更新缓存中的数据, 用于模拟来自服务器的数据更新. 但是注意这些数据没有持久化到后端, 如果你刷新你的浏览器, 这些更新的数据将会丢失.

用户不会注意到有什么区别, 如果更新数据要对所有用户可见, 需要把更新的数据持久化到后端服务器.

`writeQuery` 和 `writeFragment` 的优点是, 它们让你能够修改缓存中的数据以确保数据能够同步到服务器, 并且在执行一次服务器完全刷新的时候不会丢失你的数据更新. 这让你能够部分的修改客户端数据, 给用户提供更好的体验.

`writeQuery()` 和 `readQuery` 有相同的接口, 和 `readQuery` 不同的是 `writeQuery` 还有一个 `data` 参数. `data` 对象的结构必须和服务器返回的JSON结果的结构相同.

```
client.writeQuery({
  query: gql`
    {
      todo(id: 1) {
        completed
      }
    }
  `,
  data: {
    todo: {
      completed: true
    },
  },
});
```

同样的, `writeFragment()` 和 `readFragment()` 也有相同的接口, 并且多一个 `data` 参数. `id` 遵循和 `readFragment()` 相同的规则:

```
client.writeFragment({
  id: '5',
  fragment: gql`
    fragment todo in Todo {
      completed
    }
  `,
  data: {
    completed: true
  }
});
```

这四个方法让你能够完全的控制缓存中的数据.

## 使用读和写来更新数据

因为从缓存中获取的数据只是一份拷贝, 不影响底层的存储.

```
const query = gql`
  {
    todos {
      id
      text
      completed
    }
  }
`;

const data = client.readQuery({
  query
});

data.todos.push({
  id: 5,
  text: 'Hello, world!',
  completed: false
});

client.writeQuery({
  query,
  data
});
```

## 在一个Mutation之后执行更新

```
const text = 'Hello, world!';
client.mutate({
  // GraphQL Mutation 更新语句
  mutation: gql`
    mutation ($text: String!) {
      createTodo(text: $text) {
        id
        text
        completed
      }
    }
  `,
  // 变量
  variables: {
    text,
  },
  optimisticResponse: {
    createTodo: {
      id: -1, // Fake id
      text,
      completed: false,
    },
  },
  // 更新函数
  // 用Mutation返回的结果对存储进行更新, 并触发React UI组件的重新渲染
  update: (proxy, mutationResult) => {
    const query = gql`
      {
        todos {
          id
          text
          completed
        }
      }
    `;
    // 从缓存中读取数据
    const data = proxy.readQuery({
      query,
    });
    // 用Mutation的结果更新数据
    data.todos.push(mutationResult.createTodo);
    // 写回缓存
    proxy.writeQuery({
      query,
      data,
    });
  },
});
```

`update` 函数有两个参数:

- `proxy` 是一个 [DataProxy](http://dev.apollodata.com/core/apollo-client-api.html#DataProxy) 对象, 主要用于与底层存储进行数据交互
- `mutationResult` 是一个Mutation操作的响应, 可以使一个乐观应答, 或服务器实际的应答.

## updateQueries

也可以使用`updateQueries`回调函数对数据进行更新. 详细的API接口可参考 [updateQueries](http://dev.apollodata.com/react/api-mutations.html#graphql-mutation-options-updateQueries)

`updateQueries` 也是基于 Mutation 的返回结果对存储进行更新, 和 `update` 函数不同的是, 他会覆盖所有重叠的数据节点

## 参考资料

- https://dev-blog.apollodata.com/apollo-clients-new-imperative-store-api-6cb69318a1e3
