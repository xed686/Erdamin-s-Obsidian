# JavaEE Servlet / Filter / Listener 笔记

> **阅读前提示**：本文面向初学者，尽量把每个概念先解释"是什么、为什么存在"，再讲"安全层面关注什么"。安全相关内容会用 `> ⚠️ 安全` 标记重点说明。

---

## 一、Servlet 是什么（先建立整体概念）

### 1. 用一句话理解

**Servlet 就是一段运行在服务器端、专门处理 HTTP 请求并返回响应的 Java 程序。**

打个比方：你去餐厅点餐，服务员（Servlet）是厨房（后端数据库/业务逻辑）和顾客（浏览器）之间的中间层——顾客发出点餐请求，服务员负责接单、传给厨房、再把菜端回来。

用技术语言说：Servlet 是运行在 Web 服务器（如 Tomcat）上的 Java 程序，它接收浏览器发来的 HTTP 请求，处理之后（比如查数据库），再把结果以 HTTP 响应的形式返回给浏览器。常见用途包括：接收表单提交的数据、查询数据库并把结果显示给用户、动态生成网页内容。

### 2. Servlet 在整个请求流程中的位置

参考图示的时序图，一次完整的请求流程是：

```
浏览器(客户端)
    ↓ 1. 发送 HTTP 请求
Servlet 容器(Tomcat)
    ↓ 2. 解析请求，判断交给哪个 Servlet 处理
    ↓ 3. 如果该 Servlet 还没创建，先创建实例
    ↓ 4. 调用 init() 初始化
    ↓ 5. 调用 service() 处理请求（内部根据请求方法再分发给 doGet/doPost）
    ↓ 6. Servlet 输出响应内容
    ↓ 7. 容器把响应返回给浏览器
    ↓ 8. 服务器关闭时，调用 destroy() 销毁 Servlet
```

> ⚠️ **安全**：这个流程说明了一件很重要的事——Servlet 是由 Tomcat（Servlet 容器）统一管理的，正常情况下每个 Servlet 都要在 `web.xml` 或 `@WebServlet` 注解里**显式注册**才能被访问到。后文讲到的**内存马**，本质上就是"绕过这个正常注册流程，在服务器运行时动态注入一个新的 Servlet/Filter/Listener"——这也是为什么它很难被常规手段发现。

---

## 二、开发环境搭建（JavaEE）

需要安装并配置：
- **IDEA**：Java 主流开发 IDE（需安装激活，并安装 JavaEE 相关开发插件）
- **JDK**：Java 开发工具包
- **Tomcat**：Servlet 容器/Web 服务器

在 IDEA 中新建项目时，选择 JavaEE Web 项目类型，并配置好 Tomcat 服务器，就可以开始开发了。

---

## 三、Servlet 生命周期

### 1. 四个阶段

参考图示中的四个阶段：

| 阶段  | 对应方法                                 | 调用时机                              | 调用次数         |
| --- | ------------------------------------ | --------------------------------- | ------------ |
| 实例化 | 构造方法                                 | 第一次访问该 Servlet 时（或服务器启动时预加载），创建对象 | 仅一次          |
| 初始化 | `init()`                             | 实例创建好之后立即执行，用于做一些初始化准备工作（如读取配置文件） | 仅一次          |
| 服务  | `service()` → `doGet()` / `doPost()` | **每次有请求进来都会调用**，是真正处理业务逻辑的地方      | 多次（每次请求调用一次） |
| 销毁  | `destroy()`                          | 服务器关闭或 Servlet 被卸载时调用，用于释放资源      | 仅一次          |

### 2. 用 IDEA 快速生成内置方法

`Alt + Insert` 快捷键可以快速生成 `init`、`service`（`doGet`/`doPost` 等）、`destroy` 等内置方法的代码模板，不需要手写。

---

## 四、创建和使用 Servlet

### 1. 创建一个类继承 HttpServlet

```java
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) {
        // 处理 GET 请求的逻辑
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) {
        // 处理 POST 请求的逻辑
    }
}
```

### 2. 配置路由（两种方式，效果相同）

**方式一：`web.xml` 配置（传统写法）**
```xml
<servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.example.HelloServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>HelloServlet</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```

**方式二：注解配置（现代写法，更简洁）**
```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet { ... }
```

### 3. 接收请求参数 / 返回响应

**接收参数（HttpServletRequest）**：

| 方法                           | 作用                    |
| ---------------------------- | --------------------- |
| `getParameter("name")`       | 获取单个参数值，返回 String     |
| `getParameterValues("name")` | 获取同名多个参数值，返回 String[] |

**返回响应（HttpServletResponse）**：

| 方法                       | 作用                      |
| ------------------------ | ----------------------- |
| `setCharacterEncoding()` | 设置编码格式（防乱码）             |
| `setContentType()`       | 设置响应内容类型（如 `text/html`） |
| `getWriter()`            | 获取字符输出流，用于向浏览器写出响应内容    |

> ⚠️ **安全**：`getParameter()` 拿到的是**用户直接提交的原始数据**，从安全角度看这就是"不可信的外部输入"。如果代码把这个值直接拿去拼 SQL 语句、拼系统命令、写文件路径，或者直接输出到响应页面，就会分别对应 **SQL 注入**、**命令执行**、**路径穿越**、**XSS** 等漏洞。审计 JavaEE 项目时，看到 `getParameter` 就要立刻向下追踪这个值被拿去做了什么。

---

## 五、JavaEE 过滤器 Filter

### 1. Filter 是什么，为什么存在

**Filter（过滤器）就是在请求到达 Servlet 之前（或响应返回浏览器之前）拦截并处理它的一道"关卡"。**

可以把它理解成公司的门禁：访客（请求）要进门（到达 Servlet）之前，保安（Filter）会先检查一下——有没有证件（权限）、是不是可疑人员（恶意 payload）、进入记录是否要登记（日志）——如果合格就放行，否则就拦截。

多个 Filter 可以形成一条"过滤器链"，请求依次经过每一道关卡才到达 Servlet，响应返回时也要依次经过这些关卡（逆序）。

### 2. Filter 生命周期方法

| 方法           | 调用时机            | 作用                                      |
| ------------ | --------------- | --------------------------------------- |
| `init()`     | 服务器启动时          | 初始化过滤器                                  |
| `doFilter()` | **每次请求经过该过滤器时** | 执行过滤逻辑，处理完后调用 `chain.doFilter()` 放行给下一关 |
| `destroy()`  | 服务器关闭时          | 释放资源                                    |

### 3. Filter 配置示例（以 XSS 过滤为例）

**注解方式**：
```java
@WebFilter("/xss")
public class XssFilter implements Filter {
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        // 在这里检测/处理请求，再决定是否放行
        chain.doFilter(request, response); // 放行
    }
}
```

**`web.xml` 方式**：
```xml
<filter>
    <filter-name>xssFilter</filter-name>
    <filter-class>com.example.filter.XssFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>xssFilter</filter-name>
    <url-pattern>/xss</url-pattern>
</filter-mapping>
```

### 4. Filter 的安全场景（这是重点）

> ⚠️ **安全 — 防守方（蓝队/开发）视角**：
>
> Filter 是 JavaEE 中最常见的**统一安全防护**实现位置：
> - **XSS 过滤**：在 `doFilter()` 里检测请求参数中是否包含 `<script>`、`onerror=` 等危险字符，拦截或净化后再放行。
> - **SQL 注入过滤**：检测参数中是否含有 `' OR`、`--`、`UNION SELECT` 等 SQL 注入特征字符串。
> - **权限访问控制**：检查当前请求是否携带了合法的 Session/Cookie/Token，没有登录就直接 302 重定向到登录页，禁止访问受保护资源。
> - **其他**：压缩响应内容、统一设置字符编码、日志记录等。

> ⚠️ **安全 — 攻击方（红队）视角 / 代码审计**：
>
> Filter 实际上是代码审计中一个非常关键的分析节点，理解它的运作方式对绕过防护和理解"内存马"都至关重要：
>
> **1. 分析过滤器是否可以被绕过**：如果一个 Filter 只对 `/admin` 路径做了权限校验，而后台入口实际上还可以通过 `/Admin`、`/ADMIN`、`/admin/` 等大小写变体或末尾斜杠访问，就存在 Filter 规则未覆盖的绕过可能。这是 JavaEE 权限绕过漏洞的常见成因。
>
> **2. 分析过滤器"放行"逻辑**：如果 `doFilter()` 里的拦截逻辑写得不严谨（如用黑名单检测但过滤不全、用字符串 `contains` 检测但可被编码绕过），相当于这道"关卡"形同虚设，攻击者构造特定 payload 就能绕过。
>
> **3. 内存马的核心原理**：
> 内存马（Memory Webshell）是近年来红队最常用的一种持久化/权限维持技术。它的核心思路是：**在不写入任何文件的情况下，通过 Java 反射/反序列化等手段，直接在 Tomcat 运行时的内存中动态注册一个新的 Filter（或 Servlet/Listener）**。这个 Filter 没有在 `web.xml` 里登记，重启服务器就会消失，也不会在文件系统上留下任何痕迹，因此极难被传统的"查文件"方式检测到。
>
> 一旦内存马注册成功，攻击者就拥有了一个"隐形后门"——每次请求经过这个 Filter 时，都会触发其中内置的恶意代码，实现持续的远程命令执行。
>
> 内存马参考：https://mp.weixin.qq.com/s/hev4G1FivLtqKjt0VhHKmw
>
> **4. 蓝队如何清理内存马**：
> 正是因为内存马不在文件系统上，普通的文件扫描找不到它，必须通过 Java Agent 注入工具（如 `arthas`）或专用内存马检测工具，直接扫描 JVM 运行时内存中已注册的所有 Filter/Servlet，找到其中"不在 `web.xml` 中、来源不明"的注册项，再通过反射手段将其从内存中卸载。

---

## 六、JavaEE 监听器 Listener

### 1. Listener 是什么，为什么存在

**Listener（监听器）就是一个"自动触发"的机制——当某些特定事件发生时，系统自动调用你写好的处理函数，你不需要主动去调用它。**

类比生活：就像手机的通知功能，你不用一直刷屏检查——微信有新消息来了（事件发生），系统自动把通知推给你（调用 Listener 里的方法）。

在 JavaEE 里，Listener 可以监听三类对象的事件：

| 监听对象                    | 可监听的事件类型                     |
| ----------------------- | ---------------------------- |
| `ServletContext`（应用上下文） | 整个 Web 应用的创建和销毁（服务器启动/关闭）    |
| `HttpSession`（会话）       | Session 的创建和销毁、Session 属性被修改 |
| `ServletRequest`（请求）    | 请求对象的创建和销毁、请求属性被修改           |

### 2. Listener 生命周期方法（以 Session 监听为例）

对 Session 的监听通常实现 `HttpSessionListener` 接口：

```java
@WebListener
public class SessionListener implements HttpSessionListener {
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        // 每当有新的 Session 被创建时（即有新用户登录/访问），这里会被自动调用
        System.out.println("新 Session 创建：" + se.getSession().getId());
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        // 当 Session 被销毁时（用户登出/Session 超时），这里会被自动调用
        System.out.println("Session 销毁：" + se.getSession().getId());
    }
}
```

### 3. Listener 配置

**注解方式**：
```java
@WebListener
public class MyListener implements HttpSessionListener { ... }
```

**`web.xml` 方式**：
```xml
<listener>
    <listener-class>com.example.listener.MyListener</listener-class>
</listener>
```

### 4. Listener 的安全场景（这是重点）

> ⚠️ **安全 — 防守方视角**：
>
> - **Session 安全管理**：通过监听 Session 创建/销毁事件，可以实现"统计当前在线用户数"、"限制同一账号最多同时登录几个设备"等安全控制逻辑。
> - **异常行为感知**：结合 `ServletRequestListener` 监听每次请求的创建，可以在 Listener 里实现对访问频率的统计，辅助识别"一个 IP 短时间内大量请求"这类爆破/扫描行为，并触发告警或封禁。

> ⚠️ **安全 — 攻击方（红队）视角 / 代码审计**：
>
> **1. 代码审计中分析执行逻辑**：
> Listener 的执行时机"隐藏"在特定事件触发的背后，初学者审计时容易忽略——看起来代码里没有显式调用某段逻辑，但其实那段逻辑是作为 Listener 被事件自动触发的。例如，某段"验证 Session 合法性"的安全逻辑如果放在了 `sessionCreated` 里，而代码审计者没有注意到这个 Listener，就可能误判某个操作缺乏鉴权保护。**审计 JavaEE 项目时，应把 `web.xml` 中注册的所有 `<listener>` 和代码中所有 `@WebListener` 注解都列出来，逐一分析其触发条件和执行内容。**
>
> **2. Listener 型内存马**：
> 与 Filter 型内存马原理相同，Listener 型内存马是通过在运行时动态注册一个恶意的 `ServletRequestListener`（监听每次请求创建事件）来实现的——这样一来，攻击者的恶意代码会在**每一次 HTTP 请求到来时**被自动触发执行，实现命令执行效果，且同样不在文件系统上留下痕迹。
>
> **三种内存马类型的比较**：
>
> | 内存马类型 | 触发时机 | 优势 |
> |---|---|---|
> | Servlet 型 | 访问特定路径时触发 | 简单直接，但路径固定，相对容易被扫描发现 |
> | Filter 型 | 每次请求经过该 Filter 时触发 | 可绑定通配符路径（`/*`），覆盖面广，最常见 |
> | Listener 型 | 每次请求创建/Session 创建等事件触发 | 隐蔽性最强，不需要特定路径绑定，更难被发现 |

参考：https://blog.csdn.net/qq_52797170/article/details/124023760

---

## 七、三者关系速览（Request 流转全链路）

参考图示中的流程图，一次完整的请求处理过程是：

```
浏览器(Actor)
    ↓ 请求 Request
Listener（监听到请求创建事件，触发对应处理逻辑）
    ↓
Filter（过滤器链，依次检查/处理请求）
    ↓（放行）
Servlet（真正执行业务逻辑，处理完毕）
    ↑
Filter（响应返回时逆序经过过滤器链）
    ↑
Listener（监听到请求销毁事件，触发对应处理逻辑）
    ↑ 响应 Response
浏览器(Actor)
```

> ⚠️ **安全**：理解这条完整链路，对内存马注入点的选择和检测都有直接帮助——Filter 型内存马卡在过滤器链位置，每次请求都过；Listener 型内存马卡在请求创建事件位置，比 Filter 更早触发、也更难被常规安全检测工具发现。

---

## 八、参考资料

- Servlet 完整说明：https://mp.weixin.qq.com/s/c_4fOTBKDcByv8MZ9ayaRg
- Servlet 教程：https://blog.csdn.net/qq_52173163/article/details/121110753
- 内存马详解：https://mp.weixin.qq.com/s/hev4G1FivLtqKjt0VhHKmw
- Listener 教程：https://blog.csdn.net/qq_52797170/article/details/124023760

---

## 九、总结

1. **Servlet** 是处理 HTTP 请求的核心单元，`getParameter()` 获取的用户输入是 JavaEE 项目中所有注入类漏洞的"源头"，审计时必须追踪每一个参数的去向。
2. **Filter** 是统一安全防护的常见实现位置，同时也是内存马最常见的注入目标——理解 Filter 的工作原理，是同时理解"如何防护"和"如何绕过/植入"的基础。
3. **Listener** 因其"事件驱动、隐式触发"的特性，在代码审计中容易被忽视，同时也是三种内存马类型中隐蔽性最强的一种。
4. 三者的关系是**串联的**（Listener → Filter → Servlet），内存马本质上就是"在不写文件的情况下，往这条链路的某个位置动态插入一个恶意节点"——这是理解内存马原理最核心的一句话。
[[01 Java数据库框架与SQL注入审计笔记]]