# cyeam-open

cyeam.com 开放接口网关，基于 [Zuplo](https://zuplo.com) 部署。

## 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/roadbook/csv` | 生成旅游线路图（转发到 cyeam.com） |
| GET | `/api/qrcode?text=` | 生成二维码（转发到 cyeam.com） |

基础 URL：`https://cyeam-open-main-d02895c.zuplo.site`

## 修改流程

1. 改 `config/routes.oas.json`（路由定义）或 `config/policies.json`（策略配置）
2. 提交到 main 分支
3. Push → Zuplo 自动检测变更 → 构建 → 部署（约 1-2 分钟生效）

**不要在 Zuplo Dashboard 直接编辑项目文件**，Dashboard 的改动会被下次 push 覆盖。

## 本地开发

```bash
npm install
npm run dev
```

打开 [http://localhost:9000](http://localhost:9000) 查看效果。

## 环境变量

| 名称 | 用途 |
|---|---|
| `CYEAM_BACKEND_TOKEN` | `set-headers-inbound` 策略注入的 `X-Zuplo-Token`，供 cyeam.com 后端校验 |

需在 Zuplo Dashboard → Settings → Environment Variables 中配置。

## 项目结构

```
cyeam-open/
├── config/
│   ├── routes.oas.json     # OpenAPI 3.1 路由定义
│   └── policies.json       # 鉴权/请求处理策略
├── modules/                # 自定义 handler
├── docs/                   # Zudoku 开发者门户
├── package.json
├── tsconfig.json
└── README.md
```
