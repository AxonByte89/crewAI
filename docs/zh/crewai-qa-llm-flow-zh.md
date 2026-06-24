# CrewAI 进阶问答：Agent LLM 配置与 Crew/Flow 选型

> 基于 CrewAI 入门指南的延伸问答，涵盖多模型配置、架构选型、Agent 执行机制与能力决定因素。

---

## Q1：可以为每个 Agent 定义不同的 LLM 吗？

**完全可以！** CrewAI 原生支持为每个 Agent 单独配置不同的 LLM。

Agent 的 `llm` 字段接受字符串或 `BaseLLM` 实例，这意味着你可以为每个 Agent 指定不同的模型。

### 方式一：直接用字符串指定模型名

```python
from crewai import Agent, LLM

# Agent 1 使用 GPT-4o
researcher = Agent(
    role="高级研究员",
    goal="发现最新技术趋势",
    backstory="你是一位经验丰富的研究员。",
    llm="openai/gpt-4o"  # 直接用字符串
)

# Agent 2 使用 Claude
writer = Agent(
    role="技术作者",
    goal="撰写清晰的文章",
    backstory="你是一位优秀的技术作者。",
    llm="anthropic/claude-sonnet-4-20250514"
)

# Agent 3 使用本地 Ollama 模型
analyzer = Agent(
    role="数据分析师",
    goal="分析数据",
    backstory="你是一位数据分析师。",
    llm="ollama/llama3"
)
```

### 方式二：使用 `LLM` 对象精细控制参数

```python
from crewai import Agent, LLM

# 为研究员配置 GPT-4o，高温参数增加创造性
researcher = Agent(
    role="高级研究员",
    goal="发现最新技术趋势",
    backstory="...",
    llm=LLM(
        model="openai/gpt-4o",
        temperature=0.8,
        max_tokens=4096,
        api_key="sk-xxx"
    )
)

# 为写手配置 Claude，低温参数增加准确性
writer = Agent(
    role="技术作者",
    goal="撰写清晰的文章",
    backstory="...",
    llm=LLM(
        model="anthropic/claude-sonnet-4-20250514",
        temperature=0.3,
    )
)
```

### 方式三：在 YAML 配置中指定（项目模式推荐）

在 `agents.yaml` 中为每个 Agent 配置 `llm` 字段：

```yaml
researcher:
  role: "高级研究员"
  goal: "发现最新趋势"
  backstory: "..."
  llm: openai/gpt-4o

writer:
  role: "技术作者"
  goal: "撰写文章"
  backstory: "..."
  llm: anthropic/claude-sonnet-4-20250514
```

### 支持的模型提供商

| 前缀格式               | 提供商             |
| ---------------------- | ------------------ |
| `openai/`              | OpenAI             |
| `anthropic/`           | Anthropic (Claude) |
| `gemini/` 或 `google/` | Google Gemini      |
| `azure/`               | Azure OpenAI       |
| `bedrock/` 或 `aws/`   | AWS Bedrock        |
| `ollama/`              | Ollama（本地模型） |
| `deepseek/`            | DeepSeek           |
| `openrouter/`          | OpenRouter         |
| `cerebras/`            | Cerebras           |

**核心原则**：如果你不指定 `llm`，Agent 会使用环境变量中的默认模型（如 `OPENAI_MODEL_NAME`）。一旦你为某个 Agent 显式指定了 `llm`，它只影响该 Agent，不影响其他 Agent。

---

## Q2：Crew 和 Flow 应该如何选择？

这是一个架构决策问题，两者不是互斥关系，而是**包含关系**。

### 简单决策指南

| 场景                                               | 推荐方案              |
| -------------------------------------------------- | --------------------- |
| 只有一组 Agent 协作完成一组任务                    | **直接用 Crew**       |
| 需要在多个 Crew 之间编排、有条件分支、需要状态管理 | **用 Flow 包裹 Crew** |
| 快速原型验证、脚本式任务                           | **直接用 Crew**       |
| 生产级应用、复杂业务流程                           | **用 Flow**           |

### 什么时候只用 Crew？

当你的需求是"给几个 Agent 分配任务，让他们按顺序或层级完成"时，直接用 Crew 就够了：

```python
# 简单场景：研究 + 写作，一个 Crew 搞定
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
)
result = crew.kickoff(inputs={"topic": "AI"})
```

### 什么时候用 Flow？

当你需要以下能力时，就需要 Flow：

1. **多步骤编排**：先做 A，再做 B，最后做 C，每步可能是不同的 Crew
2. **条件分支**：根据某步的结果决定走哪条路径（用 `@router`）
3. **状态管理**：在多步之间共享和传递数据（Flow 有内置的 `self.state`）
4. **混合逻辑**：有些步骤调用 Crew，有些步骤执行普通 Python 代码

```python
from crewai.flow.flow import Flow, listen, start, router
from pydantic import BaseModel

class PipelineState(BaseModel):
    topic: str = ""
    research_result: str = ""
    quality_score: float = 0.0
    final_article: str = ""

class ContentPipeline(Flow[PipelineState]):
    @start()
    def prepare(self):
        self.state.topic = "AI Agents"

    @listen(prepare)
    def research(self):
        # 调用第一个 Crew 做研究
        result = ResearchCrew().crew().kickoff(inputs={"topic": self.state.topic})
        self.state.research_result = result.raw

    @listen(research)
    def evaluate(self):
        # 普通 Python 逻辑评估质量
        self.state.quality_score = evaluate_quality(self.state.research_result)
        return self.state.quality_score

    @router(evaluate)
    def decide_next(self):
        # 条件分支：质量好就发布，不好就重写
        if self.state.quality_score >= 0.8:
            return "publish"
        return "rewrite"

    @listen("publish")
    def publish_article(self):
        # 调用第二个 Crew 做排版发布
        result = PublishCrew().crew().kickoff(...)
        self.state.final_article = result.raw

    @listen("rewrite")
    def rewrite_article(self):
        # 调用第三个 Crew 重写
        result = RewriteCrew().crew().kickoff(...)
        self.state.research_result = result.raw
```

### 一句话总结

> **Crew 是"团队"**——解决"谁来做、做什么"的问题。
> **Flow 是"流水线"**——解决"什么顺序做、出问题了怎么办"的问题。
>
> 简单任务 → 一个 Crew 就够。
> 复杂业务 → 用 Flow 编排多个 Crew + 自定义逻辑。

---

## Q3：Agent 执行任务时是多轮循环还是只调用一次大模型？

**Agent 是多轮循环执行的，不是一次调用就完成。**

从源码 `CrewAgentExecutor._invoke_loop()` 可以看到，Agent 内部有一个 `while` 循环，每一轮执行以下流程：

```
┌─────────────────────────────────────────────────┐
│              Agent 执行循环                       │
│                                                   │
│  1. 调用 LLM → 模型返回"思考"和"行动"            │
│         │                                         │
│         ▼                                         │
│  2. 判断：是要执行工具，还是给出最终答案？         │
│         │                                         │
│    ┌────┴────┐                                    │
│    ▼         ▼                                    │
│  AgentAction  AgentFinish                         │
│  (要用工具)   (任务完成)                           │
│    │              │                               │
│    ▼              ▼                               │
│  3. 执行工具    返回最终结果                       │
│    │                                              │
│    ▼                                              │
│  4. 把工具结果追加到对话历史                       │
│    │                                              │
│    └──── 回到第1步，继续循环 ────┘                │
│                                                   │
└─────────────────────────────────────────────────┘
```

### 具体执行过程

1. **自动分析问题**：Agent 接收到 Task 后，会自动拆解问题，决定下一步做什么
2. **自主决定调用工具**：如果配置了搜索、文件读取等工具，Agent 自己判断何时该用哪个工具
3. **观察结果再思考**：工具返回的结果追加到对话上下文，LLM 基于结果继续推理
4. **循环直到完成**：只有当 LLM 返回 `AgentFinish` 时，循环才结束

### 最大迭代次数保护

Agent 有 `max_iter` 参数（默认 25 次），防止陷入无限循环：

```python
agent = Agent(
    role="研究员",
    goal="深度研究AI",
    backstory="...",
    max_iter=50  # 允许更多轮思考
)
```

### 两种执行策略

框架根据 LLM 能力自动选择执行策略：

| 模式                  | 适用条件                                  | 工作方式                                                 |
| --------------------- | ----------------------------------------- | -------------------------------------------------------- |
| **Native Tools 模式** | LLM 支持原生函数调用（如 GPT-4o、Claude） | LLM 直接返回结构化 tool_call，框架执行后把结果喂回       |
| **ReAct 文本模式**    | LLM 不支持原生函数调用                    | LLM 在文本中输出 `Action / Action Input`，框架解析并执行 |

### 开启规划能力（Planning）

框架支持让 Agent 在执行前先制定计划，按计划逐步执行：

```python
from crewai import Agent, PlanningConfig

agent = Agent(
    role="研究员",
    goal="深度研究AI",
    backstory="...",
    planning=True,
    planning_config=PlanningConfig(
        reasoning_effort="high",  # 规划力度
        max_attempts=3            # 允许重新规划次数
    )
)
```

### 一句话总结

> Agent 内置了"思考→行动→观察"的多轮循环，会自主分析问题、调用工具、观察结果、再思考下一步，直到任务完成。这正是 Agent 与普通"一问一答"ChatGPT 的核心区别。

---

## Q4：决定一个 Agent 是否厉害的关键因素是什么？是提示词吗？

**提示词只是冰山一角。** 一个 Agent 的真正能力由以下几个层次决定：

### 第一层：基座模型能力（最关键，占 60%+）

这是最根本的差异。同一个框架、同一个提示词，换一个模型效果天差地别。

| 能力维度           | 为什么重要                                       |
| ------------------ | ------------------------------------------------ |
| **推理能力**       | 能否正确拆解复杂问题，而不是在错误方向上反复循环 |
| **指令遵循度**     | 是否严格遵守框架的输出格式（如 JSON、tool_call） |
| **工具调用准确性** | 能否在正确时机、用正确参数调用正确工具           |
| **长上下文理解**   | 多轮循环后对话历史变长，能否保持一致性不迷失     |
| **自我纠错能力**   | 工具返回错误时，能否调整策略而不是死循环         |

### 第二层：工具生态（占 20%）

工具的质量和丰富度直接决定 Agent 能做什么：

```
只有搜索工具的 Agent               → 只能搜索和写文章
搜索+代码执行+文件操作+数据库的 Agent → 能做真正的工程任务
```

Claude Code 之所以强大，核心之一就是它有一套丰富的工具链（文件读写、代码搜索、终端执行、Git 操作、LSP 导航等）。

### 第三层：提示词与框架设计（占 15%）

提示词的本质是**释放模型已有的能力**，而不是凭空创造能力。

**有效的提示词策略：**

```
普通：  "你是一个研究员，帮我搜索信息"
优秀：  "你是一个研究员。执行步骤：
        1. 先用搜索工具获取最新信息
        2. 验证信息来源的可靠性
        3. 交叉对比多个来源
        4. 如果信息矛盾，明确指出
        5. 输出格式要求：..."
```

但提示词有**边际递减效应**——超过一定复杂度后，再加更多指令反而会降低模型表现。

框架层面的价值体现在循环控制策略：

| 策略              | 说明                                          |
| ----------------- | --------------------------------------------- |
| **迭代控制**      | 什么时候该停？死循环了怎么办？                |
| **错误恢复**      | 工具调用失败后，是重试、换工具、还是放弃？    |
| **上下文管理**    | 对话太长时，如何压缩历史而不丢失关键信息？    |
| **多 Agent 协作** | 一个 Agent 做不好时，能否交给更专业的 Agent？ |
| **规划与重规划**  | 计划执行到一半发现不对，能否自动调整？        |

### 第四层：产品化能力（占 5%）

- 流式输出：实时看到 Agent 的思考过程
- 中断与恢复：暂停、修改方向、继续
- 人类反馈：关键节点让人审核（CrewAI 的 `@human_feedback`）
- 可观测性：出了问题能快速定位原因

### 用一个类比总结

把 Agent 想象成一个**员工**：

| 因素            | 类比                         | 权重    |
| --------------- | ---------------------------- | ------- |
| **基座模型**    | 员工的智商和专业能力         | **60%** |
| **工具生态**    | 员工能用的工具和资源         | **20%** |
| **提示词/框架** | 给员工的任务说明书和工作流程 | **15%** |
| **产品化**      | 办公环境和沟通机制           | **5%**  |

### 写提示词时应该关注什么？

**不是写得更长更复杂，而是：**

1. **把角色和目标定义清楚** — `role`、`goal`、`backstory` 写准确
2. **给合适的工具** — 不多不少，工具太多模型反而会选错
3. **选对模型** — 简单任务用便宜模型，复杂推理用强模型
4. **设计好流程** — 用 Flow 把复杂任务拆成多个 Crew，比让一个 Agent 做所有事更可靠

> **一句话：基座模型决定了 Agent 的能力上限，工具和流程设计决定了你能用到多少上限，提示词只是确保模型按照你期望的方式发挥能力。**

---

## Q5：Native Tools 模式 与 ReAct 文本模式 有什么区别？

框架会根据 LLM 的能力自动选择执行策略，判断逻辑如下：

```python
# 源码 CrewAgentExecutor._invoke_loop()
use_native_tools = (
    llm.supports_function_calling()  # 模型支持原生函数调用
    and agent.original_tools         # Agent 配置了工具
)
if use_native_tools:
    _invoke_loop_native_tools()   # 原生工具模式
else:
    _invoke_loop_react()           # ReAct 文本模式
```

### Native Tools 原生工具模式

**适用模型**：GPT-4o、Claude、Gemini、DeepSeek 等支持 Function Calling 的模型。

**原理**：这些模型在训练阶段就专门学习过“如何使用工具”，框架只需把工具定义以 JSON Schema 的形式作为 API 参数传给模型，模型自己决定何时用哪个工具、传什么参数，并以结构化 JSON 返回调用指令，框架直接执行，无需解析文本。

```
框架传给模型：
{
  "tools": [{
    "function": {
      "name": "search",
      "description": "搜索网页信息",
      "parameters": { "query": {"type": "string"} }
    }
  }]
}

模型返回结构化 JSON：
{ "tool_calls": [{ "function": { "name": "search", "arguments": "{\"query\": \"AI Agent\"}" } }] }
     ↓
框架直接读取 JSON → 执行工具 → 结果喂回模型
```

### ReAct 文本模式

**适用模型**：Ollama/Llama3、较老模型、或不支持 Function Calling API 的模型。

**原理**：模型本身具备推理能力，但没有训练过“工具调用”这个专用能力。框架把工具信息写进 prompt 文本，并通过固定格式指令“教”模型用文字表达工具调用意图，再用正则表达式解析提取。

```
模型输出纯文本：
Thought: 我需要搜索最新信息
Action: search
Action Input: "AI Agent"
     ↓
框架用正则表达式解析文本 → 提取 tool 名 + 参数 → 执行工具
```

### 两者对比

| 对比维度         | Native Tools 模式             | ReAct 文本模式               |
| ---------------- | ----------------------------- | ---------------------------- |
| **触发条件**     | 模型支持函数调用 + 有配置工具 | 模型不支持函数调用，或有工具 |
| **工具定义传递** | 作为 API 参数（JSON Schema）  | 写在 prompt 文本里           |
| **LLM 输出格式** | 结构化 JSON（直接读）         | 文本（需正则解析）           |
| **稳定性**       | 高（模型原生保证）            | 较低（格式可能出错）         |
| **错误处理**     | 几乎无解析错误                | 需处理 OutputParserError     |
| **代表模型**     | GPT-4o、Claude、Gemini        | Ollama/Llama3、小模型        |

### 本质区别（类比理解）

> **Native Tools** 就像一个受过专业训练的厨师——他本来就学过标准点菜格式，你给他菜单，他自然按标准格式下单。
>
> **ReAct** 就像一个聪明但没受过这个训练的人——他很聪明能理解你要什么，但他不知道标准格式，所以你要告诉他：“想点菜就写‘我要点XX菜，配料是XX’”。他大概率能做到，但偶尔可能写错格式。

**一句话总结：本质区别不是“推理能力强弱”，而是模型有没有在训练阶段学过工具调用的专用能力。随着越来越多模型原生支持 Function Calling，ReAct 正逐步成为兼容性兗底方案。**

---

## Q6：如何定义自定义工具让模型使用？

你不需要“教”模型使用工具，只需把工具描述清楚，框架会自动把描述转换成 JSON Schema 传给模型，模型自己判断何时调用。

工具的三个关键字段就是给模型看的“说明书”：

| 字段                     | 作用     | 模型用它来判断什么     |
| ------------------------ | -------- | ---------------------- |
| `name`                   | 工具名称 | “这个工具叫什么”       |
| `description`            | 工具描述 | “什么时候该用这个工具” |
| `args_schema` / 函数签名 | 参数定义 | “需要传什么参数”       |

### 方式一：`@tool` 装饰器（最简单）

```python
from crewai.tools import tool

@tool("查询天气")
def get_weather(city: str) -> str:
    """查询指定城市的实时天气信息，输入城市名，返回天气数据"""
    import requests
    resp = requests.get(f"https://api.weather.com/{city}")
    return resp.json()["weather"]
```

框架自动提取：

- `name` → `"查询天气"`
- `description` → 函数的 docstring
- `参数` → 从函数签名 `city: str` 自动推断

### 方式二：继承 `BaseTool`（适合复杂工具）

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field

# 用 Pydantic 精确定义参数结构（给模型看的）
class WeatherInput(BaseModel):
    city: str = Field(description="城市名称，如：北京、上海")
    days: int = Field(default=1, description="预报天数，1-7天")

class WeatherTool(BaseTool):
    name: str = "查询天气预报"
    description: str = "查询指定城市未来几天的天气预报。当用户询问天气相关问题时使用此工具。"
    args_schema: type[BaseModel] = WeatherInput

    def _run(self, city: str, days: int = 1) -> str:
        import requests
        resp = requests.get(f"https://api.weather.com/{city}?days={days}")
        return resp.text
```

### 方式三：`@tool` + Pydantic Schema（简洁又精确）

```python
from crewai.tools import tool
from pydantic import BaseModel, Field

class QueryInput(BaseModel):
    sql: str = Field(description="要执行的 SQL 查询语句")
    database: str = Field(default="main", description="数据库名称")

@tool("执行SQL查询", args_schema=QueryInput)
def run_sql(sql: str, database: str = "main") -> str:
    """在指定数据库上执行 SQL 查询并返回结果。当需要查询数据时使用。"""
    import sqlite3
    conn = sqlite3.connect(f"{database}.db")
    return str(conn.execute(sql).fetchall())
```

### 框架把工具“翻译”成什么给模型看？

无论哪种方式，框架最终都转换成如下 JSON Schema 传给模型（Native Tools 模式）：

```json
{
  "type": "function",
  "function": {
    "name": "查询天气预报",
    "description": "查询指定城市未来几天的天气预报。当用户询问天气相关问题时使用此工具。",
    "parameters": {
      "type": "object",
      "properties": {
        "city": { "type": "string", "description": "城市名称，如：北京、上海" },
        "days": {
          "type": "integer",
          "description": "预报天数，1-7天",
          "default": 1
        }
      },
      "required": ["city"]
    }
  }
}
```

模型看到这份“说明书”后：判断用户的任务是问天气 → 决定调用此工具 → 填好参数传回 → 框架执行，把结果喂给模型继续推理。

### description 写得好坏直接影响效果

模型决定“用不用这个工具”完全依赖 `description`：

```
差的：  "一个工具"
一般：  "查询天气"
优秀：  "查询指定城市未来1-7天的实时天气预报，包含温度、降水概率、
        空气质量等信息。当用户询问天气、气温、是否需要带伞等问题时使用。"
```

**写好 description 的三个原则：**

| 原则             | 说明                                               |
| ---------------- | -------------------------------------------------- |
| **说清能做什么** | “查询天气预报” 而不是 “一个工具”                   |
| **说清何时该用** | “当用户询问天气、气温、是否需要带伞时”             |
| **说清输入什么** | 参数用 `Field(description="...")` 描述每个字段含义 |

> **一句话：你不需要“教”模型使用工具，只需把工具的描述写清楚——就像写一个好的函数注释，模型自己能看懂并决定何时调用。**
