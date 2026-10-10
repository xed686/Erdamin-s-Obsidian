# JavaScript 零基础学习手册（安全视角版）

> **适合人群**：完全没接触过 JS / HTML 的初学者。
> **学习目标**：能读懂前端代码；理解 HTML、DOM、BOM 各自的作用；掌握前端相关漏洞（XSS、DOM 型漏洞、原型链污染、开放重定向等）的成因与防御；能用浏览器开发者工具分析网站前端 JS。
> **合规提醒**：本文所有攻击示例仅用于在**自己搭建的靶场**或**获得明确授权的目标 / Bug Bounty 范围内**学习与测试，未授权测试他人系统是违法的。

---

## 目录

1. [先建立全局认知：浏览器是怎么工作的](#1-先建立全局认知浏览器是怎么工作的)
2. [HTML：标签与属性（安全重点）](#2-html标签与属性安全重点)
3. [JavaScript 语言基础（每节带安全提示）](#3-javascript-语言基础每节带安全提示)
4. [DOM：JS 操作页面的接口](#4-domjs-操作页面的接口)
5. [BOM：JS 操作浏览器的接口](#5-bomjs-操作浏览器的接口)
6. [浏览器安全机制](#6-浏览器安全机制)
7. [前端常见漏洞详解](#7-前端常见漏洞详解)
8. [实战：用 DevTools 分析网站前端 JS](#8-实战用-devtools-分析网站前端-js)
9. [延伸：Node.js 与 Android WebView 中的 JS](#9-延伸nodejs-与-android-webview-中的-js)
10. [练习路线与靶场](#10-练习路线与靶场)
11. [速查表与术语表](#11-速查表与术语表)

---

## 1. 先建立全局认知：浏览器是怎么工作的

### 1.1 一次访问发生了什么

```
输入 URL
  → DNS 解析出 IP
  → TCP 连接 (+ TLS 握手，如果是 https)
  → 发送 HTTP 请求
  → 服务器返回响应（HTML / JS / CSS / JSON / 图片 ...）
  → 浏览器解析 HTML，构建 DOM 树；解析 CSS，构建样式
  → 遇到 <script> 就下载并执行 JS
  → JS 可以修改 DOM、发起新请求、读写 Cookie / 存储
  → 最终渲染出页面
```

### 1.2 前端三件套

| 技术 | 作用 | 类比 |
|------|------|------|
| **HTML** | 页面的结构和内容 | 房子的骨架 |
| **CSS** | 页面的外观样式 | 装修 |
| **JavaScript** | 页面的行为和交互 | 电路和开关 |

### 1.3 JS 的三个组成部分（理解 DOM / BOM 的前提）

| 组成 | 全称 | 作用 | 核心对象 |
|------|------|------|----------|
| **ECMAScript** | 语言标准 | 语法本身：变量、函数、对象、Promise… | `Object`、`Array`、`Promise`、`JSON`… |
| **DOM** | Document Object Model | 操作**网页内容**（标签、文字、属性） | `document` |
| **BOM** | Browser Object Model | 操作**浏览器本身**（地址栏、历史、存储、请求） | `window`、`location`、`navigator`… |

> 一句话记忆：**DOM 管页面里的东西，BOM 管页面外的东西**。`window` 是 BOM 的顶层对象，`document` 是 DOM 的入口，而 `document` 本身又挂在 `window` 下（`window.document`）。

### 1.4 安全视角的三条底层认知

1. **前端代码完全暴露在用户手里**：用户（包括攻击者）可以查看、修改、断点调试、重放任何前端代码。所以前端校验、前端加密、前端权限判断**都不是安全边界**，只有服务端校验才是。
2. **一切来自用户的数据都不可信**：URL 参数、表单、Cookie、请求头、`postMessage`、本地存储……
3. **漏洞的本质 = 不可信数据（Source）流入了危险函数（Sink），且中间没有正确处理**。

### 1.5 Source / Sink 模型（贯穿全文）

| 概念 | 含义 | 前端举例 |
|------|------|----------|
| **Source（污染源）** | 攻击者能控制的数据入口 | `location.search`、`location.hash`、`document.referrer`、`window.name`、`document.cookie`、`localStorage`、`postMessage` 的 `event.data`、接口返回的数据 |
| **Sink（危险函数）** | 数据流到这里会产生危害 | `innerHTML`、`outerHTML`、`document.write`、`eval`、`setTimeout(字符串)`、`new Function()`、`location.href = ...`、`element.src = ...`、jQuery 的 `$()` / `.html()` |

> 做 JS 代码审计时，本质就是：**找 Sink → 往回追踪数据是否来自 Source → 中间有没有过滤/编码**。

---

## 2. HTML：标签与属性（安全重点）

### 2.1 基本结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>页面标题</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <h1>你好</h1>
  <script src="app.js"></script>
</body>
</html>
```

- `<head>`：元信息（编码、标题、样式、CSP 策略等），不直接显示。
- `<body>`：页面可见内容。
- **标签**（tag）：`<p>`；**属性**（attribute）：`<a href="...">` 中的 `href`；**元素**（element）：开始标签 + 内容 + 结束标签的整体。

### 2.2 常用标签速览

| 类别 | 标签 | 说明 |
|------|------|------|
| 文本 | `h1`~`h6`、`p`、`span`、`div`、`br`、`strong`、`em` | `div` 块级容器，`span` 行内容器 |
| 链接 | `a` | 超链接 |
| 图片 | `img` | 图片 |
| 列表 | `ul`、`ol`、`li` | 无序 / 有序列表 |
| 表格 | `table`、`tr`、`td`、`th` | 表格 |
| 表单 | `form`、`input`、`textarea`、`select`、`button`、`label` | 向服务器提交数据 |
| 多媒体 | `audio`、`video`、`source` | 音视频 |
| 嵌入 | `iframe`、`embed`、`object` | 嵌入其他页面或内容 |
| 脚本 / 样式 | `script`、`style`、`link` | 引入 JS / CSS |
| 元信息 | `meta`、`base`、`title` | 页面配置 |
| 图形 | `svg`、`canvas` | 矢量图 / 画布（`svg` 里也能执行 JS） |

### 2.3 通用属性（几乎所有标签都能用）

| 属性 | 作用 | 备注 |
|------|------|------|
| `id` | 元素唯一标识 | JS 常用 `getElementById`；**还会产生 DOM Clobbering**（见 4.8） |
| `class` | 样式分类 | 选择器常用 |
| `name` | 名称 | 表单字段名；也可能被 `document.xxx` 引用 |
| `style` | 内联样式 | 注入 CSS 的入口之一 |
| `hidden` | 隐藏元素 | 只是视觉隐藏，源码里仍可见 |
| `data-*` | 自定义数据 | 前端框架常用，也常被 JS 当作数据源 |
| `on*`（事件属性） | 事件触发时执行 JS | **XSS 重灾区**，见 2.5 |

### 2.4 安全重点：标签与属性的作用及风险

| 标签 | 关键属性 | 作用 | 安全风险 / 注意点 |
|------|----------|------|-------------------|
| `<a>` | `href` | 跳转目标 | `href="javascript:alert(1)"` 点击即执行 JS；开放重定向；`target="_blank"` 且无 `rel="noopener"` 时，新页面可通过 `window.opener` 操控原页面（现代浏览器已默认按 noopener 处理，但老环境仍有） |
| `<a>` | `rel` | 链接关系 | `noopener` / `noreferrer`（不带 Referer，防泄露 URL 里的敏感参数） |
| `<img>` | `src` | 图片地址 | 常用作 **XSS 探测载体**：`<img src=x onerror=alert(1)>`；`src` 可触发对内网/外部的请求（SSRF/信息外带的客户端形态） |
| `<script>` | `src` | 引入外部 JS | 引入第三方 JS = 信任该第三方（供应链风险）；可用 `integrity`（SRI）校验 |
| `<script>` | `nonce` / `integrity` / `defer` / `async` | CSP 随机数 / 完整性校验 / 延迟 / 异步执行 | CSP 配合使用 |
| `<iframe>` | `src` / `srcdoc` | 嵌入页面 | 点击劫持；`srcdoc` 可内联 HTML；`javascript:` 作为 `src` 可执行脚本 |
| `<iframe>` | `sandbox` | 沙箱限制 | 防御属性：限制脚本、表单、弹窗、同源等 |
| `<form>` | `action` / `method` / `enctype` | 提交地址 / 方式 / 编码 | 表单提交是 CSRF 的载体；`action` 可被注入改写到攻击者服务器 |
| `<input>` | `type` / `value` / `name` | 输入框 | `type=hidden` 的字段用户可改；`value` 输出时未转义会逃逸属性 |
| `<input>` | `maxlength` / `readonly` / `disabled` / `pattern` / `required` | 前端限制 | **全部可被绕过**（改 DOM / 抓包），不能作为安全控制 |
| `<input>` | `autocomplete` | 自动填充 | 密码框可控制是否被浏览器记住 |
| `<input type=file>` | `accept` | 限制文件类型 | 仅前端提示，可绕过，服务端必须校验（文件上传漏洞） |
| `<meta>` | `http-equiv="refresh"` | 自动跳转 | 可用于跳转到恶意页面 |
| `<meta>` | `http-equiv="Content-Security-Policy"` | 设置 CSP | 防御；也可能因配置不当被绕过 |
| `<meta>` | `charset` | 字符编码 | 编码不一致可导致绕过（如 UTF-7 历史问题） |
| `<base>` | `href` | 改变页面所有相对路径的基准 | **注入 `<base href=//evil>` 后，所有相对路径的 JS/资源都从攻击者服务器加载** |
| `<link>` | `rel` / `href` | 引入 CSS、预加载等 | `rel=import`（已废弃）、`prefetch`；CSS 注入可造成信息泄露 |
| `<object>` / `<embed>` | `data` / `src` | 嵌入外部内容（Flash 等遗留） | 可加载 SVG/HTML 执行脚本 |
| `<svg>` | `onload` 等 | 矢量图 | `<svg onload=alert(1)>`；SVG 文件上传可造成存储型 XSS |
| `<video>` / `<audio>` | `src` / `onerror` | 音视频 | 同 `img`，可用 `onerror` 触发 |
| `<style>` | — | 内联样式 | CSS 注入可通过属性选择器逐字符窃取页面数据 |
| `<textarea>` / `<title>` | — | 内容按文本解析 | 注入时需要先闭合标签 `</textarea>` 再插入脚本 |

### 2.5 事件属性（On-event handlers）

事件属性的值就是一段 JS 代码，事件触发就执行。

```html
<button onclick="alert('点击了')">点我</button>
<img src="x" onerror="alert('图片加载失败')">
<body onload="init()">
<input onfocus="alert(1)" autofocus>   <!-- autofocus 让它无需用户操作自动触发 -->
```

常见事件属性（做 XSS 测试时经常用到）：

| 事件 | 触发时机 | 是否需要用户交互 |
|------|----------|------------------|
| `onerror` | 资源加载失败（`img`、`script`、`video`…） | 否（`src` 写个错的地址即可） |
| `onload` | 资源/页面加载完成（`body`、`svg`、`iframe`、`img`） | 否 |
| `onfocus` + `autofocus` | 获取焦点 | 否 |
| `onclick` / `ondblclick` | 点击 | 是 |
| `onmouseover` / `onmouseenter` | 鼠标移入 | 弱交互 |
| `onanimationstart` / `ontoggle`（`<details open>`） | 动画/展开 | 否 |

> **为什么这是重点？** 即使网站过滤了 `<script>`，只要还能注入一个带事件属性的标签，往往仍能执行 JS。所以防御的核心是**对输出做正确编码**，而不是简单黑名单。

### 2.6 表单与 HTTP 请求的对应关系

```html
<form action="/login" method="POST">
  <input name="username" value="">
  <input name="password" type="password">
  <input name="role" type="hidden" value="user">
  <button type="submit">登录</button>
</form>
```

提交后浏览器发出的请求：

```
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=xxx&password=xxx&role=user
```

- `method="GET"`：参数拼在 URL 上（会进浏览器历史、日志、Referer，不适合敏感数据）。
- `method="POST"`：参数在请求体。
- `enctype`：`application/x-www-form-urlencoded`（默认）、`multipart/form-data`（上传文件）、`text/plain`。
- **安全点**：
  - 隐藏字段 `role=user` 可被改成 `role=admin`（越权测试经典点）。
  - 没有 CSRF Token 的 POST 表单可能被跨站伪造提交。
  - 前端 `maxlength`、`pattern` 等限制抓包后全部无效。

### 2.7 HTML 实体与输出编码（防御 XSS 的核心）

HTML 中有特殊含义的字符，需要转成"实体"才能被当作普通文本显示：

| 字符 | 实体 |
|------|------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `&` | `&amp;` |
| `"` | `&quot;` |
| `'` | `&#x27;` 或 `&#39;` |

**关键概念：输出位置（上下文）不同，需要的编码方式不同。**

| 输出位置 | 例子 | 逃逸方式 | 正确防御 |
|----------|------|----------|----------|
| HTML 标签内容 | `<div>用户输入</div>` | 注入 `<script>`、`<img onerror>` | HTML 实体编码 |
| HTML 属性值 | `<input value="用户输入">` | 用 `"` 闭合属性后加事件 | 属性编码（务必给属性加引号 + 转义引号） |
| JS 字符串内 | `var a = '用户输入';` | 用 `'` 闭合后接语句 `';alert(1);//` | JS 字符串转义 / 用 JSON 序列化输出 |
| URL 内 | `<a href="用户输入">` | `javascript:` 伪协议 | 校验协议白名单（仅 http/https）+ URL 编码 |
| CSS 内 | `<style>body{color:用户输入}` | 注入 CSS 规则 | CSS 转义 / 避免动态输出 |

### 2.8 实战理解：四种最典型的注入上下文

假设某页面把参数 `q` 直接输出：

```html
<!-- ① 标签之间 -->
<div>搜索结果：{q}</div>
<!-- q = <img src=x onerror=alert(1)> -->

<!-- ② 属性内 -->
<input value="{q}">
<!-- q = " onfocus="alert(1)" autofocus x=" -->
<!-- 结果：<input value="" onfocus="alert(1)" autofocus x=""> -->

<!-- ③ JS 字符串内 -->
<script>var keyword = '{q}';</script>
<!-- q = ';alert(1);// -->
<!-- 结果：var keyword = '';alert(1);//'; -->

<!-- ④ href 内 -->
<a href="{q}">点击</a>
<!-- q = javascript:alert(1) -->
```

> 记住：**先判断输出落在哪个上下文，再决定怎么逃逸（或怎么防御）**。这是 XSS 学习的第一性原理。

---

## 3. JavaScript 语言基础（每节带安全提示）

### 3.1 JS 代码在哪里运行、怎么引入

```html
<!-- ① 内联脚本 -->
<script>
  console.log("hello");
</script>

<!-- ② 外部脚本 -->
<script src="/static/app.js"></script>

<!-- ③ 事件属性 -->
<button onclick="alert(1)">x</button>

<!-- ④ javascript: 伪协议 -->
<a href="javascript:alert(1)">x</a>
```

- **浏览器控制台**（F12 → Console）：可以直接输入并执行 JS，是最好的练习场。
- **Node.js**：在服务器/命令行运行 JS（见第 9 章）。

> 安全提示：以上 ①~④ 四种执行方式，同时也是 XSS 攻击者想"落地"的四种位置。CSP（第 6 章）就是通过限制这些来源来防御的。

### 3.2 变量与常量

```js
var a = 1;      // 旧写法：函数作用域，存在变量提升，可重复声明
let b = 2;      // 块级作用域，可重新赋值
const c = 3;    // 块级作用域，不可重新赋值（但对象内容可改）
```

- `var` 的**变量提升**：声明会被提升到作用域顶部，但赋值不会。

```js
console.log(x);  // undefined（不会报错）
var x = 10;
```

> 安全提示：全局变量（直接写在最外层的 `var`、函数声明）会挂到 `window` 上，可被其他脚本甚至注入的代码读写、覆盖。逆向时可以直接在控制台访问页面的全局变量/函数。

### 3.3 数据类型

| 类型 | 例子 | 说明 |
|------|------|------|
| `number` | `1`、`3.14`、`NaN`、`Infinity` | 数字 |
| `string` | `"abc"`、`'abc'`、`` `abc${x}` `` | 字符串（反引号为模板字符串） |
| `boolean` | `true` / `false` | 布尔 |
| `undefined` | `undefined` | 未赋值 |
| `null` | `null` | 空值 |
| `object` | `{a:1}`、`[1,2]`、函数 | 引用类型（数组、函数也是对象） |
| `symbol` / `bigint` | — | 进阶，暂可略过 |

```js
typeof 123        // "number"
typeof "a"        // "string"
typeof null       // "object"  ← 历史遗留 bug
typeof [], typeof {}  // 都是 "object"
Array.isArray([]) // true
```

### 3.4 类型转换与 `==` 的陷阱（安全相关）

```js
"1" == 1          // true   （== 会自动转换类型）
"1" === 1         // false  （=== 严格相等，类型也要相同）
null == undefined // true
0 == false        // true
"" == false       // true
"0" == false      // true
[] == false       // true
NaN == NaN        // false  （NaN 不等于任何值，包括自己）
"10" + 1          // "101"  （字符串拼接）
"10" - 1          // 9      （数值运算）
```

> 安全提示：
> - **松散比较绕过**：服务端 / 前端用 `==` 判断权限或 Token 时，可能被类型转换绕过（PHP 同理，有"弱类型"问题）。
> - JSON 里把 `"admin": "false"` 当布尔用，字符串 `"false"` 在条件判断中是 **truthy**（真值），这是常见逻辑漏洞来源。
> - 假值（falsy）一共这些：`false`、`0`、`""`、`null`、`undefined`、`NaN`（以及 `-0`、`0n`）。其余都是真值，包括 `"0"`、`"false"`、`[]`、`{}`。

### 3.5 运算符与流程控制

```js
// 条件
if (score >= 90) { ... } else if (score >= 60) { ... } else { ... }

// 三元
const level = age >= 18 ? "adult" : "minor";

// switch
switch (role) {
  case "admin": ...; break;
  default: ...;
}

// 循环
for (let i = 0; i < 5; i++) { ... }
while (cond) { ... }
for (const item of [1,2,3]) { ... }      // 遍历值
for (const key in obj) { ... }           // 遍历键（会遍历到原型链上的可枚举属性！）
```

逻辑运算符的"短路"特性常见于代码里：

```js
const name = user && user.name;     // user 为真才取 name
const port = config.port || 8080;   // 左边为假值则用右边
const x = a ?? "default";           // 仅当 a 为 null/undefined 时才用默认值
const city = user?.address?.city;   // 可选链，中间为 null/undefined 不报错
```

### 3.6 函数

```js
// 函数声明
function add(a, b) { return a + b; }

// 函数表达式
const add2 = function (a, b) { return a + b; };

// 箭头函数
const add3 = (a, b) => a + b;

// 立即执行函数（IIFE）——打包后的代码和混淆代码里极其常见
(function () {
  var secret = 123;   // 外部访问不到
})();
```

- 函数是**一等公民**：可以赋值给变量、作为参数传递、作为返回值。回调函数、事件处理、Promise 都建立在这上面。
- **`arguments`**：函数内部的参数集合。
- **默认参数 / 剩余参数**：`function f(a, b = 1, ...rest) {}`

> 安全提示：
> - 逆向分析时经常要找"加密函数"：它就是一个普通函数，找到后可以在控制台直接调用 `encrypt("123456")` 看结果。
> - IIFE 内的变量外部访问不到，但可以通过**断点**或 **Hook** 获取。

### 3.7 对象、数组与 JSON

```js
// 对象
const user = { name: "alice", age: 20, isAdmin: false };
user.name;         // 点语法
user["name"];      // 方括号语法（key 可以是变量）
user.role = "x";   // 增加属性
delete user.age;   // 删除属性

// 数组
const arr = [1, 2, 3];
arr.push(4); arr.pop(); arr.length;
arr.map(x => x * 2);
arr.filter(x => x > 1);
arr.includes(2);
arr.forEach(x => console.log(x));

// JSON：对象与字符串互转
JSON.stringify(user);              // 对象 → JSON 字符串
JSON.parse('{"name":"alice"}');    // JSON 字符串 → 对象
```

> 安全提示：
> - `JSON.parse` 比 `eval` 安全；早期代码用 `eval("(" + str + ")")` 解析 JSON 会导致代码执行。
> - 对象里用 `obj[key] = value`，当 `key` 可控时，可能触发 **原型链污染**（见 7.5）。

### 3.8 作用域与闭包

```js
function outer() {
  let count = 0;
  return function inner() {
    count++;
    return count;
  };
}
const counter = outer();
counter(); // 1
counter(); // 2   inner 记住了 outer 里的 count —— 这就是闭包
```

- **作用域链**：内部函数找变量时，先在自己作用域找，找不到逐层向外找，直到全局（`window`）。
- **闭包**：函数"记住"了它被创建时所在的作用域。

> 安全 / 逆向提示：
> - 前端把密钥、加密函数藏在闭包里，是因为外面访问不到；但调试时在 Sources 面板断点后，**Scope 面板**能直接看到闭包里的变量值。
> - 这就是"JS 逆向"里定位密钥、盐值的核心手段之一。

### 3.9 `this` 与原型链

```js
const obj = {
  name: "a",
  say() { console.log(this.name); }   // this 指向调用它的对象
};
obj.say();  // "a"
const f = obj.say;
f();        // this 变成 window 或 undefined（严格模式），取不到 name
```

**原型链**：对象找不到某个属性时，会沿着 `__proto__`（即构造函数的 `prototype`）向上找，一直到 `Object.prototype`。

```js
const o = {};
o.toString;               // 来自 Object.prototype
o.__proto__ === Object.prototype;  // true
```

> 安全提示：**如果攻击者能给 `Object.prototype` 增加一个属性，所有对象都会"继承"到它**——这就是原型链污染（7.5）的原理。

### 3.10 错误处理

```js
try {
  JSON.parse(userInput);
} catch (e) {
  console.error(e.message);
} finally {
  // 无论如何都会执行
}
```

> 安全提示：前端把详细的异常信息直接显示（或接口返回堆栈）会造成信息泄露。

### 3.11 异步编程（理解网络请求必备）

JS 是**单线程**的，耗时操作（网络请求、定时器）不会阻塞主线程，而是异步完成。

```js
// 1. 回调
setTimeout(() => console.log("1秒后"), 1000);

// 2. Promise
fetch("/api/user")
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err));

// 3. async / await（Promise 的语法糖，最常用）
async function loadUser() {
  const res = await fetch("/api/user");
  const data = await res.json();
  return data;
}
```

**事件循环（Event Loop）简述**：

```
同步代码先执行 → 微任务（Promise.then / await 之后）→ 宏任务（setTimeout / 事件回调）→ 下一轮
```

```js
console.log(1);
setTimeout(() => console.log(2), 0);
Promise.resolve().then(() => console.log(3));
console.log(4);
// 输出：1 4 3 2
```

> 安全 / 逆向提示：
> - 所有前后端交互（登录、查询、提交）基本都是 `fetch` / `XMLHttpRequest` / `axios` 发出的。**在这些函数上下断点或 Hook，是定位接口和加密参数的最快办法**。
> - 竞态条件（Race Condition）漏洞也与"请求并发/异步"有关。

### 3.12 模块与打包

```js
// ES Module
export function add(a, b) { return a + b; }
import { add } from "./math.js";
```

现代网站使用 webpack / vite 等把几百个模块打包压缩成少量 JS 文件，如 `app.3f2a1b.js`、`chunk-vendors.xxx.js`。

> 安全 / 逆向提示：
> - 打包文件里常有**接口路径、密钥、测试账号、内部域名**。
> - 如果线上还保留 `.map`（Source Map）文件，可以**直接还原出开发时的完整源码**——这本身就是常见的信息泄露漏洞。

### 3.13 危险函数清单（必须记住）

| 函数 / 属性 | 危险原因 |
|-------------|----------|
| `eval(str)` | 把字符串当代码执行 |
| `new Function(str)` | 同上 |
| `setTimeout(str, ms)` / `setInterval(str, ms)` | 传字符串时等同 eval |
| `document.write(html)` / `writeln` | 直接写入 HTML |
| `element.innerHTML = x` / `outerHTML` | 把字符串解析为 HTML |
| `element.insertAdjacentHTML(pos, html)` | 同上 |
| `location = x` / `location.href = x` | 可为 `javascript:` 或开放重定向 |
| `element.src / href / action = x` | 同上，协议和目标不受控 |
| `$(x)` / `$.html(x)`（jQuery 旧版） | `$(location.hash)` 会把含 `<` 的字符串当 HTML 解析 |
| `Vue: v-html` / `React: dangerouslySetInnerHTML` | 框架里的"不转义"开关 |
| `postMessage` 的接收处理 | 未校验 `origin` 时可被任意页面发消息 |
| `Object.assign` / 递归 merge / `lodash.merge` 等（版本相关） | 可能引发原型链污染 |

---

## 4. DOM：JS 操作页面的接口

### 4.1 什么是 DOM

浏览器把 HTML 解析成一棵**树**，每个标签、属性、文本都是树上的节点，JS 通过 `document` 对象访问和修改它。

```
document
 └── html
      ├── head
      │    └── title
      └── body
           ├── h1  ("你好")
           └── div#box.red
                └── p  ("内容")
```

节点类型：**元素节点**（标签）、**属性节点**、**文本节点**、**注释节点**、**文档节点**。

> 重要区别：**DOM ≠ 源码**。右键"查看网页源代码"看到的是服务器返回的原始 HTML；F12 → Elements 看到的是 JS 执行后、浏览器**解析修正过的 DOM**。有些 XSS 只存在于 DOM 中，源码里根本看不到。

### 4.2 选取元素

```js
document.getElementById("box");            // 按 id
document.getElementsByClassName("red");    // 按 class（返回集合）
document.getElementsByTagName("p");        // 按标签名
document.querySelector("#box .red");       // CSS 选择器，返回第一个
document.querySelectorAll("a[href]");      // CSS 选择器，返回全部
```

### 4.3 读取与修改内容

```js
const el = document.getElementById("box");

el.innerText;       // 可见文本（受 CSS 影响）
el.textContent;     // 纯文本，不解析 HTML  ← 安全
el.innerHTML;       // 读取/写入 HTML 字符串  ← 危险
el.outerHTML;       // 含自身标签的 HTML

el.textContent = "<b>不会被当 HTML</b>";   // 原样显示尖括号，安全
el.innerHTML   = "<b>会被当 HTML</b>";     // 被解析为加粗标签
```

> **核心防御原则：往页面插入用户数据时，优先用 `textContent`，不要用 `innerHTML`。**

关于 `innerHTML` 的一个常见误区：

```js
el.innerHTML = "<script>alert(1)</script>";   // ❌ 不会执行（HTML5 规定 innerHTML 插入的 script 不运行）
el.innerHTML = "<img src=x onerror=alert(1)>"; // ✅ 会执行！
el.innerHTML = "<svg onload=alert(1)>";        // ✅ 同样可能执行
```

### 4.4 操作属性与样式

```js
el.getAttribute("href");
el.setAttribute("href", "https://example.com");
el.removeAttribute("hidden");
el.id; el.className; el.src;     // 直接访问属性
el.dataset.userId;               // 对应 data-user-id
el.style.display = "none";
el.classList.add("active");
```

### 4.5 创建与增删节点

```js
const p = document.createElement("p");
p.textContent = "新段落";
document.body.appendChild(p);    // 添加
el.remove();                     // 删除
el.replaceWith(p);               // 替换
```

> 安全对比：`createElement + textContent`（安全）vs 拼接字符串 + `innerHTML`（危险）。

### 4.6 事件机制

```js
// 推荐：addEventListener
btn.addEventListener("click", function (e) {
  console.log(e.target);
});

// 也可以：属性赋值
btn.onclick = () => alert(1);
```

- **事件冒泡 / 捕获**：事件从触发元素向上传播到父级（冒泡）；可用 `e.stopPropagation()` 阻止。
- **事件委托**：把监听器挂在父元素上，统一处理子元素的事件。
- **`e.preventDefault()`**：阻止默认行为（如表单提交、链接跳转）。

> 安全 / 逆向提示：
> - 在 DevTools 的 Elements 面板 → **Event Listeners** 可以看到元素绑定了哪些事件；Sources 面板里有 **Event Listener Breakpoints**，可以在 `click`、`submit` 等事件发生时自动断下，找到登录按钮背后的加密逻辑。

### 4.7 DOM 型 XSS（DOM-based XSS）

特点：**漏洞完全发生在前端**，恶意数据可能不经过服务器（例如放在 `#` 后面）。

```html
<div id="out"></div>
<script>
  // Source：location.hash   Sink：innerHTML
  const name = decodeURIComponent(location.hash.slice(1));
  document.getElementById("out").innerHTML = "欢迎，" + name;
</script>
```

访问：`https://target/page.html#<img src=x onerror=alert(1)>` 即触发。

**修复**：

```js
document.getElementById("out").textContent = "欢迎，" + name;
```

另一个经典例子（jQuery 选择器注入）：

```js
$(location.hash)   // 旧版 jQuery 对以 # 开头的字符串当选择器，但如果含 < 则可能被当 HTML 解析
```

### 4.8 DOM Clobbering（DOM 覆盖）

浏览器会把带 `id` / `name` 的元素自动挂到 `window` / `document` 上：

```html
<img id="config">
<script>
  console.log(window.config);   // 输出 <img id="config"> 这个元素，而不是 undefined
</script>
```

如果代码这样写：

```js
const url = window.config?.url || "/default.js";
loadScript(url);
```

攻击者在能注入"无害 HTML"（如 `<a>` 标签，没有 JS）的位置，注入：

```html
<a id="config" href="//evil.com/x.js"></a>
<a id="config" name="url" href="//evil.com/x.js"></a>
```

使 `window.config.url` 被覆盖为攻击者的值，从而劫持脚本加载。这种攻击**绕过了"过滤脚本"的防御**，因为注入内容里没有任何 JS。

**防御**：不要依赖未声明的全局变量；用 `const config = window.config instanceof Object ? ...`；使用 DOMPurify 时开启 `SANITIZE_DOM`（默认开启）。

### 4.9 Mutation XSS（了解即可）

浏览器在解析和序列化 HTML 时会"修正"结构，某些看似无害的字符串经过 `innerHTML` 往返后会变成可执行结构。这也是为什么**自己写正则过滤 HTML 几乎一定会被绕过**，应使用成熟库（如 DOMPurify）。

---

## 5. BOM：JS 操作浏览器的接口

BOM 没有统一标准，但各浏览器基本一致，核心是 `window` 对象。

### 5.1 `window`：全局对象

```js
window.innerWidth;       // 窗口宽度
window.alert("x");       // 弹窗（可直接写 alert("x")）
window.confirm("确定？"); // 确认框
window.prompt("输入：");  // 输入框
window.open(url);        // 打开新窗口
window.close();
window.setTimeout(fn, ms);  window.setInterval(fn, ms);
```

全局变量、全局函数都挂在 `window` 上，所以 `window.a` 和 `a` 等价。

### 5.2 `location`：地址栏对象（安全最高频）

以 `https://a.com:8080/path/page.html?id=1&name=x#section` 为例：

| 属性 | 值 |
|------|----|
| `location.href` | 完整 URL |
| `location.protocol` | `https:` |
| `location.host` | `a.com:8080` |
| `location.hostname` | `a.com` |
| `location.port` | `8080` |
| `location.pathname` | `/path/page.html` |
| `location.search` | `?id=1&name=x` |
| `location.hash` | `#section` |
| `location.origin` | `https://a.com:8080` |

```js
location.href = "https://example.com";   // 跳转
location.reload();
location.replace(url);                   // 跳转且不留历史记录

// 解析参数的推荐方式
const params = new URLSearchParams(location.search);
params.get("id");
```

> 安全提示：
> - `location.search` / `location.hash` 是最常见的 **Source**。
> - 给 `location.href` 赋值的内容可控时：
>   - 可以是 `javascript:alert(1)` → XSS；
>   - 可以是 `//evil.com` → **开放重定向**（登录后跳转 `?next=` / `?redirect=` 是常见点）。

### 5.3 `history`：历史记录

```js
history.back(); history.forward(); history.go(-2);
history.pushState(state, "", "/new-path");   // 修改地址栏但不刷新（SPA 路由用）
```

### 5.4 `navigator`：浏览器信息

```js
navigator.userAgent;   // UA 字符串
navigator.language;
navigator.platform;
navigator.cookieEnabled;
navigator.webdriver;   // 自动化浏览器通常为 true
```

> 安全提示：网站常用 `userAgent`、`webdriver`、屏幕尺寸、插件等做**浏览器指纹**与**反爬/反自动化检测**。

### 5.5 `screen`：屏幕信息

`screen.width`、`screen.height`，同样用于指纹。

### 5.6 `document.cookie`：Cookie

```js
document.cookie;                       // 读取当前页面可见的 Cookie（HttpOnly 的读不到）
document.cookie = "a=1; path=/";       // 设置
```

Cookie 的关键属性（服务端 `Set-Cookie` 设置）：

| 属性 | 作用 | 安全意义 |
|------|------|----------|
| `HttpOnly` | JS 无法读取 | 即使有 XSS 也不能直接偷走该 Cookie（但 XSS 仍可借用会话发请求） |
| `Secure` | 仅 HTTPS 传输 | 防明文窃听 |
| `SameSite=Strict/Lax/None` | 跨站请求是否携带 | 防 CSRF；现代浏览器默认按 `Lax` |
| `Domain` / `Path` | 生效范围 | 范围过宽可导致子域间互相影响 |
| `Expires` / `Max-Age` | 有效期 | 过长增加被盗风险 |

### 5.7 Web Storage 与其他存储

| 存储 | 容量 | 生命周期 | 范围 | 特点 |
|------|------|----------|------|------|
| `localStorage` | ~5MB | 永久，除非清除 | 同源 | 不会自动随请求发送；**XSS 可直接读取** |
| `sessionStorage` | ~5MB | 标签页关闭即清除 | 同源 + 同标签页 | 同上 |
| Cookie | ~4KB | 可设有效期 | 按 Domain/Path | 会随请求自动发送 |
| IndexedDB | 大 | 永久 | 同源 | 结构化数据存储 |

```js
localStorage.setItem("token", "abc");
localStorage.getItem("token");
localStorage.removeItem("token");
localStorage.clear();
```

> 安全提示：
> - 把 JWT / 令牌存在 `localStorage` 中，一旦存在 XSS，令牌可被直接读走；存在 `HttpOnly` Cookie 则 JS 读不到（但要配合 CSRF 防护）。这是前端认证设计里的经典取舍。
> - 逆向时，在 DevTools → Application 面板可以查看/修改所有存储，修改本地存储的"角色""开关"值也是前端逻辑漏洞的测试方式。

### 5.8 网络请求：`XMLHttpRequest` 与 `fetch`

```js
// fetch
fetch("/api/login", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ user: "a", pass: "b" }),
  credentials: "include"      // 跨域时是否携带 Cookie
}).then(r => r.json()).then(console.log);

// XMLHttpRequest（老式，仍大量存在）
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/info");
xhr.onload = () => console.log(xhr.responseText);
xhr.send();
```

> 安全提示：
> - 受同源策略（第 6 章）限制：跨域请求要服务器通过 CORS 允许才能读到响应。
> - **XSS 拿到执行权限后，可以用 `fetch` 带着受害者的 Cookie 向任意接口发请求**（改密码、转账、读数据）——这就是为什么 `HttpOnly` 不能完全消除 XSS 的危害。
> - 逆向时在 `fetch` / `XMLHttpRequest.prototype.send` 上做 Hook，可以看到所有请求的明文参数。

### 5.9 `postMessage`：跨窗口通信

```js
// 发送方（例如 a.com 页面）
otherWindow.postMessage({ type: "hello" }, "https://b.com");

// 接收方（b.com 页面）
window.addEventListener("message", (e) => {
  // ❌ 危险：没有校验来源
  document.getElementById("out").innerHTML = e.data;

  // ✅ 安全：校验 origin 并使用 textContent
  if (e.origin !== "https://a.com") return;
  document.getElementById("out").textContent = e.data;
});
```

> 安全提示：
> - 接收端**不校验 `e.origin`** + 数据流入 Sink = 任意网站都能向它发消息触发 XSS。
> - 发送端 `targetOrigin` 写成 `"*"` 并发送敏感数据，则可能被恶意页面接收。

### 5.10 `window.opener` 与 `window.name`

- `window.open()` 打开的新窗口通过 `window.opener` 引用原窗口；带 `target="_blank"` 的链接也可能出现。若不加 `noopener`，新页面可执行 `window.opener.location = "钓鱼页面"`（Reverse Tabnabbing）。
- `window.name` 在页面跳转（甚至跨域）后仍保留，历史上用来跨域传数据，也是 DOM XSS 的 Source 之一。

### 5.11 DOM 与 BOM 对照总结

| | DOM | BOM |
|---|-----|-----|
| 管什么 | 页面内容（标签、文本、属性） | 浏览器环境（地址、历史、存储、请求、窗口） |
| 入口对象 | `document` | `window` |
| 典型 API | `querySelector`、`innerHTML`、`createElement`、`addEventListener` | `location`、`history`、`navigator`、`localStorage`、`fetch`、`open` |
| 典型漏洞 | DOM XSS、DOM Clobbering | 开放重定向、postMessage 漏洞、存储窃取、`opener` 劫持 |

---

## 6. 浏览器安全机制

### 6.1 同源策略（SOP, Same-Origin Policy）

**同源 = 协议 + 域名 + 端口 完全相同。**

| URL（对比 `https://a.com/x`） | 是否同源 |
|------|------|
| `https://a.com/y` | ✅ |
| `http://a.com/x` | ❌ 协议不同 |
| `https://b.a.com/x` | ❌ 域名不同 |
| `https://a.com:8443/x` | ❌ 端口不同 |

同源策略限制：**一个源的脚本不能读取另一个源的数据**（DOM、Cookie、存储、`fetch` 响应）。注意：

- 限制的是"**读**"，而不是"**发**"：跨站的请求（如 `<img>`、`<form>` 提交）依然能发出去，所以才有 CSRF。
- `<script src>`、`<img src>`、`<link>` 可以跨域加载（这是 JSONP 能存在的原因）。

### 6.2 CORS（跨源资源共享）

服务器通过响应头声明"允许哪些源读取我的响应"：

```
Access-Control-Allow-Origin: https://a.com
Access-Control-Allow-Credentials: true
```

**常见配置错误（漏洞点）**：

| 错误配置 | 危害 |
|----------|------|
| 直接把请求的 `Origin` 反射到 `Access-Control-Allow-Origin`，且 `Allow-Credentials: true` | 任意恶意网站可带着受害者 Cookie 读取接口数据 |
| 允许 `null` 源并带凭据 | 可通过沙箱 iframe 构造 `null` 源利用 |
| 白名单校验写成"包含/后缀匹配"（如 `endsWith("a.com")`） | `evil-a.com` 绕过 |
| `Allow-Origin: *` 配合敏感接口 | `*` 时浏览器不允许携带凭据，但敏感的无凭据数据（内网接口）仍可能被读 |

> 补充：带自定义头或 `application/json` 的跨域请求会先发一个 **OPTIONS 预检请求**，通过后才发真正请求。

### 6.3 CSP（内容安全策略）

通过响应头或 `<meta>` 限制页面能加载/执行哪些资源，是防御 XSS 的重要纵深手段：

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'; object-src 'none'; frame-ancestors 'none'
```

| 指令 | 作用 |
|------|------|
| `default-src` | 其他资源类型的默认来源 |
| `script-src` | 脚本来源（`'self'`、域名、`'nonce-xxx'`、`'unsafe-inline'`、`'unsafe-eval'`） |
| `img-src` / `style-src` / `connect-src` | 图片 / 样式 / `fetch` 目标 |
| `frame-ancestors` | 谁能把本页面嵌进 iframe（防点击劫持） |
| `object-src` | 插件嵌入 |
| `base-uri` | 限制 `<base>` 标签 |

注意：`'unsafe-inline'` 会让内联脚本和事件属性重新可用（削弱防护）；`'unsafe-eval'` 允许 `eval`；白名单里包含有 JSONP 接口或可托管任意 JS 的 CDN 域名时，策略可被绕过。

### 6.4 SRI（子资源完整性）

```html
<script src="https://cdn.example.com/lib.js"
        integrity="sha384-xxxxxxxx" crossorigin="anonymous"></script>
```

CDN 上的文件被篡改后，哈希不匹配，浏览器拒绝执行。

### 6.5 `iframe sandbox`

```html
<iframe src="..." sandbox="allow-scripts"></iframe>
```

默认 `sandbox` 会禁用脚本、表单、弹窗、同源访问等，按需用 `allow-*` 放开。**同时写 `allow-scripts` 和 `allow-same-origin` 会让沙箱形同虚设**。

### 6.6 其他重要响应头

| 响应头 | 作用 |
|--------|------|
| `X-Frame-Options: DENY / SAMEORIGIN` | 防点击劫持（被 CSP `frame-ancestors` 取代，但仍广泛使用） |
| `X-Content-Type-Options: nosniff` | 禁止浏览器猜测 MIME 类型 |
| `Strict-Transport-Security` (HSTS) | 强制 HTTPS |
| `Referrer-Policy` | 控制 Referer 泄露 |
| `Permissions-Policy` | 限制摄像头、定位等浏览器能力 |

### 6.7 Trusted Types（了解）

Chrome 等浏览器支持的机制：配置后，向 `innerHTML` 等 Sink 直接赋字符串会被浏览器拒绝，必须经过安全的转换函数。是从根源上防 DOM XSS 的方向。

---

## 7. 前端常见漏洞详解

### 7.1 XSS（跨站脚本）

| 类型 | 数据流 | 特点 |
|------|--------|------|
| **反射型** | URL 参数 → 服务器 → 响应 HTML | 需要诱导点击构造好的链接 |
| **存储型** | 提交内容 → 存数据库 → 其他用户访问时输出 | 危害最大（评论、昵称、工单、文件名等位置） |
| **DOM 型** | URL / 其他 Source → 前端 JS → Sink | 不一定经过服务器，需读前端代码才能发现 |

**危害**：在受害者浏览器中以该网站身份执行 JS，可读取页面数据与非 HttpOnly 的存储、以受害者身份发请求（改密、转账、发帖）、植入钓鱼表单、键盘记录等。

**测试思路（授权环境）**：

1. 找输入点：搜索框、评论、个人资料、URL 参数、请求头、文件名。
2. 输入一个无害的唯一标记（如 `xss123abc`），看它**出现在响应/DOM 的什么位置**。
3. 判断上下文（2.7 / 2.8 节表格），构造对应的逃逸（先逃出当前上下文，再执行）。
4. 如果有过滤，观察过滤了什么：关键字、标签、引号、括号、事件属性，再换同类的替代（如换标签、换事件、换编码）。
5. 用 `alert(1)` / `print()` 作为最小验证，再评估真实危害。

**防御**：

- 按上下文做**输出编码**（首选）；
- 前端用 `textContent`，避免 `innerHTML`；
- 必须输出富文本时使用 **DOMPurify** 等成熟库；
- 配置 **CSP**；
- Cookie 设置 `HttpOnly`；
- 框架中避免 `v-html`、`dangerouslySetInnerHTML`。

### 7.2 CSRF（跨站请求伪造）

原理：受害者已登录 `bank.com`，访问恶意页面，恶意页面让浏览器向 `bank.com` 发请求，**浏览器会自动带上 Cookie**，服务器误以为是用户本人操作。

```html
<!-- 恶意页面 -->
<form action="https://bank.com/transfer" method="POST" id="f">
  <input name="to" value="attacker">
  <input name="amount" value="1000">
</form>
<script>document.getElementById("f").submit();</script>
```

防御：

- **CSRF Token**（服务端生成、校验）；
- Cookie 设置 `SameSite=Lax/Strict`；
- 校验 `Origin` / `Referer`；
- 敏感操作二次验证；
- 不用 GET 做修改性操作。

> 与 XSS 的区别：CSRF **不能读取**响应，只能"发请求"；XSS 既能读又能发，且能绕过 CSRF Token（因为脚本就在同源页面里）。

### 7.3 开放重定向（Open Redirect）

```js
// 漏洞代码
const next = new URLSearchParams(location.search).get("next");
location.href = next;
// 攻击：?next=https://evil.com  或  ?next=javascript:alert(1)
```

危害：钓鱼、绕过 OAuth 回调校验、配合其他漏洞窃取令牌。
防御：重定向目标使用**白名单**，或只允许站内相对路径；严格校验解析后的 host 而不是字符串包含。

### 7.4 点击劫持（Clickjacking）

攻击者把目标网站用透明 iframe 覆盖在诱导按钮上，用户以为点的是"领奖"，实际点的是目标网站的"删除/转账"。
防御：`X-Frame-Options` / CSP `frame-ancestors`；敏感操作加二次确认；前端"frame busting"脚本不可靠，不能单独依赖。

### 7.5 原型链污染（Prototype Pollution）

回顾原型链：所有对象都继承 `Object.prototype`。如果攻击者能给 `Object.prototype` 添加属性，则全局对象都会被影响。

```js
// 有缺陷的递归合并函数
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] = merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

// 攻击者输入（注意 JSON.parse 会把 "__proto__" 当普通键创建出来）
const payload = JSON.parse('{"__proto__": {"isAdmin": true}}');
merge({}, payload);

// 之后：
const user = {};
console.log(user.isAdmin);   // true ！所有对象都被污染
```

危害：

- 前端：污染后影响页面逻辑，进一步构成 DOM XSS（某些库读取配置属性后进入 Sink）；
- 后端 Node.js：绕过权限判断、参数注入，甚至 RCE（取决于使用的库和调用链）。

防御：过滤 `__proto__`、`constructor`、`prototype` 键；使用 `Object.create(null)` 或 `Map`；升级存在已知污染问题的库；`Object.freeze(Object.prototype)` 等。

### 7.6 JSONP 与信息泄露

JSONP 通过 `<script src="https://api.com/data?callback=fn">` 绕过同源策略取数据：

```js
// 服务器返回：
fn({"user":"alice","token":"xxx"});
```

风险：

- 恶意网站直接引用该接口，页面里的 `fn` 回调就能拿到受害者（带 Cookie）的数据 → **JSONP 劫持**；
- `callback` 参数没过滤 → 反射型 XSS。

### 7.7 前端逻辑与客户端校验类问题

| 问题 | 说明 | 测试思路 |
|------|------|----------|
| 前端权限控制 | 仅隐藏按钮/菜单，接口无服务端鉴权 | 直接请求接口，改 ID 测越权（IDOR） |
| 前端校验 | `maxlength`、正则、文件类型限制 | 抓包改参数绕过 |
| 前端加密/签名 | 密码、`sign` 参数在 JS 里加密 | 逆向算法，脚本复现，或直接 Hook 修改明文后再加密 |
| 硬编码敏感信息 | JS 里的 `accessKey`、`token`、测试账号、内部接口 | 全局搜索关键字 |
| Source Map 泄露 | `.map` 文件还原源码 | 访问 `xxx.js.map` |
| 调试代码残留 | `console.log` 打印敏感信息、`debugger` | 看控制台与网络 |
| 弱类型比较、falsy 判断错误 | 如 `"false"` 为真 | 修改请求中的类型 |

### 7.8 其他需要了解的点

- **供应链/第三方 JS**：引入的第三方脚本被篡改，等于网站被注入。
- **WebSocket 劫持（CSWSH）**：握手没有校验 `Origin`。
- **Service Worker / PWA**：被注入后可长期劫持请求。
- **XS-Leaks / 侧信道**：通过计时、帧计数等推断跨站信息（进阶）。

---

## 8. 实战：用 DevTools 分析网站前端 JS

### 8.1 DevTools 面板对应用途

| 面板 | 用途 |
|------|------|
| **Elements** | 查看/修改 DOM、样式、事件监听器 |
| **Console** | 执行 JS、查看报错、直接调用页面函数 |
| **Sources** | 查看 JS 文件、下断点、单步调试、查看 Scope（作用域变量）、全局搜索（`Ctrl+Shift+F`） |
| **Network** | 看所有请求与响应，筛 `Fetch/XHR`，右键复制为 cURL |
| **Application** | Cookie、LocalStorage、SessionStorage、IndexedDB、Service Worker |
| **Security** | 证书与混合内容信息 |

### 8.2 信息收集流程（针对 JS）

1. **收集 JS 文件**：Network → JS 筛选；或查看页面源码里所有 `<script src>`；分析 `webpack` 的 chunk 加载逻辑找到未加载的路由模块。
2. **检查 Source Map**：看文件末尾是否有 `//# sourceMappingURL=xxx.map`，尝试访问 `.map` 文件。
3. **全局搜索关键字**（Sources 里 `Ctrl+Shift+F`，或把 JS 下载后 `grep`）：

```
/api/    /v1/    /admin    /internal    baseURL    axios    fetch(
token    secret   password  apikey   accessKey    Authorization   Bearer
encrypt  decrypt  sign      md5      sha256   AES    RSA    CryptoJS    JSEncrypt
eval(    innerHTML    document.write    postMessage    location.href    location.search    location.hash
```

4. **提取接口与参数**：整理未在页面中出现的接口（后台、测试、旧版本接口），逐个测试鉴权与参数。
5. **确认 Source → Sink**：找 `location`、`document.cookie`、`postMessage` 等 Source，追到 `innerHTML`、`eval` 等 Sink（DOM XSS 审计）。

### 8.3 分析"加密参数/签名"的通用思路

目标：登录或查询请求里的 `password` / `sign` 是怎么算出来的。

1. 在 Network 里找到请求 → 看 **Initiator（发起调用栈）**，点进去就到发起请求的代码。
2. 在 `XHR/fetch` 断点（Sources → XHR/fetch Breakpoints，填入 URL 关键字）让请求发出前断下。
3. 看 **Call Stack（调用栈）** 往上逐层查看，找到加密函数；在 **Scope** 面板里读取密钥、IV、盐等变量值。
4. 在 Console 里直接调用该函数验证；或把算法用 Python/Node 复现。
5. 若有混淆（变量名 `_0x1a2b`、字符串数组 + 解码函数、控制流平坦化），先格式化（`{}` 美化按钮）、利用断点与 Hook 观察运行时值，而不是通读混淆代码。

### 8.4 常用 Hook 思路（在 Console 中先体验）

```js
// Hook JSON.parse / JSON.stringify，观察数据流
const _parse = JSON.parse;
JSON.parse = function (s) {
  console.log("[parse]", s);
  return _parse.apply(this, arguments);
};

// Hook fetch，打印请求
const _fetch = window.fetch;
window.fetch = function (...args) {
  console.log("[fetch]", args);
  return _fetch.apply(this, args);
};

// Hook Cookie 写入，在设置时下断点
let _cookie = Object.getOwnPropertyDescriptor(Document.prototype, "cookie");
Object.defineProperty(document, "cookie", {
  get() { return _cookie.get.call(document); },
  set(v) { debugger; _cookie.set.call(document, v); }
});

// 绕过简单的反调试：禁用无限 debugger
// 在 Sources 里右键该行 → "Never pause here" / 条件断点设为 false
```

> 核心理解：**因为 JS 是动态语言，函数和对象都能被运行时替换，所以前端的任何逻辑都能被观察和篡改**。

### 8.5 辅助工具（均为非中文资源社区常用）

- **Burp Suite**：抓包、重放、改包；配合 JS 分析插件（如 JS Miner 等）自动提取 JS 中的接口和敏感信息。
- **LinkFinder / SecretFinder**：从 JS 文件中提取 URL 路径和疑似密钥。
- **Chrome DevTools Overrides（本地替换）**：把线上 JS 替换成本地修改后的版本再调试。
- **Prettier / de4js 等美化与反混淆工具**：处理压缩/混淆代码。
- **DOM Invader（Burp 内置浏览器扩展）**：辅助发现 DOM XSS、原型链污染、DOM Clobbering。

---

## 9. 延伸：Node.js 与 Android WebView 中的 JS

### 9.1 Node.js（服务端 JS）

- 前端 JS 受浏览器沙箱限制；**Node.js 能读写文件、执行系统命令、监听端口**。
- 与安全相关的典型问题：
  - `eval`、`child_process.exec(用户输入)` → **命令执行**；
  - `require(用户输入)` → 任意模块加载；
  - 原型链污染 → 逻辑绕过或 RCE；
  - 模板引擎注入（SSTI）、`path` 拼接导致路径穿越；
  - `npm` 供应链投毒、依赖含已知漏洞。
- 先掌握浏览器 JS，再学 Node 会很顺：语法一致，只是运行环境不同（没有 `window`/`document`，有 `process`、`fs`、`require`）。

### 9.2 Android WebView（与后续 Android 逆向的衔接）

很多 App 内嵌网页（混合应用）。关键点：

```java
webView.getSettings().setJavaScriptEnabled(true);
webView.addJavascriptInterface(new Bridge(), "android");   // 暴露 Java 对象给网页 JS
```

- 网页中的 JS 可以调用 `android.xxx()`，触达 App 的原生能力（读文件、拨号、获取设备信息等）。
- 风险：加载了不可信页面（或可被 XSS 的页面）+ 暴露了敏感接口 = 网页漏洞升级为 App 原生层漏洞；
- 还有 `setAllowFileAccess`、`setAllowUniversalAccessFromFileURLs`、`loadUrl("javascript:...")`、深链接（Deep Link）参数直接传入 `loadUrl` 等问题。
- 因此**扎实的 JS / DOM / BOM 基础，是 Android 混合应用与小程序类目标分析的底层能力**。

---

## 10. 练习路线与靶场

### 10.1 分阶段学习建议

| 阶段 | 内容 | 产出 |
|------|------|------|
| **阶段 1：语言基础** | 变量、类型、函数、对象数组、作用域闭包、异步 | 在 Console 里能写小函数；能读懂 50 行左右的业务代码 |
| **阶段 2：HTML + DOM** | 标签/属性、表单、选取与修改元素、事件、`innerHTML` vs `textContent` | 自己写一个"搜索框回显"页面，故意写出 XSS 再修复 |
| **阶段 3：BOM + 网络** | `location`、存储、`fetch`、Cookie 属性、同源策略与 CORS | 写页面用 `fetch` 调接口，观察 Network |
| **阶段 4：漏洞原理** | XSS 三类、CSRF、开放重定向、原型链污染、CORS 错误配置 | 每种漏洞：能手写漏洞代码 + 修复代码 + 利用演示 |
| **阶段 5：实战分析** | DevTools 断点、Hook、搜索敏感信息、还原加密逻辑 | 对靶场的 JS 完成一次完整分析并复现请求 |

### 10.2 推荐靶场与资源（英文 / 国际）

- **PortSwigger Web Security Academy**：XSS、DOM-based 漏洞、原型链污染、CORS、CSRF、点击劫持，覆盖本文大部分漏洞，并且有 DOM Invader 的教程。
- **Google XSS Game**（`xss-game.appspot.com`）：循序渐进的 XSS 小关卡，很适合入门。
- **OWASP Juice Shop**：现代 SPA（Angular）靶场，练前端漏洞和 JS 分析。
- **DVWA / bWAPP**：本地搭建，练基础 XSS、CSRF。
- **TryHackMe / Hack The Box**：有 Web 与 JS 相关的房间和机器。
- **MDN Web Docs（英文）**：JS / HTML / DOM / BOM 最权威的参考文档。
- **OWASP Cheat Sheet Series**：XSS Prevention、DOM based XSS Prevention、CSRF Prevention、Content Security Policy 等速查。
- **HackTricks**：XSS、Prototype Pollution、CORS 等章节的技巧集合。

### 10.3 自检清单

- [ ] 我能解释 DOM 与 BOM 的区别，并各举 3 个 API 例子。
- [ ] 我能说出 `innerHTML` 和 `textContent` 的区别，以及为什么前者危险。
- [ ] 我能区分 XSS 的三种类型，并为每种写出一个最小漏洞示例。
- [ ] 我能说明 `HttpOnly`、`Secure`、`SameSite` 各自的作用。
- [ ] 我能解释同源策略限制的是什么、没限制什么（为什么 CSRF 仍然存在）。
- [ ] 我能用 DevTools 在 `fetch/XHR` 上下断点，找到一个请求的发起位置。
- [ ] 我能写一个有原型链污染问题的 `merge` 函数并说明如何修复。
- [ ] 我能在一份 JS 中搜索并整理出接口路径与疑似敏感信息。

---

## 11. 速查表与术语表

### 11.1 XSS 输出位置速查

| 位置 | 逃逸思路 | 防御 |
|------|----------|------|
| 标签之间 | 插入新标签 + 事件 | HTML 实体编码 |
| 属性值内 | 闭合引号 + 新属性 | 属性编码 + 必须加引号 |
| JS 字符串内 | 闭合引号 + 新语句 | JS 转义 / JSON 序列化 |
| `href` / `src` | `javascript:` 伪协议 | 协议白名单 |
| CSS 内 | 注入规则 | 避免动态输出 / CSS 转义 |

### 11.2 Source / Sink 速查

| Source（来源） | Sink（危险点） |
|---------------|----------------|
| `location.search` / `.hash` / `.href` | `innerHTML` / `outerHTML` |
| `document.referrer` | `document.write` |
| `document.cookie` | `eval` / `new Function` / `setTimeout(str)` |
| `window.name` | `location` / `location.href` 赋值 |
| `localStorage` / `sessionStorage` | `element.src / href / action` 赋值 |
| `postMessage` 的 `event.data` | jQuery `$()` / `.html()` / `.append()` |
| 接口返回的 JSON | `v-html` / `dangerouslySetInnerHTML` |

### 11.3 常用术语

| 术语 | 含义 |
|------|------|
| DOM | 文档对象模型，页面内容的树形结构与操作接口 |
| BOM | 浏览器对象模型，窗口、地址栏、历史、存储等 |
| Source / Sink | 污染源 / 危险函数 |
| SOP | 同源策略 |
| CORS | 跨源资源共享 |
| CSP | 内容安全策略 |
| SRI | 子资源完整性 |
| XSS | 跨站脚本 |
| CSRF | 跨站请求伪造 |
| IDOR | 不安全的直接对象引用（越权） |
| Hook | 在运行时替换/包装函数以观察或修改其行为 |
| Source Map | 把压缩代码映射回源码的文件 |
| Sanitize | 净化：过滤危险 HTML 保留安全部分（如 DOMPurify） |
| Payload | 用于验证漏洞的输入内容 |
| PoC | 概念验证，证明漏洞存在的最小复现 |

### 11.4 本文一页总结

1. **HTML 是结构，CSS 是样式，JS 是行为；DOM 管页面内容，BOM 管浏览器环境。**
2. **漏洞 = 不可信的 Source 流向危险的 Sink，且没有正确处理。**
3. **前端永远不可信**：校验、加密、权限都必须在服务端完成。
4. **防 XSS：按上下文做输出编码 + `textContent` 代替 `innerHTML` + CSP + HttpOnly。**
5. **前端分析的主要工具：DevTools 的 Sources（断点 / Scope / 搜索）与 Network（Initiator）。**
6. 始终在**授权范围**内测试。
