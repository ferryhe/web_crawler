# Copilot 研究助手指令

本仓库是一个用于对 GitHub 项目进行深度研究的研究库。当你在此仓库中工作时，请遵循以下指令。

## 仓库目的

对目标 GitHub 项目进行深度技术研究，产出可复用的分析报告，内容包括：架构分析、问题识别、改进建议、技术对比等。

## 研究技能

执行 GitHub 项目研究任务时，**始终参考** `skills/github_project_research.md` 中定义的工作流程：

1. 使用 GitHub MCP 工具获取项目信息（文件内容、commit 历史、Issues、PRs）
2. 深度分析代码架构、技术栈、核心模块
3. 按 `templates/research_report_template.md` 模板撰写报告
4. 将报告保存至 `reports/YYYYMMDD-[主题].md`

## 报告规范

- 文件名格式：`reports/YYYYMMDD-[主题描述].md`（例如 `reports/20260301-ScrapeGraphAI分析.md`）
- 正文使用中文，代码和专有名词保持英文
- 技术结论必须有代码或文档依据，不猜测
- 改进建议必须包含具体实现路径和代码示例

## 目录结构

```
reports/    ← 所有研究报告（按日期命名）
templates/  ← 报告模板
skills/     ← 研究技能定义
```

## 工具使用优先级

1. **GitHub MCP 工具**（`get_file_contents`、`list_commits`、`search_code` 等）用于获取项目信息
2. **web_search** 用于查找项目背景资料、对比工具信息
3. **view / edit / create** 用于撰写和保存报告

研究任务中，优先使用 GitHub MCP 工具直接读取源代码，而不是依赖搜索引擎描述。
