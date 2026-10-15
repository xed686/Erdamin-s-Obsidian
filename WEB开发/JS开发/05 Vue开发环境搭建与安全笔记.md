# Vue 开发环境搭建与安全笔记

## 一、Vue 是什么，以及它在安全层面值得注意的地方

### 1. Vue 的作用（简要）
Vue 是一款**渐进式 JavaScript 前端框架**，核心用途是用"组件化"的方式构建用户界面：把页面拆分成一个个可复用的组件（`.vue` 单文件组件），通过**响应式数据绑定**（数据变化自动驱动页面更新）和**虚拟 DOM**（提升渲染性能）来简化前端开发。它和 React 一样，都是当前前后端分离架构中前端这一侧最主流的技术选型之一。

### 2. 对安全层面的影响（这是重点）

Vue 框架本身不是漏洞，但它的一些设计特性，决定了**审计一个 Vue 项目时该重点看哪里**：

| 安全相关特性                  | 说明                                                                                                                                                                      |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 模板插值 `{{}}` 默认自动转义      | Vue 的模板语法在输出文本内容时，默认会对 HTML 特殊字符做转义处理（类似后端的 `htmlspecialchars()`），这是 Vue "默认安全"的设计，正常使用插值语法不会产生 XSS                                                                     |
| `v-html` 指令是官方提供的"逃生舱口" | 一旦开发者使用 `v-html` 指令渲染内容，Vue 就会**放弃自动转义**，直接把内容当作原始 HTML 插入页面——这和此前《DOM与BOM及基于DOM的漏洞笔记》里讲的 `innerHTML`/`document.write()` 是同一类 **Sink（危险接收点）**，是 Vue 项目里 DOM-XSS 的头号排查对象 |
| 单文件组件（SFC）需要构建打包        | `.vue` 文件本身浏览器不能直接运行，必须经过 Vite/Webpack 等工具编译打包，这意味着 Vue 项目同样会面临"打包配置不当导致源码泄露"的问题（见本文第五节，与此前《Webpack技术与源码泄露笔记》是同一套风险）                                                    |
| 业务逻辑/密钥常驻前端代码           | 前后端分离架构下，Vue 组件里经常会出现接口地址拼接、前端加密调用等逻辑（对应此前《前端加密与代码混淆笔记》），同样需要在 Vue 项目的组件代码中重点排查硬编码密钥                                                                                    |
| 依赖生态庞大（npm 第三方包）        | Vue 项目依赖大量 npm 第三方包，这些包本身的已知漏洞（供应链安全）也是审计范围的一部分，可结合 `package.json`/`package-lock.json` 核对版本是否存在已知 CVE                                                                   |

> 一句话总结：**Vue 默认是"安全"的（自动转义），但只要开发者主动使用了 `v-html` 这类"信任原始 HTML"的 API，就相当于主动打开了一个 XSS 入口**——这也是接下来第四节实验要演示的内容。

---

## 二、环境搭建

参考文档：https://cn.vuejs.org/

**前置要求**：已安装 18.3 或更高版本的 Node.js。

---

## 三、创建 Vue 项目

```bash
# 使用 Vue 官方脚手架创建
npm create vue@latest

# 使用 Vite 创建（Vite 是更现代、更快的构建工具，Vue 官方脚手架底层目前也基于 Vite）
npm create vite@latest
```

---

## 四、启动与构建

```bash
cd <your-project-name>

# 安装依赖
npm install

# 开发模式启动（本地调试）
npm run dev

# 打包构建（生成生产环境部署文件）
npm run build
```

---

## 五、学习路线建议

可以在网上找几个实际的 Vue 开源项目源码，重点关注：
1. 项目如何启动（`package.json` 里的 `scripts` 字段）。
2. 目录架构（组件如何拆分、路由如何组织、接口请求封装在哪个目录）。
3. 代码逻辑（数据流向、状态管理方式，如是否使用了 Pinia/Vuex）。

> 这一步属于熟悉前端工程结构的基础练习，安全相关性不高，简单了解项目组织方式即可，重点仍放在后面两节的具体漏洞演示上。

---

## 六、安全实战：Vue 项目中的 XSS 演示（`v-html` 导致的 DOM-XSS）

### 1. 搭建演示项目

```bash
npm create vite@latest
cd vue-xss-demo
npm install
```

### 2. 修改 `App.vue`，构造存在漏洞的代码

```vue
<template>
  <div>
    <h1>XSS 漏洞演示</h1>
    <input v-model="userInput" placeholder="输入你的内容" />
    <button @click="showContent">显示内容</button>
    <div v-html="displayContent"></div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      userInput: '',      // 用户输入
      displayContent: ''  // 显示的内容
    };
  },
  methods: {
    showContent() {
      // 直接将用户输入的内容渲染到页面
      this.displayContent = this.userInput;
    }
  }
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  margin-top: 60px;
}
</style>
```

### 3. 漏洞成因分析

这段代码的问题就出在 `<div v-html="displayContent"></div>` 这一行——`displayContent` 的值完全来自用户在输入框里填写的内容（`userInput`），点击按钮后未经任何过滤就被赋值给 `displayContent`，而 `v-html` 又会把它当作**原始 HTML 直接插入页面并执行**。这正是一条典型的 **Source（用户输入）→ Sink（`v-html`）** 未经处理的数据流。

在 Vue 中，`v-html` 的核心含义是：**将一段包含 HTML 标签的字符串，作为真实的 HTML 代码渲染到页面上，而不是作为纯文本显示。**

### 4. 启动并测试

```bash
npm run dev
```

在输入框中填入以下 payload，点击按钮：
```html
<img src="x" onerror="alert('XSS')" />
```

由于 `x` 不是一个有效的图片地址，图片加载会失败，从而触发 `onerror` 事件，执行其中的 `alert('XSS')`，证明存在 DOM-XSS。

### 5. 修复方式

**把 `v-html` 换成普通的文本插值 `{{}}`**：

```vue
<!-- 有漏洞的写法 -->
<div v-html="displayContent"></div>

<!-- 修复后的写法 -->
<div>{{ displayContent }}</div>
```

`{{}}` 插值语法会自动对内容做 HTML 转义，`<img src="x" onerror="alert('XSS')" />` 会被当作纯文本显示出来，而不会被当成 HTML 标签解析执行。

**审计/开发建议**：业务代码中若必须使用 `v-html` 渲染内容（比如渲染富文本编辑器产出的内容），应该在赋值前先用专门的 HTML 过滤库（如 DOMPurify）对内容做净化处理，而不是直接渲染原始输入。

其他安全实践参考官方文档：https://cn.vuejs.org/guide/best-practices/security

---

## 七、Webpack / Vite + Vue 源码泄露

### 1. 风险配置

Vue 项目目前主流使用 Vite 构建（也有项目仍用 Webpack），无论哪种工具，道理和此前《Webpack技术与源码泄露笔记》完全一致：**只要构建配置里开启了 Source Map 生成，并且该文件被部署到了可公开访问的环境，就能被还原出完整源码。**

以 Vite 为例，`vue.config.js`（或 `vite.config.js`）中的典型风险配置：

```js
// vue.config.js
export default defineConfig({
  plugins: [vue()],
  build: {
    sourcemap: true, // 如果需要生成 Source Map
  },
})
```

执行构建：
```bash
npm run build
```

如果 `sourcemap: true` 这项配置被保留在生产构建中，打包产物里就会连带生成 `.js.map` 文件，一旦这些文件被部署上线且可直接访问，就可以用此前笔记里提到的 `shuji`、`reverse-sourcemap` 等工具一键还原出完整的 Vue 项目源码（包括 `.vue` 组件的原始代码结构）。

### 2. 审计/防御建议
- 审计时：检查目标站点加载的 `.js` 文件末尾是否有 `//# sourceMappingURL=xxx.js.map` 注释，或直接尝试访问对应的 `.map` 路径。
- 开发时：生产构建应将 `sourcemap` 设置为 `false`，或仅在内部排查报错时临时启用、绝不对外部署。

### 3. 案例文章
- https://mp.weixin.qq.com/s/30XIDREyo0Ose4v8Aa9g2w
- https://mp.weixin.qq.com/s/4KgOZcWUnvor_GfxsMlInA

---

## 八、总结

1. Vue 本身默认是"安全"的框架设计（自动转义），真正的风险点集中在开发者主动绕开这层保护的位置——`v-html` 就是最典型的代表，审计 Vue 项目时应优先全局搜索这个关键词。
2. Vue 项目同样跑不开"构建打包"这一环节，Source Map 源码泄露问题与此前 Webpack 笔记完全通用，审计思路可以直接复用。
3. Vue/React 这类现代前端框架改变的只是"开发方式"，并没有改变 Web 安全的底层逻辑——依然是"找 Source、找 Sink、看中间有没有过滤"这套统一框架，只是 Sink 的具体 API 名称换成了框架自己的（如 `v-html` 对应原生的 `innerHTML`）。
