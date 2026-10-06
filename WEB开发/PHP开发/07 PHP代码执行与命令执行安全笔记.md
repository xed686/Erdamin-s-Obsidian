# PHP 代码执行与命令执行安全笔记

PHP 提供了大量"代码执行类"函数，本意是方便开发者灵活处理数据（动态求值、动态回调、动态加载等）。但如果这些函数的参数中混入了**用户可控内容**且未经过滤，攻击者就能让服务器执行任意 PHP 代码甚至系统命令，这是 Web 安全中危害等级最高的一类漏洞（可直接导致 RCE / Getshell）。

---

## 一、代码执行（Code Execution）

### 1. 核心高危函数

| 函数                  | 风险点                                                                |
| ------------------- | ------------------------------------------------------------------ |
| `eval()`            | 将字符串作为 PHP 代码执行，最典型的代码执行入口                                         |
| `assert()`          | 低版本 PHP（<8.0）中传入字符串参数时等效于 `eval()` 执行                              |
| `preg_replace()`    | 历史上配合 `/e` 修饰符，会把替换内容当作 PHP 代码执行（PHP 7 已移除 `/e` 修饰符，但遗留系统/老代码中仍常见） |
| `create_function()` | 本质是对传入字符串调用 `eval()` 动态创建函数，PHP 7.2 起已废弃，PHP 8 移除                  |

> 审计提示：以上四个函数是代码执行漏洞审计中**最优先核查**的对象，只要其参数中存在用户可控拼接，基本可以直接判定为高危。

### 2. 完整高危函数清单（易导致代码执行）

> 下表函数的共性是：**它们都会把某个参数当作"可执行的回调/代码"来处理**，一旦该参数（或其内部间接传入的内容）被用户输入污染，就可能被滥用执行任意逻辑。

| 函数 | 函数 | 函数 |
|---|---|---|
| `assert()` | `array_map()` | `array_reduce()` |
| `array_filter()` | `array_diff_uassoc()` | `array_diff_ukey()` |
| `array_udiff()` | `array_udiff_assoc()` | `array_udiff_uassoc()` |
| `array_intersect_assoc()` | `array_intersect_uassoc()` | `array_uintersect()` |
| `array_uintersect_assoc()` | `array_uintersect_uassoc()` | `array_walk()` |
| `array_walk_recursive()` | `create_function()` | `usort()` |
| `uasort()` | `uksort()` | `preg_replace()`（配合 `/e`） |
| `pcntl_exec()` | `register_shutdown_function()` | `register_tick_function()` |
| `set_error_handler()` | `stream_filter_register()` | `escapeshellcmd()` |
| `exec()` | `shell_exec()` | `system()` |
| `include` / `include_once()` | `require()` / `require_once()` | `ob_start()` |
| `xml_set_character_data_handler()` | `xml_set_default_handler()` | `xml_set_element_handler()` |
| `xml_set_end_namespace_decl_handler()` | `xml_set_external_entity_ref_handler()` | `xml_set_notation_decl_handler()` |
| `xml_set_processing_instruction_handler()` | `xml_set_start_namespace_decl_handler()` | `xml_set_unparsed_entity_decl_handler()` |

**按风险类型分组理解**：
- **直接代码执行**：`eval`、`assert`、`preg_replace(/e)`、`create_function`
- **回调函数类**（第二个参数若为用户可控字符串，被当作函数名调用）：`array_map`、`array_filter`、`usort`/`uasort`/`uksort`、`array_walk`/`array_walk_recursive`、`array_udiff` 系列、`array_uintersect` 系列
- **事件/钩子回调类**：`register_shutdown_function`、`register_tick_function`、`set_error_handler`、`stream_filter_register`、各类 `xml_set_*_handler()`
- **文件包含类**：`include`/`require` 系列（详见此前的《PHP文件操作安全笔记》）[[06 PHP文件相关操作安全笔记]]
- **进程/命令执行类**：`exec`、`system`、`shell_exec`、`pcntl_exec`（与下方"命令执行"章节重合，PHP 中代码执行和命令执行的边界并不绝对）

参考资料：
- https://www.jb51.net/article/264470.htm

---

## 二、命令执行（Command Execution）

PHP 同时提供了一批用于**直接调用操作系统命令**的函数，设计初衷是方便与系统层交互（如调用外部程序处理文件）。若调用时拼接的参数包含用户可控内容且未经转义/过滤，攻击者可以注入额外的系统命令。

### 核心函数

| 函数/写法 | 说明 |
|---|---|
| `exec()` | 执行外部程序，仅返回最后一行输出（可通过参数获取全部输出） |
| `system()` | 执行外部程序并直接输出结果 |
| `passthru()` | 执行外部程序并将原始输出直接返回给浏览器（适合处理二进制输出，如图片） |
| `popen()` | 打开一个指向进程的管道，可读可写 |
| `shell_exec()` | 执行命令并以字符串形式返回完整输出 |
| `pcntl_exec()` | 在当前进程空间执行指定程序 |
| `` ` ` ``（反引号运算符） | 等效于 `shell_exec()`，执行并返回命令输出 |

> 审计要点：只要上述任意函数的参数中拼接了用户输入，且未使用 `escapeshellarg()` / `escapeshellcmd()` 进行转义（或转义不彻底），都应视为潜在命令注入点。需特别注意：`escapeshellcmd()` 本身也在"易导致代码执行"的函数清单中，使用不当（如仅转义了部分而遗漏了命令分隔符场景）同样可能被绕过。

---

## 三、代码执行 vs 命令执行 —— 审计辨析

| 维度 | 代码执行 | 命令执行 |
|---|---|---|
| 执行的内容 | PHP 代码本身（在 PHP 解释器内执行） | 操作系统命令（交由 Shell/系统执行） |
| 典型函数 | `eval`、`assert`、`preg_replace(/e)` | `exec`、`system`、`shell_exec`、反引号 |
| 危害表现 | 可完全控制 PHP 运行时上下文（读写变量、调用任意内置/自定义函数） | 直接获得系统命令执行能力，常用于写入 Webshell、反弹 Shell |
| 共同点 | 都属于"外部输入被当作可执行指令"这一根本成因，防御核心都是**输入不可信、禁止拼接执行** |

---

## 四、代码审计案例

| 案例 | 类型 | 参考链接 |
|---|---|---|
| Yccms | 代码执行 | https://mp.weixin.qq.com/s/4i4MLsNAlMuLjySBc_rySw |
| CmsEasy | 代码执行 | https://xz.aliyun.com/t/2577 |
| BJCMS | 命令执行 | https://blog.csdn.net/qq_44029310/article/details/125860865 |
| WBCE | 命令执行 | https://developer.aliyun.com/article/1566395 |
| D-Link（路由器设备） | 命令执行 | https://xz.aliyun.com/t/2941 |
| 安恒明御安全网关 | 命令执行 | https://blog.csdn.net/smli_ng/article/details/128301954 |

> 从案例分布可以看出：代码执行多见于 **CMS 类 Web 应用**（PHP 代码层面逻辑缺陷），命令执行除了 CMS 外，也常见于**嵌入式设备/安全网关**等固件系统（往往是 Web 管理界面调用底层系统命令处理网络配置等功能时引入）。

---

## 五、总结

1. 代码执行与命令执行是 Web 安全中危害最高的两类漏洞，审计时应对"易导致代码执行的函数清单"保持高度敏感，尤其是 `eval`/`assert`/`preg_replace(/e)`/`create_function` 和各类回调型函数（`array_map`、`usort` 等易被忽视）。
2. 命令执行类函数（`exec`/`system`/`shell_exec`/反引号等）需重点核查参数拼接处是否做了 `escapeshellarg()` 转义，而非仅依赖黑名单过滤特殊字符。
3. 审计思路优先级：先搜索代码中是否存在上述高危函数调用 → 再回溯参数来源是否可被用户输入污染 → 最后验证中间是否存在有效的过滤/转义环节。
