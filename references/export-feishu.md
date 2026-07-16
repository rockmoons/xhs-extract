# xhs-extract · 导出飞书

> **已验证：** 应用通过 `tenant_access_token` 创建的多维表格，自动拥有完整读写权限，无需额外授权。

---

## 前置检查

从 config.json 读取 `export.feishu`：

```bash
feishu_app_id=$(python -c "import json; print(json.load(open('config.json'))['export']['feishu']['app_id'])")
feishu_app_secret=$(python -c "import json; print(json.load(open('config.json'))['export']['feishu']['app_secret'])")
```

如果 `app_id` 或 `app_secret` 为空 → 引导用户：

```
💡 飞书导出需要先配置，请按以下步骤操作：

1. 打开 https://open.feishu.cn/app
2. 创建企业自建应用
3. 开启「多维表格」和「云文档」权限
4. 获取 App ID 和 App Secret
5. 填到 config.json 的 export.feishu 下

详细教程见 references/export-feishu-setup.md
```

---

## 实现流程

### 1. 获取 Token

```python
# feishu_token.py
# -*- coding: utf-8 -*-
import requests, json, sys

app_id = sys.argv[1]
app_secret = sys.argv[2]

resp = requests.post(
    "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
    json={"app_id": app_id, "app_secret": app_secret}
)
data = resp.json()
print(data.get("tenant_access_token", ""))
```

### 2. 创建多维表格（首次使用）

如果 `bitable_id` 为空，自动创建并建表：

```python
# feishu_create.py
# -*- coding: utf-8 -*-
import requests, json, sys

token = sys.argv[1]

# 创建多维表格
resp = requests.post(
    "https://open.feishu.cn/open-apis/bitable/v1/apps",
    headers={"Authorization": f"Bearer {token}"},
    json={"name": "小红书笔记数据"}
)
data = resp.json()
bitable_id = data["data"]["app"]["app_token"]
print(f"BITABLE_ID={bitable_id}")

# 建表（一次性定义全部22个字段）
fields = [
    {"field_name": "笔记ID", "type": 1},
    {"field_name": "笔记类型", "type": 1},
    {"field_name": "作者昵称", "type": 1},
    {"field_name": "作者头像", "type": 1},
    {"field_name": "账号ID", "type": 1},
    {"field_name": "作者ID", "type": 1},
    {"field_name": "IP归属地", "type": 1},
    {"field_name": "发布时间", "type": 1},
    {"field_name": "作品简介", "type": 1},
    {"field_name": "作品封面", "type": 1},
    {"field_name": "作品图片", "type": 1},
    {"field_name": "图片数量", "type": 1},
    {"field_name": "点赞数", "type": 2},
    {"field_name": "收藏数", "type": 2},
    {"field_name": "评论数", "type": 2},
    {"field_name": "分享数", "type": 2},
    {"field_name": "话题", "type": 1},
    {"field_name": "@的用户", "type": 1},
    {"field_name": "背景音乐ID", "type": 1},
    {"field_name": "网页链接", "type": 1},
    {"field_name": "分享链接", "type": 1},
    {"field_name": "微信分享描述", "type": 1},
]

resp = requests.post(
    f"https://open.feishu.cn/open-apis/bitable/v1/apps/{bitable_id}/tables",
    headers={"Authorization": f"Bearer {token}"},
    json={"table": {"name": "笔记列表", "fields": fields}}
)
data = resp.json()
table_id = data["data"]["table_id"]
print(f"TABLE_ID={table_id}")
```

### 3. 写入记录

```python
# feishu_write.py
# -*- coding: utf-8 -*-
import requests, json, sys
from datetime import datetime, timezone, timedelta

token = sys.argv[1]
bitable_id = sys.argv[2]
table_id = sys.argv[3]
records_json = sys.argv[4]  # JSON 字符串，包含所有记录

records = json.loads(records_json)

# 处理每条记录
rows = []
for rec in records:
    # 处理时间戳
    ts = rec.get("发布时间", 0)
    if isinstance(ts, (int, float)) and ts > 0:
        dt = datetime.fromtimestamp(ts, tz=timezone(timedelta(hours=8)))
        create_time = dt.strftime("%Y-%m-%d %H:%M")
    else:
        create_time = str(ts)
    
    row = {
        "fields": {
            "笔记ID": rec.get("笔记id", ""),
            "笔记类型": rec.get("笔记类型", ""),
            "作者昵称": rec.get("作者昵称", ""),
            "作者头像": rec.get("作者头像", ""),
            "账号ID": rec.get("账号id", ""),
            "作者ID": rec.get("作者id") or rec.get("用户id", ""),
            "IP归属地": rec.get("ip地址", ""),
            "发布时间": create_time,
            "作品简介": rec.get("作品简介", ""),
            "作品封面": rec.get("作品封面", ""),
            "作品图片": rec.get("作品图片", ""),
            "图片数量": rec.get("图片数量", ""),
            "点赞数": rec.get("点赞数", 0),
            "收藏数": rec.get("收藏数", 0),
            "评论数": rec.get("评论数", 0),
            "分享数": rec.get("分享数", 0),
            "话题": rec.get("话题", ""),
            "@的用户": rec.get("@的用户", ""),
            "背景音乐ID": rec.get("背景音乐ID", ""),
            "网页链接": rec.get("网页链接", ""),
            "分享链接": rec.get("分享链接", ""),
            "微信分享描述": rec.get("微信分享描述", "").replace("\n", " "),
        }
    }
    rows.append(row)

# 每批最多写500条
batch_size = 500
url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{bitable_id}/tables/{table_id}/records/batch_create"

for i in range(0, len(rows), batch_size):
    batch = rows[i:i+batch_size]
    resp = requests.post(
        url,
        headers={"Authorization": f"Bearer {token}"},
        json={"records": batch}
    )
    if resp.status_code != 200:
        print(f"❌ 写入失败: {resp.text}")
        break

print(f"✅ 成功写入 {len(rows)} 条记录到飞书")
```

## 完成提示

```
✅ 已导出到飞书多维表格「小红书笔记数据」
📊 共 {N} 条笔记记录
📋 打开飞书 → 云文档 → 即可查看
```
