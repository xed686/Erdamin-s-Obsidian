# DOM / BOM 基础与基于 DOM 的漏洞笔记

> JS 安全结合点（发现敏感信息、危险代码、逻辑校验等审计思路）已在《JavaScript安全分析与Ajax技术笔记》中详细整理过，此处不再重复，仅聚焦 **DOM / BOM 的概念**与**基于 DOM 的漏洞类型**这两块新内容。

学习资料：原生 JS 教程 https://www.w3school.com.cn/js/index.asp

---

## 一、DOM（Document Object Model，文档对象模型）

DOM 是浏览器把 HTML 页面解析成的一棵**可被 JS 操作的树状结构**，JS 通过 DOM 能做三类事：

| 能力 | 说明 |
|---|---|
| 访问文档 | 动态获取和修改页面上已有的内容 |
| 修改文档结构 | 添加、删除、移动或替换页面元素 |
| 处理事件 | 为页面元素绑定并响应交互事件（点击、悬停等） |

---

## 二、BOM（Browser Object Model，浏览器对象模型）

BOM 是 JS 用来操作**浏览器本身**（而不是页面内容）的一套对象体系：

| 对象 | 作用 |
|---|---|
| `Window` | 操作浏览器窗口的打开、关闭、返回、新建等 |
| `Screen` | 包含客户端显示屏的相关信息 |
| `Navigator` | 浏览器对象本身，包含浏览器名称、版本等信息 |
| `Location` | 包含当前页面 URL 的相关信息，常用于页面跳转 |
| `History` | 包含用户访问过的 URL 历史，常用于前进/后退式页面跳转 |
| `Document` | 文档对象，**既属于 BOM，也属于 DOM**（是两者的交汇点） |

> 审计意义：很多 DOM-based 漏洞的触发点，正是通过 BOM 对象（尤其是 `Location`、`Document`）把"用户可控的浏览器端数据"引入到了"会被当作代码/HTML/URL 执行"的位置。

---

## 三、基于 DOM 的漏洞类型一览表

基于 DOM 的漏洞（DOM-based Vulnerabilities）统一的成因模式是：**用户可控的输入（Source，如 URL、`document.referrer`、`localStorage` 等）未经处理，直接流入了某个危险的"接收点"（Sink），被当作代码、HTML 或指令执行。**

下表中"接收器示例"列出的就是常见的 **Sink 函数/属性** —— 审计 JS 代码时，可以直接搜索这些关键词，再反向追踪它们的参数是否来自用户可控的输入源。

| 基于 DOM 的漏洞类型 | 接收器（Sink）示例 |
|---|---|
| DOM 型跨站攻击（DOM XSS） | `document.write()` |
| 打开重定向（Open Redirection） | `window.location` |
| 操纵 Cookie（Cookie Manipulation） | `document.cookie` |
| JS 注入（JavaScript Injection） | `eval()` |
| 文档域操作（Document-domain Manipulation） | `document.domain` |
| WebSocket-URL 中毒（WebSocket-URL Poisoning） | `WebSocket()` |
| 操纵链接（Link Manipulation） | `element.src` |
| 操纵网络消息（Web Message Manipulation） | `postMessage()` |
| Ajax 请求头操作（Ajax Request-header Manipulation） | `setRequestHeader()` |
| 本地文件路径操作（Local File-path Manipulation） | `FileReader.readAsText()` |
| 客户端 SQL 注入（Client-side SQL Injection） | `ExecuteSql()` |
| HTML5 存储操作（HTML5-storage Manipulation） | `sessionStorage.setItem()` |
| 客户端 XPath 注入（Client-side XPath Injection） | `document.evaluate()` |
| 客户端 JSON 注入（Client-side JSON Injection） | `JSON.parse()` |
| DOM 数据操作（DOM-data Manipulation） | `element.setAttribute()` |
| 拒绝服务（Denial of Service） | `RegExp()` |

### 审计思路小结
1. 先在前端 JS 代码中**搜索上表的 Sink 函数/属性**（如 `grep` 关键词 `document.write|eval(|innerHTML|location =` 等）。
2. 找到 Sink 后，**向上追溯这个参数的来源**——是不是来自 URL（`location.search`/`location.hash`）、`document.referrer`、`postMessage` 接收到的消息、本地存储等用户可控的 Source。
3. 若 Source → Sink 之间**没有经过任何过滤/编码处理**，就构成了对应类型的 DOM 漏洞。
4. 不同 Sink 对应不同的漏洞类型，但排查方法完全一致，只是"危险操作"的具体表现形式不同（执行代码、跳转、污染存储、注入查询语句等）。

---

## 四、案例

1. **DOM-XSS / URL 重定向等**
   参考：https://mp.weixin.qq.com/s/iUlMYdBiOrI8L6Gg2ueqLg

2. **代码审计：DOM-XSS**
   参考：https://xz.aliyun.com/t/12499

---

## 五、总结

1. DOM 负责"页面内容"，BOM 负责"浏览器本身"，`Document` 对象是两者的交界点，也是很多 DOM 漏洞的高发位置。
2. 基于 DOM 的漏洞本质都是 **Source（用户可控输入）→ Sink（危险接收点）** 这一条未经处理的数据流，审计思路统一：先定位 Sink，再反查 Source，核心看中间有没有过滤。
3. 该表格可以直接当作关键词速查表使用：遇到陌生 Sink 函数时，先查表确认它对应哪类漏洞，再按上面的审计步骤走。
[[02 前端加密与代码混淆笔记]]