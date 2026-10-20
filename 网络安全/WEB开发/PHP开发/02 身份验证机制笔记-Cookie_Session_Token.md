#  身份验证机制笔记：数据库 / Cookie / Session / Token

## 一、五种身份验证实现方式（代码审计练习脉络）

| 序号 | 验证方式 | 涉及文件流转 |
|---|---|---|
| 1 | 数据库操作验证用户 | `login.php` → `index.php` |
| 2 | Cookie 验证后台登录 | `loginc.php` → `indexc.php` → `loginc_out.php` |
| 3 | Session 验证后台登录 | `logins.php` → `indexs.php` → `logins_out.php` |
| 4 | Session + Token 防爆破登录 | `loginst.php` → `logincheck.php` → `indexst.php` → `loginst_out.php` |
| 5 | 代码审计实战案例 | XHCMS（Cookie 脆弱）、YXCMS（Session 固定） |

> 这五个层级本质上是身份验证方案的演进路径：从最原始的"数据库比对"，到"客户端凭证"（Cookie），到"服务端会话"（Session），再到"Session + Token 双因子"用于防御暴力破解。审计时应重点关注每一级新增的验证逻辑是否被正确实现，而不是简单存在。

---

## 二、Cookie 身份验证

### 1. 工作原理

```
Client                              Server
  |--- 1. HTTP request -------------->|
  |<-- 2. Set-Cookie ------------------|
  |    3. Save cookie                 |
  |--- 4. HTTP request (带Cookie) --->|
  |    5. Read cookie                 |
  |    6. Process request             |
  |<-- 7. HTTP response ---------------|
```

**流程说明：**
1. 客户端向服务器发送 HTTP 请求。
2. 服务器检查请求头中是否包含 Cookie 信息。
3. 若包含则用其识别客户端；若不包含则生成一个新的 Cookie。
4. 服务器在响应头（`Set-Cookie`）中下发 Cookie。
5. 客户端接收并将 Cookie 保存在本地。
6. 客户端后续请求会自动携带该 Cookie。
7. 服务器每次收到请求都会校验 Cookie 的有效性，无效则要求重新登录。

### 2. 相关函数

| 函数/变量         | 作用                             |     |
| ------------- | ------------------------------ | --- |
| `$_COOKIE`    | 关联数组，包含客户端传递给当前脚本的所有 Cookie    |     |
| `setcookie()` | 设置一个 Cookie 并发送到客户端浏览器         |     |
| `unset()`     | 删除指定的 Cookie（需配合设置过期时间清除客户端副本） |     |

### 3. 安全审计要点（从渗透测试角度）
- **明文/弱加密存储敏感信息**：若 Cookie 中直接存放用户名、管理员标识、权限等级（如 `is_admin=1`），可被客户端篡改，属于典型的"Cookie 脆弱"漏洞（对应下文 XHCMS 案例）。
- **缺少 `HttpOnly` 标志**：未设置会导致 Cookie 可被 JavaScript 读取，增加 XSS 窃取会话的风险。
- **缺少 `Secure` 标志**：Cookie 可能在 HTTP 明文信道中被中间人截获。
- **缺少 `SameSite` 属性**：增加 CSRF 攻击面。
- **可预测/可爆破的 Cookie 值**：若身份凭证是简单规律的字符串（如用户 ID 拼接），可被枚举伪造。

---

## 三、Session 身份验证

### 1. 工作原理

```
Client                                          Server
  |--- 1. Sends HTTP Request --------------------->|
  |<-- 2. 生成唯一 session ID 并存储于服务端 --------|
  |<-- 3. HTTP Response，session ID 作为 Cookie 下发-|
  |    4. Client 将 session ID 作为 Cookie 保存 ---- |
```

**存储对比：**
- Session ID 存储：**客户端**（以 Cookie 形式）
- Session 数据：**服务端**（文件、数据库、Redis 等）

### 2. 流程说明
1. 客户端发送 HTTP 请求。
2. 服务器为客户端生成唯一的 Session ID，并将会话数据存储在服务端存储器中。
3. 服务器将 Session ID 作为 Cookie 发送给客户端。
4. 客户端保存该 Cookie，后续请求自动携带。
5. 服务器通过 Session ID 检索对应的会话数据，实现状态保持。

### 3. 相关函数

| 函数/变量               | 作用                                    |
| ------------------- | ------------------------------------- |
| `session_start()`   | 启动或恢复一个已存在的会话                         |
| `$_SESSION`         | 关联数组，包含当前脚本的所有 Session 内容             |
| `session_destroy()` | 销毁当前会话中的所有数据                          |
| `session_unset()`   | 释放当前会话中的所有变量                          |
| Session 存储路径        | 由 `php.ini` 中的 `session.save_path` 设置 |

### 4. 安全审计要点
- **Session 固定（Session Fixation）**：若登录成功后未调用 `session_regenerate_id()` 重新生成 Session ID，攻击者可预先诱导受害者使用已知的 Session ID 登录，从而劫持会话（对应下文 YXCMS 案例）。
- **Session 劫持**：Session ID 一旦泄露（如通过 Referer、URL 参数传递），等同于账号被接管。
- **服务端存储路径可被读取/写入**：若 `session.save_path` 可被文件包含（如 LFI），配合 Session 文件写入可能导致代码执行。
- **未校验 Session 归属**：仅校验 Session 是否存在而不校验对应用户角色，可能导致越权。

---

## 四、Token 身份验证（防爆破 / 唯一性判断）

### 使用要点
1. 生成 Token 并将其存储在 Session 中（服务端持有比对基准）。
2. 生成 Token 并绑定到 Cookie 中触发下发。
3. 登录表单提交时携带 Token，服务端校验一致性后才放行登录逻辑。
4. **安全特性思考**：
   - Token 应具备**一次性**（提交后立即失效，防止重放）。
   - Token 应具备**随机性**（避免可预测，防止伪造）。
   - Token 应设置**有效期**（防止长期有效被截获后重放）。
   - 主要用途是**防止登录接口被暴力破解/自动化脚本批量提交**，本质类似验证码/CSRF Token 的思路，而非替代完整身份认证。

---

## 五、机制对比

### 1. Cookie vs Session

| 维度 | Cookie | Session |
|---|---|---|
| 存储位置 | 客户端（浏览器） | 服务端 |
| 安全性 | 较低，易被窃取/篡改 | 较高，敏感数据留在服务端 |
| 存储容量 | 有限，约 4KB | 理论无限制，取决于服务端配置 |
| 生命周期 | 可设置长期有效，浏览器关闭后仍可能存在 | 默认浏览器关闭即失效（依赖 Session Cookie） |
| 访问方式 | 可被 JavaScript 访问（除非设置 HttpOnly） | 仅服务端可访问 |
| 典型场景 | 存储少量非敏感信息（如用户名、偏好设置） | 存储购物车、登录状态等重要数据 |

**选型建议**：涉及敏感信息或数据量较大时优先使用 Session；仅需少量数据且需要客户端读取时可用 Cookie。

### 2. Token 机制 vs Session 机制

| 维度     | Token 机制                              | Session 机制                        |
| ------ | ------------------------------------- | --------------------------------- |
| 身份验证方式 | 登录后下发 Token，每次请求携带校验                  | 登录后创建 Session，请求携带 Session ID 校验  |
| 服务端存储  | 通常无需存储完整登录状态（如 JWT 自包含），仅需存储/校验 Token | 需在服务端持久化保存会话状态                    |
| 安全性    | Token 泄露一般不直接暴露密码等敏感信息                | 服务端一旦被攻破，可能连带泄露大量用户会话/敏感数据        |
| 跨域支持   | 通过 HTTP 头 `Authorization` 传递，天然适合跨域   | 依赖 Cookie 传递，跨域需额外处理（CORS + 凭证配置） |
| 实现复杂度  | 需自行实现签发、校验、刷新、吊销逻辑，相对复杂               | 框架原生支持，相对简单                       |

**结论**：Token 机制安全性与跨域适配更优，但实现成本更高；Session 机制简单但集中式存储风险更高。具体选型需结合业务架构（单体 vs 前后端分离/微服务）权衡。

---

## 六、代码审计实战案例

### 1. XHCMS — Cookie 脆弱
- **漏洞本质**：身份/权限信息直接存储在客户端 Cookie 中且缺乏签名校验，攻击者可直接修改 Cookie 值伪造管理员身份。
- **审计思路**：定位登录成功后 `setcookie()` 写入的字段 → 检查后台鉴权逻辑是否仅依赖该 Cookie 字段的存在性/取值，而未做服务端二次校验或签名验证。

### 2. YXCMS — Session 固定
- **漏洞本质**：登录流程中未在身份确认后重新生成 Session ID，导致攻击者可预先设置目标的 Session ID 并诱导其登录，从而在登录后复用同一 Session ID 完成会话劫持。
- **审计思路**：定位登录成功后是否调用 `session_regenerate_id(true)`；若缺失，则登录前后 Session ID 保持不变即为漏洞点。

**参考资料**：https://xz.aliyun.com/t/2025

---

## 七、审计速查清单（渗透测试视角总结）

- [ ] Cookie 中是否存储了可被篡改利用的敏感/权限字段？
- [ ] Cookie 是否设置了 `HttpOnly` / `Secure` / `SameSite`？
- [ ] Session ID 是否在权限变更（登录成功）时重新生成？
- [ ] Session 数据的服务端校验是否与 Cookie/Token 校验存在逻辑割裂（只判断存在不判断归属）？
- [ ] Token 是否具备随机性、一次性、有效期三要素？
- [ ] 登录接口在 Session+Token 双因子下是否仍可被爆破（如 Token 校验可被绕过或未生效）？
[[鉴权与身份验证技术全景笔记]]
[[03 PHP弱类型比较漏洞笔记]]