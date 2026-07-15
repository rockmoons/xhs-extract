# xhs-extract · 本地 ASR 提取文案

> 注意：目前 xhs API 返回的 `data.video_urls` 为空数组，暂无可下载的视频链接。
> 此文件保留以备接口升级后使用。当前运行时需先检查 `video_urls` 是否有内容。

---

## 前置条件

- Python 3.8+
- 首次运行会自动安装 `faster-whisper` 和 `imageio-ffmpeg`
- 模型首次下载约 500MB（HF 镜像：`hf-mirror.com`）

## 流程

### 1. 检查环境

```bash
python3 -c "import faster_whisper" 2>/dev/null && echo "ok" || echo "need_install"
```

如未安装，执行：

```bash
pip install faster-whisper imageio-ffmpeg -i https://pypi.tuna.tsinghua.edu.cn/simple
```

设置 HuggingFace 镜像（加速模型下载）：

```bash
export HF_ENDPOINT=https://hf-mirror.com
```

### 2. 下载音频

从 `video_urls[0]` 下载视频并提取音频：

```python
# download_audio.py
# -*- coding: utf-8 -*-
import urllib.request
import sys
import os

video_url = sys.argv[1]
audio_path = sys.argv[2]

# 下载视频
urllib.request.urlretrieve(video_url, "temp_video.mp4")

# 用 ffmpeg 提取音频
os.system(f'ffmpeg -i temp_video.mp4 -vn -acodec pcm_s16le -ar 16000 -ac 1 "{audio_path}" -y >nul 2>&1')

# 清理
os.remove("temp_video.mp4")
print(f"音频已保存: {audio_path}")
```

### 3. 运行 Whisper

```python
# whisper_transcribe.py
# -*- coding: utf-8 -*-
import sys
from faster_whisper import WhisperModel

audio_path = sys.argv[1]
model_size = sys.argv[2]  # e.g. "Systran/faster-whisper-small"
language = sys.argv[3]     # e.g. "zh"

model = WhisperModel(model_size, device="cpu", compute_type="int8")
segments, info = model.transcribe(audio_path, language=language)

for segment in segments:
    print(f"[{segment.start:.2f}s -> {segment.end:.2f}s] {segment.text}")
```

### 4. 临时文件清理

```bash
if [ -f "temp_audio.wav" ]; then rm temp_audio.wav; fi
if [ -f "temp_video.mp4" ]; then rm temp_video.mp4; fi
if [ -f "download_audio.py" ]; then rm download_audio.py; fi
if [ -f "whisper_transcribe.py" ]; then rm whisper_transcribe.py; fi
```

## 常见错误

| 错误 | 原因 | 解决 |
|------|------|------|
| `pip install` 失败 | 网络问题 | 换镜像源 `-i https://pypi.tuna.tsinghua.edu.cn/simple` |
| `No module named 'faster_whisper'` | 未安装 | 执行安装命令 |
| 模型下载超时 | 网络慢/墙 | 设置 `HF_ENDPOINT=https://hf-mirror.com` |
| OOM / MemoryError | 内存不足 | 换更小模型如 `tiny` |
| `ffmpeg` 未找到 | 未安装 | `pip install imageio-ffmpeg` |
| CPU 运行很慢 | 无 GPU | 正常，首次运行需等待 |
