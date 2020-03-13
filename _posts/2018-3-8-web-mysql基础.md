---
layout: post
title:
modified:
categories: Tech
tags: [mysql, web]

comments: true
---

<!-- TOC -->

- [数据类型](#数据类型)
- [CREATE 创建表](#create-创建表)
- [SELECT 查询行](#select-查询行)
- [使用函数](#使用函数)
- [汇总或统计](#汇总或统计)
- [子查询](#子查询)
- [join 查询](#join查询)
- [UNION](#union)
- [修改](#修改)
- [高级特性之 约束 CONSTRAINT](#高级特性之-约束-constraint)
- [高级特性之 索引 INDEX](#高级特性之-索引-index)
- [高级特性之 触发器](#高级特性之-触发器)
- [视图 VIEW](#视图-view)
- [高级特性之 存储过程](#高级特性之-存储过程)
- [高级特性之 事务](#高级特性之-事务)
- [高级特性之 游标](#高级特性之-游标)
- [临时表](#临时表)
- [OTAINFO 实例](#otainfo-实例)
- [数据库设计范式](#数据库设计范式)

<!-- /TOC -->

### 数据类型

### CREATE 创建表

```sql
CREATE database test; ---创建数据库
USE test;
CREATE TABLE pet(
        name varchar(20) not null primary key default 'hey',  ---名字
        owner varchar(20),       ---主人
        species varchar(20),     ---种类
        sex char(1),             ---性别
        birth date,              ---出生日期
        death date               ---死亡日期
);
+---------+-------------+------+-----+---------+-------+
| Field   | Type        | Null | Key | Default | Extra |
+---------+-------------+------+-----+---------+-------+
| name    | varchar(20) | YES  | MUL | hey     |       |
| owner   | varchar(20) | YES  |     | NULL    |       |
| species | varchar(20) | YES  |     | NULL    |       |
| birth   | date        | YES  |     | NULL    |       |
| death   | date        | YES  |     | NULL    |       |
| des     | char(100)   | YES  |     | NULL    |       |
+---------+-------------+------+-----+---------+-------+
```

`RENAME` 重命名表

```sql
RENAME TABLE pet to animals;
```

`DESC` 描述表

```sql
DESC animals;
```

`ALTER TABLE` 修改表列结构

```sql
---pet增加一列
ALTER TABLE pet ADD des char(100) null;
---删除列
ALTER TABLE pet DROP COLUMN des;
```

`DELETE` 删除行数据,`DROP` 删除表

```sql
DELETE FROM pet WHERE name="snaky"; --- 删除表项
DROP TABLE pet;  --- 删除数据表pet
```

### SELECT 查询行

SELECT 最终输出的都是符合条件的某行,多行，全部行

```sql
SELECT pes FROM animals;
SELECT * FROM animals;

---查找并按des的降序排序
SELECT * FROM pet ORDER BY des DESC;

---des一致，则按age排序
SELECT * FROM pet ORDER BY des,age DESC;

---条件过滤
SELECT * FROM pet WHERE name = "kitty" ;

---IN 在in指定的范围内
SELECT * FROM OTAINFO WHERE model IN ('zhixuan') ;

---多个条件
SELECT * FROM pet WHERE (name = "kitty" OR name = "doggy") AND (age >= 10) ;

---通配符搜索条件
---'%'匹配任意多个 '_'匹配一个
---匹配以任意1个开头，3 or 4作为第2个字符，0101为后续4个，再匹配任意多字符的字符串
---^表示否定
---通配性能要差些 方便但是要看情况
SELECT * FROM pet WHERE des LIKE "%cry%"; ---类似正则
SELECT * FROM OTAINFO WHERE version_code LIKE '_401%'  ;
SELECT * FROM OTAINFO WHERE version_code LIKE '_[^34]0101%'  ;

---格式化输出字段 like printf
SELECT CONCAT(model ,' hey') FROM OTAINFO;

---可以给新的查询取别名(alias),也叫导出列名
SELECT CONCAT(model ,' hey') AS hey_model FROM OTAINFO;
| hey_model   |
+-------------+
| zhixuan hey |
| 408 hey     |
---也可以给表取别名，后续再引用表时，更简洁
SELECT CONCAT(model ,' hey') AS hey_model FROM OTAINFO as OI

---可以通过计算值的列，得到新的结果列
SELECT price,num,num*price AS total FROM records

---增加一列时间
SELECT CURDATE() AS fetch_time,LIKE 'S%' AS baseline FROM OTAINFO;


```

### 使用函数

上面已经有一些使用函数的例子了。
不同 DBMS 的函数定义会不同
一般有`文本处理`和`计算`,`时间日期处理`
![2018-03-11-00-05-54](https://images-1257933000.cos.ap-chengdu.myqcloud.com/2018-03-11-00-05-54.png)

计算相对用到的少。

时间函数举例:

```sql
---表中选出2018的version_code列，并按时间排序
SELECT version_code FROM OTAINFO WHERE YEAR(last_updated_time)=2018 ORDER BY last_updated_time
```

### 汇总或统计

确定某种组合多少行，多少列什么的

```sql
---统计name出现的次数，次数在COUNT(*)列, GROUP BY 表示以什么为分类统计的依据
--- COUNT(some_col) 统计some_col列不为Null的行数
SELECT model,COUNT(model) AS counts FROM OTAINFO GROUP BY model;

+---------+-----------+
| model   | counts    |
+---------+-----------+
| 408     |         2 |
| zhixuan |        12 |
+---------+-----------+

---数据分组
SELECT model,COUNT(model) AS counts FROM OTAINFO GROUP BY model;

---可以看成是下面的组合，显然更方便
SELECT model,COUNT(model) AS counts FROM OTAINFO WHERE model = '408';
SELECT model,COUNT(model) AS counts FROM OTAINFO WHERE model = 'zhixuan';

---有的时候也不想要GROUP BY后的全部的分组，再用类似WHERE的HAVING 作为过滤条件:
SELECT model,COUNT(model) AS counts FROM OTAINFO GROUP BY model HAVING model='zhixuan';

---分页查询,查询第10000个记录后开始的10个,即10000-10009
SELECT sn from OTAINFO where sn>100 limit 10000,10;
```

### 子查询

```sql
SELECT nums FROM TABA WHERE num IN (
        SELECT ages FROM TABB WHERE date IN (
                SELECT * FROM ...
        )
)
```

### join 查询

`join`意味着至少一列同时出现在多个表中.

- 内部联结

FROM 后面为需要 join 的表，WHERE 为 join 的条件,只匹配那些满足 join 条件的行。

```sql
SELECT vendor,price,desc FROM PRODOCT,VENDERS WHERE PRODOCT.vid = VENDERS.vid
```

换一个语法写法:`SELECT ... FROM A inter join B on condition`

```sql
SELECT vendor,price,desc FROM PRODOCT INTER JOIN VENDERS on PRODOCT.vid = VENDERS.vid
```

- 自联结

如果是正常的先要查出 goodman 的 price，再根据 price 查大于它的 price.自联结显然要好理解一下,a,b 的别名来自同一个表。

```sql
SELECT b.*
FROM shopping as a,shopping as b
WHERE a.name='goodman'
and a.price<b.price
order by b.id
```

- 左(右)外部联结

有时候需要包含没有关联行的行:列出所有产品即订购数量，包括没人订购的产品.

关键字`left/right outer join`,`left` or `right` 指定 OUTER JOIN 哪边的表将指定所有行.

```sql
SELECT PRODUCT.vendor,PRODUCT.price,VENDORS.desc FROM PRODOCT LEFT OUTER JOIN VENDERS on PRODOCT.vid = VENDERS.vid
```

另外一种简单的写法用`*=`表示`left`或者`=*`表示`right`

```sql
SELECT PRODUCT.vendor,PRODUCT.price,VENDORS.desc FROM PRODOCT,VENDERS WHERE PRODOCT.vid *= VENDERS.vid
```

另外一种简单的写法用`*=`表示`left`或者`=*`表示`right`

### UNION

两次查询的并集,会去掉重复的行，也要求查询的列必须相同
`UNION ALL`可保留所有行

```sql
--- 两个表的name打印出来,去重复
SELECT name FROM pet UNION SELECT name FROM master;
--- 两个表相同的name才打印出来，
SELECT a.name FROM pet a JOIN master b on a.name=b.name; ---交集 用JOIN
```

### 修改

`INSERT` 添加行数据.只添加部分行时，省略的列必须为 NULL 或者具有默认值

```sql
INSERT into pet(name,owner,species,birth,death,des)
values
("kitty","yang","good cat",'2017-01-01','2017-02-01',"lovely baby");
```

`INSERT SELECT`添加的数据来自 select 的检索,表合并什么的

`SELECT INTO`创建新表，将旧表导入，有点复制表的意思

```sql
---mysql的写法
CREATE TABLE new SELECT *  FROM old
```

`UPDATE` 更新行的某些数据

```sql
UPDATE pet SET owner="yangyucheng" WHERE name="doggy";
--- 清除列值 如果允许NULL
UPDATE pet SET owner=NULL WHERE name="doggy";
```

### 高级特性之 约束 CONSTRAINT

`约束`是管理如何插入或处理数据库数据的规则

数据表上施加`约束`来保证`引用完整性`.

- 主键 PRIMARY KEY

保证主键所在列的行的值都是唯一的.可以安全的`update`,`delete`,`select`

列本身不更新，不修改

主键必须`NOT NULL`

在表中定义:

```sql
CREATE TABLE test (
        main_name varchar(20) NOT NULL PRIMARY KEY DEFAULT 'default',
        addr varchar(20) NULL

);
```

已有表添加:

```sql
ALTER TABLE test ADD CONSTRAINT PRIMARY KEY (main_name);
```

- 外键 FOREIGN KEY
  外键也是表中的一列，其值由另一个表的主键给出,即`绑定某列到其他表的主键`

```sql
CREATE TABLE Customers (
        cust_ids char(10) NOT NULL PRIMARY KEY DEFAULT 'def',
        names varchar(30) NOT NULL DEFAULT 'def'
);
CREATE TABLE Orders (
        goods_ids varchar(30) NOT NULL PRIMARY KEY DEFAULT 'def',
        cust_ids   char(10) NOT NULL REFERENCES Customers(cust_ids)
);
CREATE TABLE Orders (
        goods_ids varchar(30) NOT NULL PRIMARY KEY DEFAULT 'def',
        cust_ids   char(10) NOT NULL DEFAULT 'def'
);
```

同样也可以直接用语句:

```sql
ALTER TABLE Orders ADD CONSTRAINT FOREIGN KEY(cust_ids) REFERENCES
Customers (cust_ids);
```

得到表:

```
desc Orders;
+-----------+-------------+------+-----+---------+-------+
| Field     | Type        | Null | Key | Default | Extra |
+-----------+-------------+------+-----+---------+-------+
| goods_ids | varchar(30) | NO   | PRI | def     |       |
| cust_ids  | char(10)    | NO   | MUL | def     |       |
+-----------+-------------+------+-----+---------+-------+
desc Customers;
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| cust_ids | char(10)    | NO   | PRI | def     |       |
| names    | varchar(30) | NO   |     | def     |       |
+----------+-------------+------+-----+---------+-------+
```

对于设置了`外键`的表(子表)，不能随意增删改表内容，要依据外键引用的表(父表)的主键内容。如果外键对应的主键列里存在某个值，才允许添加，也不能删除。总之是为了保证`引用完整性`.

- 唯一约束 UNIQUE
  类似主键，每个数据唯一，但是:

表中可以有多个 UNIQUE 列,可以包含 NULL 值;

该列可以被更新;

UNIQUE 列不可以定位为外键;

- 检查约束 CHECK
  用来保证指定列满足指定的条件

```sql
ALTER TABLE Orders ADD COLUMN price int CHECK (price > 10);
```

### 高级特性之 索引 INDEX

`索引`用来加快搜索和排序的速度,就是保存了内容已经排好序的列表.索引要占据存储空间，而且更新时，因为索引也要同步更新，所以大量的更新时，不太适合索引。

可以建索引的一般是 where、order by 或者 group by 后面的字段

`索引`完放那就可以了，查找时如果有索引，自然的会加快

添加了`主键`和`UNIQUE`的列默认就是有索引的。

```sql
---CREATE建立单列name的索引
CREATE INDEX idx_name ON pet(name);
---ALTER添加索引
ALTER TABLE pet ADD INDEX idx_name(name);
---显示索引
SHOW INDEX FROM pet; \G
---删除索引
DROP INDEX index_name ON talbe_name;
ALTER TABLE table_name DROP INDEX index_name;
ALTER TABLE table_name DROP PRIMARY KEY;
```

- 单列索引，联合索引

[多个索引在什么时候用好?](https://www.zhihu.com/question/40736083)

### 高级特性之 触发器

一张表发生了某件事（插入、删除、更新操作），然后自动触发了预先编写好的若干条 SQL 语句的执行.

某些修改后，可以自动完成其他的动作:

```sql
DROP TRIGGER IF EXISTS `tri_insert_user`;
---修改分号，保证语句中的;不自动识别为语句的结束，从而可以写多个语句
DELIMITER ;;
CREATE TRIGGER `tri_insert_user` AFTER INSERT ON `user` FOR EACH ROW begin
    INSERT INTO user_history(user_id, operatetype, operatetime) VALUES (new.id, 'add a user',  now());
end
;;
---恢复分号
DELIMITER ;
```

可能的应用场景:

1. 如上例，插入某个 usr 后，自动生成插入历史记录到 user_history table
2. update or insert 后，自动更新数据为大写格式;
3. 结合事务，update 后如果超出范围，则在触发器
4. 计算更新某些时间戳

### 视图 VIEW

视图将查询打包,可以通过视图隐藏复杂的查询。有点编程中的`引用`的味道.

视图是`虚拟表`，本身不存储数据，而是按照指定的方式进行查询.

像使用`普通表`一样使用视图即可

可以一次性编写基础的`SQL`,然后作为视图供多次使用

```sql
CREATE VIEW ProView AS SELECT a,b,c from A join B join C IN (A.i = B.i)
SELECT * FROM ProView WHERE A.id = 'heyman'
```

### 高级特性之 存储过程

### 高级特性之 事务

[ref](http://www.runoob.com/mysql/mysql-transaction.html)

管理成批的 SQL 修改任务，`insert,update,delete`

保证`原子性`，`一致性`，`隔离性`,`持久性`

事务的隔离级别:

- READ UNCOMMITTED 读未提交
- READ COMMITTED 读提交
- REPEATABLE READ 可重复读
- SERIALIZABLE 串行化

操作:

```sql
START TRANSACTION;
BEGIN ;同上
SET AUTOCOMMIT=0 ---禁止自动提交
COMMIT; ---提交
ROLLBACK; ---回滚
ROLLBACK TRANSACTION identifier; ---回滚到保存点
SAVEPOINT identifier； ---设置保存点
RELEASE SAVEPOINT identifier；---删除存储点
SET TRANSACTION lvl; ---设置事务隔离级别
```

### 高级特性之 游标

有时候需要再已查询的结果行前进或者回退若干行。

`游标`是存储在 db server 上的一个查询结果,存储了游标后，应用程序可以根据需要滚动浏览其中的数据。

`游标`提供了基于游标位置的增删改查能力.

`游标`的可设置特性包括:

`游标`主要用于交互式的数据应用，如滚动查看数据。

1. 只读游标，不能 update 和 delete;
2. 定向控制,next backward,first ,last ,abs pos relative pos 等等
3. 标记只编辑某些列
4. 规定范围，在存储过程中有效，或全局有效
5. 指示对检索数据做复制，在游标访问期间，数据不变化

游标+存储过程的例子:

```sql
drop procedure if exists cursor_test;
delimiter //
create procedure cursor_test()
begin
    -- 声明与列的类型相同的四个变量
    declare id varchar(20);
    declare pname varchar(20);
    declare pprice varchar(20);
    declare pdescription varchar(20);

-- 1、定义一个游标mycursor
    declare mycursor cursor for
  select *from shops_info;
-- 2、打开游标
    open mycursor;
-- 3、使用游标获取列的值
    fetch  next from mycursor into id,pname,pprice,pdescription;
-- 4、显示结果
    select id,pname,pprice,pdescription;
-- 5、关闭游标
    close mycursor;
end;
//
delimiter ;
call cursor_test();
```

### 临时表

临时表只在当前连接可见，当关闭连接时，Mysql 会自动删除表并释放所有空间.
有什么用?

[何时使用临时表](https://www.zhihu.com/question/21675233)

```sql
CREATE TEMPORARY TABLE SalesSummary(
        varchar(20) name NOT NULL  PRIMARY KEY
)
```

### OTAINFO 实例

```sql
---允许远程登录
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '!@Le201801' WITH GRANT OPTION;


---OTAINFO
CREATE TABLE OTAINFO(
        sn char(30) not null primary key,
        model char(30) not null ,
        version_code  char(50) not null ,
        last_updated_time datetime not null ,
        authorized int(1)
);
```

### 数据库设计范式

<https://segmentfault.com/a/1190000013695030?utm_source=tag-newest>
