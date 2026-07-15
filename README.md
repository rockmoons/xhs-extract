# 🌸 xhs-extract · 小红书笔记数据提取工具

> 一键提取小红书笔记详情 · 自动识别图文/视频 · AI 改写 · 导出 Excel / 飞书

---

## ⚡ 快速开始

1. 把 `xhs-extract` 文件夹放到你的 ZCode 技能目录
2. 打开 `config.json`，检查 `xhs.apikey` 是否已填写
3. 在聊天中发送小红书链接或 `/xhs-extract`

**就这么简单，零配置上手～**

---

## 🔧 环境要求

| 项目 | 要求 |
|------|------|
| Python | 3.8+（仅 ASR/导出时需要） |
| 硬盘 | ASR 首次需下载约 500MB 模型 |
| 网络 | 能访问 `api.rockmoons.com` |

> 纯查询笔记数据**不需要**本机 Python，只有「提取文案」「导出 Excel」「导出飞书」才需要。

---

## 🎯 功能一览

### 1️⃣ 查笔记详情

发小红书链接或笔记ID，自动识别图文/视频：

```
📷 橙柿互动 · ❤️1050 · ⭐190 · 💬469
   📍 Zhejiang · 🕐 2026-07-15 14:32

📝 邹市明创业投入不止2亿，原因是太讲情怀了
🏷️ #邹市明
🖼️ 共5张图片

💡 改写文案 | 导出Excel | 查下一个
```

### 2️⃣ 提取文案（视频笔记）

对视频笔记提取音频转文字（需有视频链接）。

### 3️⃣ AI 改写

支持 6 种改写模式：去重优化、总结精简、扩写丰富、转风格、提取金句、拆分多条。

### 4️⃣ 导出 Excel

一键导出 `.xlsx` 文件，22 列完整字段。

### 5️⃣ 导出飞书

配置飞书后，自动创建多维表格并写入数据。

### 6️⃣ 批量查询

支持多条链接或文件（.txt/.csv/.xlsx/.docx）批量导入。

---

## 📎 支持的红书链接格式

```
http://xhslink.com/o/xxx           ← 分享短链
https://www.xiaohongshu.com/explore/{id}  ← 网页链接
https://www.xiaohongshu.com/discovery/item/{id}  ← 分享链接
6a56e1aa000000002103db4f          ← 纯笔记ID
```

---

## ⚙️ 配置说明

```json
{
  "xhs": {
    "apikey": "你的API密钥",         ← 必填，从 www.rockmoons.com 获取
    "base_url": "https://api.rockmoons.com/blade-links/openapi/plugin"
  },
  "export": {
    "feishu": {
      "app_id": "",                 ← 飞书配置（可选）
      "app_secret": ""
    }
  }
}
```

---

## 🔌 获取 API Key

1. 打开 [www.rockmoons.com](https://www.rockmoons.com)
2. 注册并开通 BladeX-Links 平台
3. 获取你的 API Key
4. 填到 `config.json` 的 `xhs.apikey`

---

## 📞 联系作者

| 方式 | 信息 |
|------|------|
| 抖音 | 阿南 rockmoons |
| 微信 | rockmoons |
| 网站 | www.rockmoons.com |

---

## 🚧 说明

- **提取文案**：目前小红书接口未提供视频/音频下载链接，此功能需等接口升级后可用
- **批量查询**：每条笔记消耗 10 积分
- **飞书导出**：需要自行配置飞书应用，详见 `references/export-feishu-setup.md`
