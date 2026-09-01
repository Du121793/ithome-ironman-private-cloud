# Port、NAT 與流量矩陣

本表記錄最終需要的流量。是否已建立規則仍以對應實作日為準；未列出的跨 VLAN 流量由 Default Deny 阻擋。

## OPNsense Alias 最終內容

| Alias | Type | 內容 |
|---|---|---|
| `INTERNAL_NETWORKS` | Network(s) | `10.77.10.0/24`、`10.77.20.0/24`、`10.77.30.0/24`、`10.77.40.0/24`、`10.77.50.0/24` |
| `PVE_NODES` | Host(s) | `10.77.10.11`、`10.77.10.12`、`10.77.10.13` |
| `PROXY_NODES` | Host(s) | `10.77.20.21`、`10.77.20.22` |
| `APP_NODES` | Host(s) | `10.77.20.31`、`10.77.20.32` |
| `PG_NODES` | Host(s) | `10.77.30.11`、`10.77.30.12`、`10.77.30.13` |
| `CA_NODE` | Host(s) | `10.77.30.10` |
| `BACKUP_NODE` | Host(s) | `10.77.40.11` |
| `BASTION_HOST` | Host(s) | `10.77.50.11` |
| `BASTION_TARGETS` | Host(s) | proxy01／02、app01、monitor01、pg01～03、backup01 的固定 IP |
| `WEB_VIP` | Host(s) | `10.77.20.10` |
| `DB_RW_VIP` | Host(s) | `10.77.20.11` |
| `DB_RO_VIP` | Host(s) | `10.77.20.12` |
| `CLOUDFLARE_IPV4` | URL Table (IPs) | `https://www.cloudflare.com/ips-v4`，每日更新 |
| `VPN_ADMIN_CLIENTS` | OpenVPN group | `vpn-admin` 目前已連線成員的 Tunnel IP |
| `WEB_PORTS` | Port(s) | `80`、`443` |
| `PGSQL_PORT` | Port(s) | `5432` |
| `PATRONI_API` | Port(s) | `8008` |
| `ETCD_PORTS` | Port(s) | `2379`、`2380` |
| `STEP_CA_PORT` | Port(s) | `9000` |
| `BASIC_OUTBOUND_TCP` | Port(s) | `80`、`443` |

`BASTION_TARGETS` 是網路層可到達上限；Alice、Bob 等個別帳號仍由 jump01 的 SSH `PermitOpen` 進一步縮小到各自獲准的主機。

## 外部與管理入口

| 來源 | 目的 | Protocol／Port | 行為 | 建立日 |
|---|---|---|---|---:|
| WAN | fw01 OpenVPN | UDP 1194 | Allow；建立 VPN Tunnel | 17 |
| WAN | fw01 WAN 45222 → jump01 22 | TCP 45222 DNAT 至 TCP 22 | 公開受限 Bastion SSH | 15 |
| Cloudflare `CLOUDFLARE_IPV4` | Web VIP | TCP 443 | DNAT＋Allow；其他 WAN 來源由 Default Deny 拒絕 | 27 |
| VPN Admin | pve01～03 | TCP 8006 | Allow PVE Web UI | 17 |
| VPN Client | pve01～03 | TCP 22／其他 | Block | 17 |
| VPN Client | pg01～03 | any | 明確 Block，不直連 DB／Patroni／etcd | 17 |
| VPN 授權群組 | DB RW／RO VIP | TCP 5432 | 依群組放行 | 26 |

## PVE Management 出口

| 來源 | 目的 | Protocol／Port | 行為 |
|---|---|---|---|
| PVE_NODES | fw01 LAN Address | TCP/UDP 53 | DNS Allow |
| PVE_NODES | fw01 LAN Address | ICMP | Gateway 診斷 Allow |
| PVE_NODES | Internet／NTP 來源 | UDP 123 | NTP Allow |
| PVE_NODES | INTERNAL_NETWORKS | any | Block，避免 PVE 任意跨區 |
| PVE_NODES | Internet | TCP 80、443 | 套件更新 Allow |
| PVE_NODES | 其他目的 | any | Block |

## 內部服務

| 來源 | 目的 | Protocol／Port | 用途 |
|---|---|---|---|
| LAB_INTERNAL | fw01 | TCP/UDP 53 | DNS |
| LAB_INTERNAL | fw01 | UDP 123 | NTP |
| LAB_INTERNAL | Internet | TCP 80、443 | 套件與更新；先排除內部網段 |
| jump01 | BASTION_TARGETS | TCP 22 | 受 Alias 與 SSH `PermitOpen` 限制的管理連線 |
| proxy01／02 | app01／02 | TCP 8080 | Nginx 到 Flask Backend |
| proxy01／02 | pg01～03 | TCP 5432 | HAProxy 到 PostgreSQL |
| proxy01／02 | pg01～03 | TCP 8008 | HAProxy 依 Patroni Role 做 Health Check |
| pg01～03 | pg01～03 | TCP 5432 | PostgreSQL Streaming Replication |
| pg01～03 | pg01～03 | TCP 2379 | etcd Client API |
| pg01～03 | pg01～03 | TCP 2380 | etcd Peer |
| pg01～03 | ca01 | TCP 9000 | 使用一次性 Token 申請／更新 PostgreSQL 與 etcd Certificate |
| pg01～03 | backup01 | TCP 22 | pgBackRest Archive／Restore SSH |
| backup01 | pg01～03 | TCP 22 | pgBackRest 讀取 Primary／Standby |
| pg-restore01 | backup01 | TCP 22 | PITR Restore；不接受 Application 流量 |
| proxy01 ↔ proxy02 | 對方 Unicast IP | IP Protocol 112（VRRP） | Keepalived VIP 狀態；不是 TCP／UDP Port |

## Proxy 與 VIP 監聽

| 位址 | Port | 服務 |
|---|---:|---|
| proxy01／02 Service IP | TCP 80／443 | Nginx 內部 HTTP／Origin HTTPS Health |
| Web VIP `10.77.20.10` | TCP 80／443 | 內部 HTTP 與 Cloudflare Origin HTTPS 入口 |
| proxy01／02 Service IP | TCP 5432 | HAProxy RW 測試入口 |
| proxy01／02 Service IP | TCP 5433 | HAProxy RO 測試入口 |
| DB RW VIP `10.77.20.11` | TCP 5432 | Primary 入口 |
| DB RO VIP `10.77.20.12` | TCP 5432 | Replica 入口 |

## 監控 Port

| 來源 | 目的 | TCP Port | 用途 |
|---|---|---:|---|
| monitor01 | Debian VM／節點 | 9100 | Node Exporter |
| monitor01 | pg01～03 | 8008 | Patroni Metrics／Status |
| monitor01 | pg01～03 | 9187 | PostgreSQL Exporter |
| monitor01 | proxy01／02 | 8405 | HAProxy Prometheus Exporter |
| monitor01 | Cloudflare 公開 Hostname | TCP 443 | 端到端 Blackbox HTTPS Probe |
| VPN 授權管理者 | monitor01 | 3000 | Grafana |
| monitor01 本機 | 9090 | Prometheus |
| monitor01 本機 | 9093 | Alertmanager |

Ceph `10.77.70.0/24` 與 Corosync `10.77.80.0/24` 是無 Gateway 封閉網路，不建立 OPNsense 跨 VLAN 規則。其 Daemon Port 以部署當下產生的 Ceph／Corosync 設定與官方版本文件為準，不將未實測範圍硬寫成防火牆白名單。

`pg01～03 → ca01 TCP 9000` 是同 VLAN 流量，OPNsense 看不到。實際 Allow 由 ca01 的 nftables Host Firewall 建立；ca01 不加入 `BASTION_TARGETS`、不接受 SSH，也不得主動連線 PostgreSQL 5432、Patroni 8008 或 etcd 2379／2380。
