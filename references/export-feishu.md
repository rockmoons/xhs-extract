# xhs-extract · 导出飞书

---

## 前置检查

从 config.json 读取 `export.feishu`：

```bash
feishu_app_id=$(python3 -c "import json; print(json.load(open('config.json'))['export']['feishu']['app_id'])")
feishu_app_secret=$(python3 -c "import json; print(json.load(open('config.json'))['export']['feishu']['app_secret'])")
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

> 注意：这里的 export-feishu-setup.md 与 dy-extract 共用同一份教程，因为飞书配置流程完全相同。如有需要，可指引用户查看 dy-extract 的对应文件。

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

如果 `bitable_id` 和 `table_id` 为空，自动创建：

```python
# 创建多维表格
resp = requests.post(
    "https://open.feishu.cn/open-apis/drive/v1/metas/batch_create",
    headers={"Authorization": f"Bearer {token}"},
    json={
        "requests": [{
            "type": "bitable",
            "title": f"xhs_extract_{datetime.now().strftime('%Y%m%d')}"
        }]
    }
)
```

### 3. 添加应用为协作者

```python
# 添加协作者权限
bitable_url = f"https://open.feishu.cn/open-apis/drive/v1/permissions/{bitable_token}/members?need_notification=false"
requests.post(
    bitable_url,
    headers={"Authorization": f"Bearer {token}"},
    json={
        "member_type": "openid",
        "member_id": app_id,
        "perm": "full_access"
    }
)
```

### 4. 写入记录

```python
# feishu_write.py
# -*- coding: utf-8 -*-
import requests, json, sys
from datetime import datetime, timezone, timedelta

def write_to_feishu(token, app_id, bitable_id, table_id, records):
    """
    records: list of dict，每个dict是合并后的字段数据
    """
    url = f"https://open.feishu.cn/open-apis/bitable/v1/apps/{bitable_id}/tables/{table_id}/records/batch_create"
    
    rows = []
    for rec in records:
        # 处理时间戳
        ts = rec.get("发布时间", 0)
        if isinstance(ts, (int, float)) and ts > 0:
            dt = datetime.fromtimestamp(ts, tz=timezone(timedelta(hours=8)))
            create_time = dt.strftime("%Y-%m-%d %H:%M")
        else:
            create_time = str(ts)
        
        # 处理作品图片
        images_raw = rec.get("作品图片", "")
        first_image = ""
        if images_raw:
            try:
                images = json.loads(images_raw)
                if isinstance(images, list) and len(images) > 0:
                    first_image = images[0]
            except:
                first_image = str(images_raw)
        
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
                "作品图片": first_image,
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
    for i in range(0, len(rows), batch_size):
        batch = rows[i:i+batch_size]
        resp = requests.post(
            url,
            headers={"Authorization": f"Bearer {token}"},
            json={"records": batch}
        )
        if resp.status_code != 200:
            print(f"❌ 写入失败: {resp.text}")
    
    print(f"✅ 成功写入 {len(rows)} 条记录到飞书")
```

## 完成提示

```
✅ 已导出到飞书多维表格
📊 共 {N} 条笔记记录
📋 打开飞书即可查看
```
