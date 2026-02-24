# Self-Hosted DBaaS Blue/Green Deployment 設計文件

這篇筆記探討在 Kubernetes 上運行 MariaDB（透過 mariadb-operator）時，如何設計 Blue/Green Deployment 流程。目標是參考 AWS RDS Blue/Green Deployment 的概念，為自建多租戶 DBaaS 提供**零停機**的版本升級、設定變更與 Schema Migration 能力。

> **背景**：與 AWS RDS 不同，自建的 Kubernetes 叢集使用 [mariadb-operator/mariadb-operator](https://github.com/mariadb-operator/mariadb-operator)（社群版）來管理 MariaDB 實例。因此無法直接使用 AWS 提供的 Blue/Green API，需要自行設計對應的流程。

---

## 1. 目前架構

### 整體拓撲

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

### 架構要點

| 項目 | 說明 |
|------|------|
| **連線方式** | App Cluster → Ingress Gateway（跨叢集）→ DB Cluster 內的 K8s Service → MariaDB Pods |
| **Operator** | mariadb-operator/mariadb-operator（社群版） |
| **Zone 內複製** | Semi-synchronous Replication（半同步） |
| **跨 Zone 複製** | GTID-based Asynchronous Replication（非同步） |
| **多租戶** | 同一 DB Cluster 上有多個團隊的 DB 實例 |
| **關鍵細節** | DB 和 App 在**不同的 K8s Cluster** |

> **與 AWS RDS 的對照**：AWS 的 Blue/Green 在底層也是使用 binlog replication 搭配 GTID 來同步 Blue 與 Green 環境。自建環境的設計思路相同，只是需要自行管理這些元件。

---

## 2. 資訊盤點清單

在實作 Blue/Green 之前，需要先收集以下資訊。此清單可作為 **可填寫的模板** 供相關團隊或應用程式擁有者填寫。

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
| mariadb-operator 版本 & CRD schema | 決定可用功能（如同一 Namespace 是否支援多個 MariaDB CR） | `___________` |
| K8s Cluster 版本 | 影響可用 API（如 Gateway API vs Ingress） | `___________` |
| Storage Provisioner 能力 | 是否支援 Volume Cloning / Snapshot？（Green 環境快速建立的關鍵） | `___________` |
| GTID 模式確認 | `gtid_strict_mode=ON`？這是可靠 Replication 串接的前提 | `___________` |
| 備份策略 | 使用什麼工具？（mariabackup、mysqldump、operator 內建？） | `___________` |
| 監控堆疊 | Prometheus metrics？有哪些 Replication Lag 的 Dashboard？ | `___________` |
| CI/CD Pipeline | MariaDB CR 目前如何部署？（Helm？Kustomize？ArgoCD？） | `___________` |
| Ingress Gateway 類型 & 設定 | 哪種 Gateway？（Istio、Envoy、Nginx 等）連線 timeout、health check、drain 設定 | `___________` |
| 跨叢集網路 | App Cluster → DB Cluster 的連線如何建立？（VPN、Peering、Service Mesh？） | `___________` |

### C. 應用程式端資訊（每個租戶）

| 項目 | 為什麼重要 | 填寫欄位 |
|------|-----------|---------|
| Connection String 格式 | 使用 Ingress Gateway endpoint？Service DNS？IP？Switchover 如何影響？ | `___________` |
| Connection Pool Library & 設定 | Pool size、max idle time、connection lifetime、validation query | `___________` |
| Retry/Reconnect 行為 | App 斷線後會重試嗎？多快？ | `___________` |
| Read/Write Split | App 是否從 Replica 讀取？如何做？（獨立 Service？ProxySQL？） | `___________` |
| 可接受的停機窗口 | 即使「零停機」，Switchover 仍需短暫暫停寫入——可容忍多久？（1s？5s？30s？） | `___________` |
| Migration 工具 | App Team 用什麼做 Schema 變更？（Flyway？Liquibase？手動 SQL？） | `___________` |

### D. 營運限制

| 項目 | 為什麼重要 | 填寫欄位 |
|------|-----------|---------|
| Cluster 剩餘資源 | Green 環境會暫時加倍資源使用——是否有足夠空間？ | `___________` |
| 維護窗口 | Switchover 可以在什麼時間執行？有沒有封鎖期？ | `___________` |
| 變更管理流程 | 誰批准？Rollback 權限是什麼？ | `___________` |
| Zone-B SLA | Switchover 期間 Zone-B 的 Replication 會暫時中斷——可接受的最大間隔是多久？ | `___________` |

---

## 3. 關鍵設計決策

### Decision 0（最關鍵）：Blue/Green 策略——建立新叢集 vs 重用 Zone-B

這是最重要的架構決策，有兩種根本不同的路線。

#### Option A：建立新的 Green Cluster（經典 Blue/Green）

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

| 優點 | 缺點 |
|------|------|
| Zone-B DR 在整個過程中保持完整 | 暫時需要 3 倍資源（Blue + Green + Zone-B） |
| 乾淨的 Rollback——直接切回 Blue | 需要佈建新叢集、還原資料 |
| Zone-B 只在 Switchover 成功後才重新指向 | 設置較複雜 |

#### Option B：將 Zone-B (DR) 重新作為 Green

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

| 優點 | 缺點 |
|------|------|
| 不需要額外資源——重用現有 Zone-B | **升級 + Switchover 窗口期間 DR 消失** |
| Zone-B 已經有所有資料 | Zone-B 在不同 Zone——App 可能感受延遲變化 |
| 較簡單——不需要佈建新叢集 | Switchover 後 Primary 在 Zone-B（Zone 改變！） |
| GTID 讓 Replication 重新指向很乾淨 | 之後必須重建 Zone-A 作為新的 DR |
| | Rollback 較困難——需要把 Zone 切回來 |
| | 如果 Zone-B 升級失敗，DR 和 Blue/Green 都受影響 |

#### Option C（混合方案）：升級期間建立臨時最小 DR

```
[DURING Zone-B UPGRADE]
  Zone-A (Blue) 1P+2R ──async──► Temp-DR (1P+0R)    ◄── 臨時最小 DR
                               Zone-B 升級中...

[AFTER SWITCHOVER]
  Zone-A (new DR) 1P+2R ◄──async── Zone-B (Green/Primary) 1P+2R
  Temp-DR → 刪除
```

#### 選擇考量

| 問題 | Option A | Option B | Option C |
|------|----------|----------|----------|
| 升級期間 DR 是否存在？ | ✅ 完整 DR | ❌ DR 中斷 | ⚠️ 臨時最小 DR |
| 資源需求 | 3x（最多） | 1x（不增加） | ~1.3x |
| Rollback 難度 | 低（切回 Blue） | 高（需要切 Zone） | 中 |
| Primary Zone 是否改變？ | 否 | 是（移到 Zone-B） | 是（移到 Zone-B） |
| 實作複雜度 | 高 | 低 | 中 |

> **評估**：Option B 在營運上較簡單且省資源，但 **DR 缺口是真實風險**。如果 SLA 允許計畫性維護時暫時失去 DR（例如有維護窗口），Option B 很有吸引力。如果 DR 必須隨時存在，Option A 更安全。Option C 是折衷方案。

#### 與 AWS RDS Blue/Green 的對照

**結論：Option A 是 AWS-like 的做法；Option B 沒有 AWS 對應物。**

AWS RDS Blue/Green 的核心流程是：

```
Snapshot Blue → Restore 為 Green → 自動建立 binlog Replication → 測試 → Switchover（endpoint rename）→ 舊 Blue 保留
```

這直接對應 **Option A**——建立全新 Green 環境、Blue → Green Replication 同步、DR 全程保持完整、Switchover 後舊 Blue 保留作為 Rollback、Primary Zone 不改變。AWS 不會把 DR / Read Replica 拿來當 Green 用，因此 **Option B 沒有 AWS 對應物**。

##### AWS 自動化 vs 自建環境的差距

以下比較 AWS 全自動化流程與典型自建環境（使用 mariadb-operator 管理 semi-sync cluster，跨 Zone async replication 使用腳本）的能力差距：

| 生命週期階段 | AWS 自動完成 | 自建環境典型能力 | 需要補足的缺口 |
|------------|-------------|----------------|--------------|
| **1. Snapshot/Backup** | 自動 Snapshot | 有 backup/restore 腳本，但通常針對特定場景 | 需通用化，支援同 Zone 的 Blue→Green 備份 |
| **2. 建立 Green 實例** | 自動從 Snapshot Restore 為新 DB instance | 手動建立 MariaDB CR YAML | 需設計 Green CR 生成方式（見下方問題清單） |
| **3. Blue→Green Replication** | 自動配置 binlog replication | 有 `CHANGE MASTER TO` 經驗，但通常僅限跨 Zone | 腳本需支援同 Zone 的 Blue→Green Replication |
| **4. Green 環境驗證** | 提供臨時 endpoint 供測試 | 通常沒有標準化流程 | 需設計臨時 Service + Smoke Test 流程 |
| **5. Switchover** | 自動 endpoint rename + connection drain | 已設計（Decision 3 的 Layer 2 切換） | 已有方案，需實作 |
| **6. Rollback** | 自動支援 Switchover 失敗回退 | 通常沒有標準化流程 | 需設計 Rollback 腳本和判斷標準 |
| **7. 清理（刪除舊 Blue）** | 手動刪除（使用者決定） | 刪除 CR/PVC 即可 | 無重大缺口 |

##### 自建環境需要回答的問題清單

選擇 Option A 時，自建環境需要回答以下問題。每個問題都有多種方案可選，建議分階段演進：

**部署面（Deploy）**

| # | 問題 | 可選方案（由簡到繁） | 建議起步 |
|---|------|-------------------|---------|
| 1 | **Green CR 如何生成？** | 手動複製 YAML → Shell Script 修改欄位 → Kustomize overlay → Helm chart | 手動複製 YAML 或 Shell Script |
| 2 | **PVC 資料怎麼來？** | Backup + Restore → Volume Snapshot Clone → 全新實例 + Replication 追趕 | 對應 Decision 1 |

> Green CR 可放同 Namespace——前提是 operator 支援同 Namespace 多個 MariaDB CR（mariadb-operator 社群版支援此功能）。

**控制面（Control）**

| # | 問題 | 可選方案（由簡到繁） | 建議起步 |
|---|------|-------------------|---------|
| 3 | **Blue→Green Replication 如何管理？** | Shell Script（參數化 `CHANGE MASTER TO`）→ K8s Job → Ansible Playbook → 自訂 CRD + Controller | Shell Script 搭配 Runbook |
| 4 | **整個 Blue/Green 流程用什麼驅動？** | Runbook + Shell Script → Makefile/Taskfile → Argo Workflows / Tekton → 自訂 K8s Controller | Runbook + Shell Script |
| 5 | **Green 環境如何監控？** | Script 輪詢（`SHOW SLAVE STATUS`）→ Prometheus + mysqld_exporter → 完整 Grafana Dashboard | 已有 Prometheus 則直接使用，否則 Script 輪詢 |

> 上述問題的建議演進路線見 [Section 6：建議的下一步行動](#6-建議的下一步行動)。

---

### Decision 1：Green 環境初始化策略

*（主要適用於 Option A——Option B 可跳過此步驟，因為 Zone-B 已有資料）*

| 選項 | 優點 | 缺點 | 適用場景 |
|------|------|------|---------|
| **A. Backup + Restore** | 簡單，流程成熟 | 大型 DB 很慢（100GB+ 需數小時） | Storage 不支援 Snapshot |
| **B. Volume Snapshot Clone** | 快速（分鐘級），Storage 層操作 | 需要 CSI Snapshot 支援，相同 StorageClass | Storage 支援 Snapshot（**推薦**） |
| **C. 全新實例 + GTID Replication 追趕** | 設置乾淨 | 大型資料集非常慢，網路負載重 | 小型 DB |

> **建議**：優先使用 Volume Snapshot Clone（如果 Storage 支援），Backup + Restore 作為備選。

---

### Decision 2：Blue/Green 期間的 Replication 拓撲

```
Option A: 簡單鏈式（Simple Chain）
  Blue Primary ──GTID async──► Green Primary ──semi-sync──► Green R1, R2

Option B: 扇出式（Fan-out）
  Blue Primary ──GTID async──► Green Primary
  Blue Primary ──GTID async──► Green R1
  Blue Primary ──GTID async──► Green R2
```

| 選項 | 優點 | 缺點 |
|------|------|------|
| **Simple Chain** | 符合 mariadb-operator 管理 Replication 的方式，Green 作為獨立 CR 管理 | Green Primary 是單點 |
| **Fan-out** | Green Replica 直接從 Blue 同步，延遲更低 | 增加 Blue Primary 負載，且與 operator 管理模式衝突 |

> **建議**：Option A（Simple Chain）——Green Cluster 由自己的 MariaDB CR 管理，Green Primary 從 Blue Primary 複製。

---

### Decision 3：Switchover 機制

App 與 DB 在不同的 K8s Cluster，連線路徑經過多個層次。理解這些分層是設計 Switchover 的前提。

#### A. 連線路徑分層

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

> **關鍵觀念**：Ingress Gateway 與 ProxySQL **不是**二擇一的關係。Gateway 是 Layer 1 的固定入口，永遠在路徑上。真正的決策點在 Layer 2——DB Cluster 內部用什麼機制切換流量。

#### B. Switchover 共用步驟（DB-level，與 Layer 2 流量切換方式無關）

無論 Layer 2 使用哪種方案，以下 DB 層操作都相同：

| 順序 | 操作 | 說明 |
|------|------|------|
| 1 | Blue: `SET GLOBAL read_only=ON` | 阻擋新寫入，**寫入暫停開始** |
| 2 | 等待 GTID sync | 確認 Green 已追上 Blue 最後的 GTID |
| 3 | Green: `STOP SLAVE; RESET SLAVE ALL;` | 切斷 Blue → Green 的 Replication |
| 4 | Green: `read_only=OFF` | Green 準備接受寫入 |
| 5 | **切換流量** | ← Layer 2 方案決定如何執行，見下方 Decision 表格 |
| 6 | 驗證端到端連線 | 確認 App 透過 Ingress Gateway → Green 正常讀寫，**寫入暫停結束** |

> **重要**：短暫的寫入暫停是不可避免的（通常 1-5 秒）。AWS RDS Blue/Green 也是這樣做的。

#### C. Decision 表格：Layer 2 流量切換方式

| 比較維度 | 方案 A：K8s Service selector | 方案 B：ProxySQL |
|---------|---------------------------|-----------------|
| **切換動作** | `kubectl patch svc` 更新 selector | 更新 ProxySQL server list / 調整 weight |
| **切換粒度** | 全量切換（All-or-Nothing） | 可漸進式切換（調整 weight 逐步移轉流量） |
| **額外元件** | 無——使用 K8s 原生機制 | 需要部署和維護 ProxySQL |
| **讀寫分離** | 不支援（需額外 Service） | 原生支援（ProxySQL hostgroup） |
| **切換速度** | 快（K8s Service 更新幾乎即時） | 快（ProxySQL runtime 動態更新） |
| **回滾方式** | 反向 patch selector 回 Blue Pods | 反向更新 server list 回 Blue |
| **適合階段** | **現階段（推薦）**——簡單、無額外依賴 | 未來——當需要讀寫分離或漸進式切換時引入 |

#### D. Ingress Gateway 注意事項（兩種 Layer 2 方案都適用）

無論 Layer 2 使用 K8s Service selector 或 ProxySQL，流量都經過 Ingress Gateway，因此以下注意事項**同時適用於兩種方案**：

| 注意事項 | 說明 |
|---------|------|
| DNS/Routing 快取 | Ingress Gateway 增加了額外的 DNS/Routing 層——需要了解其快取行為 |
| 連線殘留 | 如果 Gateway 快取 backend 連線，這些 stale 連線必須被 drain |
| Health Check 設定 | Gateway 的 Health Check 應該能快速偵測到 backend 變更 |
| Connection Timeout | Gateway 上的 timeout 設定會影響 App 多快看到新的 backend |
| 端到端延遲 | 跨叢集的 DNS/Routing 延遲意味著 App 感受到的 Switchover 可能有額外延遲 |

---

### Decision 4：Switchover 期間的 Zone-B 處理

此決策高度依賴 Decision 0 的選擇：

#### 如果選 Option A（建立新 Green Cluster）

```
Before:   Zone-A-Blue-Primary ──async──► Zone-B-Primary
After:    Zone-A-Green-Primary ──async──► Zone-B-Primary  (使用 CHANGE MASTER + GTID 重新指向)
```

Zone-B 在整個過程中維持 DR 角色。Switchover 後，只需把 Zone-B 重新指向 Green。

#### 如果選 Option B（Zone-B 作為 Green）

```
Before:   Zone-A-Primary ──async──► Zone-B-Primary
During:   Zone-A-Primary            Zone-B 升級中（DR 缺口！）
After:    Zone-A (new DR) ◄──async── Zone-B-Primary（Apps 連到這裡）
```

Zone-A 變成新的 DR。Replication 方向反轉。

---

### Decision 5：交付格式

| 選項 | 工作量 | 彈性 | 適合階段 |
|------|--------|------|---------|
| **Runbook + Scripts** | 低（週） | 手動、容易出錯 | **第一階段（推薦）** |
| **Argo Workflows / Tekton Pipeline** | 中 | 半自動化、宣告式 | 第二階段 |
| **擴展 mariadb-operator** | 中高 | 可貢獻上游，但受 operator roadmap 影響 | 長期 |
| **自建 K8s Controller/CRD** | 高（月） | 完全自動化、自助式 | 長期 |

> **建議**：先用 Runbook + Scripts 驗證流程，再逐步升級到 Pipeline 或 Controller。

> **補充**：此 Decision 涵蓋 [Decision 0 自動化缺口分析](#與-aws-rds-blugreen-的對照)中「流程驅動」的面向。完整的多維度自動化演進（包含 CR 生成、Replication 管理、監控等）見 [Section 6](#6-建議的下一步行動)。

---

## 4. Blue/Green 完整流程

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
| 1 | Snapshot Blue Primary 的 PVC（或進行 Backup） | 優先使用 Volume Snapshot |
| 2 | 建立新的 MariaDB CR（Green），使用目標版本/設定 | 需要新的 CR 名稱，可能需要新的 Namespace |
| 3 | 將資料還原到 Green Primary | 從 Snapshot 或 Backup 還原 |
| 4 | 設定 GTID Replication：Green Primary → 從 Blue Primary 複製 | `CHANGE MASTER TO MASTER_USE_GTID=slave_pos` |
| 5 | 等待 Green 追上（`Seconds_Behind_Master = 0`） | 時間取決於資料量和寫入速度 |
| 6 | Green 的 Semi-sync Replica 自動追上 | 由 operator 管理 |

#### Phase 2：驗證 Green 環境

| 步驟 | 操作 | 備註 |
|------|------|------|
| 7 | 對 Green 執行 Smoke Test（透過臨時 Service 或 port-forward） | 驗證基本連線和查詢 |
| 8 | 驗證 Schema 相容性、Query Plan、App 連線能力 | 特別注意升級後的 SQL 行為變化 |
| 9 | 租戶可選擇性地對 Green 進行測試 | 提供臨時連線方式 |

#### Phase 3：Switchover

| 步驟 | 操作 | 備註 |
|------|------|------|
| 10 | 公告維護窗口（即使是「零停機」） | 通知所有受影響的租戶 |
| 11 | 在 Blue Primary 設定 `SET GLOBAL read_only=ON` 阻擋新寫入 | **寫入暫停開始** |
| 12 | 等待 GTID 同步：Green 追上 Blue 最後的 GTID | 通常只需幾秒 |
| 13 | 在 Green 停止 Replication（`STOP SLAVE`） | 切斷 Blue → Green 的連結 |
| 14 | 確認 Green Primary `read_only=OFF` | 準備接受寫入 |
| 15 | **Layer 2 切換**：更新 DB Cluster 中的 K8s Service selector → 指向 Green Pods | Layer 1（Ingress Gateway）不需要修改——Gateway 路由到相同 Service 名稱，只有 Layer 2（Service selector）變動 |
| 16 | 驗證 App 透過 Ingress Gateway 的端到端連線 | **寫入暫停結束** |
| 17 | 重新將 Zone-B 的 Replication 指向 Green Primary（`CHANGE MASTER`） | 恢復 DR 保護 |

#### Phase 4：清理

| 步驟 | 操作 | 備註 |
|------|------|------|
| 18 | 持續監控（保留 Blue 環境作為 Rollback 窗口） | 建議至少觀察 24-48 小時 |
| 19 | 驗證期過後 → 刪除 Blue 環境 | 釋放資源 |
| 20 | 更新 IaC 狀態（Helm values、ArgoCD 等） | 確保 GitOps 一致 |

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
| 2 | 就地升級 Zone-B 或重建 Zone-B（使用目標版本） | 取決於升級路徑是否支援就地升級 |
| 3 | 確認 Zone-B 升級成功，服務正常 | 驗證版本號、基本查詢 |

#### Phase 2：Replication 追趕

| 步驟 | 操作 | 備註 |
|------|------|------|
| 4 | 重新建立 Zone-A → Zone-B 的 GTID Replication | GTID 允許從中斷處繼續同步 |
| 5 | 等待 Zone-B 追上（`Seconds_Behind_Master = 0`） | 時間取決於升級期間的寫入量 |
| 6 | 對 Zone-B（Green）執行 Smoke Test | 透過臨時 Service 或 port-forward |

#### Phase 3：Switchover

| 步驟 | 操作 | 備註 |
|------|------|------|
| 7 | 公告維護窗口 | 通知所有受影響的租戶 |
| 8 | 在 Zone-A Blue Primary 設定 `read_only=ON` | **寫入暫停開始** |
| 9 | 等待 GTID 同步 | 確認 Zone-B 已追上 |
| 10 | 在 Zone-B 停止 Replication（`STOP SLAVE`） | 切斷 Zone-A → Zone-B 的連結 |
| 11 | 確認 Zone-B Primary `read_only=OFF` | 準備接受寫入 |
| 12 | **Layer 1 + Layer 2 都需要切換**：更新 Ingress Gateway 路由目標 → Zone-B（Layer 1），更新 K8s Service → 指向 Zone-B Green Pods（Layer 2） | 與 Option A 不同——Option A 同 Zone 切換只需變動 Layer 2；Option B 因為 Primary 跨 Zone 搬遷，**兩層都需要修改** |
| 13 | 驗證 App 端到端連線 | **寫入暫停結束** |

#### Phase 4：重建 DR

| 步驟 | 操作 | 備註 |
|------|------|------|
| 14 | 重建 Zone-A 作為新的 DR（可能需要全新 Restore） | Replication 方向反轉 |
| 15 | 建立 Zone-B → Zone-A 的 GTID Replication | Zone-A 現在是 DR |
| 16 | 等待 Zone-A 追上 | **DR 缺口結束** |
| 17 | 刪除舊的 Zone-A 環境、更新 IaC 狀態 | 確保 GitOps 一致 |

---

## 5. 風險與緩解策略

### 技術風險

| 風險 | 影響 | 緩解策略 |
|------|------|---------|
| **mariadb-operator 不原生支援跨 CR Replication** | 無法單靠 operator CRD 建立 Blue→Green Replication | 需要手動執行 `CHANGE MASTER` 或擴展 operator |
| **Storage 不支援 Volume Snapshot** | Green 建立需退回到較慢的 Backup/Restore | 事先確認 CSI Driver 能力 |
| **Semi-sync Replication 在 Switchover 期間中斷** | 短暫的資料不一致可能性 | `read_only` + GTID 同步步驟可防止此問題 |
| **Ingress Gateway 連線快取** | Gateway 可能在 Service selector 變更後仍持有指向舊 Blue Pods 的持久連線 | 確認 Gateway 的 connection drain 行為、health check interval、idle timeout 設定 |
| **跨叢集 DNS/Routing 延遲** | App 在不同叢集中透過 Gateway 感受到的 Switchover 有額外延遲 | 測試端到端 Switchover 延遲（包含 Gateway 傳播時間） |

### 營運風險

| 風險 | 影響 | 緩解策略 |
|------|------|---------|
| **Zone-B Replication 缺口（Switchover 期間）** | DR 能力暫時降低 | 縮短 Switchover 窗口，密切監控 Zone-B Lag |
| **多租戶協調** | 共享實例上的所有 App 同時受影響 | 溝通維護窗口，驗證所有租戶的 retry 邏輯 |
| **資源壓力** | 同時運行 Blue + Green 會加倍資源使用 | 確認 Cluster 有足夠餘量，或使用 Cluster Autoscaler |
| **Rollback 失敗** | 如果 Green 有問題但 Blue 已被清除，無法回退 | 保留 Blue 至少 24-48 小時的驗證期 |

### 與 AWS RDS Blue/Green 的風險對比

| 風險項目 | AWS RDS 如何處理 | 自建環境需要自行處理 |
|---------|-----------------|-----------------|
| Green 環境建立 | AWS 自動從 Snapshot 建立，包含 CR 生成、資料還原、Replication 設定一條龍 | 有 operator 管理 cluster、有跨 Zone 腳本，但缺少整合的 Green 環境建立流程——CR 生成、同 Zone Replication、驗證等步驟需逐一補足（見 Decision 0 差距分析） |
| Replication 設定 | AWS 自動配置 binlog replication | 需手動或腳本執行 `CHANGE MASTER` |
| Switchover 原子性 | AWS 使用 DNS CNAME flip（通常 < 1 分鐘） | 需自行管理 K8s Service selector + Ingress Gateway |
| Rollback | AWS 支援 Switchover 失敗自動回退 | 需自行設計 Rollback 流程 |
| 監控 | CloudWatch 整合 | 需自行設定 Prometheus + Grafana Dashboard |

---

## 6. 建議的下一步行動

本節整合 [Decision 0 的自動化缺口分析](#自建環境需要回答的問題清單)與實作步驟，分為兩大部分：Phase 0（實作前準備）和 Phase 1-4（自動化演進路線）。

### Phase 0：實作前準備

在進入自動化演進之前，先完成以下準備工作：

```
Step 1: 盤點審計
  ├── 確認 Storage Class 是否支援 Snapshot
  ├── 確認 mariadb-operator 版本與可用功能
  ├── 確認所有實例的 GTID 設定
  └── 收集 Section 2 的所有資訊

Step 2: 團隊討論
  ├── 使用此文件對齊 Decision 0（New Green vs Zone-B-as-Green）
  ├── 確定其他關鍵決策（Decision 1-5）
  └── 定義 Rollback 標準和 Switchover SLA

Step 3: 選擇一個 Pilot 實例
  ├── 選擇低風險、小型的 DB 實例
  ├── 確認該實例的所有租戶已知悉
  └── 準備測試環境
```

### Phase 1-4：自動化演進路線

以下表格對應 [Decision 0 問題清單](#自建環境需要回答的問題清單)的 5 個維度，定義每個階段的目標狀態。每個維度可以獨立演進——不需要所有維度同時進入下一階段。

| 階段 | CR 生成（問題 1） | PVC 資料（問題 2） | Replication 管理（問題 3） | 流程驅動（問題 4） | 監控（問題 5） |
|------|-----------------|------------------|------------------------|-----------------|-------------|
| **Phase 1：手動驗證** | 手動複製 YAML | Backup + Restore | Shell Script | Runbook + Script | Script 輪詢 |
| **Phase 2：腳本化** | Shell Script | Volume Snapshot（如支援） | K8s Job | Makefile/Taskfile | Prometheus Stack |
| **Phase 3：自動化** | Kustomize 或 Helm | Volume Snapshot | K8s Job | Argo Workflows | Prometheus + Grafana |
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
      ├── 建立 Pre-flight Checklist
      └── 建立 Rollback Playbook
```

> 完成 Phase 1 後，依據實際需求選擇各維度的下一步演進方向。

---

## 附錄 A：Switchover 期間的連線流程圖

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

## 附錄 B：關鍵 SQL 指令參考

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
