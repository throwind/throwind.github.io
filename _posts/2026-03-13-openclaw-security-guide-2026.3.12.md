---
layout: post
title:  "OpenClaw 安全配置指南 v2026.3.12"
date:   2026-03-13 16:40:00 +0800
categories: openclaw security ai
tags: [openclaw, 安全，AI, 配置指南]
author: haoqi
excerpt: "基于 OpenClaw v2026.3.12 源码的完整安全配置指南，涵盖 Gateway 认证、访问控制、工具权限、沙箱隔离、凭证管理等核心安全主题。"
---

> **文档版本**: 2026-03-13  
> **源码版本**: OpenClaw v2026.3.12 (main@402f2556b)  
> **信任模型**: 个人助理（单信任边界）

---

## 📋 核心安全原则

### 信任模型声明

**OpenClaw 采用"个人助理"信任模型，不是多租户隔离系统：**

- ✅ **支持**: 一个受信任操作员边界，可运行多个 Agent
- ❌ **不支持**: 多个敌对用户共享同一 Gateway/配置
- ⚠️ **如需多用户隔离**: 使用独立 Gateway + 独立 OS 用户/主机

### 安全分层策略

```
身份认证 → 访问范围 → 工具权限 → 沙箱隔离 → 模型选择
   ↓          ↓          ↓          ↓          ↓
谁能对话    在哪行动    能做什么   限制影响   抗注入能力
```

---

## 🔐 一、Gateway 认证配置

### 1.1 认证模式选择

| 模式 | 适用场景 | 安全等级 |
|------|----------|----------|
| `token` | 推荐默认 | ⭐⭐⭐⭐⭐ |
| `password` | 环境变量管理 | ⭐⭐⭐⭐ |
| `trusted-proxy` | 企业反向代理 | ⭐⭐⭐⭐ |
| `none` | ❌ 禁止使用 | ⚠️ 危险 |

### 1.2 最小安全配置

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",  // 仅本地访问
    port: 18789,
    auth: {
      mode: "token",
      token: "${OPENCLAW_GATEWAY_TOKEN}"  // 使用环境变量
    },
    // 可选：速率限制
    rateLimit: {
      maxAttempts: 10,
      windowMs: 60000,    // 1 分钟窗口
      lockoutMs: 300000   // 5 分钟锁定
    }
  }
}
```

### 1.3 网络暴露风险矩阵

| 配置 | 风险等级 | 建议 |
|------|----------|------|
| `bind: "loopback"` + 有 auth | ✅ 安全 | 推荐默认 |
| `bind: "lan"` + 有 auth | ⚠️ 中风险 | 需防火墙限制 |
| `bind: "lan"` + 无 auth | ❌ 严重 | 立即修复 |
| `tailscale.mode: "funnel"` | ❌ 公开暴露 | 避免使用 |
| `tailscale.mode: "serve"` | ✅ 尾网内安全 | 推荐远程访问 |

### 1.4 Tailscale Serve 配置（推荐远程访问）

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: {
      mode: "serve",  // 仅尾网可访问
      allowTailscale: true  // 允许 Tailscale 身份头认证
    }
  }
}
```

---

## 🚪 二、DM 和群组访问控制

### 2.1 DM 策略（所有频道）

```json5
{
  channels: {
    whatsapp: { dmPolicy: "pairing" },  // 默认，需配对码
    telegram: { dmPolicy: "pairing" },
    discord: { dmPolicy: "pairing" },
    slack: { dmPolicy: "pairing" }
  }
}
```

**DM 策略选项：**

| 策略 | 行为 | 安全等级 |
|------|------|----------|
| `pairing` | 未知发送者收到配对码，批准后才能对话 | ⭐⭐⭐⭐⭐ |
| `allowlist` | 仅允许名单内用户 | ⭐⭐⭐⭐⭐ |
| `open` | 任何人可 DM（需显式 `"*"`） | ⚠️ 危险 |
| `disabled` | 完全忽略 DM | ⭐⭐⭐⭐⭐ |

### 2.2 群组策略

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true }  // 所有群组需@提及
      }
    }
  }
}
```

### 2.3 DM 会话隔离（多用户场景）

```json5
{
  session: {
    dmScope: "per-channel-peer"  // 每个频道 + 发送者独立会话
  }
}
```

---

## 🛠️ 三、工具权限管理

### 3.1 危险工具清单

**默认应拒绝的高风险工具：**

```json5
{
  tools: {
    deny: [
      "sessions_spawn",  // 生成新 Agent 会话（RCE 风险）
      "sessions_send",   // 跨会话消息注入
      "cron",            // 持久化自动化控制
      "gateway",         // Gateway 配置修改
      "whatsapp_login"   // 交互式登录
    ]
  }
}
```

### 3.2 推荐工具配置文件

```json5
{
  tools: {
    profile: "messaging",  // 仅消息工具
    fs: { workspaceOnly: true },  // 文件系统限制在工作区
    exec: {
      security: "deny",   // 默认拒绝执行
      ask: "always",      // 总是询问
      applyPatch: {
        workspaceOnly: true  // 补丁仅限工作区
      }
    },
    elevated: {
      enabled: false  // 禁用提升权限
    }
  }
}
```

### 3.3 Gateway HTTP API 工具限制

Gateway 的 `/tools/invoke` HTTP 端点默认拒绝以下工具：

```typescript
const DEFAULT_GATEWAY_HTTP_TOOL_DENY = [
  "sessions_spawn",
  "sessions_send",
  "cron",
  "gateway",
  "whatsapp_login"
];
```

---

## 📦 四、沙箱隔离

### 4.1 沙箱模式配置

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",           // 所有会话启用沙箱
        scope: "agent",        // 每 Agent 独立容器
        workspaceAccess: "none" // 不挂载工作区（最安全）
      }
    }
  }
}
```

### 4.2 沙箱 Docker 安全配置

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          network: "bridge"  // 禁止使用 "host"
        }
      }
    }
  }
}
```

### 4.3 禁止的 Bind Mounts

以下主机路径禁止挂载到沙箱：

```typescript
const BLOCKED_HOST_PATHS = [
  "/etc", "/proc", "/sys", "/dev",
  "/root", "/boot",
  "/run", "/var/run",
  "/var/run/docker.sock"
];
```

### 4.4 常见错误配置

```json5
// ❌ 错误：沙箱配置了但模式是 off
{
  agents: { defaults: { sandbox: { mode: "off" } } },
  tools: { exec: { host: "sandbox" } }  
  // 结果：exec 直接在主机运行！
}

// ✅ 正确
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  tools: { exec: { host: "sandbox" } }
}
```

---

## 🔑 五、凭证和密钥管理

### 5.1 敏感文件位置

```
~/.openclaw/
├── openclaw.json                    # 配置（600 权限）
├── credentials/
│   ├── whatsapp/<accountId>/creds.json
│   ├── <channel>-allowFrom.json    # 配对允许列表
│   └── oauth.json                   # 遗留 OAuth
├── agents/<agentId>/
│   └── agent/auth-profiles.json    # API 密钥配置
├── secrets.json                     # 文件密钥载荷（可选）
└── agents/<agentId>/sessions/      # 会话记录（*.jsonl）
```

### 5.2 文件权限要求

```bash
# Linux/macOS
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
chmod 600 ~/.openclaw/credentials/**/*.json

# Windows (PowerShell)
icacls "$env:USERPROFILE\.openclaw" /inheritance:r /grant:r "$($env:USERNAME):(F)"
```

### 5.3 SecretRef 使用（推荐）

```json5
{
  secrets: {
    providers: {
      env: { type: "env" },
      file: { type: "file", path: "~/.openclaw/secrets.json" }
    }
  },
  gateway: {
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}"
    }
  }
}
```

---

## 🌐 六、网络安全加固

### 6.1 mDNS 发现（信息泄露风险）

```json5
{
  discovery: {
    mdns: {
      mode: "minimal"  // 默认，不广播 cliPath/sshPort
    }
  }
}
```

### 6.2 浏览器 SSRF 策略

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,  // 严格模式
      hostnameAllowlist: ["*.example.com"]
    }
  }
}
```

---

## 🤖 七、多 Agent 权限隔离

### 7.1 Agent 权限分级

```json5
{
  agents: {
    list: [
      // 个人 Agent - 完全访问
      {
        id: "personal",
        sandbox: { mode: "off" },
        tools: { profile: "full" }
      },
      
      // 家庭/工作 Agent - 沙箱 + 只读
      {
        id: "family",
        sandbox: {
          mode: "all",
          workspaceAccess: "ro"
        },
        tools: {
          allow: ["read"],
          deny: ["write", "edit", "exec", "browser"]
        }
      },
      
      // 公开 Agent - 仅消息
      {
        id: "public",
        sandbox: {
          mode: "all",
          workspaceAccess: "none"
        },
        tools: {
          allow: ["whatsapp", "telegram"],
          deny: ["read", "write", "exec", "browser", "cron", "gateway"]
        }
      }
    ]
  }
}
```

---

## 📊 八、安全审计

### 8.1 审计命令

```bash
# 基础审计
openclaw security audit

# 深度审计（含 Gateway 探测）
openclaw security audit --deep

# 自动修复（安全操作）
openclaw security audit --fix

# JSON 输出（CI/CD 集成）
openclaw security audit --json | jq '.findings[] | select(.severity=="critical")'
```

### 8.2 关键检查项 (CheckIDs)

| CheckID | 严重性 | 说明 |
|---------|--------|------|
| `fs.state_dir.perms_world_writable` | critical | 状态目录全局可写 |
| `gateway.bind_no_auth` | critical | 非 loopback 绑定无认证 |
| `gateway.tailscale_funnel` | critical | Tailscale Funnel 公开暴露 |
| `gateway.control_ui.device_auth_disabled` | critical | 禁用设备身份检查 |
| `security.exposure.open_groups_with_elevated` | critical | 开放群组 + 提升工具 |

### 8.3 危险配置标志清单

```json5
// Gateway Control UI
gateway.controlUi.allowInsecureAuth: true
gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true
gateway.controlUi.dangerouslyDisableDeviceAuth: true

// Hooks
hooks.gmail.allowUnsafeExternalContent: true

// 工具
tools.exec.applyPatch.workspaceOnly: false
```

---

## 🚨 九、事件响应

### 9.1 应急步骤

**1. 遏制（Contain）**
- 停止 Gateway 进程
- 设置 `bind: "loopback"`
- 设置 `dmPolicy: "disabled"`

**2. 轮换（Rotate）**
- 生成新 token：`openclaw doctor --generate-gateway-token`
- 轮换所有 API 密钥和凭证
- 更新配置并重启

**3. 审计（Audit）**
- 检查 Gateway 日志：`/tmp/openclaw/openclaw-*.log`
- 查看会话记录：`~/.openclaw/agents/*/sessions/*.jsonl`
- 运行深度审计：`openclaw security audit --deep`

---

## 📝 十、安全检查清单

### 每日检查
- [ ] 运行 `openclaw security audit`
- [ ] 检查 Gateway 日志异常
- [ ] 验证配对请求（如有）

### 每周检查
- [ ] 审查新增会话记录
- [ ] 检查凭证文件权限
- [ ] 更新依赖和插件

### 配置变更后
- [ ] 运行 `openclaw security audit --deep`
- [ ] 验证所有 critical 发现已修复
- [ ] 测试关键功能正常

---

## 🔮 十一、未来安全功能计划

### 11.1 已实现（v2026.3.12）

- ✅ 增强的 Gateway 认证速率限制
- ✅ 沙箱 Bind Mount 验证（阻止危险路径）
- ✅ 环境变量过滤（阻止危险变量）
- ✅ 插件代码安全检查
- ✅ mDNS 信息泄露防护（minimal 模式）
- ✅ HTTP API 工具默认拒绝列表
- ✅ Sub-Agent 委派深度限制

### 11.2 建议功能

- 📋 **细粒度工具授权**: 基于用户/角色的工具级权限
- 📋 **会话级加密**: 敏感会话记录加密存储
- 📋 **审计日志导出**: SIEM 系统集成支持
- 📋 **动态策略更新**: 无需重启的策略热更新
- 📋 **多因素认证**: Gateway 管理操作 MFA

---

## 📚 附录：安全配置模板

完整的安全基线配置：

```json5
{
  gateway: {
    mode: "local",
    bind: "loopback",
    port: 18789,
    auth: {
      mode: "token",
      token: "${OPENCLAW_GATEWAY_TOKEN}",
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000
      }
    },
    trustedProxies: ["127.0.0.1"],
    allowRealIpFallback: false
  },
  discovery: { mdns: { mode: "minimal" } },
  session: { dmScope: "per-channel-peer" },
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } }
    }
  },
  tools: {
    profile: "messaging",
    deny: ["gateway", "cron", "sessions_spawn", "sessions_send"],
    fs: { workspaceOnly: true },
    exec: {
      security: "deny",
      ask: "always"
    },
    elevated: { enabled: false }
  },
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
        workspaceAccess: "none"
      }
    }
  },
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false
    }
  },
  logging: { redactSensitive: "tools" }
}
```

---

## 参考文档

- [OpenClaw 安全文档](https://docs.openclaw.ai/gateway/security)
- [OpenClaw 沙箱文档](https://docs.openclaw.ai/gateway/sandboxing)
- [OpenClaw 配置参考](https://docs.openclaw.ai/gateway/configuration-reference)
- [威胁模型](https://trust.openclaw.ai)

---

> **最后更新**: 2026-03-13  
> **下次审查**: 2026-04-13 或源码重大更新后
