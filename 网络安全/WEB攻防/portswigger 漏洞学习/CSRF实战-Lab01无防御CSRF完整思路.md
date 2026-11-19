---
tags: [网络安全, WEB攻防, CSRF, portswigger, 实战]
---

# CSRF 实战 · Lab01:无防御的 CSRF —— 完整思路(含现实渗透视角)

> [!info] 🧭 知识导航
> 所属:[[portswigger 漏洞学习目录]] ｜ 上级:[[MOC - WEB攻防]]
> 🔗 相关:[[SSRF实战-过滤绕过与BlindSSRF-PortSwigger靶场笔记]](同系列靶场打法)
> 🔗 概念前置:CSRF 原理(三个必要条件、浏览器自动带 cookie)——见 CSRF 基础笔记

> 一句话总结:**这一关是最纯粹的 CSRF——目标站改邮箱功能没有任何防御(没 token、没 SameSite),你做一个"自动提交改邮箱表单"的页面,让登录着的受害者访问,他的浏览器就自动带着他的身份替你把邮箱改了。**
> 来源:PortSwigger Web Security Academy — Lab: CSRF vulnerability with no defenses(APPRENTICE)。
> 适用前提:仅限授权环境、靶场、CTF。
> **本篇整体定级**:🔴 核心。这是 CSRF 的地基,后面所有带防御的 CSRF lab 都是在这个流程上加东西让你绕。重点不是"过关",是**把"现实中怎么判断+怎么利用+怎么确认"的完整流程走通**。

> [!tip] 📊 分级图例
> 🔴 核心 —— 必须吃透,能独立复现 ｜ 🟡 重要 —— 懂原理会查 ｜ 🟢 了解 ｜ ⚪ 扫过

---

## 0. 靶场目标 vs 现实渗透(先建立映射)

**靶场说**:email change 功能有 CSRF,构造 HTML 改掉受害者邮箱,传到 exploit server。账号 `wiener:peter`。

**但靶场"作弊"了——它直接告诉你哪里有 CSRF。现实渗透里没人告诉你。** 所以这份笔记按"**你什么都不知道**"的现实视角走,靶场的每个部件对应现实的什么,先记住这张映射表:

| 靶场部件 | 现实渗透里对应什么 |
|---|---|
| "email change 有 CSRF"(lab 描述) | **没有**。你得自己判断哪个功能可能有 CSRF(§1) |
| exploit server + Body 框 | **你自己控制的一台服务器 / 一个恶意网页** |
| "Deliver exploit to victim" 按钮 | **社会工程**:钓鱼邮件/消息,诱导真实受害者点开你的链接 |
| Access log 里的 `(Victim)` | **带外确认 / 观察目标账户是否真被改**,确认攻击是否生效 |
| 改掉邮箱就通关 | 改邮箱只是**第一步**,现实里要延伸到账户接管(§5) |

---

## 1. 现实的起点:怎么判断"这里有 CSRF"🔴 核心 —— 这是靶场没教、现实最重要的一步

现实里你面对一个网站,没人告诉你哪有洞。你要**主动找"值得攻击、又没防护的状态改变操作"**。流程:

### 1.1 先找"有价值的状态改变操作"(对应条件①:A relevant action)

浏览网站,列出所有**会改变账户状态**的功能:改邮箱、改密码、改收货地址、转账、加管理员、改权限…… 这些是 CSRF 的候选目标。只读功能(看资料)不算——CSRF 偷不到数据,只能"替你做操作"。

### 1.2 抓这个操作的请求,做"三条件体检"🔴

对每个候选操作,操作一次、用 Burp 抓它的请求,逐条检查 CSRF 三要素(原理见 CSRF 基础笔记):

```
POST /my-account/change-email HTTP/1.1
Host: 0ac2001a....web-security-academy.net
Cookie: session=abc...                 ← 看这里
Content-Type: application/x-www-form-urlencoded

email=wiener@normal-user.com           ← 看这里
```

| 体检项 | 看什么 | 这一关的结果 |
|---|---|---|
| ① 有价值的操作 | 这操作值不值得攻击 | ✅ 改邮箱 → 能接管账户 |
| ② 纯 cookie 认证(条件②) | 身份识别是不是**只靠 Cookie**,没有别的 | ✅ 只有 `Cookie: session=`,没有自定义认证头 |
| ③ 无不可预测参数(条件③) | 请求里有没有**攻击者猜不到的值**(CSRF token、旧密码、随机数) | ✅ 只有一个 `email`,值你完全能控,**没有任何 token** |

**三条全 ✅ → 判定:这个操作可以 CSRF。** 这一步就是现实渗透里"发现 CSRF"的全部——没有魔法,就是**抓请求 + 对着三条件逐项核对**。

> 关键直觉:**CSRF 体检的核心是条件③**。看到请求里有一个像 `csrf=xxxx`、`token=xxxx` 的随机值,基本就没戏(除非该 token 有实现缺陷,那是后面几关的事)。这一关一个 token 都没有,所以"无防御"。

---

## 2. 构造 PoC:用一个 HTML 表单"复刻"这个请求 🔴 核心

CSRF 攻击的本质:**用一个 HTML 表单,重建你抓到的那个请求,让受害者的浏览器替你发出去。** 表单的每一块,对应请求的每一块:

```html
<form action="https://0ac2001a....web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com" />
</form>
<script>document.forms[0].submit();</script>
```

| 表单部件 | 复刻请求的哪一块 | 要点 |
|---|---|---|
| `action="https://目标站/my-account/change-email"` | 请求行的 URL(发给谁、哪个接口) | **域名必须是目标站(web-security-academy.net),不是你的服务器** |
| `method="POST"` | 请求方法 | 和抓到的请求**一致**,原来 POST 就 POST |
| `<input name="email" value="...">` | 请求体参数 | 参数名照抄,值填你想改成的(**要和受害者当前邮箱不同**) |
| `<script>...submit()</script>` | —— | 页面一加载**自动提交**,受害者不用点 |
| (没有 Cookie) | 请求里的 Cookie | **不用写**,受害者浏览器访问目标站时自动带上 |

**省事做法**:Burp 里对着抓到的改邮箱请求右键 → **Engagement tools → Generate CSRF PoC** → 自动生成上面这段。(它可能把 `@` 编码成 `&#64;`、`.` 编码成 `&#46;`,这是正常的 HTML 实体,浏览器会解析回 `@`/`.`,不用管。)

> 现实里,这段 HTML 就是你要挂到**自己服务器**上的恶意页面;靶场里,它填进 exploit server 的 **Body** 框。

---

## 3. 投递:靶场 exploit server 怎么用 🔴 核心(含踩坑)

exploit server 模拟"你的攻击者服务器返回一个网页"。一个网页响应 = **响应头(Head)+ 响应体(Body)**,别填错框:

- **Head 框**:只放响应头,保持默认两行即可:
  ```
  HTTP/1.1 200 OK
  Content-Type: text/html; charset=utf-8
  ```
- **Body 框**:放 §2 的那段 HTML。

然后底部按钮,**认准中文名**:

| 按钮 | 作用 | 什么时候用 |
|---|---|---|
| 存储 (Store) | 把当前内容保存到服务器 | 每次改完先点它 |
| 查看漏洞利用 (View exploit) | **你自己**访问这个页面 | 自测:看你**自己**邮箱会不会被改 |
| **将漏洞利用发送给受害者 (Deliver exploit to victim)** | 让 **PortSwigger 的受害者机器人**访问 | **← 通关靠这个** |
| 访问日志 (Access log) | 看谁访问过你的页面 | 排查投递有没有成功 |

**通关流程**:填好 Body → **存储** → **将漏洞利用发送给受害者** → 回 **lab 页面刷新**看是否"已解决"。

---

## 4. 怎么确认攻击成功 🔴(这次踩坑最多的地方)

### 4.1 看 Access log 判断"受害者到底来没来"

不确定投递成不成功,别盯状态条瞎猜,**看访问日志**。学会分辨日志里谁是谁:

```
221.155.18.125  GET /exploit  ...  Chrome/157 ... Edg/157      ← 你自己(带 Edg = Edge 浏览器)
10.0.4.217      GET /exploit/ ...  (Victim) ... Chrome/157     ← 受害者机器人(标着 Victim、无 Edg)
```

- 看到 **IP 不同、user-agent 标着 `(Victim)`** 的那条 `GET /exploit/ 200` → **受害者确实来访问了**,攻击触发了。
- (受害者加载页面后,表单自动 POST 到**目标站**改邮箱——那个 POST 发给 `web-security-academy.net`,**不会**出现在 exploit server 日志里,所以日志只到"受害者加载了页面"为止。对无防御这关,加载=提交=邮箱已改。)

> 现实对应:你没有 Access log,就靠**带外确认**(受害者浏览器会不会回连你埋的点)或**直接观察目标账户**(邮箱真被改了吗 / 能不能用新邮箱走密码重置)来确认。

### 4.2 通关状态在 lab 页面,不在 exploit server

**exploit server 页面顶部那个"未解决"状态条不会自动刷新**——它是你打开那一刻的旧快照。受害者来了、邮箱改了,你在 exploit server 页面还是看到"未解决"。
→ **切到 lab 描述页 / 购物站标签,按 F5 刷新**,才看得到变绿的"已解决 / Congratulations"。

---

## 5. 危害延伸:改邮箱不是终点,是账户接管的跳板 🟡 重要(渗透报告视角)

靶场改完邮箱就通关,但**现实渗透要说清"这个洞到底能造成多大危害"**,否则报告会被降级。改邮箱的真正杀伤链:

```
CSRF 改掉受害者邮箱 → 邮箱变成攻击者的
  → 攻击者在目标站点"忘记密码" → 重置链接发到攻击者邮箱
  → 攻击者重置密码 → 完全接管账户
  → 若受害者是管理员 → 接管整个应用
```

> 写报告时的"证明影响":不止说"能改邮箱",要说清**这条链**——一个无交互的 CSRF 如何升级到账户完全接管。这决定漏洞定级(高危/严重)。

---

## 6. 我这次踩的坑(实战避坑清单)⚪ 扫过 —— 对照着别再犯

| 坑 | 现象 | 正确做法 |
|---|---|---|
| HTML 填进了 **Head** 框 | 受害者拿到的页面还是默认内容,表单不执行 | HTML 放 **Body** 框,Head 保持默认两行 |
| `action` 指向了 **exploit server 自己**的域名 | 表单提交给了自己的服务器,目标邮箱不变 | `action` 必须指向**目标站** `web-security-academy.net` |
| 分不清 **View exploit** 和 **Deliver to victim** | 一直用 View(只改自己邮箱),以为失败 | 通关点 **Deliver exploit to victim** |
| 一直盯 exploit server 顶部状态条 | 显示"未解决",以为没过 | 通关状态在 **lab 页面**,F5 刷新才更新 |
| 不知道投递成没成功 | 反复瞎试 | 看 **Access log**,找标 `(Victim)` 的访问记录 |

> 规律:**"投递类"的 lab(CSRF、XSS exploit server 那类)卡住时,第一反应是看 Access log 确认受害者来没来,再看 lab 页面(不是 exploit server 页面)的真实状态。** 这个排查习惯比盯状态条靠谱得多。

---

## 7. 防御视角 + 这一关为什么"无防御" 🟢 了解

这一关能成,是因为 CSRF 三条件全满足、尤其**条件②③毫无防护**。现实中(和后面几关)会出现的防御:

- **CSRF token**:服务器给页面发一个随机值,改邮箱请求必须带上它。攻击者**猜不到、浏览器也不会自动带** → 破坏条件③,你构造不出有效请求。(下一关起的主题:token 存在但实现有缺陷,你去绕。)
- **SameSite cookie**:限制跨站请求自动带 cookie → 破坏条件②的利用基础。
- **验证 Referer/Origin**:检查请求是不是从本站发起的。

> 下一关 "How to deliver a CSRF exploit" / 带 token 的 lab,就是在这个流程上**加一层防御让你绕**——和你打 SSRF 绕过(黑/白名单)是一个节奏:先有纯净版,再逐关加防护练绕过。

---

## 8. 自测(先自己想,再看答案)

1. 现实里没人告诉你哪有 CSRF,你怎么判断一个操作能不能 CSRF?
2. PoC 表单里 `action` 为什么必须填目标站域名、不能填你自己服务器?为什么不用写 Cookie?
3. 投递了却显示"未解决",你会按什么顺序排查?
4. 改邮箱这个 CSRF,你在报告里怎么证明它的危害不止"改了个邮箱"?

<details>
<summary>参考答案</summary>

1. 抓这个操作的请求,做三条件体检:① 是不是有价值的状态改变;② 身份是不是**只靠 cookie**;③ 请求里有没有**攻击者猜不到的参数**(token/旧密码)。三条全中 → 可 CSRF,关键看第③条有没有 token。
2. action 填目标站,浏览器才会向目标站发请求、并自动带上受害者**在目标站**的 cookie;填自己服务器就提交给自己了,目标账户不变。Cookie 不用写,是因为浏览器访问目标站时会自动附上——这正是 CSRF 的原理。
3. ① 看 **Access log** 确认受害者(标 Victim 的 IP)来没来;② 来了就回 **lab 页面 F5 刷新**看真实状态(exploit server 状态条不刷新);③ 还不行就换个新 email 值重新 Store + Deliver;④ 确认 action 域名 = 当前 lab 域名。
4. 说清杀伤链:改邮箱 → 用"忘记密码"把重置链接发到攻击者邮箱 → 重置密码 → 完全接管账户 → 若受害者是管理员则接管整个应用。一个无交互 CSRF 升级到账户接管,属高危/严重。

</details>

## 9. 来源

- PortSwigger Web Security Academy — Lab: CSRF vulnerability with no defenses。
- 工具:Burp Suite(抓包、Generate CSRF PoC)、exploit server(Store / View exploit / Deliver to victim / Access log)。
