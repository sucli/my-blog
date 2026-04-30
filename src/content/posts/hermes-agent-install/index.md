---
title: Hermes Agent 安装与配置完全指南
published: 2026-04-30
description: "从零开始安装 Hermes Agent，一个开源的 AI 终端代理框架，支持 20+ 模型提供商和 10+ 消息平台。"
tags: ["Hermes Agent", "AI", "终端工具", "CLI"]
category: 技术
draft: false
---

## 什么是 Hermes Agent

Hermes Agent 是 [Nous Research](https://nousresearch.com) 开发的开源 AI 代理框架。它运行在终端、聊天平台和 IDE 中，属于和 Claude Code（Anthropic）、Codex（OpenAI）、OpenClaw 同类的自主编码和任务执行代理。

和其他 AI Agent 工具相比，Hermes 的独特之处在于：

- **自我学习**：通过"技能"机制积累经验，解决复杂问题后可保存为可复用的技能文档
- **跨平台网关**：同一个 Agent 可以运行在 Telegram、Discord、Slack、WhatsApp、微信等 10+ 平台
- **多模型提供商**：支持 OpenRouter、Anthropic、OpenAI、DeepSeek 等 20+ 个提供商，随时切换
- **持久记忆**：跨会话记住你的偏好、环境信息和经验教训
- **可扩展**：支持插件、MCP 服务器、自定义工具、定时任务等

## 安装前准备

### 系统要求

Hermes Agent 支持以下平台：

- **Linux**（推荐 Ubuntu 20.04+、Debian 11+）
- **macOS**（Monterey 12.0+）
- **Windows WSL**（WSL2 推荐）

### 前置依赖

你需要安装以下工具：

```bash
# 1. Python 3.10 或更高版本
python3 --version  # 确认版本 >= 3.10

# 2. pip（Python 包管理器）
pip3 --version

# 3. Git（用于从源码安装）
git --version

# 4. Node.js（可选，用于 Gateway 和某些技能）
node --version  # 推荐 v18+
```

如果还没有安装 Python，可以使用系统包管理器：

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3 python3-pip python3-venv

# macOS（使用 Homebrew）
brew install python@3.12

# 验证安装
python3 --version  # 应显示 3.10+
```

## 安装 Hermes Agent

### 方式一：一键安装（推荐）

这是最简单的方式，适用于所有支持的平台：

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

这条命令会自动：

1. 克隆 Hermes Agent 源码到 `~/.hermes/hermes-agent/`
2. 创建 Python 虚拟环境
3. 安装所有依赖
4. 创建 `hermes` 命令的符号链接
5. 生成配置文件 `~/.hermes/config.yaml` 和 `.env`

安装完成后，终端会输出类似这样的信息：

```
✓ Hermes Agent installed successfully!
  Config:  ~/.hermes/config.yaml
  Skills:  ~/.hermes/skills/
  Sessions: ~/.hermes/sessions/

Run 'hermes setup' to configure your first provider.
```

### 方式二：从源码安装（开发者/高级用户）

如果你想参与开发或需要最新的未发布功能：

```bash
# 克隆仓库
git clone https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent

# 进入目录
cd ~/.hermes/hermes-agent

# 创建并激活虚拟环境
python3 -m venv .venv
source .venv/bin/activate  # Linux/macOS
# Windows WSL: source .venv/bin/activate

# 安装依赖
pip install -e ".[dev]"

# 创建命令符号链接
ln -sf "$(pwd)/hermes_cli/main.py" /usr/local/bin/hermes
chmod +x "$(pwd)/hermes_cli/main.py"

# 初始化配置
mkdir -p ~/.hermes
cp config.example.yaml ~/.hermes/config.yaml
```

### 方式三：Docker 安装（容器化部署）

如果你不想在主机上安装 Python 环境：

```bash
# 拉取镜像
docker pull ghcr.io/nousresearch/hermes-agent:latest

# 运行
docker run -it --rm \
  -v ~/.hermes:/root/.hermes \
  ghcr.io/nousresearch/hermes-agent:latest \
  hermes chat
```

## 初始配置

### 运行设置向导

安装完成后，运行交互式向导来配置基本选项：

```bash
hermes setup
```

向导会依次引导你设置：

1. **模型提供商** — 选择你想要使用的 AI 模型
2. **终端后端** — 本地执行、Docker 或 SSH
3. **网关配置** — 消息平台连接（可选）
4. **工具集** — 启用或禁用特定工具
5. **语音功能** — 文字转语音和语音转文字

### 配置模型和提供商

Hermes 支持 20+ 个提供商。最常用的是：

#### OpenRouter（推荐新手）

OpenRouter 提供一个统一的 API 接口，可以访问 Claude、GPT、Gemini 等多种模型：

```bash
# 1. 获取 API Key：https://openrouter.ai/keys
# 2. 设置环境变量
export OPENROUTER_API_KEY="sk-or-v1-your-key-here"

# 或者写入 .env 文件
echo 'OPENROUTER_API_KEY=sk-or-v1-your-key-here' >> ~/.hermes/.env

# 3. 选择模型
hermes model
```

#### Anthropic（Claude）

```bash
# 1. 获取 API Key：https://console.anthropic.com/
echo 'ANTHROPIC_API_KEY=sk-ant-your-key-here' >> ~/.hermes/.env

# 2. 选择模型
hermes model
# 然后选择 Anthropic → claude-sonnet-4 或 claude-opus-4
```

#### DeepSeek

```bash
echo 'DEEPSEEK_API_KEY=sk-your-key-here' >> ~/.hermes/.env
hermes model
```

#### Google Gemini

```bash
echo 'GOOGLE_API_KEY=your-key-here' >> ~/.hermes/.env
hermes model
```

#### Nous Portal（OAuth 登录）

```bash
hermes login --provider nous
# 会打开浏览器进行 OAuth 认证
```

### 配置文件详解

配置文件位于 `~/.hermes/config.yaml`，主要配置项：

```yaml
# 模型配置
model:
  default: "claude-sonnet-4"      # 默认模型
  provider: "anthropic"            # 默认提供商
  # api_key: "your-key"           # 也可以直接写在这里（推荐用 .env）
  # base_url: "https://..."       # 自定义 API 端点
  context_length: 200000           # 上下文长度

# Agent 配置
agent:
  max_turns: 90                    # 单次会话最大轮次
  tool_use_enforcement: true       # 强制使用工具

# 终端配置
terminal:
  backend: "local"                 # local / docker / ssh / modal
  cwd: "~"                         # 默认工作目录
  timeout: 180                     # 命令超时（秒）

# 压缩配置（控制上下文管理）
compression:
  enabled: true
  threshold: 0.50                  # 上下文使用率超过 50% 时触发
  target_ratio: 0.20              # 压缩到 20%

# 显示配置
display:
  skin: "default"                  # 主题皮肤
  tool_progress: true              # 显示工具进度
  show_reasoning: "minimal"        # 显示推理过程（none/minimal/low/medium/high）
  show_cost: true                  # 显示 Token 费用

# 内存配置
memory:
  memory_enabled: true             # 启用跨会话记忆
  user_profile_enabled: true       # 启用用户画像

# 安全配置
security:
  tirith_enabled: true             # 命令安全检查
```

### 密钥管理

API Key 存放在 `~/.hermes/.env` 文件中（不要提交到 Git）：

```bash
# 编辑密钥文件
nano ~/.hermes/.env

# 添加你的 API Key（每行一个）
OPENROUTER_API_KEY=sk-or-v1-xxx
ANTHROPIC_API_KEY=sk-ant-xxx
DEEPSEEK_API_KEY=sk-xxx
```

## 启动和使用

### 基本使用

```bash
# 启动交互式聊天
hermes

# 单次查询（非交互）
hermes chat -q "什么是机器学习？"

# 指定模型
hermes chat -m "anthropic/claude-sonnet-4" -q "解释量子计算"

# 查看当前配置
hermes config

# 检查系统状态
hermes doctor
```

### 常用命令

安装完成后，这些是你最常用的命令：

```bash
# 运行健康检查（首次配置后必做）
hermes doctor

# 切换模型
hermes model

# 查看/编辑配置
hermes config
hermes config edit

# 管理工具
hermes tools                    # 交互式工具管理
hermes tools list               # 查看所有工具状态
hermes tools enable browser     # 启用浏览器工具

# 管理技能
hermes skills list              # 查看已安装技能
hermes skills search "code"     # 搜索技能
hermes skills install code-review  # 安装技能

# 会话管理
hermes sessions list            # 查看最近会话
hermes sessions browse          # 交互式浏览

# 查看版本
hermes --version
```

### 交互式会话中的快捷命令

在交互式聊天中，你可以输入 `/` 开头的快捷命令：

```
/new          重置会话，开始新对话
/model        切换模型
/retry        重试上一条消息
/undo         撤销上一轮对话
/title        给会话命名
/compress     手动压缩上下文
/help         显示帮助
/quit         退出
```

## 高级功能

### 定时任务（Cron）

```bash
# 创建定时任务
hermes cron create "0 9 * * *"    # 每天早上 9 点

# 查看任务列表
hermes cron list

# 暂停/恢复任务
hermes cron pause <job_id>
hermes cron resume <job_id>
```

### 网关（消息平台）

将 Hermes 连接到 Telegram、Discord 等聊天平台：

```bash
# 配置网关
hermes gateway setup

# 前台运行（调试用）
hermes gateway run

# 安装为后台服务
hermes gateway install
hermes gateway start

# 查看状态
hermes gateway status
```

支持的平台：

- Telegram
- Discord
- Slack
- WhatsApp
- Signal
- Matrix
- 邮件
- 短信
- 钉钉 / 飞书 / 企业微信
- Home Assistant
- API Server（供 Open WebUI 等调用）

### MCP 服务器

```bash
# 添加 MCP 服务器
hermes mcp add my-server --command "npx my-mcp-server"

# 查看已配置的服务器
hermes mcp list

# 测试连接
hermes mcp test my-server
```

### 多实例（Profiles）

创建独立的配置实例，每个实例有自己的配置、技能和记忆：

```bash
# 创建新 profile
hermes profile create work

# 使用特定 profile
hermes profile use work
hermes -p work

# 列出所有 profiles
hermes profile list
```

### 凭证池（Credential Pools）

管理多个 API Key，自动轮换：

```bash
# 交互式添加凭证
hermes auth add

# 查看凭证池
hermes auth list openrouter
```

## 常见问题排查

### 安装失败

```bash
# 检查 Python 版本（需要 3.10+）
python3 --version

# 检查 pip 是否可用
pip3 --version

# 手动安装依赖
pip3 install -r ~/.hermes/hermes-agent/requirements.txt

# 重新运行安装脚本
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

### 模型连接问题

```bash
# 运行诊断
hermes doctor

# 检查 API Key 是否设置
cat ~/.hermes/.env

# 测试连接
hermes chat -q "Hello, are you working?"
```

### 权限问题

```bash
# 确保 hermes 命令有执行权限
chmod +x ~/.hermes/hermes-agent/hermes_cli/main.py

# 如果使用符号链接，确保链接正确
ls -la $(which hermes)
```

### 更新到最新版本

```bash
# 一键更新
hermes update

# 或手动更新
cd ~/.hermes/hermes-agent
git pull
pip install -e .
```

## 目录结构

安装完成后，Hermes 会在你的系统中创建以下目录：

```
~/.hermes/
├── config.yaml           # 主配置文件
├── .env                  # API Key 和密钥
├── auth.json             # OAuth 令牌和凭证池
├── hermes-agent/         # 源码（源码安装时）
├── skills/               # 已安装的技能
├── sessions/             # 会话记录
├── logs/                 # 网关和错误日志
└── profiles/             # 多 profile 实例
    └── <name>/           # 每个 profile 的独立目录
```

## 写在最后

Hermes Agent 是一个功能强大且高度可定制的 AI 代理框架。无论你是想用它来辅助编程、管理服务器、进行研究，还是搭建一个多平台的 AI 助手，它都能胜任。

安装过程本身只需要一行命令，真正的功夫在于后续的配置和定制。建议从简单的终端交互开始，熟悉基本用法后，再逐步探索网关、技能、定时任务等高级功能。

相关资源：

- 官方文档：https://hermes-agent.nousresearch.com/docs/
- GitHub 仓库：https://github.com/NousResearch/hermes-agent
- 技能市场：`hermes skills browse`
