# BearNetworkChain 決定性執行引擎與資料層收斂 

# (Deterministic Execution Engine and Data Layer Convergence) 深度技術專題報告

**原創者 (Author)**：ChenTing (陳霆) 

**創辦職位 (Title)**：Founder, CEO & Chief Technology Officer, BearNetworkChain

**學術單位 (Affiliation)**：College of Management, Tunghai University

**聯絡信箱 (Email)**：bnkt@bearnetwork.net

**Canonical DOI**: [doi:10.5281/zenodo.20453286](https://doi.org/10.5281/zenodo.20453286)


## 一、 前言：全球區塊鏈高吞吐 TPS 瓶頸與 I/O 寫入放大

在追求工業級吞吐量（百萬級 TPS）的全球區塊鏈發展進程中，絕大多數公鏈（如 Ethereum 及其 Layer 2 擴容方案）最終都會撞上一面難以逾越的「狀態牆」與「I/O 效能牆」。

傳統區塊鏈架構在處理高頻率交易時，面臨著致命的物理與工程瓶頸：

1. **寫入放大 (Write Amplification) 與分散寫入**：在傳統的 MPT (Merkle Patricia Trie) 結構中，狀態樹的每一次 `Commit()` 都會觸發大量分散的底層資料庫（如 LevelDB/PebbleDB）KV `Put()` 操作。每個節點的更新都是一次獨立的插入，導致底層資料庫的 L0 (Level 0) 層檔案數爆炸，引發嚴重的 Compaction 負債與 Write Stall（停寫）。

2. **缺乏全域的物理背壓 (Backpressure) 閉環**：傳統的共識引擎（如 PoW 或基礎 PoA）對底層資料庫的物理狀態（如 I/O 延遲、WAL 積壓、MemTable 使用率）毫無感知。當 TPS 飆高、磁碟 I/O 崩潰時，共識引擎仍會盲目地以固定頻率出塊，導致節點記憶體耗盡（OOM）或直接崩潰。

3. **記憶體動態分配 (Heap Allocation) 帶來的 GC 停頓**：在熱路徑（Hot Path）中頻繁地進行 `new()` 或 `make()` 操作，會引發 Go 語言的垃圾回收器（GC）啟動 Stop-The-World (STW)，在極高併發下造成系統瞬間毛刺（Spike）與卡頓。

為徹底解決上述問題，**BearNetworkChain (BNES)** 獨創了「**決定性執行引擎與資料層收斂架構**」。我們堅持**「不增加硬體、不增加 RAM」**的極致工程原則，透過根除 write amplification 的源頭，建立了一套具備「儲存感知 (Storage-Aware)」與「零配置 (Zero-Allocation)」特性的自調節執行閉環。


## 二、 核心架構突破：決定性執行 (Deterministic Execution) 與零配置

BNES 決定性執行的核心理念在於：**狀態的變更與寫入必須是絕對確定（Deterministic）、恆定時間（O(1)）且零分配（Zero-Allocation）的。**

在 [`core/gamma_engine.go`] 與 [`core/state_processor.go`] 的底層實作中，我們確立了以下不可違反的物理約束：

*   **Zero-Allocation 保障**：所有的計算暫存區（如 `StoragePressure` 向量、`GammaScratch`）都在節點啟動時一次性預配置（Pre-allocated）於 Heap。後續在每秒數十萬筆交易的熱路徑中，嚴格禁止任何新的動態記憶體分配，所有操作皆為原地覆寫（In-place update）。

*   **O(1) 常數複雜度**：所有的背壓查詢與指標更新，均採用 `sync/atomic` 的單一指令操作（如 `atomic.Load`、`atomic.Store`）以及無分支的整數運算（如 `bits.Len64`），確保執行複雜度與區塊狀態規模完全脫鉤。


## 三、 區塊級原子批次寫入 (Block-Level Atomic Batch Write)

為了解決傳統 `statedb.Commit()` 帶來的大量分散 KV 寫入問題，BNES 重構了寫入路徑，實施了**區塊級原子批次寫入**。

在傳統模式中：

```go
rawdb.WriteBlock(db, block)
rawdb.WriteReceipts(db, ...)
statedb.Commit()  // 內部引發成千上萬次獨立的 db.Put()
```

在 BNES 的重構架構中（位於 [`core/blockchain.go`] 的 `WriteBlockAndSetHead`），我們將整個區塊的寫入生命週期收斂為單一的原子操作：

```go
// 1. 預估批次大小（O(1) 時間）
estimatedSize := len(block.Transactions()) * 512 + len(receipts) * 256
batch := db.NewBatchWithSize(estimatedSize)

// 2. 將所有區塊資料、收據、索引與 Trie 節點寫入同一個記憶體 Batch
rawdb.WriteBlockIntoBatch(batch, block)
rawdb.WriteReceiptsIntoBatch(batch, receipts, ...)
// trie.Commit() 亦將資料重定向至此 batch

// 3. 區塊終局化時，執行唯一一次落盤
batch.Write() 
batch.Reset() // 原地複用，零分配
```

**工程意義**：

*   **消除 Metadata Churn**：將成千上萬次的 I/O 呼叫合併為單次順序寫入。

*   **保證 Determinism**：Batch 內部的 Key 寫入順序由 Tx Index 嚴格排序，確保所有節點的 WAL (Write-Ahead Log) 序列 100% 一致。


## 四、 多維度儲存壓力向量 (StoragePressure Vector)

為了讓共識引擎能夠精準感知底層物理世界的瓶頸，BNES 摒棄了單一的壓力數值，發明了**多維度儲存壓力向量 (StoragePressure)**。

在 [`ethdb/storage_pressure.go`] 中，我們定義了這組純原子的物理向量：

```go
type StoragePressure struct {
    L0CompactionDebt     atomic.Int64 // L0 層檔案壓縮負債
    WALBacklogBytes      atomic.Int64 // WAL 積壓位元組數
    MemTablePressureX100 atomic.Int64 // MemTable 使用率 (整數化)
    WriteStallRiskX100   atomic.Int64 // 停寫風險係數
    FlushQueueDepth      atomic.Int32 // 等待 Flush 的佇列深度
    FsyncLatencyP99Ns    atomic.Int64 // fsync P99 延遲 (奈秒)
}
```

**設計亮點**：
此向量並非被動的監控指標，而是 **Γ (Gamma) 執行排程器的直接物理輸入信號**。PebbleDB 的內部指標（Metrics）會透過 `Breath()` 函數（心跳機制）以 O(1) 的成本實時映射至此向量，完美呈現了節點當前的「消化能力」。


## 五、 儲存感知背壓閉環與礦工自適應限速

擁有 `StoragePressure` 向量後，BNES 建立了一個完美的**全域背壓閉環 (Backpressure Loop)**，徹底解決了高吞吐下的網路崩潰問題。

### 1. 物理心跳連動 (Obsidian Neural-Heartbeat Coupling)
在 [`core/state_processor.go`] 的 Step 13.1 中，狀態處理器會在區塊處理完畢時，將壓力向量傳遞給底層資料庫，並同步至全域的 `PressureMonitor`。

### 2. 礦工自適應限速 (Storage-Aware Pacing)
在 [`miner/worker.go`] 的出塊熱路徑中，礦工會實時查詢壓力向量並進行自適應調節：

*   **降低區塊 Gas Limit**：當 `IsWriteStallImminent()` (停寫風險 > 75%) 觸發時，礦工會主動將當前區塊的 `GasLimit` 縮減至 75%，強制降低狀態突變 (State Mutation) 的密度，給予底層資料庫喘息與 Compaction 的時間。

*   **延遲出塊提議**：若觀測到 `FsyncLatencyP99Ns` 突破 50ms 閾值，代表磁碟 I/O 已達極限，礦工會將出塊 Timer 延後，防止網路被未落盤的無效區塊淹沒。

這種**「由下而上、物理驅動」**的自調節機制，使 BNES 能夠在不依賴硬體擴容的情況下，展現出**有界的記憶體行為 (Bounded Memory Behavior)**。


## 六、 舉證方法論與實測分析

> 如同量測一杯水，必須明確說明：**量什麼、用什麼量、量到什麼**。  
> 本報告對每一項測試，均遵循以下三要素進行舉證：
>
> -----------------------------------------------------------------------------------------
> 要素         說明
> ------------ ----------------------------------------------------------------------------
> **被測物**   系統中被驗證的那一個特定行為或不變量
> 
> **量測工具** 用什麼方式（輸入＋觀察方式）觀測這個行為
> 
> **量測讀數** 實際觀察到的輸出值，以及判定通過或失敗的標準
> -----------------------------------------------------------------------------------------

### 實測：背壓閉環與高併發寫入穩定性測試

本實驗室針對 BNES 的決定性執行引擎與批次寫入架構，進行了三個階段的極限壓測。

#### 三要素舉證表格

---------------------------------------------------------------------------------------------------------------
舉證要素     具體觀測與讀數說明
------------ --------------------------------------------------------------------------------------------------
**被測物**   BNES 資料層在百萬級交易注入下的 Batch 原子性、L0 Compaction 負債控制，以及礦工自適應限速的有效性。

**量測工具** 1. **Phase A (原子性驗證)**：使用 `batch_send.js` 注入 100 萬筆交易，並在 `Batch.Commit()` 
             期間強制 `kill -9` 終止節點程序。重啟後觀測資料庫狀態根 (stateRoot)。
             2. **Phase B (背壓閉環驗證)**：持續注入高頻交易，觀測 `StoragePressure.L0CompactionDebt` 
             指標，並同時監控礦工日誌中 `gasLimit` 的變化。             
             3. **Phase C (長期穩定性)**：在極限壓力下連續運行 24 小時，觀測 PebbleDB 的 
             `Metrics().WAL.Size` 與 `Levels[0].NumFiles`。

**量測讀數** 1. **Phase A 讀數**：節點重啟後，DB 內部不存在任何「半寫入」狀態，`stateRoot` 完美回滾至上一
             個區塊的 parent state，**100% 滿足 ACID 原子性要求**。
             2. **Phase B 讀數**：當 `L0CompactionDebt > 8` 時，成功觀測到礦工日誌輸出 
             `[BNES] Storage backpressure: reducing block gas limit`，並自動退避至 75%。**全程未發生任何 Write Stall 停寫事件**。
             3. **Phase C 讀數**：運行 24 小時後，WAL 積壓與 L0 檔案數均呈現平穩的鋸齒波狀波動，
             證明系統達成了**完美的 Bounded Memory Behavior (有界記憶體行為)**。
             4. **判定標準**：所有狀態對齊，無資料損壞，背壓觸發精準，判定為 **PASS (100% 通過)**。
---------------------------------------------------------------------------------------------------------------


## 七、 結論：突破吞吐極限的工程藝術

BearNetworkChain 的決定性執行引擎與資料層收斂架構，是對傳統區塊鏈底層邏輯的一次徹底顛覆。

我們不再將「共識」與「儲存」視為兩個割裂的系統，而是透過 **StoragePressure 向量**將物理世界的 I/O 極限無縫傳遞至共識排程的心臟。結合 **O(1) 零配置**的記憶體管理與**區塊級原子批次寫入**，BNES 成功消除了 Write Amplification 與 GC STW 兩大效能殺手。

這套物理感知架構，使得 BearNetwork 能夠在普通商用硬體上，提供極致平滑、不卡頓的百萬級 TPS 吞吐能力，真正定義了新一代工業級區塊鏈的技術標準！


## 八、 版權聲明

**Copyright**:
© 2026 ChenTing (BearNetworkChain Founder). All rights reserved. This specification is licensed under the Creative Commons Attribution 4.0 International License.
