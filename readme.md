# Web Crawler Research

本仓库用于对 GitHub 项目进行深度技术研究，产出可复用的分析报告。研究方向包括：AI 爬虫工具、混合检索引擎、项目架构改进等。

## 目录结构

```
web_crawler/
├── reports/          # 所有研究报告（按日期命名：YYYYMMDD-主题.md）
├── templates/        # 报告模板
│   └── research_report_template.md
├── skills/           # 研究技能定义
│   └── github_project_research.md
└── .github/
    └── copilot-instructions.md  # Copilot 研究助手指令
```

## 如何产出研究报告

1. **使用 Copilot Coding Agent**：在 Issue 或 PR 中描述研究目标（例如"对 [仓库 URL] 进行深度分析"），Agent 将自动按照 `skills/github_project_research.md` 中的工作流执行研究并输出报告。

2. **手动撰写**：复制 `templates/research_report_template.md`，填写各节内容，保存至 `reports/YYYYMMDD-[主题].md`。

## 研究报告列表

| 日期 | 报告 | 研究主题 |
|------|------|---------|
| 2026-02-19 | [AI 爬虫工具研究与选型建议](./reports/20260219-AI爬虫工具研究与选型建议.md) | ScrapeGraphAI、AI 爬虫工具对比与选型 |
| 2026-02-19 | [两个项目爬虫功能改进方案](./reports/20260219-两个项目爬虫功能改进方案.md) | marathon_calendar & AI_actuarial_inforsearch 改进建议 |
| 2026-02-25 | [AI_actuarial_inforsearch 检索能力改进分析](./reports/20260225-AI_actuarial_inforsearch检索能力改进分析.md) | QMD 混合搜索引擎集成分析 |

## 已研究的项目

| 项目 | 语言 | 研究方向 |
|------|------|---------|
| [ScrapeGraphAI](https://github.com/ScrapeGraphAI/Scrapegraph-ai) | Python | LLM 驱动的爬虫库，架构与选型分析 |
| [web_scrapping_ai_agent](https://github.com/Shubhamsaboo/awesome-llm-apps/tree/main/starter_ai_agents/web_scrapping_ai_agent) | Python | ScrapeGraphAI UI 原型，价值评估 |
| [QMD](https://github.com/tobi/qmd) | TypeScript | 本地混合检索引擎，BM25 + 向量 + 重排序 |
| [marathon_calendar](https://github.com/ferryhe/marathon_calendar) | TypeScript | 马拉松赛事爬虫框架改进 |
| AI_actuarial_inforsearch | Python | 精算信息搜索平台，AI 爬虫与检索增强 |
