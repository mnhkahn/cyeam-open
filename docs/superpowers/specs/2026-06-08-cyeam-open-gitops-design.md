# cyeam-open Zuplo GitOps 化设计

日期：2026-06-08
仓库：[mnhkahn/cyeam-open](https://github.com/mnhkahn/cyeam-open)
Zuplo 服务：`cyeam-open-main-d02895c.zuplo.site`

## 背景

当前 `cyeam-open` 在 Zuplo 上提供两个反向代理接口：

| 方法 | 路径 | 转发目标 |
|---|---|---|
| POST | `/api/roadbook/csv` | `https://www.cyeam.com`（生成旅游线路图） |
| GET  | `/api/qrcode`       | `https://www.cyeam.com`（生成二维码）   |

两者都通过 `urlForwardHandler` 转发，没有自定义业务逻辑。

**问题**：配置只存在于 Zuplo Dashboard 中，无版本管理、无 review、无变更历史；新增接口要在 UI 中点选，容易遗漏。

## 目标

- 把 Zuplo 项目的所有配置（路由、policies）作为代码托管在 GitHub
- 推送到 `main` 分支即自动部署到生产环境
- 现有两个接口的 URL、请求体、返回结构、鉴权策略全部保持不变
- 后续修改接口 = 改文件 + push，不再登录 Zuplo Dashboard

## 非目标

- 不引入 PR 预览环境（用户明确不需要）
- 不引入 GitHub Actions / 第三方 CI（Zuplo 自带 GitHub 集成已足够）
- 不重构接口逻辑（仍然是反向代理）

## 方案

### 部署链路

```
本地修改 → git push main
            ↓
Zuplo 监听 GitHub webhook
            ↓
拉取 cyeam-open 仓库
            ↓
读取 config/routes.oas.json 等约定文件
            ↓
构建 → 部署到 cyeam-open-main-d02895c.zuplo.site
```

Zuplo 与 GitHub 的连接在 Dashboard 一次性配置，不在仓库中。

### 仓库结构

Zuplo 项目结构是**强约定**，不能自定义文件名/路径：

```
cyeam-open/
├── config/
│   ├── routes.oas.json     # OpenAPI 3.1 路由定义（核心）
│   └── policies.json       # 入站/CORS 策略定义
├── modules/                # 自定义 handler 代码（当前为空，因为只用内置 urlForwardHandler）
├── tests/                  # 测试（暂不引入）
├── package.json            # 声明 zuplo 依赖
├── tsconfig.json           # TypeScript 配置（即使没 ts 文件也得有，否则 Zuplo 构建会报错）
├── .gitignore
├── README.md               # 项目说明 + 修改流程
└── docs/superpowers/specs/ # 本设计文档
```

### routes.oas.json

完整保留用户提供的 OAS：

- `paths./api/roadbook/csv.post` —— 含 `requestBody`（text/plain CSV）、`x-zuplo-route.handler` = `urlForwardHandler`、`baseUrl: https://www.cyeam.com`、`policies.inbound: [api-key-inbound-1, set-headers-inbound]`
- `paths./api/qrcode.get` —— 含 `parameters.text` query、handler 同上、`policies.inbound: [api-key-inbound]`
- `info.title` 改为 `cyeam-open`（原模板里是 "Todo API"，需要更正）
- 顶层 `components.securitySchemes` 保留

### policies.json

OAS 里只通过名字引用了三个 policy：`api-key-inbound`、`api-key-inbound-1`、`set-headers-inbound`。它们的完整定义不在 OAS 里，需要从 Zuplo Dashboard 拷过来。

**处理方式**：先在 policies.json 里写出**结构骨架 + 占位**，由用户从 Dashboard 复制每个 policy 的实际 JSON 填进去。这样既保证仓库是 single source of truth，又不在文档里强行编出可能不准的密钥配置。

```jsonc
{
  "policies": [
    {
      "name": "api-key-inbound",
      "policyType": "api-key-inbound",
      "handler": {
        "export": "ApiKeyInboundPolicy",
        "module": "$import(@zuplo/runtime)",
        "options": {
          // TODO: 从 Dashboard 拷 options
        }
      }
    }
    // ... 另外两个同理
  ]
}
```

### package.json

```json
{
  "name": "cyeam-open",
  "version": "1.0.0",
  "scripts": { "dev": "zuplo dev" },
  "dependencies": { "zuplo": "6.70.66" }
}
```

### Zuplo Dashboard 一次性操作

1. cyeam-open 项目 → Settings → GitHub → Connect Repository
2. 选 `mnhkahn/cyeam-open`，分支 `main`
3. 启用自动部署
4. （已完成，用户确认）

**重要约束**：连上 GitHub 之后，Dashboard 上对项目文件的直接编辑会被下次 push 覆盖。所有配置必须从仓库改，Dashboard 只用于查看部署状态、管理 Environment Variables、查看 API Keys。

## 数据流（请求路径）

以 `GET /api/qrcode?text=hello` 为例：

```
客户端
  → cyeam-open-main-d02895c.zuplo.site/api/qrcode?text=hello
    → Zuplo 边缘节点匹配 routes.oas.json 中的 path
    → 执行 inbound policies（api-key-inbound 校验密钥）
    → urlForwardHandler 转发到 https://www.cyeam.com/api/qrcode?text=hello
    → 返回响应
```

URL 行为与现在一字不差。

## 错误处理

- **构建失败**：Zuplo 部署面板会显示错误日志，旧版本继续在线，新版本不切流，因此推错不会让接口挂掉
- **policies.json 写错**：构建期会校验 schema，同上不切流
- **OAS 引用了不存在的 policy 名**：构建期报错，不部署

## 测试策略

不引入自动化测试。验证手段：

1. 每次 push 后，用 curl 调一次 `/api/qrcode?text=test`，确认返回 200
2. 检查响应 header 中的 `zuplo-build` ID 是否变了（变了 = 新版本生效）

## 迁移步骤

1. 在仓库写入：`config/routes.oas.json`、`config/policies.json`、`package.json`、`tsconfig.json`、`.gitignore`、更新 `README.md`
2. 提交并 push 到 `main`
3. 在 Zuplo Dashboard 监控部署进度
4. 用户从 Dashboard 把三个 policy 的实际定义补到 `policies.json`
5. 再次 push，验证生效
6. 之后所有变更走 git 流程

## 风险

| 风险 | 缓解 |
|---|---|
| policies.json 占位不完整导致鉴权失效 | 第一次 push 后立即用 curl 验证 401/200 行为；不通就回滚或在 Dashboard 临时改回 |
| Zuplo 自动部署失败 | 旧版本继续在线，不影响生产；查 Dashboard 日志修复后重 push |
| 仓库与 Dashboard 不同步 | 在 README 写明：所有配置改动只走仓库，禁止在 Dashboard 直接编辑 |
