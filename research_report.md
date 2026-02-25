# 学习报告：AI 爬虫工具研究与选型建议

> 研究对象：[ScrapeGraphAI](https://github.com/ScrapeGraphAI/Scrapegraph-ai) 与 [Shubhamsaboo / web_scrapping_ai_agent](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/web_scrapping_ai_agent)

---

## 1. ScrapeGraphAI 深度分析

### 1.1 核心概念

ScrapeGraphAI 是一个 Python 爬虫库，核心思路是**用 LLM 替代手写的 CSS 选择器 / XPath**。开发者只需用自然语言描述"要提取什么"，库会自动完成：

1. 抓取页面（Playwright/requests）
2. 将 HTML/Markdown 喂给 LLM
3. LLM 按指定 schema 返回结构化 JSON

### 1.2 主要管道（Pipeline）类型

| 管道 | 说明 | 适用场景 |
|------|------|----------|
| `SmartScraperGraph` | 单页面，给一个 URL + prompt，返回 JSON | 最常用，结构已知或未知均可 |
| `SearchGraph` | 自动搜索引擎 → 爬取 top-N 结果 | 需要批量搜索并汇总 |
| `SmartScraperMultiGraph` | 并发抓取多个 URL，单一 prompt | 同类页面批量处理 |
| `ScriptCreatorGraph` | 生成可复用的爬虫 Python 脚本 | 源结构固定、需要提速时 |
| `SpeechGraph` | 抓取 + 生成音频摘要 | 特殊内容输出 |

### 1.3 支持的 LLM 后端

| 类别 | 示例 |
|------|------|
| 商用 API | OpenAI (gpt-4o), Azure OpenAI, Groq, Gemini, Anthropic |
| 国产 API | 通义千问 (Qwen) — 兼容 OpenAI API 格式 |
| 本地模型 | Ollama (llama3.2, mistral 等) — 零成本开发调试 |

### 1.4 最简使用示例

```python
from scrapegraphai.graphs import SmartScraperGraph

graph_config = {
    "llm": {
        "api_key": "YOUR_OPENAI_KEY",
        "model": "openai/gpt-4o-mini",
    },
}

result = SmartScraperGraph(
    prompt="提取赛事名称、城市、比赛日期和报名链接",
    source="https://example-marathon-site.com/race/2026",
    config=graph_config,
).run()

# result 是结构化 dict，例如：
# {"name": "北京马拉松2026", "city": "北京", "date": "2026-10-18", "registration_url": "..."}
```

### 1.5 与传统爬虫方式对比

| 维度 | 传统（CSS选择器 / Cheerio / BeautifulSoup） | ScrapeGraphAI |
|------|---------------------------------------------|---------------|
| 开发速度 | 慢（需逐站点分析 DOM） | 快（自然语言 prompt 即可） |
| 维护成本 | 高（页面改版即失效） | 低（LLM 能适应布局变化） |
| 运行成本 | 极低 | 有 LLM API 费用（gpt-4o-mini 成本已很低） |
| 结果稳定性 | 高（规则确定） | 中（LLM 偶有幻觉或格式错误） |
| 动态页面 | 需配合 Puppeteer/Playwright | 内置 Playwright 支持 |
| 规模化速度 | 快（无 LLM 延迟） | 较慢（每页均需 LLM 调用） |
| 适合场景 | 结构固定、高频、大批量 | 结构多变、低频、探索性 |

---

## 2. Shubhamsaboo / web_scrapping_ai_agent 分析

### 2.1 项目定位

该项目是 ScrapeGraphAI 的一个 **Streamlit UI 包装器**，本质是演示如何快速搭建一个可视化 AI 爬虫工具，面向非开发者或快速验证场景。

### 2.2 两个版本

| 文件 | 说明 |
|------|------|
| `ai_scrapper.py` | 使用 OpenAI API（gpt-4o / gpt-5），需输入 API key |
| `local_ai_scrapper.py` | 使用本地 Ollama（llama3.2 + nomic-embed-text），零费用 |

### 2.3 核心代码逻辑

```python
# ai_scrapper.py 核心流程（简化）
graph_config = {"llm": {"api_key": key, "model": "gpt-4o"}}
smart_scraper_graph = SmartScraperGraph(
    prompt=user_prompt,
    source=url,
    config=graph_config
)
result = smart_scraper_graph.run()
```

### 2.4 价值评估

- **直接价值**：快速验证"LLM 能否从某页面提取目标数据"，适合原型演示
- **局限性**：无批量处理、无持久化、无错误重试、无定时调度；不适合生产

---

## 3. 核心问题解答：是直接调用 API 还是自己编程？

### 3.1 决策框架

```
问题：该爬虫场景用 ScrapeGraphAI 还是自定义开发？

1. 数据源结构是否固定且稳定？
   ├── 是 → 优先自定义 CSS 选择器规则（快、稳、省钱）
   └── 否（结构多变 / 未知） → 继续判断↓

2. 页面数量和爬取频率？
   ├── 少量 / 低频（<1000页/天） → ScrapeGraphAI 成本可接受
   └── 大批量 / 高频 → 自定义 + LLM 仅用于兜底

3. 是否需要 JavaScript 渲染？
   ├── 否 → ScrapeGraphAI (requests 模式) / 自定义 + cheerio
   └── 是 → ScrapeGraphAI (playwright 模式) / 自定义 + Puppeteer

4. 语言环境？
   ├── Python 项目 → 直接用 scrapegraphai 包
   └── Node.js 项目 → 用 scrapegraph-js SDK 或调用 ScrapeGraphAI REST API
```

### 3.2 推荐策略：分层混合架构

```
第一层：规则提取（Rule-based）
  - 对已知结构稳定的数据源，维护 CSS 选择器配置
  - 成本最低，速度最快

第二层：JSON-LD / 结构化标记自动提取
  - 优先解析 <script type="application/ld+json">
  - 无需 LLM 即可获得高质量数据

第三层：AI 兜底（AI Fallback）
  - 仅在前两层失败时触发
  - 使用 ScrapeGraphAI 或直接调用 LLM API
  - 对结果做验证（格式、范围、逻辑）

第四层：人工审核队列
  - AI 置信度低时进入人工审核
  - 审核结果反哺规则库
```

### 3.3 关于模块化调用的结论

**可以且推荐模块化调用 ScrapeGraphAI，但需注意：**

1. **用 `ScriptCreatorGraph` 生成规则脚本**：首次用 LLM 分析页面结构，生成可复用的 Python/JS 脚本，后续直接执行脚本（无需再调 LLM）。这是最佳的成本控制策略。

2. **将 AI 提取限定为兜底层**：与 marathon_calendar 现有架构一致，AI 只在规则提取失败时才调用。

3. **本地 Ollama 用于开发调试**：开发阶段用 llama3.2 本地模型，零成本验证提取效果，生产环境再切换到 gpt-4o-mini 等商用模型。

4. **为每个数据源缓存提取结果**：相同内容哈希不重复调用 LLM（marathon_calendar 的 `contentHash` 机制已体现这一思路）。

---

## 4. 技术生态全景

### 4.1 ScrapeGraphAI 集成矩阵

| 集成类型 | 工具 |
|----------|------|
| LLM 框架 | LangChain, LlamaIndex, CrewAI, Agno |
| 低代码平台 | Zapier, n8n, Dify, Pipedream |
| MCP Server | smithery.ai/server/@ScrapeGraphAI/scrapegraph-mcp |
| SDK | Python (`scrapegraphai`), Node.js (`scrapegraph-js`) |

### 4.2 竞品简要对比

| 工具 | 特点 | 适合场景 |
|------|------|----------|
| ScrapeGraphAI | LLM 驱动，结构化输出，开源 | 多变结构，Python 项目 |
| Firecrawl | 托管 API，专注 Markdown 输出，速度快 | 内容提取为主，LLM RAG 场景 |
| Playwright + AI | 完全可控，支持交互 | 复杂登录/JS 页面 |
| Apify | 云端执行，Actor 市场 | 企业级规模爬取 |

---

## 5. 学习建议与后续方向

1. **立即可做**：用 `SmartScraperGraph` + Ollama 在本地验证几个目标数据源的提取效果（无 API 费用）
2. **短期（1-2 周）**：针对高价值数据源，用 `ScriptCreatorGraph` 生成提取脚本，固化为规则配置
3. **中期（1 个月）**：在两个项目中实现分层架构（规则 → JSON-LD → AI兜底），降低 LLM 调用比例
4. **长期**：建立数据源健康监控，当规则失效率上升时自动触发重新学习

---

*报告日期：2026-02-19*
