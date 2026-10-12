# 个人知识归档系统 —— 可行性分析

> 目标：自托管服务，丢入链接 → 自动抓取 **微信公众号文章正文** / **B站·抖音视频** → 下载媒体 →
> AI 摘要与分类 → 归档；支持「一个主页 → 全部下载」；可选挂飞书机器人。
> 部署目标机：单台 Linux VPS，Ubuntu 22.04，**2 vCPU / 1963 MiB RAM / 49 GiB 磁盘（已用 15G，余 34G）/ 无 GPU**。
>
> **⚠️ 本文是「路线 A」的完整可行性分析，是 2026-10-12 上午的选型过程记录。最终已改用路线 C**（mp-tools 下载层 + PC 分析层），决策见 [../PLAN.md](../PLAN.md) ADR-001。
>
> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) 愿景速览 ｜ [vision.md](vision.md) 需求源头 ｜ **[execution-plan.md](execution-plan.md)施工单** ｜ **[environment.md](environment.md)环境事实与铁律** ｜ [../PLAN.md](../PLAN.md) 规划 ｜ [roadmap.md](roadmap.md) 三条路线 ｜ [download-layer-comparison.md](download-layer-comparison.md) 下载层逐条对比 ｜ **本文** 可行性与组件核实 ｜ [mp-tools-docker.md](mp-tools-docker.md) Docker 化 ｜ [douyin-erma0.md](douyin-erma0.md) 第四候选
>
> 原始未脱敏版本：`../../docs/knowledge-archive-feasibility.md`（母仓库内，**勿外传**）

---

## 0. 结论先行

**可行，但需要拼装，不是装一个软件就能搞定。** 现实的最优解是三个现成项目的组合（下称**路线 A**）：

| 能力 | 选型 | 成熟度 |
|---|---|---|
| AI 归档中枢 + 飞书机器人 | [ybh-ybh/know-where](https://github.com/ybh-ybh/know-where)（知归） | 104★，2026-10-05 |
| 抖音「主页 → 全部下载」 | [JoeanAmier/TikTokDownloader](https://github.com/JoeanAmier/TikTokDownloader) | **16703★**，2026-09-22 |
| B站「UP主 → 全部下载」 | [yutto-dev/yutto](https://github.com/yutto-dev/yutto) | 2064★，2026-10-11 |
| 公众号「历史文章 → 全量」 | [wechat-article/wechat-article-exporter](https://github.com/wechat-article/wechat-article-exporter) | **13054★**，2026-09-30 |

know-where 负责「链接进来 → 提取正文 → ASR 转录视频音频 → LLM 摘要分类 → 写飞书多维表格」；
TikTokDownloader 负责「抖音主页批量落盘成文件」，两者通过共享目录衔接。

---

## 1. 本机资源实测（决定了什么能做、什么不能做）

```
Mem:   1963 MiB total / 804 used / 975 available
Swap:  2047 MiB (197 used)
Disk:  /dev/vda1 49G  15G used  34G available (30%)
CPU:   2 vCPU, load 1.73
现有 10 个容器内存合计 ≈ 634 MiB（warpgate 342M / agent2api 104M 占大头）
ffmpeg: 未安装    chromium/playwright 缓存: 无
```

### 1.1 本地跑 LLM 做摘要 —— ❌ 不现实

| 模型 | 权重 | + KV cache | 峰值 | 叠加容器 400–600MB | 结论 |
|---|---|---|---|---|---|
| Qwen2.5-0.5B Q4 | 0.4 GB | ~0.2 GB | 0.7–0.9 GB | 1.1–1.5 GB | 能启动，中文摘要质量不可用 |
| Qwen2.5-1.5B Q4 | 1.0–1.1 GB | 0.3–0.5 GB | 1.4–1.6 GB | 1.8–2.2 GB | **超过 1963 MiB，必 OOM** |

2 vCPU 上 1.5B CPU 推理约 3–6 tok/s，一篇 3000 字文章的 300 token 摘要要 **1–3 分钟**，且与抓取任务抢同一份内存。

**→ 必须走外部 LLM API。** know-where 默认就是 DeepSeek（`KW_LLM_BASE_URL=https://api.deepseek.com/v1`、`KW_LLM_MODEL=deepseek-chat`），成本远低于扩容 VPS。

### 1.2 无头浏览器 —— ⚠️ 尽量避免

Chromium 常驻 400–700 MiB。在 975 MiB available 的机器上塞一个 Chromium 会把 OOM 风险变成必然。

**好消息：know-where 和 TikTokDownloader 都不依赖浏览器**（见 §3、§4），这是本次选型的首要判据。

### 1.3 ASR（语音转录）—— ⚠️ 走腾讯云

know-where 的 `KW_ASR_PROVIDER` 二选一：`faster_whisper`（本地 CPU，会装 PyTorch + faster-whisper，镜像和内存都撑不住）或 `tencent`（腾讯云录音文件识别）。本机必须用 `tencent`，且 `KW_TEMP_STORAGE_PROVIDER=tencent_cos`（临时音频传 COS，ASR 拉预签名 URL）。

---

## 2. 关于那段聊天记录 —— 有三条是错的，我逐条核实了

不能直接采信，我 clone 了仓库逐行读代码后修正如下。

| 聊天记录的说法 | 实际核实结果 |
|---|---|
| 「f2 B站支持在 0.0.1.8 版加入」 | ❌ **不成立**。`f2/apps/` 下只有 `bark/ douyin/ tiktok/ twitter/ weibo`，无 bilibili 模块；bilibili 仅出现在 `__apps__.py` 文档字符串和翻译文件里 |
| 「know-where 走 wechat2rss / RSSHub 抓公众号」 | ❌ **不成立**。`grep -rn 'rsshub\|wechat2rss\|wechat-mp' src/` **0 命中**。它有自带的 `src/knowwhere/adapters/wechat.py`，纯 httpx + BeautifulSoup 解析 `#js_content`，**无 cookie、无浏览器** |
| 「know-where 单容器、SQLite、不需要 PG」 | ❌ **不成立**。`pyproject.toml` 依赖含 `psycopg[binary]`，`database.py` 有 `from sqlalchemy.dialects.postgresql import JSONB`，`.env.example` 是 `KW_DATABASE_URL=postgresql+psycopg://...` → **PostgreSQL 是硬依赖，多一个容器** |

另有一处「子调研报告」称 know-where 用 Playwright + Chromium + `CONFIG_AI_PROVIDER` + `ast-shanghai-embedding` 向量索引 —— 这些在我 clone 的仓库里**全部不存在**：`grep -in 'playwright|chromium' Dockerfile pyproject.toml` 0 命中，`.env.example` 变量前缀统一是 `KW_*`，无 `CONFIG_AI_PROVIDER`。**该报告不可采信。**

---

## 3. 归档中枢：know-where（知归）

```yaml
# https://api.github.com/repos/ybh-ybh/know-where （实测）
stars: 104   license: Apache-2.0   language: Python+TS
created_at: 2026-08-28   pushed_at: 2026-10-05
```

- **规模**：48 个 py，2.3 MB，`Dockerfile` + `compose.yml`（2 个 service：`app`、`gateway`，均需 `build: context: .` 本地构建，无官方预构建镜像）
- **数据库**：**PostgreSQL（必需）**，alembic 迁移 `0001_initial.py` / `0002_douyin_media.py`，含 JSONB 列
- **浏览器依赖：无** ← 关键优势

### 3.1 适配器（`src/knowwhere/adapters/`）

```
wechat.py  bilibili.py  douyin.py  xiaohongshu.py  juejin.py  github.py
article_router.py  feishu_bitable.py  llm_openai.py  glm_vision.py
cos_artifact_store.py  local_temp_storage.py  faster_whisper_asr.py  tencent_asr.py
```

| 平台 | 实现机制（实测代码） | 凭据 |
|---|---|---|
| 微信公众号 | `wechat.py:94` 只接受 `https://mp.weixin.qq.com`；httpx GET → `soup.select_one("#js_content")` → 去掉 script/style → 正文 ≥200 字才接受；标题取 `og:title` / `#activity-name`；时间戳取页面 `ct` | **无需 cookie** |
| B站 | `bilibili.py` 纯 httpx，手工实现 WBI 签名（`_build_mixin_key` / `_sign_wbi` / `_build_device_params`），调 `x/web-interface/view` + `x/player/wbi/playurl`，再 `_extract_best_audio` 取 DASH 音频 | 未用 SESSDATA 时拿不到 1080p+ |
| 抖音 | `douyin.py` 纯 httpx，先 `_create_ttwid()`（`ttwid.bytedance.com/ttwid/union/register/`）拿 cookie，再打 `aweme/v1/web/aweme/detail/` | 自建 ttwid，无需登录 |
| 小红书 | `xiaohongshu.py` | `KW_XIAOHONGSHU_COOKIE` 可选，留空匿名 |

### 3.2 飞书是硬依赖，架构上不可剥离

输入必须是飞书机器人消息，产物必须写入飞书多维表格（`KW_FEISHU_APP_ID` / `KW_FEISHU_APP_SECRET` / bitable token / table id）。没有 Notion/Obsidian/纯 SQLite 导出路径。

**这点要提前确认：如果你没有飞书自建应用，这个方案的 60% 价值就没了。**

### 3.3 它不做「视频下载」

`KW_TEMP_DELETE_AFTER_PROCESS=true`（默认）—— 媒体只是转录的中间产物，**处理完就删**。视频/音频文件不会归档。要留文件得改这个开关 + 接 `cos_artifact_store`。

**所以「视频下载下来」这件事 know-where 干不了，得靠 §4 的下载器落盘。**

---

## 4. 抖音：TikTokDownloader（唯一的赢家）

```yaml
# https://api.github.com/repos/JoeanAmier/TikTokDownloader （实测）
stars: 16703   forks: 2808   license: GPL-3.0   language: JavaScript/Python
pushed_at: 2026-09-22
官方镜像: joeanamier/tiktok-downloader
topics: docker, ffmpeg, sqlite, httpx, api, server
```

README 实测能力（逐条对应你的需求）：

- ✅ **下载抖音账号发布/喜欢/收藏/收藏夹作品** ← 就是你要的「主页全部下载」
- ✅ 批量下载链接作品　✅ 多账号批量下载作品
- ✅ 下载抖音合集作品　✅ **支持账号作品增量下载** ← 适合长期归档
- ✅ **记录已下载作品 ID**　✅ 多线程下载
- ✅ **监听剪贴板链接下载作品** ← 就是你要的「丢链接进去」
- ✅ 按文件夹归档保存作品文件
- ✅ 采集评论数据　✅ 筛选发布时间　✅ 自定义规则筛选

**依赖实测**：`requirements.txt` = `curl-cffi`（签名）、`fastapi`/`uvicorn`（自带 HTTP API，端口 5555）、`aiosqlite`、`lxml`、`pydantic`、`rich`、`openpyxl`。
`grep -il 'playwright|selenium|pyppeteer'` → **0 命中，无浏览器依赖**。镜像带 ffmpeg。

**结论：抖音这一半，TikTokDownloader 是完美匹配，且能直接跑在这台机器上。**

---

## 5. B站：yutto（首选，无浏览器、依赖极轻）

```yaml
# https://api.github.com/repos/yutto-dev/yutto （实测）
stars: 2064   forks: 175   license: GPL-3.0   language: Python
created_at: 2021-05-02   pushed_at: 2026-10-11
官方镜像: siguremo/yutto    pip: pip install yutto
```

**源码实证（我 clone 后核对）**：

- `src/yutto/api/space.py:22` `get_user_space_all_videos_avids(scope, mid, ...)` → `space_videos_api = "https://api.bilibili.com/x/space/wbi/arc/search"`，`ps=30` 翻页拉全，**WBI 签名库内自签**（`encode_wbi(params, wbi_img)`，来自 `api/user_info.py`）。
- `src/yutto/auth.py:29-30` `class AuthInfo`：`SESSDATA: str` / `bili_jct: str | None`，`format_auth_inline(sessdata, bili_jct)` 支持多 profile。
- **浏览器依赖：无。** `pyproject.toml` 依赖只有 8 个：`biliass`、`colorama(win32)`、`typing-extensions`、`dict2xml`、`pydantic`、`returns`、`segno`、`websockets`。
  - ⚠️ **勘误（2026-10-12）**：本节原先写「连 ffmpeg 都不需要（biliass 负责弹幕 ASS 烧录）」，**这是错的**。实测查证：`README.md:99` 明写「请确保安装 Python3.11 及以上版本，并**配置好 FFmpeg**」；`src/yutto/utils/ffmpeg.py:18` `FFMPEG_PATH = "ffmpeg"`；`src/yutto/__main__.py:23` 无条件 `from yutto.utils.ffmpeg import FFmpeg`；`src/yutto/validator.py:118/139` 用 `ffmpeg.video_encodecs` / `audio_encodecs` 校验用户设定的编码。**FFmpeg 对 yutto 是硬依赖**（用于分 P 合流与编码转换，不是弹幕）。详见 [download-layer-comparison.md](download-layer-comparison.md) §2.2。

**这是本次选型里最适合这台 VPS 的组件**：几乎零内存开销。

### 5.1 B站 1080p+ 与 Cookie

SESSDATA 直接决定画质档位——文档中「1080P 高码率 AVC/HEVC/AV1」属登录大会员项。
**设计建议：把 Cookie 做成可选输入**，无 Cookie 时降级下载，不要硬性必填。
（无 Cookie 时普通 1080P 档能否拿到——未实测，需探针验证。）

### 5.2 备选

| 项目 | Star | 状态 |
|---|---|---|
| `ScottSloan/Bili23-Downloader` | 8296 | push 2026-10-02，但 GUI 血统，Linux 部署麻烦 |
| `yt-dlp` | — | README 有 `#### bilibili` extractor，可做兜底 |
| `nilaoda/BBDown` | — | master 分支只有 LICENSE+README，.NET 应用，代码在 release，**不建议做核心依赖** |
| `yutto-dev/bili`（旧包） | 1173 | 停更于 2024-10-26，**用 yutto 不用它** |

### 5.3 ⚠️ 法律风险：bilibili-API-collect 已因律师函停更

`SocialSisterYi/bilibili-API-collect`（20173★）实测状态：`archived: true`、`default_branch: "deprecated"`、`pushed_at: 2026-01-30`、`description: null`、`language: null`。

社区说明：2026-01-28 收到 B 站委托律所律师函，认定其「系统性收集、整理并向不特定公众传播」非公开 API 构成侵权，即日起停维护。

**→ 不要把它当作 B站 API 的依据来源。** 好在 yutto 自己实现了 WBI 签名，不依赖它。

---

## 5.4 抖音 Cookie 说明

公开主页作品**不必须 cookie**。f2 的 API 表把「主页作品 `fetch_user_post_videos`」「合集作品 `fetch_user_mix_videos`」都标为 🟢/⚫「无需登录即可下载他人公开作品」；签名侧 `TokenManager.gen_real_msToken` / `XBogusManager.str_2_endpoint` / `ABogusManager.str_2_endpoint` / `SecUserIdFetcher.get_sec_user_id` **全部库内自签**，不依赖外部服务。

Cookie 只在**私密账号作品 / 自己收藏点赞 / 风控降级**时才需要。
⚠️ **未实测**：2026 现网风控是否已收紧到「无 cookie 拿不到」——这是最大不确定性，建议第一周写探针实测。

---

## 6. 公众号全量批量：全网项目实盘（GitHub Search 39 条，实测前 5）

### 6.1 全量检索结果

GitHub Search API `q=公众号 批量下载&sort=stars`，`total_count=39`。**只有两个项目上四位数星。**

| 项目 | Star | 最后 push | 形态 | 能在无头 Linux VPS 跑？ |
|---|---|---|---|---|
| [wechat-article/wechat-article-exporter](https://github.com/wechat-article/wechat-article-exporter) | **13054** | 2026-09-30 | Nuxt 全栈，**有 Dockerfile** | ✅ **可以** |
| [qiye45/wechatDownload](https://github.com/qiye45/wechatDownload) | **9731** | 2026-10-07 | 桌面客户端（Windows/macOS） | ❌ 不行 |
| [x554960766/wechat-mp-tools](https://github.com/x554960766/wechat-mp-tools) | 186 | 2026-10-10 | 桌面客户端（Win/mac 双平台） | ❌ 不行 |
| [yangbuyiya/EasyWechatDownload](https://github.com/yangbuyiya/EasyWechatDownload) | 84 | 2026-07-14 | HTML 工具 | ❌ |
| [yibeigen/wechat-article-exporter](https://github.com/yibeigen/wechat-article-exporter) | 35 | 2026-10-10 | Python | ⚠️ 待验 |
| 其余 34 个 | ≤34 | — | 杂 | ❌ 玩具级 |

### 6.2 为什么 9731★ 的 wechatDownload 不能用

实测 clone：仓库根目录只有 `README.md`、`example/`（示例输出含封面/视频）、`skills/`、`.github/`。
**`find . -name '*.py' | wc -l` = 0，没有源码目录，没有 Dockerfile。**
README 自述：「一款可以批量下载微信公众号文章内容的小工具，**无需安装证书，支持 Window、MacOS**」，更新日志里全是「获取密钥」「适配微信 4.0」「监听剪切板」「修复获取不到密钥」——这是**Hook 微信 PC 客户端内存**的桌面工具。功能确实强（评论、合集、历史消息、html/mhtml/md/pdf/docx/csv、图/视频/音频、贴图号），还有 MCP/Skill 调用接口。

**但它在 Linux 服务器上无法运行**，只能在你本地 Windows/Mac 上跑，然后把文件同步到服务器。许可证字段为空（API `license: null`）。

### 6.3 wechat-mp-tools：功能最全，但是客户端

README 标题即「微信公众号 / 视频号 / 抖音 / 快手 / 小红书 / 哔哩哔哩 下载与转码工具箱」，**功能覆盖你全部需求**，而且很新（2026-10-10 更新）：
- 公众号：历史文章列表完整获取、时间范围筛选、分页、断点续传、HTML/Markdown/PDF、**内置 RSS 2.0 服务**、后台定时增量同步
- B站：UP 主空间投稿采集、FFmpeg 混流、弹幕 ASS、CC 字幕 SRT/ASS
- 抖音：作者主页全作品批量、增量去重、合集、直播回放、双通道风控自愈
- 小红书 / 快手 / 微信视频号博主作品批量

但仓库结构是 `windows/`、`mac/`、`injection_scripts/`、`wechat_mp_tools.spec`（PyInstaller 打包）、GitHub Actions 自动构建双平台客户端 → **是 GUI 客户端，不是服务**。README 还说「很多同类开源项目已陆续闭源转向收费」。

**唯一值得注意的**：它内置了标准 **RSS 2.0 服务** + 定时增量同步。如果哪天它开放服务端模式，会是最贴合归档需求的方案。

### 6.4 唯一能在本机跑的：wechat-article-exporter

```yaml
# https://api.github.com/repos/wechat-article/wechat-article-exporter （实测）
stars: 13054   forks: 2275   license: MIT   language: TypeScript (Nuxt 3)
created_at: 2024-07-17   pushed_at: 2026-09-30   open_issues: 47
Dockerfile: node:22-alpine 构建 → node:22-slim 运行，NITRO_KV_DRIVER=fs
```

**功能（README 特性列表）**：搜索公众号、导出 html/json/excel/txt/md/docx（html 打包图片和样式 100% 还原）、文章列表缓存、按作者/标题/发布时间/原创/合集过滤、合集下载、图片分享消息、视频分享消息、导出评论+回复+阅读量+转发量（需抓包 credentials）、Docker 部署、Cloudflare 部署。

**关键：它的「批量」原理（README「原理」章节原文）**

> 在公众号后台写文章时支持搜索其他公众号的文章功能，以此来实现抓取指定公众号所有文章的目的。

配套代码印证：`server/api/web/login/bizlogin.post.ts`、`getqrcode.get.ts`、`scan.get.ts` —— **扫码登录你自己的公众号后台**，然后借用后台编辑器的「搜索其他公众号」能力来枚举目标号的历史文章。

README 还郑重声明：「不会利用您扫码登录的公众号进行任何形式的私有爬虫，也不存在类似账号池的东西。您的公众号只会服务于您自己的抓取文章的目的。」

⚠️ **在线网站域名 2026-10-30 到期**，届时 `down.mptext.top` 不可访问，只能自己 Docker 部署。

**→ 门槛：必须有一个能扫码登录的、你自己名下的公众号。**
（个人订阅号是否具备后台「搜索其他公众号文章」能力——README 未说明，**未核实**，建议实机验证。）

---

## 7. 建议架构

```
                 ┌──────────── 飞书机器人（唯一入口）────────────┐
                 │  丢链接：公众号文章 / B站视频 / 抖音视频      │
                 └───────────────────┬─────────────────────────┘
                                     ▼
        ┌────────────────────────────────────────────────┐
        │  know-where（app + gateway + PostgreSQL）      │
        │  纯 httpx 提取正文 → LLM 摘要分类 → 飞书多维表格 │
        │  零浏览器依赖                                      │
        └──────────┬──────────────────────────┬──────────┘
                   │ 需要音频转录              │ 需要媒体落盘
                   ▼                          ▼
        腾讯云 ASR (+ 腾讯 COS)      ┌──────────────────────────────┐
                                     │ TikTokDownloader  抖音       │
                                     │ 主页/合集/链接批量, 增量下载   │
                                     │ Docker 官方镜像，无浏览器     │
                                     ├──────────────────────────────┤
                                     │ yutto  B站                    │
                                     │ UP主空间/收藏夹/合集批量        │
                                     │ 8 个轻依赖，无浏览器无 ffmpeg  │
                                     └──────────┬───────────────────┘
                                                ▼
                                    /opt/kb-archive/media/
                                    （与飞书表格按标题对齐）
```

选型原则：**优先无浏览器依赖的组件**，因为这台机器 available 只有 975 MiB。
本方案的三个核心组件（know-where / TikTokDownloader / yutto）**全部无浏览器依赖** —— 这是刻意筛选的结果。

### 7.1 备选：一体化方案 Evil0ctal/Douyin_TikTok_Download_API

20593★ · Apache-2.0 · push 2026-10-02 · async REST API + MCP + Web 控制台 + PostgreSQL 归档 + 自愈身份池（无头浏览器铸游客身份）。
⚠️ **坑**：README 明确 v5(main) **砍掉了 B站**，B站只在 v4（`evil0ctal/douyin_tiktok_download_api:V4.1.2`）。且自愈身份池用无头浏览器，与本机资源冲突。**备而不选。**

---

## 8. 风险清单

| 风险 | 等级 | 说明 |
|---|---|---|
| 本地 LLM 不可行 | 🔴 已确认 | 必须外部 API（DeepSeek），产生按 token 费用 |
| 飞书硬依赖 | 🔴 | 无飞书自建应用则方案要重做 |
| bilibili-API-collect 因律师函停更 | 🔴 | 不可作为 B站 API 依据；yutto 自签 WBI 不受影响 |
| 公众号全量批量需凭据 | 🟠 | 单篇无凭据可行，主页全量必须自有号/付费实例 |
| 内存紧张 | 🟠 | 975 MiB available；新增 3 容器需逐一压测，可能要开更大 swap |
| 磁盘 34 GiB | 🟡 | 视频批量下载会吃很快，需配 `KW_TEMP_DELETE_AFTER_PROCESS` 和定期清理 |
| 抖音/B站 反爬 | 🟡 | 接口会变，需跟随上游更新 |
| 站点风控/IP | 🟡 | 服务器在海外；如遇验证码需住宅代理（本机已有 WARP 池 10010–10025 可用） |

---

## 9. 待你确认的三件事

1. **有没有飞书自建应用？** 没有的话 know-where 路线要重新设计（可退而求其次：只跑下载器 + 本地 SQLite + Markdown 归档）。
2. **抖音/B站的 Cookie 你手上有吗？** 决定能否拿 1080p+ 和绕过风控。
3. **视频文件要不要长期留？** know-where 默认删中间产物；TikTokDownloader 落盘。按需决定存储规划。

---

## 附录：本次核实方式

全部结论均来自亲自 clone 到 VPS 后读源码，或直接查 GitHub API，**未采信二手描述**。

已核实的仓库：`douyin_erma0`、`videodl`、`know-where`、`TikTokDownloader`、`f2`、`BBDown`、`yutto`、`wechatDownload`、`wechat-mp-tools`、`wechat-article-exporter`。

---

## 相关文档（双向导航）

| 文档 | 内容 | 关联本文哪节 |
|---|---|---|
| [../README.md](../README.md) | 项目首页 | — |
| [../VISION.md](../VISION.md) | 愿景速览（近期/远期目标、可扩展性硬定义） | §0 |
| [vision.md](vision.md) | **【需求源头】** 愿景原话拆解、S1–S6 场景约束 | §0 / §7 |
| [../PLAN.md](../PLAN.md) | 项目规划与实施计划（ADR-001 改用路线 C、ADR-005 kb-hub） | — |
| **[execution-plan.md](execution-plan.md)** | **施工单**：本文的资源约束如何变成每个 Phase 的验收标准 | **§1 硬约束** |
| **[environment.md](environment.md)** | **环境事实与五条铁律** | **§1 资源硬约束** |
| [roadmap.md](roadmap.md) | **三条路线 A/B/C 的定义与决策** | 全文背景 |
| [download-layer-comparison.md](download-layer-comparison.md) | 下载层功能逐条对比 + 四个候选功能矩阵 + **抖音二选一专项** + 路线 C+ | §2 B站、§4 yutto 勘误 |
| [mp-tools-docker.md](mp-tools-docker.md) | 路线 B 的 Docker 化调研（浏览器依赖逐文件实测） | §8 缺失平台 |
| [douyin-erma0.md](douyin-erma0.md) | 第四候选 douyin_erma0 完整调研与四个坑 | 附录 |
| `../../docs/knowledge-archive-feasibility.md` | **本文原始未脱敏版本**（母仓库内，勿外传） | — |