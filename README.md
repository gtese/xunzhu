# xunzhu · 内网教材知识库 + 作业辅导系统

面向一个年级（约 15 名学生）的教材知识库问答与作业拍照辅导系统。部署在企业内网 Ubuntu 服务器上，以飞书网页应用（H5）为唯一入口，模型能力通过公有云 API 调用（无本地 GPU）。

## 当前状态

| 项 | 状态 |
|---|---|
| 阶段 | 设计定稿 v1.1，未进入编码 |
| 环境核查 | 已完成（OS / Docker / 端口 / 内存 / 磁盘实测） |
| 磁盘扩容 | 已完成，根分区 58G → 983G，在线扩容未重启 |
| 代码 | 尚无，仅设计与规划 |
| 上线阻塞项 | ICP 备案覆盖范围未确认 |

## 关键参数

| 项 | 取值 |
|---|---|
| 目标机 | `172.16.50.166`（Ubuntu 24.04.5 LTS，8C / 7.7G / 983G） |
| 域名 | `edu.v2txt.cn` |
| 对外端口 | `52066`（唯一对外，内部服务全部绑 `127.0.0.1`） |
| 前端 | Vue 3 + Vant 4 + KaTeX，飞书内嵌 H5 |
| 后端 | FastAPI + Uvicorn，RQ 异步任务 |
| 数据 | PostgreSQL 16 + pgvector（业务与向量合一）、Redis 7、MinIO |
| 模型 | 阿里云百炼 DashScope（qwen-max / qwen-vl-max / qwen-vl-ocr / qwen-turbo / text-embedding-v3 / CosyVoice） |
| 角色 | 学生 / 家长 / 管理员（无老师端） |
| 编排 | Docker Compose，独立 `edu-net` 网络，与现有 v2txt 栈零共享 |

## 文档索引

| 文档 | 内容 |
|---|---|
| [docs/01-requirements-and-decisions.md](docs/01-requirements-and-decisions.md) | 需求澄清过程、技术选型决策与否决项 |
| [docs/02-design-v1.1.md](docs/02-design-v1.1.md) | 架构设计正文（数据模型、解析管线、检索、API、实施计划、风险） |
| [docs/03-environment-and-progress.md](docs/03-environment-and-progress.md) | 服务器环境实测结论、扩容执行记录、端口与内存规划 |
| [docs/04-open-questions.md](docs/04-open-questions.md) | 未确定事项与待确认清单 |

## 敏感信息规则

本仓库**不存放**任何服务器口令、API Key、AccessKey/SecretKey、Token、证书私钥、`.env` 实际取值。所有凭据只存在于生产机 `/opt/edu-tutor/.env`（权限 600）与运维凭据库中。文档中出现的内网 IP、域名仅用于架构说明，属不可路由的私网地址。

## 版本

- v1.0 — 初版架构设计
- v1.1 — 确定对外端口 52066；磁盘扩容完成；角色由「学生/老师/管理员」调整为「学生/家长/管理员」，新增家长端绑定模型与接口；风险表补充家长越权防护
