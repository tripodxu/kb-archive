# 知识归档 —— 总路线图（Plan A / Plan B / Plan C）

> 这份文档存在的理由：之前有两份计划各自成立但**没有交接到一起**——
> - [feasibility.md](feasibility.md) §7 建议架构（原计划，下称**路线 A**）
> - [mp-tools-docker.md](mp-tools-docker.md)（新计划，下称**路线 B**）
>
> 本文把两者并排放，并给出**第三条混合路线 C**。**2026-10-12 用户已裁定采用路线 C**（见 [../PLAN.md](../PLAN.md) ADR-001）。
> 日期：2026-10-12
>
> **导航**：[../README.md](../README.md) 首页 ｜ [../VISION.md](../VISION.md) 愿景速览 ｜ [../PLAN.md](../PLAN.md) 规划 ｜ **本文** 路线定义 ｜ [download-layer-comparison.md](download-layer-comparison.md) 下载层功能逐条对比 ｜ [feasibility.md](feasibility.md) 可行性与核实 ｜ [mp-tools-docker.md](mp-tools-docker.md) Docker 化调研 ｜ [douyin-erma0.md](douyin-erma0.md) 第四候选调研
>
> ⚠️ **本文只解决「下载层选谁 + 分析层放哪」；用户 2026-10-12 之后提出的「多端统一入口 + 资产索引 + 可扩展平台层」需求，由 ADR-005 kb-hub 承接 → [../PLAN.md](../PLAN.md) ADR-005 与 [vision.md](vision.md)。**

---

## 0. 先说结论

**这不是「二选一」，而是「下载层选谁 + 分析层放哪」两个独立决策。**

| 决策点 | 选项 | 裁定 |
|---|---|---|
| 下载层（抓取 + 落盘） | 路线 A 的四件套 / 路线 B 的 wechat-mp-tools / **两者混合** | ✅ **C**（mp-tools + wae） |
| 分析层（转写 + 摘要 + 入库） | know-where（VPS，依赖飞书）/ **PC 本地** / 先不做 | ✅ **PC 本地** |

**已定：下载层走路线 B 的精简版（砍掉浏览器），分析层放 PC，不上 know-where。**
理由在 §3。下载层的功能差异逐条量化见 [download-layer-comparison.md](download-layer-comparison.md)。

---

## 1. 两条路线摆一起

### 路线 A —— 四件套拼装（原计划，2026-10-12 上午定）

```
飞书机器人  →  know-where (app+gateway+PostgreSQL)
                   │  httpx 提取正文 → LLM 摘要 → 飞书多维表格
         ┌─────────┴──────────┐
    腾讯云 ASR            下载器（共享目录衔接）
   (+ COS)          TikTokDownloader 抖音 / yutto B站 / wechat-article-exporter 公众号
```

| 组件 | Star | 镜像/依赖 | 浏览器 | 内存 |
|---|---|---|---|---|
| know-where（知归） | 104 | 需本地 build + **PostgreSQL 容器** | ❌ 无 | 未实测 |
| TikTokDownloader | **16703** | 官方镜像 `joeanamier/tiktok-downloader` | ❌ 无 | 轻 |
| yutto | 2064 | 官方镜像 `siguremo/yutto`，**8 个 Python 依赖**，🔴 **FFmpeg 硬依赖** | ❌ 无 | 极轻 |
| wechat-article-exporter | **13054** | Nuxt，`node:22-alpine → node:22-slim` | ❌ 无 | 中 |

- ✅ **四个组件全部零浏览器依赖**（这是当初刻意筛的第一判据）
- ✅ 分析链路完整：ASR + 摘要 + 分类 + 自动写飞书表格
- ✅ 每个组件都有官方 Docker 镜像或 Dockerfile，star 数高、可持续性好
- ❌ **四个独立项目要各自升级**，其中两个是个人/小团队维护
- ❌ **know-where 飞书硬依赖**：必须有飞书自建应用，否则这条路要重新设计
- ❌ know-where 拉 PostgreSQL，等于**多一个容器**
- ❌ 只能覆盖抖音+B站+公众号，**没有小红书/快手/视频号**
- ❌ 每个平台各有一套配置和账号体系，没有统一 UI

### 路线 B —— wechat-mp-tools 单体（2026-10-12 下午定）

```
        ┌────────── Flask 单体（17 个 blueprint，自带 React UI）──────────┐
        │ 公众号文章 │ B站 │ 抖音 │ 小红书 │ 快手 │ 视频号 │ 微信读书    │
        │ 自带 RSS 2.0 服务 + 后台定时增量调度 + 下载历史 + 账号池        │
        └────────────────────────────┬───────────────────────────────────┘
                                     ▼
                              /opt/mp-tools/data/  →  dav
```

| 项 | 值 |
|---|---|
| Star | 186（**⚠️ 远低于 A 的四个组件**） |
| 许可 | MIT (c) 2026 xuyi |
| 形态 | Flask Web 应用（`app.py:41`）+ PyInstaller 桌面壳 |
| 平台覆盖 | **6 个，最广** |
| 浏览器依赖 | B站❌无 / 抖音❌主路径无 / 快手✅必需 / 小红书✅必需(可降级) / 视频号✅必需+有头 |
| 登录 | 抖音(:232) 小红书(:46) 快手(:160) 全是 `headless=False`，要可见窗口扫码 |
| 分析能力 | **完全没有**（无 ASR、无摘要、无向量检索） |

- ✅ **平台覆盖最广**，含 A 方案没有的小红书/快手/视频号/微信读书
- ✅ **自带 UI + 自带 RSS 定时增量调度 + 自带下载历史去重**，开箱即用
- ✅ 单项目单容器，运维成本最低
- ❌ **没有任何 AI 分析能力**——所以你的「分析在 PC」设想不是妥协，是必需
- ❌ Linux 上快手/小红书/视频号基本不可用（除非接受 Chromium 700MiB）
- ❌ **公众号「主页全量」在 Linux 上不可用**（靠注入微信 PC 客户端）
- ❌ 186★，作者自述「很多同类项目已闭源转向收费，本项目坚持开源」——**建议 fork 私有化**

---

## 2. 逐条需求打分

| 你的需求 | 路线 A | 路线 B |
|---|---|---|
| 公众号文章抓取（单篇） | ✅ know-where 纯 httpx | ✅ |
| 公众号「主页 → 全量历史」 | ✅ wechat-article-exporter（**需自有公众号扫码**） | ❌ Linux 上不可用 |
| B站「UP主 → 全部」 | ✅ yutto 自签 WBI | ✅ bilibili.py 纯 HTTP |
| 抖音「主页 → 全部」 | ✅ TikTokDownloader（16703★，增量+去重） | ✅ douyin.py 主路径纯 HTTP |
| 小红书 / 快手 | ❌ 没有 | ⚠️ 需 Chromium |
| 视频号 | ❌ 没有 | ⚠️ 需 Chromium + mitmproxy |
| 「丢链接进来」自动触发 | ✅ 飞书机器人 | ⚠️ 只能用自带 UI 手工贴 |
| 定时增量同步 | ❌ 无 | ✅ 内置 RSS 调度器 |
| ASR 视频转写 | ✅ 腾讯云 ASR | ❌ 无 |
| AI 摘要 / 分类 | ✅ know-where + LLM | ❌ 无 |
| 自动入库飞书 | ✅ 核心能力 | ❌ 无 |
| 无浏览器依赖 | ✅ 四件套全部无 | ⚠️ 取决于选哪个方案 |
| 内存压力 | 3–4 个容器 | **1 个容器，砍浏览器后 150–250 MiB** |
| 长期维护风险 | 🟢 星高、多项目分散 | 🔴 186★ 单点 |

---

## 3. 为什么我推荐混合（路线 C）

**观察：A 和 B 根本不冲突——它们是上下游，不是竞品。**

- 路线 B 强在**下载**（平台广、有 UI、有增量调度、有去重）
- 路线 A 的 know-where 强在**分析**（ASR + LLM + 飞书入库）
- 路线 A 里的三个下载器（TikTokDownloader / yutto / wechat-article-exporter）**能力上并不比 B 强**，只是更成熟、更省心

### 路线 C 的形态

```
┌─ VPS ────────── 只做下载（方案 B 精简版：砍掉 pywebview/playwright/mitmproxy/subtitle_remover）
│   wechat-mp-tools  →  /opt/mp-tools/data/  →  rsync →  dav（已有容器 127.0.0.1:6065）
│
│   ＋ 兜底：wechat-article-exporter（Docker，127.0.0.1:8012）专攻「公众号主页全量」
└──────────────────────────────────┬─────────────────────────────────────────────┘
                                   │ WebDAV 拉取
┌─ PC ──────────────────────────────▼─────────────────────────────────────────────┐
│  faster-whisper 转写 → LLM 摘要/分类 → Obsidian / 飞书多维表格
└───────────────────────────────────────────────────────────────────────────────┘
```

### 三条理由

1. **know-where 的飞书硬依赖可以延后**。你现在连飞书自建应用都还没建（问了两轮），先上 know-where 会卡住；而且它要拖一个 PostgreSQL 容器，在 975 MiB available 上是纯浪费。分析放 PC 完全没损失，还白赚一个 GPU/本地模型选项。
2. **下载层用 B 更省心**。一个容器替代三个，还多了小红书/快手/视频号，代价只是砍浏览器——而砍掉的正是你 975 MiB 装不下的东西。
3. **公众号是唯一缺口，单独补**。wechat-mp-tools 在 Linux 上做不了公众号「主页全量」，这恰恰是 13054★ 的 wechat-article-exporter 的主场。两者不冲突，各管一段。

### 但要诚实说清楚路线 C 的代价

- 没有飞书机器人入口了，「丢链接进去自动处理」这个交互要靠 PC 端脚本或手动贴 URL
- RSS 调度器用 JSON 文件存（`rss_articles.json`，每号上限 200 篇），量大后要换 SQLite
- 路线 C 里 PC 端那套（whisper + LLM + 入库）**是我给你画的蓝图，还没有成品代码**
- **抖音深度落后路线 A 约 6 项能力**（评论采集、点赞收藏统计、剪贴板监听、多账号批量、Web API、收藏夹）。逐条量化见 [download-layer-comparison.md](download-layer-comparison.md) §2.1

---

## 3.5 第四个候选：douyin_erma0

项目最早（2026-10-11）评估的方案，曾准备部署到 `127.0.0.1:8010`，后因 `pywebview[gtk]` 构建失败**长期悬空**。完整档案见 [douyin-erma0.md](douyin-erma0.md)。

| 项 | 值 |
|---|---|
| 规模 | **31 py / 1.4 MB / 10 依赖** —— 四个候选里最轻 |
| 形态 | FastAPI + React 前端 + Aria2，**单端口单容器**（`server.py:157` 直接 mount 静态文件） |
| 采集类型 | **10 种**，含 `hashtag`（话题挑战）、`music`（音乐原声）、`search`（关键词搜索）、`following` / `follower`（关注粉丝）—— **四项是别家都没有的** |
| 增量 | ✅ 真增量（`crawler.py` 传 `results_old` 到 `DataParser.parse_awemes`） |
| 阻塞 | 🔴 **四个坑**（公网 80 暴露 / build ARG 烘焙 / Dockerfile 未装 aria2 / `pywebview[gtk]` 构建失败）→ 约 30 分钟可修完，见 [douyin-erma0.md](douyin-erma0.md) §4 |
| 定位 | 🟡 **可选补充**，不作任何里程碑依赖（ADR-005 落地后可改为 hub 的一个 adapter 文件） |

### 路线 C+（可选增强，Phase 11 专项评估）

| 补充 | 补什么 | 成本 | 触发条件 |
|---|---|---|---|
| **+ TikTokDownloader** | 抖音的评论采集 / 点赞统计 / 剪贴板监听 / 多账号 / Web API / 收藏夹 | 官方镜像，1 容器，共享数据目录；或做成 `hub/adapters/tiktok_downloader.py` | 抖音是主力平台时 |
| **+ douyin_erma0** | 话题挑战 / 音乐原声 / 关键词搜索 / 关注粉丝 | 1 容器 + 修四个坑（约 30 分钟）；或做成 `hub/adapters/douyin_erma0.py` | 上述四项确有需求时 |
| ~~+ yutto~~ | — | — | ❌ 不建议：B 站能力已被 mp-tools 打平，FFmpeg 硬依赖，无净收益 |

> ⚠️ **不要现在就装**。1.9 GiB 是本项目第一约束，任何补充都应在 M1–M3 跑通后再按需引入。若只加一个，加 **TikTokDownloader**（补深度）优先于 douyin_erma0（补广度）。两者二选一专项对比见 [download-layer-comparison.md](download-layer-comparison.md) §3.1。

---

## 4. 决策矩阵

| 如果你更在意 | 选 |
|---|---|
| 「丢链接进飞书就自动归档」的完整体验 | 路线 A（必须建飞书自建应用） |
| 平台覆盖广、有 UI、有定时增量 | 路线 B |
| 内存安全、长期可维护 | 路线 A 的下载器 + 路线 B 的 UI，或纯 A |
| 现在就想跑起来、别再等 | 路线 C（只差 PC 端脚本要自己写） |
| **抖音评论/统计/多账号** | 路线 A，或 **C + TikTokDownloader** |
| **综合推荐** | **路线 C** ✅ 已定 |

---

## 5. 执行顺序与决策状态

| 优先级 | 事项 | 状态 |
|---|---|---|
| 🔴 P0 | 抖音/小红书/快手 Cookie —— **PC 上登录一次导出，贴进 VPS**，省掉 Xvfb 那一套 | 未答 |
| 🟠 P1 | ~~选路线（A / B / C）~~ | ✅ **已定：路线 C**（2026-10-12） |
| 🔴 P0 | ~~是否接受 ADR-005 kb-hub 薄门面~~ | ⏳ **待答（Q0）** —— 承接多端统一入口 / 资产索引 / 可扩展平台层需求 |
| 🟠 P1 | 飞书自建应用 | ⏸ **推迟到 Phase 10**；ADR-002 记录「后续提供」，且**不再阻塞 S2**（`POST /api/ingest` 即可） |
| 🟠 P1 | 确认有无可扫码登录的公众号（Phase 4 前置） | 未答（Q2） |
| 🟡 P2 | 视频长期留还是转写完删（磁盘 34 G） | 未答（Q8） |
| 🟡 P2 | 「自己的云端」具体指什么（VPS 盘 / 对象存储 / NAS） | 未答（Q7） |
| 🟡 P2 | Obsidian vault 路径（Phase 9 前置） | 未答 |
| 🟡 P2 | docker 网络清理：`docker network rm cli-proxy-api_default`（空网络） | 已提议未答 |
| ✅ — | `jev-router` 去留 | ✅ **已定：长期保留**（ADR-003） |
| ⚪ — | douyin_erma0 部署 | 🟡 **降级为 C+ 可选项（Phase 11.2）**，不阻塞任何里程碑；档案见 [douyin-erma0.md](douyin-erma0.md) |
| ⚪ — | 是否追加 TikTokDownloader | 🟡 C+ 可选项（Phase 11.1），按抖音实际需求定 |

---

## 6. 相关文档（双向导航）

### 本项目内

| 文档 | 内容 | 关联本文哪节 |
|---|---|---|
| [../README.md](../README.md) | 项目首页 | — |
| [../VISION.md](../VISION.md) | 愿景速览（近期/远期目标、可扩展性四条硬定义） | §0 |
| **[vision.md](vision.md)** | **【需求源头】愿景原话拆解、S1–S6 场景约束、Q7–Q11** | §0 / §5 |
| [../PLAN.md](../PLAN.md) | 项目规划与实施计划（ADR-001 路线 C / ADR-005 kb-hub） | §0 / §3 |
| [**download-layer-comparison.md**](download-layer-comparison.md) | **下载层功能逐条对比（A vs C）+ 四个候选功能矩阵 + 抖音二选一专项 + 路线 C+** | §1 / §2 / §3.5 |
| [feasibility.md](feasibility.md) | 路线 A 的完整可行性分析、组件源码级核实、聊天记录纠错 | §1 路线 A |
| [mp-tools-docker.md](mp-tools-docker.md) | 路线 B 的 Docker 化调研、浏览器依赖逐文件实测、三种部署方案 | §1 路线 B |
| [douyin-erma0.md](douyin-erma0.md) | 第四候选 douyin_erma0 完整调研与**四个坑**的修法 | §3.5 |
| [../deploy/README.md](../deploy/README.md) | 部署物料位（含 hub 容器） | §3 |
| [../analysis/README.md](../analysis/README.md) | PC 端分析流水线规划位 | §3 分析层 |

### 母仓库（未脱敏，**勿外传**）

| 文档 | 内容 |
|---|---|
| `../../docs/knowledge-archive-feasibility.md` | [feasibility.md](feasibility.md) 的原始未脱敏版本 |
| `../../docs/knowledge-archive-roadmap.md` | **本文的原始未脱敏版本** |
| `../../docs/wechat-mp-tools-vps-plan.md` | [mp-tools-docker.md](mp-tools-docker.md) 的原始未脱敏版本 |
| `../../docs/container-topology.md` | VPS 现有 10 个容器与宿主组件全景 |
| `../../docs/opencode-proxy-runbook.md` | opencode2api / WARP 池运维手册 |

> 本项目文档同时同步到服务器 `/root/kb-archive/`。