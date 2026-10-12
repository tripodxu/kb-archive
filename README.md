# kb-archive —— 个人知识归档系统

> **自下载 · 可保存 · 自行整理 · 多媒介 · 独属于自己 · 本地化**
>
> 看到一个有意思的内容 → 终端 / 手机 / 电脑 → **直接告诉 AI 发个链接**
> → 自由选择「单个作品」还是「这个作者的全部作品」 → 本地化下载 → 存到自己的云端 → 自动归档整理。

**完整愿景与需求 → [VISION.md](VISION.md) / [docs/vision.md](docs/vision.md)**
**实施计划 → [PLAN.md](PLAN.md)**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Status](https://img.shields.io/badge/status-roadmap%20locked-orange.svg)

---

## 这是什么

一套把微信公众号文章、B 站视频、抖音视频**自动抓取下来、存好、归好类**的自托管知识库。

设计上刻意把两件事拆开：

| | 跑在哪 | 干什么 | 为什么 |
|---|---|---|---|
| **下载层** | VPS | 抓取正文、下载媒体、去重、定时增量同步、归档索引 | 需要长时间运行和网络稳定 |
| **分析层** | 你的 PC | 语音转写、摘要提炼、分类、入库 | 需要算力，且数据不出本地 |

VPS 只有 1.9 GiB 内存、**没有 GPU**，本项目实测过在此跑本地 LLM 会必然 OOM。分开是必然选择，不是妥协。

### 近期目标

**抖音 + B站 + 公众号的服务器端下载功能，且必须可扩展。**

可扩展性写成四条可验收的硬定义：

1. 新增平台 = 一个 adapter 文件 + 一行注册，不改中枢代码
2. 新增媒体类型 = 只改 asset schema 的 `files` 字段
3. 新增入口（CLI / Web / IM / 浏览器插件）= 只实现「调一次 `POST /api/ingest`」
4. 新增存储后端（本地盘 / 对象存储 / NAS）= 一个 storage 接口

---

## 能力清单

- [x] 公众号文章抓取 —— 正文提取 + 图片本地化，HTML 100% 还原排版
- [x] 公众号「主页 → 全量历史文章」
- [x] B 站「UP主 / 合集 / 收藏夹 → 全部下载」 —— 视频 + 弹幕(ASS) + 字幕(SRT)
- [x] 抖音「主页 / 合集 → 全部下载」 —— 无水印 + 增量去重
- [ ] 抖音「评论采集 / 点赞统计 / 剪贴板监听 / 多账号」 —— 需补 TikTokDownloader（Phase 11.1 可选）
- [ ] 抖音「话题挑战 / 音乐原声 / 关键词搜索」 —— 需补 douyin_erma0（Phase 11.2 可选）
- [ ] 小红书 / 快手 / 视频号 / 微信读书 —— 名义覆盖，实际受浏览器依赖限制（1.9 GiB 下受限）
- [ ] **多入口统一**：Web UI / CLI / **任何 AI 直接调 HTTP API**（Phase 8）
- [ ] **资产索引与幂等去重**（Phase 2）
- [x] Web UI 粘贴链接即可下载
- [x] 定时增量同步（内置 RSS 订阅调度）
- [ ] AI 摘要与知识提取（PC 端）
- [ ] 归档入库 —— Markdown / Obsidian
- [ ] 飞书多维表格（可选）

> 逐条功能的**四个下载器能力矩阵**与 A/B/C 路线差异量化 → [docs/download-layer-comparison.md](docs/download-layer-comparison.md)

---

## 架构

```
┌─ 入口层 ──────────────────────────────────────────────┐
│  Web UI（手机/电脑）  │  CLI  │  AI: curl POST /api/ingest │
└──────────────────────────┬────────────────────────────┘
                           ▼
┌─ kb-hub（FastAPI + SQLite，薄门面）────────────────────┐
│  链接解析 → adapter 注册表 → 任务队列 → 资产索引/去重      │
│  bilibili │ wechat │ douyin │ (xiaohongshu │ kuaishou …) │
└──────┬───────────────────────────────┬─────────────────┘
       │ 调用                           │ 落盘 + 索引
       ▼                               ▼
┌─ 抓取执行器 ───────────┐    ┌─ 资产库 ─────────────────────┐
│  mp-tools（Flask :5200） │    │ /opt/kb-archive/media/       │
│  wae（公众号全量补位）    │    │  <platform>/<author>/<id>/   │
└──────────────────────────┘    │   meta.json video.mp4 …     │
                                └──────────┬─────────────────┘
                                           │ rsync → dav (WebDAV)
┌─ PC（分析层）───────────────────────────▼────────────────┐
│  faster-whisper 转写 → LLM 摘要分类 → Obsidian            │
└──────────────────────────────────────────────────────────┘
```

> ADR-005：kb-hub 是**薄门面**，不重写抓取逻辑 —— 抓取代码仍来自 mp-tools，
> hub 只做「入口归一 + 资产语义」，每一步都能独立验收、独立回滚。详见 [PLAN.md](PLAN.md) ADR-005。

---

## 技术选型

| 层 | 组件 | Star | License | 浏览器依赖 |
|---|---|---|---|---|
| 下载 · 主 | [x554960766/wechat-mp-tools](https://github.com/x554960766/wechat-mp-tools) | 186 | MIT | 剥离后**无** |
| 下载 · 公众号 | [wechat-article/wechat-article-exporter](https://github.com/wechat-article/wechat-article-exporter) | **13054** | MIT | **无** |
| 同步 | hacdias/webdav | — | — | — |
| 分析 · 转写 | faster-whisper | — | MIT | — |
| 分析 · 摘要 | 外部 LLM API | — | — | — |

### 为什么这么选

**1.9 GiB 内存是这个项目的第一约束**，所以选型的首要判据是「无浏览器依赖」——headless Chromium 常驻 400–700 MiB，而 Chromium 一旦 OOM 会连带打死同机其它业务。

选型过程中做了大量源码级核实，**推翻了几条广为流传的错误说法（含我自己写错过的一条）**：

| 常见说法 | 实际情况 |
|---|---|
| 「f2 支持 B 站」 | ❌ `f2/apps/` 里没有 bilibili 模块 |
| 「know-where 靠 RSSHub 抓公众号」 | ❌ 它自带 `adapters/wechat.py`，纯 httpx + BeautifulSoup |
| 「know-where 用单容器 SQLite」 | ❌ 硬依赖 PostgreSQL |
| 「yutto 连 ffmpeg 都不需要」（**我自己写错的**） | ❌ `README.md:99` 明写需配置 FFmpeg，`utils/ffmpeg.py` 无条件 import，**硬依赖** |

### 三条路线与最终选择

评估过三条完整路线，最终采用**路线 C**（详见 [PLAN.md](PLAN.md) §0 与 [docs/roadmap.md](docs/roadmap.md)）：

| | 路线 A（know-where 四件套） | 路线 B（mp-tools 原样全量） | **路线 C ✅** |
|---|---|---|---|
| 容器数 | 4（+PG） | 1 | **2 + hub** |
| 浏览器依赖 | ❌ 全无 | 🔴 700–1200 MiB | ❌ 剥离后无 |
| 飞书依赖 | 🔴 硬依赖 | ❌ | ❌（Phase 10 可选） |
| 主要短板 | 平台仅 3 个 | 快手/小红书/视频号不可用 | **抖音深度落后 A 约 6 项** |

两个抖音下载器候选，都降级为路线 C+ 可选项：

| 候选 | 定位 | 部署 |
|---|---|---|
| **TikTokDownloader**（16703★，2026-09-16 活跃） | 成品，深度强（评论/统计/剪贴板/多账号/Web API） | ✅ 官方镜像开箱即用 |
| **douyin_erma0**（31 py，2026-05-15 停滞） | 半成品，广度强（话题/音乐原声/搜索/关注粉丝） | 🔴 四个坑，约 30 分钟修完 |

二选一专项对比 → [docs/download-layer-comparison.md](docs/download-layer-comparison.md) §3.1

**⚠️ 关于 wechat-mp-tools**：上游仅 186★，作者在 README 中提到「很多同类开源项目已陆续闭源转向收费」。**本项目计划 fork 私有化并固定 commit 后使用**，不直连上游。

---

## 项目状态

**规划已定，开发待启动。** ADR-005（kb-hub 归档中枢）已于 2026-10-12 获批，当前处于 Phase 0 前置。

> **要接手执行？先读这两份：**
> **[docs/execution-plan.md](docs/execution-plan.md)**（施工单：逐项任务 + 验收命令 + 回滚）
> **[docs/environment.md](docs/environment.md)**（环境事实 + 五条铁律）

| 里程碑 | 内容 | 状态 |
|---|---|---|
| M1 冷启动 | mp-tools 容器跑通，三平台单链接抓取成功，内存实测 | ⏳ 待启动 |
| M2 中枢 | `POST /api/ingest` 可用，asset schema 冻结，重复链接幂等 | ⏳ |
| **M3 近期目标** | **抖音 + B站 + 公众号：单作品 / 作者全部 均可下载入库** | ⏳ |
| M4 端到端 | VPS → 云端同步（dav / 对象存储） | ⏳ |
| M5 多端入口 | 手机 / 终端 / AI 三种入口 | ⏳ |
| M6 闭环 | 丢链接 → 带摘要的 Obsidian 归档 | ⏳ |
| M7 增强 | 飞书入库 / 抖音深度补齐 | ⏳ |

**M1 与 M2 不需要任何凭据即可开工**（无需 Cookie、无需飞书、无需 Chromium）。

---

## 文档

| 文档 | 内容 | 什么时候看 |
|---|---|---|
| **[VISION.md](VISION.md)** | **愿景速览**（近期/远期目标、可扩展性的四条硬定义） | 先看这个 |
| **[docs/vision.md](docs/vision.md)** | **【需求源头】** 愿景、场景约束 S1–S6、近期/远期目标、待澄清问题 Q7–Q11 | 开工前必读 |
| **[PLAN.md](PLAN.md)** | **项目规划与实施计划** —— 三条路线速览、ADR-001~005 决策记录、架构、资产 schema、API 契约、Phase 0–11 WBS、风险登记册 | 开工前通读 |
| **[docs/execution-plan.md](docs/execution-plan.md)** | **施工单** —— 给执行者：六条开工铁律、Phase 0–11 逐项任务、可复制的验收命令、回滚演练、DoD 清单 | **动手前必读** |
| **[docs/environment.md](docs/environment.md)** | **环境事实与五条铁律** —— 主机规格、Docker 29 陷阱、防火墙规则、现有 10 容器、端口占用、开工复核清单 | **动手前必读** |
| [docs/roadmap.md](docs/roadmap.md) | 三条路线 A/B/C 完整定义、逐需求打分、决策矩阵、决策状态台账 | 想知道「为什么是 C」 |
| **[docs/download-layer-comparison.md](docs/download-layer-comparison.md)** | **下载层功能逐条对比（A vs C）+ 四个下载器能力矩阵 + 抖音二选一专项 + 路线 C+ 建议** | 想量化「C 到底差在哪」 |
| [docs/feasibility.md](docs/feasibility.md) | 路线 A 可行性分析：资源实测、组件源码级核实、错误说法纠正 | 想看完整选型过程 |
| [docs/mp-tools-docker.md](docs/mp-tools-docker.md) | mp-tools Docker 化：各模块浏览器依赖逐文件实测、三种部署方案 | Phase 1 实施前 |
| [docs/douyin-erma0.md](docs/douyin-erma0.md) | douyin_erma0 完整调研 + **四个坑**（含「Dockerfile 未装 aria2」）与修法 | Phase 11.2 决策时 |
| [deploy/README.md](deploy/README.md) | 部署物料位：Dockerfile 要点、网络约束（必须 `network_mode: bridge`） | Phase 1 / 2 实施时 |
| [analysis/README.md](analysis/README.md) | PC 端分析流水线规划位 | Phase 9 规划时 |
| [docs/ai-usage.md](docs/ai-usage.md) | AI 调用说明 + 可复制的 curl 示例 | Phase 8 产出 |
| [docs/operations.md](docs/operations.md) | 运维手册 | Phase 6 产出 |

> 本文各文档之间**已建立双向反向链接** —— 每份文档顶部都有「导航」行，文末都有「相关文档」表，可从任意一份跳到其余任意一份。

---

## 法律与合规声明

- 本项目仅用于**个人自用的内容归档**
- **不做**账号池、**不**对外提供抓取服务、**不**绕过付费墙
- 公众号批量抓取限定为**你自己名下公众号的自用抓取**（上游项目 README 已明确声明此边界）
- 不传播下载内容；请遵守各平台服务条款与著作权法

---

## License

MIT —— 见 [LICENSE](LICENSE)。

> 注意：项目**调用**的开源组件各有其许可（wechat-mp-tools: MIT、wechat-article-exporter: MIT、webdav: MIT）。本仓库的 LICENSE 仅覆盖本项目自身的代码与文档。