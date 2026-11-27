# xunzhu · 内网教材知识库 + 作业辅导系统

面向一个年级（约 15 名学生）的教材知识库问答与作业拍照辅导系统。部署在企业内网 Ubuntu 服务器上，以飞书网页应用（H5）为唯一入口，模型能力通过公有云 API 调用（无本地 GPU）。

## 当前状态

| 项 | 状态 |
|---|---|
| 阶段 | **P1 底座已落地**（DB / 缓存 / 反代 / TLS 全部运行），P2 应用代码未开始 |
| 环境核查 | 已完成（OS / Docker / 端口 / 内存 / 磁盘实测） |
| 磁盘扩容 | 已完成，根分区 58G → 983G，在线扩容未重启 |
| 内存 | 已扩至 31 GiB，可用 29 GiB |
| DNS | **已完成**，`edu.v2txt.cn` A 记录指向源站（灰云，TTL 300） |
| TLS | **已完成**，Let's Encrypt DNS-01，有效期至 2027-02-25，自动续期已挂 |
| 底座容器 | `edu-postgres`(healthy) / `edu-redis`(healthy) / `edu-nginx` 运行中 |
| 代码 | 尚无，仅设计与底座配置 |
| 上线阻塞项 | 公网 52066 入向放行、ICP 备案覆盖范围、飞书自建应用创建 |

## 关键参数

| 项 | 取值 |
|---|---|
| 目标机 | `172.16.50.166`（Ubuntu 24.04.5 LTS，8C / 31G / 983G） |
| 域名 | `edu.v2txt.cn`（Cloudflare 权威，灰云直连源站） |
| 对外端口 | `52066`（唯一对外，内部服务全部绑 `127.0.0.1`） |
| 前端 | Vue 3 + Vant 4 + KaTeX，飞书内嵌 H5 |
| 后端 | FastAPI + Uvicorn，RQ 异步任务 |
| 数据 | PostgreSQL 16.16 + pgvector 0.8.7（业务与向量合一）、Redis 7 |
| 文件存储 | 本地卷 `/opt/edu-tutor/data/files`（原 MinIO 方案因镜像下架调整，见 `docs/05` §2.3） |
| 模型 | 阿里云百炼 DashScope（qwen-plus / qwen-vl-max / qwen-vl-ocr / text-embedding-v3） |
| 角色 | 学生 / 家长 / 管理员（无老师端） |
| 编排 | Docker Compose，独立 `edu-net` 网络，与现有 v2txt 栈零共享 |

## 目录结构

```
.
├── README.md
├── SECURITY.md
├── deploy/                          # 实际部署配置（已脱敏，可直接复现）
│   ├── docker-compose.yml           # 三服务编排（MinIO 段以注释留档）
│   ├── nginx-edu.conf               # 52066 端口 HTTPS 反代
│   └── .env.example                 # 24 个环境变量占位，无真实值
└── docs/
    ├── 01-requirements-and-decisions.md
    ├── 02-design-v1.1.md
    ├── 03-environment-and-progress.md
    ├── 04-open-questions.md
    └── 05-deployment-log-2026-11-27.md
```

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/01-requirements-and-decisions.md](docs/01-requirements-and-decisions.md) | 需求澄清过程、技术选型决策与否决项 |
| [docs/02-design-v1.1.md](docs/02-design-v1.1.md) | 架构设计正文（数据模型、解析管线、检索、API、实施计划、风险） |
| [docs/03-environment-and-progress.md](docs/03-environment-and-progress.md) | 环境实测结论、扩容记录、端口内存规划、**底座落地实况** |
| [docs/04-open-questions.md](docs/04-open-questions.md) | 未确定事项与待确认清单 |
| [docs/05-deployment-log-2026-11-27.md](docs/05-deployment-log-2026-11-27.md) | **部署执行日志**：镜像通道、MinIO 替代、DNS-01 证书、复现与回滚步骤 |

## 敏感信息规则

本仓库**不存放**任何服务器口令、API Key、AccessKey/SecretKey、Token、证书私钥、`.env` 实际取值、Cloudflare 凭据。所有凭据只存在于生产机 `/opt/edu-tutor/.env`（600）与 `/root/.secrets/`（600）中。文档中出现的私网 IP 属不可路由地址，公网出口 IP 与出口域名已做 `<PUBLIC_IP>` 占位处理。

## 版本

- v1.0 — 初版架构设计
- v1.1 — 确定对外端口 52066；磁盘扩容完成；角色由「学生/老师/管理员」调整为「学生/家长/管理员」
- v1.2 — **底座落地**：DNS A 记录、Let's Encrypt 证书（DNS-01）、pgvector 16 与 Redis 7 运行、nginx 52066 HTTPS 生效、对象存储由 MinIO 调整为本地卷、补 `deploy/` 与部署日志

---
