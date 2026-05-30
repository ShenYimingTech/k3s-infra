# K3s 集群配置架构说明

> 版本：基于 2026-05-17 实际 SSH 探查的硬件配置 + 既定决策。
> 范围：8 台节点（sh / ot / bo / mu / ph / au / sc / di）。di 当前可通过 103.102.135.165 登录，`di.k4s.live` 已指向该 IP，109.111.55.170 是同机附加 IP。
> 备注：部分角色基于推荐方案（标 ▲），尚未最终确认。

---

## 1. 节点清单与角色

| 节点 | CPU | RAM | 磁盘 | 地区 | 服务商 | 集群角色 | k3s 状态 |
|---|---|---|---|---|---|---|---|
| **bo** | 8C AMD EPYC-Milan | **32G** | **250G / 15% used** | HK (Tung Chung) | GreenCloud | ▲ **control-plane + storage 主副本** | 未安装 |
| **sh** | 2C AMD EPYC 7K83 | 1G | 40G / 19% used | 上海 | 腾讯云 | worker（轻量，保留 derp/rustdesk） | 未安装 |
| **mu** | 2C AMD EPYC-Milan | 4G | 40G / 40% used | SG | GreenCloud | worker + anytls | 未安装 |
| **ot** | 2C Xeon 8255C | 2G | 30G / 21% used | HK | 腾讯云轻量 | 受限 worker（保留 AdGuard，调度轻量非关键服务） | 未安装 |
| **au** | 1C Xeon SF | 1G | 21G / 19% used | LA | 16clouds(BWG) | worker + anytls | 未安装 |
| **sc** | 1C EPYC-Genoa | 768M | 15G / 28% used | LA | 16clouds(BWG) | worker + anytls | 未安装 |
| **ph** | 2C AMD EPYC 9655 | 2G | 40G / 31% used | LA | DMIT | worker + anytls | 未安装 |
| **di** | 4C Xeon E5-2680 v4 | 8G | 50G / 6% used | FR (?) | EasyHeberg | worker + anytls + storage 副本候选 | 未安装 |

**MVP 存储策略**：第一阶段使用 bo 本地持久卷 + R2 备份，不启用 Longhorn 多副本。Longhorn 放到第二阶段再评估，候选为 bo + di，ph 可作第三候选。

**control-plane 选择说明**：原定 sh，但 sh 仅 1G RAM 跑不动 k3s server + Rancher + ArgoCD + VictoriaMetrics 全家桶（合计需 4G+），且公网 80/443 被 derper 占用。bo 是全场配置最高且磁盘余量最大，跨境到其他海外节点延迟低，作 control-plane 最合适。

---

## 2. 网络拓扑

```
                          ┌─────────────────────────────────────────┐
                          │  Tailscale Mesh  (taila41d4.ts.net)    │
                          │  100.x.x.x/16                          │
                          │  ─ k3s 控制面 (apiserver/kubelet)      │
                          │  ─ 节点间 CNI (flannel over tailscale0)│
                          │  ─ 运维 ssh                            │
                          │  DERP 中继：sh (118.25.102.60)         │
                          └─────────────────────────────────────────┘
                                          │
        ┌─────────┬─────────┬─────────┬───┴─────┬─────────┬─────────┐
        │         │         │         │         │         │         │
       bo        mu        ph        au        sc        di        sh
     [c-plane] [worker]  [worker]  [worker]  [worker]  [worker]  [worker]
       │         │         │         │         │         │         │
       │         │ 公网 80/443 入口（海外节点入口层 + sing-box；sh 排除）
       └─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
                                          │
                          ┌───────────────┴───────────────┐
                          │     Public Internet           │
                          │  客户端 anytls + Web 流量    │
                          └───────────────────────────────┘
```

### 流量分层

| 流量类型 | 路径 | 端口 |
|---|---|---|
| k3s 控制面（apiserver/kubelet） | tailscale 100.x.x.x | 6443 / 10250（内网） |
| Pod 跨节点通信 (CNI) | tailscale | flannel over tailscale0 |
| 运维 SSH / kubectl | tailscale | 22 |
| 业务 Web (Rancher/ArgoCD/Grafana) | 公网入 → ingress 层 | TLS 终结 |
| anytls 代理流量 | 公网入 → ingress/TCP passthrough → sing-box | SNI 路由 |
| 监控 vmagent 远程写 | 公网 → ingress (`vm.obs.k4s.live`) | basic-auth + TLS |
| Longhorn 备份 | bo → R2 (公网) | HTTPS |

---

### 2.2 Cloudflare Mesh 叠加层（2026-04 新发布产品）

> CF Mesh = WARP Connector 升级版，跟 Cloudflare One（Access/Tunnel/Workers VPC）原生整合。
> 跟 tailscale 是**同层并存**，各管一部分，不替代。

**覆盖范围**：

| 节点 | tailscale | CF Mesh | 备注 |
|---|---|---|---|
| bo | ✅ | ✅ mesh node | control-plane，双 mesh 接入 |
| mu / au / sc / ph / di | ✅（di 待安装） | ✅ mesh node（待实施） | 5 台海外 worker，双 mesh 接入 |
| sh | ✅ | ❌ | 境内节点，CF 国内不稳，不接 |
| ot | ✅（待确认/安装） | ❌ | 受限 worker，不跑入口/存储/管理面 |
| 你的 mac | ✅ | ✅ WARP client | 双客户端，运维 + 管理面接入 |

**职责分工**：

| 场景 | 主路径 | 备/补 |
|---|---|---|
| k3s 节点间通信（apiserver / kubelet / CNI） | tailscale | – |
| 服务器运维 SSH | tailscale | CF Mesh（备用） |
| 跨境节点接入（sh ↔ 海外） | tailscale（自建 DERP @ sh） | – |
| 客户端访问管理面（rancher/argocd 等） | **公网 ingress** | **CF Mesh + Access SSO** |
| 未来跨云（AWS/GCP）接入 | – | CF Mesh（主） |
| 未来 AI agent 访问私有 API | – | CF Mesh（主） |

**管理面双路径**：

```
                  rancher.ops.k4s.live
                       │
        ┌──────────────┴──────────────┐
        │                             │
   公网 DNS 解析                CF Mesh 内访 (经 WARP)
   → 任意入口节点               → mesh node 内部网络
   → managed Traefik            → managed Traefik (同入口)
   → rancher pod                → rancher pod
   
   两条路径终点同一 ingress，对 rancher pod 无感知差异。
   公网路径仍受 ingress 上的鉴权策略保护；
   CF Mesh 路径可选叠加 CF Access SSO（更强）。
```



### bo（control-plane + 主存储 + 现有业务节点）

> **现状（探查得知）**：bo 上已运行 docker 业务全家：
> - **lobehub 全套**：lobe-postgres (paradedb pg17) / lobe-searxng / lobe-redis / lobe-rustfs / lobehub
> - **authentik 2026.2.2**（worker + server，SSO/IdP）
> - **prometheus**（已有监控）
> - **new-api**
> 资源占用约 4.4G RAM / 36G 磁盘。32G/250G 总量仍宽裕。

**k3s 新增组件**：

```
[k3s server]
[cert-manager]
[managed Traefik ingress / TCP passthrough 入口层]
[sealed-secrets controller]
[argocd]
[victoria-metrics-single]    -- 跟现有 prometheus 并存（见 §8）
[victoria-logs-single]
[grafana]                    -- 跟旧 grafana@mu 并存或替代
[alertmanager]
[sub-store]
[sing-box anytls]
```

MVP 暂缓组件：

- Rancher：第二阶段再上，MVP 先用 kubectl + ArgoCD。
- Longhorn：第二阶段再上，MVP 先用 bo 本地持久卷 + R2 备份。

**共存策略**：现有 docker 容器全部保留运行，k3s 与 docker 共用 host（k3s 用 containerd，docker 用 dockerd，互不影响）。bo 不需要"迁移"，只是叠加 k3s。


### sh（境内 worker，轻量）

实测状态（2026-05-17）：

- OS: Ubuntu 24.04.4 LTS，Tencent Cloud CVM，2C / 945Mi RAM / 40G disk
- 内存：约 411Mi used，533Mi available，swap 1.9Gi（已用约 115Mi）
- 磁盘：`/` 40G，已用 7.1G，19%
- Tailscale IP: `100.91.85.73`
- Docker: active；containerd: active；tailscaled: active；node_exporter: active
- 运行容器：`derper`、`rustdesk-server`

```
[k3s agent]
[node-exporter + vmagent + vector]
[tailscale derp]             -- Docker derper，占公网 80/443/3478，保留现状，不动
[rustdesk-server]            -- Docker，hbbs/hbbr 21115-21119，保留现状，不动
[standalone anytls]           -- 独立高端口，不抢 derper 的 443
```

不跑：anytls inbound（境内出口风险）、Longhorn storage（资源不足）、Rancher/ArgoCD/VM、公网 ingress（80/443 已被 derper 占用）。

### mu / au / sc / ph / di（海外 worker + anytls）

```
[k3s agent]
[managed Traefik ingress / TCP passthrough 入口层]
[sing-box anytls DaemonSet]  -- hostPort 8443，ssl-passthrough 入
[node-exporter + vmagent + vector]
```

mu 磁盘占用已降到 40%，可投入；ph 已验证可登录；di 通过 103.102.135.165 登录，待安装 tailscale。

### ot（受限 worker + 现有 DNS 节点）

```
[k3s agent]                  -- taint: node-role.k4s.live/restricted=true:NoSchedule
[AdGuard Home]               -- 现状保留，host 级服务，不迁入 k3s
[node-exporter + vmagent + vector]
[轻量非关键服务]              -- 仅通过明确 toleration 调度
```

可调度到 ot 的服务：

- `sub-store` 副本或客户端配置相关轻量 API
- `reminder_bot` 等小型 bot
- `vmagent` / `vector` / `blackbox-exporter`
- 临时测试 pod、低优先级 CronJob

不调度到 ot 的服务：

- control-plane / Rancher / ArgoCD / Grafana / VictoriaMetrics / VictoriaLogs
- Longhorn storage replica
- 公网入口层 / anytls edge
- Postgres / Redis / RustFS / 其他有状态核心服务

---

## 4. 数据流

### 4.1 客户端访问 anytls

```
client (mihomo) ──TLS ClientHello SNI=hk1.edge.k4s.live ──→ hk1 节点 :443
                                                                    │
                                                          ingress/TCP passthrough
                                                          (海外节点，sh 排除)
                                                                    │
                                                        SNI 匹配 *.edge.k4s.live
                                                          → ssl-passthrough
                                                                    │
                                                        本节点 sing-box :8443
                                                        (internalTrafficPolicy: Local)
                                                                    │
                                                              出站到 Internet
```

### 4.2 客户端访问管理面

```
client ──→ rancher.ops.k4s.live:443 ──→ 任意入口节点
                                          │
                                  cert-manager TLS termination
                                          │
                                  proxy_pass → rancher service
                                          │
                                          → rancher pod（钉在 bo）
```

### 4.3 GitOps 推送

```
本机 git push → GitHub (private repo: ShenYimingTech/k3s-infra)
                            │
                            │ webhook 或 pull 间隔
                            ▼
                       ArgoCD (bo)
                            │
                       同步 manifests
                            │
                            ▼
                    k3s apiserver (bo)
                            │
                            ▼
                     各节点 kubelet 应用变更
```

### 4.4 监控数据流

```
每节点 vmagent ──HTTPS (basic-auth)──→ vm.obs.k4s.live ingress ──→ VictoriaMetrics (bo)
每节点 vector  ──HTTPS────────────────→ vl.obs.k4s.live ingress ──→ VictoriaLogs (bo)
                                                                       │
                                                                       ▼
                                                              alertmanager
                                                                       │
                                                            ┌──────────┴──────────┐
                                                            ▼                     ▼
                                                       Telegram bot          (Bark 待启用)
```

### 4.5 备份流

```
Longhorn 每日全量+增量 ──S3 协议──→ Cloudflare R2 (EU)
                                       bucket: k3s-longhorn-backup
ArgoCD 配置 ←─────────────────────── GitHub repo（即配置层备份）
sealed-secrets master key ──手动导出──→ 本机离线 + 1Password
```

MVP 阶段实际备份：

```
bo 本地 PV 数据 ──定时备份/快照──→ Cloudflare R2
ArgoCD 配置 ←────────────────────── GitHub repo
sealed-secrets master key ─────────→ 本机离线 + 1Password
```

---

## 5. 关键参数与约束

### 5.1 域名规划（按地区重排）

> `k4s.live` 已确认为正式根域。

#### 5.1.1 地区边缘入口

| 地区 | 节点 | 域名 | 指向 |
|---|---|---|---|
| HK | bo | `hk1.edge.k4s.live` | `45.128.220.149` |
| SG | mu | `sg1.edge.k4s.live` | `45.89.219.248` |
| US-LA | ph | `us1.edge.k4s.live` | `69.63.219.203` |
| US-LA | au | `us2.edge.k4s.live` | `45.62.119.159` |
| US-LA | sc | `us3.edge.k4s.live` | `104.194.88.247` |
| FR | di | `fr1.edge.k4s.live` | `103.102.135.165` |

约定：

- anytls 客户端只使用 `*.edge.k4s.live`，客户端策略组按 `HK / SG / US / FR` 分组。
- 同地区多节点按稳定编号命名，不暴露机器名；换机器时优先复用原地区编号。
- `sh` 不跑 anytls；`ot` 虽作为受限 worker 进集群，但不跑 anytls，不纳入 edge 域名。
- 预留通配证书：`*.edge.k4s.live`。

#### 5.1.2 管理面

| 域名 | 用途 | 指向 |
|---|---|---|
| `rancher.ops.k4s.live` | Rancher UI | 管理面入口 |
| `argocd.ops.k4s.live` | ArgoCD UI | 管理面入口 |
| `longhorn.ops.k4s.live` | Longhorn UI | 管理面入口 |
| `grafana.ops.k4s.live` | 新 Grafana | 管理面入口 |
| `alert.ops.k4s.live` | Alertmanager | 管理面入口 |

约定：

- `ops.k4s.live` 只放管理 UI，后续统一接 authentik / CF Access。
- 管理面入口优先指向 `bo` 或海外入口池；不指向 `sh`。
- 预留通配证书：`*.ops.k4s.live`。

#### 5.1.3 观测与写入端点

| 域名 | 用途 | 指向 |
|---|---|---|
| `vm.obs.k4s.live` | VictoriaMetrics remote-write | 入口层 |
| `vl.obs.k4s.live` | VictoriaLogs ingest | 入口层 |
| `probe.obs.k4s.live` | blackbox / 外部探测入口（预留） | 入口层 |

约定：

- `obs.k4s.live` 放机器写入或采集端点，不作为普通人访问入口。
- 写入端点继续使用 basic-auth / token；证书用 `*.obs.k4s.live`。

#### 5.1.4 订阅与客户端配置

| 域名 | 用途 | 指向 |
|---|---|---|
| `sub.client.k4s.live` | sub-store 后台/API | 入口层 |
| `profile.client.k4s.live` | 客户端订阅 profile（预留） | 入口层 |

约定：

- `client.k4s.live` 放最终给客户端使用的订阅/API 地址。
- 不再使用容易冲突的短名 `sub.k4s.live`。

#### 5.1.5 保留的独立服务

| 服务 | 建议域名 | 节点 | 说明 |
|---|---|---|---|
| DERP | `derp.cn.k4s.live` | sh | 继续指向 `118.25.102.60`，占 80/443/3478 |
| RustDesk | `rustdesk.cn.k4s.live` | sh | 如需域名，指向 `118.25.102.60` |
| AdGuard | `dns.hk.k4s.live` | ot | 继续作为 host/Docker 独立服务，不迁入 k3s |

地区子域约定：

- `cn.k4s.live`：境内独立服务，不作为 k3s 入口。
- `hk.k4s.live`：香港独立服务或区域服务。
- 新 k3s 入口统一走 `edge / ops / obs / client` 四个功能域，避免和旧服务混在一起。

### 5.2 端口约定

| 端口 | 用途 | 监听节点 |
|---|---|---|
| 443 | managed Traefik HTTPS（含 TCP passthrough → anytls） | `bo/mu/ph/au/sc/di`；排除 `sh/ot` |
| 80 | managed Traefik HTTP（HTTP-01 / redirect） | `bo/mu/ph/au/sc/di`；排除 `sh/ot` |
| 18443 | 旧业务保留端口（grafana/api/dns 等使用） | 当前各节点的旧 docker | 
| 8443 | sing-box anytls 监听（被 ingress passthrough 转发） | anytls 节点 hostPort |
| 6443 | k3s apiserver（仅 tailscale） | bo |
| 10250 | kubelet（仅 tailscale） | 全部 k3s 节点 |
| 22 | SSH（建议改走 tailscale ssh） | 全部 |

### 5.3 认证

| 系统 | 鉴权方式 |
|---|---|
| Rancher | 内置账号 + 后期可启用 2FA / OIDC |
| ArgoCD | 内置 admin + 后期接 GitHub OAuth |
| Grafana | 内置 admin |
| sub-store 后台 | token URL + ingress UA 限制 |
| vmagent 远程写入 | ingress basic-auth |
| anytls (sing-box) | UUID/密码 |
| sealed-secrets | 由集群内 controller 解密，master key 离线备份 |

---

## 6. 待解决与风险

### 待用户决策

- [x] **control-plane 改 bo**：bo 8C/32G/250G，已定为 server
- [x] **ph SSH 探查**：已通过 keyboard-interactive 登录，2C/2G/40G，磁盘 31%
- [x] **di 存活确认**：通过 `103.102.135.165` 登录，4C/8G/50G，`109.111.55.170` 为同机附加 IP
- [x] **mu 磁盘复核**：当前 40% used，不再是满盘状态
- [x] **ot 磁盘复核**：当前 21% used，作为受限 worker 加入；AdGuard 继续独立保留
- [x] **MVP 存储策略**：bo 本地持久卷 + R2 备份；Longhorn 第二阶段
- [x] **入口控制器选型**：禁用 k3s 默认 Traefik/ServiceLB，使用自管 managed Traefik，只调度到 `bo/mu/ph/au/sc/di`
- [x] **正式域名确认**：`k4s.live` 已确认为正式根域，Cloudflare Zone 已可用

### CF Mesh 实施待细化

- [ ] CF Mesh node 装到 6 海外节点 + mac 后，是否启用 split DNS（mesh 内 `rancher.ops.k4s.live` 解析到 mesh node IP，公网解析到公网 IP）
- [ ] CF Access 鉴权策略：是否启用 OIDC（GitHub / Google），覆盖哪些应用
- [ ] mesh node 数量额度：CF 单账户上限 50 节点，当前用 7 个（充裕）
- [ ] 跟 anytls 流量是否互相影响（anytls 走公网 443，CF Mesh 走自家 UDP，不冲突）

### 风险点

| 风险 | 说明 | 缓解 |
|---|---|---|
| bo 单点 | control-plane 单 master，挂了暂时不能调度 | 业务 pod 仍跑；定期 etcd snapshot（k3s 默认） + R2 备份 |
| MVP 本地存储单点 | bo 本地 PV 依赖 bo 存活 | 定时备份到 R2；Longhorn 第二阶段再评估 |
| 跨境延迟 | sh → bo 跨境 100-200ms | k3s 节点间走 tailscale 已规避大部分干扰；apiserver 操作偶尔慢可接受 |
| ot 受限 worker | 现有 AdGuard / Caddy / xray 保留在 host/Docker；k3s 只调度轻量非关键 pod | taint + toleration 控制调度，不跑入口/存储/管理面 |
| sh 资源紧（1G） | derp + k3s agent 共存可能 OOM | sh 上不调度业务 pod，nodeSelector 限制 |
| di 尚无 tailscale | 目前只有公网 SSH 和双 IP，未入 tailnet | k3s agent 前先安装/接入 tailscale |

### 后续扩展点（不在 MVP 范围）

- Authentik 统一 SSO
- Cloudflare R2 备份再镜像到飞牛（双备份）
- 引入 cilium 替代 flannel（如需 NetworkPolicy）
- ArgoCD ApplicationSet（多环境管理）
- VictoriaMetrics 远程长期归档到 R2

---

## 7. 改造前后对比

| 维度 | 改造前 | 改造后 |
|---|---|---|
| 编排 | docker / 散落 | k3s 集群 + ArgoCD GitOps |
| 代理协议 | vless (xray) | anytls (sing-box) |
| 入口管理 | 各节点自管端口 | 统一入口层 + cert-manager 自动证书 |
| 监控 | grafana 单点 | VM + VL + 集中告警 |
| 备份 | 无统一 | MVP: bo 本地 PV → R2；Phase2: Longhorn → R2 |
| 订阅 | 手维护节点链接 | sub-store 自动转换 |
| 密钥管理 | 配置文件 / 环境变量 | sealed-secrets 进 git |
| 客户端策略 | 散落配置 | sub-store 统一节点分组 + 策略组 |

---

## 8. 现有业务整合策略

> bo 上已跑的 docker 服务在改造时分三种处理：复用、并存、独立。

### 8.1 复用（接入 k3s）

| 现有服务 | 复用方式 | 涉及组件 |
|---|---|---|
| **authentik (bo)** | 作为 k8s 所有管理面的 **SSO/OIDC 提供者** | Rancher OIDC、ArgoCD OIDC、Grafana OIDC、sub-store 后台、未来其他管理 UI |
| **lobe-rustfs (bo, S3 兼容)** | 备选 S3 后端（与 R2 互补）。可作 Longhorn **次级备份目标** 或 内部对象存储 | Longhorn backup target、应用 S3 后端 |
| **lobe-postgres (bo, paradedb pg17)** | 可作为 k8s 应用的**外部 Postgres 后端**（authentik 已经用了；新应用如要 PG 可共享） | 任何需要 PG 的 helm chart |
| **lobe-redis (bo)** | 同上，作 redis 共享 | 任何需要 redis 的应用 |

**SSO 收益**：所有管理面（Rancher / ArgoCD / Grafana / Longhorn UI 等）统一走 authentik 登录，单点登入 + 2FA + 用户管理集中。

### 8.2 并存（双轨运行）

| 现有服务 | k3s 新装 | 关系 |
|---|---|---|
| **prometheus (bo, docker)** | **VictoriaMetrics**（k3s） | 双写：vmagent 同时 remote-write 到 VM 和现有 prometheus；前期对照，稳定后逐步只保留 VM |
| **grafana (mu, docker)** | **grafana**（k3s） | 旧 grafana 服务旧业务（如有），新 grafana 服务 k3s 监控；后期看是否合并 |
| **AdGuard (ot, docker)** | ot 同时作为受限 k3s worker | AdGuard 维持 host/Docker 独立，不迁入 k3s |

### 8.3 独立（不进 k3s）

| 服务 | 节点 | 理由 |
|---|---|---|
| **lobehub / new-api / chat** | bo / mu | 业务应用（用户/AI 工具），上 k3s 收益小，docker compose 维护更直接 |
| **AdGuard** | ot | DNS 服务继续跟 k3s 解耦；ot 即使加入集群也不迁移该服务 |
| **rustdesk-server / derper** | sh | 现状保留，sh 节点的 host 级服务 |
| **reminder_bot** | ot | 小型 bot，独立跑 |

### 8.4 不能动的现状（依赖项）

- **sh: derper**（自建 DERP）—— tailscale 集群依赖，绝不能停
- **bo: authentik**—— 改造后会成为 SSO 中心，更不能停
- **ot: AdGuard**—— 多个客户端的 DNS，停了会广泛影响
- **bo: lobe-postgres**—— 多服务共享 DB，停 = lobehub + authentik 都挂

### 8.5 与 k3s 的资源边界

```
bo 资源池（32G RAM / 250G disk）
├─ 现有 docker 业务：约 4.4G RAM / 36G disk
├─ k3s 新增组件估算：约 5-6G RAM / 30G disk
│   ├─ k3s server: 500M
│   ├─ Rancher: 1G
│   ├─ ArgoCD: 500M
│   ├─ VictoriaMetrics: 1G
│   ├─ VictoriaLogs: 500M
│   ├─ Longhorn: 1G + storage 数据
│   └─ ingress controller / cert-manager 等: 500M
└─ 余量：约 20G RAM / 180G disk —— 充足
```

---

## 9. 架构审查修正（2026-05-30，多维 review 结论）

> 本节为**权威覆盖**：与前文冲突处以本节为准。来源：4 维度（一致性/技术/完整性/安全）审查，共 47 条，去重后整合。

### 9.1 技术设计纠正（影响能否跑通，必须采纳）

**T1. anytls 入口机制——用 Traefik TCP Router + SNI passthrough（不是 ingress-nginx）**
- 入口控制器统一为 **managed Traefik**（前文残留的 `ingress-nginx` 字样作废）。
- Traefik **能**按 SNI 做 passthrough（审查 agent 曾误判其不能）：
  ```yaml
  # TCP Router（IngressRouteTCP）
  tcp:
    routers:
      anytls:
        entryPoints: ["websecure"]   # :443
        rule: "HostSNI(`*.edge.k4s.live`)"
        tls:
          passthrough: true          # 不终结 TLS，原样转发
        service: singbox-anytls
  ```
- passthrough 模式下 **TLS 证书在 sing-box 手里**，不是 cert-manager 管的 Traefik。`*.edge` 证书需 cert-manager 签发后挂进 sing-box pod（配 reloader 自动重启），并设到期告警。
- `*.ops / *.obs / *.client` 走 Traefik **TLS 终结**（cert-manager 正常签发），与 `*.edge` 的 passthrough 分流。

**T2. sing-box 用 Deployment + nodeSelector，不用 DaemonSet + hostPort**
- DaemonSet 会在 sh/ot 也起 pod 抢端口、调度不一致；hostPort + `internalTrafficPolicy:Local` 有 hairpin DNAT 风险。
- 正解：sing-box 部署为 **Deployment + nodeSelector**（只钉 `mu/bo/ph/au/sc/di`）+ ClusterIP Service；Traefik passthrough 指向该 Service。

**T3. flannel over tailscale 必须降 MTU**
- tailscale(WireGuard) MTU 1280，flannel VXLAN 再套约 50B → 双层封装大包静默丢，pod 跨节点间歇不通。
- 正解：k3s 启动 `--flannel-iface=tailscale0`，并把 flannel MTU 显式设为 **1230**。部署后用 `ping -M do -s 1200 <pod-ip>` 验证。Phase2 如不稳再评估 Cilium。

**T4. ot 受限调度用 label+nodeAffinity，不用裸 NoSchedule taint**
- 自定义 taint `restricted=true:NoSchedule` 会挡住 flannel/kube-proxy/vmagent 等必须全节点跑的系统 DaemonSet（它们不 tolerate）→ ot 上 pod 拿不到 IP。
- 正解：用 `node-type=restricted` **label** + 业务 pod 的 nodeAffinity 做软排除；若坚持 taint，必须给所有系统 DaemonSet 显式加 toleration。

**T5. k3s 安装参数（补全前文缺口）**
- 所有节点：`--node-ip=<tailscale-100.x>`、`--flannel-iface=tailscale0`。
- control-plane(bo)：`--cluster-init`、`--tls-san` 至少含 `100.120.204.107`(ts) + `45.128.220.149`(公网) + `bo.taila41d4.ts.net`。
- etcd 防膨胀：`--etcd-arg=quota-backend-bytes=2147483648`（2G）+ 默认每日 snapshot 推 R2。
- kubeconfig：导出 `/etc/rancher/k3s/k3s.yaml`，server 改 `https://bo.taila41d4.ts.net:6443`（依赖 MagicDNS）或 `https://100.120.204.107:6443`；mac 走 tailscale 访问。

**T6. bo 单点 + 单 etcd（MVP 接受，但补防护）**
- MVP 维持 bo 单 master；必须：etcd 每日 snapshot→R2、给 bo 上 docker 业务设 `--memory` limit 防被 k3s OOM、关键有状态 pod 配 PodDisruptionBudget。
- Phase2 评估把 di（4C8G）提为第二 server 组 etcd（但跨境 etcd 抖动需实测）。

### 9.2 安全闸（go-live 前必须，详见 review 全量清单）

- **anytls 密钥必须全新生成**：客户端旧配置里的 vless UUID 已公开，复用=没换钥匙。sing-box 强制 TLS1.3。
- **管理面 `*.ops` 不得裸公网**：上线前接 authentik OIDC 或 CF Access；未就绪前只走 tailscale 内网（不建公网 DNS）。
- **凭据**：`k3s-prep.md` 明文凭据用完即轮换并迁密码库；R2 bucket 开 Object Lock + 版本控制防备份被删；备份开 SSE 加密；sealed-secrets master key 离线 + 1Password(MFA)。
- **authentik 是 SSO 单点**：bo docker 上的 authentik 要纳入 R2 备份与恢复演练。

### 9.3 用户决策（2026-05-30 已敲定）

- **Q7 现有 docker 服务 → 逐步迁入 k3s**：Phase1 保留独立运行不动；后续分批改成 k3s 部署（authentik/new-api/grafana 等按依赖顺序迁）。迁移顺序与回滚方案待 Phase1 跑稳后单列。
- **sh standalone anytls → 跑**：sh 在 k3s 外、独立高端口跑 anytls 作境内中转入口（不抢 derper 的 443，不进 k3s 调度）。与 6 台海外 anytls 共用同一套新 UUID/证书。⚠️ 注意 sh 为个人实名 IP，跑代理有合规风险，自用自担。
- **CF Mesh → Phase 2**：Phase1 只用 tailscale；集群稳定后再叠加 CF Mesh + CF Access。§2.2 整段降级为 Phase2 规划。
- **anytls 密钥 → 全新生成 + cert-manager 证书**：每节点新 UUID（不复用旧 vless），`*.edge.k4s.live` 证书由 cert-manager 签发挂进 sing-box；旧 vless UUID 视为已泄露作废。
- **待补（非阻塞）**：Q4 JP fallback 机场清单；sub-store 服务端节点来源/分组/订阅鉴权落地。

### 9.4 一致性修正记录

- §2.2 双路径图 `ingress-nginx` → `managed Traefik`（已改）
- §8.5 资源估算含已延后的 Rancher/Longhorn：MVP 实际约 **3-4G RAM**（不含这两者），完整估算见 Phase2
- §4.5 “Longhorn 每日全量+增量”为 **Phase2** 备份流；MVP 实际为 bo 本地 PV→R2
- §3 sh `[standalone anytls]` 矛盾 → 已统一：sh **在 k3s 外**跑独立高端口 anytls（§3 第162行成立）；§5.1.1“sh 不跑 anytls”应理解为“sh 不在 k3s 入口层跑 anytls、不纳入 *.edge 域名”，但有独立中转入口
- bo 双重身份（control-plane + hk1.edge）：control-plane 组件与 anytls 通过 namespace + NetworkPolicy 隔离

---

## 10. 部署日志

### Phase 1 / Step 1-3（2026-05-30）✅ k3s 集群就绪

- **k3s 版本**：v1.35.5+k3s1（全节点一致）
- **control-plane**：bo（节点名 `leo-hk`），`--cluster-init` 启用 embedded etcd，禁用内置 traefik/servicelb
- **8 节点全 Ready**，k8s 通信走 tailscale，节点名/角色/region：
  | 节点名 | region | role | anytls | storage |
  |---|---|---|---|---|
  | leo-hk(bo) | hk | control-plane | ✓ | ✓ |
  | mu | sg | edge | ✓ | |
  | ph | us | edge | ✓ | |
  | au | us | edge | ✓ | |
  | sc | us | edge | ✓ | |
  | di | fr | edge | ✓ | |
  | ot | hk | restricted | | |
  | sh | cn | relay | | |
- **网络验证**：flannel over tailscale，`flannel.1` MTU 自动 = 1230（tailscale0 1280 − 50 VXLAN）；跨境 pod-to-pod(bo↔ph) 0 丢包 144ms；1200B 通过 / 1400B 干净失败（无黑洞）；CoreDNS 正常。→ **T3 风险自动解除，无需手动配 MTU**
- **bo docker 共存验证**：装 k3s 后 10 个生产容器全部健康（postgres/redis/authentik/lobehub/grafana 等），iptables 仅增不删（3966→4062），零扰动
- **kubeconfig**：mac `~/.kube/k4s-config`，server=`https://100.120.204.107:6443`（走 tailscale），本地 kubectl 正常
- **sh 特殊处理**：CN 镜像安装（github 在境内限速）；kubelet 加 system-reserved=150Mi + eviction-hard 保护 derper
- **备份**：装 k3s 前 bo postgres 全库 dump（19MB，4 库）+ authentik 数据 → bo 本地 + mac `~/k3s-backups/` + R2 `bo-pre-k3s/20260530/`（3 副本）

### 待办（Step 4-8）
- [ ] Step 4：cert-manager(CF DNS-01) + managed Traefik + sealed-secrets
- [ ] Step 5：ArgoCD + repo 骨架
- [ ] Step 6：anytls（sing-box Deployment + 新 UUID + *.edge 证书 + Traefik TCPRoute passthrough）+ sh 独立 anytls
- [ ] Step 7：VictoriaMetrics/Logs/Grafana + vmagent/vector + alertmanager→TG
- [ ] Step 8：sub-store
- [ ] MagicDNS 启用（你点）

### Phase 1 / Step 4（2026-05-31）✅ 核心基建就绪

- **sealed-secrets**：controller 运行于 bo（kube-system，fullname=sealed-secrets-controller）
- **cert-manager**：钉 bo，ClusterIssuer `letsencrypt-staging` + `letsencrypt-prod`（CF DNS-01，token=cloudflare-api-token@cert-manager）
- **DNS-01 全链路验证**：staging 签 `*.ops.k4s.live` 通过；prod 通配证书 `ops-wildcard-tls`@traefik 有效期至 2026-08-28
- **Traefik v3.7.1**：DaemonSet + hostNetwork，仅 `node.k4s/ingress=true` 节点（当前仅 bo）；root+NET_BIND_SERVICE 绑 80/443；metrics 改 **9201**（避开 bo 上 node-exporter:9100 / xray_exporter:9101）
- **端到端验证**：di(FR) 外部 curl `https://whoami.ops.k4s.live` → 公网DNS→bo:443→Traefik→pod，HTTP 200 + LE 生产证书 ✓（测试资源已清理）

**运维要点（重要）**：
- **bo 公网 IP 22 被 fail2ban**（本轮 SSH 太频繁）→ 运维一律走 bo tailscale IP `100.120.204.107`
- ingress 当前仅 bo；mu/ph/sc 的 443 被旧 xray 占用，待 Step 6 停旧 xray 后接管；au/di 的 443 空闲但暂未启用 ingress
- helm 在 bo 上运行（`/etc/rancher/k3s/k3s.yaml`），chart 走 bo 直连

### Phase 1 / Step 5（2026-05-31，进行中）ArgoCD + GitOps

**5.1 ArgoCD 安装**
- helm `argo/argo-cd` → ns `argocd`，全组件 `global.nodeSelector` 钉 bo(leo-hk)
- `configs.params.server.insecure=true`（TLS 由 Traefik 终结），dex/notifications 关闭
- pods 全 Running：application-controller / applicationset / redis / repo-server / server
- admin 初始密码：`DnUMpvOzQnOGgVqt`（登录后应改，记入密码库）

**5.2 UI 暴露**
- IngressRoute `argocd`@traefik → svc `argo-cd-argocd-server:80`（跨 ns，allowCrossNamespace）+ tls `ops-wildcard-tls`
- DNS `argocd.ops.k4s.live` → 45.128.220.149（bo 公网）
- 验证：di 外部 `https://argocd.ops.k4s.live/healthz` = HTTP 200 + LE 证书 ✓

**5.3 仓库连接（只读 Deploy Key，安全收敛）**
- bo→github.com:22 可达；在 bo 生成 ed25519 deploy key
- 公钥加到 repo（`read_only=true`，title=argocd-readonly）；私钥拉取验证成功（HEAD 匹配）
- ArgoCD repository secret `repo-k3s-infra`@argocd（type=git，SSH 私钥），私钥临时文件已 shred
- → ArgoCD 拉取走 SSH deploy key（只读），不再用宽权限 classic token

**5.4 app-of-apps 骨架**（进行中）
