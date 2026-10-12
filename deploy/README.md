# 部署物料

服务器侧部署文件存放处。按 Phase 逐步填充。

> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) 愿景速览 ｜ [../docs/vision.md](../docs/vision.md) 需求源头 ｜ **[../docs/execution-plan.md](../docs/execution-plan.md)施工单** ｜ **[../docs/environment.md](../docs/environment.md)环境事实与铁律** ｜ [../PLAN.md](../PLAN.md) 规划 ｜ [../docs/roadmap.md](../docs/roadmap.md) 路线 ｜ [../docs/download-layer-comparison.md](../docs/download-layer-comparison.md) 功能对比 ｜ [../docs/mp-tools-docker.md](../docs/mp-tools-docker.md) Docker 化调研 ｜ [../docs/douyin-erma0.md](../docs/douyin-erma0.md) 第四候选 ｜ **本文** 部署物料 ｜ [../analysis/README.md](../analysis/README.md) PC 分析

## Phase 1 — mp-tools（下载主容器）

| 文件 | 状态 | 说明 |
|---|---|---|
| `mp-tools/Dockerfile` | 待写 | `python:3.12-slim` + 剥离 playwright/scrapling/pywebview/mitmproxy + ffmpeg |
| `mp-tools/docker-compose.yml` | 待写 | `network_mode: bridge`，发布 `127.0.0.1:8011:5200`，内存上限 512 MiB |
| `mp-tools/patches/` | 待写 | 对上游的本地补丁 |
| `mp-tools/.env` | **不入库** | 含 Cookie 等凭证，已被 .gitignore 排除 |

**Dockerfile 要点**（详见 `docs/mp-tools-docker.md`）：

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
# 剥离浏览器相关依赖——这是 1.9 GiB 内存下能跑起来的关键
RUN sed -i '/playwright/d; /scrapling/d; /pywebview/d; /mitmproxy/d' requirements.txt \
 && pip install --no-cache-dir -r requirements.txt \
 && apt-get update && apt-get install -y --no-install-recommends ffmpeg \
 && rm -rf /var/lib/apt/lists/*
COPY . .
# 删除 vendored 的字幕擦除模块（含 torch 依赖，启用需数 GB）
RUN rm -rf backend/subtitle_remover
EXPOSE 5200
CMD ["python", "app.py", "--host", "0.0.0.0", "--port", "5200", "--no-browser"]
```

**启动参数**：`--host` / `--no-browser` 是上游 `app.py:162-163` 已有的参数，`--no-browser` 避免容器内尝试打开浏览器失败。

## Phase 2 — kb-hub（归档中枢，ADR-005）

若 Q0 获批，hub 与 mp-tools 同为 Phase 1/2 的产物。

| 文件 | 状态 | 说明 |
|---|---|---|
| `hub/Dockerfile` | 待写 | `python:3.12-slim` + FastAPI/uvicorn，**不含任何浏览器依赖** |
| `hub/docker-compose.yml` | 待写 | `network_mode: bridge`，发布 `127.0.0.1:8318:8000`，内存上限 256 MiB |
| `hub/assets/schema.json` | 待写 | 统一资产 schema（19 字段，见 [../PLAN.md](../PLAN.md) §3） |
| `hub/storage/` | 待写 | storage 接口，实现 local disk / dav / 对象存储（Q7 待定） |
| `hub/.env` | **不入库** | 含 `KB_TOKEN` 与各平台 Cookie，已被 .gitignore 排除 |

**要点**：hub 是薄门面，**不重写抓取逻辑**；与 mp-tools 同在 docker0，才能用宿主 WARP 池口。

## Phase 6 — dav 同步

| 文件 | 状态 |
|---|---|
| `dav-sync.sh` | 待写 |
| `crontab` | 待写（每 10 分钟 rsync 到 dav） |

## Phase 4 — wae（公众号全量补位）

| 文件 | 状态 |
|---|---|
| `wae/docker-compose.yml` | 待写，发布 `127.0.0.1:8012` |

---

## 网络约束（踩过的坑）

若需经宿主 WARP 代理池出网，**容器必须用 `network_mode: bridge`**（即默认 docker0）。
自定义 bridge 网络的容器连宿主代理口会被防火墙静默丢包，表现为 `i/o timeout` 而非 connection refused。

---

## 目录约定

```
服务器：
  /opt/mp-tools/          容器代码与 compose
  /opt/mp-tools/data/     下载产物（配置、账号、下载历史、文章）
  /opt/kb-archive/        hub 代码与 compose
  /opt/kb-archive/media/  统一资产库 <platform>/<author>/<id>/
```

---

## Phase 11 — 可选增强容器（默认不部署）

若 M3 跑通后确认需要补抖音深度/广度，再从下列方案中选。**详见 [../docs/download-layer-comparison.md](../docs/download-layer-comparison.md) §5 与 §3.1。**

| 容器 | 补什么 | 部署要点 |
|---|---|---|
| `joeanamier/tiktok-downloader` | 评论采集 / 点赞统计 / 剪贴板监听 / 多账号 / Web API / 收藏夹 | 官方镜像，`-p 端口:5555 -v <数据目录>:/app/Volume`，**共享 `/opt/mp-tools/data/`**；或在 hub 里做成 adapter 调其 `:5555` |
| douyin_erma0（自建） | 话题挑战 / 音乐原声 / 关键词搜索 / 关注粉丝 | 必须先按 [../docs/douyin-erma0.md](../docs/douyin-erma0.md) §4 修**四个坑**（含 `apt-get install aria2`）；端口 `127.0.0.1:8010`，**build ARG `VITE_API_BASE_URL` 必须与发布端口一致** |
| ~~yutto~~ | ❌ 不建议 | B 站能力已被 mp-tools 打平，且 FFmpeg 是硬依赖 |

---

## 相关文档

| 文档 | 内容 |
|---|---|
| [../README.md](../README.md) | 项目首页 |
| [../VISION.md](../VISION.md) | 愿景速览（近期/远期目标、可扩展性四条硬定义） |
| [../docs/vision.md](../docs/vision.md) | **【需求源头】** 愿景原话拆解、S1–S6 场景约束 |
| [../PLAN.md](../PLAN.md) | 项目规划（Phase 0–11 WBS、ADR-005、资产 schema、API 契约） |
| **[../docs/execution-plan.md](../docs/execution-plan.md)** | **施工单**：每个 Phase 的具体产出文件与验收 |
| **[../docs/environment.md](../docs/environment.md)** | **环境事实与五条铁律**：网络约束、端口占用 |
| [../docs/roadmap.md](../docs/roadmap.md) | 三条路线定义与决策 |
| [../docs/download-layer-comparison.md](../docs/download-layer-comparison.md) | 下载层功能逐条对比 + 抖音二选一专项 + C+ 建议 |
| [../docs/mp-tools-docker.md](../docs/mp-tools-docker.md) | mp-tools Docker 化调研（本文 Dockerfile 的来源） |
| [../docs/douyin-erma0.md](../docs/douyin-erma0.md) | douyin_erma0 部署踩坑与修法 |
| [../analysis/README.md](../analysis/README.md) | PC 端分析流水线（消费本文产出的 WebDAV 目录） |