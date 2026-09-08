# Modular RAG MCP Server

一个**模块化、可插拔、可观测**的检索增强生成（RAG, Retrieval-Augmented Generation）知识库服务，基于 [Model Context Protocol (MCP)](https://modelcontextprotocol.io) 标准构建。它既可以作为独立的 RAG 引擎运行，也可以作为 MCP Server 无缝接入 GitHub Copilot、Claude Desktop 等 AI 助手，让私有知识库「一次接入，处处可用」。

- **全链路可插拔**：LLM、Embedding、分块、向量库、重排、评估等每一个环节都定义了抽象接口，改配置即可切换后端，无需改代码。
- **混合检索 + 精排**：Dense（语义）+ Sparse（关键词）双路召回，经 RRF 融合，可选 Cross-Encoder / LLM 精排。
- **多模态**：通过 Vision LLM 为文档图片生成文字描述（Image-to-Text），复用纯文本检索链路实现「搜文出图」。
- **白盒可观测**：全链路 Trace 记录 + 本地 Streamlit 可视化面板（6 大页面）。
- **自动化评估**：集成 Ragas 与自定义指标，用量化分数驱动调优。

---

## 目录

- [特性](#特性)
- [系统架构](#系统架构)
- [目录结构](#目录结构)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [MCP 工具](#mcp-工具)
- [接入 AI 助手](#接入-ai-助手)
- [测试](#测试)
- [评估](#评估)
- [技术栈](#技术栈)

---

## 特性

### 智能 RAG 流水线

- **分块策略**：语义感知切分 + 上下文增强（标题、页码、图片描述注入）。
- **混合检索**：稀疏检索（BM25）负责专有名词精确匹配，稠密检索（Embedding）负责同义词与模糊语义，RRF 融合取长补短。
- **两段式精排**：「粗排（低成本泛召回）→ 精排（高成本精过滤）」，在保证速度的同时提升 Top 结果精准度。

### 全链路可插拔

| 组件 | 内置实现 | 切换方式 |
|------|---------|---------|
| LLM | OpenAI / Azure OpenAI / Ollama / DeepSeek | `settings.yaml` 中改 `provider` |
| Vision LLM | OpenAI / Azure OpenAI / Ollama | 同上 |
| Embedding | OpenAI / Azure OpenAI / Ollama | 同上 |
| 分块器 | LangChain `RecursiveCharacterTextSplitter` | `BaseSplitter` 接口 + 工厂 |
| 向量库 | Chroma（嵌入式） | `BaseVectorStore` 接口 + 工厂 |
| 重排器 | None / Cross-Encoder / LLM Rerank | `settings.yaml` 中改 `provider` |
| 评估器 | Ragas / 自定义指标（Hit Rate、MRR、Faithfulness） | `settings.yaml` 中改 `provider` |

### 多模态图片处理

采用 **Image-to-Text** 策略：摄取阶段用 Vision LLM 为文档中的图片生成文字描述，注入到对应 Chunk 的正文/元数据中，检索阶段即可通过自然语言命中图片，命中后以 Base64 编码返回图片内容。

### 白盒可观测与可视化

- **双链路 Trace**：完整记录 Ingestion（load → split → transform → embed → upsert）与 Query（query_processing → dense → sparse → fusion → rerank）两条链路各阶段耗时与中间状态。
- **本地 Dashboard**：基于 Streamlit，提供系统总览、数据浏览器、Ingestion 管理、Ingestion 追踪、Query 追踪、评估面板 6 大页面。

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│               MCP Clients (GitHub Copilot / Claude Desktop) │
└──────────────────────────────┬──────────────────────────────┘
                               │ JSON-RPC 2.0 (Stdio)
┌──────────────────────────────▼──────────────────────────────┐
│                     MCP Server 层 (接口层)                    │
│   query_knowledge_hub · list_collections · get_document_summary │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│                        Core 层 (核心业务)                     │
│   Query Engine (混合检索/融合/重排) · Response Builder · Trace │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│                      Ingestion 层 (摄取流水线)                │
│   Loader → Splitter → Transform → Embed → Upsert             │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│        可插拔组件库 (LLM / Embedding / Reranker / VectorStore)│
└─────────────────────────────────────────────────────────────┘
```

---

## 目录结构

```
RAG-SERVER/
├── main.py                      # 入口占位（实际 MCP Server 入口在 src/mcp_server/server.py）
├── config/
│   ├── settings.yaml            # 主配置文件
│   ├── prompts/                 # Prompt 模板（分块精炼/图片描述/元数据增强/重排）
│   └── test_credentials.yaml.example
├── scripts/
│   ├── ingest.py                # 数据摄取 CLI
│   ├── query.py                 # 查询 CLI
│   ├── start_dashboard.py       # 启动 Dashboard
│   └── evaluate.py              # 运行评估
├── src/
│   ├── core/                    # 核心业务（设置、类型、查询引擎、响应、追踪）
│   ├── ingestion/               # 摄取流水线（分块/增强/编码/存储/文档管理）
│   ├── libs/                    # 可插拔组件库（LLM/Embedding/Loader/Splitter/Reranker/VectorStore/Evaluator）
│   ├── mcp_server/              # MCP 协议处理与工具定义
│   └── observability/           # 日志、Dashboard、评估
└── tests/
    ├── unit/                    # 单元测试
    ├── integration/             # 集成测试
    ├── e2e/                     # 端到端测试
    └── fixtures/                # 测试数据与黄金测试集
```

---

## 快速开始

### 1. 环境要求

- Python >= 3.10
- 一个可用的 LLM / Embedding API Key（或本地 Ollama）

### 2. 安装

```bash
# 克隆并进入项目
cd RAG-SERVER

# 创建虚拟环境（推荐）
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Linux/macOS:
source .venv/bin/activate

# 安装依赖（含开发依赖）
pip install -e ".[dev]"
```

### 3. 配置

编辑 [`config/settings.yaml`](config/settings.yaml)，填入你的 LLM / Embedding 凭据：

```yaml
llm:
  provider: "openai"          # openai | azure | ollama | deepseek
  model: "gpt-4o"
  api_key: "YOUR_API_KEY_HERE"

embedding:
  provider: "openai"
  model: "text-embedding-ada-002"
  api_key: "YOUR_API_KEY_HERE"
```

> 使用本地 Ollama 时，将 `provider` 改为 `ollama` 并配置 `base_url`，无需 API Key。

### 4. 摄取文档

```bash
# 摄取单个 PDF
python scripts/ingest.py --path documents/report.pdf --collection my_docs

# 摄取整个目录
python scripts/ingest.py --path documents/ --collection my_docs

# 强制重新处理（忽略历史记录）
python scripts/ingest.py --path documents/ --collection my_docs --force

# 预览将处理哪些文件（不实际处理）
python scripts/ingest.py --path documents/ --dry-run
```

### 5. 查询知识库

```bash
# 基础查询
python scripts/query.py --query "Azure OpenAI 配置步骤" --collection my_docs

# 显示 Dense/Sparse/融合/重排各阶段结果
python scripts/query.py --query "RRF 是什么" --verbose

# 禁用重排
python scripts/query.py --query "RRF 是什么" --no-rerank
```

### 6. 启动可视化面板

```bash
python scripts/start_dashboard.py          # 默认 http://localhost:8501
python scripts/start_dashboard.py --port 8502
```

### 7. 启动 MCP Server

```bash
python -m src.mcp_server.server
```

Server 通过 Stdio 传输协议运行，`stdout` 仅输出 MCP JSON-RPC 消息，日志统一输出到 `stderr`。

---

## 配置说明

完整配置见 [`config/settings.yaml`](config/settings.yaml)，关键项如下：

| 配置段 | 说明 |
|--------|------|
| `llm` | 推理 LLM 的 provider、model、API Key 等 |
| `embedding` | Embedding 模型配置 |
| `vision_llm` | 图片描述（Captioning）用的多模态模型 |
| `vector_store` | 向量库（当前为 Chroma）及持久化目录 |
| `retrieval` | 召回参数：`dense_top_k` / `sparse_top_k` / `fusion_top_k` / `rrf_k` |
| `rerank` | 精排后端（`none` / `cross_encoder` / `llm`）及是否启用 |
| `evaluation` | 评估后端与指标 |
| `observability` | 日志级别、Trace 开关与文件路径 |
| `ingestion` | 分块大小、重叠、splitter 类型、批处理大小、LLM 增强开关 |

---

## MCP 工具

Server 通过 `tools/list` 对外暴露以下工具：

| 工具名 | 功能 | 主要参数 |
|--------|------|---------|
| `query_knowledge_hub` | 主检索入口，执行混合检索 + 重排，返回带引用的结构化结果 | `query`, `top_k?`, `collection?` |
| `list_collections` | 列出知识库中可用的文档集合 | 无 |
| `get_document_summary` | 获取指定文档的摘要与元信息 | `doc_id` |

检索结果携带完整引用信息（`source_file`、`page`、`chunk_id`、`score`），并支持在 `content` 数组中同时返回文本与 Base64 图片内容。

---

## 接入 AI 助手

### Claude Desktop

在 `claude_desktop_config.json` 中添加：

```json
{
  "mcpServers": {
    "modular-rag-mcp-server": {
      "command": "python",
      "args": ["-m", "src.mcp_server.server"],
      "cwd": "D:\\云计算\\暑假作业\\RAG-SERVER"
    }
  }
}
```

### VS Code / GitHub Copilot

在 VS Code 的 MCP 配置中同样指定启动命令与工作目录即可，重启后 Copilot 会自动发现并调用本 Server 提供的工具。

---

## 测试

项目采用 **测试驱动开发（TDD）**，按测试金字塔分层：

```bash
# 运行全部单元测试
pytest tests/unit

# 运行集成测试
pytest tests/integration

# 运行端到端测试
pytest tests/e2e

# 跳过需要真实 LLM API 的测试
pytest -m "not llm"

# 生成覆盖率报告
pytest --cov=src --cov-report=term-missing
```

自定义 pytest 标记：

| 标记 | 含义 |
|------|------|
| `unit` | 单元测试（快速、无外部依赖） |
| `integration` | 集成测试 |
| `e2e` | 端到端测试 |
| `llm` | 需要真实 LLM API 调用 |
| `slow` | 慢速测试 |

---

## 评估

基于黄金测试集对检索与生成质量进行量化评估：

```bash
# 使用默认测试集与自定义评估器
python scripts/evaluate.py

# 指定测试集与集合
python scripts/evaluate.py --test-set tests/fixtures/golden_test_set.json --collection my_docs

# 输出 JSON 格式报告
python scripts/evaluate.py --json
```

支持指标：`hit_rate`、`mrr`、`faithfulness` 等（可插拔扩展 Ragas）。

---

## 技术栈

| 类别 | 选型 |
|------|------|
| 语言 | Python >= 3.10 |
| MCP | 官方 `mcp` SDK（Stdio Transport） |
| 文档解析 | `markitdown[pdf]`（PDF → Markdown） |
| 分块 | `langchain-text-splitters`（RecursiveCharacterTextSplitter） |
| 向量库 | `chromadb` |
| 稀疏检索 | 自研 BM25 索引 |
| 可视化 | `streamlit` |
| 评估 | `ragas` + 自定义指标 |
| 中文分词 | `jieba` |
| 测试 | `pytest`、`pytest-cov`、`pytest-asyncio`、`pytest-mock` |
| 代码质量 | `ruff`、`mypy` |

---

## License

[MIT](LICENSE)
