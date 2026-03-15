# MCP Doc 工具架构说明

本文档说明飞书插件中 MCP（Model Context Protocol）Doc 工具的架构和调用链路。

## 概述

MCP Doc 工具通过 MCP 协议与飞书服务端通信，实现云文档的创建、获取和更新操作。

## 工具列表

| 工具名称 | MCP 工具名 | 功能 |
|---------|-----------|------|
| `feishu_create_doc` | `create-doc` | 从 Markdown 创建云文档 |
| `feishu_fetch_doc` | `fetch-doc` | 获取云文档内容 |
| `feishu_update_doc` | `update-doc` | 更新云文档内容 |

## 核心文件结构

```
src/tools/mcp/
├── doc/
│   ├── index.js          # 工具注册入口
│   ├── create.js         # create-doc 工具定义
│   ├── fetch.js          # fetch-doc 工具定义
│   └── update.js         # update-doc 工具定义
└── shared.js             # MCP 客户端和共享工具
```

## 调用链路

```
用户请求
    ↓
OpenClaw 工具系统
    ↓
registerMcpTool() [shared.js]
    ↓
invoke() 权限检查
    ↓
callMcpTool() [shared.js]
    ↓
HTTP POST → MCP 服务端
    ↓
JSON-RPC 2.0 响应
    ↓
结果解析和返回
```

## 核心组件

### 1. 工具注册入口

**文件**: `src/tools/mcp/doc/index.js`

```javascript
export function registerFeishuMcpDocTools(api) {
    // 1. 检查配置和账户状态
    // 2. 提取 MCP endpoint URL
    // 3. 注册工具
    registerCreateDocTool(api);
    registerFetchDocTool(api);
    registerUpdateDocTool(api);
}
```

### 2. 工具定义

**文件**: `src/tools/mcp/doc/create.js`

```javascript
const CreateDocSchema = Type.Object({
    markdown: Type.Optional(Type.String({ description: 'Markdown 内容' })),
    title: Type.Optional(Type.String({ description: '文档标题' })),
    folder_token: Type.Optional(Type.String({ description: '父文件夹 token' })),
    wiki_node: Type.Optional(Type.String({ description: '知识库节点 token' })),
    wiki_space: Type.Optional(Type.String({ description: '知识空间 ID' })),
    task_id: Type.Optional(Type.String({ description: '异步任务 ID' })),
});

export function registerCreateDocTool(api) {
    registerMcpTool(api, {
        name: 'feishu_create_doc',
        mcpToolName: 'create-doc',
        toolActionKey: 'feishu_create_doc.default',
        label: 'Feishu MCP: create-doc',
        description: '从 Markdown 创建云文档（支持异步 task_id 查询）',
        schema: CreateDocSchema,
    });
}
```

### 3. MCP 客户端

**文件**: `src/tools/mcp/shared.js`

#### 3.1 调用 MCP 工具

```javascript
export async function callMcpTool(name, args, toolCallId, uat, brand) {
    const endpoint = getMcpEndpoint(brand);
    const auth = buildAuthHeader();

    const body = {
        jsonrpc: '2.0',
        id: toolCallId,
        method: 'tools/call',
        params: {
            name,
            arguments: args,
        },
    };

    const headers = {
        'Content-Type': 'application/json',
        'X-Lark-MCP-UAT': uat,
        'X-Lark-MCP-Allowed-Tools': name,
        'User-Agent': getUserAgent(),
    };

    if (auth) headers.authorization = auth;

    const res = await fetch(endpoint, {
        method: 'POST',
        headers,
        body: JSON.stringify(body),
    });

    return unwrapJsonRpcResult(data.result);
}
```

#### 3.2 MCP 端点优先级

```javascript
function getMcpEndpoint(brand) {
    // 优先级：运行时覆盖 > 配置文件 > 环境变量 > 默认值
    return (mcpEndpointOverride ||
        readMcpUrlFromOpenclawJson() ||
        process.env.FEISHU_MCP_ENDPOINT?.trim() ||
        `${mcpDomain(brand)}/mcp`);
}
```

**端点优先级**:
1. **运行时覆盖** - `mcpEndpointOverride`（代码中设置）
2. **配置文件** - `.openclaw/openclaw.json` 中的 `channels.feishu.mcpEndpoint`
3. **环境变量** - `FEISHU_MCP_ENDPOINT`
4. **默认值** - `https://www.feishu.cn/mcp` 或 `https://www.larksuite.com/mcp`

#### 3.3 通用工具注册

```javascript
export function registerMcpTool(api, config) {
    registerTool(api, {
        name: config.name,
        async execute(toolCallId, params) {
            // 1. 执行参数验证
            config.validate?.(p);

            // 2. 通过 invoke 进行权限检查
            const result = await client.invoke(
                config.toolActionKey,
                async (_sdk, _opts, uat) => {
                    return callMcpTool(config.mcpToolName, p, toolCallId, uat, brand);
                },
                { as: 'user' }
            );

            // 3. 解析并返回结果
            return formatToolResult(result);
        },
    });
}
```

## JSON-RPC 请求格式

```json
{
    "jsonrpc": "2.0",
    "id": "tool-call-id",
    "method": "tools/call",
    "params": {
        "name": "create-doc",
        "arguments": {
            "title": "文档标题",
            "markdown": "# 文档内容"
        }
    }
}
```

## 请求头

| 头部 | 说明 |
|------|------|
| `Content-Type` | `application/json` |
| `X-Lark-MCP-UAT` | 用户访问令牌（OAuth token） |
| `X-Lark-MCP-Allowed-Tools` | 允许调用的工具名称 |
| `User-Agent` | 客户端标识 |
| `Authorization` | 可选，服务端鉴权（若配置） |

## 响应处理

MCP 服务端返回 JSON-RPC 2.0 响应：

```json
{
    "jsonrpc": "2.0",
    "id": "tool-call-id",
    "result": {
        "content": [
            {
                "type": "text",
                "text": "{\"doc_id\":\"doxcnXXX\",\"doc_url\":\"https://...\"}"
            }
        ]
    }
}
```

`unwrapJsonRpcResult()` 函数递归解包，确保返回纯结果对象。

## 权限模型

MCP Doc 工具使用 **用户身份（user_access_token）** 调用：

1. **应用权限** - 需要在飞书开放平台申请 `docx:docx:readonly`、`docx:docx:edit` 等权限
2. **用户授权** - 用户需要通过 OAuth 授权，授予应用访问其文档的权限

权限检查通过 `invoke()` 方法执行，若权限不足会自动触发授权流程。

## 配置示例

`.openclaw/openclaw.json`:

```json
{
    "channels": {
        "feishu": {
            "mcpEndpoint": "https://www.feishu.cn/mcp",
            "tools": {
                "doc": true
            }
        }
    }
}
```

## 相关文档

- [create-doc Skill](../skills/feishu-create-doc/SKILL.md) - 工具使用说明
- [update-doc Skill](../skills/feishu-update-doc/SKILL.md) - 更新文档使用说明
- [fetch-doc Skill](../skills/feishu-fetch-doc/SKILL.md) - 获取文档使用说明
