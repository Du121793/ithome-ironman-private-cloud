# Day 29｜在使用者回報前看見異常：Prometheus、Grafana、Exporter 與 Alertmanager 告警

對應文章：[Day 29｜在使用者回報前看見異常：Prometheus、Grafana、Exporter 與 Alertmanager 告警](https://ithelp.ithome.com.tw/users/20183351/ironman/9461)

本日沿用 Day 16 已建立的 monitor01，安裝 Prometheus、Grafana 與 Alertmanager，收集 Linux、Patroni、PostgreSQL、HAProxy，並從公開 Hostname 探測 Day 27 建立的 Cloudflare HTTPS。本次以單台 VM 完成監控流程，監控平台高可用性列為正式環境延伸。

## 本日操作順序

1. 只在 monitor01 確認容量與網路，再安裝 Prometheus、Grafana、Alertmanager。
2. 逐台在被監控的 Debian VM 安裝 Node Exporter。ca01 透過 PVE Console 操作，先做一台確認 Target Up，再擴到其他主機。
3. 逐台在 pg01、pg02、pg03 安裝 PostgreSQL Exporter。
4. 逐台在 proxy01、proxy02 啟用 HAProxy Metrics。
5. 回到 monitor01 寫入 Prometheus Scrape Config，並先在本機驗證設定與服務。
6. 在 OPNsense 授權 `vpn-monitor01` 存取監控介面，連上 VPN 後才從管理電腦檢查 Prometheus Target。
7. 在 monitor01 加入 Alert Rule、Alertmanager 本機路由與 Grafana Data Source。
8. 一次停止一個測試目標，確認 Pending、Firing、Alertmanager Active Alert 與恢復後的清除流程，再立即恢復服務。

## 1. 確認 monitor01

**操作節點：只在 monitor01。不要重新 Clone VMID 251。**

- VMID／Name：`251`／`monitor01`。
- Node：pve02。
- vCPU／RAM：2／2GB。
- Disk：24GB。
- VLAN／IP：VLAN 20／`10.77.20.51/24`。
- Gateway／DNS：`10.77.20.1`。

進入既有 monitor01，確認：

```bash
hostnamectl --static
ip -br address
cloud-init status --long
resolvectl status
resolvectl query deb.debian.org
```

Hostname 應為 `monitor01`，IP 應為 `10.77.20.51/24`。Day 16 建立的 Bob SSH 帳號與 Key 保留。

`resolvectl status` 必須在 Global 或網路介面區塊看到 DNS Server `10.77.20.1` 與 DNS Domain `lab.home`。若缺少，先建立持久化設定：

```bash
sudo install -d -m 0755 /etc/systemd/resolved.conf.d
printf '%s\n' \
  '[Resolve]' \
  'DNS=10.77.20.1' \
  'Domains=lab.home' |
  sudo tee /etc/systemd/resolved.conf.d/10-lab-dns.conf
sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches
resolvectl status
resolvectl query deb.debian.org
```

不要直接修改 `/etc/resolv.conf`。確認名稱解析正常後才安裝監控套件。

```bash
sudo apt update
sudo apt install -y dnsutils prometheus prometheus-alertmanager prometheus-blackbox-exporter prometheus-node-exporter
```

## 2. 安裝 Grafana OSS

**操作節點：只在 monitor01。Prometheus 與 Alertmanager 也集中在此節點。**

依 Grafana 官方 APT Repository：

```bash
sudo apt install -y apt-transport-https wget gnupg
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc
echo 'deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main' | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
apt-cache policy grafana
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

`apt-cache policy grafana` 的 Candidate 應來自 `https://apt.grafana.com`。安裝後先在 monitor01 確認服務、TCP 3000、Grafana Health API 與本機資料庫：

等待就緒後:

```bash
systemctl is-enabled grafana-server
systemctl is-active grafana-server
sudo ss -lntp | grep ':3000'
curl -fsS http://127.0.0.1:3000/api/health
sudo journalctl -u grafana-server -b --no-pager |
  grep -E 'Connecting to DB|dbtype=sqlite3' | tail
sudo stat /var/lib/grafana/grafana.db
```

預期服務顯示 `enabled`、`active`，Health API 的 JSON 包含 `"database":"ok"`，Log 顯示 `dbtype=sqlite3`，而且 `/var/lib/grafana/grafana.db` 存在。這只能確認 Grafana 在 monitor01 本機正常啟動；管理電腦要等第 6.1～6.3 節建立 OPNsense 授權規則並連上 VPN 後才能登入。

![monitor01 的 Grafana 服務、Health API 與 SQLite 資料庫檢查](../../source/Day29/day29-main-fig01.png)

圖（一）Grafana 服務已啟用並正常執行，Health API 與 SQLite 資料庫也可正常讀取

本 Lab 是單一 Grafana Instance，可使用套件預設的 SQLite 保存使用者、Data Source 與 Dashboard。正式 Grafana HA 需要多個 Grafana Instance 共用外部 PostgreSQL 或 MySQL；每台各自使用本機 SQLite 會保存不同狀態，無法形成一致的 HA 服務。

## 3. 逐台安裝 Node Exporter

**操作節點：所有要納管的 Debian VM。先挑一台完成安裝與遠端連線測試，再逐台擴充。**

在 proxy、app、pg、jump、backup：

```bash
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
curl -fsS http://127.0.0.1:9100/metrics | sed -n '1,10p'
```

OPNsense 只允許 monitor01 `10.77.20.51` 到各 VLAN Node Exporter TCP 9100；其他來源拒絕。因部分主機與 monitor01 同 VLAN，還要用 Host Firewall 限制來源。

![Node Exporter 服務與本機 Metrics Endpoint 檢查](../../source/Day29/day29-main-fig02.png)

圖（二）Node Exporter 已啟用並正常執行，本機 Metrics Endpoint 可輸出指標

### 3.1 放行 monitor01 讀取 jump01 Node Exporter

jump01 在 Day 15 已使用 `policy drop` 的 nftables Host Firewall。安裝 Node Exporter 後，必須在現有的 Input Chain 放行 monitor01，否則 Prometheus 無法連入 TCP 9100。從 PVE Console 或現有的授權 SSH Session 登入 jump01，先確認服務正常：

```bash
systemctl is-active prometheus-node-exporter
sudo ss -lntp | grep ':9100'
curl -fsS http://127.0.0.1:9100/metrics | sed -n '1,2p'
```

備份並開啟 Host Firewall 設定：

```bash
sudo cp -a /etc/nftables.conf /etc/nftables.conf.before-day29
sudo nano /etc/nftables.conf
```

在 `table inet filter` 的 `chain input` 內，放在 Chain 結束的 `}` 之前，加入：

```nftables
ip saddr 10.77.20.51 tcp dport 9100 accept
```

保留既有的 Loopback、Established Connection、ICMP 與 SSH Rule。儲存後檢查並重新載入：

```bash
sudo nft --check --file /etc/nftables.conf
sudo systemctl reload nftables
sudo nft list chain inet filter input
```

回到 monitor01 確認可以讀取 jump01 的 Metrics：

```bash
curl -fsS --connect-timeout 3 http://10.77.50.11:9100/metrics |
  sed -n '1,2p'
```

預期輸出以 `# HELP` 與 `# TYPE` 開頭的 Metrics。

### 3.2 納管 ca01 與憑證期限

ca01 已停用網路 SSH，因此使用 PVE Console 登入 VMID 224：

```bash
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
systemctl is-active step-ca prometheus-node-exporter
curl -fsS http://127.0.0.1:9100/metrics | sed -n '1,10p'
```

在 ca01 的 `/etc/nftables.conf` Input Chain 加入下列規則，位置放在最後 Drop 之前：

```bash
sudo nano /etc/nftables.conf
```

```nftables
ip saddr 10.77.20.51 tcp dport 9100 accept
```

套用並檢查：

```bash
sudo nft -c -f /etc/nftables.conf
sudo systemctl reload nftables
sudo nft list ruleset
```

OPNsense 的 `LAB_INTERNAL` Group Rule 只允許 monitor01 前往 `CA_NODE:9100`。這段是 VLAN 20 到 VLAN 30 的跨 VLAN 流量，先由 OPNsense 過濾，抵達 ca01 後再由 nftables 檢查來源與 Port。Target Down 時依序檢查 OPNsense Rule 與 ca01 nftables。

CA 暫時離線與憑證即將到期是兩種不同事件。至少先保留以下檢查結果，之後可交給 Node Exporter Textfile Collector 或既有憑證監控工具定期輸出：

```bash
sudo -u step-ca curl --cacert /var/lib/step-ca/certs/root_ca.crt \
  https://ca01.lab.home:9000/health
sudo -u step-ca openssl x509 -checkend 2592000 -noout \
  -in /var/lib/step-ca/certs/intermediate_ca.crt
echo "intermediate_ca check exit=$?"
```

Health API 預期回傳 `{"status":"ok"}`。憑證有效期超過未來 30 天時，OpenSSL 會顯示 `Certificate will not expire`，而 Exit Code 是 `0`；顯示 `Certificate will expire` 或非 `0` 時才需要處理續期。

![ca01 的 Health API 與中繼 CA 憑證期限檢查](../../source/Day29/day29-main-fig03.png)

圖（三）ca01 Health API 回傳正常，中繼 CA 憑證有效期也超過未來 30 天

另外在 pg01～03 分別對 PostgreSQL 與 etcd Leaf Certificate 執行 `openssl x509 -checkend 2592000`。`2592000` 代表 30 天；檢查失敗時應先續期，不要等到服務重啟或新連線才發現憑證已過期。

### 3.3 輸出 Cloudflare Origin Certificate 期限

**操作節點：proxy01、proxy02。**

Day 27 的 Origin Certificate 必須在兩台 Proxy 分開監控。先確認 Debian 套件的 Textfile Collector 目錄：

```bash
systemctl cat prometheus-node-exporter |
  grep -- '--collector.textfile.directory'
sudo install -d -o root -g root -m 0755 \
  /var/lib/prometheus/node-exporter
sudo nano /usr/local/sbin/export-origin-cert-expiry
```

貼上：

```bash
#!/bin/sh
set -eu

cert=/etc/nginx/tls/iron-origin.crt
out_dir=/var/lib/prometheus/node-exporter
not_after=$(openssl x509 -enddate -noout -in "$cert" | cut -d= -f2-)
expiry=$(date -d "$not_after" +%s)
tmp=$(mktemp "$out_dir/iron-origin-cert.prom.XXXXXX")

printf '%s\n' \
  '# HELP iron_origin_cert_expiry_timestamp_seconds Origin certificate expiry time.' \
  '# TYPE iron_origin_cert_expiry_timestamp_seconds gauge' \
  "iron_origin_cert_expiry_timestamp_seconds $expiry" > "$tmp"
chmod 0644 "$tmp"
mv -f "$tmp" "$out_dir/iron-origin-cert.prom"
```

儲存後建立定期執行的 Service 與 Timer：

```bash
sudo chown root:root /usr/local/sbin/export-origin-cert-expiry
sudo chmod 0750 /usr/local/sbin/export-origin-cert-expiry
sudo nano /etc/systemd/system/iron-origin-cert-metric.service
```

```ini
[Unit]
Description=Export Cloudflare Origin certificate expiry metric

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/export-origin-cert-expiry
```

```bash
sudo nano /etc/systemd/system/iron-origin-cert-metric.timer
```

```ini
[Unit]
Description=Refresh Cloudflare Origin certificate expiry metric

[Timer]
OnBootSec=2min
OnUnitActiveSec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

兩台各自執行：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now iron-origin-cert-metric.timer
sudo systemctl start iron-origin-cert-metric.service
systemctl status iron-origin-cert-metric.timer --no-pager
curl -fsS http://127.0.0.1:9100/metrics |
  grep '^iron_origin_cert_expiry_timestamp_seconds'
```

兩台都必須有 Metric。只檢查 proxy01 會遺漏 proxy02 憑證未同步或將要過期的情況。

## 4. PostgreSQL Exporter

**操作順序：三台先安裝套件；Database Role 只在目前 Leader 建立一次；HBA 寫入 Patroni Dynamic Configuration；最後逐台設定並啟動 Exporter。**

pg01～03：

```bash
sudo apt install -y prometheus-postgres-exporter
```

### 4.1 找出目前 Leader

在 pg01、pg02 或 pg03 任一台執行：

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml \
  list iron-pg -e
```

查看 `Role` 欄位，記下顯示 `Leader` 的 Member。Leader 可能因先前的 Switchover／Failover 而改變，不要直接假設一定是 pg01。切換到該節點後再次確認：

```bash
hostnamectl --static
sudo -u postgres psql -X -d postgres -Atqc \
  'SELECT pg_is_in_recovery();'
```

必須回傳 `f`；`t` 代表目前位於 Replica，回到 `patronictl list` 找出正確 Leader。

![Patroni 叢集顯示目前 Leader 與兩個 Replica](../../source/Day29/day29-main-fig04.png)

圖（四）建立監控帳號前先確認目前由 pg02 擔任 Leader

### 4.2 只在 Leader 建立監控 Role

在剛確認的 Leader 開啟 psql：

```bash
sudo -u postgres psql -X -d postgres -P pager=off
```

在 psql 中執行：

```sql
CREATE ROLE monitor_user LOGIN;
GRANT pg_monitor TO monitor_user;
\password monitor_user
```

`\password` 會要求輸入並再次確認密碼，輸入內容不會顯示。實際密碼不得出現在文件、Git、指令歷史或公開畫面。完成後驗證 Role：

```sql
SELECT rolname,
       rolcanlogin,
       pg_has_role(rolname, 'pg_monitor', 'member') AS member_of_pg_monitor
FROM pg_roles
WHERE rolname = 'monitor_user';
\q
```

預期 `rolcanlogin` 與 `member_of_pg_monitor` 都是 `t`。Role 與密碼會透過 PostgreSQL Streaming Replication 傳到 Replica，不要在三台重複執行 `CREATE ROLE`。

![monitor_user 可以登入並具有 pg_monitor 成員資格](../../source/Day29/day29-main-fig05.png)

圖（五）`monitor_user` 已具備登入能力與 `pg_monitor` 成員資格

### 4.3 允許三台 Exporter 連到各自的 PostgreSQL

在任一台正常的 Patroni 節點執行：

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml \
  edit-config iron-pg
```

在 Dynamic Configuration 的 `postgresql` → `pg_hba` 清單中，將以下三條規則加在最終 `host all all 0.0.0.0/0 reject` 之前；保留原有規則：

```yaml
- hostssl postgres monitor_user 10.77.30.11/32 scram-sha-256
- hostssl postgres monitor_user 10.77.30.12/32 scram-sha-256
- hostssl postgres monitor_user 10.77.30.13/32 scram-sha-256
```

儲存並確認差異後輸入 `y`，再 Reload 三台成員：

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml \
  reload iron-pg --force
```

等待一個 Patroni Loop Cycle，再於 pg01、pg02、pg03 逐台確認 HBA 已載入且沒有語法錯誤：

```bash
sudo -u postgres psql -X -d postgres -P pager=off \
  -c "SELECT line_number,type,database,user_name,address,auth_method,error
      FROM pg_hba_file_rules
      WHERE 'monitor_user' = ANY(user_name)
      ORDER BY line_number;"
```

每台都應看到三條 `hostssl` 規則，且 `error` 為空。

![三條 monitor_user hostssl 規則已載入且沒有錯誤](../../source/Day29/day29-main-fig06.png)

圖（六）三條 `monitor_user` HBA 規則已載入，`error` 欄位皆為空

### 4.4 逐台設定並啟動 Exporter

每台 `/etc/default/prometheus-postgres-exporter` 設定連向自己的 PostgreSQL 節點 IP。密碼如含 URL 保留字，必須先做 URL Encoding；實際密碼不寫入公開文章：

```bash
sudo nano /etc/default/prometheus-postgres-exporter
```

```ini
# pg01 使用 10.77.30.11；pg02 使用 10.77.30.12；pg03 使用 10.77.30.13
DATA_SOURCE_NAME='postgresql://monitor_user:URL編碼後的實際密碼@10.77.30.11:5432/postgres?sslmode=require'
```

三台分別把 Host 改成自己的固定 IP。按 `Ctrl+O`、Enter、`Ctrl+X`，再執行：

```bash
sudo chown root:root /etc/default/prometheus-postgres-exporter
sudo chmod 600 /etc/default/prometheus-postgres-exporter
sudo systemctl enable prometheus-postgres-exporter
sudo systemctl restart prometheus-postgres-exporter
sudo systemctl status prometheus-postgres-exporter --no-pager
curl -fsS http://127.0.0.1:9187/metrics | sed -n '1,10p'
curl -fsS http://127.0.0.1:9187/metrics |
  grep '^pg_up '
```

預期服務為 `active (running)`，且 `pg_up 1`。若 Metrics Endpoint 可開啟但 `pg_up` 為 `0`，依序檢查該節點的 `DATA_SOURCE_NAME`、密碼、PostgreSQL 監聽位址與 `monitor_user` HBA。

![PostgreSQL Exporter 回報 pg_up 1](../../source/Day29/day29-main-fig07.png)

圖（七）PostgreSQL Exporter 成功連入本機資料庫並回報 `pg_up 1`

## 5. HAProxy Metrics

**操作節點：proxy01、proxy02。逐台修改並先做 Config Check。**

先在 proxy01 開啟 HAProxy Config：

```bash
sudo nano /etc/haproxy/haproxy.cfg
```

在檔案最後新增：

```haproxy
frontend prometheus
    bind 10.77.20.21:8405
    mode http
    log global
    option dontlog-normal
    http-request use-service prometheus-exporter if { path /metrics }
```

proxy02 改綁 `10.77.20.22:8405`。`log global` 讓 Frontend 使用 Global Log Target，`option dontlog-normal` 不記錄正常的 Metrics Scrape，只保留異常連線 Log，避免每次 Config Check 都顯示「沒有 Log Target，卻設定了 Log Format」的 Warning。

按 `Ctrl+O`、Enter、`Ctrl+X`，再執行：

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
curl -fsS http://10.77.20.21:8405/metrics | sed -n '1,10p'
```

在 proxy02 重複相同操作，但 `bind` 與 `curl` 都改用 `10.77.20.22:8405`。

![proxy01 的 HAProxy Metrics Endpoint](../../source/Day29/day29-main-fig08.png)

圖（八）proxy01 通過 HAProxy 設定檢查並可從 TCP 8405 讀取 Metrics

![proxy02 的 HAProxy Metrics Endpoint](../../source/Day29/day29-main-fig09.png)

圖（九）proxy02 可從 TCP 8405 讀取 HAProxy Metrics

### 5.1 放行 monitor01 必要的 Scrape 流量

操作位置：OPNsense Web UI。

1. 進入 `Firewall` → `Aliases`，按 `Add`。Name 填 `MONITOR_NODE`、Type 選 `Host(s)`、Content 填 monitor01 的 IP `10.77.20.51`，Description 填 `Prometheus monitoring node`，按 `Save`。
2. 再按 `Add`。Name 填 `MONITOR_PORTS`、Type 選 `Port(s)`。
3. Content 依次加入 `8008`、`8405`、`9100`、`9187`。
4. Description 填 `Patroni HAProxy Node and PostgreSQL metrics`，按 `Save` 與 `Apply`。
5. 進入 `Firewall` → `Rules`，在頁面左上角的 Interface 選單選 `LAB_INTERNAL`。這是 Day 14 建立的 Interface Group，包含 `SERVICE`、`DATABASE`、`BACKUP` 與 `BASTION`；不要在這裡選單一的 `SERVICE` Interface。
6. 按 `Add`，Action 選 `Pass`、Protocol 選 `TCP`。
7. Source 選 Alias `MONITOR_NODE`，Destination 選 Alias `INTERNAL_NETWORKS`，Destination Port 選 Alias `MONITOR_PORTS`。`INTERNAL_NETWORKS` 包含 `10.77.10.0/24`、`10.77.20.0/24`、`10.77.30.0/24`、`10.77.40.0/24` 與 `10.77.50.0/24`。
8. Description 填 `monitor01 scrape internal targets`，按 `Save`。
9. 將這條 Pass Rule 移到 `Block LAB_INTERNAL to internal networks` 上方，再按 `Apply`。

![MONITOR_PORTS Alias 包含四個監控連接埠](../../source/Day29/day29-main-fig10.png)

圖（十）`MONITOR_PORTS` 集中管理 Patroni、HAProxy、Node Exporter 與 PostgreSQL Exporter Port

![LAB_INTERNAL 的監控放行規則位於內部網路封鎖規則上方](../../source/Day29/day29-main-fig11.png)

圖（十一）monitor01 的監控流量規則位於跨區封鎖規則上方

### 5.2 限制同 VLAN 的監控連入流量

monitor01 與 proxy01、proxy02、app01、app02 都位於 VLAN 20。這些主機之間的封包由虛擬交換器直接轉送，不會經過 OPNsense，因此還要使用各主機的 nftables 限制 TCP 9100 與 8405 的來源。

#### 5.2.1 限制 proxy01 與 proxy02

**操作節點：proxy01、proxy02，兩台都要執行。**

Proxy 同時提供 Node Exporter TCP 9100 與 HAProxy Metrics TCP 8405。先安裝 nftables，並備份現有設定：

```bash
sudo apt install -y nftables
sudo cp -a /etc/nftables.conf /etc/nftables.conf.before-day29
sudo nano /etc/nftables.conf
```

保留檔案原有內容，將下列區塊加在 `/etc/nftables.conf` 最後。不要把它放進另一個 Table 或 Chain 內，也不要另外新增 `flush ruleset`：

```nftables
table inet day29_monitor {
  chain input {
    type filter hook input priority 10; policy accept;

    iifname "lo" tcp dport { 9100, 8405 } accept
    ip saddr 10.77.20.51 tcp dport { 9100, 8405 } accept
    tcp dport { 9100, 8405 } drop
  }
}
```

這個獨立 Table 只過濾 TCP 9100 與 8405：Loopback 與 monitor01 可以連入，其他來源則丟棄；其餘 Port 維持 `policy accept`，不會改變現有 SSH、Nginx、HAProxy 與 Keepalived 流量。

按 `Ctrl+O`、Enter、`Ctrl+X` 儲存，先檢查語法，確認通過後才套用：

```bash
sudo nft --check --file /etc/nftables.conf
sudo systemctl enable --now nftables
sudo systemctl reload nftables
systemctl is-active nftables
sudo nft list table inet day29_monitor
sudo ss -lntp | grep -E ':(8405|9100)\b'
```

`nft --check` 不得出現錯誤，服務應顯示 `active`，而 Ruleset 內應看到 monitor01 `10.77.20.51` 的 Allow 與其後的 Drop。

#### 5.2.2 限制 app01 與 app02

**操作節點：app01、app02，兩台都要執行。**

App 只提供 Node Exporter TCP 9100。先安裝 nftables，並備份現有設定：

```bash
sudo apt install -y nftables
sudo cp -a /etc/nftables.conf /etc/nftables.conf.before-day29
sudo nano /etc/nftables.conf
```

保留原有內容，在檔案最後加入：

```nftables
table inet day29_monitor {
  chain input {
    type filter hook input priority 10; policy accept;

    iifname "lo" tcp dport 9100 accept
    ip saddr 10.77.20.51 tcp dport 9100 accept
    tcp dport 9100 drop
  }
}
```

儲存後檢查並套用：

```bash
sudo nft --check --file /etc/nftables.conf
sudo systemctl enable --now nftables
sudo systemctl reload nftables
systemctl is-active nftables
sudo nft list table inet day29_monitor
sudo ss -lntp | grep ':9100'
```

#### 5.2.3 進行正向與負向測試

**正向測試位置：monitor01。**

```bash
for ip in 10.77.20.21 10.77.20.22 10.77.20.31 10.77.20.32; do
  echo "=== ${ip}:9100 ==="
  curl -fsS --connect-timeout 3 "http://${ip}:9100/metrics" |
    sed -n '1,2p'
done

for ip in 10.77.20.21 10.77.20.22; do
  echo "=== ${ip}:8405 ==="
  curl -fsS --connect-timeout 3 "http://${ip}:8405/metrics" |
    sed -n '1,2p'
done
```

每個 Target 都應輸出以 `# HELP` 或 `# TYPE` 開頭的 Metrics。

![monitor01 可以讀取 proxy01 與 app01 的 Metrics](../../source/Day29/day29-main-fig12.png)

圖（十二）monitor01 可成功讀取受監控節點的 Metrics Endpoint

**負向測試位置：client01。** client01 與這些主機同在 VLAN 20，可用來確認未經授權的同 VLAN 來源已被 Host Firewall 擋下：

```bash
curl -fsS --connect-timeout 3 http://10.77.20.21:9100/metrics
echo "proxy01 node exporter exit=$?"

curl -fsS --connect-timeout 3 http://10.77.20.21:8405/metrics
echo "proxy01 haproxy metrics exit=$?"

curl -fsS --connect-timeout 3 http://10.77.20.31:9100/metrics
echo "app01 node exporter exit=$?"
```

三個 `curl` 都應失敗並回傳非 `0` 的 Exit Code。這組負向測試只限於 Metrics Port；Web Backend、Web VIP 與其他原有服務應繼續正常。

![client01 無法連入受保護的 Metrics Port](../../source/Day29/day29-main-fig13.png)

圖（十三）client01 對 Metrics Port 的連線由主機防火牆拒絕

### 5.3 限制 monitor01 的管理介面與本機 Metrics

**操作節點：monitor01。**

monitor01 與 client01、proxy01、proxy02、app01、app02 同在 VLAN 20，同 VLAN 流量不會經過 OPNsense。除了前面限制各受監控節點的 Metrics Port，monitor01 本身也要限制 Grafana、Prometheus、Alertmanager、Node Exporter 與 Blackbox Exporter 的連入來源。

先安裝 nftables 並備份現有設定：

```bash
sudo apt install -y nftables
sudo cp -a /etc/nftables.conf /etc/nftables.conf.before-day29
sudo nano /etc/nftables.conf
```

保留檔案原有內容，在最後加入：

```nftables
table inet day29_monitor {
  chain input {
    type filter hook input priority 10; policy accept;

    iifname "lo" tcp dport { 3000, 9090, 9093, 9100, 9115 } accept
    ip saddr 10.77.60.0/24 tcp dport { 3000, 9090 } accept
    tcp dport { 3000, 9090, 9093, 9100, 9115 } drop
  }
}
```

Loopback 規則保留 Grafana、Prometheus、Alertmanager、Node Exporter 與 Blackbox Exporter 在 monitor01 內部互相存取；`10.77.60.0/24` 只開放 Grafana TCP 3000 與 Prometheus TCP 9090。實際可通過的 VPN 使用者會在第 6.1～6.3 節由 OPNsense 的 `VPN_MONITOR_CLIENTS` 再縮小到 `vpn-monitor01`。

檢查語法並套用：

```bash
sudo nft --check --file /etc/nftables.conf
sudo systemctl enable --now nftables
sudo systemctl reload nftables
systemctl is-active nftables
sudo nft list table inet day29_monitor
```

套用後，在 monitor01 確認五個本機 Endpoint 可正常存取：

```bash
curl -fsS http://127.0.0.1:3000/api/health
curl -fsS http://127.0.0.1:9090/-/ready
curl -fsS http://127.0.0.1:9093/-/ready
curl -fsS http://127.0.0.1:9100/metrics | sed -n '1,2p'
curl -fsS http://127.0.0.1:9115/-/healthy
```

再從同 VLAN 的 client01 執行負向測試：

```bash
for port in 3000 9090 9100; do
  curl -fsS --connect-timeout 3 "http://10.77.20.51:${port}/"
  echo "monitor01 port ${port} exit=$?"
done
```

三個連線都應失敗並回傳非 `0` 的 Exit Code。後續使用 `vpn-monitor01` 完成 OPNsense 授權後，TCP 3000 與 9090 才會從 VPN 管理路徑開放。

## 6. Prometheus Scrape Config

**操作節點：只在 monitor01。每新增一組 Target 就先 Reload 並到 Targets 頁確認。**

先備份原始檔案，再開啟 Editor：

```bash
sudo cp -a /etc/prometheus/prometheus.yml /etc/prometheus/prometheus.yml.before-day29
sudo nano /etc/prometheus/prometheus.yml
```

保留既有 `global` 區塊。找到 Debian 預設的 `scrape_configs:`，將它和下方所有內容整段刪除，再以下列 `rule_files`、`alerting` 與 `scrape_configs` 完整取代。不要把新區塊追加在原本的 `job_name: prometheus` 後面；整份檔案只能有一個 `scrape_configs:`，而每個 `job_name` 也只能出現一次：

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["127.0.0.1:9093"]

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node
    static_configs:
      - targets:
          - localhost:9100
          - 10.77.20.21:9100
          - 10.77.20.22:9100
          - 10.77.20.31:9100
          - 10.77.20.32:9100
          - 10.77.30.10:9100
          - 10.77.30.11:9100
          - 10.77.30.12:9100
          - 10.77.30.13:9100
          - 10.77.40.11:9100
          - 10.77.50.11:9100

  - job_name: patroni
    metrics_path: /metrics
    static_configs:
      - targets: ["10.77.30.11:8008", "10.77.30.12:8008", "10.77.30.13:8008"]

  - job_name: postgres
    static_configs:
      - targets: ["10.77.30.11:9187", "10.77.30.12:9187", "10.77.30.13:9187"]

  - job_name: haproxy
    static_configs:
      - targets: ["10.77.20.21:8405", "10.77.20.22:8405"]

  - job_name: public-web-https
    metrics_path: /probe
    params: { module: [http_2xx] }
    static_configs:
      - targets: ["https://app.example.com/health"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

把 `app.example.com` 改成 Day 27 的真實公開 Hostname。這個 Probe 檢查的是經公開 DNS、Cloudflare Edge 到 Origin 的完整路徑。在重啟 Prometheus 前，先從 monitor01 確認：

```bash
dig app.example.com A +short
curl -fsS --connect-timeout 10 https://app.example.com/health
```

按 `Ctrl+O`、Enter、`Ctrl+X`。先確認 `scrape_configs` 與 `job_name: prometheus` 各只出現一次，再檢查語法；成功才 Restart：

```bash
grep -n '^scrape_configs:' /etc/prometheus/prometheus.yml
grep -nE '^[[:space:]]*- job_name:[[:space:]]*prometheus[[:space:]]*$' \
  /etc/prometheus/prometheus.yml
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
sudo systemctl status prometheus --no-pager
curl http://127.0.0.1:9090/-/ready
```

### 6.1 建立 VPN-Monitor 群組與 Alias

**操作位置：OPNsense Web UI。**

Day 17 已建立 `IRON-LAB-ROAD-WARRIOR` OpenVPN Instance、`vpn-monitor01` 帳號與 Client Profile。本節不重新建立 VPN，而是授權 `vpn-monitor01` 連入 monitor01 的 Grafana 與 Prometheus Web UI。Alertmanager 由 monitor01 本機透過 `amtool` 管理，不對 VPN 開放 TCP 9093。

1. 進入 `System` → `Access` → `Groups`，按 `Add`。
2. Group Name 填 `vpn-monitor`，Description 填 `Monitoring UI operators`。
3. Members 只選 `vpn-monitor01`；不要加入 `vpn-ro01`、`vpn-rw01` 或 `vpn-admin01`，按 `Save`。
4. 進入 `Firewall` → `Aliases`，按 `Add`。
5. Name 填 `VPN_MONITOR_CLIENTS`、Type 選 `OpenVPN group`、Content 選剛建立的 `vpn-monitor`。
6. Description 填 `Connected OpenVPN members of vpn-monitor`，按 `Save`。
7. 再按 `Add`，Name 填 `MONITOR_UI_PORTS`、Type 選 `Port(s)`，Content 依次加入 `3000`、`9090`。
8. Description 填 `Grafana and Prometheus UI`，按 `Save` 與 `Apply`。

`VPN_MONITOR_CLIENTS` 是動態 Alias。`vpn-monitor01` 尚未連線時可以是空的；連線後，OPNsense 會將該 Client 取得的 `10.77.60.x` Tunnel IP 加入 Alias。

![vpn-monitor 群組只包含 vpn-monitor01](../../source/Day29/day29-main-fig14.png)

圖（十四）`vpn-monitor` 群組只納入監控用途的 `vpn-monitor01`

![VPN_MONITOR_CLIENTS 使用 OpenVPN group 動態 Alias](../../source/Day29/day29-main-fig15.png)

圖（十五）`VPN_MONITOR_CLIENTS` 會依 `vpn-monitor` 的已連線成員動態更新

### 6.2 建立 OpenVPN 到監控介面的規則

1. 進入 `Firewall` → `Rules`，在頁面左上角的 Interface 選單選 `OpenVPN`；不要選 `WAN`、`SERVICE` 或 `LAB_INTERNAL`。
2. 按 `Add`，Action 選 `Pass`、Direction 選 `in`、Version 選 `IPv4`、Protocol 選 `TCP`。
3. Source 選 Alias `VPN_MONITOR_CLIENTS`。
4. Destination 選 Alias `MONITOR_NODE`。
5. Destination Port 選 Alias `MONITOR_UI_PORTS`。
6. 勾選 Log，Description 填 `ALLOW_VPN_MONITOR_TO_MONITORING_UI`，按 `Save` 與 `Apply`。
7. 確認這條 Allow 位於所有可能拒絕 OpenVPN 內部流量的 Block Rule 上方。未命中 Allow 的其他 VPN 使用者會由 OPNsense 的隱含 Default Deny 拒絕。

![OpenVPN 群組只允許 VPN_MONITOR_CLIENTS 存取監控介面](../../source/Day29/day29-main-fig16.png)

圖（十六）OpenVPN 規則只允許 `VPN_MONITOR_CLIENTS` 存取 monitor01 的管理介面

### 6.3 使用 vpn-monitor01 連線並測試

在管理電腦匯入或選擇 Day 17 匯出的 `vpn-monitor01` Client Profile，連上 `IRON-LAB-ROAD-WARRIOR`。不要改用 `vpn-admin01`、`vpn-rw01` 或 `vpn-ro01`。

連線後在 PowerShell 確認兩個監控 Port 都已通過防火牆：

```powershell
Test-NetConnection 10.77.20.51 -Port 3000
Test-NetConnection 10.77.20.51 -Port 9090
```

`TcpTestSucceeded` 都必須是 `True`。若為 `False`，先確認目前使用的 Profile 是 `vpn-monitor01`，再檢查 `VPN_MONITOR_CLIENTS` 是否已展開為該 Client 的 Tunnel IP、`ALLOW_VPN_MONITOR_TO_MONITORING_UI` 規則順序，以及 Grafana 與 Prometheus 的監聽狀態。

保持第 6.3 節建立的 `vpn-monitor01` OpenVPN Connection，在管理電腦開啟 `http://10.77.20.51:9090`。如果畫面顯示 Debian 未內建現代 React Web UI 的提示，這是 Debian Prometheus 套件的預期畫面。按 `Use classic web UI`，再進入 `Status` → `Targets`；也可直接開啟：

```text
http://10.77.20.51:9090/classic/targets
```

本 Lab 使用 Grafana 作為主要圖形介面，Prometheus Web UI 只用來檢查 Target、Rule 與查詢結果，因此不需要執行提示頁上的 `/usr/share/prometheus/install-ui.sh`。先確認 `node` Targets，再確認 Patroni、PostgreSQL、HAProxy 與 Web Probe；不要在 Target Down 時直接繼續建 Dashboard。

正常時應看到 `prometheus (1/1 up)`、`node (11/11 up)`、`patroni (3/3 up)`、`postgres (3/3 up)`、`haproxy (2/2 up)` 與 `public-web-https (1/1 up)`。若十個遠端 TCP 9100 Target 出現在 `prometheus` Job 下，代表 `/etc/prometheus/prometheus.yml` 的縮排或 `job_name` 放錯：`prometheus` 只保留 `localhost:9090`，本機與遠端的 TCP 9100 Target 全部放在 `node` Job。修正後再執行：

```bash
sudo promtool check config /etc/prometheus/prometheus.yml
sudo systemctl restart prometheus
```

![Prometheus Targets 顯示各監控工作與 Target 狀態](../../source/Day29/day29-main-fig17.png)

圖（十七）Prometheus Targets 顯示各工作預期的 Target 數量與狀態

## 7. Alert Rules

**操作節點：只在 monitor01。**

建立目錄並開啟 Rule File：

```bash
sudo install -d -m 755 /etc/prometheus/rules
sudo nano /etc/prometheus/rules/iron-lab.yml
```

貼上：

```yaml
groups:
  - name: iron-lab
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "{{ $labels.job }} {{ $labels.instance }} is down"
          runbook: "Day29-InstanceDown"

      - alert: FilesystemLow
        expr: node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} < 0.15
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Filesystem below 15% on {{ $labels.instance }}"
          runbook: "Day29-FilesystemLow"

      - alert: PatroniHasNoLeader
        expr: sum(patroni_primary) != 1
        for: 30s
        labels: { severity: critical }
        annotations:
          summary: "Patroni cluster does not have exactly one leader"
          runbook: "Day29-PatroniLeader"

      - alert: PublicHttpsDown
        expr: probe_success{job="public-web-https"} == 0
        for: 2m
        labels: { severity: critical }
        annotations:
          summary: "Public Cloudflare HTTPS probe is failing"
          runbook: "Day29-PublicHttps"

      - alert: PublicEdgeCertificateExpiring
        expr: probe_ssl_earliest_cert_expiry{job="public-web-https"} - time() < 1209600
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Public edge certificate expires within 14 days"
          runbook: "Day29-PublicEdgeCertificate"

      - alert: OriginCertificateExpiring
        expr: iron_origin_cert_expiry_timestamp_seconds - time() < 2592000
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "Origin certificate expires within 30 days on {{ $labels.instance }}"
          runbook: "Day29-OriginCertificate"
```

先到 Prometheus `/metrics` 查證實際 Patroni Metric Name；若目前版本名稱不同，修改 Expression 後用 `promtool check rules`，不要照抄一條永遠沒有資料的規則。

按 `Ctrl+O`、Enter、`Ctrl+X`，再執行：

```bash
sudo promtool check rules /etc/prometheus/rules/iron-lab.yml
sudo systemctl reload prometheus
```

![六條 Prometheus 告警規則已載入並處於 Inactive](../../source/Day29/day29-main-fig18.png)

圖（十八）六條告警規則已成功載入，正常狀態下全部為 Inactive

## 8. Alertmanager 本機路由

**操作節點：只在 monitor01。本 Lab 使用不掛載外部通知整合的本機 Receiver。**

本節只建立 Alertmanager 的本機 Routing Tree，讓 `warning` 與 `critical` 依 `severity` Label 分類到不同 Receiver。Receiver 不掛載通知整合，因此 Alert 只透過 monitor01 上的 `amtool` 或 HTTP API 查看，不會寄送 Email 或呼叫 Webhook。Debian 的 Alertmanager 套件未內建 Web UI，瀏覽器開啟 TCP 9093 時看到提示頁屬於預期行為。

先備份再開啟設定檔：

```bash
sudo cp -a /etc/prometheus/alertmanager.yml /etc/prometheus/alertmanager.yml.before-day29
sudo nano /etc/prometheus/alertmanager.yml
```

將內容完整取代為：

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: warning-local
  group_by: [alertname, job, instance]
  group_wait: 1m
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - receiver: critical-local
      matchers:
        - 'severity="critical"'
      group_wait: 10s
      repeat_interval: 30m

    - receiver: warning-local
      matchers:
        - 'severity="warning"'
      group_wait: 1m
      repeat_interval: 4h

receivers:
  - name: warning-local
  - name: critical-local
```

頂層 Route 不設 Matcher，讓所有 Alert 先進入同一棵 Routing Tree。`critical` 會匹配 `critical-local`，`warning` 會匹配 `warning-local`；沒有 `severity` Label 的 Alert 使用頂層 `warning-local`，避免未分類 Alert 被遺漏。

按 `Ctrl+O`、Enter、`Ctrl+X`，再驗證設定與路由結果：

```bash
sudo chown root:prometheus /etc/prometheus/alertmanager.yml
sudo chmod 640 /etc/prometheus/alertmanager.yml
sudo amtool check-config /etc/prometheus/alertmanager.yml
sudo amtool config routes test \
  --config.file=/etc/prometheus/alertmanager.yml \
  --verify.receivers=warning-local \
  severity=warning
sudo amtool config routes test \
  --config.file=/etc/prometheus/alertmanager.yml \
  --verify.receivers=critical-local \
  severity=critical
```

預期兩次 Route Test 都顯示 `SUCCESS`。重新啟動 Alertmanager，並確認 Readiness Endpoint：

```bash
sudo systemctl restart prometheus-alertmanager
sudo systemctl status prometheus-alertmanager --no-pager
curl -fsS http://127.0.0.1:9093/-/ready
```

服務應為 `active (running)`，Readiness Endpoint 應回覆 `OK`。本日會在第 10 節停止真實 Target，觀察 Alert 觸發與恢復後的清除流程，不另外建立 Webhook 測試 Alert。

![Alertmanager Readiness Endpoint 回覆 OK](../../source/Day29/day29-main-fig19.png)

圖（十九）Alertmanager Readiness Endpoint 回覆 `OK`

要在 monitor01 列出 Alertmanager 目前收到的 Active Alert，執行：

```bash
sudo amtool alert query \
  --alertmanager.url=http://127.0.0.1:9093
```

> **正式環境延伸：**可在 Receiver 下加入 `email_configs` 或 `webhook_configs`，並將 `warning` 送往維運通知、`critical` 送往即時通知管道。外部 Receiver 上線前要實際驗證 Firing 與 Resolved 通知，敏感 Token 不得進入 Git。本次 Lab 使用本機 Receiver 觀察告警路由與 Active Alert 狀態。

## 9. Grafana

**操作位置：從管理電腦開啟 monitor01 的 Grafana Web UI。**

管理電腦已在第 6.3 節使用 `vpn-monitor01` 連上 `IRON-LAB-ROAD-WARRIOR`，並已通過 TCP 3000 與 9090 的連線測試。

### 9.1 首次登入、修改密碼並驗證

1. 以瀏覽器開啟 `http://10.77.20.51:3000`。
2. 在登入頁的 `Email or username` 輸入 `admin`，`Password` 輸入初始密碼 `admin`，按 `Log in`。
3. 首次登入會顯示修改密碼畫面。在 `New password` 與 `Confirm new password` 輸入相同的新密碼，按 `Submit`；新密碼不得出現在文件、Git、指令歷史或公開畫面。
4. 進入 Grafana 首頁後，打開使用者選單並按 `Sign out`。
5. 再次開啟 `http://10.77.20.51:3000`，使用帳號 `admin` 與剛設定的新密碼登入。
6. 確認能再次進入 Grafana 首頁。舊密碼 `admin` 已失效且新密碼可登入，才算完成首次帳密更換驗證。

### 9.2 加入 Prometheus Data Source 與 Dashboard

1. 左側選 `Connections` → `Data sources`。
2. 按 `Add new data source`，選 `Prometheus`。
3. Name 填 `Prometheus`，Prometheus server URL 填 `http://127.0.0.1:9090`。
4. 按頁面底部 `Save & test`，確認顯示連線成功。
5. 從本 Repository 取得 [`Day29_IRON-LAB-Overview.dashboard.json`](./DAY29_IRON-LAB-Overview.dashboard.json)，不要修改檔案副檔名。
6. 回到 Grafana，左側選 `Dashboards`，按 `New` → `Import`。
7. 按 `Upload dashboard JSON file`，選擇 `Day29_IRON-LAB-Overview.dashboard.json`。
8. 在 Import 畫面的 Prometheus Data Source 選單選擇剛建立的 `Prometheus`。
9. Name 保持 `IRON-LAB Overview`、Folder 保持 `General`，按 `Import`。
10. 確認 Dashboard 出現 Node CPU、Memory、Filesystem、Patroni Role、PostgreSQL Connections、HAProxy Backend 與 Public HTTPS Probe 七個 Panel。
11. 將右上角時間範圍保持為 `Last 6 hours`，等待至少一次 15 秒 Refresh；各 Panel 應顯示資料，Public HTTPS Probe 應顯示 `UP`。

若某個 Panel 顯示 `No data`，先在 Prometheus 的 Graph 頁面查詢該 Panel 使用的 Metric，並確認對應 Target 是 `UP`；不要在匯入後直接改寫 Query。Dashboard JSON 使用 Import Input 綁定 Prometheus，不包含 Data Source Password、Grafana 密碼或外部通知 Credential。

![Grafana IRON-LAB Overview 顯示七個監控面板](../../source/Day29/day29-main-fig20.png)

圖（二十）Grafana 儀表板整合主機、資料庫、代理與公開 HTTPS 指標

## 10. 實際觸發

**操作順序：先保持監控畫面開啟，再到單一目標節點注入故障，確認告警恢復後才進下一項。**

1. 在管理電腦開啟 Prometheus `Alerts` 與 Grafana Dashboard；另外在 monitor01 開啟一個 Terminal，執行 `watch -n 2 "amtool alert query --alertmanager.url=http://127.0.0.1:9093"`。
2. 到 app02 執行 `date -Is` 與 `sudo systemctl stop prometheus-node-exporter`。
3. 等 `InstanceDown` 從 Pending 變成 Firing，記錄時間。
4. 確認 monitor01 的 `amtool` 輸出出現該 Alert，並檢查 Severity、Summary 與 Runbook Annotation。
5. 回 app02 執行 `sudo systemctl start prometheus-node-exporter`。
6. 確認 Prometheus Target 回到 Up、Prometheus 不再顯示 Firing，而且該 Alert 隨後從 `amtool alert query` 的 Active Alert 清單消失。

![app02 停止 Node Exporter 以注入故障](../../source/Day29/day29-main-fig21.png)

圖（二十一）在 app02 停止 Node Exporter，建立可控制的 Target 故障

![app02 的 InstanceDown 進入 Pending](../../source/Day29/day29-main-fig22.png)

圖（二十二）`InstanceDown` 先進入 Pending，等待 `for: 2m` 持續時間

![Alertmanager 收到 app02 的 Active Alert](../../source/Day29/day29-main-fig23.png)

圖（二十三）條件持續成立後，Alertmanager 收到 app02 的 Active Alert

![app02 恢復後 Active Alert 已清除](../../source/Day29/day29-main-fig24.png)

圖（二十四）app02 恢復後，該告警從 Active Alert 清單消失

### 10.1 停止一個 Replica 的 Patroni

**操作節點：pg01，以及本次選定的一個 Replica。不可停止 Leader。**

先確認 app02 的 Node Exporter 已恢復，前一項 `InstanceDown` 也已從 Active Alert 清單消失。接著在 pg01 列出 Patroni Cluster：

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml \
  list -e
```

開始故障測試前必須符合：

- 剛好一個 Member 是 `Leader`，State 是 `running`。
- 另外兩個 Member 是 `Replica`，State 是 `streaming`。
- 三個 Member 的 Timeline 相同，Replication Lag 沒有異常累積。

從兩個 Replica 中選一個測試。下方以 `pg03` 為例；只有剛才的清單明確顯示 `pg03` 是 `Replica` 時，才可以照抄。

在 monitor01 另開一個 Terminal，持續查看 Alertmanager 收到的 Active Alert：

```bash
watch -n 2 \
  'amtool alert query --alertmanager.url=http://127.0.0.1:9093'
```

再登入 pg03 執行下列命令；不要留在 pg01 執行。本 Lab 的 Patroni REST API 只監聽各節點的 `10.77.30.x:8008`，沒有監聽 `127.0.0.1:8008`。`/replica` 只會在該節點當下是 Replica 時回傳 HTTP 200；若它已變成 Leader，`curl -f` 會失敗，後方的停止命令就不會執行：

```bash
date -Is
curl -fsS http://10.77.30.13:8008/replica >/dev/null && \
  sudo systemctl stop patroni
systemctl is-active patroni
```

預期 `systemctl is-active` 顯示 `inactive`。回到管理電腦的 Prometheus：

1. 進入 `Status` → `Targets`，確認 `10.77.30.13:8008` 變成 `DOWN`。
2. 進入 `Alerts`，確認 `InstanceDown` 先進入 `Pending`。
3. 等待 `for: 2m` 持續時間通過，確認 `InstanceDown` 變成 `Firing`，其 Label 為 `instance="10.77.30.13:8008"`、`job="patroni"`。
4. 確認 monitor01 的 `amtool` 出現這筆 `InstanceDown`。
5. 確認 `PatroniHasNoLeader` 依然是 `Inactive`，不會出現在 `amtool` 的 Active Alert 清單。

這兩條 Alert 代表不同問題：`InstanceDown` 表示 Prometheus 無法抓取 pg03 的 Patroni Metrics；`PatroniHasNoLeader` 只在 `sum(patroni_primary) != 1` 持續 30 秒時成立。本次只停止 Replica，Leader 持續回報 `patroni_primary 1`，因此後者不應觸發。

![pg03 停止 Patroni 前先確認其角色為 Replica](../../source/Day29/day29-main-fig25.png)

圖（二十五）確認 pg03 是 Replica 後停止 Patroni，避免中斷 Primary

![pg03 Patroni Target 變成 Down](../../source/Day29/day29-main-fig26.png)

圖（二十六）pg03 停止 Patroni 後，Patroni 工作顯示兩個 Target 正常、一個 Down

![pg03 的 InstanceDown 進入 Firing](../../source/Day29/day29-main-fig27.png)

圖（二十七）pg03 的 Patroni Target 持續失聯後，`InstanceDown` 進入 Firing

![PatroniHasNoLeader 維持 Inactive](../../source/Day29/day29-main-fig28.png)

圖（二十八）單一 Replica 故障期間，Primary 正常讓 `PatroniHasNoLeader` 維持 Inactive

### 10.2 恢復 Replica 並確認重新加入 Cluster

在 pg03 恢復 Patroni：

```bash
sudo systemctl start patroni
systemctl is-active patroni
sudo systemctl status patroni --no-pager
```

預期服務回到 `active`。回到 pg01 重複查看 Cluster：

```bash
sudo -u postgres patronictl \
  -c /etc/patroni/config.yml \
  list -e
```

等到 pg03 顯示 `Replica` 與 `streaming`，Timeline 與 Leader 相同，Receive LSN、Replay LSN 繼續前進。接著確認：

1. Prometheus 的 `10.77.30.13:8008` Target 回到 `UP`。
2. `InstanceDown` 不再是 `Firing`。
3. 稍後該 Alert 從 `amtool alert query` 的 Active Alert 清單消失。
4. `PatroniHasNoLeader` 在整個 Replica 故障期間都沒有觸發。

三個 Member 全部回到正常狀態後，按 `Ctrl+C` 結束 monitor01 的 `watch`。

![恢復後 Prometheus 告警規則全部回到 Inactive](../../source/Day29/day29-main-fig29.png)

圖（二十九）恢復 Patroni 後，Prometheus 告警規則全部回到 Inactive

![恢復後所有 Prometheus Targets 回到 Up](../../source/Day29/day29-main-fig30.png)

圖（三十）恢復後，各 Prometheus Target 回到 Up

![pg03 恢復為 Replica 並重新加入叢集](../../source/Day29/day29-main-fig31.png)

圖（三十一）pg03 恢復為 Replica 並重新加入 Patroni 叢集
