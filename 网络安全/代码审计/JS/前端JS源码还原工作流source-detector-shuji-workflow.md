---
tags: [网络安全, 代码审计, JS]
---

# Source Detector + shuji 前端JS源码还原工作流

> [!info] 🧭 知识导航
> 上级：[[MOC - 代码审计]] ｜ 首页：[[🏠 知识库首页]]
> 🔗 相关：[[02 前端加密与代码混淆笔记]] · [[05 框架开发与代码审计笔记（TP）]] · [[04 Webpack技术与源码泄露笔记]]

> 本工作流用于：发现目标网站泄露的 Source Map 文件 → 还原出未压缩的原始前端源码 → 为后续代码审计 / 敏感信息挖掘提供素材。整体分两个阶段：**插件发现** 与 **命令行还原**。

---

## 一、背景知识：为什么能还原源码

前端工程化构建（Webpack/Vite/Rollup）会把源码压缩、合并、转译（TS→JS、ES6+→ES5），生成的线上 `.js` 文件人类几乎看不懂。为了方便开发环境调试，构建工具通常会同时生成 `.js.map` 文件，记录"压缩后代码"与"原始源码"的映射关系，包括原始文件名、目录结构、变量名等。

**关键问题**：很多网站在生产环境部署时，忘记关闭或限制对 `.map` 文件的公开访问，导致任何人都能下载这个映射文件，进而**完整还原出未压缩的原始工程源码**。

这条工作流就是利用这一点，做前端资产的深度信息收集。

---

## 二、阶段一：Source Detector（浏览器插件，负责"发现"）

### 作用
Chrome 扩展，自动检测并收集网站上的 `.map`（Source Map）文件和 CRX（Chrome 扩展安装包）文件，省去手动翻 DevTools 的 Sources 面板去找 `webpack://` 目录的麻烦。

### 核心功能
- 浏览网页时自动检测 `.map` 文件，插件图标上以徽标显示发现数量
- 收集浏览器扩展详情页/扩展页面上的 CRX 安装包
- 支持一键下载 source map 文件及其解析还原后的文件
- 支持下载 CRX 包及解析内容
- 配套桌面应用，可查看历史记录、按域名分类管理已发现的资产

### 使用方法
1. 正常浏览目标网站
2. 插件图标出现数字徽标，说明在当前页面检测到了 `.map` 文件
3. 点击插件图标，查看检测到的文件列表
4. 选择需要的文件，点击下载（原始 `.map` 文件 或 解析后的版本）

---

## 三、阶段二：shuji（命令行工具，负责"还原"）

### 作用
解析 `.js.map` 文件，按照原始工程的目录结构，把压缩前的源码文件完整还原到本地指定目录。

### 基本用法
```bash
shuji xxx.js.map -o ./output_dir
```

| 参数 | 说明 |
|---|---|
| `xxx.js.map` | 输入的 Source Map 文件路径 |
| `-o, --output` | 指定还原后源码的输出目录 |

执行后，`output_dir` 目录下会出现按原始工程目录结构还原的源码文件（比如 `src/components/Login.vue`、`src/api/user.js` 这类路径都会被还原出来）。

---

## 四、完整工作流串联

```
1. 正常浏览目标网站，插件自动检测 .map 文件（图标徽标提示数量）
        ↓
2. 点击插件图标，下载检测到的 xxx.js.map 文件
        ↓
3. 用 shuji 命令行工具还原源码
   shuji xxx.js.map -o ./restored_source
        ↓
4. 得到接近原始工程结构的完整源码（.vue/.js/.ts 文件 + 目录层级）
        ↓
5. 在还原出的源码中人工审计，或用 grep / trufflehog 等工具批量搜索敏感信息
```

### 批量处理示例
如果一次性发现多个 `.map` 文件，可以写个简单循环批量还原：
```bash
for f in *.js.map; do
  shuji "$f" -o "./restored_$(basename "$f" .js.map)"
done
```

---

## 五、还原源码后重点关注的信息

- **接口地址和参数**：如 `axios.get('/api/internal/...')`，可挖出后端未公开接口
- **硬编码的密钥/Token**：API Key、内部测试账号密码等
- **业务逻辑细节**：前端若包含权限判断逻辑（如 `if (user.role === 'admin')`），往往提示后端也可能存在权限校验不严的问题，是挖掘越权漏洞的重要线索
- **注释里的调试信息**：TODO、FIXME、测试接口说明，常直接暴露攻击面

### 配合敏感信息扫描工具
```bash
# 用 trufflehog 批量扫描还原出的源码目录
trufflehog filesystem ./restored_source --only-verified

# 或简单粗暴地 grep 关键词
grep -r -i "api_key\|secret\|password\|token" ./restored_source
```

---

## 六、适用范围与局限

- 主要针对 **JavaScript** 源码还原（Webpack/Vite/Rollup 打包产物最常见），CSS 理论上也可通过同样方式还原，但渗透价值较低
- 只对**存在对应 `.map` 文件**的目标生效；如果目标构建时未生成 source map，或部署前已清理/限制访问，这条路走不通，需转向反混淆分析、接口枚举等其他手段
- 还原不了后端源码（PHP/Java/Python/Go 等），这类代码不经过前端打包流程，不存在对应 source map

---

## 七、注意事项

1. **合规前提**：仅在已授权的渗透测试项目或 SRC/Bug Bounty 明确范围内的资产上使用，未授权收集/还原网站源码存在法律风险
2. **shuji 遇到解析失败时**，可以换用同类工具 `reverse-sourcemap` 再试一次：
   ```bash
   reverse-sourcemap --output-dir ./ xxx.js.map
   ```

---

## 八、参考资料

- Source Detector GitHub：https://github.com/SunHuawei/SourceDetector
- Source Detector Chrome 商店：https://chromewebstore.google.com/detail/aioimldmpakibclgckpdfpfkadbflfkn
- shuji npm 包：https://www.npmjs.com/package/shuji