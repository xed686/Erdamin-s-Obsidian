---
tags: [hello-agent, 智能体的经典范式构建]
---

# Hello-Agents 第一章实践笔记：从跑通第一个 Agent 到封装 LLM 客户端

> [!info] 🧭 知识导航
> 所属：[[MOC - 智能体的经典范式构建]] ｜ 上级：[[MOC - Hello-Agent]] ｜ 首页：[[🏠 知识库首页]]
> 下一篇：[[01 工具的定义与实现]]
> 🔗 相关：[[01 工具的定义与实现]] · [[Agent常用英文术语与Python基础速查]]

> 记录时间：2026-10-19
> 对应教程：Datawhale 的 Hello-Agents（环境配置 Extra07、第一章 FirstAgentTest.py、第四章 4.1 封装 LLM 客户端）
> 运行环境：Windows + PyCharm，Python 3.14，虚拟环境 `.venv`，模型服务 AIHubmix

---

## 一、这份笔记记录了什么

1. 怎么把教程里的 Agent 代码在自己电脑上跑通。
2. 遇到的每一个报错，**为什么**会出，**怎么**解决。
3. 这套代码背后的 Agent 思路（ReAct）。
4. 把"调用大模型"封装成类（`HelloAgentsLLM`）的写法，以及看不懂的方法名速查。

学完这份笔记，你应该能回答：

- Agent 到底是什么，它在代码里长什么样？
- `base_url` 为什么要写到 `/v1`？
- `.env` 里放什么，`os.getenv()` 里写什么？
- 流式输出的代码每一行在干什么？

---

## 二、整体路线图

```
准备环境（虚拟环境、.env）
        ↓
跑通 FirstAgentTest.py（ReAct：思考 → 行动 → 观察 的循环）
        ↓
踩坑：URL 路径、模型名、getenv 的用法
        ↓
封装 HelloAgentsLLM（2.py）：把调用模型这件事做成可复用的工具
        ↓
后续章节：基于它实现 ReAct、Plan-and-Solve 等范式
```

---

## 三、环境准备

### 3.1 虚拟环境（`.venv`）

教程 4.1 节让你手动创建虚拟环境。**用 PyCharm 新建项目时，它已经自动帮你创建好了。** 判断方法：运行代码时，输出的第一行路径里有 `.venv`：

```
C:\Users\Administrator\PyCharmMiscProject\.venv\Scripts\python.exe ...
```

**虚拟环境是什么：** 每个项目一间独立的工具房，装的包互不干扰。

**不用的后果：**

1. 不同项目需要不同版本的包，全局只能装一个，会有项目出问题。
2. 包越装越多，出问题难以排查。
3. 想把环境告诉别人时，说不清依赖了什么。
4. 环境乱了想重来，删掉 `.venv` 就行，不影响别的。

**使用注意：**

- 装包要装进这个环境：在 PyCharm 底部的终端里装，命令行最前面看到 `(.venv)` 说明在环境里；或用 PyCharm 的"Python 包"面板。
- 用电脑自带命令行直接 `pip install`，包会装到全局，PyCharm 里反而找不到。

### 3.2 需要安装的包

```bash
pip install requests tavily-python openai python-dotenv
```

### 3.3 `.env` 文件：存放配置

在项目文件夹（和 `.py` 文件同一目录或其上一层）新建一个名为 `.env` 的文件：

```env
OPENAI_API_KEY="你的密钥"
OPENAI_BASE_URL="https://aihubmix.com/v1"
MODEL_NAME="从模型中心复制的模型名"
TAVILY_API_KEY="你的 Tavily 密钥"
```

注意事项：

- 文件名就叫 `.env`，不是 `.env.txt`（Windows 会隐藏扩展名，容易搞错）。
- 等号两边不要有空格。
- 左边的名字（标签）要和代码里 `os.getenv("...")` 的名字**一字不差**。

---

## 四、第一个 Agent：FirstAgentTest.py

### 4.1 目标

做一个"智能旅行助手"：用户说"帮我查今天北京的天气，再根据天气推荐景点"，Agent 自己决定先查天气，再搜景点，最后给出答案。

### 4.2 核心思想：ReAct（边想边做）

模型每一轮只输出两样东西：

- `Thought`：我现在在想什么。
- `Action`：下一步要做什么。

代码把 `Action` 解析出来、真正去执行，把结果作为 `Observation` 回传给模型，如此循环，直到模型输出 `Finish[最终答案]`。

```
用户请求
   ↓
┌─────────────────────────────────┐
│ 把历史记录发给模型                │
│        ↓                          │
│ 模型输出 Thought + Action         │
│        ↓                          │
│ 是 Finish？ ── 是 → 结束，输出答案 │
│        │否                        │
│ 代码解析 Action，执行对应的工具函数 │
│        ↓                          │
│ 结果记为 Observation，追加到历史   │
└──────────── 回到顶部 ────────────┘
```

**三个关键事实：**

1. **模型自己不会"动手"。** 它只生成文字。把 `Action: get_weather(city="北京")` 这行文字解析出来、找到同名函数去执行的，是你的代码。
2. **模型没有记忆。** 每一轮都要把完整的 `prompt_history` 重新发给它，它才"记得"之前查过什么。
3. **工具列表和格式约定写在 System Prompt 里。** 模型靠理解这些文字来遵守约定，格式不对，代码就解析不出 `Action`。

### 4.3 完整代码（密钥改为从 `.env` 读取）

```python
import os
import re
import requests
from dotenv import load_dotenv
from openai import OpenAI
from tavily import TavilyClient

load_dotenv()

# ---------- 1. System Prompt：告诉模型有哪些工具、输出什么格式 ----------
AGENT_SYSTEM_PROMPT = """
你是一个智能旅行助手。你的任务是分析用户的请求，并使用可用工具一步步地解决问题。

# 可用工具:
- `get_weather(city: str)`: 查询指定城市的实时天气。
- `get_attraction(city: str, weather: str)`: 根据城市和天气搜索推荐的旅游景点。

# 输出格式要求:
你的每次回复必须严格遵循以下格式，包含一对Thought和Action：

Thought: [你的思考过程和下一步计划]
Action: [你要执行的具体行动]

Action的格式必须是以下之一：
1. 调用工具：function_name(arg_name="arg_value")
2. 结束任务：Finish[最终答案]

# 重要提示:
- 每次只输出一对Thought-Action
- Action必须在同一行，不要换行
- 当收集到足够信息可以回答用户问题时，必须使用 Action: Finish[最终答案] 格式结束

请开始吧！
"""


# ---------- 2. 工具函数 ----------
def get_weather(city: str) -> str:
    """通过 wttr.in 查询真实天气。"""
    url = f"https://wttr.in/{city}?format=j1"
    try:
        response = requests.get(url)
        response.raise_for_status()
        data = response.json()
        current_condition = data['current_condition'][0]
        weather_desc = current_condition['weatherDesc'][0]['value']
        temp_c = current_condition['temp_C']
        return f"{city}当前天气：{weather_desc}，气温{temp_c}摄氏度"
    except requests.exceptions.RequestException as e:
        return f"错误：查询天气时遇到网络问题 - {e}"
    except (KeyError, IndexError) as e:
        return f"错误：解析天气数据失败，可能是城市名称无效 - {e}"


def get_attraction(city: str, weather: str) -> str:
    """根据城市和天气，用 Tavily 搜索景点推荐。"""
    api_key = os.environ.get("TAVILY_API_KEY")
    if not api_key:
        return "错误：未配置TAVILY_API_KEY。"

    tavily = TavilyClient(api_key=api_key)
    query = f"'{city}' 在'{weather}'天气下最值得去的旅游景点推荐及理由"

    try:
        response = tavily.search(query=query, search_depth="basic", include_answer=True)
        if response.get("answer"):
            return response["answer"]

        formatted_results = []
        for result in response.get("results", []):
            formatted_results.append(f"- {result['title']}: {result['content']}")
        if not formatted_results:
            return "抱歉，没有找到相关的旅游景点推荐。"
        return "根据搜索，为您找到以下信息：\n" + "\n".join(formatted_results)
    except Exception as e:
        return f"错误：执行Tavily搜索时出现问题 - {e}"


# 工具字典：名字 -> 函数，方便按名字调用
available_tools = {
    "get_weather": get_weather,
    "get_attraction": get_attraction,
}


# ---------- 3. LLM 客户端 ----------
class OpenAICompatibleClient:
    """调用任何兼容 OpenAI 接口的服务。"""

    def __init__(self, model: str, api_key: str, base_url: str):
        self.model = model
        self.client = OpenAI(api_key=api_key, base_url=base_url)

    def generate(self, prompt: str, system_prompt: str) -> str:
        print("正在调用大语言模型...")
        try:
            messages = [
                {'role': 'system', 'content': system_prompt},
                {'role': 'user', 'content': prompt},
            ]
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                stream=False,
            )
            answer = response.choices[0].message.content
            print("大语言模型响应成功。")
            return answer
        except Exception as e:
            print(f"调用LLM API时发生错误: {e}")
            return "错误：调用语言模型服务时出错。"


# ---------- 4. 配置（全部从 .env 读取）----------
API_KEY = os.getenv("OPENAI_API_KEY")
BASE_URL = os.getenv("OPENAI_BASE_URL")   # 注意：要写到 /v1，且是小写 v
MODEL_ID = os.getenv("MODEL_NAME")

llm = OpenAICompatibleClient(model=MODEL_ID, api_key=API_KEY, base_url=BASE_URL)

# ---------- 5. 主循环 ----------
user_prompt = "你好，请帮我查询一下今天北京的天气，然后根据天气推荐一个合适的旅游景点。"
prompt_history = [f"用户请求: {user_prompt}"]

print(f"用户输入: {user_prompt}\n" + "=" * 40)

for i in range(5):  # 最多循环 5 次，防止无限循环
    print(f"--- 循环 {i + 1} ---\n")

    # 5.1 把历史拼成完整 prompt，发给模型
    full_prompt = "\n".join(prompt_history)
    llm_output = llm.generate(full_prompt, system_prompt=AGENT_SYSTEM_PROMPT)

    # 5.2 模型可能多输出了后面的 Thought-Action，截断只保留第一对
    match = re.search(
        r'(Thought:.*?Action:.*?)(?=\n\s*(?:Thought:|Action:|Observation:)|\Z)',
        llm_output, re.DOTALL)
    if match:
        truncated = match.group(1).strip()
        if truncated != llm_output.strip():
            llm_output = truncated
            print("已截断多余的 Thought-Action 对")
    print(f"模型输出:\n{llm_output}\n")
    prompt_history.append(llm_output)

    # 5.3 解析 Action
    action_match = re.search(r"Action: (.*)", llm_output, re.DOTALL)
    if not action_match:
        observation = "错误: 未能解析到 Action 字段。请确保你的回复严格遵循 'Thought: ... Action: ...' 的格式。"
        observation_str = f"Observation: {observation}"
        print(f"{observation_str}\n" + "=" * 40)
        prompt_history.append(observation_str)
        continue
    action_str = action_match.group(1).strip()

    # 5.4 如果是 Finish，结束
    if action_str.startswith("Finish"):
        final_answer = re.match(r"Finish\[(.*)\]", action_str).group(1)
        print(f"任务完成，最终答案: {final_answer}")
        break

    # 5.5 否则解析出工具名和参数，执行工具
    tool_name = re.search(r"(\w+)\(", action_str).group(1)
    args_str = re.search(r"\((.*)\)", action_str).group(1)
    kwargs = dict(re.findall(r'(\w+)="([^"]*)"', args_str))

    if tool_name in available_tools:
        observation = available_tools[tool_name](**kwargs)
    else:
        observation = f"错误：未定义的工具 '{tool_name}'"

    # 5.6 记录观察结果，进入下一轮
    observation_str = f"Observation: {observation}"
    print(f"{observation_str}\n" + "=" * 40)
    prompt_history.append(observation_str)
```

### 4.4 一次成功运行的过程解读

| 循环 | 模型输出的 Action | 代码做了什么 | Observation |
|---|---|---|---|
| 1 | `get_weather(city="北京")` | 解析出函数名和参数，调用 wttr.in | 北京当前天气：Moderate rain，气温 17 摄氏度 |
| 2 | `get_attraction(city="北京", weather="Moderate rain")` | 调用 Tavily 搜索下雨天景点 | 一段景点推荐（英文） |
| 3 | `Finish[...]` | 检测到 Finish，跳出循环 | 打印最终答案 |

**每一轮发生了什么：**

- 循环 1：模型想"需要先查天气"，输出一行文字；代码用正则把它拆成函数名 `get_weather` 和参数 `city="北京"`，真的去查天气。
- 循环 2：代码把**到目前为止的全部历史**再发给模型，模型看到下雨，决定调用景点工具，并把天气当作参数传进去。
- 循环 3：模型判断信息够了，输出 `Finish[...]`，把英文结果整理成中文并选出一个景点。

### 4.5 值得记住的几点

- **Agent = 大模型 + 一段循环代码 + 一组工具函数。**
- **格式约定很脆弱。** 格式一旦不对，解析不出 `Action`，后面的流程就断了。这也是后续章节引入更稳妥的工具调用方式（Function Calling）的原因。
- **循环上限 5 次是保险。** 万一模型一直不输出 `Finish`，程序也不会无限跑。

---

## 五、踩坑记录（按出现顺序）

### 坑 1：`BASE_URL` 没写 `/v1`

**现象：** 循环里反复出现这条错误：

```
调用LLM API时发生错误: <?xml version='1.0' ...><Error><Code>InvalidArgument</Code>
<Message>Invalid argument.</Message><Details>POST object expects Content-Type multipart/form-data</Details></Error>
```

**原因：** 代码里写的是 `BASE_URL = "https://aihubmix.com"`。`OpenAI` 客户端会把 `/chat/completions` 接在 `base_url` 后面，拼出来是 `https://aihubmix.com/chat/completions`。网站根目录下没有这个接口，请求被网站自己的程序接住，回了一段无关的"文件上传"错误。

**解决：** `BASE_URL = "https://aihubmix.com/v1"`。

### 坑 2：把 `v1` 写成了大写 `V1`

**现象：** 改了之后，报错一模一样。

**原因：** 网址的路径部分**区分大小写**。服务器上只有 `/v1`，没有 `/V1`。

**解决：** 全部小写，结尾不要多加斜杠或空格。

### 坑 3：错误重复了 5 次，看不出重点

**原因：** `generate` 出错时只是返回一句"错误：调用语言模型服务时出错"的字符串，程序没有停。这句话解析不出 `Action`，主循环就把它当成"格式不对"，让模型重来，一共循环 5 次。退出代码为 0，只说明程序没崩溃，不代表成功。

**经验：** 看到重复的报错，**先看第一条红色/错误信息**，不要被后面重复的内容带偏。

### 坑 4：模型名要原样复制

教程推荐的免费模型名可能与你看到的不同。去 AIHubmix 的模型中心，筛选"免费"，把模型名**原样复制**。报错变成"模型不存在"这类内容时，就是模型名的问题。

**排查技巧：** 先用最小的测试代码确认连通，把"网络和密钥"与"Agent 逻辑"分开排查：

```python
from openai import OpenAI

client = OpenAI(api_key="你的key", base_url="https://aihubmix.com/v1")
response = client.chat.completions.create(
    model="从模型中心复制的模型名",
    messages=[{"role": "user", "content": "Hello"}],
    max_tokens=10,
)
print(response.choices[0].message.content)
```

### 坑 5：`os.getenv()` 里写成了"值"，而不是"名字"

**现象：**

```
模型ID、API密钥和服务地址必须被提供或在.env文件中定义。
```

**错误写法：**

```python
os.getenv("qwen3.8-flash-free")      # 把值当成标签名
```

**原理：** `.env` 是一排带标签的储物格：

```
MODEL_NAME      →  里面放着模型名
OPENAI_API_KEY  →  里面放着密钥
OPENAI_BASE_URL →  里面放着服务地址
```

`os.getenv("标签名")` 的意思是"去找叫这个名字的储物格，把里面的东西给我"。传进去的是**标签名**，返回的才是**值**。传了值进去，找不到这个标签，就返回 `None`。

**正确写法：**

```python
self.model = model or os.getenv("MODEL_NAME")
apiKey = apiKey or os.getenv("OPENAI_API_KEY")
baseUrl = baseUrl or os.getenv("OPENAI_BASE_URL")
```

**排查顺序：**

1. 在 `load_dotenv()` 之后临时加一行 `print(os.getenv("MODEL_NAME"))`，打印出 `None` 说明 `.env` 没被读到。
2. 文件名是不是真的叫 `.env`。
3. 标签名是否一字不差，包括大小写，等号两边不要有空格。

### 坑 6：超时时间读错了标签

**错误写法：**

```python
timeout = timeout or int(os.getenv("TAVILY_API_KEY", 60))
```

**现象：**

```
invalid literal for int() with base 10: 'tvly-dev-...'
```

**原因：** `timeout` 要的是"等多少秒"，却去读了 Tavily 的密钥，把一长串字母数字交给 `int()` 转整数，当然转不了。

**为什么只看到一行报错：** `int()` 抛出的也是 `ValueError`，被最下面的 `except ValueError as e: print(e)` 接住只打印了文字，程序算正常结束，退出代码仍是 0。同一类型的异常可能来自不同地方，只靠 `print(e)` 看不出是哪一行出的。

**正确写法：**

```python
timeout = timeout or int(os.getenv("LLM_TIMEOUT", 60))
```

`.env` 里不用写 `LLM_TIMEOUT`，找不到就用默认值 60。如果以后模型很慢、60 秒不够，在 `.env` 里加一行 `LLM_TIMEOUT=120` 即可，代码不用改。

**这一行改动不影响 Tavily：** Tavily 的密钥是在 `get_attraction` 里用 `os.environ.get("TAVILY_API_KEY")` 读取的，和 `timeout` 互不相关。

**总规律：** 两次报错同一类原因，都是把"值"和"标签名"搞混。`os.getenv("xxx")` 括号里一定是 `.env` 左边的名字。

---

## 六、URL 的结构（`/v1` 到底是什么）

```
https://aihubmix.com/v1/chat/completions
└──── 域名 ────┘└前缀┘└──── 具体接口 ────┘
```

- **域名**：找到这台服务器。
- **`/v1`**：API 的第 1 版，告诉服务器"我要的是 API 服务，不是网页"。**它不是一个接口，而是所有 API 接口共同的路径前缀。**
- **`/chat/completions`**：具体接口，"请对话生成回答"。同一前缀下还有 `/v1/models`（查看有哪些模型）、`/v1/embeddings`（把文字转成向量）等。

**你写的和实际请求的区别：**

```
你在代码里填的 BASE_URL：   https://aihubmix.com/v1
客户端自动补在后面的部分：                         /chat/completions
实际发出请求的完整地址：     https://aihubmix.com/v1/chat/completions
```

- 在你自己写的 `BASE_URL` 里，`v1` 在**最后**，写到 `/v1` 就停，后面的部分不用管。
- 在完整地址里，`v1` 在中间。
- 路径是从大范围到小范围一层层往下找的：服务器 → API 第 1 版 → 具体接口。所以版本号一定在具体接口前面。

**为什么带版本号：** 将来接口要做不兼容的大改动时，可以新开 `/v2`，旧程序继续用 `/v1`，不会一夜之间全部失效。

---

## 七、封装 LLM 客户端：`HelloAgentsLLM`（2.py）

### 7.1 设计思路

一句话：**把"调用大模型"这件事包装成一个可以反复使用的工具。** 后面写的所有 Agent，只需要 `llm.think(messages)` 就能拿到回答，不用关心密钥、地址、流式输出这些细节。

与前一份代码的对比：

| | `1.py` | `2.py` |
|---|---|---|
| 调用方式 | 直接写在主程序里 | 封装成一个类 |
| 方法名 | `generate` | `think` |
| 输出 | 一次性返回 | 流式输出 |
| 配置 | 开始时写死在代码里 | 从 `.env` 读取 |

这样改的原因：Agent 后面会越来越复杂（ReAct、Plan-and-Solve 等），每一种都要调用大模型。把调用的细节藏进类里，上层代码就只关心"怎么思考、怎么用工具"。

### 7.2 完整代码

```python
import os
from openai import OpenAI
from dotenv import load_dotenv
from typing import List, Dict

# 加载 .env 文件中的环境变量
load_dotenv()


class HelloAgentsLLM:
    """
    为本书 "Hello Agents" 定制的LLM客户端。
    它用于调用任何兼容OpenAI接口的服务，并默认使用流式响应。
    """

    def __init__(self, model: str = None, apiKey: str = None, baseUrl: str = None, timeout: int = None):
        """
        初始化客户端。优先使用传入参数，如果未提供，则从环境变量加载。
        """
        self.model = model or os.getenv("MODEL_NAME")
        apiKey = apiKey or os.getenv("OPENAI_API_KEY")
        baseUrl = baseUrl or os.getenv("OPENAI_BASE_URL")
        timeout = timeout or int(os.getenv("LLM_TIMEOUT", 60))

        if not all([self.model, apiKey, baseUrl]):
            raise ValueError("模型ID、API密钥和服务地址必须被提供或在.env文件中定义。")

        self.client = OpenAI(api_key=apiKey, base_url=baseUrl, timeout=timeout)

    def think(self, messages: List[Dict[str, str]], temperature: float = 0) -> str:
        """
        调用大语言模型进行思考，并返回其响应。
        """
        print(f"🧠 正在调用 {self.model} 模型...")
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                temperature=temperature,
                stream=True,
            )

            # 处理流式响应
            print("✅ 大语言模型响应成功:")
            collected_content = []
            for chunk in response:
                if not chunk.choices:
                    continue
                content = chunk.choices[0].delta.content or ""
                print(content, end="", flush=True)
                collected_content.append(content)
            print()  # 在流式输出结束后换行
            return "".join(collected_content)

        except Exception as e:
            print(f"❌ 调用LLM API时发生错误: {e}")
            return None


# --- 客户端使用示例 ---
if __name__ == '__main__':
    try:
        llmClient = HelloAgentsLLM()

        exampleMessages = [
            {"role": "system", "content": "You are a helpful assistant that writes Python code."},
            {"role": "user", "content": "写一个快速排序算法"}
        ]

        print("--- 调用LLM ---")
        responseText = llmClient.think(exampleMessages)
        if responseText:
            print("\n\n--- 完整模型响应 ---")
            print(responseText)

    except ValueError as e:
        print(e)
```

### 7.3 逐段讲解

**导入部分**

- `os`：读取环境变量。
- `OpenAI`：官方客户端类，很多服务兼容它的格式，所以也能调用别的平台的模型。
- `load_dotenv`：读取 `.env` 的工具。
- `List, Dict`：只用来写**类型提示**，不影响运行，作用是告诉读代码的人"这里应该传什么类型"。

**`load_dotenv()`**

打开 `.env`，把每行 `名字=值` 放进当前程序的环境变量，之后 `os.getenv("名字")` 才能取到值。没有这一行，`getenv` 什么都读不到。

**`__init__`：创建对象时的准备工作**

- `__init__` 是 Python 固定的名字，创建 `HelloAgentsLLM()` 时自动执行。
- `model or os.getenv(...)`：**传了参数就用传的，没传才去读环境变量**。因为 `or` 在左边有值时不会去算右边。
- 为什么只有 `model` 前面有 `self.`：`model` 在 `think` 方法里还要用，所以存进对象；密钥和地址只在创建客户端时用一次，用完不需要保留。
- `all([...])`：列表里每一项都"有内容"才返回 `True`，有任何一个是 `None` 或空字符串就是 `False`。
- `raise ValueError(...)`：主动抛出错误，意思是"配置不全，直接停下来告诉你原因"，比等到调用时才出莫名其妙的错误好排查。
- `OpenAI(...)`：创建真正干活的客户端，存进 `self.client`。

**`think`：真正调用模型**

- `think` 只是作者起的名字，因为在 Agent 里"调用大模型"相当于"让它思考一次"。
- `messages` 是一个列表，每项是一个字典：

```python
[
    {"role": "system", "content": "你是一个助手"},
    {"role": "user", "content": "写一个快速排序"}
]
```

`role` 有三种：

| role | 含义 |
|---|---|
| `system` | 给模型定规矩（你之前的 ReAct 里的 System Prompt 就放在这里） |
| `user` | 用户说的话 |
| `assistant` | 模型之前的回答 |

- `temperature=0`：控制回答的"随机程度"。0 表示尽量每次给出相同、稳定的答案，写代码和做 Agent 时通常希望稳定。

**`self.client.chat.completions.create` 为什么这么长**

一层层往下找：

- `client`：客户端对象。
- `.chat`：其中"对话"这一类功能。
- `.completions`：对话里"生成回复"这一项。
- `.create`：发起一次请求。

它对应的就是前面讲的完整地址 `.../v1/chat/completions`，名字不是巧合。

**流式输出（`stream=True`）**

没有流式时，模型想完才一次返回，你要干等；有了流式，它**一小块一小块地返回**，像打字机。所以 `response` 不是字符串，而是可以用 `for` 遍历的"数据块序列"：

```python
for chunk in response:
    if not chunk.choices:
        continue
    content = chunk.choices[0].delta.content or ""
    print(content, end="", flush=True)
    collected_content.append(content)
```

| 代码 | 含义 |
|---|---|
| `chunk` | 每一小块数据 |
| `chunk.choices` | 这一块给出的候选回答列表，通常只有一个，所以取 `[0]`；有的块是控制信息，列表为空，所以先 `if not chunk.choices: continue` 跳过 |
| `.delta.content` | 这一块**新增**的几个字（`delta` 是"增量"）；有的块没有文字，值是 `None`，所以后面加 `or ""` 当空字符串，避免出错 |
| `print(..., end="", flush=True)` | `end=""` 打印完不换行，让字能接着往后连；`flush=True` 立刻显示，不要攒在缓冲区里，否则看不到打字机效果 |
| `collected_content.append(content)` | 同时把每一小块存进列表 |

```python
return "".join(collected_content)
```

`"".join(列表)` 用空字符串当"胶水"，把列表里所有小块拼成完整字符串。用列表再 `join`，比反复 `+=` 拼接字符串更高效，也更常见。

**异常处理**

任何地方出错，就把原因打印出来，返回 `None`，让调用的人知道这次失败了。

**`if __name__ == '__main__':`**

这个固定写法的意思是：**只有直接运行这个文件时，才执行下面的内容。** 以后别的文件 `import` 这个类来用，下面的测试代码不会被执行，所以它是放"自己测试一下"的最佳位置。

---

## 八、方法名与语法速查表

| 写法 | 含义 |
|---|---|
| `load_dotenv()` | 把 `.env` 里的内容读进环境变量 |
| `os.getenv("名字")` | 按标签名取环境变量的值，找不到返回 `None` |
| `os.getenv("名字", 60)` | 找不到时返回默认值 60 |
| `a or b` | a 有内容就用 a，否则用 b |
| `all([...])` | 列表里每一项都有内容才为 `True` |
| `raise ValueError("...")` | 主动抛出错误并停下 |
| `__init__` | 创建对象时自动执行的初始化方法 |
| `self.xxx` | 存在对象里的变量，其他方法也能用 |
| `client.chat.completions.create(...)` | 发起一次对话请求，对应接口 `/v1/chat/completions` |
| `stream=True` | 让模型一小块一小块地返回 |
| `chunk.choices[0].delta.content` | 流式返回里，这一块新增的文字 |
| `print(x, end="", flush=True)` | 不换行、立即显示 |
| `"".join(列表)` | 把列表里的字符串拼成一个 |
| `temperature` | 随机程度，0 最稳定 |
| `if __name__ == '__main__':` | 只在直接运行本文件时执行 |
| `re.search(...)` | 用正则表达式在文字里找匹配内容 |
| `raise_for_status()` | 响应状态码不是成功时抛出异常 |

---

## 九、动手练习（建议依次做）

1. 在 `think` 方法里的 `for chunk in response:` 下面加一行 `print(chunk)`，亲眼看每个数据块长什么样，`choices`、`delta`、`content` 就一目了然了。
2. 在 `1.py` 的主循环里，把 `full_prompt` 打印出来，观察它一轮比一轮长，理解"模型没有记忆"。
3. 把北京换成别的城市，看模型的 Thought 怎么变。
4. 新增一个自己的工具函数（比如 `get_time()`），加进 `available_tools` 和 System Prompt，再问一个需要用到它的问题。
5. 故意删掉 System Prompt 里"Action 必须在同一行"这句，看会不会出现解析失败，体会"格式约定"有多重要。
6. 把 `1.py` 里的 `OpenAICompatibleClient` 换成 `2.py` 的 `HelloAgentsLLM`，体会封装后上层代码有多简洁。

---

## 十、术语表

| 术语 | 解释 |
|---|---|
| Agent（智能体） | 大模型 + 循环代码 + 工具函数，能自己决定下一步做什么 |
| ReAct | Reasoning + Acting，边想边做的 Agent 范式 |
| Thought / Action / Observation | 思考 / 行动 / 观察结果 |
| System Prompt | 给模型定规矩的第一条消息，写明角色、工具、输出格式 |
| 工具（Tool） | 供模型调用的普通 Python 函数，如查天气、搜索 |
| base_url | 服务地址，写到 `/v1` 为止 |
| API Key | 调用服务的密钥，只放在 `.env` 里 |
| 环境变量 | 程序运行时可读取的"名字=值"配置 |
| 流式输出（stream） | 模型一小块一小块地返回，像打字机 |
| chunk / delta | 流式返回的一小块 / 其中新增的内容 |
| temperature | 控制回答随机程度的参数 |
| 虚拟环境（.venv） | 每个项目独立的 Python 环境 |
| 正则表达式 | 用规则在文字里查找并提取内容的工具 |

---

## 十一、一句话总结

> **Agent 不神秘：模型只负责"说"下一步做什么，真正动手的是外面的循环和函数。** 封装 LLM 客户端，是为了让后面的所有 Agent 都能用同一个"思考"接口。遇到报错时，先看第一条错误，再把"网络配置"和"程序逻辑"分开排查。

[[01 工具的定义与实现]]