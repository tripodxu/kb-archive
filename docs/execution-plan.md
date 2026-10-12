# 施工单 —— kb-archive 执行计划

> **这份文档是给「执行者」看的，不是给读者看的。**
> 执行者可能是另一个 agent，也可能是三个月后的你。
> 它假设你**没有参与过任何前期讨论**，所以每一步都写明了：前置条件、要做什么、怎么算做完、做砸了怎么退。
>
> **动手前必读**：**[environment.md](environment.md)（环境事实与五条铁律）**
> **方案依据**：[../PLAN.md](../PLAN.md)（ADR 决策、架构、资源预算、风险）
>
> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) ｜ [vision.md](vision.md) ｜ [environment.md](environment.md) ｜ [**本文** 施工单] ｜ [../PLAN.md](../PLAN.md) ｜ [roadmap.md](roadmap.md) ｜ [download-layer-comparison.md](download-layer-comparison.md) ｜ [feasibility.md](feasibility.md) ｜ [mp-tools-docker.md](mp-tools-docker.md) ｜ [douyin-erma0.md](douyin-erma0.md) ｜ [../deploy/README.md](../deploy/README.md) ｜ [../analysis/README.md](../analysis/README.md)

---

## 0. 怎么用这份施工单

### 0.1 开工铁律（每条都对应真实踩过的坑）

| # | 规则 | 为什么 |
|---|---|---|
| **W-1** | **一次只做一个 Phase。** Phase N 的验收不通过，不许开 Phase N+1 | 本机只有 1 GiB 可用内存，同时铺三个容器必炸 |
| **W-2** | **每个 Phase 开工前，先跑一遍 [environment.md](environment.md) §7 的复核清单并把输出存档** | 否则出问题时分不清是自己搞坏的还是本来就坏的 |
| **W-3** | **每个 Phase 结束前，跑一遍 [../PLAN.md](../PLAN.md) §7 的回滚步骤演练（dry-run），确认回滚路径仍然有效** | 没演练过的回滚方案等于没有回滚方案 |
| **W-4** | **代码在本地仓库写、在服务器跑；不要在服务器上直接 `vim` 改代码** | 服务器上的改动无法追溯、无法回滚 |
| **W-5** | **每个 Phase 的验收输出（命令 + 原始输出）写进当 Phase 的 PR 描述或提交信息** | 「跑通了」这种口头结论不可验证 |
| **W-6** | **不要为了让测试通过而放宽断言。** 测试失败先判断是代码错还是环境错 | 放宽断言 = 把 bug 变成沉默 |

### 0.2 全程适用的验收硬指标

| 指标 | 阈值 | 为什么是这个数 |
|---|---|---|
| 新增常驻内存合计 | **≤ 450 MiB** | 现有 10 容器已占 ~630 MiB，本机 available ~1009 MiB |
| 对现有 10 个容器的影响 | **零** | 它们有公网入口和外部调用方 |
| 磁盘峰值 | **≤ 50 GiB**（即把剩余 34 GiB 用到剩 15 GiB 前必须停） | 见 R14 |
| 端口暴露面 | **零新增公网监听** | 见 R-2 |
| 破坏 `docker ps` 里其它容器的能力 | **零** | |

### 0.3 明确不在本施工单范围内

- ❌ **飞书集成**（Phase 10）—— 用户尚未建立自建应用，且 ADR-005 之后 `curl /api/ingest` 已能满足 S2
- ❌ **VPS 上的任何 ASR / LLM / 转码**（R-3）
- ❌ **小红书 / 快手 / 视频号 / 微信读书** —— 需要 Chromium，与 1.9 GiB 冲突（Phase 7 是决策门，不是承诺）
- ❌ **`docker network rm cli-proxy-api_default`** —— 这是运维侧的独立待办（空网络），与本项目无关
- ❌ **动 jev-router 去留** —— ADR-003 已定：长期保留

---

## 1. 任务总览与依赖

```
Phase 0  脚手架（本地，不碰服务器）
   │
Phase 1  mp-tools 容器跑通  ★ 第一个「能吃」的里程碑
   │
Phase 2  kb-hub 架构核心  ★★ 整个项目的地基
   │
   ├── Phase 3  B站 adapter      ┐
   ├── Phase 4  公众号 adapter    ├─ 三个可并行，但都依赖 Phase 2
   └── Phase 5  抖音 adapter     ┘
   │
Phase 6  dav 同步（资产出 VPS）
   │
Phase 7  浏览器能力（决策门，可跳过）
   │
Phase 8  多端入口（Web UI / CLI / AI 调用说明）
   │
Phase 9  PC 分析流水线（真正的「自行整理」）
   │
Phase 10 飞书（可选）
Phase 11 路线 C+ 增强（默认不做）
```

| Phase | 依赖凭据吗？ | 能否立刻开工 | 产出价值 |
|---|---|---|---|
| 0 | ❌ 不需要 | ✅ | — |
| 1 | ❌ 不需要 | ✅ | 能下载、能看 |
| 2 | ❌ 不需要 | ✅ | **告诉 AI 发链接**（S2 达成） |
| 3 | ❌ 不需要（B站无需 Cookie） | ✅ | 近期目标 1/3 |
| 4 | ⚠️ 4.1–4.2 不需要；**4.3 需要**自有可扫码公众号 | ✅ 部分 | 近期目标 2/3 |
| 5 | ⚠️ 5.2 公开作品不需要；私密作品需 Cookie | ✅ 部分 | 近期目标 3/3 |
| 6 | ❌ 不需要 | ✅ | PC 能看到文件 |
| 7 | ⚠️ 需要平台 Cookie | ⛔ 决策门 | 小红书/快手 |
| 8 | ❌ 不需要 | ✅ | 多端可用（S1 达成） |
| 9 | ⚠️ 需要 LLM API key（PC 侧） | ✅ 部分 | **自行整理**（S5 达成） |
| 10 | ⚠️ 需要飞书 App ID/Secret | ⛔ | 飞书入库 |
| 11 | ⚠️ 需要 Cookie | ⛔ 按需 | 抖音深度/广度 |

> **结论：Phase 0 → 3 全链路不需要用户任何凭据。** 卡住的只有公众号「全量历史」、抖音私密作品、小红书/快手、飞书这四处。

---

## 2. Phase 0 — 脚手架（纯本地）

**前置**：无

| # | 任务 | 产出 |
|---|---|---|
| 0.1 | 在 `hub/` 下建包结构：`hub/{api,adapters,assets,storage,workers,web}`，每个目录放 `__init__.py` | `hub/` |
| 0.2 | 建 `pyproject.toml`（依赖：fastapi / uvicorn / httpx / pydantic / click / python-multipart），**pin 死版本** | `hub/pyproject.toml` |
| 0.3 | 建 `tests/` 骨架 + `pytest` 配置，`pytest` 能跑通（哪怕零测试） | `hub/tests/` |
| 0.4 | 建 `Makefile`：`make test` / `make lint` / `make build` | `hub/Makefile` |
| 0.5 | CI：GitHub Actions 跑 `make test`（矩阵 py3.12） | `.github/workflows/ci.yml` |
| 0.6 | 写 `hub/README.md`：本地怎么跑、怎么调 mp-tools、怎么跑测试 | `hub/README.md` |

**验收**
```bash
cd hub && make test          # 退出码 0
cd hub && python -c "import hub.server"   # 能导入
```

**回滚**：`rm -rf hub/ .github/` —— 纯本地，无副作用。

---

## 3. Phase 1 — mp-tools 容器跑通 ★

**前置**：Phase 0 完成

**为什么第一个**：mp-tools 是抓取能力的主体，所有 adapter 最终都要包它的代码。它跑不起来，后面全是空谈。

| # | 任务 | 说明 |
|---|---|---|
| 1.1 | fork `x554960766/wechat-mp-tools` 到私有仓库，**记录 commit 并固定**（R2：作者声明同类项目在闭源收费，fork 必须早） | 私有 fork |
| 1.2 | 按 [mp-tools-docker.md](mp-tools-docker.md) 的方案 B 写 `Dockerfile`：`python:3.12-slim` + `sed` 摘掉 playwright/scrapling/pywebview/mitmproxy 四项 + 装 `ffmpeg` + `rm -rf backend/subtitle_remover` | `deploy/mp-tools/Dockerfile` |
| 1.3 | 写 compose：`network_mode: bridge`（**R-1**）、`ports: ["127.0.0.1:8011:5200"]`（**R-2**）、`mem_limit: 512m`、`restart: unless-stopped`、挂 `./data:/app/data` | `deploy/mp-tools/docker-compose.yml` |
| 1.4 | 部署到 `/opt/mp-tools/`，`docker compose up -d` | — |
| 1.5 | 验证 `GET /api/settings` 返回 200 | — |
| 1.6 | 验证 B站：给一个 BV 号，`get_video_detail` 能出标题 + 封面 | — |
| 1.7 | 验证抖音：给一个公开作品链接，能出无水印直链 | — |
| 1.8 | **`docker stats --no-stream` 实测常驻内存 —— 这是决定后续所有方案的核心数据点** | 记录进提交信息 |
| 1.9 | 跑满 30 分钟，确认 RSS 调度器（`rss_scheduler.start()`）不 OOM、不崩溃重启 | — |
| 1.10 | 把 `mp-tools` 加入 `/root/o2a-watchdog.py` 的 `CONTAINERS` 列表（**R-5**），手动跑一次 `--json` 确认被识别 | — |
| 1.11 | 把 `/opt/mp-tools/compose.yml` + `data/*.json` 加进 `/root/backup-all.sh` | — |

**验收**
```bash
# 三个平台各成功一次真实抓取
curl -s http://127.0.0.1:8011/api/settings | head -c 300
docker stats --no-stream mp-tools          # 内存 ≤ 512 MiB
docker ps --filter name=mp-tools --format '{{.Status}}'   # 无 unhealthy、无反复重启
```

**DoD**：
- [ ] 公众号文章、B站视频、抖音视频**各成功抓取一次**
- [ ] 常驻内存实测值已记录（不是估算）
- [ ] 连续 30 分钟无 OOM、无 restart
- [ ] 现有 10 个业务容器 `docker ps` 状态与开工前完全一致
- [ ] watchdog 能看到它，backup-all.sh 能备份它

**回滚**：`docker compose -f /opt/mp-tools/compose.yml down && rm -rf /opt/mp-tools`，再把 watchdog / backup 里的条目删掉。无外部依赖。

---

## 4. Phase 2 — kb-hub 架构核心 ★★

**前置**：Phase 1 完成（即使 hub 还没接 adapter，这一步也应先跑通 `noop` 路径）

**这是整个项目的地基**。愿景的 S2（AI 发链接）、S4（去重）、S5（归档）、S6（失效可感知）与「必须可扩展」全部落在这一步。[../PLAN.md](../PLAN.md) §3 有完整架构图、资产 schema、API 契约。

| # | 任务 | 产出 | 关键约束 |
|---|---|---|---|
| 2.1 | 建 `hub/` 包结构 | `hub/` | Phase 0 已建骨架 |
| 2.2 | **冻结 asset schema**，写成 JSON Schema | `hub/assets/schema.json` | 见 [../PLAN.md](../PLAN.md) §3，**改动需改版本号** |
| 2.3 | 链接解析器：正则识别 B站/抖音/公众号/小红书/快手/微博/YouTube | `hub/links.py` | 识别不了要明确返回 unknown，**不能猜** |
| 2.4 | SQLite 建表 `assets` / `tasks` / `authors` / `tags` + 索引 | `hub/storage/sqlite.py` | WAL 模式；迁移脚本要可回滚 |
| 2.5 | 实现 7 个 API 端点 | `hub/api/` | 见 [../PLAN.md](../PLAN.md) §3 API 契约 |
| 2.6 | adapter 协议 + 注册表 + 一个 `noop` 示例 adapter | `hub/adapters/base.py` | 协议必须足够小（见 2.10） |
| 2.7 | 落盘规范 `<platform>/<author>/<asset_id>/` + `meta.json` **原子写** | `hub/storage/filesystem.py` | 写临时文件 → fsync → rename；**顺序错了索引会和磁盘不一致（R15）** |
| 2.8 | 最小任务 worker：轮询 SQLite | `hub/workers/poller.py` | **不引入 Redis / Celery**；用 `SELECT ... FOR UPDATE SKIP LOCKED` 或文件锁防重入 |
| 2.9 | Dockerfile + compose：`network_mode: bridge`、`127.0.0.1:8018:8000`、`mem_limit: 256m` | `deploy/hub/` | R-1 / R-2 |
| 2.10 | **`POST /api/ingest` 端到端可用**（noop adapter 也返回任务记录） | — | — |
| 2.11 | 加入 watchdog `CONTAINERS`（**R-5**）+ `backup-all.sh` | — | — |
| 2.12 | 加 `kb verify` 子命令：比对索引与磁盘实际文件，双向报告差异 | `hub/cli.py` | R15 的检测手段 |

**验收**
```bash
T=$(curl -s -XPOST http://127.0.0.1:8018/api/ingest \
      -H 'Content-Type: application/json' -H "X-KB-Token: $KB_TOKEN" \
      -d '{"url":"https://www.bilibili.com/video/BV1xx","scope":"single"}')
echo "$T"                                   # 200 + task_id

# S4 幂等：重复提交必须拿到同一个 asset_id
curl -s -XPOST http://127.0.0.1:8018/api/ingest -H 'Content-Type: application/json' \
     -H "X-KB-Token: $KB_TOKEN" \
     -d '{"url":"https://www.bilibili.com/video/BV1xx","scope":"single"}'

# S6：不认识的链接必须 400 + 可读错误，不许静默成功
curl -s -XPOST http://127.0.0.1:8018/api/ingest -H 'Content-Type: application/json' \
     -H "X-KB-Token: $KB_TOKEN" -d '{"url":"https://example.com/nope"}'

docker stats --no-stream kb-hub             # ≤ 150 MiB
```

**可扩展性验收（W-6：不许降低标准）**
```bash
# 写一个只返回假数据的空 adapter，注册它，然后确认无需改动任何其它文件
# 就能通过 POST /api/ingest 处理该平台的链接
```

**DoD**：
- [ ] 上面 4 条 curl 全部符合预期
- [ ] 常驻内存 ≤ 150 MiB
- [ ] 重复提交同一链接 → 同一 `asset_id`（S4）
- [ ] 未知链接 → 400 + 可读错误（S6）
- [ ] 新增一个空 adapter = 1 个文件 + 1 行注册，**零核心文件改动**
- [ ] `kb verify` 能查出「索引有但磁盘没有」与「磁盘有但索引没有」
- [ ] mp-tools 容器未受影响

**回滚**：`docker compose -f /opt/kb-archive/compose.yml down`，删掉 watchdog/backup 条目。Phase 1 成果全部保留。

---

## 5. Phase 3 — B站 adapter

**前置**：Phase 2 完成。**不需要任何凭据。**

| # | 任务 | 需不需要 SESSDATA |
|---|---|---|
| 3.1 | `adapters/bilibili.py`：包 mp-tools `backend/bilibili.py` 的纯 HTTP 函数（`get_playurl:314` / `get_collection_details:438` / `xml_to_ass:122` / `json_to_srt:217`） | — |
| 3.2 | `scope=single`：BV/av → 视频 + 封面 + 音频 + 字幕 SRT + 弹幕 ASS | ❌ |
| 3.3 | `scope=author`：UP 主全部投稿（分页 + 增量去重） | ❌ |
| 3.4 | `scope=collection`：合集全部 | ❌ |
| 3.5 | `scope=favorites`：收藏夹 | ✅ 无则**降级跳过并明确报错**，不许假装成功 |
| 3.6 | 落盘 + `meta.json` + 入索引 | — |
| 3.7 | adapter 单测：mock HTTP，断言 URL 构造与解析 | — |

**验收**：给一个 BV 号与一个 UP 主 UID，`scope` 分别取 `single` / `author` / `collection`，
产出视频文件 + ASS 弹幕 + SRT 字幕，`meta.json` 字段齐全，**重复执行不产生重复资产**。

**DoD**：三个 scope 各跑通一次；内存不超 256 MiB 容器上限；mp-tools 日志无新增异常。

---

## 6. Phase 4 — 公众号 adapter

**前置**：Phase 2 完成

| # | 任务 | 需不需要凭据 |
|---|---|---|
| 4.1 | `adapters/wechat.py`：包 mp-tools `backend/articles.py`（纯 HTTP，浏览器依赖 0 命中） | — |
| 4.2 | `scope=single`：正文抽取（`#js_content`）+ 图片本地化 + 转 Markdown | ❌ |
| 4.3 | `scope=author`：**公众号全量历史** —— 走 wechat-article-exporter 的扫码登录后台搜索能力 | ⚠️ **需要自有可扫码公众号** |
| 4.4 | wae 容器 compose，`127.0.0.1:8012:8317` | — |

> **4.3 的凭据阻塞**：用户尚未确认是否有可扫码登录的公众号（Q2）。
> **在用户回答之前，4.3 标记为「阻塞」并继续做 4.1/4.2/4.4，不要停下来等。**

**验收**：单篇文章归档为 Markdown + 本地图片，**全链路无外链图片残留**（这是最容易漏的验收点）；
若配置了可扫码公众号，「输入公众号名 → 全量历史」跑通。

---

## 7. Phase 5 — 抖音 adapter

**前置**：Phase 2 完成

| # | 任务 | 需不需要 Cookie |
|---|---|---|
| 5.1 | `adapters/douyin.py`：包 mp-tools `backend/douyin.py` + `douyin_sign.py` + `douyin_mstoken.py`（主链路纯 HTTP，不走 Playwright） | — |
| 5.2 | `scope=single`：无水印直链 + 封面 | ❌ 公开作品 |
| 5.3 | `scope=author`：主页全部作品 | ⚠️ 私密号需要 |
| 5.4 | `scope=collection`：合集 | — |
| 5.5 | 增量去重：按 `platform_id` 比对索引 | — |
| 5.6 | ⚠️ **反爬检测（R16）**：落盘前校验文件头与大小，0 字节或明显 HTML 一律写 `error` 字段，**不许静默成功** | — |
| 5.7 | `adapter.health_check()`：Cookie 失效时能被探活发现（R17） | — |

**验收**：公开作品链接 + 作者主页链接均能全量下载，去重生效。
**已知风险**：上游有 429 频发记录（实测 `probe=429`），**429 不等于失败**，要区分「限流，退避重试」与「真失败」。

---

## 8. Phase 6 — dav 同步

**前置**：Phase 3–5 至少完成一个。

| # | 任务 | 产出 |
|---|---|---|
| 6.1 | 在已有 dav 容器中建 `/dav/kb-archive/raw` 目录 | — |
| 6.2 | rsync 同步脚本（增量 + 断点续传 + 日志） | `deploy/dav-sync.sh` |
| 6.3 | 加入 crontab（每 10 分钟） | `deploy/crontab` |
| 6.4 | PC 端 WebDAV 客户端验证能拉到文件 | — |
| 6.5 | **Q7 待决**：若「自己的云端」指对象存储，实现 `hub/storage/objectstore.py` 适配器 | 条件执行 |

**验收**：VPS 下载的文件在 PC 上 10 分钟内可见。

**回滚**：删 crontab 行，dav 目录保留无害。

---

## 9. Phase 7 — 浏览器能力（**决策门，可跳过**）

**这不是一个承诺，是一次测量。**

| # | 任务 |
|---|---|
| 7.1 | 单次拉起 headless Chromium，**实测内存增量** |
| 7.2 | 增量 < 400 MiB → 接受，按需拉起；> 500 MiB → 放独立 sidecar 或**放弃** |
| 7.3 | 补 `adapters/xiaohongshu.py` / `adapters/kuaishou.py`（条件执行） |

**为什么默认是「放弃」**：Chromium 常驻 400–700 MiB，本机 available 只有 ~1009 MiB，
拉起来之后 OOM 会连带打死同机的 10 个生产容器。**这个风险不值得为一个非近期目标平台承担。**

**关于登录**：抖音（`douyin_login.py:232`）、小红书（`xiaohongshu_login.py:46`）、快手（`kuaishou_auth.py:160`）
**全部是 `headless=False`**，需要可见窗口扫码。
**首选路径：在 PC 上正常登录一次 → 导出 Cookie → 贴进 VPS 配置**，可完全绕开 Xvfb + noVNC。

---

## 10. Phase 8 — 多端入口

**前置**：Phase 2 完成（`/api/ingest` 此时**早已可用**，本 Phase 只是补 UI/CLI/文档）

| # | 任务 | 产出 |
|---|---|---|
| 8.1 | hub 自带极简 Web UI：输入框 + 平台/scope 选择 + 任务列表 | `hub/web/` |
| 8.2 | CLI：`kb add <url> --scope author` / `kb ls` / `kb show <id>` / `kb verify` | `hub/cli.py` |
| 8.3 | **AI 调用说明文档 + 可直接复制的 curl 示例** | `docs/ai-usage.md` |
| 8.4 | 移动端适配（响应式，竖屏可用） | — |
| 8.5 | 可选：Cloudflare Tunnel 暴露（自建子域名），端口仍只绑 `127.0.0.1` | 需用户侧建子域名 |

**验收**：手机浏览器、电脑终端、AI curl 三种方式提交同一链接，
三者返回**同一个 `asset_id`**（幂等），且都能查到下载进度。

> ⚠️ 若挂 Cloudflare，**后台源地址必须写 IP**（如 `172.17.0.1:8018`），写容器名会 502。
> 且 hub 有鉴权，挂公网前建议先加 Cloudflare Access。

---

## 11. Phase 9 — PC 分析流水线（**本项目最大的实际工作量**）

**前置**：Phase 6 完成。跑在**用户的电脑上**，不占 VPS 内存。

| # | 任务 | 产出 |
|---|---|---|
| 9.1 | 同步脚本：WebDAV 拉取到本地工作区（或直接读 hub 的 `/api/assets`） | `analysis/sync.py` |
| 9.2 | 视频音轨提取 + faster-whisper 转写 → SRT | `analysis/transcribe.py` |
| 9.3 | 文档/字幕归一化为纯文本 | `analysis/normalize.py` |
| 9.4 | LLM 摘要 + 关键词 + 自动分类 | `analysis/summarize.py` |
| 9.5 | 输出 Markdown 到 Obsidian vault，按日期/平台分目录 | `analysis/inbox.py` |
| 9.6 | 全链路串跑 + 失败重试 + 断点续传 | `analysis/pipeline.py` |
| 9.7 | 摘要回写 hub 的 `analysis` 字段 | — |
| 9.8 | 使用说明 | `analysis/README.md` |

**验收**：往系统里丢 3 个链接（B站/抖音/公众号各一），24 小时内 PC 上出现带摘要的归档 Markdown。

---

## 12. Phase 10 / 11 — 可选

| Phase | 前置 | 说明 |
|---|---|---|
| 10 飞书 | 用户建好自建应用（Q3） | ADR-002 已推迟。若走 know-where 需额外 PostgreSQL，**必须重估内存预算** |
| 11.1 TikTokDownloader | 抖音是主力平台 | 优先做成 hub 的一个 adapter（它自带 FastAPI `:5555`），而不是多一个容器 |
| 11.2 douyin_erma0 | 确有话题/音乐/搜索需求 | **必须先修四个坑**（见 [douyin-erma0.md](douyin-erma0.md) §4：`aria2` 缺失 / `pywebview` / 公网 80 / build ARG 烘焙），约 30 分钟 |
| 11.3 yutto | — | ❌ **不建议**：B 站能力已被 mp-tools 打平，FFmpeg 是硬依赖，无净收益 |

> **顺序约束**：11.1 优先于 11.2（补深度 > 补广度）。任何补充都必须等 Phase 3–5 跑通、
> 且内存实测有余量之后再做 —— 1.9 GiB 是第一约束。

---

## 13. 里程碑与交付物

| 里程碑 | 包含 Phase | 用户能看到什么 |
|---|---|---|
| **M1 冷启动** | 0 + 1 | 网页上能下载 B站/抖音/公众号的单个作品 |
| **M2 中枢** | 2 | **一句 `curl` 就能入库，重复提交不重复** |
| **M3 近期目标达成** | 3 + 4 + 5 | 三平台的「单个 / 作者全部」都能下 |
| **M4 端到端** | 6 | PC 上能看到下载好的文件 |
| **M5 多端入口** | 7（可选）+ 8 | 手机、终端、AI 三种方式都能用 |
| **M6 闭环** | 9 | PC 上出现**带摘要的整理好的 Markdown** |
| **M7 增强** | 10 + 11 | 飞书 / 抖音深度（按需） |

**M1 和 M2 不需要用户任何凭据 —— 这是最快能拿出来看的东西。**

---

## 14. 待用户回答（阻塞对应任务，但不阻塞其它任务）

| # | 问题 | 阻塞 | 建议默认值 |
|---|---|---|---|
| Q1 | 抖音 / 小红书 / 快手的 Cookie | 5.3、Phase 7、11 | PC 上登录一次导出 |
| Q2 | 有无可扫码登录的自有公众号 | **4.3 公众号全量历史** | 有就做，没有则该功能不做 |
| Q3 | 飞书自建应用 App ID/Secret | Phase 10 | 继续推迟 |
| Q4 / Q8 | 视频长期留还是转写完即删 | Phase 6.5、R14 | 先留，磁盘见底再说 |
| Q5 | Obsidian vault 路径 | 9.5 | — |
| Q6 | 抖音是否深度使用 | Phase 11 | — |
| Q7 | 「自己的云端」= 本地盘 / 对象存储 / NAS | 6.5 | 先 VPS + 现有 dav |
| Q9 | 是否需要多用户权限 | hub 鉴权设计 | 单用户 + 静态 token（当前设计） |
| Q10 | 评论要不要入库 | `scope=comments` | 建议入库（JSON，体积极小） |
| Q11 | 是否要浏览器插件 | Phase 8 之后 | 不进近期 |

---

## 15. 相关文档

| 文档 | 内容 | 关联本文哪节 |
|---|---|---|
| [environment.md](environment.md) | **必读**：环境事实、五条铁律、端口占用、复核清单 | 全文 |
| [../PLAN.md](../PLAN.md) | ADR 决策、目标架构 v2、资产 schema、API 契约、风险登记册 | Phase 2 |
| [vision.md](vision.md) | **需求源头**：S1–S6 场景约束、可扩展性四条硬定义 | §1 依赖图、§14 |
| [mp-tools-docker.md](mp-tools-docker.md) | 浏览器依赖逐文件实测、三种部署方案、Dockerfile 要点 | Phase 1 |
| [douyin-erma0.md](douyin-erma0.md) | 第四候选调研 + 四个坑 | Phase 11.2 |
| [download-layer-comparison.md](download-layer-comparison.md) | A vs C 逐条对比、抖音二选一 | Phase 11 |
| [roadmap.md](roadmap.md) | 三条路线定义与决策台账 | §0.3 |
| [feasibility.md](feasibility.md) | **为什么不能本地跑 LLM**（铁律 R-3 的来源） | §0.3 |
| [../deploy/README.md](../deploy/README.md) | 部署物料位（Dockerfile / compose / dav-sync） | 各 Phase 产出 |
| [../analysis/README.md](../analysis/README.md) | PC 分析流水线规划位 | Phase 9 |