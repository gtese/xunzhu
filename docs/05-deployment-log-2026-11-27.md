# 05 · 底座部署执行日志（2026-11-27）

记录 P0（环境）与 P1（底座）的实际落地过程、遇到的具体障碍与处置方式。所有凭据均未记录。

## 1. 执行流水

| # | 动作 | 结果 |
|---|---|---|
| 1 | 解析 `edu.v2txt.cn`（阿里/腾讯/Google DoH） | 初查无 A 记录，主域权威 NS 为 `apollo/rosemary.ns.cloudflare.com` |
| 2 | 探测出网白名单 | `open.feishu.cn`、`dashscope.aliyuncs.com`、`github.com` 可达；`registry-1.docker.io` 超时 |
| 3 | 出口公网 IP 确认 | `<PUBLIC_IP>` |
| 4 | `sudo` 提权方式修正 | 密码经 stdin 传入 `sudo -S`，expect 方案因无 prompt 会超时故弃用 |
| 5 | 创建 `/opt/edu-tutor` 目录骨架 | `nginx/conf.d`、`app`、`web`、`logs`、`certs`、`data/{pg,redis,minio,files,certbot}` |
| 6 | 写入 `.env`（umask 077，600） | 24 个键，PG/Redis/APP_SECRET 均为随机生成，未回显 |
| 7 | 写入 `docker-compose.yml` | `name: edu-tutor`，`edu-net` bridge |
| 8 | 拉取镜像 | `docker.1ms.run` 通道，本地 `docker tag` 成标准名 |
| 9 | `compose up` postgres + redis | 首次因镜像名回落 `docker.io` 被 403 拒绝，打短标签后成功 |
| 10 | 验证 PG 与 pgvector | `PostgreSQL 16.16`，`CREATE EXTENSION vector` 成功，`vector:0.8.7` |
| 11 | MinIO 多通道尝试 | 全部失败，见 §2.3；改用 local 存储 |
| 12 | Cloudflare token 权限复核 | 具备 `Zone:DNS:Edit`，可直接创建记录 |
| 13 | 创建 A 记录 | `edu.v2txt.cn` → `<PUBLIC_IP>`，`proxied=false`，TTL 300 |
| 14 | 拉取 `nginx:1.27-alpine` | 成功（74.5MB） |
| 15 | 安装 certbot | `certbot 2.9.0` + `python3-certbot-dns-cloudflare`，走阿里云 apt 源 |
| 16 | DNS-01 签发证书 | 成功，60s 传播等待，有效期至 2027-02-25 |
| 17 | 部署证书 + 续期钩子 | 拷贝至 `/opt/edu-tutor/certs`，注册 `renewal-hooks/deploy` |
| 18 | nginx 切 HTTPS（52066） | `nginx -t` 通过，严格证书校验 curl 成功 |
| 19 | compose 误注释修正 | 见 §2.4，整体重写后 `docker compose config` 通过 |
| 20 | 三容器齐活 + 隔离验证 | 见 `03` 文档 §6 |

## 2. 障碍与处置

### 2.1 Docker Hub 被投毒，官方加速器失效

`daemon.json` 中原配的阿里云个人加速器（`*.mirror.aliyuncs.com`）返回 **403 Forbidden**；`registry-1.docker.io` 直连超时。逐站探测后确认仅 `docker.1ms.run` 可用，且该站只对部分仓库有缓存 —— `minio/minio`、`chrislusf/seaweedfs`、`nginx:1.27-alpine` 曾出现 `NAME_UNKNOWN`，而 `pgvector/pgvector`、`redis:7-alpine`、`nginx:*` 正常。

处置：所有拉取统一走 `docker.1ms.run/library/<image>` 或 `docker.1ms.run/<org>/<repo>`，成功后 `docker tag` 为短名，避免 compose 回落 `docker.io`。

### 2.2 compose 镜像名回落导致 403

compose 中写 `image: redis:7-alpine` 时，Docker 会补全为 `docker.io/library/redis:7-alpine`，而 `docker.io` 恰好走失效的加速器 → 403。

处置：先 `docker pull docker.1ms.run/library/redis:7-alpine`，再 `docker tag` 成 `redis:7-alpine`，本地已有标准名后 compose 不再回源。

### 2.3 MinIO 社区镜像全渠道失效

| 通道 | 结果 |
|---|---|
| `docker.1ms.run/minio/minio` tags/list | `NAME_UNKNOWN`（上游 404） |
| `minio/minio:RELEASE.2025-07-23T15-54-02Z` 等 3 个 tag | 全部拉取失败 |
| `dl.min.io` 官方二进制 | HTTP **410**（已下架） |
| `quay.io` v2 / API | 401 |
| `bitnami/minio`（1ms） | 不可用 |

处置：`STORAGE_BACKEND=local`，应用层做存储抽象（local / s3 双实现）。备选 S3 方案为 Garage v1.1.0（实测 `docker.1ms.run/dxflrs/garage:v1.1.0` 可拉），配置项已预留，compose 中 MinIO 段以注释形式留档。

### 2.4 compose 注释范围越界

批量注释 MinIO 段时，因该段位于文件末尾之后仍包含 `edu-nginx` 定义，导致 nginx 服务一并被注释，`docker compose config --services` 只剩 `edu-redis` / `edu-postgres`。已在执行中通过 `config --services` 立即发现。

处置：整体重写 `docker-compose.yml`，把 MinIO 段移至文件末尾并以 `#` 前缀独立注释，`edu-nginx` 恢复到有效区。修正后 `config --quiet` 通过、三服务齐全。

### 2.5 内网访问公网 IP 不可用（NAT 回环）

从内网 Mac 对 `<PUBLIC_IP>` 的 80/443/8080/52066 全部 CLOSED，但同时服务器本机 `curl 127.0.0.1:52066` 正常。判断为公司出口 NAT 不支持 hairpin（回流），非端口封禁。

处置：不入内网自测结论，改由 4G 移动网络验证；服务端验证用 `--resolve` 指向 127.0.0.1 完成。

### 2.6 为何选 DNS-01 而非 HTTP-01

对外只开放 52066 一个端口，且该端口本身承载 HTTPS，ACME 的 80 端口校验无法完成；`--http-01-port` 又会与业务端口冲突。故采用 `dns-cloudflare` 插件做 DNS-01，只依赖 Cloudflare API 写权限，不依赖任何入向端口。

## 3. 复现步骤（从零到 HTTPS 可用）

```bash
# 1. 目录与凭据
sudo mkdir -p /opt/edu-tutor/{nginx/conf.d,app,web,logs,certs,data/{pg,redis,files,certbot}}
cd /opt/edu-tutor && umask 077 && cp deploy/.env.example .env && vi .env   # 填真实值

# 2. 镜像（经可用加速站拉取后打短名）
for i in library/redis:7-alpine library/nginx:1.27-alpine pgvector/pgvector:pg16; do
  docker pull docker.1ms.run/$i && docker tag docker.1ms.run/$i "${i#library/}"
done

# 3. 起底座
cp deploy/docker-compose.yml deploy/nginx-edu.conf .
cp deploy/nginx-edu.conf nginx/conf.d/edu.conf
docker compose config --quiet && docker compose up -d

# 4. 证书（需 Cloudflare API Token，权限 Zone:DNS:Edit）
apt-get install -y certbot python3-certbot-dns-cloudflare
install -m 600 /dev/stdin /root/.secrets/cf-edu.ini <<< 'dns_cloudflare_api_token = <TOKEN>'
certbot certonly --dns-cloudflare --dns-cloudflare-credentials /root/.secrets/cf-edu.ini \
  --dns-cloudflare-propagation-seconds 60 -d edu.v2txt.cn --agree-tos -m <EMAIL> -n

# 5. 部署证书并启用 TLS
install -m 644 /etc/letsencrypt/live/edu.v2txt.cn/fullchain.pem /opt/edu-tutor/certs/
install -m 600 /etc/letsencrypt/live/edu.v2txt.cn/privkey.pem  /opt/edu-tutor/certs/
docker exec edu-nginx nginx -t && docker exec edu-nginx nginx -s reload

# 6. 验证
curl -s --resolve edu.v2txt.cn:52066:127.0.0.1 https://edu.v2txt.cn:52066/healthz
```

DNS A 记录需在 Cloudflare 侧预先创建：`A edu.v2txt.cn -> <PUBLIC_IP>`，**Proxy 关闭（灰云）**。原因：Cloudflare 代理的请求体上限为 100MB（免费/Pro/Business 均如此），单册 200MB 教材会被直接拒绝。

## 4. 回滚方式

```bash
cd /opt/edu-tutor && docker compose down          # 停栈，数据卷保留在 ./data
docker network rm edu-net                          # 如需彻底清理
# DNS：Cloudflare 控制台删除 edu.v2txt.cn 的 A 记录
# 证书：certbot delete --cert-name edu.v2txt.cn
```

---
