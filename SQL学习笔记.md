---
title: SQL学习笔记
date: 2019-07-24 23:08:09
description: 数据库作为搭建服务器的重要环节可谓是至关重要. 这里记录了典型关系型数据库管理系统SQL的学习过程,主要以MySQL为例. 方便回来查找以及与大家交流分享.
tags:
    -数据库
categories:
    -服务器相关
mathjax:
    true
---

# Linux下安装配置MySQL

## 查看系统是否自带MySQL

```
rpm -qa | grep mysql
```

## 卸载MySQL

```
sudo apt-get remove mysql-server
sudo apt-get autoremove
sudo apt-get remove package-name
dpkg -l | grep mysql | grep ii //查看是否删除.
```

## 安装MySQL

先去[官网](https://www.mysql.com/downloads/)下载适合自己版本的安装包到安装目录.当然官网会提供安装方式的,不过是全英文的,也不是很难. 这里做一个简单的介绍. (以Ubuntu为例). 对于任何问题, 官网往往是我们第一个应该想到的解决方式. 毕竟官网是规则的制定者, 当然考虑到很多官网是全英文, 对于像我这样的菜鸟会将官网优先级降低, 不过如果查看不到解决方案则最终也应该考虑官网.

解压下载文件:

```terminal
shell> sudo dpkg -i /PATH/version-specific-package-name.deb
```

更新本地环境:

```
shell> sudo apt-get update
```

安装:

```
shell> sudo apt-get install mysql-server
```

在安装过程中, 会要求输入一个密码, 作为root用户使用MySQL的密码.



查看MySQL状态:

```
shell> sudo service mysql status
```

停止MySQL服务器:

```
shell> sudo service mysql stop
```

启动MySQL服务器:

```
shell> sudo service mysql start
```

打开MySQL(确保MySQL服务器启动中)

```
mysql -u root -p
```

# SQL常用指令

## 展示并选择数据库

打开MySQL后可以通过命令`show databases; `展示存在的数据库.

选择要使用的数据库`use dataname;` 选择使用的数据库.

展示数组库中的表: `show tables`

展示表头`show columns from tablename`

因此MySQL结构为: 一个MySQL下有多个数据库. 一个数据库下可以有多个表, 表里面用来存储数据.

创建一个新的数据库:`create databases dataname;`

设置数据集使用用户名及密码.`create user 'username'@'localhost' identified by 'password'; `

 赋予用户使用数据集的相应权限:`GRANT ALL ON dataname.* TO 'username'@'localhost';`

## MySQL数据类型:

在 MySQL 中，有三种主要的类型：Text（文本）、Number（数字）和 Date/Time（日期/时间）类型。

### Text 类型：

| 数据类型         | 描述                                                         |
| :--------------- | :----------------------------------------------------------- |
| CHAR(size)       | 保存固定长度的字符串（可包含字母、数字以及特殊字符）。在括号中指定字符串的长度。最多 255 个字符。 |
| VARCHAR(size)    | 保存可变长度的字符串（可包含字母、数字以及特殊字符）。在括号中指定字符串的最大长度。最多 255 个字符。**注释：**如果值的长度大于 255，则被转换为 TEXT 类型。 |
| TINYTEXT         | 存放最大长度为 255 个字符的字符串。                          |
| TEXT             | 存放最大长度为 65,535 个字符的字符串。                       |
| BLOB             | 用于 BLOBs（Binary Large OBjects）。存放最多 65,535 字节的数据。 |
| MEDIUMTEXT       | 存放最大长度为 16,777,215 个字符的字符串。                   |
| MEDIUMBLOB       | 用于 BLOBs（Binary Large OBjects）。存放最多 16,777,215 字节的数据。 |
| LONGTEXT         | 存放最大长度为 4,294,967,295 个字符的字符串。                |
| LONGBLOB         | 用于 BLOBs (Binary Large OBjects)。存放最多 4,294,967,295 字节的数据。 |
| ENUM(x,y,z,etc.) | 允许您输入可能值的列表。可以在 ENUM 列表中列出最大 65535 个值。如果列表中不存在插入的值，则插入空值。**注释：**这些值是按照您输入的顺序排序的。可以按照此格式输入可能的值： ENUM('X','Y','Z') |
| SET              | 与 ENUM 类似，不同的是，SET 最多只能包含 64 个列表项且 SET 可存储一个以上的选择。 |

### Number 类型：

| 数据类型        | 描述                                                         |
| :-------------- | :----------------------------------------------------------- |
| TINYINT(size)   | 带符号-128到127 ，无符号0到255。                             |
| SMALLINT(size)  | 带符号范围-32768到32767，无符号0到65535, size 默认为 6。     |
| MEDIUMINT(size) | 带符号范围-8388608到8388607，无符号的范围是0到16777215。 size 默认为9 |
| INT(size)       | 带符号范围-2147483648到2147483647，无符号的范围是0到4294967295。 size 默认为 11 |
| BIGINT(size)    | 带符号的范围是-9223372036854775808到9223372036854775807，无符号的范围是0到18446744073709551615。size 默认为 20 |
| FLOAT(size,d)   | 带有浮动小数点的小数字。在 size 参数中规定显示最大位数。在 d 参数中规定小数点右侧的最大位数。 |
| DOUBLE(size,d)  | 带有浮动小数点的大数字。在 size 参数中规显示定最大位数。在 d 参数中规定小数点右侧的最大位数。 |
| DECIMAL(size,d) | 作为字符串存储的 DOUBLE 类型，允许固定的小数点。在 size 参数中规定显示最大位数。在 d 参数中规定小数点右侧的最大位数。 |

> **注意：**以上的 size 代表的并不是存储在数据库中的具体的长度，如 int(4) 并不是只能存储4个长度的数字。
>
> 实际上int(size)所占多少存储空间并无任何关系。int(3)、int(4)、int(8) 在磁盘上都是占用 4 btyes 的存储空间。就是在显示给用户的方式有点不同外，int(M) 跟 int 数据类型是相同的。
>
> 例如：
>
> 1、int的值为10 （指定zerofill）
>
> ```
> int（9）显示结果为000000010
> int（3）显示结果为010
> ```
>
> 就是显示的长度不一样而已 都是占用四个字节的空间

### Date 类型：

| 数据类型    | 描述                                                         |
| :---------- | :----------------------------------------------------------- |
| DATE()      | 日期。格式：YYYY-MM-DD**注释：**支持的范围是从 '1000-01-01' 到 '9999-12-31' |
| DATETIME()  | *日期和时间的组合。格式：YYYY-MM-DD HH:MM:SS**注释：**支持的范围是从 '1000-01-01 00:00:00' 到 '9999-12-31 23:59:59' |
| TIMESTAMP() | *时间戳。TIMESTAMP 值使用 Unix 纪元('1970-01-01 00:00:00' UTC) 至今的秒数来存储。格式：YYYY-MM-DD HH:MM:SS**注释：**支持的范围是从 '1970-01-01 00:00:01' UTC 到 '2038-01-09 03:14:07' UTC |
| TIME()      | 时间。格式：HH:MM:SS**注释：**支持的范围是从 '-838:59:59' 到 '838:59:59' |
| YEAR()      | 2 位或 4 位格式的年。**注释：**4 位格式所允许的值：1901 到 2155。2 位格式所允许的值：70 到 69，表示从 1970 到 2069。 |

*即便 DATETIME 和 TIMESTAMP 返回相同的格式，它们的工作方式很不同。在 INSERT 或 UPDATE 查询中，TIMESTAMP 自动把自身设置为当前的日期和时间。TIMESTAMP 也接受不同的格式，比如 YYYYMMDDHHMMSS、YYMMDDHHMMSS、YYYYMMDD 或 YYMMDD。

## create table语句

创建数据库中的表

语法:

```
create table table_name (column_name1 data_type(size), column_name2 data_type(size), ...);//创建时就要指明表中数据属性及对于类型.

ex:
create table students (ID varchar(255), name varchar(255), score int, classify int);
注意: 如果使用默认值不自己指定长度则不用括号.
```



## select语句

select从数据库中选取数据. 结果被存储在一个表中, 被称为结果表.

对于每个表格, 第一行是属性名称, 叫column_name

语法:

```C++
select column_name, column_name from table_name;//读取表中某几列元素

select * from table_name; //读取整张表

select distinct column_name, column_name from table_name; //读取表格中某列的元素, 并且只返回唯一不同的值, 有点类似np.unique()函数.
```

## where语句

where用于过滤提取那些满足要求的记录

语法:

```mysql
select column_name, column_name from table_name where column_name operator values; //where相当于一个限制条件

ex:
select * from db where Select_priv='Y';

where限制条件: 当是字符项时要使用引号(区分大小写), 当是数字是不能加引号.
= -等于
<> -不等于
>, >=, <, <=;
between -在某个范围内
like -搜索某种模式
in -指定针对某个列的多可能值
```

对于where的限制条件可以组合

在各个条件中可以加入and和or和not, 其中优先级顺序
() not and or

特殊条件:

1. is null 非空条件 : select * from emp where comm is null;
2. between and 在..之间: select * from emp where sal between 1500 and 3000;
3. in 在所列的元素中: select * from emp where sal in (500, 3000, 15000);
4. like 模糊查询: select * from emp where ename like 'M%'; 其中like后为一个正则表达式. '%'表示多个字符的占位符, '_'表示一个字符的占位符. 例如
   1. 'M%': 表示以'M'为开头的字符串.
   2. '%M%': 表示包含'M'的字符串.
   3. '%M_': 表示'M'在倒数第二个位置的字符串.

## order by语句

order by关键字对结果安装一列或多列进行排序. 默认使用升序, 要使用降序时只需要在最后加上desc

语法:

```mysql
select column_name, column_name from tables_name order by column_name, column_name asc|desc;

对于order by后面有多个列时先按照第一个列进行排序, 如果相等, 使用第二列, 以此类推. 同时可以定义每一列的升序降序, 在每一列的后面标注asc|desc;

select * from db where user like 'm%' order by Select_priv desc;
```

## insert into语句

insert into向表中插入新的记录.

语法:

```mysql
insert into table_name values (values1, value2, value3, ...) //向名为table_name的表中添加元素, 值为括号中参数, 由于没有指明元素属性(属于哪一列), 所以要求所以列都要有参数.

insert into table_name (column1, column2, column3, ...) values (values1, value2, value3, ...) //向名为table_name的表中添加元素, 添加的元素与值一一对应, 此时不要求每个属性都要有.
```

## update语句

update用于更新已经存在的记录.

语法:

```mysql
update table_name set column_name1 = value1, coulumn_name2 = value2, ... where some_column = some_value; // 更新存在数据, 注意where语句, 如果没有where语句则整个表格均会被更新.
```

## delete语句

delete语句用于删除某个记录.

语法:

```mysql
delete from table_name where some_column = some_value;
注意, 如果没有where限制,整个表均会被删除(赶快跑路了).
```

## alter table

alter table 用于向表格中插入, 更改, 删除列.

### 添加列

```mysql
alter table table_name add column_name datatype;
ex:
alter table students add dateofbrith date;
```

### 删除列

```mysql
alter table table_name drop column column_name;
ex:
alter table students drop column classify;
```

### 更改列(数据类型)

```mysql
alter table table_name modify column column_name datatype default value;
ex:
alter table students modify column dataofbirth date not null default ’2019-08-13‘;
```



## select top

用于返回按照一点规则排序后的前一定数量个.

mysql语法:

```mysql
select column(s) from table_name limit number;

ex:
select * from students order by score desc limit 1; //找分数最低的学生信息
```

## 通配符

通配符一般与like联合使用, 用于搜索表中数据.

"%" : 代替0个或多个字符.

"_": 代替一个字符.

"[charlist]" : 字符列表中任何单一字符,

"\[^charlist]" 或者 "[!cgarlist]": 不在字符列表中的单一字符.

对于字符列表,除非有位置限制,  否则匹配方式是只要字符串中(不)存在charlist的字符就算匹配成功.

MySQL不支持charlist方式, 会将[]当做字符的一部分, 因此使用charlist要用正则表达式.

### MySQL正则表达式:

<table>
    <tr>
        <th>模式</th>
        <th>描述</th>
    </tr>
    <tr>
        <th>^</th>
        <th>匹配输入字符串的开始位置(匹配第一个字符)。如果设置了 RegExp 对象的 Multiline 属性，^ 也匹配 '\n' 或 '\r' 之后的位置。</th>
    </tr>
    <tr>
        <th>$</th>
        <th>匹配输入字符串的结束位置(最后位置)。如果设置了RegExp 对象的 Multiline 属性，$ 也匹配 '\n' 或 '\r' 之前的位置。</th>
    </tr>
    <tr>
        <th>[...]</th>
        <th>字符集合。匹配所包含的任意一个字符。例如， '[abc]' 可以匹配 "plain" 中的 'a'。</th>
    </tr>
    <tr>
        <th>[^...]</th>
        <th>负值字符集合。匹配未包含的任意字符。例如， '[^abc]' 可以匹配 "plain" 中的'p'。</th>
    </tr>
    <tr>
        <th>p1|p2|p3</th>
        <th>匹配 p1 或 p2 或 p3。例如，'z|food' 能匹配 "z" 或 "food"。'(z|f)ood' 则匹配 "zood" 或 "food"。</th>
    </tr>
    <tr>
        <th>*</th>
        <th>匹配前面的子表达式零次或多次。例如，zo* 能匹配 "z" 以及 "zoo"。* 等价于{0,}。</th>
    </tr>
    <tr>
        <th>+</th>
        <th>匹配前面的子表达式一次或多次。例如，'zo+' 能匹配 "zo" 以及 "zoo"，但不能匹配 "z"。+ 等价于 {1,}。</th>
    </tr>
    <tr>
        <th>{n}</th>
        <th>n 是一个非负整数。匹配确定的 n 次。例如，'o{2}' 不能匹配 "Bob" 中的 'o'，但是能匹配 "food" 中的两个 o。</th>
    </tr>
    <tr>
        <th>{n,m}</th>
        <th>m 和 n 均为非负整数，其中n小于等于m.最少匹配n次且最多匹配m次。</th>
    </tr>
</table>

```
select * from students where ID rlike '^[1234]';
```

## in

属于

```mysql
select * from table where colunm_name in {v1, v2, v3 , ...}
```

## between

between a and b. 选取介于两个值之间的数据范围内的值。

```mysql
select * from table_name where column_name between v1 and v2;
```

其中v1, v2可以是整数或时间.

ex:

```mysql
select * from table where data between '2016-10-1' and '2017-10-1';
```

## 别名

列的别名:

```mysql
select column_name as allas_name from table_name;
ex:
select ID as indetify, score as class from students;
```



表的别名:

```mysql
select column_name(s) from table_name as allas_name;
ex:
select w.ID w.score from students as w;
==
select ID, score from students;
```

## join

join是把两个表或者多个表按照行结合起来.

主要使用方式有如下几种:

![join](https://www.runoob.com/wp-content/uploads/2019/01/sql-join.png)

上图中A, B分别表示一个表(将两个表的`where`限制部分的列抽出). 相交部分表示满足`where`限制的交集部分. 而在最终展示时,是从图中满足条件部分所属的行中选取出要展示的元素. 具体实现可以理解为: 对A, B分别建立一个`class`, 其中包含`where`后面的限制元素以及对于的行索引. 对两个列表的每一行实例化这样一个类. 之后按照选取规则筛选出符合条件的A,B中实例, 而后按照实例中的索引和要求展示的元素对满足规则的行进行两个表格的合并,输出即可. 因此应该尽量要求两个表无相同的表头.(合并冲突)



## UNION操作符

`UNION`操作符合并两个或者多个`select`语句结果. 要求`selec`t语句拥有相同数量的列, 列也必须拥有相同的数据类型. 每个`select`语句列的顺序必须相同.

语法:

```
SELECT column_name(s) FROM table1
UNION [all | distinct]
SELECT column_name(s) FROM table2;
```

`[all | distinct]`表示可选的。union默认只会列出不同的元素（distinct）, 要想列出所以元素则应该使用`union all`.

可以理解为`union`为取两个或多个`select`语句组成的集合的`unique`, 而`union all`为整个集合.



## SELECT INTO

复制一个表的信息到另一个表中. 

语法:

```
SELECT column_name(s)
INTO newtable [IN externaldb]
FROM table1;
```

该命令表示可以使用外部的数据库中的表.

MySQL不支持`select into`语法

## INSERT INTO SELECT

与上述语句类似, 复制一个表的数据到另一个表.

语法

```
INSERT INTO table2
(column_name(s))
SELECT column_name(s)
FROM table1;
```

与`select into`区别为, `insert into select`要求插入的元素在被插入的表中原本就是存在的. 即插入部分在被插入的表中存在对于结构(因此上述命令有两个column_name(s)). 而`select into`则是在被插入的表中不存在对应结构, 命令会自动创建对应结构. 需要注意, `insert into select`在在当前表后面追加而不是将选取的部分合并到指定的列.

## SQL约束

`SQL`约束用于规范表中数据规则. 如果存在违反数据规则的行为, 该行为将会被终止. 约束可以在建表时规定(通过`create table`), 或者在建表之后规定(通过`alter table`语句).

约束类型:

1. `NOT NULL`指示某列不能为空. 与NULL比较不能用关系运算符, 只能用 `is/is not`
2. `unique`:指示某列元素要相互不同.
3. `primary key`: `not null`与`unique`的结合. 确保某列(或多列)有唯一标识, 有助于更快更容易找到表中一个特定值.
4. `foreign key`: 保证一个表中的数据匹配另一个表中的值的参照完整性.
5. `check`: 保证列中的值符合指定条件.
6. `default`: 规定没有给列赋值时的默认值.

建表时约束:

```
CREATE TABLE Persons
(
    Id_P int NOT NULL,
    LastName varchar(255) NOT NULL,
    FirstName varchar(255),
    Address varchar(255),
    City varchar(255),
    PRIMARY KEY (Id_P)  //PRIMARY KEY约束
)
```

`alter add`时约束

```
alter table tablename add column columnname type default(0)
```

`alter modify`更改约束:

```
alter table x modify column_name null;
alter table x modify column_name not null;
```



## UNIQUE约束

`UNIQUE` 和 `PRIMARY KEY` 约束均为列或列集合提供了唯一性的保证。`PRIMARY KEY` 约束拥有自动定义的 `UNIQUE` 约束。每个表可以有多个 `UNIQUE` 约束，但是每个表只能有一个` PRIMARY KEY` 约束。注意: 如果对表中已存在的结构添加约束, 如果当前表内容不符合约束则无法添加.

### 创建表时添加`unique`约束:

```
CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
UNIQUE (P_Id)
)
```

### 定义多个列的约束并命名`unique`约束:

```
CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
CONSTRAINT uc_PersonID UNIQUE (P_Id,LastName)
)
```

注意: 该处表示(P_Id,LastName)构成的结果整体唯一. 只有一个是相同的是被允许的.

### `alter table`时的`unique`约束

添加约束:

```
ALTER TABLE Persons
ADD UNIQUE (P_Id)
```

添加多个列并命名约束:

```
ALTER TABLE Persons
ADD CONSTRAINT uc_PersonID UNIQUE (P_Id,LastName)
```

### 撤销`unique`约束

MySQL:

```
ALTER TABLE Persons
DROP INDEX uc_PersonID
```

SQL Server / Oracle / MS Access：

```
ALTER TABLE Persons
DROP CONSTRAINT uc_PersonID
```

## `PRIMARY KEY`约束

`PRIMARY KEY` 约束唯一标识数据库表中的每条记录。主键必须包含唯一的值。主键列不能包含 NULL 值。每个表都应该有一个主键，并且每个表只能有一个主键。

### `CREATE TABLE`时的`PRIMARY KEY`约束:

```
CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
CONSTRAINT pk_PersonID PRIMARY KEY (P_Id,LastName)
)
```

该出与上述`unique`一致, 表示两个构成的集合.

### `ALTER TABLE`时的`PRIMARY KEY`约束

添加约束:

```
ALTER TABLE Persons
ADD CONSTRAINT pk_PersonID PRIMARY KEY (P_Id,LastName)
```

撤销约束:

```
ALTER TABLE Persons
DROP PRIMARY KEY
```

## `FOREIGN KEY`约束

一个表中的 `FOREIGN KEY` 指向另一个表中的 `UNIQUE KEY`(唯一约束的键)。即在该表中, `foreign key`约束始终与另一个表中的`unique key`约束对应. `FOREIGN KEY` 约束用于预防破坏表之间连接的行为。也能防止非法数据插入外键列，因为它必须是它指向的那个表中的值之一。

### `CREATE TABLE`时的`FOREIGN KEY`

```
CREATE TABLE Orders
(
O_Id int NOT NULL,
OrderNo int NOT NULL,
P_Id int,
PRIMARY KEY (O_Id),
CONSTRAINT fk_PerOrders FOREIGN KEY (P_Id)
REFERENCES Persons(P_Id)
)
```

### `alter table`时的`FOREIGN KEY`

建立外链连接:

```
ALTER TABLE Orders
ADD CONSTRAINT fk_PerOrders
FOREIGN KEY (P_Id)
REFERENCES Persons(P_Id)
```

撤销外链连接:

```
ALTER TABLE Orders
DROP FOREIGN KEY fk_PerOrders
```

## `CHECK`约束

`check`约束用来限制列中值的范围. 当对一个列设置限制时, 那么该列只允许特定的值(或范围). 如果对一个表定义 `CHECK` 约束，那么此约束会基于行中其他列的值在特定的列中对值进行限制。(如列a要大于列b)

### `CREATETABLE`时`CHECK`约束

```
CREATE TABLE Persons
(
P_Id int NOT NULL,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
CONSTRAINT chk_Person CHECK (P_Id>0 AND City='Sandnes')
)
```

### `alter table`时约束

增加约束:

```
ALTER TABLE Persons
ADD CONSTRAINT chk_Person CHECK (P_Id>0 AND City='Sandnes')
```

撤销约束

```
ALTER TABLE Persons
DROP CHECK chk_Person
```

注意: 在MySQL数据库中, 只是check，但是不强制check(和没有有啥区别吗?). 因此要想实现限制有两种方式.

1. 如果范围较小,可枚举,则可将列类型设置为enum()或set().
2. 创建约束器.(见下文)

## `AUTO_INCREMENT`字段

有时我们希望在每次插入新记录时可以自动的创建主键字段的值. 此时我们可以在表中创建一个`auto_increment`字段. 注意: `auto_increment`所在列必须为主键, 即存在`primary key`约束.

`create table`时方式:

```
CREATE TABLE Persons
(
ID int NOT NULL AUTO_INCREMENT,
LastName varchar(255) NOT NULL,
FirstName varchar(255),
Address varchar(255),
City varchar(255),
PRIMARY KEY (ID)
)
```

`alter table`时:

```
ALTER TABLE table_name CHANGE column_name column_name data_type(size) constraint_name AUTO_INCREMENT;

ex:
ALTER TABLE student CHANGE id id INT(11) NOT NULL AUTO_INCREMENT;
```

此时默认的, `AUTO_INCREMENT`起始值为1, 每条记录递增1. 此时再插入数据时可以不用为ID赋值, 会自动添加一个唯一值.

## `SQL default`约束

`create ablet`时设置default约束:

```
CREATE TABLE Persons
(
    P_Id int NOT NULL,
    LastName varchar(255) NOT NULL,
    FirstName varchar(255),
    Address varchar(255),
    City varchar(255) DEFAULT 'Sandnes'
)
```

`alter table`时设置:

```
结构已存在时:
ALTER TABLE Persons
ALTER City SET DEFAULT 'SANDNES'

结构不存在时:
alter table table_name
add column column_name default value comment "info";
```

撤销默认值:

```
ALTER TABLE Persons
ALTER City DROP DEFAULT
```

## `CREATE INDEX`语句

CREATE INDEX 语句用于在表中创建索引。

创建简单索引语法:

```
CREATE INDEX index_name
ON table_name (column_name)
```

创建唯一索引语法:唯一的索引意味着两个行不能拥有相同的索引值.

```
CREATE UNIQUE INDEX index_name
ON table_name (column_name)
```

## `DROP`语句

使用 DROP 语句，可以轻松地删除索引、表和数据库。

删除索引:`ALTER TABLE table_name DROP INDEX index_name`

删除表: `DROP TABLE table_name`

删除数据库: `DROP DATABASE database_name`

## SQL 视图

视图是可视化的表. 视图是基于SQL语句的结果集的可视化的表. 视图包含行和列, 如同一个真实的表. 视图中的字段来自一个或者多个数据库中真实的表中的字段. 可以向视图中添加函数, where以及json语句, 也可以输出视图数据.

### SQL CREATEVIEW语句

```
CREATE VIEW view_name AS
SELECT column_name(s)
FROM table_name
WHERE condition
```

**注释：**视图总是显示最新的数据！每当用户查询视图时，数据库引擎通过使用视图的 SQL 语句重建数据。

例:

```
create view moreavgage_user_id_user_name as
select id, name
from user
where age>(select avg(age) from user) 
```

### 更新视图

```
CREATE OR REPLACE VIEW view_name AS
SELECT column_name(s)
FROM table_name
WHERE condition
```

### 撤销视图

```
DROP VIEW view_name
```

## NULL

NULL 值代表遗漏的未知数据。默认地，表的列可以存放 NULL 值。NULL 值的处理方式与其他值不同。NULL 用作未知的或不适用的值的占位符。无法使用比较运算符来测试 NULL 值，比如 =、< 或 <>。我们必须使用 IS NULL 和 IS NOT NULL 操作符。

## IFNULL函数

该函数用于处理数据为空的情况, 例如:

```
SELECT ProductName,UnitPrice*(UnitsInStock+IFNULL(UnitsOnOrder,0))
FROM Products
```

此时,当UnitsOnOrder为空, 我们就用0代替.

# SQL函数

|   函数   |                       作用                       |                             语法                             |
| :------: | :----------------------------------------------: | :----------------------------------------------------------: |
|  AVG()   |                返回数值例的评价值                |          `select avg(column_name) from table_name`           |
| COUNT()  |              返回匹配指定条件的行数              | `select count(column_name) from table_name`(返回指定列的数目,null不计).  `select count(*) from table_name`(返回表中记录数目).  `select count(distinct column_name) from table_name`(返回指定列不同值的数目). |
|  MAX()   |                 返回指定列最大值                 |          `select max(column_name) from table_name`           |
|  MIN()   |                 返回指定列最大值                 |          `select min(column_name) from table_name`           |
|  SUM()   |                  返回指定列之和                  |             `select sum(column_name) from table`             |
| GROUP BY | 句用于结合聚合函数,根据一个或多个列结果进行分组. | `select column_name, aggregate_function(column_name) from table_name where column_name operator valus group bu column_name` |
| UCASE()  |              把字段的大写转换成小写              |         `select UCASE(column_name) from table_name`          |
| LCASE()  |              把字段的小写转换成大写              |         `select LCASE(column_name) from table_name`          |
|  MID()   |               从文本字段中提取字符               |  `select MID(column_name, start[, length]) from table_name`  |
|  LEN()   |               返回文本字段值的长度               |          `select LEN(column_name) from table_name`           |
| ROUND()  |          把数值字段舍入为指定的小数位数          |    `select  ROUND(column_name, decimals) from table_name`    |
|  NOW()   |             返回当前系统的日期和时间             |                `SELECT NOW() FROM table_name`                |
| FORMAT() |              对字段的显示进行格式化              | `SELECT FORMAT(column_name,format) FROM table_name` 例: `SELECT name, url, DATE_FORMAT(Now(),'%Y-%m-%d') AS date FROM Websites;` |

# MySQL

## RDBMS术语

| 数据库     | 数据库是一些关联表的集合。                                   |
| ---------- | ------------------------------------------------------------ |
| **数据表** | 表是数据的矩阵。在一个数据库中的表看起来像一个简单的电子表格。 |
| 列         | 一列(数据元素) 包含了相同类型的数据。                        |
| 行         | 一行（=元组，或记录）是一组相关的数据。                      |
| 冗余       | 存储两倍数据，冗余降低了性能，但提高了数据的安全性。         |
| 主键       | 主键是唯一的。一个数据表中只能包含一个主键。你可以使用主键来查询数据。 |
| 外键       | 外键用于关联两个表。                                         |
| 复合键     | 复合键（组合键）将多个列作为一个索引键，一般用于复合索引。   |
| 索引       | 使用索引可快速访问数据库表中的特定信息。索引是对数据库表中一列或多列的值进行排序的一种结构 |
| 参照完整性 | 参照的完整性要求关系中不允许引用不存在的实体。与实体完整性是关系模型必须满足的完整性约束条件，目的是保证数据的一致性。 |
| 表头       | 每一列的名字。                                               |

## MySQL用户设置

### 添加用户

添加mysql用户，可以在mysql数据库（use mysql）中的user表中添加新用户即可。可以通过`show columns from user`中查看需要那些内容，而后通过`insert into user （columns）value（columns_value）`对用户赋予响应配置。而后使用`FLUSH PRIVILEGES `重新载入授权表。

第二种添加用户方式为：

```
grant select,insert,update,delete,create,drop on database.* to 'name@localhost' identified by 'password'
```

授权相应权限给新加的用户。

## MySQL分组

`group by`语句更具一个或多个列对结果进行分组。在分组的列上，我们可以使用`count`，`sum`，`avg`等函数。

语法：

```
SELECT column_name, function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name;
```

举例：

通过如下操作导入数据：

```
SET NAMES utf8;
SET FOREIGN_KEY_CHECKS = 0;
CREATE TABLE `employee_tbl` (
  `id` int(11) NOT NULL,
  `name` char(10) NOT NULL DEFAULT '',
  `date` datetime NOT NULL,
  `singin` tinyint(4) NOT NULL DEFAULT '0' COMMENT '登录次数',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
BEGIN;
INSERT INTO `employee_tbl` VALUES ('1', '小明', '2016-04-22 15:25:33', '1'), ('2', '小王', '2016-04-20 15:25:47', '3'), ('3', '小丽', '2016-04-19 15:26:02', '2'), ('4', '小王', '2016-04-07 15:26:14', '4'), ('5', '小明', '2016-04-11 15:26:40', '4'), ('6', '小明', '2016-04-04 15:26:54', '2');
COMMIT;
SET FOREIGN_KEY_CHECKS = 1;
```

此时使用`select`得到数据库为：

![Screenshot from 2019-08-14 22-49-49](/home/chst/Pictures/Screenshot from 2019-08-14 22-49-49.png)

使用`group by`语句进行按名字分组，并统计没人的记录数量。

```
SELECT name, COUNT(*) FROM   employee_tbl GROUP BY name;
```

结果为：

![Screenshot from 2019-08-14 22-52-18](/home/chst/Pictures/Screenshot from 2019-08-14 22-52-18.png)

`with rollup`可实现在分组统计基础上再进行相同的统计。如在按名字分组后统计每个人登录次数。

```mysql
SELECT name, SUM(singin) as singin_count FROM  employee_tbl GROUP BY name WITH ROLLUP;
```

![Screenshot from 2019-08-14 22-57-02](/home/chst/Pictures/Screenshot from 2019-08-14 22-57-02.png)

使用`select coalesce(a,b,c)`来设置取代`null`的值。该语句表示，如果`a==null`则取`b`， 如果`b==null`则取`c`。

```mysql
SELECT coalesce(name, '总数'), SUM(singin) as singin_count FROM  employee_tbl GROUP BY name WITH ROLLUP;
```

![Screenshot from 2019-08-14 23-01-25](/home/chst/Pictures/Screenshot from 2019-08-14 23-01-25.png)

## MySQL事务

[菜鸟教程](https://www.runoob.com/mysql/mysql-transaction.html)

## MySQL临时表

[菜鸟教程](https://www.runoob.com/mysql/mysql-temporary-tables.html)

## MySQL复制表

[菜鸟教程](https://www.runoob.com/mysql/mysql-clone-tables.html)

## MySQL元数据

[菜鸟教程](https://www.runoob.com/mysql/mysql-database-info.html)

## MySQL处理重复数据

[菜鸟教程](https://www.runoob.com/mysql/mysql-handling-duplicates.html)

## MySQL导出数据

[菜鸟教程](https://www.runoob.com/mysql/mysql-database-export.html)

## MySQL导入数据

[菜鸟教程](https://www.runoob.com/mysql/mysql-database-import.html)

# C++操作MySQL数据库

