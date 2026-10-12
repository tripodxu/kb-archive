# douyin_erma0 —— 完整调研与部署评估

> **定位**：本项目最早（2026-10-11）就被评估过的候选下载器，一度准备部署，后因 `pywebview` 构建失败**长期悬空**。
> 本文补齐它的调研档案，明确它在**路线 C+** 中可选的位置（补抖音的话题/音乐/搜索/关注粉丝采集）。
>
> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) ｜ [vision.md](vision.md) ｜ **[execution-plan.md](execution-plan.md)施工单** ｜ **[environment.md](environment.md)环境事实与铁律** ｜ [../PLAN.md](../PLAN.md) ｜ [roadmap.md](roadmap.md) ｜ [download-layer-comparison.md](download-layer-comparison.md) ｜ [feasibility.md](feasibility.md) ｜ [mp-tools-docker.md](mp-tools-docker.md) ｜ **本文** 第四候选调研 ｜ [../deploy/README.md](../deploy/README.md)
>
> 关联：[download-layer-comparison.md](download-layer-comparison.md) §3.1 ｜ [roadmap.md](roadmap.md) ｜ [../PLAN.md](../PLAN.md) Phase 11.2

---

## 0. 结论速览

| 项 | 值 |
|---|---|
| 仓库 | https://github.com/tripodxu/douyin_erma0（fork 自 `erma0/douyin`） |
| 形态 | FastAPI 后端 + React18/Vite 前端 + Aria2 下载器，**三合一单容器** |
| 规模 | **31 个 py / 2.0 MB / 10 个 Python 依赖** —— 四个候选里最轻 |
| License | ⚠️ **GPL-3.0**（≠ mp-tools 的 MIT） |
| 维护状态 | ⚠️ 最后提交 **2026-05-15**，个人 fork，停滞 5 个月 |
| 采集类型 | **10 种**（`aweme` `post` `favorite` `collection` `search` `music` `hashtag` `mix` `following` `follower`）—— 最全 |
| 独有价值 | 话题挑战 / 音乐原声 / 关键词搜索 / 关注粉丝列表（别家都没有） |
| **四个坑** | 🔴 ①公网 80 暴露 ②`VITE_API_BASE_URL` 构建期烘焙 ③**Dockerfile 未装 aria2** ④`pywebview[gtk]` 构建失败（已实测复现） |
| 修法成本 | 低（约 30 分钟，Dockerfile 改 4 处，见 §2 / §4） |
| 决策 | 🟡 **可选增强**（PLAN.md Phase 11.2）；不作 M1–M3 任何里程碑依赖 |

---

## 1. 实测数据

```bash
git clone --depth 1 https://github.com/tripodxu/douyin_erma0.git
du -sh douyin_erma0                 # 1.4M
find . -name '*.py' | wc -l        # 31
cat requirements.txt                # 10 个依赖
```

### 1.1 依赖清单（`requirements.txt`，共 10 个）

```
click>=8.3.1
fastapi>=0.128.0
loguru>=0.7.3
psutil>=7.2.1
pyexecjs>=1.5.1
pyperclip>=1.11.0
requests>=2.4.32
ujson>=5.11.0
uvicorn>=0.40.0
pywebview[gtk]>=6.1; sys_platform == "linux"     ← 唯一的坑
```

### 1.2 采集类型（`backend/lib/douyin/crawler.py:88-103` 的 `self.type` 分支，实测）

```python
if self.type in ["following", "follower"]:
    self.get_awemes_list()
elif self.type in ["post", "favorite", "collection", "search", "music", "hashtag", "mix"]:
    self.get_awemes_list()
elif self.type == "aweme":
    self.get_aweme_detail()
else:
    quit(f"获取目标类型错误, type: {self.type}")
```

| type | 含义 | 其他候选是否有 |
|---|---|---|
| `aweme` | 单个作品 | 三家都有 |
| `post` | 用户主页作品 | 三家都有 |
| `favorite` | 用户喜欢作品 | TikTokDownloader 有 |
| `collection` | 用户收藏作品 | TikTokDownloader 有 |
| `mix` | 合集作品 | mp-tools / TikTokDownloader 有 |
| **`hashtag`** | **话题挑战作品** | ❌ 独有 |
| **`music`** | **音乐原声作品** | ❌ 独有 |
| **`search`** | **关键词搜索作品** | ❌ 独有 |
| **`following`** | **关注用户（仅 CLI 模式）** | ❌ 独有 |
| **`follower`** | **粉丝用户（仅 CLI 模式）** | ❌ 独有 |

**增量能力**：`crawler.py:206-215` 的解析分支传入 `self.results_old`（上次结果）→ `DataParser.parse_awemes(items_list, self.results, self.results_old, ...)`，是**真增量**，不是「靠调度器间接实现」。

**不具备**：评论采集（代码里只有 `comment_count` 字段，无评论接口）。

### 1.3 架构与接口

| 项 | 内容 |
|---|---|
| 单端口 | `backend/server.py:157` `app.mount("/", StaticFiles(directory=_frontend_dist_dir, html=True), name="static")` —— FastAPI 直接吐前端，**不需要 nginx** |
| 监听地址 | `backend/server.py:189` `envvar="DOUYIN_HOST"`（click `--host`），compose 里设 `DOUYIN_HOST: 0.0.0.0` |
| 端口 | 容器内 8000 |
| REST API | `/api/task/start`、`/api/task/status`、`/api/task/results/{id}`、`/api/settings`、`/api/aria2/start`（`backend/routers/{task,settings,system,aria2,file}.py`） |
| 实时推送 | `backend/sse.py` → `/api/events` |
| 下载器 | `backend/lib/aria2_manager.py`，Aria2 RPC，支持视频/图片批量 |
| 凭证 | `DOUYIN_COOKIE` 环境变量（compose `env_file: .env`），**仓库无 `.env.example`，需手写** |
| Cookie 必需性 | 必须含 `sessionid` 与 `ttwid`；官方推荐浏览器 F12 复制 |

---

## 2. 四个部署坑（都已在源码中定位）

### 坑 1：`compose.yaml` 把面板暴露到公网 80 端口 ⚠️ 安全

上游原版：

```yaml
services:
  douyin:
    build:
      context: .
      args:
        - VITE_API_BASE_URL=http://localhost     # ← 坑 2 的根源
    ports:
      - "80:8000"                                  # ← 坑 1：公网暴露，无鉴权
    container_name: douyin-crawler
    environment:
      DOUYIN_HOST: 0.0.0.0
    env_file:
      - .env
```

**这个面板没有任何鉴权**（源码里无 auth router）。直接 `docker compose up` 等于把一个能拿你抖音 Cookie 发请求的接口暴露到公网。

**修法**：

```yaml
    ports:
      - "127.0.0.1:8010:8000"     # 不绑 0.0.0.0
```

### 坑 2：前端 API 地址是**构建期**烘焙进去的

`VITE_API_BASE_URL` 是 `Dockerfile` 的 **build ARG**（`ARG VITE_API_BASE_URL=http://localhost:8000`），Vite 构建时就把 base URL 编译进 JS bundle。

上游 compose 传的是 `http://localhost`（无端口），**只有在宿主机 80 端口被占用时才正确**。改端口发布后如果不同时改这个 ARG，**前端会请求 `http://localhost/api/...` 而失败**。

**修法**：

```yaml
    build:
      context: .
      args:
        - VITE_API_BASE_URL=http://localhost:8010    # 必须与发布端口一致
```

> 这是坑 1 的连带项 —— 改端口必须**同时**改 build ARG，两处一起。

---

## 3. 🔴 坑 3：`pywebview[gtk]` 构建失败（已实测复现，容器起不来）

### 3.1 因果链

```
Dockerfile:26-31   把系统依赖整段注释掉（libwebkit2gtk-4.1-dev / libglib2.0-dev 被注释）
Dockerfile:37      仍执行 pip install -r requirements.txt
requirements.txt:15  pywebview[gtk]>=6.1; sys_platform == "linux"
       ↓
pywebview[gtk] → PyGObject → pycairo（需要 C 编译器）
       ↓
基础镜像 nikolaik/python-nodejs:python3.12-nodejs22-slim 里没有 cc / gcc
       ↓
构建失败
```

### 3.2 实测错误输出

```
The following exception(s) were encountered:
    Running `cc --version` gave "[Errno 2] No such file or directory: 'cc'"
    Running `gcc --version` gave "[Errno 2] No such file or directory: 'gcc'"
  × Encountered error while generating package metadata.
  ╰─> pycairo
  ERROR: Failed to build 'PyGObject' when installing build dependencies for pygobject
```

### 3.3 后果：容器起不来

即使构建勉强通过，启动后立刻崩：

```
ModuleNotFoundError: No module named 'webview'
```

因为**这是顶层无条件 import**，不是一个可选功能分支：

```
backend/lib/cookie_login.py:13   import webview            ← 顶层
backend/routers/system.py:14     from ..lib.cookie_login import get_cookie_by_login
```

`system.py` 依赖 → FastAPI app 在导入期就炸，**整个服务不可用**。

被依赖的是 `get_cookie_by_login` —— **GUI 扫码登录**功能。服务器无桌面环境，这个功能**本来就用不了**，而 Cookie 本来就要 F12 手动复制。

---

## 4. 四个坑的修法（总计约 30 分钟）

### 修坑 3：两种方案（各一行，代价只是丢 GUI 扫码入口）

### 方案 A（推荐）：从 requirements 摘掉 linux 那行

```dockerfile
COPY requirements.txt .
RUN sed -i '/pywebview\[gtk\]/d' requirements.txt \
 && pip install --no-cache-dir -r requirements.txt
```

保留 `pywebview>=6.1; sys_platform != "linux"`（Windows/Mac 桌面版不受影响），只摘 Linux 专用行。

### 方案 B：改成懒加载

`backend/lib/cookie_login.py:13` 的 `import webview` 挪进函数体，仅在真正调用 GUI 登录时导入。

> 方案 B 更「正确」，但要改上游源码、会增加与上游的合并成本；方案 A 零源码改动，**本项目推荐 A**。

### 坑 4：Dockerfile 根本没装 aria2（2026-10-12 新发现）🔴 下载功能不可用

`aria2/README.md` 原文：「此目录用于存放 aria2c 可执行文件，实现开箱即用的下载功能」，
但下载指引只给了 Windows 步骤（`scripts/setup/aria2.ps1`），
并明确「**程序会自动检测**：1. 优先 `aria2/aria2c.exe`（内置）2. 其次系统 PATH 中的 `aria2c` 3. 都没有则提示用户下载」。

源码印证 `backend/lib/aria2_manager.py:162-176`：

```python
bundled_aria2 = os.path.join(RESOURCE_ROOT, "aria2", "aria2c.exe")   # Windows 分支
...
bundled_aria2 = os.path.join(RESOURCE_ROOT, "aria2", "aria2c")       # :169 Linux 分支
...
system_aria2 = shutil.which("aria2c")                                  # :176 PATH 回退
```

而 `Dockerfile`（共 47 行，已逐行核对）里：

```
:1   FROM node:20-alpine AS frontend-builder
:22  FROM nikolaik/python-nodejs:python3.12-nodejs22-slim
:26-31  # RUN apt-get install -y ... ← 整段被注释
:37  RUN pip install --no-cache-dir -r requirements.txt
:47  CMD ["python", "-m", "backend.server"]
```

**没有任何一行安装 aria2**，`aria2/` 目录里只有一个 `README.md`（8 KB）。

**后果**：容器内 `shutil.which("aria2c")` 返回 `None`，aria2 下载通道直接不可用。
必须补一行 `apt-get install -y aria2`（见下方修好的 Dockerfile 要点）。

> 这是继坑 1（公网暴露）、坑 2（build ARG 烘焙）、坑 3（pywebview 构建失败）之后**新发现的第四个坑**，
> 三个部署坑变四个。**仍然不改变「它是可选增强」的结论** —— 因为 31 py 的代码量意味着改起来很快，
> 但**开箱即用的评价是错的**。

### 修好之后的 compose（完整版）

```yaml
services:
  douyin:
    build:
      context: .
      args:
        - VITE_API_BASE_URL=http://localhost:8010   # 坑 2：必须与发布端口一致
    ports:
      - "127.0.0.1:8010:8000"                       # 坑 1：不绑 0.0.0.0
    container_name: douyin-crawler
    environment:
      DOUYIN_HOST: 0.0.0.0
    env_file:
      - .env                                        # 需手写 DOUYIN_COOKIE
    volumes:
      - ./download:/app/download
    mem_limit: 384m
```

### 修好之后的 Dockerfile 改动（共 4 处）

```dockerfile
# 1) :26-31 取消注释并补 aria2（坑 4）
RUN apt-get update && \
    apt-get install -y --no-install-recommends aria2 ffmpeg && \
    rm -rf /var/lib/apt/lists/*

# 2) :37 之前，从 requirements 摘掉 pywebview 的 linux 行（坑 3）
RUN sed -i '/pywebview\[gtk\]/d' requirements.txt && \
    pip install --no-cache-dir -r requirements.txt

# 3) aria2/README.md 那个目录保持空即可，程序会走 PATH 分支（坑 4）
# 4) build ARG 保持与发布端口一致（坑 2）
```

---

## 5. 与路线 C 的关系

| 问题 | 答案 |
|---|---|
| 它能替代 mp-tools 的抖音模块吗？ | 🟡 部分。10 种采集类型更全，但**无评论采集、无点赞收藏统计、无剪贴板监听、无多账号**（后四项 TikTokDownloader 有） |
| 能替代 TikTokDownloader 吗？ | ❌ 不能。后者的 6 项独有能力它都没有 |
| 它是 M1 的依赖吗？ | ❌ **不是**。M1 只做 mp-tools 单链接抓取 |
| 什么时候考虑装它 | Phase 5 之后，且仅当你需要「话题挑战 / 音乐原声 / 关键词搜索 / 关注粉丝」这四项 |
| 成本 | 1 容器 + ~400 MB 镜像 + Cookie 维护 |
| 修好成本 | Dockerfile 改 4 处（aria2 / pywebview / 端口 / build ARG），约 30 分钟 |
| 维护状态 | ⚠️ 上游最后提交 **2026-05-15**（停滞 5 个月），且是个人 fork |
| License | ⚠️ **GPL-3.0**（与 mp-tools 的 MIT 不同，混用需注意传染性） |

> 📌 **排序建议**（同 [download-layer-comparison.md](download-layer-comparison.md) §5 的 C+ 方案）：
> **TikTokDownloader（补深度）优先级 > douyin_erma0（补广度）**。
> 若只加一个，加 TikTokDownloader；若抖音需求已满足，再评估 douyin_erma0。
>
> 📌 **ADR-005 之后**：两者都不必以「独立容器」形式存在 —— 都可以做成 `hub/adapters/` 下的
> 一个 adapter 文件，入口、索引、去重、归档全部复用。TikTokDownloader 自带 FastAPI :5555，
> 接 adapter 成本更低。

---

## 6. 悬空事项回顾（历史记录）

| 时间 | 事件 |
|---|---|
| 2026-10-11 | 用户要求「下载 tripodxu/douyin_erma0 并部署到当前主机容器」，克隆到 `/tmp/douyin_erma0` |
| 2026-10-11 | 决定**先在 127.0.0.1:8010 跑通，确认功能可用后再决定要不要开隧道** |
| 2026-10-11 | 测试 `pip install pywebview[gtk]` 触发 600s 超时，未取得结论 |
| 2026-10-11 晚 | 查明真正原因即 §3 的构建失败（非超时问题），给出两种绕过方案 |
| — | **此后未推进**，本文为补档 |

**当前状态：仍未部署。** 若要启用，从 §4 方案 A 开始，预计 20 分钟内可跑通单链接采集。

---

## 7. 复核命令

```bash
git clone --depth 1 https://github.com/tripodxu/douyin_erma0.git && cd douyin_erma0
du -sh . && find . -name '*.py' | wc -l
cat requirements.txt
sed -n '85,110p' backend/lib/douyin/crawler.py        # 10 种 type
cat compose.yaml                                       # 坑 1
sed -n '1,12p' Dockerfile && sed -n '24,40p' Dockerfile # 坑 3
grep -n 'import webview' backend/lib/cookie_login.py   # 顶层 import
grep -n 'mount' backend/server.py                      # 单端口吐前端
```

---

## 8. 相关文档

| 文档 | 内容 |
|---|---|
| [download-layer-comparison.md](download-layer-comparison.md) | 四个下载器功能矩阵；本文结论在 §3.1 与 §5 |
| [roadmap.md](roadmap.md) | 路线 A/B/C 定义与决策 |
| [vision.md](vision.md) | **【需求源头】** S3「单个作品 vs 作者全部」正是 douyin_erma0 采集类型表要回答的 |
| [mp-tools-docker.md](mp-tools-docker.md) | mp-tools Docker 化调研（本文是其姊妹篇） |
| [../PLAN.md](../PLAN.md) | 项目规划；Phase 11.2 可选位 |
| **[execution-plan.md](execution-plan.md)** | **施工单**：本项目里它的位置是 Phase 11.2 |
| **[environment.md](environment.md)** | 环境事实与五条铁律 |
| [../deploy/README.md](../deploy/README.md) | 部署物料位 |