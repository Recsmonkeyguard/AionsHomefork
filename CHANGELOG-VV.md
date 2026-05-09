# VV 定制版修改记录

> 基于 AionsHome fork，macOS 适配 + 个性化魔改

---

## 2026-05-08 — 初次启动 & macOS 兼容

### 依赖安装
- 跳过 `pywin32`（Windows 专属，Mac 不可用）
- `sounddevice` 成功安装
- `opencv-python` 成功安装
- 其余 Python 依赖正常安装

### pyncm 替代
- **问题**：`pyncm` 已从 PyPI 下架，且 GitHub 仓库无法访问
- **方案**：在 Python site-packages 下手动创建 stub 模块
- **文件**：
  - `/Library/Frameworks/Python.framework/Versions/3.13/lib/python3.13/site-packages/pyncm/__init__.py`
  - `/Library/Frameworks/Python.framework/Versions/3.13/lib/python3.13/site-packages/pyncm/apis/__init__.py`
  - `/Library/Frameworks/Python.framework/Versions/3.13/lib/python3.13/site-packages/pyncm/apis/login.py`
  - `/Library/Frameworks/Python.framework/Versions/3.13/lib/python3.13/site-packages/pyncm/apis/cloudsearch.py`
  - `/Library/Frameworks/Python.framework/Versions/3.13/lib/python3.13/site-packages/pyncm/apis/track.py`
- **影响**：音乐点歌功能不可用，其余正常

### 已知 Mac 限制
- PC 活动采集不可用（依赖 win32gui）
- 摄像头监控可能需调整（DirectShow 是 Windows 后端）

---

## 2026-05-08 — 新增 DeepSeek Provider

### 涉及文件

| 文件 | 改动 |
|------|------|
| `aion-chat/config.py` | `load_settings()` 默认 key 加入 `deepseek_key`；`get_key()` 新增 `"deepseek"` 分支；`MODELS` 字典新增 `DeepSeek-V3`、`DeepSeek-R1` |
| `aion-chat/ai_providers.py` | 新增 `call_deepseek()` 函数（OpenAI 兼容格式，180s 超时）；`stream_ai()` 新增 `"deepseek"` provider 分发 |
| `aion-chat/routes/settings.py` | `SettingsUpdate` 模型新增 `deepseek_key` 字段；`get_settings` 响应新增 `deepseek_key` / `deepseek_key_masked`；`update_settings` 新增 `deepseek_key` 处理 |
| `aion-chat/static/settings.html` | 新增 DeepSeek API Key 输入框 + JS 读写逻辑 |

### 新增模型
- **DeepSeek-V3** → `deepseek-chat`
- **DeepSeek-R1** → `deepseek-reasoner`

### API 端点
- `https://api.deepseek.com/v1/chat/completions`（OpenAI 兼容）

---

## 2026-05-09 — Embedding 从 Gemini 切换到硅基流动

### 改动文件
| 文件 | 改动 |
|------|------|
| `aion-chat/memory.py` | `get_embedding()` API 从 Gemini `embedContent` 改为硅基流动 `POST /v1/embeddings`（OpenAI 兼容格式）；`EMBEDDING_MODEL` → `BAAI/bge-large-zh-v1.5`；`EMBEDDING_DIMS` 3072 → 1024；Key 从 `gemini_free` 改为 `siliconflow` |

### 变更详情
- **旧**：Gemini `gemini-embedding-001`，3072 维，Google REST API
- **新**：硅基流动 `BAAI/bge-large-zh-v1.5`，1024 维，OpenAI 兼容 `/v1/embeddings`
- **Key**：复用设置页已有的「硅基流动 API Key」，无需新增字段
- **效果**：单条向量 12KB → 4KB，余弦计算快 ~3 倍，中文语义更精准

---

## 当前系统 API Key 一览

| Key | 用途 | 是否必须 |
|-----|------|----------|
| DeepSeek Key | 对话（DeepSeek-V3/R1） | ✅ vv 主要使用 |
| 硅基流动 Key | Embedding、TTS、ASR、硅基模型 | ✅ vv 使用（Embedding） |
| Gemini Key | Gemini 模型对话、AI 生图 | ❌ 暂不用 |
| Gemini Free Key | 哨兵分析、即时哨兵 | ⚠️ 仍需 Gemini！ |
| 中转站 Key | 第三方中转（Claude/GPT） | ❌ 暂不用 |

---

## 2026-05-09 — 即时哨兵从 Gemini 切换到 DeepSeek v4-flash

### 改动文件

| 文件 | 改动 |
|------|------|
| `aion-chat/config.py` | `load_settings()` 默认 key 加入 `deepseek_flash_key`；`get_key()` 新增 `"deepseek_flash"` 分支 |
| `aion-chat/routes/settings.py` | `SettingsUpdate` 模型新增 `deepseek_flash_key`；`get_settings` 响应新增 `deepseek_flash_key` / `deepseek_flash_key_masked`；`update_settings` 新增处理 |
| `aion-chat/static/settings.html` | 新增「DeepSeek Flash Key（哨兵/记忆路由）」输入框 + JS 读写 |
| `aion-chat/memory.py` | `instant_digest()` 从 Gemini `generateContent` 改为 DeepSeek OpenAI 兼容 `POST /v1/chat/completions`；模型从 `gemini-3.1-flash-lite-preview` 改为 `deepseek-v4-flash`；Key 从 `gemini_free` 改为 `deepseek_flash`（与聊天 Key 分离） |

### 变更详情
- **旧**：Gemini `gemini-3.1-flash-lite-preview`，Google REST API
- **新**：DeepSeek `deepseek-v4-flash`，OpenAI 兼容 `/v1/chat/completions`，284B/13B 激活参数
- **Key**：新增独立 `deepseek_flash_key`，与聊天用的 `deepseek_key` 分离
- **注意**：`instant_digest()` 每次发消息都触发，影响记忆检索路由

---

## 当前系统 API Key 一览

| Key | 用途 | 是否必须 |
|-----|------|----------|
| DeepSeek Key | 对话（DeepSeek-V3/R1） | ✅ vv 使用 |
| DeepSeek Flash Key | 即时哨兵/记忆路由（RAG 守门） | ✅ vv 使用 |
| 硅基流动 Key | Embedding、TTS、ASR | ✅ vv 使用（Embedding） |
| Gemini Key | Gemini 模型对话、AI 生图 | ❌ 暂不用 |
| Gemini Free Key | 哨兵截图分析、位置哨兵 | ⚠️ 仍为 Gemini！ |
| 中转站 Key | 第三方中转（Claude/GPT） | ❌ 暂不用 |

---

## 待办

- [x] ~~Embedding 从 Gemini 换到硅基流动~~ → `BAAI/bge-large-zh-v1.5`（1024 维）
- [x] ~~即时哨兵从 Gemini 换到 DeepSeek~~ → `deepseek-v4-flash`
- [ ] Sentinel 截图分析（camera.py:711）仍为 Gemini flash-lite → Mac 上暂不可用，不急
- [ ] 位置哨兵（location.py:704）仍为 Gemini flash-lite → 未配置 GPS，不急
- [ ] TTS / ASR 选型（目前绑死硅基流动 CosyVoice2 / SenseVoiceSmall）
- [ ] 音乐功能恢复（pyncm 替代方案）
