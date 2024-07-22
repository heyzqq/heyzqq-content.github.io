---
title: "MySQL 去重之 distinct"
date: 2020-08-22T21:30:03+08:00
tags: ["tech", "MySQL"]
author: "Spring"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "在使用 MySQL 时，有时需要查询出某个字段不重复的记录，这时可以使用 MySQL 提供的 distinct 这个关键字来过滤重复的记录。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

在使用 MySQL 时，有时需要查询出某个字段不重复的记录，这时可以使用 MySQL 提供的 `distinct` 这个关键字来过滤重复的记录[[1]](https://www.cnblogs.com/shiluoliming/p/6604407.html "失落的黎明. mysql 中去重 distinct 用法")。

语法格式：

```sql
select 
    distinct expression[,expression...] 
from 
    tables 
[where conditions];
```

## 01 distinct 的用法

例如，我们有表如下：

```SQL
mysql> select * from user;
+----+----------+----------+
| id | username | password |
+----+----------+----------+
|  1 | taylor   | pass123  |
|  2 | spring   | pass456  |
|  3 | Yahto    | taf23    |
|  4 | Lillie   | flowwer  |
|  5 | Lia      | wherend  |
|  6 | Lia2     | wherend  |
|  7 | Lia3     | wherend  |
|  8 | Lia4     | wherend  |
+----+----------+----------+
```

### 1.1 简单的用法

用 `distinct` 返回不重复的密码：

```SQL
mysql> select distinct password from user;
+----------+
| password |
+----------+
| pass123  |
| pass456  |
| taf23    |
| flowwer  |
| wherend  | -- 去除了其他 3 个重复字段
+----------+
```

但是，这样只能查出不同的密码，不能查出密码对应的用户 id 或用户名。

实际开发中，我们往往用 `distinct` 来返回不重复字段的条数（`count(distinct id)`），就是因为 `distinct` 只能返回它的目标字段，而无法返回其他字段。

### 1.2 distinct 的注意事项

从最前面 `distinct` 的语法格式可以看出[[2]](https://blog.csdn.net/zhangzehai2234/article/details/88361586 "有梦想的攻城狮. mysql 中的 distinct 的用法")：

- `distinct` 要放在所有字段的*前面*
- 如果去重的字段大于一个，则会进行组合去重，只有多个字段组合起来相同时才会被去重

```SQL
mysql> select distinct username, password from user;
+----------+----------+
| username | password |
+----------+----------+
| taylor   | pass123  |
| spring   | pass456  |
| Yahto    | taf23    |
| Lillie   | flowwer  |
| Lia      | wherend  | -- 由于 username 不同，故 password 相同也不能去重
| Lia2     | wherend  |
| Lia3     | wherend  |
| Lia4     | wherend  |
+----------+----------+
```

## 02 可能遇到的其他用法

### 2.1 错误使用 distinct(c)

不知道你们有没有遇到过这样的例子：

> 要求查出所有用户，并根据手机号过滤重复记录。。。

可能一不小心就会遇到这样子的查询语句（这里用过滤相同密码来试验）：

```SQL
mysql> select distinct(password), username from user;
+----------+----------+
| password | username |
+----------+----------+
| pass123  | taylor   |
| pass456  | spring   |
| taf23    | Yahto    |
| flowwer  | Lillie   |
| wherend  | Lia      | -- 其实还是过滤 password+username
| wherend  | Lia2     |
| wherend  | Lia3     |
| wherend  | Lia4     |
+----------+----------+
```

`distinct(column_name)` 并没有按照函数操作那样，仅对括号内的列进行去重，而是依旧对 `distinct` 后面的所有列进行组合去重。（其实这里 `distinct` 也只能放在最前面，放到后面就会报错了）

### 2.2 计数 count(distinct c)

计数方式的两种情况。

第一种，计算指定字段的出现次数，可以直接用 `count`：

```SQL
mysql> select count(username) AS total 
    -> from user 
    -> where password = 'wherend';
+-------+
| total |
+-------+
|   4   |
+-------+
```

第二种，计算所有字段的出现次数，先根据 password 分组，然后计数时根据不同的 username 来计算（如果用户名一样，则认为是相同的）

```SQL
mysql> select
    ->     password, count(distinct username) as num
    -> from
    ->     user
    -> group by
    ->     password;
+----------+-----+
| password | num |
+----------+-----+
| flowwer  |   1 |
| pass123  |   1 |
| pass456  |   1 |
| taf23    |   1 |
| wherend  |   4 |
+----------+-----+
```

## 总结

1. 因为 `distinct` 只能返回他的目标字段，而无法返回其他字段，故一般用来计算不重复字段的条数（须先分组）
```SQL
mysql> select 
    ->   password, count(distinct id) AS num 
    -> from
    ->   user 
    -> group by password;
+----------+-----+
| password | num |
+----------+-----+
| flowwer  |   4 |
| pass123  |   1 |
+----------+-----+
```
2. 单纯去除重复，以便获取去重后的集合
```SQL
mysql> select distinct password from user;
+----------+
| password |
+----------+
| pass123  |
| pass456  |
+----------+
```

完。


## REFERENCES 

[1] 失落的黎明. mysql 中去重 distinct 用法:
https://www.cnblogs.com/shiluoliming/p/6604407.html

[2] 有梦想的攻城狮. mysql 中的 distinct 的用法:
https://blog.csdn.net/zhangzehai2234/article/details/88361586