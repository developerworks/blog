
> 数据库高可用之前说了, 现在我们来摆一下后面的事情. 那就是数据库的监视和管理. 目标是保持数据库良好稳定的运行, 为业务提供支撑. 一旦出现问题可以及时发现, 及时解决. 减少损失.

## 安装

安装包括服务器端和客户端的安装步骤

### 服务器端

官方提供了3中部署方式, 这里我选择Docker, 简单, 方便!

其他安装方式在[这里](https://www.percona.com/doc/percona-monitoring-and-management/deploy/server/docker.html#run-server-docker)

首先, 安装Docker

```
curl -sSL https://get.daocloud.io/docker | sh
```

其次, 拉取服务器镜像, 创建PMM数据容器, 创建PMM服务器容器

```
# 拉取服务器镜像

docker pull percona/pmm-server:latest

# 创建PMM数据容器

docker create \
   -v /opt/prometheus/data \
   -v /opt/consul-data \
   -v /var/lib/mysql \
   -v /var/lib/grafana \
   --name pmm-data \
   percona/pmm-server:latest /bin/true

# 创建PMM服务器容器, 同时设置登录用户名(SERVER_USER)和密码(SERVER_PASSWORD), 根据需要进行修改. 默认使用80端口, 如果需要可以更改.

docker run -d -p 80:80 \
  --volumes-from pmm-data \
  --name pmm-server \
  -e SERVER_USER=test \
  -e SERVER_PASSWORD=test \
  -e DISABLE_TELEMETRY=true \
  -e METRICS_MEMORY=262144 \
  --restart always \
  percona/pmm-server:latest
```

如果有要启用 Orchestrator, 如下:

```
docker run -d -p 80:80 \
  --volumes-from pmm-data \
  --name pmm-server \
  -e SERVER_USER=test \
  -e SERVER_PASSWORD=test \
  -e DISABLE_TELEMETRY=true \
  -e METRICS_MEMORY=262144 \
  -e ORCHESTRATOR_ENABLED=true \
  --restart always \
  percona/pmm-server:latest
```

> Orchestrator 是一个复制拓扑的可视化和管理工具.

![Orchestrator 是一个复制拓扑的可视化和管理工具.](https://www.percona.com/blog/wp-content/uploads/2016/02/dgq4Yz6IRS.gif)

### 客户端

仓库准备, 需要执行以下命令

```
# 下载仓库文件
wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
# 安装
dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
# 更新
apt-get update
```

这里我使用Ubuntu 16.04作为例子, 其他系统操作系统, 参考官方文档

```
# 安装客户端
apt-get install pmm-client
```

## 开始收集数据

在每个需要性能统计的服务器上安装pmm-client, 并且使用下面的命令连接到服务器并向服务器上报数据:

```
pmm-admin config --server 172.18.149.207 --server-user test --server-password test
```

获取MySQL的性能数据, 需要让 pmm-admin 过一次MySQL的登陆验证, 这里我直接使用了MySQL的Unix Socket:

```
pmm-admin add mysql --user root --password root --socket /var/run/mysqld/mysqld.sock
```

为了避免命令行密码留在命令历史中, 可以把上面的命令放到SHELL脚本里面.

```
#!/bin/bash
pmm-admin add mysql --user root --password root --socket /var/run/mysqld/mysqld.sock
```


## 参考资料

- [PMM 对MYSQL 的监控配制](https://www.cnblogs.com/zengkefu/p/7232938.html)
- [MySQL 性能监控软件 慢日志分析利器](http://blog.csdn.net/john1337/article/details/70855293)
- [Percona Monitoring and Management Documentation](https://www.percona.com/doc/percona-monitoring-and-management/index.html)
