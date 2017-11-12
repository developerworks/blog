## MySQL Router

前面一片文章说了如何配置MySQL的分组复制. 基于分组复制的机制, 当主节点崩溃离开集群, 剩余的其他节点会相互协商, 然后选举一个新的主节点. 这里有一个问题, 就是应用程序端如果连接到了主节点, 这时主节点崩溃离开集群. 可用的数据库IP地址发生变化. 客户端应用程序这个时候还是会想失败的节点尝试连接, 虽然可以修改客户端应用程序的连接配置, 但是这种情况基本是不现实的.

虽然我们可以通过下面的SQL语句获取主节点的IP地址

```
SELECT * FROM performance_schema.replication_group_members
    WHERE MEMBER_ID = (
        SELECT VARIABLE_VALUE
        FROM performance_schema.global_status
        WHERE VARIABLE_NAME= 'group_replication_primary_member'
   );
```

但是通过应用程序动态的获取可用数据库的IP地址. 这种方式感觉不怎么好. 很麻烦, 要写多余的代码.

> 配置MySQL Router首先需要MySQL Shell 工具, 在 MySQL Shell 部分有详细的说明

## 概述

下面是MySQL Router 和集群的基本关系图

![mysql-router-positioning.png](https://dev.mysql.com/doc/mysql-router/2.1/en/images/mysql-router-positioning.png)

上图充分说明了, MySQL Router 在InnoDB集群里面的角色. 主要作用是为数据库集群提供一个虚拟IP. 作为应用程序单一连接点. 通过这个单一的连接点实现负载均衡, 读写分离, 故障转移等数据库高可用方案.



推荐安装在应用程序所在的机器上, 原因包括:

- 通过Unix套接字连接, 而不是TCP/IP, 提升性能
- 降低网络延迟
- MySQL实例不需要额外的账号, 只需要一个 router@198.51.100.45, 而不是 myapp@%
- 提升应用程序服务器的弹性

## 安装和配置 MySQL Router

```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.9-1_all.deb
dpkg -i mysql-apt-config_0.8.9-1_all.deb
aptitude update
aptitude install -y mysql-router
```

### 生成配置和启动脚本

```
mysqlrouter --bootstrap 172.18.149.213:3306 --directory /data/mysqlrouter --user=root --conf-use-sockets --force
```

```
Please enter MySQL password for root:

Bootstrapping MySQL Router instance at /data/mysqlrouter...
MySQL Router  has now been configured for the InnoDB cluster 'dc'.

The following connection information can be used to connect to the cluster.

Classic MySQL protocol connections to cluster 'dc':
- Read/Write Connections: localhost:6446
- Read/Write Connections: /data/mysqlrouter/mysql.sock
- Read/Only Connections: localhost:6447
- Read/Only Connections: /data/mysqlrouter/mysqlro.sock

X protocol connections to cluster 'dc':
- Read/Write Connections: localhost:64460
- Read/Write Connections: /data/mysqlrouter/mysqlx.sock
- Read/Only Connections: localhost:64470
- Read/Only Connections: /data/mysqlrouter/mysqlxro.sock
```

输入密码后, 文件就生成到 /data/mysqlrouter 目录了.

> **--conf-use-sockets** 选项还会监听Unix套接字


### 启动MySQL Router


```
cd /data/mysqlrouter
./start.sh
PID 3976 written to /data/mysqlrouter/mysqlrouter.pid
```

### 查看MySQL Router的监听端口

下面列出了MySQL Router 监听的TCP端口

```
netstat -anpt |grep router
tcp        0      0 0.0.0.0:64460           0.0.0.0:*               LISTEN      3976/mysqlrouter
tcp        0      0 0.0.0.0:6446            0.0.0.0:*               LISTEN      3976/mysqlrouter
tcp        0      0 0.0.0.0:6447            0.0.0.0:*               LISTEN      3976/mysqlrouter
tcp        0      0 0.0.0.0:64470           0.0.0.0:*               LISTEN      3976/mysqlrouter
tcp        0      0 172.18.149.215:44350    172.18.149.213:3306     ESTABLISHED 3976/mysqlrouter
```

> 各个端口的含义我们在之前创建MySQL Router配置时的输出已经说明了, 另外配置文件中也详细说明了每个端口的含义:

```
cat /data/mysqlrouter/mysqlrouter.conf
```

```
# File automatically generated during MySQL Router bootstrap
[DEFAULT]
user=root
logging_folder=/data/mysqlrouter/log
runtime_folder=/data/mysqlrouter/run
data_folder=/data/mysqlrouter/data
keyring_path=/data/mysqlrouter/data/keyring
master_key_path=/data/mysqlrouter/mysqlrouter.key

[logger]
level = DEBUG

[metadata_cache:dc]
router_id=1
bootstrap_server_addresses=mysql://172.18.149.213:3306,mysql://172.18.149.215:3306,mysql://172.18.149.214:3306
user=mysql_router1_4d98tioywyow
metadata_cluster=dc
ttl=300

[routing:dc_default_rw]
bind_address=0.0.0.0
bind_port=6446
socket=/data/mysqlrouter/mysql.sock
destinations=metadata-cache://dc/default?role=PRIMARY
mode=read-write
protocol=classic

[routing:dc_default_ro]
bind_address=0.0.0.0
bind_port=6447
socket=/data/mysqlrouter/mysqlro.sock
destinations=metadata-cache://dc/default?role=SECONDARY
mode=read-only
protocol=classic

[routing:dc_default_x_rw]
bind_address=0.0.0.0
bind_port=64460
socket=/data/mysqlrouter/mysqlx.sock
destinations=metadata-cache://dc/default?role=PRIMARY
mode=read-write
protocol=x

[routing:dc_default_x_ro]
bind_address=0.0.0.0
bind_port=64470
socket=/data/mysqlrouter/mysqlxro.sock
destinations=metadata-cache://dc/default?role=SECONDARY
mode=read-only
protocol=x
```

下面列举除了MySQL Router监听的Unix套接字(如果使用了**--conf-use-sockets**选项)

```
netstat -anpx |grep router
unix  2      [ ACC ]     STREAM     LISTENING     41668    3791/mysqlrouter    /tmp/myrouter/mysql.sock
unix  2      [ ACC ]     STREAM     LISTENING     42187    3791/mysqlrouter    /tmp/myrouter/mysqlro.sock
unix  2      [ ACC ]     STREAM     LISTENING     43361    3791/mysqlrouter    /tmp/myrouter/mysqlx.sock
unix  2      [ ACC ]     STREAM     LISTENING     42189    3791/mysqlrouter    /tmp/myrouter/mysqlxro.sock
```

我们看到有4个Unix套接字

| Unix 套接字  | 说明  | 读写模式 |
|-------------| ---- | ---- |
| **mysql.sock**  |  MySQL协议 | 读写 |
| **mysqlro.sock** | MySQL协议 | 只读  |
| **mysqlx.sock**  | X 协议 | 读写  |
| **mysqlxro.sock**  | X 协议   | 只读 |

> 注意: 该表只是说明了每个Unix套接字承担的角色, 没有限制你连接到只读模式的端口执行写操作.

### 连接到 MySQL Router 执行SQL

```
mysql -u root -h 172.18.149.215 -P 6446 -p
# 创建数据库
mysql> create database b;
Query OK, 1 row affected (0.00 sec)
# 切换
mysql> use b;
Database changed
# 创建表
mysql> create table a (c int);
Query OK, 0 rows affected (0.00 sec)
# 插入值
# 出错了, 因为没有主键
mysql> insert into a values(1);
ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.

# 添加主键列

mysql> alter table a add column a_id int(4);
Query OK, 0 rows affected (0.00 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> alter table a add primary key pk (a_id);
Query OK, 0 rows affected (0.00 sec)
Records: 0  Duplicates: 0  Warnings: 0

# 插入值
mysql> insert into a values (1,2);
Query OK, 1 row affected (0.01 sec)
```

> OK, MySQL Router到这里就配置好了, 在应用程序代码里面直接连接到MySQL Router的IP地址可以了.


## MySQL Shell

- 交互式代码执行
- 支持的语言: Javascript, Python, SQL
- 批处理
- 输出格式: 制表符, 表格, JSON
- 多行支持
- 日志
- X Protocol 支持, 遗留协议支持


### 安装

```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.9-1_all.deb
dpkg -i mysql-apt-config_0.8.9-1_all.deb
aptitude update
aptitude install -y mysql-shell
```

### 连接方式: 通过URL的方式

- mysqlx://user@localhost:33065
- mysql://user@localhost:3333
- mysqlx://user@server.example.com/
- mysqlx://user@198.51.100.14:123
- mysqlx://user@[2001:db8:85a3:8d3:1319:8a2e:370:7348]
- mysqlx://user@198.51.100.1/world%5Fx
- mysqlx://user@198.51.100.2:33060/world

### 连接方式: 通过参数方式

- --dbuser (-u) value
- --dbpassword value
- --host (-h) value
- --port (-P) value
- --schema (-D) value
- --password (-p)
- --socket (-S)

### 参数别名

- --user is equivalent to --dbuser
- --password is equivalent to --dbpassword
- --database is equivalent to --schema

### 覆盖问题

参数的优先级比URL高, 下面的例子通过 otheruser 用户进行连接而不是URL中指定的 user

```
mysqlsh --uri user@localhost:33065 --user otheruser
```

```
# MySQL协议
mysqlsh --mysql -u user -h localhost
# X协议
mysqlsh --mysqlx -u user -h localhost -P 33065
```


### 通过Javascript连接

```
# X协议

var session=mysqlx.getSession('root@localhost:33060', 'password');

# MySQL协议

var session = mysql.getClassicSession('root@localhost:3306', 'password');
```

### 在代码中使用加密连接

```
var session=mysqlx.getSession({host: 'localhost',
  dbUser: 'root',
  dbPassword: 'password',
  ssl_ca: "path_to_ca_file",
  ssl_cert: "path_to_cert_file",
  ssl_key: "path_to_key_file"
});
```

### 安装X插件的集中方式

> 如果从支持X Protocol的客户端连接到数据库, 在数据库端需要安装X Plugin以提供对 X Protocol的支持, 安装X Plugin只需要对 **mysql.plugin** 表有 **INSERT** 权限即可.

**1.通过 mysqlsh 安装**

```
# 安装需要的仓库, 选中 MySQL Tools & Connectors (Currently selected: Enabled)
wget https://dev.mysql.com/get/mysql-apt-config_0.8.9-1_all.deb
dpkg -i mysql-apt-config_0.8.9-1_all.deb
```

执行如下命令, 输入密码:

```
mysqlsh -u root -h localhost --classic --dba enableXProtocol
```

如果看到如下输出, 标识安装完成:

```
mysqlsh -u root -h 172.18.149.213 --classic --dba enableXProtocol
```

```
Creating a Classic Session to 'root@172.18.149.213'
Enter password:
Your MySQL connection id is 70
Server version: 5.7.20-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
enableXProtocol: Installing plugin mysqlx...
enableXProtocol: done
```

**2.通过MySQL客户端**

```
mysql -u user -p
mysql> INSTALL PLUGIN mysqlx SONAME 'mysqlx.so';
```

> X 插件的加载需要 mysql.session 用户, mysql.session 用户在 MySQL 5.7.19 中已经添加. 早期的版本需要执行 mysql_upgrade, 否则加载X插件的时候回报错:
> There was an error when trying to access the server with user: mysql.session@localhost. Make sure the user is present in the server and that mysql_upgrade was ran after a server update..


### 验证

```
mysqlsh -u root --sqlc -e "show plugins"
mysql -u root -p -e "show plugins"

mysql> select * from plugin;
+-------------------+----------------------+
| name              | dl                   |
+-------------------+----------------------+
| group_replication | group_replication.so |
| mysqlx            | mysqlx.so            |
+-------------------+----------------------+
```

如果在输出中看到 mysqlx.so 标识X插件安装成功. X插件安装过程会自动创建一个用户: mysqlxsys@localhost

### 卸载X插件

```
UNINSTALL PLUGIN mysqlx;
```

删除插件的同时也会删除 mysqlxsys@localhost 用户.



## 参考资料


https://dev.mysql.com/doc/refman/5.7/en/mysqlsh.html
https://dev.mysql.com/doc/mysql-router/2.1/en/mysqlrouter.html#mysql-router-command-options-bootstrap
https://dev.mysql.com/doc/mysql-router/2.1/en/mysql-router-deploying-bootstrapping.html
https://dev.mysql.com/doc/mysql-router/2.1/en/
