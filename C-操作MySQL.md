---
title: C++操作MySQL
date: 2019-09-03 23:39:42
tags: MySQL，C++
categories: 数据库
mathjax:
    true
description: MySQL作为开源数据库，可以说是十分常用了，这里简单的介绍一下如何使用C/C++连接并操作mysql数据库。这里只是简单介绍如何使用，并不包含mysql相关细节。具体mysql的学习将会之后系统学习后上传。这里也只是简单的使用，具体更为复杂的后续会有补充，具体参照官方文档。
---

<center/><font size = 12>C++连接MySQL（C API）</font></center>
## ubuntu下配置环境

### 安装MySQL

参考前面的文章。

### 下载配置文件

```
sudo apt-get update
sudo apt-get libmysqld-dev
```

### 测试

写如下代码：

```
#include<my_global.h>
#include<mysql.h>
#include<iostream>
using namespace std;
int main(int argc, char **argv)
{
    cout<<"MySQL client version:"<<mysql_get_client_info()<<endl;
    return 0;
}
```

编译运行：

```
g++ test.cpp -o test.out `mysql_config --libs --include`
```

其中`test.cpp`是我们写的程序，`-o`指明文件输出的名字，这里是`test.out`。之后的`mysql_config --lib --include`(前后要有`)，是指明编译需要的引用文件和库，并且位置不是随意放置（不然报错）。

具体需要添加什么可以根据如下命令查看：

```
mysql_config
```

而后运行生成的文件`./test.out`能够正确打印即可。

## 配置需求

通讯缓冲区必须足够大，以包含单个SQL语句（用于客户端到服务器的流量）和一行返回的数据（用于服务端到客户端的流量）。每个会话的通讯缓冲区都会被动态放大，以处理任何查询，直到最大限制。例如，BLOB值包含最多16MB数据，则必须具有至少16MB的缓冲区限制（在服务器和客户端中）。客户端库中内置默认配置最大值是1GB，但服务器中默认最大值为1MB，可以通过`max_allowed_packet`在服务器启动时更改参数来增加此值。

## 数据结构

### MYSQL

MYSQL是连接数据库的句柄，几乎在所以函数都会用到，不应该尝试拷贝一个MYSQL，不能保证拷贝的结果可以正常使用。

### MYSQL_RES

该结构表示为返回行的查询语句的结果（select，show，describe，explain）。该结果在下文被称之为结果集。

### MYSQL_ROW

该数据结构是一行数据的类型安全表示。目前被实现为计数字节字符串的数组。通过调用`mysql_fetch_row()`获得。

### MYSQL_FIELD

该结构包含元数据：有关字段的信息，例如名字，类型和大小。可以通过不断重复调用`mysql_fetch_field()`函数获得每个元素的信息。字段的值并不是该结构的一部分，而是被包含在MYSQL_ROW。

### MYSQL_FIELD_OFFSET

这是MySQL字段列表中偏移量的类型安全表示。使用`mysql_field_seek()`获得。偏移量是一行中的字段编号，从零开始。

### my_ulonglong

该数据类型表示行的数量和`mysql_affected_rows()`,`mysql_num_rows()`,`mysql_insert_id()`函数。该类型提供一个范围[0, 1.84e19]。一些使用该类型作为返回值的类型，通过返回-1表示出现错误或意外情况。检测是否出现-1可以使用返回值和`(my_ulonglong)-1`或`(my_ulonglong)~0`比较。一些系统无法直接打印该类型，可以先将其转换为`unsigned long`并且使用`%lu`打印该值。例如：

```c
printf("Number of rows: %lu\n", (unsigned long)mysql_num_rows(result));
```

### my_bool

bool类型。

### char *name

字段名字。对于存在别名时，则值为别名。

### char *org_name

字段原名。

### char * table

包含此字段的表的名称（不是字段名称）。对于计算字段，该table为空。如果从视图中选择列，则为视图table命名。如果存在别名，则为别名。对于UNION，值为空字符，对于过程参数，值为过程名。

### char *org_table

与上述基本一致，不过当存在别名时，依然是原始名。

### char *db

数据库的名字。

### char * catalog

目录名称。该值总是"def"

### char * def

字段默认值，作为以null结尾的字符串。仅在使用是设置此项`mysql_list_fields()`。

### unsigned long length

字段宽度，反应了展示时以字节为单位的长度。服务器在确定length在生成查询结果之前，因此这是能够容纳列中最大长度的最小值。

### unsigned long max_length

结果集的字段最大宽度。如果使用`mysql_store_result()`或`mysql_list_fields()`则包含字段的最大长度。如果使用`mysql_use_result()`则该变量的值为0。例如检束float列并且“最宽”值为-12.345，max_length则为7。

### unsigned int name_length

name length.

### unsigned int org_name_length

org_name length

### unsigned int table_length

table length

### unsigned int db_length

db length

### unsigned int catalog_length

catalog length

### unsigned int def_length

def length

### unsigned int flags

flags是一个二进制数字，每一个元素都有一个对应的flags，每一位表示元素一个属性，具体如下。

|          位           |             含义             |
| :-------------------: | :--------------------------: |
|     NOT_NULL_FLAG     |        元素不能是空。        |
|     PRI_KEY_FLAG      |     元素是主键的一部分。     |
|    UNIQUE_KEY_FLAG    |  元素是唯一性约束的一部分。  |
|   MULTIPLE_kEY_FLAG   |  元素是非唯一性键的一部分。  |
|     UNSIGNED_FLAG     |     元素具有无符号属性。     |
|     ZEROFILL_FLAG     |    元素具有0填充的属性。     |
|      BINARY_FLAG      |      元素有二进制属性。      |
|  AUTO_INCREMENT_FLAG  |    元素有自动增加的属性。    |
|       ENUM_FLAG       |         元素是enum。         |
|       SET_FLAG        |         元素是set。          |
|       BLOB_FLAG       | 元素是blob或text。（已弃用） |
|    TIMESTAMP_FLAG     |   元素是时间戳。（已弃用）   |
|       NUM_FLAG        |         元素是数字。         |
| NO_DEFAULT_VALUE_FLAG |        元素无默认值。        |

要检查BOLB或者TIMESTAMP值，检查type是否是MYSQL_TYPE_BLOB或MYSQL_TYPE_TIMESTAMP.

ENUM和SET值作为字符串返回，对于这两个，检查type值为MYSQL_TYPE_STRING并且非lags中ENUM_FLAG或SET_FLAG被置位。

NUM_FLAG表明这一列是数字。这表明类型应该是下面之一：

```
MYSQL_TYPE_DECIMAL, MYSQL_TYPE_NEWDECIMAL, MYSQL_TYPE_TINY, MYSQL_TYPE_SHORT,
MYSQL_TYPE_LONG, MYSQL_TYPE_FLOAT, MYSQL_TYPE_DOUBLE, MYSQL_TYPE_NULL,
MYSQL_TYPE_LONGLONG, MYSQL_TYPE_INT24,MYSQL_TYPE_YEAR.
```

NO_DEFALT_VALUE_FLAG表明该列无设置默认值。这不能被用于NULL列（由于该列设有默认值NULL）和AUTO_INCREAM列（有着隐式默认值）。

下面的列子表明上述内容如何使用：

```c
if(field->flags & NOT_NULL_FLAG)
    print("field connot be null\n");
```

可以使用下表中显示的便捷宏来确定值的布尔状态 `flags`。

| 标志状态             | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| `IS_NOT_NULL(flags)` | 如果此字段定义为NOT NULL，则为True ``                        |
| `IS_PRI_KEY(flags)`  | 如果此字段是主键，则为True                                   |
| `IS_BLOB(flags)`     | 如果此字段为a [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)或 [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)（不建议使用;请测试 `field->type`），则为True |

### unsigned int decimals

数字字段的小数位数，以及时间字段的小数秒精度。

### unsigned int charsetnr

### enum enum_field_type type

元素类型。

| 类型值                  | 类型说明                                                     |
| ----------------------- | ------------------------------------------------------------ |
| `MYSQL_TYPE_TINY`       | [`TINYINT`](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html) 类型 |
| `MYSQL_TYPE_SHORT`      | [`SMALLINT`](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html)类型 |
| `MYSQL_TYPE_LONG`       | [`INTEGER`](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html) 类型 |
| `MYSQL_TYPE_INT24`      | [`MEDIUMINT`](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html) 类型 |
| `MYSQL_TYPE_LONGLONG`   | [`BIGINT`](https://dev.mysql.com/doc/refman/5.7/en/integer-types.html) 类型 |
| `MYSQL_TYPE_DECIMAL`    | [`DECIMAL`](https://dev.mysql.com/doc/refman/5.7/en/fixed-point-types.html)或 [`NUMERIC`](https://dev.mysql.com/doc/refman/5.7/en/fixed-point-types.html)类型 |
| `MYSQL_TYPE_NEWDECIMAL` | 精确数学[`DECIMAL`](https://dev.mysql.com/doc/refman/5.7/en/fixed-point-types.html)或 [`NUMERIC`](https://dev.mysql.com/doc/refman/5.7/en/fixed-point-types.html) |
| `MYSQL_TYPE_FLOAT`      | [`FLOAT`](https://dev.mysql.com/doc/refman/5.7/en/floating-point-types.html) 类型 |
| `MYSQL_TYPE_DOUBLE`     | [`DOUBLE`](https://dev.mysql.com/doc/refman/5.7/en/floating-point-types.html)或 [`REAL`](https://dev.mysql.com/doc/refman/5.7/en/floating-point-types.html)类型 |
| `MYSQL_TYPE_BIT`        | [`BIT`](https://dev.mysql.com/doc/refman/5.7/en/bit-type.html) 类型 |
| `MYSQL_TYPE_TIMESTAMP`  | [`TIMESTAMP`](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) 类型 |
| `MYSQL_TYPE_DATE`       | [`DATE`](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) 类型 |
| `MYSQL_TYPE_TIME`       | [`TIME`](https://dev.mysql.com/doc/refman/5.7/en/time.html) 类型 |
| `MYSQL_TYPE_DATETIME`   | [`DATETIME`](https://dev.mysql.com/doc/refman/5.7/en/datetime.html) 类型 |
| `MYSQL_TYPE_YEAR`       | [`YEAR`](https://dev.mysql.com/doc/refman/5.7/en/year.html) 类型 |
| `MYSQL_TYPE_STRING`     | [`CHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)或 [`BINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html)类型 |
| `MYSQL_TYPE_VAR_STRING` | [`VARCHAR`](https://dev.mysql.com/doc/refman/5.7/en/char.html)或 [`VARBINARY`](https://dev.mysql.com/doc/refman/5.7/en/binary-varbinary.html)类型 |
| `MYSQL_TYPE_BLOB`       | [`BLOB`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)或 [`TEXT`](https://dev.mysql.com/doc/refman/5.7/en/blob.html)字段（用于 `max_length`确定最大长度） |
| `MYSQL_TYPE_SET`        | [`SET`](https://dev.mysql.com/doc/refman/5.7/en/set.html) 类型 |
| `MYSQL_TYPE_ENUM`       | [`ENUM`](https://dev.mysql.com/doc/refman/5.7/en/enum.html) 类型 |
| `MYSQL_TYPE_GEOMETRY`   | 空间类型                                                     |
| `MYSQL_TYPE_NULL`       | `NULL`- 字段                                                 |

`MYSQL_TYPE_TIME2`， `MYSQL_TYPE_DATETIME2`和 `MYSQL_TYPE_TIMESTAMP2`类型码仅在服务器侧使用。客户看到 `MYSQL_TYPE_TIME`， `MYSQL_TYPE_DATETIME`和 `MYSQL_TYPE_TIMESTAMP`代码。

您可以使用`IS_NUM()`宏来测试字段是否具有数字类型。将`type`值传递 给`IS_NUM()` 它，如果该字段是数字，则计算结果为TRUE：

```c
if (IS_NUM(field->type))
    printf("Field is numeric\n");
```

## 程序流程

### 初始化MySQL客户端库

`mysql_library_init()`。

### 连接和初始化

调用`mysql_init()`初始化一个连接的头。使用`mysql_real_connect()`连接到服务器。

### 关闭与MySQL服务器的连接

`mysql_close()`

### 结束使用MySQL客户端库

`mysql_library_end()`

### 交互

在连接处于活动状态时，客户端可以使用`msql_query()`或`mysql_real_query()`向服务器发送SQL语句。二者区别为: `mysql_query()`期望查询指定为以null结尾的字符串，`mysql_real_query()`期待一个计数的字符串。即两个函数只有传递参数形式存在差异。如果字符串中包含二进制数据（可能存在null）必须使用`mysql_real_query()`。

对于非查询语句（如update，insert，delete），我们可以使用`mysql_affected_rows()`获取该语句造成的多少行数据更改。

### 处理结果集

处理结果集有两种方式。一种方式是一次直接获取所有结果，使用`mysql_store_result()`，该命令请求服务器返回查询指令获得的所有行并存储在客户端。第二种方式是客户端迭代的一行一行的获取结果集，命令为`mysql_use_result()`.g该命令迭代索引，但并未从服务器获得真正的行。

无论使用那种方式，访问行均是通过`mysql_fetch_row()`函数。对于`mysql_store_result()`,`mysql_fetch_row()`访问早已经从服务器获取的行。对于`mysql_use_result()`,`mysql_fetch_row()`真正向服务器请求行。每一行可得到的大小可以通过函数`mysql_fetch_length()`获得。

在处理完结果集后，应该使用`mysql_free_result()`去释放其使用的内存。

两种方式的比较：

|          方式          |                             优点                             |                             缺点                             |
| :--------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
| `mysql_store_result()` | 由于行已经被完全获得，因此不但能够连续的访问行，而且可以使用`mysql_data_seek()`和`mysql_row_seek()`去改变当前行所处位置。也能够通过`mysql_num_rows()`获取行的数量。 | 需要是否大的内存来完整的存储所有行，很可能遭遇超出内存的错误。 |
|  `mysql_use_result()`  | 一次只用维护一行，需要很少的内存。并且，由于很少的存储开销，会很快。 | 需要快速的处理每一行以避免占用服务器。不能随意访问，只能按行访问。且，在访问完所以行之前，行的数量未知。最糟的是，必须访问完所以的行，即使在获取一半后查找到了需要的信息。 |

API使客户端可以适当的响应语句，而无需知道语句是否为一个select（查询）。在`mysql_query()`/`mysql_real_query()`之后调用`mysql_store_result()`。如果结果集正确响应，该语句是select并且可以读取行。如果结果集调用失败，调用`mysql_field_count()`函数去确定是否真正发生了未期待的结果。如果返回值为0，表面无返回数据，则表示该命令是非查询语句（update，insert，delete...）。如果结果不是0，则表面已经应该返回了行，但是没有，发生了一个错误。

### 字段信息

`mysql_store_result()`和`mysql_use_result()`使得用户可以获得组成结果集的字段的信息（字段数量，名字，类型等等）。我们可以重复的调用`mysql_fetch_field()`函数来连续的获取行内字段信息，或者通过字段在行内编号通过`mysql_fetch_field_direct()`函数直接获取信息。游标的当前位置可以改变通过`mysql_field_seek()`。设置字段游标影响接下来的`mysql_fetch_field()`函数调用结果。还可以通过一次调用`mysql_fetch_fields()`获取所以字段的信息。

### 错误

为了检测和报告错误，MySQL通过`mysql_errno()`和`mysql_error()`函数提供对错误信息的访问。这些函数返回最近调用的可能成功或失败的错误代码或错误消息，使得用户可以确定错误发生时间和错误信息。

## API

### mysql_library_init()

函数原型：

```
int mysql_library_init(int argc, char **argv, char **groups)
```

该函数用来在调用其他函数之前初始化MySQL客户端库。为了防止内存泄露，应该在使用完成（例如中断连接后）后，调用函数`mysql_library_end()`函数来清理并且释放库产生的资源。参数argc和argv是与main函数类似的，这是作为服务器的可选参数，为了方便，一般argc是0，此时调用该函数通常是`myslq_library_int(0, NULL, NULL)`。0表示正确执行并返回，否则表示错误。

测试代码：

```c++
#include<my_global.h>
#include<mysql.h>
#include<iostream>
using namespace std;
int main(int argc, char **argv)
{
    if(mysql_library_init(0, NULL,NULL))
    {
        cout<<"could not initialize MySQL client library"<<endl;
        return 0;
    }
    else
    {
        cout<<"succeed initialize mysql"<<endl;
        mysql_library_end();
    }
    return 0;
}
```

### mysql_init()

函数原型：

```
MySQL *mysql_init(MYSQL *mysql);
```

为`mysql_real_connect()`分配或初始化一个MYSQL实例。如果参数为NULL，则分配，初始化并返回一个新的MYSQL指针。否则，仅仅只是初始化。当调用`mysql_close()`时释放。正常返回一个初始化后的MYSQL*头，当没有足够的内存时，会返回NULL。

### mysql_real_connect()

#### 函数原型：

```
MYSQL *mysql_real_connect(MYSQL *mysql, const char *host, const char *user, const char *passwd, const char *db, unsigned int port, const char *unix_socket, unsigned long client_flag)
```

该函数试图与运行在host的数据库引擎建立连接。在执行需要MYSQL连接句柄的任意函数在执行前必须要保证该函数正确执行完成。

#### 参数解释

mysql为一个已经存在的mysql的指针，在调用该函数之前使用`mysql_init()`获得。同时，可以使用`mysql_options()`设置很多的连接选项。

host要么是一个主机名，要么是一个IP地址。客户端连接方式为：

1. 如果host是NULL或者"localhost"，会建立一个到本地主机的连接。
   1. 在windows下，如果服务器可得到共享存储的连接，客户端连接到一个使用共享存储的连接。
   2. 在unix下，客户端连接到使用套接字的文件。这里的`unix_socket`参数或者可变的`MYSQL_UNIX_PROT`环境可能被用来明确套接字的名字。
2. 在windows下，如果host是"."，或者TCP/IP不可得并且`unix_socket`未被指明或者host是空，如果服务器存在已命名的管道连接可获得，则会连接到一个已命名的管道上，否则就会报错。
3. 其他情况，将使用TCP/IP

用户也可以使用`MYSQL_OPT_PROTOCOL`或者`MYSQL_OPT_NAMED_PIPE`选项通过`mysql_options()`函数来更该连接方式。

user是数据库的用户名。

passward是用户名对于的密码。

db是数据库的名字。

当port不是0时，代表TCP/IP连接的端口，需要注意，采用哪种连接由host参数决定。

如果`unix_socket`不为NULL这个字符串表面了使用的套接字或者管道的名字。同样，连接类型有host决定。

`client_flag`通常是0，也可以设置为下列值的联合来决定连接特性。

1. 待补充

#### 返回

如果调用正确返回为一个MYSQL *，否则返回NULL。

#### 错误

|    CR_CONN_HOST_ERROR     |                 连接服务器失败。                  |
| :-----------------------: | :-----------------------------------------------: |
|    CR_CONNECTION_ERROR    |              连接到本地服务器失败。               |
|      CR_IPSOCK_ERROR      |              创建一个IP套接字失败。               |
|     CR_OUT_OF_MEMORY      |                    超出内存。                     |
|  CR_SOCKET_CREATE_ERROR   |               创建unix套接字失败。                |
|      CR_UNKNOW_ERROR      |               寻找主机IP地址失败。                |
|     CR_VERSION_ERROR      |            服务端库与客户端库不匹配。             |
|  CR_NAMEDPIPEOPEN_ERROR   |            创建一个管道失败（windows）            |
|  CR_NAMEDPIPEWAIT_ERROR   |              等待管道失败（windows）              |
| CR_NAMEPIPESETSTATE_ERROR |            获取管道句柄失败（windows）            |
|      CR_SERVER_LOST       | 服务器进程终止，或者connect_timeout超过设定时间。 |
|   CR_ALREADY_CONNECTED    |              MYSQL句柄已经存在连接。              |

### mysql_query()

函数原型：

```
int mysql_query(MYSQL *mysql, const char *stmt_str)
```

执行以空字符作为结尾的stmt_str的SQL指令。通常，该字符串必须包含一条SQL指令，并且不能存在分号和`\g`。该指令里面的字符串不能包含二进制数据，如果存在，必须使用`mysql_real_query()`代替，这是因为二进制数据中可能存在`\0`，这会被认为是终止。如果想要知道是否该指令返回了结果集，可以使用`mysql_field_count()`函数确定。具体方法看上文的程序流程中的处理结果集部分。

返回值为0表示成功，否则表示发生错误。

错误：

| CR_COMMANDS_OUT_OF_SYNC |        目录执行顺序错误。        |
| :---------------------: | :------------------------------: |
|  CR_SERVER_GONE_ERROR   |        MySQL服务器终止。         |
|     CR_SERVER_LOST      | 在请求时与服务器直接的连接中断。 |
|    CR_UNKNOWN_ERROR     |      发生了一个未知的错误。      |

### mysql_field_count()

函数原型：

```
unsigned int mysql_field_count(MYSQL *mysql)
```

函数返回最近连接中请求生成结果集的列的数量。通常在`mysql_store_result()`返回为NULL时调用该函数去确定是否`mysql_store_result()`应该已经生成了一个非空结果。这使得客户端能够采取使得的行为当不知道请求是否为SELECT（类SELECT）时。函数返回值为一个非负整数，如果是0，表示该指令为非查询指令，否则表示为查询指令。

例：

```
if (mysql_query(&mysql,query_string))
{
    // error
}
else // query succeeded, process any data returned by it
{
    result = mysql_store_result(&mysql);
    if (result) // there are rows
    {
        num_fields = mysql_num_fields(result);
        // retrieve rows, then call mysql_free_result(result)
    }
    else // mysql_store_result() returned nothing; should it have?
    {
        if(mysql_field_count(&mysql) == 0)
        {
            // query does not return data
            // (it was not a SELECT)
            num_rows = mysql_affected_rows(&mysql);
        }
        else // mysql_store_result() should have returned data
        {
            fprintf(stderr, "Error: %s\n", mysql_error(&mysql));
        }
    }
}
```

### mysql_store_result()

#### 函数原型：

```
MYSQL_RES *mysql_store_result(MYSQL *mysql)
```

#### 描述

在调用完`mysql_query()`或`mysql_real_query()`生成结果集的指令（select，show，describe，explain，table等等）后必须调用`mysql_store_result()`/`mysql_use_result()`。同时在处理完结果集后需要调用`mysql_free_result()`函数。在不产生结果集的语句中，不必调用`mysql_store_result()`/`mysql_use_result()`，但在任何情况下都调用`mysql_store_result()`也不会有任何危害或者产生明显性能下降。能够通过检测是否`mysql_store_result()`返回的为非0值来判断是否存在一个结果集。

如果支持多指令，则应该在调用`mysql_query()`或`mysql_real_query()`后循环调用`mysql_next_result()`查看更多结果集。

如果想要检查一条指令是否应该返回一个结果集，可以使用`mysql_field_count()`函数进行检查。

`mysql_store_result()`读取完整的客户端发出的指令产生的结果，并且配置一个MYSQL_RES类的实例存储结果。

`mysql_store_result()`返回NULL，当指令未返回一个结果集（如，指令为insert）或产生一个错误或者读取结果集失败。

当没有结果集没有行存在时，返回一个空的结果集（空的结果集不同于空指针）。

在调用`mysql_store_result()`后正确获得结果集后，可以调用`mysql_num_rows()`获得行数。调用`mysql_fetch_row()`从结果集中获取行。或者`mysql_row_seek()`/`mysql_row_tell()`来获取或者/设置当前行的位置。

#### 返回值

返回一个MYSQL_RES指针。如果不存在结果集或发生错误，返回NULL。检测是否发生错误可以使用`mysql_error()`判断是否返回非空字符串，`mysql_errno()`返回值非0，或者`mysql_field_count()`返回值非零。

#### 错误

| CR_COMMANDS_OUT_OF_SYNC | 命令执行顺序异常 |
| :---------------------: | :--------------: |
|    CR_OUT_OF_MEMORY     |     超出内存     |
|  CR_SERVER_GONE_ERROR   |    服务器终止    |
|     CR_SERVER_LOST      | 与服务器连接中断 |
|    CR_UNKNOWN_ERROR     |     未知异常     |

### mysql_use_result()

#### 函数原型

```
MYSQL_RES *mysql_use_result(MYSQL *result)
```

#### 描述

在调用完`mysql_query()`或`mysql_real_query()`生成结果集的指令（select，show，describe，explain，table等等）后必须调用`mysql_store_result()`/`mysql_use_result()`。同时在处理完结果集后需要调用`mysql_free_result()`函数。

`mysql_use_result()`初始化了一个结果集的检索而并未真正读取到了结果集。相反的，每一行必须被独立的检索通过调用`mysql_fetch_now()`，此命令直接从服务器请求行而不是装载临时表或者本地内存数据。

另一方面，`mysql_use_result()`如果对客户端的每一行进行大量处理，或者输出发送到用户可以键入的屏幕`^S`（停止滚动），则不应使用 锁定读取。这会占用服务器并阻止其他线程更新从中获取数据的任何表。

在使用`mysql_use_result()`时，必须处理`mysql_fetch_row()`直到返回一个NULL。否则，剩下的行将会作为下一条指令的结果集的一部分返回。C API会报错：Commands out of sync; you can't run this command now。

不得使用 `mysql_data_seek()`， `mysql_row_seek()`， `mysql_row_tell()`， `mysql_num_rows()`，或 `mysql_affected_rows()`从返回的结果 `mysql_use_result()`，也不得发出其他查询，直至 `mysql_use_result()`完成。（但是，在获取所有行之后，`mysql_num_rows()`准确地返回获取的行数。）

一旦处理完成结果集，必须调用`mysql_free_result()`.

#### 返回值

返回一个MYSQL_RES类指针。如果出错，返回NULL。

#### 错误

| CR_COMMANDS_OUT_OF_SYNC | 命令执行顺序异常 |
| :---------------------: | :--------------: |
|    CR_OUT_OF_MEMORY     |     超出内存     |
|  CR_SERVER_GONE_ERROR   |    服务器终止    |
|     CR_SERVER_LOST      | 与服务器连接中断 |
|    CR_UNKNOWN_ERROR     |     未知异常     |



### mysql_fetch_row()

#### 函数原型

```
MYSQL_ROW mysql_fetch_row(MYSQL_RES *result)
```

#### 描述

`mysql_fetch_row()`索引结果集的下一行。对于`mysql_store_result()`,如果返回空表示达到结尾。对于`mysql_use_result()`，如果返回空表示到达结尾或发生错误。

行内元素数量被确定通过`mysql_num_fields(result)`。如果row是调用`mysql_fetch_row()`的返回值，指针内值被获取通过`row[0]`到`row[mysql_num_fields(result)-1]`。NULL值通过空指针表示。

可以通过调用`mysql_fetch_lengths()`获得行中的字段值的长度。空字段和包含`NULL`两者的字段长度为0; 您可以通过检查字段值的指针来区分这些。如果指针是 `NULL`，则字段为`NULL`; 否则，该字段为空。

#### 返回值

MYSQL_ROW，或者NULL。对于`mysql_use_result()`，返回空时应该通过`mysql_error()`/`mysql_errno()`判断。

#### 错误

CR_SERVER_LOST

查询期间丢失了与服务器的连接。

CR_UNKNOWN_ERROR

出现未知错误。

#### 例

```
MYSQL_ROW row;
unsigned int num_fields;
unsigned int i;

num_fields = mysql_num_fields(result);
while ((row = mysql_fetch_row(result)))
{
   unsigned long *lengths;
   lengths = mysql_fetch_lengths(result);
   for(i = 0; i < num_fields; i++)
   {
       printf("[%.*s] ", (int) lengths[i],
              row[i] ? row[i] : "NULL");
   }
   printf("\n");
}
```

### mysql_affected_rows()

#### 函数原型：

```
my_ulonglong mysql_affected_rows(MYSQL *mysql)
```

#### 描述

该函数可以在调用`mysql_query()`/`mysql_real_query()`后调用。它返回被更改行的数量（试用于update, delete, insert)。对于select命令，该函数与`mysql_num_rows()`作用一样。

由于`mysql_affected_rows()`返回一个无符号整数，与-1比较可以试用`(my_ulonglong)-1`(或者`(my_ulonglong)~0`，等价)

### mysql_close()

```
void mysql_close(MYSQL *mysql)
```

关闭已经打开的连接。

### mysql_data_seek()

```
void mysql_data_seek(MYSQL_RES *result, my_ulonglong offset)
```

将游标定位到结果集的指定行。其中offset指定行的索引。该值处于0到`mysql_num_rows(result)-1`之间。该函数只能在`mysql_store_result()`生成的结果集中使用。

### mysql_errno()

```
unsigned int mysql_errno(MYSQL *mysql)
```

该函数返回最近因此API调用产生的错误代码。非零意味着存在错误，否则表示没有错误。客户端错误信息被列出在MySQL errmsg.h中，服务器错误信息被列出在mysql_error.h。

一条重要规则是：所以请求服务器的指令如果成功会重置`mysql_errno()`。

### mysql_error()

```
const char *mysql_error(MYSQL *mysql)
```

该函数返回最近一次调用API造成的错误信息，如果没有发生错误，则会返回上一次的错误或者空字符串。

一条重要规则是：所以请求服务器的指令如果成功会重置`mysql_error()`。

### mysql_fetch_field()

```
MYSQL_FIELD *mysql_fetch_field(MYSQL_RES *result)
```

该函数返回一个MYSQL_FIELD结果的指针。重复调用该函数来获取所以列字段的信息。当返回空是表示没有了。

### mysql_fetch_field_direct()

```
MYSQL_FIELD *mysql_fetch_field_direct(MYSQL_RES *result, unsigned int fieldnr)
```

该函数提供一个索引来获取对于字段信息。fieldnr处于0到`mysql_num_fields(result)-1`之间。

### mysql_fetch_fields()

```
MYSQL_FIELD *mysql_fetch_fields(MYSQL_RES *result)
```

返回一个字段信息结构的矩阵,元素是MYSQL_FIELD，不是指针。

### mysql_fetch_lengths()

函数原型：

```
unsigned long *mysql_fetch_lengths(MYSQL_RES *result)
```

函数返回在当前结果集当前游标行中，各个元素对应的长度。当我们想要拷贝对应内容时，该函数是是否有用的，这可以使我们避免调用strlen()函数。并且，如果结果集包含二进制数据，我们必须使用该函数来确定数据的大小，这是由于如果字符串中包含空字符，strlen()将返回错误的结果。对于空的字段或者字段值为NULL，长度为0.

返回值为一个非负长整型数组，代表每个字段长度（不包含结尾空字符）。如果出错返回NULL。

## 例

```
#include<iostream>
#include<mysql.h>
using namespace std;
int main()
{
    mysql_library_init(0, NULL, NULL);
    MYSQL *db = mysql_init(NULL);
    if(mysql_real_connect(db,"localhost","root","****","testdb",0,NULL,0))
    {
        int is_succeed = mysql_query(db,"select * from students");
        MYSQL_RES *result = mysql_store_result(db);
        if(result)
        {

            int num = mysql_num_rows(result);
            cout<<"testdb has num:"<<num<<endl;
            MYSQL_ROW row = mysql_fetch_row(result);
            int columns = mysql_field_count(db);
            MYSQL_FIELD *columns_info = mysql_fetch_fields(result);
            for(int i=0;i<columns;i++)
            {
                cout<<i+1<<"th's name is:"<<columns_info[i].name<<endl;
            }
            while(row)
            {
                unsigned long *column_size = mysql_fetch_lengths(result);
                for(int i=0;i<columns;i++)
                {
                    cout<<column_size[i]<<"  "<<(column_size[i] ? row[i]:"NULL");
                    cout<<"  ";
                }
                cout<<endl;
                row = mysql_fetch_row(result);
            }
            if(mysql_errno(db))
                cout<<mysql_error(db)<<endl;
            else
                cout<<"everything in right"<<endl;
        }
        mysql_free_result(result);
        mysql_query(db,"update students set score=0");
        int affected_num =  mysql_affected_rows(db);
        cout<<"affected row is:"<<affected_num<<endl;
        mysql_library_end();
    }
    return 0;
}
```

