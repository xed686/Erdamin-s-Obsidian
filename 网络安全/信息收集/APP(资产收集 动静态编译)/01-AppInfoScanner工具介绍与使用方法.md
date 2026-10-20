# AppInfoScanner 工具介绍与使用方法

GitHub: https://github.com/kelvinBen/AppInfoScanner
国内加速通道: https://gitee.com/kelvin_ben/AppInfoScanner

---

## 一、工具定位

AppInfoScanner是一款面向HW行动/红队/渗透测试团队场景的**多端资产信息收集扫描工具**，支持Android、iOS、Web、H5、静态网站，能帮渗透测试人员快速从APP或Web站点中收集出关键资产信息，如Title、Domain、CDN、指纹信息、状态信息等。

**核心定位**：这是资产收集流程里**广度扩展**的工具，作用类似于Web渗透里的subfinder——只不过它面向的不是单纯的域名，而是APP/网站背后关联的整个资产网络。它不做漏洞深挖，只负责把"目标背后牵连了哪些域名/服务器/CDN"这件事快速摸清楚。

### 核心能力一览

| 能力 | 说明 |
|---|---|
| 多端支持 | Android、iOS、Web、H5、静态网站均可扫描 |
| 自动脱壳/修复 | 识别到APK有壳会尝试自动脱壳，支持一键自动修复 |
| 网络嗅探 | 内置嗅探模块，提供基础信息输出 |
| AI辅助过滤 | 简单AI识别功能，快速过滤掉无关的第三方URL |
| AK/SK检测 | 检测硬编码的云服务Access Key/Secret Key |
| 自动下载 | 可自动下载APP或缓存H5页面，无需手动准备样本 |
| 国际化语言包 | 支持多语言 |

---

## 二、环境要求

- Java 1.8及以下（用于apktool/baksmali反编译）
- Python 3环境

---

## 三、安装部署

```bash
cd ~/tools/recon

# 检查Java版本，需要1.8及以下
java -version
# 没有/版本不对则安装
sudo apt install openjdk-8-jdk -y

# 下载源码
git clone https://github.com/kelvinBen/AppInfoScanner.git
# 国内网络较慢可用Gitee加速通道
git clone https://gitee.com/kelvin_ben/AppInfoScanner.git

cd AppInfoScanner

# 安装Python依赖
pip3 install -r requirements.txt --break-system-packages

# 验证安装
python3 app.py --help
```

---

## 四、目录结构说明

```
AppInfoScanner
├── libs
│   ├── core
│   │   ├── parses.py      # 解析文件中的静态信息
│   │   ├── download.py    # 自动下载APP/H5页面
│   │   └── net.py         # 网络嗅探
│   └── task
│       ├── android_task.py    # Android相关任务处理
│       ├── ios_task.py        # iOS相关任务处理
│       ├── web_task.py        # Web/H5相关任务处理
│       └── net_task.py        # 网络嗅探任务处理
├── tools
│   ├── apktool.jar     # APK反编译
│   └── baksmali.jar    # dex反编译
├── app.py              # 主运行程序
└── config.py           # 全局配置文件
```

---

## 五、基本使用方法

### 扫描本地APK文件

```bash
python3 app.py -t android -p /path/to/target.apk
```

### 扫描iOS IPA文件

```bash
python3 app.py -t ios -p /path/to/target.ipa
```

### 扫描Web/H5站点

```bash
python3 app.py -t web -p https://target.example.com
```

### 扫描本地已存在的目录（比如已反编译过的源码目录）

```bash
python3 app.py -t android -p /path/to/decompiled_source_dir
```

具体参数名以实际`python3 app.py --help`输出为准，不同版本可能略有调整。

---

## 六、产出结果

扫描完成后，工具会在结果目录下生成：

- **Excel/txt格式的资产清单**：域名、IP、CDN识别结果
- **指纹识别信息**：用到了什么中间件、框架
- **AK/SK检测结果**：如果代码中存在硬编码的云服务密钥会单独标出

---

## 七、典型使用场景

### 场景一：拿到一个APK，想知道它背后关联了哪些服务器资产

```bash
python3 app.py -t android -p target.apk
```

跑完后拿到域名清单，作为后续子域名扩展、C段扫描的输入。

### 场景二：批量处理多个APP

写个简单的bash循环，批量跑：

```bash
for apk in ./apks/*.apk; do
  python3 app.py -t android -p "$apk"
done
```

### 场景三：资产清单产出后的下一步

```bash
# 拿到的域名继续做子域名扩展
subfinder -d example.com -o subdomains.txt
```

---

## 八、常见问题

| 问题 | 解决方法 |
|---|---|
| Java版本不兼容报错 | 确认`java -version`显示的是1.8，多版本共存用`update-alternatives`切换 |
| pip3命令找不到 | `sudo apt install python3-pip -y` |
| 反编译长时间无响应 | 检查是否是大型APK或加固APK，必要时先用MobSF/jadx预处理一遍 |
| Windows下路径含空格解析失败 | 官方已知问题，建议路径不要包含空格，或使用较新版本（该问题在后续release中修复） |

---

## 九、在整体工作流中的位置

AppInfoScanner是资产收集流程的**第一步（广度阶段）**，产出域名/CDN/资产清单后，再交给MobSF做单个APK的深度分析，或者把域名清单交给subfinder/amass等工具做Web端资产扩展，形成完整的"广度→深度"闭环。
