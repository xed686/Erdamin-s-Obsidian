# APP渗透测试 - 工具联动工作流

> 面向Android APP渗透测试的信息收集→深度分析→漏洞验证完整链路
> 基于 AppInfoScanner / MobSF / APKDeepLens / APKLeaks / Frida / Burp Suite 的组合使用

---

## 一、整体思路：从广度到深度

移动端渗透测试的资产收集逻辑，本质上和Web渗透"先摸清资产面，再针对性深挖"是同一套思路，只是把"域名"换成了"APP及其背后关联的服务端资产"。

```
广度收集 (资产面)  →  深度分析 (单点突破)  →  漏洞验证 (实际利用)
AppInfoScanner        MobSF                    Frida + Burp Suite
                       APKDeepLens
                       APKLeaks
```

---

## 二、阶段一：广度资产收集

**目标**：拿到目标APP背后牵连的所有域名、服务器、CDN、接口清单，为后续横向扩展做准备。

### 使用工具：AppInfoScanner

```bash
cd ~/tools/recon/AppInfoScanner
python3 app.py -t android -p /path/to/target.apk
```

**产出物**：
- 域名/IP清单
- CDN识别结果
- 接口URL列表
- 指纹信息（用到了什么中间件、框架）

**下一步动作**：
把拿到的域名清单，丢给Web端的资产扩展工具继续扩大攻击面：

```bash
subfinder -d example.com -o subdomains.txt
# 或
amass enum -d example.com
```

这一步产出的域名列表，本质上就成了你**Web端渗透测试**的资产清单，APP渗透和Web渗透在这里正式交汇。

---

## 三、阶段二：单个APK深度分析

**目标**：针对广度收集阶段挑出的重点APP，做静态+动态的完整安全评估。

### 3.1 主力工具：MobSF（静态 + 动态）

```bash
docker run -d --name mobsf -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
```

浏览器访问 `http://127.0.0.1:8000`，上传APK。

**静态分析重点关注**：
| 模块 | 关注点 |
|---|---|
| Manifest分析 | 哪些组件是`exported="true"`（潜在攻击面） |
| 权限 | 是否申请了不必要的危险权限 |
| Hardcoded Secrets | 有没有硬编码密钥、Token |
| 网络安全配置 | 是否允许明文HTTP、证书锁定情况 |
| 第三方库/SDK | 是否用了有已知CVE的旧版本组件 |

**动态分析重点关注**：
- 实时API调用、文件读写行为
- 网络流量（配合Burp Suite抓包）
- Frida Hook结果（是否成功绕过SSL Pinning / Root检测）

### 3.2 交叉验证：APKDeepLens

MobSF的敏感信息扫描规则和APKDeepLens不完全一致，跑一遍做补充验证，减少漏报：

```bash
python3 APKDeepLens.py -apk target.apk -report
```

重点看它输出的OWASP Top 10对照报告，跟MobSF报告做交叉核对。

### 3.3 专项扫描：APKLeaks

只想快速看一眼有没有明显的密钥/URL泄露，不需要MobSF这种重量级流程时用：

```bash
apkleaks -f target.apk
```

**这一阶段的产出物**：
- 一份漏洞/风险点清单（哪些组件暴露、哪些接口没鉴权、哪些密钥硬编码了）
- 每一项都标注好"需要人工验证"还是"已确认"

---

## 四、阶段三：漏洞验证与实际利用

**目标**：把上一步分析报告里"疑似有问题"的点，逐一实际验证是否可以利用。

### 4.1 组件暴露验证

针对MobSF报告里标出的`exported`组件，用adb直接调用测试是否有越权访问：

```bash
adb shell am start -n com.target.app/.SomeExportedActivity
```

### 4.2 网络流量分析

打开Burp Suite做中间人代理，配合手机/模拟器设置代理，观察真实请求：

- 如果APP做了SSL Pinning，需要先用Frida Hook掉校验逻辑
- 常用现成脚本：`frida-multiple-unpinning`一类的通用脱钩脚本

```bash
frida -U -f com.target.app -l ssl-unpinning.js --no-pause
```

### 4.3 加固/混淆情况下的动态脱壳

如果静态分析阶段发现APP做了加固（看不到真实dex），走脱壳流程：

```bash
# 借助Frida脚本或FART在运行时dump出真实dex
frida -U -f com.target.app -l dump_dex.js --no-pause
```

脱壳后拿到的真实dex，重新丢回MobSF或jadx做二次静态分析，回到阶段二重新走一遍。

---

## 五、完整流程图

```
┌─────────────────────┐
│  AppInfoScanner      │  广度收集：域名/CDN/资产清单
└──────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  subfinder / amass    │  域名扩展（转向Web端资产面）
└──────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  MobSF (静态+动态)     │  单APK深度分析，产出风险点清单
│  ├─ APKDeepLens       │  交叉验证
│  └─ APKLeaks          │  专项密钥扫描
└──────────┬───────────┘
           │
           ▼
┌─────────────────────┐
│  是否加固/混淆？        │
└──────────┬───────────┘
      是 │       │ 否
         ▼       ▼
   ┌──────────┐  ┌──────────────────┐
   │ Frida脱壳 │  │ Frida Hook + Burp │  漏洞验证/实际利用
   │ 回到MobSF │  │ adb组件调用测试     │
   └──────────┘  └──────────────────┘
```

---

## 六、当前阶段建议

你现在的工具链已经搭好了MobSF + APKDeepLens + APKLeaks + AppInfoScanner，属于阶段一和阶段二的工具都齐了。

**下一步学习顺序建议**：
1. 先把InsecureBankv2靶场用MobSF完整走一遍，吃透报告里每一项的含义
2. 学Frida基础用法（Hook函数、绕过SSL Pinning），这是阶段三绕不开的核心技能
3. 装Burp Suite（Kali一般自带），学会配合Frida做流量分析
4. 等这几步都跑通了，再回头把AppInfoScanner用在真实的多目标场景里，体会广度收集的价值

[[01-AppInfoScanner工具介绍与使用方法]]
[[02-MobSF工具介绍与使用方法]]
[[03-subfinder-工具说明]]
[[APP渗透测试学习笔记01]]