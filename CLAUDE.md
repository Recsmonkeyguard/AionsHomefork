# CLAUDE.md — AionsHome VV 定制版

> AI Agent 入口说明书。任何新窗口先读此文件（~250 行），无需遍历全部源码。

---

## 项目身份

从 GitHub [AionsHome](https://github.com/death34018-hue/AionsHome) fork 的本地 AI 伴侣系统。当前跑在 vv 的 Mac 上，已做 macOS 适配 + 模型供应商替换。

## 一句话

**Python FastAPI 后端 + SQLite 数据库 + 原生 JS 前端 + WebSocket 多端同步**，服务监听 `0.0.0.0:8080`。

---

## 目录结构速览

```
AionsHomefork/
├── aion-chat/                    ← ★ 核心代码，99% 工作在这里
│   ├── main.py                   ← 入口：FastAPI 创建、lifespan、路由注册
│   ├── config.py                 ← 全局配置：路径、Key、模型列表、世界书
│   ├── database.py               ← SQLite 建表（启动时自动执行）
│   ├── ws.py                     ← WebSocket 连接管理（ConnectionManager 单例）
│   ├── ai_providers.py           ← AI 调用核心：stream_ai() 统一分发到各 provider
│   ├── memory.py                 ← 记忆引擎：Embedding、RAG 召回、即时哨兵、自动总结
│   ├── camera.py                 ← 摄像头采集 + Sentinel 监控哨兵（Mac 不可用）
│   ├── voice.py                  ← 语音唤醒 + 半双工通话（WebRTC VAD）
│   ├── tts.py                    ← 服务端流式 TTS（硅基流动 CosyVoice2）
│   ├── location.py               ← 高德定位 + GPS 心跳处理
│   ├── schedule.py               ← 日程/闹铃管理器
│   ├── activity.py               ← PC 前台窗口采集（Win 专属，Mac 禁用）
│   ├── gift.py / fund.py / book.py / music.py / image_gen.py / ghost_forest.py
│   ├── routes/                   ← ★ API 路由层，一个功能一个文件
│   │   ├── chat.py               ← 核心：发消息、重新生成、对话 CRUD
│   │   ├── settings.py           ← 设置、世界书、TTS、模型列表
│   │   ├── memories.py           ← 记忆库 CRUD + 手动总结触发
│   │   ├── music.py / cam.py / voice.py / schedule.py / location.py
│   │   ├── activity.py / heart_whispers.py / gift.py / fund.py
│   │   ├── book.py / theater.py / ghost_forest.py / wallpaper.py
│   │   └── files.py              ← 上传、聊天记录导出
│   ├── static/                   ← ★ 前端，原生 JS 无框架
│   │   ├── chat.html             ← 主聊天页（最大最复杂的页面）
│   │   ├── home.html             ← 手机风格主页（图标启动器）
│   │   ├── memory.html           ← 记忆库管理页
│   │   ├── settings.html         ← 设置页（API Key 管理）
│   │   ├── common.css / common.js← 子页面共享样式和工具函数
│   │   └── ...                   ← 其余 10+ 子页面
│   ├── data/                     ← ★ 所有数据，备份只拷此目录
│   │   ├── chat.db               ← SQLite 数据库（对话/消息/记忆/日程/礼物）
│   │   ├── settings.json         ← API Key 持久化
│   │   ├── worldbook.json        ← AI/用户人设
│   │   ├── digest_anchor.json    ← 记忆总结锚点（时间戳）
│   │   └── uploads/ chats/ screenshots/ tts_cache/ ...
│   └── requirements.txt
├── public/                       ← 静态资源（图标、壁纸、提示音）
├── AionApp/                      ← Android WebView 壳（Java）
├── vendor/                       ← 本地依赖库
├── CHANGELOG-VV.md               ← ★ VV 定制修改记录
└── CLAUDE.md                     ← ★ 本文件
```

---

## 架构：三层 + 一条总线

```
┌──────────────────────────────────────────────┐
│ 第一层：前端                                   │
│ static/*.html — 原生 JS，每个页面独立完整       │
│ common.js: api() 封装、WS 连接、通知/弹窗       │
└────────────────┬─────────────────────────────┘
                 │ HTTP REST + SSE + WebSocket
┌────────────────▼─────────────────────────────┐
│ 第二层：路由 (routes/*.py)                     │
│ 接收请求 → 调用业务模块 → 返回数据              │
│ 规律: /api/xxx → routes/xxx.py                 │
└────────────────┬─────────────────────────────┘
                 │
┌────────────────▼─────────────────────────────┐
│ 第三层：业务模块 (根目录 *.py)                   │
│ ai_providers / memory / camera / voice / ...  │
└────────────────┬─────────────────────────────┘
                 │ httpx 异步请求
┌────────────────▼─────────────────────────────┐
│ 外部 API: DeepSeek / 硅基流动 / Gemini / 高德   │
└──────────────────────────────────────────────┘

WebSocket (ws.py) 是贯穿所有层的消息总线
新消息、新记忆、闹铃、TTS 分片……全通过它广播到所有连接的设备
```

---

## 核心流程：发一条消息经过什么

```
用户点击发送
  → POST /api/conversations/{id}/send (SSE)
  → routes/chat.py → send_message()
      ├── ① instant_digest()     → RAG 路由（判断要不要搜记忆）
      ├── ② build_surfacing_memories() → 背景记忆浮现
      ├── ③ recall_memories()    → 向量+关键词混合检索（仅当①判定需要时）
      ├── ④ fetch_source_details() → 原文追溯（仅当①要求时）
      ├── ⑤ 组装 Prompt（人设+记忆+日程+位置+能力指令+聊天历史）
      ├── ⑥ stream_ai()          → ai_providers.py 分发到对应模型
      ├── ⑦ 后处理（检测 [MUSIC]/[ALARM]/[CAM_CHECK] 等指令）
      ├── ⑧ 存 DB + WebSocket 广播
      └── ⑨ 触发 TTS（若开启）
```

---

## 数据库核心表

全部在 `data/chat.db`（SQLite），`database.py` 启动时自动建表：

| 表 | 关键字段 | 说明 |
|----|---------|------|
| `conversations` | id, title, model, updated_at | 对话列表 |
| `messages` | id, conv_id, role, content, created_at | 消息（user/assistant/system） |
| `memories` | id, content, embedding(BLOB), keywords(JSON), importance, source_start_ts, source_end_ts, unresolved | 向量记忆 |
| `schedules` | id, type, trigger_at, content, status | 日程/闹铃/定时监控 |
| `gifts` | id, image_path, message, status | AI 送礼记录 |
| `heart_whispers` | id, conv_id, msg_id, content | AI 心语（秘密日记） |

记忆检索时全量读取 memories 表到内存做余弦相似度计算——SQLite 无向量插件，当前数据量（<万条）足够。

---

## API Key 全景

| Key 字段 | 来源 | 用途 |
|----------|------|------|
| `deepseek_key` | vv 的 DeepSeek 账号 | 聊天对话（V3/R1） |
| `deepseek_flash_key` | vv 的 DeepSeek 账号 | 即时哨兵 RAG 路由（v4-flash） |
| `siliconflow_key` | vv 的硅基流动账号 | Embedding（BGE）+ TTS + ASR |
| `gemini_key` | （留空） | Gemini 模型对话、AI 生图 |
| `gemini_free_key` | （留空） | 摄像头哨兵、位置哨兵 |
| `aipro_key` | （留空） | 第三方中转站 |

**Key 存储**：`data/settings.json` → 内存 `SETTINGS` dict → `config.get_key(provider)` 读取

---

## 当前魔改状态

详见 [CHANGELOG-VV.md](CHANGELOG-VV.md)。核心变更：

1. **macOS 适配**：跳过 pywin32，PC 活动采集禁用
2. **新增 DeepSeek Provider**：`ai_providers.py` 的 `call_deepseek()`，OpenAI 兼容格式
3. **Embedding 换硅基流动**：`BAAI/bge-large-zh-v1.5`，1024 维，替代 Gemini embedding
4. **即时哨兵换 DeepSeek**：`instant_digest()` 用 `deepseek-v4-flash`，替代 flash-lite
5. **pyncm stub**：音乐功能不可用

---

## 常见任务 → 先看哪个文件

| 你想做的事 | 入口文件 | 次要涉及 |
|-----------|---------|---------|
| 加新 AI 模型/Provider | ai_providers.py → 加 call_xxx() + stream_ai() 分支 | config.py → MODELS + get_key() |
| 改记忆库逻辑 | memory.py | database.py → 表结构 |
| 加新的 API Key | config.py + routes/settings.py + static/settings.html | 三件套 |
| 改聊天行为 | routes/chat.py（send_message 函数） | ai_providers.py |
| 改前端样式 | static/chat.html | static/common.css |
| 改设置页 | static/settings.html + routes/settings.py | — |
| 改数据库表结构 | database.py（加 CREATE TABLE / ALTER TABLE） | 对应的 routes 和业务模块 |
| 看有哪些 API 端点 | 翻 routes/ 目录即可，文件名对应 URL 路径 | — |
| 加新功能 | routes/ 新建文件 → main.py 注册路由 → static/ 建页面 | — |

---

## 注意事项

- **不要引入框架**：前端是原生 JS（无 React/Vue），后端是 FastAPI（无 ORM）
- **数据都在 data/**：备份拷走整个目录即可；数据库文件是 `chat.db` 不是 `aion.db`
- **WebSocket 是命脉**：多端同步靠它，`ws.py` 的 `manager.broadcast()` 到处被调用
- **启动指令**：`cd aion-chat && python3 main.py --reload`（改 .py 文件自动重启），服务在 `0.0.0.0:8080`
- **Mac vs Windows**：`activity.py`（win32gui）、摄像头 DirectShow 后端在 Mac 上不可用
- **Python 3.13**：pyncm 已从 PyPI 下架，stub 文件在 site-packages 下
- **修改记录**：每次改完代码更新 [CHANGELOG-VV.md](CHANGELOG-VV.md)，避免遗忘
