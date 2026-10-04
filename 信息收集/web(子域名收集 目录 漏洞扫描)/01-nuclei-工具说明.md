# Nuclei 工具说明

GitHub 仓库：https://github.com/projectdiscovery/nuclei

## 这款工具是干什么的

Nuclei 是 ProjectDiscovery 团队开发的一款基于 YAML 模板的漏洞扫描器。它的核心定位是"用模板描述检测逻辑，批量对大量目标做已知漏洞特征验证"。

项目官方定位自己为一款快速、可高度自定义的漏洞扫描引擎，依靠简单的 YAML DSL（领域特定语言）驱动，支持社区协作贡献检测规则。它能覆盖应用、API、网络、DNS 以及云环境配置等多种层面的安全检查。

## 工作逻辑

Nuclei 的执行逻辑可以拆成三层：

**1. 模板层（Template）**

每个 YAML 模板定义"发什么请求 + 怎么判断命中"，包含：
- 请求构造（路径、方法、请求头、请求体，支持变量注入和多步骤请求链）
- 匹配器（matchers）：判断响应是否命中，支持关键字、正则、状态码、响应时间、DSL 表达式等多种类型
- 提取器（extractors）：从响应中提取信息供后续步骤复用

**2. 执行层**

拿到目标列表后，按"目标 × 模板"做批量调度，用并发 worker 发请求；根据模板声明的协议类型（HTTP、DNS、TCP、SSL、File、Headless、Whois、Websocket 等）路由到对应的执行器；支持速率限制，避免打崩目标或触发 WAF 封禁。

**3. 输出层**

命中 matcher 的结果会输出模板 ID、严重等级、匹配到的请求/响应内容，方便复现和整理报告。

## 官方特性摘要（翻译自项目 GitHub 说明）

- 面向大规模目标的快速扫描，检测逻辑与请求发送分离，误报率低
- 支持 TCP、DNS、HTTP、SSL、File、Whois、Websocket、Headless 等多种协议的安全检查
- 拥有一个由社区（据项目介绍已有 300 余位安全研究者参与）维护的独立模板仓库 nuclei-templates，模板持续更新
- 从 v2.5.2 版本起默认支持模板自动下载与更新，无需手动维护模板库
- 可以用自己熟悉的 YAML DSL 编写自定义检测逻辑，适合把手工测试流程自动化、规模化

## 安装

```bash
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

也支持 Homebrew（`brew install nuclei`）和 Docker（`docker pull projectdiscovery/nuclei:latest`）等方式。

## 基础用法

```bash
# 扫描单个目标
nuclei -u https://target.com

# 扫描一批目标
nuclei -l targets.txt

# 只跑特定标签的模板（如 CVE 相关）
nuclei -u https://target.com -tags cve,exposure

# 更新模板库
nuclei -update-templates
```

## 能检测什么

- 已知 CVE 漏洞
- 敏感信息泄露（.git、.env、备份文件、API 文档等）
- 默认凭证/弱口令
- 常见错误配置
- 服务/技术指纹识别
- 子域名接管
- 网络层未授权服务

## 局限性

- 依赖模板库，没有对应模板的漏洞（包括 0day、逻辑漏洞、越权类问题）无法检出
- 模板匹配特征可能被 WAF 拦截或目标环境差异导致漏报
- 大部分默认模板针对未授权场景，需要认证态的检测需要额外配置
