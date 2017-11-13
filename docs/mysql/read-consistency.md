## MySQL 5.7: 组复制(MySQL Group Replication)的问题

> MGR为我们解决了跨节点复制的数据一致性问题.但是MGR解决的是最终一致性. 当主节点发生故障, 内部会执行此选举, 选举出新的主节点. MySQL Router会感知到这个变化. 只要你的应用程序是连接到MySQL Router的, 那么应用程序端是感知不到这个变化的. 因此无需对应用程序进行重新配置或代码修改.

MGR目前的问题, MGR只是消除了传统主从复制带来的数据一致性问题. MGR的高可用还要解决下面几个问题.

- 跨节点的一致性读(使用ProxySQL 解决这个问题)
- 应用程序端故障感知(MySQL Router解决了这个问题)

### MySQL Router 和 ProxySQL

### 跨节点的一致性读


### 应用程序端故障感知


## 中间件之战

MySQL Proxy 十年前的技术, 出生在2007. 2014年结束生命周期.
Mariadb MaxScale




## 参考资料

http://www.voidcn.com/article/p-mhjegnns-bao.html
http://www.voidcn.com/link?url=http://lefred.be/content/ha-with-mysql-group-replication-and-proxysql/
https://www.slideshare.net/bytebot/the-proxy-wars-mysql-router-proxysql-mariadb-maxscale
