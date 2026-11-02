---
tags: [hello-agent, 智能体的经典范式构建]
---

# ReAct 智能体的实现与调试

> [!info] 🧭 知识导航
> 所属：[[MOC - 智能体的经典范式构建]] ｜ 上级：[[MOC - Hello-Agent]] ｜ 首页：[[🏠 知识库首页]]
> 上一篇：[[01 工具的定义与实现]] ｜ 下一篇：[[03 Plan-and-Solve规划器与执行器]]
> 🔗 相关：[[03 Plan-and-Solve规划器与执行器]] · [[01 工具的定义与实现]]

> 本篇是《工具的定义与实现》的续写。
> 前文已经造好三个零件：`HelloAgentsLLM`（调用模型）、`search`（搜索工具）、`ToolExecutor`（工具箱）。
> 本篇负责把它们组装起来：设计提示词模板，实现 `ReActAgent` 的核心循环，最后总结 ReAct 的特点、局限性和调试技巧。
> 对应教程：Hello-Agents 第四章 4.2 ReAct。

---

## 一、本篇在做什么

前文的测试代码里，`tool_name` 和 `tool_input` 是**手写**的。现在要让它们由**模型自己决定**：

```
用户问题
   ↓
拼提示词（工具清单 + 问题 + 历史）──▶ 调用模型 ──▶ 得到一段文字
   ↑                                              ↓
   │                                    解析出 Thought 和 Action
   │                                              ↓
   │                              是 Finish？──是──▶ 输出最终答案，结束
   │                                      │否
   │                              取出工具函数并执行 ──▶ Observation
   │                                              ↓
   └──────────── 把 Action 和 Observation 追加进历史 ◀┘
```

这就是 ReAct（Reasoning + Acting，边想边做）的完整闭环。

---

## 二、系统提示词设计：`REACT_PROMPT_TEMPLATE`

### 2.1 代码

```python
REACT_PROMPT_TEMPLATE = """
请注意，你是一个有能力调用外部工具的智能助手。

可用工具如下:
{tools}

请严格按照以下格式进行回应:

Thought: 你的思考过程，用于分析问题、拆解任务和规划下一步行动。
Action: 你决定采取的行动，必须是以下格式之一:
- `{{tool_name}}[{{tool_input}}]`:调用一个可用工具。
- `Finish[最终答案]`:当你认为已经获得最终答案时。
- 当你收集到足够的信息，能够回答用户的最终问题时，你必须在Action:字段后使用 Finish[最终答案] 来输出最终答案。

现在，请开始解决以下问题:
Question: {question}
History: {history}
"""
```

### 2.2 思路：模板是模型的"操作手册"

模型不会自己知道有哪些工具、该输出什么格式。你要在提示词里用文字**把规则讲清楚**。这个模板有四块：

| 部分 | 作用 |
|---|---|
| 角色设定 | 告诉模型"你是一个能调用工具的助手" |
| `{tools}` | 可用工具清单，由 `getAvailableTools()` 自动生成 |
| 格式要求 | 规定每次必须输出 `Thought` 和 `Action`，且 `Action` 只能是"调用工具"或"Finish"两种形式 |
| `{question}` 和 `{history}` | 用户的问题，以及之前所有的行动和观察结果 |

其中最关键的是**格式要求**：代码要靠它把模型输出的文字解析成可执行的动作。格式一变，解析就会失败。

### 2.3 语法细节：为什么有的花括号是双层的

模板用 `.format(...)` 填充，`.format` 会把 `{名字}` 当作占位符替换：

```python
{tools}        # 占位符：之后会被替换成工具清单
{question}     # 占位符：替换成用户问题
{history}      # 占位符：替换成历史记录
{{tool_name}}  # 双层花括号：转义，输出时变成字面的 {tool_name}
```

也就是说，`{{tool_name}}[{{tool_input}}]` 最终展示给模型的是：

```
`{tool_name}[{tool_input}]`
```

这是在告诉模型"Action 的格式是：工具名[工具输入]"。如果这里写成单层 `{tool_name}`，`.format` 会找不到这个参数，直接报 `KeyError`。

> **规律：** 想在用了 `.format` 的字符串里输出字面的花括号，就写成两层 `{{ }}`。后面如果在模板里加示例（例如含有 JSON 的示例），也要注意这一点。

### 2.4 `Action` 的两种格式

```
Search[英伟达最新的GPU型号是什么]     ← 调用工具：工具名[工具输入]
Finish[最终答案是……]                  ← 结束任务
```

注意这种格式里，工具只能接收**一个**输入（方括号里的整段文字）。所以前文设计的工具都是"接收一个字符串、返回一个字符串"。这是 `[ ]` 格式的特点，也是它的局限（后文会谈到）。

---

## 三、`ReActAgent`：核心循环

### 3.1 构造方法 `__init__`

```python
class ReActAgent:
    def __init__(self, llm_client: HelloAgentsLLM, tool_executor: ToolExecutor, max_steps: int = 5):
        self.llm_client = llm_client
        self.tool_executor = tool_executor
        self.max_steps = max_steps
        self.history = []
```

| 属性 | 含义 |
|---|---|
| `llm_client` | 前文的 `HelloAgentsLLM` 对象，用来"思考" |
| `tool_executor` | 前文的 `ToolExecutor` 对象，用来找工具、列工具 |
| `max_steps` | 最多循环几步，是防止无限循环的保险，默认 5 |
| `history` | 一个列表，记录之前每一步的 Action 和 Observation |

参数后面的 `: HelloAgentsLLM` 只是类型提示，不影响运行。

**设计思想：** Agent 本身不关心"怎么调用模型""有哪些工具"，这些都从外面传进来。这种做法叫**依赖注入**，好处是以后想换模型、换工具，只需要换传进去的对象，`ReActAgent` 的代码不用改。

### 3.2 `run` 方法：完整代码

```python
def run(self, question: str):
    """运行ReAct智能体来回答一个问题。"""
    self.history = []  # 每次运行时重置历史记录
    current_step = 0   # 循环次数

    while current_step < self.max_steps:
        current_step += 1
        print(f"--- 第 {current_step} 步 ---")

        # 1. 格式化提示词
        tools_desc = self.tool_executor.getAvailableTools()
        history_str = "\n".join(self.history)
        prompt = REACT_PROMPT_TEMPLATE.format(
            tools=tools_desc,
            question=question,
            history=history_str
        )

        # 2. 调用LLM进行思考
        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages)

        if not response_text:
            print("错误:LLM未能返回有效响应。")
            break

        # 3. 解析LLM的输出
        thought, action = self._parse_output(response_text)

        if thought:
            print(f"思考: {thought}")

        if not action:
            print("警告:未能解析出有效的Action，流程终止。")
            break

        # 4. 执行Action
        if action.startswith("Finish"):
            final_answer = re.match(r"Finish\[(.*)\]", action).group(1)
            print(f"🎉 最终答案: {final_answer}")
            return final_answer

        tool_name, tool_input = self._parse_action(action)
        if not tool_name or not tool_input:
            # ... 处理无效Action格式 ...
            continue

        print(f"🎬 行动: {tool_name}[{tool_input}]")

        tool_function = self.tool_executor.getTool(tool_name)
        if not tool_function:
            observation = f"错误:未找到名为 '{tool_name}' 的工具。"
        else:
            observation = tool_function(tool_input)  # 调用真实工具

        print(f"👀 观察: {observation}")

        # 将本轮的Action和Observation添加到历史记录中
        self.history.append(f"Action: {action}")
        self.history.append(f"Observation: {observation}")

    # 循环结束
    print("已达到最大步数，流程终止。")
    return None
```

### 3.3 逐步讲解

**① 重置历史、设置计数**

```python
self.history = []
current_step = 0
```

每次提问都从零开始，避免上一个问题的历史混进来。

**② `while current_step < self.max_steps:`**

最多执行 `max_steps` 轮。万一模型一直不输出 `Finish`，程序也不会无限跑。

**③ 拼提示词（第 1 步）**

```python
tools_desc = self.tool_executor.getAvailableTools()
history_str = "\n".join(self.history)
prompt = REACT_PROMPT_TEMPLATE.format(tools=tools_desc, question=question, history=history_str)
```

- `getAvailableTools()` 返回前文那段"- 名字: 说明"的清单。
- `"\n".join(self.history)` 把历史列表拼成一整段文字。
- `.format(...)` 把三个值填进模板的占位符。

**重点：每一轮都重新拼一次完整提示词。** 模型自己没有记忆，它"记得"之前做过什么，全靠这个 `history` 被重新塞进提示词里。所以 `history` 越长，每次发给模型的内容就越长，耗时和费用也越高。

**④ 调用模型（第 2 步）**

```python
messages = [{"role": "user", "content": prompt}]
response_text = self.llm_client.think(messages=messages)
```

这里把整个提示词当作一条 `user` 消息发出去（前文是把规则放在 `system` 里，这里把规则、问题和历史都放在同一条里，写法不同，但效果都是让模型照着做）。

`think` 在出错时返回 `None`，所以后面用 `if not response_text:` 判断，失败就 `break` 退出循环。

**⑤ 解析输出（第 3 步）**

```python
thought, action = self._parse_output(response_text)
```

- 左边是**元组解包**：函数返回两个值，同时赋给两个变量。
- 方法名前面的下划线 `_` 是 Python 的约定，表示"这是类内部用的辅助方法，外面不要直接调用"，不影响运行。
- `if not action:` 表示一个有效的 `Action` 都没解析出来（模型没遵守格式），这时直接终止。

**⑥ 判断是不是结束（第 4 步）**

```python
if action.startswith("Finish"):
    final_answer = re.match(r"Finish\[(.*)\]", action).group(1)
    return final_answer
```

- `startswith("Finish")`：字符串是否以 `Finish` 开头。
- `re.match(r"Finish\[(.*)\]", action)`：正则匹配，`\[` 和 `\]` 是转义后的字面方括号，`(.*)` 是"括号里的任意内容"，被圆括号包住的部分叫**捕获组**。
- `.group(1)`：取出第一个捕获组，也就是方括号里的最终答案。
- 用 `return` 直接把答案交给调用者，整个方法结束。

**⑦ 解析工具调用并执行**

```python
tool_name, tool_input = self._parse_action(action)
```

把 `Search[英伟达最新的GPU型号]` 拆成两个部分：

```
tool_name  = "Search"
tool_input = "英伟达最新的GPU型号"
```

```python
tool_function = self.tool_executor.getTool(tool_name)
if not tool_function:
    observation = f"错误:未找到名为 '{tool_name}' 的工具。"
else:
    observation = tool_function(tool_input)
```

这里就是前文测试代码里的那几行，区别在于现在的 `tool_name` 是模型决定的。找不到工具时把错误信息当作 Observation，让模型看到之后自己改正。

**⑧ 记录历史**

```python
self.history.append(f"Action: {action}")
self.history.append(f"Observation: {observation}")
```

把本轮的行动和结果追加进去，下一轮拼提示词时模型就能看到。

**⑨ 循环结束**

`while` 正常结束（没有用 `return` 提前退出），说明用完了步数还没得出答案，打印提示并返回 `None`。

### 3.4 `break`、`continue`、`return` 的区别

| 关键字 | 作用 | 在这里用在哪 |
|---|---|---|
| `break` | 跳出整个循环 | 模型没返回、解析不出 Action |
| `continue` | 跳过本轮剩余部分，直接进入下一轮 | 工具名或输入无效 |
| `return` | 结束整个方法并返回结果 | 得到 `Finish` 最终答案 |

---

## 四、两个解析方法（教程里随后给出，需要补上）

你粘贴的代码里，`run` 调用了 `self._parse_output(...)` 和 `self._parse_action(...)`，但这两个方法没有出现，需要补进 `ReActAgent` 类里。同时文件顶部要有 `import re`，否则 `re.match` 会报 `NameError`。

下面是一个经过测试的写法（思路与教程一致，细节做了加强）：

```python
import re

class ReActAgent:
    # ... __init__ 和 run 如上 ...

    def _parse_output(self, text: str):
        """从模型输出中提取 Thought 和 Action。"""
        thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|\Z)", text, re.DOTALL)
        action_match = re.search(
            r"Action:\s*(.*?)(?=\n\s*(?:Thought:|Observation:)|\Z)", text, re.DOTALL)
        thought = thought_match.group(1).strip() if thought_match else None
        action = action_match.group(1).strip() if action_match else None
        return thought, action

    def _parse_action(self, action_text: str):
        """把 'Search[关键词]' 拆成 ('Search', '关键词')。"""
        match = re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
        if match:
            return match.group(1), match.group(2)
        return None, None
```

### 4.1 正则表达式含义

| 写法 | 含义 |
|---|---|
| `Thought:\s*` | 匹配 `Thought:` 以及后面的空白 |
| `(.*?)` | 捕获任意内容，`?` 表示**尽量少匹配**（非贪婪） |
| `(?=\nAction:\|\Z)` | 向前断言：后面必须紧跟换行加 `Action:`，或者到了文本末尾；它只用来"划边界"，自己不被取走 |
| `\Z` | 文本的结尾 |
| `re.DOTALL` | 让 `.` 也能匹配换行，这样多行内容也能取到 |
| `(\w+)\[(.*)\]` | 第一个捕获组是"字母数字下划线或汉字"组成的工具名，第二个是方括号里的内容 |
| `.group(1)`、`.group(2)` | 取第 1、第 2 个捕获组 |
| `x if 条件 else y` | 条件表达式，匹配成功就取值，否则给 `None` |

### 4.2 为什么 `Action` 的正则要加"遇到 Thought 或 Observation 就截断"

模型有时不老实，会在同一次回复里**自己续写**后面几步，甚至编造一个 Observation：

```
Thought: 我要搜一下
Action: Search[华为最新手机]
Observation: （模型凭空编的结果）
Thought: 够了
Action: Finish[...]
```

如果不截断，`Action` 会把后面这些全吃进去，工具名和输入就乱了。截断之后，只取**第一对** Thought 和 Action，编造的内容会被丢掉，真正的 Observation 由工具返回。

### 4.3 方法名速查

| 写法 | 含义 |
|---|---|
| `_parse_output` | 解析模型的整段输出，得到 `(thought, action)` |
| `_parse_action` | 解析 Action，得到 `(tool_name, tool_input)` |
| `re.search` | 在文字**任意位置**找第一个匹配 |
| `re.match` | 只从文字**开头**开始匹配 |
| `.strip()` | 去掉字符串首尾的空白 |
| `str.startswith(x)` | 是否以 x 开头 |
| `"\n".join(列表)` | 用换行把列表拼成字符串 |
| `列表.append(x)` | 往列表末尾加一项 |
| `str.format(...)` | 把值填进 `{名字}` 占位符 |
| `a, b = 函数()` | 元组解包 |

---

## 五、代码里的几个隐患，建议修改

### 5.1 缺少 `import re` 和两个解析方法

见第四节，补上即可。

### 5.2 `Action` 无效时直接 `continue`，会原地打转

教程里这里写的是 `# ... 处理无效Action格式 ...`，如果直接 `continue`，本轮什么都没有追加进 `history`，下一轮拼出来的提示词和这一轮一模一样，模型多半又给出同样的错误输出，白白消耗步数，直到用完。

**改法：** 把错误告诉模型，让它看到并改正：

```python
tool_name, tool_input = self._parse_action(action)
if not tool_name or not tool_input:
    self.history.append(f"Observation: 格式错误：无法解析 Action「{action}」，请使用 工具名[输入] 或 Finish[答案]。")
    continue
```

### 5.3 `Finish` 的提取可能报错

```python
re.match(r"Finish\[(.*)\]", action).group(1)
```

如果模型写的是 `Finish 答案`（没有方括号），`re.match` 返回 `None`，`None.group(1)` 会报 `AttributeError`。多行答案也会因为 `.` 默认不匹配换行而失败。**改法：**

```python
m = re.match(r"Finish\[(.*)\]", action, re.DOTALL)
final_answer = m.group(1) if m else action
```

### 5.4 历史里没有保存 `Thought`

教程版只把 `Action` 和 `Observation` 追加进历史，模型下一轮看不到自己上一轮的思考。如果想让它的推理更连贯，可以加上：

```python
self.history.append(f"Thought: {thought}")
self.history.append(f"Action: {action}")
self.history.append(f"Observation: {observation}")
```

代价是提示词变长，耗时和费用略增。

### 5.5 `HelloAgentsLLM` 里的笔误

如果你的文件里还是 `os.getenv("TAVILY_API_KE", 60)`，请改成 `os.getenv("LLM_TIMEOUT", 60)`（前文已讲）。

### 5.6 最后的使用示例要放在文件**最底部**

Python 从上往下执行，必须先定义类，再使用：

```python
if __name__ == '__main__':
    llm = HelloAgentsLLM()

    tool_executor = ToolExecutor()
    search_desc = "一个网页搜索引擎。当你需要回答关于时事、事实以及在你的知识库中找不到的信息时，应使用此工具。"
    tool_executor.registerTool("Search", search_desc, search)

    agent = ReActAgent(llm_client=llm, tool_executor=tool_executor)
    agent.run("华为最新的手机是哪一款？它的主要卖点是什么？")
```

如果文件里还有前面的测试块（也是 `if __name__ == '__main__':`），运行时它们会**全部**执行。想只跑 Agent，把前面的测试块暂时注释掉。

---

## 六、一次完整运行的过程（示意）

以问题"华为最新的手机是哪一款？它的主要卖点是什么？"为例：

| 步骤 | 模型输出 | 代码做的事 | 追加进 history |
|---|---|---|---|
| 1 | `Thought: 需要先搜索最新机型`<br>`Action: Search[华为最新手机型号]` | 解析出 `Search` 和关键词，调用 `search` | `Action: Search[...]`<br>`Observation: （搜索结果）` |
| 2 | `Thought: 已知型号，再搜它的卖点`<br>`Action: Search[该型号 主要卖点]` | 再次调用 `search` | 又追加一对 |
| 3 | `Thought: 信息足够了`<br>`Action: Finish[该型号是……，卖点是……]` | 检测到 `Finish`，提取答案并 `return` | 结束 |

注意第 2 步：模型是**根据第 1 步的 Observation** 才决定搜什么。这就是"走一步，看一步"。

---

## 七、ReAct 的特点、局限性与调试技巧

通过亲手实现一个 ReAct 智能体，我们不仅掌握了其工作流程，也应该对其内在机制有了更深刻的认识。任何技术范式都有其闪光点和待改进之处，下面对 ReAct 进行总结。

### 7.1 ReAct 的主要特点

1. **高可解释性**
   ReAct 最大的优点之一就是透明。通过 `Thought` 链，我们可以清晰地看到智能体每一步的"心路历程"：它为什么会选择这个工具，下一步又打算做什么。这对于理解、信任和调试智能体的行为至关重要。

2. **动态规划与纠错能力**
   与一次性生成完整计划的范式不同，ReAct 是"走一步，看一步"。它根据每一步从外部世界获得的 `Observation` 来动态调整后续的 `Thought` 和 `Action`。如果上一步的搜索结果不理想，它可以在下一步中修正搜索词，重新尝试。

3. **工具协同能力**
   ReAct 范式天然地将大语言模型的推理能力与外部工具的执行能力结合起来。LLM 负责运筹帷幄（规划和推理），工具负责解决具体问题（搜索、计算），二者协同工作，突破了单一 LLM 在知识时效性、计算准确性等方面的固有局限。

### 7.2 ReAct 的固有局限性

1. **对 LLM 自身能力的强依赖**
   ReAct 流程的成功与否，高度依赖于底层 LLM 的综合能力。如果 LLM 的逻辑推理能力、指令遵循能力或格式化输出能力不足，就很容易在 `Thought` 环节产生错误的规划，或者在 `Action` 环节生成不符合格式的指令，导致整个流程中断。

2. **执行效率问题**
   由于其循序渐进的特性，完成一个任务通常需要多次调用 LLM。每一次调用都伴随着网络延迟和计算成本。对于需要很多步骤的复杂任务，这种串行的"思考-行动"循环可能会导致较高的总耗时和费用。

3. **提示词的脆弱性**
   整个机制的稳定运行建立在一个精心设计的提示词模板之上。模板中的任何微小变动，甚至是用词的差异，都可能影响 LLM 的行为。此外，并非所有模型都能持续稳定地遵循预设的格式，这增加了在实际应用中的不确定性。

4. **可能陷入局部最优**
   步进式的决策模式意味着智能体缺乏一个全局的、长远的规划。它可能会因为眼前的 `Observation` 而选择一个看似正确但长远来看并非最优的路径，甚至在某些情况下陷入"原地打转"的循环中。

### 7.3 这些局限，在你的代码里是怎么体现的

| 局限 | 代码中的对应表现 |
|---|---|
| 对 LLM 能力强依赖 | 模型不按 `Thought/Action` 格式输出，`_parse_output` 解析不出 `Action`，流程中断 |
| 执行效率 | `while` 每一轮都调用一次 `think`；`history` 越长，提示词越长，越慢越贵 |
| 提示词脆弱 | `REACT_PROMPT_TEMPLATE` 里改一句话，模型行为就可能变；`[ ]` 格式也只支持单个输入 |
| 局部最优 | 达到 `max_steps` 还没得出答案；或者反复搜相似的词，原地打转 |

### 7.4 调试技巧

当你构建的 ReAct 智能体行为不符合预期时，可以从以下几个方面入手：

1. **检查完整的提示词**
   在每次调用 LLM 之前，将最终格式化好的、包含所有历史记录的完整提示词打印出来。这是追溯 LLM 决策源头的最直接方式。

```python
prompt = REACT_PROMPT_TEMPLATE.format(...)
print("=== 完整提示词 ===")
print(prompt)
```

2. **分析原始输出**
   当输出解析失败时（例如，正则表达式没有匹配到 `Action`），务必将 LLM 返回的原始、未经处理的文本打印出来。这能帮助你判断是 LLM 没有遵循格式，还是你的解析逻辑有误。

```python
response_text = self.llm_client.think(messages=messages)
print("=== 模型原始输出 ===")
print(repr(response_text))   # repr 能把换行、空格等不可见字符显示出来
```

3. **验证工具的输入与输出**
   检查智能体生成的 `tool_input` 是否是工具函数所期望的格式，同时也要确保工具返回的 `observation` 格式是智能体可以理解和处理的。

```python
print(f"tool_name={tool_name!r}, tool_input={tool_input!r}")
print(f"observation={observation!r}")
```

4. **调整提示词中的示例（Few-shot Prompting）**
   如果模型频繁出错，可以在提示词中加入一两个完整的"Thought-Action-Observation"成功案例，通过示例来引导模型更好地遵循你的指令。

```
示例：
Question: 珠穆朗玛峰有多高？
Thought: 这是一个事实性问题，需要搜索确认。
Action: Search[珠穆朗玛峰海拔高度]
Observation: 珠穆朗玛峰海拔 8848.86 米。
Thought: 已经得到答案。
Action: Finish[珠穆朗玛峰海拔约 8848.86 米。]
```

   > 注意：模板用了 `.format`，示例里如果含有花括号（比如 JSON），要写成双层 `{{ }}`，否则会报 `KeyError`。

5. **尝试不同的模型或参数**
   更换一个能力更强的模型，或者调整 `temperature` 参数（通常设为 0 以保证输出的确定性），有时能直接解决问题。

### 7.5 调试时的排查顺序

```
先看模型原始输出  →  格式对不对？
        ↓ 对
看解析结果        →  thought / action 是不是取对了？
        ↓ 对
看工具输入输出    →  tool_input 合不合理？observation 对不对？
        ↓ 对
看完整提示词      →  history 里的内容是否符合预期？
        ↓
换模型 / 调温度 / 加示例
```

按"由外到内、由近到远"的顺序，把问题缩小到具体一环，比盲目改提示词有效得多。

---

## 八、常见报错对照

| 现象 | 可能的原因 |
|---|---|
| `NameError: name 're' is not defined` | 文件顶部没有 `import re` |
| `AttributeError: 'ReActAgent' object has no attribute '_parse_output'` | 没有把两个解析方法写进类里 |
| `KeyError: 'tool_name'` | 模板里 `{tool_name}` 没写成双层 `{{ }}` |
| `AttributeError: 'NoneType' object has no attribute 'group'` | `Finish` 没有按 `Finish[...]` 格式写，`re.match` 返回了 `None` |
| 每一步都输出同样的东西 | 无效 Action 直接 `continue`，历史没变；应追加错误 Observation |
| "警告:未能解析出有效的Action" | 模型没遵守格式，先打印原始输出看看 |
| "已达到最大步数" | 问题太复杂、模型原地打转，或步数设得太小 |
| `'str' object is not callable` | 注册工具时传成了 `search(...)` 而不是 `search` |

---

## 九、练习

1. 在 `run` 里打印完整提示词，观察 `History` 一轮比一轮长。
2. 把 `max_steps` 改成 1，观察会发生什么；再改成 10。
3. 故意问一个**不需要搜索**的问题（比如"1 加 1 等于几"），看模型会不会直接 `Finish`。
4. 把提示词里"Action 必须是以下格式之一"这一段删掉，观察输出格式是否变乱，体会提示词的脆弱性。
5. 给提示词加上一个 Few-shot 示例，对比前后模型遵循格式的稳定程度。
6. 再注册一个工具（比如计算器），观察模型能不能自己选对工具。

---

## 十、总结

- **`REACT_PROMPT_TEMPLATE`** 是模型的操作手册：讲清楚工具、格式、问题和历史；用 `.format` 填充，字面花括号要写成 `{{ }}`。
- **`ReActAgent.run`** 的每一轮：拼提示词 → 调用模型 → 解析 Thought 和 Action → 是 `Finish` 就返回，否则执行工具 → 把 Action 和 Observation 追加进历史。
- **模型没有记忆**，它"记得"之前做过什么，靠的是每轮重新塞进提示词的 `history`。
- **解析靠格式约定**，所以格式是整个机制最脆弱的一环；无效 Action 要告诉模型，不能静默跳过。
- **ReAct 的优点**：可解释、能动态纠错、能与工具协同；**缺点**：依赖模型能力、效率低、提示词脆弱、可能陷入局部最优。
- **调试顺序**：模型原始输出 → 解析结果 → 工具输入输出 → 完整提示词，再考虑换模型、调温度、加示例。
[[03 Plan-and-Solve规划器与执行器]]