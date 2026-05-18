# Ramulator 2.0 lab
This lab is based on [Ramulator2.0](https://github.com/CMU-SAFARI/ramulator2)


## 背景知識

### Ramulator 2.0 簡介

Ramulator 2.0 是由 CMU SAFARI Lab 開發的模組化、可擴展的 cycle-accurate DRAM 模擬器。它可以模擬不同的 DRAM 標準（DDR3/4/5、LPDDR5、GDDR6、HBM 等），並支援各種記憶體控制器排程策略與 RowHammer 緩解機制。

### 關鍵概念

| 概念 | 說明 |
|------|------|
| **Channel** | DRAM 通道，每個通道有獨立的資料匯流排 |
| **Rank** | 同一通道上共用資料匯流排的一組 DRAM 晶片 |
| **BankGroup / Bank** | DRAM 內部的儲存陣列單位，不同 bank 可平行操作 |
| **Row / Column** | Bank 內的列與欄，構成二維陣列 |
| **Row Buffer** | 每個 bank 有一個 row buffer，存放最近開啟的整列資料 |
| **Row Hit** | 存取的 row 已在 row buffer 中，不需重新開啟 |
| **Row Miss** | 存取的 row 不同於目前開啟的 row，需要先關閉再開啟 |
| **FRFCFS** | First-Ready First-Come-First-Served 排程策略，優先服務 row buffer 中已有的請求 |
| **Address Mapper** | 將物理位址映射到 channel/rank/bankgroup/bank/row/column |

---

## 環境建置

### 前置需求

- 作業系統：Linux（推薦 Ubuntu 20.04+）
- C++ 編譯器（需支援 C++20）： `g++-12` 或 `clang++-15`
- CMake： ≥ 3.14
- Python 3
- Git


### 步驟一：安裝 Conda

如果你的系統尚未安裝 conda，請執行以下指令安裝 Miniconda：

```bash
# 下載 Miniconda 安裝腳本
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# 執行安裝（依照提示操作）
bash Miniconda3-latest-Linux-x86_64.sh

# 重新載入 shell
source ~/.bashrc
```

### 步驟二：建立 Conda 虛擬環境

```bash
# 建立名為 ramulator2-env 的環境，指定 g++-12 與 CMake
conda create -n ramulator2-env -c conda-forge gxx_linux-64=12 cmake make -y

# 啟動環境
conda activate ramulator2-env
```



### 步驟三：驗證工具鏈

```bash
# 確認 g++ 版本（應顯示 12.x）
${CC} --version

# 確認 CMake 版本
cmake --version
```

### 步驟四：取得 Ramulator 2.0 原始碼

```bash
# 如果你還沒有原始碼
git clone https://github.com/CMU-SAFARI/ramulator2.git
cd ramulator2
```

### 步驟五：編譯 Ramulator 2.0

```bash
mkdir -p build && cd build

# 使用 conda 提供的編譯器
cmake .. -DCMAKE_C_COMPILER=${CC} -DCMAKE_CXX_COMPILER=${CXX}
make -j$(nproc)

# 將可執行檔複製到專案根目錄
cp ./ramulator2 ../ramulator2
cd ..
```

### 步驟六：驗證安裝

```bash
# 使用範例設定檔執行模擬
./ramulator2 -f example_config.yaml
```

如果看到類似以下的 YAML 輸出，表示安裝成功：

```yaml
Frontend:
  ...
MemorySystem:
  ...
```

---

## Lab 練習
需將助教提供的 lab 資料夾放到 Ramulator2 根目錄
---

### 練習 0

1. 使用提供的 baseline 設定檔執行模擬：

```bash
./ramulator2 -f lab/configs/baseline.yaml
```

2. 觀察輸出的 YAML 統計結果，記錄以下數值：
   - `row_hits_0`
   - `row_misses_0`
   - `row_conflicts_0`
   - `avg_read_latency_0`
   - `num_read_reqs_0`

3. **問題 0-1**：什麼是 row hit？它與 row miss 和 row conflict 有何不同？

---

### 練習 1：比較不同 DRAM 類型

在這個練習中，你將比較 DDR4 和 DDR5 的效能差異。

**步驟：**

1. 執行 DDR4 baseline 設定：

```bash
./ramulator2 -f lab/configs/baseline.yaml > lab/results_baseline.txt
```

2. 執行 DDR5 設定：

```bash
./ramulator2 -f lab/configs/exercise1_ddr5.yaml > lab/results_ddr5.txt
```

3. 比較兩者的輸出結果。

**問題：**

- **1-1**：DDR5 相較於 DDR4，`avg_read_latency_0` 有何變化？為什麼？
- **1-2**：查看兩個設定檔中的 `timing` 區塊，DDR4 使用的 timing preset 是什麼？DDR5 呢？它們的速率（rate）分別是多少 MT/s？
- **1-3**：DDR5 的 DRAM 層級結構中，bankgroup 和 bank 的數量分別是多少？與 DDR4 有何不同？

---

### 練習 2：觀察 Row Policy 的影響

Row Policy 決定了記憶體控制器在存取完一個 row 後的行為：
- **ClosedRowPolicy**：關閉目前開啟的 row，釋放 row buffer
- **OpenRowPolicy**：保持 row 開啟，若下次存取同一 row 則為 row hit

**步驟：**

1. 執行 ClosedRowPolicy（ baseline 設定，使用 `cap: 4`）：

```bash
./ramulator2 -f lab/configs/baseline.yaml > lab/results_closed.txt
```

2. 執行 OpenRowPolicy 設定：

```bash
./ramulator2 -f lab/configs/exercise2_openrow.yaml > lab/results_openrow.txt
```

**問題：**

- **2-1**：OpenRowPolicy 下的 `row_hits_0` 與 ClosedRowPolicy 相比如何？
- **2-2**：`avg_read_latency_0` 在兩種 policy 下有何差異？哪種 policy 對順序存取模式比較有利？
- **2-3**： baseline 設定中 `cap: 4` 參數代表什麼意思？（提示：查看 ClosedRowPolicy 的實作）

---

### 練習 3：修改 Channel 與 Rank 配置

在這個練習中，你將觀察不同的 channel/rank 配置對效能的影響。

**步驟：**

1. 執行 baseline 設定（1 channel, 2 ranks）：

```bash
./ramulator2 -f lab/configs/baseline.yaml > lab/results_1ch_2rk.txt
```

2. 執行 2 channel, 1 rank 設定：

```bash
./ramulator2 -f lab/configs/exercise3_multichannel.yaml > lab/results_2ch_1rk.txt
```

**問題：**

- **3-1**：兩種配置的 `avg_read_latency_0` 哪個較低？為什麼？
- **3-2**：多通道（multi-channel）配置的優點是什麼？（提示：想想位址映射）
- **3-3**：觀察兩種配置的 `row_hits_0` 和 `row_misses_0`，多通道配置的 row hit 率是否有變化？

---

### 練習 4：使用不同 Trace 模式觀察存取行為

**步驟：**

1. 修改 `lab/configs/baseline.yaml` 中的 `traces` 欄位，將路徑改為 `lab/traces/random.trace`：

```yaml
Frontend:
  ...
  traces: 
    - lab/traces/random.trace
```

2. 執行模擬並記錄結果：

```bash
./ramulator2 -f lab/configs/baseline.yaml > lab/results_random.txt
```

3. 將結果與順序存取（sequential）的結果進行比較。

**問題：**

- **4-1**：隨機存取模式下的 `row_hits_0` 與順序存取相比如何？
- **4-2**：隨機存取模式下的 `avg_read_latency_0` 是增加還是減少？為什麼？

---


## 提交要求

請將以下內容整理成一份報告（PDF 格式）：

1. **各練習的問題回答**：附上模擬輸出的關鍵數值截圖或表格。
2. **實驗觀察與心得**：你從這個實驗學到了什麼？不同 DRAM 設定如何影響效能？

