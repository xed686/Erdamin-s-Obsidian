# ffuf 工具说明

GitHub 仓库：https://github.com/ffuf/ffuf

## 这款工具是干什么的

ffuf（全称 "Fuzz Faster U Fool"）是一款用 Go 编写的快速网络模糊测试（fuzzing）工具，项目开发受到 gobuster 和 wfuzz 两个经典工具的启发。核心用途是**目录/文件爆破**，但设计上不局限于目录，只要是 HTTP 请求里的任意一个位置，都可以拿来做批量猜测。

## 工作逻辑

ffuf 的核心机制非常简单直接：在请求的任意位置放一个 `FUZZ` 关键字作为占位符，ffuf 会用字典文件里的每一行逐个替换这个占位符，发出请求，然后根据响应特征（状态码、响应长度、行数、词数等）判断这个猜测是否"命中"。

它不关心响应内容具体是什么，只做"存在 / 不存在"这类判断，因此运行速度很快，适合对大规模字典做暴力枚举。

典型判断依据：
- 状态码（比如 200 存在、404 不存在）
- 响应体长度/行数是否和默认的"不存在"页面不同
- 支持用 `-fs`（过滤响应大小）、`-fc`（过滤状态码）等参数排除误报噪音

## 官方特性摘要（翻译自项目 GitHub 说明）

- 运行速度快，是项目本身的核心卖点
- 可以对 HTTP 请求头的值、HTTP 方法、POST 数据以及 URL 的各个部分（包括 GET 参数名和参数值）做模糊测试
- 提供静默模式（`-s`），输出干净，便于用管道接入其他工具处理
- 模块化架构，方便和现有工具链集成
- 过滤器与匹配器可以灵活组合，两者可以互相配合使用

## 安装

```bash
# 需要 Go 1.16 及以上
go install github.com/ffuf/ffuf/v2@latest

# macOS 也可以用 Homebrew
brew install ffuf
```

## 基础用法

**目录爆破**（在 URL 末尾放 `FUZZ` 关键字）：

```bash
ffuf -w /path/to/wordlist -u https://target/FUZZ
```

**虚拟主机（vhost）探测**，通过 Host 请求头模糊测试，并过滤掉默认响应大小：

```bash
ffuf -w /path/to/vhostlist -u https://target -H "Host: FUZZ" -fs 4242
```

**GET 参数名爆破**：

```bash
ffuf -w /path/to/paramnames.txt -u https://target/script.php?FUZZ=test_value -fs 4242
```

**POST 数据模糊测试**（比如爆破密码字段）：

```bash
ffuf -w /path/to/postdata.txt -X POST \
     -d "username=admin&password=FUZZ" \
     -u https://target/login.php -fc 401
```

## 能检测什么 / 适用场景

- 网站目录、隐藏文件、备份文件的枚举
- 未公开链接但真实存在的路径发现
- 虚拟主机名枚举（同一 IP 下的多个站点）
- 参数名/参数值的猜测（配合越权、注入类漏洞的手动测试）

## 局限性

- 只判断"路径是否存在"，不判断具体是什么漏洞，命中后仍需人工或配合其他工具（如 nuclei）二次确认
- 字典质量直接决定发现效果，字典之外的路径无法发现
- 高频请求容易被 WAF/限流拦截，需要控制并发和速率
