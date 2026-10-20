# httpx 使用文档

> httpx 是 ProjectDiscovery 团队开发的 HTTP 探测工具，用于对大量域名/URL 进行存活验证、指纹识别和信息收集。是子域名收集之后、漏洞扫描之前的关键衔接工具。

#工具介绍
> httpx 不做域名生成，只做已有域名的验证

**子域名爆破（OneforAll/subfinder 那一步）在做什么：**

- 输入：只有一个主域名，比如 `target.com`
- 动作：**生成**大量可能存在的子域名（字典拼接、CT log查询、API查询等），本质是"猜/查有哪些域名可能存在"
- 输出：一份"可能存在"的子域名列表，比如 `dev.target.com`、`api.target.com`、`test.target.com`……这里面很多其实是不存在的，或者存在但没有实际服务

**httpx 在做什么：**

- 输入：已经是**现成的**域名/URL 列表（上一步爆破出来的结果）
- 动作：对列表里**每一个**域名发起真实的 HTTP(S) 请求，看它有没有响应、返回什么状态码、是什么网站、跑的什么技术栈
- httpx **不生成任何新域名**，它只是拿着别人给它的清单去"敲门"，看哪些门后面真的有人应答

### 打个比方

- **OneforAll/subfinder** 相当于：翻遍电话黄页、问邻居、查户籍档案，列出这个小区里"可能住着人"的所有门牌号
- **httpx** 相当于：拿着这份门牌号清单，一家一家去敲门，敲开的记下"有人应门，是个开小卖部的"，没人应的就划掉

### 为什么这一步必要

子域名收集阶段（尤其用了字典爆破+排列组合猜测的话）会产生大量**假阳性**：

- 域名有 DNS 解析记录，但服务器实际没开 Web 服务（80/443端口没监听）
- 域名解析到了 CDN 或者泛解析（wildcard DNS），看起来"存在"但其实是噪音
- 域名早就废弃了，DNS 记录还留着但服务器已经下线

如果不做 httpx 这一步筛选，直接拿几千个"疑似域名"去跑 nuclei，会浪费大量时间在无效目标上，而且很多请求会直接超时或报错，干扰你看真正有价值的结果。

所以准确的说法是：**子域名爆破是"广撒网找可能性"，httpx 是"对撒出去的网做一次真实性核验，把死鱼和活鱼分开"**，两者是收集链路里前后相继、职责完全不同的两步，不是同一类工具的重复。

---

## 一、安装

### 使用 Go 安装（推荐）
```bash
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
```

### 使用预编译二进制
从 [GitHub Releases](https://github.com/projectdiscovery/httpx/releases) 下载对应系统的压缩包，解压后将二进制文件放入 `$PATH`。

### 验证安装
```bash
httpx -version
```

---

## 二、基本用法

### 单个目标探测
```bash
httpx -u target.com
```

### 批量探测（从文件读取）
```bash
cat subdomains.txt | httpx -o alive.txt
```

或使用 `-l` 参数指定输入文件：
```bash
httpx -l subdomains.txt -o alive.txt
```

### 输入格式支持
httpx 输入可以是以下任意形式，会自动补全协议：
```
target.com
http://target.com
https://target.com:8443
192.168.1.1
192.168.1.0/24
```

---

## 三、核心参数速查表

| 参数                     | 说明                                    |
| ---------------------- | ------------------------------------- |
| `-l, -list`            | 指定输入文件（域名/URL列表）                      |
| `-u, -target`          | 指定单个目标                                |
| `-o, -output`          | 指定输出文件                                |
| `-json`                | 以 JSON 格式输出，便于脚本处理                    |
| `-sc, -status-code`    | 显示 HTTP 状态码                           |
| `-title`               | 显示网页标题                                |
| `-td, -tech-detect`    | 技术栈识别（CMS、框架、Web服务器等）                 |
| `-ip`                  | 显示解析到的 IP 地址                          |
| `-cdn`                 | 标记目标是否使用 CDN                          |
| `-cl, -content-length` | 显示响应内容长度                              |
| `-ct, -content-type`   | 显示 Content-Type                       |
| `-server`              | 显示 Server 响应头                         |
| `-location`            | 显示重定向地址                               |
| `-favicon`             | 计算 favicon hash（用于资产聚类）               |
| `-hash`                | 计算响应体 hash（如 mmh3, sha256）            |
| `-ports`               | 指定探测端口，如 `-ports 80,443,8080,8443`    |
| `-path`                | 指定探测路径，如 `-path /login`               |
| `-mc, -match-code`     | 只保留指定状态码的结果，如 `-mc 200,302`           |
| `-fc, -filter-code`    | 过滤掉指定状态码的结果                           |
| `-mr, -match-regex`    | 按正则匹配响应内容                             |
| `-threads`             | 并发线程数，默认 50                           |
| `-rate-limit`          | 每秒请求数限制                               |
| `-timeout`             | 超时时间（秒）                               |
| `-retries`             | 失败重试次数                                |
| `-proxy`               | 设置代理，如 `-proxy http://127.0.0.1:8080` |
| `-follow-redirects`    | 跟随重定向                                 |
| `-silent`              | 静默模式，只输出结果不显示 banner                  |
| `-verbose`             | 显示详细日志                                |

---

## 四、实战常用命令组合

### 1. 信息收集阶段的标准组合
```bash
cat subdomains.txt | httpx -sc -title -td -ip -cdn -o alive_info.txt
```
一次性拿到：状态码、标题、技术栈、IP、是否 CDN。

### 2. 输出 JSON 便于后续脚本处理
```bash
cat subdomains.txt | httpx -sc -title -td -json -o result.json
```

### 3. 多端口探测（应对非标准端口的服务）
```bash
cat subdomains.txt | httpx -ports 80,443,8080,8443,8000,9000 -o alive_multiport.txt
```

### 4. 只保留 200/302 状态码的有效资产
```bash
cat subdomains.txt | httpx -mc 200,302 -o valid_200.txt
```

### 5. 过滤掉常见的无效响应（如 404）
```bash
cat subdomains.txt | httpx -fc 404 -o filtered.txt
```

### 6. 探测指定路径（批量找后台/接口）
```bash
cat subdomains.txt | httpx -path /admin -mc 200 -o admin_panels.txt
```

### 7. 配合代理走 BurpSuite 便于人工复查
```bash
cat subdomains.txt | httpx -proxy http://127.0.0.1:8080
```

### 8. 通过 favicon hash 聚类资产（找同类型系统，如某 CMS）
```bash
cat subdomains.txt | httpx -favicon -json -o favicon.json
```

---

## 五、与其他工具的联动

### 与子域名收集工具联动
```bash
subfinder -d target.com -silent | httpx -sc -title -o alive.txt
```

### 与 nuclei 联动（漏洞扫描前置）
```bash
cat subdomains.txt | httpx -silent | nuclei -t nuclei-templates/ -severity critical,high
```

### 提取纯 URL（去掉状态码等附加信息）后传给下一工具
```bash
cat alive_info.txt | awk '{print $1}' > urls.txt
```

---

## 六、输出结果解读示例

```
https://api.target.com [200] [Nginx] [API Gateway] [1.2.3.4] [CDN]
```
对应字段依次是：**URL、状态码、Server、技术栈、IP、是否CDN**。

重点关注：
- **状态码 200/302** 且**非 CDN** 的资产：真实源站，优先测试
- **技术栈标签**：决定后续用哪类 nuclei 模板（如识别到 `Jenkins` 就上 Jenkins 专项模板）
- **CDN 标记**：走 CDN 的目标，IP 通常不是真实服务器 IP，扫描价值和方式需要区别对待

---

## 七、注意事项

1. **合规前提**：仅在获得授权的渗透测试项目或 SRC/Bug Bounty 明确范围内的资产上使用，未授权扫描属于违法行为。
2. **速率控制**：对目标发起大量并发请求可能触发 WAF 封禁或影响目标业务，建议合理设置 `-rate-limit` 和 `-threads`。
3. **泛解析干扰**：部分域名做了泛解析（wildcard DNS），会导致大量子域名都返回同一个页面，需要结合 `-hash` 或 `-cl` 字段去重甄别，避免被虚假存活结果误导。
4. **及时更新**：`httpx` 本身也在持续迭代指纹库，建议定期执行 `go install` 更新到最新版本。

---

## 八、参考资料

- 官方仓库：https://github.com/projectdiscovery/httpx
- 官方文档：https://docs.projectdiscovery.io/tools/httpx
