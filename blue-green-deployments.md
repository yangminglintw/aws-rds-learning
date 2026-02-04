# AWS RDS Blue/Green Deployments 學習筆記

這篇筆記整理了 AWS RDS Blue/Green Deployments 的運作原理，主要是自己閱讀官方文件後的理解。

## 什麼是 Blue/Green Deployment？

簡單來說，就是建立一個跟 Production 一模一樣的 Staging 環境，在 Staging 做完變更、測試完成後，再把流量切過去。

| 環境 | 角色 |
|------|------|
| **Blue** | 目前的 Production 環境 |
| **Green** | 同步的 Staging 副本 |

### 為什麼需要它？

傳統的資料庫升級方式：

1. 預約維護時間（通常是半夜）
2. 停機
3. 備份
4. 升級
5. 測試
6. 如果失敗，rollback
7. 恢復服務

這種方式的問題：
- 停機時間長（可能 30 分鐘到數小時）
- 壓力大（只有一次機會）
- 很難完整測試（時間有限）

Blue/Green 的優點：
- 可以提前在 Green 環境測試
- Switchover 通常 **< 1 分鐘**
- 出問題可以快速切回 Blue

## 支援的資料庫引擎

| 引擎 | 支援狀態 | 備註 |
|------|---------|------|
| RDS for MySQL | ✅ | 5.7, 8.0 |
| RDS for MariaDB | ✅ | 10.4+ |
| RDS for PostgreSQL | ✅ | 11.21+, 12.16+, 13.12+, 14.9+, 15.4+ |
| Amazon Aurora | ❌ | 有獨立的 Blue/Green 功能 |
| RDS for Oracle | ❌ | 不支援 |
| RDS for SQL Server | ❌ | 不支援 |

## 運作原理

### Topology 複製

當你建立 Blue/Green Deployment 時，AWS 會複製整個 Topology：

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

### Replication 機制

不同引擎使用不同的 Replication 方式：

#### MySQL / MariaDB

使用 **Binary Log Replication**：
- 透過 binlog 複製所有變更
- Green 環境可以讀寫（但寫入會在 Switchover 時被捨棄）

#### PostgreSQL

有兩種模式：

| 模式 | 使用時機 | 特點 |
|------|---------|------|
| **Physical Replication** | Minor 版本升級 | Green 為唯讀，使用 WAL |
| **Logical Replication** | Major 版本升級 | Green 可讀寫，有較多限制 |

PostgreSQL Logical Replication 的限制比較多（後面會詳述）。

## Switchover 流程

這是 Blue/Green 最關鍵的部分。我自己理解的流程：

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
│     - db-prod-green  →  db-prod                                  │
│     - db-prod        →  db-prod-old1                             │
└─────────────────────────────────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 允許新連線                                                   │
│     - 應用程式連到原本的 endpoint，自動連到新的 Green             │
└─────────────────────────────────────────────────────────────────┘
```

### 我的理解：為什麼用 Endpoint 重新命名？

這是 AWS 最聰明的設計之一。

傳統做法是用 DNS 切換，但 DNS 有 TTL 的問題：
- 即使你把 TTL 設成 60 秒
- 有些 Client 會 cache 更久
- 導致部分 Client 還連到舊的 Server

AWS 的做法是直接改 **Instance Identifier**：
- `mydb-green-xxx` 直接變成 `mydb`
- Endpoint 格式是 `{instance-id}.xxx.{region}.rds.amazonaws.com`
- 所以 Endpoint 自動就變了

這樣就不用等 DNS TTL，可以做到秒級切換。

### Timeout 設定

可以設定 Switchover 的 Timeout：

| 設定值 | 說明 |
|--------|------|
| 最小值 | 30 秒 |
| 預設值 | 300 秒（5 分鐘）|
| 最大值 | 3600 秒（1 小時）|

如果在 Timeout 時間內無法完成同步，Switchover 會失敗並回滾。

### 影響 Switchover 時間的因素

- **長時間執行的 Transaction**：必須等它完成或 rollback
- **DDL 語句**：ALTER TABLE 這類可能會鎖表
- **連線數量**：連線越多，斷開越久
- **Replica Lag**：Lag 越大，等待時間越長

## 重要限制

### 功能限制

這些功能目前不支援 Blue/Green：

| 限制 | 影響 |
|------|------|
| RDS Proxy | 如果用了 RDS Proxy，無法使用 Blue/Green |
| Cross-Region Read Replica | 有跨區 Replica 的不能用 |
| CloudFormation | 無法用 CFN 管理 Blue/Green Deployment |
| Multi-AZ DB Cluster | 只支援 Multi-AZ DB Instance |
| Cascading Read Replica | 有串接 Replica 的不能用 |

另外：
- **不能改變加密狀態**：從加密變非加密（或反過來）不行
- **不能降級版本**：只能升級，不能從 8.0 降到 5.7

### PostgreSQL Logical Replication 特殊限制

這個比較麻煩，做 Major 版本升級時要特別注意：

| 限制 | 說明 |
|------|------|
| DDL 不複製 | `CREATE TABLE`, `ALTER TABLE` 等不會同步到 Green |
| 需要 Primary Key | 沒有 PK 的表無法執行 `UPDATE`/`DELETE` 複製 |
| Large Objects | `bytea` 之外的 Large Objects 不會複製 |
| 某些 Extension 必須停用 | `pg_partman`, `pg_cron`, `pglogical` |
| Sequence 值 | Sequence 的 last value 不會自動同步 |

> 我的理解：這些限制是因為 Logical Replication 的原理是複製「SQL 語句的結果」而不是「底層的資料變更」，所以有些操作無法正確複製。

## Best Practices

### Switchover 前檢查清單

根據官方建議，Switchover 前要確認：

- [ ] Replica Lag 接近 0
- [ ] 沒有長時間執行的 Query
- [ ] 沒有正在執行的 DDL
- [ ] 應用程式有 Retry 機制（連線中斷時會自動重連）
- [ ] 選擇低流量時段（雖然停機短，但還是會有短暫中斷）

### CloudWatch Metrics 監控

建議監控這些指標：

| Metric | 說明 |
|--------|------|
| `ReplicaLag` | Blue 到 Green 的延遲 |
| `DatabaseConnections` | 連線數量 |
| `CPUUtilization` | 確保 Green 沒有過載 |

### DNS TTL 建議

雖然 AWS 用 Endpoint 重新命名，但如果你的應用程式有自己的 DNS cache：

- 把 TTL 設短一點（如 60 秒）
- 或確保應用程式會遵守 DNS TTL

## 成本考量

AWS 文件沒有明確說定價，但可以推估：

| 項目 | 成本影響 |
|------|---------|
| Green 環境的 Instance | 額外的 EC2 費用（跟 Blue 相同規格）|
| Green 環境的 Storage | 額外的 Storage 費用 |
| Replication 流量 | 同 Region 通常免費 |

**建議**：建立 Green 環境後儘快完成測試和 Switchover，不要讓 Green 環境跑太久。

## 參考資料

- [AWS RDS Blue/Green Deployments Overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-overview.html)
- [Creating Blue/Green Deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-creating.html)
- [Switching a Blue/Green Deployment](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-switching.html)
- [Blue/Green Deployment Considerations](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments-considerations.html)
