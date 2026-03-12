# Self-Hosted DBaaS Blue/Green Deployment 設計文件

> **文件定位**：這是一份**團隊決策討論工具**——聚焦「我們要做哪些決策」，團隊可以逐項討論、記錄結論。

---

**目錄**

- [1. 目標與範圍](#1-目標與範圍)
- [2. 決策地圖（Decision Map）](#2-決策地圖decision-map)
- [3. 策略決策](#3-策略決策)
  - [Decision 0：策略選擇](#decision-0策略選擇new-green-vs-zone-b)
  - [Decision 1：Green 初始化](#decision-1green-環境初始化策略)
  - [Decision 2：Replication 拓撲](#decision-2bluegreen-期間的-replication-拓撲)
  - [Decision 3：Switchover 機制](#decision-3switchover-機制)
  - [Decision 4：Zone-B 處理](#decision-4switchover-期間的-zone-b-處理)
  - [Decision 5：交付格式](#decision-5交付格式)
  - [Decision 6：Guardrails 檢查](#decision-6guardrails-檢查)
  - [Decision 7：回滾與 Timeout](#decision-7回滾與-timeout)
  - [Decision 8：排程任務處理](#decision-8排程任務處理)
- [4. 執行流程](#4-執行流程)
- [5. 風險與緩解](#5-風險與緩解)
- [6. 下一步行動](#6-下一步行動)
- [附錄 A：Switchover 連線流程圖](#附錄-aswitchover-連線流程圖)
- [附錄 B：SQL 指令參考](#附錄-bsql-指令參考)
- [附錄 C：資訊盤點清單](#附錄-c資訊盤點清單)
- [附錄 D：AWS RDS Blue/Green 對照](#附錄-daws-rds-blugreen-對照)

---

## 1. 目標與範圍

### 為什麼需要 Blue/Green

在 Kubernetes 上運行 MariaDB（透過 mariadb-operator）時，版本升級、設定變更與 Schema Migration 都會影響服務可用性。目前缺乏標準化的**零停機**升級流程，每次變更都是高風險的手動操作。

**問題陳述**：我們需要為自建多租戶 DBaaS 設計一套 Blue/Green Deployment 流程，參考 AWS RDS Blue/Green 概念，提供可重複、可回滾的升級機制。

> **背景**：與 AWS RDS 不同，自建的 Kubernetes 叢集使用 [mariadb-operator/mariadb-operator](https://github.com/mariadb-operator/mariadb-operator)（社群版）來管理 MariaDB 實例。因此無法直接使用 AWS 提供的 Blue/Green API，需要自行設計對應的流程。

### 目前架構

```
┌─── App K8s Cluster(s) ───┐
│                           │
│   App-A    App-B   App-C  │
│     │        │       │    │
└─────┼────────┼───────┼────┘
      │        │       │
      ▼        ▼       ▼
  ┌─────────────────────────┐
  │    Ingress Gateway       │  ◄── 跨叢集連線入口
  └────────────┬────────────┘
               │
┌──────────────┼── DB K8s Cluster ──────────────────────┐
│              ▼                                         │
│  ┌──────────── Zone-A (Primary Zone) ────────────┐       │
│  │                                             │       │
│  │   Primary ──semi-sync──► Replica-1          │       │
│  │            └─semi-sync──► Replica-2         │       │
│  │                                             │       │
│  └──────────────────┬──────────────────────────┘       │
│                     │ async (GTID-based)               │
│                     ▼                                  │
│  ┌──────────── Zone-B (DR Zone) ─────────────────┐       │
│  │                                             │       │
│  │   Primary ──semi-sync──► Replica-1          │       │
│  │            └─semi-sync──► Replica-2         │       │
│  │                                             │       │
│  └─────────────────────────────────────────────┘       │
│                                                        │
└────────────────────────────────────────────────────────┘
```

| 項目 | 說明 |
|------|------|
| **連線方式** | App Cluster → Ingress Gateway（跨叢集）→ DB Cluster 內的 K8s Service → MariaDB Pods |
| **Operator** | mariadb-operator/mariadb-operator（社群版） |
| **Zone 內複製** | Semi-synchronous Replication（半同步） |
| **跨 Zone 複製** | GTID-based Asynchronous Replication（非同步） |
| **多租戶** | 同一 DB Cluster 上有多個團隊的 DB 實例 |
| **關鍵細節** | DB 和 App 在**不同的 K8s Cluster** |

---

## 2. 決策地圖（Decision Map）

### 決策依賴關係

```
Decision 0: 策略選擇（New Green vs Zone-B）
  ├──► Decision 1: Green 初始化（僅 Option A）
  ├──► Decision 2: Replication 拓撲（僅 Option A）
  ├──► Decision 4: Zone-B 處理
  ├──► Decision 8: 排程任務處理
  │
  ▼
Decision 3: Switchover 機制
  ├──► Decision 6: Guardrails 檢查
  └──► Decision 7: 回滾與 Timeout

Decision 5: 交付格式（獨立，可最後決定）
```

### 一頁式總覽

| # | Decision | 核心問題 | 選項數 | 前置 | 建議 |
|---|----------|---------|-------|------|------|
| 0 | 策略選擇 | New Green or reuse Zone-B？ | 3 | 無 | 視 DR SLA |
| 1 | Green 初始化 | 資料怎麼來？ | 3 | D0=A | Snapshot |
| 2 | Replication 拓撲 | Chain or Fan-out？ | 2 | D0=A | Chain |
| 3 | Switchover 機制 | Layer 2 用什麼切？ | 2 | D0 | K8s Service |
| 4 | Zone-B 處理 | Switchover 期間 DR 怎麼辦？ | - | D0 | 依 D0 |
| 5 | 交付格式 | 自動化到什麼程度？ | 4 | 無 | Runbook 起步 |
| 6 | Guardrails | Switchover 前查什麼？ | - | D3 | 新增 |
| 7 | 回滾 + Timeout | 失敗怎麼辦？等多久？ | - | D3 | 新增 |
| 8 | 排程任務 | Green Event Scheduler？ | 2 | D0 | 關閉 |

---

## 3. 策略決策

> **格式說明**：每個 Decision 使用統一模板，團隊可逐項討論並填入結論。

---

### Decision 0：策略選擇（New Green vs Zone-B）

**問題**：Blue/Green 策略要建立全新 Green Cluster，還是重用現有 Zone-B？
**影響範圍**：Decision 1, 2, 3, 4, 8 的選項都取決於此
**前置決策**：無（這是最上游的決策）

#### 選項比較

**Option A：建立新的 Green Cluster（經典 Blue/Green）**

```
[BEFORE]
  Zone-A (Blue) 1P+2R ──async──► Zone-B (DR) 1P+2R
  ▲ Apps 連到這裡

[DURING]
  Zone-A (Blue) 1P+2R ──async──► Zone-B (DR) 1P+2R
  ▲ Apps            └──GTID──► Zone-A (Green) 1P+2R  ◄── 新建，升級版本

[AFTER SWITCHOVER]
  Zone-A (Green) 1P+2R ──async──► Zone-B (DR) 1P+2R    ◄── Zone-B 重新指向 Green
  ▲ Apps 連到這裡
  Zone-A (old Blue) → 刪除
```

**Option B：將 Zone-B (DR) 重新作為 Green**

```
[BEFORE]
  Zone-A (Blue) 1P+2R ──async──► Zone-B (DR) 1P+2R
  ▲ Apps 連到這裡

[STEP 1 — 升級 Zone-B]
  Zone-A (Blue) 1P+2R ──async──► Zone-B (Green) 1P+2R  ◄── 就地升級或重建
  ▲ Apps              Replication 追上後同步

[STEP 2 — Switchover]
  Zone-A (old Blue) 1P+2R        Zone-B (Green) 1P+2R
                                ▲ Apps 透過 Ingress Gateway 連到這裡

[STEP 3 — 重建 DR]
  Zone-A (new DR) 1P+2R ◄──async── Zone-B (Green/Primary) 1P+2R
                                  ▲ Apps 連到這裡
```

**Option C（混合方案）：升級期間建立臨時最小 DR**

```
[DURING Zone-B UPGRADE]
  Zone-A (Blue) 1P+2R ──async──► Temp-DR (1P+0R)    ◄── 臨時最小 DR
                               Zone-B 升級中...

[AFTER SWITCHOVER]
  Zone-A (new DR) 1P+2R ◄──async── Zone-B (Green/Primary) 1P+2R
  Temp-DR → 刪除
```

| 維度 | Option A | Option B | Option C |
|------|----------|----------|----------|
| 升級期間 DR 是否存在？ | ✅ 完整 DR | ❌ DR 中斷 | ⚠️ 臨時最小 DR |
| 資源需求 | 3x（最多） | 1x（不增加） | ~1.3x |
| Rollback 難度 | 低（切回 Blue） | 高（需要切 Zone） | 中 |
| Primary Zone 是否改變？ | 否 | 是（移到 Zone-B） | 是（移到 Zone-B） |
| 實作複雜度 | 高 | 低 | 中 |

#### 建議

> Option B 在營運上較簡單且省資源，但 **DR 缺口是真實風險**。如果 SLA 允許計畫性維護時暫時失去 DR（例如有維護窗口），Option B 很有吸引力。如果 DR 必須隨時存在，Option A 更安全。Option C 是折衷方案。

#### 討論問題

- [ ] 目前 DR SLA 是什麼？是否允許計畫性維護時暫時失去 DR？
- [ ] Cluster 剩餘資源是否足以支撐 Option A 的 3x 需求？
- [ ] Primary Zone 改變（Option B/C）對網路延遲和 App 效能的影響可接受嗎？
- [ ] 我們有多少信心能在 Zone-B 升級失敗時快速恢復 DR？

#### 結論

> _（討論後填寫）_

---

### Decision 1：Green 環境初始化策略

**問題**：Green 環境的資料從哪裡來？
**影響範圍**：決定 Green 環境建立時間，影響整體流程時長
**前置決策**：Decision 0 = Option A（Option B 可跳過此決策，Zone-B 已有資料）

#### 選項比較

| 維度 | A. Backup + Restore | B. Volume Snapshot Clone | C. 全新實例 + GTID 追趕 |
|------|-------------------|----------------------|---------------------|
| 速度 | 慢（100GB+ 需數小時） | 快（分鐘級） | 非常慢（大型資料集） |
| 前提條件 | 無特殊要求 | CSI Snapshot 支援 | 無特殊要求 |
| 適用場景 | Storage 不支援 Snapshot | Storage 支援 Snapshot | 小型 DB |
| 網路負載 | 中 | 低（Storage 層操作） | 高 |

#### 建議

> 優先使用 Volume Snapshot Clone（如果 Storage 支援），Backup + Restore 作為備選。

#### 討論問題

- [ ] 目前 Storage Class 是否支援 CSI Snapshot？
- [ ] 最大 DB 實例的大小是多少？對應的 Snapshot/Restore 時間預估？
- [ ] Backup 工具（mariabackup/mysqldump）是否已在生產環境驗證？

#### 結論

> _（討論後填寫）_

---

### Decision 2：Blue/Green 期間的 Replication 拓撲

**問題**：Blue Primary 與 Green Cluster 之間的 Replication 用什麼拓撲？
**影響範圍**：影響 Green 環境的同步延遲和 Blue Primary 負載
**前置決策**：Decision 0 = Option A

#### 選項比較

```
Option A: 簡單鏈式（Simple Chain）
  Blue Primary ──GTID async──► Green Primary ──semi-sync──► Green R1, R2

Option B: 扇出式（Fan-out）
  Blue Primary ──GTID async──► Green Primary
  Blue Primary ──GTID async──► Green R1
  Blue Primary ──GTID async──► Green R2
```

| 維度 | A. Simple Chain | B. Fan-out |
|------|----------------|------------|
| 與 operator 相容性 | ✅ Green 作為獨立 CR 管理 | ❌ 與 operator 管理模式衝突 |
| Blue Primary 負載 | 低（只有一條 Replication） | 高（多條 Replication） |
| Green Replica 延遲 | 較高（經過 Green Primary 中轉） | 較低（直接從 Blue 同步） |
| 單點風險 | Green Primary 是單點 | 無 |

#### 建議

> Option A（Simple Chain）——Green Cluster 由自己的 MariaDB CR 管理，Green Primary 從 Blue Primary 複製。

#### 討論問題

- [ ] Green Replica 延遲增加（Chain 中轉）是否影響驗證測試的準確性？
- [ ] operator 是否確實不支援外部 Replication 管理？

#### 結論

> _（討論後填寫）_

---

### Decision 3：Switchover 機制

**問題**：DB Cluster 內部用什麼機制切換流量（Layer 2）？
**影響範圍**：Decision 6, 7 的設計都建立在此機制之上
**前置決策**：Decision 0（Option B 需額外處理 Layer 1）

#### 連線路徑分層

```
┌─ Layer 1：跨叢集（固定不變）───────────────────────────────────────────┐
│                                                                      │
│   App Cluster ──────► Ingress Gateway ──────► DB K8s Cluster         │
│                                                                      │
│   · Ingress Gateway 是跨叢集的固定入口，兩種方案都必經                     │
│   · Option A（同 Zone 切換）：Layer 1 不需要修改                         │
│   · Option B（跨 Zone 切換）：Layer 1 的 Gateway 路由目標需要更新          │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─ Layer 2：DB Cluster 內部（Switchover 的決策點）─────────────────────────┐
│                                                                      │
│   方案 A（K8s Service selector）：                                     │
│     Ingress GW → K8s Service ──selector──► MariaDB Pods              │
│     切換方式：更新 Service 的 selector 指向 Green Pods                   │
│                                                                      │
│   方案 B（ProxySQL）：                                                 │
│     Ingress GW → K8s Service → ProxySQL → MariaDB Pods               │
│     切換方式：更新 ProxySQL 的 server list / weight                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

> **關鍵觀念**：Ingress Gateway 與 ProxySQL **不是**二擇一的關係。Gateway 是 Layer 1 的固定入口，永遠在路徑上。真正的決策點在 Layer 2。

#### DB-level 共用步驟（與 Layer 2 方案無關）

| 順序 | 操作 | 說明 |
|------|------|------|
| 1 | Blue: `SET GLOBAL read_only=ON` | 阻擋新寫入，**寫入暫停開始** |
| 2 | 等待 GTID sync | 確認 Green 已追上 Blue 最後的 GTID |
| 3 | Green: `STOP SLAVE; RESET SLAVE ALL;` | 切斷 Blue → Green 的 Replication |
| 4 | Green: `read_only=OFF` | Green 準備接受寫入 |
| 5 | **切換流量** | ← Layer 2 方案決定如何執行 |
| 6 | 驗證端到端連線 | 確認 App 透過 Ingress Gateway → Green 正常讀寫，**寫入暫停結束** |

#### 選項比較

| 維度 | A. K8s Service selector | B. ProxySQL |
|------|------------------------|-------------|
| 切換動作 | `kubectl patch svc` 更新 selector | 更新 ProxySQL server list / weight |
| 切換粒度 | 全量切換（All-or-Nothing） | 可漸進式切換（調整 weight） |
| 額外元件 | 無——K8s 原生機制 | 需要部署和維護 ProxySQL |
| 讀寫分離 | 不支援（需額外 Service） | 原生支援（hostgroup） |
| 回滾方式 | 反向 patch selector 回 Blue | 反向更新 server list 回 Blue |
| 適合階段 | **現階段** | 未來——需要讀寫分離或漸進式切換時 |

#### Ingress Gateway 注意事項（兩種方案都適用）

| 注意事項 | 說明 |
|---------|------|
| DNS/Routing 快取 | Gateway 增加了額外的 DNS/Routing 層——需要了解其快取行為 |
| 連線殘留 | 如果 Gateway 快取 backend 連線，stale 連線必須被 drain |
| Health Check 設定 | Gateway 的 Health Check 應能快速偵測 backend 變更 |
| Connection Timeout | Gateway 上的 timeout 設定影響 App 多快看到新 backend |

#### 建議

> 方案 A（K8s Service selector）——簡單、無額外依賴，適合現階段。

#### 討論問題

- [ ] 目前 Ingress Gateway 的 connection drain 和 health check 設定是什麼？
- [ ] 是否有任何租戶需要讀寫分離？（影響是否需要 ProxySQL）
- [ ] Gateway 層的連線快取行為是否已被理解和驗證？

#### 結論

> _（討論後填寫）_

---

### Decision 4：Switchover 期間的 Zone-B 處理

**問題**：Switchover 期間和之後，Zone-B（DR）如何處理？
**影響範圍**：影響 DR 可用性和 Switchover 後的恢復流程
**前置決策**：Decision 0（行為完全取決於 D0 的選擇）

#### 選項比較

**如果 D0 = Option A（建立新 Green Cluster）：**

```
Before:   Zone-A-Blue-Primary ──async──► Zone-B-Primary
After:    Zone-A-Green-Primary ──async──► Zone-B-Primary  (CHANGE MASTER + GTID 重新指向)
```

Zone-B 在整個過程中維持 DR 角色。Switchover 後，只需把 Zone-B 重新指向 Green。

**如果 D0 = Option B（Zone-B 作為 Green）：**

```
Before:   Zone-A-Primary ──async──► Zone-B-Primary
During:   Zone-A-Primary            Zone-B 升級中（DR 缺口！）
After:    Zone-A (new DR) ◄──async── Zone-B-Primary（Apps 連到這裡）
```

Zone-A 變成新的 DR。Replication 方向反轉。

| 維度 | D0=A | D0=B |
|------|------|------|
| DR 中斷 | 無 | 有（升級期間） |
| Replication 方向 | 不變 | 反轉 |
| 額外工作 | 僅 `CHANGE MASTER` 重指向 | 需重建 Zone-A 作為 DR |

#### 建議

> 依 Decision 0 結果決定。無獨立選項。

#### 討論問題

- [ ] Zone-B 重新指向（Option A）或重建（Option B）的預估時間？
- [ ] Zone-B SLA：Switchover 期間 Replication 中斷可接受的最大間隔？

#### 結論

> _（討論後填寫）_

---

### Decision 5：交付格式

**問題**：Blue/Green 流程的自動化程度做到什麼層級？
**影響範圍**：決定 Phase 1 的實作工作量和長期維護成本
**前置決策**：無（獨立決策，可最後決定）

#### 選項比較

| 維度 | A. Runbook + Scripts | B. Argo/Tekton Pipeline | C. 擴展 operator | D. 自建 Controller |
|------|---------------------|------------------------|-----------------|-------------------|
| 工作量 | 低（週） | 中 | 中高 | 高（月） |
| 彈性 | 手動、容易出錯 | 半自動化、宣告式 | 可貢獻上游 | 完全自動化 |
| 適合階段 | 第一階段 | 第二階段 | 長期 | 長期 |
| 維護成本 | 低 | 中 | 受 operator roadmap 影響 | 高 |

#### 建議

> 先用 Runbook + Scripts 驗證流程，再逐步升級到 Pipeline 或 Controller。

#### 討論問題

- [ ] 團隊目前使用什麼工具做 DB 營運自動化？（Ansible？Shell Script？）
- [ ] 預計多久後需要從手動升級到半自動化？
- [ ] 是否有計畫貢獻 mariadb-operator 上游？

#### 結論

> _（討論後填寫）_

---

### Decision 6：Guardrails 檢查

**問題**：Switchover 執行前，必須通過哪些檢查項目？
**影響範圍**：決定 Switchover 腳本中的 pre-flight check 清單
**前置決策**：Decision 3（Switchover 機制確定後才能定義具體檢查項）

#### 選項比較

此決策不是選項二擇一，而是確定 **檢查清單的範圍**：

| 檢查項目 | 說明 | 必要/建議 |
|---------|------|----------|
| Replication Lag = 0 | Green 已追上 Blue 的 GTID | 必要 |
| Green Smoke Test 通過 | 基本 SELECT/INSERT 驗證 | 必要 |
| 所有 Green Replica 健康 | Semi-sync Replica 狀態正常 | 必要 |
| Blue 無長時間交易 | 避免 `read_only` 被 long-running tx 阻擋 | 建議 |
| Connection Pool 狀態 | 確認 App 端 connection pool 健康 | 建議 |
| Zone-B Replication 健康 | DR 在 Switchover 前是正常的 | 建議 |
| 磁碟空間充足 | Green 和 Blue 都有足夠空間 | 建議 |
| 無活躍 DDL | 避免 Schema 變更中途切換 | 必要 |

#### 建議

> 將「必要」項目做成自動化 pre-flight script，任何一項失敗即中止 Switchover。「建議」項目列入 Runbook 人工確認。

#### 討論問題

- [ ] 是否有其他業務層面的檢查需求？（如：不能在月結日執行）
- [ ] pre-flight check 失敗時，是自動中止還是允許人工覆蓋？
- [ ] 長時間交易（long-running tx）的定義門檻？（> 30s？> 60s？）

#### 結論

> _（討論後填寫）_

---

### Decision 7：回滾與 Timeout

**問題**：Switchover 失敗時怎麼辦？等多久算失敗？
**影響範圍**：決定 Switchover 腳本中的 timeout 和 rollback 邏輯
**前置決策**：Decision 3（回滾動作取決於 Switchover 機制）

#### 選項比較

**Timeout 策略：**

| 維度 | 保守（短 Timeout） | 寬鬆（長 Timeout） |
|------|-------------------|-------------------|
| GTID Sync Timeout | 30s | 120s |
| 端到端驗證 Timeout | 15s | 60s |
| 寫入暫停總時長上限 | 60s | 300s |
| 風險 | 可能誤判為失敗 | 寫入暫停過長影響業務 |

**Rollback 動作（按 Switchover 階段）：**

| 失敗發生在 | Rollback 動作 | 複雜度 |
|-----------|-------------|--------|
| GTID Sync 未完成 | Blue 解除 `read_only`，中止 Switchover | 低 |
| Service selector 已切換但驗證失敗 | 反向 patch selector 回 Blue，Green 設 `read_only` | 中 |
| 已運行一段時間後發現問題 | 反向 Switchover（Green→Blue），需處理 Green 期間的寫入 | 高 |

#### 建議

> 第一階段使用保守 Timeout，寧可中止也不要讓寫入暫停太久。Rollback 腳本必須在 Switchover 前就準備好。

#### 討論問題

- [ ] 租戶可容忍的最大寫入暫停時間？（直接決定 Timeout 上限）
- [ ] 是否需要「半自動 Rollback」（自動偵測失敗 + 人工確認回滾）？
- [ ] Switchover 後多久內算「驗證期」？（驗證期內保留 Blue 環境）

#### 結論

> _（討論後填寫）_

---

### Decision 8：排程任務處理

**問題**：Green 環境的 Event Scheduler 是否開啟？
**影響範圍**：避免 Blue 和 Green 同時執行排程任務導致資料不一致
**前置決策**：Decision 0（需知道 Green 環境的建立方式）

#### 選項比較

| 維度 | A. Green Event Scheduler 關閉 | B. Green Event Scheduler 開啟 |
|------|------------------------------|------------------------------|
| 資料一致性 | ✅ 安全——只有 Blue 執行排程任務 | ❌ 風險——Blue 和 Green 可能同時執行 |
| 設定方式 | Green CR 中設定 `event_scheduler=OFF` | 依賴 Replication filter 或 app-level 防護 |
| Switchover 後動作 | 手動開啟 Green 的 Event Scheduler | 無需額外動作 |
| 複雜度 | 低 | 高 |

#### 建議

> 關閉 Green 的 Event Scheduler，Switchover 完成後再手動開啟。

#### 討論問題

- [ ] 目前有哪些 DB 實例使用 Event Scheduler？排程任務的用途？
- [ ] 排程任務是否有冪等性？（即使重複執行也不會造成問題）

#### 結論

> _（討論後填寫）_

---

## 4. 執行流程

> 根據 Decision 0-8 的結果，對應的完整執行流程。

### 4.1 Option A 流程：建立新 Green Cluster

```
                        Phase 1                    Phase 2
                    ┌──CREATE GREEN──┐         ┌──VALIDATE──┐
                    │                │         │            │
  ┌────────┐    ┌───▼───┐    ┌──────▼──┐   ┌──▼──────┐   ┌▼─────────┐
  │Snapshot │───►│Create │───►│Restore  │──►│Setup    │──►│Run Smoke │
  │Blue PVC │    │Green  │    │Data to  │   │GTID     │   │Tests on  │
  │         │    │MariaDB│    │Green    │   │Repl.    │   │Green     │
  └────────┘    │CR     │    │Primary  │   │Blue→Grn │   │          │
                └───────┘    └─────────┘   └─────────┘   └──────────┘

                        Phase 3                    Phase 4
                    ┌──SWITCHOVER──┐           ┌──CLEANUP──┐
                    │              │           │           │
  ┌─────────┐   ┌──▼──────┐   ┌──▼──────┐   ┌▼─────────┐
  │Set Blue  │──►│Wait for │──►│Switch   │──►│Re-point  │
  │read_only │   │GTID Sync│   │K8s Svc  │   │Zone-B to    │
  │= ON      │   │         │   │Selector │   │Green     │
  └─────────┘   └─────────┘   │→ Green  │   └──────────┘
                               └─────────┘
```

#### Phase 1：建立 Green 環境

| 步驟 | 操作 | 備註 |
|------|------|------|
| 1 | Snapshot Blue Primary 的 PVC（或 Backup） | 優先使用 Volume Snapshot |
| 2 | 建立新的 MariaDB CR（Green），使用目標版本/設定 | 需新 CR 名稱 |
| 3 | 將資料還原到 Green Primary | 從 Snapshot 或 Backup 還原 |
| 4 | 設定 GTID Replication：Green Primary → 從 Blue Primary 複製 | `CHANGE MASTER TO MASTER_USE_GTID=slave_pos` |
| 5 | 等待 Green 追上（`Seconds_Behind_Master = 0`） | 時間取決於資料量和寫入速度 |
| 6 | Green 的 Semi-sync Replica 自動追上 | 由 operator 管理 |

#### Phase 2：驗證 Green 環境

| 步驟 | 操作 | 備註 |
|------|------|------|
| 7 | 對 Green 執行 Smoke Test（透過臨時 Service 或 port-forward） | **測試必須唯讀**——避免寫入汙染 Green |
| 8 | 驗證 Schema 相容性、Query Plan、App 連線能力 | 特別注意升級後的 SQL 行為變化 |
| 9 | 租戶可選擇性地對 Green 進行測試 | 提供臨時連線方式 |
| 10 | 確認 Storage 已預熱（如使用 Snapshot Clone） | 避免 Switchover 後 I/O 效能低落 |

#### Phase 3：Switchover

| 步驟 | 操作 | 備註 |
|------|------|------|
| 11 | 公告維護窗口 | 通知所有受影響的租戶 |
| 12 | **執行 Guardrails 檢查**（見 [Decision 6](#decision-6guardrails-檢查)） | 任何必要項失敗即中止 |
| 13 | 在 Blue Primary 終止長時間連線 | `KILL` 超過 threshold 的 session |
| 14 | 在 Blue Primary 設定 `SET GLOBAL read_only=ON` | **寫入暫停開始** |
| 15 | 等待 GTID 同步：Green 追上 Blue 最後的 GTID | 通常只需幾秒 |
| 16 | 在 Green 停止 Replication（`STOP SLAVE; RESET SLAVE ALL;`） | 切斷 Blue → Green 連結 |
| 17 | 確認 Green Primary `read_only=OFF` | 準備接受寫入 |
| 18 | **Layer 2 切換**：更新 K8s Service selector → 指向 Green Pods | Layer 1 不需修改 |
| 19 | 驗證 App 端到端連線 | **寫入暫停結束** |
| 20 | 重新將 Zone-B Replication 指向 Green Primary（`CHANGE MASTER`） | 恢復 DR 保護 |

#### Phase 4：清理

| 步驟 | 操作 | 備註 |
|------|------|------|
| 21 | Blue 設定 `read_only=ON` 保持唯讀 | 保留作為 Rollback 窗口，防止意外寫入 |
| 22 | 持續監控（保留 Blue 環境） | 建議至少觀察 24-48 小時 |
| 23 | 更新相關資源（Helm values、ArgoCD、監控 Dashboard、告警規則） | Post-Switchover 資源更新清單 |
| 24 | 驗證期過後 → 刪除 Blue 環境 | 釋放資源 |

---

### 4.2 Option B 流程：將 Zone-B 重新作為 Green

```
  ┌──────────────────────────────────────────────────────────┐
  │  Phase 1: 升級 Zone-B                                       │
  │                                                          │
  │  Zone-A (Blue) 1P+2R ──async──► Zone-B 升級中 (DR 缺口!)       │
  │  ▲ Apps                                                   │
  └──────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Phase 2: Replication 追趕                               │
  │                                                          │
  │  Zone-A (Blue) 1P+2R ──async──► Zone-B (Green) 1P+2R 追趕中   │
  │  ▲ Apps                                                   │
  └──────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Phase 3: Switchover                                     │
  │                                                          │
  │  Zone-A (old Blue)            Zone-B (Green) 1P+2R             │
  │                             ▲ Apps 透過 Ingress Gateway   │
  └──────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Phase 4: 重建 DR                                        │
  │                                                          │
  │  Zone-A (new DR) 1P+2R ◄──async── Zone-B (Primary) 1P+2R      │
  │                                  ▲ Apps                   │
  └──────────────────────────────────────────────────────────┘
```

#### Phase 1：升級 Zone-B

| 步驟 | 操作 | 備註 |
|------|------|------|
| 1 | 停止 Zone-A → Zone-B 的 Replication | **DR 缺口開始** |
| 2 | 就地升級 Zone-B 或重建 Zone-B（使用目標版本） | 取決於升級路徑 |
| 3 | 確認 Zone-B 升級成功，服務正常 | 驗證版本號、基本查詢 |

#### Phase 2：Replication 追趕 + 驗證

| 步驟 | 操作 | 備註 |
|------|------|------|
| 4 | 重新建立 Zone-A → Zone-B 的 GTID Replication | GTID 允許從中斷處繼續同步 |
| 5 | 等待 Zone-B 追上（`Seconds_Behind_Master = 0`） | 時間取決於升級期間的寫入量 |
| 6 | 對 Zone-B（Green）執行 Smoke Test | **測試必須唯讀** |
| 7 | 確認 Storage 已預熱 | 避免 Switchover 後效能問題 |

#### Phase 3：Switchover

| 步驟 | 操作 | 備註 |
|------|------|------|
| 8 | 公告維護窗口 | 通知所有受影響的租戶 |
| 9 | **執行 Guardrails 檢查**（見 [Decision 6](#decision-6guardrails-檢查)） | 任何必要項失敗即中止 |
| 10 | 在 Zone-A Blue Primary 終止長時間連線 | `KILL` 超過 threshold 的 session |
| 11 | 在 Zone-A Blue Primary 設定 `read_only=ON` | **寫入暫停開始** |
| 12 | 等待 GTID 同步 | 確認 Zone-B 已追上 |
| 13 | 在 Zone-B 停止 Replication（`STOP SLAVE`） | 切斷連結 |
| 14 | 確認 Zone-B Primary `read_only=OFF` | 準備接受寫入 |
| 15 | **Layer 1 + Layer 2 切換**：更新 Ingress Gateway 路由 + K8s Service | 與 Option A 不同——兩層都需修改 |
| 16 | 驗證 App 端到端連線 | **寫入暫停結束** |

#### Phase 4：重建 DR

| 步驟 | 操作 | 備註 |
|------|------|------|
| 17 | Zone-A 設定 `read_only=ON` 保持唯讀 | 防止意外寫入 |
| 18 | 重建 Zone-A 作為新的 DR（可能需全新 Restore） | Replication 方向反轉 |
| 19 | 建立 Zone-B → Zone-A 的 GTID Replication | Zone-A 現在是 DR |
| 20 | 等待 Zone-A 追上 | **DR 缺口結束** |
| 21 | 更新相關資源（Helm values、ArgoCD、監控、告警） | Post-Switchover 資源更新清單 |
| 22 | 刪除舊環境、更新 IaC 狀態 | 確保 GitOps 一致 |

---

## 5. 風險與緩解

### 技術風險

| 風險 | 影響 | 緩解策略 |
|------|------|---------|
| **mariadb-operator 不原生支援跨 CR Replication** | 無法單靠 operator CRD 建立 Blue→Green Replication | 需手動 `CHANGE MASTER` 或擴展 operator |
| **Storage 不支援 Volume Snapshot** | Green 建立需退回到較慢的 Backup/Restore | 事先確認 CSI Driver 能力 |
| **Semi-sync Replication 在 Switchover 期間中斷** | 短暫的資料不一致可能性 | `read_only` + GTID 同步步驟可防止此問題 |
| **Ingress Gateway 連線快取** | Gateway 在 Service selector 變更後仍持有指向舊 Blue Pods 的連線 | 確認 Gateway 的 connection drain、health check interval、idle timeout 設定 |
| **跨叢集 DNS/Routing 延遲** | App 感受到的 Switchover 有額外延遲 | 測試端到端 Switchover 延遲（含 Gateway 傳播時間） |
| **Connection Pool 殘留連線** | App 的 connection pool 持有指向 Blue 的連線，新請求仍送到 Blue | 確認 App 的 connection pool 有 `maxLifetime` 設定，或 Switchover 後主動觸發 pool refresh |
| **外部 Replication 消費者（CDC/ETL）** | CDC/ETL pipeline 可能仍指向 Blue 的 binlog 位置 | 盤點所有 binlog 消費者，Switchover 後更新指向 Green |

### 營運風險

| 風險 | 影響 | 緩解策略 |
|------|------|---------|
| **Zone-B Replication 缺口（Switchover 期間）** | DR 能力暫時降低 | 縮短 Switchover 窗口，密切監控 Zone-B Lag |
| **多租戶協調** | 共享實例上的所有 App 同時受影響 | 溝通維護窗口，驗證所有租戶的 retry 邏輯 |
| **資源壓力** | 同時運行 Blue + Green 加倍資源使用 | 確認 Cluster 有足夠餘量，或使用 Cluster Autoscaler |
| **Rollback 失敗** | Green 有問題但 Blue 已被清除，無法回退 | 保留 Blue 至少 24-48 小時的驗證期 |

### 與 AWS RDS Blue/Green 的風險對比

| 風險項目 | AWS RDS 如何處理 | 自建環境需要自行處理 |
|---------|-----------------|-----------------|
| Green 環境建立 | 自動從 Snapshot 建立，一條龍 | 需逐一補足——CR 生成、同 Zone Replication、驗證等（見[附錄 D](#附錄-daws-rds-blugreen-對照)） |
| Replication 設定 | 自動配置 binlog replication | 需手動或腳本 `CHANGE MASTER` |
| Switchover 原子性 | DNS CNAME flip（< 1 分鐘） | 需自行管理 K8s Service selector + Ingress Gateway |
| Rollback | 支援 Switchover 失敗自動回退 | 需自行設計 Rollback 流程（見 [Decision 7](#decision-7回滾與-timeout)） |
| 監控 | CloudWatch 整合 | 需自行設定 Prometheus + Grafana Dashboard |

---

## 6. 下一步行動

### Phase 0：實作前準備

```
Step 1: 盤點審計
  ├── 確認 Storage Class 是否支援 Snapshot
  ├── 確認 mariadb-operator 版本與可用功能
  ├── 確認所有實例的 GTID 設定
  └── 收集附錄 C 的所有資訊

Step 2: 團隊討論
  ├── 使用此文件逐項討論 Decision 0-8
  ├── 填寫每個 Decision 的「結論」欄位
  └── 定義 Rollback 標準和 Switchover SLA

Step 3: 選擇一個 Pilot 實例
  ├── 選擇低風險、小型的 DB 實例
  ├── 確認該實例的所有租戶已知悉
  └── 準備測試環境
```

### Phase 1-4：自動化演進路線

以下表格定義每個階段的目標狀態。每個維度可以獨立演進——不需要所有維度同時進入下一階段。

| 階段 | CR 生成 | PVC 資料 | Replication 管理 | 流程驅動 | 監控 |
|------|--------|---------|-----------------|---------|------|
| **Phase 1：手動驗證** | 手動複製 YAML | Backup + Restore | Shell Script | Runbook + Script | Script 輪詢 |
| **Phase 2：腳本化** | Shell Script | Volume Snapshot | K8s Job | Makefile/Taskfile | Prometheus Stack |
| **Phase 3：自動化** | Kustomize/Helm | Volume Snapshot | K8s Job | Argo Workflows | Prometheus + Grafana |
| **Phase 4：平台化** | Helm + ArgoCD | Volume Snapshot | 自訂 CRD | 自訂 Controller | 完整 Observability |

> **流程驅動**維度的詳細選項比較見 [Decision 5](#decision-5交付格式)。

#### Phase 1 重點：Pilot 手動驗證

Phase 1 使用 Phase 0 選定的 Pilot 實例，手動走一遍完整流程：

```
Pilot 驗證
  ├── 在 Pilot 實例上手動走一遍 Section 4 的 Phase 1-4 流程
  ├── 記錄每個階段的時間
  │    ├── Snapshot 時間
  │    ├── Replication 追趕時間
  │    ├── Switchover 持續時間（寫入暫停時長）
  │    └── Zone-B 重新指向時間
  ├── 記錄遇到的問題和解決方案
  └── 將原型轉為可重複執行的 Runbook
      ├── 建立 Pre-flight Checklist（見 Decision 6）
      └── 建立 Rollback Playbook（見 Decision 7）
```

---

## 附錄 A：Switchover 連線流程圖

```
  時間軸
  ──────────────────────────────────────────────────────────────►

  ┌────────────┐  ┌──────┐  ┌────────────────┐  ┌────────────┐
  │  正常運作   │  │ 暫停  │  │  切換 + 驗證    │  │  恢復運作   │
  │  Blue 服務  │  │ 寫入  │  │  Green 接手     │  │  Green 服務 │
  └────────────┘  └──────┘  └────────────────┘  └────────────┘
                  ◄─ 1~5s ─►

  App 視角：
  ─── 正常讀寫 ──── 寫入失敗/等待 ──── 正常讀寫（連到 Green）───►
                    (read OK)

  DB 視角：
  ─── Blue read_only=ON ─── GTID sync ─── Service selector 切換 ───►
```

---

## 附錄 B：SQL 指令參考

### 設定 GTID Replication（Green 從 Blue 複製）

```sql
-- 在 Green Primary 上執行
CHANGE MASTER TO
  MASTER_HOST='<blue-primary-host>',
  MASTER_PORT=3306,
  MASTER_USER='repl_user',
  MASTER_PASSWORD='<password>',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
```

### 檢查 Replication 狀態

```sql
-- 在 Green Primary 上執行
SHOW SLAVE STATUS\G

-- 關鍵欄位：
--   Seconds_Behind_Master: 0        ← 已追上
--   Gtid_Slave_Pos: <GTID>         ← 與 Blue 的 Gtid_Current_Pos 比對
--   Slave_IO_Running: Yes
--   Slave_SQL_Running: Yes
```

### Switchover 指令序列

```sql
-- Step 1: 在 Blue Primary 阻擋寫入
SET GLOBAL read_only=ON;

-- Step 2: 確認 Green 已追上（在 Green 上檢查）
SHOW SLAVE STATUS\G
-- 確認 Seconds_Behind_Master = 0

-- Step 3: 在 Green 停止 Replication
STOP SLAVE;
RESET SLAVE ALL;    -- 清除 Replication 設定

-- Step 4: 確認 Green 可寫入
SET GLOBAL read_only=OFF;

-- Step 5: 切換 K8s Service selector（kubectl 操作，非 SQL）

-- Step 6: 重新將 Zone-B 指向 Green
-- 在 Zone-B Primary 上執行
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='<green-primary-host>',
  MASTER_PORT=3306,
  MASTER_USER='repl_user',
  MASTER_PASSWORD='<password>',
  MASTER_USE_GTID=slave_pos;

START SLAVE;
```

### 確認 GTID 設定

```sql
-- 確認 GTID 嚴格模式已開啟
SHOW GLOBAL VARIABLES LIKE 'gtid_strict_mode';
-- 應為 ON

-- 查看目前 GTID Position
SELECT @@gtid_current_pos;
SELECT @@gtid_slave_pos;
```

---

## 附錄 C：資訊盤點清單

在實作 Blue/Green 之前，需要先收集以下資訊。此清單可作為**可填寫的模板**供相關團隊或應用程式擁有者填寫。

### A. 每個租戶 DB 的實例資訊

| 項目 | 為什麼重要 | 填寫欄位 |
|------|-----------|---------|
| MariaDB 版本 | 決定升級路徑、GTID 相容性 | `___________` |
| 實例規格（CPU/Memory requests & limits） | Green 環境需要相同或更大的資源 | `___________` |
| 儲存大小 & StorageClass | 影響 clone/restore 時間和 PVC 佈建 | `___________` |
| Replica 數量與拓撲 | Green 環境必須複製相同拓撲 | `___________` |
| 自訂 `my.cnf` / ConfigMap 覆寫 | 需要帶到 Green 環境或在 Green 中變更 | `___________` |
| 資料庫大小（GB） | 直接影響 Green 環境建立時間 | `___________` |
| 連線的應用程式及其 Namespace | 需要通知應用程式負責人，確認 retry/reconnect 邏輯 | `___________` |

### B. 平台層資訊

| 項目 | 為什麼重要 | 填寫欄位 |
|------|-----------|---------|
| mariadb-operator 版本 & CRD schema | 決定可用功能（如同 Namespace 是否支援多個 MariaDB CR） | `___________` |
| K8s Cluster 版本 | 影響可用 API（如 Gateway API vs Ingress） | `___________` |
| Storage Provisioner 能力 | 是否支援 Volume Cloning / Snapshot？ | `___________` |
| GTID 模式確認 | `gtid_strict_mode=ON`？ | `___________` |
| 備份策略 | 使用什麼工具？（mariabackup、mysqldump、operator 內建？） | `___________` |
| 監控堆疊 | Prometheus metrics？有哪些 Replication Lag 的 Dashboard？ | `___________` |
| CI/CD Pipeline | MariaDB CR 目前如何部署？（Helm？Kustomize？ArgoCD？） | `___________` |
| Ingress Gateway 類型 & 設定 | 哪種 Gateway？連線 timeout、health check、drain 設定 | `___________` |
| 跨叢集網路 | App Cluster → DB Cluster 的連線如何建立？ | `___________` |

### C. 應用程式端資訊（每個租戶）

| 項目 | 為什麼重要 | 填寫欄位 |
|------|-----------|---------|
| Connection String 格式 | 使用 Ingress Gateway endpoint？Service DNS？IP？ | `___________` |
| Connection Pool Library & 設定 | Pool size、max idle time、connection lifetime、validation query | `___________` |
| Retry/Reconnect 行為 | App 斷線後會重試嗎？多快？ | `___________` |
| Read/Write Split | App 是否從 Replica 讀取？如何做？ | `___________` |
| 可接受的停機窗口 | Switchover 仍需短暫暫停寫入——可容忍多久？ | `___________` |
| Migration 工具 | App Team 用什麼做 Schema 變更？ | `___________` |

### D. 營運限制

| 項目 | 為什麼重要 | 填寫欄位 |
|------|-----------|---------|
| Cluster 剩餘資源 | Green 環境會暫時加倍資源使用 | `___________` |
| 維護窗口 | Switchover 可以在什麼時間執行？ | `___________` |
| 變更管理流程 | 誰批准？Rollback 權限是什麼？ | `___________` |
| Zone-B SLA | Switchover 期間 Replication 中斷可接受的最大間隔 | `___________` |

---

## 附錄 D：AWS RDS Blue/Green 對照

### 核心流程對照

AWS RDS Blue/Green 的核心流程：

```
Snapshot Blue → Restore 為 Green → 自動建立 binlog Replication → 測試 → Switchover（endpoint rename）→ 舊 Blue 保留
```

這直接對應 **Decision 0 Option A**——建立全新 Green 環境。AWS 不會把 DR / Read Replica 拿來當 Green 用，因此 **Option B 沒有 AWS 對應物**。

### 自動化能力差距分析

以下比較 AWS 全自動化流程與自建環境（mariadb-operator + 腳本）的能力差距：

| 生命週期階段 | AWS 自動完成 | 自建環境典型能力 | 需要補足的缺口 |
|------------|-------------|----------------|--------------|
| **1. Snapshot/Backup** | 自動 Snapshot | 有 backup/restore 腳本 | 需通用化，支援同 Zone Blue→Green |
| **2. 建立 Green 實例** | 自動從 Snapshot Restore | 手動建立 MariaDB CR YAML | 需設計 Green CR 生成方式 |
| **3. Blue→Green Replication** | 自動配置 binlog replication | 有 `CHANGE MASTER TO` 經驗（通常僅跨 Zone） | 腳本需支援同 Zone Blue→Green Replication |
| **4. Green 環境驗證** | 提供臨時 endpoint | 通常沒有標準化流程 | 需設計臨時 Service + Smoke Test 流程 |
| **5. Switchover** | 自動 endpoint rename + connection drain | 已設計（Decision 3 的 Layer 2 切換） | 已有方案，需實作 |
| **6. Rollback** | 支援自動回退 | 通常沒有標準化流程 | 需設計 Rollback 腳本和判斷標準（見 Decision 7） |
| **7. 清理** | 手動刪除 | 刪除 CR/PVC 即可 | 無重大缺口 |

### 自建環境需要回答的問題清單

選擇 Option A 時，以下問題需要逐一解決：

**部署面（Deploy）**

| # | 問題 | 可選方案（由簡到繁） | 建議起步 |
|---|------|-------------------|---------|
| 1 | **Green CR 如何生成？** | 手動複製 YAML → Shell Script → Kustomize overlay → Helm chart | 手動複製 YAML 或 Shell Script |
| 2 | **PVC 資料怎麼來？** | Backup + Restore → Volume Snapshot Clone → 全新實例 + Replication 追趕 | 對應 Decision 1 |

> Green CR 可放同 Namespace——前提是 operator 支援同 Namespace 多個 MariaDB CR（mariadb-operator 社群版支援此功能）。

**控制面（Control）**

| # | 問題 | 可選方案（由簡到繁） | 建議起步 |
|---|------|-------------------|---------|
| 3 | **Blue→Green Replication 如何管理？** | Shell Script → K8s Job → Ansible Playbook → 自訂 CRD + Controller | Shell Script 搭配 Runbook |
| 4 | **整個 Blue/Green 流程用什麼驅動？** | Runbook + Shell Script → Makefile/Taskfile → Argo Workflows → 自訂 Controller | Runbook + Shell Script |
| 5 | **Green 環境如何監控？** | Script 輪詢 → Prometheus + mysqld_exporter → 完整 Grafana Dashboard | 已有 Prometheus 則直接使用 |

> 上述問題的建議演進路線見 [Section 6：下一步行動](#6-下一步行動)。
