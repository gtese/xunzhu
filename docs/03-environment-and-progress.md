# 03 · 环境实测与执行记录

## 1. 服务器实测（`172.16.50.166`）

| 项 | 实测值 | 结论 |
|---|---|---|
| 操作系统 | Ubuntu 24.04.5 LTS，内核 5.15.0-204 | apt 体系，非 Rocky/dnf |
| CPU | 8 核 | 够用 |
| 内存 | **31 GiB（已扩容），可用 29 GiB** | 宽裕，但仍保留限额防失控 |
| 磁盘 | `/` 扩容后 983G，实际使用 19G，可用 924G | 已解决 |
| Docker | 29.10.0 | 已安装 |
| Docker Compose | v5.8.0 | 已安装 |
| 现有容器 | `v2txt-api-1`，`172.16.50.166:8080->8000` | 需避让 |
| 52066 端口 | 原空闲，现已由 `edu-nginx` 占用 | 已落地 |
| 出口公网 IP | `<PUBLIC_IP>` | 已确认，用于 DNS A 记录 |

## 2. 磁盘扩容（已完成）

原始状态：虚拟磁盘 58G，根分区 `/` 48G，剩余约 29G。该容量无法承载教材多册 + 作业原始照片 + TTS 音频 + PostgreSQL 数据 + Docker 镜像（约 5G）。

处理过程：

1. 虚拟化层将虚拟盘从原容量扩至 1000G
2. 在线扩容，未重启业务：
   ```
   growpart /dev/sda 3
   pvresize /dev/sda3
   lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
   resize2fs /dev/ubuntu-vg/ubuntu-lv
   ```
3. 结果：`/` 从 48G 扩至 **983G，可用 925G**（文件系统为 ext4，非 XFS）
4. LVM 元数据备份留存于 `/var/backups/lvm/`

验证：扩容后现有 `v2txt-api-1` 容器运行正常，文件系统无异常。

## 3. 端口规划（实际落地状态）

| 服务 | 容器内 | 宿主暴露 | 状态 |
|---|---|---|---|
| nginx | 52066 | `0.0.0.0:52066` | **已运行**，唯一对外端口 |
| api | 8000 | `127.0.0.1:8001` | 待 P2 落地 |
| postgres | 5432 | `127.0.0.1:5433` | **已运行**（healthy） |
| redis | 6379 | `127.0.0.1:6380` | **已运行**（healthy） |
| minio api | 9000 | `127.0.0.1:9001` | 已放弃（见 §6.3） |
| minio console | 9001 | `127.0.0.1:9002` | 已放弃（见 §6.3） |

容器统一 `edu-*` 前缀，独立 `edu-net` bridge 网络（`172.19.0.0/16`），与现有 `v2txt_default` 网络零共享。

## 4. 内存限额（已按 31G 实况保留原限额）

| 服务 | limit | 实测 | 关键参数 |
|---|---|---|---|
| postgres | 1536M | `1610612736` | pgvector/pgvector:pg16 |
| redis | 256M | `268435456` | `maxmemory 192mb`, `allkeys-lru`, AOF |
| minio | 512M | — | 已停用 |
| nginx | 128M | `134217728` | 仅反代与 TLS |
| api | 1024M | — | 待落地 |
| worker | 1536M | — | RQ 并发 2，逐页流式处理 |

系统内存已扩至 31G，限额不再紧迫，但保留以约束异常泄漏。Worker 并发固定 2，逐页处理即释放，不累积整册，避免大页 300dpi 渲染触发 OOM。

## 5. 出网与镜像通道（实测）

内网无公网 TCP 出口，按域名白名单放行，Docker Hub 官方域名被 DNS 投毒 + SNI 拦截。

| 目标 | 结果 | 用途 |
|---|---|---|
| `registry-1.docker.io` | **不可用**（超时） | 官方 registry 被拦 |
| `<aliyun-mirror>`（daemon.json 内配置） | **403 Forbidden** | 原加速器已失效 |
| `docker.1ms.run` | **可用** | 实际拉取通道 |
| `hub.rat.dev` / `docker.xuanyuan.me` / `dockerpull.org` / `docker.1panel.live` / `docker.m.daocloud.io` | 不可用 | — |
| `dashscope.aliyuncs.com` | 可达 | 模型 API |
| `open.feishu.cn` | 可达 | 飞书开放平台 |
| `archive.ubuntu.com` / `mirrors.aliyun.com` | 200 | apt 装 certbot |

实际拉取方式：`docker pull docker.1ms.run/library/<image>` 后本地 `docker tag` 成标准名，供 compose 使用。

## 6. 底座落地实况（2026-11-27）

### 6.1 已运行服务

| 容器 | 镜像 | 状态 | 端口 |
|---|---|---|---|
| `edu-nginx` | `nginx:1.27-alpine` | Up | `0.0.0.0:52066 -> 52066` |
| `edu-postgres` | `pgvector/pgvector:pg16` | Up (healthy) | `127.0.0.1:5433 -> 5432` |
| `edu-redis` | `redis:7-alpine` | Up (healthy) | `127.0.0.1:6380 -> 6379` |

PostgreSQL 实测版本 `16.16 (Debian 16.16-1.pgdg12+1)`，扩展 `plpgsql:1.0` + **`vector:0.8.7`** 已创建成功，向量检索底座可用。

### 6.2 TLS 证书

| 项 | 值 |
|---|---|
| 签发方式 | certbot 2.9.0 + `dns-cloudflare` 插件，DNS-01 挑战 |
| 证书主体 | `CN=edu.v2txt.cn`，SAN `DNS:edu.v2txt.cn` |
| Issuer | `Let's Encrypt CN=YE2` |
| 有效期 | 2026-11-27 → **2027-02-25** |
| 序列号 | 前 8 位 `05480B6A` |
| 自动续期 | `certbot.timer`（systemd），部署钩子 `/etc/letsencrypt/renewal-hooks/deploy/edu-tutor.sh` 自动拷贝证书并 reload nginx |
| 私钥位置 | `/opt/edu-tutor/certs/privkey.pem`（600，root），**不入仓库** |

选用 DNS-01 而非 HTTP-01：对外只有 52066 一个端口且该端口本身要跑 HTTPS，无法让 ACME 在 80 端口完成校验。

### 6.3 对象存储方案变更（MinIO → 本地卷）

MinIO 社区版镜像已从 Docker Hub 下架，实测所有可得加速站均无法回源：

- `docker.1ms.run/minio/minio` tags/list → `NAME_UNKNOWN`（上游 404）
- 多个 RELEASE tag（2025-07-23 / 2025-04-22 / 2024-12-18）全部失败
- `dl.min.io` 官方二进制 → HTTP 410（已下架）
- `quay.io` API → 401

决策：改用**宿主机卷 + 应用层存储抽象**。

| 项 | 变更后 |
|---|---|
| 配置 | `STORAGE_BACKEND=local`，`STORAGE_LOCAL_DIR=/opt/edu-tutor/data/files` |
| 容量 | 受 924G 可用磁盘约束，单册 200MB 教材可容纳数千册 |
| 备选 | 可切 Garage（`docker.1ms.run/dxflrs/garage:v1.1.0` 实测可拉），保留 `S3_*` 配置项预留 |

compose 中 MinIO 段保留为注释，`STORAGE_BACKEND=s3` 时可按需启用。

### 6.4 网络隔离验证

| 检查 | 结果 |
|---|---|
| `edu-net` 存在 | `172.19.0.0/16` bridge |
| `v2txt_default` 独立 | 是，未共享 |
| 容器互解析 | `edu-nginx` 可解析 `edu-postgres` / `edu-redis`，不可达 `v2txt-api-1` 服务名 |
| 中间件对外暴露 | 无，PG/Redis 均只监听 `127.0.0.1` |

### 6.5 HTTPS 验证结果

```
$ curl --resolve edu.v2txt.cn:52066:127.0.0.1 https://edu.v2txt.cn:52066/healthz
edu-tutor-nginx-ok                      # 严格证书校验通过（未加 -k）

$ curl ... https://edu.v2txt.cn:52066/api/
{"detail":"api not deployed yet"}       # 503 占位，P2 接入后替换
```

nginx 已配置 `client_max_body_size 300m`、`proxy_read_timeout 300s`，满足单册 200MB 教材上传。

## 7. 配置文件落盘清单

| 路径 | 权限 | 是否入库 |
|---|---|---|
| `/opt/edu-tutor/docker-compose.yml` | 644 | 是（`deploy/docker-compose.yml`） |
| `/opt/edu-tutor/nginx/conf.d/edu.conf` | 644 | 是（`deploy/nginx-edu.conf`） |
| `/opt/edu-tutor/.env` | **600** | **否**，仅以 `deploy/.env.example` 占位形式入库 |
| `/opt/edu-tutor/certs/privkey.pem` | **600 root** | **否** |
| `/root/.secrets/cf-edu.ini` | **600 root** | **否** |
| `/opt/edu-tutor/data/{pg,redis,files}` | 700/755 | **否** |

## 8. 当前阻塞与待办

| 项 | 状态 | 说明 |
|---|---|---|
| DNS A 记录 | **已完成** | `edu.v2txt.cn` → `<PUBLIC_IP>`，灰云（DNS only），TTL 300，阿里/腾讯/服务器三方解析均一致 |
| TLS 证书 | **已完成** | DNS-01 签发，2027-02-25 到期，自动续期已挂 |
| 公网入向 52066 放行 | **待确认** | 华为防火墙需放行 `WAN → <PUBLIC_IP>:52066`；内网访问公网 IP 存在 NAT 回环限制，需用 4G 手机实测 |
| ICP 备案 | **待确认** | `v2txt.cn` 备案是否覆盖 `edu.v2txt.cn` 子域 |
| 飞书自建应用 | **待创建** | 需 App ID / App Secret，回填后重启 api |
| 应用代码 P2+ | **未开始** | FastAPI 骨架、前端 H5、教材解析管线 |

---
