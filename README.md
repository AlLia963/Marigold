# Marigold

本地优先的对话 Agent：日常对话用本地模型（内置 llama.cpp 推理引擎），复杂任务按路由策略升级到云端模型（任意 OpenAI 兼容接口）。提供原生桌面客户端与轻量网页端，支持流式输出、思考过程展示、多会话管理、双模型路由。

> 个人学习探索项目，持续迭代中。

---

## 背景

想法源于对机器人框架的观察：平台的负担主要来自"平台绑定 + 功能堆叠"，而不是模型本身。因此核心只做四件事——**会话、上下文、路由、模型调用**，入口（终端/桌面/网页/未来的 QQ）做成可插拔的适配器。

架构上模型与引擎都可替换：换模型、换推理框架，不动对话核心。这也是后续探索自研小模型的地基。

---

## 功能特性

- **本地推理**：内置 llama.cpp，支持任意 GGUF 量化模型（已实测 Qwen3 4B Q4_K_M）
- **双模型路由**：本地管日常、云端管难题，可自动（关键词触发）或手动（界面开关）切换
- **思考过程展示**：流式解析 reasoning，默认折叠、点击展开
- **上下文管理**：token 预算估算与自动裁剪，支持"思考是否计入上下文"实时开关
- **桌面客户端**（PySide6）：会话侧边栏、流式输出、思考折叠、设置面板、token 统计
- **网页客户端**：单文件、零依赖、移动端优先、无登录墙（访问令牌走 URL）
- **服务端模式**：WebSocket JSON 协议 + 访问令牌鉴权，可部署到任意 Linux 主机
- **会话管理**：新建/切换/重命名/删除，首条消息自动命名

---

## 架构

```
┌───────────────────────────────┐
│ 客户端层                        │
│  桌面 GUI (PySide6)             │
│  网页客户端 (web/index.html)    │
└───────────────┬───────────────┘
                │ 本地直连 / WebSocket
┌───────────────▼───────────────┐
│ Agent 核心层                   │
│  Agent      会话管理/路由/上下文 │
│  Router     本地 ↔ 云端 决策     │
│  Conversation token 预算/裁剪   │
│  SessionStore 会话存储          │
└───────┬───────────────┬───────┘
        ▼               ▼
┌───────────────┐  ┌───────────────┐
│ Provider 层    │  │ 本地引擎层      │
│ 本地/云端      │  │ llama.cpp      │
│ OpenAI 兼容    │  │ (子进程管理)    │
└───────────────┘  └───────────────┘
```

各层职责：

| 层 | 职责 |
|---|---|
| 客户端层 | 展示、交互、设置（桌面 + 网页） |
| Agent 核心层 | 会话、路由、上下文、历史 |
| Provider 层 | 统一 OpenAI 兼容接口（本地/云端） |
| 本地引擎层 | llama.cpp 子进程生命周期（拉起/健康检查/停止） |

---

## 快速开始（本地）

### 环境要求

- Windows 10/11 x64（开发机为 RTX 4060 Laptop 8GB）
- Python 3.13
- 一份 GGUF 模型（如 qwen3:4b Q4_K_M）

### 安装

```bash
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
copy .env.example .env
```

将 llama.cpp 运行库（`llama-server.exe` + CUDA DLL）放入 `llama/` 目录，`.env` 指向模型文件。

### 运行

```bash
# 桌面客户端
.venv\Scripts\python -m marigold.clients.qt_app

# 服务端模式（WebSocket）
.venv\Scripts\python main.py
.venv\Scripts\python -m marigold.clients.cli

# 本地网页调试（内置假后端，不加载模型）
.venv\Scripts\python scripts\dev_fake_server.py
# 浏览器打开 http://127.0.0.1:8899/?ws=ws://127.0.0.1:8765/ws
```

### 测试

```bash
.venv\Scripts\python -m unittest discover -s tests -v
```

---

## 部署到服务器（可选）

服务端模式可部署到任意 Linux 主机（建议 2C2G+），配合 nginx 反向代理提供网页与 WebSocket：

```nginx
server {
    listen 80;
    server_name _;
    root /opt/marigold/web;
    index index.html;

    location /ws {
        proxy_pass http://127.0.0.1:8765;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

systemd 服务示例：

```ini
[Unit]
Description=Marigold server
After=network.target

[Service]
WorkingDirectory=/opt/marigold
EnvironmentFile=/opt/marigold/.env
ExecStart=/opt/marigold/.venv/bin/python -m marigold.server_main
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

网页访问形式：`http://<服务器IP>/?token=<访问令牌>`（token 自动从 URL 读取，无登录墙）。

> 服务器只跑 Agent 核心与网页，本地模型留在本机 GPU 上；服务器上的推理走云端 API（OpenAI 兼容），2C2G 足够。

---

## 配置说明

全部配置通过环境变量（`.env`）注入。常用项：

| 变量 | 说明 |
|---|---|
| `MARIGOLD_SYSTEM_PROMPT` | 模型角色/系统提示词 |
| `MARIGOLD_TEMPERATURE` | 采样温度（默认 0.7） |
| `MARIGOLD_MAX_CONTEXT_TOKENS` | 上下文 token 预算（默认 8192） |
| `MARIGOLD_ROUTER_STRATEGY` | 路由策略：auto / local / remote |
| `MARIGOLD_ESCALATE_KEYWORDS` | 触发云端的升级关键词 |
| `MARIGOLD_LOCAL_ENGINE` | 本地引擎：auto / ollama / llamacpp |
| `MARIGOLD_LOCAL_MODEL` | 本地模型名 |
| `MARIGOLD_REMOTE_ENABLED` | 启用云端模型 |
| `MARIGOLD_REMOTE_BASE_URL` | 云端 OpenAI 兼容地址 |
| `MARIGOLD_REMOTE_MODEL` | 云端模型名 |
| `MARIGOLD_REMOTE_API_KEY` | 云端 API key |
| `MARIGOLD_AUTH_TOKEN` | 服务端访问令牌（留空不鉴权） |
| `MARIGOLD_INCLUDE_REASONING_IN_CONTEXT` | 思考内容是否计入上下文（网页端可实时切换） |

云端接入示例（任意 OpenAI 兼容服务，占位）：

```ini
MARIGOLD_ROUTER_STRATEGY=remote
MARIGOLD_REMOTE_ENABLED=true
MARIGOLD_REMOTE_BASE_URL=https://your-provider.example/v1
MARIGOLD_REMOTE_MODEL=your-model
MARIGOLD_REMOTE_API_KEY=your-key
```

---

## 目录结构

```
Marigold/
├── main.py                  # 服务端模式入口
├── requirements.txt         # 本地依赖（httpx / websockets / PySide6）
├── requirements-server.txt  # 服务器端依赖（无 PySide6）
├── .env.example             # 配置模板
├── marigold/
│   ├── config.py            # 配置（环境变量驱动）
│   ├── llm.py               # OpenAI 兼容客户端（httpx，流式）
│   ├── providers.py         # Provider 抽象 + 本地引擎决策
│   ├── router.py            # 本地↔云端路由
│   ├── context.py           # token 估算与裁剪
│   ├── sessions.py          # 会话存储
│   ├── agent.py             # Agent 编排
│   ├── server.py            # WebSocket 服务端（鉴权）
│   ├── engines/llamacpp.py  # 内置 llama.cpp 引擎
│   └── clients/             # CLI + 桌面客户端
├── web/index.html           # 网页客户端（单文件）
├── tests/                   # 单元测试
└── scripts/                 # 安装与开发辅助脚本
```

---

## 技术栈

- Python 3.13、httpx、websockets
- PySide6（桌面 GUI）
- llama.cpp（本地推理引擎）
- nginx（可选，网页/WS 反代）

---

## 路线图

- 会话持久化（SQLite）
- 工具调用执行器
- 多模态输入（模型端已支持视觉）
- 平台适配器（QQ 等）
- 自检/反思机制

---

## 安全说明

- `.env` 与服务端配置含密钥，已 gitignore，**不要提交到公开仓库**
- 服务端建议开启 `MARIGOLD_AUTH_TOKEN`，防止公网滥用
- 云端模型若为免费档，注意数据使用条款，勿传输敏感内容

---

## 声明

本项目为个人学习探索项目，无开源许可声明。
