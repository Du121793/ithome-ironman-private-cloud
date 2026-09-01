# Bridge、NIC 與 VLAN 接線

## L0 Bridge

| L0 介面 | VLAN-aware | Host IP | 用途 |
|---|---|---|---|
| `vmbr0` | 否 | Router DHCP 後固定 | pve-l0 Management／Rescue、Nested PVE Bootstrap、fw01 WAN |
| `vmbr2` | 是 | 不直接配置服務 IP | VLAN 10～50 Internal Trunk |
| `vmbr2.10` | VLAN 子介面 | `10.77.10.10/24`，無 Gateway | L0 到 OPNsense Management 的受限管理路徑 |
| `vmbr4` | 否 | 無 | Nested Ceph 封閉網路 |
| `vmbr5` | 否 | 無 | Nested Corosync 封閉網路 |

L0 不建立 `vmbr3`。`vmbr2.10` 是掛在 `vmbr2` 上的 VLAN 10 子介面，不是另一座 Linux Bridge。

## 每台 Nested PVE 在 L0 的四張 NIC

| L0 VM NIC | L0 Bridge | VLAN Tag | L1 用途 |
|---|---|---|---|
| net0 | vmbr0 | 無 | Day 03～15 Router Bootstrap；Day 15 後只保留無 Gateway 救援 |
| net1 | vmbr2 | 無 | L1 Guest VLAN Trunk 與 Host Management VLAN 10 |
| net2 | vmbr4 | 無 | Ceph Storage Network |
| net3 | vmbr5 | 無 | Corosync Link 0 |

## L1 PVE 內部介面

| L1 介面 | 連接來源 | 位址 | 用途 |
|---|---|---|---|
| `vmbr0` | net0 對應 VirtIO NIC | Router Bootstrap 固定租約 | 建置／救援，不再持有 Day 15 後 Default Gateway |
| `vmbr1` | net1 對應 VirtIO NIC | 無 | VLAN-aware Guest Trunk，允許 VLAN 2～4094 |
| `vmbr1.10` | vmbr1 VLAN 子介面 | pve01～03 為 `10.77.10.11`～`.13/24` | 正式 PVE Management；Gateway／DNS `10.77.10.1` |
| `vmbr2` | net2 對應 VirtIO NIC | `10.77.70.11`～`.13/24` | Ceph；無 Gateway |
| 第四張 VirtIO NIC | net3 | `10.77.80.11`～`.13/24` | Corosync Link 0；無 Gateway；不另建 Bridge |

介面名稱必須以 MAC Address 與 `ip -br link` 對照；pve01 實測第四張 NIC 是 `ens21`，但不可假定 pve02、pve03 名稱相同。

## fw01 三張 NIC

| fw01 NIC | L0 Bridge | PVE VLAN Tag | OPNsense 角色 |
|---|---|---:|---|
| net0／`vtnet0` | vmbr0 | 無 | WAN，向上游路由器 DHCP 取址 |
| net1／`vtnet1` | vmbr2 | 10 | Management／LAN，直接設 `10.77.10.1/24` |
| net2／`vtnet2` | vmbr2 | 無 | VLAN 20～50 Trunk |

OPNsense 在 `vtnet2` 上建立 VLAN 20、30、40、50；VLAN 10 已由 PVE Tag 10 交給未標記的 `vtnet1`，不得在 `vtnet2` 重複建立。

## L2 Guest NIC

| VLAN Tag | VM |
|---:|---|
| 20 | proxy01、proxy02、app01、app02、client01、monitor01 |
| 30 | ca01、pg01、pg02、pg03、pg-restore01 |
| 40 | backup01 |
| 50 | jump01 |

所有 Guest NIC 接 L1 `vmbr1`。服務 VM 原則上只接一張 NIC，跨 VLAN 流量交給 OPNsense，不用多張 NIC 繞過防火牆。
