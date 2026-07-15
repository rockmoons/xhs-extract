---
name: xhs-extract
description: >
  小红书笔记数据提取工具。当用户发送小红书链接（xhslink.com、xiaohongshu.com/explore、xiaohongshu.com/discovery/item）、
  小红书笔记ID，或提到「查笔记」「提取文案」「改写」「导出Excel」「导出飞书」「批量查询」「读这个文件」「处理这个表格」时使用。
  也是 /xhs-extract 命令的处理器。
  作者：阿南 rockmoons（抖音），微信：rockmoons，网站：www.rockmoons.com
---

# 🌸 xhs-extract · 小红书笔记数据提取工具

> 按需借鉴 dy-extract，所有设计基于实际 API 测试数据。

---

## ⚠️ 中文编码铁律（重要）

所有涉及中文字符处理的 shell 命令，**必须写成 Python 脚本文件**执行，不得直接在命令行中传递中文。

**正确做法：** 写一个 `.py` 文件（含 `# -*- coding: utf-8 -*-`），然后 `python xxx.py`
（如果 `python` 不可用，试试 `python3`）

**错误做法：** `curl ... | python3 -c "print(json.load(...))"`（含中文会乱码）

这条规则同样适用于：Excel 导出、飞书导出、ASR、改写输出、任何包含中文的 JSON 解析。

---

## Step 0 · 读取配置

**每次启动技能第一步就是读配置。**

```
Read config.json
```

用以下命令读取：

```bash
cat config.json
```

**检查项（AI 读取后检查）：**
- `xhs.apikey` 为空（`""`）→ 引导：「💡 请先到 www.rockmoons.com 获取 API Key，我来帮你写入 config.json」。用户提供后，用 `Edit` 工具写入 `config.json` 的 `xhs.apikey` 字段。
- `xhs.apikey` 已有值 → 继续。**AI 记住这两个值供后续步骤使用：**
  - `{xhs.base_url}` → 例如 `https://api.rockmoons.com/blade-links/openapi/plugin`
  - `{xhs.apikey}` → 例如 `XNGlhX0llFgXmr0RkBUXMUR6SxsmBZSY`
- 文件不存在 → 复制 `config.example.json` 为 `config.json`，提示用户填写 API Key。

---

## Step 1 · 解析输入

### 社交/无意义输入

用户说的不是链接也不是操作指令：

| 用户说 | 回应 |
|--------|------|
| "你好" "嗨" "在吗" "hello" | 「你好呀～ 发我一条小红书链接，我帮你查详情 📕」 |
| "谢谢" "好的" "明白了" | 「不客气～ 还有什么需要吗？😊」 |
| "嗯" "哦" "ok" | 无明确意图，等待用户下一步 |
| 纯数字（如 "123" "123456"） | 「这个是笔记ID吗？小红书笔记ID是20~30位的字母+数字组合哦～」 |

### 欢迎/无输入

如果用户只输入了 `/xhs-extract` 或 `/xhs-extract 帮助`，或其他无实质内容的进入方式，显示：

```
📕 xhs-extract · 小红书笔记提取工具

发我小红书链接或笔记ID，帮你查详情～

📎 支持的链接格式：
   xhslink.com/xxx          ← 分享短链
   xiaohongshu.com/explore  ← 网页链接
   xiaohongshu.com/discovery/item  ← 分享链接
   纯笔记ID（20~30位字母数字）

💬 示例：http://xhslink.com/o/8lsiyy1t7Ay

💡 说"帮助"查看全部功能
```

### 检测到链接/ID 后的第一步

一旦 AI 从用户输入中识别出小红书链接或纯 ID，**先显示进度反馈**：

```
🔍 正在查询笔记信息...
```

然后再执行后续的解析和 API 调用。

### 匹配规则（按优先级）

| 用户输入 | 提取方式 | 示例 |
|---------|---------|------|
| `xhslink.com/xxx` 或 `http://xhslink.com/o/xxx` | `curl -L` follow 跳转 → 从目标 URL 提取 `/item/{note_id}` | `http://xhslink.com/o/8lsiyy1t7Ay` |
| `xiaohongshu.com/explore/{id}` | 正则 `/explore/([^?]+)` 提取 | `/explore/6a56e1aa000000002103db4f` |
| `xiaohongshu.com/discovery/item/{id}` | 正则 `/item/([^?]+)` 提取 | `/discovery/item/6a56e1aa000000002103db4f` |
| 纯 ID（20~30 位字母数字） | 正则 `^[a-zA-Z0-9]{20,30}$` | `6a56e1aa000000002103db4f` |
| 多条链接（换行分隔） | 跳 Step 6 批量 | |
| 文件（.txt/.csv/.xlsx/.docx） | 跳 Step 6 批量 | |

### xhslink 短链处理

```bash
curl -s -L --connect-timeout 10 --max-time 15 -o /dev/null -w "%{url_effective}" "http://xhslink.com/o/xxx" 2>/dev/null
```

**AI 处理步骤：**
1. 执行 curl，获取跳转后的完整 URL
2. 从 URL 中提取 note_id：
   - 用 `grep -o '/item/[a-zA-Z0-9]*'` 提取 `/item/xxx` 部分
   - 再用 `cut -d'/' -f3` 取出 ID
3. 提取不到 → 该链接不是有效的小红书笔记链接
4. curl 返回空或超时 → 「网络请求失败，请检查链接后重试」

### 纯 ID 检测

用 Shell 或 Python 判断：

```bash
echo "$user_input" | grep -P '^[a-zA-Z0-9]{20,30}$'
```

匹配则直接作为 `note_id`。

### 输入验证

- **非小红书链接/非 ID** → 「这个链接我识别不了，请发小红书笔记链接或笔记ID～」
- **纯文字无链接**（"查一下" "看看这个"） → 「我需要一条小红书链接或笔记ID哦～」
- **"帮助" "怎么用" "help" "?"** → 跳 Step 8
- **"提取文案" "改写" "导出" 等操作指令** → 使用上一次查询的数据（如有），否则提示「还没有查过笔记哦，先发一条链接吧～」

---

## Step 2 · 调 API

这是核心步骤。

**在调 API 之前，先给用户进度反馈：**
```
🔍 正在查询笔记信息...
```
（API 调用约需 2-3 秒，避免用户以为没反应。）

### 调用流程

```
提取到 note_id
    ↓
🔍 正在查询笔记信息...（先给用户反馈）
    ↓
调 图文接口（通杀两种类型）
    ↓
判断 笔记类型
  ├─ "normal" → 只用图文接口数据 ✅
  └─ "video"  → 先显示「📦 正在获取补充数据...」
                 再调视频接口补全数据 ✅
```

### 图文接口（必调）

```bash
curl -s --connect-timeout 10 --max-time 30 \
  "{xhs.base_url}/xhs_fetch_image_note_info?note_id={note_id}&apikey={xhs.apikey}"
```

> AI 注意：将 `{xhs.base_url}` 和 `{xhs.apikey}` 替换为 Step 0 从 config.json 读到的实际值；将 `{note_id}` 替换为 Step 1 提取的笔记 ID。

路由：`xhs_fetch_image_note_info`
参数：`note_id`（纯ID，非完整链接）
返回：20 个字段（详见 api-reference.md）

### 视频接口（仅类型="video"时补调）

```bash
curl -s --connect-timeout 10 --max-time 30 \
  "{xhs.base_url}/xhs_fetch_video_note_info?note_id={note_id}&apikey={xhs.apikey}"
```

> AI 注意：替换 `{xhs.base_url}`、`{xhs.apikey}`、`{note_id}` 为实际值。

路由：`xhs_fetch_video_note_info`
参数：`note_id`（纯ID）
返回：20 个字段

### 数据合并规则（视频笔记）

两个接口数据合并时，按以下优先级：

| 字段 | 优先级策略 | 原因 |
|------|-----------|------|
| 作品简介 | **取视频接口**（完整版） | 图文接口会截断，只剩标签 |
| @的用户 | 取视频接口 | 视频接口独有字段 |
| 背景音乐ID | 取视频接口 | 视频接口独有字段 |
| 分享链接 | **取图文接口** | 视频接口可能完全缺失此字段 |
| 作者id/用户id | 统一为"作者ID" | 两接口值一样，字段名不同 |
| 点赞/收藏/评论/分享 | **取视频接口** | 视频接口数据更新（调得更晚） |
| 作品图片 | 取图文接口 | 图文接口才有（视频接口无此字段） |
| 图片数量 | 取图文接口 | 图文接口才有 |
| 作品简介 | **优先视频接口** | 图文接口对部分视频笔记会截断简介 |
| 其他基础字段 | 任一个均可 | 值完全一致 |

> **注意缺失字段：** 视频接口的 `分享链接`、`背景音乐ID` 可能完全不存在于响应中（而非空字符串）。合并时用 `fields.get("分享链接", "") or fields.get("分享链接", "")` 兜底。

### 解析响应

JSON 响应结构：

```json
{
  "code": 200,
  "success": true,
  "msg": "操作成功",
  "data": {
    "feishu": { "table_fields": "...", "records": [...] },
    "video_urls": [],
    "metadata": {
      "fields": "{ \"作者昵称\": \"...\", \"点赞数\": 1050, ... }"  ← 注意是JSON字符串
    }
  }
}
```

**关键：** `data.metadata.fields` 是 **JSON 字符串**，需要用 `json.loads()` 解析后才能拿到字段。由于字段名含中文，**必须写成 Python 脚本文件执行**（遵守中文编码铁律）。

**解析模板**（保存为 `parse_response.py` 执行）：

```python
# parse_response.py
# -*- coding: utf-8 -*-
import json, sys
from datetime import datetime, timezone, timedelta

# 从文件读入 API 响应（避免命令行长度限制）
with open("xhs_api_response.json", "r", encoding="utf-8") as f:
    raw = f.read()
resp = json.loads(raw)

if resp.get("code") != 200:
    print(f"ERROR|{resp.get('msg', '未知错误')}")
    sys.exit(0)

# 解析 metadata.fields（里面的字段名是中文）
meta_str = resp.get("data", {}).get("metadata", {}).get("fields", "{}")
fields = json.loads(meta_str)

# 特殊处理：作品图片（可能是 JSON 数组 ["url1","url2"]，也可能是单条 URL 字符串）
images_raw = fields.get("作品图片", "")
images = []
if images_raw:
    try:
        parsed = json.loads(images_raw)
        if isinstance(parsed, list):
            images = parsed
        else:
            images = [str(parsed)]
    except (json.JSONDecodeError, TypeError):
        # 不是 JSON 数组，当作单条 URL
        images = [str(images_raw)]
fields["_作品图片列表"] = images
fields["_图片张数"] = len(images)
fields["_首张图片"] = images[0] if images else ""

# 特殊处理：发布时间（Unix 时间戳 → 可读日期）
ts = fields.get("发布时间", 0)
if isinstance(ts, (int, float)) and ts > 0:
    dt = datetime.fromtimestamp(ts, tz=timezone(timedelta(hours=8)))
    fields["_发布时间_显示"] = dt.strftime("%Y-%m-%d %H:%M")
else:
    fields["_发布时间_显示"] = str(ts)

# 检查 video_urls
video_urls = resp.get("data", {}).get("video_urls", [])
fields["_有视频链接"] = len(video_urls) > 0
fields["_视频链接列表"] = video_urls

# 输出所有字段（供 AI 查看使用）
print(json.dumps(fields, ensure_ascii=False, indent=2))
```

使用方式：
```bash
# 调 API → 保存响应到文件
curl -s --connect-timeout 10 --max-time 30 \
  "{xhs.base_url}/xhs_fetch_image_note_info?note_id={note_id}&apikey={xhs.apikey}" \
  > xhs_api_response.json
# 解析
	python parse_response.py
# AI 从输出中读取解析后的字段
```

**视频笔记额外解析**（如果调了视频接口，将第二份响应另存）：

```bash
curl -s --connect-timeout 10 --max-time 30 \
  "{xhs.base_url}/xhs_fetch_video_note_info?note_id={note_id}&apikey={xhs.apikey}" \
  > xhs_api_response.json
python parse_response.py
```

> 注意：第二次调用会覆盖 `xhs_api_response.json`，AI 需先保存第一次的结果，或分别存为不同文件名（如 `xhs_image_response.json` 和 `xhs_video_response.json`）。

**特殊处理说明：**
- `作品图片`：已解析为 `_作品图片列表`（数组）和 `_首张图片`（第一张URL）
- `图片数量`：字符串类型，可能为空 `""`
- `发布时间`：已转为 `_发布时间_显示`（`YYYY-MM-DD HH:MM` 格式）
- `微信分享描述`：可能含 `\n` 换行符
- `分享链接`：视频接口可能返回空，优先取图文接口的值

**临时文件清理：**
```bash
rm -f parse_response.py xhs_api_response.json xhs_image_response.json xhs_video_response.json
```

### 错误处理

| 条件 | 用户提示 |
|------|---------|
| curl 请求失败（网络超时/无连接） | 「🌐 网络请求失败，请检查网络后重试」 |
| 返回非 JSON 内容 | 「⚠️ API 异常，请稍后重试」 |
| `code != 200` 且 `msg` = "插件不存在或已禁用" | 「🔌 该接口未上线，请联系平台 www.rockmoons.com」 |
| `code != 200` 且 `msg` 含"上游调用失败" | 「❌ 小红书接口返回异常，可能是链接无效或该笔记已删除/违规」 |
| `code == 200` 但 `data.metadata.fields` 为空 | 「📭 笔记数据为空，可能已删除或设为私密」 |
| 其他 `code != 200` | 显示 `msg` 原文给用户 |
| 未知异常 | 「❌ 发生未知错误，请稍后重试」 |

**视频接口调用的附加处理：**
- 如果图文接口成功但视频接口失败（仅 video 笔记），**仍需展示数据**，但标注：
  - 「⚠️ 补充数据获取失败，部分字段可能不完整」
  - 用图文接口的数据作为兜底

---

## Step 3 · 展示结果

### 图文笔记展示

```
📷 {作者昵称} · ❤️{点赞数} · ⭐{收藏数} · 💬{评论数}
   📍 {ip地址} · 🕐 {YYYY-MM-DD HH:MM}

📝 {作品简介}
🏷️ {话题}
🖼️ 共{N}张图片
   [作品图片第1张URL]

🔗 网页链接（如有分享链接则追加 " | 分享链接"）

────────────────────
✅ 查询完成
💡 改写文案 | 导出Excel | 查下一个
```

### 视频笔记展示

```
🎬 {作者昵称} · ❤️{点赞数} · ⭐{收藏数} · 💬{评论数}
   📍 {ip地址} · 🕐 {YYYY-MM-DD HH:MM}

📝 {完整作品简介}（超过200字截断）
🏷️ {话题}
🎵 背景音乐ID: {bgm_id}（如无则不显示）

🔗 网页链接（如有分享链接则追加 " | 分享链接"）

────────────────────
✅ 查询完成
💡 提取文案 | 改写文案 | 导出Excel | 查下一个
```

### 格式化规则

| 字段 | 处理 |
|------|------|
| 点赞数/收藏数/评论数/分享数 | ≥ 10000 → `X.X万`，< 10000 → 原样显示 |
| 发布时间 | `datetime.fromtimestamp(ts, tz=UTC+8).strftime("%Y-%m-%d %H:%M")` |
| 微信分享描述 | 如有 `\n` 替换为空格 |
| 作品图片数组 | 显示张数：`共{N}张图片`，展示第一张URL |
| 分享链接 | 如果为空字符串则隐藏此行 |
| @的用户 | 如果为空字符串则隐藏 |
| 背景音乐ID | 如果为空字符串则隐藏 |
| 作品简介 | 如果过长（>200字）可截断显示，提示「…完整内容见导出」 |

### 原始数据

展示完毕后，在 `<details>` 折叠块中显示完整 JSON 数据（方便进阶用户）：

```markdown
<details>
<summary>📋 查看原始数据</summary>

```json
{...全部字段...}
```
</details>
```

### 下一步提示

展示结束后，等待用户选择：

| 用户说 | 跳转 |
|--------|------|
| "提取文案" / "转文字" | Step 4（仅视频笔记） |
| "改写" / "重写" / "二创" | Step 5 |
| "导出" / "导出Excel" | Step 6 |
| "导出飞书" | Step 7 |
| "帮助" / "怎么用" | Step 8 |
| 新链接 / 新ID | 回到 Step 1 |
| "查下一个" / "再来一个" | 回到 Step 1 |

### 跨轮对话状态保留

用户可能在看完结果后聊几句别的，再说「改写」或「导出」。AI 需要：

1. **在当前对话中记住**解析后的字段数据（`fields` 字典），不要清空
2. 如果用户说了新链接/新ID → 重新走 Step 1，替换为新的数据
3. 如果用户明确说「换一个」「查别的」→ 清空旧数据，走 Step 1

---

## Step 4 · 分支：提取文案（视频笔记·运行时检测）

> 触发词：「提取文案」「转文字」「ASR」「音频」

### 图文笔记

```
📷 这篇是图文笔记，没有音频可以提取哦～
💡 改写文案 | 导出Excel
```

### 视频笔记

检查 `data.video_urls` 是否为空（AI 从 Step 2 的 API 响应中已获取此信息）：

```
情况 1：video_urls 有内容（非空数组）
→ 走 ASR 流程，读取 references/asr-local.md

情况 2：video_urls 为空 []
→ 显示：
```

  ```
  🎬 这篇视频笔记暂未获取到视频链接，无法提取文案哦～
  目前小红书接口未提供视频/音频下载地址。
  
  💡 改写文案 | 导出Excel | 查下一个
  ```

### ASR 流程（有视频链接时）

读取并执行 `references/asr-local.md`，主要步骤：

1. 下载视频/音频（从 `video_urls[0]`）
2. 安装 faster-whisper（首次自动安装）
3. 运行 Whisper 转文字
4. 展示文案结果
5. 清理临时文件

---

## Step 5 · 分支：改写

> 触发词：「改写」「重写」「二创」「去重」「总结」「精简」「扩写」「转风格」「金句」「拆分」

读取并执行 `references/rewrite-guide.md`。

6 种模式（与 dy-extract 一致）：
1. **去重优化** — 删除口语词、重复表达
2. **总结精简** — 压缩到 15秒/30秒/60秒
3. **扩写丰富** — 补充细节、案例
4. **转风格** — 小红书风 / 新闻风 / 幽默 / 深度 / 励志 / 悬念
5. **提取金句** — 摘出核心观点
6. **拆分多条** — 按话题拆分为多个段落

用户说"改写"但没说具体模式 → 展示 6 种让用户选。

---

## Step 6 · 分支：批量处理

> 触发条件：
> - 用户一次性发了多条链接（聊天中换行分隔）
> - 用户发了文件（.txt / .csv / .xlsx / .docx）
> - 用户说「批量查」「处理这个文件」「读这个文件」

读取并执行 `references/batch-input.md`。

**核心流程：**
1. 读取输入（聊天文本或文件）
2. 逐条提取 note_id（去重）
3. 判断数量：
   - ≤ 50 条 → 直接处理
   - > 50 条 → 估算时间和积分，让用户确认
4. 循环处理：每批 50 条，间隔 500ms
5. 实时进度反馈
6. 完成汇总：成功/失败条数 + 失败原因

---

## Step 7 · 分支：导出 Excel / 飞书

### 导出 Excel

> 触发词：「导出」「导出Excel」「Excel」

读取并执行 `references/export-excel.md`。

- 使用 openpyxl
- 22 列表头（详见 export-excel.md）
- 文件名：`xhs_extract_{YYYYMMDD}_{HHMMSS}.xlsx`
- 当前笔记单条导出

### 导出飞书

> 触发词：「导出飞书」「飞书」「Feishu」

读取并执行 `references/export-feishu.md`。

- 检查 config 中 `export.feishu` 是否完整配置
- 未配置 → 「💡 飞书导出需要先配置，请参考 references/export-feishu-setup.md」
- 已配置 → 自动创建/更新飞书多维表格

---

## Step 8 · 帮助系统

> 触发词：「帮助」「怎么用」「help」「?」「使用说明」「功能」「能做什么」

```
📕 xhs-extract · 小红书笔记提取工具

📎 查笔记详情
   发小红书链接（xhslink / explore / discovery / 纯ID）
   自动识别图文/视频，展示作者、互动数、内容、图片

📝 提取文案（仅视频）
   说"提取文案" → 视频转文字
   ⚠️ 目前小红书未提供视频链接，此功能暂不可用

✏️ AI 改写
   说"改写" → 6种模式可选
   去重优化 / 总结精简 / 扩写丰富 / 转风格 / 提取金句 / 拆分多条

📥 导出 Excel
   说"导出" → 22列完整数据，一键生成 .xlsx

📋 导出飞书
   说"导出飞书" → 自动写入飞书多维表格（需先配置）

📄 批量查询
   发多条链接或上传文件（.txt/.csv/.xlsx/.docx）

💬 示例
   http://xhslink.com/o/8lsiyy1t7Ay
   https://www.xiaohongshu.com/explore/6a56e1aa000000002103db4f
   6a56e1aa000000002103db4f

📞 遇到问题？
   作者：阿南 rockmoons
   微信：rockmoons
   网站：www.rockmoons.com
```

---

## 附录：完整字段映射表

| 展示名 | API字段（图文接口） | API字段（视频接口） | 说明 |
|--------|-------------------|-------------------|------|
| 作者头像 | 作者头像 | 作者头像 | |
| 作者昵称 | 作者昵称 | 作者昵称 | |
| 账号ID | 账号id | 账号id | |
| 作者ID | 作者id | 用户id | 同一字段不同名 |
| IP归属地 | ip地址 | ip地址 | |
| 发布时间 | 发布时间 | 发布时间 | Unix时间戳→日期 |
| 作品简介 | 作品简介 | 作品简介 | 视频笔记取视频接口版 |
| 作品封面 | 作品封面 | 作品封面 | |
| 作品图片 | 作品图片 | — | JSON数组字符串 |
| 图片数量 | 图片数量 | — | 字符串类型 |
| 点赞数 | 点赞数 | 点赞数 | 优先视频接口 |
| 收藏数 | 收藏数 | 收藏数 | 优先视频接口 |
| 评论数 | 评论数 | 评论数 | 优先视频接口 |
| 分享数 | 分享数 | 分享数 | 优先视频接口 |
| 话题 | 话题 | 话题 | |
| @的用户 | — | @的用户 | 视频独有 |
| 笔记ID | 笔记id | 笔记id | |
| 笔记类型 | 笔记类型 | 笔记类型 | normal/video |
| 网页链接 | 网页链接 | 网页链接 | |
| 分享链接 | 分享链接 | 分享链接 | 优先图文接口 |
| 微信分享描述 | 微信分享描述 | 微信分享描述 | 可能含换行 |
| 背景音乐ID | — | 背景音乐ID | 视频独有 |
