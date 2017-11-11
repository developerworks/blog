## 组复制的限制

- 存储引擎必须为**Innodb**
- 每个表必须提供**主键**
- 只支持**ipv4**，网络需求较高
- 一个组最多只能有**9台**服务器
- 不支持 Replication event checksums，
- 不支持 Savepoints
- **多主模式**不支持SERIALIZABLE事务隔离级别
- **多主模式**不能完全支持级联外键约束
- **多主模式**不支持在不同节点上对同一个数据库对象并发执行DDL(在不同节点上对同一行并发进行RW事务，后发起的事务会失败)

## 配置过程

每个节点执行如下命令安装官方仓库的MySQL Server

```
apt-get update
cd /tmp
wget https://dev.mysql.com/get/mysql-apt-config_0.8.9-1_all.deb
dpkg -i mysql-apt-config_0.8.9-1_all.deb
apt-get update
aptitude install -y mysql-server
```

验证复制插件是否存在:

```
root@iZwz98pmxwulw67n9gxnl2Z:/etc/mysql# ll /usr/lib/mysql/plugin/
total 4120
drwxr-xr-x 3 root root    4096 Nov 11 11:14 ./
drwxr-xr-x 3 root root    4096 Nov 11 11:14 ../
...
-rw-r--r-- 1 root root 1751560 Sep 14 01:01 group_replication.so
...
```

> 注意: 不要安装Ubuntu 16.04自带的MySQL, 16.04自带的MySQL安装后没有 group_replication.so 这个东西, 一定要通过 **mysql-apt-config_0.8.9-1_all.deb** 提供的仓库安装. MySQL 官方版本的最新版本的仓库可以在这里下载: https://dev.mysql.com/downloads/repo/apt/

每个节点创建目录, 用于放置MGR(MySQL Group Replication)的配置文件

```
mkdir /etc/mysql/mgr.d
```

### 修改MGR配置文件

配置文件在本文Git仓库Markdown文件相同目录下, 文件名称依次为:

```
mgr-01.conf
mgr-02.conf
mgr-03.conf
```

修改上述三个文件对应的IP地址, 详细的说明参考: https://www.howtoing.com/how-to-configure-mysql-group-replication-on-ubuntu-16-04/

生成复制组所需要的UUID备用.

```
root# uuidgen
00d17eae-73c8-4a7d-abf5-051bb68a9d7d
```

用上面生成的UUID替换 loose-group_replication_group_name

### 上传配置文件

```
#!/bin/bash
# mkdir /etc/mysql/mgr.d
scp mgr-01.conf root@172.18.149.213:/etc/mysql/mgr.d/mgr.cnf
scp mgr-02.conf root@172.18.149.214:/etc/mysql/mgr.d/mgr.cnf
scp mgr-03.conf root@172.18.149.215:/etc/mysql/mgr.d/mgr.cnf
```

在每一个节点的 /etc/mysql/my.cnf 文件最后一行添加如下指令:

```
!includedir /etc/mysql/mgr.d/
```

## 更换MySQL的数据盘

双十一新购了3台本地SSD的ECS, 想把MySQL的数据目录移动到独立的SSD(/dev/vdb)上. 因此停止数据指标移动数据目录:


### 格式化磁盘

```
fdisk /dev/vdb
```

创建挂载点

```
mkdir /data
```

创建文件系统

```
mkfs.ext4 /dev/vdb1
```

查看磁盘UUID

```
blkid
```

复制 /dev/vdb1 的UUID, 在 /etc/fstab 下添加:

```
UUID=${UUID} /data           ext4    errors=remount-ro 0       1
```

用你自己的磁盘UUID替换 **${UUID}**

挂载文件系统

```
mount /dev/vdb1 /data
```

创建软连接并启动MySQL

```
mv /var/lib/mysql /data
ln -s /data/mysql /var/lib/mysql
systemctl start mysql
```

### 目录权限问题

启动数据库查看日志(**tail -f /var/log/mysql/error.log**)发现如下错误消息:

```
2017-11-11T03:24:49.003621Z 0 [ERROR] InnoDB: The innodb_system data file 'ibdata1' must be writable
2017-11-11T03:24:49.003637Z 0 [ERROR] InnoDB: The innodb_system data file 'ibdata1' must be writable
2017-11-11T03:24:49.003642Z 0 [ERROR] InnoDB: Plugin initialization aborted with error Generic error
2017-11-11T03:24:49.604050Z 0 [ERROR] Plugin 'InnoDB' init function returned error.
2017-11-11T03:24:49.604070Z 0 [ERROR] Plugin 'InnoDB' registration as a STORAGE ENGINE failed.
2017-11-11T03:24:49.604076Z 0 [ERROR] Failed to initialize plugins.
2017-11-11T03:24:49.604079Z 0 [ERROR] Aborting
```

### 解决办法

编辑如下文件

```
vi /etc/apparmor.d/usr.sbin.mysqld
```

在文件末尾 **}** 的上一行添加下面两行:

```
/data/mysql/ r,
/data/mysql/** rwk,
```

**/data/mysql** 为新的数据目录路径. 上面两行的作用是分配目录的读写权限, Ubuntu 16.04 默认的MySQL数据目录为 /var/lib/mysql.

## 复制配置

进入MySQL控制台

```
mysql -uroot -p
```


在所有节点执行如下命令:

```
SET SQL_LOG_BIN=0;
CREATE USER 'repl'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;

CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
INSTALL PLUGIN group_replication SONAME 'group_replication.so';

# 验证插件是否安装成功
mysql> SHOW PLUGINS;
+----------------------------+----------+--------------------+----------------------+---------+
| Name                       | Status   | Type               | Library              | License |
+----------------------------+----------+--------------------+----------------------+---------+
...
| group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | GPL     |
+----------------------------+----------+--------------------+----------------------+---------+

```

### 配置第一个节点

```
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
```

### 配置余下节点


```
START GROUP_REPLICATION;
```

### 加入组的问题


> 事务问题, 各个节点的事务状态不一致.
> ```
> 2017-11-10T19:07:26.918531Z 0 [ERROR] Plugin group_replication reported: 'This member has more > executed transactions than those present in the group. Local transactions: c3c274ff-c63e-11e7-> b339-00163e0c0288:1-4 > Group transactions: 2e6bfa69-0439-41c9-add7-795a9acfd499:1-10,
> c5898a84-c63e-11e7-bc8b-00163e0af475:1-4'
> ```

解决办法:
http://blog.csdn.net/yuanlin65/article/details/53782020

```
set global group_replication_allow_local_disjoint_gtids_join=ON;
```


## IP地址变化的问题

对于一个3节点的单主集群来说, 当主节点挂了, 另外两个节点会自动选主. 其中一个会成为主节点, 并自动切换为读写模式.
因为对于单主模式来说, 只有主节点能够执行写操作. 那么我们如何知道主节点的IP地址呢?

可以在任意一个MySQL节点上通过如下SQL获取主节点的IP地址

```
SELECT * FROM performance_schema.replication_group_members WHERE MEMBER_ID = (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME= 'group_replication_primary_member');
```


## 参考资料

https://dev.mysql.com/doc/refman/5.7/en/group-replication-launching.html
https://www.howtoing.com/how-to-configure-mysql-group-replication-on-ubuntu-16-04/
http://blog.csdn.net/yuanlin65/article/details/53782020
https://stackoverflow.com/questions/41356052/how-mysql-group-replication-get-primary-node-ip-address

