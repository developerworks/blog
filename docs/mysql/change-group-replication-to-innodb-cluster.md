> MySQL InnoDB 集群是一堆集合到一起工作的产品, 用于提供一个完整的高可用方案.

- 可以使用MySQL Shell来配置MySQL服务器组以创建一个MySQL InnoDB数据库集群.
- 默认为**单主**模式, 集群中的服务器只有一个能写.
- 余下的服务器作为主服务器的备用.
- 创建集群至少需要三台机器.
- 客户端应用程序通过MySQL Router连接到主服务器.
- 如果主服务器失败, 集群自动选主, MySQL Router自动把连接路由到**新主**上
- 高级用于可以配置集群有**多主**

由于前面我们已经创建了一个复制组.

```
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+----------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST    | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+----------------+-------------+--------------+
| group_replication_applier | 255a577c-c68f-11e7-a0a7-00163e0c0288 | 172.18.149.215 |        3306 | ONLINE       |
| group_replication_applier | 77befd8c-c68e-11e7-9623-00163e0af475 | 172.18.149.213 |        3306 | ONLINE       |
| group_replication_applier | a44c7ec7-c68e-11e7-9cf8-00163e0afda9 | 172.18.149.214 |        3306 | ONLINE       |
+---------------------------+--------------------------------------+----------------+-------------+--------------+
3 rows in set (0.00 sec)
```

现在我们只需要安装MySQL Router和MySQL Shell

```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.9-1_all.deb
dpkg -i mysql-apt-config_0.8.9-1_all.deb
```

查看版本信息

```
root:~# mysqlrouter --version
MySQL Router v2.1.4 on Linux (64-bit) (GPL community edition)
```

## 使用 MySQL Shell 配置集群

登录

```
root:~# mysqlsh --uri root@172.18.149.213:3306
Creating a Session to 'root@172.18.149.213:3306'
Enter password:
Your MySQL connection id is 80
Server version: 5.7.20-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
MySQL Shell 1.0.10

Copyright (c) 2016, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type '\help' or '\?' for help; '\quit' to exit.

Currently in JavaScript mode. Use \sql to switch to SQL mode and execute queries.
mysql-js>
```

创建集群

```
mysql-js> var cluster = dba.createCluster('dc', {adoptFromGR: true});
```

执行成功后会自动创建一个新的数据库: **mysql_innodb_cluster_metadata**, 用于存放集群的元数据, 该元数据会被MySQL Router用到, 以实现高可用性.

## 参考资料


- [MySQL正式发布高可用架构——MySQL InnoDB Cluster](https://www.hi-linux.com/posts/41986.html)
- [Ubuntu/Debian上的InnoDB集群](http://mysqlserverteam.com/mysql-innodb-cluster-real-world-cluster-tutorial-for-ubuntu-and-debian)
- [初探MySQL InnoDB集群](http://www.imooc.com/article/17440)
- [MySql InnoDB Cluster 生产环境部署](http://www.jianshu.com/p/e5c2f4b3a8a5)
- [从已有的复制组创建InnoDB集群](https://ronniethedba.wordpress.com/2017/04/23/creating-an-innodb-cluster-router-from-an-existing-group-replication-deployment/)
- [Mysql 高可用 InnoDB Cluster 多节点搭建过程](http://www.php230.com/1493151387.html)
- [Adopting a Group Replication Deployment](https://dev.mysql.com/doc/refman/5.7/en/mysql-innodb-cluster-from-group-replication.html)
- [从已有的组复制搭建InnoDB Cluster环境](http://wangshengzhuang.com/2017/05/08/%E6%95%B0%E6%8D%AE%E5%BA%93%E7%9B%B8%E5%85%B3/MySQL/MySQL%20HA/InnoDB%20Cluster/%E4%BB%8E%E5%B7%B2%E6%9C%89%E7%9A%84%E7%BB%84%E5%A4%8D%E5%88%B6%E6%90%AD%E5%BB%BAInnoDB%20Cluster%E7%8E%AF%E5%A2%83)


## dba.createCluster('dc', {adoptFromGR: true}) 的BUG

```
root:~# mysqlsh --version
mysqlsh   Ver 1.0.10 for Linux on x86_64 - for MySQL 5.7.19 (MySQL Community Server (GPL))
```

从现有的复制组创建 InnoDB 集群的问题: https://bugs.mysql.com/bug.php?id=87599, MySQL Shell 的一个BUG, 降级到 1.0.9 正常.



