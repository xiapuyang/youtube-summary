# Summary Output Format Templates

Reference this file when generating video summaries. Use section headers and formats matching the target language.

---

## Section Headers

| Section | English | Chinese |
|---------|---------|---------|
| Chapter Overview | `## Chapter Overview` | `## 章节概览` |
| Overall Summary | `## Overall Summary` | `## 总体摘要` |
| Topic Chapters | `## Topic Chapters` | `## 话题章节` |
| Key Quotes | `## Key Quotes` | `## 关键引用` |
| Novel Ideas | `## Novel Ideas` | `## 新颖观点` |
| Counter-intuitive Views | `## Counter-intuitive Views` | `## 反直觉观点` |
| Core Tensions | `## Core Tensions` | `## 核心张力` |
| Methodology | `## Methodology` | `## 方法论` |
| Key Data | `## Key Data` | `## 关键数据` |

---

## Chapter Overview  /  章节概览

Quick navigation table — the first section in every summary. Use actual chapter timestamps and titles from video metadata if available; otherwise infer 5–8 chapter breaks from topic shifts in the transcript, using VTT/SRT timestamps for accuracy.

**English:**

```
## Chapter Overview

| Time | Chapter | What happens |
|------|---------|--------------|
| 00:00 | Introduction | Host frames the core problem and what the video will cover |
| 05:30 | Background | Reviews prior work; explains why existing solutions fall short |
| 12:20 | Core Method | Walks through the proposed approach step by step |
| 24:10 | Results | Benchmark comparisons and key findings |
| 31:55 | Q&A | Audience questions on scalability and next steps |
```

**Chinese:**

```
## 章节概览

| 时间 | 章节 | 内容概述 |
|------|------|---------|
| 00:00 | 引言 | 主讲人介绍核心问题与视频内容框架 |
| 05:30 | 背景 | 回顾已有研究，解释现有方案的局限性 |
| 12:20 | 核心方法 | 逐步讲解所提方案的实现过程 |
| 24:10 | 结果 | 基准测试对比与核心发现 |
| 31:55 | Q&A | 观众就扩展性与后续工作的提问 |
```

---

## Overall Summary  /  总体摘要

3–5 sentence synthesis of the full content, followed by **Key Highlights / 最值得关注的几个点**: 3–5 bullets (2–3 sentences each).

**English:**

```
## Overall Summary

[3–5 sentences covering the video's main arc, central argument, and conclusion.]

**Key Highlights:**
- [Most surprising or high-value insight — 2–3 sentences with evidence or reasoning]
- [Second highlight]
- [Third highlight]
```

**Chinese:**

```
## 总体摘要

[3–5 句话，覆盖视频主线、核心论点与结论。]

**最值得关注的几个点：**
- [最出人意料或最有价值的洞察 — 2–3 句，含论据或推理]
- [第二个要点]
- [第三个要点]
```

---

## Topic Chapters  /  话题章节

4–8 major topics, each as `### [Topic Name]` with 3–5 bullets at 2–3 sentence depth. Reflect actual conversation flow.

```
## Topic Chapters

### [Topic Name]
- [Claim — 2 sentences: what was said + supporting detail]
- [Claim — 2–3 sentences including nuance or implication]
```

---

## Key Quotes  /  关键引用

10–15 verbatim quotes as blockquotes, each followed by one sentence of context.

```
## Key Quotes

> "[Verbatim quote from transcript]"

[One sentence: who said it and in what context.]
```

---

## Novel Ideas  /  新颖观点

```
## Novel Ideas

- [Idea name in bold] **[Name]** — [2–3 sentences: what the idea is, the reasoning behind it, why it's non-mainstream]
```

---

## Counter-intuitive Views  /  反直觉观点

```
## Counter-intuitive Views

- **Common belief**: [X] → **Actual claim**: [Y] — [1–2 sentences on what makes Y non-obvious and what evidence supports it]
```

---

## Core Tensions  /  核心张力

```
## Core Tensions

**[Tension name]**
- Side A: [2–3 sentences]
- Side B: [2–3 sentences]
- Resolution: [whether the speaker resolves it or leaves it open]
```

---

## Methodology  /  方法论

```
## Methodology

- **[Framework name]**: [What it is, how to apply it, what problem it solves]
  - Step 1: ...
  - Step 2: ...
```

---

## Key Data  /  关键数据

```
## Key Data

- [Specific number or metric] — context: [who said it, about what topic]
```

Only figures explicitly stated in the transcript. Never fabricate.

---

## Frontmatter Templates

### Raw Transcript File (`$SLUG.md`)

```yaml
---
title: "<title>"
source: "<URL>"
author:
  - "[[<channel>]]"
published: "<YYYYMMDD>"
description: "<DESCRIPTION>"
tags:
  - "<PLATFORM>"
ctime: "<NOW>"
mtime: "<NOW>"
words: "<WORDS>"
site: "<SITE_NAME>"
domain: "<SITE_DOMAIN>"
channel: "<channel>"
duration: "<duration>"
category: "<CATEGORY>"
subtitle_lang: "<SUBTITLE_LANG>"
chapters:
<CHAPTERS or empty>
type: "source"
---
```

### Summary File (`$SLUG-summary.md`)

```yaml
---
title: "<title>"
source: "<URL>"
author:
published: "<YYYYMMDD>"
description: "<DESCRIPTION>"
tags:
  - "<PLATFORM>"
ctime: "<NOW>"
mtime: "<NOW>"
words: "<SUMMARY_WORDS>"
site: "<SITE_NAME>"
domain: "<SITE_DOMAIN>"
channel: "<channel>"
duration: "<duration>"
category: "<CATEGORY>"
subtitle_lang: "<SUBTITLE_LANG>"
chapters:
<CHAPTERS or empty>
type: "source"
lang: "<SUMMARY_LANG>"
---
```
