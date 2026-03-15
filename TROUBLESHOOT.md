# 飞书插件故障排查案例

本文档收录飞书插件使用过程中的常见问题和解决方案。

---

## 案例 1：用户身份权限检查未通过 - Token 需刷新

### 错误现象

运行 `/feishu doctor` 诊断命令后显示：

```
❌ 用户身份权限检查未通过
⚠️ 授权状态：共 1 个用户 | ✓ 有效：0, ⟳ 需刷新：1, ✗ 已过期：0
```

### 原因分析

1. 用户曾完成 OAuth 授权，token 已存储
2. access_token 即将过期（5 分钟内）或已过期
3. refresh_token 仍有效，可以刷新

### 相关代码

| 文件 | 说明 |
|------|------|
| `src/commands/doctor.js:checkUserPermissions()` (第 355-481 行) | 诊断逻辑 |
| `src/commands/doctor.js` (第 389 行) | 错误信息输出 |
| `src/core/token-store.js:tokenStatus()` (第 306-315 行) | Token 状态判断 |
| `src/core/token-store.js:getStoredToken()` (第 267-277 行) | Token 读取 |

### Token 状态说明

`tokenStatus()` 函数根据以下逻辑判断状态：

```javascript
const REFRESH_AHEAD_MS = 5 * 60 * 1000; // 5 分钟

export function tokenStatus(token) {
    const now = Date.now();
    if (now < token.expiresAt - REFRESH_AHEAD_MS) {
        return 'valid';         // ✓ 有效
    }
    if (now < token.refreshExpiresAt) {
        return 'needs_refresh'; // ⟳ 需刷新
    }
    return 'expired';           // ✗ 已过期
}
```

### 解决方案

#### 方案 1：重新授权（推荐）

在飞书中向机器人发送消息：

```
/feishu auth
```

这会触发 OAuth 重新授权流程，自动刷新 token。

#### 方案 2：检查应用权限配置

如果重新授权后仍然失败，检查飞书开放平台的应用配置：

1. 登录飞书开放平台：https://open.feishu.cn/app
2. 选择您的应用
3. 进入 **权限管理**
4. 确保开通了必要的用户身份权限（user token scopes）
5. 创建新版本并发布

#### 方案 3：删除旧 token 重新授权

如果 token 存储损坏，可以删除后重新授权：

**macOS**（使用 Keychain）：
```bash
security delete-generic-password -s "openclaw-feishu-uat" -a "{appId}:{open_id}"
```

**Linux**：
```bash
rm ~/.local/share/openclaw-feishu-uat/*.enc
```

**Windows**：
```powershell
Remove-Item "$env:LOCALAPPDATA\openclaw-feishu-uat\*.enc"
```

然后重新运行 `/feishu auth` 进行授权。

---

## 案例 2：应用身份权限检查未通过

### 错误现象

```
❌ 应用身份权限检查未通过
缺少 X 个必需权限。需应用管理员申请开通
```

### 原因分析

应用在飞书开放平台未开通必需的 API 权限。

### 解决方案

1. 访问诊断报告中的「申请」链接
2. 或手动前往飞书开放平台 → 应用 → 权限管理
3. 申请开通缺失的权限
4. 创建版本并提交审核
5. 审核通过后发布

---

## 案例 3：卡片按钮点击无反应

### 错误现象

点击消息卡片按钮后没有任何反应，或提示报错。

### 原因分析

应用未开通「消息卡片回传交互」权限。

### 解决方案

1. 登录飞书开放平台：https://open.feishu.cn/app
2. 选择您的应用 → **事件与回调**
3. 在回调配置中，修改订阅方式为"长链接"并添加回调：
   - **卡片回传交互** (`card.action.trigger`)
4. 创建应用版本 → 提交审核 → 发布

---

## 诊断命令参考

使用 `/feishu doctor` 命令进行全面诊断：

```bash
# 在飞书聊天中发送
/feishu doctor
```

诊断内容：
- ✅ 环境信息检查（插件版本、凭证完整性、API 连通性）
- ✅ 应用身份权限检查
- ✅ 用户身份权限检查
- ✅ 权限对照表
