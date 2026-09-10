# Placement 筆記

> 資料來源：《數位積體電路後端設計》（田曉華主編，武漢理工大學出版社，2019）第 4 章「Placement」，頁 99–142，基於 Synopsys IC Compiler（積體電路編譯器，簡稱 ICC）。

> 本筆記下列專有名詞直接使用英文，不再翻譯成中文：**floorplan**、**script**、**place／placement**、**routing**。本章主題本身就是 placement，因此原書「布局」一詞全數譯為 **placement**。

---

## 縮寫對照表（全稱一覽）

| 縮寫 | 英文全稱 | 中文意義 |
|---|---|---|
| ICC | IC Compiler | Synopsys 的積體電路後端 placement／routing 工具 |
| DC | Design Compiler | Synopsys 的邏輯合成工具 |
| CTS | Clock Tree Synthesis | 時脈樹合成 |
| GRC | Global Routing Cell | 全域 routing 單元格 |
| NDR | Non-Default Routing（rule） | 非預設 routing 規則（如加寬、加間距） |
| CCS | Composite Current Source | 複合電流源模型（適用 90nm 及更新製程的時序模型） |
| ECSM | Effective Current Source Model | 有效電流源模型（Cadence 提出，與 CCS 對應） |
| NLDM | Nonlinear Delay Model | 非線性延遲模型（適用 130nm 及更舊製程） |
| HFN | High Fanout Net | 高扇出網路 |
| AHFS | Automatic High-Fanout net Synthesis | 自動高扇出網路合成 |
| DFT | Design For Test（ability） | 可測試性設計 |
| ATPG | Automatic Test Pattern Generation | 自動測試樣式產生 |
| CG | Clock Gating | 時脈閘控 |
| ICG | Integrated Clock Gating（cell） | 集成式時脈閘控單元 |
| MTCMOS | Multi-Threshold CMOS | 多閾值電壓 CMOS 技術 |
| Vt | Threshold Voltage | 閾值電壓（LVt／HVt／SVt：低／高／標準閾值電壓） |
| UPF | Unified Power Format | 統一功耗格式（描述電源域與功耗策略的 Tcl 檔案格式） |
| DVFS | Dynamic Voltage and Frequency Scaling | 動態電壓頻率調節技術 |
| PVT | Process, Voltage, Temperature | 製程、電壓、溫度（三項變異因素） |
| IR（drop） | Current（I）× Resistance（R） | 電流與電阻乘積，即電壓降 |
| SAIF | Switching Activity Interchange Format | 電平切換活動交換格式（記錄訊號翻轉率的檔案） |
| TR | Toggle Rate | 訊號翻轉率 |
| LPP | Low Power Placement | 低功耗 placement（ICC 的一種 placement 策略） |
| GLPO | Gate Level Power Optimization | 閘級功耗最佳化 |
| RP（group） | Relative Placement（group） | 關聯 placement 組（用於資料通路 placement） |
| PD | Physical Datapath | 物理資料通路 |
| SCANDEF | Scan Definition（file format） | 掃描鏈定義檔案格式 |
| MUX | Multiplexer | 多工器／複用器 |
| DFF | D Flip-Flop | D 型正反器 |
| RET | Retention（register） | 保持暫存器（斷電後仍保留資料的暫存器） |
| STA | Static Timing Analysis | 靜態時序分析 |
| MCMM | Multi-Corner Multi-Mode | 多角（corner）多模式 |
| AP | Application Processor | 應用處理器（手機 SoC 常見用語） |
| RC | Resistance-Capacitance | 電阻電容（寄生參數） |

---

## 章節地圖

| 小節 | 主題 |
|---|---|
| 4.1 | Placement 背景知識 |
| 4.2 | 功耗控制相關技術 |
| 4.3 | Placement 前設定 |
| 4.4 | Placement 及最佳化 |
| 4.5 | 壅塞及時序最佳化 |
| 4.6 | 其他 Placement 技術 |
| 4.7 | 小結 |

Placement 是在晶片後端設計的 floorplan 完成後、時脈樹合成（CTS）前的設計步驟。Placement 的目標可簡單概括為：透過合理放置晶片或模組內的標準單元，最小化電路總面積和互連線。Placement 完成後，晶片內絕大多數單元位置將被固定下來，訊號走線長度可以被更精確地估算，因此晶片的 routing 品質與 placement 品質具有高度關聯性。

---

## 4.1 Placement 背景知識

### 4.1.1 Placement 基本流程

晶片設計流程中的 placement 階段（圖 4-1）：
```
設計指標 → 邏輯設計與驗證 → 邏輯合成（輸出網表）
  → [floorplan → placement → 時脈樹合成 → routing]（物理設計階段，需搭配物理設計約束與物理庫）
```

Placement 前設計需要完成的前期工作包括：
- 設計 floorplan
- 對於利用 floorplan 資訊進行二次合成的設計，完成基於 floorplan 資訊的第二次邏輯合成以及第二次資料設定
- 設計 floorplan 產生的設計單元可以用於 placement：定義核心 core 和外圍 peripheral 區域、定義禁止 placement 區、電源網路完成 routing PNS、巨集位置固定，且 floorplan 階段 VFP 放置的標準單元已從設計單元中清除

**Placement 階段需要考慮的代價因素（影響設計的主要因素）與可採用的解決方式（圖 4-2）**：

| 代價因素 | 可考慮採用的 placement 方法 |
|---|---|
| 面積、routing 長度、單元的重疊 | 傳統的 placement 方法 |
| 時序 | 時序驅動 placement |
| 壅塞 | 壅塞驅動 placement |
| 時脈 | 時脈閘控 Clock Gating |
| 功耗 | 多電壓域多組供電的 placement |

傳統 placement 主要考慮電路面積、單元放置位置對 routing 長度的影響、單元擺放的重疊等問題，減小面積和走線長度並確保單元不重疊放置是傳統 placement 的基本目標。由於 placement 策略的差異會造成後續對設計的時序、壅塞（routing 成功率）、時脈樹網路效能以及設計功耗產生影響，因此 placement 階段針對時序最佳化需要採用時序驅動 placement；針對潛在壅塞問題採用壅塞驅動 placement；針對時脈樹以及電路功耗的最佳化可以考慮在時脈樹添加時脈閘控（clock gating），減小時脈網路的動態功耗，並透過多電壓域和多組電源供電的 placement 最佳化系統功耗。

**1. Placement 基本原理**

Placement 與最佳化步驟從實現的過程和技術原理可細分為：**全域 placement**、**詳細 placement**、**placement 最佳化**。Placement 的輸出資訊包括 placement 後的版圖設計、單元的 placement 位置參數、物理參考庫的時序、製程相關資訊等。

- **全域 placement**：placement 工具按照劃分的規劃組（plan group）計算 placement 問題，為每個規劃組內的標準單元大致分配擺放位置（圖 4-3 左圖）。規劃組是邏輯功能緊密聯繫的小功能集合或子模組，placement 在同一小區域將有利於提高時序效能。全域 placement 不考慮單個標準單元的具體放置，placement 演算法主要考慮規劃組的整體放置合理性。
- **詳細 placement**：與全域 placement 對比（圖 4-3 右圖），詳細 placement 要完成規劃組內的每個單元的具體放置。Placement 演算法完成對規劃組內的單元在分配 placement 區內具體位置、方向的確定。詳細 placement 除了保證單元合規放置外，需要結合單元之間邏輯連接關係盡可能減小 routing 長度。較差的 placement 的管腳以及內部單元放置位置不合理，造成大量交叉的長走線，影響後續 routing 成功率；而較好的 placement 的管腳位置配置合理，單元之間連線較短且交叉較少。
- 詳細 placement 中初略擺放與合規調整（圖 4-5）：先完成單元大致分配的**初略擺放**，再進行**合規調整（legalize）**，將單元對齊到標準單元的 placement 基本單元（unit tile）。

**2. ICC Placement 流程的時序、壅塞、功耗控制策略**

隨著後端設計工具不斷發展，ICC 的 placement 階段功能集成度不斷提高，單次 placement 操作即可完成上述多個步驟。ICC 具體設計流程中，placement 步驟並沒有按照上述劃分步驟執行，而是在完成初始 placement 後，進行較多增量最佳化步驟。

**ICC 對時序、壅塞、功耗的分析和控制流程（圖 4-6）**：
```
Fast 快速 placement
  ↓
壅塞可以接受？
  ├─ 否 → 修改物理約束或者修改 floorplan 階段設計 → （回到 Fast 快速 placement）
  └─ 是 → placement 與最佳化（place_opt / place_opt -congestion / place_opt -effort high / place_opt -effort high -congestion）
             ↓
           時序/功耗/壅塞的問題嚴重？
             ├─ 否 → 壅塞不嚴重？
             │        ├─ 是 → placement 小調整 refine
             │        └─ 否 → 時序和功耗的問題較小？
             │                  ├─ 是 → 增量最佳化
             │                  └─ 否 →（繼續嘗試其他 placement 最佳化策略）
             └─ 是 →（繼續嘗試其他 placement 最佳化策略）
```
當快速 placement 壅塞結果最佳化後，可以進行 placement 與最佳化步驟。該步驟主要指令是 `place_opt`，可以嘗試各種選項和選項的不同組合使結果改善。如果時序、功耗和壅塞結果的問題不嚴重，則進行後續最佳化；否則繼續嘗試其他 placement 最佳化策略。Placement 流程的最後兩步是 `refine`（`refine_placement`）和增量最佳化。`refine` 針對壅塞較大問題，調用 `define_placement` 指令進行 placement 小調整，最後針對時序和功耗結果調用 `psynopt` 指令進行增量最佳化，改進 placement 效能。

**3. ICC 的 Placement 整體流程（圖 4-7）**

```
設計 floorplan
  → Placement 設定與檢查
  → DFT 相關設定
  → 功耗控制設定
  → Placement 與最佳化
  → 改善壅塞和時序
  → CTS
```
在後端設計完成晶片級或模組級的 floorplan 後：
1. 首先完成 placement 設定並檢查設定
2. 對於添加測試電路的設計需要完成 DFT 相關設定
3. 對於需要在後端設計最佳化功耗指標的設計，完成功耗控制相關設定
4. 在接下來的 placement 與最佳化步驟完成單元的放置和最佳化調整
5. 最後針對存在壅塞和時序問題的設計進一步最佳化調整 placement

Placement 達到預期目標後進行時脈樹合成 CTS。

### 4.1.2 Placement 的資料準備、壅塞、時序驅動、多扇出網路

**1. Placement 階段的時序模型**

Placement 階段可以採用複合電流源模型（CCS，Composite Current Source）用於 90nm 及更新製程節點，以支援更精確的時延計算；而非線性延遲模型 NLDM（Nonlinear Delay Model）用於 130nm 及更舊的製程節點。CCS 電流源模型由 Synopsys 提出，用於奈米級電路的時序、功耗、雜訊等多種分析。與 CCS 模型對應的 Cadence 早前提出的是 ECSM（Effective Current Source Model，有效電流源模型）。

**2. Placement 階段的庫設定**

目標庫包括標準單元庫，還包括用於多電壓域設計的電壓轉換單元（level shifter）以及隔離單元。除了標準單元的時序和功能資訊外，鏈接庫還提供巨集以及 IO 接腳單元的時序資訊。

**3. Placement 的壅塞分析**

Floorplan 階段的虛擬展開 placement VFP 使用壅塞分析評估主要 IP 巨集擺放位置的合理性。Placement 階段的壅塞分析是對設計中所有巨集和單元的 placement 是否存在潛在壅塞風險的評估，以避免 routing 階段 routing 困難。Routing 階段是按照虛線所示的等距離 routing 軌道放置金屬線。每一層金屬 routing 的最小寬度、最小 routing 間距由設計規則定義。

當較多條訊號線需要經過較小的 routing 區域時，可能造成 routing 壅塞。Routing 壅塞是透過全域 routing 單元 GRC 計算。每一層 routing 層被劃分為虛線分割的 GRC，而每一個 GRC 區域內能夠支援的四邊最大走線數量由每個方向上的 routing 軌道數量決定。

單個 GRC routing 壅塞程度取決於 GRC 的每邊最大 routing 軌道數量和實際上通過改變走線數量的插值。在 GUI 圖像化介面的 GRC 壅塞熱圖顯示可透過顏色和數量標記直觀顯示各個區域壅塞嚴重程度。不同 placement 策略的單元密度熱圖對比顯示：局部密度高、不均勻的 placement 會造成大量區域出現壅塞問題，而密度均勻的 placement 壅塞情況則改善很多。

要減少壅塞可以透過設置禁止 placement 區域控制單元密度等方式，也可以透過工具自動嘗試單元的 placement 位置，減小高密度連線區域。如果各種調整最佳化無法減少壅塞，則需要重新進行 floorplan 設定。

**4. 時序驅動 Placement**

ICC 採取時序驅動 placement。單元的擺放位置需要考慮到對相關時序路徑延遲的影響。Placement 單元之間的訊號連線延遲可以透過多種方式和模型計算。最精確的是開啟全域 routing 的 placement，在 placement 同時透過全域 routing 計算延遲。Routing 延遲與訊號線阻抗 Rnet 及電容 Cnet 有關，簡單的計算方式如表 4-1 所示，透過訊號扇出數量計算電阻和電容值，結合估算走線長度和驅動負載就可以估算走線時延。

**表 4-1　根據扇出估算走線的 Rnet 和 Cnet**

| Routing 扇出 | 電阻/kΩ | 電容/pF |
|---|---|---|
| 1 | 0.0498 | 0.045 |
| 2 | 0.1295 | 0.0812 |
| 3 | 0.2092 | 0.1312 |
| 4 | 0.2888 | 0.1811 |

可根據時序驅動的 placement 結合估算時延調整單元的 placement：透過緊湊化 placement 關鍵路徑上單元，將相關單元在同一 placement 列或相鄰列排列，減小估算的路徑時延。

**5. 多扇出網路的 Placement 最佳化**

高扇出網路（HFN，High Fanout Net），如系統復位訊號、模組使能訊號，需要 placement 工具特殊處理。多扇出網路驅動大量電路單元，如果不添加驅動單元提高負載驅動能力，時序結果將與理想網路階段的時序結果有較大差距。多扇出網路的最佳化主要方法包括：
(a) 透過時序驅動減小 routing 長度
(b) 增加驅動緩衝器，提高訊號負載驅動能力，減小時延，但增加面積，影響原始 placement

Placement 工具結合 placement 約束和閘限約束，可以自動識別並最佳化 HFN 網路。高扇出網路需要類似於時脈樹生成方式添加訊號驅動緩衝器，因此使用者可以控制緩衝器的選擇範圍。HFN 在邏輯合成階段作為理想網路並沒有添加緩衝器單元。因此在 placement 階段添加緩衝器，必然增大面積並影響其他 placement 單元。完成緩衝器生成和 placement 添加後，需要額外的壅塞和時序分析，確保 HFN 網路不會造成明顯的時序或壅塞問題，否則需要調整 placement 策略，繼續最佳化 placement。

**高扇出網路的 Placement 步驟（圖 4-14）**：
```
分析 placement 結果 + placement 的約束 → 門限設定路徑
  → 定義高扇出網路
  → 選擇採用的緩衝器件（參考庫）
  → 添加緩衝單元到 HFN
  → 進一步完成添加緩衝單元 placement
  → 檢查時序和壅塞？
      ├─ 否 → 時序和壅塞驅動 placement （回到「添加緩衝單元到 HFN」）
      └─ 是 → 最佳化其他指標
```

### 4.1.3 物理合成的概念

**1. 物理合成技術發展概述**

物理合成（physical synthesis）是一個與前端設計的邏輯合成相對應的概念。數位積體電路後端設計也被稱為物理設計，因為電路設計涉及更多底層物理實現細節。後端設計的文獻資料中常提到物理合成，實際上該概念及相關技術是 20 世紀 90 年代末已經由包括 Synopsys 的積體電路 EDA 廠商提出，以應對積體電路製程進入深亞微米階段後物理設計收斂困難的問題。

早期的數位積體電路設計流程中，前端邏輯合成的閘級電路在後端進行 placement／routing。如果後端設計問題無法解決，則返回前端更新電路邏輯設計和邏輯合成，如此反覆直至結果收斂。由於在先進製程節點（深亞微米和奈米級）的晶片互連訊號時延變成影響晶片效能指標的決定性因素之一，前端設計對走線延遲的估算偏差導致前端結果並不能保證後端設計 placement／routing 結果收斂（不能得到滿足指標限制的設計輸出），物理合成成為實現設計收斂的必要武器。在高複雜度通用處理器、圖形處理晶片、高效能 ASIC 等設計中，物理合成技術都成為設計方法的核心組成部分。

**2. 物理合成定義**

作為後端設計的必要功能步驟，物理合成可以定義為：以前端完成邏輯合成的網表為輸入，在後端設計流程中進行邏輯最佳化，輸出新網表以及對應的物理版圖設計，以滿足時序、面積、功耗、布通率等複雜後端設計的約束組合條件。簡而言之，物理合成是後端設計過程中結合設計的物理資料對電路邏輯的調整和最佳化過程。可以認為物理合成是在後端設計流程中對傳統 placement 和 routing 步驟的再一次封裝，並將基於合成的電路最佳化技術與 placement／routing 步驟交織融合在一起完成。

**3. 物理合成技術發展**

物理合成通常從 placement 初步完成後根據時序分析結果進行。對於較差的時序和電路結果，需要採取時序最佳化技術調整電路，包括添加緩衝器、調整閘的尺寸、不同 Vt 值的相同功能單元替換（swap）、電路的複製、連接順序交換等。掃描鏈和時脈網路的添加也在物理合成過程中完成。

例如時脈閘控技術在邏輯合成階段基於電平切換統計資料添加時脈閘控 CG 單元，並最佳化使能訊號的時序；在後端設計中利用物理合成工具在 placement 最佳化階段合併時脈閘門，開啟功耗最佳化；在生成 CTS 時脈樹階段進一步最佳化使能訊號時序，並開啟時脈樹最佳化引擎。後端時脈網路的生成和調整（包括時脈閘控電路）都依賴於物理合成工具。

物理合成的計算精度隨著後端設計步驟的進行而逐步提高。開始階段用走線長度估計和時序分析可採用 Steiner 最小生成樹演算法（rectilinear Steiner tree），而後面可以基於全域 routing 結果以及更精確的詳細 routing 結果。隨著走線長度精度的提高，最佳化演算法的複雜度迅速提高，因此最佳化計算精度反而需要設置得更粗略。各個階段的設計精度選擇和最佳流程選擇一般是採取專屬性方案設計。雖然物理合成技術發展相對成熟，但並沒有通用的解決方案。

為了提高計算效率，物理合成可以採用啟發式方法（heuristic approach），即在後端設計早期階段進行較大的電路調整嘗試。因為早期分析的成本較低。在後續設計階段，隨著電路 placement／routing 精度提高，分析成本提高，電路最佳化的調整範圍縮小，直至設計收斂。要實現物理合成電路效能的完善，在每次電路調整的嘗試後，都需要進行快速增量（incremental）時序分析，以決定是否接受電路改變。由於電路調整帶來 placement 改變，工具需要增量化 placement 調整，並修改網表。

隨著製程不斷提升，物理合成遇到的問題也越來越多，需要評估的不僅僅是時序收斂，而且要考慮設計是否最後能 routing 成功。成功 routing 需要考慮很多新策略，如分散放置單元、重構邏輯、尋找可替換的緩衝策略等。邏輯合成、placement、時脈樹、routing 不再是獨立的設計步驟，相互關聯緊密。增量最佳化工具對於解決棘手的時序和壅塞問題變得必不可少。

---

## 4.2 功耗控制相關技術

數位積體電路在前端邏輯合成和後端 placement／routing 階段都可以根據設計指標最佳化功耗。相對於時脈樹合成和 routing 階段，placement 前標準單元的位置還未確定，EDA 工具針對功耗、時序的電路調整最佳化具有更大的靈活度。

**CMOS 電路功耗主要分 3 種**：
1. **靜態功耗**：主要與製程以及電路結構相關
2. **短路電流功耗**：主要與驅動電壓、PMOS 和 NMOS 管同時打開時產生的最大電流、翻轉頻率，以及上升／下降時間有關
3. **開關電流功耗**：主要與負載電容、驅動電壓、翻轉頻率有關

一個緩衝器的功耗示意圖（圖 4-15）：靜態功耗與訊號翻轉無關，主要由電晶體流向 Gnd 的漏電流 I_leak 造成，電晶體供電即會產生漏電功耗。動態功耗中，電晶體閘極電壓處於臨界區間（Vdd 電壓的 20%～80%）時，P 管和 N 管同時導通形成電源到地的短路通路，形成短路電流 I_short，產生短路電流功耗。而單元內部負載電容 C_int 和外部負載電容 C_load 在電晶體開關狀態切換時的充放電分別產生開關電流 I_intsw 和 I_extsw，造成開關電流功耗。

短路電流功耗和開關電流功耗構成了電路的**動態功耗**。低功耗設計必須考慮影響功耗的因素，應最大限度減小動態功耗，同時針對深亞微米及奈米級先進製程應有效降低靜態功耗。低功耗設計手段較為複雜，對於不同的設計或者不同的製程，實現方法各不相同。

### 4.2.1 不同製程節點的功耗因素

目前國內 0.18μm 製程仍在較多低端數位積體電路晶片中使用。0.18μm 及更早製程節點應用的低功耗技術較有限，主要原因在於電路的靜態功耗很小，基本不用特別處理；而動態功耗方面，主要的功耗來自開關功耗（Switching Power），即與負載電容、電壓以及工作中的訊號翻轉頻率相關。減小負載電容，就必須在設計上下功夫，減小電路規模。減少訊號翻轉頻率，除了降低時脈頻率外，還要在設計上考慮，避免不必要的訊號翻轉。0.18μm 及更早製程的閾值電壓有一定的限制，降低驅動電壓，可以減少動態功耗，但由於電壓降低，驅動能力也同時被減弱，電路元件延時增大。

為了解決時延問題，製程尺寸開始減小，以便在減小驅動電壓的情況下增加寬長比（aspect ratio），以達到增加驅動電流，控制並降低元件延時。進入更小尺寸的製程，柵極氧化層厚度也隨之減小，閾值電壓減小，器件速度進一步提高。但因為氧化層厚度在減小，漏電流也變大。在 90nm 及以下製程中，漏電流成為功耗控制的考慮因素之一。有標準單元數據對比顯示 90nm 製程下的靜態功耗已是 0.18μm 製程下功耗的 3.5 倍左右。某設計案例利用 0.18μm 設計出來的約 40 萬門的電路，靜態電流約 200μA，功耗 360μW（按 1.8V 供電電壓計算）；同樣規模的電路在 90nm 製程下則可能達到 1.26mW 左右、1.05mA 的靜態功耗（按 1.2V 供電電壓計算）。對比可見深亞微米電路隨著製程節點提升，靜態功耗迅速增加。一個直接有效降低靜態功耗的方法是關斷靜止狀態電路的電源輸入。

在不同的製程節點進行功耗控制或低功耗設計，主要有關斷電源、低電壓驅動（利用多電壓混合設計）、多閾值電壓 Multi-Vt 等方法。隨著製程尺寸不斷減小和技術的演進，IP 提供商對晶圓廠工藝流程能提供足夠的實驗資料給 EDA 工具進行自動化設計，減少工程師重複性工作量。

### 4.2.2 時脈閘控（Clock Gating）

時脈閘控（clock gating）主要用於減小電路不必要的動態功耗，即時脈樹網路不必要的電平切換能耗。在暫存器的電路設計中，暫存器內部時脈輸入端都會有一個反向器負載，即使暫存器資料輸入端不發生變化，時脈的變化也會造成該反向器的變化，由此產生動態功耗。如果該暫存器輸入在某種條件下等於輸出（即輸出保持不變）時，可以採用時脈閘控，在資料輸入無變化時切斷暫存器的時脈訊號，以減少無效的時脈翻轉帶來的動態功耗。

由於現在的設計方式中大多數是同步設計，設計人員只需考慮資料路徑，時脈往往是不做處理的。因此要實現時脈閘控，只需提供可以識別的控制訊號（在時脈沿使能訊號 d 傳遞到 q 的 en）給邏輯合成工具（如 DC），即可自動插入時脈閘控，電路中添加時脈閘控單元及閘控時脈訊號的驅動 buffer，而每個暫存器輸入端的訊號選擇多工器 MUX2 在時脈閘控電路中不再需要。

**表 4-2　時脈閘控添加步驟**

| 步驟 | 指令 | 說明 |
|---|---|---|
| 1 | `set_clock_gating_style` | 設定時脈閘控單元插入的約束 |
| 2 | `insert_clock_gating -global` | 開始插入時脈閘控腳 |
| 3 | `uniquify` | 將所有時脈閘控單元做 uniquify 操作，以便後續 placement／routing |
| 4 | `hookup_testports -se_port \ ATPGSE_Pad -se_pin \ uPad/uATPGSE_Pad/C -verbose` | 將所有時脈閘控單元的 scan_enable 訊號與測試用 SE 訊號連接起來（用於添加自動測試功能的設計；若設計不含 ATPG，可不用此指令） |
| 5 | `propagate_constraints -gate_clock` | 將閘控單元資訊傳遞給整個電路 |
| 6 | `report_clock_gating` | 查看時脈閘控單元插入的情況，以便修改電路或修改閘控單元設定 |

完成這些設定後，前端設計流程只需要做常規的邏輯合成即可。在 DC 2008.09 版本以後，第 2～5 步驟都可以省略：若利用 `compile_ultra` 進行合成最佳化，第 2、3 步驟會被自動執行，第 4、5 步驟會在 DFT 測試電路插入（用 `insert_dft` 指令實現）時被執行。形式驗證工具 `formality` 進行形式驗證，設定 `verification_clock_gate_hold_mode` 為 low、high 或 any，`formality` 就可以識別時脈閘控單元，並與 RTL 進行形式驗證。

**時脈閘控單元的驅動範圍**：可以驅動單組暫存器或多組暫存器（圖 4-17）。驅動單組暫存器時，時脈閘控 CG 單元從被驅動模組中移出；驅動多組模組的暫存器時，電路面積最佳化力度相對於驅動單組更大，但對 CG 單元的驅動能力要求更高。

**時脈閘控單元（clock gating cell）**：是指專門設計的集成式時脈閘控單元（integrated clock gating cell，簡稱 ICG），即利用鎖存器 Latch 和與門／或門實現的一個獨立標準單元。其優勢在於以硬 IP 實現，時序易於掌握，物理實現中對 placement／routing 有幫助。若單元庫不提供專門的時脈閘控單元，EDA 工具也可以利用與門、或門、Latch 甚至是暫存器等實現閘控單元，但效果都沒有 ICG 好用。圖 4-17 所示的閘控單元是一種典型的利用負沿使能鎖存器 Latch 以及與門組成的上升沿有效時脈閘控單元，只有時脈下降沿後才會將時脈閘控制住（時脈訊號保持 0），保證不產生時脈毛刺。

在 Liberty 格式（.lib）檔案中，某個 Cell 單元需要有 `clock_gating_integrated_cell` 標記，才能讓 EDA 工具認識到該單元是一種 ICG。對於不同的 `clock_gating_integrated_cell`，需要在 DC 設定 `set_clock_gating_style` 時做相應的設定，才可能使用 ICG。同時在 ICG 的不同 Pin 上，必須有下表所示的屬性，向 DC 指明該 Pin 在 ICG 使用中的具體功能。

**表 4-3　Liberty 庫檔案中 ICG 單元的 Pin 接腳屬性設定**

| 單元的 Pin 接腳屬性名稱 | 接腳屬性說明 |
|---|---|
| `clock_gate_enable_pin` | 該 Pin 是時脈使能控制訊號（圖中 en 訊號） |
| `clock_gate_out_pin` | 該 Pin 是時脈輸出訊號（圖中 ICG 輸出的閘控時脈訊號） |
| `clock_gate_clock_pin` | 該 Pin 是時脈輸入訊號（圖中 clk 訊號） |
| `clock_gate_test_pin` | 該 Pin 是 scan_enable 或 test_mode 訊號（dft 電路中使用） |

範例（SILVACO 公司 Liberty 檔案片段，`ICG` 單元屬性 `clock_gating_integrated_cell` 設為 `latch_posedge`，即上升沿鎖存的 latch；使能腳 `e` 的屬性 `clock_gate_enable_pin` 設為 `true`）：
```
library (Cell_EX10_library) {
  ...
  cell (ICG) {
    area : 10;
    cell_footprint : lat;
    clock_gating_integrated_cell : latch_posedge;
    ...
    pin (e) {
      direction : input ;
      clock_gate_enable_pin : true;
      ...
    }
    ...
  }
}
```

### 4.2.3 功率閘控（Power Gating）

閘控時脈有效減少時序電路中的時脈訊號翻轉，降低動態功耗（dynamic power），而在 90nm 以及更先進製程節點下，即使訊號不翻轉，電晶體的漏電流也會產生靜態功耗（static power）。功率閘控（power gating）採用功率電晶體開關控制電路的供電，當電路在較長時間內不需要工作時，切斷電路供電，使電路進入到待機 standby 或睡眠 sleep 模式，從而最大程度地降低漏電功耗。

**1. 功率開關單元（power switching cell）**

`header`（標頭）與 `footer`（標尾）功率開關（電源開關）控制邏輯電路的供電通斷。功率開關保持工作狀態，因此採用高閾值電壓 Vt 的 MOS 管以減小功率開關本身的漏電流。在邏輯電路空閒狀態下透過睡眠控制訊號關斷功率開關，從而關斷邏輯電路供電，減小電路漏電流。Header 開關 PMOS 管連接真實 Vdd 與電路的虛擬 Vdd，footer 開關連接電路的真實 GND 與虛擬 GND。邏輯電路模組常採用低 Vt 或常規 Vt 的 MOS 管實現。

單元如果同時採用 header 和 footer 功率開關控制供電，占用電路面積較大。Header 功率開關通常漏電流較小，而 footer 開關的面積更小且驅動能力更強，因此採用 footer 功率開關實現功率閘控的設計更多。

功率閘控電路可以完全關斷休眠電路的動態功耗，但是漏電流和相應的靜態功耗只會減少，不會消失。原因在於功率閘控技術需要加入一些隔離單元和保持單元（retention cell），而這些單元維持供電工作，帶來漏電功耗。

**2. 拓展知識點：多閾值電壓電晶體 MTCMOS 技術**

在同一製程中實現多種閾值電壓 Vt 的電晶體被稱作 MTCMOS 技術。該技術常被用於在同一電路中實現不同的功耗和速度等級。前端或後端 EDA 工具可以選擇高 Vt 單元實現低速率低漏電低功耗的邏輯功能；採用低 Vt 單元實現高速率但高漏電高功耗的功能。對於一些非關鍵路徑來說，如果能使用高 Vt 的元件，則可以在滿足時序的前提下減少靜態功耗了。

MTCMOS 技術可以使用阱偏置（Well Bias）技術，使襯底 Substrate 的電壓與 Source 的電壓存在一定的電壓差，就可以改變 Vt 值。使用較多的製程方法是分別對 NMOS 和 PMOS 管增加 1 層光敏膜 Mask 來提高 Vt 或減小 Vt。通常情況下 IP 提供商會提供多套不同的 Vt 值設計庫，如 TSMC 90nm LP 製程的單元庫，就會提供普通 Vt、High Vt、Low Vt 以及 Ultra Low Vt 四套單元庫。目前主流 EDA 工具均支援基於多套 Vt 庫的邏輯合成和後端設計。Multi-Vt 庫定義屬性和多個 Vt 庫設置涉及：庫的 Multi-Vt 屬性、每個單元的 Multi-Vt 屬性，分別對應高 Vt 庫、低 Vt 庫、標準 Vt 庫。

Multi-Vt 設計在邏輯合成（Logic Synthesis）階段可透過 Synopsys DC 完成，主要是從目標庫 `target_library` 的多套邏輯單元庫中找到合適的邏輯單元，在滿足時序約束的情況下使用最低 leakage power 單元實現。在 ICC 中可以採用 `report_threshold_voltage_group` 指令報告設計中採用的各個 Vt 庫的單元數量統計，例如某閾值電壓組報告顯示：採用低 Vt 庫（LVt）的單元只占 8.33%，說明要滿足設計的時序要求，只需要提高少部分電路的開關速度，而 86.12% 的單元採用高 Vt 庫單元可以較大程度減少漏電。

**閾值電壓組報告範例**：
```
* * * * * * * * * * * * * * * * * * * * * * * * * * * *
Threshold Voltage Group Report
* * * * * * * * * * * * * * * * * * * * * * * * * * * *
Threshold Voltage Group    Number of Cells    Percentage
LVt                        90                 8.33%
HVt                        931                86.12%
SVt                        59                 5.46%
undefined                  1                  0.09%
```

後端設計在電路最佳化階段可對非關鍵路徑上用高 Vt 單元替換低 Vt 單元，進一步減少設計的漏電功耗。功能相同 Vt 值不同的單元替換通常在時脈樹合成後或在 routing 完成後執行，因為這兩個階段已完成了單元 placement，訊號線延遲可以精確計算或估算，電路時序結果大體確定，採用多 Vt 單元最佳化對設計指標影響較小。

如在 IC Compiler 裡，在 routing 後最佳化時可以使用如下 `physopt` 語句進行最佳化：
```tcl
physopt -preserve_footprint -only_power_recovery -post_route \
  -incremental
```
該語句設置了 `-preserve_footprint` 以及 `-only_power_recovery`，因此只是針對相同 footprint 的單元做漏電功耗最佳化。例如，如果某個使用了 High Vt X2 的 Buffer 需要減小延遲，則可以替換成 Low Vt X2 的 Buffer。由於 footprint 相同，替換後可以保持原有的 routing 結果。

當出現 Multi-Vt 的單元庫後，每個單元庫都會多一個 `default_threshold_voltage_group` 以及 `threshold_voltage_group` 屬性來說明該單元庫是 High Vt、標準 Vt 還是 Low Vt。可以利用 `report_threshold_voltage_group`（DC 和 IC Compiler 都支援）指令報告設計中每種單元庫單元佔整個設計的百分比，如 High Vt 的單元庫占 60%，標準 Vt 占 25%。這樣做可以使設計者了解不同 Vt 對自己設計的影響，如果設計要求的電路速度不快，則 Low Vt 的庫元件可能占很少，甚至沒有使用，那麼完全可以在設計過程中直接不使用該庫。

**3. 隔離單元（Isolation cell）**

隔離單元用於隔離功率閘控控制的邏輯電路與其他非功率閘控電路，防止短路電流。由於電源關斷後電路的輸出訊號沒有電路驅動，對於驅動電路來說，就會出現輸入浮空的狀態，有可能造成後續被驅動電路的 PMOS 管和 NMOS 管同時導通，形成短路電流。為了解決這個問題，就需要在關閉電源的電路輸出端添加一個額外的保持電路，當控制電源關閉後保持電路表現為輸出等於輸入的緩衝器。另外，如果被電源關閉電路輸入固定電壓，可能產生對地的電流，需要一個特別的單元對該部分電流進行保護。輸出和輸入端口加上的單元被稱為隔離單元。一般來說，隔離單元的輸出部分有較大的電容負載，即隔離單元延時將會比較大，對時序有一定負面影響。

隔離單元可以利用邏輯閘來實現，包括簡單的與門或門。利用與門可以使輸出在關閉電源時為 0，被稱為低電平鉗位隔離訊號（low clamped isolated signal）；利用或門可以使輸出在關閉電源時為 1，被稱為高電平鉗位隔離訊號（high clamped isolated signal）。關斷電源的電路輸出訊號 IN 值為 X，EN 訊號分別設置為低電平有效和高電平有效的開關控制訊號。

**4. 保持暫存器單元（retention register cell）**

對於暫存器來說，如果斷電，則原有的資料就無法保存，重新打開電源後，就一定會出現原有資料丟失的情況。因此可以為一些必須保存資料的暫存器建立一個備份，電源關閉前，將暫存器的數值保存到備份器件上，電源打開後從備份器件上將資料重新寫入暫存器中。這種具有備份和恢復功能的暫存器被定義為保持暫存器單元。電路恢復供電後並從保持暫存器導入資料後，電路恢復到斷電前的工作狀態。

保持暫存器在斷電前把暫存器的值儲存到內部的 RET 鎖存器，重新打開電源後將資料存回暫存器。該暫存器在斷電時，一部分電源保持供電，如 RET。主從鎖存器 Master/Slave Latches 構成常規功能的暫存器，工作在 VDD_SW 電壓域，也就是可以關斷的電壓域，D、CLK、RESETN 和 Q 是寄存器的控制和輸出訊號。RET 電路工作在 VDD 電壓域，不會被關閉，當「SAVE」訊號有效時，RET 會把暫存器的值保存起來，而 RESTORE 有效時，RET 將保存的數值寫入暫存器中。保持暫存器單元通常比普通暫存器面積要大約 20%，如果設計的魯棒性好一些，面積甚至會增加超過 50%。保持暫存器還具有邊界掃描功能，支援資料 D 或者掃描鏈輸入訊號 SI 輸入，並透過 SE 訊號選通輸入。

保持暫存器不但可以保持暫存器的值，還可以保持 Latch 的值，保持暫存器單元的工作時序（圖 4-22）：當 Save 和 Restore 都是 0 的時候，使能保持電路不工作，而暫存器的置位和復位端無效，暫存器正常工作。如果需要進入 Sleep 模式，首先需要停止暫存器的 Clk（Clk_On 訊號關閉），接著準備進入 Sleep 模式。將 Save 為 1，Q 端資料被採集進保持電路中，然後可以關閉電源（Power_on 關閉）。打開電源後（Power_on 打開），將 Restore 置 1，如果保持電路輸出為 1，則對暫存器做異步置位，否則做異步復位，使暫存器輸出與保持電路一致。需要注意的是，Save 從 0 置 1 後，不可再有時鐘改變 Q 端輸出，且 Save 必須有一定的寬度，保證 Q 端資料被記錄下來，同時關閉電源一定要等資料被保持下來後才可以進行。打開電源後，Restore 一定要維持一定的時間，使暫存器充分復位或置位，同時在下一個時鐘沿到來之前，一定要將 Restore 清 0，否則 Q 端不會發生變化，電路可能出錯。當資料被讀回後，就可以打開時脈（Clk_On 打開），開始正常工作了。

上述功率開關單元、隔離單元、保持暫存器單元被統稱為 **always-on 邏輯單元**，即這些單元不能被關閉電源。這些單元的 Liberty 格式描述中會有一個屬性「always-on」是 true。同時對於 always-on 邏輯單元，電源的接腳 `pg_pin` 描述一般會有兩組，主要的（primary）和備用的（backup），工具看到該單元為 always on，就會把兩組電源和地都接到常開的電源／地網路。

**5. 功率閘控的電路模式分類**

功率閘控的兩種模式（圖 4-23）：
- **局部控制**：每個功能模組由獨立的功率開關 MOS 管控制
- **模組級全域控制**：多個功能模組的電源通斷由多個功率開關聯合控制

在模組內部標準單元這一級，功率閘控可分為**精細粒度**功率閘控以及**粗粒度**功率閘控兩種模式。

**（1）精細粒度功率閘控**：每個 switch 都放在 cell 內部（如每一個標準單元閘），因此電源開關稱為單元的一部分。這樣使得面積增大 1x～3x。這樣做的優勢：可以更好地控制由於 IR-Drop 而導致的時序問題。精細粒度的功率閘控的設計實現相對簡單，因為製程庫 IP 提供商通常將睡眠控制的功率開關電晶體作為標準單元的一部分。精細粒度功率閘控電路的時序和功耗的分析與常規單元類似。由於有大量單元需要控制，實現難點在於全域的功率開關控制訊號的驅動和 routing。

**（2）粗粒度功率閘控**：粗粒度功率閘控（coarse grain power gating）採用單獨的閘控器件單元，而不是與標準單元整合。在實現時，由一個或多個閘控單元控制某一塊電路的電源通斷。一個 block 的邏輯閘單元共同擁有一組功率開關（switch cell）。相對於精細力度的每個單元設置功率開關，粗粒度設計使用較少數量的開關單元，比較節約面積，且對晶片製造的 PVT（製程、電壓、溫度）差異不敏感。粗粒度設計必須控制好起電（power up）的衝擊電流，防止對邏輯功能影響和對晶片的損壞，並且需要控制電源網路中的電壓降（IR-drop）。由於實現成本更低，粗粒度功率閘控方案被較多採用。該方式對單元庫的設計要求不高，但需要使用 EDA 工具完成複雜控制。

**6. 功率閘控的後端設計環節**

數位積體電路後端設計要實現功率閘控，首先需要根據前端的定義建立電壓域（voltage area 或 power domain）；在 placement 階段，屬於該電壓域的單元都會自動 placement 到定義的區域範圍內，而不屬於該供區域的單元將被放在該供電域區域外。對於涉及功率閘控的電路設計，floorplan 階段的電源規劃也非常重要，要求精確計算電壓域內需要的電源條（strap）的數量和寬度。

功率開關電晶體可以只採用 header，只用 footer，或被兩者交替排列使用。對於粗粒度功率閘控，可以採用 `add_header_footer_cell_array` 指令將開關單元在電壓域內均勻分布放置，減小區域內電路的電壓降差異。功率閘控開關單元的尺寸越大，則電壓降的波動越小，開關單元的間距也直接影響電源完整性。經法法則是：單元尺寸越大，電壓降越小，但上電的衝擊電流（rush current）越大。

### 4.2.4 多電壓域設計 Placement

多電壓域設計的出發點是實現動態功耗與時序的平衡。多電壓域供電設計將屬於同一個電源域（power domain）的單元 placement 到物理劃分的 placement 區域。針對電源域劃分的 placement 區域稱為電壓域 voltage area。

降低驅動電壓 VDD 是減小動態功耗最簡單的方法。因此在滿足時序的情況下，適當降低驅動電壓，可以有效減小動態功耗。設計可以使用多驅動電壓的設計方法，對於速度要求快的電路，提供高一些的驅動電壓，例如相對於 1.2V 的標準電壓，驅動電壓提高到 1.3V；對於速度要求不高的模組，則只需要提供較低的驅動電壓，如 1.0V。

對於邏輯合成來說，合成工具 DC 中首先需要對不同電壓域電路設置不同的工作條件 `operating_condition`，工具就可以對該電壓域電路進行初步分析和最佳化了。如果使用 UPF（Unified Power Format，統一功耗格式）檔案導入約束，工具會根據 UPF 的描述自動尋找相應的庫檔案進行分析。在 Synopsys 公司提出的 UPF 是一組 TCL 指令構成的檔案格式，用於描述電源連接和晶片功耗域進行設計約束。

在具體設計中，電源域（供電電壓）需要先設置。每個電壓域需要提供電源網路和電源接腳，電壓域和電壓域如何相關聯都需要設置，電壓域與電壓域的電源網路綁定也需要設定。

**低功耗設計方案舉例（圖 4-24／4-25）**：設計主要由 3 個模組組成：8051 控制器透過 SFR 總線對另兩個模組進行控制；U_Des 是一個算法模組，負責高效能計算；U_Pcu 是功耗控制單元，主要完成對 U_Des 單元的功率閘控。整個設計都在 Clk 的控制下進行工作，只有 U_Des 工作在 ClkF 下。設 Clk（28ns）是 ClkF（7ns）的 4 分頻時脈，且與 ClkF 同源，以保證 U_Des 控制的正確性。

芯片設計的頂層電壓域劃分為 8051mcu、pcu 功耗控制單元以及 1.2V 單獨供電的計算單元 DES 三個電壓域。需要根據設計的邏輯階層和功能劃分電壓域，每個電壓域周邊設置有保護帶（guard band）以減小供電和訊號干擾。

**（1）設計方案簡介**：
1. 設計包含 2 個電壓域，TOP 以及模組 U_Des 的電壓域 DES_domain
2. 電源設計外部提供，分 1.2V 的 VDD12 和 1.0V 的 VDD10
3. 設計中除 DES 電路外，其他電路工作在 1.0V 的 VDD 電壓域
4. U_Des 由於工作速度要求較高，工作在 1.2V 的 VDD12 電壓域 DES_DOMAIN。由於不使用的時候需要關斷，以降低靜態功耗，因此透過一組 PowerSwitch 進行控制，控制訊號來自於 U_Pcu 的 PcuSfrDatOut[0] 輸出
5. 由於 U_Des 電路處於 VDD12 電壓域，而 U_Pcu 處於 VDD 電壓域，因此需要添加電平轉換單元 Level Shifter 進行電平轉換。主要供電（頂層設計供電）直接與標準單元的上下邊緣電源網路（電源軌道 power rail）供電連接，而小範圍電壓域使用的電源透過單獨的電源條 strap 引出的供電 routing 連接
6. U_Des 電路輸出訊號，需要通過一個隔離單元 Isolation Cell，在關斷電源時提供穩定電平，而該電平為 1.2V，因此還需要利用電平轉換成 1.0V 電壓域訊號，Isolation Cell 的 Enable 訊號來自於 U_Pcu 模組的 PcuSfrDatOut[1] 訊號
7. U_Des 電路中的暫存器需要使用保持暫存器 Retention Register。該暫存器的 save 和 restore 控制訊號分別來自於 U_Pcu 輸出 PcuSfrDatOut[2] 和 PcuSfrDatOut[3]，並經過電平轉換單元產生的訊號

**（2）設計的主要步驟**：
1. 首先聲明各個電壓域 TOP 與 DES_domain，並聲明 VDD、VDD12 以及 VSS 等電壓端口
2. 聲明每個電壓域中的電源網路，建議從頂層往底層進行聲明，如先聲明 TOP 的 VSS、VDD，再聲明 DES_domain 的電源網路
3. 將電源網路和電源端口連接起來
4. 為每個電源域主要電源網路來源，主電源就是該電壓域普通邏輯工作使用的電源
5. 建立功率開關 Power Switch，並映射功率開關（在合成時只會進行語法檢查，不會有實際電路構建效果，後端 ICC 輸入 DC 輸出相應的 UPF，才會真正添加功率開關）
6. 建立電源狀態表格 PST，對電壓域的有效電壓，以及不同電壓域之間的關係進行設置，同時建立電壓分布及關斷的情景（scenarios），可以用於靜態電壓分析、模擬等
7. 為 DES_domain 輸出給 TOP 的輸出訊號添加隔離單元
8. 將 DES_domain 中在關斷電源後需要保留資料的暫存器替換為保持暫存器 RR
9. 最後對兩個電壓域之間的交互訊號插入電平轉換單元 Level Shifters，實現 1.0V 與 1.2V 訊號的轉換

上述設計步驟均可在 UPF 檔案中定義操作指令，具體 UPF 技術細節請讀者參考 Synopsys 相關技術文檔。

### 4.2.5 低功耗標準單元庫

由於動態功耗中，驅動電壓對功耗的影響也相當大，0.18μm 製程的單元常規電壓是 1.8V 或 1.5V，因此，如果能採用一套電壓只有 1V 的標準單元庫進行設計，仍然可以達到降低動態功耗的目的。但電壓的降低勢必引起元件延時增加，且由於 0.18μm 製程下閾值電壓為 0.4V 左右，驅動電壓的穩定性需求也相當大，否則，可能會導致致命性的錯誤。

IP 設計公司如 ARM 等通常與晶圓廠的 IP 設計團隊合作，如果設計需要的翻轉頻率不高，可以考慮利用低功耗的庫進行設計，達到降低功耗的目的。Synopsys 提供面向大學教育領域的 SAED 90nm 完整的前端和後端設計流程，也提供低功耗庫設計支援。

### 4.2.6 動態電壓頻率調節技術（DVFS）

動態電壓頻率調節技術（dynamic voltage and frequency scaling，DVFS）指的是近年來在功耗敏感且積集處理器內核的 SoC 晶片設計（如智慧手機的應用處理器 AP）常採用的軟硬體協同功耗控制技術。上述時脈閘控、功率閘控以及低功耗標準單元庫等底層電路單元層面的功耗控制，應用於模組內部的電路改動，DVFS 技術可理解為在晶片設計的系統級對主要功能模組的電源和時脈進行調節，屬於系統上層電路控制技術。

DVFS 技術透過上層的軟體演算法決策調度實現對晶片不同功能模組的動態電壓和時脈頻率控制。晶片設計需要實現更複雜的內部電源模組和時脈源模組，以提供對可配置模組的不同時脈頻率和供電電壓輸出，配合軟體設置晶片的不同工作模式，如高效能、低功耗、待機等。軟體切換電壓和頻率模式需要進行較複雜的狀態保存與恢復等控制步驟。該技術除了用於手機處理器外，也可以用於 PC 和伺服器處理器。軟體透過即時採集系統狀態資料，以確定最優的工作模式。

DVFS 通常用於複雜的多模組多階層設計。為了便於電壓和時脈控制，硬體設計時首先需要合理劃分電壓域，將工作模式、時脈頻率控制相似模組劃分到相同電壓域，以便集中控制電壓。對應劃分的電壓、時脈頻率組合的是模組不同的工作模式，如 active、inactive、pending（idle）、sleep 等。在邏輯合成和物理設計階段，需要滿足各個模式下對應的時脈頻率指標要求。EDA 工具早期不支援多種模式並行合成最佳化，只能採取先實現一個電壓與時脈頻率電路合成，滿足時序約束後再針對其他模式做靜態時序分析 STA。如果 STA 分析未達到約束要求，則對電路增量進行最佳化，直至全部模式通過時序分析。支援 MCMM 多角多模的 EDA 工具的 DVFS 合成則根據設定的各個工作模式和工作條件拐角 corner 組合進行同步電路合成最佳化。在後端設計流程中，可採用前文介紹的 MTCMOS 庫對 DVFS 的閘級網表調整，在滿足時序約束的前提下實現功耗最佳化。

---

## 4.3 Placement 前設定

常規的數位後端設計在 placement 前設定階段需要完成三個步驟（圖 4-26）：
```
設計 floorplan
  → Placement 前設定與檢查
  → 測試電路 DFT 相關設定
  → 功耗控制設定
  → Placement 與最佳化
  → 改善壅塞和時序
  → CTS
```
第一步是常規 placement 設定。第二步 DFT 設定是針對測試電路的 placement 最佳化，在針對產品的可測試性設計中都需要添加 DFT 電路。由於 DFT 電路網路單元數量多、分佈範圍廣，placement 階段的單元最佳化放置影響設計效能。第三步功耗控制設定是深亞微米和奈米級電路設計時必須考慮的因素。

### 4.3.1 Placement 前設定與檢查

**（1）固定巨集的 placement 位置**

在絕大部分情況下，巨集單元的 placement 都是在 floorplan 階段確定的，位置從此固定不變。因此 placement 前最好先用 `set_dont_touch_placement` 指令固定巨集位置：
```tcl
set_dont_touch_placement [all_macro_cells]
```
如果 placement 前重新打開前一階段 floorplan 後設計，可採用 `open_mw_cel DESIGN_floorplanned` 讀入前一階段保存的設計單元，並採用 `source tim_opt_ctl.tcl` 加載預先保存的時序和最佳化控制 script。

**（2）驗證層與 placement 約束**

驗證內容包括：忽略的 routing 層、電源／地網路下的禁止 placement 區、全域的禁止 placement 區。

| 指令 | 說明 |
|---|---|
| `report_ignored_layers` | 報告忽略的走線層 |
| `report_pnet_options` | 報告電源／地網路下的 placement 禁止區 |
| `printvar physopt_hard_keepout_distance` | 打印全域的禁止 placement 區設定，包括硬性設定 |
| `printvar placer_soft_keepout_channel_width` | 打印通道區域軟性設定 |

如果需要更新這些設定，可根據下表設置：

**表 4-4　忽略 routing 層、電源網路下方禁止 placement、靜止 placement 距離等設定**

| 設定 tcl 指令 | 指令說明 |
|---|---|
| `remove_ignored_layers -all` | 清除忽略的走線層設定 |
| `set_ignored_layers -max_routing_layer M7` | 設定最大走線層 |
| `set_pnet_options -none {M6 M7}` | 清除兩層的電源／地網路下禁止 placement |
| `remove_pnet_options` | 清除所有電源／地網路下禁止 placement |
| `set_pnet_options -partial {M2 M3}` | 設定兩層電源／地網路下部分禁止 placement |
| `set_pnet_options -complete {M2 M3}` | 設定兩層電源／地網路下完全禁止 placement |
| `set_app_var physopt_hard_keepout_distance < # >` | 設定硬性禁止 placement 距離 |
| `set_app_var placer_soft_keepout_channel_width < # >` | 設定軟性禁止 placement 通道寬度 |

**（3）非預設時脈 routing 規則（Clock NDR）**

非預設走線規則 NDR（non-default routing）主要包括：雙倍間距、雙倍寬度、訊號屏蔽等。ICC 可以對時脈訊號採取這些 NDR 規則 routing，提高訊號品質。時脈訊號 routing 採取增加線寬和增加與相鄰訊號間距的 NDR 措施，可以：
1. 增大間距，可以減小相鄰訊號串擾的影響
2. 增大時脈線寬，可以減小時脈訊號線的電遷移（electro-migration）效應，提高訊號線的可靠性

NDR 規則 routing 將占用更多電路 routing 資源（routing 軌道），會加劇 routing 壅塞狀況，但 ICC 在全域 routing 驅動的 placement 過程會考慮 NDR 壅塞的影響。

設定時脈訊號線寬和線間距的 tcl 指令：
```tcl
define_routing_rule MY_ROUTE_RULES \
  -widths {METAL3 0.4 METAL4 0.6 METAL5 0.6} \     # 不同層的線寬設定
  -spacings {METAL3 0.5 METAL4 0.65 METAL5 0.65}   # 線間距設定
```
設定時脈樹的 routing 規則時，可採用 `set_clock_tree_options` 指令，並應用 `-routing_rule` 選項調用設定的 routing 規則，在 `-layer_list` 選項中可設定 routing 規則應用的 min/max 金屬層：
```tcl
set_clock_tree_options -clock_trees [all_clocks] \
                        -routing_rule MY_ROUTE_RULES \  # 調用規則
                        -layer_list "METAL3 METAL5"     # 應用規則的層
```

**（4）檢查 Placement 的條件完備性**

採用物理設計檢查指令 `check_physical_design`。Placement 前設計檢查的階段是 `pre_place_opt` 階段，即 placement 與最佳化之前。檢查內容包括 floorplan、網表、設計約束等是否完整。

採用物理約束檢查指令 `check_physical_constraints`，檢查輸出的內容包括：
- 是否有單元放置在硬性禁止 placement 區
- 金屬層的設定是否與庫不一致
- Routing 層的 R/C 設定
- 窄通道 placement 區域
- 單元 placement 的合理位置
- 金屬層之間較大的 RC 違例情況

```tcl
check_physical_design -stage pre_place_opt
check_physical_constraints
```

Placement 前檢查後，根據需要，可以修改 floorplan、約束以及庫的設定。

**Placement 前的設定與檢查 script 彙整**：
```tcl
open_mw_cel DESIGN_floorplanned   # 讀入設計單元
source tim_opt_ctl.tcl            # 加載時序和最佳化控制 script
set_dont_touch_placement [all_macro_cells]   # 固定巨集
# 報告忽略層和禁止 placement 相關設定
report_ignored_layers
report_pnet_options
printvar physopt_hard_keepout_distance
printvar placer_soft_keepout_channel_width
# 時脈樹 NDR routing 規則設定並加載設定
define_routing_rule MY_ROUTE_RULES \
  -widths {METAL3 0.4 METAL4 0.6 METAL5 0.6} \
  -spacings {METAL3 0.5 METAL4 0.65 METAL5 0.65}
set_clock_tree_options -clock_trees [all_clocks] \
                        -routing_rule MY_ROUTE_RULES \
                        -layer_list "METAL3 METAL5"
# Placement 前設定檢查
check_physical_design -stage pre_place_opt
check_physical_constraints
```

### 4.3.2 測試電路 DFT 相關設定

對於具有 DFT（面向測試的設計）功能的設計，測試掃描鏈電路在前端設計添加。設計頂層添加的 SCAN_IN 訊號串聯各個暫存器後，掃描訊號從 SCAN_OUT 輸出。由於邏輯合成階段工具並不知道暫存器最終 placement 放置的位置資訊，可能無法反映暫存器之間的空間關聯性，通常，工具根據暫存器名稱的字母順序命名。

後端 placement 預設是時序驅動時，優先按電路的正常邏輯功能與時序約束放置暫存器等標準單元，掃描鏈上的相鄰暫存器有可能與 placement 位置相隔較遠，造成掃描鏈的走線較長，影響測試鏈路時序結果甚至 routing 成功率。

**SCANDEF 檔案**提供前端 DFT 掃描鏈添加流程輸出的掃描鏈資訊。基於該資訊，ICC 可以在 placement 階段結合 placement 的位置資訊，對掃描單元重新排列。包含 SCANDEF 資訊的 ddc 網表不需要單獨導入 SCANDEF 檔案，而對於 .v 或 .vhd 格式的 hdl 網表檔案需要單獨導入 SCANDEF。設計的每一條掃描鏈都根據邏輯功能重新排列。如果掃描鏈中包含鎖存單元（lookup latch）或者複用單元（MUX），則掃描鏈以這些間隔單元劃分為重排列桶（bucket），並在各個重排列桶中重排掃描鏈連接次序。

為了提高重排的效能，ICC 允許多個掃描鏈之間的重排列，聯合組成新的劃分區域（partition），並在劃分區域內重排列。劃分區域內原本屬於不同鏈的暫存器允許交換位置。重排掃描鏈的 routing 效能也進一步提高。

**表 4-5　SCANDEF 檔案舉例**

| SCANDEF 檔案 | 說明 |
|---|---|
| `DESIGN my_design ;` | 設計名稱 |
| `SCANCHAINS 2 ;` | 掃描鏈數量 2 |
| `- 1` | 第一條鏈 |
| `+ START PIN test_si1` | 輸入 pin 腳 |
| `+ FLOATING A ( IN SI ) ( OUT Q )` `        B ( IN SI ) ( OUT Q )` `        C ( IN SI ) ( OUT Q )` `        D ( IN SI ) ( OUT Q )` | Floating 表示鏈中暫存器可以改變順序 |
| `+ PARTITION CLK_45_45` | 劃分區域名為 CLK_45_45，同名的劃分可以交換暫存器 |
| `+ STOP PIN test_so1` | 掃描鏈輸出腳 |
| `- 2` | 第二條鏈 |
| `+ START PIN test_si2` | 輸入 pin 腳 |
| `+ FLOATING E ( IN SI ) ( OUT Q )` `        F ( IN SI ) ( OUT Q )` `        G ( IN SI ) ( OUT Q )` `        H ( IN SI ) ( OUT Q )` | — |
| `+ PARTITION CLK_45_45` | 同名劃分區域 |
| `+ STOP PIN test_so2` | 掃描鏈輸出腳 |

兩條掃描鏈相同名稱劃分區域暫存器重排前後對比：重排後，掃描鏈的走線長度明顯減小，placement／routing 複雜度降低。

**扫描鏈重排要點**：
1. Placement 前先用 `read_def` 讀入 SCANDEF 檔案
2. 在 placement 最佳化時用 `place_opt` 指令開啟 `optimize_dft` 最佳化 dft 選項

**表 4-6　較大規模設計的掃描鏈暫存器重排指令前後 placement 效果和 placement 指令**

| 項目 | 不進行掃描鏈重排的 placement | 掃描鏈重排後 placement |
|---|---|---|
| Placement 指令 | `place_opt` ＃直接 placement 最佳化 | `read_def DESIGN.scandef`／`report_scan_chain`／`place_opt -optimize_dft` |
| 掃描鏈連接狀態對比 | （壅塞、走線混亂） | （走線長度明顯減小，複雜度降低） |

實際 placement 效果顯示重排掃描鏈後，掃描鏈的鏈長度明顯減小，placement／routing 複雜度降低。要顯示掃描鏈連接關係，可以在 ICC 的工具欄 visual mode 按鈕點擊下箭頭，並在彈出的選項列表選擇 Scan Chain，查看設計的掃描鏈連接狀態。

### 4.3.3 功耗控制設定

**1. 動態功耗抑制技術**

動態功耗的抑制主要途徑是發現高訊號翻轉率 TR（toggle rate）的訊號連線，並採取減少線上電容、減小線上閘的尺寸、降低訊號翻轉率的方法降低動態功耗。ICC 中可使用低功耗 placement 設定和閘級功耗最佳化兩類技術實現功耗抑制。

訊號的翻轉率 TR（toggle rate）定義為單位時間內訊號的翻轉次數（0→1 或者 1→0）。要得到翻轉率，可以透過讀入 SAIF 檔案。SAIF（電平切換活動交換格式 switching activity interchange format）檔案是在電路模擬過程中記錄電平跳變次數後寫出的檔案。

> **思考題**：請計算下圖所示的翻轉率 TR。（0ns、15ns、30ns 三個時間點，訊號在 0ns～15ns 為低、15ns～30ns 為高的方波示意圖）**答案：1/10**

SAIF 檔案記錄了電路節點的翻轉次數 TC、訊號為 1 的時間 T1 以及訊號為 0 的時間 T0、模擬時間 DURATION，透過這些資料可以得到計算電路各節點翻轉率。

**表 4-7　SAIF 檔案示例（節錄）**
```
(SAIFILE
 (SAIFVERSION "2.0")
 (DIRECTION "backward")
 (DESIGN )
 (DATE "Mon May 17 02:33:48 2004")
 (VENDOR "Synopsys, Inc")
 (PROGRAM_NAME "VCS-Scirocco-MX Power Compiler")
 (VERSION "1.0")
 (DIVIDER /)
 (TIMESCALE 1 ns)
 (DURATION 10000.00)
 (INSTANCE I_TOP
   (INSTANCE macinst
     (NET
       (z\[3\]
         (T0 6488) (T1 3493) (TX 18)
         (TC 26) (IG 0)
       ) ......
       (z\[32\]
         (T0 6488) (T1 3493) (TX 18)
         (TC 26) (IG 0)
       )
     )
     ......
     (INSTANCE U3
       (PORT
         (Y
           (T0 4989) (T1 5005) (TX 6)
           (COND ((D1 * ! D0) | (! D1 * D0))
             (RISE)
             (IOPATH S (TC 22) (IG 0))
           )
           (COND ((D1 * ! D0) | (! D1 * D0))
             (FALL)
             (IOPATH S (TC 21) (IG 0))
           )
           (COND_DEFAULT (TC 0) (IG 0))
         )
       )
     )
   )
 )
)
```

進行電路的動態功耗最佳化前需要讀入 SAIF 檔案，可採用 `read_saif` 指令，並可用 `report_saif` 顯示讀入 saif 資訊：
```tcl
read_saif -input DESIGN.saif -instance_name I_TOP
report_saif
```

如果無法得到 SAIF 檔案，怎麼根據信號翻轉率經驗估算設定信號翻轉率？下面 script 採用的 `set_switching_activity` 指令對 a、b、c 3 個端口設定了翻轉率，並對設定翻轉率無法觸及的訊號設定預設翻轉率。將不翻轉（不需要統計）的訊號設定為不分析模式，包括復位 rst、掃描使能 se、測試模式 tm。腳本翻轉率設定可以完全覆蓋設計的端口和內部訊號線：
```tcl
create_clock -p 4 [get_ports clk]
set_case_analysis 0 [get_ports "rst se tm"]
set_switching_activity -toggle_rate 0.02 a
set_switching_activity -toggle_rate 0.06 b
set_switching_activity -toggle_rate 0.11 c
set power_default_toggle_rate 0.003
report_power
...
```

上述的手動設置翻轉率或者讀入 SAIF 檔案方式在獲得訊號翻轉率後，即可利用 `report_power` 指令計算得到訊號動態功耗。

**2. 低功耗 Placement 設定（LPP）**

Placement 階段，低功耗 placement LPP 策略包括：
1. Placement 最佳化前完成時脈樹的選項設定，使 placement 工具了解時脈樹設定，最佳化單元放置位置，決定哪些單元可以擺放得更靠近
2. 最小化連線長度，降低線間電容 C
3. Placement 時減小高電平轉換率（TR）的走線長度，減小線間電容
4. 暫存器靠近放置以減小時脈樹電容

使用 LPP 技術，需要按照如下腳本方式先打開 LPP（預設是關閉的），後續 placement 最佳化增加 `-power` 選項：
```tcl
set_power_options -low_power_placement true
...
place_opt -power
```

**3. 閘級功耗最佳化技術（GLPO）**

除了閘控時脈、低功耗 placement LPP 等技術外，IC Compiler 中常用的動態功耗最佳化技術還包括閘級功耗最佳化（gate level power optimization），如：
1. 對訊號跳變（transition）時延較大的電路插入緩衝器，減小驅動級的負載電容，減小跳變時延
2. 調整關鍵路徑上的邏輯單元尺寸，最佳化功耗
3. 根據閘輸入接腳電容以及訊號的翻轉率 TR（toggle rate），對 TR 較大的訊號選擇連接接腳電容較小的閘接腳，以降低閘電路動態功耗
4. 用邏輯等價的電路替換實現計算階段調整，避免在 TR 較高訊號的路徑放置驅動單元（如反相器）

緩衝器插入主要用於提高訊號驅動能力，減小訊號線傳輸時延。隨著奈米級電路中訊號連線時延占比逐漸加大，後端時序最佳化中所添加的緩衝器數量和面積占比逐漸加大，造成功耗增加加大。緩衝器插入演算法需要同時考慮時序和功耗最佳化，合理設置添加緩衝器的數量和位置，以較小的面積和功耗開銷為代價實現後端設計的時序收斂。

單元尺寸調整（gate sizing）是數位積體電路的邏輯合成與最佳化中常用的一種技術，其主要理論是基於 20 世紀 90 年代提出的 logical effort。該理論用於 CMOS 邏輯電路的時延估算，包括用於最小化路徑時延的閘的選擇、閘尺寸計算。關鍵路徑上的兩個與非閘尺寸調整實際上是將輸入訊號連接的與非門和驅動輸出負載的與非門交換。由於兩個閘的 logical effort 相同，而交換後的差異是基於負載輸入電容與本級閘輸入電容之比的 electrical effort 值的差異，對於增大的閘本級輸入負載增加，電容減小，因此本級的 electrical effort 降低，使得閘延遲減小；而輸出級閘則相反，驅動能力減弱，electrical effort 加大，閘延遲增加。關鍵路徑延遲變化取決於第一級時延減小量與第二級增加量的差異。

從閘級功耗角度考慮，統計資料顯示第一級與非閘 n1 的訊號變化率明顯大於第二級閘（第二級輸出還受到閘 n2 控制）。因此兩個與非閘對換將有利於減小第一級驅動的負載，並顯著降低該級以及整體的動態功耗。

閘級功耗最佳化除上述技術外，還包括：技術映射（technology mapping）實現將翻轉率高的訊號線映射到單元內部，減小翻轉功耗（內部負載較低）；降低電路的開關頻率（factoring）。

在 ICC 中設置閘級功耗最佳化的 script 如下：
```tcl
set_power_options -low_power_placement true
place_opt -power
...
set_power_options -dynamic true
psynopt -power
```
上述 script 中首先設定打開低功耗 placement LPP 選項 `low_power_placement`，並進行針對功耗（`-power`）的 placement 最佳化。該步驟實際上同時進行靜態的漏電功耗最佳化和 LPP 低功耗 placement 最佳化。

後續再打開功耗選項的動態功耗（dynamic）開關，並在 `psynopt` 指令執行閘級功耗最佳化，在 `psynopt` 中主要進行閘級功耗最佳化 GLPO。GLPO 與 LPP 最佳化可以同時在 `place_opt` 中完成，結果類似，而分步執行的優勢是執行速率更快。

後端設計工程師也可以自己設定功耗最佳化的最高門限，例如以下範例對漏電功耗和動態功耗上限的設定。相對於預設值 0，放寬的功耗上限有利於功耗最佳化快速完成：
```tcl
set_max_leakage_power 10 uW
set_max_dynamic_power 500 mW
```

**4. 小結：功耗最佳化流程**

ICC placement 相關的功耗最佳化涉及幾個階段多個相關指令和邏輯庫設定。

**表 4-8　Placement 的功耗最佳化指令及說明**

| 功耗最佳化指令 | 說明 |
|---|---|
| `set_app_var target_library "hvt.db svt.db lvt.db"`／`# set_min_library` 對應 target lib 的 max db／`create_mw_lib ... -mw_reference_library \ "mw/sc_hvt mw/sc_svt mw/sc_lvt mw/io mw/ram32"` | 資料設定階段：設定多閾值電壓庫、目標庫，建立 mw 設計庫並指定 mw 參考庫 |
| `read_saif -input DESIGN.saif -instance_name I_TOP`／`# source DESIGN_toggle_rate.tcl` | Placement 的功耗設定階段：讀入 saif 檔案或設定翻轉率 |
| `set_power_options -low_power_placement true`／`report_power_options`／`report_saif` | 設定功耗選項，打開 LPP，漏電最佳化將與 LPP 同時進行 |
| `place_opt -power ...`／`report_power` | Placement 最佳化階段：placement 對功耗最佳化 |
| `set_power_options -dynamic true`／`psynopt -power` | 設定 GLPO 閘級功耗最佳化，GLPO 與漏電功耗同時最佳化 |

**複習題（教材原文，未附解答，供自我檢測）**

1. NDR 規則定義了非預設的 routing 規則，為什麼？NDR 需要在 placement 前設定嗎？
2. SCANDEF 檔案中包含的掃描次序資訊在執行 `place_opt -optimize_dft` 操作時會維持不變，對嗎？
3. `place_opt -power` 指令的漏電功耗最佳化是預設打開的，且對於多閾值電壓庫的設計更有效，對嗎？
4. 低功耗 placement（LPP）可以（　）。
   - a. 移動單元以減短高活躍度的連線
   - b. 將高活躍的單元分散開，以減小功耗密度
   - c. 對於設置了 `set_power_options -dynamic true` 是使能的
   - d. 以上全對
5. SAIF 相對於使用者設定翻轉率 TR 的方法能更好地最佳化漏電功耗，對嗎？

---

## 4.4 Placement 及最佳化

### 4.4.1 Placement 最佳化流程及初始 Placement

Placement 最佳化主要包括三部分操作，並包括兩次壅塞判斷，決定是否進行後續壅塞處理操作。

**表 4-9　Placement 最佳化的主要步驟及指令**

| 步驟 | 指令 | 說明 |
|---|---|---|
| 1 | ＃初始 placement／`save_mw_cel -as DESIGN_preplace_setup`／`place_opt -area_recovery \ |-optimize_dft| |-power| |-congestion|` | 保存 placement 前設計單元，執行 placement 最佳化指令及可選選項設定 |
| 時序、壅塞判斷 | 是否壅塞或者時序建立時間違例？ | 是，則進行第 2 步；否，則完成 placement |
| 2 | `create_placement_blockage ...`／`group_path -name CLK -critical_range <cr> -weight 5`／`set_power_options -dynamic true`／`psynopt -area_recovery |-power| |-congestion|` | 添加或者修改禁止 placement 區設定以及單元密度設定；時序違例則設定關鍵範圍和權重；開啟閘級功耗最佳化 GLPO；進行 placement 後增量最佳化，可添加選項，如壅塞則加 `-congestion` |
| 壅塞判斷 | 是否仍然有嚴重的壅塞？ | 是，則進行第 3 步；否，則完成 placement |
| 3 | `close_mw_cel`／`open_mw_cel DESIGN_preplace_setup`／`set_app_var placer_enable_enhanced_router TRUE` | 重新加載 placement 前設計單元，設定完成回到第 1 步，繼續 |

**1. `place_opt` 指令**

整個 placement 最佳化過程核心指令 `place_opt` 的主要功能是實現時序和壅塞驅動的 placement 以及邏輯最佳化。Placement 階段的時序最佳化只針對建立時間，而對保持時間的違例不做處理。因為在 placement 階段，時脈樹 CTS 還沒有生成，無法精確計算 hold 時間。保持時間違例在設計的下一階段時脈樹網路建立後分析並調節修復。

`place_opt` 指令可選的 4 個主要控制選項：
1. `-area_recovery`：面積恢復選項，選擇對非關鍵路徑去除 buffer 或減小單元面積，但會影響路徑的時序結果
2. `-optimize_dft`：最佳化 dft 選項，主要用於前一節介紹的掃描鏈重排列最佳化，選項值針對包含 DFT 的設計
3. `-power`：功耗選項，觸發漏電功耗（靜態）與動態功耗的最佳化，選項針對需要最佳化功耗的設計
4. `-congestion`：壅塞選項，在 placement 計算中開啟額外的壅塞驅動最佳化演算法。請注意壅塞選項在不存在壅塞問題的設計中不要打開

**2. `place_opt` 指令執行過程**

`place_opt` 指令執行過程可以分為 4 個步驟：
1. **初略 placement**：將單元大致擺放
2. **AHFS**：完成對高扇出訊號的自動合成，例如 rst、scan_en 等訊號；添加或調整扇出網路的緩衝器單元
3. **邏輯最佳化**：除時脈網路以外所有電路的單元器件最佳化
4. **合規 placement**：如本章前文介紹，調整所有的單元按照單元的基本單元（tile）對齊放置，完成單元 placement

**3. 壅塞選項討論**

Placement 採用 `-congestion` 最佳化選項的應用需要滿足的條件是：在 placement 前壅塞已經是一個高優先級問題。可以應用壅塞控制選項的設計階段包括：
1. 設計 floorplan 階段 `set_fp_placement_strategy` 可以設定 VFP placement 的壅塞力度等級 `-congestion_effort`，並在 VFP 指令 `create_fp_placement` 打開 `-congestion` 選項
2. 在導入第三方 floorplan 後，設計查看階段 `place_opt -effort low -congestion` 用於查看導入的 floorplan 的合理性，檢查是否存在壅塞等問題
3. 在 placement 階段的第一步初始 placement 預設是不打開壅塞選項的，當壅塞問題較大時，才考慮其他措施，並嘗試開啟壅塞選項的增量最佳化 `psynopt`

**4. 初始 Placement 後進行時序和壅塞分析**

採用圖形化或文本方式進行時序和壅塞分析。如果沒有時序或壅塞問題，或問題可以忽略，則可以進行 CTS 時脈樹合成步驟。

### 4.4.2 Placement 的 `psynopt` 增量最佳化

**1. 壅塞問題修復設定**

當初始 placement 有壅塞問題時，可以利用設計 floorplan 的減少壅塞技術，採用設定或修改禁止 placement 區（placement blockage）或者修改單元密度（cell density）等約束設定，指令包括：
```tcl
set_app_var physopt_hard_keepout_distance < # >
set_app_var placer_soft_keepout_channel_width < # >
set_pnet_options -partial | -complete ...
create_placement_blockage ...   # 禁止 placement 區域設定
set_keepout_margin ...          # 禁止 placement 邊緣設定
set_congestion_options ...
```

**2. 時序違例分析與解決**

時序問題首先需要區分違例的類型。根據時序報告確定違例路徑主要是在輸入路徑、輸出路徑，還是在暫存器之間（reg to reg）路徑。輸入路徑、輸出路徑的時序違例有可能是過度約束造成，而造成 ICC 不能最佳化屬於在相同路徑分組的暫存器之間違例路徑。

減少時序違例的主要方式是路徑分組（path group），提高時序最佳化效率。

**（1）ICC 的時序路徑組的劃分和最佳化方式**：時序路徑是根據控制路徑終點（endpoint）的時脈劃分路徑組，將相同時脈控制的路徑劃分為同一組。ICC 的時序最佳化策略是對於多個路徑分組，每次從一個分組最佳化一條路徑，從組的關鍵路徑開始最佳化。

**（2）次關鍵路徑最佳化問題**：時序路徑最佳化的常見問題是忽略次關鍵路徑。次關鍵路徑中可能包含大量需要解決的問題時序，但是最佳化器如果卡在最大時延的關鍵路徑無法解決，則會忽略剩餘違例的次關鍵路徑。特別是關鍵路徑如果是由於過度約束造成，例如設置了太小的 IO 端口的路徑時序約束，最佳化器忽略的次關鍵路徑才是真正的時序問題。單一路徑分組的 ICC 時序最佳化：由於未劃分路徑分組，所有類型的路徑同時最佳化，ICC 選擇最佳化時延最大的路徑（關鍵路徑），由於該路徑時延與其他路徑差異很大，在 placement 工具終止時序最佳化時，關鍵路徑與約束目標仍有較大差距，而其他 slack 為負值的次關鍵路徑沒有做任何最佳化。

**（3）路徑分組設定**：針對上述問題，最佳化次關鍵路徑的解決方式是路徑分組。以下 tcl 腳本設定了名稱為 COMBO、INPUTS、OUTPUTS 的 3 個時序路徑分組，分別對應設計中的輸入到輸出純組合邏輯路徑、輸入端口到暫存器路徑以及暫存器到輸出端口路徑。除了設定這三組路徑外，剩餘路徑均為暫存器到暫存器路徑，即 CLK 路徑分組。CLK 分組是設計的預設分組，名稱是 ICC 根據時脈名自動定義：
```tcl
group_path -name INPUTS -from [all_inputs]
group_path -name OUTPUTS -to [all_outputs]
group_path -name COMBO -from [all_inputs] -to [all_outputs]
```
更改路徑分組設定後，ICC 會均勻地分配計算資源給每個分組最佳化時序，使 COMBO、CLK、INPUTS、OUTPUTS 路徑分組的關鍵路徑時延減少，但 CLK 的次關鍵路徑仍然需要最佳化。

**（4）路徑關鍵範圍設定**：要解決 CLK 的次關鍵路徑最佳化，可以設定 CLK 路徑分組的關鍵範圍 critical range：
```tcl
group_path -name CLK -critical 0.3
```
`group_path` 指令對 CLK 分組增加選項 `-critical`，數值設定為 0.3，工具會最佳化 T_critical_path − 0.3ns ～ T_critical_path 範圍內的路徑。其中 T_critical_path 是關鍵路徑時延。

**（5）路徑權重設定**：ICC 的時序最佳化預設每一組時序路徑分組的權重一樣，weight 都為 1，因此最後的時延總代價是各個路徑時延總和。但如果 CLK 分組的路徑時延需要優先最佳化，一個方式是在路徑分組指令 `group_path` 中設定路徑的權重，選項為 `-weight`。以下指令對 CLK 設定了權重 5，遠高於預設的權重 1：
```tcl
group_path -name CLK -weight 5
```
在設定路徑權重後，ICC 的路徑最佳化將選擇不同的方案，結果有利於對 CLK 分組的路徑最佳化。設定 INPUTS 和 CLK 路徑分組前後的時序 slack 對比：原本 INPUTS 分組和 CLK 分組權重都為 1，INPUTS 分組的 -0.4ns slack 路徑與 CLK 的一條 -0.1ns slack 路徑的時延總代價（cost）為 -(0.4×1+0.1×1)=-0.5，而如果將暫存器 FFslw 替換為鎖存速度更快的 FFfst 暫存器後，兩條路徑時延變為 -0.6ns 和 0ns；如果權重不變，則總代價為 -0.6，最佳化不會被接受；但如果設置 CLK 權重為 5，則最佳化前後的代價分別為 -0.9 和 -0.6，總代價降低，最佳化結果被採用。

> **思考題**：請結合 FFslw 與 FFfst 的建立時間（setup）和 clk→Q 時間的差異，分析修改權重後輸入路徑的 -0.6ns slack 以及 FFfst 到 FF2 的路徑 0ns slack 的原因。

對於多時脈驅動設計，可以結合路徑分組，對各個時脈設置不同的路徑分組，並結合關鍵範圍 critical range 和權重設定，提高重點路徑的最佳化優先級。多時脈約束範例：CLK1 分組的權重高且關鍵範圍最大，因此該分組的路徑最佳化具有更高的優先級：
```tcl
group_path -name CLK1 -critical_range 0.3 -weight 5
group_path -name CLK2 -critical_range 0.1 -weight 5
group_path -name CLK3 -critical_range 0.2 -weight 5
group_path -name INPUTS -from [all_inputs]
group_path -name OUTPUTS -to [all_outputs]
group_path -name COMBO -from [all_inputs] -to [all_outputs]
report_path_group
```

**3. 執行 Placement 增量最佳化**

增量邏輯最佳化針對初始 placement 結果對電路進行局部最佳化調整，最佳化工具是 `psynopt` 指令。通常在完成了針對壅塞的 placement 設定修復以及針對時序問題的路徑分組設定後執行 `psynopt`。該步驟主要工作包括：(a) 時序的增量最佳化；(b) 將標準單元合規放置（legalize placement）。

如果設計存在壅塞，則執行 `psynopt` 指令時，要添加 `-congestion` 選項。如果同時要最佳化功耗，則添加 `-power` 選項；而如果要犧牲一部分時序效能換取電路面積受控，可以開啟 `-area_recovery` 選項。

如果要同時進行閘級功耗最佳化 GLPO，在 `psynopt` 之前可以設定功耗選項，開啟動態功耗最佳化（dynamic），指令如下：
```tcl
set_power_options -dynamic true; # GLPO
psynopt -area_recovery [-power] [-congestion]
```

### 4.4.3 Placement 階段開啟全域 Routing 器

增量最佳化後，再次評估壅塞問題。如果壅塞問題可以忽略，則完成 placement 階段設計。增量最佳化也無法解決的壅塞問題，可以在 placement 階段嘗試全域 routing（global router）的方式，將單元的 placement 與全域 routing 工具同時計算，精確定位壅塞位置，並及時調整 placement。該方式需要重新 placement，打開 placement 階段的增強 routing 選項（全域 routing），在 placement 階段改善壅塞。

這種方式與前面嘗試的減少壅塞方式的區別是：之前採用的是走線估計，而這裡是進行真實的全域 routing 的第一階段 routing，透過真實的全域 routing 資料判斷 placement 過程中的潛在壅塞情況。

**（1）本步驟效果**：
1. 大部分設計的壅塞狀況變化不大
2. 有些設計可能會有顯著的壅塞改進
3. 所有設計在 placement 階段開啟全域 routing，都會顯著延長計算時間

**（2）本步驟的適應情況和設置方式**：非常嚴重的壅塞情況才適合打開全域 routing。應用前需要返回到 placement 前設計狀態（前期 placement 資料不保存）；設定 placement 打開增強型 routing 器開關變數；應用 placement 最佳化指令 `place_opt` 並開啟 `-congestion` 選項：
```tcl
close_mw_cel                                              # 關閉當前 placement 單元
open_mw_cel DESIGN_preplace_setup                         # 打開 placement 前設計單元
set_app_var placer_enable_enhanced_router TRUE            # 打開 placement 使能增強 routing 器開關
place_opt -area_recovery -congestion [-optimize_dft] [-power]  # 重新 placement
```

---

## 4.5 壅塞及時序最佳化

**1. 壅塞最佳化**

Placement 階段的最後一步可以利用工具進一步最佳化前面設計步驟存在的時序和壅塞問題。主要應用的工具指令是 `refine_placement`。指令的應用格式如下：
```tcl
refine_placement [-coordinate {X1 Y1 X2 Y2}]
                  [-congestion_effort high]
                  [-perturbation_level < high|max>]
```
該指令可以選擇座標 {X1 Y1 X2 Y2} 指定的矩形區域或者整個設計區域最佳化設計（不指定區域的座標）。壅塞最佳化力度選項包括低 low、中 medium、高 high。擾動等級 `perturbation_level` 選項用於調節局部最佳化範圍，可選擇小 min、中 medium、高 high、最大 max 四檔。

`refine_placement` 只做 placement 增量最佳化，對網表不做修改，因此單元 placement 調整後時序結果有可能變差，需要進一步時序最佳化調整。

**2. 時序最佳化**

採用 `psynopt` 指令進行時序驅動（建立時間）的增量最佳化。可以嘗試選項 `-congestion` 保持現有壅塞狀態或者改善壅塞狀態。`psynopt` 指令的其他選項包括 `-congestion`、`-power`、`-area_recovery`，可以將選項組合使用，但如果時序最佳化級別高，則其他選項可以不開啟。指令的另外兩個設計規則選項 `-no_design_rule`、`-only_design_rule` 以及只調整單元尺寸的 `-size_only` 選項的應用也能改善時序，因此在下例中 `psynopt` 分兩步最佳化，嘗試不同的最佳化組合。這裡最佳化結果如果不理想，並不需要恢復到之前的設計節點，即可嘗試新最佳化選項：
```tcl
psynopt [-power] [-area_recovery] [-congestion]
psynopt -no_design_rule | -only_design_rule | -size_only
```

壅塞和時序最佳化步驟主要要用到上述 `refine_placement` 和 `psynopt` 指令組合。如果結果不理想，可以繼續重複上述兩組指令的不同選項組合模式，並驗證結果是否改進。

**複習題（教材原文，未附解答，供自我檢測）**

1. 預設設定下 `place_opt` 執行的操作包括：（　）
   - a. 針對壅塞最佳化 placement 與邏輯
   - b. 針對建立時間最佳化 placement 與邏輯
   - c. 針對漏電功耗最佳化邏輯
   - d. a、b
   - e. a、b、c
2. 對 IO 邏輯通路設置單獨的路徑組的好處是什麼？
3. 設定時序關鍵範圍是做什麼用的？有什麼好處？
4. 對一個路徑組設定大於 1 的權重將增加設計的關鍵路徑時延，對嗎？
5. `refine_placement` 執行增量的什麼操作？（　）
   - a. 時序驅動 placement
   - b. 壅塞驅動 placement
   - c. 壅塞驅動邏輯最佳化
   - d. a、b
   - e. a、b、c
6. 預設設定下，`psynopt` 執行什麼增量操作？（　）
   - a. 時序驅動邏輯最佳化
   - b. 壅塞驅動 placement
   - c. 壅塞驅動邏輯最佳化
   - d. a、b
   - e. a、b、c

---

## 4.6 其他 Placement 技術

本節主要討論兩種典型的需要特殊處理的電路 placement 最佳化：(a) 數位系統的高扇出網路，如復位、使能訊號；(b) 用於高速高效能資料處理的資料通路（data path）。

### 4.6.1 高扇出網路緩衝樹控制

**1. 建立使用者控制的均衡緩衝樹**

如果高扇出網路採用自動高扇出網路合成（AHFS），由於缺少使用者控制，建立的緩衝樹 buffer tree 有可能各個分支不均衡，造成訊號時序偏差。因此在 `place_opt` 步驟後可以檢查緩衝樹。採用 `remove_buffer_tree` 去除不合理的緩衝樹，採用 `set_cbt_options` 指令控制建立緩衝樹的選項。最後執行 `create_buffer_tree` 指令，針對訊號或接腳建立平衡式緩衝樹，使各個分支末端延遲更接近。

具體指令如下，包括移除、設定和建立樹三步，其中 `set_cbt_options` 指令的 `cbt` 表示創建緩衝樹，選項 `references` 指定使用的緩衝單元列表，而選項 `threshold` 設定自動建立緩衝樹的扇出數量門限，高於門限值則建立樹：
```tcl
remove_buffer_tree -from <pins_or_nets>
set_cbt_options -references <buffer_list> -threshold <t>
create_buffer_tree -from <pins_or_nets>
```

**2. 建立偏移 skew 最佳化的緩衝樹**

重要的緩衝訊號如復位訊號 reset 需要樹的終端節點的訊號偏移（skew）最小化。建立緩衝樹的高扇出訊號的步驟如下：
1. 在 placement 前設置訊號為理想網路，placement 完成前不生成緩衝樹
2. 進行 placement 和最佳化操作
3. 在 placement 完成後去除理想網路設定
4. 採用針對時脈訊號的 `compile_clock_tree` 指令建立 skew 偏移最小化的高扇出網路

具體指令範例如下：
```tcl
set_ideal_network -no_propagate [get_nets Reset]     # placement 前設置理想網路
place_opt                                            # placement 最佳化
remove_ideal_network -no_propagate [get_nets Reset]  # 去除理想網路設定
compile_clock_tree -high_fanout_net [get_nets Reset] # 生成緩衝樹
```

### 4.6.2 資料通路 Placement（data path）

資料通路（data path）邏輯廣泛應用在很多數位訊號處理器或 CPU 設計中。資料通路的特點是資料匯流排上每個比特訊號進行相同的比特操作，電路以比特為基本單位進行資料並行操作。資料通路包含兩種資料流向：一種是資料流入到流出的資料流；另一種是控制訊號流，包括全局的時脈、選擇訊號、使能訊號，以及局部的進位鏈訊號。資料通路的每一種操作對應一種特定電路功能，例如複用器、加法器、暫存器、乘法器等。

**1. 理想的資料通路 Placement 方式**

理想的資料通路 placement 方式是按照資料通路訊號傳遞和處理的規律 placement 計算單元。通路按照比特條（bit slice）結構 placement，將同一個比特的各個操作單元緊密地 placement 到同一列，這種比特條的列結構多列重複，構成多個比特處理的平行排列電路，控制訊號在垂直方向上連接各個比特條。資料通路的理想 placement 結構：簡言之，控制訊號垂直方向傳遞，資料通路水平方向傳遞。

**2. 基於比特條的 Placement 結構的優點**

1. 處理單元 placement 規則，資料通路的單元 placement 面積總體最小
2. 控制訊號和資料訊號可分別採取垂直直線和水平走線，這種規則 routing 方式可以：
   - a) 減小 routing 壅塞，減小連線的寄生 RC，並且減小相應的時延和功耗
   - b) 最小化時脈和資料訊號的 skew 偏移

**3. 傳統 Placement／Routing 工具資料通路設計方式及問題**

傳統工具進行資料通路 placement／routing，結果在面積、壅塞、時延、功耗、skew 偏移等方面都有欠缺。原因在於 placement 以及更早的 floorplan 階段缺乏結合資料處理流向的單元模組放置指導，造成關聯比特位之間的控制訊號無序連接，影響後續 routing 結果。

傳統工具的解決辦法是使用者自訂／手動 placement，依賴設計者對資料通路的結構理解，在標準單元 placement 區手動放置資料通路單元，並完成 routing。

手動設計存在的問題是：
1. 設計時間成本顯著增加，需要手動選擇閘單元尺寸、放置 IP 或通路單元
2. 對自動最佳化工具的利用率低，時序、功耗、面積、DRC 等結果變差
3. 設計修改成本提高，更高邏輯的版圖修改成本高

**4. ICC 關聯 Placement 解決資料通路設計**

ICC 採用關聯 placement（relative placement）方法解決資料通路設計問題。在 Synopsys 的參考文獻資料中，關聯 placement 也被稱為「物理資料通路」（physical datapath）。關聯 placement 控制相關聯的單元緊湊擺放，提高時序和 routing 效能。

對比特通路上每個功能基本單元建立關聯 placement 組（RP group）。例如對於加法器或二選一多工器等基本功能模組定義為一個 RP 組。一個功能模組的關聯 placement（RP）組內部 placement 形式，效果類似於貼瓷磚。行列坐標控制 RP 組內部的單元的擺放位置。RP 組的內部每個單元分配固定高度，即行高度相同，每一列的單元寬度相同，並按照同一列最大寬度單元設定列寬度。這種排列方式的 RP 組內單元排列規整，便於後期 routing。完成 RP 組的定義後，資料通路則由實現不同功能的 RP 組構建。

單元 placement 在 placement 前的設計步驟一般包括：① 設定物理資料通路 PD；② 建立階層化 PD 的關聯 placement 組；③ 單元對齊；④ 單元的 Pin 接腳對齊；⑤ 設定整體單元的定位點；⑥ 自動設定方向；⑦ 使用率控制。

建立關聯 placement 的約束，自動產生調用單元以及 placement 控制資訊的矩陣結構（關聯 placement RP）；RP 組需要定義組內的單元行數和列數；產生的網表可以作為一個組或實體用於 ICC 的物理設計，即 RP 組可以作為一個整體用於 placement。整體 placement 時，指定 x、y 兩個方向坐標固定 placement 組，可以確定一個 RP 組在版圖中的位置。多個 RP 組可以按照阵列形式構成的資料通路 placement，組之間阵列形式等間距排列。

每個 RP 組內的單元列構成葉子單元，支援的對齊方式包括左下邊沿對齊、右下邊沿對齊，同時也支援按照單元的指定 Pin 腳對齊。

RP 組內的單元列排列方式是根據資料流 data flow 方向自動排列（orientation）。如果定義版圖中資料流方向改變（左右鏡像），關聯的單元列將重新排列。

RP 組的列之間的最小間隔為 0，且間隔距離可控，實現不同的區域使用率。可設定 RP 組的面積使用率（如 100%、20%、40% 等），使用率低的 RP 組的單元間隙可以用於 routing 和放置其他標準單元。

在複雜設計中，RP 組支援階層化設計，即大 RP 組嵌套小 RP 組。階層化的物理資料通路設計將 RP 組有效整合，實現功能更複雜的資料通路結構。階層化的物理資料通路構建先建立多個關聯 placement 組；關聯組根據功能、資料關係合並組合成資料通路；根據資料流向調整 RP 分組順序，組成大組。

RP 組內單元排列次序在設計 placement 之前完成，placement 階段 RP 組作為整體擺放，類似於放置標準單元。由於 RP 組整體的尺寸規則一致，RP 組排列構成的資料通路也具有規則性。相比自由擺放模式，基於 RP 組的關聯 placement 方式的 placement 效果，單元放置更整齊規則，更趨近資料理想 placement 實現結果。

**物理資料通路關聯 Placement 的關鍵特點**：
1. 在 placement 階段同時擺放和最佳化標準單元與資料通路的 RP 組
2. 物理資料通路的關聯 placement 在時脈樹合成 CTS、routing、routing 後階段維持不變
3. Placement 及後續階段允許對關聯 placement 邏輯按需要最佳化調整，調整包括對 RP 組的尺寸調整、RP 組在核心區域內 placement 的自動調整
4. ICC 工具有易用的 GUI 介面，便於查看、分析查錯、編輯
5. 物理資料通路關聯 placement 的優勢包括：減少壅塞、訊號偏移、功耗、面積等；設計結果更好，且周期縮短

**物理資料通路的關聯 Placement 其他高階特性包括**：
- a. 提供提高 QoR 結果設定選項，包括單元對齊、接腳對齊、方向、定位點和使用率控制
- b. 階層化物理資料通路設計可提高重用率和積集度
- c. 支援單元的 Tap placement，支援多電壓
- d. 可以從物理版圖設計直接快速生成物理資料通路

---

## 4.7 小結

本章主要介紹了後端設計的 placement 階段的以下知識和技能：
- Placement 前設定，包括針對 placement、可測試性設計 DFT 以及功耗最佳化的設定
- 執行 placement 與最佳化的流程和指令
- 在 placement 後如何分析壅塞熱圖和報告，並根據分析結果進行遞進式壅塞和時序最佳化
- 其他更多使用者可控制的 placement 技術

讀者透過本章知識點學習，可以了解 placement 的基本步驟及 placement 設定與操作對後續設計流程的影響。

---

## 附：全章 Tcl 指令速查表

| 分類 | 主要指令 |
|---|---|
| Placement 前設定與檢查 | `set_dont_touch_placement`、`open_mw_cel`、`report_ignored_layers`、`report_pnet_options`、`printvar physopt_hard_keepout_distance`、`printvar placer_soft_keepout_channel_width`、`check_physical_design -stage pre_place_opt`、`check_physical_constraints` |
| 時脈 NDR routing 規則 | `define_routing_rule`、`set_clock_tree_options -routing_rule` |
| 時脈閘控（Clock Gating） | `set_clock_gating_style`、`insert_clock_gating -global`、`uniquify`、`hookup_testports`、`propagate_constraints -gate_clock`、`report_clock_gating` |
| 多閾值電壓（MTCMOS） | `report_threshold_voltage_group`、`physopt -preserve_footprint -only_power_recovery -post_route -incremental` |
| DFT 掃描鏈重排 | `read_def DESIGN.scandef`、`report_scan_chain`、`place_opt -optimize_dft` |
| 動態功耗抑制 | `read_saif`、`report_saif`、`set_switching_activity`、`set power_default_toggle_rate`、`report_power` |
| 低功耗 placement／閘級功耗最佳化 | `set_power_options -low_power_placement true`、`set_power_options -dynamic true`、`place_opt -power`、`psynopt -power`、`set_max_leakage_power`、`set_max_dynamic_power` |
| Placement 與最佳化核心指令 | `place_opt`（`-area_recovery`／`-optimize_dft`／`-power`／`-congestion`）、`psynopt`（`-power`／`-area_recovery`／`-congestion`／`-no_design_rule`／`-only_design_rule`／`-size_only`） |
| 壅塞相關 | `create_placement_blockage`、`set_keepout_margin`、`set_congestion_options`、`set_app_var placer_enable_enhanced_router TRUE` |
| 路徑分組（時序） | `group_path -name ... -from/-to/-critical_range/-weight`、`report_path_group` |
| Placement 增量最佳化 | `refine_placement`（`-coordinate`／`-congestion_effort`／`-perturbation_level`） |
| 高扇出網路緩衝樹 | `remove_buffer_tree`、`set_cbt_options`、`create_buffer_tree`、`set_ideal_network`、`remove_ideal_network`、`compile_clock_tree -high_fanout_net` |
| 資料通路關聯 placement | RP group（relative placement group）：單元對齊、Pin 對齊、定位點、方向、使用率控制（GUI／ICC 關聯 placement 工具為主，無單一固定指令名） |

---

## 附：實務案例對照（Cadence Innovus，gcd 設計，Step.4～5）

> 出處：`~/Downloads/2025_Fall_Training_Package/3.Post_layout_Simulation/APR/scripts/gcd_soce.tcl`（承接 `floorplan.md` 附錄的 Step.1～3）

> 同樣提醒：以下為 **Cadence Innovus** 語法，與本章基於 **Synopsys ICC** 的指令是概念對照、非語法對照。

**Step.4　Placement**
```tcl
setPlaceMode -fp false
placeDesign -prePlaceOpt
addTieHiLo -cell {TIELO TIEHI} -prefix LTIE
fit
setOptMode -fixCap true -fixTran true -fixFanoutLoad false
optDesign -preCTS
```
- `setPlaceMode -fp false`：設定 placement 模式，`-fp false` 表示不使用階層化 flow planning（區塊）模式，即扁平化（flat）標準單元 placement，對應本筆記反覆強調的「非階層化」placement。
- `placeDesign -prePlaceOpt`：執行標準單元 placement，`-prePlaceOpt` 選項在 placement 前先做一次邏輯最佳化，概念對應 **4.1.3 節「物理合成」**；整體對應 ICC 的 `create_fp_placement`（VFP）＋`place_opt`（**4.4.1 節**）合併為一個指令。
- `addTieHiLo -cell {TIELO TIEHI} -prefix LTIE`：插入固定接高/低電位的 tie cell（TIELO/TIEHI），前綴命名為 `LTIE`，用於把未使用或需固定邏輯電平的接腳綁到電源/地。這是 Innovus 獨立的一步指令，ICC 中對應概念含在 `derive_pg_connection -tie`（`floorplan.md` **3.2.5 節**）裡，但不是獨立單元插入指令。
- `fit`：把版圖視窗縮放至剛好容納整個設計（純 GUI 顯示指令，不影響設計本身）。
- `setOptMode -fixCap true -fixTran true -fixFanoutLoad false`：開啟修復最大電容（`max_cap`）與最大轉換時間（`max_tran`）違例，暫不修復扇出負載違例；對應本章 DRC 違例相關概念（**4.3 節**）。
- `optDesign -preCTS`：CTS 前的時序最佳化（以建立時間為主），對應 ICC 的 `place_opt`／`psynopt`（**4.4 節**）。

**Step.5　Clock Tree Synthesis（CTS）**
```tcl
set_ccopt_property buffer_cells   { CLKBUF* }
set_ccopt_property use_inverters true
set_ccopt_property update_io_latency false
setDesignMode -process 90
create_ccopt_clock_tree_spec -file ccopt.spec
source ccopt.spec
ccopt_design -cts
setOptMode -fixCap true -fixTran true -fixFanoutLoad true
optDesign -postCTS
optDesign -postCTS -hold
```
> 教材《數位積體電路後端設計》第 5 章「時鐘樹綜合」尚未整理成筆記，這裡先記錄 Innovus 對應指令的作用，供之後對照。

- `set_ccopt_property buffer_cells { CLKBUF* }`：CCOpt（Clock Concurrent Optimization，Innovus 的 CTS 引擎）只能使用名稱符合 `CLKBUF*` 的專用時脈緩衝器單元。
- `set_ccopt_property use_inverters true`：允許時脈樹中使用反相器（不只緩衝器），常用於平衡延遲或做時脈反相。
- `set_ccopt_property update_io_latency false`：不更新輸入輸出延遲數值，維持原有設定。
- `setDesignMode -process 90`：設定製程節點為 90nm，套用對應的製程預設規則。
- `create_ccopt_clock_tree_spec -file ccopt.spec`：依目前設計自動產生一份時脈樹合成規格檔（實務上通常會先手動編修這份檔案再套用，此範例直接套用預設值）。
- `source ccopt.spec`：讀入並套用上一步產生的規格檔。
- `ccopt_design -cts`：真正執行時脈樹合成，插入緩衝器/反相器並生成實體時脈網路，對應 ICC CTS 章節的核心動作。
- `setOptMode -fixCap true -fixTran true -fixFanoutLoad true`：CTS 完成、時脈樹結構已固定後，重新開啟扇出負載違例修復。
- `optDesign -postCTS`：CTS 後的時序最佳化（以建立時間為主）。
- `optDesign -postCTS -hold`：CTS 後針對**保持時間**做最佳化——正好對應 `STA.md` 第 4 節「Placement 後 STA」提到的：「時脈樹（CT）插入後，保持時間才能被精確計算與修復」。

> 後續 Step.6～9（routing、DFM、驗證、輸出）整理於 `routing.md`。
