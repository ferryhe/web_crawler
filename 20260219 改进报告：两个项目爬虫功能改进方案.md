# 改进报告：两个项目爬虫功能改进方案

---

## 项目一：marathon_calendar（TypeScript / Node.js）

### 现状分析

**已有的优秀设计：**
- `crawler/types.ts` 定义了 `CrawlerSource` 接口，支持 RSS / HTML / API 三种策略
- `server/syncScheduler.ts` 实现了：fetch → contentHash 去重 → 规则提取 → AI 兜底 → 原始数据存库 → 人工审核队列
- `server/aiExtractor.ts` 已调用 LLM API（兼容 OpenAI 格式，支持通义千问）提取 raceDate / registrationStatus / registrationUrl
- 分层提取顺序：规则（CSS选择器配置）→ JSON-LD → Regex → AI 兜底

**当前不足：**

| 问题 | 影响 |
|------|------|
| `fetch()` 不支持 JS 渲染 | 动态加载的赛事页面无法抓取 |
| AI 提取只提取3个字段，prompt 未使用 schema 约束 | 输出格式不稳定，遗漏字段 |
| `crawler/` 目录只有 `types.ts`，无实际爬虫实现 | RSS / API 策略无法使用 |
| 无多级页面爬取（列表页 → 详情页）| 无法从列表页批量发现新赛事 |
| 无爬虫健康监控 | 规则失效后无告警 |

---

### 改进方案

#### 改进1：为 `CrawlerSource` 实现 RSS 策略

`crawler/types.ts` 已定义 `strategy: 'RSS'`，补全实现：

```typescript
// crawler/sources/rssSource.ts
import Parser from 'rss-parser';
import type { CrawlerSource, RawEvent } from '../types';

const parser = new Parser();

export function createRssSource(config: {
  sourceId: string;
  name: string;
  baseUrl: string;
  feedUrl: string;
  priority?: number;
}): CrawlerSource {
  return {
    sourceId: config.sourceId,
    name: config.name,
    baseUrl: config.baseUrl,
    strategy: 'RSS',
    priority: config.priority ?? 5,

    async fetchEvents(): Promise<RawEvent[]> {
      const feed = await parser.parseURL(config.feedUrl);
      return feed.items.map((item) => ({
        name: item.title ?? '',
        city: '',                          // RSS 通常无城市字段，需后续 AI 补全
        date: item.pubDate ?? item.isoDate ?? '',
        registrationUrl: item.link,
        status: 'unknown',
        rawDescription: item.contentSnippet ?? item.summary ?? '',
      }));
    },

    parseEvent(raw: RawEvent) {
      return {
        canonicalName: raw.name,
        name: raw.name,
        city: raw.city || '',
        date: new Date(raw.date),
        registrationUrl: raw.registrationUrl ?? '',
        registrationStatus: 'unknown' as const,
        metadata: { source: 'rss', rawDescription: raw.rawDescription },
      };
    },
  };
}
```

#### 改进2：用 Puppeteer 替代 `fetch()` 处理 JS 渲染页面

在 `server/syncScheduler.ts` 的 `fetchWithTimeout` 下方增加：

```typescript
// server/fetchStrategies.ts
import puppeteer from 'puppeteer';

/**
 * 使用 Puppeteer 获取 JS 渲染后的完整 HTML。
 * 仅在 source.strategy === 'HTML' 且 source.config.requireJs === true 时调用。
 */
export async function fetchWithPuppeteer(
  url: string,
  timeoutMs: number,
): Promise<{ html: string; httpStatus: number }> {
  const browser = await puppeteer.launch({ headless: true });
  try {
    const page = await browser.newPage();
    await page.setUserAgent(
      'marathon-calendar/1.0 (+https://github.com/ferryhe/marathon_calendar)',
    );
    const response = await page.goto(url, {
      waitUntil: 'networkidle2',
      timeout: timeoutMs,
    });
    const html = await page.content();
    return { html, httpStatus: response?.status() ?? 200 };
  } finally {
    await browser.close();
  }
}
```

在 `syncScheduler.ts` 中的 `fetchWithTimeout` 调用处添加判断：

```typescript
// syncScheduler.ts 中的修改（在 while 循环内）
const requireJs = Boolean((params.source.config as any)?.requireJs);
let raw: string;
let httpStatus: number;

if (requireJs) {
  const result = await fetchWithPuppeteer(params.sourceUrl, timeoutMs);
  raw = result.html;
  httpStatus = result.httpStatus;
} else {
  const response = await fetchWithTimeout(params.sourceUrl, { timeoutMs, method: 'GET', headers: {} });
  raw = await response.text();
  httpStatus = response.status;
}
```

#### 改进3：增强 AI Extractor 使用 Schema 约束

修改 `server/aiExtractor.ts` 的 prompt，要求 LLM 输出完整赛事结构：

```typescript
// server/aiExtractor.ts 中 prompt 改进

const EXTRACTION_SCHEMA = {
  raceDate: "YYYY-MM-DD 格式的比赛日期，如 2026-10-18",
  raceName: "完整赛事名称，如 '2026北京国际马拉松'",
  city: "举办城市，如 '北京'",
  registrationStatus: "报名状态：open/closed/sold-out/not-open/unknown 之一",
  registrationUrl: "报名链接（完整 URL）或 null",
  registrationDeadline: "报名截止日期 YYYY-MM-DD 或 null",
  distance: "赛事距离，如 '全程马拉松42.195公里' 或 null",
};

const prompt = [
  "你是一个专业的马拉松赛事信息提取助手。",
  "从以下 HTML 页面中提取结构化的赛事信息，严格按照 JSON Schema 输出。",
  `Schema: ${JSON.stringify(EXTRACTION_SCHEMA, null, 2)}`,
  "所有字段均为字符串或 null，不要输出数组或嵌套对象。",
  `pageUrl: ${params.pageUrl}`,
  "html:",
  snippet,
].join("\n");
```

#### 改进4：实现列表页多级爬取

当前只能爬取已知的单个赛事页面。增加列表页爬取以自动发现新赛事：

```typescript
// crawler/sources/listPageSource.ts
import { load } from 'cheerio';
import type { CrawlerSource, RawEvent } from '../types';

export function createListPageSource(config: {
  sourceId: string;
  name: string;
  baseUrl: string;
  listUrl: string;
  /** CSS selector to find event links on the list page */
  linkSelector: string;
  priority?: number;
}): CrawlerSource {
  return {
    sourceId: config.sourceId,
    name: config.name,
    baseUrl: config.baseUrl,
    strategy: 'HTML',
    priority: config.priority ?? 5,

    async fetchEvents(): Promise<RawEvent[]> {
      const response = await fetch(config.listUrl, {
        headers: { 'User-Agent': 'marathon-calendar/1.0' },
      });
      const html = await response.text();
      const $ = load(html);
      const events: RawEvent[] = [];

      $(config.linkSelector).each((_, el) => {
        const name = $(el).text().trim();
        const href = $(el).attr('href');
        if (!name || !href) return;
        const registrationUrl = new URL(href, config.baseUrl).toString();
        events.push({
          name,
          city: '',
          date: '',
          registrationUrl,
          status: 'unknown',
        });
      });

      return events;
    },

    parseEvent(raw: RawEvent) {
      return {
        canonicalName: raw.name,
        name: raw.name,
        city: raw.city || '',
        date: raw.date ? new Date(raw.date) : new Date(0),
        registrationUrl: raw.registrationUrl ?? '',
        registrationStatus: 'unknown' as const,
        metadata: { source: 'list-page' },
      };
    },
  };
}
```

#### 改进5：数据源健康监控

在管理后台数据源列表中增加以下监控指标（可添加到现有的 `sources` 表显示）：

- **连续失败次数**：`consecutiveFailures`
- **规则有效率**：`ruleSuccessRate`（过去 30 次中规则提取成功的比例）
- **AI 兜底率**：当 AI 兜底率 > 50% 时，自动发送告警提示规则需要更新

```sql
-- 建议添加到 sources 表的字段（Drizzle schema 扩展）
consecutiveFailures integer default 0,
ruleSuccessCount integer default 0,
aiExtractionCount integer default 0,
lastRuleUpdateAt timestamp,
```

---

### marathon_calendar 改进优先级

| 优先级 | 改进项 | 工作量 | 价值 |
|--------|--------|--------|------|
| P0 | RSS 策略实现（爱燃烧、最酷体育有 RSS） | 1天 | 立即获取批量数据 |
| P1 | AI prompt 增强（schema 约束） | 半天 | 提升提取成功率和字段完整性 |
| P1 | 列表页多级爬取 | 2天 | 自动发现新赛事，减少人工录入 |
| P2 | Puppeteer JS 渲染支持 | 1天 | 支持动态加载页面 |
| P3 | 健康监控指标 | 1天 | 提前发现规则失效 |

---

## 项目二：AI_actuarial_inforsearch（Python）

> **注**：该 GitHub 仓库目前为私有仓库，以下改进建议基于精算信息搜索的通用需求和问题域特点。

### 推测的现状

精算信息搜索平台的 collectors 目录通常负责：
- 从精算学会网站、保险监管网站抓取政策法规、考试信息
- 从学术数据库获取精算论文/研究报告
- 对抓取内容进行分类、摘要、存储

### 改进方案

#### 改进1：引入 ScrapeGraphAI 作为核心提取引擎

**安装：**
```bash
pip install scrapegraphai
playwright install
```

**替代手写 BeautifulSoup 选择器：**

```python
# ai_actuarial/collectors/base_collector.py
from scrapegraphai.graphs import SmartScraperGraph, SearchGraph
import os

# 配置：优先使用通义千问（成本低），回退到 OpenAI
def get_graph_config(use_local: bool = False) -> dict:
    if use_local:
        # 本地 Ollama（开发调试用，零成本）
        return {
            "llm": {
                "model": "ollama/qwen2.5:7b",
                "base_url": "http://localhost:11434",
                "format": "json",
            },
            "embeddings": {
                "model": "ollama/nomic-embed-text",
                "base_url": "http://localhost:11434",
            },
            "verbose": False,
        }
    
    # 通义千问 API（兼容 OpenAI 格式）
    return {
        "llm": {
            "api_key": os.environ["DASHSCOPE_API_KEY"],
            "model": "openai/qwen-turbo",           # 通义千问最低成本模型
            "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
        },
        "verbose": False,
    }


def scrape_page(url: str, prompt: str, use_local: bool = False) -> dict:
    """
    使用 ScrapeGraphAI 从单个页面提取结构化信息。
    
    Args:
        url: 目标页面 URL
        prompt: 提取指令（中文），例如 "提取文件标题、发布日期、发布机构和正文摘要"
        use_local: True 时使用本地 Ollama 模型（开发调试用）
    
    Returns:
        提取的结构化数据字典
    """
    graph = SmartScraperGraph(
        prompt=prompt,
        source=url,
        config=get_graph_config(use_local),
    )
    return graph.run()
```

#### 改进2：精算信息专用收集器

```python
# ai_actuarial/collectors/regulatory_collector.py
"""
收集保险监管政策文件的爬虫。
目标网站：国家金融监督管理总局（www.nfra.gov.cn）
"""
from scrapegraphai.graphs import SmartScraperGraph, SearchGraph
from .base_collector import get_graph_config
from typing import List, Optional
import hashlib
import json
from datetime import datetime


REGULATORY_EXTRACT_PROMPT = """
从该保险监管网页中提取以下信息，以 JSON 格式返回：
- title: 文件标题（字符串）
- doc_number: 文件编号，如 "银保监发〔2023〕1号"（字符串或 null）
- publish_date: 发布日期 YYYY-MM-DD（字符串或 null）
- issuer: 发布机构，如 "国家金融监督管理总局"（字符串）
- category: 文件类别，如 "规章制度"/"通知"/"公告"/"指引"（字符串）
- summary: 100字以内的核心内容摘要（字符串）
- effective_date: 施行日期 YYYY-MM-DD（字符串或 null）
"""


def collect_regulatory_document(url: str) -> Optional[dict]:
    """从单个监管文件页面提取结构化信息。"""
    try:
        graph = SmartScraperGraph(
            prompt=REGULATORY_EXTRACT_PROMPT,
            source=url,
            config=get_graph_config(),
        )
        result = graph.run()
        result["source_url"] = url
        result["collected_at"] = datetime.utcnow().isoformat()
        result["content_hash"] = hashlib.sha256(
            json.dumps(result, ensure_ascii=False).encode()
        ).hexdigest()[:12]
        return result
    except Exception as e:
        return {"error": str(e), "source_url": url}


EXAM_INFO_PROMPT = """
从该精算师考试相关网页提取以下信息：
- exam_name: 考试名称（字符串）
- exam_date: 考试日期 YYYY-MM-DD 或日期范围（字符串或 null）
- registration_start: 报名开始日期 YYYY-MM-DD（字符串或 null）
- registration_end: 报名截止日期 YYYY-MM-DD（字符串或 null）
- subjects: 考试科目列表（字符串数组）
- fee: 报名费用信息（字符串或 null）
- announcement_url: 报名通知链接（字符串或 null）
"""


def collect_exam_info(url: str) -> Optional[dict]:
    """从精算师考试网页提取报名和考试信息。"""
    try:
        graph = SmartScraperGraph(
            prompt=EXAM_INFO_PROMPT,
            source=url,
            config=get_graph_config(),
        )
        result = graph.run()
        result["source_url"] = url
        result["collected_at"] = datetime.utcnow().isoformat()
        return result
    except Exception as e:
        return {"error": str(e), "source_url": url}
```

#### 改进3：基于搜索的批量发现（SearchGraph）

```python
# ai_actuarial/collectors/search_collector.py
"""
使用 SearchGraph 从搜索引擎批量发现精算相关新信息。
"""
from scrapegraphai.graphs import SearchGraph
from .base_collector import get_graph_config


REGULATORY_SEARCH_PROMPT = """
搜索并整合最近发布的人身保险、财产保险相关监管政策。
对每条结果，提取：
- title: 标题
- date: 发布日期
- issuer: 发布机构
- url: 原文链接
- key_points: 核心要点（2-3条，每条不超过50字）
"""


def search_recent_regulations(
    keywords: str = "保险监管 新规 2026",
    max_results: int = 5,
) -> list:
    """
    通过搜索引擎发现最新监管文件。
    
    Args:
        keywords: 搜索关键词
        max_results: 最多获取的结果数量
    
    Returns:
        结构化监管信息列表
    """
    graph = SearchGraph(
        prompt=REGULATORY_SEARCH_PROMPT,
        config={
            **get_graph_config(),
            "max_results": max_results,
        },
    )
    return graph.run()
```

#### 改进4：ScriptCreatorGraph 生成可复用爬虫

对于结构固定的常用数据源（如精算学会考试页面），一次性用 LLM 生成爬虫脚本，
后续直接执行脚本，无需每次调用 LLM：

```python
# 只需执行一次：生成爬虫脚本
from scrapegraphai.graphs import ScriptCreatorGraph

graph = ScriptCreatorGraph(
    prompt="提取该页面中所有精算师考试的科目名称、考试日期、报名费用",
    source="https://www.actuaries.org.cn/site/exam",
    config=get_graph_config(),
)

script = graph.run()
print(script)  # 输出可复用的 Python 爬虫脚本

# 将生成的脚本保存到 collectors/generated/ 目录
with open("collectors/generated/actuaries_exam_scraper.py", "w") as f:
    f.write(script)
```

#### 改进5：内容去重与增量更新

```python
# ai_actuarial/collectors/dedup.py
"""
基于内容哈希的去重机制，避免重复处理已抓取内容。
"""
import hashlib
import json
from pathlib import Path
from typing import Optional


class ContentCache:
    """简单的文件系统内容哈希缓存。生产环境可替换为 Redis。"""

    def __init__(self, cache_dir: str = ".cache/crawler"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def _hash_key(self, url: str) -> str:
        return hashlib.sha256(url.encode()).hexdigest()[:16]

    def is_changed(self, url: str, content: str) -> bool:
        """Returns True if content has changed since last check."""
        key = self._hash_key(url)
        cache_file = self.cache_dir / f"{key}.json"
        new_hash = hashlib.sha256(content.encode()).hexdigest()

        if cache_file.exists():
            cached = json.loads(cache_file.read_text())
            if cached.get("hash") == new_hash:
                return False

        cache_file.write_text(json.dumps({"url": url, "hash": new_hash}))
        return True

    def get_last_hash(self, url: str) -> Optional[str]:
        key = self._hash_key(url)
        cache_file = self.cache_dir / f"{key}.json"
        if cache_file.exists():
            return json.loads(cache_file.read_text()).get("hash")
        return None
```

---

### AI_actuarial_inforsearch 改进优先级

| 优先级 | 改进项 | 工作量 | 价值 |
|--------|--------|--------|------|
| P0 | 引入 ScrapeGraphAI + 通义千问配置 | 半天 | 大幅减少选择器维护 |
| P0 | 监管文件收集器（collect_regulatory_document） | 1天 | 核心业务场景 |
| P1 | 考试信息收集器（collect_exam_info） | 半天 | 直接用户价值 |
| P1 | SearchGraph 批量发现新信息 | 半天 | 自动化发现，减少人工 |
| P2 | ScriptCreatorGraph 固化常用源 | 1天 | 长期降低 LLM 成本 |
| P2 | 内容去重缓存（ContentCache） | 半天 | 避免重复抓取，节省 API 费用 |

---

## 综合架构建议

### 两个项目的分层爬虫架构对比

```
marathon_calendar (Node.js)          AI_actuarial_inforsearch (Python)
─────────────────────────────        ────────────────────────────────
第1层: CrawlerSource 接口            第1层: BaseCollector 抽象类
       RSS / HTML / API 策略                 fetch + content hash
       
第2层: contentHash 去重              第2层: ContentCache 去重
       skipIfUnchanged                       skip if hash same
       
第3层: 规则提取 (CSS config)         第3层: 规则提取 (CSS / regex)
       + JSON-LD 自动解析                    + ScriptCreator 生成脚本
       
第4层: AI 兜底 (aiExtractor.ts)     第4层: AI 提取 (SmartScraperGraph)
       通义千问 / OpenAI                     通义千问 / Ollama 本地
       
第5层: 人工审核队列                  第5层: 结果验证 + 入库
       rawCrawlData.status='needs_review'     + Pydantic schema 校验
```

### 关键共同原则

1. **内容哈希去重**：相同内容不重复处理（两项目均已有或建议引入）
2. **AI 仅作兜底**：规则失效时才调用，控制成本
3. **schema 约束输出**：强制 LLM 输出符合预期格式，减少后处理错误
4. **本地模型开发**：Ollama + qwen2.5:7b 或 llama3.2 本地调试，零成本
5. **生产使用通义千问**：成本低于 OpenAI，中文效果好，且两项目都已有配置

---

*报告日期：2026-02-19*
