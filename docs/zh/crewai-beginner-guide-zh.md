# CrewAI 框架入门指南

> 面向后端程序员的 CrewAI 框架全面介绍与使用指南

---

## 一、CrewAI 是什么？

CrewAI 是一个**开源的多智能体（Multi-Agent）编排框架**，用 Python 编写，完全独立于 LangChain 等其他框架。它的核心理念是：让多个 AI 智能体（Agent）像团队协作一样，自主地完成复杂任务。

简单来说，CrewAI 帮你解决的核心问题是：**如何让多个 AI "员工"分工协作，自动完成一项复杂工作。**

### 核心特点

| 特性         | 说明                                         |
| ------------ | -------------------------------------------- |
| **独立框架** | 完全从零构建，不依赖 LangChain 等其他框架    |
| **高性能**   | 优化了速度和资源使用，执行速度快             |
| **灵活定制** | 从高层工作流到底层 Agent 行为都可以精细控制  |
| **生产就绪** | 适合企业级生产环境部署                       |
| **丰富工具** | 支持网页搜索、文件读取、数据库查询等多种工具 |

---

## 二、核心概念

CrewAI 有四个核心概念，理解它们是入门的关键：

### 2.1 Agent（智能体）

Agent 是框架中的基本工作单元，类似于一个"有特定技能的员工"。

每个 Agent 有三个核心属性：

- **role（角色）**：Agent 的身份，如"高级数据研究员"
- **goal（目标）**：Agent 要达成的目标
- **backstory（背景故事）**：Agent 的背景和个性描述

```python
from crewai import Agent

researcher = Agent(
    role="高级数据研究员",
    goal="发现 AI 领域的最新进展",
    backstory="你是一位经验丰富的研究员，擅长发现最新技术趋势。",
    verbose=True  # 开启详细日志
)
```

### 2.2 Task（任务）

Task 是 Agent 要完成的具体工作，类似于"分配给员工的任务单"。

每个 Task 包含：

- **description（描述）**：任务的具体内容
- **expected_output（期望输出）**：任务完成后应该产出什么
- **agent（执行者）**：由哪个 Agent 负责执行

```python
from crewai import Task

research_task = Task(
    description="研究 AI Agent 的最新发展，找出 10 个最重要的趋势",
    expected_output="一份包含 10 个要点的 AI 趋势报告",
    agent=researcher
)
```

### 2.3 Crew（团队）

Crew 是 Agent 的集合，定义了 Agent 们如何协作完成任务。

两种执行模式：

- **Sequential（顺序执行）**：任务按定义顺序依次执行
- **Hierarchical（层级执行）**：由一个"经理"Agent 协调分配任务

```python
from crewai import Crew, Process

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,  # 顺序执行
    verbose=True
)

result = crew.kickoff()  # 启动执行
```

### 2.4 Flow（工作流）

Flow 是 CrewAI 的生产级编排框架，用于构建复杂的多步骤工作流。它是整个应用的"骨架"，管理状态和执行顺序。

```python
from crewai.flow.flow import Flow, listen, start
from pydantic import BaseModel

class ResearchState(BaseModel):
    topic: str = ""
    report: str = ""

class ResearchFlow(Flow[ResearchState]):
    @start()
    def prepare_topic(self):
        self.state.topic = "AI Agents"

    @listen(prepare_topic)
    def run_research(self):
        # 在这里调用 Crew 执行研究
        result = ResearchCrew().crew().kickoff(inputs={"topic": self.state.topic})
        self.state.report = result.raw
```

### 概念关系图

```
Flow（工作流骨架）
  │
  ├── Step 1: 准备数据
  │
  ├── Step 2: 调用 Crew 执行分析
  │     │
  │     └── Crew（团队）
  │           ├── Agent 1（研究员）→ Task 1（搜索信息）
  │           └── Agent 2（分析师）→ Task 2（分析数据）
  │
  └── Step 3: 保存结果
```

---

## 三、安装与环境准备

### 3.1 环境要求

- Python >= 3.10，< 3.14
- 建议使用 `uv` 作为包管理工具

### 3.2 安装步骤

**第一步：安装 uv（如果尚未安装）**

Windows 用户（PowerShell）：

```shell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

macOS/Linux 用户：

```shell
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**第二步：安装 CrewAI CLI**

```shell
uv tool install crewai
```

**第三步：验证安装**

```shell
uv tool list
```

**第四步：安装 CrewAI 包（在项目中）**

```shell
uv pip install crewai
# 如果需要额外工具（搜索等）
uv pip install 'crewai[tools]'
```

### 3.3 配置 API Key

在项目根目录创建 `.env` 文件：

```env
OPENAI_API_KEY=sk-your-key-here
SERPER_API_KEY=your-serper-key-here    # 可选，用于网页搜索
```

---

## 四、快速上手：创建你的第一个项目

### 4.1 创建 Crew 项目

```shell
crewai create crew my-first-crew
cd my_first_crew
```

项目结构：

```
my_first_crew/
├── .env                 # 环境变量（API Key）
├── pyproject.toml       # 项目依赖配置
├── src/
│   └── my_first_crew/
│       ├── main.py      # 入口文件
│       ├── crew.py      # Crew 定义
│       ├── config/
│       │   ├── agents.yaml   # Agent 配置
│       │   └── tasks.yaml    # Task 配置
│       └── tools/
│           └── custom_tool.py
```

### 4.2 定义 Agent（agents.yaml）

```yaml
# src/my_first_crew/config/agents.yaml
researcher:
  role: >
    {topic} 高级数据研究员
  goal: >
    发现 {topic} 领域的最新进展
  backstory: >
    你是一位经验丰富的研究员，擅长发现最新技术趋势，
    并以清晰简洁的方式呈现信息。

writer:
  role: >
    {topic} 内容作者
  goal: >
    根据研究结果撰写清晰的文章
  backstory: >
    你是一位优秀的技术作者，擅长将复杂概念
    转化为易于理解的内容。
```

### 4.3 定义 Task（tasks.yaml）

```yaml
# src/my_first_crew/config/tasks.yaml
research_task:
  description: >
    对 {topic} 进行全面研究，找出最重要的发展趋势。
    当前年份是 2026 年。
  expected_output: >
    一份包含 10 个要点的 {topic} 最新信息列表
  agent: researcher

writing_task:
  description: >
    根据研究结果，撰写一篇详细的技术文章。
    确保内容丰富、结构清晰。
  expected_output: >
    一篇完整的技术文章，使用 Markdown 格式
  agent: writer
  output_file: article.md
```

### 4.4 编写 Crew 逻辑（crew.py）

```python
# src/my_first_crew/crew.py
from crewai import Agent, Crew, Process, Task
from crewai.project import CrewBase, agent, crew, task
from crewai_tools import SerperDevTool

@CrewBase
class MyFirstCrew():
    """我的第一个 Crew"""

    @agent
    def researcher(self) -> Agent:
        return Agent(
            config=self.agents_config['researcher'],
            verbose=True,
            tools=[SerperDevTool()]  # 网页搜索工具
        )

    @agent
    def writer(self) -> Agent:
        return Agent(
            config=self.agents_config['writer'],
            verbose=True
        )

    @task
    def research_task(self) -> Task:
        return Task(
            config=self.tasks_config['research_task']
        )

    @task
    def writing_task(self) -> Task:
        return Task(
            config=self.tasks_config['writing_task']
        )

    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,
            tasks=self.tasks,
            process=Process.sequential,
            verbose=True,
        )
```

### 4.5 运行项目

```bash
# 安装依赖
crewai install

# 运行
crewai run
```

或者：

```bash
python src/my_first_crew/main.py
```

---

## 五、快速上手：创建你的第一个 Flow 项目

Flow 是 CrewAI 推荐的生产级应用架构。

### 5.1 创建 Flow 项目

```shell
crewai create flow my-first-flow
cd my_first_flow
```

### 5.2 Flow 核心装饰器

| 装饰器            | 作用                     |
| ----------------- | ------------------------ |
| `@start()`        | 标记 Flow 的起始方法     |
| `@listen(method)` | 监听某个方法完成后触发   |
| `@router(method)` | 根据返回值决定下一步走向 |
| `@human_feedback` | 暂停等待人类反馈         |

### 5.3 Flow 示例

```python
from crewai.flow.flow import Flow, listen, start
from pydantic import BaseModel

class MyState(BaseModel):
    input_data: str = ""
    result: str = ""

class MyFlow(Flow[MyState]):
    @start()
    def step_one(self):
        self.state.input_data = "Hello CrewAI"
        return self.state.input_data

    @listen(step_one)
    def step_two(self, data):
        self.state.result = f"处理完成: {data}"
        return self.state.result

flow = MyFlow()
result = flow.kickoff()
print(result)
```

---

## 六、进阶功能

### 6.1 工具（Tools）

工具赋予 Agent 额外能力，如搜索网页、读取文件等。

```python
from crewai_tools import SerperDevTool, FileReadTool, WebsiteSearchTool

agent = Agent(
    role="研究员",
    goal="搜索最新信息",
    tools=[SerperDevTool(), WebsiteSearchTool()]
)
```

常用工具列表：

| 工具                | 用途            |
| ------------------- | --------------- |
| `SerperDevTool`     | Google 搜索     |
| `WebsiteSearchTool` | 网页内容搜索    |
| `FileReadTool`      | 读取文件        |
| `DirectoryReadTool` | 读取目录        |
| `PDFSearchTool`     | PDF 内容搜索    |
| `GithubSearchTool`  | GitHub 仓库搜索 |

### 6.2 结构化输出

使用 Pydantic 模型获取结构化输出：

```python
from pydantic import BaseModel

class Report(BaseModel):
    title: str
    summary: str
    key_findings: list[str]

task = Task(
    description="撰写研究报告",
    expected_output="结构化的研究报告",
    agent=researcher,
    output_pydantic=Report
)

result = crew.kickoff()
print(result.pydantic.title)
print(result.pydantic.key_findings)
```

### 6.3 异步执行

```python
# 异步运行 Crew
result = await crew.akickoff(inputs={"topic": "AI"})

# 异步运行 Flow
result = await flow.kickoff_async()
```

### 6.4 Memory（记忆）

Agent 可以保持对话记忆，跨任务保留上下文：

```python
agent = Agent(
    role="分析师",
    goal="分析数据",
    memory=True  # 启用记忆
)
```

### 6.5 Guardrails（护栏/验证）

为 Task 输出添加验证：

```python
from crewai import TaskOutput

def validate_output(result: TaskOutput):
    if len(result.raw.split()) < 100:
        return (False, "输出太短，至少需要 100 个词")
    return (True, result.raw)

task = Task(
    description="写一篇文章",
    expected_output="一篇有深度的文章",
    agent=writer,
    guardrail=validate_output
)
```

---

## 七、常用 CLI 命令

| 命令                         | 说明              |
| ---------------------------- | ----------------- |
| `crewai create crew <name>`  | 创建 Crew 项目    |
| `crewai create flow <name>`  | 创建 Flow 项目    |
| `crewai run`                 | 运行当前项目      |
| `crewai install`             | 安装项目依赖      |
| `crewai flow plot`           | 可视化 Flow       |
| `crewai replay -t <task_id>` | 从某个任务重放    |
| `crewai log-tasks-outputs`   | 查看任务输出日志  |
| `crewai deploy create`       | 部署到 CrewAI AMP |

---

## 八、后端程序员的 CrewAI 思维映射

作为后端程序员，你可以这样类比理解 CrewAI 的概念：

| 后端概念                 | CrewAI 对应概念           |
| ------------------------ | ------------------------- |
| 微服务架构               | Crew（多个 Agent 协作）   |
| 工作流引擎（如 Airflow） | Flow（编排执行流程）      |
| 函数/方法                | Task（具体任务）          |
| 类/对象                  | Agent（有状态的工作单元） |
| 中间件/插件              | Tools（扩展能力）         |
| 数据库                   | Memory + Knowledge        |
| API 网关                 | LLM 配置（连接不同模型）  |
| 消息队列                 | Agent 之间的通信          |
| 验证器/拦截器            | Guardrails（输出验证）    |

---

## 九、学习资源

- **官方文档**: https://docs.crewai.com
- **GitHub 仓库**: https://github.com/crewAIInc/crewAI
- **示例项目**: https://github.com/crewAIInc/crewAI-examples
- **社区论坛**: https://community.crewai.com
- **在线课程**: https://learn.crewai.com

---

## 十、新手建议提问清单

作为一个刚接触 CrewAI 的后端程序员，以下是你在学习过程中可能会遇到的疑问，建议逐步探索和提问：

### 基础理解类

1. **Agent 和 LLM 的关系是什么？** Agent 是如何调用大语言模型的？可以切换不同的模型吗（如 OpenAI、本地模型 Ollama）？
2. **`role`、`goal`、`backstory` 这些提示词对 Agent 的行为有多大影响？** 如何写出好的提示词？
3. **Sequential 和 Hierarchical 两种 Process 模式有什么区别？** 什么场景下该用哪种？
4. **`kickoff()` 和 `kickoff_async()` 的区别是什么？** 什么时候需要异步执行？

### 实践操作类

5. **如何给 Agent 添加自定义工具？** 如何编写一个连接数据库的 Tool？
6. **如何让多个 Task 共享上下文？** `context` 参数怎么使用？
7. **如何让 Agent 的输出是结构化的 JSON 或 Pydantic 对象？** `output_pydantic` 和 `output_json` 有什么区别？
8. **如何在 Flow 中使用条件分支（Router）？** 如何实现复杂的业务流程编排？

### 架构设计类

9. **Crew 和 Flow 应该如何选择？** 是直接用 Crew 还是放在 Flow 里面？
10. **如何设计一个生产级的多 Agent 系统？** 有哪些最佳实践和架构模式？
11. **Flow 的状态管理（State）怎么做持久化？** 如何实现断点续传（Checkpoint）？
12. **如何实现 Human-in-the-Loop（人类参与审核）的工作流？**

### 进阶探索类

13. **Agent 的 Memory 机制是如何工作的？** 短期记忆、长期记忆、实体记忆有什么区别？
14. **如何为 Crew 添加知识库（Knowledge）？** RAG 是怎么集成的？
15. **如何监控和调试 Agent 的执行过程？** 有哪些可观测性工具？
16. **CrewAI 如何连接到不同的 LLM 提供商？** 如何使用本地模型（如 Ollama）？
17. **如何编写自定义的 Guardrail 来验证 Agent 的输出质量？**
18. **如何将 CrewAI 项目部署到生产环境？** CrewAI AMP 是什么？

---

> 提示：以上问题可以直接向我提问，我会结合 CrewAI 的源码和文档给你详细的解答和代码示例。
