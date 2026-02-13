# AWS RDS Replication 學習筆記

這篇筆記整理了 AWS RDS 與 Aurora 所有複製機制的完整內容，涵蓋 Read Replica、Multi-AZ、Aurora Replication 與 Global Database。

---

## 1. 簡介與概觀

### Replication 的三大目的

| 目的 | 說明 |
|------|------|
| **讀取擴展 (Read Scaling)** | 將讀取流量分散到多個 Replica，降低 Primary 負載 |
| **高可用性 (High Availability)** | 自動 Failover 到 Standby，減少停機時間 |
| **災難復原 (Disaster Recovery)** | 跨 AZ 或跨 Region 備援，確保資料安全 |

### RDS 複製類型總覽

| 複製類型 | 主要用途 | 複製方式 | Replica 可讀 | 支援引擎 |
|---------|---------|---------|-------------|---------|
| **Read Replica（同區域）** | 讀取擴展 | 非同步 | ✅ | MySQL, MariaDB, PostgreSQL, Oracle, SQL Server |
| **Cross-Region Read Replica** | DR / 降低跨區延遲 | 非同步 | ✅ | MySQL, MariaDB, PostgreSQL, Oracle |
| **Multi-AZ DB Instance** | 高可用性 | 同步 | ❌ | MySQL, MariaDB, PostgreSQL, Oracle, SQL Server |
| **Multi-AZ DB Cluster** | 高可用性 + 讀取 | 半同步 | ✅（2 Readers）| MySQL 8.0.28+, PostgreSQL 13.4+ |
| **Aurora Replica** | 讀取擴展 + HA | 共享儲存 | ✅ | Aurora MySQL, Aurora PostgreSQL |
| **Aurora Global Database** | 跨區 DR + 讀取 | Storage-level | ✅ | Aurora MySQL, Aurora PostgreSQL |

> **我的理解**：這六種機制不是互斥的，實際架構中常常組合使用。例如 Multi-AZ（HA）+ Read Replica（讀取擴展）是非常常見的搭配。

> **來源**: [Working with DB instance read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)

---

## 2. Read Replica（同區域）

### 非同步複製機制

Read Replica 使用各引擎原生的非同步複製技術：

| 引擎 | 複製技術 | 說明 |
|------|---------|------|
| **MySQL** | Binary Log (binlog) Replication | 透過 binlog 複製所有變更，需要 `binlog_format = ROW` |
| **MariaDB** | Binary Log (binlog) Replication | 同 MySQL，支援 GTID |
| **PostgreSQL** | WAL (Write-Ahead Log) Streaming | 透過 WAL 串流複製，物理複製 |
| **Oracle** | Active Data Guard | 使用 Oracle 原生的 Data Guard 技術 |
| **SQL Server** | Not supported for same-region | 只支援 Multi-AZ，不支援同區域 Read Replica |

### 支援引擎與最大 Replica 數

| 引擎 | 最大 Read Replica 數 | Replica Chain | 自動備份需求 |
|------|---------------------|---------------|-------------|
| **MySQL** | 15 | ✅ 支援（最多 4 層）| Backup Retention > 0 |
| **MariaDB** | 15 | ✅ 支援（最多 4 層）| Backup Retention > 0 |
| **PostgreSQL** | 15 | ✅ 支援（最多 3 層）| Backup Retention > 0 |
| **Oracle** | 5 | ❌ 不支援 | Backup Retention > 0 |
| **SQL Server** | 5 | ❌ 不支援 | Backup Retention > 0 |

> **我的理解**：Replica Chain 就是「Replica 的 Replica」。Source → RR1 → RR2 → RR3 這樣串接。好處是分散 Source 的複製負載，壞處是 Lag 會累加。

### 架構圖

**基本架構：Source → 多個 Read Replica**

```
                    ┌─────────────────┐
              ┌────►│  Read Replica 1 │  (讀取流量)
              │     └─────────────────┘
┌─────────────┤     ┌─────────────────┐
│   Source     ├────►│  Read Replica 2 │  (讀取流量)
│  (Primary)  │     └─────────────────┘
│  讀寫        │     ┌─────────────────┐
└─────────────┤────►│  Read Replica 3 │  (讀取流量)
              │     └─────────────────┘
              │         ...
              │     ┌─────────────────┐
              └────►│  Read Replica N │  (最多 15 個)
                    └─────────────────┘
```

**Replica Chain 架構（MySQL/MariaDB）**

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Source   │────►│   RR 1   │────►│   RR 2   │────►│   RR 3   │
│ (Primary) │     │ (Level 1)│     │ (Level 2)│     │ (Level 3)│
└──────────┘     └──────────┘     └──────────┘     └──────────┘

每一層的 Replica Lag 會累加！
Source → RR1: 1秒 → RR2: 2秒 → RR3: 3秒
```

### Storage 與 Instance Class 差異

Read Replica 不需要與 Source 完全相同：

| 設定 | 可以不同？ | 說明 |
|------|----------|------|
| **Instance Class** | ✅ | Replica 可以用不同規格（但不建議比 Source 小）|
| **Storage Type** | ✅ | 可以用不同的 Storage 類型（gp2, gp3, io1 等）|
| **Storage Size** | ✅ | 可以不同，但必須 ≥ Source 的大小 |
| **Multi-AZ** | ✅ | Replica 本身也可以啟用 Multi-AZ |
| **Parameter Group** | ✅ | 可以使用不同的參數群組 |
| **Region** | ✅ | 可以在不同 Region（Cross-Region RR）|

### Replica Lag 監控

#### CloudWatch Metrics

| Metric | 引擎 | 說明 |
|--------|------|------|
| `ReplicaLag` | 全部 | Replica 落後 Source 的秒數 |
| `OldestReplicationSlotLag` | PostgreSQL | 最舊 Replication Slot 的 Lag（bytes）|
| `TransactionLogsDiskUsage` | PostgreSQL | WAL 使用的磁碟空間 |

#### SQL 查詢

**MySQL / MariaDB：**
```sql
SHOW REPLICA STATUS\G
-- 觀察 Seconds_Behind_Source 欄位
```

**PostgreSQL：**
```sql
SELECT
    now() - pg_last_xact_replay_timestamp() AS replica_lag;
```

### Promotion 流程

Read Replica 可以被 Promote 為獨立的 DB Instance：

1. Promotion 後，Replica 變成獨立的讀寫 DB Instance
2. 與原 Source 的 Replication 關係會被切斷
3. 原本的 Endpoint 不變
4. Promotion 過程中會有短暫的重啟

```bash
# Promote Read Replica
aws rds promote-read-replica \
    --db-instance-identifier my-read-replica

# 確認狀態
aws rds describe-db-instances \
    --db-instance-identifier my-read-replica \
    --query 'DBInstances[0].DBInstanceStatus'
```

### 建立 Read Replica CLI

```bash
# 建立同區域 Read Replica
aws rds create-db-instance-read-replica \
    --db-instance-identifier my-read-replica \
    --source-db-instance-identifier my-source-db \
    --db-instance-class db.r6g.xlarge \
    --allocated-storage 200 \
    --max-allocated-storage 1000
```

### 限制

| 限制 | 說明 |
|------|------|
| **不支援 Circular Replication** | A → B → A 的循環複製不支援 |
| **Source 刪除行為** | 刪除 Source 時，所有 Read Replica 會被 Promote 為獨立 Instance |
| **跨帳戶** | 不支援跨 AWS 帳戶建立 Read Replica |
| **VPC 限制** | 建議 Source 與 Replica 在同一 VPC |
| **加密** | 加密的 Source 只能建立加密的 Replica |
| **自動備份** | Source 必須啟用自動備份（Backup Retention > 0）|

> **來源**: [Working with DB instance read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)

---

## 3. Cross-Region Read Replica

### 三大用途

| 用途 | 說明 |
|------|------|
| **跨區域災難復原 (DR)** | 當主要 Region 故障時，可以 Promote 跨區域 Replica |
| **降低讀取延遲** | 讓各地區的使用者就近讀取，減少網路延遲 |
| **Region 遷移** | 先建立跨區域 Replica，再 Promote，達成資料庫遷移 |

### 建立流程

建立 Cross-Region Read Replica 的內部流程比同區域更複雜：

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 在 Source Region 建立 Snapshot                               │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 將 Snapshot 複製到目標 Region                                │
│     - 如果有加密，使用目標 Region 的 KMS Key 重新加密              │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 在目標 Region 從 Snapshot 還原 DB Instance                   │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 建立跨 Region 的 Replication Channel                        │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Replica 追趕 Source 的變更（Catch Up）                       │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. Replica 進入穩定的複製狀態                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 各引擎支援表格

| 引擎 | 支援跨區域 RR | Source 刪除時 Replica 行為 |
|------|-------------|-------------------------|
| **MySQL** | ✅ | Replica 被 Promote 為獨立 Instance |
| **MariaDB** | ✅ | Replica 被 Promote 為獨立 Instance |
| **PostgreSQL** | ✅ | Replica 被 Promote 為獨立 Instance |
| **Oracle** | ✅ | Replica 被 Promote 為獨立 Instance |
| **SQL Server** | ❌ 不支援 | N/A |

### 加密需求

跨區域複製的加密是一個重點：

| 場景 | 需求 |
|------|------|
| **Source 未加密** | Replica 也不加密 |
| **Source 加密** | 必須在目標 Region 指定 KMS Key |
| **KMS Key** | 不能使用 Source Region 的 Key，必須用目標 Region 的 |
| **Presigned URL** | API 建立加密的跨區域 Replica 時需要 Presigned URL |

```bash
# 建立加密的 Cross-Region Read Replica
aws rds create-db-instance-read-replica \
    --db-instance-identifier my-cross-region-rr \
    --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:my-source-db \
    --region us-west-2 \
    --kms-key-id arn:aws:kms:us-west-2:123456789012:key/abcd1234-5678-90ab-cdef-1234567890ab \
    --db-instance-class db.r6g.xlarge
```

### IAM 授權

Cross-Region Read Replica 的 IAM 設定比同區域複雜，常見的錯誤：

**錯誤 1：沒有授權 Source Region**
```json
{
    "Effect": "Allow",
    "Action": "rds:CreateDBInstanceReadReplica",
    "Resource": "*",
    "Condition": {
        "StringEquals": {
            "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
    }
}
```
必須同時包含 Source 和 Destination 兩個 Region。

**錯誤 2：沒有包含 Source DB Instance ARN**
```json
{
    "Effect": "Allow",
    "Action": "rds:CreateDBInstanceReadReplica",
    "Resource": [
        "arn:aws:rds:us-west-2:123456789012:db:my-cross-region-rr",
        "arn:aws:rds:us-east-1:123456789012:db:my-source-db"
    ]
}
```
Resource 必須同時包含 Source 和 Replica 的 ARN。

### 成本考量

| 費用項目 | 說明 |
|---------|------|
| **跨區域資料傳輸** | Source → Replica 的資料傳輸有費用（依 Region 定價）|
| **Replica Instance** | 目標 Region 的 Instance 費用 |
| **Replica Storage** | 目標 Region 的 Storage 費用 |
| **Snapshot 傳輸** | 初次建立時的 Snapshot 跨區傳輸費用 |

**Replica Chain 省費架構**

如果需要在多個 Region 建立 Replica，使用 Chain 可以減少 Source 的負載：

```
                                        ┌──────────────┐
                                   ┌───►│ us-west-2    │
┌──────────────┐     ┌──────────────┐   │ (RR Level 2) │
│  us-east-1   │────►│  eu-west-1   │   └──────────────┘
│  (Source)    │     │ (RR Level 1) ├───►┌──────────────┐
└──────────────┘     └──────────────┘   │ ap-southeast-1│
                                        │ (RR Level 2) │
                                        └──────────────┘

Source 只需要一條跨區複製，其餘由 Level 1 Replica 負責
```

### CLI 範例

```bash
# 建立 Cross-Region Read Replica（非加密）
aws rds create-db-instance-read-replica \
    --db-instance-identifier my-cross-region-rr \
    --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:my-source-db \
    --region us-west-2 \
    --db-instance-class db.r6g.xlarge

# Promote Cross-Region Replica（災難復原時使用）
aws rds promote-read-replica \
    --db-instance-identifier my-cross-region-rr \
    --region us-west-2
```

> **來源**: [Creating a read replica in a different AWS Region](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html)

---

## 4. Multi-AZ DB Instance Deployment

### 同步複製機制

Multi-AZ DB Instance 使用**同步複製**，在不同 AZ 維護一個 Standby Replica：

- Primary 的每個寫入操作都會**同步**複製到 Standby
- Standby **不可讀、不可寫**（純粹用於 Failover）
- 使用 Amazon 自有的 Failover 技術（MySQL, MariaDB, PostgreSQL, Oracle）
- SQL Server 使用 Database Mirroring (DBM) 或 Always On AG

### 架構圖

```
              Region: us-east-1
┌───────────────────────────────────────────────────┐
│                                                   │
│  AZ-a                        AZ-b                │
│  ┌──────────────┐           ┌──────────────┐     │
│  │   Primary    │  Sync     │   Standby    │     │
│  │   (Active)   │─────────►│  (Passive)   │     │
│  │              │Replication│              │     │
│  │  讀寫         │           │  不可讀寫     │     │
│  └──────┬───────┘           └──────┬───────┘     │
│         │                          │              │
│         │        DNS CNAME         │              │
│         │   ┌──────────────┐       │              │
│         └───┤  Endpoint    ├───────┘              │
│             │ mydb.xxx.    │                      │
│             │ rds.amazonaws│  ← Failover 時       │
│             │ .com         │    自動切換指向       │
│             └──────────────┘                      │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Failover 觸發場景

| 觸發場景 | 說明 | 是否自動 |
|---------|------|---------|
| **AZ 故障** | Primary 所在的 AZ 發生中斷 | ✅ 自動 |
| **Instance 故障** | Primary DB Instance 當機 | ✅ 自動 |
| **Instance 類型變更** | 修改 Instance Class | ✅ 自動（維護期間）|
| **軟體修補** | OS 或 DB Engine 更新 | ✅ 自動（維護期間）|
| **手動 Failover** | 使用 `reboot-db-instance --force-failover` | 手動觸發 |
| **Storage 故障** | Primary 的 Storage 發生問題 | ✅ 自動 |

### Failover 時間

| 項目 | 數值 |
|------|------|
| **典型 Failover 時間** | 60-120 秒 |
| **最快可達** | 約 60 秒（零資料遺失）|
| **DNS 切換方式** | CNAME 指向 Standby |

> **我的理解**：Failover 時間取決於多個因素：未完成的 Transaction 數量、DB Instance 大小、Recovery 過程的 I/O。建議應用程式要有 Retry 邏輯。

### DNS Failover 與 JVM TTL 設定

Failover 時，AWS 會自動將 DB Endpoint 的 **DNS CNAME** 從 Primary 指向 Standby。應用程式需要重新建立連線。

**Java 應用程式的關鍵設定：**

某些 JVM 預設會永久快取 DNS，導致 Failover 後仍連到舊的 Primary。AWS 建議 JVM TTL 設為**不超過 60 秒**：

```java
// 在建立任何網路連線之前設定
java.security.Security.setProperty("networkaddress.cache.ttl", "60");
```

或在 `$JAVA_HOME/jre/lib/security/java.security` 中設定：
```
networkaddress.cache.ttl=60
```

### 效能影響

- **Write Latency 增加**：同步複製需要等 Standby 確認，寫入延遲會比 Single-AZ 高
- **建議使用 Provisioned IOPS**：可以降低 I/O 延遲的影響
- **Standby 不分擔讀取**：所有讀寫都在 Primary，Standby 純粹備援
- **備份不影響 Primary**：自動備份在 Standby 上執行，不影響 Primary I/O

### 與 Read Replica 搭配

Multi-AZ 負責 HA，Read Replica 負責讀取擴展：

```
              AZ-a                    AZ-b                    AZ-c
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│     Primary      │     │     Standby      │     │  Read Replica    │
│   (讀寫)         │────►│   (同步備援)     │     │   (唯讀)         │
│                  │Sync │   不可讀寫        │     │                  │
└────────┬─────────┘     └──────────────────┘     └──────────────────┘
         │                                                 ▲
         │            Async Replication                    │
         └─────────────────────────────────────────────────┘
```

### CLI 範例

```bash
# 建立 Multi-AZ DB Instance
aws rds create-db-instance \
    --db-instance-identifier my-multi-az-db \
    --db-instance-class db.r6g.xlarge \
    --engine mysql \
    --engine-version 8.0 \
    --master-username admin \
    --master-user-password <password> \
    --allocated-storage 100 \
    --multi-az

# 將現有的 Single-AZ 轉換為 Multi-AZ
aws rds modify-db-instance \
    --db-instance-identifier my-single-az-db \
    --multi-az \
    --apply-immediately

# 手動觸發 Failover（測試用）
aws rds reboot-db-instance \
    --db-instance-identifier my-multi-az-db \
    --force-failover
```

> **來源**: [Multi-AZ DB instance deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html)

---

## 5. Multi-AZ DB Cluster Deployment

### 架構概述

Multi-AZ DB Cluster 是較新的部署模式：**1 Writer + 2 Readable Readers**，分佈在 3 個 AZ。

與 Multi-AZ DB Instance 最大的差異：**Readers 可以處理讀取流量**。

### 架構圖

```
              Region: us-east-1
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  AZ-a                   AZ-b                   AZ-c           │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐│
│  │    Writer     │      │   Reader 1   │      │   Reader 2   ││
│  │   Instance   │─────►│   Instance   │      │   Instance   ││
│  │              │ Semi │   (可讀)     │      │   (可讀)     ││
│  │   讀寫       │ Sync │              │      │              ││
│  └──────────────┘      └──────────────┘      └──────────────┘│
│         │                      ▲                      ▲       │
│         │         Semi-Sync    │                      │       │
│         └──────────────────────┴──────────────────────┘       │
│                                                                │
│  Endpoints:                                                    │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Cluster Endpoint   → Writer（讀寫）                     │   │
│  │ Reader Endpoint    → Reader 1 或 2（唯讀，負載平衡）    │   │
│  │ Instance Endpoint  → 指定的特定 Instance               │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 半同步複製機制

Multi-AZ DB Cluster 使用**半同步複製 (Semi-Synchronous Replication)**：

- Writer commit 時，等待**至少 1 個 Reader** 確認（ACK）收到變更
- 不需要兩個 Reader 都確認（這是與完全同步的差異）
- 確保至少一份最新的資料副本存在於不同 AZ

### 同步 vs 半同步 vs 非同步比較

| 特性 | 同步 (Multi-AZ Instance) | 半同步 (Multi-AZ Cluster) | 非同步 (Read Replica) |
|------|------------------------|--------------------------|---------------------|
| **commit 等待** | 等 Standby 確認 | 等至少 1 個 Reader ACK | 不等待 |
| **資料一致性** | 強一致 | 幾乎強一致 | 最終一致 |
| **Write 延遲影響** | 較高 | 中等 | 無影響 |
| **Replica 可讀** | ❌ | ✅ | ✅ |
| **Failover 時間** | 60-120 秒 | < 35 秒 | 手動 Promote |
| **資料遺失風險** | 零 | 極低 | 可能有 |

> **我的理解**：Multi-AZ DB Cluster 的 commit latency 比 Multi-AZ Instance 更低（最高快 2x），因為它使用 local storage 來寫 transaction logs，而不是透過網路同步。

### Failover 時間

| 項目 | 數值 |
|------|------|
| **典型 Failover 時間** | < 35 秒 |
| **資料遺失** | 零（半同步確保至少一份副本）|
| **Failover 過程** | Reader 被提升為 Writer |

### Failover 引擎差異

| 引擎 | Failover 行為 |
|------|-------------|
| **MySQL** | 兩個 Reader 都必須 apply 完未完成的 transactions 後，才能升級其中一個 |
| **PostgreSQL** | 只有 lag 最低的 Reader 需要 apply 完成後升級，速度較快 |

### Flow Control（Replica Lag 控制機制）

Multi-AZ DB Cluster 有內建的 Flow Control 機制來控制 Replica Lag：

- 當 Reader 的 Lag 超過閾值，系統會自動**降低 Writer 的寫入速度**
- 在交易結束時加入延遲 (delay) 來降低寫入吞吐量
- 目的是防止 Replica Lag 無限增大
- 適合 OLTP 短交易高併發的工作負載

#### 各引擎的 Flow Control 參數

| 引擎 | 參數 | 預設值 | 關閉方式 |
|------|------|--------|---------|
| **MySQL** | `rpl_semi_sync_master_target_apply_lag` | 120 秒 | 設為 86,400 秒 |
| **PostgreSQL** | `flow_control.target_standby_apply_lag` | 2 分鐘 | 從 `shared_preload_libraries` 移除 extension |

**MySQL 監控 Flow Control：**
```sql
SHOW GLOBAL STATUS LIKE '%flow_control%';
-- 觀察 Rpl_semi_sync_master_flow_control_current_delay（單位：微秒）
```

### Endpoint 架構

| Endpoint 類型 | 用途 | 指向 |
|-------------|------|------|
| **Cluster Endpoint** | 讀寫操作 | Writer Instance |
| **Reader Endpoint** | 唯讀操作 | Reader Instance（負載平衡）|
| **Instance Endpoint** | 連接特定 Instance | 指定的 Writer 或 Reader |

### 支援引擎

| 引擎 | 最低版本 |
|------|---------|
| **MySQL** | 8.0.28+ |
| **PostgreSQL** | 13.4+ |

### 支援的 Instance Class

- `db.m5d`, `db.m6gd`, `db.m6id`, `db.m6idn`
- `db.r5d`, `db.r6gd`, `db.r6id`, `db.r6idn`
- `db.c6gd`, `db.x2iedn`

> **注意**：只支援帶有 local NVMe SSD 的 Instance Class（用於寫 transaction logs）。

### 限制

| 限制 | 說明 |
|------|------|
| **不支援 Storage Autoscaling** | 需要手動擴展 Storage |
| **不支援停止/啟動** | 不能暫停 Cluster |
| **不能複製 Snapshot** | 不支援複製 Multi-AZ Cluster 的 Snapshot |
| **不能事後加密** | 建立時就要決定是否加密 |
| **不支援 Option Groups** | 僅限 MySQL |
| **MySQL 限制** | 所有 Table 必須有 Primary Key |
| **PostgreSQL 限制** | 不支援 `aws_s3` 和 `pg_transport` Extension |
| **不支援 IPv6** | 不支援 dual-stack mode |

### CLI 範例

```bash
# 建立 Multi-AZ DB Cluster
aws rds create-db-cluster \
    --db-cluster-identifier my-multi-az-cluster \
    --engine mysql \
    --engine-version 8.0.28 \
    --master-username admin \
    --master-user-password <password> \
    --db-cluster-instance-class db.r6gd.xlarge \
    --allocated-storage 100 \
    --storage-type io1 \
    --iops 3000 \
    --availability-zones us-east-1a us-east-1b us-east-1c

# 手動 Failover
aws rds failover-db-cluster \
    --db-cluster-identifier my-multi-az-cluster \
    --target-db-instance-identifier my-cluster-reader-1
```

> **來源**: [Multi-AZ DB cluster deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html)

---

## 6. Aurora Replication

### 共享儲存架構

Aurora 與標準 RDS 最大的差異在於**共享儲存**架構：

- 資料自動存儲 **6 份副本**，分佈在 **3 個 AZ**（每個 AZ 2 份）
- Writer 和 Readers **共享同一個 Storage Volume**
- Readers 不需要從 Writer 複製資料，而是直接讀取共享儲存

### 架構圖

```
┌────────────────────────────────────────────────────────────────┐
│                      Aurora Cluster                            │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   Writer     │  │  Reader 1    │  │  Reader N    │        │
│  │  Instance    │  │  Instance    │  │  Instance    │        │
│  │  (讀寫)      │  │  (唯讀)     │  │  (唯讀)     │        │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │
│         │                 │                  │                │
│         ▼                 ▼                  ▼                │
│  ┌────────────────────────────────────────────────────────┐   │
│  │              Shared Storage Volume                     │   │
│  │                                                        │   │
│  │   AZ-a          AZ-b          AZ-c                    │   │
│  │  ┌─────┐       ┌─────┐       ┌─────┐                 │   │
│  │  │Copy1│       │Copy3│       │Copy5│                  │   │
│  │  │Copy2│       │Copy4│       │Copy6│                  │   │
│  │  └─────┘       └─────┘       └─────┘                 │   │
│  │                                                        │   │
│  │  6 份副本跨 3 個 AZ，自動管理                           │   │
│  └────────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 與 RDS Read Replica 比較

| 特性 | Aurora Replica | RDS Read Replica |
|------|---------------|-----------------|
| **最大 Replica 數** | 15 | 15（MySQL/MariaDB/PG）、5（Oracle/SQL Server）|
| **複製方式** | 共享儲存（不需要複製資料）| 非同步複製 |
| **Replica Lag** | 通常 < 100ms（常 < 20ms）| 秒級（取決於負載）|
| **Failover 支援** | ✅ 自動 Failover | ❌ 需要手動 Promote |
| **Failover 時間** | 約 30 秒 | 分鐘級（手動操作）|
| **對 Source 效能影響** | 極小 | 有影響（因為要複製資料）|
| **獨立 Storage** | ❌ 共享 | ✅ 各自獨立 |

> **我的理解**：Aurora 的 Replica Lag 極低是因為 Readers 直接讀取共享儲存，不需要等 Writer 把資料「傳送」過來。只需要更新 Buffer Cache 中的 metadata，就知道有新資料了。

### Replica Lag

- **典型值**：< 100ms，常見 < 20ms
- **原因**：Writer 寫入共享儲存後，Readers 只需要更新 Cache 中的 page 資訊

### Failover Priority（tier 0-15）

Aurora 使用 **Priority Tier** 來決定 Failover 時哪個 Reader 被提升為 Writer：

| Tier | 優先順序 | 說明 |
|------|---------|------|
| **0** | 最高 | 最優先被選為新的 Writer |
| **1** | 高 | 次優先 |
| **2-14** | 中 | 一般優先順序 |
| **15** | 最低 | 最後才被選（適合用於分析用途的 Reader）|

同一 Tier 內，選擇 Instance 規格最大的。如果規格也相同，則隨機選擇。

**Failover 時間：**
- 有 Reader 存在時：**< 30 秒**
- 沒有 Reader 時：需要建立新的 Instance，可能需要 **< 10 分鐘**
- 如果 5 次嘗試都在同一 Tier 失敗，Aurora 會忽略 Tier 設定，選擇任何可用的 Reader

```bash
# 設定 Failover Priority
aws rds modify-db-instance \
    --db-instance-identifier my-aurora-reader \
    --promotion-tier 0

# 手動 Failover 到指定 Instance
aws rds failover-db-cluster \
    --db-cluster-identifier my-aurora-cluster \
    --target-db-instance-identifier my-aurora-reader
```

### Endpoint 架構

| Endpoint 類型 | 用途 | 說明 |
|-------------|------|------|
| **Cluster Endpoint** | 讀寫 | 指向 Writer Instance |
| **Reader Endpoint** | 唯讀 | 負載平衡到所有 Reader Instances |
| **Custom Endpoint** | 自定義 | 可以指定一組特定的 Instances |
| **Instance Endpoint** | 直連 | 直接連到特定的 Instance |

### CLI 範例

```bash
# 建立 Aurora Cluster
aws rds create-db-cluster \
    --db-cluster-identifier my-aurora-cluster \
    --engine aurora-mysql \
    --engine-version 8.0.mysql_aurora.3.04.0 \
    --master-username admin \
    --master-user-password <password>

# 建立 Writer Instance
aws rds create-db-instance \
    --db-instance-identifier my-aurora-writer \
    --db-instance-class db.r6g.xlarge \
    --engine aurora-mysql \
    --db-cluster-identifier my-aurora-cluster

# 新增 Reader Instance
aws rds create-db-instance \
    --db-instance-identifier my-aurora-reader-1 \
    --db-instance-class db.r6g.xlarge \
    --engine aurora-mysql \
    --db-cluster-identifier my-aurora-cluster
```

> **來源**: [Replication with Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html)

---

## 7. Aurora Global Database

### 概述

Aurora Global Database 提供**跨 Region** 的複製能力：

- **1 個 Primary Region** + **最多 10 個 Secondary Regions**
- 使用 **Storage-level Replication**（不是 Engine-level）
- 跨區複製延遲通常 **< 1 秒**

### 架構圖

```
┌─────────────────────────────────────────────────────────┐
│  Primary Region (us-east-1)                             │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐                    │
│  │    Writer     │  │  Reader(s)   │                    │
│  │   Instance    │  │  Instance    │                    │
│  └──────┬───────┘  └──────┬───────┘                    │
│         │                 │                             │
│         ▼                 ▼                             │
│  ┌────────────────────────────────────┐                 │
│  │   Primary Cluster Storage Volume  │                 │
│  └──────────────┬─────────────────────┘                 │
│                 │                                       │
└─────────────────┼───────────────────────────────────────┘
                  │ Storage-level Replication
                  │ (< 1 秒延遲)
                  ▼
┌─────────────────────────────────────────────────────────┐
│  Secondary Region (eu-west-1)                           │
│                                                         │
│  ┌────────────────────────────────────┐                 │
│  │  Secondary Cluster Storage Volume │                 │
│  └──────────────┬─────────────────────┘                 │
│         ▲                 ▲                             │
│         │                 │                             │
│  ┌──────┴───────┐  ┌──────┴───────┐                    │
│  │  Reader 1    │  │  Reader N    │                    │
│  │  Instance    │  │  Instance    │                    │
│  │  (唯讀)     │  │ (最多 16 個) │                    │
│  └──────────────┘  └──────────────┘                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Storage-level vs Engine-level 複製

| 特性 | Storage-level (Global DB) | Engine-level (Cross-Region RR) |
|------|--------------------------|-------------------------------|
| **複製層級** | 儲存層 | 資料庫引擎層 |
| **延遲** | < 1 秒 | 秒級 |
| **對 Writer 效能影響** | 極小 | 有影響 |
| **複製的內容** | 底層 Storage Blocks | SQL / WAL / Binlog |
| **頻寬使用** | 更有效率 | 較高 |
| **可用於** | Aurora 限定 | 所有 RDS 引擎 |

### Managed Planned Failover（Switchover）

用於**計畫性**的跨區域切換（如 Region 遷移、DR 演練）：

- **零資料遺失**：切換前會確保所有資料都已同步
- 將 Secondary Region 提升為新的 Primary Region
- 原本的 Primary Region 變成 Secondary
- 過程中 Database 暫時不可用

```bash
# Managed Planned Failover (Switchover)
aws rds switchover-global-cluster \
    --global-cluster-identifier my-global-cluster \
    --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:my-secondary-cluster
```

### Managed Unplanned Failover

用於**非計畫性**的災難復原：

- **RPO < 1 秒**：可能遺失最多不到 1 秒的資料
- 將 Secondary Region 強制提升為 Primary
- 適用於 Primary Region 完全故障的場景
- 會自動 detach 無法連線的 Region

```bash
# Managed Unplanned Failover
aws rds failover-global-cluster \
    --global-cluster-identifier my-global-cluster \
    --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:my-secondary-cluster
```

### Write Forwarding

Write Forwarding 讓 Secondary Region 的 Reader 可以執行寫入操作：

```
Secondary Region                        Primary Region
┌──────────────┐                        ┌──────────────┐
│   Reader     │   Write Forwarding     │    Writer    │
│  Instance    │──────────────────────►│   Instance   │
│              │     (透明轉發寫入)      │              │
│  應用程式連接 │◄──────────────────────│   執行寫入    │
│  這裡讀寫    │     (回傳結果)          │              │
└──────────────┘                        └──────────────┘
```

**Write Forwarding 支援的一致性模式（PostgreSQL）：**

| 模式 | 說明 |
|------|------|
| **SESSION** | 看到本 Session 的寫入結果，不等其他 Session |
| **EVENTUAL** | 可能看到舊資料，不等複製 |
| **GLOBAL** | 看到所有已提交的變更，等待完全同步 |
| **OFF** | 關閉 Write Forwarding |

**Write Forwarding 支援的一致性模式（MySQL）：**

| 模式 | 說明 |
|------|------|
| **SESSION** | 同上 |
| **EVENTUAL** | 同上 |
| **GLOBAL** | 同上 |

> **我的理解**：Write Forwarding 的延遲會比直接寫 Primary 高，因為寫入要經過兩次跨區網路傳輸（1. 轉發到 Primary；2. 結果回傳）。適合低頻寫入的場景。

### 限制

| 限制 | 說明 |
|------|------|
| **不支援 Aurora Serverless v1** | 只支援 Provisioned 和 Serverless v2 |
| **不支援 Backtracking** | Global DB 不能使用 Backtracking |
| **Cloning 限制** | 只能在 Primary Cluster 上做 Clone |
| **Write Forwarding 不支援 DDL** | 只能轉發 DML（INSERT, UPDATE, DELETE）|
| **每個 Secondary Cluster** | 最多 16 個 Reader Instance |
| **最多 Secondary Regions** | 10 個 |

### CLI 範例

```bash
# 建立 Global Database
aws rds create-global-cluster \
    --global-cluster-identifier my-global-cluster \
    --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:my-primary-cluster

# 新增 Secondary Region
aws rds create-db-cluster \
    --db-cluster-identifier my-secondary-cluster \
    --engine aurora-mysql \
    --engine-version 8.0.mysql_aurora.3.04.0 \
    --global-cluster-identifier my-global-cluster \
    --region eu-west-1 \
    --enable-global-write-forwarding

# 在 Secondary Region 新增 Reader Instance
aws rds create-db-instance \
    --db-instance-identifier my-secondary-reader-1 \
    --db-instance-class db.r6g.xlarge \
    --engine aurora-mysql \
    --db-cluster-identifier my-secondary-cluster \
    --region eu-west-1
```

> **來源**: [Using Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)

---

## 8. Replica Lag 深入解析

### 定義

Replica Lag 是指 Replica 的資料落後 Source/Primary 的時間差。Lag 越大，Replica 上讀到的資料越「舊」。

### 影響因素

| 因素 | 影響 |
|------|------|
| **Write 負載** | Source 的寫入量越大，Lag 越容易增加 |
| **網路延遲** | 跨 AZ 或跨 Region 的網路延遲 |
| **Replica 規格** | Replica 的 Instance Class 太小會導致追不上 |
| **長時間 Transaction** | 大型 Transaction 會造成突增的 Lag |
| **DDL 操作** | 如 `ALTER TABLE` 可能導致 Lag 大幅增加 |

### 各複製類型的 Lag 特性

| 複製類型 | 典型 Lag | 說明 |
|---------|---------|------|
| **Read Replica（同區域）** | 秒級 | 非同步複製，Lag 取決於寫入量和 Replica 規格 |
| **Cross-Region RR** | 秒到分鐘級 | 跨區網路延遲 + 非同步複製 |
| **Multi-AZ Instance** | N/A（同步）| 同步複製，不存在傳統意義的 Lag |
| **Multi-AZ Cluster** | 極低 | 半同步，Reader 通常落後極短時間 |
| **Aurora Replica** | < 100ms（通常 < 20ms）| 共享儲存，只需更新 Cache |
| **Aurora Global DB** | < 1 秒 | Storage-level 跨區複製 |

### 監控方式

#### CloudWatch Metrics

| Metric | 適用類型 | 說明 |
|--------|---------|------|
| `ReplicaLag` | 所有 | Replica 延遲（秒）|
| `AuroraReplicaLag` | Aurora | Aurora Replica 延遲（毫秒）|
| `AuroraReplicaLagMaximum` | Aurora | 所有 Replica 中最大的 Lag |
| `AuroraReplicaLagMinimum` | Aurora | 所有 Replica 中最小的 Lag |
| `AuroraGlobalDBReplicatedWriteIO` | Aurora Global | Global DB 的複製 I/O |
| `AuroraGlobalDBDataTransferBytes` | Aurora Global | 跨區傳輸的資料量 |
| `AuroraGlobalDBReplicationLag` | Aurora Global | Global DB 的複製延遲 |

#### SQL 查詢

**MySQL / MariaDB：**
```sql
-- 在 Replica 上執行
SHOW REPLICA STATUS\G
-- 重點觀察：
-- Seconds_Behind_Source: 落後秒數
-- Relay_Master_Log_File: 正在複製的 binlog 檔案
-- Exec_Master_Log_Pos: 正在執行的 binlog 位置
```

**PostgreSQL：**
```sql
-- 在 Source 上查看所有 Replica 的狀態
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- 在 Replica 上查看 Lag
SELECT
    now() - pg_last_xact_replay_timestamp() AS replica_lag;
```

> **來源**: [Working with DB instance read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)

---

## 9. 成本考量

### 各複製類型費用比較

| 複製類型 | Instance 費用 | Storage 費用 | 資料傳輸費用 | 總成本概估 |
|---------|-------------|-------------|-------------|----------|
| **Read Replica（同區域）** | 額外 Instance | 額外 Storage | 同 AZ 免費 | ~1x 額外 Instance |
| **Cross-Region RR** | 額外 Instance | 額外 Storage | 跨區傳輸費 | ~1x + 傳輸費 |
| **Multi-AZ Instance** | 包含在 Multi-AZ 定價 | 包含 | 同 Region 免費 | ≈ 2x Single-AZ |
| **Multi-AZ Cluster** | 3 個 Instance | 共享 Storage | 同 Region 免費 | ≈ 3x Single Instance |
| **Aurora Replica** | 額外 Instance | 共享（不額外收費）| 同 Region 免費 | ~1x 額外 Instance |
| **Aurora Global DB** | 各 Region Instance | 各 Region Storage | 跨區複製費 | 各 Region 分別計費 |

### Multi-AZ 定價模式

| 部署模式 | 定價 | 說明 |
|---------|------|------|
| **Single-AZ** | 基本價格 | 只有一個 Instance |
| **Multi-AZ Instance** | ≈ 2x Single-AZ | 包含 Standby 的費用 |
| **Multi-AZ Cluster** | ≈ 3x 單一 Instance | 1 Writer + 2 Readers |

### 成本最佳化建議

| 策略 | 說明 |
|------|------|
| **Reserved Instances** | 對長期使用的 Replica 購買 RI，可省 30-60% |
| **Replica Chain** | 跨區複製時用 Chain 架構，減少 Source 負載和跨區傳輸 |
| **Aurora Serverless v2** | Reader 可用 Serverless v2，依用量計費，低流量時更省 |
| **適當的 Instance Class** | Replica 不需要跟 Source 一樣大（如果讀取負載較低）|
| **定期檢視** | 監控 Replica 使用率，關閉不必要的 Replica |

> **來源**: [Amazon RDS Pricing](https://aws.amazon.com/rds/pricing/)

---

## 10. 綜合比較與決策指南

### 完整功能比較

| 特性 | Read Replica | Cross-Region RR | Multi-AZ Instance | Multi-AZ Cluster | Aurora Replica | Aurora Global DB |
|------|-------------|----------------|-------------------|-----------------|---------------|-----------------|
| **主要用途** | 讀取擴展 | DR / 跨區讀取 | HA | HA + 讀取 | 讀取 + HA | 跨區 DR + 讀取 |
| **複製方式** | 非同步 | 非同步 | 同步 | 半同步 | 共享儲存 | Storage-level |
| **Replica 可讀** | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| **自動 Failover** | ❌ | ❌ | ✅ | ✅ | ✅ | ✅（Managed）|
| **Failover 時間** | 手動 | 手動 | 60-120s | < 35s | ~30s | 分鐘級 |
| **跨 Region** | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| **最大 Replica 數** | 15 | 5 per Region | 1 Standby | 2 Readers | 15 | 16 per Region |
| **Replica Lag** | 秒級 | 秒到分鐘 | N/A | 極低 | < 100ms | < 1s |
| **資料遺失風險** | 有 | 有 | 零 | 極低 | 極低 | RPO < 1s |
| **額外成本** | +1x Instance | +1x + 傳輸 | ≈2x | ≈3x | +1x Instance | 各 Region 分計 |

### 決策樹

```
需要資料庫高可用性嗎？
│
├─ 否 → 只需要讀取擴展嗎？
│        ├─ 是 → 使用 Read Replica
│        └─ 否 → Single-AZ Instance（最省成本）
│
├─ 是 → 使用 Aurora 還是 Standard RDS？
│        │
│        ├─ Aurora
│        │   ├─ 需要跨 Region DR？
│        │   │   ├─ 是 → Aurora Global Database
│        │   │   └─ 否 → Aurora Cluster（Writer + Readers）
│        │   └─ 需要讀取擴展？
│        │       ├─ 是 → 增加 Aurora Replica
│        │       └─ 否 → 最少 1 Writer + 1 Reader
│        │
│        └─ Standard RDS
│            ├─ 需要 Replica 可讀？
│            │   ├─ 是 → Multi-AZ DB Cluster（MySQL 8.0.28+ / PG 13.4+）
│            │   └─ 否 → Multi-AZ DB Instance
│            └─ 需要跨 Region DR？
│                ├─ 是 → Multi-AZ + Cross-Region Read Replica
│                └─ 否 → Multi-AZ Instance / Cluster
```

### 常見架構模式

#### 模式一：基本 HA（最常見）

```
Multi-AZ DB Instance
┌──────────┐     ┌──────────┐
│ Primary  │────►│ Standby  │
│ (AZ-a)   │Sync │ (AZ-b)   │
└──────────┘     └──────────┘
```
**適用**：需要 HA 但不需要讀取擴展的場景。

#### 模式二：HA + 讀取擴展

```
┌──────────┐     ┌──────────┐
│ Primary  │────►│ Standby  │  ← Multi-AZ (HA)
│ (AZ-a)   │Sync │ (AZ-b)   │
└────┬─────┘     └──────────┘
     │ Async
     ├──────────►┌──────────┐
     │           │ Read RR  │  ← Read Replica (讀取)
     │           │ (AZ-c)   │
     └──────────►┌──────────┐
                 │ Read RR  │  ← Read Replica (讀取)
                 │ (AZ-a)   │
                 └──────────┘
```
**適用**：需要 HA + 讀取分流的場景。

#### 模式三：Aurora 全方位

```
Aurora Cluster
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Writer   │  │ Reader 1 │  │ Reader 2 │
│ (AZ-a)   │  │ (AZ-b)   │  │ (AZ-c)   │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     ▼             ▼             ▼
┌────────────────────────────────────────┐
│        Shared Storage Volume          │
│     (6 copies across 3 AZs)          │
└────────────────────────────────────────┘
```
**適用**：需要低 Lag 讀取、自動 Failover、高效能的場景。

#### 模式四：跨區域 DR

```
Primary Region                Secondary Region
┌──────────────────┐         ┌──────────────────┐
│  Aurora Cluster  │         │  Aurora Cluster  │
│  ┌────┐ ┌────┐  │         │  ┌────┐ ┌────┐  │
│  │ W  │ │ R  │  │         │  │ R  │ │ R  │  │
│  └──┬─┘ └──┬─┘  │         │  └──┬─┘ └──┬─┘  │
│     ▼      ▼    │         │     ▼      ▼    │
│  ┌──────────┐   │ Storage │  ┌──────────┐   │
│  │ Storage  │───┼─Level──►│  │ Storage  │   │
│  │ Volume   │   │  < 1s   │  │ Volume   │   │
│  └──────────┘   │         │  └──────────┘   │
└──────────────────┘         └──────────────────┘
```
**適用**：需要跨區域 DR + 各地區低延遲讀取。

> **來源**: [Amazon RDS Read Replicas](https://aws.amazon.com/rds/features/read-replicas/)

---

## 11. 參考資料

### 官方文檔完整連結

| 主題 | 連結 |
|------|------|
| Read Replicas 概觀 | [USER_ReadRepl.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html) |
| 建立 Read Replica | [USER_ReadRepl.Create.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.Create.html) |
| Cross-Region Read Replica | [USER_ReadRepl.XRgn.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.XRgn.html) |
| Multi-AZ DB Instance | [Concepts.MultiAZSingleStandby.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZSingleStandby.html) |
| Multi-AZ DB Cluster | [multi-az-db-clusters-concepts.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html) |
| Multi-AZ Cluster 限制 | [multi-az-db-clusters-concepts.Limitations.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.Limitations.html) |
| Aurora Replication | [Aurora.Replication.html](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html) |
| Aurora Global Database | [aurora-global-database.html](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html) |
| Aurora Write Forwarding (MySQL) | [aurora-global-database-write-forwarding-mysql.html](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding-mysql.html) |
| Aurora Write Forwarding (PostgreSQL) | [aurora-global-database-write-forwarding-apg.html](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-write-forwarding-apg.html) |
| RDS Read Replicas 功能介紹 | [aws.amazon.com/rds/features/read-replicas](https://aws.amazon.com/rds/features/read-replicas/) |
| RDS 定價 | [aws.amazon.com/rds/pricing](https://aws.amazon.com/rds/pricing/) |
| Aurora 定價 | [aws.amazon.com/rds/aurora/pricing](https://aws.amazon.com/rds/aurora/pricing/) |

### 相關資源

- [AWS RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)
- [AWS Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- [AWS CLI rds Commands](https://docs.aws.amazon.com/cli/latest/reference/rds/)
