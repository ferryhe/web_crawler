# QMD 研究报告：对 AI_actuarial_inforsearch 检索能力的改进分析

> 研究对象：[QMD (Query Markup Documents)](https://github.com/tobi/qmd) — 基于 BM25 + 向量语义搜索 + LLM 重排序的本地混合检索引擎

---

## 1. QMD 项目深度分析

### 1.1 核心定位

QMD 是一个**本地端混合搜索引擎**，专为 Markdown 文档、会议记录、知识库等文本内容设计。它将三种检索技术融合为一条流水线：

| 检索层 | 技术 | 特点 |
|--------|------|------|
| 全文检索 | BM25（SQLite FTS5） | 速度快，对精确关键词敏感 |
| 语义搜索 | 向量嵌入（余弦相似度） | 理解语义，不依赖精确词汇 |
| 智能重排序 | LLM（Qwen3-reranker GGUF） | 综合理解，按相关性重新打分 |

所有模型均在**本地运行**，使用 `node-llama-cpp` 加载 GGUF 量化模型，无需调用外部 API。

---

### 1.2 混合检索流水线（`query` 命令）

```
用户问题
    │
    ├─── 查询扩展（Query Expansion）
    │         LLM 生成 1-2 个近义/改写版本
    │         原始问题权重 ×2
    │
    ├─── 并行检索（每个查询变体）
    │         ├── BM25 全文检索
    │         └── 向量语义检索
    │
    ├─── RRF 融合（Reciprocal Rank Fusion）
    │         score = Σ 1/(k + rank + 1)，k=60
    │         前1名奖励 +0.05，前2-3名 +0.02
    │         保留 Top 30
    │
    └─── LLM 重排序（Yes/No + logprobs）
              ├── 排名 1-3：  75% RRF + 25% 重排序
              ├── 排名 4-10： 60% RRF + 40% 重排序
              └── 排名 11+：  40% RRF + 60% 重排序
```

### 1.3 三种检索模式对比

| 命令 | 模式 | 速度 | 适用场景 |
|------|------|------|----------|
| `qmd search` | 纯 BM25 | 极快（毫秒级） | 精确关键词检索 |
| `qmd vsearch` | 纯向量 | 快（几十毫秒） | 语义相似度检索 |
| `qmd query` | 混合+重排序 | 较慢（1-5 秒） | 最高质量检索，推荐用于问答 |

### 1.4 评分标准

| 分数范围 | 含义 |
|---------|------|
| 0.8 – 1.0 | 高度相关 |
| 0.5 – 0.8 | 较为相关 |
| 0.2 – 0.5 | 一定相关 |
| 0.0 – 0.2 | 低相关 |

### 1.5 本地模型配置

| 模型 | 用途 | 大小 |
|------|------|------|
| `embeddinggemma-300M-Q8_0` | 生成向量嵌入 | ~300MB |
| `qwen3-reranker-0.6b-q8_0` | 重排序打分 | ~640MB |
| `qmd-query-expansion-1.7B-q4_k_m` | 查询扩展（微调） | ~1.1GB |

---

## 2. AI_actuarial_inforsearch 问题检索现状分析

### 2.1 精算信息检索的典型挑战

精算行业信息检索存在以下特殊难点：

| 挑战 | 说明 |
|------|------|
| **专业术语多义** | 如"准备金"在不同语境（寿险/非寿险/IFRS17）含义不同 |
| **问法多样** | 用户可能用"责任准备金怎么算"或"IBNR 计算方法"问同一个问题 |
| **文档结构复杂** | 监管文件、考试大纲、精算准则等格式各异 |
| **中文分词问题** | 专业词汇如"非寿险精算"易被错误分词 |
| **跨文档推理** | 一个问题的答案可能散布在多份文档中 |

### 2.2 当前检索模式的局限

若 AI_actuarial_inforsearch 目前使用的是：
- **关键词检索**：无法处理同义词和语义相近的问法
- **纯向量检索**：可能丢失精确术语的精确匹配
- **单次 LLM 直接问答**：上下文窗口有限，召回率受原始输入限制

---

## 3. QMD 能带来的具体改进

### 3.1 改进点对比

| 场景 | 当前问题 | QMD 的改进方式 |
|------|----------|----------------|
| 用户用不同表述提问 | 关键词不匹配导致漏检 | **查询扩展**自动生成改写版本覆盖多种表述 |
| 精确术语查询 | 向量检索可能语义漂移 | **BM25** 保证精确术语命中，RRF 融合防止遗漏 |
| 模糊概念检索 | 关键词检索无结果 | **向量语义搜索**捕捉语义相似文档 |
| Top 结果排序质量低 | 检索分数不代表真实相关性 | **LLM 重排序**从语义角度重新评分 |
| 多文档知识库 | 不知道相关内容在哪 | **Collection + Context** 管理知识分类 |

### 3.2 精算问答的检索质量提升

**示例：用户问"非寿险 IBNR 怎么计算"**

| 步骤 | 处理过程 | 结果 |
|------|----------|------|
| 原始问题 | "非寿险 IBNR 怎么计算" | BM25 检索命中"IBNR"相关文档 |
| 查询扩展 | "已发生未报告未决赔款估算方法"、"Non-life IBNR estimation" | 覆盖中英文表述 |
| 向量检索 | 嵌入语义相似度 | 命中"链梯法"、"B-F法"等相关文档 |
| RRF 融合 | 合并排名 | 综合全文 + 语义的最佳候选 |
| LLM 重排序 | 语义相关性打分 | 最终精准排序 |

---

## 4. 集成 QMD 的技术方案

### 4.1 方案一：MCP Server 集成（推荐）

QMD 提供了完整的 MCP（Model Context Protocol）服务器，可直接作为 AI_actuarial_inforsearch 的检索后端：

```json
// Claude Desktop / AI Agent 配置
{
  "mcpServers": {
    "actuarial_knowledge": {
      "command": "qmd",
      "args": ["mcp"],
      "env": {
        "QMD_INDEX": "actuarial"
      }
    }
  }
}
```

**MCP 工具列表：**

| 工具 | 功能 | 使用场景 |
|------|------|----------|
| `qmd_search` | BM25 关键词搜索 | 精确术语查询 |
| `qmd_vector_search` | 向量语义搜索 | 模糊概念检索 |
| `qmd_deep_search` | 混合搜索+重排序 | 完整问答流程（推荐） |
| `qmd_get` | 按路径/ID 获取文档 | 检索到结果后获取全文 |
| `qmd_multi_get` | 批量获取多个文档 | 多文档聚合回答 |

### 4.2 方案二：HTTP API 集成（共享服务）

启动 QMD HTTP 服务，多个客户端共享同一个常驻模型进程（避免每次重复加载模型）：

```bash
# 启动 QMD HTTP 服务（后台常驻）
qmd mcp --http --port 8181 --daemon

# 检查服务状态
qmd status
# 输出: MCP: running (PID 12345)
```

```python
# ai_actuarial/search/qmd_client.py
"""
通过 HTTP MCP 调用 QMD 混合检索服务。
"""
import httpx
import json
from typing import Optional


QMD_MCP_URL = "http://localhost:8181/mcp"


def deep_search(
    query: str,
    collection: Optional[str] = None,
    top_n: int = 5,
    min_score: float = 0.3,
) -> list[dict]:
    """
    使用 QMD 混合检索 + LLM 重排序查找最相关的精算文档片段。

    Args:
        query: 用户问题（支持中文自然语言）
        collection: 限定检索范围（如 "regulations"、"exam_materials"）
        top_n: 返回结果数量
        min_score: 最低相关分数阈值（0.0-1.0）

    Returns:
        按相关性降序排列的文档片段列表
    """
    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": "qmd_deep_search",
            "arguments": {
                "query": query,
                "n": top_n,
                **({"collection": collection} if collection else {}),
            },
        },
    }

    response = httpx.post(QMD_MCP_URL, json=payload, timeout=30)
    result = response.json()

    # 解析 MCP 返回格式
    content = result.get("result", {}).get("content", [])
    items = json.loads(content[0]["text"]) if content else []

    # 过滤低分结果
    return [item for item in items if item.get("score", 0) >= min_score]


def keyword_search(
    query: str,
    collection: Optional[str] = None,
    top_n: int = 10,
) -> list[dict]:
    """
    使用 BM25 关键词检索（速度快，适合精确术语）。
    """
    payload = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "tools/call",
        "params": {
            "name": "qmd_search",
            "arguments": {
                "query": query,
                "n": top_n,
                **({"collection": collection} if collection else {}),
            },
        },
    }

    response = httpx.post(QMD_MCP_URL, json=payload, timeout=10)
    result = response.json()
    content = result.get("result", {}).get("content", [])
    return json.loads(content[0]["text"]) if content else []
```

### 4.3 方案三：命令行集成（轻量原型）

在不改动现有架构的前提下，通过调用 `qmd` CLI 来增强检索：

```python
# ai_actuarial/search/qmd_cli_search.py
"""
通过子进程调用 qmd CLI，快速集成混合检索（原型阶段用）。
"""
import subprocess
import json
from typing import Optional


def qmd_query(
    question: str,
    collection: Optional[str] = None,
    top_n: int = 5,
) -> list[dict]:
    """
    调用 qmd query 命令执行混合检索+重排序。

    Returns:
        JSON 格式的检索结果列表，每项包含 path, score, snippet, context
    """
    cmd = ["qmd", "query", question, "--json", f"-n", str(top_n)]
    if collection:
        cmd += ["-c", collection]

    result = subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        timeout=60,
    )

    if result.returncode != 0:
        raise RuntimeError(f"qmd query failed: {result.stderr}")

    return json.loads(result.stdout)
```

### 4.4 知识库建立流程

将精算信息索引到 QMD 的步骤：

```bash
# 1. 安装 QMD
npm install -g @tobilu/qmd

# 2. 建立精算知识库 Collection
qmd collection add ./data/regulations     --name regulations
qmd collection add ./data/exam_materials  --name exam_materials
qmd collection add ./data/actuarial_standards --name standards

# 3. 添加语义上下文（帮助 LLM 理解每个 Collection 的内容）
qmd context add qmd://regulations      "保险监管政策、银保监会文件、精算规定"
qmd context add qmd://exam_materials   "精算师考试大纲、历年真题、学习材料"
qmd context add qmd://standards        "中国精算准则、IFRS17、IASB 文件"

# 4. 生成向量嵌入（首次需要下载模型 ~2GB，之后增量更新）
qmd embed

# 5. 验证索引状态
qmd status
```

---

## 5. 与现有 ScrapeGraphAI 方案的协同

QMD 与 ScrapeGraphAI 在 AI_actuarial_inforsearch 中承担**不同角色**，可以形成互补：

```
数据流向：

┌──────────────────────────────────────────────────────────────┐
│                   AI_actuarial_inforsearch                   │
│                                                              │
│  数据采集层（ScrapeGraphAI）                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  SmartScraperGraph → 抓取监管文件、考试通知            │   │
│  │  SearchGraph       → 发现最新精算政策                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                           ↓ 存储为 Markdown                  │
│  知识检索层（QMD）                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  qmd embed   → 对抓取内容生成向量嵌入                  │   │
│  │  qmd query   → BM25 + 向量 + 重排序回答用户问题         │   │
│  │  MCP Server  → 为 AI Agent 提供结构化检索接口           │   │
│  └──────────────────────────────────────────────────────┘   │
│                           ↓ 检索结果                         │
│  问答生成层（LLM）                                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  通义千问 / Qwen → 基于检索结果生成精准回答             │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

| 层次 | 工具 | 职责 |
|------|------|------|
| 数据采集 | ScrapeGraphAI | 从网页抓取、结构化精算信息 |
| 知识存储 | Markdown 文件 | 统一存储格式（QMD 的原生格式） |
| 知识检索 | QMD | 混合检索 + 重排序，返回最相关片段 |
| 问答生成 | 通义千问/Qwen | 基于检索结果生成回答（RAG 模式） |

---

## 6. 实施建议与优先级

### 6.1 优先级规划

| 优先级 | 任务 | 工作量 | 预期收益 |
|--------|------|--------|----------|
| **P0** | 安装 QMD，建立精算知识库 Collection | 半天 | 立即可用的混合检索 |
| **P0** | 用 `qmd embed` 对现有文档建立向量索引 | 1小时 | 语义搜索能力 |
| **P1** | 集成 QMD MCP Server 到现有 AI Agent 流程 | 半天 | 系统化检索增强 |
| **P1** | 为不同文档类型设置 Collection + Context | 1小时 | 按类别精准检索 |
| **P2** | 替换/增强现有关键词检索为 `qmd query` | 1天 | 完整混合检索体验 |
| **P2** | 使用 HTTP 模式常驻 MCP 服务（避免模型重复加载） | 2小时 | 生产环境性能优化 |

### 6.2 快速验证（最小化原型）

用最少的步骤验证 QMD 对精算问答的提升效果：

```bash
# 步骤1：安装 QMD
npm install -g @tobilu/qmd

# 步骤2：将几份精算文档放入目录
mkdir ~/actuarial_test && cp *.md ~/actuarial_test/

# 步骤3：建立索引
qmd collection add ~/actuarial_test --name test
qmd embed

# 步骤4：对比检索质量
# 方式A：关键词检索
qmd search "IBNR 计算方法"

# 方式B：混合检索（质量最好）
qmd query "非寿险未决赔款准备金如何估算"

# 步骤5：输出 JSON 供后续处理
qmd query "精算假设如何确定" --json -n 5
```

---

## 7. 综合评估

### 7.1 QMD 对 AI_actuarial_inforsearch 的适配度

| 评估维度 | 评分 | 说明 |
|----------|------|------|
| **精确术语检索** | ⭐⭐⭐⭐⭐ | BM25 对"IBNR"、"DCF"、"偿付能力"等专业词精确命中 |
| **语义相似检索** | ⭐⭐⭐⭐⭐ | 向量搜索处理中文自然语言提问 |
| **多文档知识库** | ⭐⭐⭐⭐⭐ | Collection + Context 管理监管文件、考试材料等多类型文档 |
| **本地化/隐私** | ⭐⭐⭐⭐⭐ | 全部本地运行，精算数据不泄露给外部 API |
| **中文支持** | ⭐⭐⭐⭐ | 使用 Qwen 系列模型，中文效果较好 |
| **接入成本** | ⭐⭐⭐⭐ | Node.js 依赖，Python 项目通过 CLI 或 HTTP 调用较简单 |
| **模型体积** | ⭐⭐⭐ | 首次下载约 2GB GGUF 模型 |

### 7.2 结论

**QMD 能显著提升 AI_actuarial_inforsearch 的问题检索质量**，尤其在以下场景中：

1. **用户用口语提问精算术语**：查询扩展 + 向量搜索弥合口语与专业术语的差距
2. **跨多个文档类型检索**：Collection 管理使监管文件、准则、考题各归其位
3. **减少 LLM 幻觉**：精准检索后的 RAG 模式优于直接让 LLM 凭记忆回答
4. **本地敏感数据**：精算内部数据可在不联网的环境下完成检索

建议将 QMD 作为 AI_actuarial_inforsearch 的**知识检索核心组件**，与 ScrapeGraphAI 的数据采集能力形成完整的"爬取 → 索引 → 检索 → 问答"流水线。

---

*报告日期：2026-02-25*
