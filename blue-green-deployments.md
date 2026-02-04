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

建立 Blue/Green Deployment 時，AWS 會複製整個 Topology：

```
Blue (Production)              Green (Staging)
┌─────────────────┐            ┌─────────────────┐
│  Primary        │            │  Primary        │
│  (db-prod)      │───────────►│  (db-prod-green)│
└────────┬────────┘ Replication└─────────────────┘
         │                              │
         │                              │
┌────────▼────────┐            ┌────────▼────────┐
│  Read Replica   │            │  Read Replica   │
│  (db-read)      │            │  (db-read-green)│
└─────────────────┘            └─────────────────┘
```

重點：
- Primary **和** Read Replica 都會被複製
- Blue 到 Green 之間會自動建立 Replication
- Green 環境的 DB Instance 名稱會加上 `-green-<隨機字串>` 後綴

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

#### MySQL / MariaDB 特定
- 必須啟用 Binary Logging
- `binlog_format` 必須設為 `ROW`
- 如果有 Read Replica，必須使用 GTID-based replication

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

### 為什麼用 Endpoint 重新命名？

傳統做法是用 DNS 切換，但 DNS 有 TTL 的問題：
- 即使 TTL 設成 60 秒
- 有些 Client 會 cache 更久
- 導致部分 Client 還連到舊的 Server

AWS 的做法是直接改 **Instance Identifier**：
- `mydb-green-xxx` 直接變成 `mydb`
- Endpoint 格式是 `{instance-id}.xxx.{region}.rds.amazonaws.com`
- 所以 Endpoint 自動就變了

這樣就不用等 DNS TTL，可以做到秒級切換。

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

如果你有外部的 Read Replica（不在 Blue/Green Topology 內），Switchover 後需要手動更新它們的 Replication 設定，指向新的 Primary。

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
| **分區表限制** | 某些分區操作可能有問題 |

### Extension 限制

以下 Extension 在 Logical Replication 時可能有問題：

| Extension | 限制 |
|-----------|------|
| **pg_partman** | 必須在 Switchover 前停用自動維護 |
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

#### 效能調校

```
# 在測試環境可以考慮放寬此設定以提高效能
# 但在 Production 建議保持預設值 (1) 以確保資料安全
innodb_flush_log_at_trx_commit = 1
```

**注意**：`innodb_flush_log_at_trx_commit = 2` 可以提高複製效能，但會降低資料安全性，請謹慎評估。

### PostgreSQL 最佳化設定

#### WAL 設定（適用於 Logical Replication）

```
# 確保有足夠的 replication slots
max_replication_slots = 10  # 根據需求調整

# 確保有足夠的 WAL senders
max_wal_senders = 10

# Logical Replication 需要
wal_level = logical

# 增加 WAL 保留時間，避免複製中斷
wal_keep_size = 1024  # MB
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

### DNS TTL 建議

雖然 AWS 用 Endpoint 重新命名，但如果應用程式有自己的 DNS cache：

- 把 TTL 設短一點（如 60 秒）
- 或確保應用程式會遵守 DNS TTL
- 考慮在應用程式層實作連線重試邏輯

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
