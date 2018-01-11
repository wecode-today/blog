title: 使用DockerCompose搭建MySQL高可用集群，并平滑迁移
date: 2017/06/26 19:40:00
categories:
  - mysql
tags:
  - ha
  - mysql
---
**!!!由于MySQL官方驱动还不支持Group Replication，因此当前只是服务端可用，客户端驱动还需要等待新版本。**

本文描述如何使用Docker+DockerCompose来部署一个MySQL的高可用集群。

此集群由3个MySQL实例组成，一个Master节点，2个Secondary节点。Master可读写，Secondary可读。集群之间会自动同步数据。Master失效剩余的Secondary会自动选举Master。

- 请准备三台Linux VM(Ubuntu14.04)，最好是异地的
- 每台VM上请部署docker

**由于公司内部网络隔离，很多地址需要从公司内镜像网站下载。请注意，某些配置步骤在可以直连外网的情况下并不需要。**

# 配置Docker环境（每台VM）
1. 在配置文件/etc/default/docker中DOCKER_OPTS选项添加`--insecure-registry hub.witcloud.huawei.com`，并重启docker服务
2. 执行`apt-get install python-pip`
3. 配置pip公司内网镜像
    ```
    在~/.pip/目录下增加pip.conf文件

    然后编辑其内容为：

    [global]
    trusted-host=rnd-mirrors.huawei.com
    index-url=http://rnd-mirrors.huawei.com/pypi/simple/
    ```
4. 安装docker-compose

    ```
    pip install docker-compose
    ```
部署完成后就可以使用docker-compose来进行镜像的下载和容器的启动了。

<!-- more -->

# 使用Docker部署MySQL（每台VM）
首先我们要编写DockerCompose文件，用于配置MySQL镜像的启动。

```
mkdir /home/docker/mysql -p
vim /home/docker/mysql/docker-compose.yml
```

**注意：因为数据库容量未来可能较大，因此最好将大容量磁盘挂载到`/data`目录。**

```yaml
version: '2'
services:
  mysql:
    network_mode: "host"
    environment:
      MYSQL_ROOT_PASSWORD: "blu3Spoon55"
    image: hub.witcloud.huawei.com/it-ark/mysql
    restart: unless-stopped
    volumes:
      - "/data/mysql/db:/var/lib/mysql"
      - "/data/mysql/conf:/etc/mysql/conf.d"
```

`MYSQL_ROOT_PASSWORD`为管理员密码，请自己设置一个强密码。

# 配置Group Replication节点（每台VM）

**需要注意的差异如下：**
1. `server_id`每台VM不同
    ```
    节点1: 1
    节点2: 2
    节点3: 3
    ```
2. `loose-group_replication_local_address`每台VM不同
    ```
    节点1: 10.183.22.167:24901
    节点2: 100.112.98.176:24901
    节点3: 10.167.117.41:24901
    ```
3. `bind-address`和`report_host`绑定为各VM的IP地址

我们使用配置文件来对集群进行配置：
```
mkdir /data/mysql/conf -p
vim /data/mysql/conf/ha.cnf
```

文件内容：
```
[mysqld]  
#Encoding
collation-server = utf8_unicode_ci
init-connect='SET NAMES utf8'
character-set-server = utf8

#Replication Framework  
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE
log_slave_updates=ON
log_bin=binlog
binlog_format=ROW

# Host specific replication configuration
server_id=1
bind-address="10.183.22.167"
report_host="10.183.22.167"
loose-group_replication_local_address ="10.183.22.167:24901"
  
#Group Replication  
transaction_write_set_extraction=XXHASH64  
loose-group_replication_group_name="9a261783-7436-4df7-be2c-97fc7b37eb85"         
loose-group_replication_start_on_boot=on  
loose-group_replication_group_seeds="10.183.22.167:24901,100.112.98.176:24901,10.167.117.41:24901"  
loose-group_replication_bootstrap_group=off  
loose-group_replication_ip_whitelist="10.183.22.167,100.112.98.176,10.167.117.41,127.0.0.1/8"
```

# 依次启动每个节点，进行数据库内部配置
1. 启动节点数据库
    ```
    docker-compose up -d
    ```
2. 使用数据库工具链接节点
3. **仅针对节点2,3，第一个节点不用**:清理Docker容器内部产生的binlog，从完全干净的状态进行集群同步
    ```sql
    RESET MASTER ;
    ```
4. 创建group replication所需要的用户
    ```sql
    SET SQL_LOG_BIN =0;
    CREATE USER rpl_user@'%' IDENTIFIED BY 'rpl_pass';
    GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
    FLUSH PRIVILEGES ;
    SET SQL_LOG_BIN =1;
    CHANGE MASTER TO MASTER_USER ='rpl_user', MASTER_PASSWORD ='rpl_pass' FOR CHANNEL 'group_replication_recovery';
    ```
5. 安装组复制插件
    ```sql
    INSTALL PLUGIN group_replication SONAME 'group_replication.so';
    ```    

    检查是否插件是否启用：
    ```
    mysql> SHOW PLUGINS;
    +----------------------------+----------+--------------------+----------------------+-------------+
    | Name                       | Status   | Type               | Library              | License     |
    +----------------------------+----------+--------------------+----------------------+-------------+
    | binlog                     | ACTIVE   | STORAGE ENGINE     | NULL                 | PROPRIETARY |
    (...)
    | group_replication          | ACTIVE   | GROUP REPLICATION  | group_replication.so | PROPRIETARY |
    +----------------------------+----------+--------------------+----------------------+-------------+
    ```
6. 启动group replication
    - **节点1：**
        ```sql
        SET GLOBAL group_replication_bootstrap_group=ON;
        START GROUP_REPLICATION;
        SET GLOBAL group_replication_bootstrap_group=OFF;
        ``` 
    - **节点2/3**
        ```sql
        START GROUP_REPLICATION;
        ``` 

7. **仅针对节点1:** 增加一些测试数据，用于测试节点2和节点3是否同步了数据
    ```sql
    CREATE DATABASE test;
    use test;
    CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL);
    INSERT INTO t1 VALUES (1, 'mysql集群测试');
    ```

# 客户端使用
**!!!由于MySQL官方驱动还不支持Group Replication，因此当前只是服务端可用，客户端驱动还需要等待新版本。**

~~官方[ConnectJ](https://dev.mysql.com/downloads/connector/j/)支持使用Replication，以支持MySQL Group Replication。~~

~~只需要在连接字符串中配置即可：~~

```
jdbc:mysql:loadbalance://100.112.98.176:3306,10.183.22.167:3306,10.167.117.41:3306
```

~~应用程序代码无需修改，驱动会自动判断可用的Master。~~

## Master/Secondary异步延时
由于Master/Secondary之间的数据同步存在一定延时，可能导致先写入后读取的时候，读取不到数据。

举例说明：

1. 线程A在Master 1上插入一条数据，id为6217614201566007296
2. 线程A把此ID(6217614201566007296)给线程B
3. 线程B在Secondary 1上通过此ID(6217614201566007296)读取数据
4. 但是此时数据还未从Master1同步到Secondary 1
5. 线程B读取数据为空

因此，在这种情况下，由于写入到读取之间可能存在延时，那么在写入之后的处理阶段（比如线程B），需要增加重试或者延时，来等待数据同步。
这也是分布式数据库**最终一致性带来的问题**。

# 参考资料：
> 1. [MySQL Group Replication官方文档](https://dev.mysql.com/doc/refman/5.7/en/group-replication.html)
> 2. [细细探究MySQL Group Replication — 配置维护故障处理全集](http://www.cnblogs.com/xinysu/p/6674832.html#autoid-4-2-0)
> 3. [使用ConnectJ配置负载均衡](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-usagenotes-j2ee-concepts-managing-load-balanced-connections.html)
> 4. [数据库分库分表（二）Twitter-Snowflake（64位分布式ID算法）分析与JAVA实现](http://www.jianshu.com/p/80e68ae9e3a4)
