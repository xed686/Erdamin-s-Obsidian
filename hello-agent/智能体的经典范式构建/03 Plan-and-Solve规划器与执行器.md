---
tags: [hello-agent, 智能体的经典范式构建]
---

# Plan-and-Solve：规划器、执行器与组装

> [!info] 🧭 知识导航
> 所属：[[MOC - 智能体的经典范式构建]] ｜ 上级：[[MOC - Hello-Agent]] ｜ 首页：[[🏠 知识库首页]]
> 上一篇：[[02 ReAct智能体的实现与调试]]
> 🔗 相关：[[02 ReAct智能体的实现与调试]] · [[00 hello-agents-ch1-notes]]

> 本篇是《ReAct智能体的实现与调试》的续写，对应教程第四章 4.3 Plan-and-Solve。
> 内容：`Planner`（规划器）、`Executor`（执行器），以及把二者组装起来的 `PlanAndSolveAgent`。前面已经讲过的 `HelloAgentsLLM`、`.format`、`self` 等不再重复。

---

## 一、Plan-and-Solve 是什么

**先规划，再执行。** 面对一个复杂问题，不再"走一步看一步"，而是：

1. **规划阶段：** 让模型把问题一次性拆成一份按顺序排好的步骤清单。
2. **执行阶段：** 拿着这份清单，一步一步完成，每一步的结果带进下一步。

```
用户问题
   ↓
Planner：生成计划  ["步骤1", "步骤2", "步骤3"]
   ↓
Executor：循环执行
   ├─ 步骤 1：提示词 = 问题 + 完整计划 + 历史（无）           + 当前步骤 1 → 结果 1
   ├─ 步骤 2：提示词 = 问题 + 完整计划 + 历史（步骤1和结果1）   + 当前步骤 2 → 结果 2
   └─ 步骤 3：提示词 = 问题 + 完整计划 + 历史（步骤1、2和结果）+ 当前步骤 3 → 结果 3
   ↓
最后一步的结果 = 最终答案
```

### 与 ReAct 的对比

| | ReAct | Plan-and-Solve |
|---|---|---|
| 决策方式 | 每一步现想下一步 | 先一次性定好全部步骤 |
| 记录的历史 | Action 和 Observation | 步骤和结果 |
| 工具 | 每一步都可以调用 | 本节的执行器里没有接入工具 |
| 模型调用次数 | 步数不确定 | 1 次规划 + 计划有几步就调用几次 |
| 擅长 | 需要根据中间结果灵活调整 | 步骤明确、逻辑有序的任务 |
| 风险 | 缺乏长远规划，可能原地打转 | 计划一旦出错，后面全受影响 |

---

## 二、规划阶段：`Planner`

### 2.1 提示词模板 `PLANNER_PROMPT_TEMPLATE`

```python
PLANNER_PROMPT_TEMPLATE = """
你是一个顶级的AI规划专家。你的任务是将用户提出的复杂问题分解成一个由多个简单步骤组成的行动计划。
请确保计划中的每个步骤都是一个独立的、可执行的子任务，并且严格按照逻辑顺序排列。
你的输出必须是一个Python列表，其中每个元素都是一个描述子任务的字符串。

问题: {question}

请严格按照以下格式输出你的计划,```python与```作为前后缀是必要的:
```python
["步骤1", "步骤2", "步骤3", ...]
```
"""
```

**思路：**

1. **设定角色和任务：** 要把复杂问题拆成多个简单步骤，每步是独立的、可执行的子任务，按逻辑顺序排列。
2. **规定输出类型：** 必须是 Python 列表，每个元素是描述子任务的字符串。
3. **规定输出格式：** 必须用 ```` ```python ```` 和 ```` ``` ```` 包起来。

**为什么要让模型加代码块标记：** 模型的输出里除了列表，还可能夹带"好的，下面是计划："之类的话。有了固定的前后缀，代码就能准确地把列表那一段"裁"出来，不受闲话影响。

**占位符：** 只有一个 `{question}`。示例里用的是方括号 `[...]`，不是花括号，所以不用担心 `.format` 的转义问题。

### 2.2 `Planner` 类

```python
import ast

class Planner:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def plan(self, question: str) -> list[str]:
        """根据用户问题生成一个行动计划。"""
        prompt = PLANNER_PROMPT_TEMPLATE.format(question=question)

        messages = [{"role": "user", "content": prompt}]

        print("--- 正在生成计划 ---")
        response_text = self.llm_client.think(messages=messages) or ""

        print(f"✅ 计划已生成:\n{response_text}")

        try:
            plan_str = response_text.split("```python")[1].split("```")[0].strip()
            plan = ast.literal_eval(plan_str)
            return plan if isinstance(plan, list) else []
        except (ValueError, SyntaxError, IndexError) as e:
            print(f"❌ 解析计划时出错: {e}")
            print(f"原始响应: {response_text}")
            return []
        except Exception as e:
            print(f"❌ 解析计划时发生未知错误: {e}")
            return []
```

### 2.3 逐行讲解

**`import ast`**

`ast.literal_eval` 来自这个标准库，**必须导入**，否则会报 `NameError: name 'ast' is not defined`。这个错误会被最后的 `except Exception` 接住，你只会看到"解析计划时发生未知错误: name 'ast' is not defined"，要看清这句话。

**`__init__`：把模型存进对象**

```python
def __init__(self, llm_client):
    self.llm_client = llm_client
```

- `__init__` 在创建 `Planner(llm)` 时自动执行一次。
- 参数 `llm_client` 是临时变量，只在 `__init__` 执行期间有效；`self.llm_client` 是存在对象身上的**属性**，之后 `plan` 等方法都能用。
- 如果不存，`plan` 里的 `self.llm_client` 会报 `AttributeError`。
- 模型从外面传进来（依赖注入），好处是灵活、可共用、方便测试。

**`prompt = PLANNER_PROMPT_TEMPLATE.format(question=question)`**

- `.format` 把值填进模板里的 `{question}`，`名字=值`：等号**左边**是模板里的占位符名，**右边**是方法参数里的实际内容（只是恰好重名）。
- 它返回一个**新字符串**，模板本身不变，下次还能继续用。
- 如果模板里有别的占位符而你没传，会报 `KeyError`。

**`messages = [{"role": "user", "content": prompt}]`**

把整个提示词包成一条 `user` 消息。

**`response_text = self.llm_client.think(messages=messages) or ""`**

- 调用前面封装好的 `think`，得到模型返回的文字。
- `think` 出错时返回 `None`，`or ""` 把它变成空字符串，下面的 `.split(...)` 就不会对 `None` 调用方法而崩溃。
- 因为 `think` 用的是流式输出，**循环在 `think` 内部**（每收到一小块就 `print` 一次），所以计划会边生成边打印一遍；而下一行 `print(f"✅ 计划已生成:...")` 又完整打印一遍，所以你会看到计划**出现两次**。这是两行不同的 `print` 造成的，不是流式输出生成了两遍。不想重复的话，删掉那行 `print` 即可。

**解析：把文字变成列表（最关键的三行）**

```python
plan_str = response_text.split("```python")[1].split("```")[0].strip()
plan = ast.literal_eval(plan_str)
return plan if isinstance(plan, list) else []
```

假设模型输出是：

````
好的，计划如下：
```python
["确定 iPhone 最新的型号", "找出主要卖点", "综合回答"]
```
````

| 代码 | 做了什么 |
|---|---|
| `.split("```python")` | 以 ```` ```python ```` 为分隔符切成两段，`[1]` 取代码块开头之后的内容 |
| `.split("```")[0]` | 再以结尾的 ```` ``` ```` 切开，取第一段，也就是列表那一段文字 |
| `.strip()` | 去掉首尾的换行和空格 |
| `ast.literal_eval(plan_str)` | 把"长得像列表的字符串"变成真正的 Python 列表 |
| `isinstance(plan, list)` | 确认解析出来的真的是列表 |
| `plan if ... else []` | 条件表达式：是列表就返回，否则返回空列表 |

此时 `plan_str` 还只是**字符串**，只有经过 `literal_eval` 才变成真正的列表。

**为什么用 `ast.literal_eval`，而不是 `eval`**

`eval(字符串)` 会把字符串**当作 Python 代码执行**。如果模型的输出里混进了别的东西（比如受搜索结果里恶意文字影响），`eval` 就会把它执行掉，非常危险。`ast.literal_eval` 只允许解析**字面量**（字符串、数字、列表、字典、元组、布尔值、`None`），碰到函数调用之类的会直接报错，不会执行。

> 凡是"把不可信的文字当代码执行"的地方，都是风险点，这在安全方向会经常遇到。

**异常处理**

| 异常 | 什么时候出现 |
|---|---|
| `IndexError` | 模型没有输出 ```` ```python ````，`split(...)` 之后取 `[1]` 越界 |
| `SyntaxError` | 列表文字不合法，比如引号没配对、漏了逗号 |
| `ValueError` | 内容不是合法的字面量 |
| `Exception` | 兜底，处理没预料到的错误 |

出错时会打印出错原因和模型的**原始响应**（调试时最需要看的就是它），然后返回空列表 `[]`，表示"没能生成计划"。

### 2.4 规划器的几个隐患

1. **格式很脆弱。** 模型写成 ```` ```Python ````（大写 P）、没写 `python`、或没用代码块，`split("```python")` 就找不到分隔符，触发 `IndexError`，返回空列表。
2. **列表可能不合法。** 比如步骤里含有没转义的引号 `["搜索"最新"手机"]`，会触发 `SyntaxError`。
3. **返回空列表时，调用方必须处理。** 不能拿着空计划继续往下执行。

---

## 三、执行阶段：`Executor`

### 3.1 提示词模板 `EXECUTOR_PROMPT_TEMPLATE`

```python
EXECUTOR_PROMPT_TEMPLATE = """
你是一位顶级的AI执行专家。你的任务是严格按照给定的计划，一步步地解决问题。
你将收到原始问题、完整的计划、以及到目前为止已经完成的步骤和结果。
请你专注于解决"当前步骤"，并仅输出该步骤的最终答案，不要输出任何额外的解释或对话。

# 原始问题:
{question}

# 完整计划:
{plan}

# 历史步骤与结果:
{history}

# 当前步骤:
{current_step}

请仅输出针对"当前步骤"的回答:
"""
```

**四个占位符：**

| 占位符 | 填入的内容 | 为什么需要 |
|---|---|---|
| `{question}` | 原始问题 | 让模型始终知道最终目标 |
| `{plan}` | 完整计划 | 让模型知道全局，每一步在整体中处于什么位置 |
| `{history}` | 已完成的步骤和结果 | 让模型能用到前面步骤的成果 |
| `{current_step}` | 当前要做的这一步 | 让模型专注于眼前这一步 |

**提示词里几个要求的用意：**

- "严格按照给定的计划"：防止模型自作主张，另起炉灶。
- "专注于当前步骤"：一次只做一件事。
- "仅输出该步骤的最终答案，不要输出任何额外的解释或对话"：这一步的输出会被原样存进 `history`，成为下一步的输入。如果夹带"好的，我来回答……"这类闲话，这些废话会一路堆进后面的提示词，污染后面的步骤。

### 3.2 `Executor` 类

```python
class Executor:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def execute(self, question: str, plan: list[str]) -> str:
        """根据计划，逐步执行并解决问题。"""
        history = ""  # 用于存储历史步骤和结果的字符串

        print("\n--- 正在执行计划 ---")

        for i, step in enumerate(plan):
            print(f"\n-> 正在执行步骤 {i + 1}/{len(plan)}: {step}")

            prompt = EXECUTOR_PROMPT_TEMPLATE.format(
                question=question,
                plan=plan,
                history=history if history else "无",
                current_step=step
            )

            messages = [{"role": "user", "content": prompt}]

            response_text = self.llm_client.think(messages=messages) or ""

            history += f"步骤 {i + 1}: {step}\n结果: {response_text}\n\n"

            print(f"✅ 步骤 {i + 1} 已完成，结果: {response_text}")

        final_answer = response_text
        return final_answer
```

### 3.3 逐行讲解

| 代码 | 讲解 |
|---|---|
| `def execute(self, question: str, plan: list[str]) -> str:` | 接收原始问题和 `Planner` 生成的计划（字符串列表）；`-> str` 表示返回字符串（最终答案），只是类型提示 |
| `history = ""` | 创建一个空字符串，用来**累积**每一步的步骤和结果。这就是这个类里的"状态"，是函数内的局部变量，每次调用都从空开始 |
| `print("\n--- 正在执行计划 ---")` | 开头的 `\n` 是换行，让输出和前面的内容隔开一行 |
| `for i, step in enumerate(plan):` | `enumerate` 同时给出序号和内容；注意 `i` 从 **0** 开始，所以显示给人看时要写 `i + 1` |
| `len(plan)` | 计划一共有几步 |
| `print(f"... {i + 1}/{len(plan)}: {step}")` | f-string 把变量填进文字，显示成"步骤 2/3: 找出主要卖点" |
| `.format(question=..., plan=plan, ...)` | 用法同前；`plan` 传的是**列表**，`.format` 会自动转成带方括号和引号的文字，模型能读懂 |
| `history if history else "无"` | 条件表达式：`history` 有内容就用它，否则用 `"无"`。空字符串在 Python 里算"假"，所以第一步会显示"无"，比一片空白更清楚 |
| `messages = [{"role": "user", "content": prompt}]` | 包成一条 `user` 消息 |
| `self.llm_client.think(messages=messages) or ""` | 调用模型；失败返回 `None` 时用空字符串代替 |
| `history += f"步骤 {i + 1}: {step}\n结果: {response_text}\n\n"` | **状态管理的核心。** `+=` 是在末尾追加；每做完一步，就把"步骤 + 结果"记进 `history`；`\n\n` 空一行让步骤之间分段清楚 |
| `print(f"✅ 步骤 ... 已完成，结果: ...")` | 再完整打印一遍这一步的结果（原因同规划器：流式输出已经打印过一遍） |
| `final_answer = response_text` | 循环结束后，`response_text` 里存的是**最后一步**的结果，设计上就把它当作最终答案 |
| `return final_answer` | 返回给调用者 |

追加几步后，`history` 大致是这样：

```
步骤 1: 确定 iPhone 最新的型号
结果: 最新型号是……

步骤 2: 找出该型号的主要卖点
结果: 主要卖点是……

```

### 3.4 一个例子走一遍

问题："iPhone 最新的型号是哪一款？它的主要卖点是什么？"
计划：`["确定 iPhone 最新的型号", "找出该型号的主要卖点", "综合以上信息，回答原问题"]`

| 步骤 | 填进提示词的 `history` | 当前步骤 | 模型输出（示意） |
|---|---|---|---|
| 1 | `无` | 确定 iPhone 最新的型号 | 某某型号 |
| 2 | 步骤 1 + 结果 | 找出该型号的主要卖点 | 卖点 A、B、C |
| 3 | 步骤 1、2 + 结果 | 综合以上信息，回答原问题 | 完整的最终答案 |

第 3 步的输出，就是返回的最终答案。**这个设计有个前提：计划的最后一步，应该是"综合前面结果，回答原问题"这类步骤**，否则最终答案只是某个中间结果。

---

## 四、组装：`PlanAndSolveAgent`

### 4.1 思路

`Planner`（规划）和 `Executor`（执行）是两个零件，`PlanAndSolveAgent` 是把它们装在一起的**外壳**：它自己几乎不做具体的事，只负责**安排先后顺序**，对外提供一个简单的 `run` 方法。使用的人只需要知道 `run(问题)`，不用关心里面分了规划和执行两步。

```
调用 agent.run(问题)
        │
        ▼
 1. 让 Planner 生成计划   plan = self.planner.plan(question)
        │
        ▼
 2. 计划为空？是 → 打印"任务终止"，直接结束
        │否
        ▼
 3. 让 Executor 按计划执行   final_answer = self.executor.execute(question, plan)
        │
        ▼
 4. 打印最终答案
```

### 4.2 代码

```python
class PlanAndSolveAgent:
    def __init__(self, llm_client):
        """初始化智能体，同时创建规划器和执行器实例。"""
        self.llm_client = llm_client
        self.planner = Planner(self.llm_client)
        self.executor = Executor(self.llm_client)

    def run(self, question: str):
        """运行智能体的完整流程：先规划，后执行。"""
        print(f"\n--- 开始处理问题 ---\n问题: {question}")

        # 1. 调用规划器生成计划
        plan = self.planner.plan(question)

        # 检查计划是否成功生成
        if not plan:
            print("\n--- 任务终止 --- \n无法生成有效的行动计划。")
            return

        # 2. 调用执行器执行计划
        final_answer = self.executor.execute(question, plan)

        print(f"\n--- 任务完成 ---\n最终答案: {final_answer}")
```

### 4.3 `__init__`：创建并组装两个零件

| 代码 | 讲解 |
|---|---|
| `class PlanAndSolveAgent:` | 定义一个类，名字就是"Plan-and-Solve 智能体" |
| `def __init__(self, llm_client):` | 创建对象时自动执行；只需要传入一个模型对象，别的都在里面自己造 |
| `self.llm_client = llm_client` | 把模型存进对象里，作为 Agent 自己的属性 |
| `self.planner = Planner(self.llm_client)` | **创建一个 `Planner` 对象**，把模型交给它，存成 `self.planner` |
| `self.executor = Executor(self.llm_client)` | **创建一个 `Executor` 对象**，同样交给它同一个模型，存成 `self.executor` |

要点：

1. **`Planner(...)` 和 `Executor(...)` 在这里被"造出来"。** `Planner.__init__(self, llm_client)` 需要一个模型，所以这里把模型传进去。这三行执行完，这个 Agent 里就有了三样东西：模型、规划器、执行器。
2. **两个零件用的是同一个模型对象。** 传给它们的是**同一个对象的引用**，不是复制了两份。好处是配置一次、共用同一套密钥和地址。
3. **为什么存成 `self.planner`、`self.executor`？** 因为 `run` 是另一个方法，要用到它们。只有存在对象身上（`self.xxx`），别的方法才拿得到，和 `self.llm_client` 是同一个道理。

### 4.4 `run`：运行完整流程

| 代码 | 讲解 |
|---|---|
| `def run(self, question: str):` | 公开的入口方法，参数是用户的问题 |
| `print(f"\n--- 开始处理问题 ---\n问题: {question}")` | f-string 把问题填进去；`\n` 是换行 |
| `plan = self.planner.plan(question)` | **第一阶段：规划。** 得到一个字符串列表，比如 `["确定型号", "找卖点", "综合回答"]` |
| `if not plan:` | 判断计划是否为空。空列表 `[]` 在 Python 里算"假"，`not` 把它变成"真"，所以只有**计划生成失败**时才进入分支 |
| `print("\n--- 任务终止 --- ...")` | 提示用户无法生成计划 |
| `return` | **直接结束 `run`**，后面的代码不再执行；后面没有跟值，所以返回 `None` |
| `final_answer = self.executor.execute(question, plan)` | **第二阶段：执行。** 把原问题和计划交给执行器，拿回最后一步的结果，也就是最终答案 |
| `print(f"\n--- 任务完成 ---\n最终答案: {final_answer}")` | 打印最终答案 |

**一处重要的设计：`if not plan:` 正好堵住了前面提到的那个 bug。** `Executor.execute` 如果收到空计划，会因为 `response_text` 没有被创建而报 `UnboundLocalError`。现在 `run` 在调用执行器**之前**就把空计划拦住了，所以通过 `PlanAndSolveAgent.run` 使用时，执行器不会收到空计划。

通用的好习惯：**上一个阶段的输出可能不合格，在交给下一个阶段之前，要先检查。** 以后写 Agent，遇到"阶段衔接"的地方，都应该想一想"上一步失败了怎么办"。

### 4.5 "组合优于继承"和"协调者"

教程里有一句话：这个类的设计体现了"组合优于继承"的原则，它本身不包含复杂的逻辑，而是作为一个协调者（Orchestrator），清晰地调用其内部组件来完成任务。

**组合（Composition）：** 让一个类**拥有**别的类的对象，关系是"有一个"（has-a）。`PlanAndSolveAgent` 里面"装着"一个 `Planner` 和一个 `Executor`。

**继承（Inheritance）：** 让一个类**变成**另一个类的子类，关系是"是一个"（is-a）。如果写成 `class PlanAndSolveAgent(Planner)`，意思是"Agent 是一个规划器"，显然不对：它既要规划又要执行，同时继承两者会很别扭，结构会越来越乱。

| 方面 | 组合 | 继承 |
|---|---|---|
| 关系 | 有一个（has-a） | 是一个（is-a） |
| 灵活性 | 想换哪个零件，传另一个对象就行 | 父类一改，所有子类都受影响 |
| 耦合 | 松，各零件独立 | 紧，子类被绑死在父类上 |
| 测试 | 容易，零件可以单独测试、单独替换 | 较难 |

**协调者（Orchestrator）：** `PlanAndSolveAgent` 自己不会规划，也不会执行，它只是安排顺序：先叫规划器，再叫执行器，中间检查一下。就像项目经理：自己不写代码，只分配任务、把控流程。好处是：规划和执行的细节各自改动，都不用动这个外壳；想以后加"反思"环节，只需在 `run` 里再多叫一个组件。

### 4.6 完整的使用方式

三个类都定义好之后，使用只要三行：

```python
if __name__ == '__main__':
    llm = HelloAgentsLLM()                    # 1. 造模型
    agent = PlanAndSolveAgent(llm)            # 2. 造 Agent（里面自动造规划器、执行器）
    agent.run("iPhone 最新的型号是哪一款？它的主要卖点是什么？")   # 3. 运行
```

注意：这几行要放在**所有类定义之后**（Python 从上往下执行）。如果文件里还有前面的测试块（也是 `if __name__ == '__main__':`），运行时它们会**全部执行**，想只跑 Agent，把旧的测试块暂时注释掉。

### 4.7 `PlanAndSolveAgent` 的两个小问题

1. **`run` 只打印了答案，没有 `return` 它。** 所以 `answer = agent.run("...")` 拿到的是 `None`。如果想在别的地方使用答案（存起来、展示在网页上），在最后加一行 `return final_answer`。对比 `ReActAgent.run`，它在拿到 `Finish` 时用了 `return final_answer`，行为更完整。
2. **执行阶段没有接入工具。** 参见 5.2：每一步只是问模型，不能联网，问"最新的 iPhone"这类时效性问题，答案可能过时。

### 4.8 整个 Plan-and-Solve 的调用关系

```
PlanAndSolveAgent（协调者）
    │
    ├── self.llm_client ─── HelloAgentsLLM（共用的模型对象）
    │
    ├── self.planner ─── Planner
    │        └─ plan(question) → ["步骤1", "步骤2", ...]
    │
    └── self.executor ─── Executor
             └─ execute(question, plan) → 最终答案
```

与 ReAct 对比：

| | `ReActAgent` | `PlanAndSolveAgent` |
|---|---|---|
| 自己做什么 | 自己包含循环、解析、调用工具的全部逻辑 | 只负责协调，细节交给 `Planner` 和 `Executor` |
| 代码结构 | 一个大类 | 一个外壳 + 两个小零件 |
| 好处 | 逻辑集中，一处看全 | 分工清楚，容易单独修改、替换 |

---

## 五、这两段代码需要留意的问题

### 5.1 计划为空时会报错（真实的 bug）

如果 `Planner` 解析失败，返回了 `[]`，`for` 循环一次都不会执行，`response_text` 这个变量**根本没有被创建**。执行到 `final_answer = response_text` 时会报：

```
UnboundLocalError: cannot access local variable 'response_text' where it is not associated with a value
```

**修改：** 开头拦住空计划，并给 `response_text` 一个初始值：

```python
def execute(self, question: str, plan: list[str]) -> str:
    if not plan:
        print("计划为空，无法执行。")
        return ""

    history = ""
    response_text = ""   # 先给初始值，保证循环外也能使用
    ...
```

### 5.2 这个执行器没有用到工具

每一步都只是"问模型"，模型只能靠自己训练时学到的知识回答，**不能联网搜索**。所以问"最新的 iPhone"这类时效性问题时，答案可能过时。想让 Plan-and-Solve 也能联网，需要在每一步执行时把工具接进来，这是后面可以扩展的方向。

### 5.3 出错时会静默继续

`think` 失败返回 `""`，`history` 里就会记录一个空结果，后面的步骤会在错误的基础上继续。可以加判断：

```python
if not response_text:
    print(f"⚠️ 步骤 {i + 1} 没有得到结果")
```

### 5.4 `history` 越来越长

每步都把全部历史再发一遍，步骤很多时提示词会变得很长，耗时和费用都会上升。

### 5.5 完整计划每步都重复发送

这是有意为之（让模型始终看到全局），代价同样是提示词变长。

### 5.6 加固后的 `execute`（已用模拟模型测试）

```python
def execute(self, question: str, plan: list[str]) -> str:
    if not plan:
        print("计划为空，无法执行。")
        return ""

    history = ""
    response_text = ""

    print("\n--- 正在执行计划 ---")

    for i, step in enumerate(plan):
        print(f"\n-> 正在执行步骤 {i + 1}/{len(plan)}: {step}")

        prompt = EXECUTOR_PROMPT_TEMPLATE.format(
            question=question,
            plan=plan,
            history=history if history else "无",
            current_step=step,
        )

        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages) or ""

        if not response_text:
            print(f"⚠️ 步骤 {i + 1} 没有得到结果")

        history += f"步骤 {i + 1}: {step}\n结果: {response_text}\n\n"
        print(f"✅ 步骤 {i + 1} 已完成，结果: {response_text}")

    return response_text
```

---

## 六、方法名和语法速查

| 写法 | 含义 |
|---|---|
| `enumerate(列表)` | 同时给出序号（从 0 开始）和内容 |
| `len(列表)` | 列表的长度 |
| `x if 条件 else y` | 条件表达式，条件为真取 x，否则取 y |
| `字符串 += 内容` | 在字符串末尾追加内容 |
| `f"...{变量}..."` | f-string，把变量的值填进文字 |
| `\n` / `\n\n` | 换行 / 空一行 |
| `模板.format(a=..., b=...)` | 把值填进模板的 `{a}`、`{b}` 占位符 |
| `... or ""` | 前面是 `None` 或空，就用空字符串代替 |
| `字符串.split(分隔符)` | 按分隔符切成列表 |
| `字符串.strip()` | 去掉首尾空白 |
| `ast.literal_eval(字符串)` | 安全地把字面量字符串转成 Python 对象 |
| `isinstance(x, list)` | 判断 x 是不是列表 |
| `list[str]` | 类型提示：由字符串组成的列表 |
| `-> str` | 类型提示：返回值是字符串 |
| `__init__` / `self` | 初始化方法 / 对象自己 |
| `self.planner = Planner(self.llm_client)` | 创建一个 `Planner` 对象并存进自己的属性 |
| `self.planner.plan(question)` | 通过自己拥有的规划器，调用它的 `plan` 方法 |
| `if not plan:` | 计划为空（空列表算"假"）时成立 |
| `return`（不带值） | 结束方法，返回 `None` |
| 组合 / 继承 | "有一个" / "是一个"的关系 |
| Orchestrator | 协调者：自己不做细节，只安排各组件的先后顺序 |

---

## 七、常见报错对照

| 现象 | 可能的原因 |
|---|---|
| `NameError: name 'ast' is not defined` | 文件里没有 `import ast` |
| 反复出现"解析计划时出错: list index out of range" | 模型没有按 ```` ```python ```` 格式输出，先看"原始响应" |
| `SyntaxError` | 模型输出的列表不合法（引号不配对、漏逗号） |
| `UnboundLocalError` | 计划为空，循环没执行，`response_text` 未创建 |
| `KeyError: 'xxx'` | 模板里的占位符和 `.format` 里传的名字对不上 |
| 计划和每步结果都出现两遍 | `think` 流式输出打印了一遍，外层又 `print` 了一遍，属于正常 |
| `agent.run(...)` 的返回值是 `None` | `run` 里只 `print` 了答案，没有 `return final_answer` |
| 提示"无法生成有效的行动计划" | `Planner` 返回了空列表，先看规划阶段打印的"原始响应" |
| 最终答案只是中间结果 | 计划最后一步不是"综合回答"，可以在规划提示词里要求最后一步总结 |
| 答案里的信息过时 | 执行器没有接入搜索工具，模型只能靠自己的知识 |

---

## 八、练习

1. 在 `execute` 里调用 `think` 之前加 `print(prompt)`，亲眼看每一步的提示词，`history` 是怎么一步步变长的。
2. 用一个"假模型"（一个有 `think` 方法、返回固定文字的类）传给 `Planner`，让它返回没有代码块的文字，观察会走哪个 `except`，返回什么。
3. 让 `Planner` 返回 `[]`，再调用旧版 `Executor.execute`，观察 `UnboundLocalError`，然后按 5.1 修改，确认不再报错。
4. 把执行器提示词里"仅输出该步骤的最终答案，不要输出任何额外的解释"删掉，观察模型输出变长，`history` 被污染。
5. 先问一个不需要联网的问题（例如"计算 12 乘以 15，再加上 8，最后告诉我结果是不是偶数"），再问一个需要最新信息的问题，对比结果，体会这个执行器没有工具的局限。
6. 对同一个问题，分别用 ReAct 和 Plan-and-Solve 做一遍，比较模型调用次数和耗时。
7. 在 `PlanAndSolveAgent.run` 的 `plan = ...` 后面加一行 `print(plan)`，确认规划器返回的是真正的列表。
8. 给 `run` 的最后加上 `return final_answer`，再写 `answer = agent.run("...")` 并打印，对比加之前的结果。
9. 试着在 `__init__` 里不把 `self.llm_client` 传给 `Executor`，观察报什么错，体会"零件需要的东西要从外面传进来"。

---

## 九、总结

- **`Planner`** 把问题交给模型，要求它用固定格式输出一份步骤列表；代码用"按标记切字符串"裁出列表文字，再用 `ast.literal_eval` 安全地转成真正的列表；格式不对时返回空列表。
- **`Executor`** 用一个不断变长的字符串 `history` 记录已完成的步骤和结果，每一步都把"问题 + 完整计划 + 历史 + 当前步骤"一起交给模型，让模型只回答当前这一步；最后一步的结果就是最终答案。
- **`__init__`** 的作用是把传进来的模型存成 `self.llm_client`，让对象的其他方法能用。
- **状态管理**：模型自己没有记忆，靠每一步都把累积的 `history` 重新发给它，前后步骤才能衔接。
- **`PlanAndSolveAgent`** 在 `__init__` 里造好规划器和执行器，在 `run` 里按"先规划、检查计划、再执行、输出答案"的顺序调用它们；它自己不含复杂逻辑，只做协调，这就是"组合优于继承"。
- **衔接处要检查**：`run` 里的 `if not plan:` 拦住了空计划，避免执行器出错。
- **隐患**：格式脆弱、空计划会报错（单独用 `Executor` 时）、没有工具、历史越来越长、`run` 没有返回答案；可以扩展的方向是给执行阶段接入工具，或再加上"反思"环节。