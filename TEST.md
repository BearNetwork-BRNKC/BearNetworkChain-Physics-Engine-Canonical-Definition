# BRNKC 跨鏈橋專案執行流程及 SDK 調度完整性報告

## 📋 專案總覽與執行流程圖

```mermaid
graph TB
    subgraph "前端客戶端"
        A1[用戶輸入轉移資訊] --> B1[Wallet 連接驗證]
        B1 --> C1[簽名交易提交至鏈上]
        C1 --> D1[等待確認並讀取 txHash]
    end

    subgraph "REST API 層"
        E1[接收 /api/v1/bridge/transfer POST]
        E2[驗證 request body schema]
        E3[呼叫 bridge.ExecuteTransfer]
    end

    subgraph "Bridge Orchestrator"
        F1[normalizeAndValidate<br/>字串规范化與完整性檢查]
        F2[RPCVerifier.Verify<br/>雙交易驗證]
        F3[EvidenceGenerator.Generate<br/>證據綁定 hash 生成]
        F4[buildReleasePlan<br/>目的鏈釋放指令編譯]
    end

    subgraph "RPC Verifier"
        G1[BSC: feeTx 驗證<br/>(0.001 BNB → official)]
        G2[BSC/Native: mainTx 驗證<br/>(BRNKC 轉官方 / BRNKC native→official)]
        G3[ERC20 Transfer 事件比對]
        G4[Native transfer value 檢查]
    end

    subgraph "證據生成"
        H1[canonical JSON 序列化]
        H2[traceHash = SHA256(canonical)]
        H3[stateRoot = SHA256(trace + feeTx + mainTx)]
        H4[circuitHash = SHA256(state + version + mode)]
        H5[evidenceHash = SHA256(trace+state+circuit)]
    end

    subgraph "儲存層"
        I1[TransferRecord 寫入 store]
        I2[ID = hex(SHA256(sourceChain|destChain|sourceTxHash|userAddress))]
    end

    subgraph "目的鏈釋放準備"
        J1[BearNet: nativeTransfer<br/>from official wallet]
        J2[BSC: BRNKC transfer()<br/>to user address]
    end

    C1 --> E1
    E3 --> F1
    F1 --> F2
    F2 --> G1 & G2
    G1 & G2 --> H1
    H1 --> H5
    H5 --> I1
    I1 --> J1 & J2

    style C1 fill:#f96,stroke:#333
    style E1 fill:#4a8,elevation:3
    style F1 fill:#8bc,stroke-dasharray:5 5
    style G1 fill:#b7c,stroke-width:3px
    style H1 fill:#e8d,color:white
```

---

## 📦 SDK（bnessdk）適用性與引用規範

### 6.1 SDK 定義與定位

本專案所引用的 **BearNetworkChain SDK (bnessdk)**：

* **官方來源**：<https://github.com/BearNetwork-BRNKC/bnessdk>
* **程式語言**：Go
* **版本管理**：`go.mod` / `go.sum` 控制依賴（見 `/backend/go.mod`）
* **位置**：專案根目錄下的獨立模組 `/bearnetwork-crosschain-bridge/backend/bnessdk/`

本 SDK 在跨鏈橋中的角色屬於 **「外部能力封裝層」**，負責處理與 BearNet 節點協議、狀態證據產生及 PQC/ZK 相關的低階實作。Bridge Orchestrator 透過明確介面調用此 SDK，而非直接依賴節點原始碼或內部變數，確保核心邏輯可與後端實現解耦。

### 6.2 SDK 分類判定：外部引用層級

依 CLAUDE.md「SDK 適用性」規範判斷：

| 屬性 | 判定結果 |
|------|----------|
| **用途** | 供外部開發者（其他專案或工具鏈）整合 BearNet 能力 |
| **版本穩定度** | Go mod 封裝，可獨立 bump major/minor/patch |
| **介面明確性** | 透過 `/backend/bnessdk` 下的 package exports 提供 API surface |
| **與主倉庫同步必要性** | 否 — Bridge Orchestrator 僅調用公開函式與結構體，不修訂內部變數或依賴未匯出成員 |

因此本專案將 bnessdk 視為 **「外部 SDK 引用」**：Bridge Layer 透過明確的 `SDKAttestation` / `EvidenceBundle` 資料結構作為資料交換契約，而非直接存取節點記憶體。此設計符合「不可隨意破壞原則」與「擴展優先原則」。

### 6.3 SDK 適用方式（本專案實作）

在 `/backend/bridge/evidence.go` 中可觀察到：

* `EvidenceBundle` 結構體包含 `SDK` 欄位，其類型為 `SDKAttestation`
* `SDKAttestation` 定義如下重點：

```go
type SDKAttestation struct {
    Name              string `json:"name"`                // "bnessdk"
    Path              string `json:"path"`                // config.BNESSDKPath
    Compatibility     string `json:"compatibility"`       // bridge-local evidence binds verified chain data only; PQC/ZK must come from external BNES attestation boundary
    ForcedTxType      string `json:"forcedTxType"`        // requiredTxType (來源鏈交易類型規範)
    EvidenceSemantics string `json:"evidenceSemantics"`   // EvidenceHash 綁定已驗證轉帳載荷，不接觸 BNES 節點私料
}
```

此欄位明確宣示：本橋服務生成的證據 hash **僅保證「上鏈數據正確」**，而與 PQC、ZK proof 等更高等級安全性相關的內容，則必須依賴外部 BNES attestation service（由 `NodeAttestation.Endpoint` / `PublicTrustRoot` 提供）。這正體現了 SDK 引用層級的邊界設計。

### 6.4 引用規範摘要

| 項目 | 規則 |
|------|------|
| **版本相容** | Bridge Config (`config/config.go`) 中的 `BNESSDKPath` 必須指向與本橋節點相容的 bnessdk release，避免 ABI mismatch |
| **介面契約** | Bridge Orchestrator 僅透過 `EvidenceBundle` / `SDKAttestation` 資料結構交換資訊；不直接呼叫 SDK 內部變數或依賴未匯出成員 |
| **獨立測試** | `/backend/bridge/orchestrator_test.go`、`rpc_verifier_test.go` 均為純單位測試，無外部節點依賴，確保可重現驗證邏輯正確性 |

---

## 🌉 API 層設計原則與前後端交互細節

### 7.1 REST Endpoints 總覽（後端 Go API）

專案根目錄 `/backend/main.go` 定義了單一的跨鏈轉移入口點：

```
POST /api/v1/bridge/transfer
Request body: TransferRequest (JSON)
Response: TransferResponse { success, transferId, status, evidence?, message }
```

此設計簡潔明確，前端只需呼叫一次 POST 即可提交「雙交易證據」。後續查詢狀態可透過 `/status/{id}`（由 `GetStatus` 提供）。

### 7.2 Request Schema：TransferRequest

定義於 `/backend/bridge/types.go`：

| Field | Type | Purpose |
|-------|------|---------|
| sourceChain / destChain | string | "bsc" ↔ "bearnet"（必須不同） |
| amount | decimal string | 要轉移的 BRNKC 數量（十進位字串，`new(big.Int)` 解析） |
| userAddress | ETH-address | 發起者錢包（來源鏈與目的鏈必須相同） |
| recipientAddress | ETH-address | 接收地址；預設 = userAddress |
| nonce | uint64 | 前端提交時序號，避免重複提交同一交易 |
| feeTxHash | tx-hash | **手續費交易** hash：用戶支付 `0.001 BNB` 至官方地址的交易 ID（來源鏈） |
| mainTxHash | tx-hash | **代幣轉移**交易 hash：BRNKC → 官方橋地址的交易 ID（來源鏈）；若 sourceChain = "bearnet"，則為 Native transfer |
| observedTxType | string (opt) | BNES strict mode 下需比對的交易類型（如熊網鏈的 `0x7F`） |

前端提交時會將這七個欄位打包成 JSON Body：

```json
{
  "sourceChain": "bsc",
  "destChain": "bearnet",
  "amount": "1000000",
  "userAddress": "0x...",
  "nonce": 17523456789,
  "feeTxHash": "0x...",
  "mainTxHash": "0x..."
}
```

### 7.3 Response Schema：TransferResponse

成功時返回：

| Field | Meaning |
|-------|---------|
| success | bool（true） |
| transferId | hex(SHA256(sourceChain\|destChain\|sourceTxHash\|userAddress)) — 可用作查詢鍵 |
| status | "evidence_verified" / "pending_release" / "rejected" … |
| evidence | EvidenceBundle（見下節）— 供前端展示或第三方驗證器使用 |
| message | opt，錯誤訊息或提示文字 |

### 7.4 Frontend BridgeService.ts：前後端交互實作

位於 `/bearnetwork-crosschain-bridge/src/services/bridgeService.ts`。核心函式 `submitBridgeTransfer(request)` 的邏輯如下：

1. **API URL**：讀取 `VITE_BRIDGE_API_URL`，若未設定則預設為開發環境 `http://localhost:8080`
2. **HTTP POST**：  
   ```ts
   fetch(`${baseUrl}/api/v1/bridge/transfer`, {
     method: 'POST',
     headers: { 'Content-Type': 'application/json' },
     body: JSON.stringify(request),
   })
   ```
3. **錯誤處理**：若 `!response.ok`，則從 body 讀取 `error.message`；否則拋出「Bridge backend rejected transfer (${status})」通用訊息。
4. **成功回應**：直接返回原始 body（已被 TS 標註為 `BridgeTransferResponse`）。

前端 App.tsx 在用戶點擊「跨鏈轉移」按鈕時，會先連狐狸錢包取得簽名後的 feeTxHash / mainTxHash，再呼叫上述 service。整個流程無阻塞式 await，符合現代 SPA 設計模式。

### 7.5 RPC Verifier（後端）：證據驗證邏輯細節

位於 `/backend/bridge/rpc_verifier.go`。核心 `Verify(req TransferRequest)` 會依序執行三個步驟：

1. **verifyFee** —— 檢查手續費交易是否正確
   * 從來源鏈（固定為 "bsc"）讀取 feeTxHash
   * 驗證 receipt.Status == "0x1"
   * 確認 confirmations >= `config.Networks["bsc"].Confirmations`
   * **From** = userAddress，**To** = config.FeeRecipient，**Value** = FixedFeeWei（對應 `0.001 BNB`）

2. **verifyMain** —— 檢查代幣轉移交易
   * 讀取 mainTxHash → receipt
   * From = userAddress
   * 依來源鏈分支：
     - **BSC ERC20**：驗證 tx.To == TokenAddress，並掃描 Receipt.Logs，尋找 Transfer(event) 事件，要求 `from`（log.Topics[1]）== userAddress、`to`（log.Topics[2]）== BridgeAddress、Data == amount
     - **BearNet Native**：驗證 tx.To == BridgeAddress、tx.Value == amount；若 strict mode 開啟且 network.RequiredTxType != ""，則再比對 `tx.Type == RequiredTxType`

3. 所有檢查通過後返回 nil（成功），否則拋出帶有 cause 的 error。

此邏輯確保「雙交易」皆正確無誤，才允許橋接服務繼續處理證據生成與目的鏈釋放準備。

### 7.6 Bridge Orchestrator：核心協調流程說明

位於 `/backend/bridge/orchestrator.go`。`ExecuteTransfer(req)` 的執行步驟如下：

1. **normalizeAndValidate** —— 字串规范化（轉小寫、去空白）與完整性檢查
   * sourceChain / destChain 必須為 config.Networks 中的 key
   * userAddress / recipientAddress 符合 ETH-address regex `^0x[0-9a-fA-F]{40}$`
   * amount 可用 `new(big.Int).SetString(...,10)` 解析且 > 0
   * nonce ≠ 0；feeTxHash ≠ mainTxHash

2. **重複交易檢查**：透過 store.ExistsTx(txHash) 避免同一筆 txHash 被提交多次。

3. **驗證器調用**：`o.verifier.Verify(req)`（預設為 RPCVerifier）確認雙交易正確性。

4. **證據生成**：`evidence := o.evidence.Generate(req, requiredTxType)`
   * `requiredTxType` 若來源鏈是 "bearnet" 且 config.StrictBNESTxType = true，則填入 `"0x7F"`；否則為空。
   * EvidenceGenerator 會依序計算：canonical JSON → traceHash → stateRoot（trace + feeTx + mainTx）→ circuitHash（state + version + mode）→ evidenceHash（trace+state+circuit）。這些 hash 值會被寫入 `EvidenceBundle`。

5. **儲存**：構建 `TransferRecord{ID, Status: "pending_release", Request, Evidence, ReleasePlan, ...}`，並透過 store.Save 寫入資料層。

6. **建立 release plan**（見下節）。

### 7.7 Release Plan：目的鏈釋放指令編譯

`buildReleasePlan(req TransferRequest, evidence EvidenceBundle)` 依來源與目的鏈組合決定不同行動：

#### BearNet → BSC
* `Action: "send_bsc_brnkc_from_official_bsc_wallet"`
* `SenderAddress: config.Networks["bsc"].BridgeAddress`（官方橋錢包）
* `TargetAddress: TokenAddress`（BRNKC ERC20 contract）
* `Method: "transfer(address,uint256)"`
* Arguments：包含 recipient、amount、srcTxHash、bridgeEvidenceHash 等

#### BSC → BearNet
* `Action: "send_native_brnkc_from_official_bearnet_wallet"`
* `SenderAddress: config.Networks["bearnet"].BridgeAddress`
* `Method: "nativeTransfer(address,uint256)"`
* Arguments：同樣包含證據 hash、stateRoot 等

此設計確保橋接服務的釋放動作完全由「來源鏈雙交易驗證」驅動，而非硬編碼任何目的鏈邏輯。

---

## 🐳 Docker Compose 部署架構（簡要）

位於 `/bearnetwork-crosschain-bridge/docker-compose.yml`：

| Service | Image / Build | Role |
|---------|---------------|------|
| `api` | `golang:1.23-bookworm` + multi-stage build | 編譯 Go 後端並作為長跑容器（port 8080） |
| `frontend` | `node:22-alpine` | npm install → vite build，然後 serve dist/ |

啟動流程：先 `docker compose up -d`，前端與 API 分別監聽不同端口。此結構符合「最小必要修改」原則，無額外部署工具鏈。

---

## 📌 驗證結果摘要（本報告涵蓋範圍）

| 檢查項目 | 狀態 |
|----------|------|
| REST endpoint schema (POST /api/v1/bridge/transfer) 符合 TransferRequest | ✅ |
| Bridge Orchestrator normalizeAndValidate 與 RPCVerifier.Verify 邏輯完整度 | ✅ |
| EvidenceBundle hash 計算可追溯性（canonical → traceHash → stateRoot → circuitHash → evidenceHash） | ✅ |
| ReleasePlan 依來源/目的鏈分支正確產生對應 Action / Method | ✅ |
| Docker Compose 服務定義與端口分配符合預期 | ✅ |

---

## ⚠️ 剩餘風險與後續建議

1. **BNES attestation service**：目前 `config.BNESSDKPath`、`NodeAttestation.Endpoint` 多為空或未配置；若未來需要更高層級的 PQC/ZK proof，必須另行部署外部 attestation 服務並更新 config。
2. **測試覆蓋率提升**：現行 `/backend/bridge/orchestrator_test.go`、`rpc_verifier_test.go` 已驗證核心邏輯，建議加入端到端（E2E）流程測試（前端 → API → store），確保整體交付物無縫運作。
3. **部署監控與 Log**：Docker Compose 啟動後建議配置 `docker logs --tail=100` 以觀察任何初始化錯誤；現階段尚未設定集中式 log aggregation，可於未來依需求擴增。

---

報告撰寫完成。以上內容已整合本專案所有公開程式碼文件與部署定義，供後續審計或交付備查使用。
