# Ramulator 2.0 lab — 參考解答

---

## 練習 0

### 問題 0-1：什麼是 row hit？它與 row miss 和 row conflict 有何不同？

**答**：
- **Row Hit**：記憶體控制器收到的存取請求所指向的 row 已經在 row buffer 中被開啟。此時只需發出 RD/WR 命令即可，不需額外的 ACT（Activate）命令，延遲最低。
- **Row Miss**：目標 row 所在的 bank 目前沒有任何 row 被開啟（處於 Closed 狀態）。需要先發出 ACT 命令開啟 row，再發出 RD/WR 命令。
- **Row Conflict**：目標 row 所在的 bank 已有另一個 row 被開啟（但不是目標 row）。需要先發出 PRE（Precharge）命令關閉目前的 row，再發出 ACT 命令開啟目標 row，最後才能 RD/WR。延遲最高。

**延遲排序**：Row Hit < Row Miss < Row Conflict

---

## 練習 1：比較不同 DRAM 類型

### 參考數值（DDR4 vs DDR5）

| 指標 | DDR4 (2400R) | DDR5 (3200C) |
|------|-------------|-------------|
| `avg_read_latency_0` | ~450-500 cycles | ~600-700 cycles |
| `row_hits_0` | 有值（順序存取下較高） | 類似或略低 |
| `row_misses_0` | 有值 | 有值 |

### 問題 1-1：DDR5 相較於 DDR4，`avg_read_latency_0` 有何變化？為什麼？

**答**：DDR5 的平均讀取延遲（以 cycle 計）通常較高。原因：
1. DDR5 的時序參數（如 nCL、nRCD）以 cycle 計數通常較大。
2. DDR5 雖然傳輸速率更高（3200 MT/s vs 2400 MT/s），但延遲參數也相應增加。
3. DDR5 引入了 bankgroup 架構的變化（8 bankgroup × 2 bank vs DDR4 的 4 bankgroup × 4 bank），這可能影響 row buffer hit 率。

不過，若以實際時間（奈秒）計算，DDR5 因為更高的時鐘頻率，實際延遲可能相當甚至更低。

### 問題 1-2：DDR4 和 DDR5 的 timing preset 和速率

**答**：
- DDR4 使用 `DDR4_2400R`，速率為 **2400 MT/s**
- DDR5 使用 `DDR5_3200C`，速率為 **3200 MT/s**

### 問題 1-3：DDR5 的層級結構

**答**：
- DDR5 (`DDR5_8Gb_x8`)：**8 bankgroup，每個 bankgroup 2 個 bank**（共 16 banks）
- DDR4 (`DDR4_8Gb_x8`)：**4 bankgroup，每個 bankgroup 4 個 bank**（共 16 banks）

DDR5 增加了 bankgroup 數量但減少了每個 bankgroup 的 bank 數，這樣可以減少 bankgroup 間的時序限制，提高並行度。

---

## 練習 2：觀察 Row Policy 的影響

### 問題 2-1：OpenRowPolicy 下的 `row_hits_0` 與 ClosedRowPolicy 相比如何？

**答**：OpenRowPolicy 下的 `row_hits_0` **顯著高於** ClosedRowPolicy。因為 OpenRowPolicy 保持 row 開啟，當連續存取同一 row 的不同 column 時，每次都會是 row hit。

### 問題 2-2：`avg_read_latency_0` 的差異

**答**：對於順序存取模式（sequential access pattern），**OpenRowPolicy 的 `avg_read_latency_0` 較低**，因為順序存取傾向於存取相鄰的位址，這些位址很可能映射到同一個 row，從而產生大量的 row hits，減少了 ACT 和 PRE 命令的需要。

OpenRowPolicy 對順序存取模式特別有利。

### 問題 2-3：`cap: 4` 參數的意義

**答**：`cap: 4` 是 ClosedRowPolicy 的一個參數，表示 **同時保持開啟的 row 數量上限**。當同一個 rank 中開啟的 row 數量超過這個上限時，最少使用的 row 會被關閉（precharge）。這是一種折衷策略：
- `cap` 設得越大，越接近 OpenRowPolicy（更多 row 保持開啟）
- `cap` 設得越小（如 0），越接近嚴格的 ClosedRowPolicy（每次存取後立即關閉）

---

## 練習 3：修改 Channel 與 Rank 配置

### 參考數值

| 指標 | 1ch × 2rk | 2ch × 1rk |
|------|-----------|-----------|
| `avg_read_latency_0` | 較高 | 較低 |
| `row_hits_0` | 有值 | 可能較低 |

### 問題 3-1：哪種配置的 `avg_read_latency_0` 較低？

**答**：**2 channel × 1 rank 的配置 `avg_read_latency_0` 較低**。多通道可以提供更高的記憶體頻寬，兩個通道可以同時處理不同的請求，減少排隊等待時間。

### 問題 3-2：多通道配置的優點

**答**：多通道配置的主要優點是**提高記憶體頻寬**。在 RoBaRaCoCh 位址映射下，低位元的位址先映射到 column，再依序到 rank、bank、row，最後才是 channel。因此相鄰的存取會分散到不同 channel，讓兩個通道可以並行處理請求。這減少了每個通道的負載，降低了平均延遲。

### 問題 3-3：row hit 率的變化

**答**：在多通道配置下，由於相同的位址空間被分散到兩個通道，每個通道看到的存取序列可能不再是連續的 row 存取，因此 **row hit 率可能略有下降**。但整體效能（以延遲衡量）仍然提升，因為並行度增加了。

---

## 練習 4：使用不同 Trace 模式觀察存取行為

### 問題 4-1：隨機存取下的 `row_hits_0`

**答**：隨機存取模式下的 `row_hits_0` **大幅低於**順序存取。因為隨機存取的位址沒有空間局部性，每次存取很可能落在不同的 row，導致幾乎每次都是 row miss 或 row conflict。

### 問題 4-2：隨機存取下的 `avg_read_latency_0`

**答**：隨機存取模式下的 `avg_read_latency_0` **顯著增加**。原因：
1. 每次存取幾乎都需要 ACT（開啟新 row），增加了延遲。
2. 如果是 row conflict，還需要先 PRE（關閉目前 row），進一步增加延遲。
3. 排程器（FRFCFS）在隨機存取下幾乎無法利用 row buffer locality 來優化排程。

這個實驗說明了 **存取模式對 DRAM 效能的巨大影響**，也解釋了為什麼在實際應用中，提升空間局部性（例如使用 cache line、prefetching）對效能至關重要。