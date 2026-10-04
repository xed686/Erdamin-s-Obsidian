# Subfinder 工具说明
**subfinder/amass之类**：针对拿到的域名再做子域名扩展（进一步拉宽资产面)
GitHub 仓库：https://github.com/projectdiscovery/subfinder

## 这款工具是干什么的

Subfinder 是 ProjectDiscovery 团队开发的**被动子域名枚举工具**，给定一个主域名，返回这个域名下能被发现的有效子域名。项目官方定位它"只做一件事——被动子域名枚举，并且做得很好"，架构简单、模块化，以速度为优化重点。

## 工作逻辑

Subfinder 的枚举方式以**被动查询**为主（也支持主动解析验证），不是靠字典爆破去猜子域名，而是从多个公开数据源查询已经被互联网记录过的子域名信息，主要来源包括：

- 证书透明度日志（如 crt.sh）——网站申请 HTTPS 证书时，子域名信息会被公开记录，这是最常用也相对准确的数据源
- 各类被动 DNS / 网络空间测绘服务（如 Shodan、SecurityTrails、Censys、VirusTotal、ZoomEye 等，多数需要配置对应的 API Key 才能启用）
- 搜索引擎和其他公开聚合数据源

因为是查询已有记录，而不是主动对目标发大量探测流量，所以速度快，且相对"隐蔽"（不会给目标资产带来明显的扫描痕迹）。项目本身也强调其被动模式设计是为了遵守各数据源的许可与使用限制，同时兼顾渗透测试者和 Bug Bounty 猎人的使用需求。

拿到子域名列表后，Subfinder 内置解析和泛解析（wildcard）过滤模块，会尝试排除因为泛解析配置导致的无效/误报子域名，保证输出结果尽量是真实有效的。

## 官方特性摘要（翻译自项目 GitHub 说明）

- 快速且强大的解析与泛解析剔除模块
- 精心筛选的被动数据源，尽量提高发现结果的覆盖面
- 支持多种输出格式（JSON、文件、标准输出）
- 优化了速度，资源占用轻量

## 安装

```bash
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
```

## 安装后配置

Subfinder 装好之后可以直接使用，但部分数据源需要配置 API Key 才能生效（比如 Shodan、SecurityTrails、Censys、VirusTotal、ZoomEye、GitHub 等）。这些配置保存在：

```
~/.config/subfinder/provider-config.yaml
```

首次运行工具时会自动生成这个文件，之后可以手动编辑填入自己申请到的 API Key。

查看当前支持哪些数据源：

```bash
subfinder -ls
```

## 基础用法

```bash
# 枚举单个域名的子域名
subfinder -d target.com

# 只输出纯子域名，方便配合管道传给下一个工具
subfinder -d target.com -silent

# 输出为 JSON 格式
subfinder -d target.com -oJ -o result.json
```

## 能做什么 / 适用场景

- 快速摸清一个目标公司/组织的子域名资产范围，是资产收集流程的第一步
- 挖掘测试环境、旧版本系统、内部管理后台等容易被遗忘、防护相对薄弱的资产，是 Bug Bounty 中常见的高价值信息来源
- 输出结果可直接通过管道传给 httpx（存活探测）、ffuf（路径爆破）、nuclei（漏洞验证）等下游工具，构成完整资产收集链路

## 局限性

- 被动模式依赖公开数据源的记录，无法发现从未被这些数据源收录过的"全新"子域名（比如刚创建、还没申请过证书的内部子域名）
- 很多高质量数据源（Shodan、SecurityTrails 等）需要付费 API Key 才能发挥完整效果，免费额度下结果覆盖面有限
- 只负责"发现子域名"，不判断这些子域名是否存活、有没有漏洞，需要配合 httpx、nuclei 等工具做后续处理
