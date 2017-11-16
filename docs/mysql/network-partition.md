## MySQL 5.7: 网络分区处理

对于 MariaDb 的 Galera 集群来说, 网络连接断开后, 从整个集群的角度看, 仍然存在和发的大多数(**quorum**). 当失败的节点重新连接到集群后, 它会重新连接到主节点, 连接恢复后, 分区的节点会自动同步集群并重新加入汲取.

但是对于MySQL复制组来说, 分区的节点已经被集群开除, 并且当连接恢复时不会自动重新加入集群. 当网络断开后, 我们可以在MySQL的错误日志文件中看到如下信息:

```
2017-01-17T11:12:18.562305Z 0 [ERROR] Plugin group_replication reported: 'Member was expelled from the group due to network failures, changing member status to ERROR.'
2017-01-17T11:12:18.631225Z 0 [Note] Plugin group_replication reported: 'getstart group_id ce427319'
2017-01-17T11:12:21.735374Z 0 [Note] Plugin group_replication reported: 'state 4330 action xa_terminate'
2017-01-17T11:12:21.735519Z 0 [Note] Plugin group_replication reported: 'new state x_start'
2017-01-17T11:12:21.735527Z 0 [Note] Plugin group_replication reported: 'state 4257 action xa_exit'
2017-01-17T11:12:21.735553Z 0 [Note] Plugin group_replication reported: 'Exiting xcom thread'
2017-01-17T11:12:21.735558Z 0 [Note] Plugin group_replication reported: 'new state x_start'
```

我们查看复制组成员的状态, 看到:

```
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| CHANNEL_NAME | MEMBER_ID | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| group_replication_applier | 329333cd-d6d9-11e6-bdd2-0242ac130002 | f18ff539956d | 3306 | ERROR |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
1 row in set (0.00 sec)
```

看起来唯一的方法是手工重新启动组复制.

```
mysql> START GROUP_REPLICATION;
ERROR 3093 (HY000): The START GROUP_REPLICATION command failed since the group is already running.

mysql> STOP GROUP_REPLICATION;
Query OK, 0 rows affected (5.00 sec)

mysql> START GROUP_REPLICATION;
Query OK, 0 rows affected (1.96 sec)

mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| CHANNEL_NAME | MEMBER_ID | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
| group_replication_applier | 24d6ef6f-dc3f-11e6-abfa-0242ac130004 | cd81c1dadb18 | 3306 | ONLINE |
| group_replication_applier | 329333cd-d6d9-11e6-bdd2-0242ac130002 | f18ff539956d | 3306 | ONLINE |
| group_replication_applier | ae148d90-d6da-11e6-897e-0242ac130003 | 0af7a73f4d6b | 3306 | ONLINE |
+---------------------------+--------------------------------------+--------------+-------------+--------------+
3 rows in set (0.00 sec
```
