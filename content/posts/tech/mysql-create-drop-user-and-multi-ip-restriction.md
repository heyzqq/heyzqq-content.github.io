---
title: "MySQL 用户管理：创建、权限配置和删除"
date: 2023-06-16T20:30:31+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "MySQL"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "文章主要讲解了 MySQL 用户管理的相关知识，包括创建用户、查询用户、操作权限限制等内容，并提供了相应的 SQL 示例。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---

## 碎碎念

遇到过的很多小伙伴，无论在开发环境，还是在正式环境，都是“一键 ROOT”。包括但不限于 Linux 服务器、MySQL 数据库或者其他中间件，一律使用 ROOT 账户登录使用。

虽然这么做，可以减少权限等其他不重要的问题，省去很多麻烦，但是，我觉得这个习惯并不好。如果是正式环境，应该尽量避免这种情况。而且，真正需要自己去配置这些的时候，整个过程都会磕磕绊绊。

作为一个爱折腾的小码农，还是忍不住要弄它几下的。

## 01 创建用户

### 1.1 查询用户

MySQL 所有的用户信息，均保存在数据库名为 `mysql` 的 `user` 表中。

```sql
mysql> select user, host, plugin from mysql.user ;
+------------------+-----------+-----------------------+
| user             | host      | plugin                |
+------------------+-----------+-----------------------+
| root             | %         | mysql_native_password |
| mysql.infoschema | localhost | caching_sha2_password |
| mysql.session    | localhost | caching_sha2_password |
| mysql.sys        | localhost | caching_sha2_password |
| root             | localhost | mysql_native_password |
+------------------+-----------+-----------------------+

```

从上面的结果可以看到，除了几个内置的账户，默认只有 root 用户。

- `host`：表示**允许该用户连接到 MySQL 服务器的主机**，参考以下示例
  - `%`：通配符，允许所有主机连接
  - `192.168.%`：允许所有内网 `192.168` 网段的主机连接
- `plugin`：MySQL 的加密规则
  - `mysql_native_password`：MySQL 5 默认
  - `caching_sha2_password`：MySQL 8 默认，安全级别更高

### 1.2 创建用户

#### 1.2.1 MySQL 8 创建用户

创建用户命令格式：`CREATE USER 'USER_NAME'@'HOST' IDENTIFIED [WITH plugin] BY 'PASSWORD';`

```SQL
mysql> CREATE USER 'springx'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'springx';
Query OK, 0 rows affected (0.01 sec)
```

创建用户 springx，仅允许从本地登录连接，使用 `caching_sha2_password` 加密方式，密码为 `springx`。

```SQL
mysql> select user, host, plugin from mysql.user where user = 'springx';
+---------+-----------+-----------------------+
| user    | host      | plugin                |
+---------+-----------+-----------------------+
| springx | localhost | caching_sha2_password |
+---------+-----------+-----------------------+
```

测试，能正常登录：

```SQL
$ mysql -uspringx -pspringx
mysql: [Warning] Using a password on the command line interface can be insecure.
...
mysql>
```

#### 1.2.2 MySQL 5 创建用户

MySQL 5 的用户创建与 8 版本的类似，只不过版本 8 需要先创建用户，然后再进行授权，而版本 5 可以**直接创建用户并授权**。

```SQL
mysql> GRANT ALL PRIVILEGES ON `halo_db`.* TO 'springx'@'%' IDENTIFIED BY 'springx' WITH GRANT OPTION;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

创建成功，可以正常登录。*（warning 应该是数据库 halo_db 不存在，所以提醒的）*

当然，版本 5 也只**可以先创建用户，只不过不支持指定加密方式**，默认 `mysql_native_password`：

```SQL
mysql> CREATE USER 'springx'@'localhost' IDENTIFIED BY 'springx2';
Query OK, 0 rows affected (0.00 sec)

mysql> select user, host, plugin from mysql.user;
+---------------+-----------+-----------------------+
| user          | host      | plugin                |
+---------------+-----------+-----------------------+
| springx       | %         | mysql_native_password | -- 第一次创建（密码 springx）
| springx       | localhost | mysql_native_password | -- 本次创建  （密码 springx2）
+---------------+-----------+-----------------------+
```

**注意**，当存在 host2 包含在 host1 内时，优先使用 host2 规则（即“最小范围原则”——我瞎取的）。

> ∃ HOST2 ⊆ HOST1 时，激活 HOST2 规则，否则使用 HOST1 规则

所以，当从服务器登录时（localhost），只能使用 `springx2` 密码登录。

## 02 权限配置

### 2.1 登录 IP 限制

添加多个 IP 限制，其实就是「**添加多个账户**」，只不过他们**账户名相同**而已，**密码可以不同**。如果要把两个账户当成一个多 host 账户，密码设置一样即可。

在上节 1.2 中，创建了用户 `springx@localhost`，现在添加一个 IP 测试一下：

```SQL
mysql> CREATE USER 'springx'@'112.86.28.%' IDENTIFIED WITH caching_sha2_password BY 'springx2';
Query OK, 0 rows affected (0.01 sec)

mysql> select user, host, plugin from mysql.user ;
+------------------+--------------+-----------------------+
| user             | host         | plugin                |
+------------------+--------------+-----------------------+
| springx          | localhost    | caching_sha2_password | -- 密码 springx
| springx          | 112.86.28.%  | caching_sha2_password | -- 密码 springx2
+------------------+--------------+-----------------------+
```

- 注意，虽然用户同名，但是新的用户使用的密码是 `springx2`

测试一下服务器登录：

```SQL
$ mysql -uspringx -pspringx2
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 1045 (28000): Access denied for user 'springx'@'localhost' (using password: YES)
```

无法从服务器登录，这是预料之中的。原因是密码错误，因为是从服务器（localhost）登录，正确密码应该是第一次添加的用户的密码（即 `springx`）。

而从我自己的电脑，使用 `springx + springx2`，可以正常登录（用 Navicat 就不截图了）。

**注意，由于是创建了新的用户，库表权限需要重新授权。**

另外，也可用通过防火墙进行限制，这不是本文要说的重点，有兴趣的小伙伴可以自行折腾。

### 2.2 操作权限限制

*数据库的授权，请查看 [《MySQL 用户数据权限配置》](/posts/tech/mysql-data-permission)。*

## 03 删除用户

### 3.1 禁用用户

如果只是临时禁用，可以锁定账户，不用完全删除。

如果要锁定账户，只要修改 `mysql.user` 的 `account_locked` 字段，设置为 `Y` 即可：

```SQL
mysql> select user, host, plugin, account_locked from mysql.user where user='springx';
+---------------+-----------+-----------------------+----------------+
| user          | host      | plugin                | account_locked |
+---------------+-----------+-----------------------+----------------+
| springx       | %         | mysql_native_password | N              |
| springx       | localhost | mysql_native_password | N              |
+---------------+-----------+-----------------------+----------------+

mysql> update mysql.user set account_locked = 'Y' where user='springx';
Query OK, 2 rows affected (0.00 sec)
Rows matched: 2  Changed: 2  Warnings: 0
```

修改权限标志后，记得刷新一下：

```SQL
mysql> flush privileges;
```

对于 `mysql`.`user` 表的修改，和修改普通表一样，直接 `UPDATE` 加条件更新即可。现在，用户已被禁用：

```SQL
mysql> select user, host, plugin, account_locked from mysql.user where user='springx';
+---------+-----------+-----------------------+----------------+
| user    | host      | plugin                | account_locked |
+---------+-----------+-----------------------+----------------+
| springx | %         | mysql_native_password | Y              |
| springx | localhost | mysql_native_password | Y              |
+---------+-----------+-----------------------+----------------+
```

上面的 SQL，一次性把用户名为 springx 的所有用户都锁定了，此时登录，会提示“Account is locked”：

```SQL
$ mysql -uspringx -pspringx2
mysql: [Warning] Using a password on the command line interface can be insecure.
ERROR 3118 (HY000): Access denied for user 'springx'@'localhost'. Account is locked.
```

锁定成功，现在用户 springx 无法登录了


### 3.2 删除用户

对于以后不再使用的用户，可以使用 `drop` 彻底删除：

```SQL
mysql> DROP USER 'springx'@'localhost';
Query OK, 0 rows affected (0.00 sec)

mysql> select user, host, plugin, account_locked from mysql.user where user='springx';
+---------+------+-----------------------+----------------+
| user    | host | plugin                | account_locked |
+---------+------+-----------------------+----------------+
| springx | %    | mysql_native_password | Y              |
+---------+------+-----------------------+----------------+
```

DROP 的命令格式：`DROP USER 'USER_NAME'[@'HOST']`。

如果不指定 host，那么将删除所有同名账户：

```SQL
mysql> DROP USER 'springx';
Query OK, 0 rows affected (0.00 sec)

mysql> select user, host, plugin, account_locked from mysql.user where user='springx';
Empty set (0.00 sec)
```


## 04 总结

1. MySQL 8 需要先创建用户，再授权；MySQL 5 可以直接创建用户并授权；
2. 多 IP 登录限制，原理就是创建多个同名账户，账户密码可以不一样，但是库表权限需要重新授权（因为实际是多个不同的用户）；
3. 对于没用的账户，可以修改 `mysql.user` 表的 `account_locked` 为 `Y` 禁用账户，也可以使用 `DROP` 彻底删除用户
