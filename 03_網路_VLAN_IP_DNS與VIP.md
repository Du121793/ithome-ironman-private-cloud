# 網路、VLAN、IP、DNS 與 VIP

## 網路區域

| 區域 | VLAN | 網段 | Gateway | DHCP | 經 OPNsense |
|---|---:|---|---|---|---|
| Router Bootstrap／Rescue | 無 | 現有路由器 LAN | 現有路由器 | 建置期使用 | 否 |
| Management | 10 | `10.77.10.0/24` | `10.77.10.1` | 關閉 | 是 |
| Service | 20 | `10.77.20.0/24` | `10.77.20.1` | 關閉 | 是 |
| Database | 30 | `10.77.30.0/24` | `10.77.30.1` | 關閉 | 是 |
| Backup | 40 | `10.77.40.0/24` | `10.77.40.1` | 關閉 | 是 |
| Bastion DMZ | 50 | `10.77.50.0/24` | `10.77.50.1` | 關閉 | 是 |
| OpenVPN Tunnel | 非 VLAN | `10.77.60.0/24` | OPNsense 管理 | VPN Pool | OPNsense 終結 |
| Ceph | 非 VLAN | `10.77.70.0/24` | 無 | 關閉 | 否 |
| Corosync | 非 VLAN | `10.77.80.0/24` | 無 | 關閉 | 否 |

`192.168.100.0/24` 已被其他環境使用，本專案不採用。部署前仍須確認 `10.77.0.0/16` 不與公司網路、家用 LAN、其他 VPN、Docker 或 WSL 衝突。

## PVE 與基礎設施位址

| 名稱 | FQDN | 位址 | Gateway／說明 |
|---|---|---|---|
| pve-l0 | `pve-l0.lab.home` | Router DHCP 後依 MAC 固定；本次曾為 `192.168.0.146/24` | 保留上游路由器 Gateway，供 L0 與 Console 救援 |
| pve01 Bootstrap | `pve01.lab.home` | 本次 `192.168.0.191/24` | Day 15 後移除 Default Gateway，只供同網段救援 |
| pve02 Bootstrap | `pve02.lab.home` | 依路由器固定租約記錄 | Day 15 後移除 Default Gateway |
| pve03 Bootstrap | `pve03.lab.home` | 依路由器固定租約記錄 | Day 15 後移除 Default Gateway |
| pve01 Management | `pve01.lab.home` | `10.77.10.11/24` | `10.77.10.1` |
| pve02 Management | `pve02.lab.home` | `10.77.10.12/24` | `10.77.10.1` |
| pve03 Management | `pve03.lab.home` | `10.77.10.13/24` | `10.77.10.1` |
| pve01 Ceph | — | `10.77.70.11/24` | 無 Gateway |
| pve02 Ceph | — | `10.77.70.12/24` | 無 Gateway |
| pve03 Ceph | — | `10.77.70.13/24` | 無 Gateway |
| pve01 Corosync | — | `10.77.80.11/24` | 無 Gateway |
| pve02 Corosync | — | `10.77.80.12/24` | 無 Gateway |
| pve03 Corosync | — | `10.77.80.13/24` | 無 Gateway |
| fw01 Management | `fw01.lab.home` | `10.77.10.1/24` | OPNsense Management／LAN |

fw01 WAN 由上游路由器 DHCP／固定租約取得；`192.168.0.84` 只是本次實測值，指令與文章不得假定永遠不變。

## Service、Database、Backup 與 Bastion

| 名稱／服務 | FQDN | 位址 | VLAN |
|---|---|---|---:|
| Web VIP | `web.lab.home` | `10.77.20.10` | 20 |
| DB RW VIP | `db-rw.lab.home` | `10.77.20.11` | 20 |
| DB RO VIP | `db-ro.lab.home` | `10.77.20.12` | 20 |
| proxy01 | `proxy01.lab.home` | `10.77.20.21` | 20 |
| proxy02 | `proxy02.lab.home` | `10.77.20.22` | 20 |
| app01 | `app01.lab.home` | `10.77.20.31` | 20 |
| app02 | `app02.lab.home` | `10.77.20.32` | 20 |
| client01 | `client01.lab.home` | `10.77.20.41` | 20 |
| monitor01 | `monitor01.lab.home` | `10.77.20.51` | 20 |
| ca01 | `ca01.lab.home` | `10.77.30.10` | 30 |
| pg01／etcd01 | `pg01.lab.home` | `10.77.30.11` | 30 |
| pg02／etcd02 | `pg02.lab.home` | `10.77.30.12` | 30 |
| pg03／etcd03 | `pg03.lab.home` | `10.77.30.13` | 30 |
| pg-restore01 | `pg-restore01.lab.home` | `10.77.30.21` | 30 |
| backup01 | `backup01.lab.home` | `10.77.40.11` | 40 |
| jump01 | `jump01.lab.home` | `10.77.50.11` | 50 |

三組 VIP 由 proxy01／proxy02 的 Keepalived 持有，不配置在 OPNsense。OPNsense 只負責把獲准的流量路由或 NAT 到 VIP。

## 公開 DNS 與 Origin

| 項目 | 設定 |
|---|---|
| 網域註冊商 | GoDaddy |
| 權威 DNS | Cloudflare Full Setup |
| 公開 Hostname | 以 `app.example.com` 作為文件佔位，實作必須改成自己的真實網域 |
| 公開 Record | Proxied A Record，指向 Origin 公網 IPv4 |
| Origin IPv4 | 以當次實際公網 IPv4 為準，不寫入 Git |
| Origin 內部目標 | Web VIP `10.77.20.10:443` |
| TLS Mode | Cloudflare Full (strict) |

`web.lab.home` 是內部 DNS 名稱，只解析到私有 Web VIP；公開 Hostname 由 Cloudflare 權威 DNS 回答。兩者不可交換使用。

## OpenVPN 與命名規則

| 項目 | 設定 |
|---|---|
| Tunnel Network | `10.77.60.0/24` |
| OPNsense Tunnel Gateway | `10.77.60.1` |
| OpenVPN Server Certificate | `vpn.lab.home` |
| 推送路由 | `10.77.10.0/24`、`10.77.20.0/24`；不推送整個 `10.77.0.0/16` |
| DNS Server | `10.77.10.1` |
| DNS Search Domain | `lab.home` |
| Redirect Gateway | 不啟用，採 Split Tunnel |

內部 DNS Domain 統一使用 `lab.home`，不得混用 `lab.internal` 或 `home.lab`。Day 27 的公開 Web 入口則使用自己持有的真實網域。FQDN 的 DNS Record、TLS SAN、Nginx `server_name` 與應用程式設定必須一致。

`ca01` 與 pg01～03 位於相同 VLAN 30，因此它們之間的 Layer 2 Traffic 不會經過 OPNsense。Lab 使用 ca01 的 nftables Host Firewall 限制來源與 Port；正式環境建議把 CA LXC 放入獨立 PKI／Security VLAN。
