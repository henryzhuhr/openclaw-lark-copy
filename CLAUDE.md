# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在此代码仓库中工作提供指导。

## 项目概述

OpenClaw 飞书/Lark 插件 - OpenClaw 官方飞书/Lark 渠道插件。使 AI 代理能够通过飞书/Lark API 读写消息、文档、多维表格、日历、任务等。

## 构建与开发

```bash
# 安装依赖
npm install

# 插件是 ESM 模块 - 无需构建，直接在 OpenClaw 中运行
```

## 架构

### 入口文件
- `index.js` - 主插件入口，注册渠道和所有工具
- `package.json` - 插件元信息（名称：`@larksuite/openclaw-lark`）

### 核心结构 (`src/`)

```
src/
├── channel/        # 渠道插件实现 (feishuPlugin)
├── core/           # 核心工具 (lark-client, config-schema, accounts, auth)
├── messaging/      # 消息收发处理
│   ├── inbound/    # 消息解析、@提及处理、策略
│   ├── outbound/   # 发送消息、卡片、媒体
│   └── converters/ # 消息格式转换
├── tools/
│   ├── oapi/       # 直接调用飞书开放 API 的工具
│   │   ├── im/     # 即时消息
│   │   ├── calendar/
│   │   ├── task/
│   │   ├── bitable/
│   │   ├── drive/
│   │   ├── wiki/
│   │   ├── sheets/
│   │   └── search/
│   ├── mcp/        # 基于 MCP 的工具（文档创建/获取/更新）
│   ├── oauth*.js   # OAuth 设备流程
│   └── auto-auth.js
├── card/           # 交互式卡片构建器
└── commands/       # CLI 和聊天命令 (/feishu_diagnose 等)
```

### 工具注册
- `src/tools/oapi/index.js` - 注册所有开放 API 工具（日历、任务、多维表格、IM、网盘、知识库、电子表格、搜索）
- `src/tools/mcp/doc/index.js` - 注册 MCP 文档工具
- `src/tools/oauth.js` - 注册 OAuth 工具

### 核心组件

**LarkClient** (`src/core/lark-client.js`): 中央 API 客户端，负责凭证管理、自动重试、令牌刷新。

**Config Schema** (`src/core/config-schema.js`): 基于 Zod 的插件配置校验（appId, appSecret, dmPolicy, groupPolicy 等）。

**feishuPlugin** (`src/channel/plugin.js`): ChannelPlugin 接口实现 - 定义能力、消息、目录、出站、网关。

## Skills

位于 `skills/` 目录，由 OpenClaw 自动加载：

| Skill | 用途 |
|-------|------|
| `feishu-channel-rules` | 输出风格规则（简短、对话式） |
| `feishu-im-read` | 消息历史、话题回复、搜索、资源下载 |
| `feishu-task` | 任务/清单 CRUD，使用 OAuth 用户身份 |
| `feishu-troubleshoot` | FAQ + `/feishu_doctor` 诊断命令 |
| `feishu-bitable/` | 多维表格操作 |
| `feishu-calendar/` | 日历管理 |
| `feishu-create-doc/` | 文档创建 |
| `feishu-fetch-doc/` | 文档获取 |
| `feishu-update-doc/` | 文档更新 |

## 飞书 Markdown

飞书消息卡片使用**非标准 Markdown**语法。完整参考见 `skills/feishu-channel-rules/references/markdown-syntax.md`：

- 不支持 H1-H3（`#` `##` `###` 不可用 - 使用 H4/H5 或加粗）
- 自定义标签：`<at>`, `<font color>`, `<text_tag>`, `<person>`, `<number_tag>`
- 图片语法：`![hover_text](image_key)`，其中 image_key 不是 URL

## 安全说明

- 使用 OAuth 2.0 设备流程获取用户级权限
- 双重权限模型：应用级 + 用户级 scopes
- `/feishu_doctor` 命令诊断权限问题
- 多账户安全警告见 `src/core/security-check.js`

## 故障排查

详见 [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)，收录了以下案例：

1. **Token 需刷新** - 授权状态显示"需刷新"时的解决方案
2. **应用权限缺失** - 应用身份权限检查未通过
3. **卡片按钮无反应** - 未开通卡片回传交互权限

## 开发文档

- [docs/mcp-doc-architecture.md](./docs/mcp-doc-architecture.md) - MCP Doc 工具架构说明
- [docs/uat-auth-flow.md](./docs/uat-auth-flow.md) - UAT 用户访问令牌获取流程
