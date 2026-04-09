# 项目学习规划：langchain-miniopenclaw（逐文件版）

> **重要说明：** 本文档中列出的所有文件均为仓库 `main` 分支真实存在的文件，无虚构路径。  
> 建议先运行项目（参考 `README.md` 快速开始），再按本文件顺序逐一阅读和练习。

---

## 技术栈速览

| 层 | 技术 | 在项目中的作用 |
|---|---|---|
| 后端框架 | Python 3.10 + FastAPI | HTTP 服务、SSE 流式推送、路由、请求校验 |
| Agent 编排 | LangChain 1.x `create_agent` | 构建可调用工具的对话 Agent，处理 Thought-Action 循环 |
| LLM 接入 | `langchain-openai` / `langchain-deepseek` | 封装模型调用（ChatOpenAI / ChatDeepSeek），流式输出 |
| RAG / 向量检索 | LlamaIndex Core + OpenAI Embeddings | 索引 Memory 和知识库文件，语义召回，增量缓存 |
| 配置管理 | `python-dotenv` + 自定义 `Settings` | `.env` 读取、多 Provider 适配、运行时配置 |
| Token 计数 | `tiktoken` | 统计 system prompt + 消息的 token 用量 |
| 技能系统 | 纯文件（`SKILL.md` + YAML frontmatter） | 描述每个技能的能力和执行步骤，供 Agent 读取 |
| Prompt 组装 | 多 Markdown 文件实时拼接 | 将 workspace/*.md + memory + skills 拼成系统提示词 |
| 会话持久化 | JSON 文件（`sessions/*.json`） | 落盘对话记录、摘要上下文 |
| 前端框架 | Next.js 14 App Router + React 18 | 三栏工作台页面，SSR/CSR 混合 |
| 前端状态 | React Context API（自定义 Store） | 全局会话状态、流式消息更新、文件编辑 |
| 样式 | Tailwind CSS | 原子化 CSS，三栏布局 |
| 代码编辑器 | Monaco Editor（`@monaco-editor/react`） | 内嵌在右栏，用于在线编辑 Memory / Skills / Workspace 文件 |
| Markdown 渲染 | `react-markdown` + `remark-gfm` | 前端渲染 AI 回复中的 Markdown + 表格 |
| HTTP 客户端 | `httpx`（后端）/ `fetch`（前端） | 后端工具抓取外部 URL；前端调用后端 REST API + SSE |

---

## 学习路径总览

```
0 阶段：项目认知（README + 环境）
│
├── 1 阶段：配置层（config.py + .env.example + requirements.txt）
│
├── 2 阶段：后端核心
│   ├── 应用入口（app.py）
│   ├── Prompt 组装（graph/prompt_builder.py + workspace/*.md + SKILLS_SNAPSHOT.md）
│   ├── 技能系统（tools/skills_scanner.py + skills/*/SKILL.md）
│   ├── 记忆层（memory/MEMORY.md + graph/memory_indexer.py）
│   ├── 会话层（graph/session_manager.py）
│   ├── Agent 执行（graph/agent.py）
│   ├── 工具层（tools/__init__.py + 各 tool 文件）
│   └── API 层（api/*.py）
│
└── 3 阶段：前端
    ├── 工程配置（package.json + tsconfig.json + tailwind.config.ts）
    ├── 应用入口（src/app/layout.tsx + globals.css + page.tsx）
    ├── API 客户端（src/lib/api.ts）
    ├── 全局状态（src/lib/store.tsx）
    └── 界面组件（components/*）
```

---

## 第 0 阶段：项目认知（第 1 天）

### 文件 1：`README.md`

- **技术点：** 无代码，理解项目定位
- **本文件在项目中的角色：** 描述系统架构，给出快速开始步骤，解释"文件即记忆""技能即插件""Prompt 可解释"三大设计理念
- **学习任务：**
  1. 阅读 `## 一次请求发生了什么` 中的 mermaid 流程图，手绘一遍（不超过 10 个节点）
  2. 找到项目与普通 Agent 的 5 条对比（表格），理解每条代表什么取舍
  3. 按 `## 快速开始` 把后端和前端都跑起来，发送一条消息拿到回复
- **验收：** 能用自己的话说清楚"用户发一条消息后，后端经过哪些步骤返回 SSE 流"

---

## 第 1 阶段：配置层

### 文件 2：`backend/.env.example`

- **技术点：** 环境变量、多 LLM Provider 适配
- **本文件在项目中的角色：** 定义所有可配置的运行参数——LLM Provider（zhipu/bailian/deepseek/openai）、Embedding Provider、API Key、Tavily 搜索 Key
- **学习任务：**
  1. 列出所有变量及其作用（一句话/个）
  2. 理解为什么同一个 Provider（如 bailian）有多个别名变量（`BAILIAN_API_KEY` / `DASHSCOPE_API_KEY`）
  3. 决定你自己使用哪个 LLM Provider，填写对应 Key
- **验收：** 能判断最少需要填哪几个变量才能让系统正常启动

---

### 文件 3：`backend/requirements.txt`

- **技术点：** Python 包管理，依赖版本锁定
- **本文件在项目中的角色：** 声明后端所有直接依赖及版本范围
- **学习任务：**
  1. 识别哪些包属于"框架层"（fastapi、langchain），哪些属于"工具层"（tiktoken、html2text），哪些属于"基础设施层"（uvicorn、httpx、python-dotenv）
  2. 找到哪个包负责向量索引（llama-index-core），哪个包负责 Embedding（llama-index-embeddings-openai）
  3. 思考为何不需要安装 `requests` 而用 `httpx`（异步支持）
- **验收：** 能解释 `langchain-openai` 和 `langchain-deepseek` 的职责差异

---

### 文件 4：`backend/config.py`

- **技术点：** `python-dotenv`、`dataclasses`、`lru_cache`、多 Provider 模式、线程安全配置
- **本文件在项目中的角色：** 单一入口读取 `.env` 并解析成 `Settings` 对象；同时实现 `RuntimeConfigManager` 管理运行时 JSON 配置（如 `rag_mode`）
- **关键函数：**
  - `get_settings()` — 用 `@lru_cache` 缓存，只解析一次
  - `_normalize_provider()` — 归一化 Provider 别名（如 `glm` → `zhipu`）
  - `RuntimeConfigManager.get_rag_mode()` / `set_rag_mode()` — 线程安全读写 `config.json`
- **学习任务：**
  1. 把 `.env` 的 `LLM_PROVIDER` 改成 `deepseek`，重启后端，验证模型切换生效
  2. 在代码中找到"如果既没有 `LLM_MODEL` 也没有 Provider 专属变量，最终用什么模型"的逻辑路径
  3. 理解 `RuntimeConfigManager` 用 `threading.Lock` 的原因（FastAPI 的并发场景）
- **验收：** 能画出"一个 env 变量 → 最终被哪个字段消费"的映射表

---

## 第 2 阶段：后端核心

### 文件 5：`backend/app.py`

- **技术点：** FastAPI、`lifespan` 异步上下文管理器、CORS、路由注册
- **本文件在项目中的角色：** 应用入口，定义启动时的初始化顺序：刷新技能快照 → 初始化 AgentManager → 配置 MemoryIndexer → 重建索引；然后挂载所有子路由
- **关键代码：**
  ```python
  @asynccontextmanager
  async def lifespan(_: FastAPI):
      refresh_snapshot(settings.backend_dir)   # 扫描 skills/ 生成 SKILLS_SNAPSHOT.md
      agent_manager.initialize(settings.backend_dir)
      memory_indexer.configure(settings.backend_dir)
      memory_indexer.rebuild_index()
      yield
  ```
- **学习任务：**
  1. 在 `lifespan` 中添加一行 `print("startup complete")`，重启后确认控制台打印时机
  2. 查看 CORS 配置（`allow_origins=["*"]`），解释生产环境应如何收紧
  3. 访问 `http://localhost:8002/docs`，观察 FastAPI 自动生成的 OpenAPI 文档
- **验收：** 能说清楚"如果 `memory_indexer.rebuild_index()` 失败，后端是否会拒绝启动，为什么"

---

### 文件 6：`backend/graph/prompt_builder.py`

- **技术点：** Prompt 工程、多文件拼接、字符截断
- **本文件在项目中的角色：** 每次请求调用 `build_system_prompt()` 时，从以下文件实时读取内容并拼成系统提示词（顺序固定）：`SKILLS_SNAPSHOT.md` → `workspace/SOUL.md` → `workspace/IDENTITY.md` → `workspace/USER.md` → `workspace/AGENTS.md` → `memory/MEMORY.md`（或 RAG 占位符）
- **关键代码：**
  ```python
  SYSTEM_COMPONENTS = (
      ("Skills Snapshot", "SKILLS_SNAPSHOT.md"),
      ("Soul",            "workspace/SOUL.md"),
      ("Identity",        "workspace/IDENTITY.md"),
      ("User Profile",    "workspace/USER.md"),
      ("Agents Guide",    "workspace/AGENTS.md"),
      ("Long-term Memory","memory/MEMORY.md"),
  )
  ```
- **学习任务：**
  1. 在前端打开 `GET /api/sessions/{session_id}/messages` 或 Swagger 文档，查看完整的系统提示词输出
  2. 修改 `workspace/SOUL.md`，发一条消息，验证 Prompt 是否即时生效（不需要重启）
  3. 理解 `component_char_limit = 20_000` 的作用：当某个组件文件很大时发生什么
- **验收：** 能解释"RAG 模式开启时，`memory/MEMORY.md` 的内容为何不直接拼入系统提示词"

---

### 文件 7–10：`backend/workspace/` 下的 4 个 Markdown 文件

这 4 个文件是系统提示词的"人格层"，每个对应一个概念：

| 文件 | 对应 Prompt 组件 | 作用 |
|---|---|---|
| `backend/workspace/SOUL.md` | Soul | 定义 Agent 的行为哲学（可执行、最小操作、事实与推断分离） |
| `backend/workspace/IDENTITY.md` | Identity | 定义 Agent 的名称和风格（冷静、工程化） |
| `backend/workspace/USER.md` | User Profile | 描述当前用户偏好（中文、文件结构可见） |
| `backend/workspace/AGENTS.md` | Agents Guide | 定义工具使用协议和 Memory 写入协议 |

- **技术点：** Prompt 工程——人格分层、关注点分离
- **学习任务：**
  1. 把 `SOUL.md` 里的行为规则改掉一条，观察 Agent 回复风格是否变化
  2. 在 `USER.md` 里加一条偏好（如"回复尽量简短，不超过 100 字"），验证效果
  3. 思考：为什么把这些内容放在独立 `.md` 文件而不是直接硬编码在 Python 里？（可维护性、可 diff、可在线编辑）
- **验收：** 能在前端右栏 Inspector 面板中直接编辑任意一个文件并保存，发一条消息验证效果即时生效

---

### 文件 11：`backend/SKILLS_SNAPSHOT.md`（自动生成）

- **技术点：** 自动生成的 XML-like 文本索引
- **本文件在项目中的角色：** 由 `skills_scanner.py` 启动时扫描 `skills/*/SKILL.md` 并写入；Agent 通过此文件"知道有哪些技能"，再按需用 `read_file` 读取具体 SKILL.md
- **注意：** 该文件内容每次启动自动覆盖，**不应手动编辑**
- **学习任务：**
  1. 查看文件内容，理解 `<skill name="..." path="...">` 的 schema
  2. 新建 `backend/skills/my-test-skill/SKILL.md`，写入 YAML frontmatter（含 `name` 和 `description`），重启后端，查看快照是否更新
  3. 验证：如果 SKILL.md 缺少 frontmatter，快照会用什么值作为 name
- **验收：** 能说清楚"Agent 如何从'知道有哪些技能'到'执行某技能'"的两步过程

---

### 文件 12：`backend/tools/skills_scanner.py`

- **技术点：** `pathlib.glob`、YAML frontmatter 解析（`PyYAML`）、正则表达式
- **本文件在项目中的角色：** 实现扫描逻辑——读取所有 `skills/*/SKILL.md` 的 YAML frontmatter（`name`、`description`），生成 `SKILLS_SNAPSHOT.md` 内容并写入磁盘
- **关键函数：**
  - `scan_skills(base_dir)` — 返回 `SkillRecord` 列表
  - `build_snapshot(skills)` — 生成 XML-like 快照字符串
  - `refresh_snapshot(base_dir)` — 扫描并写文件（在 `app.py` 启动和 `files.py` 保存技能文件后调用）
- **学习任务：**
  1. 在 Python REPL 或调试器里手动调用 `scan_skills(Path("backend"))` 查看输出
  2. 修改某个 SKILL.md 的 `description` 字段，通过 API `POST /api/files` 保存，验证快照自动刷新
  3. 理解为什么 `FRONTMATTER_PATTERN` 只匹配文件开头的 `---` 块
- **验收：** 能用一句话解释"什么触发快照刷新，什么不触发"

---

### 文件 13–16：`backend/skills/` 下的各 `SKILL.md`

以下是仓库中实际存在的技能文件（每个对应一种能力）：

| 文件 | 技能名 | 技术点 |
|---|---|---|
| `backend/skills/get_weather/SKILL.md` | 天气查询 | python_repl + requests、fallback 策略、Weather API |
| `backend/skills/rag-skill/SKILL.md` | 本地知识库检索 (kb-retriever) | 分层目录索引、PDF/Excel 处理流程、grep + 迭代检索 |
| `backend/skills/web-search/SKILL.md` | 联网搜索 | Tavily API、查询策略、domain filter、topic 分类 |
| `backend/skills/retry-lesson-capture/SKILL.md` | 失败恢复经验沉淀 | 自动写回 SKILL.md 和 MEMORY.md 的流程 |

- **技术点：** "技能即插件"设计模式、YAML frontmatter、SKILL.md 格式约定
- **学习任务：**
  1. 阅读 `get_weather/SKILL.md`，理解"经验教训"条目是如何避免重复失败的
  2. 阅读 `retry-lesson-capture/SKILL.md`，理解 Agent 如何"自学"并把经验写回文件
  3. 自己写一个最小技能：`backend/skills/hello-world/SKILL.md`，让 Agent 学会用 `python_repl` 打印"Hello World"
- **验收：** 能解释"为什么技能用 Markdown 而不是 Python 函数，各有什么优缺点"

---

### 文件 17：`backend/memory/MEMORY.md`

- **技术点：** 文件即记忆、长期记忆的 Markdown 结构
- **本文件在项目中的角色：** Agent 的长期记忆主文件，保存跨会话有价值的信息；RAG 模式下被向量索引，非 RAG 模式下直接拼入系统提示词
- **学习任务：**
  1. 查看当前 MEMORY.md 内容，理解现有记忆条目的格式
  2. 在前端 Inspector 右栏直接编辑并添加一条记忆（如"用户偏好简洁回答"），保存后发一条消息，观察 Agent 是否使用了这条记忆
  3. 思考：如果 MEMORY.md 很大（比如超过 `component_char_limit=20000` 字符），直接拼入 Prompt 会发生什么？这是 RAG 模式存在的原因
- **验收：** 能说清楚"RAG 模式和直接拼入模式的 token 消耗差异和召回精度差异"

---

### 文件 18：`backend/graph/memory_indexer.py`

- **技术点：** LlamaIndex `VectorStoreIndex`、`SentenceSplitter`、`OpenAIEmbedding`、增量索引（MD5 摘要判断）
- **本文件在项目中的角色：** 对 `memory/MEMORY.md` 建立向量索引（chunk_size=256, overlap=32），缓存到 `storage/memory_index/`；RAG 模式下通过 `retrieve(query, top_k=3)` 语义召回相关片段
- **关键流程：**
  ```
  rebuild_index() → 读取 MEMORY.md → SentenceSplitter 切片 →
  OpenAIEmbedding 向量化 → VectorStoreIndex → 持久化到 storage/
  ```
- **学习任务：**
  1. 修改 `chunk_size` 参数（如改为 128），重建索引，比较召回结果的颗粒度变化
  2. 删除 `storage/memory_index/` 目录内容（保留 `.gitkeep`），重启后端，观察索引自动重建
  3. 理解 `_file_digest()` 的用途：为什么用 MD5 而不是每次都重建
- **验收：** 能解释"当 embedding_api_key 未配置时，MemoryIndexer 会如何降级处理"

---

### 文件 19：`backend/graph/session_manager.py`

- **技术点：** 本地 JSON 持久化、UUID、会话归档、上下文压缩
- **本文件在项目中的角色：** 管理 `sessions/` 目录下的 JSON 文件，实现会话的增删改查、消息追加、历史压缩（把早期消息归档到 `sessions/archive/`，保留摘要）
- **关键方法：**
  - `load_session_for_agent()` — 合并 `compressed_context` + 消息历史，并去除连续的 assistant 消息
  - `compress_history()` — 把前 N 条消息移到 archive，更新 `compressed_context`
- **学习任务：**
  1. 发几条消息后，直接打开 `backend/sessions/{session_id}.json` 查看数据结构
  2. 调用 `POST /api/sessions/{session_id}/compress`，对比压缩前后 JSON 文件的变化
  3. 理解为什么 `load_session_for_agent` 要合并连续的 assistant 消息（LLM API 对 role 顺序的要求）
- **验收：** 能解释"会话摘要功能在哪里触发、摘要内容存在哪里、什么时候被拼入 Prompt"

---

### 文件 20：`backend/graph/agent.py`

- **技术点：** LangChain `create_agent`、`ChatOpenAI`/`ChatDeepSeek`、异步 SSE 流式输出、Tool Call 解析
- **本文件在项目中的角色：** 核心执行引擎。`AgentManager.astream()` 驱动整个请求：加载 RAG 检索结果 → 构建 Agent → 调用 `agent.astream()` → 解析 messages/updates 双流 → yield 不同类型的事件（token / tool_start / tool_end / retrieval / done）
- **关键代码：**
  ```python
  async for mode, payload in agent.astream(
      {"messages": messages},
      stream_mode=["messages", "updates"],   # 同时监听 token 流和工具调用
  ):
  ```
- **学习任务：**
  1. 在 `astream()` 里加 `print(f"mode={mode}")` 日志，发一条触发工具调用的消息，观察 mode 切换过程（messages → updates）
  2. 把 `temperature=0` 改成 `temperature=0.7`，对比同一问题的回复差异
  3. 理解 `generate_title()` 和 `summarize_history()` 分别在什么场景被调用
- **验收：** 能画出"一次工具调用的完整事件序列：tool_start → tool_end → new_response → token → done"

---

### 文件 21：`backend/tools/__init__.py`

- **技术点：** LangChain `BaseTool` 注册
- **本文件在项目中的角色：** 工具注册表，把 5 个工具聚合成列表传给 `create_agent`
- **学习任务：**
  1. 阅读代码，理解 `get_all_tools(base_dir)` 返回的 5 个工具的名字（agent 调用时用的就是这些名字）
  2. 在这里注释掉 `TerminalTool`，重启后端，发一条需要用 terminal 的消息，观察 Agent 如何降级
- **验收：** 能解释"LangChain Agent 如何知道有哪些工具可用（工具的 name + description + args_schema）"

---

### 文件 22：`backend/tools/read_file_tool.py`

- **技术点：** LangChain `BaseTool` 子类、`Pydantic` 参数 schema、路径遍历防护
- **本文件在项目中的角色：** 让 Agent 能读取项目根目录下的文件（用于读取 SKILL.md、workspace/*.md 等），限制在 `root_dir` 之内防止路径穿越
- **关键安全代码：**
  ```python
  if self._root_dir not in candidate.parents and candidate != self._root_dir:
      raise ValueError("Path traversal detected.")
  ```
- **学习任务：**
  1. 理解 `PrivateAttr` 的作用（不被 Pydantic 序列化，避免 `root_dir` 泄漏）
  2. 测试：让 Agent 读取 `skills/get_weather/SKILL.md`，查看 tool_start 事件中的 `input` 参数
  3. 思考：如果 Agent 传入 `../../etc/passwd` 会发生什么（路径防护的触发路径）
- **验收：** 能解释"同步 `_run` 和异步 `_arun` 为什么都要实现，FastAPI 调用哪个"

---

### 文件 23：`backend/tools/terminal_tool.py`

- **技术点：** `subprocess.run`、跨平台 shell（Windows PowerShell / Unix bash）、命令黑名单、超时控制
- **本文件在项目中的角色：** 让 Agent 能在项目根目录执行 shell 命令（如 pip 安装、文件列举），同时通过 `BLOCKED_PATTERNS` 防止高危命令
- **学习任务：**
  1. 查看 `BLOCKED_PATTERNS` 列表，思考是否有遗漏（安全边界讨论）
  2. 把 `terminal_timeout_seconds` 从 30 改为 5（在 `config.py`），测试一个耗时超 5 秒的命令
  3. 理解为何 Windows 用 `powershell`，Unix 用 `bash -lc`（`-lc` 加载 profile）
- **验收：** 能说清楚"Terminal Tool 的安全边界在哪里，还有哪些场景可能被绕过"

---

### 文件 24：`backend/tools/python_repl_tool.py`

- **技术点：** `subprocess.run([sys.executable, "-c", code])`、沙箱隔离、15 秒超时
- **本文件在项目中的角色：** 让 Agent 执行短 Python 代码片段（用于数据处理、天气查询等），通过独立子进程隔离主进程
- **学习任务：**
  1. 让 Agent 执行 `import sys; print(sys.version)` 验证子进程环境
  2. 测试：传入死循环代码，观察 15 秒后是否自动中止
  3. 思考：子进程能访问 `backend/` 目录吗？（`cwd=self._root_dir`）
- **验收：** 能解释"为什么用子进程而不是直接 `eval()`（隔离 + 超时 + 输出捕获）"

---

### 文件 25：`backend/tools/fetch_url_tool.py`

- **技术点：** `httpx` 异步 HTTP 客户端、`html2text` HTML 转 Markdown、内容类型判断
- **本文件在项目中的角色：** 让 Agent 抓取外部 URL，自动识别 JSON/HTML 并转换成可读文本，结果截断至 5000 字符防止 token 爆炸
- **学习任务：**
  1. 让 Agent 抓取 `https://api.github.com/repos/tjj245/langchain-miniopenclaw`，观察 JSON 格式化输出
  2. 让 Agent 抓取一个 HTML 页面，观察 `html2text` 输出的可读性
  3. 理解 `_run` 用同步 `httpx.Client`，`_arun` 用异步 `httpx.AsyncClient` 的原因
- **验收：** 能解释"为何结果截断到 5000 字符（上下文长度与成本的 trade-off）"

---

### 文件 26：`backend/tools/search_knowledge_tool.py`

- **技术点：** LlamaIndex 向量检索、关键词 fallback 搜索、增量指纹缓存、混合检索结果合并
- **本文件在项目中的角色：** 对 `knowledge/` 目录下的所有文件（txt/md/json/pdf 可读部分）建立向量索引，并实现语义检索 + 关键词 BM25-like fallback 的混合检索，结果按 score 合并排序
- **关键流程：**
  ```
  search(query, top_k) →
    ① fingerprint 变化? → rebuild() 重建索引
    ② 向量检索 (similarity_top_k) → 写入 combined dict
    ③ 关键词检索 (_keyword_search) → 分数累加
    ④ 按 score 排序 → 返回 top_k 条
  ```
- **学习任务：**
  1. 向 `knowledge/` 任意子目录里添加一个 `.md` 文件，发一条检索相关的消息，验证新文件是否被检索到
  2. 把 `top_k` 从 3 改为 6，对比回答引用的来源数量变化
  3. 理解 `_fingerprint()` 为何基于 `mtime + size` 而不是文件内容 hash（性能考虑）
- **验收：** 能解释"向量检索和关键词检索的分数是如何合并的，这样做的好处是什么"

---

### 文件 27：`backend/api/chat.py`

- **技术点：** FastAPI SSE（`StreamingResponse`）、Pydantic 请求校验、事件分段聚合、会话持久化
- **本文件在项目中的角色：** 后端最核心的 API 入口。接受 `POST /api/chat`，驱动 `agent_manager.astream()`，把事件流（token / tool_start / tool_end / done）转成标准 SSE 格式推送给前端，同时在 `done` 事件后把消息落盘到 session JSON
- **SSE 事件格式：**
  ```
  event: token\ndata: {"content": "..."}\n\n
  event: tool_start\ndata: {"tool": "...", "input": "..."}\n\n
  event: tool_end\ndata: {"tool": "...", "output": "..."}\n\n
  event: done\ndata: {"content": "..."}\n\n
  ```
- **学习任务：**
  1. 用 `curl -N -X POST http://localhost:8002/api/chat -H "Content-Type: application/json" -d '{"message":"hello","session_id":"test","stream":true}'` 直接观察原始 SSE 流
  2. 找到"首条消息自动生成标题"的逻辑，理解 `is_first_user_message` 的判断方式
  3. 理解 `segments` 列表的作用：为何需要把多段 assistant 消息分开存储（工具调用后会有新的 assistant 段）
- **验收：** 能画出"一次带工具调用的请求从 API 到 SSE 推送的完整事件序列"

---

### 文件 28：`backend/api/sessions.py`

- **技术点：** FastAPI 路由、Pydantic 模型、REST CRUD
- **本文件在项目中的角色：** 提供会话管理 API：创建/列出/重命名/删除会话，获取会话消息和系统提示词，自动生成标题
- **端点列表：**
  - `GET /api/sessions` — 会话列表（按更新时间降序）
  - `POST /api/sessions` — 新建会话
  - `PUT /api/sessions/{id}` — 重命名
  - `DELETE /api/sessions/{id}` — 删除
  - `GET /api/sessions/{id}/messages` — 消息 + 系统提示词
  - `GET /api/sessions/{id}/history` — 完整历史记录
  - `POST /api/sessions/{id}/generate-title` — AI 生成标题
- **学习任务：**
  1. 用 Swagger (`/docs`) 调用 `GET /api/sessions/{id}/messages`，查看当前会话的完整系统提示词
  2. 调用 `POST /api/sessions` 创建会话，再 `DELETE` 删除，验证会话文件创建和删除
- **验收：** 能解释"为什么同时有 `/messages` 和 `/history` 两个接口（一个给 Agent 用，一个给前端展示用）"

---

### 文件 29：`backend/api/files.py`

- **技术点：** 路径白名单、路径遍历防护、写后触发副作用（重建索引/刷新快照）
- **本文件在项目中的角色：** 提供文件读写 API，限制在白名单路径（`workspace/`、`memory/`、`skills/`、`knowledge/`、`SKILLS_SNAPSHOT.md`）；保存 `memory/MEMORY.md` 后自动重建向量索引，保存技能文件后自动刷新快照
- **关键白名单逻辑：**
  ```python
  ALLOWED_PREFIXES = ("workspace/", "memory/", "skills/", "knowledge/")
  ALLOWED_ROOT_FILES = {"SKILLS_SNAPSHOT.md"}
  ```
- **学习任务：**
  1. 尝试 `GET /api/files?path=config.py`，验证返回 400 拒绝
  2. 尝试 `GET /api/files?path=../../../etc/passwd`，验证路径遍历防护
  3. 通过 `POST /api/files` 保存 `memory/MEMORY.md`，查看后端日志确认 `memory_indexer.rebuild_index()` 被调用
- **验收：** 能说明"为什么 `sessions/` 不在白名单里（会话文件通过专门的 sessions API 管理）"

---

### 文件 30：`backend/api/tokens.py`

- **技术点：** `tiktoken`、token 统计、上下文预算意识
- **本文件在项目中的角色：** 提供 token 计数 API，帮助用户了解当前会话的上下文消耗（system_tokens + message_tokens），是触发"压缩"操作的依据之一
- **学习任务：**
  1. 调用 `GET /api/tokens/session/{id}`，对比添加大量文件到 workspace 后的 system_tokens 变化
  2. 理解前端 navbar 显示的 token 数来自这个接口
  3. 思考：当 total_tokens 接近模型最大上下文时，应该怎么处理（触发压缩）
- **验收：** 能解释"system_tokens 主要来自哪些文件，如何降低它"

---

### 文件 31：`backend/api/compress.py`

- **技术点：** 异步 LLM 调用（摘要生成）、会话归档
- **本文件在项目中的角色：** 提供 `POST /api/sessions/{id}/compress` 接口，调用 `agent_manager.summarize_history()` 把前 50% 的消息压缩成摘要，归档原始消息到 `sessions/archive/`，保留摘要在 `compressed_context`
- **学习任务：**
  1. 积累 6 条以上消息后调用压缩，查看 `sessions/{id}.json` 中 `compressed_context` 的内容
  2. 查看 `sessions/archive/` 中归档的 JSON，理解为什么不直接删除而是归档
  3. 理解 `n_messages = max(4, len(messages) // 2)` 的策略（至少压缩 4 条，最多压缩一半）
- **验收：** 能解释"压缩后的 compressed_context 在哪里被拼入对话历史"（→ `session_manager.load_session_for_agent()`）

---

### 文件 32：`backend/api/config_api.py`

- **技术点：** FastAPI、`RuntimeConfigManager`、配置热切换
- **本文件在项目中的角色：** 提供 `GET/PUT /api/config/rag-mode` 接口，读写 `config.json` 中的 `rag_mode` 标志，前端 RAG 切换按钮直接调用此接口
- **学习任务：**
  1. 调用 `PUT /api/config/rag-mode` 切换 RAG 模式，再发一条消息，查看是否出现 `retrieval` 类型的 SSE 事件
  2. 查看 `config.json` 文件（后端目录），理解持久化方式
- **验收：** 能解释"RAG 模式开启后，系统提示词的 Memory 部分有何变化（动态注入 vs 静态拼接）"

---

### 文件 33：知识库目录结构（`backend/knowledge/` 及各 `data_structure.md`）

以下是仓库中真实存在的知识库文件（按子目录）：

| 路径 | 内容类型 | 学习价值 |
|---|---|---|
| `backend/knowledge/data_structure.md` | 根目录索引 | 说明三个知识库领域的用途 |
| `backend/knowledge/AI Knowledge/data_structure.md` | 子目录索引 | AI 报告类文件说明 |
| `backend/knowledge/AI Knowledge/*.pdf` | AI 行业报告 (9 份) | 演示 PDF 知识库检索 |
| `backend/knowledge/E-commerce Data/data_structure.md` | 子目录索引 | 电商数据说明 |
| `backend/knowledge/E-commerce Data/customers.xlsx` | Excel | 演示 Excel 检索 |
| `backend/knowledge/E-commerce Data/employees.xlsx` | Excel | 演示员工数据检索 |
| `backend/knowledge/E-commerce Data/inventory.xlsx` | Excel | 演示库存检索 |
| `backend/knowledge/E-commerce Data/sales_orders.xlsx` | Excel | 演示销售数据检索 |
| `backend/knowledge/E-commerce Data/faq.json` | JSON 问答对 | 演示 FAQ 检索 |
| `backend/knowledge/Financial Report Data/data_structure.md` | 子目录索引 | 财报文件说明 |
| `backend/knowledge/Financial Report Data/*.pdf` | 财务报告 (3 份) | 演示财报分析 |
| `backend/knowledge/Financial Report Data/*.txt` | 财报文本版 (2 份) | 供文本检索使用 |
| `backend/knowledge/Safety Knowledge/data_structure.md` | 子目录索引 | 安全知识说明 |
| `backend/knowledge/Safety Knowledge/CSRF.txt` | 安全知识 | 演示安全文档检索 |
| `backend/knowledge/Safety Knowledge/XSS.md` | 安全知识 | 演示 Markdown 检索 |
| `backend/knowledge/Safety Knowledge/cors.md` | 安全知识 | 演示 CORS 知识检索 |
| `backend/knowledge/README.md` | 知识库使用说明 | 解释知识库整体结构 |

- **学习任务：**
  1. 先读 `backend/knowledge/data_structure.md`，理解"根目录索引 → 子目录索引 → 业务文件"的三层结构
  2. 向 Agent 提问"三一重工前三大股东是谁"，观察 `search_knowledge_base` 工具如何使用 PDF 文件
  3. 向 Agent 提问"哪些商品库存不足"，观察如何处理 Excel 文件
  4. 自己向 `E-commerce Data/` 添加一个 `.md` 文件，验证被自动纳入检索
- **验收：** 能解释"`data_structure.md` 的作用，以及为什么知识库检索技能要求先读它"

---

### 文件 34：`backend/skills/rag-skill/references/` 下的参考文档

| 文件 | 内容 |
|---|---|
| `backend/skills/rag-skill/references/pdf_reading.md` | 如何用 pdftotext / pdfplumber 提取 PDF 文本 |
| `backend/skills/rag-skill/references/excel_reading.md` | 如何用 pandas 读取 Excel 工作表 |
| `backend/skills/rag-skill/references/excel_analysis.md` | 如何用 pandas 进行数据过滤和聚合 |

- **技术点：** "学习后执行"设计模式——Agent 被要求在处理 PDF/Excel 前先读对应参考文档
- **学习任务：**
  1. 阅读这三个文件，理解每个文件的内容和格式
  2. 向 Agent 发一个 Excel 分析请求，观察它是否在 tool_calls 中出现了 `read_file` 读 `excel_reading.md` 的步骤
- **验收：** 能解释"为什么不直接把 pdf 处理方法写进 SKILL.md，而是单独放在 references/ 里"（关注点分离、按需加载减少 token 消耗）

---

## 第 3 阶段：前端

### 文件 35：`frontend/package.json`

- **技术点：** npm 包管理、Next.js 版本约束
- **本文件在项目中的角色：** 定义前端所有依赖（框架 + 组件库 + 编辑器）及构建脚本
- **关键依赖解析：**
  - `next@14.2.x` + `react@18.3.x` — 框架核心（App Router 模式）
  - `@monaco-editor/react@^4.7.0` — Monaco 内嵌编辑器（VS Code 同款）
  - `react-markdown` + `remark-gfm` — AI 回复的 Markdown + 表格渲染
  - `lucide-react` — 图标库（Plus / Database / Save 等）
  - `tailwindcss` + `tailwind-merge` — 原子化样式 + 动态类名合并
- **学习任务：**
  1. 找到哪个包实现了 Monaco Editor，访问官方 Demo 页了解其 API
  2. 理解 `tailwind-merge` 解决什么问题（条件类名合并时的冲突）
- **验收：** 能列出 `devDependencies` 里的工具链，并解释每个的作用

---

### 文件 36：`frontend/tsconfig.json`

- **技术点：** TypeScript 路径别名、严格模式
- **本文件在项目中的角色：** 配置 TypeScript 编译选项，关键设置 `"@/*": ["./src/*"]` 别名（代码中 `import { ... } from "@/lib/api"` 的来源）
- **学习任务：**
  1. 理解 `paths` 中的 `@/*` 别名如何映射到 `./src/*`
  2. 确认 `"strict": true` 已启用，理解严格模式对代码质量的影响
- **验收：** 能解释"为什么 `import from '@/lib/api'` 可以工作，而不需要写相对路径"

---

### 文件 37：`frontend/tailwind.config.ts`

- **技术点：** Tailwind CSS 自定义颜色、content 扫描范围
- **本文件在项目中的角色：** 扩展默认主题（如 `ocean` 颜色），配置 Tailwind 扫描 `src/**/*.{ts,tsx}` 以生成原子类
- **学习任务：**
  1. 在 Tailwind 配置中找到自定义颜色变量，与 `globals.css` 中的 CSS 变量对照
- **验收：** 能说清楚"Tailwind 的 content 配置漏掉某个目录会有什么后果（样式丢失）"

---

### 文件 38：`frontend/src/app/layout.tsx`

- **技术点：** Next.js App Router Root Layout、`next/font`（Google Fonts）、全局 Metadata
- **本文件在项目中的角色：** 根布局文件，加载 `Space Grotesk`（展示字体）和 `IBM Plex Mono`（等宽字体）并注入 CSS 变量，设置页面 title/description
- **学习任务：**
  1. 把 `title` 改为自己的名字，保存后刷新页面验证浏览器标签变化
  2. 理解 `variable: "--font-display"` 如何与 CSS `var(--font-display)` 对应
- **验收：** 能解释"App Router 中 layout.tsx 与 page.tsx 的职责边界"

---

### 文件 39：`frontend/src/app/globals.css`

- **技术点：** Tailwind 指令（`@tailwind`）、CSS 自定义属性（变量）、全局样式
- **本文件在项目中的角色：** 定义全局颜色变量（`--color-ink`、`--color-ocean` 等）和工具类（`.panel`、`.markdown`、`.mono`），供全局组件引用
- **学习任务：**
  1. 修改 `--color-ocean` 的颜色值，查看前端所有蓝绿色按钮是否同步变化
  2. 找到 `.panel` 类的定义，理解为什么项目里的卡片样式统一由它控制
- **验收：** 能解释"Tailwind 基础类和 CSS 变量分别负责什么，两者是如何配合的"

---

### 文件 40：`frontend/src/app/page.tsx`

- **技术点：** Next.js 页面组件、React Context Provider 包装、三栏弹性布局
- **本文件在项目中的角色：** 最终渲染的页面，用 `AppProvider` 包裹整个 `Workspace`，组装三栏布局：`Sidebar`（左）+ `ChatPanel`（中）+ `InspectorPanel`（右），`ResizeHandle` 实现拖拽调整宽度
- **学习任务：**
  1. 在 `Workspace` 函数里加一个 `console.log("rendering Workspace")`，打开浏览器 DevTools 观察渲染时机
  2. 理解 `AppProvider` 的位置为何在 `page.tsx` 而不是 `layout.tsx`（单页面 Context，避免跨页面污染）
- **验收：** 能画出三栏组件的嵌套结构图

---

### 文件 41：`frontend/src/lib/api.ts`

- **技术点：** TypeScript 类型定义、`fetch` API、SSE 流解析、模块化 API 封装
- **本文件在项目中的角色：** 前端唯一的后端通信层，定义所有数据类型（`SessionSummary`、`ToolCall`、`RetrievalResult` 等）和 API 函数，`streamChat()` 实现完整的 SSE 流解析逻辑
- **SSE 解析核心：**
  ```typescript
  // 按 \n\n 分割 SSE 块，解析 event: 和 data: 行
  const flushBlock = (block: string) => { /* 解析 event + data */ }
  while (true) { /* 读取 ReadableStream */ }
  ```
- **学习任务：**
  1. 阅读 `streamChat()` 的完整实现，理解 SSE 文本格式如何被解析成事件对象
  2. 理解 `getApiBase()` 的动态主机逻辑（为什么同一段代码 SSR 和 CSR 用不同 base URL）
  3. 尝试在 `request()` 函数里添加请求计时日志
- **验收：** 能手写一个最小版本的 SSE 客户端（只处理 `token` 和 `done` 事件）

---

### 文件 42：`frontend/src/lib/store.tsx`

- **技术点：** React Context API、`useState`/`useEffect`/`useMemo`、乐观更新（Optimistic UI）、异步状态管理
- **本文件在项目中的角色：** 全局状态中枢，`AppProvider` 持有所有状态（会话列表、消息流、RAG 模式、Inspector 内容等），通过 Context 向下传递；`sendMessage()` 实现了复杂的流式消息更新逻辑（在消息列表中原地 patch）
- **关键模式：**
  ```typescript
  // 乐观更新：先插入空 assistant 消息，再逐步 patch 内容
  const patchAssistant = (updater) => {
    setMessages(prev => prev.map(msg => msg.id === activeAssistantId ? updater(msg) : msg))
  }
  ```
- **学习任务：**
  1. 在 `sendMessage()` 里找到处理 `tool_start` 事件的代码，理解工具调用列表如何追加
  2. 理解 `activeAssistantId` 在 `new_response` 事件时如何切换（支持多段 assistant 消息）
  3. 在 `useEffect` 初始化逻辑里理解"应用启动后自动加载会话列表 + RAG 状态 + 技能列表 + Memory 文件"的流程
- **验收：** 能解释"`editableFiles` 为什么用 `useMemo` 计算，而不是直接写死（技能列表动态变化）"

---

### 文件 43：`frontend/src/components/layout/Navbar.tsx`

- **技术点：** React 组件、`useAppStore`、条件渲染样式
- **本文件在项目中的角色：** 顶部导航栏，提供：当前会话标题（可重命名）、新建会话按钮、RAG 模式切换按钮（视觉状态跟随 `ragMode`）、压缩按钮
- **学习任务：**
  1. 找到 RAG 切换按钮的样式代码，理解如何根据 `ragMode` 状态动态切换 Tailwind 类名
  2. 在 `Rename` 按钮的 `onClick` 里，理解为何用 `window.prompt`（轻量级交互，无需 Modal 组件）
- **验收：** 能把 `window.prompt` 替换成一个内联 input，实现同样的重命名功能

---

### 文件 44：`frontend/src/components/layout/Sidebar.tsx`

- **技术点：** 列表渲染、条件样式（当前会话高亮）、图标组件（`lucide-react`）
- **本文件在项目中的角色：** 左栏：上半部分是会话列表（高亮当前会话、支持点击切换和删除）；下半部分是 Raw Messages 区域（展示当前会话的消息摘要和工具调用数量）
- **学习任务：**
  1. 理解"会话列表"和"Raw Messages"如何用同一个 `messages` 状态渲染出不同 UI
  2. 找到点击会话时调用 `selectSession(session.id)` 的代码，追踪到 `store.tsx` 中 `selectSession` 的实现
- **验收：** 能解释"为什么 Raw Messages 展示的是 `messages` 而不是从后端实时拉取（本地状态驱动）"

---

### 文件 45：`frontend/src/components/layout/ResizeHandle.tsx`

- **技术点：** `onMouseDown` / `onMouseMove` / `onMouseUp` 拖拽事件、`useCallback`
- **本文件在项目中的角色：** 三栏之间的可拖拽分隔条，用户拖动时通过 `onResize(delta)` 回调通知父组件更新宽度
- **学习任务：**
  1. 阅读鼠标事件处理逻辑，理解 delta（拖动距离）是如何计算的
  2. 测试拖拽把左栏宽度拉到最小值，观察 `Math.max(260, ...)` 限制是否生效（在 `page.tsx` 中）
- **验收：** 能说清楚"mousemove 事件为何注册在 document 而不是 ResizeHandle 元素上（防止拖动过快时鼠标离开元素导致停止）"

---

### 文件 46：`frontend/src/components/chat/ChatPanel.tsx`

- **技术点：** `useRef`、`scrollIntoView`（自动滚动）、token 显示
- **本文件在项目中的角色：** 中栏聊天面板，渲染消息列表，消息追加时自动滚动到底部，顶部显示 token 统计
- **学习任务：**
  1. 找到 `endRef.current?.scrollIntoView({ behavior: "smooth" })` 的触发时机（`messages` 变化时）
  2. 理解空状态下渲染的欢迎界面，改成你自己的介绍语
- **验收：** 能解释"`messages.map()` 中 `key={message.id}` 使用 makeId() 生成的 id 而不是 index 的好处"

---

### 文件 47：`frontend/src/components/chat/ChatInput.tsx`

- **技术点：** 受控组件、`onKeyDown` 快捷键（Cmd/Ctrl + Enter 发送）、disabled 状态
- **本文件在项目中的角色：** 输入框组件，支持多行文本、Cmd/Ctrl+Enter 快捷键发送，流式输出中禁用发送按钮
- **学习任务：**
  1. 把快捷键从 `Cmd/Ctrl+Enter` 改为直接 `Enter`（去掉 `metaKey/ctrlKey` 判断），测试效果
  2. 理解 `onSend` 后 `setValue("")` 清空输入框的时机
- **验收：** 能解释"为什么 `isStreaming` 时禁用输入（防止并发请求）"

---

### 文件 48：`frontend/src/components/chat/ChatMessage.tsx`

- **技术点：** `react-markdown` + `remark-gfm`、条件渲染（用户/AI 消息不同样式）
- **本文件在项目中的角色：** 单条消息渲染组件，用户消息直接展示纯文本，AI 消息用 `ReactMarkdown` 渲染（支持表格、代码块等），并在 AI 消息上方附加 `RetrievalCard`（RAG 检索片段）和 `ThoughtChain`（工具调用链）
- **学习任务：**
  1. 让 AI 返回一个包含表格的 Markdown 回复，验证表格是否正确渲染（依赖 `remark-gfm`）
  2. 理解为什么用户消息用 `whitespace-pre-wrap` 而不是 ReactMarkdown（安全 + 简单）
- **验收：** 能解释"AI 消息加载中（`content === ""`）时显示什么文字以及原因"

---

### 文件 49：`frontend/src/components/chat/ThoughtChain.tsx`

- **技术点：** HTML `<details>/<summary>` 折叠组件、`ToolCall` 数据渲染
- **本文件在项目中的角色：** 在 AI 消息上方渲染可折叠的工具调用链，展示每次工具调用的名称、Input 参数（JSON）和 Output 结果
- **学习任务：**
  1. 让 Agent 调用天气查询工具，展开 ThoughtChain 查看 Input/Output 的原始内容
  2. 修改折叠提示文字"工具调用 N 次"为英文"N tool calls"，验证 UI 更新
- **验收：** 能解释"这个组件的可观测性价值：为什么调试 Agent 时需要能看到每次工具调用的输入输出"

---

### 文件 50：`frontend/src/components/chat/RetrievalCard.tsx`

- **技术点：** HTML `<details>/<summary>` 折叠、`RetrievalResult` 数据渲染、相关度 score 展示
- **本文件在项目中的角色：** RAG 模式下，在 AI 消息上方渲染可折叠的检索结果卡片，展示从 Memory 中召回的文本片段、来源文件路径和相关度分数
- **学习任务：**
  1. 开启 RAG 模式，向 Agent 提问 Memory 相关内容，展开 RetrievalCard 查看召回结果
  2. 对比不同问题的 score 值，理解分数高低与回答质量的关系
- **验收：** 能解释"为什么 score 显示 3 位小数（`.toFixed(3)`），score 的来源是余弦相似度"

---

### 文件 51：`frontend/src/components/editor/InspectorPanel.tsx`

- **技术点：** Monaco Editor（`@monaco-editor/react`）、文件选择器、保存状态（dirty flag）
- **本文件在项目中的角色：** 右栏在线文件编辑器，列出所有可编辑文件（固定文件 + 动态技能文件），用 Monaco Editor 展示文件内容，支持编辑和保存（保存后 dirty flag 清除）
- **学习任务：**
  1. 在 Inspector 中编辑 `workspace/SOUL.md`，不保存直接刷新，理解 `inspectorDirty` flag 的意义（提醒用户未保存）
  2. 保存后查看网络请求，确认 `POST /api/files` 被调用，payload 中包含完整文件路径和内容
  3. 添加一个新技能文件（`backend/skills/my-skill/SKILL.md`），刷新前端，验证它出现在 Inspector 文件列表中
- **验收：** 能解释"为什么 `editableFiles` 包含动态的技能路径（通过 `/api/skills` 获取），而不是硬编码"

---

## 学习验收总清单

完成所有文件学习后，你应该能做到以下所有事项：

### 项目理解
- [ ] 能用 3 分钟完整讲清楚"用户发一条消息到看到回复"的全链路（含 RAG、工具调用、SSE 推送、落盘）
- [ ] 能指出每个技术栈在项目中的具体文件落点
- [ ] 能解释"文件即记忆、技能即插件、Prompt 可解释"三个设计理念的代码实现位置

### 技术栈掌握
- [ ] 能解释 LangChain `create_agent` 和工具注册的工作原理（`tools/__init__.py` + `graph/agent.py`）
- [ ] 能解释 LlamaIndex 在项目中的两个作用（Memory 向量索引 + 知识库检索）
- [ ] 能解释 FastAPI SSE 推送的实现原理（`StreamingResponse` + 事件格式）
- [ ] 能解释 React Context + `sendMessage` 的乐观更新模式

### 动手实验
- [ ] 完成过至少 1 次参数改动实验（如 temperature、chunk_size、top_k）
- [ ] 创建过 1 个新技能文件并验证 Agent 能使用它
- [ ] 修改过 1 个 workspace 文件并验证 Prompt 即时生效
- [ ] 通过前端 Inspector 保存过文件并在下一轮对话中看到效果

### 可观测性
- [ ] 能通过前端 ThoughtChain 面板追踪工具调用的 Input/Output
- [ ] 能通过 `/api/sessions/{id}/messages` 查看完整系统提示词
- [ ] 能通过 token 统计了解上下文消耗，并知道如何通过压缩和 RAG 模式降低消耗
- [ ] 能直接打开 `sessions/*.json` 阅读会话记录和摘要

---

## 快速参考：文件–技术栈对照表

| 文件 | 核心技术 |
|---|---|
| `backend/config.py` | python-dotenv, dataclasses, lru_cache |
| `backend/app.py` | FastAPI, lifespan, CORS |
| `backend/graph/prompt_builder.py` | Prompt 工程, 多文件拼接 |
| `backend/graph/agent.py` | LangChain create_agent, ChatOpenAI, 异步流 |
| `backend/graph/memory_indexer.py` | LlamaIndex VectorStoreIndex, SentenceSplitter, OpenAIEmbedding |
| `backend/graph/session_manager.py` | JSON 持久化, UUID, 归档 |
| `backend/tools/skills_scanner.py` | PyYAML, pathlib, 自动生成 |
| `backend/tools/read_file_tool.py` | LangChain BaseTool, Pydantic, 路径安全 |
| `backend/tools/terminal_tool.py` | subprocess, 跨平台, 黑名单 |
| `backend/tools/python_repl_tool.py` | subprocess, 沙箱执行 |
| `backend/tools/fetch_url_tool.py` | httpx, html2text, 异步 HTTP |
| `backend/tools/search_knowledge_tool.py` | LlamaIndex, 混合检索, 增量缓存 |
| `backend/api/chat.py` | FastAPI SSE, StreamingResponse |
| `backend/api/tokens.py` | tiktoken |
| `frontend/src/lib/api.ts` | TypeScript, fetch, SSE 流解析 |
| `frontend/src/lib/store.tsx` | React Context, useState, 乐观更新 |
| `frontend/src/components/editor/InspectorPanel.tsx` | Monaco Editor |
| `frontend/src/components/chat/ChatMessage.tsx` | react-markdown, remark-gfm |
