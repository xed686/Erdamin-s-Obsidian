# PHP 超级全局变量与安全笔记

## 一、$_SERVER 超级全局变量列表

| 变量名称                               | 变量值含义                                                                                     |
| ---------------------------------- | ----------------------------------------------------------------------------------------- |
| `$_SERVER['PHP_SELF']`             | 当前执行脚本的文件名，与文档根目录有关。例：http://example.com/test.php 中 `$_SERVER['PHP_SELF']` 的值是：/test.php。 |
| `$_SERVER['SERVER_NAME']`          | 当前运行脚本所在的服务器的主机名。如果脚本运行于虚拟主机中，该名称是由那个虚拟主机所设置的值决定。                                         |
| `$_SERVER['SERVER_SOFTWARE']`      | 服务器标识字符串，在响应请求时的头信息中给出。例如：Apache/2.2.24。                                                  |
| `$_SERVER['SERVER_PROTOCOL']`      | 请求页面时通信协议的名称和版本。例如，HTTP/1.0 或 HTTP/1.1。                                                   |
| `$_SERVER['REQUEST_METHOD']`       | 访问页面使用的请求方法；例如，GET、HEAD、POST、PUT 等。                                                       |
| `$_SERVER['REQUEST_TIME']`         | 请求开始时的时间戳。从 PHP 5.1.0 起可用。                                                                |
| `$_SERVER['QUERY_STRING']`         | 查询字符串，如果有的话，通过它进行页面访问。                                                                    |
| `$_SERVER['HTTP_ACCEPT']`          | 当前请求头中 Accept 项的内容，如果存在的话。                                                                |
| `$_SERVER['HTTP_ACCEPT_CHARSET']`  | 当前请求头中 Accept-Charset 项的内容，如果存在的话。例如：iso-8859-1,*,utf-8。                                  |
| `$_SERVER['HTTP_HOST']`            | 当前请求头中 Host 项的内容，如果存在的话。                                                                  |
| `$_SERVER['HTTP_REFERER']`         | 引导用户代理到当前页的前一页的地址（如果存在）。由 user agent 设置决定。                                                |
| `$_SERVER['HTTP_USER_AGENT']`      | 用户代理发送的 User-Agent 头信息。如：浏览器信息、IP地址、设备信息等一些与浏览者相关的信息！                                     |
| `$_SERVER['SCRIPT_NAME']`          | 包含当前脚本的路径。这在页面需要指向自己时非常有用。                                                                |
| `$_SERVER['SERVER_ADDR']`          | 当前运行脚本所在的服务器的 IP 地址。                                                                      |
| `$_SERVER['SERVER_PORT']`          | Web 服务器使用的端口。默认值为 80。如果使用 SSL 安全连接，这个值为用户设置的 HTTP 端口，通常是 443。                             |
| `$_SERVER['SERVER_SIGNATURE']`     | 包含服务器版本和虚拟主机名的字符串。                                                                        |
| `$_SERVER['PATH_TRANSLATED']`      | 当前脚本所在文件系统（非文档根目录）的基本路径。这是在服务器进行虚拟到实际路径的映像后的结果。                                           |
| `$_SERVER['SCRIPT_FILENAME']`      | 当前执行脚本的绝对路径。                                                                              |
| `$_SERVER['SERVER_ADMIN']`         | 该值指明了 Apache 服务配置文件中的 `SERVER_ADMIN` 参数。如果脚本运行在一个虚拟主机上，则该值是那个虚拟主机的值。                      |
| `$_SERVER['HTTP_X_FORWARDED_FOR']` | 代理服务器发送的 IP 地址，通常用于识别客户端的原始 IP 地址。                                                        |
| `$_SERVER['REMOTE_ADDR']`          | 浏览当前页面的用户的 IP 地址。如果用户使用了代理IP，则需要使用 `$_SERVER['HTTP_X_FORWARDED_FOR']` 来准确获取！              |
| `$_SERVER['REMOTE_USER']`          | 当使用身份验证（如使用 Basic 身份验证时），包含通过身份验证的用户名。                                                    |
| `$_SERVER['HTTPS']`                | 用于标识请求是通过 HTTPS 协议发送的。                                                                    |
| `$_SERVER['REDIRECT_URL']`         | 原始请求被重定向到的 URL。                                                                           |
| `$_SERVER['HTTP_REFERER']`         | 用户最后一次离开的页面的 URL。这可能不准确，因为用户可以更改或禁用它。                                                     |
| `$_SERVER['CONTENT_LENGTH']`       | 请求主体的大小（以字节为单位）。                                                                          |
| `$_SERVER['CONTENT_TYPE']`         | 请求主体的内容类型（如 "application/x-www-form-urlencoded" 或 "multipart/form-data"）。                 |

## 二、$_FILES 超级全局变量列表

| 变量名称                  | 变量值含义            |
| --------------------- | ---------------- |
| `$_FILES["name"]`     | 上传文件的名称          |
| `$_FILES["type"]`     | 上传文件的类型          |
| `$_FILES["tmp_name"]` | 文件被上传到服务器上的临时文件名 |
| `$_FILES["error"]`    | 上传文件时可能出现的错误代码   |
| `$_FILES["size"]`     | 上传文件的大小          |
| `$_FILES["tmpDir"]`   | 上传文件的临时存储目录      |

## 三、$_ENV 超级全局变量列表

| 变量名称                            | 变量值含义                  |
| ------------------------------- | ---------------------- |
| `$_ENV['HTTP_HOST']`            | 客户端请求的主机名。             |
| `$_ENV['HTTP_USER_AGENT']`      | 发起请求的用户代理（通常包含浏览器信息）。  |
| `$_ENV['HTTP_ACCEPT']`          | 客户端可接受的响应内容类型。         |
| `$_ENV['HTTP_ACCEPT_LANGUAGE']` | 客户端首选的语言。              |
| `$_ENV['HTTP_ACCEPT_CHARSET']`  | 客户端首选的字符集。             |
| `$_ENV['REMOTE_ADDR']`          | 访问服务器的客户端的 IP 地址。      |
| `$_ENV['REMOTE_PORT']`          | 访问服务器的客户端的端口号。         |
| `$_ENV['SERVER_ADDR']`          | 服务器的 IP 地址。            |
| `$_ENV['SERVER_PORT']`          | 服务器端口号。                |
| `$_ENV['SERVER_NAME']`          | 服务器的名称。                |
| `$_ENV['SERVER_SOFTWARE']`      | 服务器软件名称和版本。            |
| `$_ENV['DOCUMENT_ROOT']`        | 脚本所在文档根目录。             |
| `$_ENV['SCRIPT_FILENAME']`      | 当前执行脚本的绝对路径。           |
| `$_ENV['QUERY_STRING']`         | 查询字符串（当使用 `?` 传递参数时）。  |
| `$_ENV['REQUEST_METHOD']`       | 客户端请求方法（如 GET、POST 等）。 |

## 四、安全相关分类整理

### 1. 变量覆盖安全
- **`$GLOBALS`**：这种全局变量用于在 PHP 脚本中的任意位置访问全局变量。

### 2. 数据接收安全
- **`$_REQUEST`**：用于收集 HTML 表单提交的数据。
- **`$_POST`**：广泛用于收集提交 `method="post"` 的 HTML 表单后的表单数据。
- **`$_GET`**：收集 URL 中发送的数据，也可用于提交表单数据（`method="get"`）。
- **`$_ENV`**：包含服务器端环境变量的数组。
- **`$_SERVER`**：保存关于报头、路径和脚本位置的信息。

### 3. 文件上传安全
- **`$_FILES`**：文件上传处理，包含通过 POST 方法上传给当前脚本的文件内容。

### 4. 身份验证安全
- **`$_COOKIE`**：关联数组，包含通过 cookie 传递给当前脚本的内容（本地客户端浏览器存储）。
- **`$_SESSION`**：关联数组，包含当前脚本中的所有 session 内容（目标服务端存储，存储记录的数据）。

## 五、开发工具组合

**DW + PHPStorm + PhpStudy + Navicat Premium**

| 工具              | 用途                 |
| --------------- | ------------------ |
| DW              | HTML & JS & CSS 开发 |
| PHPStorm        | 专业 PHP 开发 IDE      |
| PhpStudy        | Apache + MySQL 环境  |
| Navicat Premium | 全能数据库管理工具          |

参考：https://tutorials.wcode.net/php

## 六、代码审计应用案例

### 1. DuomiCMS 变量覆盖
思路：找变量覆盖代码 → 找此文件调用 → 选择利用覆盖 Session → 找开启 Session 文件覆盖

利用示例：
```
/interface/comment.php?_SESSION[duomi_admin_id]=10&_SESSION[duomi_group_id]=1&_SESSION[duomi_admin_name]=zmh
```

参考：https://blog.csdn.net/qq_59023242/article/details/135080259

### 2. YcCms 任意文件上传
思路：找文件上传代码 → 找此文件调用 → 找函数调用 → 过滤 type，用 MIME 绕过

利用示例：
```
?a=call&m=upLoad
```

参考：https://zhuanlan.zhihu.com/p/718742254
[[02 身份验证机制笔记-Cookie_Session_Token]]