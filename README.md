# lazy-video-helper

懒猫微服包装：[LDJ-creat/video-helper](https://github.com/LDJ-creat/video-helper) — 基于 AI 的视频学习助手。

输入 B 站 / YouTube / TikTok 视频链接（或上传本地文件），自动产出：

- **思维导图**（节点可缩放、拖拽、增删）
- **结构化重点摘要**（点摘要跳转视频对应时间戳）
- **AI 多轮问答**（基于视频内容）
- **练习画布**（AI 自动出题）

技术栈：Next.js 16 + FastAPI (Python 3.12) + faster-whisper + yt-dlp。

## 安装

懒猫商店搜 "Video Helper" 一键安装。

第一次启动会下载 faster-whisper 模型权重（数百 MB），需要等几分钟。

## 配置

装好后访问 `https://video-helper.{你的微服域名}`，进入 Settings 填：

- LLM API Key（OpenAI / 通义千问 / DeepSeek / Ollama 任意 OpenAI 兼容接口）
- LLM Base URL
- LLM Model
- Whisper 模型（CPU 建议 `tiny` 或 `base`）

也可以在安装时通过 Deploy Params 提前填入。

## 数据持久化

`/lzcapp/var/data/` —— SQLite 数据库 + 视频缓存 + 转录文本 + 思维导图 + faster-whisper 模型权重。

备份直接 tar 这个目录即可。

## 开发

参考 [CLAUDE.md](./CLAUDE.md) 了解仓库结构、patch 工作流和发布流程。

## License

MIT —— 同上游。
