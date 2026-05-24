---
name: youtube-summary
description: >
  Extract subtitles from YouTube or Bilibili, save raw transcript, and generate a structured summary.
  Triggers: ANY URL containing youtube.com, youtu.be, bilibili.com, or b23.tv — use this skill
  immediately, do NOT use fetch-content. Also triggers on: "summarize this video / get subtitles" /
  "总结这个视频" / "帮我看看这个视频" / "获取字幕" / "视频总结" / "提取字幕".
---

# Video Subtitle Extractor (YouTube + Bilibili)

Detect platform → download subtitles → clean → save raw → generate summary.

---

## Step 1 — Ensure yt-dlp is available

```bash
if ! command -v yt-dlp &>/dev/null; then
  echo "yt-dlp not found, installing..."
  pip install -q yt-dlp || pip3 install -q yt-dlp
fi
yt-dlp -U --quiet 2>/dev/null || true
```

If installation fails, stop and tell the user to install yt-dlp manually (`pip install yt-dlp` or `brew install yt-dlp`).

---

## Step 2 — Detect platform and download subtitles

Detect whether the URL is Bilibili or YouTube, then use the appropriate strategy.

```bash
URL="<user-provided URL>"
TMPDIR=$(mktemp -d)
SUB_FILE=""
SUBTITLE_LANG=""

# Detect platform
if echo "$URL" | grep -qE '(bilibili\.com|b23\.tv)'; then
  PLATFORM="bilibili"
  SITE_NAME="Bilibili"
  SITE_DOMAIN="bilibili.com"
else
  PLATFORM="youtube"
  SITE_NAME="YouTube"
  SITE_DOMAIN="youtube.com"
fi
```

### Bilibili branch

Bilibili subtitles require login cookies. Always use a cookies file — refresh from Chrome if missing or stale (>30 days):

```bash
if [ "$PLATFORM" = "bilibili" ]; then
  BILI_COOKIES="${BILIBILI_COOKIES_FILE:-$HOME/bilibili_cookies.txt}"

  NEED_REFRESH=false
  if [ ! -f "$BILI_COOKIES" ]; then
    NEED_REFRESH=true
  elif [ "$(find "$BILI_COOKIES" -mtime +30 2>/dev/null | wc -l | tr -d ' ')" -gt 0 ]; then
    echo "Bilibili cookies older than 30 days, refreshing..."
    NEED_REFRESH=true
  fi

  if [ "$NEED_REFRESH" = true ]; then
    echo "Reading cookies from Chrome (one-time keychain prompt)..."
    yt-dlp --cookies-from-browser chrome --cookies "$BILI_COOKIES" \
      --skip-download -i "https://www.bilibili.com/" 2>/dev/null
  fi

  COOKIE_ARGS="--cookies $BILI_COOKIES"

  # List available subtitle langs — capture stderr to detect login failure
  LIST_OUTPUT=$(yt-dlp --list-subs $COOKIE_ARGS "$URL" 2>&1)
  if echo "$LIST_OUTPUT" | grep -qi "login\|not logged\|需要登录\|please log"; then
    echo ""
    echo "❌ Bilibili cookies expired or invalid."
    echo "   Fix: delete the cookies file and retry — it will re-read from Chrome."
    echo "   rm \"$BILI_COOKIES\""
    rm -rf "$TMPDIR"
    exit 1
  fi
  AVAIL_LANGS=$(echo "$LIST_OUTPUT" | awk '/^[a-z]/{print $1}' | grep -v "^Language$")

  # Try ai-zh first, then any zh variant, then en
  for lang in ai-zh zh-Hans zh-CN zh en; do
    if echo "$AVAIL_LANGS" | grep -q "^${lang}$"; then
      yt-dlp \
        --write-sub \
        --sub-langs "$lang" \
        --skip-download \
        --retries 3 \
        -o "$TMPDIR/bili_%(id)s" \
        $COOKIE_ARGS \
        "$URL" 2>/dev/null
      SUB_FILE=$(find "$TMPDIR" -maxdepth 1 -name "*.${lang}.*" 2>/dev/null | head -1)
      if [ -n "$SUB_FILE" ]; then
        SUBTITLE_LANG="$lang"
        break
      fi
    fi
  done
fi
```

### YouTube branch

```bash
if [ "$PLATFORM" = "youtube" ]; then
  for lang in zh-Hans zh-CN zh en; do
    yt-dlp \
      --write-subs \
      --write-auto-subs \
      --sub-langs "$lang" \
      --skip-download \
      --sub-format vtt \
      --retries 3 \
      --sleep-requests 1 \
      -o "$TMPDIR/yt_%(id)s" \
      "$URL" 2>/dev/null
    SUB_FILE=$(find "$TMPDIR" -maxdepth 1 -name "*.${lang}.vtt" 2>/dev/null | head -1)
    if [ -n "$SUB_FILE" ]; then
      SUBTITLE_LANG="$lang"
      break
    fi
    sleep 1
  done
fi
```

### Fail if no subtitles

```bash
if [ -z "$SUB_FILE" ]; then
  echo "No subtitles found for this video."
  echo "  - No manually uploaded subtitles"
  echo "  - No auto-generated subtitles"
  echo "Cannot proceed without a transcript."
  rm -rf "$TMPDIR"
  exit 1
fi
```

---

## Step 3 — Clean subtitle file → plain text + timestamped text

Produce two outputs from the same subtitle file:

- `cleaned.txt` — no timestamps, for LLM summary input (Steps 7–9)
- `timestamped.txt` — `[mm:ss] text` format, for saving to vault and chapter inference

```bash
EXT="${SUB_FILE##*.}"

python3 - "$SUB_FILE" "$EXT" "$TMPDIR" <<'EOF'
import sys, html, re

sub_file, ext, tmpdir = sys.argv[1], sys.argv[2], sys.argv[3]

def parse_time(t):
    t = t.strip().replace(',', '.')
    parts = t.split(':')
    if len(parts) == 3:
        return int(parts[0]) * 3600 + int(parts[1]) * 60 + float(parts[2])
    return int(parts[0]) * 60 + float(parts[1])

def fmt(secs):
    secs = int(secs)
    return f"[{secs // 60:02d}:{secs % 60:02d}]"

lines = open(sub_file, encoding='utf-8').read().splitlines()

cleaned_out = []
ts_out = []
seen = set()
current_secs = 0

if ext == 'srt':
    i = 0
    while i < len(lines):
        line = lines[i].strip()
        if re.match(r'^\d+$', line):            # sequence number
            i += 1
            continue
        m = re.match(r'^(\d{2}:\d{2}:\d{2}[,\.]\d+) -->', line)
        if m:
            current_secs = parse_time(m.group(1))
            i += 1
            continue
        text = html.unescape(re.sub(r'<[^>]+>', '', line)).strip()
        if text and text not in seen:
            seen.add(text)
            cleaned_out.append(text)
            ts_out.append(f"{fmt(current_secs)} {text}")
        i += 1
else:  # vtt
    i = 0
    while i < len(lines):
        line = lines[i].strip()
        if not line or line.startswith('WEBVTT') or line.startswith('NOTE') \
                or line.startswith('Kind:') or line.startswith('Language:'):
            i += 1
            continue
        m = re.match(r'^([\d:\.]+) -->', line)
        if m:
            current_secs = parse_time(m.group(1))
            i += 1
            continue
        text = html.unescape(re.sub(r'<[^>]+>', '', line)).strip()
        if text and text not in seen:
            seen.add(text)
            cleaned_out.append(text)
            ts_out.append(f"{fmt(current_secs)} {text}")
        i += 1

open(f"{tmpdir}/cleaned.txt", 'w').write('\n\n'.join(cleaned_out) + '\n')
open(f"{tmpdir}/timestamped.txt", 'w').write('  \n'.join(ts_out) + '\n')
EOF
```

---

## Step 4 — Resolve output directory and set filename

```bash
OUTPUT_DIR="${YOUTUBE_SUBTITLES_DIR:-.}"
mkdir -p "$OUTPUT_DIR"
```

Use the original video title as the filename. Only strip characters illegal on macOS (`/` and ASCII `:`); preserve all other characters including fullwidth punctuation (`：`、`《》`、`、`). Truncate to 100 chars:

```bash
SLUG=$(echo "<title>" | python3 -c "
import sys
title = sys.stdin.read().strip()
title = title.replace('/', '').replace(':', '')
print(title[:100])
")
```

---

## Step 5 — Fetch video metadata

Use `$COOKIE_ARGS` (set in Step 2; empty string for YouTube, `--cookies FILE` for Bilibili):

```bash
yt-dlp --dump-json --no-playlist $COOKIE_ARGS "$URL" 2>/dev/null \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
desc = d.get('description','')
first_para = desc.split('\n\n')[0].replace('\n',' ')[:300]
chapters = d.get('chapters') or []
chapter_lines = '\n'.join(f'  - \"{int(c[\"start_time\"]//60)}:{int(c[\"start_time\"]%60):02d} {c[\"title\"]}\"' for c in chapters)
cats = d.get('categories') or []
print('TITLE:', d.get('title',''))
print('CHANNEL:', d.get('uploader',''))
print('DURATION:', d.get('duration_string',''))
print('DATE:', d.get('upload_date',''))
print('DESCRIPTION:', first_para)
print('CATEGORY:', cats[0] if cats else '')
print('CHAPTERS:')
print(chapter_lines)
"
```

---

## Step 6 — Save raw transcript

Write `$OUTPUT_DIR/$SLUG.md`:

```bash
NOW=$(date +"%Y-%m-%dT%H:%M")
WORDS=$(wc -w < "$TMPDIR/timestamped.txt" | tr -d ' ')
```

```markdown
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

<full timestamped transcript — contents of $TMPDIR/timestamped.txt>
```

---

## Step 7 — Validate transcript and check length

```bash
CHARS=$(wc -c < "$TMPDIR/cleaned.txt" | tr -d ' ')
```

Validate before proceeding:

- If `$CHARS < 100`: the subtitle file was empty or contained only formatting. Stop immediately and tell the user: "No readable transcript content found — the video may have subtitles disabled, or the downloaded file was empty."
- If the transcript language does not match any expected language (e.g. got `en` when `zh` was requested): note the actual language to the user and continue with what was fetched.

Length routing:

- `≤ 120000`: use full cleaned text as summary input directly.
- `> 120000`: run map-reduce first (Step 8), then use the combined bullet points as summary input.

---

## Step 8 — Map-reduce for long transcripts (> 120k only)

Split into overlapping chunks (~40,000 chars with 2,000-char overlap) to prevent context loss at boundaries:

```bash
python3 - <<'EOF'
text = open("$TMPDIR/cleaned.txt").read()
size = 40000
overlap = 2000
step = size - overlap
chunks = [text[i:i+size] for i in range(0, len(text), step)]
for i, chunk in enumerate(chunks):
    print(f"=== CHUNK {i+1}/{len(chunks)} ===")
    print(chunk)
EOF
```

For each chunk, extract structured notes in this format:

```
TOPIC: [inferred topic name for this chunk, e.g. "AI bubble assessment"]
- [key claim or fact — 2 sentences: what was said + supporting detail]
- [key claim or fact — 2 sentences]
...
QUOTES: [1–3 verbatim lines worth preserving]
DATA: [any specific numbers or metrics]
```

Extract 8–12 entries per chunk. Collect all structured notes, grouped by TOPIC, as the summary input for Step 9. Merge notes under the same topic across chunks before passing to Step 9.

---

## Step 9 — Generate summary

Resolve the summary language:

```bash
if [ -n "$YOUTUBE_SUBTITLES_SUMMARY_LANG" ]; then
  SUMMARY_LANG="$YOUTUBE_SUBTITLES_SUMMARY_LANG"
elif [[ "$SUBTITLE_LANG" == zh* ]]; then
  SUMMARY_LANG="zh"
else
  _SYS_LANG="${LANG:-${LANGUAGE:-}}"
  case "$_SYS_LANG" in
    zh*) SUMMARY_LANG="zh" ;;
    *)   SUMMARY_LANG="en" ;;
  esac
fi
```

Read `references/output-formats.md` for exact section headers, frontmatter templates, and format examples before generating the summary.

Using the summary input from Step 7 or 8, generate the summary body **in order** (omit any section where the source has no relevant content):

1. **Chapter Overview / 章节概览**
2. **Overall Summary / 总体摘要**
3. **Topic Chapters / 话题章节**
4. **Key Quotes / 关键引用**
5. **Novel Ideas / 新颖观点**
6. **Counter-intuitive Views / 反直觉观点**
7. **Core Tensions / 核心张力**
8. **Methodology / 方法论**
9. **Key Data / 关键数据**

### Chapter Overview / 章节概览

This is always the first section. Generate a timestamped navigation table:

- **If chapters are present in metadata** (from Step 5): use the actual chapter timestamps and titles directly. Add a one-line "What happens" description for each chapter inferred from the transcript.
- **If no chapters in metadata**: infer 5–8 chapter breaks from topic shifts in the transcript. Use timestamps from the VTT/SRT file to assign accurate time codes.

See `references/output-formats.md` for the exact table format in both `en` and `zh`.

Write the summary body to a temp file first:

```bash
cat > "$TMPDIR/summary_body.md" <<'SUMMARY_EOF'
<generated summary sections here>
SUMMARY_EOF
SUMMARY_WORDS=$(wc -w < "$TMPDIR/summary_body.md" | tr -d ' ')
```

Then write the final file `$OUTPUT_DIR/$SLUG-summary.md` by combining frontmatter + body. Use the summary frontmatter template from `references/output-formats.md`:

```bash
cat > "$OUTPUT_DIR/$SLUG-summary.md" <<EOF
<frontmatter here>
EOF
cat "$TMPDIR/summary_body.md" >> "$OUTPUT_DIR/$SLUG-summary.md"
```

### Voice and attribution

Write content directly. Never prefix bullets or sentences with attribution phrases like "X says", "X believes", "X points out", "X argues", "according to X", or their equivalents in any language. The speaker is already identified in the frontmatter — repeating their name before every claim is noise. State the idea itself.

Bad: "The speaker believes API businesses have no moat because users have zero loyalty."
Good: "API businesses have no moat — users switch instantly to any cheaper or better model."

### Depth requirement (applies to ALL bullet points across ALL sections)

Every bullet point must be **2–3 sentences minimum**:
1. **Claim**: state the specific view or fact
2. **Reasoning / evidence**: what reasoning, example, or data supports it (quote the transcript if helpful)
3. **Nuance or implication**: a caveat, consequence, or "so what" the reader needs

One-sentence bullets are not acceptable. If you cannot write 2–3 sentences about a point, the point is too thin to include — omit it rather than padding.

### Content rules per section

- **Overall Summary / 总体摘要**: Apply Chain of Density to produce this section — do NOT write it in one shot.

  **Step A — Compute target word count:**
  ```
  TARGET = max(80, min(TRANSCRIPT_WORDS // 25, 400))
  ```
  Examples: 1 000-word transcript → 80 w; 5 000 w → 200 w; 10 000 w → 400 w (cap).

  **Step B — Iteration 0 (broad draft):**
  Write a verbose, filler-heavy summary at exactly TARGET words. Use phrases like "this video discusses", "the speaker talks about". This is intentionally weak — it sets the baseline.

  **Step C — Iterations 1–3 (density rounds):**
  For each round:
  1. **Identify 1–3 Missing Entities** — concepts, names, numbers, or claims that are relevant, specific (≤5 words), and absent from the current summary.
  2. **Rewrite at the same TARGET word count**, incorporating missing entities through fusion and compression. Remove filler phrases ("this video discusses", "the speaker explains") to make room. Never drop an entity from a previous round.

  **Step D — Output:**
  - The Round 3 summary becomes the **Overall Summary paragraph**.
  - The Missing Entities collected across all 3 rounds become the **"最值得关注的几个点 / Key Highlights:"** bullet list (expand each entity to 2–3 sentences per the depth requirement above).

- **Topic Chapters / 话题章节**: Identify 4–8 major topics discussed. For each, write a `### [Topic Name]` subsection with 3–5 bullets at the 2–3 sentence depth. Topics should reflect the actual conversation flow (e.g. "OpenAI strategy", "AI bubble assessment", "China vs US", "investment philosophy"). Cover all significant topics — do not drop a topic because it seems minor. Even brief or personal moments (a TV show analogy, a hobby mentioned) deserve a short subsection if they carry a meaningful idea.

- **Key Quotes / 关键引用**: Verbatim quotes from the transcript as blockquotes. Aim for 10–15 quotes; prefer ones that are vivid, specific, or capture a stance in the speaker's own words. After each blockquote, add one sentence of context (who said it, in what context).

- **Novel Ideas / 新颖观点**: Ideas that are fresh, uncommon, or reframe a familiar concept. Each bullet: state the idea (2–3 sentences including the reasoning behind it). Do not include ideas that are already mainstream.

- **Counter-intuitive Views / 反直觉观点**: Claims that contradict common belief. Format each bullet as: **Common belief**: [X] → **Actual claim**: [Y] — then 1–2 sentences explaining what makes Y non-obvious and what evidence supports it.

- **Core Tensions / 核心张力**: Opposing forces, unresolved debates, or structural contradictions. Each tension: name both sides (2–3 sentences each), then note whether the speaker resolves it or leaves it open.

- **Methodology / 方法论**: Frameworks, decision processes, heuristics, or step-by-step approaches. Each entry: describe the framework (what it is, how to apply it, what problem it solves). Sub-bullets are encouraged for multi-step processes.

- **Key Data / 关键数据**: Specific numbers, statistics, metrics, named comparisons. Include the source context (who said it, about what). Do not fabricate numbers — only include figures explicitly stated in the transcript.

Omit any section where the source has no relevant content. Do not fabricate.

---

## Step 10 — Clean up and report

```bash
rm -rf "$TMPDIR"
```

```
Raw:     $OUTPUT_DIR/$SLUG.md
Summary: $OUTPUT_DIR/$SLUG-summary.md
```

---

## Error Handling

| Error | Action |
|-------|--------|
| `yt-dlp` not found, install fails | Stop. Tell user to install manually: `pip install yt-dlp` or `brew install yt-dlp`. |
| Bilibili cookies missing or stale | Auto-refresh from Chrome. If Chrome read fails, tell user to run `yt-dlp --cookies-from-browser chrome --cookies ~/bilibili_cookies.txt --skip-download -i https://www.bilibili.com/` manually. |
| Bilibili login error in `--list-subs` output | Stop. Print the cookie file path and the `rm` command to delete it. Tell user to retry — deletion triggers a fresh Chrome read. |
| No subtitles found (YouTube or Bilibili) | Stop. Tell user: no manual or auto-generated subtitles are available. Suggest checking the video page directly. |
| Subtitle file downloaded but `cleaned.txt` < 100 chars | Stop. Tell user the file was empty or contained only formatting — subtitles may be disabled or the download silently failed. |
| Language mismatch (requested `zh`, got `en`) | Continue. Note the actual language to the user before presenting the summary. |
| `yt-dlp --dump-json` fails (metadata step) | Continue without metadata. Use empty strings for title, channel, description; skip chapters. Note to user that metadata was unavailable. |
| Output directory does not exist | `mkdir -p` creates it automatically. No user action needed. |
| Transcript > 120k chars, map-reduce produces empty notes | Stop. Tell user the transcript may be in an unsupported format or language. Show the first 200 chars of `cleaned.txt` for diagnosis. |
