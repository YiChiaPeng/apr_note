# STA 筆記（Static Timing Analysis，靜態時序分析）

> 資料來源：投影片《靜態時序分析 Static Timing Analysis》（作者：于斌，`text_book/STA基本.pdf`，共 56 頁）。

> 本筆記下列專有名詞直接使用英文，不再翻譯成中文：**floorplan**、**script**、**place／placement**、**routing**。原投影片中的「布局」統一譯為 **placement**，「布线」統一譯為 **routing**。

---

## 縮寫對照表（全稱一覽）

| 縮寫 | 英文全稱 | 中文意義 |
|---|---|---|
| STA | Static Timing Analysis | 靜態時序分析 |
| RTL | Register Transfer Level | 暫存器傳輸層級 |
| DC | Design Compiler | Synopsys 的邏輯合成工具 |
| PT | PrimeTime | Synopsys 的靜態時序分析工具（本篇簡稱 PT） |
| DFT | Design For Test（ability） | 可測試性設計 |
| BIST | Built-In Self-Test | 內建自我測試（如 Memory BIST：記憶體內建自我測試） |
| JTAG | Joint Test Action Group | JTAG 邊界掃描測試標準介面 |
| LVS | Layout Versus Schematic | 版圖與電路圖比對驗證 |
| DRC | Design Rule Check | 設計規則檢查 |
| FF | Flip-Flop | 正反器（觸發器） |
| CT | Clock Tree | 時脈樹 |
| CTS | Clock Tree Synthesis | 時脈樹合成 |
| RAT | Required Arrival Time | 要求到達時間 |
| SDF | Standard Delay Format | 標準延遲格式檔案（記錄實際延遲數值） |
| FPGA | Field-Programmable Gate Array | 現場可程式化邏輯閘陣列 |
| ASIC | Application-Specific Integrated Circuit | 特殊應用積體電路 |
| PLL | Phase-Locked Loop | 相位鎖定迴路 |
| IC | Integrated Circuit | 積體電路 |
| tSU | setup time | 建立時間 |
| tH | hold time | 保持時間 |
| tCO | clock to output time | 時脈到輸出延遲 |
| tMET | metastability settling time | 介穩態到穩態的時間（settling time） |

---

## 概要（章節地圖）

1. 時序分析概述
2. 時序分析中的基本概念
3. 常用工具簡介

---

## 1. 時序分析概述

### 1.1 與時序驗證相關的完整後端流程

```
概念＋市場研究
  ↓
架構說明和 RTL 編碼 → RTL 模擬 → 邏輯合成、最佳化、掃描插入 → 形式驗證（RTL 和閘級）
  → Placement 前 STA
      ├─ 否（時序不正確）→ 回到「邏輯合成、最佳化、掃描插入」
      └─ 是 → Placement、CT 插入和全域 routing → 轉換時脈樹到 DC
                → 形式驗證（掃描插入的網表與 CT 插入的網表）
                → Placement 後 STA
                    ├─ 否 →（回饋修正）
                    └─ 是 → 詳細 routing → Routing 後 STA
                                ├─ 否 →（回饋修正）
                                └─ 是 → 結束
```

這個流程顯示時序驗證（STA）在整個後端設計中並非只做一次，而是在 **placement 前**、**placement 後（時脈樹插入後）**、**routing 後** 三個關鍵節點分別執行，每次都需要「時序正確」才能進入下一階段，否則要回饋修正前面的設計。

### 1.2 完整 ASIC 設計流程（23 步）

以下是投影片列出的較完整的數位 ASIC 設計流程步驟，涵蓋從 RTL 到交貨（tapeout）的所有階段：

1. 架構及電學特性規範
2. HDL 中的 RTL 編碼
3. 為包含記憶體單元的設計插入 DFT memory BIST
4. 為驗證設計功能，進行詳盡的動態模擬
5. 設計環境設定，包括將使用的製程庫和其他環境屬性
6. 使用 DC 對具有掃描插入（和可選 JTAG）的設計進行約束和合成設計
7. 使用 DC 的內建靜態時序分析機進行模組級的靜態時序分析
8. 設計的形式驗證，使用 Formality 將 RTL 和合成後的網表進行對比
9. 使用 PT 進行整個設計 placement 前的靜態時序分析
10. 對 placement 工具進行時序約束前的前標注
11. 具有時序驅動單元 placement、時脈樹插入和全域 routing 的初始 placement 劃分
12. 將時脈樹轉換到駐留在 DC 中的原始設計
13. 在 DC 中進行設計的 placement 最佳化
14. 使用 Formality 在合成網表和時脈樹插入的網表之間進行形式驗證
15. 在全域 routing 後（11 步）
16. 從全域 routing 得到的估計時間資料反標注到 PT
17. 使用全域 routing 後提取的估計延遲資料在 PT 中進行靜態時序分析
18. 設計的詳細 placement
19. 提取來自詳細 placement 設計的實際時間延遲
20. 實際提取時間資料反標注到 PT
21. 使用 PT 進行 placement 後的靜態時序分析
22. Placement 後的閘級功能模擬（如果需要的話）
23. 在 LVS 和 DRC 驗證之後交貨

> **筆記提醒**：原投影片第 18、19、21、22 步文字寫的是「布局」（placement），但依上下文（緊接在第 15～17 步「全域 routing 後」之後，且第 23 步已到交貨階段），這幾步實際上很可能指的是**詳細 routing 與 routing 後 STA**，而不是 placement。這可能是原投影片的用詞筆誤（「布局」與「布线」形近易混淆），閱讀時建議對照後面第 4 節「三階段 STA 的差異」（合成後／placement 後／routing 後）來理解真正的階段順序。

### 1.3 時序分析概述：與時序相關的流程

```
Design Entry → Synthesis → [Timing] → Place → [Timing] → Route → [Timing]
（每個 Timing 節點都可以回饋到 Design Entry 重新設計）
```

在上述每個 `Timing` 檢查節點，可以採用三種不同性質的驗證方式：
- **動態時序模擬**
- **靜態時序分析**
- **形式驗證**

### 1.4 動態時序模擬與靜態時序分析的差異

**動態時序模擬**是針對給定的模擬輸入訊號波形，模擬設計在元件實際工作時的功能和延遲情況，給出相應的模擬輸出訊號波形。它主要用於**驗證設計在元件實際延遲情況下的邏輯功能**。由動態時序模擬報告無法得到設計的各項時序效能指標，如最高時脈頻率等。

**靜態時序分析**則是透過分析每個時序路徑的延遲，計算出設計的各項時序效能指標，如最高時脈頻率、建立／保持時間等，發現時序違例。它僅僅聚焦於時序效能的分析，並不涉及設計的邏輯功能，**邏輯功能驗證仍需通過模擬或其他手段（如形式驗證等）進行**。靜態時序分析是最常用的分析、除錯時序效能的方法和工具。

### 1.5 靜態時序分析 STA

- STA 是一種**驗證方法**
- STA 的前提是**同步邏輯設計**
- STA 是使用工具透過路徑計算延遲的合成，並比較相對預定義時脈的延遲
- STA 僅**關注時序**間的相對關係，而**不是評估邏輯功能**
- 無需用向量去激活某個路徑，而是對所有的時序路徑進行錯誤分析，能處理百萬閘級的設計，分析速度比時序模擬工具快幾個數量級，在同步邏輯情況下，可以達到 100% 的時序路徑覆蓋
- STA 的目的是找出隱藏的時序問題，根據時序分析結果最佳化邏輯或約束條件，使設計達到**時序閉合（timing closure）**

### 1.6 STA 的作用

- **確定晶片最高工作頻率**：透過時序分析可以控制工程的合成、映射、placement／routing 等環節，減少延遲，從而盡可能提高工作頻率
- **檢查時序約束是否滿足**：可以透過時序分析來查看目標模組是否滿足約束，如不滿足，可以定位到不滿足約束的部分，並給出具體原因，進一步修改程序直至滿足時序要求
- **分析時脈品質**：時脈存在抖動（jitter）、偏移（skew）、工作週期（duty cycle）失真等不可避免的缺陷。透過時序分析可以驗證其對目標模組的影響

### 1.7 STA 的過程（三步驟）

1. 將設計打散成一個一個的 **timing path**（時序路徑）
2. 計算每條 path 的延遲
3. 檢驗延遲是否滿足設計約束的要求

---

## 2. 時序分析中的基本概念

本節涵蓋 5 個基本概念：**建立時間（setup time）**、**保持時間（hold time）**、**時脈到輸出延遲（clock to output time）**、**時脈偏移（clock skew）**、**時脈抖動（jitter）**。

### 2.1 建立時間 t_SU（setup time）

正反器（FF）的時脈訊號上升沿到來以前，資料穩定不變的時間。輸入訊號應提前時脈上升沿（假設上升沿有效）T 時間到達晶片，這個 T 就是建立時間 setup time。如不滿足 setup time，這個資料就不能被這一時脈打入正反器，只有在下一個時脈上升沿，資料才能被打入正反器。

```
Data:  ────────╳────────
                tSU  tH
Clock: ─────────────/‾‾‾
```
（`tSU` 為時脈上升沿前資料需穩定的時間，`tH` 為上升沿後資料需穩定的時間）

### 2.2 保持時間 t_H（hold time）

保持時間是指正反器的時脈訊號上升沿到來以後，資料穩定不變的時間。如果 hold time 不夠，資料同樣不能被打入正反器。

### 2.3 時脈到輸出延遲 t_CO（clock to output time）

從時脈訊號有效沿到資料有效的時間間隔。

### 2.4 介穩態（metastability）

不滿足建立／保持時間，可能出現介穩態（訊號在穩態之間震盪、無法立即穩定的狀態）：

```
DATA  ─────╲╲╲╲──────
CLK   ──────/‾‾‾‾‾‾‾‾
                  ↑ tSU、tH 不足
Q     ─────────（震盪）─── 穩定
              ← tCO →｜← tMET →
```
- `t_MET`（settling time）：介穩態到穩態的時間，**與製程無關**（是正反器電路結構固有的動態特性）

### 2.5 最小週期 T

```
INPUT → [FF1: D→Q, tCO] → (組合邏輯 tDELAY) → [FF2: D, tSU] → OUT
                              ↑ 兩者共用 CLK
```
```
T = t_CO + t_DELAY + t_SU
```
即：最小時脈週期 = 來源正反器的時脈到輸出延遲 ＋ 組合邏輯延遲 ＋ 目標正反器的建立時間。

### 2.6 時脈偏移（clock skew）

時脈偏移指的是同一個時脈訊號到達兩個不同暫存器之間的時間差值。時脈偏移永遠存在，到一定程度就會嚴重影響電路的時序：
```
FF1 --[interconnect and logic]--> FF2
clock at FF1: ‾‾‾╲___╱‾‾‾
clock at FF2:    ‾‾‾╲___╱‾‾‾   （延後到達，即 clock skew）
              ←── minimum clock period ──→
                                    ← skew →
```

### 2.7 時脈抖動（jitter）

所謂抖動，就是指兩個時脈週期之間存在的差值，這個誤差是在時脈產生器內部產生的，和晶體振盪器（crystal oscillator）或者 PLL（相位鎖定迴路）內部電路有關，**routing 對其沒有影響**。
```
jitter = T2 - T1
```
（T1、T2 為相鄰兩個時脈週期的實際長度）

### 2.8 時序路徑（4 種基本類型）

以 `IN → FF1(PRE/CLR) → FF2(PRE/CLR) → OUT` 的結構為例，時序路徑包括：
1. 從輸入端口到正反器的資料 D 端
2. 從正反器的時脈 clk 端到正反器的資料 D 端
3. 從正反器的時脈 clk 端到輸出端口
4. 從輸入端口到輸出端口

### 2.9 時序分析常用路徑（5 種命名路徑）

| 路徑名稱 | 英文 | 說明 |
|---|---|---|
| 時脈到建立 | clock to setup path | Clock → FF（經 logic and routing delay）→ 下一級 FF 的 setup |
| 時脈到接腳 | clock to pad path | Clock → FF → interconnect and logic → 輸出 pad（含 output delay、external margin） |
| 結束於時脈接腳 | paths ending at clock pin of flip-flops | Source（clock）→ interconnect and logic → FF 的 clock pin（即 clock path delay） |
| 接腳到接腳 | pad to pad | 輸入 pad → interconnect and logic（含 input/output delay）→ 輸出 pad |
| 接腳到建立 | pad to setup | 輸入 pad → interconnect and logic（含 input delay）→ FF 的 setup |

**路徑時延構成示意（時脈到建立）**：
```
clock ──╱‾‾‾╲____________╱‾‾‾
valid ──╳──╳──────────╳──╳──
        clock  logic and    flip-flop
        to     routing      setup
        output delay
        ←──── path delay ────→
        ←── minimum clock period ──→
```

### 2.10 關鍵路徑與時序最佳化方法

**關鍵路徑**通常是指同步邏輯電路中，組合邏輯時延最大的路徑。也就是說**關鍵路徑是對設計能起決定性影響的時序路徑**。

靜態時序分析可以找出邏輯電路的關鍵路徑，透過查看時序分析報告，可以確定關鍵路徑。

常用最佳化方法：**Retiming**、**Pipeline**

**Retiming（重新定時）**：在固定的 Clock Period（例如 10ns）下，將原本兩級不平衡的組合邏輯（例如 7.5ns + 11.0ns，其中 11.0ns 超過週期而違例）透過調整正反器在電路圖中的位置（把邏輯運算搬移到暫存器邊界的另一側），重新分配為兩級更均衡的延遲（例如 9.8ns + 9.0ns），在不改變整體邏輯功能與延遲總和的前提下消除單級超時問題。

**Pipeline（管線化）**：
```
單級組合邏輯（18.0ns）
  --Add Registers（插入暫存器切割成多級）-->
單級組合邏輯（18.0ns，但已分段，尚未重新平衡）
  --Retiming（重新定時，平衡各級延遲）-->
兩級管線：8.9ns ＋ 9.5ns
```
即先插入暫存器將長路徑分段，再透過 Retiming 平衡各段延遲，使每級時延都能滿足時脈週期要求，從而提高整體工作頻率。

---

## 3. 常用工具簡介

### 3.1 主流工具

- Synopsys 公司的 **PrimeTime** 主要用於全晶片的 IC 設計，PrimeTime 是業界最流行的分析工具
- 各 FPGA 廠商的工具均提供靜態時序分析功能，FPGA 的靜態時序分析比 IC 簡單

### 3.2 Timing Analyzer（Altera Quartus II 內建 STA 工具）

Altera 公司的 QuartusII 自帶的靜態時序分析工具，可以進行：
- **時序路徑的時延分析（Delay Matrix）**：可分析多個源和多個目標結點（node）之間的路徑傳播時延
- **建立／保持時間分析（Setup/Hold Matrix）**：計算輸入接腳到 DFF 的資料、時脈、時脈使能輸入端的最小要求建立、保持時間
- **同步邏輯效能（Registered Performance）**：分析同步邏輯，確定限制效能的時延、最小時脈週期、最大時脈頻率；實際上包含了各內部暫存器的建立、保持時間分析

輸出範例（Registered Performance）：`Clock period: 10.5ns, Frequency: 95.23MHz`

**Timing Analyzer Summary 報告範例**：

| Type | Actual Time | From | To |
|---|---|---|---|
| Worst-case tsu | 4.790 ns | reset | floating_cordic:f_core\|atan_ro... |
| Worst-case tco | 9.526 ns | ctl_cordic:ctl_core\|x_out[11] | x_out[11] |
| Worst-case th | -0.257 ns | reset | ctl_cordic:ctl_core\|bs... |
| Clock Setup: 'clk' | 22.49 MHz（period = 44.468 ns） | floating_cordic:f_core\|shift_right:shift_2\|tmp[4] | ctl_cordic:ctl_core\|xo[29] |

### 3.3 Synopsys PrimeTime（PT）

**3.3.1 PT 簡介**

- PrimeTime 是 Synopsys 的靜態時序分析工具，為業界標準，占據最大的市場份額
- PrimeTime 是數位 ASIC 設計的 sign-off 必選工具，受到所有 EDA 工具和 IC 廠家的支援
- FPGA 邏輯靜態時序分析，僅用到 PrimeTime 的一小部分功能

**3.3.2 Report 術語**

- **Arrival Time（信號到達時間）**：表示實際計算所得的訊號到達邏輯電路中某一點的絕對時間，等於訊號到達某條路徑起點的時間加上訊號在該條路徑上的邏輯單元間傳遞延遲的總和
- **Required Arrival Time（要求到達時間，簡稱 RAT）**：表示要求訊號在邏輯電路的某一特定點處的到達時間
- **Slack（裕度）**：表示在邏輯電路的某一特定點處要求到達時間與實際到達時間之間的差。Slack 值表示該訊號到達的太早或太晚

**3.3.3 PT 做 STA 的四步流程**

1. 讀入設計及庫
2. 約束設計
3. 指定延遲計算資訊
4. 靜態時序分析和報告

四步流程的展開細節：

1. **建立設計環境**
   - 建立搜索路徑（search path）和鏈接路徑（link path）
   - 讀入設計和庫
   - 鏈接頂層設計
   - 建立運作條件、連線負載模型、端口負載、驅動和傳輸時間
2. **說明時序聲明（約束）**
   - 定義時脈週期、波形、不確定性（uncertainty）和滯後時間（latency）
   - 說明輸入、輸出端口的延遲時間
3. **說明時序例外情況（timing exceptions）**
   - 多週期路徑（multicycle paths）
   - 不合法路徑（false paths）
   - 說明最大和最小延遲時間、路徑分割（path segmentation）和失效弧（disabled arcs）
4. **進行分析和生成報告**
   - 檢查時序
   - 生成約束報告
   - 生成路徑時序報告

**3.3.4 PT script 範例**

```tcl
set search_path ". $QUARTUS_ROOTDIR/eda/synopsys/primetime/lib"
set link_path {* alt_vtl.db apex20ke_asynch_mem_lib.db \
  apex20ke_lvds_receiver_lib.db \
  apex20ke_cam_lib.db apex20ke_lvds_transmitter_lib.db apex20ke_io_lib.db \
  apex20ke_pll_lib.db \
  apex20ke_lcell_lib.db apex20ke_pterm_lib.db}
read_verilog { $QUARTUS_ROOTDIR/eda/synopsys/primetime/lib/apex20ke_camslice_pt.v }
read_verilog { $QUARTUS_ROOTDIR/eda/synopsys/primetime/lib/apex20ke_ramslice_pt.v }
read_verilog snug_pt.vo
current_design snug
link_design snug
read_sdf snug_v.sdo
create_clock "CLK" -period 4 -waveform {0 2}

check_timing
report_analysis_coverage
report_timing
```

**3.3.5 建立時間檢查（Setup Check）**

```
Clock Delay 1 → [Comb1] → [Reg1] → [Comb2] → [Reg2, tsu] → [Comb3]
Clock Delay 2 ────────────────────↗
```
建立時間檢查公式：
```
clock delay1 - clock delay2 + max data path + t_SU ≤ clock period
```
其中 **Max data path** 是暫存器的 t_CO 加上暫存器間的組合邏輯延遲。

**範例計算**：
- clock delay1 = 0ns，clock delay2 = 0ns
- max data path = tco + path delay = 1.449ns + 0.258ns = 1.707ns
- 若 T = 4ns，則 slack = 4ns − 1.707ns = 2.293ns

**對應 PT 報告輸出格式**：
```
Startpoint: i_inst (rising edge-triggered flip-flop clocked by clk)
Endpoint: i_inst2 (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: max

Point                                    Incr      Path
--------------------------------------------------------
clock clk (rise edge)                    0.000     0.000
clock network delay (ideal)              0.000     0.000
i_inst/clk (apex20ke_lcell)              0.000     0.000 r
i_inst/regout (apex20ke_lcell)           1.449 *   1.449 r
i_inst2/datad (apex20ke_lcell)           0.258 *   1.707 r
data arrival time                                  1.707

clock clk (rise edge)                    4.000     4.000
clock network delay (ideal)              0.000     4.000
i_inst2/clk (apex20ke_lcell)             0.000     4.000 r
library setup time                       0.000     4.000
data required time                                 4.000
--------------------------------------------------------
data required time                                 4.000
data arrival time                                 -1.707
--------------------------------------------------------
slack (MET)                                        2.293
```
- **MET**：滿足建立時間
- **VIOLATED**：不滿足（違例）

**3.3.6 保持時間檢查（Hold Check）**

```
Clock Delay 1 → [Comb1] → [Reg1] → [Comb2] → [Reg2, thold] → [Comb3]
Clock Delay 2 ────────────────────↗
```
保持時間檢查公式：
```
clock delay1 - clock delay2 + min data path - t_H ≥ 0
```
其中 **min data path** 是暫存器的 t_CO 加上暫存器間的組合邏輯延遲（取最小延遲路徑）。

**範例計算**：
- clock delay1 = 0ns，clock delay2 = 0ns
- min data path = tco + path delay = 1.449ns + 0.258ns = 1.707ns
- intrinsic hold time = 1.284ns
- 則 slack = 1.707ns − 1.284ns = 0.423ns

**對應 PT 報告輸出格式**：
```
Startpoint: i_inst (rising edge-triggered flip-flop clocked by clk)
Endpoint: i_inst2 (rising edge-triggered flip-flop clocked by clk)
Path Group: clk
Path Type: min

Point                                    Incr      Path
--------------------------------------------------------
clock clk (rise edge)                    0.000     0.000
clock network delay (ideal)              0.000     0.000
i_inst/clk (apex20ke_lcell)              0.000     0.000 r
i_inst/regout (apex20ke_lcell)           1.449 *   1.449 r
i_inst2/datad (apex20ke_lcell)           0.258 *   1.707 r
data arrival time                                  1.707

clock clk (rise edge)                    0.000     0.000
clock network delay (ideal)              0.000     0.000
i_inst2/clk (apex20ke_lcell)             0.000     0.000 r
library hold time                        1.284 *   1.284
data required time                                 1.284
--------------------------------------------------------
data required time                                 1.284
data arrival time                                 -1.707
--------------------------------------------------------
slack (MET)                                        0.423
```

---

## 4. 三階段 STA 的差異（合成後／Placement 後／Routing 後）

> **問題**：三個階段的時序分析（`Synthesis→Timing`、`Place→Timing`、`Route→Timing`）有何不同？

**（1）合成後 STA**：
- 建立時間不符合 → 需重新設計（改邏輯結構/約束）
- 保持時間不符合 → 此處修改或 placement 後修改（根據違例大小決定）
- 採用的是**統計線負載模型**（wire load model，尚未有真實佈局資訊，只能用統計估算）
- 時脈扇出和時脈翻轉率是**固定假設值**（尚未生成真實時脈樹）

**（2）Placement 後 STA**：
- Placement 工具將關鍵單元彼此靠近放置，用以最小化路徑延遲
- 修改保持時間違例（或根據違例程度選擇 routing 後修改）
- 已插入了時脈樹（clock tree，CT），改變了原有設計（相比合成後階段，時脈網路已是真實結構而非理想／統計網路）

**（3）Routing 後 STA**：
- 加入寄生電容和 RC 連線延遲（真實提取的走線寄生參數，取代之前的估算模型）
- 修正保持時間（插入緩衝器）
- **最接近實際情況**（最精確的一次時序分析，通常作為簽核 sign-off 的依據）

三個階段的準確度遞增：合成後（統計估算）< placement 後（真實時脈樹＋估算走線）< routing 後（真實時脈樹＋真實走線寄生參數）。

---

## 5. 需要掌握的部分

依投影片總結，本主題需要掌握：
- 流程圖和相對應的文字說明
- 靜態時序分析的概念、目的和作用
- 建立／保持時間的概念和約束條件的計算
- PrimeTime 的基本過程

---

## 附：補充練習題（含完整解答）

### 題目

給定 setup time / hold time 的案例，要求算出最小時脈週期；也可以給定週期和 setup time 和 hold time，計算時間裕度（slack）。

**已知條件**：
- 時脈週期 = 20
- 每個正反器的 cell 延遲 = 1（即 t_CO）
- 正反器的建立時間 t_SU = 1
- 正反器的保持時間 t_H = 0.5

**電路結構**（FF1 → 多條組合邏輯路徑 → FF2）：
```
        ┌─[Logic7, Td7=2]─────────────────┐
        │                                  │
FF1(D,Q,CLK) ─┬─[Logic1,Td1=2]─[Logic2,Td2=3]─[Logic3,Td3=2]─┬─ FF2(D,Q,CLK)
              ├─[Logic4,Td4=4]─[Logic5,Td5=3]─[Logic6,Td6=1]─┤
              │                                               │
       [Logic8,Td8=2]（另一條回授/並聯路徑，接回 FF2 端）──────┘
```

**求**：圖中建立時間和保持時間的 slack。

### 解題步驟

**第一步：分析路徑，找出最長和最短路徑**

看到設計，首先要分析路徑，找出最長和最短路徑，因為 DC 的合成都是根據約束而得到最短和最長路徑來進行元件選擇的。接下來將圖中的所有路徑標出。因為沒有前級（`input_delay`）和後級電路（`output_delay`），我們只分析圖中給出的路徑。

**第二步：逐條路徑計算延遲（Td = Tcell + 各段組合邏輯延遲總和）**

| 路徑（顏色） | 組成 | 計算 | 延遲 Td |
|---|---|---|---|
| 紅色路徑 | FF1 → Logic4 → Logic5 → Logic6 → FF2 | Tcell+Td4+Td5+Td6 = 1+4+3+1 | **9** |
| 黃色路徑 | FF1 → Logic4 → Logic5 → Logic6 → Logic8 → FF2 | Tcell+Td4+Td5+Td6+Td8 = 1+4+3+1+2 | **11** |
| 紫色路徑 | FF1 → Logic1 → Logic2 → Logic3 → FF2 | Tcell+Td1+Td2+Td3 = 1+2+3+2 | **8** |
| 綠色路徑 | FF1 → Logic7 → Logic2 → Logic3 → FF2 | Tcell+Td7+Td2+Td3 = 1+2+3+2 | **8** |

**第三步：確定最長路徑與最短路徑**

- **T_longest = 11**（黃色路徑，含 Logic4→5→6→8）
- **T_shortest = 8**（紫色路徑或綠色路徑）

**第四步：計算 Slack**

- **建立時間 slack**：
  ```
  T_clk - T_longest - T_setup = 20 - 11 - 1 = 8
  ```
- **保持時間 slack**：
  ```
  T_shortest - T_hold = 8 - 0.5 = 7.5
  ```

兩者 slack 皆為正值（MET），表示此電路在給定的時脈週期 20 下，建立時間和保持時間都滿足約束，且分別還有 8 和 7.5 的餘裕空間。

---

## 附：實務案例——SDF 反標注與 Post-layout Simulation

> 出處：`~/Downloads/2025_Fall_Training_Package/3.Post_layout_Simulation/`（`APR/scripts/gcd_soce.tcl` 的 `write_sdf`；`post_sim/testbench.v`、`post_sim/sim.sh`）

本篇第 4 節提到「Routing 後 STA」是三階段中最接近實際情況的一次分析，因為此時已能提取真實的寄生電容與 RC 連線延遲。實務上，這份真實延遲資料會被輸出成一個 **SDF（Standard Delay Format）** 檔案（Cadence Innovus 指令為 `write_sdf`），交給 gate-level 的 post-layout 模擬使用：

```tcl
# gcd_soce.tcl（Step.9 Data Exports）
write_sdf -max_view func_mode_max -typ_view func_mode_max -min_view func_mode_min \
  -remashold -splitrecrem -recompute_delay_calc gcd.sdf
```

```verilog
// testbench.v
$sdf_annotate("../APR/run/gcd.sdf", u1);   // 把 SDF 的真實延遲反標注到閘級網表 u1
```

```bash
# sim.sh
vcs testbench.v ../APR/run/gcd_apr.v -v /usr/cadtool/.../tsmc090.v \
  -full64 -R -debug_access+all +v2k +neg_tchk
```

這個流程完整對應本篇 STA 三階段中的最後一步：**Routing 後 STA 產生的真實延遲數值，透過 SDF 反標注帶入 gate-level 模擬，驗證晶片在真實延遲下的功能與時序是否仍然正確**——`+neg_tchk` 選項的作用正是讓模擬器檢查是否出現負的 timing check（例如 hold time violation）。逐行指令說明詳見 `routing.md` 附錄。

**與本篇 3.3.4 節 PT script 的對照**：`write_sdf`（Innovus）與本篇 PT script 中的 `read_sdf snug_v.sdo` 是同一份 SDF 格式在不同工具間的一來一回——Innovus 用 `write_sdf` **輸出**自己算好的延遲，PrimeTime／模擬器再用 `read_sdf`／`$sdf_annotate` **讀入**別人算好的 SDF 來進行分析或模擬，兩者互為上下游。
