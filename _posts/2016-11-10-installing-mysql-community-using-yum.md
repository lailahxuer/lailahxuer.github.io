---
layout: post
title:  "使用 yum 安装最新版 MySQL community"
date:   2016-11-10 11:00:00
categories: cookbook
---

官方文档：<http://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/>

1. 从 <http://dev.mysql.com/downloads/repo/yum/> 下载 MySQL yum repository 的 rpm 包。可以使用 uname -a 查看 linux 平台类型。

2. 安装 rpm 包  

    `shell> sudo rpm -Uvh platform-and-version-specific-package-name.rpm`

3. 默认情况下 yum 会安装 MySQL community 最新的 GA 版本，如果要安装其他版本需要手动修改 `/etc/yum.repos.d/mysql-community.repo`。例如当前最新的 GA 版本是 5.7，我们要安装 5.6 版本，就需要将 `[mysql57-community]` 段中的 `enabled` 值修改为 0，将 `[mysql56-community]` 段中的 `enabled` 值修改为 1。修改后的两个配置段内容如下：

        [mysql56-community]
        name=MySQL 5.6 Community Server
        baseurl=http://repo.mysql.com/yum/mysql-5.6-community/el/6/$basearch/
        enabled=1
        gpgcheck=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

        [mysql57-community]
        name=MySQL 5.7 Community Server
        baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/6/$basearch/
        enabled=0
        gpgcheck=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

4. 安装 MySQL  

    `shell> sudo yum install mysql-community-server`

5. 启动 MySQL  

    `shell> sudo service mysqld start`

6. 在 MySQL 5.6 中，第一次启动时创建的 root 账户密码为空，需要执行 `mysql_secure_installation` 设置 root 密码：  

    `shell> mysql_secure_installation`

7. 在 MySQL 5.7 中，第一次启动时会自动安装 validate_password 插件，root 密码存储在日志中，可以使用下面的命令查看生成的密码:  

    `shell> sudo grep 'temporary password' /var/log/mysqld.log`  

    使用这个密码登录并更改密码，validate_password 插件默认要求密码长度至少为 8 位并且同时包含大写字母、小写字母、数字和特殊字符。 

    `mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';`
