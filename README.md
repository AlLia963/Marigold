# Marigold

本地优先的对话 Agent：日常对话走本地模型（内置 llama.cpp 推理引擎），复杂任务按路由策略升级到云端模型（任意 OpenAI 兼容接口）。提供原生桌面客户端与轻量网页端，支持流式输出、思考过程展示、多会话管理。

> 个人项目，持续迭代中。

---

## 功能特性

- **本地推理**：内置 llama.cpp，支持任意 GGUF 模型（已实测 Qwen3 4B 量化模型）
- **双模型路由**：本地管日常、云端管难题，可自动/手动切换
- **桌面客户端**（PySide6）：会话管理、流式输出、思考折叠、token 统计
- **网页客户端**：单文件、零依赖、移动端优先、无登录墙
- **服务端模式**：WebSocket 协议 + 访问令牌鉴权，可部署到任何 Linux 主机

---

## 架构

```
客户端层（桌面 GUI / 网页端）
        │
        ▼
Agent 核心层（会话 / 路由 / 上下文 / 历史）
        │
        ▼
Provider 层（OpenAI 兼容：本地 / 云端）
        │
        ▼
本地引擎层（llama.cpp） / 云端 API
```

- 本地单进程直连核心，无额外服务
- 服务端模式通过 WebSocket 提供远程访问，与桌面端共用同一套装配逻辑

---

## 快速开始（本地）

```bash
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
copy .env.example .env
```

运行：

```bash
# 桌面客户端
.venv\Scripts\python -m marigold.clients.qt_app

# 服务端模式
.venv\Scripts\python main.py
.venv\Scripts\python -m marigold.clients.cli
```

测试：

```bash
.venv\Scripts\python -m unittest discover -s tests -v
```

---

## 配置

全部配置通过环境变量（`.env`）注入，常用项：

| 变量 | 说明 |
|---|---|
| `MARIGOLD_SYSTEM_PROMPT` | 模型角色/系统提示词 |
| `MARIGOLD_ROUTER_STRATEGY` | 路由策略：auto / local / remote |
| `MARIGOLD_LOCAL_ENGINE` | 本地引擎：auto / ollama / llamacpp |
| `MARIGOLD_LOCAL_MODEL` | 本地模型名 |
| `MARIGOLD_REMOTE_ENABLED` | 启用云端模型 |
| `MARIGOLD_REMOTE_BASE_URL` | 云端 OpenAI 兼容地址 |
| `MARIGOLD_REMOTE_MODEL` | 云端模型名 |
| `MARIGOLD_REMOTE_API_KEY` | 云端 API key |
| `MARIGOLD_AUTH_TOKEN` | 服务端访问令牌（留空不鉴权） |

云端接入示例（占位）：

```ini
MARIGOLD_ROUTER_STRATEGY=remote
MARIGOLD_REMOTE_ENABLED=true
MARIGOLD_REMOTE_BASE_URL=https://your-provider.example/v1
MARIGOLD_REMOTE_MODEL=your-model
MARIGOLD_REMOTE_API_KEY=your-key
```

---

## 路线图（节选）

- 会话持久化（SQLite）
- 工具调用执行器
- 多模态输入（模型端已支持视觉）
- 平台适配器（QQ 等）

---

## 说明

- 项目内 `.env`、服务端配置、备忘录等文件含私密信息，请勿提交到公开仓库或外传；
- 本项目为个人学习探索项目，无开源许可声明。
