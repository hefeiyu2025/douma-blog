---
tags:
  - Mysql
  - 教程
  - 技巧
title: Mysql
createTime: 2024/09/24 16:04:38
permalink: /article/qw101ltl/
---

## Mysql使用技巧

### 设置登录权限

1. 登录mysql，输入密码，或无密码登录

```
mysql -uroot -p
```

2. 切换数据库，查询user 表的host,user,查看账户可登录IP列表，若host 为% 则允许所有IP登录

```
use mysql;
select host,user from user;
```

3. 两种方式授权，单独授权某IP或授权所有IP

- 授权root用户可以从10.10.1.35登录MySQL数据库，如下所示：

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.10.1.35' IDENTIFIED BY '密码' WITH GRANT OPTION;
```

- 授权root用户可以从任意电脑登录MySQL数据库。如下所示：

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '密码' WITH GRANT OPTION;
```

4. 刷新授权，保存授权列表

```
flush privileges;
```