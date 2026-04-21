---
name: video-to-obsidian-notes
description: "转录视频为文字并生成 Obsidian 格式的结构化笔记。支持批量处理，下载可并行，转录+总结严格串行。"
category: productivity
---

# Video to Obsidian Notes

将本地视频文件转录为文字，生成结构化的 Obsidian Markdown 笔记。

### 触发条件
- 用户发来视频链接或本地视频文件，要求"总结"或"写笔记"
- 用户有一批视频需要批量转录和总结
- 用户提到"放到 Obsidian"、"做笔记"
- 用户分享抖音/小红书链接但无法直接下载时，触发降级推理流程

## 环境信息

### 关键路径
| 用途 | 路径 |
|------|------|
| 视频存放目录 | `/Users/lomychen/.hermes/profiles/kid/home/VideoProcess/` |
| Obsidian 仓库 | `/Users/lomychen/Documents/lomyc的仓库/` |
| 笔记存放目录 | `/Users/lomychen/Documents/lomyc的仓库/总结/` |
| faster-whisper | `/Users/lomychen/claude/hermes-agent/venv/bin/python3.11` |

### 转录环境
- **工具**: `faster-whisper` (Python 包)
- **Python**: python3.11 (在 hermes-agent venv 中)
- **模型**: `base` (够用且快)
- **语言**: `zh` (中文)

### ⚠️ 已知问题与规避
1. **系统 python3.9 的 whisper CLI 可能缺 ffmpeg** → 改用 `python3.11 -c "from faster_whisper..."` 方式
2. **whisper medium 模型可能 checksum 不匹配** → 用 `base` 模型
3. **ffmpeg 可能未安装** → `.mp4` 文件 `faster-whisper` 直接支持，但 `.wav` 最稳妥
4. **转录耗时较长** → 放后台 `notify_on_complete=true`，不阻塞后续工作

## 工作流程

### 第一阶段：下载视频（可并行）
```bash
# 视频保存到 /Users/lomychen/.hermes/profiles/kid/home/VideoProcess/
# 多个视频可同时下载
```

### 第二阶段：转录（严格串行，一个接一个）

```python
from faster_whisper import WhisperModel

model = WhisperModel('base', device='cpu', compute_type='int8')
segments, info = model.transcribe('视频文件.mp4', language='zh', beam_size=5)
text = ' '.join(seg.text.strip() for seg in segments)

with open('视频文件.txt', 'w') as f:
    f.write(text)
```

**终端命令**:
```bash
cd /Users/lomychen/.hermes/profiles/kid/home/VideoProcess && python3.11 -c "
from faster_whisper import WhisperModel
model = WhisperModel('base', device='cpu', compute_type='int8')
segments, info = model.transcribe('视频名.mp4', language='zh', beam_size=5)
text = ' '.join(seg.text.strip() for seg in segments)
with open('视频名.txt', 'w') as f:
    f.write(text)
print(f'Done: {len(text)} chars')
"
```

### 第三阶段：总结笔记（严格串行，一个接一个）

#### 笔记模板
```markdown
# {视频标题}

## 📝 基本信息
- **日期**: 2026-04-16
- **来源**: 视频转录总结
- **主题**: {主题分类}

## 🎯 核心观点
> {一句话总结视频的核心观点或最震撼的一句话}

## 🔑 关键信息
{根据转录内容提炼的核心要点，用 markdown 表格/列表组织}

## 💡 行动清单 / 实用技巧
- [ ] {可操作的建议}

## 📌 总结 / 金句
> {最核心的一句话总结}

## 🔗 相关笔记
- [[{相关笔记名}]]

## 📜 完整视频转录原文
<details>
<summary>点击查看完整转录原文</summary>

{将转录的 .txt 文件内容按句子换行格式化后附在此处，每句占一行}

</details>
```

**⚠️ 强制要求：每条笔记必须附带完整转录原文，放在 `<details>` 折叠块中。** 这样在 Obsidian 里默认收起不干扰阅读，但需要时可以展开查看原文。

#### 命名规则
`2026-04-16-{简短标题}.md`（注意：文件名必须以日期开头，以便 Obsidian 双链索引正确匹配）

#### Obsidian 链接规范
- **WikiLink 格式**：统一使用 `[[文件名|显示名]]` 格式。
- **禁止带路径**：绝对不要写成 `[[总结/2026-xx-xx-标题|显示名]]`，这会导致 Obsidian 无法识别反向链接。
- **双链匹配**：确保笔记内的 `[[...]]` 链接指向实际存在的文件名（带日期前缀），例如 `[[2026-04-16-财务自由之路精读]]` 而不是 `[[财务自由之路精读]]`。

#### 存储路径
`/Users/lomychen/Documents/lomyc的仓库/总结/2026-04-16-{标题}.md`

### 第四阶段：自动更新索引（必须执行）

笔记生成后，**必须**自动更新 `2026-视频笔记索引.md`：
1. 加载 `auto-update-obsidian-index` 技能。
2. 运行该技能，将刚刚生成的笔记自动归类到索引的正确板块中。
3. 确认索引文件已包含新笔记。

## 重要规则

### 🔀 转录失败时的降级策略（针对抖音/小红书等平台）
- 若因反爬机制、IP 限制或加密导致 `yt-dlp` 下载/转录失败：
  1. **提取元数据**：尽可能抓取标题、简介、标签、话题。
  2. **知识推理**：如果标题指向知名书籍、理论或开源项目（如《牛奶可乐经济学》、Nano Banana），利用已有知识库还原核心内容。
  3. **明确告知用户**：在总结开头说明"由于平台限制，本次总结基于标题与领域知识推理，如有细节遗漏请指正"。
  4. **请求补充**：若推理出的内容与用户需求偏差较大，引导用户提供更具体的细节或更换来源。

### 🚫 禁止并行
- **转录**：必须串行，一个完成后才开始下一个
- **总结笔记**：必须串行，一个完成后才开始下一个
- 原因：转录耗时长（每个视频 5-10 分钟），并行会卡死

### ✅ 允许并行
- **视频下载**：可以同时下载多个视频

## 完整示例流程

1. 检查 `VideoProcess/` 目录下有哪些 `.mp4` 文件
2. 检查哪些已有 `.txt` 转录，找出待转录的
3. 逐个转录（serial）
4. 逐个读取 `.txt` → 生成 `.md` 笔记（serial）
5. 汇报完成状态

## 后续：推送到 GitHub

用户可能需要将 Obsidian 仓库推送到 GitHub：
```bash
cd "/Users/lomychen/Documents/lomyc的仓库"
git add .
git commit -m "Add video summary notes"
git push
```

## Pitfalls（踩坑记录）

1. **不要用系统 python3.9 跑 whisper CLI**：可能缺 ffmpeg 依赖，模型 checksum 也不匹配。统一用 `python3.11` + `faster-whisper`。
2. **转录不要并行**：会卡死。下载可以并行。
3. **笔记去重**：生成前检查同名文件是否存在，避免重复。
4. **大文件读取**：转录文本可能很长，用 `read_file` 的 `limit` 参数分段读取。
