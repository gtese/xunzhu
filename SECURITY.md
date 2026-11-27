# 敏感信息与合规约定

## 本仓库允许出现

- 架构设计、数据模型、接口定义、实施计划、风险与决策记录
- 私网地址（`172.16.x.x`，不可路由）与域名，仅用于说明部署拓扑
- 组件选型与配置项**名称**（如 `shared_buffers=384MB`）

## 本仓库禁止出现

| 类别 | 示例 |
|---|---|
| 服务器与系统口令 | SSH 密码、sudo 口令、AD 域账号口令 |
| 云服务密钥 | 阿里云 AccessKey ID / Secret、DashScope API Key |
| 平台凭据 | 飞书 App Secret、Verification Token、Encrypt Key |
| 数据库与中间件口令 | PostgreSQL / Redis / MinIO 的账号与密码 |
| 证书私钥 | `privkey.pem` 及任何 PKCS#12 文件 |
| Token 与 Cookie | 任何形式的长期令牌、会话凭据 |
| 学生隐私数据 | 真实姓名与学号对应关系、作业照片原图、成绩明细 |

## 凭据存放位置

生产环境凭据统一存放于 `/opt/edu-tutor/.env`，文件权限 `600`，属主为部署账号，仅 `docker compose` 读取。运维侧凭据不在本仓库、不在文档、不在提交历史中。

## 提交前自检

提交前执行：

```
git diff --cached | grep -iE 'password|passwd|secret|api[_-]?key|access[_-]?key|token|BEGIN .*PRIVATE KEY'
```

命中即中止提交。`.gitignore` 已覆盖 `.env`、证书、私钥与运行期数据目录。

## 数据合规

学生数据仅存于内网服务器与阿里云 API 调用过程，不出境。作业照片保留期与自动清理策略见 [04-open-questions.md](04-open-questions.md) 第 7 项，待定。
