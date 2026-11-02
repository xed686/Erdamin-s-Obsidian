---
tags: [hello-agent]
---

# Agent 常用英文术语与 Python 基础速查

> [!info] 🧭 知识导航
> 所属：[[MOC - Hello-Agent]] ｜ 上级：[[🏠 知识库首页]] ｜ 首页：[[🏠 知识库首页]]
> 🔗 相关：[[00 hello-agents-ch1-notes]] · [[渗透测试英语词汇速记表]]

> 用途：读 Agent 教程、官方文档、GitHub 项目时，快速弄懂那些"行业通用"的英文单词、函数/方法的命名习惯，以及常用 Python 方法和库的用法。
> 使用建议：先通读一遍，之后遇到不认识的词再回来查。

---

## 目录

1. 基础概念：模型与调用
2. 智能体范式与流程
3. 记忆、检索与知识
4. 安全、评估与工程
5. 函数、方法、类、变量的命名习惯
6. 命名中的常见动词与缩写
7. Python 内置函数
8. 字符串、列表、字典的常用方法
9. 异常与报错单词
10. 常用标准库
11. 常用第三方库
12. 类与面向对象的常见写法
13. HTTP / API 相关词汇
14. 在 Agent 代码里把它们串起来
15. 读不懂英文名字时怎么办

---

## 一、基础概念：模型与调用

| 英文 | 中文 | 含义与用法 |
|---|---|---|
| LLM (Large Language Model) | 大语言模型 | 能理解和生成文字的模型，变量常命名为 `llm`、`llm_client` |
| Model | 模型 | 具体用哪个模型，配置里常叫 `model`、`model_id`、`MODEL_NAME` |
| Provider | 服务提供商 | 提供模型服务的平台，如 OpenAI、Anthropic、各类聚合平台 |
| Prompt | 提示词 | 发给模型的指令和内容。变量名：`prompt`、`PROMPT_TEMPLATE` |
| System Prompt | 系统提示词 | 给模型定规矩的那条消息：角色、工具、输出格式 |
| User Prompt | 用户提示词 | 用户本次说的话 |
| Prompt Template | 提示词模板 | 带 `{占位符}` 的提示词，用 `.format(...)` 填充 |
| Prompt Engineering | 提示词工程 | 通过设计提示词来引导模型行为的方法 |
| Message | 消息 | 对话里的一条，格式是 `{"role": ..., "content": ...}` |
| Role | 角色 | `system`（定规矩）、`user`（用户）、`assistant`（模型）、`tool`（工具结果） |
| Content | 内容 | 消息的正文 |
| Completion | 补全 / 生成 | 模型生成的回复；`chat.completions` 就是"对话式生成"这个接口 |
| Inference | 推理 | 使用已经训练好的模型来生成结果（区别于训练） |
| Token | 词元 | 模型处理文字的最小单位，计费和长度限制都按它算，中文大致一个字对应一到两个 token（因模型而异） |
| Context / Context Window | 上下文 / 上下文窗口 | 模型一次能"看到"的内容总量上限，提示词太长会超出 |
| Max Tokens | 最大生成长度 | 限制模型这一次最多生成多少 token |
| Temperature | 温度 | 控制随机程度，0 最稳定，越高越随机；Agent 通常设 0 |
| Top-p | 核采样 | 另一种控制随机性的参数，一般和 temperature 二选一调节 |
| Stream / Streaming | 流式输出 | 模型一小块一小块返回，像打字机；参数 `stream=True` |
| Chunk | 数据块 | 流式返回里的一小块 |
| Delta | 增量 | 这一块**新增**的内容，所以叫 `delta.content` |
| Endpoint | 接口地址 | API 的具体路径，如 `/v1/chat/completions` |
| Base URL | 基础地址 | 服务地址前缀，如 `https://xxx/v1` |
| API Key | 接口密钥 | 调用服务的身份凭证，只放 `.env`，不写进代码 |
| Timeout | 超时 | 最多等多久，到时间就放弃 |
| Rate Limit | 速率限制 | 单位时间内允许调用的次数，超了会报 429 |
| Retry | 重试 | 失败后再试一次，常配合"退避（backoff）"：每次等得更久 |
| Latency | 延迟 | 从发出请求到拿到结果的时间 |
| Cost | 成本 | 调用费用，通常和 token 数量成正比 |

---

## 二、智能体范式与流程

| 英文 | 中文 | 含义与用法 |
|---|---|---|
| Agent | 智能体 | "大模型 + 循环代码 + 工具"，能自己决定下一步做什么 |
| Agentic | 具有智能体特性的 | 形容词，如 "agentic workflow" |
| Tool | 工具 | 供模型调用的函数，如搜索、查天气、计算器 |
| Tool Calling / Function Calling | 工具调用 / 函数调用 | 模型以结构化方式表示"我要调用哪个工具、传什么参数"，比自己解析文字更稳 |
| Action | 行动 | 模型决定做的事，如 `Search[关键词]` |
| Observation | 观察结果 | 工具执行后返回的内容，会交还给模型 |
| Thought | 思考 | 模型的推理过程 |
| Finish / Final Answer | 完成 / 最终答案 | 表示任务结束并给出答案 |
| Step | 步骤 | 循环里的一轮，如 `max_steps` |
| Loop | 循环 | Agent 反复"思考—行动—观察"的过程 |
| ReAct | 推理+行动 | Reasoning + Acting，边想边做 |
| Plan-and-Solve | 先规划后执行 | 先生成完整计划，再逐步执行 |
| Planner | 规划器 | 负责把问题拆成步骤 |
| Executor | 执行器 | 负责按计划逐步执行 |
| Reflection | 反思 | 执行后检查结果、发现问题并修正 |
| Chain of Thought (CoT) | 思维链 | 让模型"一步步想"，把推理过程写出来 |
| Zero-shot | 零样本 | 提示词里不给示例 |
| Few-shot | 少样本 | 提示词里给一两个示例引导模型 |
| Workflow | 工作流 | 预先设计好的固定流程，模型只在其中几个环节出力 |
| Orchestration | 编排 | 协调多个步骤、工具、智能体的运行 |
| Orchestrator / Worker | 编排者 / 工作者 | 一个负责分派任务，其余负责具体执行 |
| Multi-Agent | 多智能体 | 多个智能体分工协作 |
| Router | 路由器 | 根据输入决定交给哪个处理环节 |
| Handoff | 交接 | 一个智能体把任务移交给另一个 |
| Human-in-the-loop | 人在回路 | 关键环节需要人确认后再继续 |
| Trajectory | 轨迹 | 一次任务里完整的"思考—行动—观察"序列 |
| Scratchpad | 草稿区 | 智能体记录中间推理的地方，常指历史记录 |
| Hallucination | 幻觉 | 模型编造不存在的内容，如编造工具名或事实 |
| Grounding | 落地 / 有据可依 | 让回答基于真实资料，而不是凭空生成 |
| Structured Output | 结构化输出 | 让模型按固定格式（如 JSON）输出，便于解析 |
| Parser | 解析器 | 把模型输出的文字拆成程序能用的结构 |

---

## 三、记忆、检索与知识

| 英文 | 中文 | 含义与用法 |
|---|---|---|
| Memory | 记忆 | 智能体保存信息的机制 |
| Short-term Memory | 短期记忆 | 当前任务里的历史记录，常见实现是 `history` 列表 |
| Long-term Memory | 长期记忆 | 跨任务保存的信息，通常存数据库 |
| State | 状态 | 当前运行到哪一步、已有什么结果 |
| History | 历史记录 | 已完成的步骤与结果 |
| RAG (Retrieval-Augmented Generation) | 检索增强生成 | 先检索资料，再让模型据此回答 |
| Retrieval | 检索 | 从资料库里找出相关内容 |
| Embedding | 向量化 / 嵌入 | 把文字转成数字向量，用来比较语义相似度 |
| Vector Store / Vector DB | 向量库 | 存放向量并支持相似度搜索的数据库 |
| Chunk / Chunking | 分块 | 把长文档切成小段再向量化 |
| Index | 索引 | 为了加快检索而建立的数据结构 |
| Rerank | 重排序 | 检索出一批结果后，再精细排序 |
| Knowledge Base | 知识库 | 供智能体查询的资料集合 |
| MCP (Model Context Protocol) | 模型上下文协议 | 让模型以统一方式接入外部工具和数据的协议 |

---

## 四、安全、评估与工程

| 英文 | 中文 | 含义与用法 |
|---|---|---|
| Prompt Injection | 提示词注入 | 在输入或搜索结果里夹带恶意指令，诱导模型偏离原意，是智能体的典型攻击面 |
| Guardrail | 护栏 | 限制智能体行为的规则，如禁止调用某些工具 |
| Sandbox | 沙箱 | 隔离的运行环境，让工具执行不影响真实系统 |
| Permission / Scope | 权限 / 范围 | 智能体能做什么、能访问什么 |
| Least Privilege | 最小权限 | 只给完成任务所需的最少权限 |
| Evaluation (Eval) | 评估 | 衡量智能体表现的测试 |
| Benchmark | 基准测试 | 用统一题目比较不同模型或智能体 |
| Observability | 可观测性 | 能看清系统内部在做什么 |
| Logging | 日志 | 记录运行信息 |
| Tracing | 追踪 | 记录一次任务经过了哪些步骤 |
| Debug | 调试 | 查找并修复问题 |
| Fine-tuning | 微调 | 用额外数据继续训练模型 |
| Deployment / Deploy | 部署 | 把程序放到服务器上长期运行 |
| Dependency | 依赖 | 程序需要的外部包 |
| Environment Variable | 环境变量 | 程序运行时可读取的"名字=值"配置 |
| Virtual Environment (venv) | 虚拟环境 | 每个项目独立的 Python 环境 |
| Config / Configuration | 配置 | 可调整的设置 |
| Mock | 模拟 / 假对象 | 测试时用假的对象代替真实的（如假模型） |

---

## 五、函数、方法、类、变量的命名习惯

### 5.1 Python 官方风格（PEP 8）

| 对象 | 风格 | 示例 |
|---|---|---|
| 变量、函数、方法 | `snake_case`（小写加下划线） | `tool_name`、`get_weather`、`parse_output` |
| 类 | `PascalCase`（每个单词首字母大写） | `ReActAgent`、`ToolExecutor` |
| 常量 | `UPPER_SNAKE_CASE`（全大写加下划线） | `MAX_STEPS`、`REACT_PROMPT_TEMPLATE` |
| 模块（文件名） | 小写加下划线 | `llm_client.py` |
| "内部使用"的方法或变量 | 前面加一个下划线 | `_parse_output`、`_history` |
| 特殊方法 | 前后各两个下划线 | `__init__`、`__str__`、`__call__` |

> **提醒：** Hello-Agents 里的 `registerTool`、`getTool`、`apiKey`、`baseUrl` 用的是 `camelCase`（驼峰），这是教程自己的风格，并不符合 PEP 8。你读教程时照它的名字写，保证能对上；以后自己写项目，建议用 `snake_case`，也就是 `register_tool`、`get_tool`、`api_key`、`base_url`。你在 GitHub 看到的 Python 项目绝大多数用后者。

### 5.2 类名常见后缀

| 后缀 | 含义 | 例子 |
|---|---|---|
| `Agent` | 智能体 | `ReActAgent`、`PlanAndSolveAgent` |
| `Client` | 负责和外部服务通信 | `HelloAgentsLLM`、`OpenAICompatibleClient` |
| `Executor` | 负责执行 | `ToolExecutor`、`Executor` |
| `Planner` | 负责规划 | `Planner` |
| `Manager` | 负责管理多个对象 | `MemoryManager`、`SessionManager` |
| `Handler` | 负责处理某类事件或请求 | `ErrorHandler` |
| `Parser` | 负责解析 | `OutputParser` |
| `Registry` | 登记簿，登记与查找 | `ToolRegistry` |
| `Builder` | 一步步构建对象 | `PromptBuilder` |
| `Factory` | 根据条件创建对象 | `AgentFactory` |
| `Config` | 配置 | `ModelConfig` |
| `Base` / `Abstract` | 基类，让别的类继承 | `BaseAgent`、`BaseTool` |
| `Wrapper` / `Adapter` | 包装、适配 | `ToolWrapper` |

### 5.3 变量名在 Agent 代码里的"标准叫法"

| 变量名 | 通常表示 |
|---|---|
| `llm` / `llm_client` | 大模型对象 |
| `client` | 与某个服务通信的客户端对象 |
| `prompt` | 最终发给模型的提示词 |
| `messages` | 消息列表 |
| `response` / `resp` | 响应 |
| `response_text` / `output` | 模型返回的文字 |
| `result` | 一般的结果 |
| `observation` | 工具执行结果 |
| `history` | 历史记录 |
| `question` / `query` | 用户的问题 / 搜索词 |
| `plan` | 计划（列表） |
| `step` / `current_step` | 当前步骤 |
| `tool` / `tools` | 工具 / 工具集合 |
| `tool_name` / `tool_input` | 工具名 / 工具输入 |
| `max_steps` | 最大步数 |
| `api_key` / `base_url` / `model_name` | 接口密钥 / 服务地址 / 模型名 |
| `timeout` | 超时秒数 |
| `temperature` | 温度 |
| `kwargs` | 一组"名字=值"形式的参数 |

---

## 六、命名中的常见动词与缩写

### 6.1 函数和方法名的动词开头

看动词，基本就能猜到这个函数干什么：

| 动词 | 含义 | 例子 |
|---|---|---|
| `get` | 获取（已有的东西） | `get_tool`、`get_weather` |
| `set` | 设置 | `set_timeout` |
| `fetch` | 从外部（网络、数据库）取回 | `fetch_page` |
| `load` | 从文件或存储里加载 | `load_dotenv`、`load_config` |
| `save` / `store` | 保存 | `save_history` |
| `read` / `write` | 读 / 写 | `read_file` |
| `create` / `make` / `build` | 创建、构造 | `build_prompt`、`create_agent` |
| `init` / `setup` | 初始化、准备 | `init_client` |
| `register` | 登记注册 | `register_tool` |
| `run` / `execute` | 运行、执行 | `run`、`execute` |
| `call` / `invoke` | 调用 | `call_llm`、`invoke_tool` |
| `parse` | 解析（把文字拆成结构） | `parse_output`、`parse_action` |
| `format` | 格式化成文字 | `format_history` |
| `extract` | 提取 | `extract_json` |
| `validate` / `check` | 校验、检查 | `validate_input` |
| `handle` / `process` | 处理 | `handle_error` |
| `update` | 更新 | `update_state` |
| `add` / `append` | 添加 / 追加 | `add_message` |
| `remove` / `delete` / `clear` | 移除 / 删除 / 清空 | `clear_history` |
| `reset` | 重置回初始状态 | `reset` |
| `search` / `find` | 搜索、查找 | `search_web` |
| `generate` | 生成 | `generate_plan` |
| `think` | 让模型思考 | `think` |
| `plan` | 规划 | `plan` |
| `convert` / `to_` | 转换 | `to_dict`、`to_json` |
| `is_` / `has_` / `can_` | 返回布尔值（是否） | `is_valid`、`has_tool` |

### 6.2 常见缩写

| 缩写 | 全称 | 含义 |
|---|---|---|
| `msg` | message | 消息 |
| `resp` / `res` | response | 响应 |
| `req` | request | 请求 |
| `cfg` / `conf` | config | 配置 |
| `env` | environment | 环境 |
| `args` | arguments | 参数 |
| `kwargs` | keyword arguments | 关键字参数 |
| `param(s)` | parameter(s) | 参数 |
| `func` / `fn` | function | 函数 |
| `cb` | callback | 回调函数 |
| `idx` | index | 序号 |
| `val` | value | 值 |
| `tmp` | temporary | 临时的 |
| `num` / `cnt` | number / count | 数量 |
| `max_` / `min_` | maximum / minimum | 最大 / 最小，如 `max_steps` |
| `str` / `int` / `dict` / `list` | 类型 | 常作为类型提示或后缀，如 `plan_str` |
| `obj` | object | 对象 |
| `util(s)` | utility | 工具函数集合（注意：这里指"小工具函数"，不是 Agent 的 Tool） |

### 6.3 变量名后缀的惯例

| 写法 | 含义 |
|---|---|
| `plan_str` | 字符串形式的计划（还没解析成列表） |
| `tools_desc` | 工具的描述（description 的缩写） |
| `history_str` | 把历史列表拼成的字符串 |
| `tool_function` | 取出来的函数对象，区别于字符串形式的 `tool_name` |

---

## 七、Python 内置函数

| 函数 | 作用 | 示例 |
|---|---|---|
| `print(x)` | 打印 | `print("你好", end="")`，`end=""` 不换行 |
| `len(x)` | 长度 | `len([1, 2, 3])` → `3` |
| `range(n)` | 生成 0 到 n-1 的整数序列 | `for i in range(5):` |
| `enumerate(x)` | 同时给出序号和内容 | `for i, step in enumerate(plan, start=1):` |
| `zip(a, b)` | 把两个序列配对 | `zip(names, values)` |
| `sorted(x)` | 返回排好序的新列表 | `sorted(scores, reverse=True)` |
| `sum` / `min` / `max` | 求和 / 最小 / 最大 | `max([1, 5, 3])` → `5` |
| `isinstance(x, 类型)` | 判断类型 | `isinstance(plan, list)` |
| `type(x)` | 查看类型 | `type("a")` → `str` |
| `any(列表)` / `all(列表)` | 任意一项为真 / 全部为真 | `all([model, key, url])` |
| `getattr(obj, "名字", 默认)` | 按名字取对象的属性 | |
| `hasattr(obj, "名字")` | 有没有这个属性 | |
| `repr(x)` | 显示"原样"，能看出换行和空格 | 调试时 `print(repr(text))` |
| `int(x)` / `float(x)` / `str(x)` | 类型转换 | `int("60")` → `60` |
| `list(x)` / `dict(x)` | 转成列表 / 字典 | |
| `open(...)` | 打开文件 | 见下节 |
| `input(...)` | 读取用户输入 | |
| `callable(x)` | 判断能不能被调用（是不是函数） | |

---

## 八、字符串、列表、字典的常用方法

### 8.1 字符串

| 方法 | 作用 | 示例 |
|---|---|---|
| `s.format(a=1)` | 把值填进 `{a}` 占位符 | `"你好{name}".format(name="折酒")` |
| f-string | 更常用的填充方式 | `f"第 {i + 1} 步"` |
| `s.split(分隔符)` | 切成列表 | `"a,b,c".split(",")` → `["a","b","c"]` |
| `分隔符.join(列表)` | 把列表拼成字符串 | `"\n".join(history)` |
| `s.strip()` | 去掉首尾空白 | `"  hi  ".strip()` → `"hi"` |
| `s.replace(旧, 新)` | 替换 | |
| `s.startswith(x)` / `s.endswith(x)` | 是否以 x 开头 / 结尾 | `action.startswith("Finish")` |
| `s.lower()` / `s.upper()` | 转小写 / 大写 | |
| `s.find(x)` | 找位置，没有返回 -1 | |
| `x in s` | 是否包含 | `"error" in text` |
| `s[a:b]` | 切片 | `text[:100]` 取前 100 个字符 |
| 转义 | `\n` 换行，`\t` 制表，`\\` 反斜杠 | |
| 三引号 `"""..."""` | 多行字符串，提示词常用 | |
| `{{ }}` | 在 `.format` 里输出字面花括号 | |

### 8.2 列表

| 方法 | 作用 | 示例 |
|---|---|---|
| `lst.append(x)` | 末尾添加一项 | `history.append("...")` |
| `lst.extend(另一个列表)` | 末尾添加多项 | |
| `lst.insert(i, x)` | 在位置 i 插入 | |
| `lst.pop()` | 取出并删除最后一项 | |
| `lst.remove(x)` | 删除第一个等于 x 的项 | |
| `lst.sort()` | 原地排序 | 区别于 `sorted(lst)`（返回新列表） |
| `lst[0]` / `lst[-1]` | 第一项 / 最后一项 | |
| `lst[:3]` | 前三项（切片） | |
| 列表推导式 | 一行生成列表 | `[f"- {n}" for n in names]` |
| `if not lst:` | 判断列表是否为空 | |

### 8.3 字典

| 方法 | 作用 | 示例 |
|---|---|---|
| `d[key]` | 取值，不存在报 `KeyError` | |
| `d.get(key, 默认)` | 取值，不存在返回默认值 | `d.get("func")` |
| `d.items()` | 同时取键和值 | `for k, v in d.items():` |
| `d.keys()` / `d.values()` | 所有键 / 所有值 | |
| `d.update(另一个字典)` | 合并 | |
| `d.setdefault(key, 默认)` | 没有这个键就设置默认值 | |
| `d.pop(key)` | 取出并删除 | |
| `key in d` | 判断有没有这个键 | `"error" in results` |
| 字典推导式 | 一行生成字典 | `{k: v for k, v in pairs}` |

---

## 九、异常与报错单词

### 9.1 try / except 写法

```python
try:
    ...                        # 可能出错的代码
except ValueError as e:        # 捕获某一类错误，e 里是错误信息
    print(e)
except Exception as e:         # 兜底：捕获所有错误
    print(e)
else:
    ...                        # 没出错时执行
finally:
    ...                        # 不管出没出错都执行（常用来清理）

raise ValueError("说明")       # 主动抛出错误
```

### 9.2 常见异常名

| 异常 | 含义 | 典型原因 |
|---|---|---|
| `NameError` | 名字没定义 | 忘了 `import`，或函数还没定义就用 |
| `KeyError` | 字典里没有这个键 | `d["x"]` 但没有 `x`；`.format` 缺参数 |
| `IndexError` | 序号超出范围 | `lst[5]` 但只有 3 项；`split(...)[1]` 没分出来 |
| `TypeError` | 类型不对 | 把字符串当函数调用；把 `None` 当列表 |
| `ValueError` | 值不合法 | `int("abc")`；参数缺失时自己抛出 |
| `AttributeError` | 没有这个属性或方法 | 对 `None` 调用 `.group()`；属性名拼错 |
| `SyntaxError` | 语法错误 | 括号、引号不配对 |
| `ImportError` / `ModuleNotFoundError` | 导入失败 | 包没装，或装错了包 |
| `FileNotFoundError` | 文件不存在 | 路径错 |
| `UnicodeDecodeError` | 编码读不出来 | 文件编码不是 UTF-8 |
| `TimeoutError` | 超时 | 网络慢或服务无响应 |
| `ConnectionError` | 连接失败 | 网络断了，或地址错误 |
| `UnboundLocalError` | 局部变量使用前未赋值 | 变量只在没执行的分支里创建过 |
| `RecursionError` | 递归太深 | 函数无限调用自己 |
| `KeyboardInterrupt` | 用户按了 Ctrl+C | |

### 9.3 报错信息里常见的英文

| 英文 | 含义 |
|---|---|
| `not defined` | 没有定义 |
| `has no attribute` | 没有这个属性或方法 |
| `object is not callable` | 这个对象不能加括号调用 |
| `NoneType` | 值是 `None`（什么都没有） |
| `invalid literal` | 不合法的字面量（常见于类型转换失败） |
| `unexpected` | 意外的 |
| `missing` | 缺少 |
| `expected` | 期望的 |
| `permission denied` | 没有权限 |
| `unauthorized` | 未授权（常是密钥不对） |
| `forbidden` | 禁止访问 |
| `not found` | 没找到 |
| `rate limit exceeded` | 超出调用频率 |
| `quota exceeded` | 额度用完 |
| `invalid argument` | 参数不合法 |
| `timeout` | 超时 |
| `deprecated` | 已弃用，将来会移除 |

---

## 十、常用标准库

### 10.1 `os`：环境变量与系统

```python
import os

os.getenv("MODEL_NAME")            # 读环境变量，没有返回 None
os.getenv("LLM_TIMEOUT", 60)       # 没有就用默认值
os.environ["KEY"] = "value"        # 设置环境变量
os.path.join("a", "b", "c.txt")    # 拼接路径
os.path.exists("a.txt")            # 路径是否存在
os.listdir(".")                    # 列出文件夹内容
```

### 10.2 `re`：正则表达式

```python
import re

re.search(r"Action: (.*)", text)            # 在任意位置找第一个匹配
re.match(r"Finish\[(.*)\]", action)         # 只从开头匹配
re.findall(r'(\w+)="([^"]*)"', s)           # 找出所有匹配
re.sub(r"\s+", " ", text)                   # 替换
m = re.search(...)
m.group(1)                                  # 取第一个括号里捕获的内容
```

| 写法 | 含义 |
|---|---|
| `.` | 任意一个字符（默认不含换行） |
| `*` / `+` / `?` | 零个或多个 / 一个或多个 / 零个或一个 |
| `.*?` | 尽量少地匹配（非贪婪） |
| `\w` / `\d` / `\s` | 字母数字下划线汉字 / 数字 / 空白 |
| `[abc]` / `[^abc]` | 属于其中之一 / 不属于其中任何一个 |
| `( )` | 捕获组，用 `.group(n)` 取出 |
| `\[` `\]` | 转义后的字面方括号 |
| `^` / `$` / `\Z` | 开头 / 结尾 / 文本末尾 |
| `(?=...)` | 向前断言：后面要是这个，但不把它取走 |
| `re.DOTALL` | 让 `.` 也匹配换行 |
| `re.IGNORECASE` | 忽略大小写 |

> 正则字符串前面加 `r`（原始字符串），反斜杠就不会被 Python 提前处理，写正则几乎都要加。

### 10.3 `json`：JSON 数据

```python
import json

json.loads('{"a": 1}')                  # 字符串 → Python 对象
json.dumps(obj, ensure_ascii=False, indent=2)   # Python 对象 → 字符串
json.load(f)                            # 从文件读
json.dump(obj, f, ensure_ascii=False)   # 写进文件
```

- `ensure_ascii=False`：让中文原样显示，不变成 `\uXXXX`。
- `indent=2`：缩进，方便阅读。
- 记忆：带 **s** 的（`loads`、`dumps`）处理**字符串（string）**；不带 s 的处理**文件**。

### 10.4 `ast.literal_eval`：安全地把字符串转成对象

```python
import ast
ast.literal_eval('["a", "b"]')          # → ['a', 'b']
```

只允许字符串、数字、列表、字典、元组、布尔值、`None` 这类"字面量"，不会执行代码，比 `eval` 安全得多。

### 10.5 `typing`：类型提示

```python
from typing import List, Dict, Any, Optional, Callable, Union

def f(a: List[str], b: Dict[str, int], c: Optional[str] = None) -> bool: ...
```

| 写法 | 含义 |
|---|---|
| `List[str]`（或 `list[str]`） | 由字符串组成的列表 |
| `Dict[str, Any]`（或 `dict[str, Any]`） | 键是字符串、值任意的字典 |
| `Optional[str]`（或 `str \| None`） | 字符串，或 `None` |
| `Callable` | 可被调用的东西（函数） |
| `Any` | 任意类型 |
| `Union[int, str]` | 整数或字符串 |

类型提示只是"标注"，写错也不影响运行，作用是让人和编辑器看得懂。

### 10.6 `time` 与 `datetime`

```python
import time
time.sleep(1)                # 暂停 1 秒（重试时常用）
time.time()                  # 当前时间戳（秒），常用来计时

start = time.time()
...
print(f"耗时 {time.time() - start:.2f} 秒")

from datetime import datetime
datetime.now().strftime("%Y-%m-%d %H:%M:%S")   # 格式化当前时间
```

### 10.7 `logging`：日志

```python
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.debug("调试信息")
logger.info("普通信息")
logger.warning("警告")
logger.error("错误")
```

级别从低到高：`DEBUG` < `INFO` < `WARNING` < `ERROR` < `CRITICAL`。设置了某个级别，低于它的就不显示。比起 `print`，日志可以分级、可以写进文件，正式项目里更常用。

### 10.8 `pathlib`：更现代的路径写法

```python
from pathlib import Path

p = Path("notes") / "a.md"             # 用 / 拼路径
p.exists()
p.read_text(encoding="utf-8")
p.write_text("内容", encoding="utf-8")
p.parent                                # 上一级目录
p.suffix                                # 扩展名
```

### 10.9 文件读写（`with open`）

```python
with open("a.txt", "r", encoding="utf-8") as f:
    content = f.read()

with open("a.txt", "w", encoding="utf-8") as f:
    f.write("内容")
```

- `"r"` 读、`"w"` 写（会覆盖）、`"a"` 追加。
- `with` 的好处：用完自动关闭文件，即使中间出错也会关。
- 中文文件记得写 `encoding="utf-8"`。

### 10.10 `dataclasses` 与 `abc`

```python
from dataclasses import dataclass

@dataclass
class Step:
    name: str
    result: str = ""
# 自动生成 __init__、__repr__ 等，适合"只装数据"的类

from abc import ABC, abstractmethod

class BaseAgent(ABC):
    @abstractmethod
    def run(self, question: str): ...
# 抽象基类：规定子类必须实现 run 方法
```

### 10.11 `functools` 与 `collections`

```python
from functools import lru_cache, wraps

@lru_cache(maxsize=128)          # 缓存函数结果，相同参数不重复计算
def f(x): ...

from collections import deque, defaultdict
deque(maxlen=10)                 # 定长队列，满了自动挤掉最旧的（常用来保存最近 N 条历史）
defaultdict(list)                # 访问不存在的键时自动给一个空列表
```

### 10.12 `asyncio`：异步（入门认识即可）

```python
import asyncio

async def fetch():                # async def：定义异步函数
    await asyncio.sleep(1)        # await：等待结果，等待期间可以去做别的
    return "done"

asyncio.run(fetch())              # 运行异步函数
```

在 Agent 里，同时调用多个工具或多个模型时会用到，这样不用一个一个排队等。

### 10.13 其他

| 库 | 用途 |
|---|---|
| `argparse` | 解析命令行参数 |
| `uuid` | 生成唯一编号，如 `uuid.uuid4()` |
| `random` | 随机数 |
| `textwrap` | 处理多行文字的缩进，如 `textwrap.dedent` |
| `sys` | 系统相关，如 `sys.exit()` |
| `copy` | 复制对象，如 `copy.deepcopy` |

---

## 十一、常用第三方库

| 库 | 安装 | 作用 | 常见用法 |
|---|---|---|---|
| `openai` | `pip install openai` | 调用大模型（兼容接口的服务也可用） | `OpenAI(api_key=..., base_url=...)`，`client.chat.completions.create(...)` |
| `python-dotenv` | `pip install python-dotenv` | 读取 `.env` | `load_dotenv()` |
| `requests` | `pip install requests` | 发送 HTTP 请求 | 见下 |
| `tavily-python` | `pip install tavily-python` | 面向 AI 的搜索服务 | `TavilyClient(api_key=...).search(query)` |
| `google-search-results` | 同名 | SerpApi 客户端，导入时写 `from serpapi import ...` | `SerpApiClient(params).get_dict()` |
| `pydantic` | `pip install pydantic` | 定义数据结构并校验，常用于约束模型的结构化输出 | `class Plan(BaseModel): steps: list[str]` |
| `langchain` / `langgraph` | 同名 | 常见的 Agent 开发框架 | 后续章节会接触 |
| `beautifulsoup4` | 同名 | 解析网页 HTML | `BeautifulSoup(html, "html.parser")` |
| `pandas` | 同名 | 处理表格数据 | |
| `numpy` | 同名 | 数值计算，向量运算 | |

> 注意：**安装名和导入名不一定相同**。`pip install google-search-results` 装好后写的是 `from serpapi import SerpApiClient`；`python-dotenv` 导入时写 `from dotenv import ...`。装不上或导入失败时，先核对这一点。

### `requests` 的常见用法

```python
import requests

resp = requests.get(url, params={"q": "关键词"}, headers={"User-Agent": "x"}, timeout=10)
resp.status_code        # 状态码，200 表示成功
resp.text               # 文本形式的内容
resp.json()             # 把 JSON 内容转成 Python 对象
resp.raise_for_status() # 状态码不是成功时抛出异常

requests.post(url, json={"a": 1}, headers={"Authorization": "Bearer 密钥"}, timeout=10)
```

- `params=`：拼在网址后面的查询参数。
- `json=`：发送 JSON 内容，会自动设置好格式。
- `timeout=`：**一定要设**，否则网络卡住时会一直等。
- `headers=`：请求头，常用来放认证信息。

### `openai` 客户端的常见用法

```python
from openai import OpenAI

client = OpenAI(api_key="...", base_url="https://.../v1", timeout=60)

response = client.chat.completions.create(
    model="模型名",
    messages=[{"role": "user", "content": "你好"}],
    temperature=0,
    stream=False,
)
text = response.choices[0].message.content      # 非流式：取完整回答

# 流式：
for chunk in client.chat.completions.create(..., stream=True):
    piece = chunk.choices[0].delta.content or ""
```

---

## 十二、类与面向对象的常见写法

| 写法 | 含义 |
|---|---|
| `class A:` | 定义类（图纸） |
| `obj = A()` | 创建对象（按图纸造出实物） |
| `__init__(self, ...)` | 创建对象时自动执行，做初始化 |
| `self` | 指"这个对象自己"，写在方法第一个参数，调用时不用手动传 |
| `self.xxx = ...` | 把数据存进对象，别的方法也能用 |
| `__str__` / `__repr__` | 定义打印这个对象时显示什么 |
| `__call__` | 让对象能像函数一样被调用：`obj(...)` |
| `@property` | 让方法可以像属性一样访问，不加括号 |
| `@staticmethod` | 静态方法，不需要 `self` |
| `@classmethod` | 类方法，第一个参数是类本身 `cls`，常用来写"另一种创建方式" |
| `class B(A):` | 继承：B 拥有 A 的功能 |
| `super().__init__()` | 在子类里调用父类的初始化 |
| `@decorator` | 装饰器：在不改函数内部的前提下，给函数"加功能" |
| 依赖注入 | 需要什么，从外面传进来，而不是自己创建（如把 `llm_client` 传给 `Planner`） |

```python
class Agent:
    def __init__(self, llm_client, max_steps=5):    # 初始化：把依赖存起来
        self.llm_client = llm_client
        self.max_steps = max_steps

    def run(self, question):                        # 公开方法：给外面调用
        return self._step(question)

    def _step(self, question):                      # 下划线开头：内部方法
        ...
```

---

## 十三、HTTP / API 相关词汇

### 13.1 常用词

| 英文 | 含义 |
|---|---|
| Request / Response | 请求 / 响应 |
| GET / POST | 两种最常用的请求方式：取数据 / 提交数据 |
| URL | 网址 |
| Header | 请求头，附带的说明信息，如认证 |
| Body / Payload | 请求体 / 载荷，真正要发送的内容 |
| Query Parameter | 查询参数，拼在网址 `?` 后面 |
| JSON | 一种通用的数据格式，键值对 |
| Authentication (Auth) | 认证，证明你是谁 |
| Authorization | 授权，你能做什么 |
| Bearer Token | 一种放在请求头里的凭证写法：`Authorization: Bearer 密钥` |
| SDK | 软件开发包，官方提供的现成调用库 |
| Webhook | 对方有事件时，主动通知你的地址 |
| Pagination | 分页，一次只返回一部分结果 |

### 13.2 常见状态码

| 状态码 | 英文 | 含义与排查 |
|---|---|---|
| 200 | OK | 成功 |
| 400 | Bad Request | 请求参数有问题 |
| 401 | Unauthorized | 密钥没有或不对 |
| 403 | Forbidden | 有身份但没有权限 |
| 404 | Not Found | 地址不对（想想路径是不是少了 `/v1`） |
| 408 | Request Timeout | 请求超时 |
| 429 | Too Many Requests | 调用太频繁，或额度用完 |
| 500 | Internal Server Error | 对方服务器出错 |
| 502 / 503 / 504 | Bad Gateway / Service Unavailable / Gateway Timeout | 对方服务暂时不可用或超时，稍后重试 |

---

## 十四、在 Agent 代码里把它们串起来

下面这段话里，几乎每个词都出现过：

> 我们用 `HelloAgentsLLM`（**LLM client**）调用模型，通过 **base URL** 和 **API key** 访问 **provider** 的 **endpoint**。`ReActAgent` 在每一轮 **loop** 里把 **tools** 描述、**question** 和 **history** 填进 **prompt template**，让模型输出 **Thought** 和 **Action**；代码用 **regex** 做 **parse**，通过 **ToolExecutor** 找到并 **execute** 对应的 **tool**，得到 **Observation** 后追加进 **history**，直到出现 **Finish** 或达到 **max_steps**。

对应的名字拆解：

```python
class ToolExecutor:                         # Tool（工具）+ Executor（执行器）
    def register_tool(...): ...             # register（登记）+ tool
    def get_tool(...): ...                  # get（获取）+ tool
    def get_available_tools(...): ...       # 获取"可用的"工具

class ReActAgent:                           # ReAct（范式）+ Agent
    def run(self, question): ...            # run：运行
    def _parse_output(self, text): ...      # 下划线：内部方法；parse：解析；output：输出
    def _parse_action(self, action): ...    # 解析 action

REACT_PROMPT_TEMPLATE = "..."               # 全大写：常量；prompt template：提示词模板
max_steps = 5                               # max：最大；steps：步数
tool_function = executor.get_tool(name)     # 取出的是函数对象
observation = tool_function(tool_input)     # 加括号执行，得到观察结果
```

---

## 十五、读不懂英文名字时怎么办

1. **拆开看。** `getAvailableTools` → `get` + `Available` + `Tools`，"获取可用的工具"。驼峰和下划线都是在帮你断词。
2. **先看动词。** 第 6 节的动词表基本能覆盖大多数函数，看到 `parse_`、`build_`、`fetch_` 就知道大概在干什么。
3. **看返回值和参数。** 名字拿不准时，看 `return` 返回什么、参数是什么，意思就出来了。
4. **用 `print(type(x))` 和 `print(repr(x))`。** 想知道一个变量到底是什么，直接打印它的类型和原样内容。
5. **看报错信息的最后一行。** 异常名和描述（比如 `KeyError: 'tool_name'`）通常已经指出了问题。
6. **查官方文档。** 库的用法以官方文档为准；遇到新名词，先查它的英文全称，再理解缩写。
7. **养成"一词一记"的习惯。** 每遇到一个新词，记下"英文 / 中文 / 在哪见过 / 一句话含义"，积累几十个之后读教程会明显顺畅。