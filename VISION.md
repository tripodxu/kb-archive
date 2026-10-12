# 愿景（速览）

> 完整版见 **[docs/vision.md](docs/vision.md)**，本文件是仓库首页的入口摘要。

## 一句话

> 一个**自下载**的、**具有保存功能**的、**自行整理**的、**多媒介 / 多媒体**的、
> **独属于自己**的、**本地化**的知识库。

## 最佳使用场景

看到有意思的内容 → 终端 / 手机 / 电脑 → **直接告诉 AI 发个链接**（或用 Web UI）
→ 自由选择「单个作品」还是「这个作者的全部作品」，也可要评论、弹幕、收藏夹
→ 本地化下载 → 存到自己的云端 → **系统自动归档整理**。

## 近期目标

**抖音 + B站 + 公众号的服务器端下载功能，且必须可扩展。**

可扩展性的四条硬定义：

1. 新增平台 = 一个 adapter 文件 + 一行注册，不改中枢代码
2. 新增媒体类型 = 只改 asset schema 的 `files` 字段
3. 新增入口（CLI / Web / IM / 浏览器插件）= 只实现「调一次 `POST /api/ingest`」
4. 新增存储后端（本地盘 / 对象存储 / NAS）= 一个 storage 接口

## 相关文档

| 文档 | 内容 |
|---|---|
| [docs/vision.md](docs/vision.md) | 完整愿景、场景约束 S1–S6、近期/远期目标、待澄清需求 |
| [PLAN.md](PLAN.md) | 目标 G1–G12、ADR-001~005、Phase 0–11 任务分解 |
| [docs/roadmap.md](docs/roadmap.md) | 三条路线 A/B/C 完整定义与决策矩阵 |
| [docs/download-layer-comparison.md](docs/download-layer-comparison.md) | 下载层功能逐条对比、四个下载器能力矩阵 |
