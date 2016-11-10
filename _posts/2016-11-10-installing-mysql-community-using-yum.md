---
layout: post
title:  "使用 yum 安装最新版 MySQL community"
date:   2016-11-10 11:00:00
categories: cookbook
---

原文：http://dev.mysql.com/doc/mysql-yum-repo-quick-guide/en/

1. 从 http://dev.mysql.com/downloads/repo/yum/ 下载 MySQL yum repository 的 rpm 包。
可以使用 uname -a 查看 linux 平台类型。

2. 安装 rpm 包  
    `shell> sudo rpm -Uvh platform-and-version-specific-package-name.rpm`

3. 安装 MySQL  
    `shell> sudo yum install mysql-community-server`

4. 启动 MySQL  
    `shell> sudo service mysqld start`

5. MySQL 5.7 第一次启动时会安装 validate_password 插件并创建账户 'root'@'localhost'。
   密码存储在错误日志中，可以使用下面的命令查看生成的密码。  
    `shell> sudo grep 'temporary password' /var/log/mysqld.log`  
    使用这个密码登录并更改密码。
    validate_password 插件默认要求密码长度至少为 8 位并且同时包含大写字母、小写字母、数字和特殊字符。  
    `mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';`
