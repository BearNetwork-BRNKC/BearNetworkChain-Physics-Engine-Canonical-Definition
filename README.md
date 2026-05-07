> [!CAUTION]
> ### ⚠️ 重要安全警示：遠離 Bastion X 詐騙平台 (SECURITY ALERT)
> **Bear Network Chain (BNC) 官方在此聲明：**
> 惡意平台 **"Bastion X"** 正在非法冒用「BNES Γ 物理引擎」技術白皮書與創辦人陳霆之名義進行金融詐騙。
> 
> *   **事實查核**：Bastion X 已被台灣 165 反詐騙單位列為假投資陷阱。
> *   **技術聲明**：BNC 團隊與該平台無任何技術授權或合作關係。其展示之技術文件均為劫持與偽造。
> *   **官方渠道**：請僅認準本 GitHub 官方倉庫。
> 
> **Warning**: "Bastion X" is a fraudulent platform misappropriating BNES Γ intellectual property. It has been blacklisted by law enforcement (Taiwan 165 Anti-Fraud). Please protect your assets.


# BearNetworkChain BNES 節點架設指導說明書 (v1.1.0)

本文件由 BearNetworkChain 官方生成，旨在提供開發者與運維工程師一份專業、清晰且完整的 BNES 節點部署與營運指南。

---

## 1. 標題與版本資訊

*   **文件標題**：BearNetworkChain BNES 節點架設指導說明書
*   **軟體版本**：v1.1.0 Production Release
*   **更新日期**：2026-04-27
*   **適用對象**：後端開發人員、DevOps 工程師、區塊鏈節點營運者（Validator/Signer）

---

## 2. 系統需求

為了確保 BNES 引擎（Γ 物理引擎）的穩定運行，建議採用以下配置：

| 項目 | 最低配置 | 推薦配置 (生產環境) | Authority 節點 |
| :--- | :--- | :--- | :--- |
| **作業系統** | Ubuntu 20.04+ / CentOS 8+ | Ubuntu 22.04 LTS | 同推薦配置 |
| **CPU** | 4 核心 | 8 核心+ (高效能單核尤佳) | 16 核心+ |
| **記憶體** | 8 GB RAM | 16 GB - 32 GB RAM | 64 GB RAM |
| **磁碟空間** | 100 GB SSD | 500 GB NVMe SSD | 2 TB+ NVMe SSD |
| **網路頻寬** | 20 Mbps 穩定連線 | 100 Mbps+ 對等頻寬 | 1 Gbps+ |
| **Go 版本** | v1.21+ | v1.22.x | 同推薦配置 |

> [!IMPORTANT]
> **Authority (驗證器) 節點特殊需求**：必須具備極高可用性與強大的抗 DDoS 能力。強烈建議在 RPC 前端部署本指南提供的 Nginx Proxy。

---

## 3. 快速上手（2 分鐘部署）

### 方案 A：裸機部署（一鍵自動化）
```bash
# 1. 複製專案
git clone https://github.com/BearNetworkChain/BearNetworkChain-Node.git && cd BearNetworkChain-Node

# 2. 初始化配置 (複製 .env 並編輯)
cp deploy/configs/env.example deploy/configs/.env

# 3. 執行安裝並啟動
chmod +x deploy/scripts/*.sh
./deploy/scripts/install.sh && ./deploy/scripts/start.sh
```

### 方案 B：Docker 部署（容器化）
```bash
# 1. 啟動完整服務棧
docker-compose up -d

# 2. 檢查狀態
docker-compose ps
```

✅ **完成後預期結果**：執行 `curl http://localhost:8545` 或 `./deploy/scripts/status.sh` 應顯示節點已連線並開始同步。

---

## 4. 部署方式比較

| 維度 | 裸機部署 (Bare Metal) | Docker 容器化部署 |
| :--- | :--- | :--- |
| **效能** | **極佳**（直接存取硬體資源，無虛擬化損耗） | **優秀**（極微小的 I/O 損耗，適合多數場景） |
| **管理難度** | 中等（需管理環境相依、程序的存活） | **簡易**（一鍵啟動所有相關服務） |
| **隔離性** | 較低 | **高**（環境完全隔離，不污染宿主機） |
| **適用場景** | 生產環境、高性能 Authority 節點 | 開發測試、快速擴展、Staging 環境 |

---

## 5. 方式一：裸機部署（推薦生產環境）

### 前置準備
確保系統已安裝 `git`, `make`, `gcc`, `jq`, `curl` 以及 `Go 1.22+`。

### 部署流程
1.  **安裝腳本**：執行 `./deploy/scripts/install.sh`。它會驗證系統環境、編譯 `bnes` 二進位檔並初始化創世區塊。
2.  **配置調整**：根據實際需求修改 `.env` 檔案。
3.  **啟動節點**：執行 `./deploy/scripts/start.sh`。腳本會檢查 PID 鎖並載入環境變數。

### 腳本工具集（`deploy/scripts/`）
| 腳本 | 主要用途 | 常用參數 |
| :--- | :--- | :--- |
| `start.sh` | 啟動節點程序 | 無 |
| `stop.sh` | 安全關閉節點 | 無（會自動讀取 `BNES_STOP_GRACE_PERIOD`） |
| `status.sh` | 檢查健康狀況 | 無 |
| `restart.sh` | 重啟服務 | `--no-cooldown` (跳過冷卻) |
| `upgrade.sh` | 執行無痛升級 | `--force` (強制覆蓋), `--no-pull` (本地升級) |
| `rollback.sh` | 版本回滾 | `--force` (忽略警告) |

---

## 6. 方式二：Docker / Docker Compose 部署

針對 v1.1.0 生態系，我們提供了預配置的容器化方案。

### Dockerfile (最佳化多階段建置)
專屬 `Dockerfile` 採用 Go + Alpine 組合，確保映像檔最小化（約 150MB）且具備生產級安全性。

### Docker Compose 架構
預設啟動以下服務鏈：
- `bnes-node`: 核心區塊鏈節點
- `nginx-proxy`: RPC 安全防護層
- `prometheus`: 指標蒐集器
- `grafana`: 視覺化面板

### 常用操作
- **啟動所有服務**：`docker-compose up -d`
- **查看日誌**：`docker-compose logs -f bnes-node`
- **停止服務**：`docker-compose down`

---

## 7. 配置詳細說明

所有配置皆集中於 `deploy/configs/` 目錄：

*   **`env.example` ↔ `.env` 對應表**：
    - `BNES_CHAIN_ID`: **[必填]** 641230（主網固定值）。
    - `BNES_BOOTNODES`: **[必填]** 官方提供的引導節點列表。
    - `BNES_DATA_DIR`: **[選填]** 數據儲存路徑，預設 `./node-data`。
    - `BNES_CACHE`: **[進階]** 記憶體快取 (MB)，建議設為總 RAM 的 50%。
    - `BNES_GCMODE`: **[進階]** `archive` (保留全部) 或 `full` (清理舊狀態)。

*   **RPC Proxy 安全設定 (`nginx.conf`)**：
    - **限流**：每秒 100 請求 (`BNES_PROXY_RATE_LIMIT`)。
    - **Body 限制**：防止大請求攻擊 (`BNES_PROXY_MAX_BODY_SIZE=512k`)。
    - **健康檢查**：提供 `/health` 與 `/ready` 探針。

---

## 8. 節點角色與啟動參數

| 節點角色 | 說明 | 關鍵啟動參數 |
| :--- | :--- | :--- |
| **Full Node** | 完整驗證所有交易與 Γ 物理量，適合 DApp 入口。 | `--http --syncmode full --gcmode full` |
| **Archive Node**| 適合區塊瀏覽器，保留所有歷史狀態。 | `--gcmode archive` |
| **Light Node** | 快速同步，僅同步 Header，適合輕量存取。 | `--syncmode snap` |
| **Authority** | 驗證器/出塊者，負責維護網絡安全。 | `--mine --miner.etherbase 0x... --password ...` |

---

## 9. 監控與觀測（Grafana & Prometheus）

### 監控系統匯入步驟
1.  **登入 Grafana**：預設存取 `http://YOUR_IP:3000` (預設帳密 admin/admin)。
2.  **設定資料源**：新增 `Prometheus` Data Source，URL 填入 `http://prometheus:9090`。
3.  **匯入 Dashboard**：
    - 側邊欄點選 **Dashboards** -> **Import**。
    - 點選 **Upload JSON file** 並選擇 `deploy/observability/dashboard.json`。
    - 選擇剛才建立的 Prometheus 資料源。

### Dashboard 結構說明 (按優先級)
1.  **🟢 節點健康總覽**：顯示節點是否在線 (Up)、Peer 數量、Uptime 以及記憶體佔用。
2.  **📦 同步與出塊**：最關鍵區塊，監控當前區塊高度、處理延遲以及是否有鏈重組 (Reorg) 發生。
3.  **🌐 RPC 狀態**：監控 DApp 存取量 (QPS)、請求錯誤率與回應時間分佈 (P99)。
4.  **💻 資源使用**：CPU、Disk I/O 與 Goroutines 趨勢，提早預警硬體瓶頸。
5.  **📡 P2P 網路**：流量吞吐量與節點連線性分佈。

---

## 10. 升級與回滾流程

> [!CAUTION]
> 在執行升級前，請務必確保節點已完成備份或使用 `upgrade.sh` 的自動備份功能。

### 升級指令
```bash
# 自動抓取最新版本並備份舊版後升級
./deploy/scripts/upgrade.sh

# 升級失敗？自動回滾至上一版本
./deploy/scripts/rollback.sh --force
```

---

## 11. 常見問題排除（Troubleshooting）

| 症狀 | 檢查方向 | 解決方法 |
| :--- | :--- | :--- |
| **Peer 數為 0** | 檢查 `.env` 中的 `BNES_BOOTNODES` | 確保填入官方 enode 地址；開啟防火牆 30303 (TCP/UDP)。 |
| **區塊高度停滯** | 執行 `./deploy/scripts/status.sh` | 檢查是否存在磁碟空間不足 (df -h) 或記憶體溢位。 |
| **RPC 413 錯誤** | 檢查 `nginx.conf` | 請求大小超過 `client_max_body_size`，請調大配置。 |

---

## 12. 安全最佳實務

1.  **金鑰管理**：嚴禁將 `keystore` 解鎖密碼檔案 (`password.txt`) 上傳至任何 Git 倉庫。
2.  **RPC 隔離**：絕不對外開放 `admin`, `debug`, `personal` API。
3.  **權限控制**：腳本目錄與資料目錄建議權限設為 `700` 或 `755`，並使用非 root 用戶執行程序。
4.  **Authority 守則**：驗證器節點應避免與其他 DApp 服務共用硬體，確保資源不被非預期負載爭搶。

---

## 13. 後續維護與貢獻

*   **日誌查看**：`tail -f deploy/logs/node.log`。
*   **提交 PR**：請確保代碼符合 BNES 形式化驗證規格，並通過 `make test`。
*   **聯繫官方**：若遇到核心物理引擎錯誤 (RF-1)，請立即回報至官方安全性小組。

---

> [!NOTE]
> 本文件由 BearNetworkChain 官方生成，基於 v1.1.0 規格報告書與 Grafana Dashboard v1.0.0 生成。
> 
> © 2026 BearNetworkChain Technical Documentation Team.
