---
title: "MySQL 用户数据权限配置"
date: 2023-06-17T21:58:14+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "MySQL"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "MySQL 中提供了 GRANT 和 REVOKE 命令用于授权和回收权限。文章详细介绍了 MySQL 中的全局权限、库级权限、表级权限、列级权限，并且通过示例展示了如何使用 REVOKE 命令回收权限。"
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

对于普通的系统开发，很少会对数据进行细粒度的权限控制。

但是，在某些需要提供给第三方数据的场景中，要么会直接提供基础数据，要么会提供仅有部分权限的账户。如果是直接提供账户，就需要对账户进行比较精细的权限限制。

在 `mysql` 库中，有很多的系统配置表。其中，`user` 表中存储的是全局的权限，更细粒度的权限分别存储在其他表中。

## 01 权限

虽然在上一篇[《MySQL 用户管理：创建、权限配置和删除》](/posts/tech/mysql-create-drop-user-and-multi-ip-restriction)中提到，修改权限可以直接修改表（`update` 操作），但是，有些权限涉及其他表，直接修改表不方便，也不建议。

当然，MySQL 已经为权限提供了 `GRANT` 和 `REVOKE` 命令，用于授权和回收权限。

常见权限名称如下，使用别名授权，就不需要直接 `update` 数据库了：

权限别名    |   权限列名  |      权限范围     |
------------|-------------|-------------------|
 SELECT     | select_priv | Tables or columns |
 INSERT     | insert_priv | Tables or columns |
 UPDATE     | update_priv | Tables or columns |
 DELETE     | delete_priv | Tables            |
 CREATE     | create_priv | Databases, tables, or indexes |
 DROP       | drop_priv   | Databases, tables, or views   |
 INDEX      | index_priv  | Tables            |
 ...        | ...         | ...               |
 
*具体参考官方文档：[6.2.2 Privileges Provided by MySQL](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html)*


### 1.1 全局权限（`user`）

全局权限存储在 `mysql.user` 表中，作用于 MySQL 的所有数据库，除了 root，新建用户默认都没有权限（都是 `N`）

```SQL
mysql> select user, host, select_priv, insert_priv, update_priv, delete_priv, create_priv, drop_priv, grant_priv, index_priv from mysql.user where account_locked = 'N';
+------+-----------+-------------+-------------+-------------+-------------+-------------+-----------+------------+------------+
| user | host      | select_priv | insert_priv | update_priv | delete_priv | create_priv | drop_priv | grant_priv | index_priv |
+------+-----------+-------------+-------------+-------------+-------------+-------------+-----------+------------+------------+
| root | localhost | Y           | Y           | Y           | Y           | Y           | Y         | Y          | Y          |
| root | %         | Y           | Y           | Y           | Y           | Y           | Y         | Y          | Y          |
+------+-----------+-------------+-------------+-------------+-------------+-------------+-----------+------------+------------+
```

- 增删改查：`insert_priv`、`delete_priv`、`update_priv`、`select_priv`
- 创建：`create_priv`
- 删除：`drop_priv`

**测试：库的创建和查看**

新创建的用户 springx，没有分配权限，不仅看不到其他库，也没有创建数据库的权限：

```SQL
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
+--------------------+

mysql> create database test2;
ERROR 1044 (42000): Access denied for user 'springx'@'%' to database 'test2'
```

错误反馈的原因是 “拒绝访问”，也就是说用户没有 test2 的访问权限，即使 test2 不存在。

首先，给用户分配库的创建权限：

- `*`：通配符
- 格式：`库名.表名`，`*.*` 表示所有库表，`test.*` 表示 test 库中的所有表

```SQL
-- 授权所有
mysql> GRANT CREATE ON *.* TO springx@'%';

-- 查看权限
mysql> select user, host, select_priv, create_priv from mysql.user where user='springx';
+---------+------+-------------+-------------+
| user    | host | select_priv | create_priv |
+---------+------+-------------+-------------+
| springx | %    | N           | Y           |
+---------+------+-------------+-------------+
```

- 1）这里如果指定库 `test.*` 的权限，而不是所有库 `*.*`，那么就是只有固定库权限（记录在 `db` 表中）
- 2）有了创建权限，就能看到库，那么 `select_priv` 是什么作用呢？

然后，修改了权限，一般都要 `flush privileges` 刷新一下，但这里还**需要重新登录**才能创建库。（创建库完，就能看到了）

### 1.2 库级权限（`db`）

上面分配了 `*.*` 是全局权限，一般是超管账户才给分配，普通用户只需分配必要库（甚至只分配指定表）的权限。

库级权限存储在 `mysql.db` 表中，默认只有两个内置账户：

```SQL
mysql> select user, host, select_priv, create_priv from mysql.db;
+---------------+-----------+-------------+-------------+
| user          | host      | select_priv | create_priv |
+---------------+-----------+-------------+-------------+
| mysql.session | localhost | Y           | N           |
| mysql.sys     | localhost | N           | N           |
+---------------+-----------+-------------+-------------+
```

如果指定了库，该用户只能操作指定库下的数据，配置也会记录在 `mysql.db` 中：

```SQL
-- user: root 授权
mysql> GRANT CREATE ON test3.* TO springx@'%';
Query OK, 0 rows affected (0.00 sec)
mysql> select user, host, select_priv, create_priv from mysql.db;
+---------------+-----------+-------------+-------------+
| user          | host      | select_priv | create_priv |
+---------------+-----------+-------------+-------------+
| springx       | %         | N           | Y           |
+---------------+-----------+-------------+-------------+

-- user: springx
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| test3              |
+--------------------+
```

- 这里授权后，并不需要刷新权限，也不需要重启
- CREATE 仅可以对 test3 数据库进行表的创建，没有其他权限  
  - `select` 失败：`ERROR 1142 (42000): SELECT command denied to user 'springx'@'localhost' for table 'user'` 

### 1.3 表级权限（`tables_priv`）

表级顾名思义，就是针对单表进行权限控制，所以分配权限要精确到表，如 `test.employee`。

表级权限配置存储在 `mysql.tables_priv` 表中，默认也只有内置账户。

当一个用户被赋予了一个库中的其中一个表，那么，该用户就能在 `show databases` 中查看该库（无需刷新）

```SQL
-- 赋予 employee 表的创建权限
mysql> GRANT CREATE ON test.employee TO springx@'%';
Query OK, 0 rows affected (0.00 sec)

-- 查看表级权限
mysql> select user, host, table_name, grantor, table_priv from mysql.tables_priv where user='springx';
+---------+------+------------+----------------+------------+
| user    | host | table_name | grantor        | table_priv |
+---------+------+------------+----------------+------------+
| springx | %    | employee   | root@localhost | Create     |
+---------+------+------------+----------------+------------+
```

注意，即使表 employee 不存在，也能进行授权。授权后，就可以让该用户自己去创建该表：

```SQL
-- user: springx, db: test
mysql> show tables;
Empty set (0.00 sec)

mysql> CREATE TABLE employee (
    ->   id INT PRIMARY KEY AUTO_INCREMENT,
    ->   name VARCHAR(50) NOT NULL
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| employee       |
+----------------+
```

权限标志，存放在 `mysql.tables_priv` 中的 `table_priv` 字段中，多种权限用逗号 `,` 分割。

前面只分配了 `CREATE` 权限，只能创建表，不能做其他操作。这里添加一个 `SELECT` 测试一下：

```SQL
mysql> GRANT SELECT ON test.employee TO springx@'%';
Query OK, 0 rows affected (0.00 sec)

mysql> select user, host, table_name, grantor, table_priv from mysql.tables_priv where user='springx';
+---------+------+------------+----------------+---------------+
| user    | host | table_name | grantor        | table_priv    |
+---------+------+------------+----------------+---------------+
| springx | %    | employee   | root@localhost | Select,Create |
+---------+------+------------+----------------+---------------+
```

现在，`table_priv` 字段有了创建和查询的权限，就可以使用 `select` 查看 employee 表内容了。

### 1.4 列级权限（`columns_priv`）

列级权限，也就是可以对单独列做细粒度控制，配置存储在 `mysql.columns_priv` 表中。

回收上一节 `test.employee` 的所有权限，重新创建表并赋予创建权限：

```SQL
mysql> GRANT CREATE ON test.employee TO springx@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> select user, host, db, table_name, grantor, table_priv, column_priv from mysql.tables_priv where db='test' and user='springx';
+---------+------+------+------------+----------------+------------+-------------+
| user    | host | db   | table_name | grantor        | table_priv | column_priv |
+---------+------+------+------------+----------------+------------+-------------+
| springx | %    | test | employee   | root@localhost | Create     |             |
+---------+------+------+------------+----------------+------------+-------------+
```

此时，springx 用户除了创建 employee 表，没有其他权限。

**测试 1：赋予 springx 对 employee 表的插入权限**

```SQL
-- 授权
mysql> GRANT INSERT (id, `name`, birth) on test.employee to springx@'%';
Query OK, 0 rows affected (0.00 sec)

-- 表级配置
-- 可以看到，tables_priv 中的 column_priv 多了一个 insert 权限
mysql> select user, host, db, table_name, grantor, table_priv, column_priv from mysql.tables_priv where db='test' and user='springx';
+---------+------+------+------------+----------------+------------+-------------+
| user    | host | db   | table_name | grantor        | table_priv | column_priv |
+---------+------+------+------------+----------------+------------+-------------+
| springx | %    | test | employee   | root@localhost | Create     | Insert      |
+---------+------+------+------------+----------------+------------+-------------+

-- 列级配置
mysql> select user, host, db, table_name, column_name, column_priv from mysql.columns_priv where db = 'test' and user = 'springx';
+---------+------+------+------------+-------------+-------------+
| user    | host | db   | table_name | column_name | column_priv |
+---------+------+------+------------+-------------+-------------+
| springx | %    | test | employee   | id          | Insert      |
| springx | %    | test | employee   | name        | Insert      |
| springx | %    | test | employee   | birth       | Insert      |
+---------+------+------+------------+-------------+-------------+
```

配置插入权限后，可以正常插入，但还不能查看：

```SQL
mysql> insert into employee values(1, 'Spring', '1999-11-11');
Query OK, 1 row affected (0.00 sec)
```

**测试 2：赋予 springx 对 employee 表中 id 和 name 的查看权限**

```SQL
-- 授权，仅授权两个列
mysql> GRANT SELECT (id, `name`) on test.employee to springx@'%';
Query OK, 0 rows affected (0.01 sec)

-- 表级权限
-- 可以看到，tables_priv 中的 column_priv 多了一个 Select 权限
mysql> select user, host, db, table_name, grantor, table_priv, column_priv from mysql.tables_priv where db='test' and user='springx';
+---------+------+------+------------+----------------+------------+---------------+
| user    | host | db   | table_name | grantor        | table_priv | column_priv   |
+---------+------+------+------------+----------------+------------+---------------+
| springx | %    | test | employee   | root@localhost | Create     | Select,Insert |
+---------+------+------+------------+----------------+------------+---------------+

-- 列级权限
mysql> select user, host, db, table_name, column_name, column_priv from mysql.columns_priv where db = 'test' and user = 'springx';
+---------+------+------+------------+-------------+---------------+
| user    | host | db   | table_name | column_name | column_priv   |
+---------+------+------+------------+-------------+---------------+
| springx | %    | test | employee   | id          | Select,Insert |
| springx | %    | test | employee   | name        | Select,Insert |
| springx | %    | test | employee   | birth       | Insert        |
+---------+------+------+------------+-------------+---------------+
```

对两个列授权完成后，id 和 name 都有查看权限，测试一下：

```SQL
-- 失败 1：* 失效，没有所有列的查看权限
mysql> select * from employee;
ERROR 1142 (42000): SELECT command denied to user 'springx'@'localhost' for table 'employee'

-- 失败 2：对 birth 没有查看权限
mysql> select id, name, birth from employee;
ERROR 1143 (42000): SELECT command denied to user 'springx'@'localhost' for column 'birth' in table 'employee'

-- 正常
mysql> select id, name from employee;
+----+--------+
| id | name   |
+----+--------+
|  1 | Spring |
+----+--------+
```

从测试结果可以看出，未对所有列授权时，无法使用通配符（`*`），也无法使用默认插入（`INSERT INTO TB VALUES(...)`）等操作，仅能对授权列做指定操作。

### 1.5 权限回收（`REVOKE`）

> 注意，库表不存在，可以提前授权，同样的，删除库表，并不会回收权限，需要手动回收。所以，不再授权的权限要使用 `REVOKE` 回收，这也是为什么不推荐直接修改表进行授权的原因（权限越细越麻烦）。

与授权命令类似，回收命令将 `to` 换成 `from`，即“授权给xx”改成“从xx回收权限”。

e.g.

```SQL
-- 授权所有
GRANT CREATE ON *.* TO springx@'%';
-- 回收所有
REVOKE CREATE ON *.* FROM 'springx'@'%';

-- 授权表(2 个)
GRANT SELECT, INSERT ON test.user to springx@'%';
-- 回收表(1 个)
REVOKE SELECT on test3.user from springx@'%';

-- 授权列
GRANT SELECT (birth) on test.employee TO springx@'%';
-- 回收列
REVOKE SELECT (birth) on test.employee FROM springx@'%';

-- ……
```

## 02 总结

1. MySQL 提供了 `GRANT` 和 `REVOKE` 命令，用于权限的授予和回收
2. 数据权限从全局、库级、表级到列级，可以进行细粒度控制
3. 授权与回收，跟库表是否存在无关，可以预先授权，也可以删除后再回收权限
4. 如果对列进行部分授权，`SELECT *` 等默认全部列的操作将无法使用
5. 授权与回收命令比较简单，分别是 `GRANT PRIVILEGE.. ON DB.TB TO USER@HOST` 和 `REVOKE PRIVILEGE.. ON DB.TB FROM USER@HOST`

## References

[1] 爱折腾的邦邦. [玩转 MySQL 之三] MySQL 用户及权限. https://zhuanlan.zhihu.com/p/55798418, 2019-01-26.  