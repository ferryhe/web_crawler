# Web Crawler Research & Improvement Reports

本仓库为针对两个 AI 爬虫工具的研究学习报告，以及对 [marathon_calendar](https://github.com/ferryhe/marathon_calendar) 和 AI_actuarial_inforsearch 两个项目的爬虫改进建议，同时包含 QMD 混合检索引擎对问题检索能力的增强分析。

## 文档目录

| 文档 | 说明 |
|------|------|
| [research_report.md](./research_report.md) | 学习报告：ScrapeGraphAI 与 AI 爬虫工具的深度研究与选型建议 |
| [improvement_report.md](./improvement_report.md) | 改进报告：两个项目的爬虫功能具体改进方案与示例代码 |
| [qmd_search_report.md](./qmd_search_report.md) | 检索增强报告：QMD 混合搜索引擎对 AI_actuarial_inforsearch 问题检索的改进分析 |

## 研究对象

1. **[ScrapeGraphAI](https://github.com/ScrapeGraphAI/Scrapegraph-ai)** — 基于 LLM + 图逻辑的 Python 爬虫库，支持 OpenAI、Groq、Ollama 等多种模型
2. **[Shubhamsaboo / web_scrapping_ai_agent](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/web_scrapping_ai_agent)** — 基于 ScrapeGraphAI + Streamlit 的 AI 爬虫 UI 原型
3. **[QMD](https://github.com/tobi/qmd)** — 本地端混合搜索引擎，结合 BM25、向量语义搜索和 LLM 重排序，专为知识库问答检索设计

## 目标项目

- **marathon_calendar**（TypeScript / Node.js）— 马拉松赛事日历，爬虫框架已就绪，需补全数据源实现
- **AI_actuarial_inforsearch**（Python）— 精算信息搜索平台，需引入 AI 爬虫能力与混合检索增强
