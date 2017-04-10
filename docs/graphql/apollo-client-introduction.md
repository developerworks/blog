本文是在React应用程序中使用Apollo Javascript GraphQL客户端和`react-apollo`集成包的官方指南. 上手指南可以参考 [Learn Apollo](https://www.learnapollo.com).

Apollo 社区开发和维护了许多用于简化 GraphQL使用的工具, 支持不同的前端和服务器技术. 虽然本指南只关注与 与React的集成.

Apollo 还支持原始移动设备客户的, 这里有一个处于开发中的[iOS 客户的库](https://github.com/apollostack/apollo-ios), Android 客户的还在计划当中. 本文中描述的集成方法可以不加修改的用于 React Native 的两个平台(iOS, Android).

## 兼容性

- 直接支持React Native
- Redux: Apollo客户端内部使用了Redux管理前端应用的状态.
- 独立于客户端路由, 你可以选择任何你喜欢的路由库, 比如React Router
- 支持任何GraphQL 服务器

## 和其他GraphQL客户端的比较

对于使用 `react-apollo`还是其他的GraphQL客户端库, 考虑一下项目的目标是有价值的.

- [Relay][1] 是一个为了构建移动应用开发的React相关的GraphQL客户端. Apollo 有和Relay 类似的功能, 但它的目标是作为一个通用的工具和任何模式, 认识前端价格一起使用. Relay是作为一个中间层重度耦合在前端和后端之间的, 缺少一些灵活性.
- [Lokka][2] 是一个简单的GraphQL Javascript客户端, 支持基本查询缓存. Apollo 更复杂, 包括更成熟的缓存和一组更新和获取数据的高级功能.

## 学习资料

- [GraphQL.org][3] 关于GraphQL查询语言的简介和参考资料.
- [Apollodata][4] 学习Apollo开源工具的网站
- [博客][5] 包含关于GraphQL的详细信息


  [1]: https://facebook.github.io/relay/
  [2]: https://github.com/kadirahq/lokka
  [3]: http://graphql.org/
  [4]: http://www.apollodata.com/
  [5]: https://medium.com/apollo-stack
