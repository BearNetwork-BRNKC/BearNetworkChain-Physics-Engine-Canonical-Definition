# 🐻 BearNetworkChain BNES Project map (Full Canonical Edition)

本文件為 BearNetworkChain (BNC) 專案結構的完整映射，嚴格對齊 **BNES v1.3 (Canonical Locked Edition)** 規格。本文件禁止任何形式的語義省略，旨在為 AI 審計器、開發者及維運人員提供 100% 透明的程式碼地圖。

---

## Ⅰ. 核心規格與權威文檔 (Canonical Specifications)
系統的唯一語義來源，定義了所有物理算子與正確性判定標準。

- 📜 **[Execution Specification v1.3]**: 核心形式化規格，定義 $\Gamma, \Sigma, \Im, F, \mathcal{E}, \psi, k$ 等符號語義。
- 📊 **[Node Runtime 總規格報告書]**: 實作狀態、性能數據與紅旗系統審計總結。
- 📖 **[Paradigm Contract Guide]**: Solidity 範式合約開發與 $\Sigma$ 流形限制指南。
- 🛠️ **[Core Interfaces]**: 定義量子屏蔽、$\Gamma$ 獲取、$\psi$ 相位觀測等核心技術接口。
  - **`Breath(pressure int)`**: 物理-存儲耦合接口，將 $\Gamma$ 壓強傳導至資料庫層，啟動自適應 I/O 阻尼機制。


---

## Ⅱ. 核心執行與物理引擎 (Core Physics & EVM)
負責狀態轉移（State Transition）與物理律不變量（Invariant）的計算與觀測。

### 1. Gamma Engine (Γ) - 執行不變量觀測
- `core/gamma_engine.go`: 核心動力學方程實作，計算全域一致性標量。
- `core/state_processor_gamma.go`: 將 $\Gamma$ 計算掛載至區塊處理生命週期。
- `core/information_flux.go`: **ℑ (Information Flux Field)** 實作，映射交易對狀態的資訊擾動。
- `core/state_processor_poac.go`: 物理證明與共識對齊處理器。

### 2. EVM & State Manifold (Σ)
- `core/vm/`: 核心 EVM 實作，保留 100% 圖靈完備語義。
- `core/state/`: 狀態樹與管理層，提供 O(1) 物理計數器。
- `triedb/pathdb/`: **Path-based 狀態索引方案**，BNES v1.3 強制標準，用於加速 ZK 見證提取。
- `eth/ethconfig/config.go`: 核心配置中心，強制鎖定 `StateScheme: "path"` 以維持物理一致性。
- `core/blockchain.go`: 鏈式結構管理，確保決定性（Deterministic）生長。

- `core/rawdb/`: 邏輯層資料操作，對齊物理觀測路徑。
- `ethdb/`: 物理感知的驅動與介面層。
  - `pebble/pebble.go`: **唯一生產級存儲引擎**，實作 **Obsidian Resilience** 與 `Breath()` 動態壓強調優。
  - `leveldb/`: **Legacy 遺留驅動** (不支援物理調優，BNES v1.3 預設禁用)。
  - `memorydb/`: **測試用內存驅動** (僅供 Sandbox 環境模擬使用)。



- `triedb/`: Trie 專屬資料庫層，支援 Path-based 狀態方案。


### 3. Red Flag Engine - 判定與防禦
- `core/red_flag_engine.go`: 完整實作 RF-1 ~ RF-15 判定邏輯，包含 **ARI-Model** (對抗式紅旗注入防禦)。
- `core/pressure_monitor.go`: 系統飽和度與資源耗散監控。

---

## Ⅲ. 量子信任層與零知識證明 (Trust Root & ZK)
確保身分授權的量子抗性與執行過程的可驗證性。

### 1. PQC Trust Root Layer (σ)
- `bnessdk/srsg/`: SRSG (Signature Reference State Generator) 引擎。
  - `srsg.go`: 簽章生命週期統籌。
  - `compiler_impl.go`: 身份與密碼學政策（P）編譯與綁定。
- `crypto/secp256k1/`: 經典簽章支援（用於 Hybrid 模式）。
- `crypto/signify/`: 針對 BNES 優化的量子簽章封裝。

### 2. ZK Verifiable Computation (Π & τ)
- `core/zk_witness_provider.go`: **W (Witness)** 生成器，擷取執行軌跡見證。
- `core/zk_recursive_compressor.go`: 遞迴 ZK 證明壓縮技術，優化歷史驗證速度。
- `bnessdk/zk/`: ZK 證明介面與電路哈希（$\tau$）校驗。
- `bnessdk/witness/`: 執行證明路徑的拓樸追蹤。

---

## Ⅳ. 共識、排序與網路拓樸 (Consensus & Topology)
負責交易順序的確定性排列（Deterministic Ordering）與節點間的拓樸通訊。

### 1. Deterministic Ordering Layer
- `consensus/clique/`: 純化後的 PoA 共識引擎。
  - `clique.go`: 確定性出塊與簽署者驗證。
  - `snapshot.go`: 權限狀態快照。
- `internal/ordering/epoch_manager.go`: **EpochManager**，消除共識隨機性，維持 ψ 相位一致性。

### 2. IGCP 跨星系協議
- `core/igcp.go`: **IGCP (Inter-Galactic Checkpoint Protocol)**，實作返航驗證與長時空錨定。
- `core/poac.go`: 物理一致性證明校驗。

### 3. Topology Space (F)
- `bnessdk/gamma/light_verifier.go`: 拓樸觀測算子於輕節點的投影實作。
- `sim/gamma_trace_sim.go`: 物理軌跡與拓樸特徵模擬器。

---

## Ⅴ. 門戶介面、SDK 與相容層 (Gateway, SDK & Compatibility)
對外提供高效率、零摩擦的技術接入點。

- `bnessdk/sdk/`: Node JS SDK (v2.0) 核心，支持 Pre-flight Sandbox 預執行。
- `internal/observation/artifact.go`: **Artifact Layer**，將 AI 推理結果強制收斂至 BNES 再評估閉環。
- `core/types/block.go` & `header.go`: **15/16 Shadow Resolver**，實作影子欄位與跨版本欄位投影。
- `rpc/`: 高效能 JSON-RPC 2.0 介面實作，支援 `eth_` 標準介面。
- `bnql/`: **BNQL & DQK (已上線)**，全稱 *Bear Network Query Language + Deterministic Query Kernel*。支援全域金流拓樸觀測、128跳深度追蹤與 **F-Space 跨跳數特徵穿透**。具備零動態分配、可重播安全 (Replay-safe)、`LockOSThread` 隔離排程與無鎖 IPC 雙環 (Ingress/Egress Ring) 輪詢的物理安全微內核。系統將查詢軌跡編譯為幾百位元組的反事實見證信封 (`FIC`)，**用戶在手機上只需驗證一個幾百 bytes 的 ZK Proof + FIC，就能同時確認 PQC 簽名合法性與查詢結果的物理必然性**。
- `bnscan/`: **物理觀測儀 (BNC-Scan)**，提供 i18n 多語系支持（繁中/英文）的互動式 BNQL 終端與 2D/3D 流形投影視覺化界面。 (位於 `bnscan/web` 與 `bnscan/cmd/api`)
- `cmd/bnes/`: 核心入口與機器可讀審計指令集。

---

## Ⅵ. 部署、安全性與觀測（Operations & Security）
確保生產環境的強韌性與自動化運修。

### 1. Production Deployment Suite (`deploy/`)
- `deploy/scripts/`: 7 個生產級防護腳本。
  - `install.sh`: 環境預檢與初始化。
  - `start.sh`: 帶有存活驗證的服務啟動。
  - `upgrade.sh`: 包含 PQC/ZK 安全抽換的無痛升級。
  - `rollback.sh` / `restart.sh` / `stop.sh` / `status.sh`。
- `deploy/configs/`: 環境配置體系。
  - `nginx.conf`: RPC 防護代理、限流與斷路器。
  - `rpc.yaml`: JSON-RPC 路由與安全過濾。
  - `env.example`: 100% 抽離的環境變數範本。
  - `genesis_mainnet.json`: 含有 PQC 簽章政策之創世配置。

### 2. Observability Stack
- `internal/observation/`: 物理指標匯集器。

- `deploy/observability/`: Grafana Dashboard 與 Prometheus 告警 (Monitoring)。

---

## Ⅶ. 雜項與支持組件 (Miscellaneous)
- `accounts/`: 帳戶管理與金鑰庫。

- `p2p/`: 抗干擾對等網路層。
- `rlp/`: 高效能序列化（對齊物理算子）。
- `trie/`: 狀態樹結構，支援 ZK-Proof 所需的證據路徑。
- `tmp/`: 系統測試與臨時數據緩衝。

---
*Created by Antigravity (Auditing Mode) | Synchronized with BNES v1.3 Canonical Edition*
