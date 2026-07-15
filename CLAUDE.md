# CLAUDE.md

## Commands

```bash
# 安装 pre-commit hooks（首次，可选）
pre-commit install

# 提交前自查
pre-commit run --all-files

# 手动冒烟测试（不经 Claude，直接跑底层下载）
yt-dlp --list-subs "<URL>"                       # 看有哪些字幕轨
yt-dlp --write-auto-subs --sub-langs zh-Hans \
  --skip-download -o "/tmp/test_%(id)s" "<URL>"  # 拉一条字幕验证
```

> 本 skill 是 **prompt 驱动**，没有可执行入口脚本——真正的流程编排在 `SKILL.md` 的 10 个 Step 里。要验证完整流程，把链接发给 Claude Code 触发 skill 即可。

## Architecture

整个 skill 就两个文件，逻辑全部内联：

```
SKILL.md                      # ← 真理来源：10 个 Step 的完整流程
                              #   Step 1 装 yt-dlp · 2 识别平台+下字幕 · 3 清洗成
                              #   cleaned.txt/timestamped.txt · 4 文件名 · 5 元数据 ·
                              #   6 存原始转录 · 7 长度路由 · 8 map-reduce · 9 生成总结 ·
                              #   10 清理。bash + python3 heredoc 全内联，无独立模块。
references/
  output-formats.md           # 9 段总结的标题/模板 + frontmatter 模板（中英双语）
```

## Key design constraints

读这些**反直觉**约定，否则容易改坏：

- **小宇宙走完全不同的路**：xiaoyuzhou 没有字幕轨，跳过 yt-dlp，改为 curl 下 m4a → ffmpeg 转 wav → sherpa-onnx SenseVoice 转录，直接产出 `cleaned.txt`/`timestamped.txt`，Step 3 对 xiaoyuzhou 会被跳过（`SUB_FILE` 为空是预期行为，不是 bug）。
- **sherpa-onnx 复用 Type4Me 模型**：模型路径硬编码为 `~/Library/Application Support/Type4Me/models/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-int8-2024-07-17/model.int8.onnx`，无需额外下载。只需 `pip install sherpa-onnx`（约 50MB）。Whisper 系列（openai-whisper / faster-whisper）是 fallback。
- **元数据来自页面 JSON 而非 yt-dlp**：小宇宙的 title/duration/description 等从页面 `__NEXT_DATA__` 提取并写入 `$TMPDIR/xyz_meta.json`，Step 5 直接读这个文件。改元数据字段就改 Step 2 的 `xyz_meta.json` 构建逻辑。

- **没有 .py 文件可改**：所有处理逻辑（字幕清洗、元数据解析、分块）都是 `SKILL.md` 里的内联 python heredoc。改行为 = 改 SKILL.md，别去找模块。
- **Bilibili 必须带 cookies，YouTube 不带**：`$COOKIE_ARGS` 在 Step 2 设定（B 站为 `--cookies FILE`，YT 为空串），后续 Step 5 元数据复用同一变量。漏带 cookies → B 站 `--list-subs` 报登录错。
- **字幕语言优先级是写死的回退链**：Bilibili `ai-zh → zh-Hans → zh-CN → zh → en`；YouTube `zh-Hans → zh-CN → zh → en`。命中第一条即停。
- **长度路由阈值 120000 字符**：`cleaned.txt ≤ 120k` 直接喂总结；`> 120k` 先 Step 8 map-reduce（40k 块 + 2k 重叠）再汇总，否则丢边界信息。
- **「总体摘要」走 Chain of Density，不准一次成稿**：先按 `TARGET = max(80, min(words/25, 400))` 定字数，再 3 轮密度迭代。改总结质量先盯这条。
- **文件名只剥 `/` 和 ASCII `:`**：保留全角标点（`：《》、`），截断 100 字符。别顺手"清理"成 ASCII，会改变落盘文件名。
- **产物 frontmatter 是 Obsidian 风格，刻意保留**（`[[wikilink]]`、`ctime/mtime`、`type: source`）。这是设计选择不是遗留，改模板去 `references/output-formats.md`。
- **总结正文禁止归因前缀**：不要写「X 说 / X 认为 / 据 X」，说观点本身——讲者已在 frontmatter 标注，重复即噪音。

## Development Workflow

- 改动走 feature 分支 + PR（`.pre-commit-config.yaml` 里 `no-commit-to-branch` 守 main）。
- bootstrap / 首次 commit 场景可 `--no-verify`，但需审批。
- 改了流程后，用一条真实 YouTube + 一条真实 Bilibili 链接各跑一遍，确认两份产物正常落盘。

---

# Karpathy Guidelines

Behavioral guidelines to reduce common LLM coding mistakes, derived from [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- If you write 200 lines and it could be 50, rewrite it.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

- Don't "improve" adjacent code, comments, or formatting.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.
