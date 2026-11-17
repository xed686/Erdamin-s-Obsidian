# HaE 规则中文速查表 · 我的适配版

> 对应你当前实际加载的规则：**7 个分类 / 48 条规则**（39 条内置默认 + 9 条你自己加的）。
> 规则名保留英文原名（和插件界面一致），说明用中文。
> 用法：挂代理浏览目标，HaE 自动给命中的报文染色。你看到颜色后，对照本表判断"这是什么、要不要跟进"。
>
> 📌 你自己加的是第 **6**（SSRF Param Name）、**7**（Dangerous Path）两节。

---

## 先看懂颜色和几个关键列

**颜色 = 优先级信号**，不是随便配的。看到报文被染色，先按颜色决定看不看：

| 颜色 | 大致含义 | 你的反应 |
|---|---|---|
| 🔴 red | 高危 / 可能直接出洞 | 立刻看 |
| 🟠 orange | 敏感数据泄露 | 优先看 |
| 🟡 yellow | 可疑点 / 值得测 | 排队看 |
| 🔵 cyan | 参数特征，提示测试方向 | 结合场景看 |
| 🟢 green | 常规识别 / 信息提取 | 批量参考 |
| ⚪ gray | 链接、URI 等基础信息 | 做资产梳理用 |
| pink / magenta | 前端源码、路由线索 | 做 JS 分析用 |

**几个列的意思**（前面聊过，这里再固化一下）：

- **范围 Scope**：这条规则去报文哪个部位匹配（`request` 请求 / `request line` 请求行即 URL 路径 / `response body` 响应体 / `any header` 所有头 / `any` 整个报文）。限定范围是为了**少误报、跑得快**。
- **引擎 Engine**：`dfa` 快、特性少；`nfa` 慢、支持分组提取。指纹类多用 dfa，提取/锚定类多用 nfa。
- **敏感 Sensitive**：是否区分大小写。特征串大小写固定的（如 JWT）勾上可减少误报；你加的那些参数/路径词都没勾（大小写不敏感），所以 `URL=`、`Admin` 也能命中。

---

## 1. Fingerprint（指纹识别）—— 认出目标用了什么组件

这类规则干的是**资产识别**，不是直接找洞。它告诉你"目标用了什么框架"，你再凭经验把组件名和已知漏洞对上。价值高度依赖你对各框架历史漏洞的熟悉度。

| 规则名 | 抓什么 | 颜色 | 命中意味着 / 跟进方向 |
|---|---|---|---|
| **Shiro** | 响应头里 `rememberMe=` / `=deleteMe` | 🟢 | 用了 Apache Shiro → 联想 Shiro 反序列化（RCE）、硬编码密钥爆破 |
| **JSON Web Token** | `eyJ` 开头的 JWT 串 | 🟢 | 抓到 JWT → 测弱密钥爆破、`alg:none` 算法混淆、越权改 payload |
| **Swagger UI** | `swagger-ui.html`、`"swagger":` 等 | 🔴 | 接口文档暴露 → 拿到全站 API 清单，信息收集大礼包，常接未授权访问 |
| **Ueditor** | `ueditor.config/all.js` | 🟢 | 百度富文本编辑器 → 经典任意文件上传、SSRF |
| **Druid** | `Druid Stat Index` | 🟠 | 阿里 Druid 监控页 → 常未授权访问，泄露 SQL、Session、URL |
| **PDF.js Viewer** | `pdf.worker` | 🟢 | 用了 PDF.js → 部分版本有 XSS |
| **Vite DevMode** | `/@vite/client` | 🔴 | 前端开发模式没关 → 可能源码泄露、接口暴露 |

---

## 2. Maybe Vulnerability（可疑漏洞点）—— 直接指向测试动作

这类规则把"可能有洞的特征"标出来，命中后基本就是"顺着测"。

| 规则名 | 抓什么 | 颜色 | 命中意味着 / 跟进方向 |
|---|---|---|---|
| **Java Deserialization** | `javax.faces.ViewState`（JSF ViewState） | 🟡 | 存在 Java 反序列化面 → 测 ViewState 反序列化 RCE |
| **Debug Logic Parameters** | 参数名像 `debug=` `test=` `admin=` `exec=` `shell=` 等 | 🔵 | 调试/危险功能参数 → 测调试开关、功能逻辑、命令执行 |
| **URL As A Value** | 参数值是 `http(s)://...` | 🔵 | **SSRF 核心信号**，参数值是 URL → 服务端可能拿去发请求。和 Auto-SSRF 看的是同一类点 |
| **Upload Form** | 响应体里 `type="file"` | 🟡 | 页面有上传表单 → 测文件上传漏洞 |
| **DoS Paramters** | 参数名像 `size=` `page=` `limit=` `count=` | 🔵 | 可控数量/分页参数 → 测超大值导致的资源耗尽 |
| **Passwd File** | 响应里出现 `/root:/bin/bash` | 🔴 | **已经读到 Linux `/etc/passwd`** → 文件读取/穿越/SSRF 已成功，强证据 |
| **Win.ini File** | 响应里出现 `for 16-bit app`（win.ini 特征） | 🔴 | **已经读到 Windows `win.ini`** → 同上，Windows 下的文件读取成功 |

> 注：红色那两条（Passwd / Win.ini）是"**结果**"而非"入口"——它们命中说明你某个 payload 已经成功读到系统文件了，看到就是实锤。

---

## 3. Basic Information（基础信息）—— 提取个人/网络信息

| 规则名 | 抓什么 | 颜色 | 命中意味着 / 跟进方向 |
|---|---|---|---|
| **Email** | 邮箱地址 | 🟡 | 账号枚举、钓鱼面、信息泄露取证 |
| **Chinese IDCard** | 身份证号 | 🟠 | 敏感个人信息泄露，可直接作为漏洞证据 |
| **Chinese Mobile Number** | 手机号 | 🟠 | 同上，PII 泄露 |
| **Internal IP Address** | 内网 IP（10./172./192.168.） | 🔵 | 泄露内网结构 → 辅助 SSRF 目标选择、资产测绘 |
| **MAC Address** | MAC 地址 | 🟢 | 设备信息泄露，内网信息收集用 |

---

## 4. Sensitive Information（敏感信息）—— 密钥、凭证、字段

这一类最值钱，命中往往直接等于一个可提交的"敏感信息泄露"，有些还能直接利用。

| 规则名 | 抓什么 | 颜色 | 命中意味着 / 跟进方向 |
|---|---|---|---|
| **Cloud Key** | 云服务 AccessKey（阿里云/AWS 等） | 🟡 | **高价值**，拿到 AK/SK 可能直接接管云资源 |
| **Windows File/Dir Path** | Windows 路径（`C:\...`） | 🟢 | 路径泄露 → 辅助文件读取、了解部署结构 |
| **Password Field** | 响应里的 `password` 字段 | 🟡 | 可能明文/弱加密传密码 → 查是否泄露凭证 |
| **Username Field** | 响应里的 `username` 字段 | 🟢 | 账号信息，配合枚举 |
| **WeCom Key** | 企业微信相关 Key | 🟢 | 凭证泄露，可能接管企微接口 |
| **JDBC Connection** | 数据库连接串（`jdbc:...`） | 🟡 | **高价值**，常含库地址+账号密码 → 可能直连数据库 |
| **Authorization Header** | `Authorization` 头内容 | 🟡 | 泄露 token/凭证 → 测重放、越权 |
| **Sensitive Field** | 泛化的敏感字段名 | 🟡 | 兜底规则，提醒这里有敏感数据 |
| **Mobile Number Field** | 手机号字段 | 🟢 | PII |
| **Userinfo In Link** | 链接里带的用户信息 | 🟢 | URL 泄露用户数据 → 越权测试线索 |
| **User Identity** | 用户身份标识 | 🟢 | 辅助越权/枚举 |

---

## 5. Other（其他）—— 做 JS 分析 / 资产梳理

这一类主要服务"信息收集深挖"，尤其是前后端分离站点挖 JS 里的接口。

| 规则名 | 抓什么 | 颜色 | 命中意味着 / 跟进方向 |
|---|---|---|---|
| **Linkfinder** | JS 里的接口路径/链接 | gray | **挖隐藏接口的主力** → 从 JS 里扒出后端 API，再逐个测 |
| **Source Map** | `.map` 源码映射 | pink | 前端源码泄露 → 可还原出完整源码 |
| **Create Script** | 动态创建的脚本 | 🟢 | 关注动态加载逻辑，XSS/依赖分析 |
| **URL Schemes** | 各种 URL 协议头 | 🟡 | 关注非 http 协议（如 `file://`）→ SSRF/协议利用线索 |
| **Router Push** | 前端路由跳转（`router.push`） | magenta | 梳理前端路由 → 发现隐藏页面/功能 |
| **All URL** | 所有 URL | gray | 全量链接提取，资产面铺开 |
| **Request URI** | 请求行里的 URI（**默认未开启**） | gray | 需手动勾选才生效，用于按请求路径提取 |
| **302 Location** | 302 跳转的 Location 头 | gray | 追踪跳转目标 → 开放重定向、逻辑绕过线索 |
| **OSKeys** | 操作系统相关 Key | gray | 系统信息收集 |

---

## 6. SSRF Param Name（SSRF 参数名）⭐ 你自己加的

"看名知意"——很多 SSRF 从参数名就能预判。这组盯**请求里的参数**。

| 规则名 | 抓什么 | 颜色 | 命中意味着 / 跟进方向 |
|---|---|---|---|
| **SSRF Param Name** | 参数名是 `url/uri/link/src/source/target/share/wap/u/display/domain/sourceURL/imageURL` | 🔵 | 参数可能被服务端拿去发请求 → 高度怀疑 SSRF，顺着测 |
| **SSRF Param Value Is URL** | 参数**值**直接是一个完整 `http(s)://` 地址 | 🟠 | **SSRF 最强信号** → 直接丢给 Auto-SSRF，或手动把值换成 Collaborator 地址看回连 |

> 原理：一个参数叫 `url`、`imageURL`、`source`，背后代码大概率就是"拿这个值去 curl / file_get_contents / request 一下"，这正是 SSRF 的发生点。参数值已经是完整 URL 的，更是直接信号。

---

## 7. Dangerous Path（危险路径关键词）⭐ 你自己加的

这组盯**请求行（URL 路径）**，按关键词分类染色，颜色直接对应漏洞方向。

| 规则名 | 抓什么（路径含） | 颜色 | 对应方向 |
|---|---|---|---|
| **Path - Admin Entry** | `admin / manage / console / backend` | 🔴 | 后台入口 → 未授权、弱口令、垂直越权 |
| **Path - Test or Deprecated** | `test / debug / dev / demo / old / bak / v1` | 🟡 | 测试/废弃接口 → 信息泄露、弱校验、功能越权 |
| **Path - File Read** | `file / download / export / read / path / doc / img` | 🟠 | 路径穿越、任意文件读取 |
| **Path - Upload** | `upload / avatar / import / attach` | 🟡 | 文件上传漏洞 |
| **Path - Command Exec** | `exec / cmd / run / shell / ping / convert / generate` | 🔴 | 命令注入 |
| **Path - Redirect or SSRF** | `redirect / jump / callback / next / forward` | 🔵 | 开放重定向、SSRF |
| **Sensitive Path or Files** | `/actuator / /swagger / /.git / /.env / /.svn / /api/internal` | 🔴 | 内部接口、调试端点、配置/版本控制文件 → 信息泄露、未授权 |

> ⚠️ 第 6、7 节这些"看名知意"规则**天生有误报**：`url=`、`/file/`、`admin` 在正常站点里也大量出现。它们是**提示你注意**，不是"命中即有洞"。其中 **SSRF Param Value Is URL** 最准，优先信它；哪条太吵就在规则列表里**取消左边的勾**（对应 `loaded:false`），不用删。

---

## 怎么用这张表（实战顺序）

1. **先开 HaE 再浏览**：HaE 只处理"加载之后新流经"的流量，不回溯旧流量。
2. **浏览完按颜色扫 history**：🔴🟠 优先，🟡🔵 次之，🟢⚪ 批量参考。
3. **点开标亮的报文看 MarkInfo**：这条具体提取到了什么。
4. **去 Databoard（数据看板）按域名汇总查**：某站点一共泄露了哪些敏感信息，一键查看。
5. **指纹类命中 → 查历史漏洞**；**敏感信息类命中 → 直接作为证据 + 尝试利用**；**Linkfinder → 扒接口再逐个测**；**第 6/7 节命中 → 按对应方向测（SSRF / 穿越 / 命令注入 / 未授权）**。

> 记忆窍门：你不用背规则名，只要形成"**颜色 → 漏洞类型**"的条件反射即可。红色看结果和高危组件/入口，橙黄看数据和可疑点，青色看 SSRF/重定向方向，灰色做资产梳理。

---

## 附：你当前规则清单（48 条）

| # | 分类 | 条数 | 是否你加的 |
|---|---|---|---|
| 1 | Fingerprint | 7 | 默认 |
| 2 | Maybe Vulnerability | 7 | 默认 |
| 3 | Basic Information | 5 | 默认 |
| 4 | Sensitive Information | 11 | 默认 |
| 5 | Other | 9 | 默认 |
| 6 | **SSRF Param Name** | 2 | ⭐ 新增 |
| 7 | **Dangerous Path** | 7 | ⭐ 新增 |

> 全表只有 **Other 里的 Request URI** 默认没开（`loaded:false`），其余 47 条都是开启状态。
