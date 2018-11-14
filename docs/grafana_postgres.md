# Collectd

## Collectd 统计项

查看`types.db`文件中定义的Postgresql统计规范

```shell
root@ubuntu:~# cat /usr/share/collectd/types.db |grep pg_
pg_blks           value:DERIVE:0:U
pg_db_size        value:GAUGE:0:U
pg_n_tup_c        value:DERIVE:0:U
pg_n_tup_g        value:GAUGE:0:U
pg_numbackends    value:GAUGE:0:U
pg_scan           value:DERIVE:0:U
pg_xact           value:DERIVE:0:U
```

第二个字段为[数据源类型](https://collectd.org/wiki/index.php/Data_source), `types.db`规范参考[types.db.5.shtml](https://collectd.org/documentation/manpages/types.db.5.shtml)

## 关于 Collectd 的 types.db 数据规范定义文件

types.db - 系统统计收集守护进程collectd的数据集说明

### 大纲

```shell
bitrate value:GAUGE:0:4294967295
counter value:COUNTER:U:U
if_octets rx:COUNTER:0:4294967295, tx:COUNTER:0:4294967295
```

### 描述

- 对每个数据集说明, type.db文件都包含了一行. 每行由两个字段组成, 由空格或者tab分隔.
- 第一个字段定义了数据集的名称, 第二个字段定义了数据源说明的列表, 以空格分隔, 对每一个列表单元都以逗号分隔.
- 数据源说明的格式受到RRDtool的数据源说明格式影响. 每个数据源由4部分组成, 分别为`数据源名`, `类型`, `最小值`和`最大值`, 之间由`:`分隔. `ds-name:ds-type:min:max`.
- 其中`ds-type`包含4中类型, ABSOULUTE, COUNTER, DERIVE或者GAUSE.
- mix和max定义了固定值范围. 如果`U`在min或者max中指定, 则意味着不知道范围.

### 文件

`types.db` 的文件配置在 `collectd.conf` 中. 在Ubuntu中, 该文件的默认位置为 `/usr/share/collectd/types.db`.

### 定制types.db

如果你想指定一个定制的类型, 你可以在默认的 `types.db` 里添加, 或者可以另起一行在下面添加一个新的文件.

```shell
For example:
　　TypesDB "/opt/collectd/share/collectd/types.db"
　　TypesDB "/opt/collectd/etc/types.db.custom"
```

> 注意: 如果你想使用这种方式, 必须在网络中所有系统中都添加该文件.

## Collectd 的 Postgresql 配置

`postgresql` 插件从PostgreSQL数据库中查询统计信息. 它保持一个到所有配置的数据库的连接, 并且当连接中断时重连. 数据库是由一个 `<Database>` 配置块进行配置. 默认统计是从PostgreSQL的统计收集器统计的. 要使这个插件能够正常的工作, 需要启用数据库的统计搜集功能. 参考 [Statistics Collector]()文档

通过使用 `<Query>` 块指定自定义的数据库查询, 可以搜集任何数据.

```xml
<Plugin postgresql>
  <Query locks>
    Statement "
    SELECT COUNT(mode) AS count, mode
      FROM pg_locks GROUP BY mode
    UNION SELECT COUNT(*) AS count, 'waiting' AS mode
      FROM pg_locks
      WHERE granted is false;
    "
    <Result>
      Type "gauge"
      InstancePrefix "pg_locks"
      InstancesFrom "mode"
      ValuesFrom "count"
    </Result>
  </Query>
  <Query seq_scans>
    Statement "
      SELECT CASE WHEN status='OK' THEN 0 ELSE 1 END AS status FROM (
        SELECT get_seq_scan_on_large_tables AS status FROM collectd.get_seq_scan_on_large_tables
      ) AS foo;
    "
    <Result>
      Type "gauge"
      InstancePrefix "pg_seq_scans"
      ValuesFrom "status"
    </Result>
  </Query>
  <Query connections>
    Statement "
      SELECT COUNT(state) AS count, state FROM (SELECT CASE
      WHEN state = 'idle' THEN 'idle'
      WHEN state = 'idle in transaction' THEN 'idle_in_transaction'
      WHEN state = 'active' THEN 'active'
      ELSE 'unknown' END AS state FROM collectd.pg_stat_activity) state
      GROUP BY state
      UNION SELECT COUNT(*) AS count, 'waiting' AS state FROM collectd.pg_stat_activity WHERE waiting;
    "
    <Result>
      Type "pg_numbackends"
      InstancePrefix "state"
      InstancesFrom "state"
      ValuesFrom "count"
    </Result>
  </Query>
  <Query slow_queries>
    Statement "
    SELECT COUNT(*) AS count FROM collectd.pg_stat_activity
    WHERE state='active' and now()-query_start > '300 seconds'::interval
    AND query ~* '^(insert|update|delete|select)';
    "
    <Result>
      Type "counter"
      InstancePrefix "pg_slow_queries"
      ValuesFrom "count"
    </Result>
  </Query>
  <Query txn_wraparound>
    Statement "
    SELECT age(datfrozenxid) as txn_wrap_age FROM pg_database ;
    "
    <Result>
      Type "counter"
      InstancePrefix "txn_wraparound"
      ValuesFrom "txn_wrap_age"
    </Result>
  </Query>
  <Query wal_files>
    Statement "
    SELECT archived_count AS count, failed_count AS failed FROM pg_stat_archiver;
    "
    <Result>
      Type "gauge"
      InstancePrefix "pg_wal_count"
      ValuesFrom "count"
    </Result>
    <Result>
      Type "gauge"
      InstancePrefix "pg_wal_failed"
      ValuesFrom "failed"
    </Result>
  </Query>
  <Query avg_querytime>
    Statement "
    SELECT sum(total_time)/sum(calls) AS avg_querytime FROM collectd.get_stat_statements() ;
    "
    <Result>
      Type "gauge"
      InstancePrefix "pg_avg_querytime"
      ValuesFrom "avg_querytime"
    </Result>
  </Query>
  <Query scans>
    Statement "
    SELECT
      sum(idx_scan) as index_scans,
      sum(seq_scan) as seq_scans,
        sum(idx_tup_fetch) as index_tup_fetch,
        sum(seq_tup_read) as seq_tup_read
    FROM pg_stat_all_tables ;
    "
    <Result>
      Type "pg_scan"
      InstancePrefix "index"
      ValuesFrom "index_scans"
    </Result>
    <Result>
      Type "pg_scan"
      InstancePrefix "seq"
      ValuesFrom "seq_scans"
    </Result>
    <Result>
      Type "pg_scan"
      InstancePrefix "index_tup"
      ValuesFrom "index_tup_fetch"
    </Result>
    <Result>
      Type "pg_scan"
      InstancePrefix "seq_tup"
      ValuesFrom "seq_tup_read"
    </Result>
  </Query>
  <Query checkpoints>
    Statement "
    SELECT (checkpoints_timed + checkpoints_req) AS total_checkpoints
    FROM pg_stat_bgwriter ;
    "
    <Result>
      Type "counter"
      InstancePrefix "pg_checkpoints"
      ValuesFrom "total_checkpoints"
    </Result>
  </Query>
  <Query slave_lag>
    Statement "
    SELECT CASE
      WHEN pg_is_in_recovery = 'false'
      THEN 0
      ELSE COALESCE(ROUND(EXTRACT(epoch FROM now() - pg_last_xact_replay_timestamp())),0)
    END AS seconds
    FROM pg_is_in_recovery();
    "
    <Result>
      Type "counter"
      InstancePrefix "slave_lag"
      ValuesFrom "seconds"
    </Result>
  </Query>
  <Database "test">
    Host "localhost"
    Port "5432"
    User "collectd"
    Password "XXX"
    Query "backends"
    Query "transactions"
    Query "queries"
    Query "table_states"
    Query "disk_io"
    Query "disk_usage"
    Query "query_plans"
    Query "connections"
    Query "slow_queries"
    Query "txn_wraparound"
    Query "locks"
    Query "slave_lag"
    Query "scans"
    Query "checkpoints"
    Query "avg_querytime"
    Query "wal_files"
    Query "seq_scans"
  </Database>
</Plugin>
```

### 自定义查询

#### 缓存命中率

```xml
<Query cache_hit_ratio>
  Statement "
  SELECT sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
  FROM pg_statio_user_tables;
  "
  <Result>
    Type "gauge"
    InstancePrefix "cache_hit_ratio"
    ValuesFrom "cache_hit_ratio"
  </Result>
</Query>
```

#### 索引命中率

```xml
<Query cache_idx_hit_ratio>
  Statement "
  SELECT (sum(idx_blks_hit) - sum(idx_blks_read)) / sum(idx_blks_hit) as cache_idx_hit_ratio
  FROM pg_statio_user_indexes;
  "
  <Result>
    Type "gauge"
    InstancePrefix "cache_idx_hit_ratio"
    ValuesFrom "cache_idx_hit_ratio"
  </Result>
</Query>
```

#### TPS

```xml
<Query tps>
  Statement "
  SELECT datname, xact_commit + xact_rollback AS tps
  FROM pg_catalog.pg_stat_database;
  "
  <Result>
    Type "derive"
    InstancePrefix "tps"
    InstancesFrom "datname"
    ValuesFrom "tps"
  </Result>
</Query>
```

## Postgresql 统计图表配置

下载配置文件并导入, 然后根据自己的Collectd配置进行调整

[https://raw.githubusercontent.com/developerworks/blog/master/files/grafana_postgres_dashbord.json](https://raw.githubusercontent.com/developerworks/blog/master/files/grafana_postgres_dashbord.json)

最后的效果如下图

![图片描述][1]

  [1]: https://segmentfault.com/img/bVDg7Q
