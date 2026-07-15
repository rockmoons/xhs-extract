# xhs-extract · API 参考

## 基本信息

- **平台：** BladeX-Links（www.rockmoons.com）
- **认证方式：** 查询参数 `apikey`
- **基础 URL：** `https://api.rockmoons.com/blade-links/openapi/plugin`
- **目前可用的 xhs 接口：** 2 个

---

## API 01 · 图文接口（通杀类型）

**路由：** `xhs_fetch_image_note_info`

**参数：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `apikey` | 是 | API 密钥 |
| `note_id` | 是 | 小红书笔记 ID（纯ID，不支持完整链接） |

**请求示例：**
```bash
curl -s --connect-timeout 10 --max-time 30 \
  "https://api.rockmoons.com/blade-links/openapi/plugin/xhs_fetch_image_note_info?note_id=6a56e1aa000000002103db4f&apikey=xxx"
```

**返回字段（20 个）：**

| # | 字段名 | 类型 | 说明 |
|---|--------|------|------|
| 1 | 作者头像 | string | 头像URL |
| 2 | 作者昵称 | string | |
| 3 | 账号id | string | 数字型账号ID |
| 4 | 作者id | string | 作者哈希ID |
| 5 | ip地址 | string | 发布IP归属地 |
| 6 | 发布时间 | int | Unix秒级时间戳 |
| 7 | 作品简介 | string | ⚠️ 视频笔记会被截断（只剩#标签） |
| 8 | 作品封面 | string | 封面图URL |
| 9 | 作品图片 | string | JSON数组字符串，如 `["url1","url2"]` |
| 10 | 图片数量 | string | 字符串类型，可能为 `""` |
| 11 | 点赞数 | int | |
| 12 | 收藏数 | int | |
| 13 | 评论数 | int | |
| 14 | 分享数 | int | |
| 15 | 话题 | string | |
| 16 | 笔记id | string | |
| 17 | 笔记类型 | string | `normal`=图文, `video`=视频 |
| 18 | 网页链接 | string | discovery/item 格式 |
| 19 | 分享链接 | string | 含追踪参数 |
| 20 | 微信分享描述 | string | 可能含 `\n` |

**使用场景：**
- 通杀两种笔记类型
- 先调此接口判断类型
- 如果是 `video` 再补调视频接口

---

## API 02 · 视频接口（仅视频笔记）

**路由：** `xhs_fetch_video_note_info`

**参数：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `apikey` | 是 | API 密钥 |
| `note_id` | 是 | 小红书笔记 ID（纯ID） |

**请求示例：**
```bash
curl -s --connect-timeout 10 --max-time 30 \
  "https://api.rockmoons.com/blade-links/openapi/plugin/xhs_fetch_video_note_info?note_id=6a56e1aa000000002103db4f&apikey=xxx"
```

**返回字段（20 个）：**

| # | 字段名 | 类型 | 说明 |
|---|--------|------|------|
| 1 | 作者头像 | string | |
| 2 | 作者昵称 | string | |
| 3 | 账号id | string | |
| 4 | 用户id | string | 与图文接口的"作者id"值一致 |
| 5 | ip地址 | string | |
| 6 | 发布时间 | int | |
| 7 | 作品简介 | string | ✅ 完整版（含正文+标签） |
| 8 | 作品封面 | string | |
| 9 | 点赞数 | int | |
| 10 | 收藏数 | int | |
| 11 | 评论数 | int | |
| 12 | 分享数 | int | |
| 13 | 话题 | string | |
| 14 | @的用户 | string | 视频独有 |
| 15 | 笔记id | string | |
| 16 | 笔记类型 | string | |
| 17 | 网页链接 | string | |
| 18 | 分享链接 | string | ⚠️ 返回空字符串 |
| 19 | 微信分享描述 | string | |
| 20 | 背景音乐ID | string | 视频独有 |

**使用场景：**
- 仅限视频笔记（传图文ID会返回错误数据）
- 用来补全作品简介（完整版）、获取 @用户、背景音乐ID

---

## 通用响应结构

所有 API 返回统一格式：

```json
{
  "code": 200,
  "success": true,
  "msg": "操作成功",
  "data": {
    "feishu": {
      "table_fields": "[...]",
      "records": [{ "fields": "{...}" }]
    },
    "video_urls": [],
    "metadata": {
      "fields": "{ \"作者昵称\": \"...\", \"点赞数\": 1050, ... }"
    },
    "info": "咨询www.rockmoons.com"
  }
}
```

**关键点：**
- `data.metadata.fields` 是 **JSON 字符串**，需要 `json.loads()` 解析
- `data.video_urls` 目前始终为空数组（暂无视频下载链接）
- `data.feishu` 是飞书导出用的预格式化数据

## 错误码

| code | msg | 说明 |
|------|-----|------|
| 200 | 操作成功 | 正常 |
| 400 | 缺少APIKey | apikey 参数缺失 |
| 400 | 上游调用失败: HTTP xxx | 上游小红书接口异常 |
| 400 | 插件不存在或已禁用 | 接口未部署 |
