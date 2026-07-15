# xhs-extract · 批量处理

> 借鉴 dy-extract 的 batch-input.md，适配 xhs 的 note_id 提取逻辑。

---

## 输入场景

### 场景 A：聊天中多条链接

用户一次性粘贴多条链接（换行分隔）。

提取方式：按行分割 → 每行独立走 Step 1 的解析规则（支持 xhslink/explore/discovery/纯ID）。

### 场景 B：文件输入

用户发了文件，说"读这个文件" "处理这个表格"。

| 格式 | 读取方式 |
|------|---------|
| `.txt` | 按行读取，每行一个链接或ID |
| `.csv` | 读取所有列，自动识别含小红书链接/ID 的列 |
| `.xlsx` | 用 openpyxl 读取第一列（或其他指定列） |
| `.docx` | 用 python-docx 提取所有文本，正则匹配链接/ID |

## 去重

所有提取到的 note_id 做去重，防止重复查询消耗积分。

## 容量阈值

| 数量 | 行为 |
|------|------|
| ≤ 50 条 | 直接开始处理，无需确认 |
| > 50 条 | 估算时间和消耗，请用户确认 |

确认模板：

```
共 {N} 条笔记，预计约 {估算时间}，消耗约 {N×10} 积分。
将分 {批次} 批处理，每批 50 条。

确认继续吗？
```

> 积分说明：每条笔记查询消耗 10 积分。积分相关请咨询 www.rockmoons.com

## 执行循环

```python
# batch_process.py (概念代码)
# -*- coding: utf-8 -*-
import time, json

note_ids = [...]  # 去重后的ID列表
chunk_size = 50   # 从 config.batch.chunk_size 读取
interval = 0.5    # 从 config.batch.interval_ms / 1000 计算

total = len(note_ids)
success = []
failed = []

for i in range(0, total, chunk_size):
    chunk = note_ids[i:i+chunk_size]
    for note_id in chunk:
        try:
            # 调图文接口
            # 如果是 video 类型，再调视频接口
            # 记录结果到 success
            print(f"✅ [{i+1}/{total}] {note_id} 查询成功")
        except Exception as e:
            failed.append((note_id, str(e)))
            print(f"❌ [{i+1}/{total}] {note_id} 查询失败: {e}")
        time.sleep(interval)
```

## 进度反馈

每处理一条更新一次进度：

```
📊 处理进度：12/50 (24%)
✅ 成功：11  |  ❌ 失败：1
```

## 中途停止

用户说"停" "取消" "够了" → 等当前条处理完成后停止，已处理的数据保留。

```
⏹️ 已暂停。已处理 {done}/{total} 条。

💡 导出已查询的结果？ 继续处理剩余？
```

## 完成汇总

```
✅ 批量处理完成！

📊 总计：{total} 条
✅ 成功：{success_count} 条
❌ 失败：{fail_count} 条

{如有失败}
失败明细：
1. {note_id} → {错误原因}
2. ...

💡 导出到 Excel？ 导出飞书？
```
