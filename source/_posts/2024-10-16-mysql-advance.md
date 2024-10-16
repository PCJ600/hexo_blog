---
layout: next
title: MySQL进阶
date: 2024-10-16 21:58:05
categories: MySQL
tags: MySQL
---

## 体系结构
连接层 -> 服务层 -> 存储引擎 —> 存储层

InnoDB
支持事务，行级锁，外键
xxx.ibd 路径: /var/lib/mysql/*.ibd xxx表示表名, InnoDB每张表对应一个表空间文件，存储表结构、数据、索引
ibd2sdi account.ibd
逻辑存储结构: TableSpace(表空间), Segment(段), Extent(区), Page(页), Row(行)

## 索引
https://www.cs.usfca.edu/~galles/visualization/BTree.html
https://blog.csdn.net/u011240877/article/details/80490663

### 为什么用B+tree
* 相对于二叉树，层级更少，效率更高
* B-tree, 非叶子节点也保存数据，导致树高度增加，效率降低
* 相对Hash索引，支持范围匹配和排序

### 语法
创建索引
```
create [unique|fulltext] index index_name on table_name (index_col_name,...);
```
查看索引
```
show index from table_name;
```
删除索引
```
drop index index_name on table_name;
```

例：
```
select * from tb_user;
|id | name | email | profession | age | gender | status | createtime |
create table tb_user(
	id bigint auto_increment primary key,
	name varchar(10),
	email varchar(36),
	profession varchar(36),
	age tinyint unsigned,
	gender char(1),
	status int,
	createtime datetime
)engine=InnoDB;
```

name设置索引
phone非空创建唯一索引
profession,age,status设置联合索引
email建立索引
```
create index idx_user_name on tb_user(name);
create unique index idx_user_phone on tb_user(phone);
create index idx_user_pro_age_sta on tb_user(profession,age,status);
create index idx_user_email on tb_user(email);
```

### 性能分析
SQL执行效率
show [session|global] status;查看数据库insert,update,delete,select频次
```
mysql> show global status like 'com_______';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Com_binlog    | 0     |
| Com_commit    | 1     |
| Com_delete    | 2     |
| Com_import    | 0     |
| Com_insert    | 30    |
| Com_repair    | 0     |
| Com_revoke    | 0     |
| Com_select    | 231   |
| Com_signal    | 0     |
| Com_update    | 9     |
| Com_xa_end    | 0     |
+---------------+-------+
```

慢查询日志
记录所有执行时间超过指定参数long_query_time(默认:10秒)的SQL语句日志
默认没开
```
mysql> show variables like 'slow_query%';
+---------------------+-----------------------------------+
| Variable_name       | Value                             |
+---------------------+-----------------------------------+
| slow_query_log      | OFF                               |
| slow_query_log_file | /var/lib/mysql/localhost-slow.log |
+---------------------+-----------------------------------+
```



### 索引

#### 为什么需要索引
关系数据库中可能存在上万甚至上亿条记录，想要获得非常快的查询速度，就需要使用索引。

#### 什么是索引
索引是关系数据库中对某一列或多个列的值进行预排序的数据结构。
通过索引，数据库系统不必扫描整个表，而是直接定位到符合条件的记录，这样就大大加快了查询速度。

#### 查看索引
```sql
show index from student;
```

#### 添加索引
```sql
create table student (id bigint primary key not null auto_increment, class_id int, name varchar(32), gender int, score int);
alter table student add index idx_score (score);
```
* 索引的效率取决于索引列的值是否散列，即该列的值如果越互不相同，那么索引效率越高。
* 可以对一张表创建多个索引。索引的优点是提高了查询效率，缺点是在插入、更新和删除记录时，需要同时修改索引
* 对于主键，关系数据库会自动对其创建主键索引。使用主键索引的效率是最高的，因为主键会保证绝对唯一。

#### 唯一索引
添加唯一索引
```sql
ALTER TABLE student ADD UNIQUE INDEX uni_name (name);
```
只对某一列添加一个唯一约束而不创建唯一索引：
```sql
ALTER TABLE student ADD CONSTRAINT uni_name UNIQUE (name);
```
删除索引
```sql
alter table mytable drop index uni_name;
```

## 查看MySQL版本
https://github.com/mysql/mysql-server/releases/tag/mysql-8.0.36

## 进程模型
```
systemctl cat mysqld | grep ExecStart=
ExecStart=/usr/libexec/mysqld --basedir=/usr
```
单进程多线程模型
```
ps axf | grep mysqld
  16273 ?        Ssl    0:33 /usr/libexec/mysqld --basedir=/usr
ps -T 16273
    PID    SPID TTY      STAT   TIME COMMAND
  16273   16273 ?        Ssl    0:00 /usr/libexec/mysqld --basedir=/usr
  16273   16276 ?        Ssl    0:00 /usr/libexec/mysqld --basedir=/usr
  16273   16277 ?        Ssl    0:00 /usr/libexec/mysqld --basedir=/usr
	....
```
https://www.jcwlyf.com/newsContent-id-18272.html

mysqld进程
* 监听线程，接受客户端连接请求；
* 查询线程，处理客户端发来的SQL查询；
* 复制线程，处理主从复制相关操作；
* 后台线程，执行定期任务如日志清理、缓存刷新等。
