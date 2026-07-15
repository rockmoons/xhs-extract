# xhs-extract · 导出 Excel

---

## 前置条件

```bash
pip show openpyxl > /dev/null 2>&1 || pip install openpyxl -i https://pypi.tuna.tsinghua.edu.cn/simple
```

## 表头定义（22 列）

| 序号 | 列名 | 数据来源 | 格式/说明 |
|------|------|---------|----------|
| 1 | 笔记ID | `笔记id` | 文本 |
| 2 | 笔记类型 | `笔记类型` | normal / video |
| 3 | 作者昵称 | `作者昵称` | 文本 |
| 4 | 作者头像 | `作者头像` | HYPERLINK 公式 |
| 5 | 账号ID | `账号id` | 文本 |
| 6 | 作者ID | `作者id` 或 `用户id` | 文本 |
| 7 | IP归属地 | `ip地址` | 文本 |
| 8 | 发布时间 | `发布时间` | YYYY-MM-DD HH:MM |
| 9 | 作品简介 | `作品简介` | 文本 |
| 10 | 作品封面 | `作品封面` | HYPERLINK 公式 |
| 11 | 作品图片 | `作品图片` | HYPERLINK（取数组第一张） |
| 12 | 图片数量 | `图片数量` | 文本（字符串类型） |
| 13 | 点赞数 | `点赞数` | 千位分隔数字 |
| 14 | 收藏数 | `收藏数` | 千位分隔数字 |
| 15 | 评论数 | `评论数` | 千位分隔数字 |
| 16 | 分享数 | `分享数` | 千位分隔数字 |
| 17 | 话题 | `话题` | 文本 |
| 18 | @的用户 | `@的用户` | 文本（视频独有） |
| 19 | 背景音乐ID | `背景音乐ID` | 文本（视频独有） |
| 20 | 网页链接 | `网页链接` | HYPERLINK 公式 |
| 21 | 分享链接 | `分享链接` | HYPERLINK 公式 |
| 22 | 微信分享描述 | `微信分享描述` | 文本 |

## Python 实现

```python
# export_xhs_excel.py
# -*- coding: utf-8 -*-
import json
import openpyxl
from openpyxl.styles import Font, Alignment, numbers
from datetime import datetime, timezone, timedelta

def export_to_excel(record, filename=None):
    """
    record: dict，包含所有字段的字典（合并后的数据）
    filename: 可选，默认自动生成
    """
    wb = openpyxl.Workbook()
    ws = wb.active
    ws.title = "小红书笔记"

    # 表头
    headers = [
        "笔记ID", "笔记类型", "作者昵称", "作者头像", "账号ID", "作者ID",
        "IP归属地", "发布时间", "作品简介", "作品封面", "作品图片", "图片数量",
        "点赞数", "收藏数", "评论数", "分享数", "话题", "@的用户",
        "背景音乐ID", "网页链接", "分享链接", "微信分享描述"
    ]

    # 写入表头
    for col, header in enumerate(headers, 1):
        cell = ws.cell(row=1, column=col, value=header)
        cell.font = Font(bold=True)

    # 处理时间戳
    ts = record.get("发布时间", 0)
    if isinstance(ts, (int, float)) and ts > 0:
        dt = datetime.fromtimestamp(ts, tz=timezone(timedelta(hours=8)))
        create_time = dt.strftime("%Y-%m-%d %H:%M")
    else:
        create_time = str(ts)

    # 处理作品图片（JSON数组 → 取第一张）
    images_raw = record.get("作品图片", "")
    first_image = ""
    if images_raw:
        try:
            images = json.loads(images_raw)
            if isinstance(images, list) and len(images) > 0:
                first_image = images[0]
        except:
            first_image = str(images_raw)

    # 处理分享链接（视频接口可能为空，用图文接口的值）
    share_link = record.get("分享链接", "")

    # 处理@的用户
    at_users = record.get("@的用户", "")

    # 处理背景音乐ID
    bgm_id = record.get("背景音乐ID", "")

    # 写入数据行
    row_data = [
        record.get("笔记id", ""),
        record.get("笔记类型", ""),
        record.get("作者昵称", ""),
        record.get("作者头像", ""),
        record.get("账号id", ""),
        record.get("作者id") or record.get("用户id", ""),
        record.get("ip地址", ""),
        create_time,
        record.get("作品简介", ""),
        record.get("作品封面", ""),
        first_image,
        record.get("图片数量", ""),
        record.get("点赞数", 0),
        record.get("收藏数", 0),
        record.get("评论数", 0),
        record.get("分享数", 0),
        record.get("话题", ""),
        at_users,
        bgm_id,
        record.get("网页链接", ""),
        share_link,
        record.get("微信分享描述", "").replace("\n", " "),
    ]

    for col, value in enumerate(row_data, 1):
        cell = ws.cell(row=2, column=col, value=value)
        # URL字段设HYPERLINK
        if col in [4, 10, 11, 20, 21]:  # 头像、封面、图片、网页链接、分享链接
            if value:
                cell.value = f'=HYPERLINK("{value}", "查看")'
                cell.font = Font(color="0563C1", underline="single")
        # 数字字段设千位分隔
        if col in [13, 14, 15, 16] and isinstance(value, (int, float)):
            cell.number_format = '#,##0'

    # 自动调整列宽
    for col in range(1, len(headers) + 1):
        ws.column_dimensions[openpyxl.utils.get_column_letter(col)].width = 18

    # 文件名
    if not filename:
        now = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"xhs_extract_{now}.xlsx"

    wb.save(filename)
    return filename
```

## 文件名

`xhs_extract_{YYYYMMDD}_{HHMMSS}.xlsx`

## 完成提示

```
✅ 已导出到 {filename}
📊 共 1 条笔记记录
💡 双击打开～
```

## 批量导出

如果是从 Step 6（批量处理）进入，每条笔记一行，全部写入同一个 Sheet。

表头同上，数据行逐条追加。
