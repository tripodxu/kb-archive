# 个人知识归档系统 — 项目规划与实施计划

> **文档状态**：v2.0 / 2026-10-12 / 路线已定（路线 C）+ 架构演进（ADR-005）
> **项目代号**：kb-archive
> **部署目标**：单台 Linux VPS（2 vCPU / 1963 MiB RAM / 49 GiB 磁盘 / 无 GPU）
>
> **导航**：[README.md](README.md) 首页 ｜ [VISION.md](VISION.md) 愿景速览 ｜ [**docs/vision.md**](docs/vision.md) **愿景与需求** ｜ [**docs/execution-plan.md**](docs/execution-plan.md) **施工单** ｜ [**docs/environment.md**](docs/environment.md) **环境事实与铁律** ｜ **本文** 规划 ｜ [docs/roadmap.md](docs/roadmap.md) 三条路线定义 ｜ [docs/download-layer-comparison.md](docs/download-layer-comparison.md) 下载层功能逐条对比 ｜ [docs/feasibility.md](docs/feasibility.md) 可行性与核实 ｜ [docs/mp-tools-docker.md](docs/mp-tools-docker.md) Docker 化调研 ｜ [docs/douyin-erma0.md](docs/douyin-erma0.md) 第四候选 ｜ [deploy/](deploy/README.md) 部署物料 ｜ [analysis/](analysis/README.md) PC 分析流水线

> ⚠️ **v2.0 变更**：用户补充了完整愿景（自下载 / 保存 / 自行整理 / 多媒介 / 本地化）
> 与**近期目标（抖音 + B站 + 公众号的服务器下载功能，必须可扩展）**。
> 据此新增 [docs/vision.md](docs/vision.md) 作为需求源头，新增 **ADR-005 归档中枢（kb-hub）**，
> Phase 2–8 重新编号。ADR-001~004 不变。

---

## 0. 三条路线速览

评估过三条完整路线，最终采用路线 C。定义详见 [docs/roadmap.md](docs/roadmap.md)，此处仅速览：

| | **路线 A**（know-where 四件套） | **路线 B**（wechat-mp-tools 单体） | **路线 C** ✅ 采用 |
|---|---|---|---|
| 下载层 | TikTokDownloader + yutto + wechat-article-exporter | wechat-mp-tools 原样全量 | **wechat-mp-tools 精简版 + wechat-article-exporter 补位** |
| 分析层 | know-where（VPS，带 PostgreSQL） | 无（0 能力） | **PC 本地自建** |
| 容器数 | 4（+PG） | 1 | **2** |
| 平台覆盖 | 3 | 7 名义 / 4 受限 | **7 名义 / 4 受限** |
| 统一 UI | ❌ | ✅ | ✅ |
| 定时增量调度 | ❌ | ✅ | ✅ |
| 浏览器依赖 | ❌ 全无 | 🔴 700–1200 MiB | ❌ 剥离后无 |
| 飞书依赖 | 🔴 **硬依赖** | ❌ 无 | ❌ 无（Phase 6 可选） |
| 阻塞项 | 飞书自建应用未建 | 内存会挤爆同机业务 | 无（PC 端脚本自建） |
| 主要短板 | 平台窄、维护分散 | Linux 下快手/小红书/视频号不可用 | **抖音深度落后 A 约 6 项** |
| 调研文档 | [feasibility.md](docs/feasibility.md) | [mp-tools-docker.md](docs/mp-tools-docker.md) | 二者之和 |

**下载层功能差异逐条量化 → [docs/download-layer-comparison.md](docs/download-layer-comparison.md)**

**另有两个抖音下载器候选**：

- **TikTokDownloader**（16703★，GPL-3.0，2026-09-16 活跃）—— 深度强，见 [docs/download-layer-comparison.md](docs/download-layer-comparison.md) §2.1
- **douyin_erma0**（31 py，GPL-3.0，2026-05-15 停滞）—— 广度强但有三个必改坑，见 [docs/douyin-erma0.md](docs/douyin-erma0.md)

两者现降级为**路线 C+ 可选项**（Phase 8），不阻塞任何里程碑。

---

## 0.1 愿景速览（详情见 [docs/vision.md](docs/vision.md)）

> 自下载、可保存、自行整理、多媒介、**独属于自己**的本地化知识库。

| # | 场景约束（北极星） | 落在哪个 Phase |
|---|---|---|
| S1 | 多端可达（电脑终端 / 手机浏览器 / 电脑浏览器） | Phase 8（多端入口） |
| S2 | **零学习成本：告诉 AI 发链接就行** | Phase 2（`POST /api/ingest`） |
| S3 | 抓取范围可选：单作品 / 作者全部 / 合集 / 收藏夹 / 评论 | Phase 3–5 |
| S4 | 幂等去重，重复提交只入库一次 | Phase 2（资产 schema） |
| S5 | 落盘即归档：自动归位 + 元数据 + 索引 | Phase 2 |
| S6 | 失效可感知，不静默失败 | Phase 2 |

**近期目标（用户明确）**：抖音 + B站 + 公众号的**服务器端下载功能，且必须可扩展**。
**可扩展性的四条硬定义**：新平台 = 一个 adapter 文件；新媒体类型 = 改 schema 的 `files`；
新入口 = 调一次 `/api/ingest`；新存储后端 = 一个 storage 接口。

---

## 1. 项目目标

构建一套自托管的知识归档流水线：

```
把「链接」丢进来 → 抓取正文与媒体 → 落盘 → AI 摘要分类 → 入库
```

具体能力目标：

| # | 能力 | 说明 |
|---|---|---|
| G1 | 公众号文章抓取（单篇） | 正文 + 图片本地化，HTML 100% 还原排版 |
| G2 | 公众号「主页 → 全量历史」 | 需自有可扫码登录的公众号作为凭据 |
| G3 | B站「UP主/合集/收藏夹 → 全部下载」 | 视频 + 弹幕（ASS）+ 字幕（SRT） |
| G4 | 抖音「主页/合集 → 全部下载」 | 无水印 + 增量去重 + 评论抓取 |
| G5 | 「一个链接 → 全部下载」的统一入口 | Web UI 粘贴即可 |
| G6 | 定时增量同步 | 后台调度，自动发现新内容 |
| G7 | AI 摘要与知识提取 | 在 **PC 端** 执行，不占用 VPS 内存 |
| G8 | 归档入库 | Markdown / Obsidian / 飞书多维表格（可选） |
| G9 | **多入口统一**（v2.0 新增） | Web UI / CLI / **任何 AI 直接调 HTTP API** 三种入口共用一个后端 |
| G10 | **资产索引与去重**（v2.0 新增） | 统一 asset schema + SQLite 索引，重复链接自动跳过，全文可检索 |
| G11 | **可扩展平台层**（v2.0 新增） | 新增平台 = 新增一个 adapter 文件 + 一行注册，不改中枢 |
| G12 | **多媒体完整留存**（v2.0 新增） | 视频 / 音频 / 封面 / 字幕 / 弹幕 / 正文 / 图片，全部按同一目录规范落盘 |

**非目标**：视频网站内播放、分发、多用户、权限系统。

---

## 2. 架构决策记录（ADR）

### ADR-001：选定「路线 C」混合方案 ✅ 已定（2026-10-12）

**背景**：存在两条各自可行的技术路线。

- 路线 A：know-where + TikTokDownloader + yutto + wechat-article-exporter（四件套拼装）
- 路线 B：wechat-mp-tools（Flask 单体，功能覆盖最广）

**决策**：采用**路线 C = wechat-mp-tools 做下载层 + PC 端做分析层 + wechat-article-exporter 补公众号全量**。

**理由**：

1. **两者不是竞品，是上下游。** 路线 B 强在下载（6 个平台、自带 UI、自带 RSS 增量调度、自带下载去重），路线 A 的 know-where 强在分析（ASR + LLM + 飞书入库）。而路线 A 里的三个下载器能力上并不比 B 强，只是更成熟省心。
2. **VPS 内存是硬约束。** 1963 MiB total / 975 MiB available。know-where 需要额外拖一个 PostgreSQL 容器；砍掉浏览器依赖后的 wechat-mp-tools 只要 150–250 MiB。
3. **飞书自建应用尚未建立**（见 ADR-002），know-where 的飞书硬依赖会立刻卡住进度。
4. **分析放 PC 不是妥协而是必然**：wechat-mp-tools 完全没有 ASR/摘要能力（`grep 'import torch' backend/*.py` → 0 命中），把分析放 PC 既避开内存限制，又白赚本地 GPU / 本地模型选项。

**代价（明确接受）**：
- 没有飞书机器人入口，「丢链接进去自动处理」需靠 PC 端脚本或手工贴 URL
- PC 端的转写+摘要+入库流水线**目前只是蓝图，尚无成品代码**，属本项目主要工作量
- RSS 订阅数据用 JSON 文件存储（每号上限 200 篇），量大会受限

**被否决的替代方案**：
- `Evil0ctal/Douyin_TikTok_Download_API`（20593★）：v5 主分支砍掉了 B 站支持，且自愈身份池依赖无头浏览器
- `qiye45/wechatDownload`（9731★）：仓库内**无源码**（`find . -name '*.py' | wc -l` = 0），是 Hook 微信 PC 客户端的桌面工具，Linux 不可用
- `x554960766/wechat-mp-tools` 原版全量 Docker 方案：镜像 ~2.5 GB，Chromium 常驻 700–1200 MiB，有 OOM 波及同机业务容器的风险

### ADR-002：飞书入库推迟，用户后续提供 Cookie ✅ 已定

用户 2026-10-12 确认：**飞书自建应用尚未建立，后续会提供 Cookie**。

**决策**：Phase 1–9 不依赖飞书。先以「VPS 下载 + PC 分析 + Markdown/Obsidian 入库」跑通全链路。
飞书作为 Phase 10 的可选增强，接入时再评估引入 know-where 还是直接调飞书 Open API 写多维表格。
**并且「告诉 AI 发链接」不需要飞书** —— 一个 `POST /api/ingest` 端点即可满足（见 ADR-005 与 [docs/vision.md](docs/vision.md) §2）。

### ADR-003：`jev-router` 长期保留 ✅ 已定（2026-10-12）

与本项目无直接关系，但记录以免被误下线：该容器是待接入的备用多源决策网关，占用仅 6.1 MiB，**不下线**，择机接线（加 ingress 或加成 new-api 渠道）。

### ADR-004：下载与分析物理分离

**决策**：VPS 只跑「抓取 + 下载 + 落盘」，**不部署**任何 ASR、LLM、视频转码、字幕擦除组件。

**理由**：本机 1963 MiB 内存、无 GPU。本地 LLM 实测不可行（Qwen2.5-1.5B Q4 峰值 1.4–1.6 GB，叠加容器必 OOM；0.5B 中文摘要质量不可用）。本地 faster-whisper 需装 PyTorch，镜像撑不住。

**例外**：FFmpeg 保留在 VPS 容器内（体积小、纯 CPU、wechat-mp-tools 混流弹幕/音轨需要），但**不运行**字幕擦除（该功能 vendored 了带 torch 的第三方项目，启用需拉数 GB 依赖，直接删除该目录）。

### ADR-005：引入「归档中枢 kb-hub」作为薄门面 ⏳ 待确认（2026-10-12 提出）

**触发**：用户补充愿景与近期目标（见 [docs/vision.md](docs/vision.md) §2–§3），
其中四条约束 mp-tools 作为单体面板**不满足**：

| 愿景约束 | mp-tools 现状 | 缺口 |
|---|---|---|
| S2 告诉 AI 发链接 | Flask 17 个 blueprint 是**给自家前端调的**，无对外 API 文档、无鉴权 | 无法被 AI/脚本调用 |
| S3 范围可选 | 前端能选，但**没有统一请求模型** | 入口碎片化 |
| S5 落盘即归档 | 下载落 `download_history.json` | 无 schema、无索引、无去重语义 |
| 近期「必须可扩展」 | 单体 Flask，平台逻辑与路由耦合 | 加平台要动核心文件 |

**决策（✅ 已定 2026-10-12，用户答复 Q0「要做」）**：**新增一个轻量「归档中枢 kb-hub」容器，作为门面 + 资产索引，不重写抓取逻辑。**
> 具体施工由**执行 agent** 承担 → 逐项任务、验收标准、回滚步骤见 **[docs/execution-plan.md](docs/execution-plan.md)**。
> 环境事实与不可违反的规则见 **[docs/environment.md](docs/environment.md)**。

```
入口（CLI / Web UI / AI HTTP）
        │  POST /api/ingest {url, scope}
        ▼
   kb-hub  ── 链接解析 → adapter 注册表
        │         ├─ bilibili adapter ─┐
        │         ├─ wechat adapter  ──┤ 调用（进程内 import / HTTP）
        │         └─ douyin adapter  ──┘
        ▼
   mp-tools 容器（抓取执行器，沿用已验证代码）
        ▼
   统一资产 schema → 落盘 → SQLite 索引 → dav
```

**理由**：

1. **是收窄，不是推翻**。抓取代码仍来自 mp-tools（ADR-001 不变），hub 只做「入口归一 + 资产语义」。
   先把 mp-tools 跑起来（Phase 1）再套 hub（Phase 2），每一步都可独立验收、可独立回滚。
2. **S2 的最低成本实现就是 HTTP 接口**。任何 AI（包括本会话）都能 `curl` 一个 POST 端点，
   **不需要飞书、不需要 IM 集成**，这直接解掉 ADR-002 的阻塞。
3. **可扩展性落成可验收的工程结构**：adapter 注册表 + 统一 asset schema + storage 接口，
   对应 [docs/vision.md](docs/vision.md) §3 的四条硬定义。
4. **索引与去重是归档系统的地基**，mp-tools 的 `download_history.json` 撑不住长期资产库。

**代价（明确接受）**：

- 多一个常驻进程（FastAPI + SQLite，约 60–100 MiB，需实测）
- 需要维护一层自己的抽象代码（hub 约 1500–2500 行）
- mp-tools 的 Flask 路由与 hub 的 adapter 会有一层映射，长期有维护成本

**被否决的替代方案**：

- **完全重写 hub 内嵌所有抓取代码**：省掉一层映射，但要一次性吃下 174 个 py 的 186★ 项目，
  与风险 R2（作者已声明同类项目在闭源收费）叠加，审计成本不可控
- **直接把 hub 做成 mp-tools 的插件**：需要改上游 Flask 路由，破坏「私有 fork 只做最小补丁」的策略
- **继续用 mp-tools 原生 UI 不做中枢**：放弃 S2/S3/S5 与全部可扩展性，与近期目标冲突

**状态**：✅ **已采纳**（2026-10-12，用户答复 Q0：「要做」）。

若后续要推翻，必须回答「那 G9–G11（多入口统一 / 资产索引去重 / 可扩展平台层）怎么办」——
这三个目标与 hub 是绑定关系，砍掉 hub 就等于砍掉它们。

---

## 3. 系统架构

```
┌─ VPS（下载层）──────────────────────────────────────────────┐
│                                                            │
│  ┌── mp-tools（主容器）─────────────────────────────────┐   │
│  │  Flask :5200，17 个 blueprint                         │   │
│  │  ✅ 公众号文章（纯 HTTP）  ✅ B站（全功能，纯 HTTP）      │   │
│  │  ✅ 抖音主链路（自签 X-Bogus/msToken，纯 HTTP）          │   │
│  │  ✅ RSS 定时增量调度  ✅ 下载历史去重  ✅ Web UI         │   │
│  │  ❌ 已剥离：pywebview / playwright / scrapling /       │   │
│  │            mitmproxy / subtitle_remover(torch)         │   │
│  │  ⚠️ 保留但不用：快手 / 小红书（需 Chromium，见 ADR 风险） │   │
│  │  预期常驻内存 150–250 MiB                              │   │
│  └──────────────────────┬─────────────────────────────────┘   │
│                         ▼                                     │
│  ┌── wae（补位容器）────────────────────────────────────┐   │
│  │  wechat-article-exporter（Nuxt :8317→:8012）           │   │
│  │  专攻「公众号主页 → 全量历史文章」                      │   │
│  │  需自有可扫码登录的公众号提供凭据                       │   │
│  └──────────────────────┬─────────────────────────────────┘   │
│                         ▼                                     │
│              /opt/mp-tools/data/  ──rsync──►  dav（已有）    │
│                                                     │         │
│                                        /dav/kb-archive/raw      │
└─────────────────────────────────────────────────────────┼───────┘
                                                          │ WebDAV
┌─ PC（分析层）────────────────────────────▼──────────────▼───────┐
│  ① 同步 kb-archive/raw → 本地工作区                            │
│  ② faster-whisper：视频音轨 → SRT（本地 GPU/CPU）               │
│  ③ LLM 摘要 + 知识提取 + 自动分类                              │
│  ④ 入库：Markdown / Obsidian（飞书可选，Phase 6）                │
└────────────────────────────────────────────────────────────────┘
```

### 目标架构 v2（ADR-005，Phase 2 起生效）

```
┌─ 入口层（多端）──────────────────────────────────────────────┐
│  ①  Web UI（手机/电脑浏览器，hub 自带）                       │
│  ②  CLI：`kb add <url> --scope author`                       │
│  ③  AI/脚本：curl POST /api/ingest   ← 「告诉 AI 发链接」     │
└───────────────────────────┬───────────────────────────────────┘
                            ▼
┌─ kb-hub（FastAPI + SQLite，薄门面）──────────────────────────┐
│  · 链接解析：正则识别平台 → 路由                              │
│  · 任务队列：ingest 任务表，状态机 pending→running→done/failed │
│  · adapter 注册表：bilibili / wechat / douyin / ...           │
│  · 统一 asset schema + 去重（内容哈希 = platform + platform_id）│
│  · 资产索引与检索 API                                         │
└──────┬─────────────────────────────────┬───────────────────┘
       │ adapter 调用                     │ 落盘 + 索引
       ▼                                 ▼
┌─ 抓取执行器 ────────────────┐   ┌─ 资产库 ─────────────────────┐
│  mp-tools（Flask:5200）      │   │ /opt/kb-archive/media/       │
│  ✅ 公众号 / B站 / 抖音       │   │   <platform>/<author>/       │
│  （浏览器依赖已剥离）         │   │     <asset_id>/             │
│  wae（公众号全量补位）        │   │       meta.json video.mp4   │
└──────────────────────────────┘   │       cover.jpg audio.m4a   │
                                    │       subtitle.srt danmaku.ass
                                    └───────────┬─────────────────┘
                                                │ rsync
                                                ▼
                                        dav（已有 WebDAV 容器）
                                                │ WebDAV
┌─ PC（分析层）────────────────────────────────▼───────────────┐
│  ① 同步 → ② faster-whisper 转写 → ③ LLM 摘要分类 → ④ Obsidian │
└────────────────────────────────────────────────────────────────┘
```

### 统一资产 schema（v2.0，Phase 2 产出）

```json
{
  "id": "sha1(platform + ':' + platform_id)",
  "platform": "douyin | bilibili | wechat | xiaohongshu | kuaishou | ...",
  "platform_id": "7xxxxxxxxxxxxxxxxx",
  "type": "video | article | audio | image",
  "author": { "id": "...", "name": "...", "url": "..." },
  "title": "...",
  "published_at": "2026-10-12T08:00:00+08:00",
  "source_url": "https://...",
  "scope": "single | author | collection | favorites | comments",
  "files": {
    "video": "video.mp4", "audio": "audio.m4a", "cover": "cover.jpg",
    "subtitle": "subtitle.srt", "danmaku": "danmaku.ass",
    "article_html": "article.html", "article_md": "article.md", "images": "images/"
  },
  "metrics": { "play": null, "like": null, "comment": null, "danmaku": null },
  "extra": { "comments_count": 0, "fav_count": 0, "share_count": 0 },
  "status": "pending | running | done | failed",
  "error": null,
  "archived_at": "2026-10-12T08:03:11+08:00",
  "analysis": { "asr": null, "summary": null, "tags": [], "analyzed_at": null }
}
```

设计要点：

- **`id` 用内容哈希而非链接** → 天然幂等去重（S4），同一作品从不同入口提交只入库一次
- **`files` 是字典而非固定字段** → 新增媒体类型只需加 key，不改 schema（G12）
- **`analysis` 字段预留在 VPS 侧** → 分析在 PC 跑（ADR-004），但索引结构一致，便于回写

### Hub 对外 API 契约（Phase 2 产出）

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/api/ingest` | `{url, scope, platform?}` → 返回 `{task_id, asset_id?}`。**这是「告诉 AI 发链接」的落点** |
| GET | `/api/tasks/{id}` | 任务状态、进度、错误 |
| GET | `/api/assets` | 检索：`?platform=&author=&type=&q=&since=&tag=&limit=` |
| GET | `/api/assets/{id}` | 单个资产详情 |
| GET | `/api/assets/{id}/file/{name}` | 取文件 |
| POST | `/api/assets/{id}/reanalyze` | 标记待分析，触发 PC 侧拉取 |
| GET | `/api/health` | 健康检查 |

鉴权：MVP 用一个静态 `X-KB-Token` 头即可（单用户，不做账号体系，见 Q9）。

### 网络与安全约束

- mp-tools 容器用 `network_mode: bridge`（**不是**自定义 bridge 网络）——若需经宿主 WARP 代理池出网，这是必需条件，自定义网络会被防火墙静默丢包
- 端口只发布到 `127.0.0.1:8011` / `127.0.0.1:8012`，**不绑 0.0.0.0**；对外访问走 Cloudflare Tunnel
- 反爬出口可用宿主已有的 WARP 代理池（16 个 mixed 口），按需配置

---

## 4. 技术选型明细

### 4.1 下载层主组件：wechat-mp-tools

| 项 | 值 |
|---|---|
| 上游 | https://github.com/x554960766/wechat-mp-tools |
| Star | 186 |
| License | MIT (c) 2026 xuyi |
| 形态 | Flask Web 应用（`app.py`）+ PyInstaller 桌面壳 |
| 平台覆盖 | 公众号文章 / B站 / 抖音 / 小红书 / 快手 / 视频号 / 微信读书 |
| **⚠️ 可持续性** | README 自述「很多同类开源项目已陆续闭源转向收费，本项目坚持开源」。**必须 fork 私有化后使用**，不要把生产链路押在上游 |

**浏览器依赖（逐文件实测结论）**：

| 模块 | 必需浏览器 | 源码证据 |
|---|---|---|
| B站 | ❌ 不需要 | `backend/bilibili.py` 中 playwright/browser/chromium 零命中；弹幕转 ASS、字幕转 SRT 均为纯 HTTP |
| 抖音 | ❌ 主路径不需要 | `backend/douyin.py:3` 注释「使用直接 HTTP API 调用代替 Playwright 渲染」，配 `douyin_sign.py` + `douyin_mstoken.py` 自签 |
| 公众号文章 | ❌ 不需要 | `backend/articles.py` 浏览器零命中 |
| 快手 | ✅ 必需 | `backend/kuaishou.py:401` 注释「突破前端 JS 签名校验 (result=50)」，:553/:567 无条件 fallback |
| 小红书 | ✅ 必需（有降级） | `backend/xiaohongshu.py:800` 失败后 :808 降级为「主页首屏解析」 |
| 视频号 | ✅ 必需 + 必须有头 | `backend/channels.py:293` `headless=False`，依赖 mitmproxy 抓包 |
| 微信读书 | ⚠️ 保活 | `backend/weread_browser.py:120` headless Chromium 换 Cookie |

**剥离清单**（Dockerfile 中用 `sed` 删除依赖行，并删目录）：

```
playwright>=1.40
scrapling>=0.2
pywebview>=5.0
mitmproxy>=12.0
backend/subtitle_remover/      ← 含 torch 依赖的 vendored 第三方项目
```

平台专属依赖（`pythonnet`/`clr_loader` 仅 win32、`pyobjc-*` 仅 darwin）由 pip marker 自动跳过，无需处理。

### 4.2 公众号补位：wechat-article-exporter

| 项 | 值 |
|---|---|
| 上游 | https://github.com/wechat-article/wechat-article-exporter |
| Star | **13054** |
| License | MIT |
| 形态 | Nuxt 3 全栈，`Dockerfile`：node:22-alpine 构建 → node:22-slim 运行 |
| **批量原理** | README「原理」章节：借助**公众号后台编辑器**「搜索其他公众号文章」的能力来枚举目标号全部历史文章 |
| **凭据要求** | 必须有一个**能扫码登录的自有公众号**。源码印证 `server/api/web/login/{bizlogin.post.ts, getqrcode.get.ts, scan.get.ts}` |
| ⚠️ 官方在线站 | 域名 **2026-10-30 到期**，必须自建 Docker 部署 |
| 声明 | README 承诺「不会利用您扫码登录的公众号进行任何形式的私有爬虫，也不存在账号池」 |

### 4.3 分析层（PC 端，自建）

| 环节 | 选型 | 说明 |
|---|---|---|
| 视频转写 | faster-whisper（本地） | PC 有 GPU 则用 GPU；无 GPU 用 `base`/`small` 模型 |
| 摘要提取 | 外部 LLM API（DeepSeek 等） | 或本地模型，视 PC 配置 |
| 入库 | Markdown + Obsidian vault | 飞书多维表格留作 Phase 6 |

---

## 5. 目录结构规划

```
kb-archive/                       # 本仓库（GitHub）
├─ README.md                      # 项目首页
├─ PLAN.md                        # 本文件
├─ VISION.md                      # 愿景速览（软链/摘录自 docs/vision.md）
├─ LICENSE                        # MIT
├─ .gitignore
├─ docs/
│  ├─ vision.md                   # 【需求源头】愿景、场景约束、近期/远期目标
│  ├─ roadmap.md                  # 三条路线对比与决策矩阵
│  ├─ download-layer-comparison.md# 下载层功能逐条对比 + 四个下载器矩阵
│  ├─ douyin-erma0.md             # 第四候选 douyin_erma0 调研与四个坑
│  ├─ feasibility.md              # 可行性分析（资源约束、选型核实）
│  ├─ mp-tools-docker.md          # Docker 化调研（浏览器依赖逐文件实测）
│  ├─ ai-usage.md                 # Phase 8 产出：AI/脚本调用说明
│  └─ operations.md               # 运维手册（Phase 6 产出）
├─ hub/                           # ★ Phase 2 产出：归档中枢（ADR-005）
│  ├─ server.py                   # FastAPI 入口
│  ├─ cli.py                      # kb add / ls / show
│  ├─ links.py                    # 链接解析 → 平台
│  ├─ api/                        # ingest / tasks / assets 路由
│  ├─ adapters/                   # ★ 插件层：新增平台 = 新增一个文件
│  │  ├─ base.py                  #   Adapter 协议 + 注册表
│  │  ├─ bilibili.py              #   Phase 3
│  │  ├─ wechat.py                #   Phase 4
│  │  ├─ douyin.py                #   Phase 5
│  │  ├─ tiktok_downloader.py     #   Phase 11.1（可选）
│  │  ├─ douyin_erma0.py          #   Phase 11.2（可选）
│  │  ├─ xiaohongshu.py           #   Phase 7.3（可选）
│  │  └─ kuaishou.py              #   Phase 7.3（可选）
│  ├─ assets/schema.json          # 统一资产 schema（冻结）
│  ├─ storage/                    # 存储抽象：filesystem / sqlite / objectstore
│  ├─ workers/poller.py           # 极简任务轮询（无 Redis）
│  └─ web/                        # Phase 8 产出：极简 Web UI
├─ deploy/                        # 服务器侧部署物料
│  ├─ hub/
│  │  ├─ Dockerfile               # Phase 2 产出
│  │  └─ docker-compose.yml       # Phase 2 产出
│  ├─ mp-tools/
│  │  ├─ Dockerfile               # Phase 1 产出
│  │  ├─ docker-compose.yml       # Phase 1 产出
│  │  └─ patches/                 # 对上游的补丁（requirements 剥离等）
│  ├─ wae/
│  │  └─ docker-compose.yml       # Phase 4 产出
│  ├─ dav-sync.sh                 # Phase 6 产出（rsync 到 dav）
│  └─ crontab                     # Phase 6 产出
├─ analysis/                      # PC 端分析流水线
│  ├─ sync.py                     # Phase 9 产出：WebDAV 拉取
│  ├─ transcribe.py               # Phase 9 产出：faster-whisper 批处理
│  ├─ summarize.py                # Phase 9 产出：LLM 摘要
│  ├─ config.example.yaml         # 配置模板
│  └─ requirements.txt
└─ scripts/                       # 辅助脚本
```

---

## 6. 任务分解（WBS）

每个 Phase 都有明确的**产出物**、**验收标准**、**回滚方式**。

### Phase 0 — 项目脚手架 ✅ 进行中

| 项 | 状态 |
|---|---|
| 创建独立项目目录 `kb-archive/` | ✅ |
| 复制并脱敏既有调研文档 | ✅ |
| 编写 README.md 与 PLAN.md | ✅ |
| 决策记录（ADR）落档 | ✅ |

**验收**：仓库可公开，无任何密钥/内网 IP 残留。

---

### Phase 1 — mp-tools 容器跑通纯 HTTP 路径 ★最高优先级

**为什么先做**：不需要 Cookie、不需要飞书、不需要 Chromium，是整条路上风险最低的一步，产出的内存实测值决定后续所有方案。

| # | 任务 | 产出 |
|---|---|---|
| 1.1 | fork 上游仓库到私有库，固定 commit | 私有 fork |
| 1.2 | 编写 Dockerfile：python:3.12-slim + 剥离 4 个浏览器相关依赖 + ffmpeg | `deploy/mp-tools/Dockerfile` |
| 1.3 | 编写 compose：`network_mode: bridge`，发布 `127.0.0.1:8011:5200`，加内存上限 512 MiB，卷挂 `/opt/mp-tools/data` | `deploy/mp-tools/docker-compose.yml` |
| 1.4 | 部署前记录基线：`docker ps`、`free -h`、`ss -tulnp` 快照 | `docs/operations.md` 基线段 |
| 1.5 | 启动容器 | — |
| 1.6 | 验证 `GET /api/settings` 返回 200 | — |
| 1.7 | 验证 B站：单个 BV 号能取到标题/封面/时长 | — |
| 1.8 | 验证抖音：单个作品链接能取到无水印直链 | — |
| 1.9 | 验证公众号：单篇文章能抓正文并导出 Markdown | — |
| 1.10 | **实测 `docker stats` 常驻内存** | 决定 Phase 3 是否上浏览器 |
| 1.11 | 连续运行 30 分钟，观察 RSS 调度器与日志 | — |

**验收标准**
- 三个平台各成功一次真实抓取
- 常驻内存 ≤ 300 MiB
- 连续 30 分钟无 OOM、无崩溃重启
- 服务器现有 10 个业务容器零影响

**回滚**：`docker compose down && rm -rf /opt/mp-tools`，无外部依赖

---

### Phase 2 — 归档中枢 kb-hub ★★ 架构核心

**为什么是它**：愿景的 S2（AI 发链接）、S4（去重）、S5（归档）、S6（失效可感知）
以及「必须可扩展」，全部落在这一个 Phase。详见 ADR-005。

| # | 任务 | 产出 |
|---|---|---|
| 2.1 | 建仓库骨架：`hub/` 包结构（api / adapters / assets / storage / workers） | `hub/` |
| 2.2 | 定义并冻结 asset schema（§3），写成 JSON Schema | `hub/assets/schema.json` |
| 2.3 | 链接解析器：正则识别 B站 / 抖音 / 公众号 / 小红书 / 快手 / 微博 / YouTube | `hub/links.py` |
| 2.4 | SQLite 建表：`assets` / `tasks` / `authors` / `tags` + 索引 | `hub/storage/sqlite.py` |
| 2.5 | 实现 §3 的 7 个 API 端点 | `hub/api/` |
| 2.6 | adapter 协议 + 注册表 + 一个 `noop` 示例 adapter | `hub/adapters/base.py` |
| 2.7 | 落盘规范：`<platform>/<author>/<asset_id>/` 目录 + `meta.json` 原子写 | `hub/storage/filesystem.py` |
| 2.8 | 最小任务 worker（轮询 SQLite，无 Redis/Celery） | `hub/workers/poller.py` |
| 2.9 | Dockerfile + compose：`network_mode: bridge`，发布 `127.0.0.1:8018:8000`，内存上限 256 MiB | `deploy/hub/` |
| 2.10 | **`POST /api/ingest` 端到端可用**（先用 noop adapter 也返回任务记录） | — |
| 2.11 | 加入 `o2a-watchdog.py` 的 `CONTAINERS` 清单 + `backup-all.sh` | — |
| 2.12 | `kb verify`：比对索引与磁盘实际文件，双向报告差异（R15 检测手段） | `hub/cli.py` |

**验收标准**
- `curl -XPOST /api/ingest -d '{"url":"https://www.bilibili.com/video/BV1xx"}'` 返回 200 + task_id
- 重复提交同一链接返回同一 asset_id（S4 幂等）
- 平台不认识的链接返回 400 + 可读错误，**不静默失败**（S6）
- 常驻内存 ≤ 150 MiB
- 新增一个空 adapter 只需 1 个文件 + 1 行注册（可扩展性验收）
- `kb verify` 能查出「索引有但磁盘没有」与「磁盘有但索引没有」

> 逐项施工步骤、验收命令、回滚演练 → **[docs/execution-plan.md](docs/execution-plan.md)**

**回滚**：`docker compose down`，mp-tools 容器不受影响，Phase 1 成果全部保留

---

### Phase 3 — B站 adapter 接入 hub（近期目标平台 1/3）

| # | 任务 | 说明 |
|---|---|---|
| 3.1 | `adapters/bilibili.py`：包 mp-tools `backend/bilibili.py` 的纯 HTTP 函数 | 复用已验证代码 |
| 3.2 | `scope=single`：BV/av 号 → 视频 + 封面 + 音频 + 字幕 SRT + 弹幕 ASS | 无需 Cookie |
| 3.3 | `scope=author`：UP 主全部投稿（分页 + 增量去重） | 无需 Cookie |
| 3.4 | `scope=collection`：合集全部 | 无需 Cookie |
| 3.5 | `scope=favorites`：**收藏夹** | ⚠️ 需 SESSDATA，无则降级跳过并明确报错 |
| 3.6 | 落盘 + 写 `meta.json` + 入索引 | |

**验收**：给一个 BV 号和一个 UP 主 UID，`scope` 分别取 single / author，
产出视频文件 + ASS 弹幕 + SRT 字幕，`meta.json` 字段齐全，重复执行不产生重复资产

---

### Phase 4 — 公众号 adapter 接入 hub（近期目标平台 2/3）

| # | 任务 | 说明 |
|---|---|---|
| 4.1 | `adapters/wechat.py`：包 mp-tools `backend/articles.py` | 纯 HTTP |
| 4.2 | `scope=single`：正文抽取（`#js_content`）+ 图片本地化 + 转 Markdown | 无需 Cookie |
| 4.3 | `scope=author`：**公众号全量历史** —— 走 wechat-article-exporter 的扫码登录后台搜索能力 | ⚠️ 需自有可扫码公众号 |
| 4.4 | wae 容器 compose，发布 `127.0.0.1:8012` | |

**验收**：单篇文章归档为 Markdown + 本地图片，全链路无外链图片残留；
若配置了可扫码公众号，「输入公众号名 → 全量历史」跑通

**前置**：**用户侧**需确认有一个可扫码登录的自有公众号

---

### Phase 5 — 抖音 adapter 接入 hub（近期目标平台 3/3）

| # | 任务 | 说明 |
|---|---|---|
| 5.1 | `adapters/douyin.py`：包 mp-tools `backend/douyin.py` + `douyin_sign.py` + `douyin_mstoken.py` | 主链路纯 HTTP |
| 5.2 | `scope=single`：无水印直链 + 封面 | 无需 Cookie（公开作品） |
| 5.3 | `scope=author`：主页全部作品 | 私密号需 Cookie |
| 5.4 | `scope=collection`：合集 | |
| 5.5 | 增量去重：按 `platform_id` 比对索引 | S4 |
| 5.6 | **可选增强 A**：接 TikTokDownloader 补评论采集 / 点赞收藏统计 / 收藏夹 / 多账号批量 | 见 Phase 11.1 |
| 5.7 | **可选增强 B**：接 douyin_erma0 补话题挑战 / 音乐原声 / 关键词搜索 / 关注粉丝 | 见 Phase 11.2，须先修三坑 |

**验收**：公开作品链接 + 作者主页链接均能全量下载，去重生效；
若配置 Cookie，私密作品可下载

---

### Phase 6 — 接 dav 同步

| # | 任务 | 产出 |
|---|---|---|
| 6.1 | 在已有 dav 容器中创建 `/dav/kb-archive/raw` 目录 | — |
| 6.2 | 编写 rsync 同步脚本（增量 + 断点续传 + 日志） | `deploy/dav-sync.sh` |
| 6.3 | 加入 crontab（每 10 分钟） | `deploy/crontab` |
| 6.4 | PC 端 WebDAV 客户端验证能拉到文件 | — |
| 6.5 | **决策 Q7**：「自己的云端」若指对象存储，实现一个 COS/OSS storage 适配器 | `hub/storage/objectstore.py` |

**验收**：VPS 下载的文件在 PC 上 10 分钟内可见

**回滚**：删除 crontab 行，dav 目录保留无害

---

### Phase 7 — 浏览器能力（**待 Phase 1.10 数据决定是否执行，可选**）

| # | 任务 | 说明 |
|---|---|---|
| 7.1 | 单次拉起 headless Chromium，实测内存增量 | 决定性数据点 |
| 7.2 | 增量 < 400 MiB → 接受，按需拉起；> 500 MiB → 放独立 sidecar 容器或放弃 | 决策门 |
| 7.3 | 补小红书 / 快手 adapter（`adapters/xiaohongshu.py`、`adapters/kuaishou.py`） | 条件执行 |

**关于登录**：抖音（`douyin_login.py:232`）、小红书（`xiaohongshu_login.py:46`）、快手（`kuaishou_auth.py:160`）**全部是 `headless=False`**，需要可见窗口扫码。**首选路径：在 PC 上正常登录一次，导出 Cookie 贴进 VPS 配置**——这正是用户提供 Cookie 的用途（ADR-002），可完全绕开 Xvfb + noVNC 那一套。若无 Cookie，再考虑 Xvfb。

---

### Phase 8 — 入口层（多端可达，北极星场景 S1/S2）

| # | 任务 | 产出 |
|---|---|---|
| 8.1 | hub 自带极简 Web UI：输入框 + 平台/scope 选择 + 任务列表 | `hub/web/` |
| 8.2 | CLI：`kb add <url> --scope author`、`kb ls`、`kb show <id>` | `hub/cli.py` |
| 8.3 | **AI 调用说明文档 + 可直接复制的 curl 示例** | `docs/ai-usage.md` |
| 8.4 | 移动端适配（响应式，竖屏可用） | |
| 8.5 | 可选：Cloudflare Tunnel 暴露（自建子域名），端口仍只绑 `127.0.0.1` | 需用户侧建子域名 |

**验收**：手机浏览器、电脑终端、AI curl 三种方式提交同一链接，
三者都返回同一个 asset_id（幂等），且都能查到下载进度

**注意**：Phase 8 之前 `/api/ingest` 就已可用（S2 靠 curl 即可满足），
本 Phase 只是把 UI 和 CLI 补上，**不是 S2 的前置条件**

---

### Phase 9 — PC 端分析流水线（**本项目最大的实际工作量**）

| # | 任务 | 产出 |
|---|---|---|
| 9.1 | 同步脚本：WebDAV 拉取到本地工作区（或直接读 hub 的 `/api/assets`） | `analysis/sync.py` |
| 9.2 | 视频音轨提取 + faster-whisper 转写 → SRT | `analysis/transcribe.py` |
| 9.3 | 文档/字幕归一化为纯文本 | `analysis/normalize.py` |
| 9.4 | LLM 摘要 + 关键词 + 自动分类 | `analysis/summarize.py` |
| 9.5 | 输出 Markdown 到 Obsidian vault，按日期/平台分目录 | `analysis/inbox.py` |
| 9.6 | 全链路串跑 + 失败重试与断点续传 | `analysis/pipeline.py` |
| 9.7 | 摘要回写 hub 的 `analysis` 字段（`POST /api/assets/{id}/reanalyze`） | |
| 9.8 | 编写使用说明 | `analysis/README.md` |

**验收**：往系统里丢 3 个链接（B站/抖音/公众号各一），24 小时内 PC 上出现带摘要的归档 Markdown

---

### Phase 10 — 飞书入库（可选，ADR-002 后续）

| # | 任务 | 决策点 |
|---|---|---|
| 10.1 | 用户建立飞书自建应用，提供 App ID/Secret | **用户侧** |
| 10.2 | 评估：直接调飞书 Open API 写多维表格 vs 引入 know-where | 二选一 |
| 10.3 | 若走 know-where，需额外 PostgreSQL 容器 | 需重估内存预算 |

---

### Phase 11 — 路线 C+ 可选增强（**默认不做，按需触发**）

抖音深度落后路线 A 约 6 项能力，详见 [docs/download-layer-comparison.md](docs/download-layer-comparison.md) §2.1。**仅在 Phase 3–5 跑通、确有需求时执行。**

| # | 任务 | 补什么 | 成本 | 触发条件 |
|---|---|---|---|---|
| 11.1 | 追加 TikTokDownloader（`joeanamier/tiktok-downloader`） | 评论采集、点赞收藏统计、剪贴板监听、多账号批量、Web API、收藏夹 | 官方镜像，1 容器，**优先做成 hub 的一个 adapter**（它有 FastAPI :5555） | 抖音是主力平台 |
| 11.2 | 追加 douyin_erma0（**先修三坑**：aria2 缺失 / pywebview / 端口） | 话题挑战、音乐原声、关键词搜索、关注粉丝 | 1 容器 + 400 MB 镜像 + Cookie 维护 | 上述四项确有需求 |
| 11.3 | ~~追加 yutto~~ | — | — | ❌ **不建议**：B 站能力已被 mp-tools 打平，FFmpeg 硬依赖 |

> ⚠️ **顺序约束**：11.1 优先于 11.2（补深度 > 补广度）。任何补充都必须在 Phase 3–5 跑通、
> 且内存实测有余量之后再做 —— 1.9 GiB 是第一约束。
> **ADR-005 之后，新增下载器不再意味着"多一个容器 + 人工点两处"**，而是"多一个 adapter 文件"，
> 入口、索引、去重、归档全部复用。

---

## 7. 资源预算

### VPS 内存（新增部分）

| 容器 | 预期常驻 | 上限 | 说明 |
|---|---|---|---|
| mp-tools | 150–250 MiB | 512 MiB | 剥离浏览器后实测值待确认 |
| **kb-hub（ADR-005）** | **60–100 MiB** | 256 MiB | FastAPI + SQLite，纯 Python 进程；worker 轮询无常驻开销 |
| wae | 待实测 | 512 MiB | Nuxt，Node 运行时 |
| PostgreSQL（仅 Phase 10 若选 know-where） | 80–150 MiB | 256 MiB | **可避免** |
| **合计（不含 PG）** | **≈ 260–450 MiB** | 1280 MiB | |

现有 10 个容器已占 634 MiB / 975 MiB available。**预算仍通过但余量在收窄，必须逐容器压测**。
缓解手段（按需启用）：
- 扩大 swap（当前 2047 MiB，已用 197 MiB）
- 给 `warpgate`（342 MiB）之外的容器加内存上限
- 定期清理 Docker 镜像与构建缓存（约 401 MB + 1.4 GB 可回收）
- **wae 可选**：如果 Q2 长期不提供可扫码公众号，wae 容器就不必起，直接省下这一份

### 磁盘

| 项 | 预估 |
|---|---|
| 当前可用 | 34 GiB |
| mp-tools 镜像 | ~450 MB |
| wae 镜像 | ~400 MB |
| 归档产物（文章 Markdown） | 约 5–20 MB/公众号/年 |
| **视频文件（最大变量）** | 50–500 MB/条 |

**结论**：视频**不能**长期全留。建议策略：转写完只留字幕 + 缩略图，视频保留 N 天后清理，或直接转完即删（ADR-004 下转写发生在 PC，VPS 上视频只是中转）。

---

## 8. 风险登记册

| # | 风险 | 等级 | 缓解 |
|---|---|---|---|
| R1 | **全部结论来自静态读码，未实机运行** | 🔴 | Phase 1.10 实测内存是决策门；发现不符立即回退 |
| R2 | **wechat-mp-tools 仅 186★，作者声明同类项目正在闭源收费** | 🔴 | **Phase 1.1 先 fork 私有化**，固定 commit；不上游直连 |
| R3 | **公众号「主页全量」在 Linux 上不可用**（依赖注入微信 PC 客户端） | 🔴 | 已在路线 C 中用 wechat-article-exporter 单独补位（Phase 4） |
| R4 | **抖音/小红书/快手登录需可见窗口**（三处 `headless=False`） | 🟠 | 用户提供 Cookie（ADR-002），绕开 Xvfb |
| R5 | VPS 内存紧张，OOM 波及同机 10 个业务容器 | 🟠 | 容器内存上限 + Phase 1.10 实测 + 必要时扩 swap |
| R6 | 磁盘被视频吃满 | 🟠 | 转码/转写完即删策略；定期清理 |
| R7 | 平台反爬风控升级 | 🟠 | 复用宿主已有 WARP 代理池；关注上游 issue |
| R8 | RSS 数据用 JSON 存储，每号上限 200 篇 | 🟡 | 量大后迁移 SQLite（`rss_articles.json` 结构简单） |
| R9 | 上游 API 变更导致功能失效 | 🟡 | 固定 commit；定期同步上游；关键逻辑本地补丁化 |
| R10 | 微信生态账号风险 | 🟠 | 严格限定为**自有公众号的自用抓取**，不做任何账号池/对外服务（README 已明确声明） |
| R11 | B 站 API 参考项目 `bilibili-API-collect` 因律师函已停更 | 🟢 | yutto 自签 WBI，不依赖该仓库；wechat-mp-tools 亦自行实现 |
| R12 | PC 端分析流水线无成品代码，工作量被低估 | 🟠 | Phase 9 独立排期；先做最小可用（转写 + 摘要）再扩展 |
| **R13** | **hub 是自己写的代码，没有社区验证**，抽象设计错了会拖累所有 adapter | 🟠 | ADR-005 明确「薄门面、不重写抓取」；先只实现 ingest + schema + noop adapter，adapter 抽象等第二个平台接入时再定型 |
| **R14** | **资产库会无限增长**（视频 50–500 MB/条，磁盘仅余 34 GiB） | 🔴 | Phase 6 前必须定 Q7/Q8；实现 retention 策略（转写完即删 / 保留 N 天）+ 磁盘水位告警 |
| **R15** | 索引（SQLite）与磁盘文件不一致（删了文件索引还在，或反之） | 🟠 | 落盘用「先写临时目录 → 原子 rename → 再写索引」顺序；提供 `kb verify` 巡检命令 |
| **R16** | 反爬导致的静默失败（下载到 0 字节或错误页当视频存下） | 🟠 | 落盘前校验文件头/大小；任务失败必须写 `error` 字段并在 `/api/tasks` 暴露（S6） |
| **R17** | 抖音/公众号 Cookie 过期后功能静默失效 | 🟠 | adapter 统一暴露 `health_check()`，Phase 8 入口层显示各 adapter 状态 |

---

## 9. 里程碑

| 里程碑 | 包含 Phase | 完成标志 | 依赖用户输入 |
|---|---|---|---|
| **M1 冷启动** | 0, 1 | mp-tools 容器跑通，B站/抖音/公众号单链接抓取成功，内存实测 ≤300 MiB | 无 |
| **M2 中枢** | 2 | `POST /api/ingest` 可用，资产 schema 冻结，重复链接幂等 | 无 |
| **M3 近期目标达成** | 3, 4, 5 | **抖音 + B站 + 公众号三平台，单作品/作者全部 均可下载并入库** | 公众号凭据（仅全量）、Cookie（仅私密） |
| **M4 端到端** | 6 | VPS 下载的文件自动出现在 PC 上（云端同步） | Q7 云端选型 |
| **M5 多端入口** | 7, 8 | 手机 / 终端 / AI 三种入口都能提交与查询 | 无 |
| **M6 闭环** | 9 | 丢链接 → 自动出现在带摘要的 Obsidian 归档中 | Q5 vault 路径 |
| **M7 增强** | 10, 11 | 飞书入库 / 抖音深度补齐 | 飞书凭据、Q6 |

**M1 与 M2 完全不依赖用户提供的任何凭据，是即刻可开工的部分。**
**M3 只在「公众号全量历史」和「私密作品」两处需要凭据；三个平台的单作品 + 作者全部（公开部分）
不依赖任何凭据。**

---

## 10. 待确认事项

| # | 事项 | 阻塞什么 | 状态 |
|---|---|---|---|
| **Q0** | **是否接受 ADR-005 的 kb-hub 薄门面方案** | Phase 2 及之后全部 | ✅ **已定：要做**（2026-10-12）→ [docs/execution-plan.md](docs/execution-plan.md) |
| Q1 | 抖音 / 小红书 / 快手的 Cookie | Phase 7.3 浏览器能力、Phase 11 | 用户表示后续提供 |
| Q2 | 有无可扫码登录的公众号 | Phase 4.3 公众号全量 | 未确认 |
| Q3 | 飞书自建应用 App ID/Secret | Phase 10 | 未建立（ADR-002 已推迟） |
| Q4 | 视频是否长期保留（→ 风险 R14） | Phase 6.5 retention 策略 | 未确认 |
| Q5 | 分析结果入库格式（Obsidian vault 路径？） | Phase 9.5 | 未确认 |
| Q6 | 抖音是否为深度使用（决定要不要上 Phase 11.1 TikTokDownloader） | Phase 11 | 未确认 |
| **Q7** | **「自己的云端」= VPS 本地盘 / 对象存储（COS、OSS）/ NAS** | Phase 6.5 storage 实现 | 未确认，详见 [docs/vision.md](docs/vision.md) §6 |
| **Q8** | **视频长期留还是转写完即删** | 磁盘规划 | 未确认（同 Q4，vision 侧细化版） |
| **Q9** | **是否需要多用户 / 权限体系** | hub 鉴权设计 | 未确认，默认单用户 + 静态 token |
| **Q10** | **评论本身要不要入库**（评论往往是最有价值的内容） | `scope=comments` 的存储设计 | 未确认 |
| **Q11** | **是否需要浏览器插件 / 右键一键发送** | Phase 8 之后的入口扩展 | 未确认，不进近期 |

> Q7–Q11 的完整表述与默认假设见 [docs/vision.md](docs/vision.md) §6。

---

## 附录 A：核实方式

本项目所有技术结论**均来自亲自 clone 源码逐行阅读或直接查询 GitHub API**，未采信任何二手描述。关键结论的复核命令记录在各调研文档正文中。

已核实并**推翻**的三条常见说法：

1. 「f2 支持 B 站」—— ❌ `f2/apps/` 无 bilibili 模块
2. 「know-where 靠 RSSHub 抓公众号」—— ❌ 它有自带的 `adapters/wechat.py`，纯 httpx + BeautifulSoup
3. 「know-where 单容器 SQLite」—— ❌ 依赖 `psycopg[binary]`，PostgreSQL 是硬依赖

**我自己写下又推翻的一条结论**：

4. 「yutto 连 ffmpeg 都不需要」—— ❌ `README.md:99` 明写需配置 FFmpeg；`src/yutto/utils/ffmpeg.py:18` + `__main__.py:23` 无条件 import + `validator.py:118` 用它校验编码。**FFmpeg 是硬依赖**，勘误已同步进 [docs/feasibility.md](docs/feasibility.md) §5 与 [docs/download-layer-comparison.md](docs/download-layer-comparison.md) §2.2

已核实的仓库：`douyin_erma0`、`videodl`、`know-where`、`TikTokDownloader`、`f2`、`BBDown`、`yutto`、`wechatDownload`、`wechat-mp-tools`、`wechat-article-exporter`。

## 附录 B：术语

| 词 | 含义 |
|---|---|
| 纯 HTTP 路径 | 不依赖浏览器、直接调平台 API 的抓取实现 |
| 主页全量 | 「给一个博主/公众号主页 URL，自动下载其全部历史内容」 |
| 增量去重 | 记录已下载条目 ID，只拉新增的 |
| 纯文本层 | 剥离 HTML 排版后用于喂给 LLM 的正文 |
| 路线 A / B / C | 见 §0 与 [docs/roadmap.md](docs/roadmap.md) |
| 路线 C+ | 路线 C + Phase 11 的可选补充组件 |
| **asset / 资产 | 一条被归档的内容（视频/文章/音频），在 hub 里对应一行 SQLite 记录 + 一个目录 |
| **scope** | 抓取范围：`single`（单个）/ `author`（作者全部）/ `collection`（合集）/ `favorites`（收藏夹）/ `comments`（评论） |
| **ingest** | 把一条链接变成一个下载任务的入口动作，对应 `POST /api/ingest` |
| **kb-hub** | 归档中枢（ADR-005），FastAPI + SQLite，负责入口归一、任务队列、资产索引、去重、落盘规范 |
| **adapter** | 平台适配器，一个平台一个文件，负责「把某平台的链接变成资产」 |

## 附录 C：本文档集导航

| 文档 | 内容 | 本文关联 |
|---|---|---|
| [VISION.md](VISION.md) | 愿景速览（入口摘要） | §0.1 |
| [**docs/vision.md**](docs/vision.md) | **【需求源头】愿景、场景约束 S1–S6、近期/远期目标、Q7–Q11** | §0.1 / ADR-005 |
| **[docs/execution-plan.md](docs/execution-plan.md)** | **施工单：开工铁律、Phase 0–11 逐项任务、验收命令、回滚、DoD** | **§6 Phase 拆解** |
| **[docs/environment.md](docs/environment.md)** | **环境事实与五条铁律、端口占用、复核清单** | **§7 资源预算** |
| [docs/roadmap.md](docs/roadmap.md) | 三条路线 A/B/C 完整定义、需求打分、决策矩阵、决策状态台账 | §0 |
| [docs/download-layer-comparison.md](docs/download-layer-comparison.md) | **下载层功能逐条对比 + 四候选矩阵 + 路线 C+** | §0 / Phase 11 |
| [docs/feasibility.md](docs/feasibility.md) | 路线 A 可行性、资源硬约束、聊天记录纠错 | 附录 A |
| [docs/mp-tools-docker.md](docs/mp-tools-docker.md) | mp-tools Docker 化、浏览器依赖逐文件实测 | Phase 1 |
| [docs/douyin-erma0.md](docs/douyin-erma0.md) | 第四候选调研与四个必改坑 | Phase 11.2 |
| [deploy/README.md](deploy/README.md) | 部署物料位与 Dockerfile 要点 | Phase 1 / Phase 2 |
| [analysis/README.md](analysis/README.md) | PC 端分析流水线规划 | Phase 9 |