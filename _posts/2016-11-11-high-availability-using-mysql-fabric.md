---
layout: post
title:  "使用 MySQL Fabric 实现高可用"
date:   2016-11-11 10:00:00
categories: cookbook
---

官方文档：<http://dev.mysql.com/doc/mysql-utilities/1.5/en/fabric.html>

MySQL Fabric 高可用方案需要最少三个 MySQL 实例，其中一个实例负责存储集群配置和状态信息，称为 backing store ，剩下的全部服务器用于组成高可用集群，集群中的 MySQL 应尽量使用相同的版本。高可用集群使用 replication 方式实现，其中一个实例作为 master，其他实例作为 slave，当 MySQL Fabric 检测到 master 实例发生故障之后，可以自动从 slave 实例中选择一个提升为新的 master。客户端使用 connector 通过 MySQL Fabric 访问数据库时，connector 能够自动从 MySQL Fabric 获取集群状态信息，将写操作发送给 master server，将读操作在整个集群中进行负载均衡。

# 一、 下载和安装 MySQL Fabric

1. 从 <http://dev.mysql.com/downloads/utilities/> 下载 MySQL Utilities 1.5 ，MySQL Fabric 是 MySQL Utilities 1.5 的一部分。

2. 从 <http://dev.mysql.com/downloads/connector/j/> 下载 java connector，5.1.30 和之后的版本支持 MySQL Fabric。

# 二、创建 MySQL 用户

需要为 MySQL Fabric 在每个 MySQL 实例上创建用户。在集群实例上创建用户过程中需要关闭 bin log，否则 MySQL Fabric 创建主从关系的时候会在 slave 上重新执行 master 中创建同一用户的操作而导致出错。以下示例中假设 MySQL Fabric 和 backing store 实例安装在同一台服务器中：

1. 在 backing store 实例上创建 store user，将下面代码中的 `fabric_store` 和 `secret_store` 替换为合适的用户名和密码。

        CREATE USER 'fabric_store'@'localhost'
            IDENTIFIED BY 'secret_store';

        GRANT ALTER, CREATE, CREATE VIEW, DELETE, DROP, EVENT,
            INDEX, INSERT, REFERENCES, SELECT, UPDATE ON mysql_fabric.*
            TO 'fabric_store'@'localhost';

2. 在每个集群实例上创建 server user，将下面代码中的 `fabric_server` 和 `secret_server` 替换为合适的用户名和密码。

        CREATE USER 'fabric_server'@'%'
            IDENTIFIED BY 'secret_server';

        GRANT DELETE, PROCESS, RELOAD, REPLICATION CLIENT,
            REPLICATION SLAVE, SELECT, SUPER, TRIGGER ON *.*
            TO 'fabric_server'@'%';

        GRANT ALTER, CREATE, DELETE, DROP, INSERT, SELECT, UPDATE
            ON mysql_fabric.* TO 'fabric_server'@'%';

3. 在每个集群实例上创建 backup user，将下面代码中的 `fabric_backup` 和 `secret_backup` 替换为合适的用户名和密码。

        CREATE USER 'fabric_backup'@'%'
            IDENTIFIED BY 'secret_backup';

        GRANT EVENT, EXECUTE, REFERENCES, SELECT, SHOW VIEW, TRIGGER ON *.*
            TO 'fabric_backup'@'%';

4. 在每个集群实例上创建 restore user，将下面代码中的 `fabric_restore` 和 `secret_restore` 替换为合适的用户名和密码。

        CREATE USER 'fabric_restore'@'%'
            IDENTIFIED BY 'secret_restore';
        GRANT ALTER, ALTER ROUTINE, CREATE, CREATE ROUTINE, CREATE TABLESPACE, CREATE VIEW,
            DROP, EVENT, INSERT, LOCK TABLES, REFERENCES, SELECT, SUPER,
            TRIGGER ON *.* TO 'fabric_restore'@'%';

# 三、 配置 MySQL

在每个集群实例中启用 gtid-mode, bin-log 和 log-slave-updates ，并确保所有集群实例的 server-id 都不重复。

    [mysqld]
    lower_case_table_names = 1
    character-set-server = utf8
    collation-server = utf8_general_ci
    skip-character-set-client-handshake

    server-id = 1
    enforce-gtid-consistency = true
    gtid-mode = on
    log-bin = mysql-bin
    binlog-format = MIXED
    log-slave-updates = true

# 四、配置和初始化 MySQL Fabric

1. 修改 `/etc/mysql/fabric.cfg` 中的 `[storage]` 和 `[servers]` 段，配置 store user, server user, backup user 和 restore user 的用户和密码。

        [storage]
        address = localhost:3306
        user = fabric_store
        password = secret_store
        database = mysql_fabric
        auth_plugin = mysql_native_password
        connection_timeout = 6
        connection_attempts = 6
        connection_delay = 1

        [servers]
        user = fabric_server
        password = secret_server
        backup_user = fabric_backup
        backup_password = secret_backup
        restore_user = fabric_restore
        restore_password = secret_restore
        unreachable_timeout = 5

2. 执行 `mysqlfabric manage setup` 初始化 store server，执行过程中需要设置 admin/xmlrpc 的密码，这个密码是使用用户名 `admin` 访问 MySQL Fabric 时使用的密码。可以将这个密码配置到 fabric.cfg 中 `[protocol.xmlrpc]` 段的 `password` 中，否则后面每次执行 MySQL Fabric 命令都会提示输入这个密码。

# 五、启动和停止 MySQL Fabric

1. 执行 `mysqlfabric manage start` 启动 MySQL Fabric。

2. 执行 `mysqlfabric manage stop` 停止 MySQL Fabric。

# 六、创建服务器组

1. 执行 `mysqlfabric group create my_group` 创建服务器组，其中 `my_group` 为组名。

2. 执行 `mysqlfabric group add my_group master_server:port` 添加主服务器，其中 `master_server:port` 为主服务器的 IP 和端口。

3. 执行 `mysqlfabric group promote my_group` 将主服务器提升为服务器组的 master。

4. 执行 `mysqlfabric group add my_group slave_server:port` 添加从服务器，其中 `slave_server:port` 为从服务器的 IP 和端口，添加之后从服务器会自动同步主服务器上已经写入 bin log 的数据。

5. 执行 `mysqlfabric group activate my_group` 激活故障自动切换。

# 七、错误恢复

服务器出现故障之后，MySQL Fabric 中会将其标记为 FAULTY 状态，在服务器恢复之后状态不会自动恢复，需要人工将出错的服务器从组中移除再重新加入。

    `shell>mysqlfabric group remove my_group server:port`

    `shell>mysqlfabric group add my_group server:port`

# 八、使用 Java connector 连接到 MySQL Fabric

以 Tomcat datasource 配置为例，从直接连接切换成通过 MySQL Fabric 连接 MySQL 只需修改 `driverClassName` 和 `url` ：

	driverClassName="com.mysql.fabric.jdbc.FabricMySQLDriver"
    url="jdbc:mysql:fabric://localhost:32274/database?fabricServerGroup=my_group&amp;fabricUsername=admin&amp;fabricPassword=secret_xmlrpc"

其中 `localhost:32274` 是 MySQL Fabric 的服务地址，`fabricUsername` 和 `fabricPassword` 是 MySQL Fabric 的用户名和密码，`database` 是默认数据库名称，`fabricServerGroup` 是服务器组名称。
