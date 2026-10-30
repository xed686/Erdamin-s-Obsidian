---
tags: [网络安全, WEB开发, JAVAEE开发]
---

# Java 数据库框架与 SQL 注入审计笔记

> [!info] 🧭 知识导航
> 所属：[[MOC - JAVAEE开发]] ｜ 上级：[[MOC - WEB开发]] ｜ 首页：[[🏠 知识库首页]]
> 上一篇：[[00 JavaEE_Servlet_Filter_Listener安全笔记]] ｜ 下一篇：[[Java_JavaEE全景速成笔记]]
> 🔗 相关：[[00 SQL注入全数据库类型实战笔记]] · [[MySQL入门笔记+SQL注入]]

> **核心主题**：Java 项目中的 SQL 注入问题——不同框架有不同的写法，但注入的根本原因始终是同一个：**用户可控的输入被直接拼接进了 SQL 语句**。本文以"安全视角"为主线，开发配置部分尽量简要，着重讲清楚每个框架的"安全写法 vs 危险写法"以及审计时该看什么。

参考资料：https://mvnrepository.com/（Java 依赖库查询）

---

## 一、JDBC — 最底层的数据库操作

### 1. 简要说明
JDBC（Java Database Connectivity）是 Java 连接数据库最底层的方式，没有任何封装，开发者需要手写全部 SQL。正因为如此，**它是 Java 项目中 SQL 注入问题最容易出现、也最直接可见的地方**——SQL 语句写在代码里，一眼就能看出来用没用拼接。

### 2. 基本使用流程（简要）

```java
// 1. 注册驱动
Class.forName("com.mysql.jdbc.Driver");

// 2. 建立连接
String url = "jdbc:mysql://localhost:3306/phpstudy";
Connection connection = DriverManager.getConnection(url, "root", "123456");

// 3. 执行 SQL（关键步骤，安全问题在这里）
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(sql);

// 4. 处理结果
while (resultSet.next()) {
    int id = resultSet.getInt("id");
    String page_title = resultSet.getString("page_title");
}
```

### 3. ⚠️ SQL 注入：不安全写法 vs 安全写法

| 写法 | 代码示例 | 安全性 |
|---|---|---|
| **不安全（字符串拼接）** | `"SELECT * FROM admin WHERE id=" + id` | 用户输入直接拼进 SQL，典型注入点 |
| **安全（预编译）** | `"SELECT * FROM admin WHERE id=?"` | 参数与 SQL 结构分离，无法注入 |

**不安全写法的危险在哪**：
```java
// id 来自 getParameter("id")，是用户可控输入
String sql = "SELECT * FROM admin WHERE id=" + id;
statement.executeQuery(sql);
```
如果用户把 `id` 传成 `1 OR 1=1`，最终执行的 SQL 变成：
```sql
SELECT * FROM admin WHERE id=1 OR 1=1
```
这会返回整张表的全部数据——这就是一次成功的 SQL 注入。

**安全写法（PreparedStatement 预编译）**：
```java
String sql = "SELECT * FROM admin WHERE id=?";
PreparedStatement ps = connection.prepareStatement(sql);
ps.setInt(1, Integer.parseInt(id)); // 参数作为"数据"传入，不会被解析为 SQL 结构
ResultSet resultSet = ps.executeQuery();
```
预编译的核心原理：**SQL 语句结构在发送给数据库时已经固定好了，后续传入的参数只会被当作"纯数据"处理，无论参数里包含什么特殊字符，都不会改变 SQL 的结构**。

### 4. ⚠️ 审计要点（JDBC）

审计 JDBC 代码时，核心就是两步：
1. **全局搜索 `Statement`** —— 找到所有创建 `Statement` 对象的位置，检查执行的 SQL 是否用了字符串拼接。
2. **追踪拼接的变量来源** —— 看它是不是通过 `getParameter()` 或其他方式从用户输入取来的，如果是，就是注入点。
3. **如果看到的是 `PreparedStatement`** —— 不代表一定安全，还要检查 SQL 字符串里是不是在用 `?` 占位，如果开发者先拼了 SQL 再扔给 `PrepareStatement`，一样是不安全的（只是用了类但没发挥它的作用）。

---

## 二、Hibernate — ORM 框架，但同样存在注入

### 1. 简要说明
Hibernate 是一个 ORM（Object-Relational Mapping，对象关系映射）框架，核心思路是"让开发者不用写 SQL，操作 Java 对象就等于在操作数据库"。它使用 HQL（Hibernate Query Language）代替 SQL，语法非常相似，区别只是操作的是"实体类"而不是"数据库表"。

> 很多人以为用了 ORM 就不会有 SQL 注入——**这是错误的**。HQL 同样支持字符串拼接写法，一旦拼接了用户输入，同样注入。

### 2. ⚠️ SQL/HQL 注入：不安全写法 vs 安全写法

| 写法 | 代码示例 | 安全性 |
|---|---|---|
| **不安全（字符串拼接）** | `"FROM User WHERE username='" + username + "'"` | 直接拼接，可注入 |
| **安全（命名参数占位符）** | `"FROM User WHERE username=:username"` | 参数与 HQL 结构分离 |

**不安全写法**：
```java
String hql = "FROM User WHERE username='" + username + "'";
Query query = session.createQuery(hql);
List users = query.list();
```
构造 `username = ' OR '1'='1`，HQL 变成：
```
FROM User WHERE username='' OR '1'='1'
```
同样返回全部用户数据。

**安全写法（命名参数）**：
```java
String hql = "FROM User WHERE username=:username";
Query query = session.createQuery(hql);
query.setParameter("username", username); // 参数绑定，安全
List users = query.list();
```

### 3. ⚠️ 审计要点（Hibernate）

1. **搜索 `createQuery()`** —— 查看传入的 HQL 字符串是否包含字符串拼接（`+` 操作符 + 变量）。
2. **检查是否使用了参数绑定** —— 看有没有 `setParameter()` 或 `:参数名` 形式的命名参数。
3. **注意 Hibernate 配置文件（`hibernate.cfg.xml`）中的敏感信息**：

```xml
<property name="hibernate.connection.username">root</property>
<property name="hibernate.connection.password">123456</property>
```

> ⚠️ 配置文件本身是信息泄露的常见来源。如果目标是 Java 项目，且能通过路径穿越、任意文件读取等漏洞访问到 `hibernate.cfg.xml` 或 `application.properties` 等配置文件，数据库账号密码往往就在里面。审计或渗透时，发现任意文件读取漏洞后，这些配置文件路径应该优先尝试读取。

---

## 三、MyBatis — 当前最流行的 Java 持久层框架

### 1. 简要说明
MyBatis 是目前 Java 企业开发中**最广泛使用**的数据库框架，特点是"半自动 ORM"——开发者仍需自己写 SQL，但 SQL 写在 XML 文件（Mapper 文件）里，与 Java 代码分离。这一特性对渗透测试和代码审计来说非常重要：**SQL 集中在 Mapper XML 文件里，审计时只需要重点看这些文件**。

### 2. ⚠️ SQL 注入：`#{}` vs `${}` —— MyBatis 中最核心的安全知识点

MyBatis 提供了两种参数写法，**这两者的安全性有本质区别**：

| 写法 | 语法 | 安全性 | 原理 |
|---|---|---|---|
| **安全** | `#{id}` | 安全 | 底层使用 PreparedStatement 预编译，参数作为"纯数据"传入 |
| **危险** | `${id}` | **不安全，直接注入** | 底层使用字符串拼接，参数内容直接替换到 SQL 字符串中 |

**安全写法（`#{}` 占位符）**：
```xml
<select id="selectAdminById" resultType="com.example.model.Admin">
    SELECT * FROM admin WHERE id = #{id}
</select>
```
无论 `id` 传什么值，都会被当作参数传给预编译的 SQL，不会改变 SQL 结构。

**不安全写法（`${}` 直接替换）**：
```xml
<select id="selectAdminById" resultType="com.example.model.Admin">
    SELECT * FROM admin WHERE id = ${id}
</select>
```
`${id}` 会在执行前直接把参数值拼进 SQL 字符串。如果传入 `1 OR 1=1`，实际执行的 SQL 就是：
```sql
SELECT * FROM admin WHERE id = 1 OR 1=1
```
是一次完整的 SQL 注入。

### 3. `${}` 的合法使用场景（了解，防止误判）

`${}` 并非完全不能用，它有一些**无法用 `#{}` 替代的合法场景**，比如动态表名/列名：
```xml
<!-- 表名不能用预编译的占位符，只能用 ${}-->
SELECT * FROM ${tableName}
```
但这类场景**必须通过后端的白名单校验来限制可传入的值**，而不能直接把用户输入传进来。审计时如果看到 `${}` 用在表名/列名位置，应该检查调用这个 Mapper 的 Java 代码里有没有对传入值做严格的白名单限制。

### 4. ⚠️ 审计要点（MyBatis）

MyBatis 是当前 Java 项目中 SQL 注入的**最高频出现场景**，审计方法非常直接：

1. **找到所有 Mapper XML 文件**（通常在 `src/main/resources/` 目录下，文件名含 `Mapper.xml`）。
2. **全局搜索 `${`** —— 所有用了 `${}` 的 SQL 语句都是潜在注入点，逐个分析。
3. **追踪 `${}` 里的变量来源**：
   - 找到对应的 Mapper 接口方法
   - 找到调用这个方法的 Service/Controller 层代码
   - 最终确认这个参数值是否来自用户可控的 HTTP 请求参数（`getParameter()`、`@RequestParam`、`@PathVariable` 等）
4. **注意动态 SQL 拼接**：除了 `${}`，MyBatis 的 `<if>`、`<choose>`、`<foreach>` 等动态 SQL 标签本身是安全的，但如果其中混用了 `${}`，同样是注入点。

---

## 四、各框架注入特征对比（代码审计速查）

| 框架 | 危险关键词/特征 | 安全关键词/特征 |
|---|---|---|
| **JDBC** | `Statement`、SQL 中有 `+` 拼接变量 | `PreparedStatement`、SQL 中有 `?` 占位符 |
| **Hibernate** | `createQuery()` 中 HQL 有 `'"+变量+"'` | `setParameter()`、HQL 中有 `:参数名` |
| **MyBatis** | Mapper XML 中有 `${` | Mapper XML 中全是 `#{` |
| **Spring JPA** | 原生查询（`@Query(nativeQuery=true)`）中有字符串拼接 | 使用 JPQL 方法名推导或 `:参数名` 绑定 |

---

## 五、附：配置文件中的敏感信息泄露

Java 项目的各类配置文件里通常硬编码了数据库账号、密码、密钥等信息，这些文件路径是任意文件读取漏洞发生时的优先目标：

| 框架/工具 | 常见配置文件路径 |
|---|---|
| JDBC / 通用 | `src/main/resources/db.properties` |
| Hibernate | `src/main/resources/hibernate.cfg.xml` |
| MyBatis | `src/main/resources/mybatis-config.xml` |
| Spring Boot | `src/main/resources/application.properties` 或 `application.yml` |
| Maven | `pom.xml`（依赖版本信息，辅助判断组件是否有已知 CVE） |

---

## 六、总结

1. 不管用哪种 Java 数据库框架，SQL 注入的根本原因都是一样的：**用户输入未经处理直接参与了 SQL 语句的"结构构建"**。框架变了，只是注入的表现形式不同。
2. 审计 Java 项目的注入漏洞，按框架选对应的关键词搜索是最高效的方法：JDBC 搜 `Statement` 和 `+`，MyBatis 搜 `${`，Hibernate 搜 `createQuery`。
3. `${}` 和 `#{}` 是 MyBatis 审计中最高频、最核心的知识点——这两个符号的区别是 Java 代码审计面试和实战中必考的内容，需要牢记。
4. 配置文件里的硬编码凭证（数据库账号密码）是 Java 项目信息泄露的典型来源，拿到任意文件读取漏洞后应优先尝试读取以上路径的配置文件。