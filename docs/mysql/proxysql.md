MySQL 5.7: 使用ProxySQL实现应用程序的高可用

下图演示了故障转移的过程.

![故障转移](http://img.voidcn.com/vcimg/000/005/595/164_95d_2a5.png)

## 安装

wget https://github.com/sysown/proxysql/releases/download/v1.4.3/proxysql_1.4.3-ubuntu16_amd64.deb

dpkg -i proxysql_1.4.3-ubuntu16_amd64.deb

proxysql --version


mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt='ProxySQL> '

## 配置

配置读写组

```
# 读写组
ProxySQL> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (1,'172.18.149.213',3306);
# 只读组
ProxySQL> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (2,'172.18.149.214',3306);
ProxySQL> INSERT INTO mysql_servers(hostgroup_id,hostname,port) VALUES (2,'172.18.149.215',3306);
```

用户配置

```
insert into mysql_users(username, password,active,transaction_persistent) values('root', '!@#E9i6vXEqpvTRPauLfanQCXpZvKgiB', 1, 1);
```

添加调度器

```
```

应用配置

```
load mysql users to runtime;
load mysql servers to runtime;
load mysql variables to runtime;
```

## 多层配置系统


- 允许对配置进行动态更新
- 允许修改尽可能多的配置而不需要重启服务器
- 允许无效配置的回滚

这些都是通过多层配置系统来完成的.

```
+-------------------------+
|         RUNTIME         |
+-------------------------+
       /|\          |
        |           |
    [1] |       [2] |
        |          \|/
+-------------------------+
|         MEMORY          |
+-------------------------+ _
       /|\          |      |\
        |           |        \
    [3] |       [4] |         \ [5]
        |          \|/         \
+-------------------------+  +-------------------------+
|          DISK           |  |       CONFIG FILE       |
+-------------------------+  +-------------------------+
```

包含3级配置.

- 运行时
包含一个内存数据结构, ProxySQL的线程用它来处理请求
- 内存
是一个内存数据库, 与MySQL接口兼容, 你可以使用SQL语句来操作它
- 磁盘
一个SQLite3数据库用于存储配置信息
- 配置文件


登录ProxySQL

```
```


多层配置在ProxySQL中表现为数据库, 可通过SHOW DATABASES列举所有的配置层

```
proxysql> SHOW DATABASES;
+-----+---------+-------------------------------+
| seq | name    | file                          |
+-----+---------+-------------------------------+
| 0   | main    |                               |
| 2   | disk    | /var/lib/proxysql/proxysql.db |
| 3   | stats   |                               |
| 4   | monitor |                               |
+-----+---------+-------------------------------+
```

其中 main 就是内存数据库, disk 为磁盘SQLite3数据库文件, stats


具体的配置表现为ProxySQL里面的表, SHOW TABLES可以显示所有的配置表:

```
proxysql> SHOW TABLES;
+--------------------------------------------+
| tables                                     |
+--------------------------------------------+
| global_variables                           |
| mysql_collations                           |
| mysql_group_replication_hostgroups         |
| mysql_query_rules                          |
| mysql_replication_hostgroups               |
| mysql_servers                              |
| mysql_users                                |
| proxysql_servers                           |
| runtime_checksums_values                   |
| runtime_global_variables                   |
| runtime_mysql_group_replication_hostgroups |
| runtime_mysql_query_rules                  |
| runtime_mysql_replication_hostgroups       |
| runtime_mysql_servers                      |
| runtime_mysql_users                        |
| runtime_proxysql_servers                   |
| runtime_scheduler                          |
| scheduler                                  |
+--------------------------------------------+
```


- `mysql_users` 连接到 ProxySQL 的用户凭证, ProxySQL 会用相同的凭证连接到后端数据库.

## 启动过程

首先读取配置文件查找数据目录
如果数据库文件存在, 就读取并加载到内存, 然后通过内存复制到运行时
如果数据库文件不存在, 而配置文件存在, 解析并加载到内存, 然后复制到运行时

需要注意的是, 如果磁盘数据库存在, 是不会解析配置文件的.

## 初始化

## 重启

## 运行时修改配置

## 在多层之间同步配置

## Web管理工具

## 排错



**统计监视权限问题**
/var/lib/proxysql# tail -f /var/lib/proxysql/proxysql.log
2017-11-14 01:28:10 MySQL_Monitor.cpp:396:monitor_connect_thread(): [ERROR] Server 172.18.149.215:3306 is returning "Access denied" for monitoring user



## 参考资料

- [Load balancing with ProxySQL](https://www.percona.com/doc/percona-xtradb-cluster/LATEST/howtos/proxysql.html)
- [ProxySQL官方文档](https://github.com/sysown/proxysql/tree/master/doc)
- [ProxySQL快速上手](http://blog.csdn.net/cug_jiang126com/article/details/73167912)
- [Database Load Balancing for MySQL and MariaDB with ProxySQL - Tutorial](https://severalnines.com/resources/tutorials/proxysql-tutorial-mysql-mariadb)
- http://feed.askmaclean.com/archives/proxysql-admin-interface-is-not-your-typical-mysql-server.html
- [使用 ProxySQL 改进 MySQL SSL 的连接性能](http://blog.jobbole.com/112576/)
- [MySQL中间件之ProxySQL:配置系统](http://www.fordba.com/mysql_proxysql_cf_sys.html)
- [MySQL中间件之ProxySQL:读写分离/查询重写配置](http://www.fordba.com/mysql_proxysql_rw_split.html)
- [http://proxysql.blogspot.jp/2015/09/proxysql-tutorial-setup-in-mysql.html](http://proxysql.blogspot.jp/2015/09/proxysql-tutorial-setup-in-mysql.html)
