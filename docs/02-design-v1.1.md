> **脱敏说明**：本文档已移除服务器口令、API Key、AccessKey/SecretKey、Token、证书私钥等一切凭据，仅保留架构与配置口径。生产环境实际取值统一存放于服务器 `/opt/edu-tutor/.env`（600 权限），不入版本库。

# 教材知识库 + 作业辅导系统 · 架构设计 v1.1

> 目标机：172.16.50.166 (Ubuntu 24.04.5 LTS)  
> 域名：edu.v2txt.cn  
> 规模：约 15 名学生，1 个年级数理化教材（原生电子版 PDF，单册约 200MB）  
> 模型：阿里云百炼 DashScope（无本地 GPU）  
> 日期：2026-11-27

---

## 1. 环境实测结果

| 项 | 实测值 | 判断 |
|---|---|---|
| 操作系统 | Ubuntu 24.04.5 LTS，内核 5.15.0-204 | 注意：apt 系，不是 Rocky/dnf |
| CPU | 8 核 | 够用 |
| 内存 | 7.7 GiB，已用 1.6G，可用 6.1G | 偏紧，需严格限额 |
| 磁盘 | `/` **983G，已用 17G，可用 925G** | 已扩容完成（见 B1） |
| Docker | 29.10.0 | 已装 |
| Docker Compose | v5.8.0 | 已装 |
| 现有容器 | `v2txt-api-1` → `172.16.50.166:8080->8000` | 已有应用，需避让 |
| 52066 端口 | 空闲 | 可用 |
| 监听端口总数 | 13 个 | 端口资源不紧张 |

### 必须解决的三个阻塞项

**B1. 磁盘不足**  
29G 剩余无法承载：教材（多册）、作业原始照片、TTS 音频、PostgreSQL 数据、Docker 镜像（约 5G）。  
建议扩到 200G。当前为 LVM（`ubuntu--vg-ubuntu--lv`），可在线扩容，无需停机：
```
lvextend -L +150G /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
前提是卷组有空闲 PE，需先 `vgs` / `pvs` 确认；若无空闲，需先在虚拟化层加盘再 `vgextend`。

**B2. ICP 备案**  
`edu.v2txt.cn` 解析到国内公网 IP，域名必须完成 ICP 备案（备案针对域名本身，不因使用 52066 非标准端口而豁免）。需确认 v2txt.cn 主域名备案是否覆盖该子域名。

**B3. 内存偏紧**  
7.7G 总量，扣掉现有 v2txt 应用占用，新栈必须限额在 5G 内。

---

## 2. 总体架构

```
飞书客户端（PC / 手机）
   │ HTTPS  https://edu.v2txt.cn
   ▼
公司公网 IP ──端口映射──► 172.16.50.166:52066   [nginx]
                              ├─ /          → H5 静态（Vue3 + Vant + KaTeX）
                              ├─ /api/      → api:8000
                              └─ /media/    → minio:9000
                         docker network: edu-net（独立 bridge，不接入现有栈）
                              ├─ edu-postgres  (pgvector)  127.0.0.1:5433
                              ├─ edu-redis                 127.0.0.1:6380
                              ├─ edu-minio                 127.0.0.1:9001/9002
                              ├─ edu-api       (FastAPI)
                              └─ edu-worker    (RQ, 2 并发)
外部：DashScope API / 阿里云 TTS / 飞书开放平台
```

### 端口规划（全部避让现有 8080）

| 服务 | 容器内 | 宿主暴露 | 说明 |
|---|---|---|---|
| nginx | 80 | `0.0.0.0:52066` | 唯一对外端口 |
| api | 8000 | `127.0.0.1:8001` | |
| postgres | 5432 | `127.0.0.1:5433` | 避让默认 5432 |
| redis | 6379 | `127.0.0.1:6380` | 避让默认 6379 |
| minio api | 9000 | `127.0.0.1:9001` | |
| minio console | 9001 | `127.0.0.1:9002` | |

容器命名统一 `edu-*` 前缀，独立 `edu-net` 网络，**不与 v2txt 栈共享任何网络、卷、凭据**。

### 内存限额（compose `deploy.resources.limits`）

| 服务 | limit | 关键参数 |
|---|---|---|
| postgres | 1536M | `shared_buffers=384MB`, `work_mem=16MB` |
| redis | 256M | `maxmemory 128mb`, `allkeys-lru` |
| minio | 512M | |
| api | 1024M | uvicorn 2 workers |
| worker | 1536M | RQ 并发=2，逐页流式处理 |
| nginx | 128M | |
| **合计** | **~5.0G** | 系统保留 ~2G |

---

## 3. 技术选型

| 层 | 选型 | 版本 | 理由 |
|---|---|---|---|
| 前端 | Vue 3 + Vant 4 + Vite | 3.4 / 4.x | 移动端 H5 成熟组合，飞书内嵌体验好 |
| 公式渲染 | KaTeX | 0.16 | 比 MathJax 轻，H5 首屏快 |
| 后端 | FastAPI + Uvicorn | 0.115 / 0.32 | 与 PDF、VLM 同生态 |
| 数据库 | PostgreSQL + pgvector | 16 / 0.7.x | 业务与向量合一，免维护 Milvus |
| 全文检索 | PG 内置 + zhparser | — | 中文分词，混合检索 |
| 队列 | Redis 7 + RQ | 7.x / 1.16 | 比 Celery 轻，任务量小 |
| 对象存储 | MinIO | RELEASE.2025 | S3 兼容，支持预签名直传 |
| PDF 解析 | PyMuPDF | 1.24 | 抽文字层 + 区域渲染 |
| 反向代理 | Nginx | 1.26 | TLS 终结，`client_max_body_size 0` |
| 容器 | Docker + Compose | 已有 | 单机编排 |
| 监控 | 可选 Uptime Kuma + 日志挂载 | — | 轻量 |

**明确不引入**：Kubernetes、Milvus、Kafka、Celery 集群、RAGFlow/Dify（需求定制度高，套框架反而受限）。

---

## 4. 目录结构

```
/opt/edu-tutor/
├── docker-compose.yml
├── .env                      # 600 权限，所有密钥集中于此
├── nginx/
│   ├── nginx.conf
│   └── certs/                # fullchain.pem / privkey.pem
├── app/                      # api 源码（或镜像）
│   ├── main.py
│   ├── routers/
│   ├── services/
│   │   ├── pdf_parser.py     # 三档解析管线
│   │   ├── chunker.py        # 章节感知切分
│   │   ├── retrieval.py      # 混合检索 + RRF
│   │   ├── tutor.py          # 教学推理
│   │   ├── ocr.py            # 作业图片识别
│   │   └── tts.py            # LaTeX→口语→语音
│   ├── workers/
│   └── migrations/
├── web/                      # H5 静态构建产物
└── data/
    ├── pg/
    ├── minio/
    ├── redis/
    └── tmp/                  # 解析中间文件，定期清理
```

---

## 5. 数据模型

```sql
-- 用户（飞书身份）
CREATE TABLE users (
  id            BIGSERIAL PRIMARY KEY,
  feishu_open_id TEXT UNIQUE NOT NULL,
  union_id      TEXT,
  name          TEXT,
  role          TEXT NOT NULL DEFAULT 'student',  -- student/parent/admin
  grade         TEXT,                              -- 学生：如 '高一'
  relation      TEXT,                              -- 家长：'母亲'/'父亲'/'监护人'
  status        TEXT DEFAULT 'active',
  created_at    TIMESTAMPTZ DEFAULT now()
);

-- 家长—学生绑定（支持一个家长多个孩子、一个孩子多个家长）
CREATE TABLE parent_student (
  parent_id   BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  student_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  is_primary  BOOLEAN DEFAULT false,   -- 主绑定家长，接收默认推送
  created_at  TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (parent_id, student_id)
);

-- 教材
CREATE TABLE textbooks (
  id          BIGSERIAL PRIMARY KEY,
  subject     TEXT NOT NULL,        -- 数学/物理/化学
  grade       TEXT NOT NULL,
  edition     TEXT,                 -- 人教版/苏教版…
  volume      TEXT,                 -- 上册/必修一
  title       TEXT NOT NULL,
  file_key    TEXT NOT NULL,        -- MinIO 对象键
  file_size   BIGINT,
  page_count  INT,
  status      TEXT DEFAULT 'pending', -- pending/parsing/ready/failed
  parse_stats JSONB,                -- 各档解析页数、耗时、成本
  created_by  BIGINT REFERENCES users(id),
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- 知识块
CREATE TABLE doc_chunks (
  id           BIGSERIAL PRIMARY KEY,
  textbook_id  BIGINT REFERENCES textbooks(id) ON DELETE CASCADE,
  page_no      INT NOT NULL,
  heading_path TEXT[],              -- ['第二章','2.3 牛顿第二定律']
  content      TEXT NOT NULL,       -- Markdown + LaTeX
  content_type TEXT DEFAULT 'text', -- text/formula/figure/table
  bbox         JSONB,               -- 原页坐标，用于高亮
  token_count  INT,
  embedding    vector(1024),
  tsv          tsvector             -- 全文检索列
);
CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ON doc_chunks USING gin (tsv);
CREATE INDEX ON doc_chunks (textbook_id, page_no);

-- 作业/试卷提交
CREATE TABLE exercises (
  id           BIGSERIAL PRIMARY KEY,
  user_id      BIGINT REFERENCES users(id),
  source_type  TEXT,                -- homework/exam
  image_keys   TEXT[],              -- MinIO 对象键
  ocr_text     TEXT,                -- VLM 识别原文
  ocr_confirmed BOOLEAN DEFAULT false,
  questions    JSONB,               -- [{no, bbox, stem, student_answer}]
  status       TEXT DEFAULT 'uploaded',
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- 单题作答与讲解
CREATE TABLE attempts (
  id           BIGSERIAL PRIMARY KEY,
  exercise_id  BIGINT REFERENCES exercises(id) ON DELETE CASCADE,
  user_id      BIGINT REFERENCES users(id),
  q_no         INT,
  stem         TEXT,
  student_ans  TEXT,
  correct_ans  TEXT,
  is_correct   BOOLEAN,
  analysis_md  TEXT,                -- 分步解析（含引用）
  citations    JSONB,               -- [{textbook_id, page_no, heading_path}]
  audio_key    TEXT,                -- TTS 音频对象键
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- 知识点
CREATE TABLE knowledge_points (
  id         BIGSERIAL PRIMARY KEY,
  textbook_id BIGINT REFERENCES textbooks(id),
  subject    TEXT,
  name       TEXT NOT NULL,
  page_no    INT
);

-- 错题本
CREATE TABLE mistakes (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT REFERENCES users(id),
  attempt_id  BIGINT REFERENCES attempts(id) ON DELETE CASCADE,
  kp_id       BIGINT REFERENCES knowledge_points(id),
  review_count INT DEFAULT 0,
  mastered    BOOLEAN DEFAULT false,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- 掌握度（学情画像）
CREATE TABLE kp_mastery (
  user_id     BIGINT REFERENCES users(id),
  kp_id       BIGINT REFERENCES knowledge_points(id),
  total       INT DEFAULT 0,
  wrong       INT DEFAULT 0,
  mastery     NUMERIC(4,3) DEFAULT 0,   -- 0~1
  updated_at  TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (user_id, kp_id)
);

-- 异步任务
CREATE TABLE jobs (
  id         TEXT PRIMARY KEY,      -- RQ job id
  kind       TEXT NOT NULL,         -- parse_textbook/ocr_exercise/tts
  ref_id     BIGINT,
  status     TEXT DEFAULT 'queued',
  progress   INT DEFAULT 0,
  error      TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

---

## 6. PDF 解析管线（核心）

200MB 单册、数理化公式密集，**禁止整册盲跑 VLM**。按页三档分流：

| 档 | 触发条件 | 处理 | 相对成本 |
|---|---|---|---|
| L1 文字层直抽 | 页面有文本且字符数正常 | PyMuPDF `get_text("dict")` 保留 bbox 与字号 | 1× |
| L2 公式/图片区域 | 页面含图片块，或文字层出现 Type3/CID 字体（公式常为特殊字体） | 按 bbox 裁切 → `qwen-vl-max` 转 LaTeX，回填原位 | 20~50× |
| L3 整页兜底 | 抽取字符数 < 阈值，或乱码率 > 30% | 300dpi 渲染整页 → `qwen-vl-ocr` | 100× |

**判定逻辑**：先跑 L1，用「可打印字符占比 + 中文字符合理性」给每页打分，低于阈值降级到 L3；L1 通过但含图片块的，对图片块走 L2。

**输出统一格式**：
```markdown
## 2.3 牛顿第二定律
（正文 markdown）
$$F = ma \tag{2-1}$$
<!-- page=57 bbox=[72,310,540,362] -->
```

**成本预估**（单册 500 页，30% 页面含公式）：
- L2：150 页 × 约 2000 image token × 0.02 元/千 token ≈ **6 元**
- Embedding：约 30 万字 → 60 万 token × 0.0005 元/千 token ≈ **0.3 元**
- 首轮解析总成本 **10 元以内**，耗时 20~40 分钟（受 Worker 并发与 API 限流约束）

---

## 7. 切分策略

教材不能按固定 token 切，否则一道例题被切成两半。

1. **优先取 PDF 书签/TOC**（`doc.get_toc()`）建层级
2. 无 TOC 时用字号聚类 + 编号正则（`^第[一二三]章`、`^\d+\.\d+(\.\d+)?`）识别标题
3. 按「节」内聚 ，chunk 512~1024 token，overlap 128
4. **公式块完整性校验**：统计 `$$` 配对，不配对的块向后合并，不允许切断
5. 每个 chunk 写入 `heading_path[]`，这是后续引用溯源与前端高亮的唯一依据

---

## 8. 检索与教学推理

### 混合检索
```
query → text-embedding-v3 向量检索 top30
      → PG 全文检索(zhparser) top30
      → RRF 融合 → top8
```
过滤条件固定前置：`textbook_id IN (该学生年级+学科的全部教材)`。跨年级串味是最常见的 RAG 错误。

### 推理流程
```
作业图片 → qwen-vl-max 整图理解 & 题目切分(bbox)
        → qwen-vl-ocr 精细识别（打印体+手写答案）
        → 【前端确认卡：识别结果回显，用户确认/修正】
        → VLM 输出知识点标签
        → 标签 + 题干 → 混合检索命中教材章节
        → qwen-max 流式生成分步解析
        → 强制引用格式 [教材 P57 §2.3]
```

**为什么必须加确认环节**：手写体识别一旦认错，后面整套讲解全部跑偏，且用户不知道错在哪。确认成本 3 秒，收益是准确性。

### 输出约束
JSON Schema 约束输出结构：`{steps:[{no, title, content_md, citation}], final_answer, kp_tags[]}`。前端按 steps 逐条渲染，支持「讲慢一点/再讲一遍」按步骤重生成。

---

## 9. 语音朗读（TTS）

问题：`x^2` 直接送 TTS 会被念成乱码。

处理链：
```
解析 Markdown → 剥离 LaTeX → 再调一次 qwen-turbo 做「LaTeX→中文口语」转换
              → 拼接纯口语文本 → CosyVoice 合成 mp3 → 存 MinIO
```
口语转换示例：`F = ma` →「F 等于 m 乘以 a」，`\frac{1}{2}mv^2` →「二分之一 m v 平方」。

音频按 `attempt_id` 缓存，同一解析不重复合成。

---

## 10. 飞书接入

### 应用配置
1. 开放平台 → 创建**企业自建应用** → 添加「网页应用」能力
2. 桌面端主页 / 移动端主页：`https://edu.v2txt.cn/`
3. 安全设置 → 重定向 URL：`https://edu.v2txt.cn/api/auth/feishu/callback`
4. 权限申请：
   - `contact:user.base:readonly`（获取姓名）
   - `contact:user.employee_id:readonly`（获取 userid）
   - 如需发消息通知：`im:message:send_as_bot`

### 免登流程
```
H5 前端 tt.requestAuthCode({appId}) → code
  → POST /api/auth/feishu/callback {code}
  → 后端调 /open-apis/authen/v1/oidc/access_token
  → 拿 user_access_token + open_id
  → 建/查 users 表 → 签发自身 JWT（7 天）
```
**首次登录需管理员在后台绑定学生年级**，否则检索不知道该查哪个年级的教材。

### 端口注意
飞书对非标准端口的兼容性不稳定。建议**外层 443 → 内网 52066**。若公司 443 已被占用，用 52066 直接对外，需实测飞书客户端能否正常打开，不行再调整。

---

## 11. API 清单

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/auth/feishu/callback` | 飞书免登，换 JWT |
| GET | `/api/me` | 当前用户信息 |
| POST | `/api/textbooks/upload/init` | 分片上传初始化，返回 upload_id |
| PUT | `/api/textbooks/upload/{uid}/{part}` | 上传分片 |
| POST | `/api/textbooks/upload/{uid}/complete` | 合并分片，入队解析 |
| GET | `/api/textbooks` | 教材列表 |
| GET | `/api/jobs/{id}` | 解析进度（轮询） |
| POST | `/api/exercises/upload` | 作业图片上传（预签名直传 MinIO） |
| POST | `/api/exercises/{id}/ocr` | 触发识别，返回题目切分结果 |
| POST | `/api/exercises/{id}/confirm` | 用户确认识别结果 |
| POST | `/api/exercises/{id}/analyze` | 触发分步解析（SSE 流式） |
| GET | `/api/attempts/{id}` | 取解析结果 |
| POST | `/api/attempts/{id}/tts` | 生成语音 |
| GET | `/api/mistakes` | 错题本列表（学生看自己，家长看孩子） |
| GET | `/api/stats/mastery` | 学情掌握度 |

### 家长端接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/parent/children` | 我绑定的孩子列表（含 grade、教材版本） |
| POST | `/api/parent/children/{student_id}/switch` | 切换当前查看的孩子（写入 session） |
| GET | `/api/parent/children/{id}/exercises` | 孩子的作业提交记录 |
| GET | `/api/parent/children/{id}/attempts/{aid}` | 某次作业的完整解析回看 |
| GET | `/api/parent/children/{id}/mistakes` | 孩子的错题本 |
| GET | `/api/parent/children/{id}/mastery` | 知识点掌握度雷达数据 |
| GET | `/api/parent/children/{id}/weekly-report` | 周报（做题数、正确率、薄弱知识点） |
| POST | `/api/parent/bind` | 绑定孩子（需邀请码，见下） |

**家长端的核心设计取会**：家长是**只读视角 + 可触发生成**，不能代替孩子提交作业。家长端所有查询都必须校验 `parent_student` 关系，防止越权看到别人孩子。绑定走**一次性邀请码**（管理员或孩子端生成，24 小时有效，用后失效），避免家长随意绑定任意学生。

**大文件上传**：教材走**分片直传 MinIO**（预签名 URL），不经 FastAPI 中转。Nginx 设 `client_max_body_size 0`、`proxy_request_buffering off`。前端分片 8MB，支持断点续传（记录已传分片）。

---

## 12. 分阶段实施计划

| 阶段 | 内容 | 交付 | 预估 |
|---|---|---|---|
| P0 环境 | 扩盘、DNS 灰云解析、certbot 证书、飞书应用创建与权限、备案确认 | 可访问的 HTTPS 空壳 | 0.5 天（备案若未办则阻塞） |
| P1 底座 | compose 起 pg/redis/minio，健康检查、限额生效、`edu-net` 隔离验证 | 5 个服务健康 | 0.5 天 |
| P2 接入 | 飞书免登、JWT、H5 骨架页、路由与鉴权中间件 | 能在飞书里打开并识别身份 | 1 天 |
| P3 教材 | 分片上传、解析管线（L1/L2/L3）、切分、向量化、进度查询 | 上传→可检索 | 3~4 天 |
| P4 问答 | 混合检索、RRF、SSE 流式生成、引用溯源与前端高亮 | 教材内问答可用 | 2~3 天 |
| P5 作业 | 拍照/相册上传、VLM 识别、题目切分、确认环节、分步解析 | 作业辅导闭环 | 3~4 天 |
| P6 沉淀 | 错题本、知识点掌握度、LaTeX→口语→TTS | 完整功能 | 2~3 天 |
| P7 上线 | 压测、日志、备份（pg_dump + MinIO 同步）、监控、使用说明 | 交付 | 1~2 天 |

单人节奏约 **3~4 周**。P3 是最大不确定性，建议先用一册教材跑通再批量。

---

## 13. 风险清单

| 风险 | 影响 | 应对 |
|---|---|---|
| ICP 备案未覆盖子域名 | **上线阻塞** | P0 阶段立即核实 |
| 磁盘 29G 不足 | 解析中途写满 | 先扩盘，解析中间文件落 `/opt/edu-tutor/data/tmp` 并定时清理 |
| 内存 7.7G 偏紧 | 大页渲染 OOM | Worker 并发固定 2，逐页处理即释放，不累积整册 |
| 手写体识别准确率 | 讲解跑偏 | 强制确认环节；识别置信度低于阈值标黄提示 |
| 公式特数字体抽取失败 | 内容缺失 | L1 打分不达标自动降级 L3 |
| DashScope 限流 | 解析卡顿 | Worker 内令牌桶限速，指数退避重试 |
| 与现有 v2txt 栈冲突 | 生产事故 | 独立网络/独立卷/独立端口/独立凭据，零共享 |
| 学生数据隐私 | 合规 | 全程内网+阿里云，不出境；照片保留期可配 |
| 家长越权查看他人孩子 | 隐私事故 | 家长端所有接口强制校验 `parent_student` 关系，禁止仅凭 student_id 直查；邀请码一次性且 24h 过期 |

---

## 14. 待确认（阻塞 P0）

1. `v2txt.cn` 备案是否覆盖 `edu.v2txt.cn`（子域名通常随主域名备案，但需确认）
2. 公网 443 是否已被现有应用占用（现有 v2txt 若占 443，则本项目需另想办法）
3. 扩盘方式：虚拟化层加盘，还是现有卷组有空闲 PE
4. 是否需要老师端管理后台（上传教材、查看学生学情）——当前设计只含管理员，老师端可放 P7 之后