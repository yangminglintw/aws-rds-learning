# AWS RDS Blue/Green Deployments 學習筆記

這篇筆記整理了 AWS RDS Blue/Green Deployments 的完整內容，包含官方文檔中的所有重要主題。

---

## 1. 簡介

### 什麼是 Blue/Green Deployment？

Blue/Green Deployment 是一種部署策略，建立一個與 Production 完全相同的 Staging 環境，在 Staging 完成變更與測試後，再將流量切換過去。

| 環境 | 角色 |
|------|------|
| **Blue** | 目前的 Production 環境 |
| **Green** | 同步的 Staging 副本 |

### 主要優點

傳統資料庫升級的問題：
- 停機時間長（30 分鐘到數小時）
- 只有一次機會，壓力大
- 維護時間內難以完整測試

Blue/Green 的優點：
- 可以提前在 Green 環境測試
- Switchover 通常 **< 1 分鐘**
- 出問題可以快速切回 Blue
- 減少升級風險

> **來源**: [Blue/Green Deployments Overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-overview.html)

---

## 2. 支援的資料庫引擎與版本

### 支援狀況總覽

| 引擎 | 支援狀態 | 支援版本 |
|------|---------|---------|
| RDS for MySQL | ✅ | 5.7, 8.0, 8.4 |
| RDS for MariaDB | ✅ | 10.2+, 11.4, 11.8 |
| RDS for PostgreSQL | ✅ | 11.21+, 12.16+, 13.12+, 14.9+, 15.4+ |
| Amazon Aurora | ❌ | 有獨立的 Blue/Green 功能 |
| RDS for Oracle | ❌ | 不支援 |
| RDS for SQL Server | ❌ | 不支援 |

### 版本詳細說明

#### MySQL
- 5.7.44 及更新版本
- 8.0.36 及更新版本
- 8.4.0 及更新版本

#### MariaDB
- 10.2.44 及更新版本
- 10.3.39 及更新版本
- 10.4 ~ 10.11 的支援版本
- 11.4.3 及更新版本
- 11.8 及更新版本

#### PostgreSQL
- 11.21 及更新版本
- 12.16 及更新版本
- 13.12 及更新版本
- 14.9 及更新版本
- 15.4 及更新版本
- 16 及更新版本

> **來源**: [Blue/Green Deployments Supported Regions and Engines](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RDS_Fea_Regions_DB-eng.Feature.BlueGreenDeployments.html)

---

## 3. 運作原理

### Topology 複製

建立 Blue/Green Deployment 時，AWS 會複製整個 Topology，包含以下項目：

| 複製項目 | 說明 |
|---------|------|
| **Primary DB Instance** | 主要資料庫實例 |
| **Read Replicas** | Blue 有幾個就複製幾個 |
| **Multi-AZ Standby** | Blue 是 Multi-AZ → Green 也是 Multi-AZ（含 Standby） |
| **Storage Configuration** | 儲存設定（類型、大小、IOPS）|
| **Automated Backups** | 自動備份設定 |
| **Performance Insights** | 效能監控設定 |
| **Enhanced Monitoring** | 進階監控設定 |

#### 三種 Topology 情境

**情境一：Standalone（只有 Primary）**

```
Blue                           Green
┌─────────────────┐            ┌─────────────────┐
│  Primary        │            │  Primary        │
│  (mydb1)        │───────────►│  (mydb1-green-  │
│                 │ Replication│   abc123)        │
└─────────────────┘            └─────────────────┘
```

**情境二：Primary + Read Replicas**

```
Blue                           Green
┌─────────────────┐            ┌─────────────────┐
│  Primary        │            │  Primary        │
│  (mydb1)        │───────────►│  (mydb1-green-  │
└────────┬────────┘ Replication│   abc123)        │
         │                     └────────┬────────┘
         │                              │
┌────────▼────────┐            ┌────────▼────────┐
│  Read Replica   │            │  Read Replica   │
│  (mydb2)        │            │  (mydb2-green-  │
└─────────────────┘            │   abc123)       │
                               └─────────────────┘
```

**情境三：Multi-AZ + Read Replicas（含 Standby）**

```
Blue                           Green
┌─────────────────┐            ┌─────────────────┐
│  Primary        │            │  Primary        │
│  (mydb1)        │───────────►│  (mydb1-green-  │
│  [AZ-a]         │ Replication│   abc123)       │
└───────┬─────────┘            │  [AZ-a]         │
        │                      └───────┬─────────┘
        │ Sync                         │ Sync
        │ Replication                  │ Replication
┌───────▼─────────┐            ┌───────▼─────────┐
│  Standby        │            │  Standby        │
│  [AZ-b]         │            │  [AZ-b]         │
└─────────────────┘            └─────────────────┘
        │                              │
┌───────▼─────────┐            ┌───────▼─────────┐
│  Read Replica   │            │  Read Replica   │
│  (mydb2)        │            │  (mydb2-green-  │
└─────────────────┘            │   abc123)       │
                               └─────────────────┘
```

#### 命名規則

建立時，Green 環境的 DB Identifier 加上 `-green-<隨機字串>` 後綴。

Switchover 後重新命名（**整個 Topology 一起換**）：

| 角色 | Switchover 前 | Switchover 後 |
|------|-------------|-------------|
| Green Primary | `mydb1-green-abc123` | → `mydb1` |
| Green Replica | `mydb2-green-abc123` | → `mydb2` |
| 舊 Blue Primary | `mydb1` | → `mydb1-old1` |
| 舊 Blue Replica | `mydb2` | → `mydb2-old1` |

> **重點**：不只 Primary 的 endpoint 換了，**Replica 的 endpoint 也一起換了**。應用程式如果有直連 Read Replica endpoint，也會自動切到新的 Green Replica。

#### Read Replica 特殊規則

**Storage 處理**：
- Green 所有 Replica 的 **allocated storage** 會統一成 **Green Primary 的大小**
- 其他 storage 參數（IOPS、storage type）繼承自對應的 Blue Replica
- 原因：如果 Green Primary 升級了 storage，AWS 確保 Replica storage 不會比 Primary 小

**Instance Class 升級限制**：
- 建立時**只能升級 Primary 的 Instance Class**
- Read Replica 預設繼承 Blue 環境的 Instance Class
- Green 建好後，必須**手動**修改 Replica 的 Instance Class：

```bash
aws rds modify-db-instance \
    --db-instance-identifier mydb2-green-abc123 \
    --db-instance-class db.r6g.2xlarge \
    --apply-immediately
```

> **坑**：如果忘了手動改，Switchover 後會出現大台 Primary 配小台 Replica 的情況。

#### Single-AZ 轉 Multi-AZ

- 建立 Blue/Green 時，可以將 Blue（Single-AZ）升級為 Green（Multi-AZ）
- Storage initialization 會包含 **Standby node**，預熱資料時要考慮 Standby 也需要初始化
- 這是測試 Multi-AZ 架構的好時機

#### 建立本質

Green 環境的建立流程：**Snapshot → Restore → Replication**

1. 對 Blue 做 Snapshot
2. 從 Snapshot Restore 出 Green 環境
3. 建立 Blue → Green 的 Replication

> **影響**：因為是從 Snapshot Restore，Green 使用 **Lazy Loading**（首次存取的 block 才會從 S3 載入）。這會導致初期查詢延遲較高，詳見[第 5 節：Lazy Loading](#lazy-loading--storage-initialization)。

### Replication 機制

#### MySQL / MariaDB

使用 **Binary Log Replication**：
- 透過 binlog 複製所有變更
- 需要啟用自動備份（backup retention > 0）
- Green 環境可以讀寫（但寫入會在 Switchover 時被捨棄）

#### PostgreSQL

有兩種模式，根據版本升級類型自動選擇：

| 模式 | 使用時機 | 特點 |
|------|---------|------|
| **Physical Replication** | Minor 版本升級、相同版本 | Green 為唯讀，使用 WAL |
| **Logical Replication** | Major 版本升級 | Green 可讀寫，限制較多 |

> **來源**: [Blue/Green Deployments Overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-overview.html)

---

## 4. PostgreSQL 複製方法

### Physical Replication vs Logical Replication

| 特性 | Physical Replication | Logical Replication |
|------|---------------------|---------------------|
| **使用時機** | Minor version upgrade、無版本變更 | Major version upgrade |
| **Green 環境狀態** | Read-only | Read-write |
| **複製方式** | WAL (Write-Ahead Log) | 邏輯解碼 (Logical Decoding) |
| **限制** | 較少 | 較多（見下方詳述）|
| **效能影響** | 較小 | 較大 |

### 版本對照表

AWS 會根據 Blue/Green 環境的版本組合自動決定使用哪種複製方法：

| Blue 版本 | Green 版本 | 複製方法 |
|-----------|-----------|---------|
| 15.x | 15.x | Physical |
| 15.x | 16.x | Logical |
| 14.x | 15.x | Logical |
| 14.x | 14.x | Physical |

**原則**：
- 相同 Major 版本 → Physical Replication
- 不同 Major 版本 → Logical Replication

### Physical Replication 限制

- Green 環境為 **唯讀**
- 不支援變更 DB Parameter Group（除了少數參數）
- 不能在 Green 環境安裝新的 Extensions

### Logical Replication 限制

這是做 Major 版本升級時要特別注意的：

| 限制 | 說明 |
|------|------|
| DDL 不複製 | `CREATE TABLE`, `ALTER TABLE` 等不會同步到 Green |
| 需要 Primary Key | 沒有 PK 的表無法執行 `UPDATE`/`DELETE` 複製 |
| 需要 Replica Identity | 沒有 PK 的表需要設定 `REPLICA IDENTITY FULL` |
| Large Objects | `bytea` 之外的 Large Objects 不會複製 |
| Sequence 值 | Sequence 的 last value 不會自動同步 |
| TRUNCATE | `TRUNCATE` 指令不會複製 |

> **我的理解**：Logical Replication 的原理是複製「SQL 語句的結果」而不是「底層的資料變更」，所以有些操作無法正確複製。

> **來源**: [Blue/Green Deployments Replication Type](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-replication-type.html)

---

## 5. 建立 Blue/Green Deployment

### 前置條件

#### 所有引擎通用
- 必須啟用自動備份（Backup Retention Period > 0）
- Blue 環境必須處於 `available` 狀態
- 不能有進行中的 Maintenance Window
- 不能使用 RDS Proxy
- **不支援使用 Secrets Manager 自動管理 Master Password**（這是常見的坑，因為很多人使用此功能）

#### MySQL / MariaDB 特定
- 必須啟用 Binary Logging
- `binlog_format` 必須設為 `ROW`
- 如果有 Read Replica，必須使用 GTID-based replication
- **Event Scheduler 必須停用**：Green 環境的 `event_scheduler` 必須設為 `OFF`

> **為什麼要停用 Event Scheduler？** Blue 的 Event 產生的 DML（如 `DELETE`、`UPDATE`）會透過 Replication 傳到 Green。如果 Green 也同時跑相同的 Event，會**重複執行**，導致資料不一致。
>
> ```sql
> -- 範例：Blue 上的清理 Event
> CREATE EVENT cleanup_temp
> ON SCHEDULE EVERY 5 MINUTE
> DO DELETE FROM temp_data WHERE created_at < NOW() - INTERVAL 1 HOUR;
>
> -- 如果 Green 也啟用了 Event Scheduler：
> -- 1. Green 自己的 Event 跑了一次 DELETE
> -- 2. Blue 的 DELETE 又透過 Replication 傳過來
> -- → 資料不一致！
> ```

- **Custom Option Group 限制**：如果 Blue 有 custom option group，**不能同時做 major version upgrade**，必須分兩步：
  1. 先建立 Blue/Green Deployment（不升級版本）
  2. Green 建好後，再在 Green 環境裡做版本升級

#### PostgreSQL 特定
- 對於 Logical Replication，需要 `rds.logical_replication` = `1`
- 建議設定足夠的 `max_replication_slots`
- 建議設定足夠的 `max_wal_senders`

### 可設定的參數

建立 Blue/Green Deployment 時，可以為 Green 環境設定：

| 參數 | 說明 |
|------|------|
| **Engine Version** | 可以升級到更新的版本 |
| **DB Parameter Group** | 可以使用不同的參數群組 |
| **DB Instance Class** | 可以選擇不同的 Instance 規格 |
| **Storage Configuration** | 可以調整儲存設定 |

### Lazy Loading & Storage Initialization

當 Green 環境建立時，資料是透過 Snapshot 復原的，但會使用 **Lazy Loading**：

- 資料不會立即全部載入
- 首次存取的資料會從 S3 載入
- 這可能導致初期的查詢延遲較高

**Storage Initialization 建議**：
- 建立 Green 後，執行完整的資料掃描來「預熱」資料
- 可以使用 `SELECT * FROM table` 或 `pg_prewarm` extension

### 建立方式

#### Console
1. 選擇要建立 Blue/Green Deployment 的 DB Instance
2. Actions → Create Blue/Green Deployment
3. 設定 Green 環境的配置
4. 建立

#### AWS CLI
```bash
aws rds create-blue-green-deployment \
    --blue-green-deployment-name my-blue-green-deployment \
    --source arn:aws:rds:us-east-1:123456789012:db:my-db \
    --target-engine-version 8.0.35 \
    --target-db-parameter-group-name my-new-param-group
```

> **來源**: [Creating a Blue/Green Deployment](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-creating.html)

---

## 6. Switchover 流程

> ⛔ **絕對不要手動 Promote Green！** 不要在 Green 環境的 Actions 選單選「Promote」。手動 Promote 會直接斷掉 Replication，Blue/Green Deployment 進入 **Invalid configuration** 狀態，只能刪除重建。**必須使用 Switchover 功能，不是 Promote**。

### Guardrails 檢查

Switchover 開始前，AWS 會執行一系列檢查（稱為 Guardrails）：

| 檢查項目 | 說明 |
|---------|------|
| **Replication Status** | 確認 Blue 到 Green 的複製正常運作 |
| **Replication Lag** | 確認 Replica Lag 在可接受範圍內 |
| **Long-running Transactions** | 檢查是否有長時間執行的交易 |
| **Long-running Queries** | 檢查是否有長時間執行的查詢 |
| **Active DDL** | 確認沒有正在執行的 DDL 語句 |

如果任一檢查失敗，Switchover 會被取消。

#### Logical Replication 額外 Guardrail（PostgreSQL Major Upgrade）

使用 Logical Replication 時，Guardrails 會額外檢查：
- Green 環境有沒有收到**不該有的 DDL 變更**
- 有沒有 **Large Object** 被修改

如果偵測到上述問題：
- Replication state 變成 **`Replication degraded`**
- Switchover **不可用**
- **唯一解法**：刪掉 Blue/Green Deployment 重建

> **Switchover 失敗是安全的**：如果 Switchover 因任何原因中斷（timeout、guardrail 失敗），所有變更會被**回滾**。兩邊環境都不受影響，可以修正問題後重試。

### Switchover 步驟

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Guardrails 檢查                                              │
│     - 確認 Replication 正常                                      │
│     - 確認沒有長時間執行的 Transaction                            │
│     - 確認 Replica Lag 在可接受範圍                               │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 停止 Blue 寫入                                               │
│     - Blue Primary 變成 read-only                                │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 中斷連線                                                     │
│     - 斷開所有到 Blue 的連線                                      │
│     - 阻擋新連線                                                 │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 等待同步                                                     │
│     - 等 Replication 完全追上（Lag → 0）                          │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 重新命名 Endpoint                                            │
│     - db-prod-green-xxx  →  db-prod                              │
│     - db-prod            →  db-prod-old1                         │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 允許新連線                                                   │
│     - 應用程式連到原本的 endpoint，自動連到新的 Green             │
└─────────────────────────────────────────────────────────────────┘
```

### Switchover 後 Blue 變成 Read-Only

Switchover 完成後，舊的 Blue 環境會**自動變成 Read-Only**（防止應用程式誤寫舊 DB）。

| 引擎 | 設定的參數 | 說明 |
|------|----------|------|
| **MySQL** | `read_only = 1`（可能加上 `super_read_only = 1`）| 透過 `rdsadmin` 帳號設定 |
| **PostgreSQL** | `default_transaction_read_only = on` | 透過 `rdsadmin` 帳號設定 |

**為什麼 read_only 對你的帳號有效？** 因為你的 master user 只有 `rds_superuser` 角色，**沒有**真正的 MySQL `SUPER` 或 PostgreSQL `superuser` 權限，所以無法繞過 read_only。詳見[附錄 A：RDS 內部帳號架構](#附錄-a-rds-內部帳號架構)。

**如何恢復寫入**（不能直接用 SQL）：
1. 修改 Parameter Group：`read_only = 0`（MySQL）或 `default_transaction_read_only = off`（PostgreSQL）
2. Reboot Instance
3. AWS 的 `rdsadmin` 帳號會套用新設定

### Endpoint 重新命名與 DNS

AWS 的做法是直接改 **Instance Identifier**，而不是 DNS 切換：
- `mydb-green-xxx` 直接變成 `mydb`
- Endpoint 格式是 `{instance-id}.xxx.{region}.rds.amazonaws.com`
- 所以 Endpoint 自動就變了

#### DNS TTL：必須 ≤ 5 秒

RDS DNS zone 預設 TTL 就是 **5 秒**（不是 60 秒！）。問題出在 Application 和 Network 層會「偷 cache」：

```
DNS 解析流程（每一層都可能 cache）：

┌─────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────────┐
│  JVM    │ →  │  OS DNS  │ →  │  DNS Resolver│ →  │  RDS DNS     │
│  Cache  │    │  Cache   │    │  (公司/ISP)  │    │  Zone        │
│  ⚠️ 永久 │    │  因系統而異│    │  因設定而異  │    │  TTL = 5 秒  │
└─────────┘    └──────────┘    └──────────────┘    └──────────────┘
```

**最常見的坑：JVM DNS Cache**

Java 預設**永久 cache DNS**，Switchover 後 Java 應用還是連到舊的 Blue：

```java
// Java 預設行為：永久 cache！
// networkaddress.cache.ttl = -1

// 修正方式一：程式碼中設定
java.security.Security.setProperty("networkaddress.cache.ttl", "5");

// 修正方式二：在 java.security 檔案設定
// networkaddress.cache.ttl=5
```

**其他需注意的 cache 層**：
- **OS DNS Cache**：Linux `systemd-resolved`、macOS 內建 DNS cache
- **Connection Pool**：HikariCP、PgBouncer 建立連線時記住 IP，之後不重新 DNS lookup
  - 解法：設定 `maxLifetime`（HikariCP 建議 `maxLifetime = 1800000`，30 分鐘）
- **中間 Network 設備**：公司 DNS resolver、Load Balancer、VPN gateway

### Tag 行為

Switchover 時，**Blue 的 tags 會覆蓋 Green 的所有 tags**。如果在 Green 測試期間加了 tags（如 `environment=staging`），Switchover 後會被 Blue 的 tags 蓋掉。建議所有重要 tags 在建立前就設定在 Blue 上。

### PostgreSQL ANALYZE（Logical Replication）

使用 Logical Replication 做 Major version upgrade 時，Switchover 前必須在所有 database 執行 `ANALYZE`：

```sql
ANALYZE;
```

**原因**：Logical Replication **不會同步** `pg_statistic`（query optimizer 的統計資料）。如果不跑 ANALYZE，Switchover 後 query optimizer 沒有統計資料，查詢效能可能暴跌。

### Timeout 設定

| 設定值 | 說明 |
|--------|------|
| 最小值 | 30 秒 |
| 預設值 | 300 秒（5 分鐘）|
| 最大值 | 3600 秒（1 小時）|

如果在 Timeout 時間內無法完成同步，Switchover 會失敗並回滾。

### 監控 Switchover

#### CloudWatch Metrics

| Metric | 說明 |
|--------|------|
| `ReplicaLag` | Blue 到 Green 的延遲（秒）|
| `DatabaseConnections` | 連線數量 |
| `CPUUtilization` | CPU 使用率 |

#### PostgreSQL 監控查詢

檢查 Replication Lag：
```sql
-- 在 Blue 環境執行
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

檢查長時間執行的交易：
```sql
SELECT
    pid,
    now() - xact_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

### 更新外部 Replicas (MySQL/MariaDB)

如果你有外部的 Read Replica（不在 Blue/Green Topology 內），Switchover 後需要手動更新。

#### 完整流程

**Step 1**：Switchover 後，Green DB instance 會發出一個 Event，內含 binary log 座標：

```
Binary log coordinates: file mysql-bin-changelog.000003, position 40134574
```

**Step 2**：去 RDS Console → Events 找到這個事件。

**Step 3**：確保外部 Replica 已經 apply 完舊 Blue 的所有 binlog。

**Step 4**：用座標重新指向新的 Primary：

```sql
-- MySQL 8.0.23+
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='{new-writer-endpoint}',
    SOURCE_LOG_FILE='mysql-bin-changelog.000003',
    SOURCE_LOG_POS=40134574;

START REPLICA;
```

> **來源**: [Switching a Blue/Green Deployment](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-switching.html)

---

## 7. 刪除 Blue/Green Deployment

### 何時刪除

- **Switchover 完成後**：測試確認沒問題，可以刪除舊的 Blue 環境
- **不再需要時**：決定不繼續 Blue/Green Deployment
- **發現問題時**：需要放棄這次部署

### 保留 vs 刪除 Green 環境

刪除 Blue/Green Deployment 時，可以選擇是否保留 Green 環境：

| 選項 | 說明 |
|------|------|
| **保留 Green** | Green 環境變成獨立的 DB Instance，不再與 Blue 同步 |
| **刪除 Green** | Green 環境被刪除，釋放資源 |

**注意**：刪除 Blue/Green Deployment 本身不會刪除任何 DB Instance，只是解除它們之間的關聯。你需要另外手動刪除不需要的 DB Instance。

### 刪除方式

#### Console
1. 選擇要刪除的 Blue/Green Deployment
2. Actions → Delete Blue/Green Deployment
3. 確認

#### AWS CLI
```bash
aws rds delete-blue-green-deployment \
    --blue-green-deployment-identifier bgd-1234567890abcdef0 \
    --delete-target
```

`--delete-target` 參數決定是否刪除 Green 環境。

> **來源**: [Deleting a Blue/Green Deployment](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-deleting.html)

---

## 8. 限制與注意事項

### 一般限制

這些功能目前不支援 Blue/Green：

| 限制 | 說明 |
|------|------|
| **RDS Proxy** | 如果用了 RDS Proxy，無法使用 Blue/Green |
| **Cross-Region Read Replica** | 有跨區 Replica 的不能用 |
| **CloudFormation** | 無法用 CFN 管理 Blue/Green Deployment |
| **Multi-AZ DB Cluster** | 只支援 Multi-AZ DB Instance |
| **Cascading Read Replica** | 有串接 Replica 的不能用 |
| **加密狀態變更** | 不能從加密變非加密（或反過來）|
| **降級版本** | 只能升級，不能降級 |
| **每帳戶限制** | 同一帳戶同一 Region 最多 5 個 Blue/Green Deployments |
| **IAM Roles 不自動複製** | DB instance 關聯的 IAM roles（如 `aws_s3` extension 用的）不會複製到 Green，Switchover 後需手動重新關聯 |
| **Dedicated Log Volume (DLV)** | 如果啟用 DLV，**所有** DB instances（含 Read Replicas）都必須一致啟用，不能部分啟用 |
| **Zero-ETL Integration** | 如果有和 Amazon Redshift 的 zero-ETL integration，Switchover 前**必須先刪除**，之後再重建 |
| **DMS Task 不能延續** | Switchover 後 AWS DMS replication task 無法繼續，因為 Blue 的 checkpoint 在 Green 環境無效，必須用新的 checkpoint 重建 DMS task |

### MySQL 特定限制

| 限制 | 說明 |
|------|------|
| **Binary Log 必須啟用** | 需要 `binlog_format = ROW` |
| **GTID 限制** | 如果有 Read Replica，必須使用 GTID-based replication |
| **MyISAM 引擎** | 建議不要使用 MyISAM，可能有複製問題 |
| **部分表複製** | 不支援只複製部分表 |

### PostgreSQL Physical Replication 限制

| 限制 | 說明 |
|------|------|
| **Green 環境唯讀** | 無法在 Green 環境執行寫入操作 |
| **Parameter Group 限制** | 部分參數不能變更 |
| **Extension 安裝** | 不能在 Green 環境安裝新的 Extension |

### PostgreSQL Logical Replication 限制

這是做 Major 版本升級時要特別注意的：

| 限制 | 說明 |
|------|------|
| **DDL 不複製** | `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` 等不會同步 |
| **需要 Primary Key** | 沒有 PK 的表無法執行 `UPDATE`/`DELETE` 複製 |
| **需要 Replica Identity** | 沒有 PK 的表需要設定 `REPLICA IDENTITY FULL` |
| **Large Objects** | `bytea` 之外的 Large Objects 不會複製 |
| **Sequence 值** | Sequence 的 last value 不會自動同步，需手動處理 |
| **TRUNCATE** | `TRUNCATE` 指令不會複製（PG 11-14）|
| **DCL 不複製** | `GRANT`、`REVOKE` 等權限控制語句不會同步到 Green |
| **Single-Threaded** | Logical Replication 的 apply process 是**單執行緒**的，高寫入量可能追不上，考慮改用 AWS DMS |
| **Unlogged Tables** | Unlogged tables 不寫 WAL，在 Logical Replication 下**不會被複製到 Green** |
| **Materialized Views** | Switchover 後 Green 的 Materialized Views **不會自動更新**，需手動 `REFRESH MATERIALIZED VIEW` |
| **Partitioned Tables** | 部署期間**不能**對 partitioned tables 建立新的 partition（`CREATE TABLE` 是 DDL，不會複製） |

### Extension 限制

以下 Extension 在 Logical Replication 時可能有問題：

| Extension | 限制 |
|-----------|------|
| **pg_partman** | 必須在 **Blue 和 Green 都**停用自動維護（不只是 Green） |
| **pg_cron** | 建議在 Green 環境停用排程任務 |
| **pglogical** | 不相容，必須移除 |
| **postgis** | Topology 相關功能可能有問題 |
| **pg_stat_statements** | 統計資料不會複製 |

### Post-Switchover 資源更新清單

Switchover 完成後，以下資源可能需要手動更新：

| 資源類型 | 需要更新的原因 |
|---------|---------------|
| **CloudWatch Alarms** | 可能引用舊的 Instance Identifier |
| **Lambda Functions** | 如果 hardcode 了 DB endpoint |
| **Parameter Store** | 儲存了舊的連線資訊 |
| **Secrets Manager** | 儲存了舊的連線資訊 |
| **IAM Policies** | 如果 policy 引用特定 Resource ARN |
| **EventBridge Rules** | 可能引用舊的 Instance |
| **外部監控系統** | 如 Datadog, New Relic 等 |

> **來源**: [Blue/Green Deployment Considerations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-considerations.html)

---

## 9. Best Practices

### 通用建議

#### Switchover 前檢查清單

- [ ] Replica Lag 接近 0
- [ ] 沒有長時間執行的 Query（建議 < 10 秒）
- [ ] 沒有正在執行的 DDL
- [ ] 應用程式有 Retry 機制（連線中斷時會自動重連）
- [ ] 選擇低流量時段
- [ ] 已在 Green 環境完成必要測試

#### 測試建議

1. **功能測試**：在 Green 環境執行應用程式的關鍵功能測試
2. **效能測試**：比較 Blue 和 Green 的查詢效能
3. **連線測試**：確認應用程式可以正確連線到 Green

### MySQL 最佳化設定

#### GTID 設定

如果有 Read Replica，強烈建議使用 GTID：

```
gtid_mode = ON
enforce_gtid_consistency = ON
```

#### Binary Log 最佳化

```
binlog_format = ROW
sync_binlog = 1  # 確保資料一致性
```

#### 降低 Replica Lag 的兩個手段

如果 Green 的 Replica Lag 太高，可以考慮以下暫時措施：

**手段一：調整 `innodb_flush_log_at_trx_commit`**

```
# 暫時設定（在 Green 環境）
innodb_flush_log_at_trx_commit = 2
```

- 減少每次 commit 的 fsync 次數，提升 replication apply 速度
- ⚠️ **風險**：如果中間 crash，可能有 1 秒的 data loss，需要重建 Green
- **Switchover 前務必改回 `1`**

**手段二：暫時改為 Single-AZ**

- 暫時把 Green 的 Multi-AZ 改成 Single-AZ
- 減少寫入延遲（不用同步到 Standby），提高 replication throughput
- **Switchover 前再改回 Multi-AZ**

### PostgreSQL 最佳化設定

#### WAL 設定（適用於 Logical Replication）

```
# 確保有足夠的 replication slots
max_replication_slots = 10  # 根據需求調整

# 確保有足夠的 WAL senders
max_wal_senders = 10

# Logical Replication 需要
wal_level = logical

# 增加 WAL 保留大小，避免複製中斷
# 建議值：1 TiB（如果有足夠 storage）
# PostgreSQL 14+
wal_keep_size = 1048576  # 1 TiB in MB
# PostgreSQL 13 以下（用 segments 計算，每個 segment 16 MB）
# wal_keep_segments = 65536

# 避免 WAL sender/receiver 意外 timeout 重啟
# Blue 環境設定：
wal_sender_timeout = 0     # 停用 timeout

# Green 環境設定：
# wal_receiver_timeout = 0  # 停用 timeout

# 增加 Logical Decoding 記憶體，減少 disk I/O
logical_decoding_work_mem = 256  # MB，預設 65 MB
```

#### 處理長時間交易

長時間交易是 Switchover 的最大敵人。建議：

1. **設定 statement_timeout**：
```sql
SET statement_timeout = '5min';
```

2. **監控長時間交易**：
```sql
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '30 seconds'
ORDER BY xact_start;
```

3. **必要時終止長時間交易**：
```sql
SELECT pg_terminate_backend(pid);
```

#### Sequence 同步（Logical Replication）

Switchover 前，需要手動同步 Sequence 值：

```sql
-- 在 Blue 環境取得 Sequence 值
SELECT sequence_name, last_value
FROM information_schema.sequences s
JOIN pg_sequences ps ON s.sequence_name = ps.sequencename;

-- 在 Green 環境設定
SELECT setval('your_sequence_name', <value_from_blue>);
```

#### Trigger 注意事項（Logical Replication）

如果 Green 環境有設定 `ENABLE REPLICA` 或 `ENABLE ALWAYS` 的 trigger：

- Replication 會把資料變更傳過來
- Trigger **也會**執行
- 結果：**重複執行**（原始操作 + trigger 各做一次）

**建議**：Switchover 前檢查並調整 Green 環境的 trigger 設定。

### DNS TTL 建議

> ⚠️ AWS 官方要求：DNS cache TTL **不能超過 5 秒**。

雖然 AWS 用 Endpoint 重新命名來實現切換，但應用程式和網路層的 DNS cache 仍然是最大的風險：

- **JVM（最常見的坑）**：Java 預設**永久 cache DNS**，必須設定 `networkaddress.cache.ttl=5`
- **Connection Pool**：設定合理的 `maxLifetime`
- **OS DNS Cache**：確認沒有設定過長的 cache TTL
- 確保應用程式有連線重試邏輯

> 詳細的 DNS cache 問題分析和修正方式，見[第 6 節：Endpoint 重新命名與 DNS](#endpoint-重新命名與-dns)。

### CloudWatch 監控指標

建議在部署期間持續監控以下指標：

| Metric | 說明 | 注意事項 |
|--------|------|---------|
| `ReplicaLag` | Blue 到 Green 的延遲 | Switchover 前必須接近 0 |
| `ReplicationSlotDiskUsage` | Replication slot 使用的磁碟空間 | 持續增長表示 Green 追不上 |
| `OldestReplicationSlotLag` | 最老的 replication slot 的 lag | 監控 logical replication 健康度 |
| `FreeableMemory` | 可用記憶體 | Logical decoding 會消耗額外記憶體 |
| `DatabaseConnections` | 連線數量 | Switchover 時會歸零再恢復 |
| `CPUUtilization` | CPU 使用率 | Replication 會增加 CPU 負載 |

> **來源**: [Blue/Green Deployment Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-best-practices.html)

---

## 10. IAM 授權

### 建立 Blue/Green Deployment 所需權限

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds:CreateBlueGreenDeployment"
            ],
            "Resource": [
                "arn:aws:rds:*:*:db:*",
                "arn:aws:rds:*:*:deployment:*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DescribeDBClusters"
            ],
            "Resource": "*"
        }
    ]
}
```

### Switchover 所需權限

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds:SwitchoverBlueGreenDeployment"
            ],
            "Resource": [
                "arn:aws:rds:*:*:deployment:*"
            ]
        }
    ]
}
```

### 刪除 Blue/Green Deployment 所需權限

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds:DeleteBlueGreenDeployment"
            ],
            "Resource": [
                "arn:aws:rds:*:*:deployment:*"
            ]
        }
    ]
}
```

### 完整管理權限

如果需要完整管理 Blue/Green Deployment：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "rds:CreateBlueGreenDeployment",
                "rds:DeleteBlueGreenDeployment",
                "rds:DescribeBlueGreenDeployments",
                "rds:SwitchoverBlueGreenDeployment"
            ],
            "Resource": "*"
        }
    ]
}
```

> **來源**: [Authorizing Access to Blue/Green Deployment Operations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-authorizing-access.html)

---

## 11. 成本考量

### 費用項目

| 項目 | 成本影響 |
|------|---------|
| **Green 環境的 Instance** | 額外的 EC2 費用（跟 Blue 相同規格）|
| **Green 環境的 Storage** | 額外的 EBS Storage 費用 |
| **Green 環境的 IOPS** | 如果使用 Provisioned IOPS |
| **Replication 流量** | 同 Region 通常免費 |
| **Backup Storage** | Green 環境的自動備份 |

### 成本最佳化建議

1. **縮短 Green 環境存在時間**：建立後儘快完成測試和 Switchover
2. **選擇適當時機**：避免在 Green 環境上執行不必要的工作負載
3. **及時清理**：Switchover 後，評估是否需要保留舊的 Blue 環境

### 費用估算範例

假設你的 Blue 環境是：
- db.r5.xlarge (4 vCPU, 32 GB RAM)
- 1000 GB gp3 Storage
- us-east-1 Region

Green 環境存在 7 天的額外費用約：
- EC2: ~$50
- Storage: ~$6
- 總計: ~$56

（實際費用請參考 AWS 官方定價）

---

## 附錄 A: RDS 內部帳號架構

### rdsadmin 帳號

`rdsadmin` 是 AWS 植入在**每個 RDS instance** 裡的內部管理帳號。你看不到它的完整權限，也無法用它登入。

#### RDS Instance 建立流程

```
你按下 "Create Database"
        │
        ▼
┌───────────────────────────────────────┐
│ 1. AWS 啟動一台 EC2（你看不到）         │
│ 2. 在上面安裝 MySQL/PostgreSQL         │
│ 3. 用真正的 root 建立 rdsadmin 帳號     │
│ 4. 鎖死真正的 root（不再讓任何人用）     │
│ 5. 建立你的 master user                 │
│ 6. 給 master user rds_superuser 角色   │
│ 7. 回傳 endpoint 給你                   │
└───────────────────────────────────────┘
```

### 權限層級架構

```
┌──────────────────────────┐
│  rdsadmin (AWS 內部)      │ ← 真正的 root/superuser，你看不到
│  有完整 SUPER 權限         │
├──────────────────────────┤
│  你的 master user          │ ← 你能用的最高權限
│  rds_superuser 角色        │ ← 不是真正的 SUPER/superuser
├──────────────────────────┤
│  一般應用程式帳號           │
└──────────────────────────┘
```

**核心概念**：RDS **不會**給你真正的 MySQL `root` 或 PostgreSQL `superuser`。你的 master user 有 `rds_superuser` 角色，但沒有 `SUPER` privilege。

### rdsadmin 的工作

| 工作 | 說明 |
|------|------|
| **自動備份** | 每天的 automated backup |
| **Monitoring** | 收集 metrics 送到 CloudWatch |
| **Maintenance** | OS patching、minor version upgrade |
| **Replication 管理** | Blue/Green 的 replication 設定 |
| **Parameter 套用** | 當你改 Parameter Group 時，rdsadmin 去執行 |
| **Read-only 控制** | Switchover 後設定 `read_only` |

### 為什麼 read_only 對你有效

#### MySQL 的兩層 Read-Only

| 參數 | 擋誰 | 說明 |
|------|------|------|
| `read_only = 1` | 擋一般用戶 | 有 SUPER 權限的用戶**不受影響** |
| `super_read_only = 1` | 擋所有人 | 連 SUPER 權限的用戶也不能寫 |

因為你的 master user **沒有 SUPER 權限**，所以 `read_only = 1` 就足以擋住你。

#### 你不能直接用 SQL 關掉 read_only

```sql
-- MySQL
SET GLOBAL read_only = 0;
-- ❌ ERROR: 你沒有 SUPER 權限

-- PostgreSQL
SET default_transaction_read_only = off;
-- ❌ 你不是 superuser
```

必須透過 Parameter Group 修改 + Reboot。

### MySQL Master User 沒有的權限

| 缺少的權限 | 影響 |
|-----------|------|
| `SUPER` | 不能設定全域變數、不能 kill 其他人的 connection |
| `FILE` | 不能直接讀寫 server 上的檔案 |
| `SHUTDOWN` | 不能關掉 database |

### MySQL Stored Procedures（替代 SUPER 功能）

AWS 提供了 stored procedures 來替代你缺少的 SUPER 權限：

```sql
-- 替代 KILL（因為你沒有 SUPER 權限）
CALL mysql.rds_kill(thread_id);

-- 替代直接設定 replication
CALL mysql.rds_set_external_source(...);
```

### PostgreSQL Master User 權限細節

```sql
-- rdsadmin 是真正的 superuser
SELECT usename, usesuper FROM pg_user WHERE usename = 'rdsadmin';
-- rdsadmin | true

-- 你的 master user 不是 superuser
SELECT usename, usesuper FROM pg_user WHERE usename = 'myuser';
-- myuser | false

-- 但你有 rds_superuser 角色
SELECT rolname FROM pg_roles WHERE oid IN (
    SELECT member FROM pg_auth_members WHERE roleid = (
        SELECT oid FROM pg_roles WHERE rolname = 'rds_superuser'
    )
);
-- myuser
```

**rds_superuser 不能做的事**：
- `CREATE EXTENSION` 任意 extension（只能裝 AWS 白名單內的）
- 直接操作 `pg_catalog`
- 修改 `rdsadmin` 的權限
- 存取 OS file system

---

## 12. 參考資料

### 官方文檔完整連結

| 主題 | 連結 |
|------|------|
| Overview | [blue-green-deployments-overview.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-overview.html) |
| Supported Regions and Engines | [Concepts.RDS_Fea_Regions_DB-eng.Feature.BlueGreenDeployments.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RDS_Fea_Regions_DB-eng.Feature.BlueGreenDeployments.html) |
| Creating | [blue-green-deployments-creating.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-creating.html) |
| Switching | [blue-green-deployments-switching.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-switching.html) |
| Deleting | [blue-green-deployments-deleting.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-deleting.html) |
| Considerations (Limitations) | [blue-green-deployments-considerations.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-considerations.html) |
| Best Practices | [blue-green-deployments-best-practices.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-best-practices.html) |
| PostgreSQL Replication Type | [blue-green-deployments-replication-type.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-replication-type.html) |
| IAM Authorization | [blue-green-deployments-authorizing-access.html](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-authorizing-access.html) |

### 相關資源

- [AWS RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)
- [AWS CLI rds Commands](https://docs.aws.amazon.com/cli/latest/reference/rds/)
- [AWS RDS Pricing](https://aws.amazon.com/rds/pricing/)
