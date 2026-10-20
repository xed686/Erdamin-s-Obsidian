# MobSF (Mobile Security Framework) 工具介绍与使用方法

GitHub: https://github.com/MobSF/Mobile-Security-Framework-MobSF

---

## 一、工具定位

MobSF是一个**自动化、一体化**的移动应用安全检测框架，支持Android、iOS、Windows三大平台，能同时完成静态分析和动态分析，覆盖渗透测试、恶意软件分析、隐私分析等场景。目前21.8k star，被BlackArch、Pentoo、Android Tamer等安全发行版收录，多次入选Black Hat Arsenal，是移动安全领域认可度最高的开源工具之一。

### 核心能力一览

| 能力 | 说明 |
|---|---|
| 静态分析 | 支持APK、IPA、APPX及源码直接分析，自动反编译 |
| 动态分析 | 运行时行为监控，内置Frida集成 |
| Web API测试 | 内置API Fuzzer，可对接Burp Suite做流量分析 |
| REST API/CLI | 支持接入CI/CD流水线，做自动化批量扫描 |

---

## 二、安装部署

### 方式一：Docker（推荐，环境干净不易出问题）

```bash
docker pull opensecurity/mobile-security-framework-mobsf:latest
docker run -it --rm -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
```

后台常驻运行（不占终端窗口）：

```bash
docker run -d --name mobsf -p 8000:8000 opensecurity/mobile-security-framework-mobsf:latest
```

常用管理命令：

```bash
docker stop mobsf     # 停止
docker start mobsf    # 重新启动
docker logs mobsf     # 查看运行日志
```

### 方式二：本地裸装

```bash
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
cd Mobile-Security-Framework-MobSF
./setup.sh      # Linux/Mac
./run.sh 127.0.0.1:8000
```

### 默认登录信息

```
访问地址：http://127.0.0.1:8000
用户名：mobsf
密码：mobsf
```

---

## 三、静态分析使用流程

1. 打开网页，点击 **Upload & Analyze**，或直接把APK拖进网页
2. 自动开始反编译和规则扫描，等待跑完（视APK大小，几十秒到几分钟不等）
3. 报告页左侧菜单栏包含以下核心模块：

| 模块 | 内容 |
|---|---|
| Info | 包名、版本、签名证书信息 |
| 扫描选项 | 本次分析用的规则集 |
| 签名者证书 | 证书指纹、有效期、是否debug证书 |
| 权限 | 申请的权限清单，标注危险权限 |
| Android API | 使用了哪些敏感API |
| 可浏览的活动 | 哪些Activity可以被外部直接唤起 |
| 安全分析 | 核心漏洞发现列表（组件暴露、明文HTTP、弱加密等） |
| 恶意软件分析 | 是否命中已知恶意特征库 |
| 代码分析 | 具体到文件/行的问题代码定位 |

### 重点关注项

- **组件暴露**：Manifest里`exported="true"`的Activity/Service/Receiver/Provider，是渗透测试最经典的攻击面
- **硬编码密钥**：Findings里的Hardcoded Secrets类目
- **网络安全配置**：是否允许明文HTTP、SSL Pinning实现情况

---

## 四、动态分析使用流程

动态分析需要连接Android设备（真机或模拟器，需root）。

1. 顶部菜单切到 **DYNAMIC ANALYZER**
2. 选择已完成静态分析的APK，点击启动动态分析
3. MobSF会自动安装APP到连接的设备并启动，实时展示：
   - API调用记录
   - 文件读写行为
   - 网络请求日志
4. 内置Frida集成，可以直接在网页里执行预置的Hook脚本，比如：
   - SSL Pinning绕过
   - Root检测绕过
5. 可以配合 **Web API Viewer** 与Burp Suite联动，做接口Fuzzing测试

---

## 五、CLI / API使用（用于批量自动化）

MobSF提供REST API，登录后在页面顶部 **API** 菜单可以看到API Key，用于脚本化调用：

```bash
curl -F "file=@target.apk" http://127.0.0.1:8000/api/v1/upload \
  -H "Authorization: <你的API Key>"
```

配合官方封装的Python客户端 `mobsfpy`：

```bash
pip install mobsfpy --break-system-packages

mobsf -k <API Key> -s http://127.0.0.1:8000 upload target.apk
mobsf scan apk target.apk <APK的MD5>
mobsf report <APK的MD5> json -o result.json
```

适合批量扫描一堆APK时使用，不用一个个手动上传网页。

---

## 六、常见问题

| 问题 | 解决方法 |
|---|---|
| 报告英文看不懂 | 用Firefox内置翻译功能（地址栏翻译图标）整页翻译 |
| APK加固/混淆导致代码看不懂 | 静态分析价值有限，需先用Frida/FART脱壳，脱壳后重新上传 |
| 动态分析连不上设备 | 检查adb是否能正常识别设备（`adb devices`），以及Frida server是否已启动 |
| Docker镜像下载慢/卡住 | 属正常现象，镜像有几百MB到1GB左右，耐心等待或检查网络 |

---

## 七、在整体工作流中的位置

MobSF是**深度分析阶段**的主力工具，负责针对单个已确定的APK做全面的静态+动态安全评估，产出风险点清单，为后续Frida Hook验证、Burp流量分析等实际利用步骤提供依据。前置阶段（资产广度收集）由AppInfoScanner等工具完成。
