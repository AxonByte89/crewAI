# CrewAI Memory 机制学习笔记

> 基于 CrewAI 官方文档和源码的学习总结，涵盖统一 Memory 系统的工作原理、组件分工、使用方式及延伸思考。

---

## 一、核心结论

CrewAI 现在使用**统一的 `Memory` 类**，已废弃旧版的"短期记忆 / 长期记忆 / 实体记忆"三种独立类型。新版用一个智能的、自组织的记忆系统替代，核心特点：

- **LLM 自动分析**：存记忆时，LLM 自动推断 scope（作用域）、categories（分类）、importance（重要性）
- **Embedding 向量化**：把文本转成向量，用于语义相似度搜索
- **LanceDB 本地存储**：嵌入式向量数据库，数据持久化在 `.crewai/memory` 目录
- **复合评分检索**：语义相似度 + 时间衰减 + 重要性，三维度综合排序
- **自动去重合并**：相似内容自动识别并合并，防止记忆膨胀

---

## 二、Memory 默认关闭，需要手动开启

```python
# 没有 Memory（默认）—— Agent 每次"失忆"
crew = Crew(agents=[...], tasks=[...])

# 开启 Memory —— 加 memory=True 即可
crew = Crew(agents=[...], tasks=[...], memory=True)
```

开启后框架自动做两件事：

| 时机            | 自动行为                                                     |
| --------------- | ------------------------------------------------------------ |
| **Task 执行前** | 从 Memory 检索相关内容，注入 Agent prompt（让 Agent "回忆"） |
| **Task 执行后** | 把 Task 输出拆成原子事实，存入 Memory（帮 Agent "记笔记"）   |

---

## 三、三个组件的分工

Memory 系统由三个组件协作，各司其职：

```
存入一条记忆 "我们决定用 PostgreSQL"
    ↓
① LLM (gpt-4o-mini) 分析：
   → scope: /project/decisions
   → 分类: ["architecture", "database"]
   → 重要性: 0.8
    ↓
② Embedding 模型 (text-embedding-3-small) 向量化：
   → [0.023, -0.041, 0.118, ...] （1536维向量）
    ↓
③ LanceDB 存储：
   → 文本 + 向量 + 元数据 一起存下来
```

| 组件               | 角色   | 做什么                                       | 默认方案                       |
| ------------------ | ------ | -------------------------------------------- | ------------------------------ |
| **LLM**            | 理解者 | 分析内容含义，推断分类/重要性/scope          | gpt-4o-mini                    |
| **Embedding 模型** | 编码器 | 把文本压缩成向量（一串数字），用于相似度计算 | OpenAI text-embedding-3-small  |
| **LanceDB**        | 仓库   | 存储向量和元数据，执行相似度检索             | 本地嵌入式，随 CrewAI 自动安装 |

---

## 四、Embedding 模型 vs LLM 的区别

这是初学者容易混淆的概念，两者做的事完全不同：

| 维度       | LLM（大语言模型）           | Embedding 模型（嵌入模型）         |
| ---------- | --------------------------- | ---------------------------------- |
| **做什么** | 理解文字 → **生成**新的文字 | 理解文字 → **压缩**成一串数字      |
| **输出**   | 一段文本（人类能读懂）      | 一个向量（一串浮点数，人类看不懂） |
| **用途**   | 对话、写作、推理、决策      | 计算两段文本之间的"相似度"         |
| **类比**   | 会写文章的**作家**          | 给文章贴指纹标签的**档案员**       |

### 为什么需要 Embedding 而不是关键词搜索？

关键词搜索只能匹配字面相同的词：

```
搜索 "数据库" → 匹配 "PostgreSQL 数据库" ✓
搜索 "数据库" → 匹配不到 "我们选了 PG 做存储方案" ✗
```

Embedding 能理解语义，即使没有一个相同的字，只要意思相近就能找到：

```
"数据库" 的向量      →  [0.2, -0.4, 0.8, ...]
"PG 存储方案" 的向量  →  [0.2, -0.3, 0.7, ...]  ← 很接近！✓
"火锅" 的向量        →  [-0.7, 0.3, -0.1, ...] ← 很远 ✗
```

### 图书馆类比

> - **LLM** = 图书馆的**咨询员** —— 你问问题，他理解并给建议
> - **Embedding 模型** = 给每本书贴**指纹标签**的机器 —— 意思相近的书标签就相近
> - **LanceDB** = **书架系统** —— 按指纹标签排列，快速找到相似的书

---

## 五、LanceDB 不需要手动安装

LanceDB 作为 CrewAI 的依赖项写在 `pyproject.toml` 中：

```toml
"lancedb>=0.29.2,<0.30.1"
```

安装 CrewAI 时自动安装。LanceDB 是**嵌入式本地向量数据库**（类似 SQLite 的理念）：

- 不需要启动服务
- 不需要 Docker
- 不需要单独安装数据库软件
- 直接在 Python 进程内运行
- 数据以文件形式持久化在 `.crewai/memory` 目录

---

## 六、Memory 是否需要 RAG？

**Memory 内部用到了 RAG 的核心技术，但你不需要自己搭建 RAG 系统。**

RAG 的本质是"先检索再生成"，Memory 的工作方式就是这个模式：

```
Agent 要执行任务
    ↓
① 检索：从 LanceDB 中找相关记忆（向量相似度搜索）
    ↓
② 增强：把检索到的记忆注入 Agent 的 prompt
    ↓
③ 生成：Agent 带着"回忆"去执行任务
```

框架内置了所有 RAG 组件（向量数据库、Embedding、检索策略、去重），开箱即用。

> **注意**：CrewAI 还有一个独立的 **Knowledge** 功能，那个才是让你导入自己的文档（PDF、Markdown 等）做 RAG 的。Memory 是 Agent 自动"记笔记"，Knowledge 是你主动"喂文档"，两者不同。

---

## 七、层级作用域（Scope）

Memory 用类似文件夹的层级结构组织记忆，LLM 自动推断存放位置：

```
/                           ← 根目录（全局共享）
  /project
    /project/alpha          ← Alpha 项目专属
    /project/beta           ← Beta 项目专属
  /agent
    /agent/researcher       ← 研究员私有
    /agent/writer           ← 写手私有
```

支持 scope 隔离，不同 Agent 可以有不同的记忆可见范围：

```python
# 研究员有私有记忆空间
researcher = Agent(
    role="研究员",
    memory=memory.scope("/agent/researcher"),
)

# 写手用 Crew 共享记忆
writer = Agent(role="写手", ...)
```

---

## 八、复合评分检索

检索时用三个维度综合打分：

```
最终得分 = 语义权重(0.5) × 语义相似度
         + 时间权重(0.3) × 时间衰减
         + 重要性权重(0.2) × 重要性分数
```

| 维度           | 含义                          | 默认权重 |
| -------------- | ----------------------------- | -------- |
| **语义相似度** | 意思越接近分越高（向量搜索）  | 0.5      |
| **时间衰减**   | 越新的记忆分越高（指数衰减）  | 0.3      |
| **重要性**     | 存储时 LLM 自动评估的重要程度 | 0.2      |

可根据场景调整权重：

```python
# 快节奏项目：更看重新信息
memory = Memory(recency_weight=0.5, semantic_weight=0.3, importance_weight=0.2, recency_half_life_days=7)

# 知识库：更看重重要信息
memory = Memory(recency_weight=0.1, semantic_weight=0.5, importance_weight=0.4, recency_half_life_days=180)
```

---

## 九、两种检索深度

| 模式             | 说明                                 | 适用场景          |
| ---------------- | ------------------------------------ | ----------------- |
| **shallow**      | 纯向量搜索，不调 LLM，快（~200ms）   | 常规 Agent 上下文 |
| **deep**（默认） | LLM 分析意图 + 多步检索 + 置信度路由 | 复杂查询          |

```python
memory.recall("数据库选型", depth="shallow")   # 快速
memory.recall("总结本季度所有架构决策", depth="deep")  # 深度
```

---

## 十、自动合并（Consolidation）

存储相似内容时不会堆积重复记录：

```python
memory.remember("CrewAI 支持复杂工作流")
memory.remember("CrewAI 支持复杂工作流")      # 识别重复，合并为一条
memory.remember("CrewAI 支持更复杂的多Agent工作流")  # LLM 判断更新旧记录
```

批量存储时还有**批内去重**（纯向量计算，不调 LLM）：

```python
memory.remember_many([
    "CrewAI 支持复杂工作流。",
    "Python 是一门好语言。",
    "CrewAI 支持复杂工作流。",  # 与第一条近似重复，自动丢弃
])
```

---

## 十一、旧版 vs 新版对比

| 维度         | 旧版（已废弃）                                             | 新版（统一 Memory）              |
| ------------ | ---------------------------------------------------------- | -------------------------------- |
| **结构**     | ShortTermMemory / LongTermMemory / EntityMemory 三个独立类 | 一个统一的 `Memory` 类           |
| **组织方式** | 固定分类，人工预设                                         | LLM 自动推断 scope 和分类        |
| **检索**     | 简单匹配                                                   | 复合评分（语义 + 时间 + 重要性） |
| **去重**     | 无                                                         | 自动合并（consolidation）        |

---

## 十二、全景架构图

```
┌──────────────────────────────────────────────────────┐
│                   统一 Memory 系统                     │
│                                                        │
│  ┌─────────┐     ┌──────────┐     ┌───────────────┐  │
│  │ 存入记忆 │ ──→ │ LLM 分析  │ ──→ │ 向量化 + 存储  │  │
│  │remember()│     │scope/分类 │     │  LanceDB 本地  │  │
│  └─────────┘     │/重要性    │     └───────────────┘  │
│                   └──────────┘                         │
│                                                        │
│  ┌─────────┐     ┌──────────┐     ┌───────────────┐  │
│  │ 检索记忆 │ ──→ │ 复合评分  │ ──→ │ 返回排序结果   │  │
│  │ recall() │     │语义+时间  │     │  注入 Agent    │  │
│  └─────────┘     │+重要性    │     │  prompt        │  │
│                   └──────────┘     └───────────────┘  │
│                                                        │
│  自动去重 │ 自动提取原子事实 │ 层级 Scope 隔离        │
└──────────────────────────────────────────────────────┘

开启方式：crew = Crew(..., memory=True)
存储位置：.crewai/memory（本地持久化）
依赖：    Embedding 模型（默认 OpenAI，可换本地 Ollama）
         LLM（默认 gpt-4o-mini，可换其他模型）
```

---

## 十三、Memory vs Knowledge：什么时候用哪个？

### 一句话区分

| 概念          | 定义                              | 类比                                 |
| ------------- | --------------------------------- | ------------------------------------ |
| **Memory**    | Agent 在工作中**自动产生**的笔记  | 员工开会时自己记的**会议纪要**       |
| **Knowledge** | 你**主动导入**给 Agent 的参考资料 | 公司发给员工的**培训手册和规章制度** |

### 核心差异对比

| 维度           | Memory                         | Knowledge                                |
| -------------- | ------------------------------ | ---------------------------------------- |
| **内容来源**   | Agent 执行任务后**自动提取**   | 你**手动导入**的文档（PDF、CSV、文本等） |
| **谁写入**     | 框架自动写入                   | 开发者预先准备                           |
| **动态性**     | 随执行不断增长，越用越多       | 固定的，导入后不变（除非手动更新）       |
| **向量数据库** | LanceDB（`.crewai/memory`）    | ChromaDB（系统目录下的 `knowledge/`）    |
| **开启方式**   | `memory=True`                  | `knowledge_sources=[...]`                |
| **智能分析**   | LLM 自动推断 scope/分类/重要性 | 无（纯向量检索）                         |
| **去重合并**   | 自动 consolidation             | 无                                       |
| **写入时机**   | Task 完成后**增量写入**        | kickoff 时**全量加载**                   |

### 底层原理其实一样

Knowledge **也是存在向量数据库里的**（不是直接塞进 prompt）。它的底层也是 RAG 模式：

```
你导入一个 PDF 手册（100页）
    ↓
① 文档切块（chunk）：按固定大小切成几百个小段落
    ↓
② Embedding 向量化：每个段落转成向量
    ↓
③ 存入 ChromaDB 向量数据库
    ↓
④ Agent 执行任务时，用查询去 ChromaDB 做语义搜索
    ↓
⑤ 返回最相关的几个段落，注入 Agent prompt
```

### 三种使用场景

#### 场景一：只用 Memory

适合 Agent 需要从自己的工作经验中学习的场景：

```python
crew = Crew(
    agents=[researcher],
    tasks=[daily_research_task],
    memory=True,  # 只开 Memory
)
# 第一天：研究发现 A，自动记住
# 第二天：研究发现 B，能回忆昨天的发现 A
# 第七天：Agent 已积累整周的研究经验
```

典型场景：客服记住用户偏好、研究 Agent 跨多次执行积累经验、多 Agent 共享工作成果。

#### 场景二：只用 Knowledge

适合 Agent 需要参考固定的外部资料：

```python
from crewai.knowledge.source.pdf_knowledge_source import PDFKnowledgeSource
from crewai.knowledge.source.csv_knowledge_source import CSVKnowledgeSource

company_policy = PDFKnowledgeSource(file_paths=["company_policy.pdf"])
sales_data = CSVKnowledgeSource(file_paths=["sales_2024.csv"])

crew = Crew(
    agents=[hr_agent, sales_agent],
    tasks=[...],
    knowledge_sources=[company_policy, sales_data],
)
```

典型场景：客服参考产品手册、财务参考税率表、HR 参考员工手册。

#### 场景三：Memory + Knowledge 一起用（最强大）

适合 Agent 既需要参考资料，又需要积累经验：

```python
product_manual = PDFKnowledgeSource(file_paths=["product_manual.pdf"])

crew = Crew(
    agents=[support_agent],
    tasks=[support_task],
    knowledge_sources=[product_manual],  # Knowledge: 产品手册
    memory=True,                          # Memory: 记住历史工单
)
# Agent 既能查产品手册（Knowledge）
# 又能记住之前处理过的类似工单（Memory）
```

> **类比**：想象一个客服员工——Knowledge 是公司发的培训资料（入职第一天就有），Memory 是自己的工作笔记（处理过的工单、客户偏好、踩过的坑）。最厉害的客服 = 既熟读培训资料，又有丰富实战经验。

---

## 十四、Knowledge 每次 kickoff 重复嵌入的问题

### 问题：同一个 PDF 会多次存入 ChromaDB 吗？

**不会重复记录，但会重复做功。**

从源码确认，ChromaDB 中不会出现同一个 PDF 的多份副本，但每次 kickoff 确实会白白浪费算力。

### 原因一：文档 ID 是内容的哈希值

源码 `_prepare_documents_for_chromadb` 中，当没有显式指定 `doc_id` 时，ID 是通过对内容做 SHA-256 哈希生成的：

```python
doc_id = hashlib.sha256(content_for_hash.encode()).hexdigest()
```

同一个 PDF → 切出同样的文本块 → 同样的哈希值 → 同样的 ID。

### 原因二：写入用的是 upsert（覆盖写入）

源码 `add_documents` 内部调用的是 `collection.upsert()`：

```python
collection.upsert(
    ids=batch_ids,
    documents=batch_texts,
    metadatas=batch_metadatas,
)
```

`upsert` = "insert or update"：

- ID 不存在 → 插入新记录
- ID 已存在 → **覆盖更新**（不是追加）

### 实际发生的事情

```
第1次 kickoff：
  PDF → 切500块 → 嵌入500个向量 → upsert 到 ChromaDB
  ChromaDB: 500条记录

第2次 kickoff：
  同一个 PDF → 切同样的500块 → 重新嵌入500个向量（浪费！） → upsert 到 ChromaDB
  ChromaDB: 还是500条记录（ID相同，覆盖写入，数量不变）

第3次 kickoff：
  同上...
  ChromaDB: 依然500条记录
```

**数据库不会膨胀，但 Embedding API 的调用费用和等待时间是白白浪费的。**

### 类比理解

> 就像你每次上班都重新打印一份员工手册，然后走到档案柜前，把旧的那份**替换**成新的。档案柜里始终只有一份手册（不会堆积），但你**浪费了大量纸张和打印时间**。聪明的做法是：第一次打印后放好，以后直接用柜子里的那份，除非手册更新了才重新打印。

### 总结

| 问题                             | 答案                                                |
| -------------------------------- | --------------------------------------------------- |
| ChromaDB 会有重复数据吗？        | **不会**，upsert + 内容哈希保证同 ID 覆盖           |
| 每次 kickoff 会重新嵌入吗？      | **会**，这就是性能浪费所在                          |
| Embedding API 调用会重复计费吗？ | **会**，每次 kickoff 都调用一次 Embedding API       |
| 框架知道这个问题吗？             | **知道**，官方文档已引用 Github Issue #2755，待优化 |

合理的做法应该是：检查 ChromaDB 中是否已存在相同 ID 的记录，存在就跳过嵌入，只在新文档或文档有变更时才重新嵌入。

---

## 十五、延伸思考问题

以下问题值得进一步学习和实践：

1. **Memory 开关实验**：同一个 Crew 分别在 `memory=True` 和 `memory=False` 下执行，观察输出差异
2. **Embedding 模型替换**：数据敏感时如何用本地模型（Ollama）替代 OpenAI？（提示：`embedder` 参数）
3. **Scope 隔离**：3 个项目共用一个 CrewAI 系统，不设 Scope 会不会"串台"？
4. **记忆膨胀**：100 个 Task 后 Memory 积累上千条记录，检索性能如何？时间衰减能缓解吗？
5. **完全离线方案**：LLM + Embedding + Memory 全部本地运行需要配置什么？

---

## 十六、参考资源

- 官方文档：`docs/en/concepts/memory.mdx`
- Knowledge 文档：`docs/en/concepts/knowledge.mdx`
- 源码入口：`lib/crewai/src/crewai/memory/unified_memory.py`
- LanceDB 存储实现：`lib/crewai/src/crewai/memory/storage/lancedb_storage.py`
- Knowledge 存储实现：`lib/crewai/src/crewai/knowledge/storage/knowledge_storage.py`
- ChromaDB 客户端实现：`lib/crewai/src/crewai/rag/chromadb/client.py`
- 入门指南：`docs/zh/crewai-beginner-guide-zh.md`（第 539 行 Q13）
