# 🎬 Bilibili Summary Skill — B站视频 → Obsidian 笔记

[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-8A2BE2)](https://claude.ai/code)

English | [中文](#中文)

---

## English

**bilibili-summary** is a [Claude Code](https://claude.ai/code) skill that extracts Bilibili video information, downloads AI-generated subtitles, summarizes the content, and saves it as an Obsidian note.

### Features

- ✨ **Video metadata extraction** — title, description, tags, uploader, stats
- 🎯 **AI subtitle download** — uses [BBDown](https://github.com/nilaoda/BBDown) to grab AI-generated subtitles (Chinese, English, Japanese, etc.)
- 📝 **Smart summarization** — generates a structured summary with key insights and takeaways
- 📖 **Full transcript preservation** — complete subtitle text with timestamps, never truncated
- 📂 **Obsidian integration** — saves notes to a configurable directory and maintains an index

### Requirements

- **Claude Code** (desktop or CLI, v1.0+)
- **BBDown** — install via `winget install BBDown` (Windows) or download from [releases](https://github.com/nilaoda/BBDown/releases)

### Installation

**Step 1: Install BBDown**

```bash
# Windows
winget install BBDown

# macOS / Linux (via GitHub releases)
# Download from https://github.com/nilaoda/BBDown/releases
```

Restart your terminal after installing BBDown to refresh PATH.

**Step 2: Install the skill**

```bash
# Clone the repo
git clone https://github.com/DuckWu/bilibili-skills.git

# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Copy the skill into Claude Code's skill directory
# Windows PowerShell:
Copy-Item -Recurse bilibili-skills/skills/bilibili-summary "$env:USERPROFILE\.claude\skills\bilibili-summary"

# macOS / Linux:
cp -r bilibili-skills/skills/bilibili-summary ~/.claude/skills/bilibili-summary
```

**Step 3: Restart Claude Code**

The skill will appear as `/bilibili-summary` in the slash command list.

> **Note:** To update the skill later, pull the latest changes and re-copy the skill directory.

### Usage

In Claude Code CLI or desktop:

```
/bilibili-summary https://www.bilibili.com/video/BVxxxxxxxxxx
```

The skill will:
1. Fetch video info via Bilibili public API
2. Download AI subtitles via BBDown (`--sub-only --skip-ai false`)
3. Generate an Obsidian note with summary + full transcript (foldable callout sections)
4. Save to `D:\obsidian笔记\B站收藏\`
5. Update the index file `📋 索引.md`

### Output Format

The generated note has three collapsible sections in Obsidian callout syntax:

- **💡 Summary** — 3-6 sentence core thesis + 3-5 key takeaways
- **📝 Full Transcript** — complete subtitle text with timestamps, line by line
- **ℹ️ Note Attributes** — metadata (source, link, date, stats, tags)

### Known Pitfalls

> See the full [pitfall log](skills/bilibili-summary/SKILL.md#-踩坑记录维护者必读) in SKILL.md.

- Bilibili public APIs return empty subtitle lists; **BBDown is the only reliable method**
- BBDown's `--skip-ai` flag defaults to `true` — must pass `--skip-ai false` to get AI subtitles
- BBDown does **not** require login for AI subtitle downloads (only for premium member content)

### License

MIT

---

## 中文

**bilibili-summary** 是一个 [Claude Code](https://claude.ai/code) 技能，用于提取 B站视频信息、下载 AI 字幕、总结内容，并保存为 Obsidian 笔记。

### 功能

- ✨ **视频元数据提取** — 标题、描述、标签、UP主、统计数据
- 🎯 **AI 字幕下载** — 使用 [BBDown](https://github.com/nilaoda/BBDown) 获取 AI 生成字幕（中文、英文、日文等）
- 📝 **智能总结** — 生成结构化总结，包含核心观点和关键要点
- 📖 **完整字幕保留** — 逐字保留全部字幕内容，带时间戳，绝不删减
- 📂 **Obsidian 集成** — 保存到指定目录并维护索引文件

### 前提条件

- **Claude Code**（桌面版或 CLI，v1.0+）
- **BBDown** — 通过 `winget install BBDown`（Windows）安装，或从 [releases](https://github.com/nilaoda/BBDown/releases) 下载

### 安装教程

**第一步：安装 BBDown**

```bash
# Windows
winget install BBDown

# macOS / Linux（从 GitHub releases 下载）
# https://github.com/nilaoda/BBDown/releases
```

安装后重启终端，确保 BBDown 命令可用。

**第二步：安装 skill**

```bash
# 克隆仓库
git clone https://github.com/DuckWu/bilibili-skills.git

# 创建 skills 目录（如果不存在）
mkdir -p ~/.claude/skills

# 将 skill 复制到 Claude Code 的 skill 目录
# Windows PowerShell：
Copy-Item -Recurse bilibili-skills/skills/bilibili-summary "$env:USERPROFILE\.claude\skills\bilibili-summary"

# macOS / Linux：
cp -r bilibili-skills/skills/bilibili-summary ~/.claude/skills/bilibili-summary
```

**第三步：重启 Claude Code**

重启后即可在 slash 命令列表中看到 `/bilibili-summary`。

> **提示：** 后续更新 skill 只需 `git pull` 然后重新复制 skill 目录即可。

### 使用方法

在 Claude Code CLI 或桌面版中：

```
/bilibili-summary https://www.bilibili.com/video/BVxxxxxxxxxx
```

技能将依次：
1. 通过 B站开放 API 获取视频信息
2. 通过 BBDown 下载 AI 字幕（`--sub-only --skip-ai false`）
3. 生成包含总结 + 完整字幕（可折叠 callout 区块）的 Obsidian 笔记
4. 保存到 `D:\obsidian笔记\B站收藏\`
5. 更新索引文件 `📋 索引.md`

### 输出格式

生成的笔记包含三个可折叠的 Obsidian callout 区块：

- **💡 总结** — 3-6 句核心论点 + 3-5 个关键要点
- **📝 完整字幕** — 带时间戳的逐行完整字幕文本
- **ℹ️ 笔记属性** — 元数据（来源、链接、日期、互动数据、标签）

### 踩坑记录

> 完整踩坑记录见 [SKILL.md](skills/bilibili-summary/SKILL.md#-踩坑记录维护者必读)。

- B站公开 API 返回空字幕列表；**BBDown 是唯一可靠方案**
- BBDown 的 `--skip-ai` 参数**默认开启**，必须传 `--skip-ai false` 才能下载 AI 字幕
- BBDown **不需要登录**即可下载 AI 字幕（大会员专属内容除外）

### 许可证

MIT
