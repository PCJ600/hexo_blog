---
layout: next
title: MySQL基础
date: 2024-10-03 15:54:41
categories: MySQL
tags: MySQL
---

## 数据库相关概念
* 数据库(DataBase) 存储数据仓库
* 数据库管理系统(DateBase Management System)(DBMS) 操纵和管理数据库的大型软件
* SQL(Structured Query Language)(SQL) 操作关系型数据库的编程语言，定义了一套操作关系数据库的统一标准 

## 关系模型
关系模型本质上就是若干个存储数据的二维表
* 表的每一行称为记录（Record），记录是一个逻辑意义上的数据。
* 表的每一列称为字段（Column），同一个表的每一行记录都拥有相同的若干字段。
* 字段定义了数据类型（整型、浮点型、字符串、日期等），以及是否允许为NULL

## 什么是SQL
SQL(Structured Query Language)是结构化查询语言，用来访问和操作数据库系统。定义了几种操作数据库的能力：
* DDL：Data Definition Language 允许用户定义数据，也就是创建表、删除表、修改表结构这些操作。
* DML：Data Manipulation Language 为用户提供添加、删除、更新数据的能力
* DQL：Data Query Language 允许用户查询数据，这也是最频繁的数据库日常操作。

<!-- more -->

## DDL-数据库操作

### 查询所有数据库
```
show databases;
```
### 查询当前数据库
```
show database();
```
### 创建表
```
create database [if not exists] 数据库名 [default charset 字符集] [COLLATE 排序规则];
```
### 删除数据库
```
drop database [if exists] 数据库名;
```
### 使用数据库
```
use 数据库名;
```

## DDL-表操作-创建&查询

### 查询当前数据库所有表
```
show tables;
```
### 查询表结构
```
desc 表名;
```
### 查询指定表的建表语句
```
show create table 表名;
```
### 创建表
```
create table 表名 {
	字段1 字段1类型[COMMENT 字段1注释],
	字段2 字段2类型[COMMENT 字段2注释],
	...
	字段N 字段N类型[COMMENT 字段2注释],
}[COMMENT 表注释];
```
例: 创建一个学生表
```
create table student ( 
	id bigint primary key not null auto_increment, 
	name varchar(32), 
	class_id bigint
);
```
## DDL-数据类型
参考[官方手册](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)

MySQL数据类型分三类
* 数值类型
* 字符串类型
* 日期时间类型

### 数值类型
| 分类 | 类型 | 大小 | 有符号范围 | 无符号范围 |
| -- | -- | -- | -- | -- |
| TINYINT | 1 byte | (-128,127) | (0,255) | 小整数值 |
| SMALLINT | 2 bytes | (-32768,32767) | (0,65535) | 大整数值 |
| MEDIUMINT | 3 bytes | (-2^23, 2&23-1) | (0, 2^24-1) | 大整数值 |
| INT或INTEGER | 4 bytes | (-2^31, 2^31-1) | (0, 2^32-1) | 大整数值 |
| BIGINT | 8 bytes | (-2^63, 2^63-1) | (0, 2^64-1) | 极大整数值 |
| FLOAT | 4 bytes |			| 		| 单精度浮点整数值 |
| DOUBLE | 8 bytes | 		| 		| 双精度浮点整数值 |
| DECIMAL |			| 依赖于M和D的值 | 依赖于M和D的值 | 小整数值(精确定点数) |

### 字符串类型
| 类型 | 大小 | 描述 |
| -- | -- | -- |
| CHAR | 0-255 bytes | 定长字符串 |
| VARCHAR | 0-65535 bytes | 变长字符串 |
| TINYBLOB | 0-255 bytes | 二进制数据 |
| TINTTEXT | 0-255 bytes | 短文本字符串 |
| BLOB | 0-65535 bytes | 二进制形式的文本数据 |
| TEXT | 0-65535 bytes | 长文本数据 |
| MEDIUMBLOB | 0-2^24-1 | 二进制形式的中等长度文本数据 |
| MEDIUMTEXT | 0-2^24-1 | 中等长度的文本数据 |
| LONGBLOB | 0-2^32-1 | 二进制形式的极大文本数据 |
| LONGTEXT | 0-2^32-1 bytes| 极大的文本数据 |

### 日期类型
| 类型 | 大小 | 范围 | 格式 |
| -- | -- | -- | -- |
| DATE | 3 | 1000-01-01 至 9999-12-31 | YYYY-MM-DD |
| TIME | 3 | -838:59:59 至 838:59:59 | HH:MM:SS |
| YEAR | 1 | 1901 至 2155 | YYYY |
| DATETIME | 8 | 1000-01-01 00:00:00 至 9999-12-31 23:59:59 | YYYY-MM-DD HH:MM:SS |
| TIMESTAMP | 4 | 1970-01-01 00:00:01 至 2038-01-19 03:14:07 | YYYY-MM-DD HH:MM:SS |

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

## DDL-表操作-修改

### 添加字段
```
alter table 表名 add 字段名 类型(长度) [COMMENT 注释] [约束];
```
例: 给emp表添加字段nickname
```
alter table emp add nickname varchar(20);
```
### 修改数据类型
```
alter table 表名 modify 字段名 新数据类型(长度);
```
### 修改字段名和字段类型
```
alter table 表名 change 旧字段名 新字段名 类型(长度) [COMMENT 注释] [约束];
```
例：把emp表的nickname字段改成username, 类型varchar(24)
```
alter table emp change nickname username varchar(24);
```
### 删除字段
```
alter table 表名 drop 字段名;
```
例：将emp表的username字段删除
```
alter table emp drop username;
```
### 修改表名
```
alter table 表名 rename to 新表名;
```
例: 将emp表名修改为employee
```
alter table emp rename to employee;
```
### 删除表
删除表(包括表结构和内容)
```
DROP TABLE[IF EXISTS] 表名
```
删除表，并重新创建该表
```
truncate table 表名;
```

## DML-添加数据
DML全称(Data Manipulation Language), 用于对数据库中表的数据记录进行增、删、改操作

### 给指定字段添加数据
```
insert into 表名(字段1,字段2) values(值1,值2);
```
### 给全部字段添加数据
```
insert into 表名 values(值1,值2);
```
### 批量添加多条数据
```
insert into 表名 values(值1,值2),(值1,值2);
```
注：
* 插入数据时，指定字段数据需要和值顺序一一对应
* 字符串和日期类型应包含在引号中
* 插入数据大小应该在字段的指定范围内

例：插入2条员工数据
```
insert into employee values(1,'001','peter','M',18,'123456789987654321', '2024-10-14');
```

## DML-修改数据
```
update 表名 set 字段1=值1, 字段2=值2 [where 条件];
```
例: 将id为1数据的name修改为'lance'，性别修改为F
```
update employee set name='lance', gender='F' where id=1;
```
例: 所有员工入职日期修改为2005-01-01
```
update employee entrydate = '2005-01-01';
```

## DML-删除数据
```
delete from 表名 [where 条件]
```
例: 删除女性员工
```
delete from employee where gender='F';
```
例: 删除所有员工
```
delete from employee;
```

## DQL-查询
语法
```
select
	字段
from 
	表名
where 
	条件
group by
	分组字段
having
	分组后条件
order by
	排序字段
limit
	分页参数
```

### 查询多个字段
```
select 字段1,字段2 from 表名;
select * from 表名;
select 字段1 [AS 别名1], 字段2 [AS 别名2] from 表名;
```
其中AS可以省略

### 去除重复
```
select distinct 字段1 from 表名;
```

## DQL-条件查询
语法
```
select 字段列表 from 表名 where 条件列表;
```
条件
* 比较运算符 >, >=, <, <= , =, <>或!=, BETWEEN...AND..., IN(...), LIKE, IS NULL
* 逻辑运算符 AND或&&, OR或||, NOT或！

例：查询年龄在15-20岁员工
```
select * from emp where age between 15 and 20;
select * from emp where age >= 15 and age <= 20;
select * from emp where age in (15,16,17,18,19,20);
```
例: 查询姓名为两个字符员工
```
select * from emp where name like '__';
```
例: 查询身份证最后一位为X员工
```
select * from emp where idcard like '%X';
```

## DQL-聚合函数

常见聚合函数
| 函数 | 功能 |
| -- | -- |
| count | 计数 |
| max | 求最大值 |
| min | 求最小值 |
| avg | 求平均数 |
| sum | 求和 |

注: NULL值不参与聚合函数运算

例：求员工数, 员工平均年龄, 最大年龄，最小年龄，年龄之和
```
select count(*), avg(age),max(age),min(age),sum(age) from emp;
```

## DQL-分组查询

语法
```
select 字段列表 from 表名 [where 条件] group by 分组字段名 [having 分组后过滤条件];
```
注：
* where是分组前过滤，不满足where条件不参与分组；having是分组后对结果过滤
* where不能对聚合函数判断，having可以
* 分组后，查询字段一般为分组字段和聚合函数，查询其他字段无意义

例：查询男性员工和女性员工个数
```
select gender, count(*) from emp group by gender;
```

## DQL-排序查询

```
select 字段 from 表名 group by 字段1 排序方式1, 字段2 排序方式2;
```
排序方式:
* ASC 升序(默认值，可以不写)
* DESC 降序

例：根据年龄对员工升序排序，如年龄相同按入职时间降序排序
```
select * from emp order by age, entrydate desc;
```

## DQL-分页查询
```
select 字段 from 表名 limit 起始索引,查询记录数;
```
注：
* 起始索引从0开始, =（查询页码-1)*每页显示记录数
* 每个数据库对分页查询实现不同，MySQL中是limit
* 如果查询的是第一页数据，索引可以省略，直接写limit XXX

例：查询第1页员工数据，，每页10条记录
```
select * from emp limit 10;
```
例：查询第3页员工数据，每页10条记录
```
select * from emp limit 20,10;
select * from emp limit 10 offset 20;
```

## SQL-DCL-用户管理/权限控制
<!-- TODO -->

## 函数

### 字符串函数

常用
| 函数 | 功能 |
| -- | -- |
| CONCAT(s1,s2,...sn) | 拼接 |
| LOWER(str) | 全部字符转小写 |
| UPPER(str) | 全部字符转大写 |
| LPAD(str,n,pad) | 左填充，用字符串pad对str左侧填充，直到字符串长度为n |
| RPAD(str,n,pad) | 右填充，用字符串pad对str右侧填充，直到字符串长度为n |
| TRIM(str) | 去除头尾空格 |
| SUBSTRING(str,start,len) | 返回从字符串str的start位置起len个长度字符串 (首字符位置为1，不是0)|

例: 员工工位不足5位，低位补0到5位
```
update emp set workno = LPAD(workno, 5, '0') where workno = '001';
```

### 数值函数
| 函数 | 功能 |
| -- | -- |
| CEIL(x) | 向上取整 |
| FLOOR(x) | 向下取整 |
| MOD(x,y) | 返回x/y的模 |
| RAND() | 返回0-1随机数 |
| ROUND(x,y) | 求参数x的四舍五入值，保留y位小数 |

```
mysql> select ceil(1.1),floor(1.2),mod(7,4),rand(),ROUND(0.123,2);
+-----------+------------+----------+---------------------+----------------+
| ceil(1.1) | floor(1.2) | mod(7,4) | rand()              | ROUND(0.123,2) |
+-----------+------------+----------+---------------------+----------------+
|         2 |          1 |        3 | 0.19162892474944604 |           0.12 |
+-----------+------------+----------+---------------------+----------------+
```

例: 通过数据库函数，生成一个六位数的随机验证码
```
select LPAD(ROUND(rand()*1000000,0), 6, '0');
```

### 日期函数
| 函数 | 功能 |
| -- | -- |
| CURDATE() | 返回当前日期 |
| CURTIME() | 返回当前时间 |
| NOW() | 返回当前时间和日期 |
| YEAR(date) | 获取指定date年份 |
| MONTH(date) | 获取指定date月份 |
| DAY(date) | 获取指定date日期 |
| DATE_ADD(date, INTERVAL expr type) | 一个日期加上加一个时间间隔expr后的时间 |
| DATEDIFF(date1, date2) | 返回起始时间date1减去结束时间date2之间的天数 |

例:
```
mysql> select CURDATE(), CURTIME(), NOW(), YEAR(NOW()), date_add(now(), INTERVAL 70 day);
+------------+-----------+---------------------+-------------+----------------------------------+
| CURDATE()  | CURTIME() | NOW()               | YEAR(NOW()) | date_add(now(), INTERVAL 70 day) |
+------------+-----------+---------------------+-------------+----------------------------------+
| 2024-10-15 | 15:05:38  | 2024-10-15 15:05:38 |        2024 | 2024-12-24 15:05:38              |
+------------+-----------+---------------------+-------------+----------------------------------+
```

例: datediff
```
mysql> select datediff('2021-12-01', '2021-11-01');
+--------------------------------------+
| datediff('2021-12-01', '2021-11-01') |
+--------------------------------------+
|                                   30 |
+--------------------------------------+

mysql> select datediff('2021-10-01', '2021-11-01');
+--------------------------------------+
| datediff('2021-10-01', '2021-11-01') |
+--------------------------------------+
|                                  -31 |
+--------------------------------------+
```

### 流程函数
| 函数 | 功能 |
| -- | -- |
| IF(value,t,f) | 如果value为true, 返回t, 否则返回f |
| IFNULL(value1, value2) | 如果value1不为空，返回value1, 否则返回value2 |
| CASE WHEN [val1] THEN[res1] ... ELSE [default] END | 如果val1为true，返回res1, 否则返回default默认值 |
| CASE [expr] WHEN [val1] THEN [res1] ... ELSE [default] END | 如果expr值等于val1，返回res1, 否则返回default默认值 |


例：if和ifnull
```
mysql> select if(true, 'ok', 'error'), ifnull('ok', 'default'), ifnull(null, 'default');
+-------------------------+-------------------------+-------------------------+
| if(true, 'ok', 'error') | ifnull('ok', 'default') | ifnull(null, 'default') |
+-------------------------+-------------------------+-------------------------+
| ok                      | ok                      | default                 |
+-------------------------+-------------------------+-------------------------+
```

例: 查找员工姓名和工作城市，如果城市在北京或上海打印'一线城市', 否则打印'二线城市'
```
select name, (case city when '北京' then '一线城市' when '上海' then '一线城市' else '二线城市' end) as '工作地址' from emp;
select name, (case when city in ('北京','上海') then '一线城市' else '二线城市' end) as '工作地址' from emp;
```

## 约束
| 约束 | 描述 | 关键字 |
| -- | -- | -- |
| 非空约束 | 限制字段的value不能为null | NOT NULL |
| 唯一约束 | 限制字段的所有数据都是唯一，不重复的 | UNIQUE |
| 主键约束 | 非空且唯一 | PRIMARY KEY |
| 外键约束 | 用于两张表建立连接，保证数据一致性和完整性 | FOREIGN KEY |

例：创建表
| 字段名 | 字段含义 | 字段类型 | 约束条件 | 约束关键字 |
| -- | -- | -- | -- | -- |
| id | ID | int | 主键，自动增长 | primary key, auto increment |
| name | 姓名 | varchar(10) | 不为空，且唯一 | not null, unique |
| age | 年龄 | int | 大于0, 小于120 | CHECK |
| status | 状态 | char(1) | 默认值为1 | DEFAULT |
| gender | 性别 | char(1) | 无 | 无 |
```
create table user(
	id int primary key auto_increment,
	name varchar(10) not null unique, 
	age int check(age > 0 && age < 120),
	status char(1) default '1',
	gender char(1)
)Engine=InnoDB;

-- 插入数据违反约束，插入失败
insert into user(name,age,status,gender) values('tom1',19,'1','M'),('tom2',20,'0','F');
mysql> insert into user(name,age,status,gender) values('tom1',19,'1','M');
ERROR 1062 (23000): Duplicate entry 'tom1' for key 'user.name'
mysql> insert into user(name,age,status,gender) values('tom3',190,'1','M');
ERROR 3819 (HY000): Check constraint 'user_chk_1' is violated.
```

### 主键约束
* 关系表中的任意两条记录不能重复，通过某个字段能唯一区分出不同的记录，这个字段被称为主键
* 主键一般是采用完全业务无关的字段，这个字段通常为id，常见的id类型：
	* 自增整数类型：INT（约21亿）, BIGINT(约922亿亿)
	* 全局唯一GUID类型：也称UUID，使用一种全局唯一的字符串作为主键，类似8f55d96b-8acc-4636-8cb8-76bf8abc2f57。
* 关系数据库允许多个字段标识唯一记录，这种主键称为联合主键（很少用）

### 外键约束
外键用来让两张表数据之间建立连接，从而保证数据一致性和完整性

举例：创建两张表：部门表，员工表
![](image_fk1.png)
```
create table dept(id int auto_increment primary key, name varchar(50) not null);
insert into dept(id, name) values(1, '研发'), (2, '市场'), (3, '财务'), (4,'销售'), (5, '总经办');
create table emp(id int auto_increment primary key, name varchar(50) not null,age int, job varchar(20), salary int, entrydate date, managerid int,dept_id int);
insert into emp(id,name,age,job,salary,entrydate,managerid,dept_id)  values(1, '金庸', 66,'总裁',20000,'2000-01-01', 1 ,5);
```

#### 添加外键
方法1：创建表的时候添加外键
```
create table 表名(
	[CONSTRAINT] [外键名称] FOREIGN KEY(外键字段名) REFERENCES 主表(主表列名)
);
```
方法2：表创建完成后，使用alter table添加约束
```
alter table 表名 add constraint 外键名称 foreign key(外键字段名) references 主表(主表列名);
```

例：给emp表添加外键约束fk_emp_dept_id, 字段名为dept_id，主表列名为dept表中的id
```
alter table tmp add constraint fk_emp_dept_id foreign key(dept_id) references dept(id);
```

#### 删除外键
```
alter table emp drop foreign key fk_emp_dept_id;
```

#### 删除/更新行为
| 行为 | 说明 |
| -- | -- |
| NO ACTION/RESTRICT | 在父表中删除/更新对应记录时，检查该记录是否有外键，如果有则不允许删除/更新 (与RESTRICT一致) |
| CASCADE |  在父表中删除/更新对应记录时，检查该记录是否有对应外键，如果有则同时删除/更新外键在子表中的记录 |
| SET NULL | 在父表中删除/更新对应记录时，检查该记录是否有对应外键，如果有则也删除/更新外键在子表中的记录 |
| SET DEFAULT | 父表有变更时，子表将外键列设置成一个默认值(InnoDB不支持) |


例：设置级联更新/级联删除
```
alter table 表名 add constraint 外键名称 foreign key (外键字段) references 主表名(主表字段名) on update cascade on delete cascade;
```
给员工表设置级联更新，级联删除
```
alter table emp add constraint fk_emp_dept_id foreign key(dept_id) references dept(id) on update cascade on delete cascade;
```
例：更新部门表中id=5记录，把id改成6，由于设置了级联更新，员工表中dept_id=5的记录也被修改成dept_id=6
```
mysql> select * from emp where dept_id=5;
+----+--------+------+--------+--------+------------+-----------+---------+
| id | name   | age  | job    | salary | entrydate  | managerid | dept_id |
+----+--------+------+--------+--------+------------+-----------+---------+
|  1 | 金庸   |   66 | 总裁   |  20000 | 2000-01-01 |         1 |       5 |
+----+--------+------+--------+--------+------------+-----------+---------+
mysql> select * from dept where id=5;
+----+-----------+
| id | name      |
+----+-----------+
|  5 | 总经办    |
+----+-----------+

-- 级联更新
mysql> update dept set id=6 where id=5;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from emp;
+----+--------+------+--------+--------+------------+-----------+---------+
| id | name   | age  | job    | salary | entrydate  | managerid | dept_id |
+----+--------+------+--------+--------+------------+-----------+---------+
|  1 | 金庸   |   66 | 总裁   |  20000 | 2000-01-01 |         1 |       6 |
+----+--------+------+--------+--------+------------+-----------+---------+
1 row in set (0.00 sec)

-- 级联删除
mysql> delete from dept where id=6;
Query OK, 1 row affected (0.00 sec)

mysql> select * from emp;
Empty set (0.00 sec)
```

## 多表查询

### 多表关系
* 一对多
案例：比如员工表和部门表，一个部门有多个员工，一个员工只属于一个部门。
实现：多的一方建立外键，指向一的一方主键。
* 多对多
案例：比如学生和课程的关系。一个学生可以选修多个课程，一门课程可以供多个学生选择
实现：建立中间表，包含两个外键，分别关联两张表的主键
* 一对一
案例：用户与用户详情关系
实现：任意一方加入外键，关联另外一方的主键，并且设置外键为UNIQUE

### 笛卡尔积
```
select * from emp, dept;
select * from emp cross join dept;
```
解释：emp表有3条记录, dept有5条记录，该查询返回5*3=15条记录

### 内连接

**隐式内连接**
```
select 字段列表 from 表1,表2 where 条件;
```
**显式内连接**
```
select 字段列表 from 表1 [inner] join 表2 on 连接条件;
```
例：查询emp, dept两张表
```
select emp.name,dept.name from emp, dept where emp.dept_id = dept.id;
select emp.name,dept.name from emp e inner join dept d on e.dept_id = d.id;
```
### 外连接

**左外连接**
```
select 字段列表 from 表1 left [outer] join 表2 on 条件 ...;
```
相当于查询左表所有数据，再包含左表和右表交集部分的数据

**右外连接**
```
select 字段列表 from 表1 right [outer] join 表2 on 条件 ...;
```
相当于查询右表所有数据，再包含左表和右表交集部分的数据

例：查询emp表所有数据，以及对应的部门信息(左外连接)
```
select * from emp e left join dept d on emp.dept_id = d.id;
```
例：查询dept表所有数据，加上对应员工信息(右外连接)
```
select * from emp e right join dept d on e.dept_id = d.id;
```

### 自连接
```
select 字段列表 from 表A 别名A join 表A 别名B on 条件...;
```
自连接查询，可以是内连接查询，也可以是外连接查询。

例: 查询每个员工及其领导的名字
```
select e1.name, e2.name from emp e1 inner join emp e2 on e1.managerid = e2.id; 
```
例: 查询所有员工及其领导的名字，如果员工没有领导，也需要查询出来
```
select e1.name, e2.name from emp e1 left join emp e2 on e1.managerid = e2.id;
```

### 联合查询
对于联合查询，就是把多次查询的结果合并起来，形成一个新的查询结果集。语法如下：
```
select 字段列表 from 表A
union [all]
select 字段列表 from 表B;
```

例：查询工资低于5000的员工，和年龄大于50岁的员工
```
select * from emp where salary < 5000
union all
select * from emp where age > 50;
```

注：
* 对于联合查询的多张表列数必须保持一致，字段类型也需要保持一致
* union all会将全部数据合并，union对合并结果去重

### 子查询
SQL语句中嵌套select语句，成为嵌套查询，又称子查询
```
select * from t1 where column1 = (select column1 from t2);
```
根据子查询结果不同，分为:
* 标量子查询(查询结果为单个值)
* 列子查询(查询结果为一列)
* 行子查询(查询结果为一行)
* 表子查询(查询结果为多行多列)

#### 标量子查询
例: 根据部门ID，查询员工信息(标量子查询)
```
select * from emp where id = (select id from dept where name='销售部');
```
例: 查询入职时间在peter之后的所有员工(标量子查询)
```
select * from emp where entrydate > (select entrydate from emp where name = 'peter');
```
#### 列子查询
子查询返回的结果是一列(可以是多行), 这种子查询称为列子查询)
常用操作符: IN, NOT IN, ANY, SOME, ALL
| 操作符 | 描述 |
| -- | -- | 
| IN | 在指定范围内，多选一 |
| NOT IN | 不在指定范围内 |
| ANY | 满足任意一个即可 |
| SOME | 与ANY等价 |
| ALL | 子查询返回列表的所有值都必须满足 |

例: 查询市场部或销售部的员工信息
```
select * from emp where dept_id in (select id from dept where name = '销售部' or name = '市场部'); 
```

例: 查询比财务部所有人工资都高的员工信息
```
select * from emp where salary > all (select salary from emp where dept_id = (select * from dept where name = '财务部'));
```

#### 行子查询
子查询返回结果为一行多列，这种查询称为行子查询

例: 查询与tony的薪资和直属领导都相同的员工信息
```
select * from emp where (salary, managerid) = (select salary, managerid from emp where name = 'Tony'); 
```

#### 表子查询
子查询返回结果为多行多列，这种查询称为表子查询

例: 查询与peter,david的职位和薪资相同的员工信息
```
select * from emp where (job,salary) in (select job, salary from emp where name = 'peter' or name = 'david');
```

## 事务
事务是一组操作集合，不可分割的工作单位。
事务把所有操作作为一个整体向系统提交或撤销操作，这些操作要么同时成功，要么同时失败。

### 事务四大特性
* Atomicity 原子性 所有SQL作为一个整体，要么全执行，要么全不执行
* Consistency 一致性 事务完成后, 所有数据状态是一致的，即A账户减去100元，B账户必定加100元
* Isolation 隔离性 如果多个事务并发执行，每个事务做出修改必须与其他事务隔离
* Durability 持久性 事务完成后, 对数据库修改被持久化存储

示例
```
create table account(
	id int auto_increment primary key,
	name varchar(10),
	money int
);
insert into account(name,money) values('张三',2000),('李四',2000);
update account set money = 2000 where name = '张三' or name = '李四';
```
### 手动提交事务
方法1：设置autocommit
```sql
set @@autocommit = 0;
update account set money = money - 1000 where name = '张三';
update account set money = money + 1000 where name = '李四';
-- 提交事务
commit;
-- 回滚事务 rollback;
```
方法2：手动开启事务
```sql
start transaction; 
update account set money = money - 1000 where name = '张三';
update account set money = money + 1000 where name = '李四';
-- 提交事务
commit;
-- 回滚事务 rollback;
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
SQL标准定义了4种隔离级别,如下:
|  隔离级别   | 脏读(Dirty Read)  | 不可重复读(Non Repeatable Read) | 幻读(Phantom Read) |
|  ----  | ----  |  ----  | ----  |
| Read Uncommitted  | Y | Y | Y |
| Read Committed  | - | Y | Y |
| Repeatable Read  | - | - | Y |
| Serializable  | Y | Y | Y |

查看事务隔离级别
```
mysql> select @@transaction_isolation;
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         |
+-------------------------+
```
设置事务隔离级别
```
set [session|global] transcation isolation level [read uncommitted | read committed | repeatable read | serializable ];
```

#### Read Uncommitted
Read Uncommitted级别下，一个事务会读到另一个事务未提交的数据，如果另一个事务回滚，当前事务读到的就是脏数据，即脏读(Dirty Read)

例: 创建一张学生表, 插入一条学生记录
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
https://liaoxuefeng.com/books/sql/transaction/index.html

#### Repeatable Read

#### Serializable
