---
name: bilibili-summary
description: 提取 B站视频信息（标题、描述、标签、字幕），总结为 Obsidian 笔记并保存
user-invocable: true
argument-hint: <B站视频链接>
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

用户希望提取 B站视频内容并保存为 Obsidian 笔记。请按以下步骤处理：

## 常量定义
- Obsidian 保存目录: `D:\obsidian笔记\B站收藏\`

## 输入
用户提供的 B站链接: $ARGUMENTS

## 提取流程

### 步骤 1：获取视频信息

从 URL 中提取 BVID，调用 B站開放 API 获取视频基本信息 + 标签 + 字幕元数据。

```python
import json, urllib.request, re, ssl, sys, hashlib, time
from datetime import datetime, timezone, timedelta

if sys.platform == "win32":
    sys.stdout.reconfigure(encoding='utf-8')

bvid_input = """<BVID>""".strip()
m = re.search(r'(BV[A-Za-z0-9]{10,})', bvid_input)
if not m:
    print("ERROR: 无法解析BVID")
    sys.exit(1)
bvid = m.group(1)

ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Referer': 'https://www.bilibili.com/',
}

def fetch(url):
    req = urllib.request.Request(url, headers=headers)
    resp = urllib.request.urlopen(req, timeout=20, context=ctx)
    raw = resp.read()
    ce = resp.headers.get('Content-Encoding', '')
    if 'gzip' in ce:
        import gzip
        return gzip.decompress(raw).decode('utf-8', errors='ignore')
    return raw.decode('utf-8', errors='ignore')

# ---- 获取视频基本信息 ----
view_text = fetch(f'https://api.bilibili.com/x/web-interface/view?bvid={bvid}')
view_data = json.loads(view_text)
if view_data.get('code') != 0:
    print(f"ERROR: 视频信息获取失败 (code={view_data.get('code')})")
    print(f"可能原因：视频不存在、隐私视频、或需要登录")
    sys.exit(1)

vd = view_data['data']

# 解析基础字段
title = vd.get('title', '')
title = title.replace('&amp;','&').replace('&lt;','<').replace('&gt;','>').replace('&quot;','"').replace('&#39;',"'")
owner_name = vd.get('owner', {}).get('name', '未知UP主')
owner_mid = vd.get('owner', {}).get('mid', '')
desc = vd.get('desc', '')
desc = desc.replace('&amp;','&').replace('&lt;','<').replace('&gt;','>').replace('&quot;','"').replace('&#39;',"'")
pub_ts = vd.get('pubdate', 0)
pub_date = datetime.fromtimestamp(pub_ts, tz=timezone(timedelta(hours=8))) if pub_ts else datetime.now(tz=timezone(timedelta(hours=8)))
date_str = pub_date.strftime('%Y-%m-%d')
aid = vd.get('aid', 0)
cid = vd.get('cid', 0)
duration = vd.get('duration', 0)
duration_str = f"{duration // 60}:{duration % 60:02d}"
stat = vd.get('stat', {})
tname = vd.get('tname', '')
pic = vd.get('pic', '')

# ---- 获取标签 ----
tags = [tname] if tname else []
try:
    tag_text = fetch(f'https://api.bilibili.com/x/web-interface/view/detail/tag?bvid={bvid}')
    tag_data = json.loads(tag_text)
    if tag_data.get('code') == 0:
        for t in tag_data.get('data', []):
            tn = t.get('tag_name', '')
            if tn and tn not in tags:
                tags.append(tn)
except:
    pass
tags = tags[:12]
tag_str = ', '.join(tags)

# ---- 提取字幕元数据 ----
# view API 可能在 data.subtitle.list 或 data.subtitle.subtitles 中返回字幕列表
subtitle_urls = []
subtitle_infos = []

# 尝试多个可能的字段路径
subs_list = None
for path in [
    ['subtitle', 'subtitles'],
    ['subtitle', 'list'],
]:
    d = vd
    try:
        for key in path:
            d = d.get(key, {})
        if isinstance(d, list):
            subs_list = d
            break
    except:
        continue

if subs_list:
    for sub in subs_list:
        sub_url = sub.get('subtitle_url', '')
        if sub_url:
            if sub_url.startswith('//'):
                sub_url = 'https:' + sub_url
            subtitle_urls.append(sub_url)
            subtitle_infos.append({
                'url': sub_url,
                'lan': sub.get('lan', ''),
                'lan_doc': sub.get('lan_doc', '未知语言'),
            })

has_subtitles = len(subtitle_urls) > 0

# ---- 输出 ----
result = {
    'bvid': bvid, 'aid': aid, 'cid': cid,
    'title': title, 'owner_name': owner_name, 'owner_mid': owner_mid,
    'desc': desc, 'date_str': date_str, 'duration': duration,
    'duration_str': duration_str,
    'view_count': stat.get('view',0), 'like_count': stat.get('like',0),
    'fav_count': stat.get('favorite',0), 'coin_count': stat.get('coin',0),
    'share_count': stat.get('share',0), 'danmu_count': stat.get('danmaku',0),
    'reply_count': stat.get('reply',0),
    'tags': tags, 'tag_str': tag_str, 'pic': pic,
    'has_subtitles': has_subtitles,
    'subtitle_urls': subtitle_urls,
    'subtitle_infos': subtitle_infos,
}
print(json.dumps(result, ensure_ascii=False))
```

运行上述脚本，将 `<BVID>` 替换为用户提供的 BVID。捕获输出的 JSON 对象供后续步骤使用。

### 步骤 2：获取字幕文本

**首选方案：BBDown**（B站专用下载器，可获取 AI 字幕，无需登录）

B站公开 API（view/player）通常返回空字幕列表，即使视频有 AI 字幕。**必须使用 BBDown 并关闭 AI 字幕跳过**（`--skip-ai false`，此选项默认开启）。

```bash
BBDown "<视频URL>" --sub-only --skip-ai false --work-dir "<临时目录>"
```

BBDown 会下载 `.ai-zh.srt`（中文AI字幕）等文件。优先使用 `ai-zh`，其次 `ai-en`，最后其他语言。

然后解析 SRT 文件提取字幕文本和时间戳：

```python
import re, sys, os, glob

if sys.platform == "win32":
    sys.stdout.reconfigure(encoding='utf-8')

srt_dir = """<BBDOWN输出目录>"""
srt_file = None

# 优先找中文AI字幕
for pattern in ['*.ai-zh.srt', '*.ai-zh-Hans.srt', '*.zh-CN.srt', '*.ai-en.srt', '*.ai-*.srt', '*.srt']:
    matches = glob.glob(os.path.join(srt_dir, pattern))
    if matches:
        srt_file = matches[0]
        break

if not srt_file:
    print("SUB_TEXT: none (BBDown未生成字幕文件)")
    sys.exit(0)

with open(srt_file, 'r', encoding='utf-8') as f:
    srt_text = f.read()

# 解析 SRT 格式
# 格式: 序号\n时间范围\n文本\n\n
blocks = re.split(r'\n\s*\n', srt_text.strip())
subtitle_lines = []

for block in blocks:
    lines = block.strip().split('\n')
    if len(lines) >= 2:
        # 跳过序号行，解析时间行
        time_match = re.match(r'(\d{2}):(\d{2}):(\d{2})[,.](\d{3})\s*-->\s*(\d{2}):(\d{2}):(\d{2})[,.](\d{3})', lines[1])
        if time_match:
            h1, m1, s1 = int(time_match.group(1)), int(time_match.group(2)), int(time_match.group(3))
            start_sec = h1 * 3600 + m1 * 60 + s1
            # 文本是时间行之后的所有行
            text = ' '.join(line.strip() for line in lines[2:] if line.strip())
            if text:
                subtitle_lines.append({
                    'from': start_sec,
                    'content': text,
                })

if subtitle_lines:
    # 合并相邻短句为段落
    def format_timestamp(seconds):
        m = int(seconds) // 60
        s = int(seconds) % 60
        return f"[{m:02d}:{s:02d}]"

    paragraphs = []
    current_para = []
    current_start = subtitle_lines[0]['from']
    for i, line in enumerate(subtitle_lines):
        if not current_para:
            current_start = line['from']
            current_para = [line['content']]
        else:
            prev_end = subtitle_lines[i-1].get('to', subtitle_lines[i-1]['from'])
            gap = line['from'] - prev_end
            if gap < 2.0 and len(''.join(current_para)) < 200:
                current_para.append(line['content'])
            else:
                ts = format_timestamp(current_start)
                paragraphs.append((ts, ' '.join(current_para)))
                current_start = line['from']
                current_para = [line['content']]
    if current_para:
        ts = format_timestamp(current_start)
        paragraphs.append((ts, ' '.join(current_para)))

    # 从文件名提取字幕来源
    source = os.path.basename(srt_file)
    if 'ai-zh' in source:
        source = 'B站 AI自动生成字幕（中文简体）'
    elif 'ai-en' in source:
        source = 'B站 AI自动生成字幕（英文）'
    else:
        source = f'B站字幕 ({source})'

    print(f"SUB_SOURCE: {source}")
    print(f"SUB_LINES: {len(subtitle_lines)}")
    print(f"SUB_PARAS: {len(paragraphs)}")
    print("SUB_TEXT_START")
    for line in subtitle_lines:
        ts = format_timestamp(line['from'])
        print(f"{ts} {line['content']}")
    print("SUB_TEXT_END")
    print("SUB_PARAS_START")
    for ts, para in paragraphs:
        print(f"{ts} {para}")
    print("SUB_PARAS_END")
else:
    print("SUB_TEXT: none (SRT解析为空)")
```

**如果 BBDown 未安装或失败**，回退到以下方法：

1. 先尝试 `winget install BBDown` 安装
2. 若安装失败，用步骤 1 的 `subtitle_urls` 直接 fetch 字幕 JSON
3. 再不行用 WBI Player API（见原方案）
4. 最后尝试 whisper 语音转录（下载音频 + `whisper --model small --language zh`）

运行上述流程获取字幕。如果所有方法均失败，输出 `SUB_TEXT: none`。

### 步骤 3：生成 Markdown 笔记

根据步骤 1 的视频信息和步骤 2 的字幕结果，生成完整笔记。

**写作风格**：总结部分保持精炼、有洞察力。完整字幕部分逐字保留，不删减。

**文件名**：`{date_str} {短标题}.md`，短标题不超过 15 个字，是核心洞察的极简概括。

#### 有字幕版本（步骤 2 返回了字幕文本）

```markdown
# {{你根据视频内容提炼的一句话核心洞察}}

> [!tip]- 💡 总结
> {{核心论点，3-6句话。给出视频的核心判断、关键论据和主要结论。
> 不要只写2句就停——充分提取视频中的信息密度。
> 如果视频涉及多个论点，逐一列出。}}
>
> **关键要点：**
> - {{要点1：一句话概括一个重要观点}}
> - {{要点2}}
> - {{要点3}}
> - {{更多要点，至少3-5个。如果视频信息量大，可以更多}}
>
> **与我的关联：** {{读取用户 memory 判断关联性；不可用时从通用角度切入}}
>
> **值得深挖吗：** {{是/否}}。{{一句话理由}}

---

> [!tip]- 📝 完整字幕（逐字记录）
> 来源：{{字幕来源，如"AI自动生成"、"CC字幕"等}}
> 共 {{N}} 条字幕，以下为完整记录。
>
> {{步骤2 SUB_PARAS_START 到 SUB_PARAS_END 之间的段落化字幕内容，每条带时间戳，行首加 > }}

---

> [!info]- 笔记属性
> - **来源**: B站 · {{UP主名}}
> - **链接**: https://www.bilibili.com/video/{{BVID}}
> - **日期**: {{YYYY-MM-DD}}
> - **时长**: {{MM:SS}}
> - **互动**: {{N}}播放 / {{N}}点赞 / {{N}}收藏 / {{N}}弹幕
> - **分类**: {{标签}}
```

**重要：** 完整字幕部分必须包含所有字幕内容，不删减、不缩写。

#### 无字幕版本（步骤 2 返回 SUB_TEXT: none）

```markdown
# {{根据标题和描述提炼的核心洞察}}

> [!tip]- 💡 总结
> {{核心论点，从视频描述和标题中提炼。注意：部分 B站视频描述为空或极短，
> 这种情况下核心洞察应基于标题和 UP主信息推断。}}
>
> **关键要点：**
> - {{根据可用信息提炼的要点}}
>
> **与我的关联：** {{同上}}
>
> **值得深挖吗：** {{是/否}}。{{一句话理由}}

> [!tip]- 详情
> 从视频描述和标签中提取的要点：
> 
> {{描述中提取的结构化内容。如果描述为空或极短，标注"描述信息有限"}}

> [!warning]- 无字幕
> 该视频无可用字幕，以上内容仅基于标题、描述和标签生成。
> 如需深度分析，可配合 Whisper 进行语音转录。

> [!info]- 笔记属性
> - **来源**: B站 · {{UP主名}}
> - **链接**: https://www.bilibili.com/video/{{BVID}}
> - **日期**: {{YYYY-MM-DD}}
> - **时长**: {{MM:SS}}
> - **互动**: {{N}}播放 / {{N}}点赞 / {{N}}收藏 / {{N}}弹幕
> - **分类**: {{标签}}
```

**关键约束：**
- 总结部分要充分提取信息，至少 3-6 句话 + 3-5 个要点
- 有字幕时，完整字幕必须包含所有内容，绝不删减
- 标题必须是洞察/判断，不是"XX视频的总结"
- 笔记属性必须包含可点击的原始链接

### 步骤 4：保存文件

用 Write 写入：

```
路径: D:\obsidian笔记\B站收藏\{date_str} {短标题}.md
```

### 步骤 5：更新索引笔记

在保存笔记后，更新 `<Obsidian 保存目录>/📋 索引.md`。

**索引文件结构：**

```markdown
# 📋 B站收藏索引

> 按日期倒序。Cmd+F / Ctrl+F 搜标签即可定位。

- [[2026-06-02 短标题]] — 核心洞察 `#标签A #标签B`
- [[2026-05-28 另一个视频]] — 核心洞察 `#标签C`
```

**更新逻辑：**

1. 用 Glob 列出 `D:\obsidian笔记\B站收藏\` 下所有 `*.md` 文件（排除索引文件自身和 Welcome.md）
2. 用 Grep 或 Python 提取每个文件行首的 `# ` 标题和 `- **分类**:` 行里的标签
3. 按文件名中的日期前缀倒序排列（新在前）
4. 标签内联在行末，用反引号包裹
5. 用 Write 写入索引文件

**约束：**
- 不分组、不分级——纯时间线清单
- 索引文件自身不出现在列表中

---

## ⚠️ 踩坑记录（维护者必读）

以下是在开发/调试过程中发现的坑，修改代码前务必了解。

### 🔴 字幕获取

1. **B站公开 API 全部不返回 AI 字幕**
   - `view` API、`player/v2`、`player/wbi/v2` 均返回 `subtitles: []`
   - 即使带了正确的 WBI 签名 + Cookie 也不行
   - 根因：B站的 AI 字幕需要通过专门的下载器（BBDown）或登录态请求才能获取
   - **唯一可靠方案：BBDown**

2. **BBDown `--skip-ai` 默认开启**
   - 这个标志的含义是"跳过AI字幕下载"，且**默认值为 true**
   - 必须显式传 `--skip-ai false`，否则只会下载手动上传的 CC 字幕
   - ❌ 错误：`BBDown <url> --sub-only`
   - ✅ 正确：`BBDown <url> --sub-only --skip-ai false`

3. **BBDown 不需要登录也能下载 AI 字幕**
   - BBDown 会提示"你尚未登录B站账号"，但 AI 字幕下载不受影响
   - 只有大会员专属视频需要 `BBDown login` 扫码登录

4. **字幕格式是 SRT 不是 JSON**
   - 旧版 skill 假设字幕是 B站 JSON 格式（`{"body": [{"from":..., "to":..., "content":...}]}`）
   - BBDown 输出标准 SRT 格式，需要完全不同的解析逻辑
   - SRT 的时间戳精度是毫秒（`00:00:00,000`），旧代码是秒级浮点数

5. **BBDown 安装方式**
   - Windows：`winget install BBDown`（推荐）
   - 或从 https://github.com/nilaoda/BBDown/releases 下载
   - 安装后需要重启终端才能使用（PATH 更新）

### 🔴 WBI 签名

6. **旧版 WBI 签名算法完全错误**
   - ❌ 旧代码：`mix_key = sub_key[:4] + img_key[:4]`（只取前8个字符）
   - ✅ 正确：使用 64 位映射表 `mixin_key_enc_tab` 从 `img_key + sub_key` 中选取 32 个字符
   - 映射表是 B站前端混淆后的固定常量，不会频繁变动
   - 目前的回退方案仍使用 WBI Player API，签名算法已修正

### 🟡 运行环境

7. **PowerShell here-string 中 `"""` 会冲突**
   - `@'...'@` 内不能出现 `"""`（Python 三引号），会导致解析错误
   - 解决方案：将 Python 脚本写入临时 `.py` 文件再执行

8. **yt-dlp 用户可能未安装**
   - 需要 `pip install yt-dlp`
   - 仅作为 BBDown 失败后的第三回退方案

9. **whisper 已安装但不能代替字幕下载**
   - whisper 可以做语音转录，但速度慢（11分钟视频可能需要数分钟）
   - 仅在 BBDown 和 yt-dlp 均失败时考虑使用

### 🟡 输出格式

10. **Obsidian callout 语法**
    - 折叠区块必须用 `> [!type]-` 格式（`-` = 默认折叠，`+` = 默认展开）
    - callout 内所有内容行首必须加 `>`
    - 换行用 `>` 独占一行（空行也带 `>`）
    - `##` 标题不支持折叠，必须改用 callout 语法

11. **笔记模板要点数量约束**
    - 总结至少 3-6 句话 + 3-5 个关键要点
    - 有字幕时完整字幕**绝不删减**
    - 标题必须是洞察/判断，不能是"XX视频的总结"
