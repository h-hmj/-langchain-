# 个人私厨（Personal Chef）

> 上传一张冰箱/食材照片，AI 自动识别食材、评估新鲜度，并结合网络菜谱检索，流式输出结构化的食谱推荐报告。

前后端一体的全栈项目：**FastAPI + LangChain/LangGraph 多模态 Agent 后端** + **Next.js 16 前端**，集成阿里云 OSS 图片直传、Tavily 联网搜索与 DashScope 多模态大模型。

## ✨ 核心特性

- 📷 **食材图片识别**：前端通过预签名 URL 直传图片到阿里云 OSS（不暴露密钥），多模态模型识别食材并评估新鲜度
- 🔍 **联网菜谱检索**：Agent 优先调用 Tavily 搜索真实菜谱，搜不到时才自行发挥
- 💬 **SSE 流式对话**：逐 token 输出推荐结果，前端 Markdown 实时渲染
- 🧠 **多会话记忆**：LangGraph `SqliteSaver` checkpoint 持久化，按 `thread_id` 隔离，支持历史查询与会话清空
- 🖼️ **一体化部署**：后端可挂载前端构建产物（`backend/static/`），单进程访问完整应用

## 🏗️ 架构总览

```
浏览器 (Next.js 前端 :3000)
   │
   │  ① GET  /api/v1/oss/presign   ──►  获取 OSS 预签名上传 URL
   │  ② PUT  uploadUrl             ──►  图片直传阿里云 OSS
   │  ③ POST /api/v1/chat/stream   ──►  FastAPI 后端 (:8001)
   │                                       │
   │                                       ▼
   │                             LangGraph create_agent
   │                             ├─ 模型: qwen3.5-plus (DashScope, OpenAI 兼容)
   │                             ├─ 工具: TavilySearch (联网搜菜谱)
   │                             └─ 记忆: SqliteSaver → backend/db/personal_chief.db
   │
   └── SSE 流式返回食谱推荐报告 ◄──┘
```

## 📁 仓库结构

```
-langchain-/
├── backend/                     # FastAPI + LangGraph 后端
│   ├── main.py                  #   入口：CORS、路由注册、静态资源挂载
│   ├── api/v1/
│   │   ├── chat.py              #   流式对话 / 历史消息 / 清空会话
│   │   └── oss.py               #   OSS 预签名上传接口
│   ├── app/agents/
│   │   └── personal_chief.py    #   核心 Agent：模型、工具、记忆、提示词
│   ├── models/schemas.py        #   请求模型
│   ├── common/logger.py         #   统一日志
│   ├── db/                      #   SQLite checkpoint（运行时自动创建）
│   ├── static/                  #   前端构建产物（一体化部署时由 Next.js 导出）
│   └── README.md                #   后端详细文档（API、OSS 配置、踩坑记录）
├── frontend/                    # Next.js 16 + React 19 前端
│   ├── app/                     #   页面与布局（App Router）
│   ├── components/              #   ChatInput / ChatMessage / RecipeCard
│   ├── lib/api.ts               #   后端 API 封装（presign、上传、SSE 流式聊天）
│   └── README.md                #   前端脚手架说明
└── README.md                    # 本文件
```

## 🚀 快速开始

### 环境要求

- Python 3.11+（开发环境 3.13）
- Node.js 18.18+（Next.js 16 要求）
- 阿里云 OSS Bucket、DashScope API Key、Tavily API Key

### 1. 启动后端

```powershell
cd backend

python -m venv .venv
.venv\Scripts\activate

pip install fastapi uvicorn langchain langchain-openai langchain-tavily `
            langgraph langgraph-checkpoint-sqlite alibabacloud-oss-v2 python-dotenv
```

在 `backend/` 目录下创建 `.env`（完整变量说明见 [后端 README](backend/README.md#2-配置环境变量)）：

```ini
DASHSCOPE_API_KEY=sk-xxxx
TAVILY_API_KEY=tvly-xxxx
OSS_ACCESS_KEY_ID=xxx
OSS_ACCESS_KEY_SECRET=xxx
OSS_BUCKET=xxx
OSS_REGION=cn-beijing
OSS_ENDPOINT=oss-cn-beijing.aliyuncs.com
```

启动：

```powershell
python main.py
# 后端运行于 http://127.0.0.1:8001，接口文档 /docs
```

### 2. 启动前端（开发模式）

```powershell
cd frontend

npm install
npm run dev
# 前端运行于 http://localhost:3000
```

前端 API 地址在 `frontend/lib/api.ts` 中配置（默认 `http://localhost:8001`）。

### 3. 一体化部署（可选）

将前端构建产物导出到后端静态目录，由 FastAPI 单进程托管：

```powershell
cd frontend
npm run build          # 产出 .next/，导出静态站点
# 将导出结果复制到 backend/static/ 后重启后端
```

访问 <http://127.0.0.1:8001/> 即可使用完整应用。

## 📡 API 简表

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/v1/chat/stream` | 流式菜谱推荐（SSE），参数：`message`、`image_url`、`thread_id` |
| GET | `/api/v1/chat/messages?thread_id=xxx` | 查询会话历史消息 |
| DELETE | `/api/v1/chat/messages?thread_id=xxx` | 清空指定会话 |
| GET | `/api/v1/oss/presign?filename=xxx` | 获取 OSS 预签名上传 URL（有效期 1 小时） |

详细请求/响应格式见 [backend/README.md](backend/README.md#-api-文档)。

## 🧰 技术栈

| 层 | 技术 |
|---|---|
| 前端 | Next.js 16 · React 19 · TypeScript 5 · Tailwind CSS 4 · react-markdown · lucide-react |
| 后端 | FastAPI · Uvicorn · LangChain 1.x · LangGraph · langchain-openai（DashScope）· langchain-tavily · alibabacloud-oss-v2 · Pydantic |
| 存储 | 阿里云 OSS（图片）· SQLite（Agent 会话记忆） |
| 模型 | qwen3.5-plus 多模态（可切换 DeepSeek 等备用模型） |

## ⚠️ 注意事项

- `.env` 含密钥，已在 `.gitignore` 中排除，切勿提交
- OSS 上传报 `AccessDenied` / `InvalidAccessKeyId` 等常见问题见 [后端 README 踩坑记录](backend/README.md#-常见问题踩坑记录)
- 开发时前端 CORS 依赖后端 `allow_origins=["*"]`，生产环境请收敛为具体域名
