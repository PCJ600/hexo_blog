---
layout: next
title: MySQL基础
date: 2024-10-03 15:54:41
categories: MySQL
tags: MySQL
---

## 什么是SQL
SQL(Structured Query Language)是结构化查询语言，用来访问和操作数据库系统。定义了几种操作数据库的能力：
* DDL：Data Definition Language 允许用户定义数据，也就是创建表、删除表、修改表结构这些操作。
* DML：Data Manipulation Language 为用户提供添加、删除、更新数据的能力
* DQL：Data Query Language 允许用户查询数据，这也是最频繁的数据库日常操作。

MySQL是最流行的开源SQL数据库管理系统，由Oracle公司开发、分发和支持。

## MySQL8.0简介
数据类型
signed/unsigned integers 1,2,3,4,8 bytes log, float, double
char, varchar, text, blob
binary varbinary
date, time, datetime, timestamp, year
set, enum

每张表支持64个索引, 支持20万张表和50亿行
mysql, mysqladmin, mysqlcheck

## 查询MySQL版本和当前日期
```
select version(), current_date;
+-----------+--------------+
| version() | current_date |
+-----------+--------------+
| 8.0.36    | 2024-10-08   |
+-----------+--------------+
```

<!-- more -->

## 关系模型
关系模型本质上就是若干个存储数据的二维表
* 表的每一行称为记录（Record），记录是一个逻辑意义上的数据。
* 表的每一列称为字段（Column），同一个表的每一行记录都拥有相同的若干字段。
* 字段定义了数据类型（整型、浮点型、字符串、日期等），以及是否允许为NULL

### 主键(Primary Key)
* 关系表中的任意两条记录不能重复，通过某个字段能唯一区分出不同的记录，这个字段被称为主键
* 主键一般是采用完全业务无关的字段，这个字段通常为id，常见的id类型：
	* 自增整数类型：INT（约21亿）, BIGINT(约922亿亿)
	* 全局唯一GUID类型：也称UUID，使用一种全局唯一的字符串作为主键，类似8f55d96b-8acc-4636-8cb8-76bf8abc2f57。
* 关系数据库允许多个字段标识唯一记录，这种主键称为联合主键（很少用）

#### 创建表的时候设置主键自增
```sql
create table student ( id bigint primary key not null auto_increment, name varchar(32), class_id bigint);
```
### 外键

#### 添加外键约束
```sql
alter table student add constraint fk_class_id foreign key(class_id) references class(id);
```
#### 删除外键约束
```sql
alter table student drop foreign key fk_class_id;
```
注：由于外键约束会降低数据库的性能，大部分互联网应用程序为了追求速度，并不设置外键约束，而是仅靠应用程序自身来保证逻辑的正确性。

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

## 查询数据

### 创建表
创建两张表students, classes
```sql
-- 如果test数据库不存在，就创建test数据库：
CREATE DATABASE IF NOT EXISTS test;
-- 切换到test数据库
USE test;
-- 删除classes表和students表（如果存在）：
DROP TABLE IF EXISTS classes;
DROP TABLE IF EXISTS students;
-- 创建classes表：
CREATE TABLE classes (
    id BIGINT NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
-- 创建students表：
CREATE TABLE students (
    id BIGINT NOT NULL AUTO_INCREMENT,
    class_id BIGINT NOT NULL,
    name VARCHAR(100) NOT NULL,
    gender VARCHAR(1) NOT NULL,
    score INT NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
-- 插入classes记录：
INSERT INTO classes(id, name) VALUES (1, '一班');
INSERT INTO classes(id, name) VALUES (2, '二班');
-- 插入students记录：
INSERT INTO students (id, class_id, name, gender, score) VALUES (1, 1, '小明', 'M', 90);
INSERT INTO students (id, class_id, name, gender, score) VALUES (2, 1, '小红', 'F', 95);

-- OK:
SELECT 'ok' as 'result:';
```

### 查询所有数据
```sql
select * from students;
```
### 条件查询
例：查询分数大于92分的学生
```sql
select * from students where score > 92;
```

例：查询分数不低于92分的女生 (AND条件查询)
```sql
select * from students where score >= 92 and gender = 'F';
```

例：查询分数大于92分的学生，或者男生(OR条件查询)
```sql
select * from students where score > 92 or gender = 'M';
```

例：查询不在一班的学生(NOT查询)
```sql
select * from students where not class_id = 2;
```
NOT class_id = 2 可以写成 class_id <> 2, 因此NOT查询不是很常用。 例：

如果不加括号，条件运算按照NOT, AND, OR的优先级进行
```sql
select * from students where (score < 80 or score > 90) and gender = 'M';
```

例：查询分数在90分(含)-95分(含)的学生(BETWEEN AND)
```sql
select * from students where score between 90 and 95;
```
BETWEEN AND用法参考: [https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html#operator_between](https://dev.mysql.com/doc/refman/8.0/en/comparison-operators.html#operator_between)

### 查询指定的列
```sql
select id, name from students;
```
使用投影查询，并将列名重命名。
例：查询学生id, score, name, 列名score重命名为points
```sql
select id, score points, name from students;
+----+--------+----------+
| id | points | name     |
+----+--------+----------+
|  1 |     90 | XiaoMing |
|  2 |     95 | XiaoHong |
+----+--------+----------+
```

### 排序
使用order by子句，按成绩从低到高排序
```sql
mysql> select id, name, score from students order by score desc;
+----+----------+-------+
| id | name     | score |
+----+----------+-------+
|  2 | XiaoHong |    95 |
|  1 | XiaoMing |    90 |
+----+----------+-------+
```
如果score列有相同的数据，需要进一步排序，可以继续添加列
例：查询一班的学生分数, 按照(分数,性别)排序
```sql
SELECT id, name, gender, score FROM students ORDER BY score DESC, gender where class_id = 1;
```

### 计算日期
MONTH函数返回月份
```
mysql> SELECT name, birth, MONTH(birth) FROM pet;
+----------+------------+--------------+
| name     | birth      | MONTH(birth) |
+----------+------------+--------------+
| Fluffy   | 1993-02-04 |            2 |
| Claws    | 1994-03-17 |            3 |
```

### 处理NULL
使用IS NULL和IS NOT NULL，不要用=
```
mysql> SELECT 1 IS NULL, 1 IS NOT NULL;
+-----------+---------------+
| 1 IS NULL | 1 IS NOT NULL |
+-----------+---------------+
|         0 |             1 |
+-----------+---------------+
```
和NULL做算术比较的结果仍然是NULL
```
mysql> SELECT 1 = NULL, 1 <> NULL, 1 < NULL, 1 > NULL;
+----------+-----------+----------+----------+
| 1 = NULL | 1 <> NULL | 1 < NULL | 1 > NULL |
+----------+-----------+----------+----------+
|     NULL |      NULL |     NULL |     NULL |
+----------+-----------+----------+----------+
mysql> SELECT 0 IS NULL, 0 IS NOT NULL, '' IS NULL, '' IS NOT NULL;
+-----------+---------------+------------+----------------+
| 0 IS NULL | 0 IS NOT NULL | '' IS NULL | '' IS NOT NULL |
+-----------+---------------+------------+----------------+
|         0 |             1 |          0 |              1 |
+-----------+---------------+------------+----------------+
```
In MySQL, 0 or NULL means false and anything else means true. The default truth value from a boolean operation is 1.

### 分页查询
select查询结果数据量很大，可以分页显示。 通过`LIMIT <N-M> OFFSET <M>`子句实现：
例：把所有学生按照成绩从高到低排序分页显示，获取前三条记录
```bash
mysql>select * from students order by score desc limit 3 offset 0;
+----+----------+----------+--------+-------+
| id | class_id | name     | gender | score |
+----+----------+----------+--------+-------+
|  2 |        1 | XiaoHong | F      |    95 |
|  8 |        3 | 小新     | F      |    91 |
|  1 |        1 | XiaoMing | M      |    90 |
+----+----------+----------+--------+-------+
```

例：查询第7,8,9行数据(第三页)
```bash
mysql>select * from students order by score desc limit 3 offset 6;
+----+----------+--------+--------+-------+
| id | class_id | name   | gender | score |
+----+----------+--------+--------+-------+
| 10 |        3 | 小丽   | F      |    85 |
|  5 |        2 | 小白   | F      |    81 |
|  4 |        1 | 小米   | F      |    73 |
+----+----------+--------+--------+-------+
```

OFFSET超过了查询最大数量也不会报错，会返回一个空的数据集
```mysql
select * from students order by score desc;
```

### 聚合查询
查询表中一共多少条记录，用内置的COUNT()函数查询
```bash
mysql> select COUNT(*) from students where gender = 'M';
+----------+
| COUNT(*) |
+----------+
|        5 |
+----------+
```

例: 每页3条记录，如何通过聚合查询获得总页数
```bash
mysql> select ceiling((count(*)/3)) from students;
+-----------------------+
| ceiling((count(*)/3)) |
+-----------------------+
|                     4 |
+-----------------------+
```

SQL还提供了以下聚合函数:
* SUM 求和
* AVG 平均数
* MAX 最大值
* MIN 最小值

例: 使用AVG函数求学生平均分
```sql
select AVG(score) from students;
```

### 分组查询
例如：查询每个班学生数量
```bash
mysql> select class_id, count(*) num from students group by class_id;
+----------+-----+
| class_id | num |
+----------+-----+
|        1 |   4 |
|        2 |   3 |
|        3 |   3 |
+----------+-----+
```

也可以使用多个列进行分组，例如: 统计各班男生和女生人数
```
select class_id, gender, count(*) num from students group by class_id, gender;
+----------+--------+-----+
| class_id | gender | num |
+----------+--------+-----+
|        1 | M      |   2 |
|        1 | F      |   2 |
|        2 | F      |   1 |
|        2 | M      |   2 |
|        3 | F      |   2 |
|        3 | M      |   1 |
+----------+--------+-----+
```

### 多表查询
```sql
select * from students, classes;
+----+----------+----------+--------+-------+----+--------+
| id | class_id | name     | gender | score | id | name   |
+----+----------+----------+--------+-------+----+--------+
|  1 |        1 | XiaoMing | M      |    90 |  3 | 三班    |
|  1 |        1 | XiaoMing | M      |    90 |  2 | 二班    |
+----+----------+----------+--------+-------+----+--------+
```sql
结果中有两列id和name, 不容易区分, 要解决这个问题, 可以利用投影查询的"设置列的别名"
```mysql
mysql> select students.id sid, students.name, classes.id cid, classes.name cname from students, classes;
+-----+----------+-----+--------+
| sid | name     | cid | cname  |
+-----+----------+-----+--------+
|   1 | XiaoMing |   3 | 三班   |
|   1 | XiaoMing |   2 | 二班   |
```
用表名.列名方式有点麻烦，SQL还允许给表设置一个别名
```sql
mysql> select s.id sid, s.name, c.id cid, c.name from students s, classes c;
+-----+----------+-----+--------+
| sid | name     | cid | name   |
+-----+----------+-----+--------+
|   1 | XiaoMing |   3 | 三班   |
|   1 | XiaoMing |   2 | 二班   |
|   1 | XiaoMing |   1 | 一班   |
+-----+----------+-----+--------+
```

### 连接查询
INNER JOIN 返回同时存在于两张表的数据
例：选出所有学生，同时返回班级和姓名
```sql
mysql> select s.id, s.name, s.class_id, c.name class_name, s.gender, s.score from students s inner join classes c on s.class_id = c.id;
+----+----------+----------+------------+--------+-------+
| id | name     | class_id | class_name | gender | score |
+----+----------+----------+------------+--------+-------+
|  1 | XiaoMing |        1 | 一班       | M      |    90 |
|  2 | XiaoHong |        1 | 一班       | F      |    95 |
...
```

RIGHT OUTER JOIN 返回右表都存在的行，如果某一行只在右表存在，结果集以NULL填充剩下的字段:
```sql
mysql> select s.id, s.name, s.class_id, c.name class_name, s.gender, s.score from students s right outer join classes c on s.class_id = c.id;
+------+----------+----------+------------+--------+-------+
| id   | name     | class_id | class_name | gender | score |
+------+----------+----------+------------+--------+-------+
|    4 | 小米     |        1 | 一班       | F      |    73 |
|    3 | 小军     |        1 | 一班       | M      |    88 |
|    2 | XiaoHong |        1 | 一班       | F      |    95 |
|    1 | XiaoMing |        1 | 一班       | M      |    90 |
|    7 | 小林     |        2 | 二班       | M      |    85 |
|    6 | 小兵     |        2 | 二班       | M      |    55 |
|    5 | 小白     |        2 | 二班       | F      |    81 |
|   10 | 小丽     |        3 | 三班       | F      |    85 |
|    9 | 小王     |        3 | 三班       | M      |    89 |
|    8 | 小新     |        3 | 三班       | F      |    91 |
| NULL | NULL     |     NULL | 四班       | NULL   |  NULL |
+------+----------+----------+------------+--------+-------+
```

CROSS JOIN 笛卡尔积
```
mysql> select * from students cross join classes;
+----+----------+----------+--------+-------+----+--------+
| id | class_id | name     | gender | score | id | name   |
+----+----------+----------+--------+-------+----+--------+
|  1 |        1 | XiaoMing | M      |    90 |  4 | 四班   |
|  1 |        1 | XiaoMing | M      |    90 |  3 | 三班   |
|  1 |        1 | XiaoMing | M      |    90 |  2 | 二班   |
|  1 |        1 | XiaoMing | M      |    90 |  1 | 一班   |
|  2 |        1 | XiaoHong | F      |    95 |  4 | 四班   |
|  2 |        1 | XiaoHong | F      |    95 |  3 | 三班   |
|  2 |        1 | XiaoHong | F      |    95 |  2 | 二班   |
|  2 |        1 | XiaoHong | F      |    95 |  1 | 一班   |
|  3 |        1 | 小军     | M      |    88 |  4 | 四班   |
|  3 |        1 | 小军     | M      |    88 |  3 | 三班   |
|  3 |        1 | 小军     | M      |    88 |  2 | 二班   |
|  3 |        1 | 小军     | M      |    88 |  1 | 一班   |
...
+----+----------+----------+--------+-------+----+--------+
```

### 模式匹配

#### LIKE和NOT LIKE
use _ to match any single character and % to match an arbitrary number of characters (including zero characters).

查找b开头的names
```sql
mysql> SELECT * FROM pet WHERE name LIKE 'b%';
+--------+--------+---------+------+------------+------------+
| name   | owner  | species | sex  | birth      | death      |
+--------+--------+---------+------+------------+------------+
| Buffy  | Harold | dog     | f    | 1989-05-13 | NULL       |
| Bowser | Diane  | dog     | m    | 1989-08-31 | 1995-07-29 |
```

查找fy结尾的names
```sql
mysql> SELECT * FROM pet WHERE name LIKE '%fy';
+--------+--------+---------+------+------------+-------+
| name   | owner  | species | sex  | birth      | death |
+--------+--------+---------+------+------------+-------+
| Fluffy | Harold | cat     | f    | 1993-02-04 | NULL  |
| Buffy  | Harold | dog     | f    | 1989-05-13 | NULL  |
```

查找包含w的names
```
mysql> SELECT * FROM pet WHERE name LIKE '%w%';
+----------+-------+---------+------+------------+------------+
| name     | owner | species | sex  | birth      | death      |
+----------+-------+---------+------+------------+------------+
| Claws    | Gwen  | cat     | m    | 1994-03-17 | NULL       |
| Bowser   | Diane | dog     | m    | 1989-08-31 | 1995-07-29 |
| Whistler | Gwen  | bird    | NULL | 1997-12-09 | NULL       |
+----------+-------+---------+------+------------+------------+
```



## 修改数据
关系数据库的基本操作是增删改查，即CRUD：Create、Retrieve、Update、Delete。

### 插入数据
例: 向students表插入一条学生记录
```
INSERT INTO students (class_id, name, gender, score) VALUES (2, '大牛', 'M', 80);
```
也可以一次性插入多条记录, 用逗号隔开
```
INSERT INTO students (class_id, name, gender, score) VALUES (2, '二牛', 'M', 80), (2, '三牛', 'M', 80);
```

### 更新数据
使用update语句更新数据库表中记录。例：把Id=1的学生姓名改成学霸, 分数改成100分
```sql
update students set name="学霸", score=100 where id=1;
```
update语句可以一次更新多条数据。例：把id=1,2,3的学生分数改成80
```sql
update students set score=80 where id>=1 and id<=3;
```
update语句更新字段时可以使用表达式。例：把所有低于90分的分数加10分：
```sql
update students set score=score-10 where score <= 90;
```
update语句可以没有where条件，此时会更新表中所有记录。例：所有学生分数减5分
```sql
update students set score=score-5
```

### 删除数据
使用delete语句删除数据库表中记录，delete的基本语法:
```sql
delete from <表名> where ...;
```
例：删除表中id=1的记录
```sql
delete from students where id=1;
```
如果where条件没有匹配到记录，delete语句也不会报错，不会有记录被删除
```bash
mysql> delete from students where id=99999;                                 │
Query OK, 0 rows affected (0.00 sec)
```

使用不带where的delete语句，可以删除整张表数据
```bash
mysql> delete from students;                                                │
Query OK, 12 rows affected (0.00 sec)
```
注: delete仅删除表数据, drop连表数据和表结构一起删除


## 管理MySQL
### 列出所有数据库
```sql
show database;
```
### 创建数据库
```sql
create database test;
```
### 删除数据库
```sql
drop database test;
```
### 切换数据库
对一个数据库进行操作前，需要先切换到该数据库。
```sql
use test;
```
### 列出当前数据库的所有表
```sql
show tables;
```
### 查看一个表结构
```sql
desc students;
```
### 查看创建表的SQL语句
```sql
mysql> show create table classes;
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table   | Create Table                                                                                                                                                                  |
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| classes | CREATE TABLE `classes` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `name` varchar(100) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8mb3 |
+---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```
### 删除表(表结构+表数据)
```sql
drop tables students;
```
### 给表新增一列
例：给students表新增一列birth，使用`alter table`语句
```sql
alter table students add column birth varchar(10) not null;
```
### 修改表中的某一列
例: 修改birth列，把名称改为birthday, 类型改为varchar(20)
```sql
alter table students change column birth birthday varchar(20) not null;
```
### 删除列
```sql
alter table students drop column birthday;
```

### 快照
复制一张表数据刀新表，可以使用`create table`和`select`:
```sql
create table students_copy select * from students;
```

## 事务
执行SQL语句时，某些业务场景会要求一系列操作必须全部执行，不能只执行一部分，比如说转账操作:
```sql
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
```
这两条语句必须全部执行，如果因为某些原因，第一条执行成功，第二条失败，就必须全部撤销。

### 事务的ACID特性
* Atomicity 原子性 所有SQL作为一个整体，要么全执行，要么全不执行
* Consistency 一致性 事务完成后, 所有数据状态是一致的，即A账户减去100元，B账户必定加100元
* Isolation 隔离性 如果多个事务并发执行，每个事务做出修改必须与其他事务隔离
* Durability 持久性 事务完成后, 对数据库修改被持久化存储

### 手动执行事务
使用begin开启事务，使用commit提交事务
```sql
begin;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
commit;
```

### 手动回滚事务
使用rollback回滚事务
```sql
begin;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
rollback;
```

### 隔离级别

两个并发执行事务，操作同一条表记录时，可能会存在数据不一致问题，包括脏读,不可重复读,幻读。
SQL标准定义了4种隔离级别

|  隔离级别   | 脏读(Dirty Read)  | 不可重复读(Non Repeatable Read) | 幻读(Phantom Read) |
|  ----  | ----  |  ----  | ----  |
| Read Uncommitted  | Y | Y | Y |
| Read Committed  | - | Y | Y |
| Repeatable Read  | - | - | Y |
| Serializable  | Y | Y | Y |

#### Read Uncommitted
Read Uncommitted级别下，一个事务会读到另一个事务未提交的数据，如果另一个事务回滚，当前事务读到的就是脏数据，即脏读(Dirty Read)

举例: 创建一张学生表, 插入一条学生记录
```bash
create table students (id bigint not null auto_increment, name varchar(36) not null, primary key(id)) engine=InnoDB default charset=utf8;
insert into students(name) values('Alice');
mysql> select * from students;
+----+-------+
| id | name  |
+----+-------+
|  1 | Alice |
+----+-------+
1 row in set (0.00 sec)
```
分别开两个MySQL连接，按顺序执行事务A和事务B

| 时刻	| 事务A	| 事务B |
| ----  |  ----  | ----  |
|1|	SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; |	SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED; |
|2|	BEGIN; | BEGIN; |
|3|	UPDATE students SET name='Bob' WHERE id = 1; | |
|4| | SELECT * FROM students WHERE id = 1; |
|5| ROLLBACK; | |
|6|	| SELECT * FROM students WHERE id = 1; |
|7|	| COMMIT; |

![](read-uncommitted.png)

说明：
* 事务A执行完第三步后，更新了id=1的记录，但未提交，事务B在第4步读到了Bob这一条未提交的数据
* 事务A在第五步进行了回滚，事务B再次读取id=1的记录为Alice, 和上一次的数据不一致，这就是脏读

#### Read Committed

#### Repeatable Read

#### Serializable

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

## 参考
https://liaoxuefeng.com/books/sql/transaction/index.html
https://dev.mysql.com/doc/refman/8.0/en/