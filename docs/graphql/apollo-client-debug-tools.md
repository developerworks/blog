[Apollo Client Chrome DevTools][1] 是一个和 Redux 开发工具类似的工具, 用户Apollo客户端的调试.
![图片描述][5]

## GraphiQL 控制台

可以在GraphiQL控制台中, 输入各种查询, 并执行查询, 获取结果
![clipboard.png][2]

GraphiQL控制台分为三列,

- 第一列包含两栏, 第一栏是GraphQL查询, 第二栏是查询变量, 查询变量是一个JSON对象
- 第二列为查询结果, 也是一个JSON对象.
- 第三栏默认不显示, 是可以展开的, 其中显示了GraphQL的模式结构和文档说明.

## 可视化的存储查看器

![clipboard.png][3]

可视化的存储查看器, 可以帮助你查看客户端缓存的状态, 以及搜索其中的键和值. 我们把GraphQL的数据存储看做一棵树.

## 查询监视器

查询监视器可以查看所有特定页面上被监视的查询, 包括其加载状态, 正在使用什么变了, 还有如果使用React, 以及查看那些React组件附加到查询之上.

这个功能对开发大型的, 由多个查询构成的单个页面是非常有用的. 它能帮助精确的理解应用程序正在执行哪些查询, 以及他们如何和UI组件进行关联的. 一个最好用的功能是`Run in GraphiQL`按钮, 它能够复制查询和变量到GraphiQL之中, 并且获取查询结果.

![clipboard.png][7]


## 安装和配置

安装连接在这里: [Chrome Webstore][4], 需要爬梯子.

当你的应用处于开发模式的时候, Devtools 的 "Apollo" 标签会出现在Chrome Inspector中. 如果想在产品环境中也打开Apollo调试工具, 可以传递选项 `connectToDevTools: true` 给 `ApolloClient` 构造函数. 如传递 `connectToDevTools: false` 可以手动禁止调试工具.

> **注:** 修改了`connectToDevTools `之后需要重新打开 Devtools 开发工具

## 参考资料

- [Apollo Client Developer Tools](https://dev-blog.apollodata.com/apollo-client-developer-tools-ff89181ebcf)
- [GraphQL Devtools: Easier Developing for Happier Product Devs - Danielle Man][6]
- [Apollo Client Developer Tools 源码仓库](https://github.com/apollographql/apollo-client-devtools)

  [1]: https://github.com/apollostack/apollo-client-devtools
  [2]: https://segmentfault.com/img/bVKPNA
  [3]: https://segmentfault.com/img/bVKPNF
  [4]: https://chrome.google.com/webstore/detail/apollo-client-developer-t/jdkknkkbebbapilgoeccciglkfbmbnfm
  [5]: https://segmentfault.com/img/bVKTwN
  [6]: https://www.youtube.com/watch?v=3mkgOnSdg2c
  [7]: https://segmentfault.com/img/bVKPOw
