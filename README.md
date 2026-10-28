# AI 私人厨师（Personal Chef）

> 上传一张冰箱/食材照片，AI 自动识别食材、评估新鲜度，并结合网络菜谱检索，为你流式输出结构化的食谱推荐报告。

基于 **FastAPI + LangChain 1.x + LangGraph** 构建的多模态 AI Agent 后端，集成阿里云 OSS 图片存储、Tavily 联网搜索与 DashScope 多模态大模型，并附带一个 Next.js 静态前端。

## ✨ 功能特性

- 📷 **食材图片识别**：前端直传图片到阿里云 OSS（预签名 URL，无需暴露密钥），多模态模型识别食材并评估新鲜度与可用量
- 🔍 **联网菜谱检索**：Agent 优先调用 Tavily 网络搜索获取真实菜谱，搜索不到时才自行发挥
- 💬 **流式对话**：基于 SSE（`text/event-stream`）逐 token 输出推荐结果，体验如打字机
- 🧠 **多会话记忆**：LangGraph `SqliteSaver` checkpoint 持久化对话历史，按 `thread_id` 隔离，支持历史查询与会话清空
- 🖼️ **一体化部署**：FastAPI 同时挂载 API 与前端静态资源，单进程即可访问完整应用

## 🏗️ 架构与技术栈

```
浏览器 (Next.js 静态前端 /static)
   │
   │  ① GET /api/v1/oss/presign  ──►  获取 OSS 预签名上传 URL
   │  ② PUT  uploadUrl            ──►  图片直传阿里云 OSS
   │  ③ POST /api/v1/chat/stream  ──►  FastAPI (main.py, :8001)
   │                                        │
   │                                        ▼
   │                              LangGraph create_agent
   │                              ├─ 模型: qwen3.5-plus (DashScope, OpenAI 兼容)
   │                              ├─ 工具: TavilySearch (联网搜菜谱)
   │                              └─ 记忆: SqliteSaver → db/personal_chief.db
   │
   └── SSE 流式返回食谱推荐报告 ◄──┘
```

| 组件 | 技术 | 版本 |
|---|---|---|
| Web 框架 | FastAPI | 0.162.0 |
| ASGI 服务器 | Uvicorn | 0.54.0 |
| Agent 框架 | LangChain / LangGraph | 1.4.7 / 1.2.18 |
| 模型接入 | langchain-openai（OpenAI 兼容协议接 DashScope） | 1.7.3 |
| 会话记忆 | langgraph-checkpoint-sqlite | 3.1.2 |
| 联网搜索 | langchain-tavily | 0.2.18 |
| 对象存储 | alibabacloud-oss-v2 | 1.4.1 |
| 配置管理 | python-dotenv | 1.2.4 |
| 数据校验 | Pydantic | 2.14.1 |
| Python | 建议 3.11+（开发环境 3.13） | |

## 📁 目录结构

```
sichu_backend/
├── main.py                  # FastAPI 入口：CORS、路由注册、静态前端挂载、启动配置
├── api/
│   └── v1/
│       ├── chat.py          # 对话接口：流式推荐 / 历史消息 / 清空会话
│       └── oss.py           # OSS 预签名上传接口
├── app/
│   └── agents/
│       └── personal_chief.py # 核心 Agent：模型、工具、记忆、系统提示词、流式流程
├── models/
│   └── schemas.py           # 请求模型 ChatRequest
├── common/
│   └── logger.py            # 统一日志配置（控制台输出）
├── db/
│   └── personal_chief.db    # LangGraph checkpoint 持久化（SQLite，自动创建）
├── static/                  # Next.js 导出的前端静态站点（SPA fallback）
├── .env                     # 环境变量（密钥配置，勿提交）
└── .gitignore
```

## 🚀 快速开始

### 1. 创建虚拟环境并安装依赖

```powershell
# Windows PowerShell
python -m venv .venv
.venv\Scripts\activate

pip install fastapi uvicorn langchain langchain-openai langchain-tavily `
            langgraph langgraph-checkpoint-sqlite alibabacloud-oss-v2 python-dotenv
```

### 2. 配置环境变量

在项目根目录创建 `.env`：

```ini
# ===== 大模型（DashScope / 阿里云百炼，OpenAI 兼容接口）=====
DASHSCOPE_API_KEY=sk-xxxx
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

# ===== 联网搜索（Tavily，https://www.tavily.com 获取）=====
TAVILY_API_KEY=tvly-xxxx

# ===== 阿里云 OSS（食材图片存储）=====
OSS_ACCESS_KEY_ID=你的AccessKeyId
OSS_ACCESS_KEY_SECRET=你的AccessKeySecret
OSS_BUCKET=你的bucket名称
OSS_REGION=cn-beijing
OSS_ENDPOINT=oss-cn-beijing.aliyuncs.com

# ===== 可选：LangSmith 链路追踪 =====
LANGSMITH_API_KEY=
LANGSMITH_TRACING=false
LANGSMITH_PROJECT=

# ===== 可选：DeepSeek（备用模型）=====
DEEPSEEK_API_KEY=
```

### 3. 启动服务

```powershell
python main.py
# 或
uvicorn main:app --host 127.0.0.1 --port 8001 --reload
```

启动后访问：

- 前端页面：<http://127.0.0.1:8001/>
- 接口文档（Swagger）：<http://127.0.0.1:8001/docs>

## 📡 API 文档

### `POST /api/v1/chat/stream` — 流式菜谱推荐

请求体：

```json
{
  "message": "帮我看看能做什么菜",
  "image_url": "https://your-bucket.oss-cn-beijing.aliyuncs.com/food.jpg",
  "thread_id": "session-001"
}
```

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `message` | string | 是 | 用户文本输入 |
| `image_url` | string | 否 | 食材图片的 OSS 访问地址，为空则纯文本对话 |
| `thread_id` | string | 是 | 会话 ID，用于隔离多轮记忆 |

响应：`text/event-stream` SSE 流，逐块返回 AI 生成的食谱报告文本。

### `GET /api/v1/chat/messages?thread_id=xxx` — 查询历史消息

```json
{ "messages": [ { "role": "user", "content": "..." }, { "role": "assistant", "content": "..." } ] }
```

### `DELETE /api/v1/chat/messages?thread_id=xxx` — 清空指定会话

```json
{ "success": true }
```

### `GET /api/v1/oss/presign?filename=food.jpg` — 获取图片上传签名

根据扩展名自动推断 `Content-Type`（支持 jpg/jpeg/png/gif/webp），返回有效期 1 小时的预签名 PUT 地址：

```json
{
  "uploadUrl": "https://...oss.../food.jpg?x-oss-signature=...",
  "contentType": "image/jpeg",
  "accessUrl": "https://your-bucket.oss-cn-beijing.aliyuncs.com/food.jpg"
}
```

前端拿到 `uploadUrl` 后直接 `PUT` 图片二进制上传，无需经过后端。

## 🤖 Agent 工作流程

系统提示词定义了私人厨师的四步工作流（见 `app/agents/personal_chief.py`）：

1. **识别和评估食材**——辨识照片中的全部食材，评估新鲜度与可用量，产出"当前可用食材清单"
2. **智能食谱检索**——优先调用 `TavilySearch` 以食材清单为关键词搜索可行菜谱
3. **多维度评估排序**——按营养价值与制作难度量化打分，简单且营养的排前
4. **结构化输出**——生成含食谱信息、得分、推荐理由、参考图片的建议报告

## ☁️ 阿里云 OSS 准备

1. 在 OSS 控制台创建 Bucket（示例区域：华北2-北京），获取 `OSS_BUCKET` / `OSS_REGION` / `OSS_ENDPOINT`
2. 创建 **RAM 子用户**并生成 AccessKey（避免使用主账号 Key），填入 `.env`
3. **给该子用户授权 `AliyunOSSFullAccess`**（或仅含目标 bucket `PutObject/GetObject` 的自定义策略）——否则上传报 `AccessDenied`
4. 若前端需要跨域直传，在 Bucket 的 **权限控制 → 跨域设置（CORS）** 中添加规则：AllowedOrigin 填前端来源（开发期可 `*`），AllowedMethod 勾选 `PUT`、`GET`，ExposeHeaders 加 `ETag`

## ⚠️ 常见问题（踩坑记录）

| 现象 | 原因 | 解决 |
|---|---|---|
| OSS 返回 `InvalidAccessKeyId` | `.env` 里的 Key 不存在，或 **Windows 用户环境变量残留旧 Key**（SDK 的 `EnvironmentVariableCredentialsProvider` 读环境变量，且 `load_dotenv()` 不覆盖已存在的环境变量） | 核对 RAM 控制台实际 Key；删除系统环境变量 `OSS_ACCESS_KEY_ID/SECRET` 后重启服务 |
| OSS 返回 `AccessDenied`（`ImplicitDeny`） | RAM 子用户未授予 OSS 权限 | 给子用户添加 `AliyunOSSFullAccess` |
| PyCharm 全部 import 爆红但包已安装 | 项目路径含**中文**时 PyCharm 解析 venv 的 `site-packages` 失败 | 设置系统环境变量 `PYTHONUTF8=1` 后重启 PyCharm，或把项目移到纯英文路径 |
| 端口 8001 被占用 | 旧的 uvicorn 进程（含 reload 子进程）未退出 | 任务管理器结束对应 python 进程，或 `Get-Process python \| Stop-Process -Force` |

## 📝 备注

- `db/personal_chief.db` 为运行时自动创建的 checkpoint 库，删除即重置所有会话记忆
- CORS 当前为 `allow_origins=["*"]`，生产环境请收敛为具体域名
- `.env` 含密钥，已在 `.gitignore` 中排除，切勿提交到版本库
