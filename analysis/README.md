# analysis —— PC 端分析流水线

> 跑在**你的电脑**上，不占用 VPS 内存。读取 VPS 经 WebDAV 同步下来的产物，
> 做语音转写 → LLM 摘要 → 分类 → 入库。

> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) 愿景速览 ｜ [../docs/vision.md](../docs/vision.md) 需求源头 ｜ [../PLAN.md](../PLAN.md) 规划 ｜ [../docs/roadmap.md](../docs/roadmap.md) 路线 ｜ [../docs/download-layer-comparison.md](../docs/download-layer-comparison.md) 功能对比 ｜ [../docs/feasibility.md](../docs/feasibility.md) 资源硬约束 ｜ [../docs/mp-tools-docker.md](../docs/mp-tools-docker.md) Docker 化调研 ｜ [../docs/douyin-erma0.md](../docs/douyin-erma0.md) 第四候选 ｜ [../deploy/README.md](../deploy/README.md) 部署物料 ｜ **本文** PC 分析

## 状态：待开发（PLAN.md Phase 9）

这是本项目**工作量最大的部分**，但它是「下载与分析分离」架构的必然代价——
`docs/mp-tools-docker.md` 里说明了为什么下载侧完全不碰分析。

## 规划中的模块

| 文件 | 职责 |
|---|---|
| `sync.py` | WebDAV 拉取产物到本地工作区 |
| `transcribe.py` | 视频音轨提取 + faster-whisper 转写 → SRT |
| `normalize.py` | 文档/字幕归一化为纯文本（剥离 HTML 排版） |
| `summarize.py` | LLM 摘要 + 关键词 + 自动分类 |
| `inbox.py` | 输出 Markdown 到 Obsidian vault |
| `writeback.py` | **摘要/标签/ASR 文本回写 hub**（`PATCH /api/assets/{id}/analysis`） |
| `pipeline.py` | 全链路串跑 + 失败重试 + 断点续传 |
| `config.example.yaml` | 配置模板（真实配置不入库） |

## 设计原则

1. **幂等**：同一个文件重复处理不产生重复归档
2. **断点续传**：长任务中断后从上次位置继续
3. **凭证隔离**：API key 只从本地环境变量或 `.env` 读，`.gitignore` 已排除
4. **产物不入库**：`.gitignore` 已排除 `vault/`、`inbox/`、`*.mp4` 等
5. **双向闭环**：分析结果回写 hub 的 `analysis` 字段（schema 已预留），使 VPS 侧索引也能查到摘要 —— 见 [../PLAN.md](../PLAN.md) Phase 9.7

## 待确认

- LLM 用哪家 API、是否有本地模型选项
- Obsidian vault 路径
- 视频是否转写完即删（磁盘策略，Q8）
- 「自己的云端」是否就是现有 dav（Q7）

---

## 相关文档

| 文档 | 内容 | 关联本文哪节 |
|---|---|---|
| [../docs/vision.md](../docs/vision.md) | **【需求源头】** L3–L5「自行整理」远期目标就是本文这一层 | 全文定位 |
| [../PLAN.md](../PLAN.md) Phase 9 | 分析层 WBS 任务分解（含 9.7 回写） | 规划中的模块 |
| [../docs/feasibility.md](../docs/feasibility.md) §1 | **为什么不能放 VPS**：本地 LLM 必 OOM、ASR 必须腾讯云 | 状态说明 |
| [../docs/roadmap.md](../docs/roadmap.md) §3 | 路线 C 为什么把分析层放 PC | 状态说明 |
| [../docs/download-layer-comparison.md](../docs/download-layer-comparison.md) | 下载层会产出什么（本层的输入契约） | 规划中的模块 |
| [../deploy/README.md](../deploy/README.md) | 产物从 VPS 经 WebDAV 到本机 | sync.py |