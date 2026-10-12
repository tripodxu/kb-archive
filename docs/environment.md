# 环境事实与铁律

> **这份文档的读者是「没有上下文的人」** —— 无论是另一个执行 agent、三个月后的你自己，
> 还是任何一个第一次看到这个仓库的人。**动手之前先读完这一页。**
>
> 所有事实均于 **2026-10-12** 在目标主机上实测采集，命令附在各节末尾，可随时复核。
>
> **导航**：[../README.md](../README.md) ｜ [../VISION.md](../VISION.md) ｜ [vision.md](vision.md) ｜ [**本文** 环境事实与铁律] ｜ [execution-plan.md](execution-plan.md) 施工单 ｜ [../PLAN.md](../PLAN.md) 总规划 ｜ [roadmap.md](roadmap.md) ｜ [download-layer-comparison.md](download-layer-comparison.md) ｜ [feasibility.md](feasibility.md) ｜ [mp-tools-docker.md](mp-tools-docker.md) ｜ [douyin-erma0.md](douyin-erma0.md) ｜ [../deploy/README.md](../deploy/README.md)

---

## 0. 五条铁律（违反其中任何一条都会造成真实事故）

| # | 铁律 | 违反后果 |
|---|---|---|
| **R-1** | **任何新容器必须 `network_mode: bridge`（默认 docker0），不许用自定义 bridge 网络** | 自定义网络容器连宿主 WARP 池口 `172.17.0.1:10010` 会被 iptables **静默丢包**，表现为 `i/o timeout` 而非 connection refused，排查成本极高 |
| **R-2** | **任何宿主端口发布都不许绑 `0.0.0.0`**，一律 `127.0.0.1:` 或 `172.17.0.1:` | mp-tools 面板**无鉴权**，公网暴露 = 任何人可下载任意内容 |
| **R-3** | **VPS 上不许跑 ASR / LLM / 视频转码 / 模型推理** | 1.9 GiB 内存 + 无 GPU，实测必然 OOM；且 OOM 会连带打死同机其它 10 个容器 |
| **R-4** | **不许动现有 10 个容器的端口与网络**（见 §3） | 这些是生产服务，有公网入口与外部调用方 |
| **R-5** | **新增容器必须同步加入 `o2a-watchdog.py` 的 `CONTAINERS` 清单** | 否则容器停了巡检不报警，等于没有监控 |

---

## 1. 主机规格

| 项 | 实测值 | 复核命令 |
|---|---|---|
| OS | Ubuntu 22.04.5 LTS | `cat /etc/os-release` |
| CPU | **2 vCPU**（Xeon E5-2698 v4） | `nproc` |
| 内存 | **1963 MiB 总 / 770 used / ~1009 MiB available** | `free -m` |
| Swap | 2047 MiB（197 used） | `free -m` |
| 磁盘 | `/dev/vda1` **49 G，已用 15 G，余 34 G（31%）** | `df -h /` |
| 内核 | 运行中 `5.15.0-30-generic`，已装 `5.15.0-198`，**待重启** | `uname -r` |

**这台机器的可用内存只有约 1 GiB。** 现有 10 个容器合计已占约 630 MiB。
本项目新增的常驻预算（[../PLAN.md](../PLAN.md) §7）：

| 容器 | 预算 | 上限 |
|---|---|---|
| mp-tools | 150–250 MiB | 512 MiB |
| kb-hub | 60–100 MiB | 256 MiB |
| wae（可选） | 100–200 MiB | 256 MiB |

---

## 2. 网络与 Docker 的关键事实

### 2.1 Docker

- Docker Engine **29.x**（实测 `29.9.0` 一线）
- `/etc/docker/daemon.json` 已配 `log-opts: {max-size: 10m, max-file: 3}`
  → **只对 2026-10-10 之后新建的容器生效**；存量容器改不了，只能重建
  → **好消息**：本项目新建的容器天然带日志上限，不需要额外处理
- ⚠️ **`docker update --log-opt` 不存在**（`unknown flag`）。别再试。
- ⚠️ **`systemctl reload docker` 在此版本不可靠**：日志显示 `Reloaded configuration`，
  但回显配置里 `log-opts` 字段直接消失，新建容器仍是 `Config={}`。
  改动 daemon.json 后**必须实测** `docker run --rm alpine ls -s /var/lib/docker/containers` 之类不可行，
  正确验证方式是建一个临时容器看 `docker inspect --format '{{.HostConfig.LogConfig}}'`。
- 备份文件在 `/etc/docker/daemon.json.bak-20261011-0644`，可用 `dockerd --validate --config-file=...` 校验。

### 2.2 防火墙（R-1 的原因）

`o2a-firewall.service` → `ExecStart=/usr/local/sbin/o2a-fw.sh` 会**重建**下面两条规则：

```
-A INPUT -i docker0   -p tcp -m tcp --dport 10010:11109 -j ACCEPT
-A INPUT ! -i lo      -p tcp -m tcp --dport 10010:11109 -j DROP
```

含义：
- **docker0 上的容器** → 可以连宿主 WARP 代理池（16 个口 `10010`–`10025`）
- **其他任何来源** → 全部 DROP
- 因为第二条是 `! -i lo`，**自定义 bridge 网络的容器流量进不来** → 静默丢包

**没有 IPv6 的对应规则** —— 如果本项目要出 IPv6，需要单独评估。

### 2.3 WARP 代理池

| 项 | 值 |
|---|---|
| 入口 | `0.0.0.0:10010` – `10025`（16 个 sing-box mixed 口，**需认证**） |
| 内部 | `127.0.0.1:20000` – `20015`（16 个 usque） |
| 实例目录 | `/root/warp-instances/warp-0` … `warp-15` |
| 出口去重 | 16 个实例只落在约 5 个 IPv4 —— **不是真正独立的 16 个出口** |
| 使用方式 | hub / mp-tools 若需出网，proxy 写 `http://<user>:<pass>@172.17.0.1:10010` |

> ⚠️ 容器内**不能用 `127.0.0.1`** 指向宿主，必须用 `172.17.0.1`。

---

## 3. 现有 10 个容器（R-4 的依据）

| 容器 | 镜像 | 发布端口 | 内存 | 重启策略 |
|---|---|---|---|---|
| `cpa` | `eceasy/cli-proxy-api:latest` | `127.0.0.1:8317` + `172.17.0.1:8317` | ~15 MiB | unless-stopped |
| `agent2api` | `agent2api:3.0.1` | `127.0.0.1:3065` | ~104 MiB | unless-stopped |
| `newapi-tunnel` | `cloudflared` | — | ~32 MiB | always |
| `newapi-tunnel-quic` | `cloudflared` | — | ~26 MiB | always |
| `opencode2api` | `ghcr.io/jasonxu114514/opencode2api:v1.3.7-fix5` | `172.17.0.1:8080` + `127.0.0.1:8081` | ~18 MiB | unless-stopped |
| `o2a-shadow4` | `opencode2api:patched4` | `172.17.0.1:8090` | ~14 MiB | unless-stopped |
| `new-api` | `calciumion/new-api:latest` | `127.0.0.1:3000` | ~46 MiB | always |
| `jev-router` | `python:3.12-alpine` | 无 | ~6 MiB | — |
| `warpgate` | `zhoushu1/warp-pool:latest` | host 网络，`127.0.0.1:4433` | ~335 MiB / 上限 700 MiB | — |
| `dav` | `hacdias/webdav:latest` | `127.0.0.1:6065` | ~1 MiB | always |

**`dav` 是本项目的下游**（资产库 rsync 目标，`/dav/kb-archive/raw`）。

### 3.1 宿主非容器组件（部分与本项目共享资源）

| 组件 | 说明 | 本项目相关性 |
|---|---|---|
| `session-filter.service` | `172.17.0.1:8082`，MemoryMax=64M | 无直接关系 |
| `o2a-firewall.service` | 重建 §2.2 的 iptables 规则 | **R-1 的执行者** |
| `o2a-watchdog.py` | cron `*/5`，巡检 10 个关键容器 / 磁盘 / 内存 / 流量 | **R-5 要求改它** |
| `o2a-quota-guard.py` | cron `*/3` | 无 |
| `o2a-rotate-one.py`、`o2a-slot-reconcile.py` | cron `7 *` | 无 |
| `backup-all.sh` | cron `40 3`，产出 `/root/backup/full/*.tar.gz`（保留 14 天） | **新部署目录要加进备份** |
| `docker-log-truncate.sh` | cron `25 4`，阈值 50 MiB 兜底 | 无 |
| `fail2ban`、`sshd` | **仅密钥登录**（`PasswordAuthentication no`） | 连接方式 |
| `he6-policy-route` | 策略路由 | 无 |

> `backup-all.sh` 用 `add` / `addglob` 显式列路径，收录了 `opt/cli-proxy-api/*`、`opt/jev-router/*`、
> `root/docs/*.md` 等。新增 `/opt/mp-tools`、`/opt/kb-archive` 时**必须同步加进去**。

---

## 4. 端口占用实况（2026-10-12 实测）

**已被占用，不能碰**：

```
22            sshd              0.0.0.0
3000          new-api           127.0.0.1
3065          agent2api         127.0.0.1
4433          warpgate          127.0.0.1
6065          dav               127.0.0.1
8080          opencode2api      172.17.0.1
8081          opencode2api      127.0.0.1  (WebUI)
8082          session-filter    172.17.0.1
8090          o2a-shadow4       172.17.0.1
8317          cpa               127.0.0.1 + 172.17.0.1
10010-10025   sing-box × 16     0.0.0.0
20000-20015   usque × 16        127.0.0.1
53            systemd-resolved  127.0.0.53
```

**已预留、当前空闲**（本项目规划使用）：

| 宿主端口 | 容器内 | 用途 |
|---|---|---|
| `127.0.0.1:8010` | 8000 | douyin_erma0（Phase 11.2，可选） |
| `127.0.0.1:8011` | 5200 | **mp-tools** |
| `127.0.0.1:8012` | 8317 | **wae**（公众号全量补位，可选） |
| `127.0.0.1:8018` | 8000 | **kb-hub** |

复核：`ss -tlnH | grep -E ':(800[0-9]|801[0-9]|802[0-9]|5200)\b'`

---

## 5. 既有对外入口（cloudflared ingress，version=6）

| 子域名 | 后端 |
|---|---|
| newapi | `new-api:3000` |
| dav | `dav:6065` |
| jev | `opencode2api:8080` |
| agent2api | `agent2api:3065` |
| cpa | `172.17.0.1:8317` |
| fallback | 404 |

两条隧道（`newapi-tunnel`、`newapi-tunnel-quic`）共用 UUID `65f9c4f2-97df-4499-bb1f-c447cf94ef54`。

> ⚠️ **Cloudflare 后台的源地址必须写 IP，不能写容器名。**
> 写 `cpa:8317` 会报 `lookup cpa on 127.0.0.11:53: server misbehaving` → 502
> （默认 bridge 网络没有内嵌 DNS，也不跨网解析）。本项目若挂公网同理，**写 `172.17.0.1:8018`**。

---

## 6. 本项目要新增的组件一览

| 容器 | 目录 | 网络 | 端口 | 重启 | 内存上限 |
|---|---|---|---|---|---|
| `mp-tools` | `/opt/mp-tools/` | `network_mode: bridge` | `127.0.0.1:8011:5200` | `unless-stopped` | 512m |
| `kb-hub` | `/opt/kb-archive/` | `network_mode: bridge` | `127.0.0.1:8018:8000` | `unless-stopped` | 256m |
| `wae`（可选） | `/opt/wae/` | `network_mode: bridge` | `127.0.0.1:8012:8317` | `unless-stopped` | 256m |

三者都在 docker0 上，因此 IP 依次为 `172.17.0.5` / `.6` / `.7`（当前 bridge 上已有 `.2` opencode2api、`.3` o2a-shadow4、`.4` cpa）。

---

## 7. 复核清单（每次开工前跑一遍）

```bash
# 1. 资源基线
free -m && df -h / && nproc

# 2. 现有容器基线（回滚时用来对比）
docker ps --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'

# 3. 端口是否空闲
ss -tlnH | grep -E ':(8010|8011|8012|8018)\b' || echo "全空"

# 4. 防火墙规则是否还在（R-1 的前提）
iptables -S INPUT | grep 10010

# 5. 备份是否可用
bash -n /root/backup-all.sh && echo "backup-all.sh 语法 OK"
```

---

## 8. 相关文档

| 文档 | 内容 | 关联本文哪节 |
|---|---|---|
| [execution-plan.md](execution-plan.md) | **施工单**：Phase 0–11 逐项任务、验收标准、回滚 | 全文 |
| [../PLAN.md](../PLAN.md) | 总规划、ADR 决策、资源预算、风险登记册 | §1 资源预算、§7 |
| [vision.md](vision.md) | **需求源头**：S1–S6 场景约束 | 铁律的由来 |
| [feasibility.md](feasibility.md) | **为什么不能本地跑 LLM**（R-3 的来源） | §1 R-3 |
| [mp-tools-docker.md](mp-tools-docker.md) | 浏览器依赖逐文件实测、三种部署方案 | §6 mp-tools |
| [../deploy/README.md](../deploy/README.md) | Dockerfile 要点与部署物料位 | §6 |
| `../../docs/container-topology.md` | 本机 10 个容器的完整拓扑（母仓库，**未脱敏，勿外传**） | §3 |
