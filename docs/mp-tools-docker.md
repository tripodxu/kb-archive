# wechat-mp-tools 上 VPS 调研报告与实施计划

> 调研对象：[x554960766/wechat-mp-tools](https://github.com/x554960766/wechat-mp-tools)
> 仓库快照：`wechat-mp-tools`，174 个 py，5.3M，last push 2026-10-10，186★，MIT License (c) 2026 xuyi
> 调研日期：2026-10-12
> 调研方式：`git clone --depth 1` 到 VPS 后逐文件读源码，所有结论均标注文件行号作为证据
>
> **⚠️ 本文是「路线 B」的调研记录。最终采用的「路线 C」= 本文的方案 B（砍浏览器精简版） + wechat-article-exporter 补位 + PC 分析层**，决策见 [../PLAN.md](../PLAN.md) ADR-001。
>
> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) 愿景速览 ｜ [vision.md](vision.md) 需求源头 ｜ **[execution-plan.md](execution-plan.md)施工单** ｜ **[environment.md](environment.md)环境事实与铁律** ｜ [../PLAN.md](../PLAN.md) 规划 ｜ [roadmap.md](roadmap.md) 三条路线 ｜ [download-layer-comparison.md](download-layer-comparison.md) 下载层逐条对比 ｜ [feasibility.md](feasibility.md) 可行性 ｜ **本文** mp-tools Docker 化 ｜ [douyin-erma0.md](douyin-erma0.md) 第四候选
>
> 原始未脱敏版本：`../../docs/wechat-mp-tools-vps-plan.md`（母仓库内，**勿外传**）

---

## 0. 结论先行

| 问题 | 答案 |
|---|---|
| 能 Docker 化吗？ | **能，而且比想象容易。** 它本质是 Flask Web 应用，桌面感只是 PyInstaller 壳 |
| 能否跑在 1.9GiB VPS 上？ | **能，但有条件。** B站/抖音主链路不需要浏览器；快手/小红书/视频号需要 Chromium |
| 能否做成「下载在 VPS、分析在 PC」？ | **不仅可以，而且这是它最合理的用法。** 它本身没有任何 AI 分析能力，下载/分析天然分离 |
| 最大障碍 | Playwright Chromium 常驻 400–700 MiB；登录流程全是 `headless=False` 需要能看见的窗口 |
| 最大功能缺口 | **公众号批量下载在 Linux 上大概率不可用**（依赖注入微信 PC 客户端），仍需 wechat-article-exporter 补位 |

---

## 1. 关键发现：它不是「桌面应用」，是「Web 应用 + 桌面壳」

一开始我判定它是 GUI 客户端、不能上服务器。**读了 `app.py` 之后这个判断是错的**，必须更正。

```python
# app.py:41-75
from flask import Flask, send_from_directory
from flask_cors import CORS
from backend.runtime import configure_runtime, resource_dir
app = Flask(__name__, static_folder=str(static_folder_path), static_url_path="")
CORS(app)
```

```python
# app.py:158-184  ← 作者本来就写了 host 参数
def main():
    parser = argparse.ArgumentParser(description="微信公众号文章下载管理工具")
    parser.add_argument("--port", type=int, default=5200, help="服务端口 (默认 5200)")
    parser.add_argument("--host", type=str, default="127.0.0.1", help="监听地址")
    parser.add_argument("--no-browser", action="store_true", help="不自动打开浏览器")
    ...
    app.run(host=args.host, port=args.port, debug=args.debug, threaded=True)
```

`--host` 参数的存在说明作者本身就考虑过非本机访问。桌面感来自三处，**都和服务器运行无关**：

1. `app.py:151-156` `open_browser()` 启动后自动 `webbrowser.open()`
2. `wechat_mp_tools.spec` —— PyInstaller 打包配置
3. `.github/workflows` 自动构建 Windows / macOS 双平台安装包

注册的 17 个 Flask Blueprint（`app.py:78-94`）：

```
auth_bp  accounts_bp  articles_bp  proxy_bp  account_pool_bp
douyin_bp  douyin_login_bp  douyin_auth_bp
kuaishou_bp  kuaishou_auth_bp  channels_bp  transcode_bp
xhs_bp  xhs_login_bp  bilibili_bp  bilibili_login_bp  updater_bp
```

还有后台 RSS 调度器（`app.py:101-103` `rss_scheduler.start()`），这是定时增量归档的关键，**它跟 GUI 无关，服务器上照样能跑**。

**→ Docker 化在原理上完全成立。**

---

## 2. 四个 Linux 障碍（逐个实测）

### 障碍 1：Playwright + Chromium（决定性因素）

`requirements.txt` 含 `playwright>=1.40`、`scrapling>=0.2`。

我逐文件 grep 了全部 139 个 backend py，结果如下：

| 模块 | 浏览器是否必需 | 源码证据 |
|---|---|---|
| **B站** | ❌ **完全不用** | `backend/bilibili.py` 中 playwright/browser/chromium **0 命中**。`get_playurl`(:314)、`get_collection_details`(:438)、`xml_to_ass` 弹幕转 ASS(:122)、`json_to_srt` 字幕转 SRT(:217) 全是纯 HTTP |
| **抖音** | ❌ **主路径纯 HTTP** | `backend/douyin.py:3` 文件头注释原文：「移植自 douyin-downloader-rust 项目，使用**直接 HTTP API 调用代替 Playwright 渲染**」。配 `backend/douyin_sign.py` + `douyin_mstoken.py` 库内自签。浏览器只在兜底路径 `_fetch_via_browser`(douyin.py:417) 和 `_fetch_liked_via_browser`(douyin.py:474) |
| **快手** | ✅ **必需** | `backend/kuaishou.py:401 get_user_feed_via_browser` 注释：「通过后台 Chromium 无头浏览器获取用户作品列表，**突破前端 JS 签名校验 (result=50)**」。:553 和 :567 两处无条件 fallback 到浏览器 |
| **小红书** | ✅ **必需（有降级）** | `backend/xiaohongshu.py:797`「浏览器拦截获取笔记列表失败」→ :800-808 降级为「主页首屏解析」。`_XhsBrowserSession`(:854) 用 Playwright + 物理滚动 |
| **微信视频号** | ✅ **必需 + 必须有头** | `backend/channels.py:293` `launch_chromium(p.chromium, headless=False)` —— 有头模式，配 `backend/mitm_proxy.py` MITM 抓包 |
| **微信读书** | ⚠️ 保活 | `backend/weread_browser.py:120` headless Chromium 换 Cookie。`:19-20` 有全局单实例串行锁，注释明写「确保同一时刻系统只启动 1 个 Headless Chromium 实例，**节省内存**」 |
| **公众号文章** | ❌ | `backend/articles.py` 中 playwright/browser **0 命中**。但 `wechat_mp_batch_downloader.py` 命中 playwright |

**内存参数现状**：作者显然踩过内存坑，但只加了一半——
- ✅ `backend/downloader.py:393 --disable-gpu`
- ✅ `backend/weread_browser.py:139-141 --no-sandbox --disable-dev-shm-usage --disable-gpu`
- ❌ 没有 `--single-process` / `--renderer-process-limit`
- ❌ 没有全局 Chromium 单例（只有 weread 有锁，douyin/kuaishou/xhs 各拉各的）

**内存账**：headless Chromium 常驻 400–700 MiB。本机 `Mem 1963 MiB total / 975 MiB available`，现有 10 个容器占 634 MiB（warpgate 342M、agent2api 104M）。装得下但很紧，且 OOM 会连带打死在跑的业务容器。

### 障碍 2：登录流程全部 `headless=False`

```
backend/douyin_login.py:232        launch_chromium(p.chromium, headless=False, args=['--disable-blink-features=AutomationControlled'])
backend/xiaohongshu_login.py:46    launch_chromium(p.chromium, headless=False, args=['--disable-blink-features=AutomationControlled'])
backend/kuaishou_auth.py:160       headless=False,
```

抖音、小红书、快手三个平台的登录都要**弹出一个可见窗口让用户扫码**。容器里必须配 Xvfb + 截图/二维码导出/或 noVNC 远程看窗口。这是 Docker 化最主要的工程量。

### 障碍 3：mitmproxy（只影响视频号）

`requirements.txt` 含 `mitmproxy>=12.0`，`backend/mitm_proxy.py` / `proxy.py` / `proxy_worker.py` 使用。**只服务视频号/直播链路**。不做视频号就能整条砍掉。

### 障碍 4：平台专属依赖（Linux 上自动跳过，不是障碍）

```text
pythonnet>=3.0; sys_platform == "win32"
clr_loader>=0.2;  sys_platform == "win32"
pyobjc-framework-Cocoa>=10.0;   sys_platform == "darwin"
pyobjc-framework-WebKit>=10.0;  sys_platform == "darwin"
```

配合仓库里的 `injection_scripts/`、`windows/`、`mac/` 目录，说明 **Windows 端有 .NET 注入微信 PC 客户端的能力**——这正是公众号批量下载的工作原理，也是它在 Linux 上最大的功能缺口。

---

## 3. 为什么「下载与分析分离」对这个项目是天然契合

我把它的**分析能力**全查了一遍，结论是：**它根本没有分析能力**。

| 能力 | 是否有 | 证据 |
|---|---|---|
| ASR / 语音转写 | ❌ | `grep 'import torch' backend/*.py` → 0 命中；`requirements.txt` 无任何 ASR 库 |
| 字幕擦除 | ⚠️ 有代码无依赖 | `backend/subtitle_remover/` 是**第三方 vendored 项目**（含 RAFT / ProPainter / STTN 网络定义），但顶层 `requirements.txt` 里**没有 torch**——说明是可选功能，启用要拉几 GB 依赖 |
| 摘要 / 知识提取 | ❌ | 无 |
| 向量检索 / RAG | ❌ | 无 |
| 视频转码 | ✅ 但很基础 | `backend/transcode.py` 只是包 `ffmpeg` 命令行（:149/346），本机还没装 ffmpeg |

**所以**：「分析在 PC」不是妥协，而是**唯一合理的选择**。VPS 侧只做「抓取 → 下载 → 落盘」，PC 侧做「转写 → 摘要 → 入库」。这既保住了 975 MiB 可用内存，又让重活落在本机 GPU/CPU 上。

---

## 4. 推荐架构

```
┌─ VPS ─────────────────────── 1.9 GiB RAM ───────────────────┐
│                                                            │
│  wechat-mp-tools 容器（下载内核）                             │
│   ├ B站    纯 HTTP，无浏览器  ✅ 常驻                         │
│   ├ 抖音主链路 纯 HTTP 自签    ✅ 常驻                         │
│   ├ 公众号文章                ✅ 常驻                         │
│   ├ RSS 定时增量调度           ✅ 常驻                         │
│   ├ 快手 / 小红书             ⚠️ 按需拉起 Chromium            │
│   └ 视频号 / 字幕擦除 / 分析   ❌ 不部署                       │
│                          ↓                                  │
│  /opt/mp-tools/data/  ──rsync──→  dav 容器 127.0.0.1:6065   │
│                                          /dav/kb-archive/raw │
└────────────────────────────────────────────────────────────┘
                            │ WebDAV 拉取
                            ↓
┌─ 本机 PC ────────────────────────────────────────────────┐
│  ① faster-whisper 本地转写（视频音轨 → SRT）                  │
│  ② 本地/外部 LLM 摘要 + 知识提取                             │
│  ③ 入库：飞书多维表格 / Obsidian                             │
└───────────────────────────────────────────────────────────┘
```

**为什么用现成的 dav 而不是新开同步服务**：机器上已有 `dav` 容器（`hacdias/webdav:latest`，`127.0.0.1:6065`），零新增内存（1.1 MiB）。

---

## 5. 三个部署方案，请选一个

### 方案 A：整体 Docker 化（保留 Flask UI，功能最全）

```dockerfile
FROM mcr.microsoft.com/playwright/python:v1.40.0-jammy
RUN apt-get update && apt-get install -y --no-install-recommends ffmpeg && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && playwright install --with-deps chromium
COPY . .
EXPOSE 5200
CMD ["python", "app.py", "--host", "0.0.0.0", "--port", "5200", "--no-browser"]
```

compose：`network_mode: bridge`，端口只发 `"127.0.0.1:8011:5200"`（不绑 0.0.0.0），可选加 cloudflared ingress。

- ✅ 功能 100% 保留，UI 可用，改动最小
- ❌ 镜像 ~2.5 GB
- ❌ Chromium 常驻，内存峰值 700 MB–1.2 GB，**有 OOM 波及业务容器的风险**
- ❌ `headless=False` 的三处登录仍需 Xvfb + noVNC 才能扫码

### 方案 B：无浏览器下载内核（推荐）⭐

在方案 A 基础上**砍掉所有浏览器依赖**，只保留纯 HTTP 路径：

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN sed -i '/playwright/d; /scrapling/d; /pywebview/d; /mitmproxy/d' requirements.txt \
 && pip install --no-cache-dir -r requirements.txt \
 && apt-get update && apt-get install -y --no-install-recommends ffmpeg && rm -rf /var/lib/apt/lists/*
COPY . .
EXPOSE 5200
CMD ["python", "app.py", "--host", "0.0.0.0", "--port", "5200", "--no-browser"]
```

再删除 `backend/subtitle_remover/`（避免误触发 torch 拉取）。

| 指标 | 方案 A | 方案 B |
|---|---|---|
| 镜像 | ~2.5 GB | **~450 MB** |
| 常驻内存 | 700–1200 MiB | **150–250 MiB** |
| 可用平台 | 公众号文章、B站全功能、抖音全功能、快手、小红书、视频号 | 公众号文章、B站全功能、抖音主链路（**抖音的「喜欢列表」降级**） |
| 登录 | 需 Xvfb/noVNC | **同上（登录仍需浏览器）** |

- ✅ 内存安全，不威胁现有 10 个容器
- ✅ 覆盖你需求里 80% 的场景
- ❌ 快手/小红书的博主主页批量拿不到（只能单视频/单笔记解析）
- 🔧 后续如需补快手/小红书，再单独加一个 `sidecar` 容器跑 Chromium，互不影响

### 方案 C：VPS 只做下载 + 产物走 dav（= 你的设想，= 方案 B + 同步）

方案 B 之上加一条同步：

```bash
# VPS
*/10 * * * * rsync -a --delete /opt/mp-tools/data/ /dav/kb-archive/raw/ >/dev/null 2>&1
```

PC 端挂 WebDAV 客户端同步 `kb-archive/raw/` 到本地目录，后面接 faster-whisper + LLM。

---

## 6. 风险与未知（诚实标注）

| # | 风险 | 等级 | 说明 |
|---|---|---|---|
| 1 | **全部结论来自静态读码，未实机运行** | 🔴 | 我还没在容器里真正跑起来。Playwright 在 headless 下各模块的真实表现、内存实测值都未知 |
| 2 | **公众号批量在 Linux 上大概率不可用** | 🔴 | 原理是注入微信 PC 客户端（`injection_scripts/` + `pythonnet`/`clr_loader` 仅 win32）。**这一环仍需 wechat-article-exporter 补位** |
| 3 | **三个平台登录要可见窗口** | 🟠 | 抖音/小红书/快手全 `headless=False`。要么 Xvfb+noVNC，要么自己在 PC 上登录一次把 Cookie 导出贴进 VPS |
| 4 | **项目可持续性** | 🟠 | README 自述「很多同类开源项目已陆续闭源转向收费，本项目坚持开源」，但只有 186★。**务必 fork 私有化，不要把生产链路押在上游** |
| 5 | RSS 存储用 JSON 文件 | 🟡 | `backend/rss_scheduler.py:21-23` 用 `rss_subscriptions.json` / `rss_articles.json`，每号最多 `MAX_ARTICLES_PER_ACCOUNT = 200`，单次上传 `UPLOAD_BATCH_LIMIT = 100`。量大后需要换 SQLite |
| 6 | 抖音「喜欢列表」走浏览器 | 🟡 | `douyin.py:472-474 _fetch_liked_via_browser`，方案 B 下不可用 |
| 7 | ffmpeg 未装 | 🟡 | `transcode.py` 依赖它，本机没有，需在 Dockerfile 里装或放弃转码 |
| 8 | 反爬风控 | 🟡 | 可复用本机已有的 WARP 池（`172.17.0.1:10010`–`10025`，16 个 mixed 口）。但注意：**docker0 上的容器才能连**（防火墙 `iptables -A INPUT ! -i lo --dport 10010:11109 -j DROP`），独立 bridge 网络会被静默丢包——这个坑 CPA 已经踩过一次 |

---

## 7. 实施计划（分阶段，每阶段可独立回滚）

> ⚠️ **本文的阶段编号是本文件写就时的旧编号（2026-10-12 之前），已作废。**
> 全项目统一编号见 [../PLAN.md](../PLAN.md) §6（Phase 0–11）。下表只给**旧编号 → 新编号**的映射与本文件的专属细节。

| 本文旧编号 | PLAN.md 新编号 | 说明 |
|---|---|---|
| Phase 0 准备 | **Phase 0 脚手架** | 内容一致 |
| Phase 1 容器跑通纯 HTTP | **Phase 1 mp-tools 容器** | 内容一致 |
| — （无） | **Phase 2 kb-hub 中枢** | ADR-005 新增 |
| — （无） | **Phase 3 B站 adapter / Phase 4 公众号 adapter / Phase 5 抖音 adapter** | ADR-005 新增 |
| Phase 2 接 dav 同步 | **Phase 6 产物同步** | 后续内容已扩写 |
| Phase 3 浏览器能力 | **Phase 7 浏览器能力（可选）** | 结论一致 |
| Phase 4 公众号补位 | **Phase 4 公众号 adapter**（已并入） | 内容一致 |
| Phase 5 PC 侧分析 | **Phase 9 PC 分析流水线** | 后续内容已扩写 |
| — （无） | **Phase 8 多端入口 / Phase 10 飞书 / Phase 11 路线 C+** | ADR-002 / ADR-005 新增 |

### Phase 0 — 准备（纯本地，不动服务器） → 新 Phase 0
- [ ] 把 Dockerfile + compose.yml 写好放到 `E:\mimo\opencodeproxy\deploy\mp-tools-compose.yml`
- [x] 用户确认方案 A / B / C ✅ 路线 C
- [ ] 确认服务器磁盘余量（当前 34 G）

### Phase 1 — 容器跑通纯 HTTP 接口（不动浏览器） → 新 Phase 1
- [ ] 记录改动前的 `docker ps` 与 `free -h` 基线
- [ ] `/opt/mp-tools/` 落地，`docker compose up -d`
- [ ] 验证 `GET /api/settings` 200
- [ ] 验证 B站：`get_video_detail` 单个 BV 号出标题/封面
- [ ] 验证抖音：单个作品链接能出无水印直链
- [ ] **`docker stats` 实测常驻内存** ← 这是决定后续所有方案的依据
- [ ] 验收标准：连续跑 30 分钟 RSS 调度器不 OOM

### Phase 2 — 接 dav 同步 → 新 Phase 6
- [ ] 建 `/dav/kb-archive/raw/`
- [ ] 加 rsync cron
- [ ] PC 端 WebDAV 客户端验证能拉到文件

### Phase 3 — 浏览器能力（按需开） → 新 Phase 7
- [ ] 先试小红书：容器内 `launch_chromium(headless=True)` 单次拉起，实测内存增量
- [ ] 内存增量 < 400 MiB → 接受，按需拉起；> 500 MiB → 放 sidecar 或放弃
- [ ] 登录：优先走「PC 登录一次导出 Cookie → 贴进 VPS」，避开 Xvfb/noVNC

### Phase 4 — 公众号补位 → 新 Phase 4
- [ ] 部署 wechat-article-exporter（Docker，绑 `127.0.0.1:8012`）
- [ ] 确认自己有可扫码登录的公众号
- [ ] 跑通「输入公众号 → 全量历史文章」

### Phase 5 — PC 侧分析（不在 VPS） → 新 Phase 9
- [ ] faster-whisper 本地转写视频音轨 → SRT
- [ ] LLM 摘要 + 知识提取（B站弹幕/字幕已有，可直接喂）
- [ ] 入库飞书多维表格或 Obsidian

---

## 8. 已知未知，需要你确认

| # | 问题 | 状态 |
|---|---|---|
| 0 | 是否接受 ADR-005 kb-hub 薄门面（多端统一入口 + 资产索引 + 可扩展平台层）？ | ⏳ **未答，这是当前最关键的一个决定** |
| 1 | 选 A / B / C 哪个？ | ✅ **已定路线 C**（本文方案 B + wae 补位 + PC 分析），见 [../PLAN.md](../PLAN.md) ADR-001 |
| 2 | 抖音/小红书/快手的登录，有没有 PC？ | ⏳ 未答。有 PC 就登录一次导出 Cookie 贴到 VPS，省掉 Xvfb 那一套 |
| 3 | 有没有可扫码登录的公众号？ | ⏳ 未答（决定 Phase 4 的「全量历史」能不能做；单篇文章不依赖它） |
| 4 | 视频文件长期留还是转写完就删？ | ⏳ 未答（Q8；34 G 磁盘，视频很占地方） |
| 5 | 飞书自建应用建了吗？ | ⏳ 未建，**已推迟到 Phase 10**（ADR-002）；且不再阻塞「告诉 AI 发链接」（Q0 独立成立） |
| 6 | 是否要补 TikTokDownloader / douyin_erma0？ | 🟡 C+ 可选项（Phase 11.1 / 11.2），见 [download-layer-comparison.md](download-layer-comparison.md) §3.1 / §5 |
| 7 | 「自己的云端」指 VPS 盘 / 对象存储 / NAS？ | ⏳ 未答（Q7） |

---

## 附录：核实方式

```bash
git clone --depth 1 https://github.com/x554960766/wechat-mp-tools.git && cd wechat-mp-tools
git log -1 --format=%cd
# 逐条结论的复核命令见正文各节引用的 grep 表达式
```

---

## 相关文档

| 文档 | 内容 |
|---|---|
| [../README.md](../README.md) | 项目首页 |
| [../VISION.md](../VISION.md) | 愿景速览（近期/远期目标、可扩展性四条硬定义） |
| [vision.md](vision.md) | **【需求源头】** 愿景原话拆解、S1–S6 场景约束 |
| [../PLAN.md](../PLAN.md) | 项目规划与实施计划（路线 C 决策记录、ADR-005、Phase 0–11） |
| **[../docs/execution-plan.md](../docs/execution-plan.md)** | **施工单**：本文 Phase 1.2–1.3 的完整上下文与验收标准 | 
| **[../docs/environment.md](../docs/environment.md)** | **环境事实与五条铁律**：网络约束、端口占用 |
| [roadmap.md](roadmap.md) | 路线 A / B / C 定义与决策矩阵 |
| [download-layer-comparison.md](download-layer-comparison.md) | mp-tools 与 TikTokDownloader / yutto / douyin_erma0 的功能逐条对比 |
| [feasibility.md](feasibility.md) | 路线 A 可行性与资源硬约束 |
| [douyin-erma0.md](douyin-erma0.md) | 第四候选调研（与本文同类的 Docker 化踩坑） |
| [../deploy/README.md](../deploy/README.md) | 部署物料位（含 hub 容器） |
| [../analysis/README.md](../analysis/README.md) | PC 端分析流水线规划位 |