# 下载层功能对比 —— 路线 A vs 路线 C（含全部四个候选组件）

> 这份文档回答一个问题：**同样只负责「抓取下载落盘」，路线 A 的三件套和路线 C 的单体，功能上到底差多少？**
>
> 结论先行：**C 赢在广度（多 4 个平台、合并成 1 个入口、有定时调度、有统一去重），A 赢在抖音深度（多 6 项能力）。B 站两者基本等价。公众号零差异。**
>
> 所有功能项均来自源码逐文件核实，非 README 转述。核实方式见文末「复核命令」。
>
> 日期：2026-10-12 ｜ 上级文档：[../PLAN.md](../PLAN.md) ｜ 路线定义：[roadmap.md](roadmap.md) ｜ 需求源头：[vision.md](vision.md)
>
> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) ｜ [vision.md](vision.md) ｜ [../PLAN.md](../PLAN.md) ｜ [roadmap.md](roadmap.md) ｜ **本文** 下载层逐条对比 ｜ [feasibility.md](feasibility.md) ｜ [mp-tools-docker.md](mp-tools-docker.md) ｜ [douyin-erma0.md](douyin-erma0.md)

---

## 0. 先厘清「下载层」指什么

| 层 | 职责 | 路线 A | 路线 C |
|---|---|---|---|
| 下载层 | 抓正文 / 下媒体 / 去重 / 增量调度 | TikTokDownloader + yutto + wechat-article-exporter | wechat-mp-tools + wechat-article-exporter |
| 分析层 | 转写 / 摘要 / 分类 / 入库 | know-where（VPS，带 PostgreSQL 容器） | PC 本地（自建，Phase 5） |

**两条路线的分析层不同，本文档只比下载层。** 分析层对比见 [roadmap.md](roadmap.md) §3。

---

## 1. 总览：七个维度

| 维度 | 路线 A 下载层 | 路线 C 下载层 | 胜方 | 差距量级 |
|---|---|---|---|---|
| 容器数 | **3** | 2 | C | −1 个容器 |
| 平台覆盖 | 3（抖音 / B站 / 公众号） | **7**（+小红书 / 快手 / 视频号 / 微信读书） | **C** | **+4 平台** |
| 统一 Web UI | ❌ 三个独立 CLI | ✅ Flask + React | **C** | +1 |
| 定时增量调度 | ❌ 无 | ✅ 内置 RSS 调度器（`rss_scheduler.py`，`MAX_ARTICLES_PER_ACCOUNT = 200`） | **C** | +1 |
| 统一下载历史/去重 | ❌ 各下载器自理 | ✅ `download_history.json` + `add_history_item` / `clear_history` | **C** | +1 |
| 内存占用 | 3 容器，未实测 | 目标 150–250 MiB（实测值 = Phase 1.10 决策门） | C | 约 −200–400 MiB |
| 浏览器依赖 | 三个组件**全部无** | mp-tools 剥离后**无**（快手/小红书/视频号/微信读书除外） | 平 | — |
| 长期维护风险 | 3 个项目各自升级（16703★ / 2064★ / 13054★） | 1 个项目单点（186★） | **A** | 风险集中度 |

> ⚠️ **上表最后一行是 C 的真实弱点**：路线 C 把长期维护风险从「三个分散项目」压缩成「一个 186★ 单点」。PLAN.md 的 ADR-001 因此把「先 fork 私有化并固定 commit」列为 Phase 1 的第 1 项任务。

---

## 2. 逐平台细比

### 2.1 抖音 —— **A 明显领先（+6 项）**

| 功能项 | A: TikTokDownloader | C: wechat-mp-tools `backend/douyin.py` | 备注 |
|---|---|---|---|
| 发布作品 | ✅ | ✅ 主页 | |
| 喜欢作品 | ✅ | ⚠️ `_fetch_liked_via_browser`(:474) —— **需浏览器，剥离后降级** | |
| 收藏作品 | ✅ | ❌ | A 独有 |
| 收藏夹 | ✅ | ❌ | A 独有 |
| 合集 | ✅ | ✅ | |
| **评论采集** | ✅ | ❌ | **A 独有** |
| **点赞/收藏统计数据** | ✅ | ❌ | **A 独有** |
| **剪贴板链接监听** | ✅ | ❌ | **A 独有** |
| **多账号批量** | ✅ | ❌ | **A 独有** |
| **Web API 调用** | ✅ | ❌ | **A 独有**（注：A 自身目前也是 CLI，WebUI 要等 6.0 重构） |
| 自定义账号/合集标识 | ✅ | ❌ | A 独有 |
| 私密作品 | 需登录 Cookie **且登录号已关注该私密号** | — | A 独有 |
| 无水印直链 | ✅ | ✅ | |
| 增量下载 | ✅ | ⚠️ 靠 RSS 调度器间接实现 | |

**实测证据（A 侧）**：`TikTokDownloader/README.md` 能力表逐条 —— 「下载抖音账号发布/喜欢/收藏/收藏夹作品」「采集抖音作品评论数据」「下载抖音合集作品」「记录点赞收藏等统计数据」「支持账号作品增量下载」「支持自定义账号/合集标识」「支持 Web API 调用」「监听剪贴板链接下载作品」；注意事项「获取私密账号的发布作品数据需要登录后的 Cookie，且登录的账号需要关注该私密账号」「批量下载收藏作品模式目前仅支持当前已登录 Cookie 对应账号」。

**实测证据（C 侧）**：`backend/douyin.py:3` 注释「移植自 douyin-downloader-rust 项目，使用直接 HTTP API 调用代替 Playwright 渲染」；配 `backend/douyin_sign.py`（X-Bogus/ABogus 自签）+ `backend/douyin_mstoken.py`（msToken）。浏览器调用仅出现在 `_fetch_via_browser`(:417) 与 `_fetch_liked_via_browser`(:474) —— 这两个在剥离 playwright 后会失效。

> **结论：抖音是路线 C 的最大缺口。** 但补法很便宜：TikTokDownloader 有官方镜像 `joeanamier/tiktok-downloader`（`-p 端口:5555 -v vol:/app/Volume`），无浏览器依赖，`grep -il 'playwright|selenium|pyppeteer'` 0 命中，直接挂在 `/opt/mp-tools/data/` 共享目录即可。见 §5 建议方案 C+。

### 2.2 B 站 —— **两者基本等价，各有一项独有**

| 功能项 | A: yutto | C: mp-tools `backend/bilibili.py` |
|---|---|---|
| 投稿视频（BV/av） | ✅ | ✅ `download_single`(:1072) / `extract_bvid_or_avid`(:280) |
| 番剧批量 | ✅ `-b/--batch`（ep / md / ss 页面） | ✅ `get_bangumi_season_by_ep_id`(:388) / `get_bangumi_season_by_media_id`(:398) / `get_bangumi_list`(:408) |
| **课程（cheese）** | ✅ `src/yutto/api/cheese.py` | ❌ | 
| **合集** | ✅ `src/yutto/api/collection.py` | ✅ `get_collection_details`(:438) |
| **UP 主空间全量** | ✅ `src/yutto/api/space.py` | ✅ `get_user_videos`(:554, page_size=30) |
| **收藏夹** | ❌ `src/yutto/api/` 下**无** fav 模块 | ✅ `get_favorite_details`(:488) |
| 弹幕 → ASS | ✅ biliass 2.5.0 | ✅ `xml_to_ass`(:122) |
| 字幕 → SRT | ✅ | ✅ `json_to_srt`(:217) |
| 大会员画质档 | ✅ `AuthInfo.SESSDATA` 决定；`1080P 高码率 AVC/HEVC/AV1` 属大会员项 | ✅ `get_playurl`(:314) 默认 `qn=80` |
| WBI 签名 | ✅ `encode_wbi(params, wbi_img)` 自签 | ✅（`backend/bilibili.py` 内自实现） |
| 批量 | ✅ `-b/--batch` | ✅ `_run_batch_download_thread`(:857) |
| 下载历史去重 | ❌ | ✅ `add_history_item`(:96) / `get_history` / `clear_history` |
| **FFmpeg** | 🔴 **硬依赖** | 🟡 经 `backend/transcode.py` 包命令行 |
| Web UI | ❌ | ✅ |

> ⚠️ **更正一处此前结论错误**：我此前在旧文档中写「yutto 连 ffmpeg 都不需要」，**这是错的**。查证：`yutto/README.md:99` 明写「请确保安装 Python3.11 及以上版本，并配置好 FFmpeg」；`src/yutto/utils/ffmpeg.py:18` `FFMPEG_PATH = "ffmpeg"`；`src/yutto/__main__.py:23` 无条件 `from yutto.utils.ffmpeg import FFmpeg`；`src/yutto/validator.py:118/139` 用 `ffmpeg.video_encodecs` / `audio_encodecs` 校验用户设定的编码。**FFmpeg 对 yutto 是硬依赖。**

**B 站净结论**：功能上打平。C 多「收藏夹 + 下载历史去重」，A 多「课程(cheese) + 显式的系列批量解析」。**C 在 B 站上完全够用**，没有理由再挂一个 yutto。

### 2.3 公众号 —— **零差异**

两条路线的公众号「主页全量」**用的是同一个组件**：`wechat-article-exporter`（13054★，Nuxt3，MIT）。

| 能力 | A | C |
|---|---|---|
| 主页 → 全量历史文章 | ✅ wae | ✅ wae（**同一个组件**） |
| 批量原理 | 借公众号后台编辑器「搜索其他公众号文章」 | 同 |
| 前置凭据 | 需可扫码登录的**自有公众号** | 同 |
| 单篇文章正文抓取 | know-where `adapters/wechat.py`（在**分析层**顺带做） | mp-tools `backend/articles.py`（在**下载层**） |

差异只在「单篇抓取放在哪一层」，功能等价。

### 2.4 A 完全没有的平台 —— C 白赚 4 个

| 平台 | mp-tools 源码位置 | 浏览器依赖 | 在 1.9 GiB 下可用性 |
|---|---|---|---|
| 小红书 | `backend/xiaohongshu.py` | ✅ 必需（`:800` 失败 → `:808` 降级主页首屏解析；`_XhsBrowserSession`(:854) 物理滚动） | 🟠 受限，降级可用 |
| 快手 | `backend/kuaishou.py` | ✅ 必需（`:401` 注释「突破前端 JS 签名校验 (result=50)」，`:553/:567` 无条件 fallback） | 🟠 受限 |
| 视频号 | `backend/channels.py:293` | ✅ 必需 + **必须 `headless=False`** + mitmproxy 抓包 | 🔴 1.9 GiB 下基本不可用 |
| 微信读书 | `backend/weread_browser.py:120` | ⚠️ 保活 headless | 🟠 受限 |

> 这 4 个平台在路线 C 里**名义覆盖、实际受限**。1.9 GiB 内存下 Phase 1.10 的 Chromium 实测（PLAN.md 任务 3.1）是决定它们能否真正启用的唯一数据点。**不要把它们当成已到手的能力。**

---

## 3. 四个候选下载器功能矩阵

| 能力 | yutto | TikTokDownloader | wechat-mp-tools | douyin_erma0 |
|---|---|---|---|---|
| 平台 | B站 | 抖音/TikTok | 7 个 | 抖音 |
| Star | 2064 | **16703** | 186 | — |
| License | GPL-3.0 | GPL-3.0 | MIT | **GPL-3.0** |
| 官方镜像 | ✅ `siguremo/yutto` | ✅ `joeanamier/tiktok-downloader` | ❌ 自建 | ❌ 自建 |
| **Web UI** | ❌ **0 命中**（纯 CLI） | 🟡 有 API 模式 + `static/js` 前端，README 称 6.0 重构后才正式做 WebUI | ✅ | ✅ React |
| 统一 API | ❌ | ✅ FastAPI `:5555` | REST | ✅ REST + SSE |
| 定时增量调度 | ❌ | ❌ | ✅ RSS 调度器 | ❌ |
| 下载历史去重 | ❌ | ✅ 记录已下载作品 ID | ✅ | ✅ `results_old` 智能增量 |
| 批量下载 | ✅ `-b` | ✅ | ✅ | ✅ Aria2 |
| 弹幕 | ✅ ASS | — | ✅ ASS | — |
| 字幕 | ✅ | — | ✅ SRT | — |
| 评论采集 | — | ✅ | — | ❌（仅取 `comment_count`） |
| 剪贴板监听 | — | ✅ | — | — |
| 多账号 | ✅ 多 profile | ✅ | ✅ `account_pool.json` | — |
| 浏览器依赖 | ❌ | ❌ | 剥离后 ❌ | ❌（理论）/ 🔴 pywebview 构建失败 |
| FFmpeg | 🔴 **硬依赖** | 视频转码需 | 🟡 需 | ⚠️ 依赖外部 aria2c |
| 依赖数量 | **8 个** | **12 个（全 pin）** | 剥离后 ~12 | **10 个** |
| 代码量 | 中 | 9.4 M / 129 py | 174 py / 5.3 M | **2.0 M / 31 py** |
| 最后提交 | 2026-10-11 | **2026-09-16** | 2026-10-10 | ⚠️ **2026-05-15** |
| 部署就绪度 | ✅ | ✅ 官方镜像开箱即用 | 需自建 Dockerfile | 🔴 **四个坑**（见 [douyin-erma0.md](douyin-erma0.md)） |

### 3.1 TikTokDownloader vs douyin_erma0 —— 抖音二选一专项对比

> 这是近期目标平台「抖音」上唯一需要二选一的地方，故单独展开。

| 维度 | **TikTokDownloader** | **douyin_erma0** |
|---|---|---|
| 定位 | **成品**：社区大、文档全、更新勤 | **可改造的半成品**：代码量小 4 倍，但坑多 |
| 维护 | 🟢 2026-09-16 仍在提交，16703★ / 2808 fork | 🔴 2026-05-15 停滞，个人 fork 无 issue 追踪 |
| 采集维度 | 发布 / 喜欢 / 收藏 / **收藏夹** / 合集 | `aweme` `post` `favorite` `collection` `mix` + **话题挑战 / 音乐原声 / 关键词搜索 / 关注粉丝** |
| 抖音独有能力 | **评论数据采集**、点赞/收藏等统计、**剪贴板监听**、多账号批量、**Web API 模式** | 无 |
| douyin_erma0 独有能力 | 无 | 上面 4 项（+ 10 种 type 最全） |
| 增量 | 按作品 ID 记录已下载 | `crawler.py:206-215` 的 `results_old` 智能增量 |
| 部署 | ✅ 官方镜像 `joeanamier/tiktok-downloader`，`-p 5555:5555 -v vol:/app/Volume`，Dockerfile 干净双阶段 | 🔴 需自建，**四个坑**：公网 80 暴露 / build ARG 烘焙 / **Dockerfile 未装 aria2** / pywebview 构建失败 |
| 运行时 | 需 Node.js（`javascript==1.1.2.6` 执行签名 JS） | 基础镜像自带 Node 22 |
| 存储 | SQLite / MySQL / CSV / XLSX 四后端可选 | 裸 JSON，无事务 |
| 改造审计成本 | 129 py，读完成本高 | **31 py，半天读完** |
| 形态 | CLI + TUI + GUI + Web API 四合一 | Web 面板 + CLI |
| Cookie 限制 | 私密号需登录**且登录号已关注该号**；批量下收藏仅限当前登录号 | 公开内容基本免 Cookie |

**结论**：**要「能用的下载器」选 TikTokDownloader；要「好改的代码库 + 补广度」选 douyin_erma0。**
两者不是同维度替代 —— **DTK 是成品，douyin_erma0 是半成品**。

**对 ADR-005 的影响**：引入 kb-hub 后，两者都不必以「独立容器 + 人工点两处」的方式存在，
都可以做成 `hub/adapters/` 下的一个文件，入口 / 索引 / 去重 / 归档全部复用。
其中 **TikTokDownloader 自带 FastAPI `:5555`，做成 HTTP 型 adapter 成本更低**。

### 3.2 douyin_erma0 的独有价值

虽然它与本项目方向重叠，但**有 4 项能力是别家没有的**（实测自 `README.md` 功能表 + `backend/lib/douyin/crawler.py:88-103` 的 `self.type` 分支）：

| 独有能力 | 源码位置 |
|---|---|
| **话题挑战作品** | `crawler.py` type = `"hashtag"` |
| **音乐原声作品** | `crawler.py` type = `"music"` |
| **关键词搜索作品** | `crawler.py` type = `"search"` |
| **关注 / 粉丝列表采集**（仅 CLI 模式） | `crawler.py` type = `"following"` / `"follower"` |

加上它已有的：`aweme`(单作品) / `post`(主页) / `favorite`(喜欢) / `collection`(收藏) / `mix`(合集) —— **共 10 种采集类型，是四个候选里最全的**，且有 `results_old` 做的智能增量。

**但**：它有 4 个硬伤，详见 [douyin-erma0.md](douyin-erma0.md) —— 分别是公网 80 暴露、
`VITE_API_BASE_URL` 构建期烘焙、**Dockerfile 未装 aria2（下载功能直接不可用）**、
以及 `pywebview[gtk]` 在 slim 镜像中**构建失败（已实测复现）**。

---

## 4. 逐需求打分（下载层视角）

| 需求 | A | C | 说明 |
|---|---|---|---|
| 抖音主页全量 | ✅✅ | ✅ | A 的增量/去重/统计更强 |
| 抖音**评论采集** | ✅ | ❌ | A 独有 |
| 抖音话题/音乐原声/关键词搜索 | ⚠️ 需靠 douyin_erma0 补 | ⚠️ 同 | 见 §3.1 |
| B站 UP 主全量 | ✅ | ✅ | 打平 |
| B站 收藏夹 | ❌ | ✅ | C 独有 |
| B站 课程(cheese) | ✅ | ❌ | A 独有 |
| 公众号全量历史 | ✅ | ✅ | **同一个 wae 组件** |
| 小红书 / 快手 / 视频号 | ❌ | 🟠 受限 | C 独有但受限 |
| 「丢链接就下载」 | ✅ Web API（需自建调用） | ✅ Web UI 直接贴 | C 交互更好 |
| **定时增量同步** | ❌ | ✅ | C 独有 |
| **统一下载历史** | ❌ | ✅ | C 独有 |
| 容器数量 | 3 | 2 | C 少 |
| 内存 | 未实测，3 容器 | 目标 150–250 MiB | C 低 |
| **长期维护风险** | 🟢 分散在 3 个高星项目 | 🔴 186★ 单点 | **A 优** |
| 浏览器依赖 | ❌ 全无 | ❌ 剥离后无 | 平 |

**计分：C 在 13 项里领先，A 在 3 项领先（抖音深度 2 项 + 维护风险 1 项）。**

---

## 5. 建议方案：路线 C+

既然差异是结构性的（不是「谁全面更好」，而是「C 广、A 深」），最优解不是二选一，而是**以 C 为主干、按缺口补 A 的组件**：

```
路线 C+ = C 的全部内容 + 以下 2 个可选补充
```

| 补充 | 何时加 | 成本 | 触发条件 |
|---|---|---|---|
| **+ TikTokDownloader** | 抖音是主力平台、需要评论/统计/剪贴板/多账号 | 官方镜像，1 容器，共享 `/opt/mp-tools/data/` | Phase 1 之后按需 |
| **+ douyin_erma0** | 需要话题挑战/音乐原声/关键词搜索/关注粉丝 | 需先修 `pywebview` 构建（见 [douyin-erma0.md](douyin-erma0.md)） | 可选，优先级低 |
| ~~+ yutto~~ | — | — | **不建议**：B站功能已被 mp-tools 打平，FFmpeg 硬依赖，无收益 |

> ⚠️ 上表所有补充项都应**在 Phase 1–5 跑通之后再评估**，不要一开始就堆容器 —— 1.9 GiB 是本项目的第一约束（见 [feasibility.md](feasibility.md) §1）。

---

## 6. 复核命令

```bash
git clone --depth 1 https://github.com/yutto-dev/yutto.git && cd yutto
grep -rn 'ffmpeg' src/yutto/ --include='*.py' | head          # 硬依赖证据
ls src/yutto/api/                                            # bangumi/cheese/collection/space/...，无 fav
grep -ril 'fastapi\|flask\|uvicorn\|webui\|gradio' --include='*.py' .   # 0 命中 = 纯 CLI

git clone --depth 1 https://github.com/JoeanAmier/TikTokDownloader.git
grep -n '合集\|收藏\|评论\|增量\|剪贴板\|Web API' README.md

git clone --depth 1 https://github.com/x554960766/wechat-mp-tools.git
grep -n '^def \|^    def ' backend/bilibili.py                 # 完整功能清单
grep -n '^    def ' backend/douyin.py                         # 抖音能力面
grep -rn 'playwright\|browser\|chromium' backend/bilibili.py backend/articles.py  # 0 命中
sed -n '85,110p' /tmp/douyin_erma0/backend/lib/douyin/crawler.py   # 10 种 type 分支
```

---

## 7. 相关文档

| 文档 | 内容 |
|---|---|
| [../VISION.md](../VISION.md) | 愿景速览（近期/远期目标、可扩展性四条硬定义） |
| [vision.md](vision.md) | **【需求源头】** 愿景原话拆解、S1–S6 场景约束 —— 本文回答的正是 S3（范围可选） |
| [../PLAN.md](../PLAN.md) | 项目规划与实施计划（ADR 决策、Phase 0–11、风险登记册） |
| [roadmap.md](roadmap.md) | 三条路线（A / B / C）定义、需求打分、决策矩阵 |
| [feasibility.md](feasibility.md) | 资源硬约束、组件源码级核实、3 条错误说法纠正 |
| [mp-tools-docker.md](mp-tools-docker.md) | mp-tools Docker 化：浏览器依赖逐文件实测、三种部署方案 |
| [douyin-erma0.md](douyin-erma0.md) | douyin_erma0 完整调研与**四个坑**的修复方案 |

> 路线定义提醒：本文的「路线 A」= know-where 四件套方案（know-where + TikTokDownloader + yutto + wechat-article-exporter）；「路线 B」= wechat-mp-tools 单体原样部署；「路线 C」= mp-tools 精简版 + wae 补位 + PC 分析。定义见 [roadmap.md](roadmap.md) §1。