# youtube-summary · 视频字幕提取 + 结构化总结

> 一个 [Claude Code](https://docs.claude.com/en/docs/claude-code) Skill：丢给它一条 YouTube 或 Bilibili 链接，它自动拉字幕、存一份带时间戳的原始转录，再生成一份结构化的深度总结。

把链接粘进对话即可触发，无需记命令。流程：**识别平台 → 下载字幕 → 清洗 → 存原始转录 → 生成总结**。

---

## 它做什么

- **双平台**：YouTube（自动/人工字幕）+ Bilibili（需登录 cookies，自动从 Chrome 读取并缓存）。
- **两份产物**：`<标题>.md`（带 `[mm:ss]` 时间戳的完整转录）+ `<标题>-summary.md`（结构化总结）。
- **9 段式总结**：章节概览、总体摘要、话题章节、关键引用、新颖观点、反直觉观点、核心张力、方法论、关键数据——无内容的段落自动省略，不编造。
- **长视频不丢信息**：转录 > 120k 字符时先做 map-reduce（40k 字符分块 + 2k 重叠），再汇总。
- **Chain of Density 摘要**：「总体摘要」按转录长度自适应字数（`max(80, min(words/25, 400))`），经 3 轮密度迭代，而非一次成稿。
- **离线确定性**：所有逻辑是 `SKILL.md` 里的内联 bash + python，不依赖额外服务。

---

## 前置依赖

- [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) — 字幕下载核心。Skill 会在缺失时尝试 `pip install yt-dlp`，失败则提示你手动装（`pip install yt-dlp` 或 `brew install yt-dlp`）。
- **Bilibili 专属**：需要登录态 cookies。Skill 首次会用 `yt-dlp --cookies-from-browser chrome` 从 Chrome 读取并缓存到 `~/bilibili_cookies.txt`（30 天过期自动刷新）。请确保 Chrome 已登录 B 站。
- `python3`（清洗字幕 / 解析元数据 / 分块，均为标准库）。

---

## 安装

以 Claude Code Skill 形式使用，放到 `~/.claude/skills/` 下即可被自动发现：

```bash
git clone https://github.com/xiapuyang/youtube-summary.git ~/.claude/skills/youtube-summary
```

装好后重启 Claude Code，确认 skill 已加载。

---

## 使用

直接把链接发给 Claude，或用自然语言触发：

```
帮我总结这个视频 https://www.youtube.com/watch?v=xxxx
```

```
https://www.bilibili.com/video/BVxxxx  获取字幕
```

触发词：任何含 `youtube.com / youtu.be / bilibili.com / b23.tv` 的链接，或「总结这个视频」「获取字幕」「视频总结」「提取字幕」。

---

## 环境变量（可选）

| 变量 | 作用 | 默认 |
|---|---|---|
| `YOUTUBE_SUBTITLES_DIR` | 产物输出目录 | 当前目录 `.` |
| `YOUTUBE_SUBTITLES_SUMMARY_LANG` | 强制总结语言（`zh` / `en`） | 跟随字幕语言，否则跟随系统 `LANG` |
| `BILIBILI_COOKIES_FILE` | Bilibili cookies 缓存路径 | `~/bilibili_cookies.txt` |

---

## 输出格式

产物 frontmatter 采用 **Obsidian 风格**（`[[作者]]` wikilink、`ctime/mtime`、`type: source`、`tags`），方便直接落进 Obsidian 仓库。若你不用 Obsidian，这些字段无害，可按需删改 `references/output-formats.md` 里的模板。

段落标题与模板（中/英双语）见 [`references/output-formats.md`](references/output-formats.md)。

---

## License

[MIT](LICENSE) © 2026 xiapuyang
