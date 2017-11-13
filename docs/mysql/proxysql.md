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

添加调度器

```
```
