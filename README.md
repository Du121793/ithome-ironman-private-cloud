# 從裸機到叢集：以 Proxmox VE、Ceph、OPNsense 與 PostgreSQL HA 建立可驗證的安全私有雲

這是本次 iThome 鐵人賽系列的實作文件庫，收錄 30 天系列配套的操作文件、統一設定，以及提供 iThome 正文引用的圖片與 GIF。文章正文會發布於 iThome，不收錄在本倉庫。

系列正文：[從裸機到叢集：以 Proxmox VE、Ceph、OPNsense 與 PostgreSQL HA 建立可驗證的安全私有雲](https://ithelp.ithome.com.tw/users/20183351/ironman/9461)

系列將從裸機與 Proxmox VE 虛擬化平台開始，逐步建立 PVE 叢集與 Ceph 共享儲存，使用 OPNsense 劃分網路及安全邊界，最後完成可切換、可監控、可備份並能實際驗證的 PostgreSQL HA 環境。

實作文件同時提供 Markdown 與 HTML 版本，內容相同，可依自己的閱讀或使用習慣選擇。

## 資料夾與文件

- `md/`：各日配套實作文件的 Markdown 版本。
- `html/`：各日配套實作文件的 HTML 版本。
- `source/Day01/`～`source/Day30/`：實作文件引用的圖片與 GIF。
- `assets/day01/`～`assets/day30/`：iThome 正文引用的圖片與 GIF。
- `00_設定文件索引.md`：統一設定文件入口。
- `01_硬體與資源.md`～`06_版本與套件來源.md`：全系列共用的硬體、VM、網路、介面、流量及版本基準。
