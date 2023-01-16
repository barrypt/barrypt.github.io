---
title: "MySQL教程(五)---常见数据类型"
description: "MySQL 常见数据类型的简单分析"
date: 2020-03-20 22:00:00
draft: false
tags: ["MySQL"]
categories: ["MySQL"]
---

本文主要对 MySQL 常见数据类型进行了简单分析与总结，包括数值、字符、日期和NULL等。

<!--more-->

## 1. 概述

> 本文基于 MySQL 8.0 版本。

MySQL 中常用数据类型可以分为以下几类：

* 1）数值
* 2）字符串
* 3）日期
* 4）NULL

## 2. 数值

### 1. 整形

整形也就是 INT，根据大小不同又细分为如下几种：

| Type      | Storage (Bytes) | Range（SIGNED）        | Range（UNSIGNED） |
| --------- | --------------- | ---------------------- | ----------------- |
| TINYINT   | 1               | -128~127               | 0~255             |
| SMALLINT  | 2               | -32768~32767           | 0~65535           |
| MEDIUMINT | 3               | -8388608~8388607       | 0~16777215        |
| INT       | 4               | -2147483648~2147483647 | 0~4294967295      |
| BIGINT    | 8               | -2^63~2^63-1           | 0~2^64-1          |

**INT(N)形式**

在开发中，我们会碰到有些定义整型的写法是`int(10)`，int(N)我们只需要记住三点：

* 1）**不推荐这样用**，MySQL官方也准备取消对该用法的支持。
* 2）**N 表示的是最大显示宽度，也只会影响显示**，不足的用 0 补足，超过长度的不会被截字，也是直接显示整个数字。
* 3）可以使用`LPAD()`函数来实现 ZEROFILL。

### 2. 浮点型

| Type   | Storage (Bytes) | **备注**         |
| ------ | --------------- | ---------------- |
| FLOAT  | 4               | 精确到约7位小数  |
| DOUBLE | 8               | 精确到约15位小数 |

FLOAT(M,D)、DOUBLE(M、D) 的用法规则：

- **D 表示浮点型数据小数部分的位数**，假如超过 D 位则四舍五入，即1.233 四舍五入为 1.23，1.237 四舍五入为 1.24
- **M 表示浮点型数据总共的位数**，M=5,D=2 则表示总共支持五位，其中小数点前只支持三位数，

当我们不指定M、D的时候，会按照实际的精度来处理，最多处理到硬件支持的极限。

不推荐继续使用FLOAT(M,D)写法，在 MySQL 8.0.17 版本已经废弃。同时 UNSIGNED FLOAT写法也被废弃。

> FLOAT(P) 写法，其中P取值为0~24时转为FLAOT类型，25~53时转为DOUBLE类型。

### 3. 定点型

| **数据类型** | **字节数**                               | **备注**   |
| ------------ | ---------------------------------------- | ---------- |
| DECIMAL      | 对DECIMAL(M,D) ，如果M>D，为M+2否则为D+2 | 精确小数值 |

> MySQL stores `DECIMAL` values in binary format

DECIMAL(M,D)用法：

* M 为总位数，默认为 10，最大为 65。
* D 位小数位数，默认为 0，最大为30。

最后讲一下decimal和float/double的区别，主要体现在两点上：

- float/double在db中存储的是近似值，而decimal则是以字符串形式进行保存的
- decimal(M,D)的规则和float/double相同，但区别在float/double在不指定M、D时默认按照实际精度来处理而decimal在不指定M、D时默认为decimal(10, 0)

> 浮点型和定点型都可以增加UNSIGNED，但是返回却不会发生变化，同时不推荐使用该功能了。

## 2. 字符

### 1.char、varchar
| 数据类型 | 大小（字节） | 备注     |
| ------------ | ---------- | ------------ |
| CHAR        | 0-255         | 定长字符串 |
| VARCHAR       | 	0-65535          | 变长字符串 |

> char(n) 和 varchar(n) 中括号中 n 代表字符的个数，并不代表字节个数，比如 CHAR(30)或者VARCHAR(30)  就可以存储 30 个字符。

关于 char 和 varchar 的对比：

* 1）存储
  * char 为固定长度(建表时指定)，不足指定长度时会用空格补齐。
  * varchar 可变长度，会存储实际字符串。
* 2）大小
  * char 实际大小就是字符所占字节数
  * varchar 实际大小由三部分组成 `字符所占字节数`+`长度记录值(1或2字节)`+`NULL标志位(1字节)`
* 3）查询
  * char 类型查询时会处理掉结尾的所有空格（因为存储时做了补齐操作）
  * varchar 则不会处理



**varchar型数据占用空间大小及可容纳最大字符串限制探究**

**这部分和具体编码方式有关**，详情如下：

- MySQL要求一个**行**的定义长度不能超过 65535 即 64K
- 对于未指定 varchar 字段 not null的表，会有**1个字节**专门表示该字段是否为 null
- varchar(M)会有一个空间用于记录字符串长度
  - 当M范围为0<=M<=255时会专门有一个字节记录varchar型字符串长度
  - 当M>255时会专门有两个字节记录varchar型字符串的长度
  - 把这一点和上一点结合，那么65535个字节实际可用的为65535-3=65532个字节
- 不同编码方式，占用字节数也不同
  - 所有英文无论其编码方式，都占用1个字节，因此最大M=65532；
  - 对于gbk编码，一个汉字占两个字节，因此最大M=65532/2=32766；
  - 对于utf8编码，一个汉字占3个字节，因此最大M=65532/3=21844；
  - 对于utfmb4编码方式，1个字符最大可能占4个字节，那么varchar(M)，M最大为65532/4=16383

同样的，上面是一行中只有varchar型数据的情况，**如果存在其他列，比如同时存在int、double、char这些数据，需要把这些数据所占据的空间减去，才能计算varchar(M)型数据M最大等于多少**。

> 由于mysql自身的特点，如果一个数据表存在varchar字段，则表中的char字段将自动转为varchar字段。在这种情况下设置的char是没有意义的。所以要想利用char的高效率，要保证该表中不存在varchar字段；否则，应该设为varchar字段。
>
> 这里暂时没找到具体原因。。只有下面这个
>
> `InnoDB` encodes fixed-length fields greater than or equal to 768 bytes in length as variable-length fields, which can be stored off-page. For example, a `CHAR(255)` column can exceed 768 bytes if the maximum byte length of the character set is greater than 3, as it is with `utf8mb4`.



### 2. text、blob

**text存储的是字符串而blob存储的是二进制字符串**，简单说blob是用于存储例如图片、音视频这种文件的二进制数据的。


| 数据类型   | 大小（字节）           | 备注         |
| ---------- | ---------------------- | ------------ |
| TINYTEXT   | 0~255                  |              |
| TEXT       | 0-65 535               | 单精度浮点型 |
| MEDIUMTEXT | 0-16 777 215           |              |
| LONGTEXT   | 0-4 294 967 295（4GB） |              |
| TINYBLOB   | 0~255                  |              |
| BLOB       | 0-65 535               | 单精度浮点型 |
| MEDIUMBLOB | 0-16 777 215           |              |
| LONGBLOB   | 0-4 294 967 295（4GB） |              |

text和varchar是一组既有区别又有联系的数据类型，其联系在于**当varchar(M)的M大于某些数值时，varchar会自动转为text**,具体条件如下：

- M>255 时转为 tinytext
- M>500 时转为 text
- M>20000 时转为 mediumtext

所以过大的内容 varchar 和 text 没有区别，同时 varchar(M) 和 text 的区别在于：

- 单行 64K 即 65535字节的空间，varchar 只能用 65533 字节，但是 text 可以 65535 字节全部用起来
- text 可以指定 text(M)，但是 M 无论等于多少都没有影响
- text 不允许有默认值，varchar 允许有默认值

## 3. 日期类型

接着我们看一下 MySQL 中的日期类型，MySQL 支持五种形式的日期类型：

| 数据类型  | 大小（字节） | 格式                | 范围                                     | 备注                      |
| --------- | ------------ | ------------------- | ---------------------------------------- | ------------------------- |
| date      | 3            | yyyy-MM-dd          | -                                        | 存储日期值                |
| time      | 3            | HH:mm:ss            | --                                       | 存储时分秒                |
| year      | 1            | yyyy                | --                                       | 存储年                    |
| datetime  | 8            | yyyy-MM-dd HH:mm:ss | 1000-01-01 00:00:00——9999-12-31 23:59:59 | 存储日期+时间             |
| timestamp | 4            | yyyy-MM-dd HH:mm:ss | 19700101080001——20380119111407           | 存储日期+时间，可作时间戳 |

重点关注一下 datetime 与 timestamp两种类型的区别：

- **大小**
  - datetime 占 8 字节
  - timestamp 占4字节
- **范围** -大小不同，取值范围肯定也不同
  - datetime 的存储范围为1000-01-01 00:00:00——9999-12-31 23:59:59
  - timestamp 存储的时间范围为19700101080001——20380119111407
- **默认值**
  - datetime 默认值为空，当插入的值为 null 时，该列的值就是 null；
  - timestamp 默认值不为空，当插入的值为null 的时候，MySQL会取当前时间
- **时区**
  - datetime 存储的时间与时区无关
  - timestamp 存储的时间及显示的时间都依赖于当前时区

> MySQL converts `TIMESTAMP` values from the current time zone to UTC for storage, and back from UTC to the current time zone for retrieval. (This does not occur for other types such as `DATETIME`.)

**如何选择**

* timestamp 记录经常变化的更新 / 创建 / 发布 / 日志时间 / 购买时间 / 登录时间 / 注册时间等，并且是近来的时间，够用，时区自动处理，比如说做海外购或者业务可能拓展到海外
* datetime 记录固定时间如服务器执行计划任务时间 / 健身锻炼计划时间等，在任何时区都是需要一个固定的时间要做某个事情。超出 timestamp 的时间，如果需要时区必须记得时区处理

## 4. NULL

### 1. 概述

NULL 值为 MySQL 中的一个特殊值。

> “NULL columns require additional space in the row to record whether their values are NULL. For MyISAM tables, each NULL column takes one bit extra, rounded up to the nearest byte.”

NULL 代表的是一种不确定性，所以需要一个额外空行来标注当前是NULL值。

通常用 IS NULL 或者 NOT NULL 来判定一个实例数据是否为不确定的，而不是直接 == 来进行值比较。

### 2. NULL与空值的区别

- MySQL中，NULL 是未知的，且占用空间的。NULL使得索引、索引统计和值都更加复杂，并且影响优化器的判断。
- 空值(`''`)是不占用空间的，注意空值的`''`之间是没有空格。
- 在进行 count() 统计某列的记录数的时候，如果采用的 NULL 值，会被系统自动忽略掉，但是空值是会进行统计到其中的。
- 判断 NULL 使用 IS NULL 或者 NOT NULL，但判断空字符使用 =''`或者 `<>''`来进行处理。
- 对于 timestamp 数据类型，如果插入 NULL 值，则出现的值是当前系统时间。插入空值，则会出现'0000-00-00 00:00:00' 。
- 对于已经创建好的表，普通的列将 NULL 修改为 NOT NULL 带来的性能提升比较小，所以调优时没有必要特意一一查找并将其修改为 NOT NULL。
- 对于已经创建好的表，如果`计划在列上创建索引`，那么尽量修改为 NOT NULL，并且使用 0 或者一个特殊值或者空值`''`。

### 3. NULL值的坑

- NULL作为布尔值的时候，不为1也不为0，任何值和NULL使用运算符（>、<、>=、<=、!=、<>）或者（in、not in、any/some、all），返回值都为NULL，判断是否为空只能用IS NULL、IS NOT NULL
- 当IN和NULL比较时，无法查询出为NULL的记录
- 当NOT IN 后面有NULL值时，不论什么情况下，整个sql的查询结果都为空
- count(字段)无法统计字段为NULL的值，count(\*)可以统计值为null的行
- NULL导致的坑让人防不胜防，强烈建议创建字段的时候字段不允许为NULL，给个默认值

### 4. FAQ

**问题1：索引列允许为NULL，对性能影响有多少？**

存储大量的NULL值，除了计算更复杂之外，数据扫描的代价也会更高一些

**问题2：索引列允许为NULL，会额外存储更多字节吗？**

定义列值允许为NULL并不会增加物理存储代价，但对索引效率的影响要另外考虑

## 5. 参考

`https://dev.mysql.com/doc/refman/8.0/en/data-types.html`

`https://www.cnblogs.com/xrq730/p/8446246.html`

`https://learnku.com/laravel/t/2495/select-the-appropriate-mysql-date-time-type-to-store-your-time`