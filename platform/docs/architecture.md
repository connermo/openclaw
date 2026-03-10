# OpenClaw Management Platform - 架构设计

## Context

OpenClaw 当前是一个**单用户、单租户**系统。每个 Gateway 实例独立运行，没有组织/用户层级概念。随着 OpenClaw 定位为**个人助手和数字员工**，需要一个集中式管理平台来支撑企业级部署。

**核心模型：1 用户 = 1 Gateway = 1 Main Agent**。每个用户拥有一个独立的 Gateway 实例，Gateway 内运行一个 Main Agent 作为该用户的个人助手/数字员工。管理平台统一管理组织内所有用户的 Gateway 实例。

---

## 1. 系统架构

### 核心决策：独立的管理平面（Management Plane）

管理平台作为**独立服务**，通过现有的 Gateway WebSocket/HTTP API 连接和管理各个 Gateway 实例。理由：
- Gateway 已有完善的 RPC 接口（100+ 方法），可直接复用
- 现有 auth/scope 体系支持 operator 角色远程管理
- Gateway 无需改造即可纳管（Phase 0 兼容）
- 管理平面故障不影响各 Gateway 独立运行

### 组件架构

```
┌──────────────────────────────────────────────────────────┐
│                   OpenClaw 管理平面                        │
│                                                          │
│  ┌──────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │ Management   │  │ Gateway        │  │ Management   │  │
│  │ API (REST+WS)│  │ Controller     │  │ UI (React)   │  │
│  └──────┬───────┘  └───────┬────────┘  └──────┬───────┘  │
│         │                  │                   │          │
│  ┌──────┴───────┐  ┌───────┴────────┐  ┌──────┴───────┐  │
│  │ Auth & IAM   │  │ Config Engine  │  │ Analytics &  │  │
│  │ (OIDC/SAML)  │  │ (模板/策略)     │  │ Audit        │  │
│  └──────┬───────┘  └───────┬────────┘  └──────┬───────┘  │
│         └──────────────────┼──────────────────┘          │
│                    ┌───────┴───────┐                      │
│                    │  PostgreSQL   │                      │
│                    └───────────────┘                      │
└────────────────────────┬─────────────────────────────────┘
                         │ WS/HTTP (operator token)
            ┌────────────┼────────────┐
      ┌─────┴─────┐ ┌────┴────┐ ┌────┴─────┐
      │ 用户 A 的  │ │用户 B 的│ │用户 C 的 │
      │ Gateway   │ │Gateway  │ │Gateway   │
      └───────────┘ └─────────┘ └──────────┘
```

### 各组件职责

| 组件 | 职责 |
|------|------|
| **Management API** | REST + WebSocket API，处理组织/用户/Gateway CRUD |
| **Gateway Controller** | 维护到各用户 Gateway 的持久 WebSocket 连接，聚合状态，转发命令 |
| **Management UI** | React SPA，用户入口 + 管理员入口 |
| **Auth & IAM** | OAuth2/OIDC SSO、API Key、角色权限解析 |
| **Config Engine** | 配置模板管理、策略执行、批量配置下发 |
| **Analytics & Audit** | 用量聚合、成本追踪、审计日志（含 Gateway 转发事件） |

---

## 2. 数据模型

### 核心实体关系

```
Organization 1──N User 1──1 Gateway
     │               │
     │ 1──N OrgMembership (role: owner/admin/member)
     │
     │ 1──N ConfigTemplate
     │ 1──N ConfigPolicy
     │ 1──N SharedSkill
     │ 1──N AuditLog
     │ 1──N UsageSnapshot (via Gateway)
     │ 1──N ApiKey
```

**核心简化：User ↔ Gateway 是一对一关系**，无需 Team/Agent 中间层。

### 关键表设计

**组织与用户**
- `organizations` - 组织（name, slug, plan, settings）
- `users` - 用户（email, display_name, auth_provider, auth_provider_id, department）
- `org_memberships` - 组织成员（user_id, org_id, role: owner/admin/member）

**Gateway（每用户一个）**
- `gateways` - 用户的 Gateway 实例，与 user 一对一
  - `user_id` - 所属用户（UNIQUE）
  - `ws_url` - WebSocket 连接地址
  - `auth_credential_encrypted` - operator token（加密）
  - `status` - pending/online/degraded/offline
  - `version`, `last_health`, `last_seen_at`
  - `config_snapshot` - 缓存的 Gateway 配置快照
  - `tags` - 标签（用于分组筛选）

**配置管理**
- `config_templates` - 配置模板（partial OpenClawConfig JSONB）
- `config_policies` - 策略执行（enforced_paths, enforced_values, scope: org/department/user）

**审计与用量**
- `audit_log` - 审计日志（user_id, gateway_id, action, resource_type, details, ip_address）
- `usage_snapshots` - 用量快照（gateway_id, period, metrics JSONB: tokens, cost, sessions）

**共享资源**
- `shared_skills` - 共享技能（name, content, visibility: org/department/public）
- `api_keys` - 编程访问密钥（org_id, user_id, key_hash, scopes, expires_at）

---

## 3. API 设计

### REST API（`/api/v1`）

统一后端 API，通过 RBAC 控制数据范围。用户入口调用时自动过滤为当前用户数据，管理员入口可访问全量数据。

**用户入口 API（任何已登录用户）**
```
-- 我的助手
GET               /me/assistant                            -- 获取我的助手状态和摘要
PATCH             /me/assistant/config                     -- 更新助手配置（身份、模型、工具）
GET               /me/assistant/channels                   -- 我的渠道绑定
PUT               /me/assistant/workspace/:filename        -- 编辑 SOUL.md 等 workspace 文件

-- 对话
POST              /me/chat                                 -- 发起新对话
GET               /me/sessions                             -- 我的会话列表
GET               /me/sessions/:sessionId                  -- 会话详情（transcript）
GET               /me/sessions/:sessionId/events           -- 会话事件流

-- 定时任务
GET/POST          /me/cron                                 -- 我的定时任务
PATCH/DELETE      /me/cron/:jobId
POST              /me/cron/:jobId/run                      -- 手动触发

-- 技能市场
GET               /skills                                  -- 浏览可用技能（组织+公共）
GET               /skills/:skillId                         -- 技能详情
POST              /me/skills/:skillId/install              -- 安装技能
DELETE            /me/skills/:skillId/uninstall             -- 卸载技能
GET               /me/skills                               -- 我已安装的技能

-- 用量
GET               /me/usage                                -- 我的用量统计
```

**管理员入口 API（org:admin 以上）**
```
-- 组织管理
POST/GET          /orgs
GET/PATCH/DELETE  /orgs/:orgId
POST              /orgs/:orgId/invites
GET/PATCH/DELETE  /orgs/:orgId/members/:userId

-- 用户 Gateway 管理（1 用户 = 1 Gateway）
GET               /orgs/:orgId/gateways                    -- 全量用户 Gateway 列表（含状态）
GET               /orgs/:orgId/gateways/:gwId              -- 某用户 Gateway 详情
PATCH             /orgs/:orgId/gateways/:gwId/config       -- 管理员修改用户 Gateway 配置
POST              /orgs/:orgId/gateways/:gwId/connect      -- 测试连接
GET               /orgs/:orgId/gateways/:gwId/health       -- 健康检查
POST              /orgs/:orgId/gateways/:gwId/rpc          -- 通用 RPC 代理
POST              /orgs/:orgId/gateways/batch              -- 批量操作（创建/启停/应用模板）

-- 配置与策略
POST/GET          /orgs/:orgId/config-templates
POST              /orgs/:orgId/config-templates/:id/apply  -- 批量下发到指定用户
POST/GET          /orgs/:orgId/config-policies

-- 技能管理
POST              /orgs/:orgId/skills                      -- 上架技能
PATCH/DELETE      /orgs/:orgId/skills/:skillId             -- 编辑/下架
POST              /orgs/:orgId/skills/:skillId/distribute  -- 推送到指定用户

-- 监控与分析
GET               /orgs/:orgId/analytics/usage             -- 全局用量
GET               /orgs/:orgId/analytics/cost              -- 按用户成本分摊
GET               /orgs/:orgId/audit-log                   -- 审计日志

-- 系统设置
GET/POST/DELETE   /orgs/:orgId/api-keys
```

### WebSocket 实时推送

两个入口共享同一 WebSocket 连接，按角色过滤推送事件：

**用户入口事件**：
- `assistant.status.changed` - 我的助手状态变更
- `chat.message` - 新对话消息（流式）
- `cron.executed` - 定时任务执行结果
- `skill.update` - 已安装技能更新通知

**管理员入口事件**（额外）：
- `gateway.status.changed` - Gateway 状态变更
- `gateway.health.alert` - 健康告警
- `user.alert` - 用户助手异常
- `audit.new` - 新审计条目
- `usage.threshold` - 用量阈值告警

### 认证方式

- **用户登录**: OAuth2/OIDC（Google、GitHub、Azure AD、Okta）、SAML、本地密码
- **编程访问**: API Key（带 scope 和过期时间）
- **管理平面→Gateway**: 使用 Gateway 现有 operator token 认证

---

## 4. UI/UX 结构

平台分为两个独立入口，面向不同角色：

### 4.1 用户入口（User Portal）

面向每个用户，管理自己的个人助手/数字员工。路由前缀：`/app`

```
用户入口
├── 首页（Dashboard）
│   ├── 助手状态卡片           -- 在线/离线、当前模型、运行时长
│   ├── 今日摘要               -- 今日处理消息数、执行任务数、消耗 token
│   ├── 最近对话               -- 最近 N 条对话摘要，点击进入完整对话
│   ├── 任务时间线             -- 定时任务执行历史、Subagent 任务记录
│   └── 通知                  -- 助手异常、任务失败、配额告警
│
├── 助手配置（My Assistant）
│   ├── 身份设置               -- 名称、头像、Emoji、主题色
│   ├── 模型选择               -- LLM 提供商/模型切换
│   ├── 渠道绑定               -- 查看/管理已连接的渠道（Telegram、Slack 等）
│   ├── 工具与权限             -- 启用/禁用的工具列表、执行审批设置
│   ├── 定时任务               -- 个人 cron job 配置（提醒、日报等）
│   └── 个性化文件             -- SOUL.md、IDENTITY.md 等 workspace 文件编辑
│
├── 对话（Chat）
│   ├── 对话界面               -- 实时对话（WebSocket），支持多轮
│   ├── 会话列表               -- 按渠道/时间筛选历史会话
│   ├── 会话详情               -- 完整 transcript、工具调用记录、思考过程
│   └── 新建对话               -- 发起新对话
│
└── 技能市场（Skill Store）
    ├── 推荐技能               -- 组织推荐 / 热门技能
    ├── 分类浏览               -- 按类别浏览可用技能
    ├── 技能详情               -- 描述、用法、安装/卸载
    └── 我的技能               -- 已安装技能列表、启用/禁用
```

### 4.2 管理员入口（Admin Portal）

面向组织管理员，统一管理所有用户的助手实例。路由前缀：`/admin`

```
管理员入口
├── 概览（Overview）
│   ├── 组织健康大盘           -- 在线 Gateway 数、异常告警
│   ├── 用量趋势图             -- Token 消耗、API 调用、成本趋势（日/周/月）
│   ├── 活跃排行               -- 最活跃的用户/助手 Top N
│   └── 告警列表               -- Gateway 离线、渠道断连、配额超限
│
├── 用户管理（Users）
│   ├── 用户列表               -- 全量用户及其 Gateway 状态（在线/离线/未配置）
│   ├── 用户详情               -- 查看某用户的 Gateway 配置、会话历史、用量
│   ├── 批量操作               -- 批量创建 Gateway、批量应用模板、批量启停
│   ├── 创建/删除 Gateway      -- 通过 K8s API 为用户创建/销毁 Gateway 实例
│   └── 部门视图               -- 按部门查看用户助手分布
│
├── 监控与分析（Analytics）
│   ├── 用量仪表盘             -- 按组织/部门/个人的 Token、成本、会话统计
│   ├── 成本分摊               -- 按部门/用户的成本归属报表
│   └── 审计日志               -- 全量操作审计（谁在什么时候做了什么）
│
├── 配置中心（Configuration）
│   ├── 配置模板               -- 创建/编辑标准配置模板
│   ├── 模板下发               -- 选择目标（全组织/部门/个人）批量下发
│   └── 策略执行               -- 强制策略（模型白名单、工具限制、安全规则）
│
├── 技能管理（Skills）
│   ├── 技能库                 -- 组织内可用技能的管理（上架/下架/审核）
│   ├── 技能分发               -- 向指定用户推送技能
│   └── 自定义技能             -- 上传/创建组织私有技能
│
└── 系统设置（Settings）
    ├── 组织信息               -- 名称、Logo、域名
    ├── 成员管理               -- 邀请/移除成员、角色分配
    ├── API 密钥               -- 编程访问密钥管理
    └── 账单                   -- 用量配额、付费计划
```

### 4.3 两个入口的关系

```
┌─────────────────────────────────────────────────┐
│                 Management API                  │
│            (统一后端，按角色鉴权)                  │
├────────────────────┬────────────────────────────┤
│                    │                            │
│   /app/*           │         /admin/*           │
│   用户入口          │         管理员入口           │
│                    │                            │
│   • 只能看到自己    │   • 可以看到所有人           │
│     的助手和数据    │     的助手和数据             │
│   • 操作范围限于    │   • 全局配置下发             │
│     个人 Gateway   │   • 批量管理操作             │
│   • 从技能市场      │   • 技能上架审核             │
│     安装技能       │   • 用量监控与成本分摊        │
│                    │                            │
└────────────────────┴────────────────────────────┘
         共享同一套 REST + WebSocket API
         通过 RBAC 控制数据可见性和操作权限
```

**路由鉴权规则**：
- `/app/*` 路由：任何已登录用户可访问，API 过滤只返回当前用户的 Gateway 数据
- `/admin/*` 路由：仅 `org:owner` 和 `org:admin` 角色可访问
- 同一用户可同时拥有两个入口的权限（如管理员既管理自己的助手，也管理其他用户）

### 4.4 关键页面说明

**用户入口**：
- **首页**：一屏概览助手状态，重点是"我的助手在做什么"、"最近发生了什么"
- **对话界面**：与现有 Control UI 的 chat 功能对齐，但嵌入管理平台的统一 UI 框架
- **技能市场**：类似 App Store 体验，组织管理员上架的技能 + 公共技能

**管理员入口**：
- **用户管理**：核心页面，一目了然看到每个用户的 Gateway 是否在线、用量情况，可为用户创建/销毁 Gateway
- **配置模板下发**：可视化编辑 OpenClawConfig 片段，选择目标范围后一键下发，下发前 diff 预览
- **审计日志**：可搜索、可筛选的时间线，合并管理平面事件和 Gateway 转发事件

---

## 5. Gateway 自动化部署与注册

### 5.1 全自动部署流程

```
管理员点击"为用户创建助手"
  → Management API 生成 operator_token + registration_token
  → 存储到 DB gateways 表（加密）
  → 调用 K8s API 创建 Secret + ConfigMap + StatefulSet + Service
  → K8s 拉起 Gateway Pod
  → Gateway 启动:
      ① device identity 自动生成（ED25519 keypair）
      ② gateway server 启动，/healthz 返回 200
      ③ 检测到 OPENCLAW_MANAGEMENT_URL 环境变量
      ④ 出站 WebSocket 回连管理平面，携带 registration_token
  → 管理平面验证 token，匹配 gateways 记录
  → 更新 status = "online"
  → Gateway Controller 建立持久双向通信
  → 管理 UI 显示用户助手在线
```

### 5.2 K8s 资源模板

管理平面为每个用户创建以下 K8s 资源：

```yaml
# Secret — 凭据（管理平面生成并注入）
apiVersion: v1
kind: Secret
metadata:
  name: gw-{userId}-secrets
  labels:
    openclaw.ai/user-id: "{userId}"
stringData:
  gateway-token: "{generated_operator_token}"
  management-token: "{registration_token}"
  management-url: "https://manage.openclaw.example.com"

---
# ConfigMap — 最小启动配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: gw-{userId}-config
data:
  openclaw.json: |
    {
      "gateway": {
        "mode": "local",
        "bind": "lan",
        "auth": { "mode": "token" }
      }
    }

---
# StatefulSet — Gateway 实例（replicas: 1）
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: gw-{userId}
spec:
  replicas: 1
  serviceName: gw-{userId}
  template:
    spec:
      containers:
      - name: openclaw
        image: openclaw:latest
        command: ["node", "openclaw.mjs", "gateway", "--bind", "lan"]
        env:
        - name: OPENCLAW_GATEWAY_TOKEN
          valueFrom: { secretKeyRef: { name: gw-{userId}-secrets, key: gateway-token } }
        - name: OPENCLAW_MANAGEMENT_URL
          valueFrom: { secretKeyRef: { name: gw-{userId}-secrets, key: management-url } }
        - name: OPENCLAW_MANAGEMENT_TOKEN
          valueFrom: { secretKeyRef: { name: gw-{userId}-secrets, key: management-token } }
        - name: OPENCLAW_CONFIG_PATH
          value: /etc/openclaw/openclaw.json
        - name: OPENCLAW_STATE_DIR
          value: /data/state
        - name: OPENCLAW_NO_RESPAWN
          value: "1"
        volumeMounts:
        - { name: config, mountPath: /etc/openclaw, readOnly: true }
        - { name: state, mountPath: /data/state }
        livenessProbe:
          httpGet: { path: /healthz, port: 18789 }
        readinessProbe:
          httpGet: { path: /readyz, port: 18789 }
      terminationGracePeriodSeconds: 40
      volumes:
      - { name: config, configMap: { name: gw-{userId}-config } }
  volumeClaimTemplates:
  - metadata: { name: state }
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: { requests: { storage: 10Gi } }
```

### 5.3 两种连接模式

| 模式 | 方向 | 适用场景 |
|------|------|---------|
| **Pull（管理平面→Gateway）** | Gateway Controller 主动连接 Gateway WS | K8s 内网，Gateway 有稳定 Service 地址 |
| **Push（Gateway→管理平面）** | Gateway 出站连接管理平面 | NAT/防火墙环境，或作为 Pull 的补充 |

K8s 内网中两种模式**同时使用**：
- Pull：`ws://gw-{userId}.openclaw-gateways.svc:18789`（通过 K8s Service）
- Push：Gateway 检测到 `OPENCLAW_MANAGEMENT_URL` 后自动回连

### 5.4 Gateway 侧自动注册代码

Gateway 启动时新增的自动注册逻辑（`src/gateway/server-manage.ts`）：

```typescript
// 启动时检查环境变量，存在则自动注册到管理平面
async function maybeAutoRegister() {
  const url = process.env.OPENCLAW_MANAGEMENT_URL;
  const token = process.env.OPENCLAW_MANAGEMENT_TOKEN;
  if (!url || !token) return; // 无管理平面配置，跳过

  const device = await loadOrCreateDeviceIdentity();
  const ws = new WebSocket(`${url}/api/v1/gateway/register`);
  ws.on("open", () => {
    ws.send(JSON.stringify({
      type: "register",
      token,
      deviceId: device.deviceId,
      version: pkg.version,
      publicKeyPem: device.publicKeyPem,
    }));
  });
  // 心跳保活 + 断线重连
}
```

也支持 `ManageConfig` 配置段（用于非 K8s 环境手动注册）：

```typescript
type ManageConfig = {
  managementUrl?: string;
  registrationToken?: SecretInput;
  heartbeatIntervalSec?: number;  // 默认 30s
  auditForwarding?: boolean;
  usageForwarding?: boolean;
};
```

环境变量优先级高于配置文件，方便 K8s 注入。

### 5.5 RPC 代理模式

Gateway Controller 连接 Gateway 后，管理 UI 的所有操作通过 RPC 代理：

```
Management UI → Management API → Gateway Controller → Gateway WS → Gateway RPC
                 (RBAC 校验)      (连接池)             (operator token)
```

管理 API 在转发前执行自己的 RBAC 检查（确认当前用户有权操作该 Gateway），叠加在 Gateway 自身的 scope 系统之上。

### 5.6 配置下发流程

管理员下发配置模板到用户的 Gateway：

```
管理员选择模板 + 目标用户
  → Management API 校验权限
  → 通过 Gateway Controller 调用目标 Gateway 的 config.patch RPC
  → Gateway 合并配置，热重载或重启
  → Gateway 返回新配置快照
  → Management API 更新 DB 中的 config_snapshot
```

### 5.7 关键文件

| 文件 | 用途 |
|------|------|
| `src/gateway/client.ts` | 现有 Gateway WS 客户端，Gateway Controller 复用 |
| `src/infra/device-identity.ts` | device identity 自动生成（ED25519） |
| `src/gateway/startup-auth.ts` | Gateway 启动认证解析 |
| `src/config/runtime-overrides.ts` | 环境变量覆盖配置 |
| `src/gateway/server/readiness.ts` | /readyz 健康检查逻辑 |
| `src/gateway/server-manage.ts` | **新增** — 自动注册到管理平面 |
| `src/config/types.openclaw.ts` | **扩展** — ManageConfig 类型 |
| `packages/management-api/src/gateway-ctrl/connector.ts` | **新增** — Gateway Controller 连接管理 |
| `packages/management-api/src/k8s/gateway-operator.ts` | **新增** — K8s 资源编排 |
| `packages/management-api/src/k8s/resource-templates.ts` | **新增** — K8s YAML 模板生成 |

---

## 6. 安全模型

### 三层认证

| 层级 | 认证方式 | 说明 |
|------|---------|------|
| **管理平面** | OAuth2/OIDC, SAML, API Key | 用户登录管理 UI / 编程调用 |
| **管理平面→Gateway** | Operator Token（加密存储） | Gateway Controller 连接各用户 Gateway |
| **Gateway 内部** | Device Auth, Token | 移动端/CLI 直连 Gateway |

### RBAC 权限模型

```
Permission = { resource, action }
  resource: org | gateway | config | session | skill | audit
  action:   create | read | update | delete | execute

系统角色:
  org:owner  - 所有权限
  org:admin  - 除删除组织/账单外的所有权限
  org:member - 自己的 Gateway 读写 + 技能市场访问

用户只能操作自己的 Gateway，管理员可操作所有用户的 Gateway
```

### 安全加固

- 管理平面到 Gateway 强制 TLS（复用 `GatewayTlsConfig`）
- Gateway 凭据 AES-256-GCM 加密存储
- 管理 API 速率限制
- 所有状态变更操作审计日志
- CSRF + CSP 保护

---

## 7. 技术栈

### 后端（Management API）

| 技术 | 选型理由 |
|------|---------|
| Node.js 22+ | 与 OpenClaw 一致，可共享类型 |
| Fastify | 轻量、Schema-first、优秀 TS 支持 |
| PostgreSQL 15+ | JSONB 灵活 schema、RLS 多租户 |
| Drizzle ORM | TypeScript 原生、SQL-first |
| ws | 与 Gateway 相同的 WebSocket 库 |
| Lucia + Arctic | OAuth2/OIDC 认证 |
| BullMQ + Redis（可选） | 异步任务队列、会话缓存 |

### 前端（Management UI）

| 技术 | 选型理由 |
|------|---------|
| React 19 + TypeScript | 管理后台生态成熟 |
| TanStack Router + Query | 类型安全路由和服务端状态 |
| shadcn/ui + Tailwind | 可定制组件库 |
| Recharts | 分析图表 |
| TanStack Table | 数据表格 |
| Vite | 与现有 UI 构建工具一致 |

### 代码组织

```
packages/
  management-api/          -- Fastify 后端
    src/
      routes/              -- API 路由
      services/            -- 业务逻辑
      db/
        migrations/        -- DB 迁移
        schema.ts          -- Drizzle schema
      gateway-ctrl/        -- Gateway Controller
      auth/                -- 认证模块
    package.json
  management-ui/           -- React 前端
    src/
      pages/               -- 页面组件
      components/          -- 通用组件
      hooks/               -- React hooks
    package.json
```

---

## 8. K8s 部署架构

### 8.1 部署模型：每个 Gateway 一个 StatefulSet Pod

OpenClaw Gateway 是**单进程、文件系统存储**的设计，核心子系统（Channel 连接、Cron 调度、SubagentRegistry、NodeRegistry）均为内存态单例。因此 K8s 部署采用**多个独立 Gateway 实例**模式，由管理平面统一编排。

```
┌─────────────────────────────────────────────────────┐
│                   K8s Cluster                       │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │ Namespace: openclaw-management              │    │
│  │  ┌──────────────┐  ┌───────────────────┐    │    │
│  │  │ Management   │  │ PostgreSQL        │    │    │
│  │  │ API + UI     │  │ (Deployment+PVC)  │    │    │
│  │  │ (Deployment) │  └───────────────────┘    │    │
│  │  └──────────────┘                           │    │
│  └─────────────────────────────────────────────┘    │
│                        │ WS/HTTP                    │
│  ┌─────────────────────┼───────────────────────┐    │
│  │ Namespace: openclaw-gateways                │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │    │
│  │  │ GW Pod 0 │  │ GW Pod 1 │  │ GW Pod 2 │  │    │
│  │  │ (SS)     │  │ (SS)     │  │ (SS)     │  │    │
│  │  │ PVC-0    │  │ PVC-1    │  │ PVC-2    │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ┌─────────────────────────────────────────────┐    │
│  │ Ingress Controller                          │    │
│  │ gw-0.openclaw.example.com → GW Pod 0       │    │
│  │ gw-1.openclaw.example.com → GW Pod 1       │    │
│  │ manage.openclaw.example.com → Management   │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### 8.2 Gateway StatefulSet 配置

每个 Gateway 实例作为 StatefulSet（replicas: 1）部署，确保稳定网络标识和持久存储：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: openclaw-gw
  namespace: openclaw-gateways
spec:
  replicas: 1          # 单副本，Channel 连接单例
  serviceName: openclaw-gw
  template:
    spec:
      containers:
      - name: openclaw
        image: openclaw:latest
        command: ["node", "dist/index.js", "gateway", "--bind", "lan", "--port", "18789"]
        ports:
        - containerPort: 18789
          name: gateway
        env:
        - name: OPENCLAW_NO_RESPAWN
          value: "1"    # 禁用进程 respawn，由 K8s 管理
        - name: OPENCLAW_GATEWAY_TOKEN
          valueFrom:
            secretKeyRef:
              name: openclaw-secrets
              key: gateway-token
        volumeMounts:
        - name: state
          mountPath: /home/node/.openclaw
        livenessProbe:
          httpGet: { path: /healthz, port: 18789 }
          initialDelaySeconds: 15
          periodSeconds: 30
        readinessProbe:
          httpGet: { path: /readyz, port: 18789 }
          initialDelaySeconds: 20
          periodSeconds: 5
      terminationGracePeriodSeconds: 40  # 30s drain + 5s cleanup + buffer
  volumeClaimTemplates:
  - metadata:
      name: state
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### 8.3 管理平面部署

管理平面（Management API + UI）作为无状态 Deployment，可水平扩展：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-management
  namespace: openclaw-management
spec:
  replicas: 2    # 可水平扩展
  template:
    spec:
      containers:
      - name: management-api
        image: openclaw-management:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: management-secrets
              key: database-url
```

### 8.4 K8s 部署需解决的关键问题

| 问题 | 方案 |
|------|------|
| **Channel 连接单例** | 每个 Gateway Pod replicas=1，不同 bot token 分配到不同 Pod |
| **Cron 无分布式去重** | 单副本保证；管理平面可协调跨实例 cron |
| **状态持久化** | StatefulSet + PVC（每 Pod 独立存储） |
| **Webhook 回调地址** | 每个 Gateway Pod 独立 Ingress 域名或 Tailscale Funnel |
| **WebSocket sticky** | 单副本无需；管理平面 WS 用 Session Affinity |
| **Gateway 绑定地址** | 必须 `--bind lan`（默认 loopback 不可达） |
| **进程 respawn** | 设 `OPENCLAW_NO_RESPAWN=1`，由 K8s 管理重启 |
| **优雅关停** | `terminationGracePeriodSeconds: 40` |
| **敏感数据** | K8s Secrets 存储 bot token、API key、gateway token |
| **运行时包安装持久化** | 见 8.5 节详细方案 |

### 8.5 运行时包安装持久化

OpenClaw 在运行时会动态安装包，Pod 重启后非持久化路径的内容会丢失：

| 安装类型 | 运行时命令 | 默认安装路径 | PVC 覆盖？ |
|----------|-----------|-------------|-----------|
| **插件/扩展** | `npm install --omit=dev` | `~/.openclaw/extensions/<id>/` | 已覆盖（PVC 挂载 `~/.openclaw`） |
| **技能 npm 全局包** | `npm install -g` | `/usr/local/lib/node_modules/` | 未覆盖，重启丢失 |
| **技能 Go 模块** | `go install` | `/usr/local/go/bin/` | 未覆盖，重启丢失 |
| **技能 Python 包** | `uv tool install` | `~/.local/bin/` | 未覆盖，重启丢失 |
| **技能 Brew 包** | `brew install` | `/home/linuxbrew/` | 未覆盖，重启丢失 |
| **技能系统包** | `apt-get install` | `/usr/bin/` 等系统路径 | 未覆盖，重启丢失 |

**方案：将全局安装路径重定向到 PVC**

通过环境变量将各包管理器的全局安装目录指向 PVC 挂载路径下的子目录：

```yaml
env:
# 将全局包安装路径重定向到 PVC
- name: NPM_CONFIG_PREFIX
  value: /data/state/.global/npm          # npm -g 安装到 PVC
- name: GOPATH
  value: /data/state/.global/go           # go install 安装到 PVC
- name: UV_TOOL_DIR
  value: /data/state/.global/uv           # uv tool 安装到 PVC
- name: XDG_DATA_HOME
  value: /data/state/.global/share        # 通用 XDG 数据目录
- name: PATH
  value: /data/state/.global/npm/bin:/data/state/.global/go/bin:/data/state/.global/uv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
volumeMounts:
- name: state
  mountPath: /home/node/.openclaw         # 插件、配置、会话
- name: state
  mountPath: /data/state                  # 全局包持久化（同一 PVC，不同子路径）
  subPath: runtime
```

**方案对比**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **A. 环境变量重定向（推荐）** | 零侵入，不改 OpenClaw 代码；重启后包还在 | PVC 空间需预留更多；PATH 较长 |
| **B. Init Container 重装** | 镜像保持干净 | 每次重启慢（需网络）；内网需开包管理源白名单 |
| **C. 预装到自定义镜像** | 最快启动；无运行时网络依赖 | 不灵活，用户装新技能后需重新构建镜像 |
| **D. Overlay 持久层** | 对应用完全透明 | 需特权容器或 fuse-overlayfs，安全风险高 |

**推荐组合**：A（环境变量重定向）为默认方案 + C（预装常用技能到镜像）减少首次安装时间。

### 8.6 管理平面的 K8s 编排能力

管理平面应具备通过 K8s API 管理 Gateway 实例的能力：

- **创建 Gateway**: 通过 K8s API 创建新的 StatefulSet + PVC + Service + Ingress
- **删除 Gateway**: 清理 K8s 资源
- **扩缩容**: 管理 Gateway 实例数量（每个实例是独立的 StatefulSet）
- **配置注入**: 通过 ConfigMap/Secret 注入 Gateway 配置
- **健康监控**: 结合 K8s Pod 状态 + Gateway `/healthz` `/readyz`
- **滚动更新**: 更新 Gateway 镜像版本
- **日志聚合**: 通过 K8s 日志采集 Gateway 日志

新增文件：
- `packages/management-api/src/k8s/gateway-operator.ts` - K8s Gateway 生命周期管理
- `packages/management-api/src/k8s/resource-templates.ts` - K8s 资源模板（StatefulSet/Service/Ingress）
- `packages/management-api/src/k8s/health-sync.ts` - K8s Pod 状态与 Gateway 健康状态同步

---

## 9. 分阶段实施路线（30 天）

### Phase 1: 基础骨架（Day 1-8）
- **Day 1-2**: 项目初始化、DB schema、Drizzle 迁移、Fastify 骨架、认证（OAuth2 登录）
- **Day 3-4**: 组织/用户/团队 CRUD API + RBAC 中间件
- **Day 5-6**: Gateway Controller 核心 — 连接 Gateway WebSocket、RPC 代理、健康轮询
- **Day 7-8**: Gateway 手动注册（输入 WS URL + token）、Gateway 列表/状态聚合 API
- **交付物**: 可运行的后端，能为用户注册 Gateway 并读取状态，已有认证和权限

### Phase 2: 用户入口 UI（Day 9-16）
- **Day 9-10**: React 项目初始化、路由框架、布局组件、用户首页（助手状态卡片、今日摘要）
- **Day 11-12**: 对话界面 — WebSocket 实时聊天、会话列表、历史 transcript 查看
- **Day 13-14**: 助手配置页 — 身份设置、模型选择、渠道绑定、工具开关、workspace 文件编辑
- **Day 15-16**: 技能市场 — 浏览/安装/卸载、我的技能列表
- **交付物**: 用户可登录、查看助手状态、对话、配置助手、安装技能

### Phase 3: 管理员入口 UI + 监控（Day 17-24）
- **Day 17-18**: 管理员概览大盘 — 组织健康、用量趋势图（Recharts）、告警列表
- **Day 19-20**: 用户助手管理 — 全量用户列表、助手状态、批量操作（创建/启停/应用模板）
- **Day 21-22**: 配置中心 — 配置模板编辑器（JSON 可视化）、模板下发（选择目标 + diff 预览）、策略执行
- **Day 23-24**: Gateway 管理 + 审计日志 — 实例列表/详情、K8s StatefulSet 创建/删除、审计时间线
- **交付物**: 管理员可查看全局状态、管理用户助手、下发配置、查看审计日志

### Phase 4: K8s 部署 + 集成联调（Day 25-30）
- **Day 25-26**: Helm Chart — 管理平面 Deployment + PostgreSQL + Gateway StatefulSet 模板 + Ingress
- **Day 27-28**: `openclaw manage join/leave/status` CLI、Gateway 出站注册、事件转发（审计/健康/用量）
- **Day 29-30**: 端到端联调 — Kind 集群部署验证、Bug 修复、性能调优
- **交付物**: 可在 K8s 一键部署的完整平台，Gateway 自动注册和管理

### 每日产出节奏

```
Day  1  2  3  4  5  6  7  8 │ 9 10 11 12 13 14 15 16 │17 18 19 20 21 22 23 24 │25 26 27 28 29 30
     ├──── Phase 1 ─────────┤├──── Phase 2 ────────────┤├──── Phase 3 ────────────┤├── Phase 4 ────┤
     │ 后端骨架 + 认证 + RBAC │ 用户入口 4 板块           │ 管理员入口 + 监控分析     │ K8s + 联调     │
     │ GW Controller          │ 首页/对话/配置/技能市场   │ 大盘/用户管理/配置/审计   │ Helm + CLI     │
     │ Gateway 注册 + 状态    │                          │                          │ E2E 验证       │
```

---

## 10. 验证方案

1. **单元测试**: 各 API 路由、业务逻辑、权限检查
2. **集成测试**: 管理平面 ↔ Gateway 端到端（启动测试 Gateway，注册，执行 RPC）
3. **E2E 测试**: Management UI 关键流程（创建组织 → 注册 Gateway → 查看助手状态）
4. **安全测试**: RBAC 越权检查、凭据加密验证、CSRF/CSP 验证
5. **K8s 部署测试**: Helm install → 管理平面启动 → 创建 Gateway StatefulSet → Gateway 自动注册 → 用户视图展示
6. **手动验证**: 本地 Kind/Minikube 集群一键部署完整环境
