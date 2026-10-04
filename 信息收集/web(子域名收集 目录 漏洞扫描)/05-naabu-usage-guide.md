# naabu 使用文档

> naabu 是 ProjectDiscovery 团队开发的高速端口扫描工具，定位是"轻量、快速、专门配合批量资产列表使用"，与 httpx、katana 同属 ProjectDiscovery 生态，设计上就是为了互相衔接，是子域名收集之后进一步发现开放端口和服务的关键工具。

---

## 一、工作逻辑

### 1. 端口扫描引擎（两种模式可选）
- **SYN 扫描**（默认，需要 root/管理员权限）：只发 SYN 包，不完成三次握手，速度快、隐蔽性相对好，原理和 nmap 的 `-sS` 类似
- **CONNECT 扫描**（无需 root 权限）：完成完整 TCP 连接，速度稍慢但兼容性好，适合权限受限或容器环境

### 2. 底层依赖高性能包处理
底层使用 `pcap` 抓包库处理数据包收发，能做到大规模并发扫描而不明显拖慢速度，这也是它相比 nmap 在"批量扫海量目标"场景下更快的原因。

### 3. 输入即结果，天然对接下一步工具
输入可以是域名、IP、CIDR 网段，甚至直接从子域名/httpx 结果里读取；输出格式简洁（`域名:端口`），可以直接管道传给 httpx 做下一步的 HTTP 服务探测。

### 4. 内置去重与主机发现（Host Discovery）
支持先做 ICMP/ARP 存活探测，过滤掉根本没开机的主机，减少无效扫描开销。

---

## 二、安装

```bash
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
```

### 验证安装
```bash
naabu -version
```

---

## 三、基本用法

### 单目标扫描（默认扫 Top 100 常见端口）
```bash
naabu -host target.com
```

### 批量扫描（从文件读取域名/IP列表）
```bash
naabu -list subdomains.txt -o open_ports.txt
```

### 扫描指定端口
```bash
naabu -host target.com -p 80,443,8080,8443,3306,6379
```

### 扫描全端口（1-65535）
```bash
naabu -host target.com -p -
```

### 扫描 CIDR 网段
```bash
naabu -host 192.168.1.0/24 -o result.txt
```

---

## 四、核心参数速查表

| 参数 | 说明 |
|---|---|
| `-host` | 单个目标（域名/IP） |
| `-list` | 批量目标文件 |
| `-p, -port` | 指定端口，如 `-p 80,443` 或 `-p -`（全端口） |
| `-top-ports` | 扫描常见端口 Top N，如 `-top-ports 1000` |
| `-o` | 输出文件 |
| `-json` | JSON 格式输出 |
| `-rate` | 每秒发包速率，控制扫描速度 |
| `-c, -concurrency` | 并发主机数 |
| `-scan-type` | 扫描模式，`s`(SYN，需root) 或 `c`(CONNECT) |
| `-skip-host-discovery` | 跳过主机存活探测，直接扫端口 |
| `-exclude-ports` | 排除某些端口 |
| `-silent` | 静默模式，只输出结果 |
| `-passive` | 被动模式，通过 Shodan 等第三方数据获取端口信息（不主动发包） |

---

## 五、实战常用命令组合

### 1. 全端口扫描 + 提速（大范围资产时常用）
```bash
naabu -list subdomains.txt -p - -rate 1000 -o all_ports.txt
```

### 2. 只扫常见 Top 1000 端口（日常信息收集首选，兼顾速度和覆盖率）
```bash
naabu -list subdomains.txt -top-ports 1000 -o ports.txt
```

### 3. 被动模式（不发包，避免打草惊蛇，纯查第三方数据）
```bash
naabu -list subdomains.txt -passive -o passive_ports.txt
```

### 4. 与 httpx 直接联动，一条流水线走完
```bash
naabu -list subdomains.txt -silent | httpx -sc -title -td -o web_services.txt
```

### 5. 全流程串联示例
```bash
subfinder -d target.com -silent \
  | naabu -silent -top-ports 1000 \
  | httpx -sc -title -td -o final_result.txt
```

---

## 六、在信息收集链路中的位置

```
子域名收集 (subfinder/OneforAll)
      ↓
端口扫描 (naabu)  ← 发现每个资产开了哪些端口
      ↓
HTTP服务探测 (httpx)  ← 针对开放的 web 端口做指纹识别
      ↓
深度爬取 (katana) / 漏洞扫描 (nuclei)
```

naabu 补上了 httpx 覆盖不到的部分——httpx 默认只探测 80/443 这类标准 Web 端口，但很多资产在非标准端口上跑着数据库、Redis、SSH 弱口令、管理面板（比如 8080 跑 Tomcat 管理台、3306 数据库直接暴露、6379 Redis 未授权访问），这些高危点位往往就是靠端口扫描先发现，再针对性去测。

---

## 七、注意事项

1. **合规前提**：仅用于已授权的渗透测试/SRC项目范围内的资产
2. **SYN 扫描需要 root 权限**（Linux 下 `sudo naabu ...`），没有权限时会自动降级或需手动指定 `-scan-type c`
3. **速率别开太猛**：`-rate` 设太高容易被目标 IDS/防火墙识别封锁，尤其对方有 WAF/云防护时要谨慎
4. **全端口扫描耗时较长**，日常侦察建议先用 `-top-ports 1000` 摸个大概，发现异常再针对性做全端口扫描

---

## 八、参考资料

- 官方仓库：https://github.com/projectdiscovery/naabu
- 官方文档：https://docs.projectdiscovery.io/tools/naabu
