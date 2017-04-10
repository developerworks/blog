> **摘要:** 本文采用 Elixir 语言开发的 Absinthe 作为 GraphQL 的服务器端实现, 使用 Javascript 语言开发的 Apollo Client 作为 GraphQL 的客户端实现.

## 1. 持久化查询的概念

持久化查询, 是一种避免客户端直接在查询请求中包含查询文档的一种方式, 客户端只需要传递给要执行查询的ID, 服务器通过ID查询到GraphQL文档, 并在服务器端执行的过程.

**优缺点:**

客户端发送一个巨大的查询过量消耗服务器的资源
![clipboard.png](/img/bVLlAW)

客户端可以执行任意查询,容易导致安全问题
![clipboard.png](/img/bVLlA8)

**使用持久化查询的目的:**

- 避免了客户端发送GraphQL文档, 减少网络流量
- 由于GraphQL巨大的灵活性, 这也给安全带来了挑战, 这种机制实际上就是查询白名单. 只有在白名单中的查询才是能被服务器执行的.

从直接发送查询文档转换到持久化查询的效果:

**发送一个巨大的查询不再是一个问题**
![clipboard.png](/img/bVLlB8)

**客户端只能查询服务器支持的限制性的GraphQL查询(通过ID)**
![clipboard.png](/img/bVLlCf)

## 2. 要求

客户端需要使用到 [persistgraphql](https://github.com/apollographql/persistgraphql) 工具来生成对应的查询描述文档. 用于完成查询到ID的映射关系.

服务器端也需要做同样的事情.

## 3. 实施方案

本节描述了服务器端和客户端的具体实现.

### 3.1. 服务器端

- 服务器端采用Elixir语言作为开发语言和运行环境
- 采用 Absinthe 包作为 GraphQL 查询的处理模块
- 服务器端使用 `Absinthe.Plug.DocumentProvider.Compiled` 模块读取由 [persistgraphql](https://github.com/apollographql/persistgraphql) 工具生成的`extracted_queries.json`文件来达到和客户端一致.

#### 3.1.1. 服务器端查询转换原理

服务器接收到客户端发过来的查询如下:

```
{
  id: < 查询 ID >,
  variables: < 变量JSON对象 >,
}
```

通过查找ID, 把查询转换为如下形式, 把ID替换为查询文本:

```
{
  query: < GraphQL文档 >,
  variables: < 变量JSON对象 >
}
```

#### 3.1.2. 实现细节

创建一个文档Provider:

```
defmodule MyApp.ExtractedQueryProvider do
  use Absinthe.Plug.DocumentProvider.Compiled

  provide File.read!("/path/to/extracted_queries.json")
  |> Poison.decode!
  |> Map.new(fn {k, v} -> {v, k} end) # invert key/value
end
```

添加该文档 Provider 到 `Absinthe.Plug` 配置:

```
plug Absinthe.Plug,
  schema: MyApp.Schema,
  document_providers: [
    Absinthe.Plug.DocumentProvider.Default,
    MyApp.ExtractedQueryProvider
  ]
```

当构建服务器端项目的时候, 读取 `extracted_queries.json` 文件的内容, 解析, 并编译为Elixir模块, 转换为中间表示, 并执行验证. 这样的持久化查询请求就像通过ID请求一个普通的对象一样.



> **注意:**
> 详细的服务器实现方法,请参考这个 [Issue](https://github.com/absinthe-graphql/absinthe_plug/pull/53)
> 另外Absinthe对持久化查询的支持需要用到最新的代码, 请使用 `absinthe_plug` 的 `v1.3.0-alpha.0` 及以上版本.

### 3.2. 客户端

本节描述了持久化查询的客户端实现.

#### 3.2.1 基本原理

客户端通过读取 `extracted_queries.json` 文件, 把读取到的文件内容转换为一个JSON对象, 客户端向服务器发送请求之前, 先通过把整个GraphQL查询作为对象的字符串Key, 去找到对应的ID, 通过查找到的ID去执行GraphQL查询.

这个`GraphQL查询文本`到`查询ID`映射的JSON对象格式如下:

```
{
 < print(transform(GraphQL Document)) >: < ID >,
}
```

> Javascript 只允许字符串和数字作为键, 因此我们通过 `graphql-js` 模块的 `print` 函数来解决这个问题.

最终, 客户端通过下面的伪代码执行 GraphQL 查询:

```
query(request: Request) {
 // 在 request.query 结构中查找对应的ID
 // 找到对应的ID
 // 通过GraphQL查询变量把ID传递给服务器
 // 返回一个Promise对象用于获取服务器的返回结果
}
```

#### 3.2.2 实现细节

安装 `persistgraphql` 命令行工具:

```
npm install --save persistgraphql
# 或
yarn add persistgraphql
# 全局安装
yarn global add persistgraphql
```

`persistgraphql` 有两个命令行参数, 分别表示输入和输出, 输入可以是当个文件或者目录名称, 路径可以使相对或绝对路径, 安装好后, 可以执行:

```
persistgraphql queries.graphql
```

生成 `extracted_queries.json` 文件. 如果指定的输入是一个目录, 那么 persistgraphql 会去查找该目录下的所后缀为 `.graphql` 的文件. 你可以用[GitHunt-React](https://github.com/apollographql/GitHunt-React) 这个项目来实际练习一下.

```
git clone git@github.com:apollographql/GitHunt-React.git
cd GitHunt-React
persistgraphql ui/graphql extracted_queries.json
```

客户的实现方法可参考资料: [使用Apollo Client持久化GraphQL查询](https://dev-blog.apollodata.com/persisted-graphql-queries-with-apollo-client-119fd7e6bba5)


## 4. 参考资料

- [Absinthe 对持久化查询的支持](https://github.com/absinthe-graphql/absinthe/issues/266)
- [使用Apollo Client持久化GraphQL查询](https://dev-blog.apollodata.com/persisted-graphql-queries-with-apollo-client-119fd7e6bba5)

  [1]: https://segmentfault.com/img/bVLlAW
  [2]: https://segmentfault.com/img/bVLlA8
  [3]: https://segmentfault.com/img/bVLlB8
  [4]: https://segmentfault.com/img/bVLlCf
