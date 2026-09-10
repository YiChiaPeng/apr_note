# Floorplan 筆記

> 資料來源：《數位積體電路後端設計》（田曉華主編，武漢理工大學出版社，2019）第 3 章「Floorplan 設計規劃」，頁 49–98，基於 Synopsys IC Compiler（積體電路編譯器，簡稱 ICC）。

> 本筆記下列專有名詞直接使用英文，不再翻譯成中文：**floorplan**、**script**、**place／placement**、**routing**。

---

## 縮寫對照表（全稱一覽）

| 縮寫 | 英文全稱 | 中文意義 |
|---|---|---|
| ICC | IC Compiler | Synopsys 的積體電路後端 placement／routing 工具 |
| DC / DCT | Design Compiler（Topographical） | Synopsys 的邏輯合成工具（DCT 為具拓撲感知能力版本） |
| RTL | Register Transfer Level | 暫存器傳輸層級（前端電路描述層級） |
| IP | Intellectual Property | 智慧財產核（可重複使用的電路模組） |
| SoC | System on Chip | 系統單晶片 |
| GUI | Graphical User Interface | 圖形化使用者介面 |
| MW | Milkyway | Synopsys 版圖資料庫格式（非嚴格縮寫，為資料庫名稱） |
| IO | Input/Output | 輸入輸出 |
| P/G | Power/Ground | 電源／接地 |
| tdf | Top Design File | 頂層設計檔案（定義接腳位置順序） |
| sdc | Synopsys Design Constraints | Synopsys 設計約束檔 |
| VFP | Virtual Flat Placement | 虛擬展開 placement |
| VIPO | Virtual In-Place Optimization | 虛擬原位最佳化 |
| IPO | In-Place Optimization | 原位最佳化 |
| PNS | Power Network Synthesis | 電源網路合成 |
| PNA | Power Network Analysis | 電源網路分析 |
| IR（drop） | Current（I）× Resistance（R） | 電流與電阻乘積，即電壓降 |
| EM | Electromigration | 電遷移 |
| DRC | Design Rule Check | 設計規則檢查 |
| ERC | Electrical Rule Check | 電氣規則檢查 |
| GRC | Global Routing Cell | 全域 routing 單元格（全域 routing 容量分析的最小網格單位） |
| DEF | Design Exchange Format | 設計交換格式（版圖交換檔案格式） |
| ddc | （Synopsys）Design Compiler database | Design Compiler 資料庫檔案格式 |
| CTS | Clock Tree Synthesis | 時脈樹合成 |
| QoR | Quality of Results | 結果品質 |
| AHFS | Automatic High-Fanout net Synthesis | 自動高扇出網路合成 |
| max_cap | Maximum Capacitance | 最大電容值（時序設計規則） |
| max_tran | Maximum Transition time | 最大電平轉換時間（時序設計規則） |

---

## 章節地圖

| 小節 | 主題 |
|---|---|
| 3.1 | Floorplan 原理及基本流程 |
| 3.2 | 初始化 floorplan |
| 3.3 | 虛擬展開 placement（VFP） |
| 3.4 | 減少壅塞 |
| 3.5 | 電源網路合成（PNS） |
| 3.6 | Floorplan 減少時間延遲 |
| 3.7 | Floorplan 設計輸出 |
| 3.8 | 小結 |

後端設計流程中，floorplan（design planning／floorplanning）銜接前端邏輯合成得到的閘級網表，與後續的 placement、時脈樹合成（Clock Tree Synthesis, CTS）、routing 相接。本章目標：理解 floorplan 原理、用 ICC 完成扁平化（非階層化）晶片級 floorplan，產生 routing 成功率高、時序可收斂的 floorplan 結果。

---

## 3.1 Floorplan 的原理及基本流程

### 3.1.1 Floorplan 定義

Floorplan＝規劃晶片內模組的**形狀和位置**，以最佳化：
- 晶片面積
- 晶片內 routing 總長度
- 關鍵路徑時間延遲
- routing 成功率
- 雜訊、散熱

晶片級資訊包括：
- **晶片核心區（core）**：晶片內部邏輯功能區，含尺寸、形狀、標準單元 placement 列（row，橫條排列）
- **外圍設備**：輸入輸出（IO）、電源、轉角接腳（corner pad）、填充單元位置
- **巨集（macro）placement**、標準單元 placement 約束（禁止 placement 區域）
- **電源網格**：電源環（ring）、條帶（strap）、軌道（rail）

> 後端設計（版圖設計）＝將合成後網表完成 **floorplan、placement、routing** 的過程。

### 3.1.2 Floorplan 問題的數學描述

- 輸入 n 個功能模組，面積分別為 A₁, A₂, …, Aₙ
- 每個模組 Bᵢ 指定高寬比上下限 rᵢ、sᵢ
- 求每個模組座標 (xᵢ, yᵢ)、寬 wᵢ、高 hᵢ，滿足：
  1. hᵢ·wᵢ = Aᵢ
  2. rᵢ ≤ hᵢ/wᵢ ≤ sᵢ
- 目標：最佳化電路效能

### 3.1.3 代價函數 F

```
F = A + W
```
- A：設計的最小 floorplan 矩形面積
- W：routing 總長，由「區塊間中心距離」與「區塊間連線數量」的乘積決定

```
W = Σ Pᵢⱼ · cᵢⱼ · dᵢⱼ
```
- cᵢⱼ：區塊 i、j 間訊號連線數量
- dᵢⱼ：區塊 i、j 中心距離
- Pᵢⱼ：區塊 i 與 j routing 長度的計算權重

模組級 floorplan 問題可用**切塊分割（slicing）**表達：V＝垂直分割，H＝水平分割；並以**切塊分割樹（slicing tree）**編碼（依從左到右、從下到上排列樹上節點），如 `21H67V45V1H3H1V`。

### 3.1.4 與設計風格相關的 Floorplan 特例

- 區塊尺寸固定／全訂製（full-custom）風格設計：**不需要** floorplan
- 標準單元 placement：尺寸固定，問題簡化為 placement 問題；大規模設計先分割為多個子區域，規劃各子區域形狀位置
- 閘陣列（gate array）設計：基本單元固定，規劃問題等同 placement 擺放問題

### 3.1.5 積體電路發展趨勢與 Floorplan 決策方法

| 積體電路發展趨勢 | 決策方法 |
|---|---|
| 訊號連線延遲增加 | 時序與壅塞驅動的 floorplan |
| 需考慮連線間壅塞程度 | 於邏輯階層與物理階層間做取捨最佳化 |
| 積集度增加、IP 核數量增加、IP 間互連增加 | 大型設計劃分為階層式模組，實現模組劃分與接腳（Pin）分配 |
| 時脈網路數量增加 | 時脈樹合成與最佳化 |
| 需更低功耗 | 採用先進的低功耗設計 |

### 3.1.6 ICC 設計規劃方法（總流程）

1. 設定 Milkyway（MW）資料庫，定義 floorplan 約束條件
2. 對設計原型分析時序、routing 成功率、功耗、積集度等因素
3. 透過具體 floorplan 實現更佳化的估算指標
4. 達到合理預期後，進行後續 ICC 具體設計步驟

（步驟②③構成疊代回饋：③更新規劃後回到②重新估算指標，最佳化後才進行④）

### 3.1.7 設計規劃流程五大階段（圖 3-7）

1. **設計的設定階段**：`create_mv_lib`、`import_designs`、`read_sdc`、`read_io_constraints`
2. **Floorplan 階段**：`connect_pg_nets`、`initialize_floorplan`
3. **虛擬展開 placement VFP（Virtual Flat Placement）**：`create_fp_placement`
4. **電源網路合成與分析（PNS/PNA，Power Network Synthesis / Power Network Analysis）**：`set_fp_rail_constraints`、`synthesize_fp_rail`、`commit_fp_rail`、`analyze_fp_rail`
5. **Floorplan 階段原型 routing 及時序分析**：`route_fp_timing`、`extract_rc`、`optimize_fp_timing`、`report_constraint -all`、`report_timing`

簡化版（圖 3-8）：`Synthesize 合成` → `DesignSetup 資料設定` → `建立初始 floorplan` → `placement：擺放巨集與標準單元（VFP）` → `分析並最佳化調整巨集位置` ⇄（右側）`電源網路合成與分析（PNS/PNA）` → `原型 routing` → `floorplan：原位最佳化 IPO` → `Placement 最佳化`

### 3.1.8 Floorplan 結合重新合成流程（圖 3-9）

```
RTL 合成（DCT，採用預設 floorplan）→ ICC 資料設定
  → 設計 floorplan：建立初始 floorplan → 虛擬展開 placement VFP → 減少壅塞 → 產生電源網路PNS → 減少延遲 → 輸出DEF floorplan 檔案
    →（提升 QoR 流程，Quality of Results）DCT 利用 DEF floorplan 檔案重新合成 → ICC 資料設定導入重新合成的網表 → 讀入DEF檔案 → placement
```
ICC placement 與 DCT 合成的聯合最佳化，利用了 floorplan 的**拓撲（topology）**資訊，二次合成結果與後端電路結構的擬合度更好。

### 3.1.9 模組級 Floorplan 的初始化步驟

1. 讀入 IO 約束
2. 定義核心區域，擺放 IO 接腳
3. 定義 floorplan 的**直角多邊形（rectilinear）**區域（模板：L 形、T 形、U 形、十字形）
4. 寫入 IO 約束資訊
5. 新增單元列，進行 VFP 及電路效能分析
6. 移除單元列
7. 保存 floorplan 的初始資訊

### 3.1.10 晶片級的建立初始 Floorplan 步驟

1. 建立僅在物理設計使用的接腳（pad cells）
2. 指定接腳單元的位置
3. 初始化設計 floorplan
4. 在接腳之間新增接腳填充單元（pad filler cells）
5. 建立接腳的電源／接地環（P/G pad rings）
6. 指定忽略的 routing 層
7. 定義已知（確定）的巨集和標準單元 placement
8. 定義已知的禁止 placement 區域（placement blockage）

### 3.1.11 GUI：選擇設計 Floorplan Task

ICC 檔案選單 Task 可切換介面風格：`Design Planning`／`Block Implementation`／`All Tasks`（預設為 Block Implementation）。

```tcl
gui_set_current_task -name {Design Planning}
```

開啟前一章資料設定完成的設計單元，並重新套用時序最佳化控制 script：

```tcl
open_mw_cel DESIGN_data_setup
source tim_opt_ctrl.tcl
```

### 3.1.12 建立僅用於物理設計的晶片接腳單元

DC 合成得到的閘級網表**沒有**定義電源、接地接腳，也沒有晶片外圍轉角接腳。這些僅用於物理設計的接腳需在指定訊號接腳單元位置前先定義，用 `create_cell`：

```tcl
create_cell {vss_l vss_r vss_t vss_b} pv0i
create_cell {vdd_l vdd_r vdd_t vdd_b} pv0i
create_cell {CornerLL CornerLR CornerTR CornerTL} pfrelr
```
- `pv0i`：電源／接地接腳單元
- `pfrelr`：晶片 4 個轉角的接腳單元
- 接腳新增順序一般為：電源 → 轉角 → 訊號腳

### 3.1.13 基於 tdf 檔案設定晶片接腳順序

`tdf`（Top Design File，頂層設計檔案）用來定義**接腳位置**，配置晶片接腳名稱與位置的對應關係。

```tcl
; tdf 檔案
; 放置接腳單元
pad "cornerLL" "bottom"
pad "cornerLR" "right"
pad "cornerTR" "top"
pad "cornerTL" "left"
; 放置 io 和电源接腳
; 左右边的接腳编号从下往上顺序编号（除了转角的接腳）
pad "pad_data_0" "left" 1
pad "pad_data_1" "left" 2
pad "VDD_LEFT"   "left" 3
pad "VSS_LEFT"   "left" 4
pad "pad_data_2" "left" 5
; 下边和上边的接腳编号从左往右顺序编号（除了转角的接腳）
pad "Clk" "bottom" 1
pad "A_0" "bottom" 2
pad "A_1" "bottom" 3
```

格式：`pad padName padSide [padOrder] [padOffset] ["reflect"]`
- `padName`：接腳名稱
- `padSide`：接腳所在邊（bottom/top/left/right）
- `padOrder`：接腳在所在邊的序號（轉角接腳不需序號，位於所在邊頂端）
- `padOffset`：接腳相對參考邊的絕對偏移量（微米，可選；若未設定，接腳預設在每邊均勻排列）

排列規則：top/bottom 從左往右遞增，left/right 從下往上遞增；同一邊編號不需連續（可留空位置給不限定位置的接腳）。

完成 tdf 後導入 ICC：
```tcl
read_io_constraints [-append] [-cel_name] [-child_cel] < TDF_file>
```

### 3.1.14 基於 tcl 指令設定晶片接腳順序（新版 ICC，可取代 tdf 檔案）

```tcl
set_pad_physical_constraints [-pad_name][-side][-order]
```
- `side`：1＝左, 2＝上, 3＝右, 4＝下（轉角接腳分別屬於 side1/2/3/4：左上、右上、右下、左下）
- `order`：接腳在所在邊序號，轉角接腳不需 order

範例（對應上面 tdf 例子的 cornerLL、pad_data_0）：
```tcl
set_pad_physical_constraints -pad_name "cornerLL" -side 4
set_pad_physical_constraints -pad_name "pad_data_0" -side 1 -order 1
```
可將多條指令彙整為一個 tcl script 檔一次呼叫：`source - echo pads.tcl`

---

## 3.2 初始化 Floorplan

完成後定義晶片基本形態，主要完成：
- 建立晶片**核心區域**和**外圍（外設）區域**（核心區放巨集、標準單元；外圍區放接腳）
- 定義核心區的標準單元 placement 列屬性、晶片邊界／外圍區設定（核心使用率、長寬比、核心到接腳的距離等）
- 依物理約束（tdf 或接腳約束 tcl）放置 IO 接腳（含網表定義的接腳與物理設計建立的接腳）

### 3.2.1 初始化指令

- `initialize_floorplan`：常規矩形核心區
- `initialize_rectilinear_block`：直角多邊形規劃區域

GUI 主要選項（Initialize Floorplan 對話框，Control type）：
- **Aspect ratio**（比例控制）：Core utilization（核心使用率）、Row/core ratio、Aspect ratio (H/W)、Core to left/right/top/bottom（核心到接腳距離）
- **Width and height**：直接指定核心寬高
- **Row number**／**Boundary**：其他控制方式

排列選項：
- `Horizontal row`：標準單元列方向是否水平
- `Double back`：相鄰列是否翻轉、背靠背放置
- `Flip first row`：第一列是否上下翻轉

### 3.2.2 晶片核心（core）區域參數設定

兩種可選配置：
1. **比例控制**：核心使用率（依網表巨集＋單元總面積及使用率估算核心區面積）、高／寬比（控制晶片外形）、標準單元列（row）／核心區面積比（體現使用率）
2. **指定高度和寬度控制**：晶片核心具體寬高尺寸、核心到各邊接腳的距離

### 3.2.3 初始化後的設計 Floorplan

- 外圍 IO 接腳已按約束排列
- 尚未完成 placement 的巨集在晶片上方排列（佔位）
- 尚未完成 placement 的標準單元在晶片區域外側排列
- 核心區域內可見標準單元列（site rows）的位置定義

### 3.2.4 插入接腳填充單元（filler cell）

作用：接腳之間可能存在間隙，填充單元使 N 阱、P 阱、P/G 電源環保持**連續性**，讓訊號接腳功能正常並取得供電。

```tcl
insert_pad_filler -cell "fill5000 fill2000 fill1000 ..."
```
- `-cell`：指定填充單元名稱列表（如 fill5000、fill2000…，數字表示單元寬度）
- 指令依寬度**由大到小**排列填充，優先選寬度大的單元 → 使填充單元數量最少

GUI 介面可指定：插入單元名稱、插入區域、邊（Top/Bottom/Left/Right）。

### 3.2.5 建立接腳的電源與接地（P/G）電源環

兩步：
1. 先建立 pad 的電源腳（電源、接地各多個）到電源網路（net）的**邏輯連接**（虛線）
2. 再用 `create_pad_rings` 完成電源環的**物理連接**（即完成電源環 routing）

```tcl
derive_pg_connection -power_net VDD  -power_pin VDD  -ground_net VSS  -ground_pin VSS
derive_pg_connection -power_net VDDO -power_pin VDDO -ground_net VSSO -ground_pin VSSO
derive_pg_connection -power_net VDDQ -power_pin VDDQ -ground_net VSSQ -ground_pin VSSQ
derive_pg_connection -power_net PWR  -ground_net GND  -tie
create_pad_rings
```
- `-tie` 選項：將單元固定接電源或接地的**非電源腳**連接到相應電源網路（確保固定電平的 IO 端口正確接到輸入電平），對應 GUI「Reconnect existing tie pins to appropriate power nets」

### 3.2.6 執行虛擬展開 Placement VFP 前的步驟

VFP（Virtual Flat Placement，虛擬展開 placement，指令 `create_fp_placement`）用於判斷 floorplan 是否存在潛在 routing 壅塞（congestion）風險：
- 預設：VFP 在 floorplan 階段嘗試性 place 標準單元和不固定的巨集
- placement 位置由 ICC 工具自動安排
- 假設製程所有 routing 層都可用

VFP 前可先明確非預設約束：
- 指定不用的 routing 層
- 定義巨集與標準單元的位置
- 設定已知的禁止 placement 區域

### 3.2.7 忽略不用的 Routing 層

ICC 預設使用所有金屬 routing 層。若計劃使用較少的層 routing（減少使用高層金屬層、降低製造成本），但沒有明確指明，會造成 routing 前階段的估算利用了更多金屬層，對壅塞的分析過於樂觀；RC 寄生參數不准，routing 時間延遲的計算偏差較大。

```tcl
set_ignored_layers -max_routing_layer M7   # 只允許最高到 M7 routing
report_ignored_layers   # 查看
remove_ignored_layers   # 移除
```

### 3.2.8 限制巨集（macro）

VFP（`create_fp_placement`）執行後，巨集擺放位置可能不合理，造成電源網路結構複雜或需要更多匯流排 routing 資源。可在 VFP 前手動在 GUI place 巨集，並設定約束：

```tcl
set_fp_macro_options ...    # 設定巨集 placement 選項
set_fp_macro_array ...      # 設定巨集陣列，集中排列關聯密切的巨集
set_fp_relative_location ...# 設定關聯 placement，指定一個單元相對於錨點 anchor 的相對位置
```
> 此階段巨集 placement 約束是**軟性（soft）約束**：後續設計中盡量遵守約束，但並非必須遵守。

**巨集陣列** `set_fp_macro_array`：
```tcl
set_fp_macro_array -name A_array -elements \
  [list [get_cells A1 A2] [get_cells A3 A4]] \
  -x_offset 15 -y_offset 110
```
支援 1D 陣列（如 1×4）、2D 陣列（如 2×3、3×2）。VFP 中，巨集陣列被視為單一單元物件，因此不會在陣列巨集之間插入標準單元。

**巨集 Placement 控制** `set_fp_macro_options`：
```tcl
set_fp_macro_options collection_of_cells \
  -anchor_bound < l, r, t, b, bl, tl, br, tr, bm, tm, lm, rm, c>
```
- `legal_orientations`：控制巨集 placement 方向
- `anchor_bound`：控制巨集限定在核心區指定的 1/2 或 1/4 區域 placement（l/r/t/b＝邊、bl/tl/br/tr＝角、bm/tm/lm/rm＝邊中段、c＝中心）
- `side_channel`：指定巨集或巨集組到核心區四邊的距離

**巨集對齊**（GUI 或 tcl）：邊沿對齊、接腳對齊、對齊到設計核心區邊界；可設定巨集距核心區偏移量。

### 3.2.9 硬巨集／軟巨集

- **硬巨集（hard macro）**：限定矩形區域邊界內、接腳固定且內部電路固化的 IP 核（如 SoC 主要組件）；集成困難、內部電路無法靈活調整（如 resizing 調整元件尺寸）
- **軟巨集（soft macro）**：邏輯功能相對獨立，物理設計屬於特定階層，沒有固定形狀，可更全域最佳化，便於解決時序、面積約束問題。可透過參數化約束限定其在核心區的具體 placement 區域

### 3.2.10 巨集 Placement 約束設定（`set_fp_macro_placement_constraint`）

巨集約束不能太多，常用約束包括：
- 巨集按邊沿對齊
- 巨集分組
- 巨集靠晶片核心區邊沿擺放

```tcl
set_fp_macro_placement_constraint -clear_all
set_fp_macro_placement_constraint -edge r [get_cells *RAM*]   # RAM 巨集靠右邊沿 placement
```

### 3.2.11 手動巨集 Placement

GUI 工具列可：固定、移動、旋轉、翻轉（含鏡像）、對齊、分布、展開、顯示飛線和網路連線。

```tcl
set_dont_touch_placement [get_cells <list_of_cells>]     # 設定固定單元（避免 create_fp_placement 移動位置）
remove_dont_touch_placement [get_cells <list_of_cells>]   # 解除固定
```

**分布（distribute）**：相對一個邊界的多個巨集均勻排列（可相對 4 個方向邊界）。
**展開（spread）**：在一個設定矩形區域內均勻排列巨集（可選水平／垂直方向）。

**對齊和間距設定**：對於多個密集訊號連接的巨集，若排列不合理會造成大量訊號線交叉重疊，影響 placement 與 routing。可用「中線對齊」或「區塊間偏移量均勻分布」改善壅塞。

**飛線／網路連線顯示**：可視化巨集與其他巨集的連接關係與連線數量，便於 placement 調整。

### 3.2.12 巨集周邊潛在壅塞問題

放在巨集周邊的標準單元 routing 可能遇到困難（走線空間狹窄）。解決方法：設定巨集周邊禁止 routing 區：
- **硬性禁止（hard blockage）**：標準單元不會放在限制區
- **軟性禁止（soft blockage）**：在初始的初步 placement 階段禁止放置標準單元，但後續 placement 和最佳化操作中該禁止區可以被忽略（不再強制有效）

### 3.2.13 巨集與標準單元的擺放原則：連貫性（consistency）

- RAM 要保持到晶片核心邊界一定距離
- 巨集之間要留下較大的 routing 通道
- 轉角位置不放置模組接腳（該區域訊號連線密集、不適合出線）
- 大塊區域留作標準單元 placement
- 避免巨集的密集接腳靠近邊沿區，影響訊號線出線
- 適當情況下旋轉巨集（順時針或逆時針旋轉 90°）可提高連通性
- 避免狹小的 routing 通道
- 巨集周邊設定禁止區（blockage），以便 routing 到巨集接腳

### 3.2.14 禁止 Placement 區域設定

**（1）全域（global）禁止 placement 設定**（透過變數，需搭配 `.synopsys_dc.setup` 檔案自動載入）：
```tcl
set_app_var physopt_hard_keepout_distance 10     # 所有固定巨集四周硬性禁止區距離
set_app_var placer_soft_keepout_channel_width 25  # 巨集間／巨集到核心區邊緣的軟性禁止通道寬度
```

**（2）巨集特定（specific）禁止 placement 設定**：`set_keepout_margin`
```tcl
set_keepout_margin -type hard -outer {10 0 10 0} RAM5   # {左 下 右 上} 寬度
report_keepout_margin   # 報告
remove_keepout_margin   # 移除
```
`-type` 可選 `soft` 或 `hard`。

### 3.2.15 Floorplan 階段建立 Routing 指導（route guide）

巨集上方特定金屬層禁止 routing 可防止訊號干擾並便於 routing。GUI：`Floorplan → Create Route Guide…`

```tcl
create_route_guide -no_signal_layer {METAL5 METAL6} \
  -coordinate {{20 20} {75 95}}
```
禁止在指定座標範圍內、指定金屬層 routing。

### 3.2.16 建立初始 Floorplan 小結（流程）

1. 建立僅用於物理設計的接腳
2. 指定接腳單元位置
3. 初始化設計 floorplan
4. 接腳之間新增填充單元
5. 建立接腳的電源／接地環
6. 指定忽略 routing 層
7. 定義已知巨集和標準單元 placement
8. 定義已知禁止 placement 區域

```tcl
open_mw_cel DESIGN_data_setup
create_cell ...
read_io_constraints < TDF_file>          # 或 set_pad_physical_constraints ...
initialize_floorplan ...
insert_pad_filler ...
derive_pg_connection ...
create_pad_rings ...
set_ignored_layers -max M7
set_dont_touch_placement [all_macro_cells]
set_fp_macro_options ...
set_fp_macro_array ...
set_fp_relative_location ...
set_app_var physopt_hard_keepout_distance <#>
set_app_var placer_soft_keepout_channel_width <#>
set_keepout_margin ...
```

---

## 3.3 虛擬展開 Placement VFP（Virtual Flat Placement）

### 3.3.1 概念

VFP 是 floorplan 的**第二步設計**，包含兩步操作：設定 placement 策略參數 → 進行虛擬展開 placement。特點：完成快速，同時最佳化 routing 長度和時序、減少壅塞。

主要設計步驟：
1. 評估硬巨集的擺放位置
2. 約束硬巨集
3. 硬巨集的保護區設定
4. Place 硬巨集和標準單元（即完成 VFP）
5. Floorplan 編輯（調整最佳化）

### 3.3.2 巨集 Placement 約束（回顧）

- 按邊沿對齊
- 巨集分組為標準單元預留盡量大的矩形區域，便於 placement 與 routing
- 巨集與核心區的邊沿對齊擺放

### 3.3.3 硬巨集擺放位置評估

沒有直接評判標準，通常是**主觀評價**，依賴設計經驗：
- 觀察基本原則（巨集靠邊、相似功能關聯的巨集對齊等）
- 用時序結果和 routability（可 routing 通過性）判斷巨集設定結果
- QoR（Quality of Results，結果品質）一般包括：routability、時序、routing 長度、資料流、標準單元的擺放區域

### 3.3.4 設定硬巨集 Placement 的策略

主要包括：
- 建立使用者定義的硬巨集陣列
- 設定 floorplan 的巨集單元 placement 約束
- 相對參考物放置巨集單元
- 套用 VFP 策略
- 使用巨集靠邊選項，提升 VFP 效能
- 設定巨集禁止 routing 區
- 設定巨集的 padding 保護區域

**Placement 策略示例（圖 3-35）**：巨集靠晶片核心區邊沿擺放 `macros_on_edge`，並開啟巨集自動分組 `auto_grouping` 可減少壅塞；相關巨集組成分組後可按陣列形式擺放，提高時序效能，減少壅塞。

### 3.3.5 分析和調整巨集 Placement 流程（疊代，圖 3-36）

```
擺放巨集以及標準單元
   ↓
時序和壅塞結果是否可以接受？──否→ 修改和新增巨集 placement 策略 ──┐
   ↓ 是                                                      │
執行 placement 後的最佳化                                       │
（回到「擺放巨集」重新疊代 ←──────────────────────────────────┘）
```

### 3.3.6 虛擬展開 Placement 策略設定：`set_fp_placement_strategy`

```tcl
set_fp_placement_strategy
    -macro_orientation automatic | all | N
    -auto_grouping none | user_only | low | high
    -macro_setup_only on | off
    -macros_on_edge on | off                  # 巨集靠邊擺放
    -sliver_size <0.00>                       # 巨集間不放置單元的條帶 sliver 間距
    -snap_macros_to_user_grid on | off
    -fix_macros none | soft_macros_only | all
    -congestion_effort low | high
    -IO_net_weight <1.0>
    -plan_group_interface_net_weight <1.0>
    -voltage_area_interface_net_weight <1.0>
    -voltage_area_net_weight_LS_only on | off
    -spread_spare_cells on | off
    -legalizer_effort low | high
    -virtual_IPO on | off                     # 虛擬的原位最佳化
    -pin_routing_aware on | off
```

查看目前設定的策略：
```tcl
report_fp_placement_strategy
```

範例設定：
```tcl
set_fp_placement_strategy -sliver_size 10 -virtual_IPO on
```
> 限制標準單元不會被放置到巨集之間間距小於 10 微米寬度的 sliver（狹窄條帶空間）中，可減少潛在的巨集之間的 routing 壅塞。除 `virtual_IPO` 和 `sliver_size` 外，其他選項一般採用預設值。

### 3.3.7 虛擬展開 Placement VFP 結合虛擬原位最佳化（VIPO）

IPO（In-Place Optimization，原位最佳化）是基於虛擬 routing 的重複最佳化過程，可在全域 routing**之前**或**之後**執行。全域 routing 之後執行 IPO，需先刪除全域 routing 再執行，這樣才能最佳化單元尺寸等。IPO 最佳化的種類包括：時序最佳化、面積恢復（減小面積）、修復 DRC（Design Rule Check，設計規則檢查）違例問題。

在 VFP 階段，若沒開啟 VIPO（Virtual In-Place Optimization，虛擬原位最佳化），工具可能對關鍵路徑通路的標準單元緊湊 placement，減小路徑延遲，但可能造成後續壅塞；開啟 VIPO 可虛擬調整單元尺寸和插入緩衝器，解決延遲問題，避免對關鍵單元間距的不必要調整。

**圖 3-37 多扇出網路的時序最佳化對比**：
- (a) 不使用 VIPO：將標準單元和暫存器的間距縮小，以減小路徑延遲（可能加大後續壅塞）
- (b) 使用 VIPO：不改變單元位置，而增加緩衝器以驅動多扇出網路，減小延遲（對壅塞負面影響更小）

### 3.3.8 執行虛擬展開 Placement VFP，擺放巨集和標準單元

```tcl
create_fp_placement -timing_driven -no_hierarchy_gravity
```
- 標準單元和沒有固定的巨集都按照設計規則擺放，完成 placement
- 預設採用 **routing 長度驅動**的 placement，`-timing_driven` 改為**時序驅動** placement，對時序分析更準確
- VFP 用於合理性分析的快速 placement，**不做邏輯最佳化**
- 預設設計是單元之間沒有階層關係，因此階層重力（gravity）選項是關閉的（`-no_hierarchy_gravity`）

GUI（Place Macro and Standard Cells）：`Effort: Low/High`、`Congestion driven`、`Timing driven`、`Hierarchical gravity`

**階層重力（hierarchical gravity）**：
- 開啟（VFP 預設）：保留電路邏輯階層，模組的單元分布更規則，保持設計 placement 中同一模組的邊界完整性；但不利於對階層設計的邊界電路最佳化
- 關閉：模組邊界不保證完整，但更有利於時序最佳化

點擊 advanced options 可開啟**增量最佳化 incremental**選項：單元的 placement 從當前位置增量調整，增量改進 placement 效能。

---

## 3.4 減少壅塞

VFP 完成後，設計有可能存在單元或巨集壅塞，造成後續 routing 困難，因此需要在 floorplan 階段 VFP 後減少壅塞。

### 3.4.1 減少壅塞步驟流程（圖 3-40）

```
①分析壅塞 → ②修改VFP placement約束 → ③壅塞驅動的虛擬placement → ④分析壅塞
→ ⑤進行高強度的壅塞驅動VFP placement → ⑥分析壅塞 → ⑦重新floorplan（re-floorplan）
→ ⑧固定所有巨集單元placement
```
> 若步驟①④⑥分析結果可接受，可直接跳到步驟⑧。

### 3.4.2 設計壅塞判斷

```tcl
report_congestion -grc_based -by_layer -routing_stage global
```
- 基於 GRC（Global Routing Cell，全域 routing 單元格）的分層分析，透過全域 routing 結果分析 routing 的壅塞
- GUI 可顯示壅塞狀態的**熱圖**（heat map），以不同顏色區分壅塞程度

### 3.4.3 壅塞的指標和計算方法

全域 routing 單元 GRC 分析 routing 壅塞（圖 3-41）：GRC 矩形的大邊框標明邊界，每邊兩個數字如 `39/35` 表示：**需要 routing 的數量為 39，該邊能提供的 routing 軌道（track）數量為 35**。溢出 routing 數量＝需求－供給（如 40/35 → 溢出 5）。

壅塞熱圖（heat map）局部可顯示 GRC routing 存在溢出的邊界情況。GRC 尺寸系統自動設定，也可透過變數設定。壅塞圖＝各 routing 層的 GRC 單元網格＋設定的禁止 routing（routing blockage）圖＋單元的禁止 placement（cell blockage）圖疊加得到。

**嚴重壅塞（無法接受）指標**（滿足以下之一）：
- 壅塞圖中有大量或很多熱點，routing 較難順利完成
- 設計中有任何一個 routing 溢出的 GRC，溢出數字**大於 10**，說明難以 route 通
- 有大約或超過 **2%** 的 GRC 邊沿存在溢出，可能造成訊號完整性或時序效能下降

若壅塞不是大問題，floorplan 將進行下一步 PNS 電源網路合成。

### 3.4.4 Routing 過程中的壅塞紀錄資料

ICC routing 過程統計 GRC 單元 routing 溢出數量、比例（按方向和金屬層統計）：
```
phase5: Routing result:
phase5. Both Dirs: Overflow = 3621 Max = 13 GRCs = 2247 (2.73%)
phase5. H routing: Overflow = 1724 Max = 13 (1 GRCs) GRCs = 1139 (1.38%)
phase5. V routing: Overflow = 1897 Max = 8  (1 GRCs) GRCs = 1108 (1.34%)
phase5. METAL1  : Overflow = 1162 Max = 8  (3 GRCs) GRCs = 879 (2.13%)
phase5. METAL2  : Overflow = 1826 Max = 6  (4 GRCs) GRCs = 1079 (2.62%)
phase5. METAL3  : Overflow = 562  Max = 7  (2 GRCs) GRCs = 440 (1.07%)
phase5. METAL4  : Overflow = 71   Max = 4  (1 GRCs) GRCs = 58 (0.14%)
phase5. METAL5  : Overflow = 0    Max = 0  GRCs = 0 (0.00%)
phase5. METAL6  : Overflow = 0    Max = 0  GRCs = 0 (0.00%)
```
- `Overflow`：總溢出 routing 數量；`Max`：單一 GRC 溢出最大數據；`GRCs`：有溢出的 GRC 數量及占比
- 規律：靠下層（第一、二層）routing 壅塞程度比上層高

ICC 版圖壅塞分析圖形化介面：GUI 版圖介面選擇 **Global Route Congestion**，可分層獨立顯示或多層疊加顯示壅塞狀態。

### 3.4.5 修改 Placement 約束改善壅塞

透過修改 placement 約束**有可能**改善設計壅塞，包括：
- 修改巨集的 placement 約束：`set_fp_macro_options`、`set_fp_macro_array`、`set_fp_relative_location`
  - 可控屬性：增加巨集之間的間距、對齊巨集之間的匯流排訊號接腳、改變巨集方位（旋轉或鏡像）等
- 修改標準單元 placement 約束：`set physopt_hard_keepout_distance`、`set placer_soft_keepout_channel_width`、`set_keepout_margin`
- 其他約束：`set_congestion_options`（設定壅塞選項）、`create_placement_blockage`（建立禁止 placement 區域）

標準單元的**高密度**與壅塞存在**正相關**：預設設定下單元密度可達 95%，但高單元密度造成壅塞可能性增大。透過 GUI 查看單元密度熱圖，對比壅塞熱圖：若壅塞熱點和單元密度熱點位置一致，說明該區域需要降低單元的 placement 密度。

### 3.4.6 減少單元的高密度熱點

```tcl
set_congestion_options -max_util 0.4 -coordinate {x1 y1 x2 y2}
```
- `-coordinate`：設定座標點；`-max_util`：指定最高密度使用率
- 將 {x1 y1} 到 {x2 y2} 矩形區域最高密度使用率降低到 0.4 後，該區域單元 placement 密度明顯降低，壅塞也隨之改善

### 3.4.7 基於座標的 Placement 禁止區

GUI：`Floorplan → Create Placement Blockage`，可用滑鼠框選矩形範圍。

```tcl
create_placement_blockage -name LL_CORNER -type hard \
  -bbox {345.540 355.790 392.280 400.070}
remove_placement_blockage   # 移除禁止 placement 區
```
用於在壅塞程度高的區域禁止放置單元，增加 routing 通道。

### 3.4.8 修改 Floorplan 的 Placement 策略

對於 VFP 中位置不固定的巨集，可設定策略（此前只調整過 `sliver_size`、`virtual_IPO`）：
- `-macros_on_edge`：巨集排列到晶片核心區邊沿
- `-auto_grouping`：巨集按類型自動分組，同類巨集按矩陣形式更緊湊排列（可依巨集是否連接相同網路線來判斷「相關的巨集」）

### 3.4.9 壅塞驅動的虛擬展開 Placement VFP

```tcl
create_fp_placement -timing -no_hierarchy_gravity -congestion
```
開啟 `-congestion` 選項；若設計沒有壅塞問題，則不要開啟此選項。

### 3.4.10 使用高強度壅塞修復策略

```tcl
report_congestion -grc_based -by_layer -routing_stage global
```
若壅塞未解決，用 `-congestion_effort high`：
```tcl
set_fp_placement_strategy -congestion_effort high
create_fp_placement -timing -no_hierarchy_gravity -congestion
```
若高強度（high effort）壅塞驅動 placement 也沒解決壅塞問題，需回到 floorplan 的初始階段，重新規劃 placement。

### 3.4.11 重新 Floorplan（re-floorplan）

重新 floorplan 措施：
1. 修改晶片接腳位置順序、修改接腳金屬層
2. 修改晶片尺寸和寬高比，增大核心區電路面積，提供更多 routing 資源，降低核心區使用率、降低單元密度
3. 若已經產生電源網路，還包括：修改電源網路（grid）結構、使用更多金屬層、修改電源條帶 strap 的寬度和間距等

修改 floorplan 增大電路面積、擴大巨集和單元間的間距，增大 routing 空間 → 解決 floorplan 階段壅塞問題。

### 3.4.12 固定所有巨集的 Placement 位置

```tcl
set_dont_touch_placement [all_macro_cells]
```
固定完成 placement 的巨集，避免後續被工具移動。

### 3.4.13 減少壅塞步驟與對應 Script 指令彙整（表 3-1）

| 主要步驟 | Script 指令 |
|---|---|
| ①分析壅塞 | `report_congestion -grc_based -by_layer -routing_stage global` |
| ②修改 placement 約束 | `set_fp_macro_options ...`／`set_fp_macro_array ...`／`set_fp_relative_location ...`／`set physopt_hard_keepout_distance <#>`／`set placer_soft_keepout_channel_width <#>`／`set_keepout_margin ...`／`set_congestion_options ...`／`create_placement_blockage ...`／`set_fp_placement_strategy ...` |
| ③壅塞驅動的虛擬 placement | `create_fp_placement -timing -no_hier -congestion` |
| ④壅塞分析 | `report_congestion -grc_based -by_layer -routing_stage global` |
| ⑤高強度壅塞驅動虛擬展開 placement | `set_fp_placement_strategy -congestion_effort high`／`create_fp_placement -timing -no_hierarchy_gravity -congestion` |
| ⑥壅塞分析 | `report_congestion -grc_based -by_layer -routing_stage global` |
| ⑦重新 floorplan（re-floorplan） | 修改晶片接腳設定，增大核心區，降低使用率，修改電源網路設定 |
| ⑧固定所有巨集單元 placement | `set_dont_touch_placement [all_macro_cells]` |

---

## 3.5 電源網路合成 PNS（Power Network Synthesis）

基於約束的自動化電源網路設計，取代原本單調的重複性手動版圖繪製，提高效率。PNS 主要包括兩項工作：
1. 建立巨集的**電源環**（rings）
2. 建立電源網格，即連接電源環的**電源條帶**（straps）

PNS 產生過程自動完成，包括：定義電源的拓撲結構、依 IR（電流×電阻，即電壓降）參數要求計算需要的電源條帶 strap 數量，並完成具體電源和接地網路連接，並自動產生導通孔（過孔）。

### 3.5.1 電源環／電源條帶關係圖（圖 3-47）

- **Power pads（電源接腳）**：晶片接腳從外部提供晶片電源輸入；接腳與核心區外圍電源環連接
- **Power rings（電源環）**：與電源條帶連接
- **Power straps（電源條帶）**：實現向核心區均勻供電
- 巨集的電源接腳、標準單元的電源軌道（rail）與電源條帶連接，實現對巨集和單元的穩定供電

### 3.5.2 PNS 主要知識點

- 建立沒有 DRC（設計規則檢查）／ERC（電氣規則檢查）問題的電源環和電源條
- 計算電源條的寬度和數量，以滿足供電和最大電壓降 IR 的要求
- 顯示電壓降 IR 的熱圖，提高分析直觀度
- 允許在產生電源網路前重複進行 "what if" 假設分析
- **一旦產生電源網路（執行 commit）後，無法進行 undo 復原操作回到產生之前的狀態**，因此需要在產生電源網路前保存設計單元：

```tcl
save_mw_cel -as DESIGN_pre_pns
```

### 3.5.3 電源網路合成流程（圖 3-48）

**詳細步驟**：
```
保存設計單元 → 定義邏輯電源、接地連接 → 巨集組的四周建立電源接地環
→ 電源網路約束 → 產生(IR)電源網路 → 分析電壓降(IR)
→ [修改約束重新合成]（疊代）→ 新增電源和接地的接腳單元(若需要)
→ 完成電源網路 → 將單元的電源和接地接腳連接到電源網路
→ 建立電源軌道rail → 分析電壓降(IR) → 使用pnet選項 → 確認 placement 合理性
```

**簡明步驟**：
```
定義電源網路 → 設定PNS約束條件 → 執行PNS → 檢查電壓降/電遷移圖
→ OK？→（否）回到設定約束條件；（是）完成電源網路規劃
```

### 3.5.4 定義電源和接地訊號的邏輯連接

產生 PNS 之前，需確保電源接地 P/G 的邏輯連接已定義（可能在資料設定階段或建立接腳的電源／接地環時已完成）：

```tcl
derive_pg_connection [-power_net][-power_pin][-ground_net][-ground_pin]
derive_pg_connection -power_net VDD  -power_pin VDD  -ground_net VSS  -ground_pin VSS
derive_pg_connection -power_net VDDO -power_pin VDDO -ground_net VSSO -ground_pin VSSO
derive_pg_connection -power_net VDDQ -power_pin VDDQ -ground_net VSSQ -ground_pin VSSQ
derive_pg_connection -power_net PWR -ground_net GND -tie
```
GUI：`Preroute → Derive PG Connection…`

### 3.5.5 在巨集組周邊建立電源／接地環（P/G ring）

巨集組通常按陣列形式規則 placement，可在外圍手動加電源／接地環，並在環內部連接電源條帶（strap）；也可用 PNS 對每個巨集單獨加環，或完全不加電源環。需在 PNS 之前建立巨集組的電源環（與巨集組對比看，單一巨集建立獨立 P/G 環，在 PNS 過程中建立，總資源占用更多）。

**巨集組電源環建立指令（表 3-2）**：

| 指令 | 註解 |
|---|---|
| `set_fp_rail_region_constraints -polygon {…}` | 定義好一個巨集組的區域，`polygon` 選項指定需要新增電源環的多邊形區域，格式 `{{x1 y1}{x2 y2}{x3 y3}{x4 y4}…}` 指定每個頂點座標 |
| `create_fp_group_block_ring -net {VDD VSS} -horizontal_ring_layer … -vertical_ring_layer … -horizontal_strap_layer … -vertical_strap_layer …` | 建立巨集組環，設定巨集組電源環和電源條，設定環和條在 H/V 方向的 routing 層 layer、寬度 width、偏移量 offset |
| `commit_fp_group_block_ring -polygon {…}` | 產生巨集組電源環 |
| `set_fp_rail_region_constraints -remove` | 最後移除對電源區域的約束 |

### 3.5.6 設定電源網路約束

PNS 設定選項包括：
- 支援的 PNS 拓撲結構
- PNS 建立（直角多邊形的）電源規劃（可包含或不包含連接到內核電源網格 mesh 的環）
- 使用者可指定的參數設定（如條帶 straps 數量 min/max、條帶寬度 min/max、環寬度範圍、環和條帶所在層、IR 電壓降的約束條件）

其他電源網路約束：
1. **全域電源網路約束**：堆疊（Stacked）過孔；在硬核或規劃的組上方 routing 的設定；電源軌道使用最佳化設定；移除未連接（float）電源網路段；相同寬度尺寸調整
2. **區塊的電源環約束**：方向、寬度、間距；條帶的密度（調整條帶數量、間距 pitch）

GUI：`Power Network Constraints`（含 Block Rings、Layers、Region、Global 四類約束子視窗）。

`set_fp_rail_constraints` 完整格式：
```tcl
set_fp_rail_constraints
   [-add_layer | -remove_layer | -remove_all_layers | -set_ring | -skip_ring | -set_global]
   [-layer layer] [-direction vertical | horizontal]
   [-max_strap number] [-min_strap number]
   [-max_width distance] [-min_width distance]
   [-spacing distance | minimum | interleaving]
   [-offset distance]
   [-nets nets]
   [-horizontal_ring_layer layer] [-vertical_ring_layer layer]
   [-ring_width distance]
   [-ring_max_width distance] [-ring_min_width distance]
   [-ring_spacing distance] [-ring_offset distance]
   [-extend_strap core_ring | boundary | pad_ring]
   [-keep_floating_segments]
   [-no_stack_via] [-no_same_width_sizing]
   [-optimize_tracks]
   [-keep_ring_outside_core]
   [-no_routing_over_hard_macros] [-no_routing_over_soft_macros]
   [-ignore_blockages]
```

### 3.5.7 PNS 輸入資訊及輸出結果

**輸入資訊**：電源網路名稱、供電電壓名稱、設計核心區功耗數值，以及可選項：電壓降目標、電源接腳資訊、水平和垂直供電網格金屬層、條帶 strap 數量範圍、水平和垂直條帶寬度範圍、電源環寬度範圍。

**輸出結果**：滿足約束條件的電壓降視圖、電遷移圖。

### 3.5.8 電源網路合成 PNS 與電源網路分析 PNA（圖 3-52，5 個設定步驟）

1. 明確指定電源、接地網路名稱和目標 IR 電壓降
2. 指定核心區域電源網路的功耗預算（mW）或電壓降百分比
3. 指定電源接腳資訊以及指定輸出路徑
4. 產生電源網路套用並看分析結果（apply）
5. 最終產生電源網路（commit）

```tcl
synthesize_fp_rail ...   # 執行 PNS 電路產生
analyze_fp_rail ...      # 分析電源網路 PNA
```
PNS 在電路產生過程中利用約束條件**自動計算**需要的電源條帶 strap 數量。從版圖 GUI 選擇查看 IR 電壓降的熱圖，可直觀檢查電源網路設計是否存在問題。

除常規矩形區域電源網路合成外，也可定義**直角多邊形**的 PNS 結構。

### 3.5.9 修改約束重新合成

若電源網路的最大 IR 電壓降分析結果**低於**設計要求下限，需修改電源網路約束，重新進行 PNS 合成，再分析電壓降或繼續修改（疊代）。

### 3.5.10 建立虛擬電源接地接腳

對於供電不足的情況，可建立虛擬的電源和接地接腳，透過 GUI 或指令：
```tcl
create_fp_virtual_pad ...   # 新增
remove_virtual_pad ...      # 刪除
```
新增虛擬電源接腳後，新增約束，再點擊套用執行 PNS，並分析 IR 電壓降熱圖，可對比分析新增 P/G 接腳是否有作用。GUI（Virtual Power Pads）可指定接腳位置、座標，編輯虛擬電源接腳座標位置，透過增減調整接腳數量確定電源接腳數量是否合適。

### 3.5.11 新增電源接地接腳後重新進行 Floorplan

若虛擬電源接腳分析後仍需增加 P/G 電源接地接腳以降低 IR 電壓降，就必須重新進行 floorplan，並新增對新接腳的約束。新規劃前先關閉當前設計單元，再調整之前 floorplan 的相應設定（包括建立單元、設定接腳約束、初始化 floorplan 等）：

```tcl
close_mw_cel
open_mw_cel ...
set_tlu_plus_files ...
create_cell ...
set_pad_physical_constraints ...   # 或 read_io_constraints < TDF_file>
initialize_floorplan ...
...
commit_fp_rail   # 電源網路設計確認無誤後執行產生電源網路
```

### 3.5.12 連接電源／接地接腳 pin，建立電源軌道 rail

產生電源網路後，透過以下兩步操作完成 placement 之前的電源網路剩餘設計：

```tcl
prereoute_instances                                       # 將設計中巨集的電源接腳連接到電源
prereoute_standard_cells -fill_empty_rows -remove_floating_pieces
                                                            # 為標準單元放置列的電源軌道，填滿完整的單元列，確保鋪設的軌道與電源連通
```
指令中 `Preroute` 含義是**在單元 routing 之前**先完成電源網路 routing。

### 3.5.13 電源網路分析 PNA 討論

- **電壓降分析**：分析電源網路每一節點的電壓降數值。電源電壓過低會造成延遲增加和時序問題
- **電遷移（Electromigration, EM）分析**：計算每一段電源走線電流。若電流過高，會造成連接線損壞，降低晶片可靠性和壽命；電遷移分析是分析**平均電流**（因為電遷移是累積效應），分析使用寄生參數模型分析，對 DC（直流）電流更敏感（相對 AC，交流）

在 floorplan 階段進行 PNA 電源網路分析，有利於早期發現電源設計問題。ICC 的 floorplan 電源網路 PNA 與後端設計完成後的簽核（sign-off）階段的準確度相差很小（書中示例：floorplan 階段 PNA 分析功耗數值 378mW，對比設計完成簽核階段功耗分析數值 370mW）。

```tcl
analyze_fp_rail ...
```
- PNA 分析的電源網路**可以是不完整的**（在建立電源環、電源條帶和 IO 接腳以後即可進行分析）
- 對應介面：`Preroute → Analyze Power Network`
- 若電壓降結果與設計差距較大，可先關閉當前設計單元 cel，再重新開啟備份的 PNS 電源網路合成之前的設計單元

**PNA 輸入輸出資料類型（表 3-3）**：

| 資料類別 | 具體資訊 |
|---|---|
| 使用者輸入 | 電源網路名稱、電源供電電壓名稱、設計內核的電源功耗數值 |
| 可選的使用者輸入 | 電源接腳資訊、階層選項（展開或頂層）、垂直電源軌道、單元或區塊的電源資訊 |
| 電路設計資料 | 電源網路幾何形狀（走線與過孔）、單元的實例資料（位置及幾何形狀）、呼叫單元的電源端口 |
| 元件庫資訊 | 標準單元庫檔案及電源接腳、各 routing 層的阻抗表、過孔的阻抗值、所有單元的功耗數值 |
| 輸出資料 | Log 檔案、電壓降視圖、EM 電遷移視圖、文字報表 |

### 3.5.14 PNA 電源網路分析流程（圖 3-58）

```
物理資料（電源網路參數提取）+ 單元功耗資料
   → 電源網路模擬 → IR 和 EM 結果 →（使用者輸入的功耗限制值 / 外部電源分析）
電源規劃初始化 → 幾何圖形資訊、製程、端口等資訊
   ↓
【阻抗快速提取 Engine】→【電源網路分析 Engine】→【錯誤報表和圖形化介面顯示】→【What-if 假設改變】
```
最後一步 **What-if 分析**用於對電源網路設計的假設性調整（如新增虛擬電源接腳），並基於假設修改 PNS 設計，進行 PNS-PNA 設計分析回饋的疊代過程。分析目的是防止電源規劃過於保守。可調整：接腳、電源環、電源條帶以及過孔等，觀察調整後電源網路效能是否改進（圖 3-59：增加電源接腳／增加電源條帶／減小條帶寬度／只放置必需的過孔）。

### 3.5.15 電源網路 Placement Blockage 設定

在晶片核心區內主要電源網路金屬走線（主要指金屬條帶 strap）下方禁止擺放標準單元，有利於減少電源網路對訊號的干擾：
- **完全禁止**放置標準單元：訊號干擾最小，但造成更多 placement 資源浪費，及潛在壅塞問題
- **部分禁止** placement：可提高單元 placement 使用率

```tcl
set_pnet_options -complete {metal2 metal3}   # 完全禁止
set_pnet_options -partial  {metal2 metal3}   # 部分禁止
```
- `-complete`：strap 下方禁止放置單元，`{ }` 內指定哪些金屬層限制單元 placement
- `-partial`：允許單元放置在 strap 下方，但單元的接腳 Pin 到電源 strap 的最小距離要滿足要求

> 前提：大部分數位晶片設計中標準單元接腳在 routing 金屬層 metal1，電源條 strap 在 metal4 或更高層，此情況下電源條帶下方放置單元的訊號干擾不大。若設計要求在低層金屬（如 metal2/metal3）走電源線，需避免單元接腳與電源條的 DRC 衝突，滿足間距要求。若使用率要求不高、內核區域空閒較大，可設定為完全禁止在 strap 電源下方放置標準單元，使 strap 邊緣的單元接腳走線更容易。

### 3.5.16 Placement 合法化（legalization）修正

PNS 的最後一步操作是 placement 合法化修正。若按照前面設定進行了電源網路 routing 避讓區域（部分或完全禁止 placement），需規範化 placement，將衝突的標準單元從電源條 strap 附近移開：

```tcl
legalize_fp_placement
```

### 3.5.17 電源網路合成 PNS 小結（表 3-4）

| Tcl Script 指令 | 功能說明 |
|---|---|
| `save_mw_cel -as DESIGN_pre_pns` | 先保存 PNS 之前的設計單元 |
| `derive_pg_connection ...` | 電源接地的邏輯連接 |
| `set_fp_rail_region_constraints ...`／`create_fp_group_block_ring ...`／`commit_fp_group_block_ring ...` | 巨集組的電源環產生 |
| `set_fp_rail_constraints ...`／`synthesize_fp_rail ...`／`analyze_fp_rail ...` | 產生晶片電源網路 PNS，並分析 PNA |
| （修改並執行上一步指令） | 若需要，修改電源網路約束並重新 PNS 產生並再分析 |
| `create_fp_virtual_pad ...` | 若需要，建立虛擬電源接腳，並分析電源網路 |
| `create_cell ...`／`set_pad_physical_constraints ...`／`initialize_floorplan ...` | 新增新的接腳，重新初始化 floorplan 及後續步驟 |
| `commit_fp_rail` | 確認產生電源網路，完成 PNS |
| `prereoute_instances`／`prereoute_standard_cells ...` | 連接巨集和單元的電源腳，建立電源軌道 rail |
| `analyze_fp_rail ...` | 分析結果，若電壓降不能接受，開啟 `DESIGN_pre_pns`，重新開始 PNS 設計 |
| `set_pnet_options ...` | 設定層的條帶禁止 routing 區域 |
| `legalize_fp_placement` | 電源網路合法化檢查 |

---

## 3.6 Floorplan 減少時間延遲

完成 PNS 電源網路合成後，floorplan 可進行時序最佳化，減少時間延遲。

### 3.6.1 Floorplan 階段減少時延流程（圖 3-61）

```
採用預設floorplan DCT合成 → 建立初始floorplan → 減少壅塞
→ 全域routing/壅塞分析 → 修改電源網路的禁止placement區pnet設定 → 全域routing/壅塞分析
→ 時序分析
  → 1.最佳化時序(預設強度) → 2.最佳化時序(高強度) → 3.修改floorplan/重新合成
→ 減少時延（完成）
```
（灰色框內步驟為時延減少的主要步驟；最佳化時序結果若無法滿足要求，需要重新 floorplan 或回到前端設計重新合成電路）

### 3.6.2 全域 Routing 與壅塞分析

時序分析前，進行一次**真實的**全域 routing（預設時序分析採取虛擬 routing，真實全域 routing 得到更準確時間延遲資訊）：

```tcl
route_zrt_global
```
GUI：`Route → Global Route…`。全域 routing 後可進行壅塞圖分析，直觀觀察壅塞結果。

在 floorplan 階段採用全域 routing 有利於：發現由放置了硬核（IP）和 routing 通道窄所造成的壅塞，同時提供更精確的時序資訊。全域 routing 的定義是把通用的 routing 通道（track）映射到設計的線網（訊號線和時脈）。全域 routing 器使用全域 routing 單元 GRC 的三維矩陣建模全域 routing 的設計需求和實際容量。標準單元的高度被用於定義全域 routing 單元 GRC 的高度和寬度（GRC 的長寬一般為標準單元高度的整數倍）。

GUI 版圖操作：Objects 面板勾選 **Global Route** 圖層顯示效果。

### 3.6.3 修改電源網路 Placement 禁止區域

若壅塞分析情況嚴重，需檢查並修改電源網路 pnet 的禁止 placement 選項。若開啟了完全禁止 placement（complete），需調整為部分禁止（partial）或不禁止。設定指令相同，採用 `set_pnet_options`，修改為選項 `-none`，或用 `remove_pnet_options` 移除之前設定。修改後再進行一次電源網路合法化調整。

```tcl
report_pnet_options                          # 查看之前 pnet 設定
# 方式1：
remove_pnet_options                          # 移除之前選項，或
set_pnet_options -none {M6 M7}               # 移除金屬層 M6 M7 的禁止 placement 限制
# 方式2：
remove_pnet_options                          # 移除之前選項
set_pnet_options -partial {M2 M3}            # 金屬層 M2 M3 的禁止 placement 設定為部分
legalize_fp_placement                        # 電源網路合法化調整
```
電源網路 placement 禁止區域調整後，再次進行全域 routing 與壅塞分析。若需要，進行 PNS 之前的減少壅塞處理流程（3.4 節），直到壅塞控制在合理範圍再進行時序分析。

### 3.6.4 時序分析與時序最佳化

**（1）時序分析**

```tcl
extract_rc          # 提取 RC 寄生參數
report_timing        # 輸出產生的時序報告
```
精確的時序分析將自動呼叫 RC 寄生參數的提取。時序結果的**最差負時間裕度（slack）**數值若控制在要求時延的 **15%～20%**，被認為是可以接受的。時序結果正常，減少時延工作完成後，設計直接跳到輸出 DEF 格式的下一步。

**（2）時序最佳化：執行原位最佳化 IPO**

若時序報告的時間裕度 slack 負值超過設定時延的 15%～20%，要進行 IPO（In-Place Optimization，原位最佳化）：

```tcl
optimize_fp_timing -fix_design_rule <-effort high>   # IPO 時序最佳化
```
IPO 最佳化內容包括：單元尺寸調整，插入緩衝器和 AHFS（Automatic High-Fanout net Synthesis，自動高扇出網路合成）自動綜合高扇出網路，改善時序和 DRC 違例，以及 placement 合規化調整。GUI：`Timing → In Place Optimization`（選項：Effort Medium/High、`Fix design rule`、`Area recovery`）。

執行 IPO 最佳化後，再進行全域 routing 和時序分析；若結果仍無法接受，可再執行一次 IPO 最佳化，開啟高強度選項 `-effort high`。

### 3.6.5 Floorplan 階段的時序最佳化（整體）

```tcl
optimize_fp_timing
```
達到以下目標：
1. 整體最佳化時序結果，快速完成最佳化
2. 可幫助檢驗 sdc（Synopsys Design Constraints，Synopsys 設計約束）檔案和網表的時序合理性，修復設計規則 DRC 違例（包括 `max_cap` 最大電容值和 `max_tran` 最大電平轉換時間）
3. 除了時脈、標注的理想訊號外，自動合成多扇出 buffer 樹網路
4. 減小面積

可透過約束報告、時序報告、壅塞圖分析最佳化後的結果。

### 3.6.6 修改 Floorplan 或重新合成

若高強度的 IPO 最佳化也解決不了時序違例問題，只能**重新 floorplan** 或者**重新邏輯合成**：
- 重新邏輯合成：採用更合理的約束條件和更合理的功能劃分，得到時序效能更好的網表
- 重新 floorplan：可透過調整初始化 floorplan 的整體參數（如晶片使用率、面積、長寬比等）以及最佳化晶片接腳排列以及巨集手動擺放位置等提高晶片時序效能

### 3.6.7 Floorplan 過程減少時延小結（表 3-5）

| 主要最佳化和分析指令 | 解釋 |
|---|---|
| `route_zrt_global` | Floorplan 階段全域 routing 並分析壅塞 |
| `report_pnet_options` | 報告電源網路下禁止 placement 設定 |
| `set_pnet_options -none {M2 M3}`／`set_pnet_options -partial {M2 M3}` | 修改電源網路下禁止 placement 設定，放鬆限制 |
| `legalize_fp_placement` | 本階段 placement 單元的合規調整 |
| `route_zrt_global` | Floorplan 階段全域 routing |
| （壅塞分析） | 壅塞嚴重則回到之前的減少壅塞步驟 |
| `extract_rc`／`report_timing` | 提取 RC 參數並分析時序，若負裕度 neg slack 數值超過時延設定的 15%到20%，則需要時序最佳化 |
| `optimize_fp_timing -fix_design_rule` | IPO 原位最佳化改善時序結果，再次時序分析 |
| `optimize_fp_timing -fix_design_rule -effort high` | 高強度的 IPO 原位最佳化 |
| （重新 floorplan 或重新合成） | 重做前期工作，從根本上改進設計效能 |

---

## 3.7 Floorplan 設計輸出

Floorplan 設計輸出流程包括：提取 placement 與層約束條件、移除所有 placement 的標準單元、輸出 DEF（Design Exchange Format，設計交換格式）檔案、關閉設計。

### 3.7.1 匯出 DEF

```tcl
write_def -version 5.6 -placed -blockages \
  -all_vias -rows_tracks_gcells \
  -routed_nets -specialnets -output DESIGN.def   # 需指定 DEF 版本號，輸出檔案以 DEF 為副檔名
```

### 3.7.2 DEF 輸出用途

輸出的 floorplan 檔案（DEF）常見用途為：基於**拓撲結構**的合成工具 DCT（Design Compiler Topographical，Synopsys 支援拓撲資訊用於邏輯合成的 DC 工具）用來指導重新合成。基於輸出的 DEF 完成重新合成和資料設定後，再次導入 DEF 檔案，可以跳過 floorplan 階段，進行 placement。

匯出 DEF 後，可直接**關閉設計單元**，無需保存單元。保存 DEF 需注意先去掉 placement 的標準單元：

```tcl
remove_placement -object_type standard_cells
```

### 3.7.3 匯出 DEF 後需要重新載入的資訊

`write_def` 指令保存 floorplan 的各類資訊，但以下資訊**需要重新載入**：
- **需要重新載入的指令**：設定忽略的層、設定電源網路 pnet 選項（部分禁止 partial 和完全禁止 complete）placement 設定——這些設定要在導入 floorplan DEF 檔案**後**載入
- **設定的變數**：使用 `set_app_var`，包括巨集的硬性周邊禁止 placement 距離、軟性的禁止 placement 通道的寬度——這些變數設定要在啟動新的 ICC session 時載入

### 3.7.4 Placement 前重新合成的意義及三步驟

Placement 之前進行重新合成，可以提升邏輯網表與物理設計規劃的關聯性，提高電路效能。

1. DCT（Design Compiler Topographical，Synopsys 支援拓撲資訊用於邏輯合成的 DC 工具）重新合成導入真實的 floorplan 設計的 DEF 檔案
2. DCT 合成，產生新的邏輯網表，在資料設定階段向 ICC 導入新的網表
3. ICC 讀入 DEF 檔案，完成重新 floorplan，並準備 placement

### 3.7.5 Placement 前重新合成指令彙整（表 3-6）

| Script 指令 | 解釋 |
|---|---|
| `dc_shell-topo> read_verilog ORIGINAL_RTL.v`／`source ORIGINAL_CONSTRAINTS.cons`／`extract_physical_constraints DESIGN.def` | DCT 導入 DEF floorplan 檔案，並採用 `compile_ultra` 合成 |
| `dc_shell-topo> compile_ultra ...`／`write -format ddc -output DESIGN_2.ddc` | 合成結果輸出 ddc 檔案 |
| `create_mw_lib design_lib orca_2 ...`／`set_tlu_plus_files ...`／`import_designs DESIGN_2.ddc -format ddc -top DESIGN_TOP`／`derive_pg_connection ...`／`source tim_opt_ctrl.tcl` | 再次啟動 ICC 並建立設計庫，導入 ddc 邏輯網表，載入時序最佳化控制 script |
| `read_def DESIGN.def`／`set_ignored_layers -max M7`／`set_pnet_options -partial \| -complete ...` | 導入 floorplan DEF，重新設定層和電源網路 pnet 約束 |
| `save_mw_cel -as DESIGN_floorplanned` | 保存設計單元 |
| `set_app_var physopt_hard_keepout_distance 10`／`set_app_var placer_soft_keepout_channel_width 25` | 重新設定全域禁止 placement 區域 |

---

## 3.8 小結

本章介紹了後端設計 floorplan 階段主要知識和技能，包括：

- 設計（floorplan）的概念原理及基本流程
- 初始化 floorplan，建立晶片或模組的後端設計原型
- 虛擬展開 placement VFP，在 floorplan 階段嘗試對標準單元和不指定位置的巨集進行 placement，判斷 floorplan 方案對後續電路 routing 壅塞以及時序等效能的影響
- 依 VFP 結果減少設計壅塞的技術和步驟
- 電源網路合成 PNS 以及電源網路分析 PNA，完成晶片或模組的電源供電網路規劃設計及 routing，並評估電源網路設計的效能
- Floorplan 階段減少時間延遲的技術
- Floorplan 設計輸出（DEF 檔案）方法以及利用 DEF 檔案進行二次邏輯合成，改進電路效能的流程

---

## 附：全章 Tcl 指令速查表

| 分類 | 主要指令 |
|---|---|
| 設定／任務切換 | `gui_set_current_task -name {Design Planning}`、`open_mw_cel`、`source` |
| 建立物理專用接腳 | `create_cell`、`read_io_constraints`、`set_pad_physical_constraints` |
| 初始化 Floorplan | `initialize_floorplan`、`initialize_rectilinear_block` |
| 接腳填充／電源環（初始） | `insert_pad_filler`、`derive_pg_connection`、`create_pad_rings` |
| Routing 層／巨集限制 | `set_ignored_layers`、`report_ignored_layers`、`remove_ignored_layers` |
| 巨集 Placement 約束 | `set_fp_macro_options`、`set_fp_macro_array`、`set_fp_relative_location`、`set_fp_macro_placement_constraint`、`set_dont_touch_placement`、`remove_dont_touch_placement` |
| 禁止 Placement 區 | `set_app_var physopt_hard_keepout_distance`、`set_app_var placer_soft_keepout_channel_width`、`set_keepout_margin`、`report_keepout_margin`、`remove_keepout_margin`、`create_placement_blockage`、`remove_placement_blockage` |
| Routing 指導 | `create_route_guide` |
| 虛擬展開 Placement VFP | `set_fp_placement_strategy`、`report_fp_placement_strategy`、`create_fp_placement` |
| 壅塞分析與修復 | `report_congestion`、`set_congestion_options` |
| 電源網路 PNS/PNA | `save_mw_cel`、`set_fp_rail_region_constraints`、`create_fp_group_block_ring`、`commit_fp_group_block_ring`、`set_fp_rail_constraints`、`synthesize_fp_rail`、`analyze_fp_rail`、`commit_fp_rail`、`create_fp_virtual_pad`、`remove_virtual_pad`、`prereoute_instances`、`prereoute_standard_cells`、`set_pnet_options`、`report_pnet_options`、`remove_pnet_options`、`legalize_fp_placement` |
| 全域 Routing／時序 | `route_zrt_global`、`extract_rc`、`report_timing`、`optimize_fp_timing`、`report_constraint` |
| 輸出／重新合成 | `write_def`、`remove_placement`、`read_def`、（DCT）`read_verilog`、`extract_physical_constraints`、`compile_ultra`、`write -format ddc`、`import_designs` |

---

## 複習題（教材原文，未附解答，供自我檢測）

**3.2 節**
1. 什麼是只用於物理設計的單元？舉例說明只用於物理設計的接腳單元。
2. `initialize_floorplan` 完成哪些工作？
3. 軟性的禁止 placement 區的特點包括哪些？

**3.4 節**
1. `create_fp_placement` 指令的正確表述包括哪些？
2. 比較 9/1 和 20/12 兩種壅塞圖，結論正確的是？
3. 壅塞通常出現在晶片設計的哪些區域？
4. 在虛擬展開 placement VFP 之前巨集的 placement 相關的控制包括哪些？
5. 若修改 placement 約束條件和參數，並使用高強度壅塞驅動 VFP placement 後，還不能把壅塞降低到合理程度，應如何處理？

**3.7 節**
1. 電源網路合成 PNS 正確的描述是？
2. 部分的 pnet 電源網路下方 placement 禁止區，只對指定的幾個金屬層允許 placement，在其他層禁止 placement，這個說法是否正確？
3. 以下正確的關於 `optimize_fp_timing -fix_design_rule` 描述包括哪些？
4. 為什麼推薦在時序分析前執行 zrt 全域 routing？
5. DEF 檔案的用途是什麼？
6. DCT 重新合成的意義是什麼？

---

## 附：實務案例對照（Cadence Innovus／SoC Encounter，gcd 設計）

> 出處：`~/Downloads/2025_Fall_Training_Package/3.Post_layout_Simulation/APR/scripts/gcd_soce.tcl`（及同目錄 `design_data/gcd.conf`、`gcd.io`、`gcd.view`）

> **工具差異提醒**：以下指令屬於 **Cadence Innovus（前身 SoC Encounter，檔名 `gcd_soce.tcl` 的「soce」即此縮寫）** 的 tcl 語法，與本筆記其餘章節基於 **Synopsys IC Compiler（ICC）** 的指令**名稱不同、但階段概念相通**，每條指令後方標註對應本章（floorplan）的 ICC 概念供對照。

這份 script 是課程教材裡一個名為 `gcd`（Greatest Common Divisor，最大公因數電路）的完整 RTL-to-GDS 流程範例，共 9 個步驟；以下整理與 floorplan／電源規劃相關的前 3 步。

**Step.1　Design Import（設計匯入）**
```tcl
set TOP_DESIGN "gcd"
loadConfig ../design_data/${TOP_DESIGN}.conf 1
```
- `set TOP_DESIGN "gcd"`：設定 tcl 變數，之後全程以 `${TOP_DESIGN}` 代入設計名稱。
- `loadConfig gcd.conf 1`：讀入設計匯入設定檔 `gcd.conf` 並立即執行匯入（結尾的 `1` 表示自動執行）。`gcd.conf` 集中定義了：合成後網表（`gcd_syn.v`）、SDC 約束（`gcd_syn.sdc`）、LEF 檔、IO 檔（`gcd.io`）、MMMC view 檔（`gcd.view`），以及 floorplan 預設參數（`ui_core_util`＝0.5、`ui_aspect_ratio`＝1、`ui_core_to_left/right/top/bottom`＝5 等）。這一步整合了 ICC「資料設置」階段的 `create_mw_lib`、`import_designs`、`read_sdc`、`set_tlu_plus_files` 等多個指令。
  - `gcd.io` 檔案內容是一系列「`Pin: <腳名> <方位 N/S/E/W>`」，功能對應本章 **3.1.13 節** 介紹的 **tdf 檔案**（定義晶片接腳位置），只是格式更簡化（沒有 order/offset，只指定東西南北方位）。
  - `gcd.view` 檔案用 `create_library_set`／`create_constraint_mode`／`create_delay_corner`／`create_analysis_view`／`set_analysis_view` 建立 **MMMC（Multi-Mode Multi-Corner）** 視角設定（slow.lib 對應 setup 角、fast.lib 對應 hold 角），概念對應 `placement.md` **4.2.6 節** 提到的多角多模（MCMM）分析。

**Step.2　Floorplan**
```tcl
floorPlan -r 1 0.7 5 5 5 5
```
- `floorPlan -r <高寬比> <核心使用率> <核心到左> <核心到右> <核心到上> <核心到下>`：建立矩形核心區域，高寬比＝1（正方形）、核心使用率＝0.7（70%）、核心到晶片四邊的間距各為 5（微米）。對應本章 **3.2.1／3.2.2 節** 的 `initialize_floorplan -aspect_ratio ... -core_utilization ...`。

**Step.3　Power planning（電源規劃）**
```tcl
clearGlobalNets
globalNetConnect VDD -type pgpin -pin VDD -inst * -module {}
globalNetConnect VDD -type tiehi -pin VDD -inst * -module {}
globalNetConnect VSS -type pgpin -pin VSS -inst * -module {}
globalNetConnect VSS -type tielo -pin VSS -inst * -module {}
addRing -skip_via_on_wire_shape Noshape -skip_via_on_pin {} -center 1 \
  -stacked_via_top_layer M5 -type core_rings -jog_distance 0.42 -threshold 0.1 \
  -nets {VDD VSS} -follow core -stacked_via_bottom_layer M1 \
  -layer {bottom M5 top M5 right M4 left M4} -width 2 -spacing 0.5 -offset 0.1 \
  -extend_corner {tl lt tr bl br rb lb rt}
sroute -connect { corePin } -layerChangeRange { M1 M5 } \
  -blockPinTarget { nearestTarget } -corePinTarget { firstAfterRowEnd } \
  -allowJogging 1 -crossoverViaLayerRange { M1 M8 } -nets { VDD VSS } \
  -allowLayerChange 1 -targetViaLayerRange { M1 M8 }
```
- `clearGlobalNets`：清除之前的全域電源/地網路連接設定，避免重複或衝突。
- `globalNetConnect VDD -type pgpin -pin VDD -inst * -module {}`：把所有 instance（`-inst *`）上名為 VDD 的電源腳連接到全域電源網路 VDD；對應本章 **3.2.5 節** 的 `derive_pg_connection`。
- `globalNetConnect VDD -type tiehi ...`：把固定接高電位（tie-high）的接腳也連上 VDD；對應 `derive_pg_connection ... -tie`。VSS 兩行同理（`tielo` 為固定接低電位）。
- `addRing ...`：在核心區周圍建立電源環（VDD/VSS power ring）：頂/底走 M5、左/右走 M4，寬度 2、間距 0.5、offset 0.1，並設定四個角落的 jog（轉角避讓）處理方式。對應本章 **3.2.5／3.5.5 節** 的 `create_pad_rings`／PNS 電源環設計。
- `sroute ...`：Special Route，把核心邊界電源環連接到內部標準單元列的電源軌道（rail），指定可用走線層範圍 M1～M5、過孔範圍 M1～M8，目標為每列最靠近的核心電源接腳（`corePinTarget firstAfterRowEnd`）。對應本章 **3.5.12 節** PNS 收尾階段的 `prereoute_instances`／`prereoute_standard_cells`（Preroute，即「先把電源接好再做訊號 routing」）。

> 後續 Step.4～Step.9（placement、CTS、routing、DFM、驗證、輸出）整理於 `placement.md`、`routing.md`；`write_sdf` 輸出的 SDF 檔與 post-layout 模擬的關聯整理於 `STA.md`。
