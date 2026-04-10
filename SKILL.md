---
name: deep-research
description: "Deep research agent that conducts thorough investigations on any topic and saves structured results to local files. Use when the user says '调研xxx', '调研一下xxx', 'research xxx', '帮我调研', '深度调研', '简单调研', or any request to investigate/research a topic and produce a written report. Also use when the user asks follow-up questions about a previous research topic (e.g., '刚才那个调研，xxx怎么样？', '再深入看看xxx', '补充一下xxx的部分'), to continue researching and update the existing research files. Saves all research output to a dedicated directory under ./research/ (configurable). Supports both quick overviews and deep multi-round investigations."
---

# Deep Research

Conduct research on a given topic using web search, analyze findings, and save structured results to a dedicated local directory.

## Workflow

Determine whether this is a **new research** or a **follow-up** on an existing one:

**New research?** → Follow "New Research" workflow
**Follow-up on existing research?** → Follow "Follow-up" workflow

### New Research

1. **Parse request** — Extract the research topic, desired depth, language, and output path.
2. **Resolve output directory** — **CRITICAL: Always use the PROJECT ROOT as the base, NOT the current working directory.** Run `git rev-parse --show-toplevel 2>/dev/null || pwd` to reliably find the project root. The output path is `<PROJECT_ROOT>/research/<topic-slug>/`. NEVER use `pwd` alone — it may return a subdirectory (e.g., `.claude/skills/`). NEVER write to `~/research/` or `/Users/<user>/research/` or any path inside `.claude/`. Slugify the topic into a short, filesystem-safe directory name (lowercase, hyphens, no spaces). If the directory already exists, append a numeric suffix.
3. **Research** — Conduct multi-round web searches, fetch key pages, cross-reference findings.
4. **Write report** — Save results as Markdown files in the output directory.
5. **Render Mermaid diagrams** — If report.md contains ` ```mermaid ` code blocks, render them to images for cross-platform compatibility. See "Mermaid 图表渲染" section below for details.
6. **Upload rendered images to R2** — If SVG files were generated and R2 is configured, upload them and replace local paths with online URLs. See "图片上传到 R2 图床" section below for details.
7. **Summarize to user** — Print a brief summary and the path to the output directory. If rendered version was generated, mention both `report.md` (original) and `report-rendered.md` (cross-platform with online image URLs).

### Follow-up

When the user asks follow-up questions or requests deeper investigation on a previous research topic:

1. **Locate existing research** — Find the relevant research directory under `./research/` (or the custom path used). Read the existing `report.md` to understand current findings.
2. **Identify the gap** — Determine what the user wants: a specific question answered, a subtopic expanded, new angles explored, or corrections made.
3. **Research the gap** — Conduct additional web searches focused on the follow-up question.
4. **Update existing files** — Integrate new findings into the existing research materials:
   - **Answering a question**: Add a new section to `report.md` or expand the relevant existing section. Add a `## Follow-up: <question summary>` section at the end if it doesn't fit naturally into the existing structure.
   - **Expanding a subtopic**: Add or update a file in `notes/` for the subtopic, and add a cross-reference in `report.md`.
   - **Correcting information**: Edit the relevant section in-place, adding a note about the correction.
   - **Broad continuation**: Expand existing sections in `report.md` and update `sources.md` with new references.
5. **Update metadata** — Add a `> Last updated: YYYY-MM-DD` line below the research date in `report.md`.
6. **Summarize to user** — Briefly answer the user's question and indicate which files were updated.

## Depth Levels

| Keyword | Behavior |
|---|---|
| 深度调研 / deep research | Multiple search rounds, cross-validation, comprehensive report (default) |
| 简单调研 / quick research | 2-3 searches, concise summary |

Default to **deep research** unless the user explicitly requests a quick/simple version.

## Language

Match the language of the user's request. Chinese prompt → Chinese report. English prompt → English report.

## Output Structure

Adapt file structure to topic complexity:

**Simple topic** (single concept, narrow scope):
```
research/<topic-slug>/
└── report.md
```

**Complex topic** (multi-faceted, comparative, broad):
```
research/<topic-slug>/
├── report.md          # Main report with key findings
├── sources.md         # All referenced URLs with brief descriptions
└── notes/             # Optional: per-subtopic deep dives
    ├── subtopic-1.md
    └── subtopic-2.md
```

Always create `report.md`. Create `sources.md` and `notes/` only when the research is substantive enough to warrant them.

When Mermaid diagrams are present, the rendering + upload steps also produce:
```
research/<topic-slug>/
├── report.md              # Original (Mermaid source code — GitHub/GitLab/Notion friendly)
├── report-rendered.md     # Rendered (Mermaid → PNG images with online URLs — works everywhere)
├── report-rendered-1.png  # Rendered diagram images (local copies)
├── report-rendered-2.png
├── sources.md
└── notes/
    ├── subtopic-1.md
    └── subtopic-1-rendered.md  # If notes also contain Mermaid
```

If R2 is configured, `report-rendered.md` will contain online URLs like:
```
![diagram](${R2_WORKER_URL}/research/<topic-slug>/report-rendered-1.png)
```
If R2 is not configured, it will contain local relative paths (`./report-rendered-1.png`).

## Report Format (report.md)

```markdown
# <Research Topic>

> Research date: YYYY-MM-DD
> Depth: deep | quick

## TL;DR
[2-3 sentence executive summary]

## Key Findings
[Main body organized by logical sections]
[Use Mermaid diagrams where appropriate — see "Mermaid 图表规范" section below]

## Architecture / Diagrams
[Optional: Mermaid diagrams for system architecture, data flow, component relationships, etc.]

## Comparison / Analysis
[If applicable: tables, pros/cons, trade-offs]

## Conclusion
[Actionable takeaways]

## Sources
[Numbered list of key references — or "See sources.md for full list"]
```

## Directory Slug Rules

- Lowercase, hyphen-separated, max 50 chars
- Strip articles and filler words
- Examples: "调研一下 AI Agent 框架" → `ai-agent-frameworks`, "Research React vs Vue in 2025" → `react-vs-vue-2025`

## Output Path Rules

**Default behavior**: All output goes to `<PROJECT_ROOT>/research/<topic-slug>/`.

**How to find PROJECT_ROOT**: Run this command FIRST, before any file writes:
```bash
git rev-parse --show-toplevel 2>/dev/null || pwd
```
This returns the git repository root. Only falls back to `pwd` if not in a git repo. Store this path and use it as the base for ALL file operations.

**Custom path**: If the user specifies a path (e.g., "调研xxx，放到 ~/notes/"), use that path instead. Still create a topic subdirectory under the specified path.

**Common mistakes to avoid**:
- Do NOT use `./research/` in Write/Edit tool calls — the tool requires absolute paths.
- Do NOT trust `pwd` alone — it may return a subdirectory like `.claude/skills/` instead of the project root.
- Do NOT write files under `.claude/`, `node_modules/`, or any non-project directory.
- Always expand to the full absolute path (e.g., `/Users/xxx/Documents/projects/myproject/research/<topic-slug>/`).

## Mermaid 图表规范

当调研内容涉及系统架构、数据流、组件关系、时序交互、状态机、技术演进等场景时，**必须使用 Mermaid 图表**辅助说明，而非纯文字或 ASCII art。

> 详细的图表类型选择、语法规范、质量要求和常用模式，见 **[mermaid-guide.md](./mermaid-guide.md)**。写报告前请先阅读该文件。

**核心原则**：一张好的 Mermaid 图 > 三段文字描述。但不要为画图而画图——简单的列表/表格能说清楚的，不必强行上图。

## Mermaid 图表渲染（跨平台兼容）

很多平台（微信公众号、知乎、飞书、小红书、Medium 等）不支持 Mermaid 语法。报告写完后**必须尝试渲染为图片版本**。

### 渲染流程

写完所有 `.md` 文件后，执行以下步骤：

1. **检查是否需要渲染** — `grep -l '```mermaid' <output_dir>/*.md` 查找包含 mermaid 的文件
2. **如不包含**，跳过渲染步骤
3. **如包含**，对每个含 mermaid 的 `.md` 文件运行：
   ```bash
   npx -y -p @mermaid-js/mermaid-cli mmdc -i <file>.md -o <file>-rendered.md -e png -t default -b white
   ```
   > 使用 `npx -y` 自动下载执行，无需用户预装 mmdc。首次运行会自动安装。
   > 使用 `-e png` 输出 PNG 格式（而非 SVG），确保知乎、微信公众号等所有平台兼容。`-b white` 设置白色背景，避免 PNG 透明底在深色模式下不可读。
4. **mmdc 会自动**：
   - 将每个 mermaid 代码块渲染为 PNG 文件（`<file>-rendered-1.png`, `-2.png`, ...）
   - 生成 `<file>-rendered.md`，代码块替换为 `![diagram](./<file>-rendered-1.png)` 图片引用
5. **对 notes/ 下的文件也执行同样操作**（如果 notes/ 目录存在且文件含 mermaid）
6. **如果 npx/mmdc 执行失败**（网络问题、Node 版本不兼容等），**不阻塞流程**——跳过渲染，在总结中提示用户手动渲染

### 失败时的处理

如果渲染步骤失败或超时，在最终总结中追加提示：

```
💡 报告中包含 Mermaid 图表，但自动渲染未成功。如需上传到不支持 Mermaid 的平台，请手动渲染：
   npx -p @mermaid-js/mermaid-cli mmdc -i report.md -o report-rendered.md
```

### 总结时的输出说明

当渲染成功时，在总结中说明两个版本的用途：
- `report.md` — 原始版本，含 Mermaid 源码（GitHub / GitLab / Notion 直接用）
- `report-rendered.md` — 渲染版本，图表已转为 SVG 图片（微信公众号 / 知乎 / 飞书等所有平台通用）

## 图片上传到 R2 图床

渲染出的 SVG 文件是本地路径，分享到其他平台时图片无法显示。如果配置了 R2 图床，**必须自动上传并替换路径为线上 URL**。

### 前置条件

需要两个环境变量（与 paper-reader skill 共用）：
- `R2_WORKER_URL` — R2 Worker 的 base URL（例如 `https://img.example.com`）
- `R2_API_KEY` — API 认证密钥

### 上传流程

在 Mermaid 渲染成功后，执行以下步骤：

1. **检查 R2 配置** — 检查 `R2_WORKER_URL` 和 `R2_API_KEY` 环境变量是否存在
2. **如未配置**，跳过上传，本地 SVG 路径保留。在总结中提示用户配置方法
3. **如已配置**，对每个渲染出的 SVG 文件：
   ```bash
   # 上传路径格式：research/<topic-slug>/<filename>
   curl -X PUT "${R2_WORKER_URL}/research/<topic-slug>/<filename>.png" \
     -H "X-API-Key: ${R2_API_KEY}" \
     -H "Content-Type: image/png" \
     --data-binary @<local-png-file>
   ```
4. **替换 `report-rendered.md` 中的本地路径**：
   - 原：`![diagram](./report-rendered-1.png)`
   - 替换为：`![diagram](${R2_WORKER_URL}/research/<topic-slug>/report-rendered-1.png)`
5. **对 notes/ 下的 rendered 文件也执行同样操作**

### R2 未配置时的处理

在总结中追加提示：

```
💡 SVG 图片为本地路径，分享到其他平台时图片无法显示。如需上传到图床，请配置：
   export R2_WORKER_URL=https://your-r2-worker-url
   export R2_API_KEY=your-api-key
```

### 上传失败时的处理

单个文件上传失败不阻塞流程。在总结中报告哪些文件上传成功、哪些失败，保留失败文件的本地路径。

## Research Process Guidelines

### Web Search 策略：优先使用 web-access skill

**所有联网操作（搜索、网页抓取、内容提取）必须优先通过 `web-access` skill 进行。** 在开始研究前，必须先加载 web-access skill 并完成前置检查（CDP Proxy 启动）。

**web-access skill 是唯一的联网方式，不允许自行降级。** 通过 CDP 浏览器直连，能处理反爬、动态渲染、登录态等复杂场景，信息获取最完整可靠。具体包括：
- 搜索：通过 CDP 在搜索引擎页面中搜索，获取更完整的结果
- 网页抓取：通过 CDP 打开目标页面，用 `/eval` 提取内容，能处理 JS 渲染、反爬限制
- 深度探索：在页面内点击、翻页、跳转，像人一样浏览获取信息

**禁止自行降级到 WebSearch / WebFetch。** 这些工具只有在用户明确允许降级时才可使用。

**实际操作流程：**
1. 先运行 `bash ${CLAUDE_SKILL_DIR}/../web-access/scripts/check-deps.sh` 确认 CDP 可用
2. CDP 可用 → 所有搜索和页面抓取通过 web-access skill 的 CDP Proxy API 完成
3. CDP 不可用 → **停下来，告知用户 CDP 连接失败，并给出修复建议**（重启 Proxy、检查 Chrome 远程调试设置等）。等待用户确认修复后再继续，或用户明确说「可以用 WebSearch 降级」时才允许使用 WebSearch / WebFetch

**使用 web-access skill 进行研究时的要点：**
- 搜索时通过 CDP 在 Google/Bing 等搜索引擎中搜索，提取搜索结果链接
- 逐个在新 tab 中打开关键链接，用 `/eval` 提取正文内容
- 任务结束后关闭自己创建的 tab，保持环境整洁
- 对于需要多个独立调研目标的任务，参照 web-access skill 的「子 Agent 分治策略」并行执行
- 子 Agent prompt 中必须写明 `必须加载 web-access skill 并遵循指引`

### 其他研究原则

- Search in multiple languages when the topic benefits from it (e.g., a Chinese tech product may have better Chinese sources)
- Prefer primary sources (official docs, papers, announcements) over secondary summaries
- Include dates in findings — note when information may be outdated
- For comparisons, build structured tables rather than prose lists
- Cite every factual claim with its source URL
