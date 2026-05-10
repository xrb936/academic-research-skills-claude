# CNKI Search Protocol

**Status**: v1.0
**Used by**: `bibliography_agent`, `source_verification_agent`, `integrity_verification_agent`
**Dependencies**: `playwright` (`pip install playwright && playwright install chromium`)
**Env var**: `CNKI_MCP_PROFILE_DIR`（可选；浏览器 profile 目录，用于跨次调用保持 cookie）

---

## Purpose

通过 CNKI（中国知网）进行中文学术文献的程序化检索。CNKI 是最大的中文学术数据库，索引期刊论文、硕博论文、会议论文和报纸。

WebSearch 和 Semantic Scholar 对中文文献的覆盖不完整——大量中文期刊论文、硕博论文仅在 CNKI 可检索。本协议补充现有检索链，为 agent 提供 CNKI 搜索能力。

**与 Semantic Scholar 协议的区别**：S2 有公开 REST API，agent 可直接用 curl 调用。CNKI 没有公开 API，需要通过 Playwright 浏览器自动化来渲染页面并提取 DOM。因此本协议包含一段完整的 Python 脚本，agent 需要时将其写入临时文件并通过 Bash 执行。

---

## When to Use

⚠️ **IRON RULE**: 检测到以下**任一**触发条件时，CNKI 搜索是**强制性的，不是可选的**。bibliography_agent 在进入 Step 3 之前必须检查这些条件。

1. **用户明确要求**：用户明确要求搜索中文文献、提及 CNKI / 知网、或指定需要中文来源
2. **RQ 包含中文文本**：研究问题或用户消息包含中文文字
3. **中文参考文献验证**：论文中已有中文参考文献需要事实核查或存在性验证（作者和期刊均为中文，且 Semantic Scholar 未收录 `S2_NOT_FOUND`）

以上触发条件**均未命中**时，跳过 CNKI 搜索。不要仅因为研究主题与中国相关就自动触发——英文文献足以覆盖的中国研究主题不需要调用 CNKI。

WebSearch 不能替代 CNKI：大量中文期刊论文和硕博论文仅在 CNKI 可检索，WebSearch 对中文学术文献的覆盖率显著低于 CNKI。当触发条件命中时，必须调用 CNKI 而非仅用 WebSearch 搜索中文关键词。

---

## Query Patterns

### Pattern 1: 关键词文献发现（bibliography_agent Step 2.5）

将 RQ 核心概念翻译为中文关键词，2-4 个词用空格连接（CNKI 以空格为 AND）。用于发现新的中文文献来源。

```
python cnki_search.py "高等教育质量保障" --max-results 20
```

多面向 RQ 时，组合不同概念组的关键词。需要多次搜索时，每次间隔至少 5 秒。

```
python cnki_search.py "人工智能 高等教育 教学改革" --max-results 10
```

### Pattern 2: 标题验证（source_verification_agent CNKI 补充检查）

用已知论文的**完整中文标题**作为搜索词，验证该论文是否存在于 CNKI。类似 S2 的 Pattern 1 (Title Search)。返回少量候选结果，通过 Levenshtein 标题相似度 >= 0.80 匹配。

```
python cnki_search.py "基于人工智能的高校教学质量评价体系研究" --max-results 5
```

**匹配规则**：对每条返回结果，计算标题的 Levenshtein 相似度（大小写不敏感、去除标点）。相似度 >= 0.80 且年份匹配（±1 年）则判定为同一篇论文，记录为 `CNKI_VERIFIED`。若多条结果均 >= 0.80，优先选择年份完全匹配的。

### Pattern 3: DOI 交叉验证

当 CNKI 搜索结果包含 DOI 时，可与 S2 或 DOI 解析（`https://doi.org/{doi}`）交叉验证。CNKI 返回的 DOI 与其他来源的 DOI 一致 → 确认为同一篇文献（用于去重和验证）。

此模式不需要单独调用脚本，而是在 Pattern 1/2 的结果中提取 DOI 字段进行比对。

### Parameters

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `query`（位置参数） | 搜索关键词，中文或英文 | 必填 |
| `--max-results` | 返回结果数（无上限，支持自动翻页） | 20 |

CNKI 每页显示约 20 条结果。当 `--max-results` 超过 20 时，脚本自动点击翻页按钮获取后续页面。每次翻页间隔 1 秒。

**各模式建议用量**：

| 模式 | 调用 Agent | 使用 Pattern | 建议 `--max-results` |
|------|-----------|-------------|---------------------|
| `full` | bibliography_agent + source_verification_agent | Pattern 1 (发现) + Pattern 2 (验证) | 发现: 20, 验证: 5 |
| `quick` | bibliography_agent + source_verification_agent | Pattern 1 (发现) + Pattern 2 (验证) | 发现: 10, 验证: 5 |
| `lit-review` | bibliography_agent + source_verification_agent | Pattern 1 (发现) + Pattern 2 (验证) | 发现: 30-40, 验证: 5 |
| `systematic-review` | bibliography_agent + source_verification_agent | Pattern 1 (发现) + Pattern 2 (验证) | 发现: 50-100, 验证: 5 |
| `fact-check` | source_verification_agent only | Pattern 2 (验证) only | 验证: 5 |
| `review` | 不涉及文献搜索 | — | 不调用 |
| `socratic` | 不涉及文献搜索 | — | 不调用 |

脚本始终自动提取每条搜索结果的完整详情（摘要、关键词、DOI、卷期页码等），无需额外标志。

---

## Invocation

agent 需要搜索 CNKI 时，执行以下步骤：

1. 将下方 **§ CNKI Search Script** 的完整 Python 代码写入临时文件（如 `cnki_search.py`）
2. 从 RQ 派生 **2-4 组中文关键词**（与 WebSearch 多组检索策略一致）：
   - 每组用不同的同义词、概念组合、或子主题角度
   - 例：RQ "飞行员绩效" → `["飞行员 绩效评估", "民航飞行员 表现", "飞行员 工作绩效 影响因素"]`
3. 对**每组关键词**分别调用一次 CNKI 搜索（间隔至少 5 秒）：
   ```
   python cnki_search.py "飞行员 绩效评估" --max-results 20
   python cnki_search.py "民航飞行员 表现" --max-results 20
   python cnki_search.py "飞行员 工作绩效 影响因素" --max-results 20
   ```
4. 从 stdout 解析每次调用的 JSON 输出，合并结果，按标题相似度去重（Levenshtein >= 0.80）
5. 执行完毕后删除临时文件

⚠️ **仅搜索一组关键词是不够的**——不同关键词组合会命中不同的文献，多组检索才能保证覆盖率。

---

## Response Format

脚本输出 JSON 到 stdout：

```json
{
  "query": "高等教育质量保障",
  "results": [
    {
      "index": 1,
      "title": "我国高等教育质量保障体系的构建与完善",
      "url": "https://kns.cnki.net/kcms2/article/abstract?v=...",
      "authors": ["张三", "李四"],
      "source": "高等教育研究",
      "year": "2024",
      "abstract": "本文探讨了...",
      "keywords": ["高等教育", "质量保障", "评估体系"],
      "doi": "10.16298/j.cnki.1004-3667.2024.01.05",
      "journal": "高等教育研究",
      "volume": "45",
      "issue": "1",
      "pages": "55-62",
      "publish_date": "2024-01-15",
      "fund": "国家社科基金项目(20BGL001)",
      "subject": "高等教育",
      "collection": "社会科学I辑",
      "clc_number": "G649.2",
      "citation_count": 15,
      "download_count": 230
    }
  ],
  "result_count": 10,
  "location": "domestic"
}
```

### 字段映射到参考文献格式

| CNKI 字段 | 用途 |
|-----------|------|
| `title` | 参考文献标题（中文） |
| `authors` | 作者列表 |
| `source` / `journal` | 期刊 / 来源名称 |
| `year` | 出版年份 |
| `volume` / `issue` / `pages` | 卷号、期号、页码（APA 引用格式需要） |
| `publish_date` | 精确出版日期（ISO 8601） |
| `doi` | DOI，用于 Tier 1 验证和 APA 引用 |
| `abstract` | Step 3 筛选（纳入/排除标准） |
| `keywords` | 主题分类 |
| `fund` | 基金项目信息 |
| `subject` | CNKI 专题分类 |
| `collection` | CNKI 专辑分类 |
| `clc_number` | 中国图书馆分类号 |
| `citation_count` | 影响力评估（参考性） |
| `url` | CNKI 详情页 URL（验证和访问用） |

---

## Location Detection

脚本自动检测网络位置：

- **国内**（Domestic）：使用 `kns.cnki.net` — 完整访问，速度快
- **海外**（Overseas）：使用 `oversea.cnki.net` — 内容相同，全文可能受限；自动点击"中文"过滤按钮，仅检索中文数据库

检测方法：HTTP 探测 `https://kns.cnki.net/`，状态码 418 表示海外。每次会话缓存结果。

---

## Deduplication with Other Sources

合并 CNKI 结果与 Semantic Scholar / WebSearch 结果时：

1. **DOI 匹配**：CNKI 结果的 DOI 与已有来源匹配 → 同一篇论文，保留元数据更完整的版本
2. **标题相似度**：计算标准化标题相似度，双语论文（中文标题 + 英文标题）两者都检查，相似度 >= 0.80 视为重复
3. **无匹配**：视为新的独立来源

---

## Graceful Degradation

- **Playwright 未安装**：脚本报错并返回空结果 JSON。记录 `[CNKI-UNAVAILABLE: playwright not installed]`，使用中文关键词 WebSearch 替代
- **CNKI 不可达**：脚本返回空结果并附 error 字段。记录 `[CNKI-UNAVAILABLE: network error]`，继续
- **零结果**：非错误，查询可能确实无匹配。记录并继续
- **永远不因 CNKI 故障阻塞 pipeline** — CNKI 是 WebSearch + Semantic Scholar 验证链的补充

---

## Rate Limiting

- 每个详情页提取间隔 0.5 秒（脚本内置）
- 翻页间隔 1 秒（脚本内置）
- 两次 CNKI 搜索之间至少间隔 3 秒
- 多组关键词搜索时，每次间隔至少 5 秒
- CNKI 可能对频繁访问触发验证码 — 脚本以有头模式运行，遇到验证码时结果可能不完整，回退到 WebSearch
- 大量结果（>60 篇）会显著增加运行时间，agent 应权衡收益

---

## Coverage Notes

- CNKI 索引**中文**期刊、硕博论文、会议论文、报纸
- **英文**中国研究论文优先用 Semantic Scholar 或 Google Scholar
- CNKI 对**中国大陆**出版物覆盖最强；台湾/香港出版物部分覆盖
- 最近 1-2 周的论文可能尚未索引
- 海外版自动过滤为仅中文数据库

---

## § CNKI Search Script

以下是完整的 Python 脚本。agent 执行 CNKI 搜索时，将此代码写入临时文件并通过 Bash 运行。

**依赖**：`playwright`（`pip install playwright && playwright install chromium`）

```python
#!/usr/bin/env python3
"""CNKI search script — search China National Knowledge Infrastructure.

Usage:
    python cnki_search.py "高等教育质量保障" --max-results 20

Outputs JSON to stdout. Errors to stderr.
Extracted from: https://github.com/Imbad0202/cnki-mcp (search-only subset)
"""
from __future__ import annotations

import argparse
import asyncio
import json
import os
import re
import sys
import tempfile
from urllib.parse import quote, urljoin

# ---------------------------------------------------------------------------
# Selectors
# ---------------------------------------------------------------------------

SEARCH_URLS = [
    "https://kns.cnki.net/kns8s/defaultresult/index?kw={query}",
    "https://kns.cnki.net/kns8/defaultresult/index?kw={query}",
    "https://kns.cnki.net/kns/defaultresult/index?kw={query}",
]
OVERSEAS_SEARCH_URLS = [
    "https://oversea.cnki.net/kns8s/defaultresult/index?kw={query}&language=chs",
]
DOMESTIC_PROBE_URL = "https://kns.cnki.net/"

SEARCH_INPUT_SELECTORS = [
    "input[name='txt_1_value1']",
    "#txt_SearchText",
    "textarea#txt_SearchText",
    ".search-input",
    "textarea.search-input",
    "input.search-input",
    "input[placeholder*='检索']",
    "input[placeholder*='搜索']",
    "textarea[placeholder*='文献']",
    "input[placeholder*='请输入']",
    "input[type='text']",
]
SEARCH_BUTTON_SELECTORS = [
    "div.search-btn",
    "input[type='button'][value*='检索']",
    "input[type='submit'][value*='检索']",
    "button:has-text('检索')",
    "button:has-text('搜索')",
    "a:has-text('检索')",
]
TITLE_LINK_SELECTORS = [
    "a.fz14",
    "td.name a",
    "a[href*='detail']",
    "a[href*='Detail']",
    "a[href*='KCMS']",
    "a[href*='kns']",
]
RESULT_ITEM_SELECTORS = [
    ".result-table-list tbody tr",
    "table.result-table-list tbody tr",
    "tr",
    ".result-list .item",
    ".result-item",
    ".list-item",
]

# ---------------------------------------------------------------------------
# Parsing helpers
# ---------------------------------------------------------------------------

_SPACE_RE = re.compile(r"\s+")
_YEAR_RE = re.compile(r"\b(19|20)\d{2}\b")


def _clean(value):
    if not value:
        return ""
    return _SPACE_RE.sub(" ", value).strip()


def _split_people(value):
    value = _clean(value)
    if not value:
        return []
    parts = re.split(r"[;,，；\s]+", value)
    return [p for p in (_clean(x) for x in parts) if p]


def _parse_result_rows(rows, base_url, max_results):
    results = []
    seen = set()
    for row in rows:
        title = _clean(row.get("title"))
        href = _clean(row.get("href"))
        if not title or not href:
            continue
        url = urljoin(base_url, href)
        key = url.lower()
        if key in seen:
            continue
        seen.add(key)
        text = _clean(row.get("text"))
        year_raw = _clean(row.get("year"))
        if not year_raw:
            m = _YEAR_RE.search(text)
            year_raw = m.group(0) if m else None
        results.append({
            "index": len(results) + 1,
            "title": title,
            "url": url,
            "authors": _split_people(row.get("authors", "")),
            "source": _clean(row.get("source")) or None,
            "year": year_raw,
            "abstract": None,
            "keywords": [],
            "doi": None,
            "journal": None,
            "volume": None,
            "issue": None,
            "pages": None,
            "publish_date": None,
            "fund": None,
            "subject": None,
            "collection": None,
            "clc_number": None,
            "citation_count": None,
            "download_count": None,
        })
        if len(results) >= max_results:
            break
    return results


# ---------------------------------------------------------------------------
# JavaScript for DOM extraction
# ---------------------------------------------------------------------------

EXTRACT_ROWS_JS = """
({ itemSelectors, titleSelectors }) => {
  const clean = (value) => (value || '').replace(/\\s+/g, ' ').trim();
  const rows = [];
  const seen = new Set();
  const findTitleLink = (node) => {
    for (const selector of titleSelectors) {
      const link = node.querySelector(selector);
      if (link && clean(link.textContent) && link.href) return link;
    }
    const links = Array.from(node.querySelectorAll('a[href]'));
    return links.find((link) => clean(link.textContent).length >= 4 && link.href);
  };
  for (const itemSelector of itemSelectors) {
    for (const node of Array.from(document.querySelectorAll(itemSelector))) {
      const link = findTitleLink(node);
      if (!link) continue;
      const href = link.href || link.getAttribute('href') || '';
      const title = clean(link.textContent || link.getAttribute('title'));
      if (!href || !title || seen.has(href)) continue;
      seen.add(href);
      let authors = '';
      let source = '';
      let year = '';
      const authorEl = node.querySelector('td.author, .author');
      if (authorEl) authors = clean(authorEl.textContent);
      const sourceEl = node.querySelector('td.source, .source');
      if (sourceEl) source = clean(sourceEl.textContent);
      const dateEl = node.querySelector('td.date, .date');
      if (dateEl) year = clean(dateEl.textContent);
      if (!authors && !source) {
        const tds = Array.from(node.querySelectorAll('td'));
        if (tds.length >= 4) {
          authors = clean(tds[1]?.textContent);
          source = clean(tds[2]?.textContent);
          year = year || clean(tds[3]?.textContent);
        }
      }
      const fullText = clean(node.textContent);
      if (!year) {
        const m = fullText.match(/\\b(19|20)\\d{2}[-/]?\\d{0,2}[-/]?\\d{0,2}\\b/);
        if (m) year = m[0];
      }
      rows.push({ title, href, text: fullText, authors, source, year, abstract: '' });
    }
    if (rows.length > 0) break;
  }
  return rows;
}
"""

FETCH_DETAIL_JS = """
async (url) => {
    try {
        const resp = await fetch(url, { credentials: 'include' });
        if (!resp.ok) return { error: 'HTTP ' + resp.status };
        const html = await resp.text();
        const doc = new DOMParser().parseFromString(html, 'text/html');
        const clean = s => (s || '').replace(/\\s+/g, ' ').trim();

        let abstract = '';
        for (const sel of ['#ChDivSummary', '.abstract-text', 'span#ChDivSummary', 'div.abstract']) {
            const el = doc.querySelector(sel);
            if (el && el.textContent.trim()) { abstract = el.textContent.trim(); break; }
        }

        const keywords = [];
        for (const sel of ['div.keywords a', '.keywords a', 'p.keywords a', "a[href*='kw=']"]) {
            const els = doc.querySelectorAll(sel);
            if (els.length > 0) {
                els.forEach(el => {
                    const t = el.textContent.replace(/;$/, '').trim();
                    if (t && !keywords.includes(t)) keywords.push(t);
                });
                break;
            }
        }
        if (keywords.length === 0) {
            const kwSection = doc.querySelector('#doc-keyword');
            if (kwSection) {
                const scripts = kwSection.querySelectorAll('script');
                for (const s of scripts) {
                    const m = s.textContent.match(/var\\s+mtky\\s*=\\s*'([^']+)'/);
                    if (m) {
                        m[1].split(/[；;]/).forEach(k => {
                            const t = k.trim();
                            if (t && !keywords.includes(t)) keywords.push(t);
                        });
                        break;
                    }
                }
            }
        }

        let doi = '';
        const doiLink = doc.querySelector('a[href*="doi.org"]');
        if (doiLink) doi = clean(doiLink.textContent) || (doiLink.getAttribute('href') || '').replace(/.*doi\\.org\\//, '');

        let journal = '';
        for (const jsel of ['.top-tip a[href*="knavi/detail"]', 'a.journal-title', '.top-tip-scholar a[href*="journal/index"]', 'a[href*="journal/index"]']) {
            const el = doc.querySelector(jsel);
            if (el) { journal = clean(el.textContent).replace(/\\s*\\.$/, ''); break; }
        }

        let fund = '';
        const fundEls = doc.querySelectorAll('.funds a, .fund a, p.fund');
        const fundParts = [];
        fundEls.forEach(el => { const t = clean(el.textContent); if (t) fundParts.push(t); });
        fund = fundParts.join('; ');

        const authors = [];
        const authorContainer = doc.querySelector('#authorpart');
        if (authorContainer) {
            const aEls = authorContainer.querySelectorAll('a');
            if (aEls.length > 0) {
                aEls.forEach(el => { const n = clean(el.textContent); if (n && !authors.includes(n)) authors.push(n); });
            } else {
                authorContainer.querySelectorAll('span').forEach(el => {
                    const n = clean(el.textContent);
                    if (n && !authors.includes(n)) authors.push(n);
                });
            }
        }

        let source = '';
        for (const ssel of ['.top-tip a[href*="knavi/detail"]', 'a.journal-title', '.top-tip-scholar a[href*="journal/index"]', 'a[href*="journal/index"]']) {
            const el = doc.querySelector(ssel);
            if (el) { source = clean(el.textContent).replace(/\\s*\\.$/, ''); break; }
        }

        let year = '';
        let volume = '';
        let issue = '';
        let pages = '';
        let publishDate = '';
        for (const tipSel of ['.top-tip', '.top-tip-scholar', '.top-first']) {
            const tip = doc.querySelector(tipSel);
            if (!tip) continue;
            const tipText = tip.textContent;
            const links = tip.querySelectorAll('a');
            for (const a of links) {
                const m = a.textContent.match(/((?:19|20)\\d{2})/);
                if (m) { year = m[1]; break; }
            }
            if (!year) {
                const tm = tipText.match(/(?:^|[.\\s])\\s*((?:19|20)\\d{2})\\s*[.,]/);
                if (tm) year = tm[1];
            }
            let vm = tipText.match(/Volume\\s+(\\d+)/i) || tipText.match(/(\\d+)\\s*卷/) || tipText.match(/,\\s*(\\d+)\\s*\\(/);
            if (vm) volume = vm[1];
            let im = tipText.match(/Issue\\s+(\\d+)/i) || tipText.match(/(\\d+)\\s*期/) || tipText.match(/\\((\\d+)\\)/);
            if (im) issue = im[1];
            let pm = tipText.match(/PP\\s+([\\d-]+)/i) || tipText.match(/:\\s*(\\d+-\\d+)/);
            if (pm) pages = pm[1];
            const dm = tipText.match(/((?:19|20)\\d{2})[-.](\\d{1,2})[-.](\\d{1,2})/);
            if (dm) publishDate = `${dm[1]}-${dm[2].padStart(2,'0')}-${dm[3].padStart(2,'0')}`;
            if (year) break;
        }
        if (!pages) {
            const pgSpan = doc.querySelectorAll('span');
            pgSpan.forEach(el => {
                if (pages) return;
                const m = el.textContent.match(/页码[：:]\s*([\d]+-[\d]+)/);
                if (m) pages = m[1];
            });
        }

        let subject = '';
        let collection = '';
        let clcNumber = '';
        const clcEl = doc.querySelector('p.clc-code, .clc-code');
        if (clcEl) clcNumber = clean(clcEl.textContent);
        const infoEls = doc.querySelectorAll('.wx-tit li, p.base-info, .summary li, .orgn-list li, .doc li, li.top-space');
        infoEls.forEach(el => {
            const rowtit = el.querySelector('.rowtit');
            const label = rowtit ? rowtit.textContent.trim() : '';
            const valEl = el.querySelector('p') || el;
            const val = clean(rowtit ? valEl.textContent : el.textContent);
            if (label.includes('专辑') || val.startsWith('专辑')) {
                collection = collection || clean(val.replace(/^专辑[：:]?\s*/, ''));
            }
            if (label.includes('专题') || val.startsWith('专题')) {
                subject = subject || clean(val.replace(/^专题[：:]?\s*/, ''));
            }
            if (!clcNumber && (label.includes('分类号') || label.includes('CLC') || val.includes('分类号'))) {
                const clcP = el.querySelector('p.clc-code, p');
                if (clcP) clcNumber = clean(clcP.textContent);
            }
        });

        let citationCount = null;
        let downloadCount = null;
        const allSpans = doc.querySelectorAll('span');
        allSpans.forEach(el => {
            const t = el.textContent.trim();
            let m;
            m = t.match(/被引[：:]\\s*(\\d+)/);
            if (m) citationCount = parseInt(m[1], 10);
            m = t.match(/下载[：:]\\s*(\\d+)/);
            if (m) downloadCount = parseInt(m[1], 10);
        });
        const countEls = doc.querySelectorAll('.total-inform span, .count em, #citation em, #download em');
        countEls.forEach(el => {
            const num = parseInt(el.textContent, 10);
            if (isNaN(num)) return;
            const parent = clean(el.parentElement?.textContent || '');
            if (parent.includes('被引') || parent.includes('引用') || parent.includes('Cited')) citationCount = citationCount ?? num;
            else if (parent.includes('下载') || parent.includes('Download')) downloadCount = downloadCount ?? num;
        });
        const marktips = doc.querySelectorAll('.marktip, span.marktip');
        marktips.forEach(el => {
            const t = el.textContent.trim();
            let m;
            m = t.match(/Cited:\\s*(\\d+)/i);
            if (m && citationCount === null) citationCount = parseInt(m[1], 10);
            m = t.match(/Download:\\s*(\\d+)/i);
            if (m && downloadCount === null) downloadCount = parseInt(m[1], 10);
        });

        return { abstract, keywords, doi, journal, fund, authors, source, year, volume, issue, pages, publishDate, subject, collection, clcNumber, citationCount, downloadCount, error: null };
    } catch (e) {
        return { error: e.message || String(e) };
    }
}
"""

# ---------------------------------------------------------------------------
# CNKI searcher
# ---------------------------------------------------------------------------

_DETAIL_DEFAULTS = {
    "abstract": "", "keywords": [], "doi": "", "journal": "",
    "fund": "", "authors": [], "source": "", "year": "",
    "volume": "", "issue": "", "pages": "", "publishDate": "",
    "subject": "", "collection": "", "clcNumber": "",
    "citationCount": None, "downloadCount": None,
}


class CnkiSearcher:
    """Playwright-based CNKI search client (search only, no auth)."""

    def __init__(self, profile_dir=None):
        self._profile_dir = profile_dir
        self._playwright = None
        self._context = None
        self._location = "unknown"

    async def start(self):
        from playwright.async_api import async_playwright
        self._playwright = await async_playwright().start()
        user_data_dir = self._profile_dir or tempfile.mkdtemp(prefix="cnki_search_")
        self._context = await self._playwright.chromium.launch_persistent_context(
            user_data_dir=user_data_dir,
            headless=False,
            viewport={"width": 1366, "height": 900},
            ignore_https_errors=True,
        )
        self._context.set_default_timeout(45_000)

    async def stop(self):
        if self._context is not None:
            await self._context.close()
            self._context = None
        if self._playwright is not None:
            await self._playwright.stop()
            self._playwright = None

    async def __aenter__(self):
        await self.start()
        return self

    async def __aexit__(self, *exc):
        await self.stop()

    async def _get_page(self):
        assert self._context is not None
        if self._context.pages:
            return self._context.pages[0]
        return await self._context.new_page()

    async def _detect_location(self):
        if self._location != "unknown":
            return
        page = await self._get_page()
        try:
            resp = await page.request.get(DOMESTIC_PROBE_URL, timeout=10_000)
            self._location = "overseas" if resp.status == 418 else "domestic"
        except Exception:
            self._location = "overseas"

    async def _open_search(self, page, query):
        from playwright.async_api import TimeoutError as PwTimeout, Error as PwError
        await self._detect_location()
        urls = OVERSEAS_SEARCH_URLS if self._location == "overseas" else SEARCH_URLS
        home = (
            "https://oversea.cnki.net/index/?language=chs"
            if self._location == "overseas"
            else "https://www.cnki.net/"
        )
        encoded = quote(query)
        last_error = None
        for template in urls:
            try:
                await page.goto(
                    template.format(query=encoded), wait_until="domcontentloaded"
                )
                try:
                    await page.wait_for_load_state("networkidle", timeout=10_000)
                except PwTimeout:
                    pass
                if await self._has_results(page):
                    return
            except PwError as exc:
                last_error = exc
                continue
        try:
            await page.goto(home, wait_until="domcontentloaded")
        except PwError as exc:
            last_error = last_error or exc
        if await self._fill_first(page, SEARCH_INPUT_SELECTORS, query):
            if await self._click_first(page, SEARCH_BUTTON_SELECTORS):
                await page.wait_for_load_state("domcontentloaded")
                return
            await page.keyboard.press("Enter")
            await page.wait_for_load_state("domcontentloaded")
            return
        if last_error:
            raise RuntimeError(f"Unable to open CNKI search page: {last_error}") from last_error
        raise RuntimeError("Unable to locate CNKI search input.")

    async def _apply_chinese_filter(self, page):
        """Click the '中文' database switch on overseas CNKI to filter out foreign papers."""
        from playwright.async_api import TimeoutError as PwTimeout, Error as PwError
        try:
            ch_btn = page.locator("a.ch")
            if await ch_btn.count() > 0 and await ch_btn.is_visible():
                if "active" not in (await ch_btn.get_attribute("class") or ""):
                    await ch_btn.click()
                    await page.wait_for_load_state("domcontentloaded")
                    try:
                        await page.wait_for_load_state("networkidle", timeout=8_000)
                    except PwTimeout:
                        pass
                    await self._wait_for_results(page)
        except PwError:
            pass

    async def _has_results(self, page):
        for sel in TITLE_LINK_SELECTORS:
            try:
                if await page.locator(sel).count() > 0:
                    return True
            except Exception:
                continue
        return False

    async def _wait_for_results(self, page):
        from playwright.async_api import TimeoutError as PwTimeout
        try:
            await page.wait_for_load_state("networkidle", timeout=8_000)
        except PwTimeout:
            pass
        for sel in TITLE_LINK_SELECTORS:
            try:
                await page.locator(sel).first.wait_for(timeout=5_000)
                return
            except PwTimeout:
                continue

    async def _fill_first(self, page, selectors, value):
        for sel in selectors:
            loc = page.locator(sel).first
            try:
                if await loc.count() > 0 and await loc.is_visible():
                    await loc.fill(value)
                    return True
            except Exception:
                continue
        return False

    async def _click_first(self, page, selectors):
        for sel in selectors:
            loc = page.locator(sel).first
            try:
                if await loc.count() > 0 and await loc.is_visible():
                    await loc.click()
                    return True
            except Exception:
                continue
        return False

    async def _extract_rows(self, page):
        raw = await page.evaluate(
            EXTRACT_ROWS_JS,
            {
                "itemSelectors": RESULT_ITEM_SELECTORS,
                "titleSelectors": TITLE_LINK_SELECTORS,
            },
        )
        if isinstance(raw, str):
            return json.loads(raw)
        return raw

    async def _fetch_detail(self, page, url):
        try:
            result = await page.evaluate(FETCH_DETAIL_JS, url)
            result["url"] = url
            if result.get("error"):
                for k, v in _DETAIL_DEFAULTS.items():
                    result.setdefault(k, v)
            return result
        except Exception as exc:
            return {**_DETAIL_DEFAULTS, "url": url, "error": str(exc)}

    async def _goto_next_page(self, page):
        """Click the next-page button. Returns True if navigation succeeded."""
        from playwright.async_api import TimeoutError as PwTimeout
        next_selectors = [
            "#PageNext",
            "a.next",
            "a:has-text('下一页')",
            "a:has-text('>')",
            ".pager a.next",
        ]
        for sel in next_selectors:
            try:
                loc = page.locator(sel).first
                if await loc.count() > 0 and await loc.is_visible():
                    disabled = await loc.get_attribute("disabled")
                    cls = await loc.get_attribute("class") or ""
                    if disabled or "disabled" in cls:
                        return False
                    await loc.click()
                    await page.wait_for_load_state("domcontentloaded")
                    try:
                        await page.wait_for_load_state("networkidle", timeout=8_000)
                    except PwTimeout:
                        pass
                    await self._wait_for_results(page)
                    return True
            except Exception:
                continue
        return False

    def _enrich_result(self, r, detail):
        """Apply detail fields to a result dict."""
        if detail.get("error"):
            return
        field_map = {
            "abstract": "abstract", "keywords": "keywords", "doi": "doi",
            "journal": "journal", "fund": "fund", "volume": "volume",
            "issue": "issue", "pages": "pages", "subject": "subject",
            "collection": "collection",
        }
        for src_key, dst_key in field_map.items():
            if detail.get(src_key):
                r[dst_key] = detail[src_key]
        if detail.get("publishDate"):
            r["publish_date"] = detail["publishDate"]
        if detail.get("clcNumber"):
            r["clc_number"] = detail["clcNumber"]
        if detail.get("authors") and not r["authors"]:
            r["authors"] = detail["authors"]
        if detail.get("source") and not r["source"]:
            r["source"] = detail["source"]
        if detail.get("citationCount") is not None:
            r["citation_count"] = detail["citationCount"]
        if detail.get("downloadCount") is not None:
            r["download_count"] = detail["downloadCount"]
        if r["year"] and len(r["year"]) > 4 and not r.get("publish_date"):
            r["publish_date"] = r["year"]
        if r["year"]:
            m = re.search(r"((?:19|20)\d{2})", r["year"])
            if m:
                r["year"] = m.group(1)

    async def search(self, query, max_results=20):
        max_results = max(1, max_results)
        page = await self._get_page()
        await self._open_search(page, query)
        await self._wait_for_results(page)
        if self._location == "overseas":
            await self._apply_chinese_filter(page)

        all_results = []
        seen_urls = set()
        page_num = 1
        max_pages = (max_results + 19) // 20

        while len(all_results) < max_results and page_num <= max_pages:
            rows = await self._extract_rows(page)
            remaining = max_results - len(all_results)
            parsed = _parse_result_rows(rows, page.url, remaining)

            new_on_page = 0
            for r in parsed:
                if r["url"].lower() not in seen_urls:
                    seen_urls.add(r["url"].lower())
                    r["index"] = len(all_results) + 1
                    all_results.append(r)
                    new_on_page += 1

            if new_on_page == 0:
                break
            if len(all_results) >= max_results:
                break

            if not await self._goto_next_page(page):
                break
            page_num += 1
            await asyncio.sleep(1)

        for r in all_results:
            try:
                detail = await self._fetch_detail(page, r["url"])
                self._enrich_result(r, detail)
                await asyncio.sleep(0.5)
            except Exception:
                continue

        return {
            "query": query,
            "results": all_results,
            "result_count": len(all_results),
            "location": self._location,
        }


# ---------------------------------------------------------------------------
# CLI entry point
# ---------------------------------------------------------------------------


async def _main(args):
    profile_dir = os.environ.get("CNKI_MCP_PROFILE_DIR")
    searcher = CnkiSearcher(profile_dir=profile_dir)
    try:
        await searcher.start()
        result = await searcher.search(
            query=args.query,
            max_results=args.max_results,
        )
        json.dump(result, sys.stdout, ensure_ascii=False, indent=2)
        sys.stdout.write("\n")
        return 0
    except Exception as exc:
        print(f"[CNKI-SEARCH-ERROR] {exc}", file=sys.stderr)
        json.dump(
            {"query": args.query, "results": [], "result_count": 0, "error": str(exc)},
            sys.stdout, ensure_ascii=False,
        )
        sys.stdout.write("\n")
        return 1
    finally:
        await searcher.stop()


def main():
    parser = argparse.ArgumentParser(description="Search CNKI for Chinese academic literature")
    parser.add_argument("query", help="Search query (Chinese or English)")
    parser.add_argument("--max-results", type=int, default=20, help="Max results (default 20, supports pagination for >20)")
    args = parser.parse_args()
    sys.exit(asyncio.run(_main(args)))


if __name__ == "__main__":
    main()
```

---

## References

- CNKI (China National Knowledge Infrastructure): https://www.cnki.net/
- CNKI Overseas: https://oversea.cnki.net/
