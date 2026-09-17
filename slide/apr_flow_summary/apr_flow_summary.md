---
marp: true
theme: default
paginate: true
size: 16:9
footer: 'APR Flow Summary — Design Import to Sign-off'
style: |
  section {
    font-size: 27px;
  }
  section.lead h1 {
    font-size: 56px;
  }
  h1 { font-size: 34px; }
  h2 { font-size: 27px; color: #1a5276; }
  table { font-size: 21px; }
  code { font-size: 0.88em; }
  pre { font-size: 19px; }
---

<!-- _class: lead -->

# APR Design Flow 重點整理
## Design Import → Sign-off

各階段要做什麼、要注意什麼

---

## Agenda

1. 九大步驟總覽
2. Step 1–3：Design Import → Floorplan → Power Planning
3. Step 4–6：Placement → CTS → Routing
4. Step 7–9：DFM → Verification／Sign-off → Data Export
5. 總結：檢查清單＋貫穿全流程的疊代收斂循環

---

## 九大步驟總覽

| # | 階段 | 一句話重點 |
|---|---|---|
| 1 | Design Import | 匯入網表／約束／製程檔，建 MMMC |
| 2 | Floorplan | 定晶片尺寸、cell utilization |
| 3 | Power Planning | 建電源 ring／stripe／rail |
| 4 | Placement | 擺巨集與標準單元、時序收斂 |
| 5 | CTS | 合成時脈樹、平衡 skew |
| 6 | Routing | 訊號線繞線、修 DRC／antenna |
| 7 | DFM | Filler、metal fill 收尾 |
| 8 | Verification／Sign-off | DRC／LVS／timing 全部過關 |
| 9 | Data Export | 輸出 GDS／SDF／網表交付 |

---

<!-- _class: lead -->

# Step 1–3：從匯入到電源網路

---

## Step 1：Design Import — 要做什麼

匯入合成後網表、SDC 時序約束、LEF/Lib 製程檔，建立 MMMC（多角多模）分析設定。

```tcl
loadConfig ${TOP_DESIGN}.conf 1
```

一份 `.conf` 通常集中定義：netlist、SDC、LEF、IO 檔、MMMC view 檔。

---

## Step 1：Design Import — 要注意什麼

- 匯入後**不能有 load error**——網表／約束／製程檔版本要對得上
- MMMC 至少要涵蓋 **setup（慢角）** 與 **hold（快角）** 兩個 view，不能只用一組 corner
- MMMC 設定一旦建立，**全程沿用**到 Route 結束，不會每階段重建

```tcl
create_delay_corner -name corner_max -library_set libs_max -rc_corner rc_corner   ;# 慢角=setup
create_delay_corner -name corner_min -library_set libs_min -rc_corner rc_corner   ;# 快角=hold
set_analysis_view -setup {view_setup} -hold {view_hold}
```

setup 用慢角（延遲最長）、hold 用快角（延遲最短），符合「setup 抓最壞情況、hold 抓最壞情況」的物理直覺——這組設定要先建對，後面所有時序分析都靠它。

---

## Step 2：Floorplan — 要做什麼

決定 die／core 尺寸、長寬比、cell utilization，評估巨集擺放位置。

```tcl
floorPlan -r 0.75 0.7 100 100 100 100
```

`-r` 格式：aspect ratio、cell utilization、core 到 IO 四邊間距。

---

## Step 2：Floorplan — 要注意什麼

- Cell utilization 通常抓 **60–80%**——太高繞不進去，太低浪費面積
- 巨集盡量靠 die 邊緣、對齊擺放，中間留給標準單元與繞線通道
- 尺寸不是一次到位，通常先抓大概數字，看 placement／routing 結果再調整

```tcl
floorPlan -r 0.73 0.7 100 100 100 100    ;# 先試算
floorPlan -r 0.75 0.702385 100.94 100.44 100.32 100.24   ;# 看過壅塞/時序後再定案
```

---

## Step 2：補充 — Floorplan Resize 示意

同樣數量的 cell，把 core 面積放大，utilization 自然下降，繞線空間就變寬鬆：

```
Resize 前（core 太小）              Resize 後（core 放大）
┌─────────────────┐                ┌───────────────────────┐
│▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤│                │▤▤▤▤▤▤▤▤▤▤▤▤            │
│▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤│   ──resize──▶  │▤▤▤▤▤▤▤▤▤▤▤▤            │
│▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤│                │▤▤▤▤▤▤▤▤▤▤▤▤            │
└─────────────────┘                └───────────────────────┘
utilization ≈ 90%（繞不進去）        utilization ≈ 70%（留白給繞線）
```

**要注意**：resize 不是越大越好——core 太大會拉長平均線長、增加面積成本；太小又繞不進去，要在「繞得進去」跟「面積夠省」之間找平衡點，通常靠壅塞熱圖／時序報告來判斷要不要 resize。

---

## Step 3：Power Planning — 要做什麼

建立電源 ring（環）→ stripe（網格）→ rail（標準單元電源軌），三層電源網路。

```tcl
addRing -nets {VDD VSS} -type core_rings -layer {top M5 bottom M5 left M6 right M6} -width 7
```

---

## Step 3：補充 — 各層金屬在電源網路的角色

金屬層由下往上，線越粗、電阻越小，越適合承受大電流：

- **M1（最下層）**：standard cell 電源軌（rail），線最細，直接接每顆 cell 的電源腳
- **中層（M2–M4）**：以訊號 routing 為主，很少拿來走電源
- **次上層（如 M5／M6）**：線較粗、電阻較小，通常拿來做 power ring／stripe
- **最上層**：層數更多的製程才有，通常給全晶片級電源網格或很長距離的訊號

> 金屬層數是製程決定的上限，不是每個設計都會用滿——實際用到哪幾層，看設計規模與供電需求而定。

---

## Step 3：Power Planning — 要注意什麼

- `verifyConnectivity` **必須 0 error** 才能進入下一階段（浮接電源會讓後面所有時序分析失真）
- 巨集（尤其 RAM／ROM／PLL 這類硬巨集）通常要**額外加一圈 block ring**，不能只靠 core ring
- Stripe 密度要夠——依 IR drop／EM 需求決定數量與寬度

```tcl
addRing -nets {VDD VSS} -type block_rings -around selected   ;# 巨集額外包一圈
verifyConnectivity -type all -error 1000 -warning 50         ;# 收尾前一定要跑
```

---

## Step 3：解決手法 — IR Drop 太大怎麼辦

電源網路上某處電壓降（IR drop）太大，會拖慢附近邏輯時序，甚至造成誤動作。

- **加密 stripe**：密度不夠是最常見原因，補幾條上去
- **加寬 stripe／rail 線寬**：線越粗電阻越小，電壓降自然變小
- **針對熱點區域優先補強**（例如巨集附近耗電特別集中的地方）

```tcl
addStripe -nets {VDD VSS} -layer M5 -width 10 -set_to_set_distance 200   ;# 加密/加寬
```

---

<!-- _class: lead -->

# Step 4–6：擺放、時脈、繞線

---

## Step 4：Placement — 要做什麼

擺放標準單元與（未固定的）巨集，時序驅動＋壅塞驅動雙重最佳化，跑 pre-CTS 時序最佳化。

```tcl
place_opt_design
```

---

## Step 4：place_opt_design 常用設定

```tcl
setPlaceMode -congEffort high -timingDriven 1 -clkGateAware 1 \
             -powerDriven 0 -placeIOPins 0 -reorderScan 1
place_opt_design
```

- `-congEffort`（low／medium／high）：壅塞驅動強度，已知這顆設計容易壅塞就直接開 high，別等跑完才發現要重跑
- `-timingDriven 1`：邊擺邊看時序，避免關鍵路徑的 cell 被拉得太遠
- `-clkGateAware 1`：把 clock gating cell 跟它控制的那群暫存器擺近，減少額外 skew／繞線
- `-powerDriven`：多電壓域設計要打開，確保 cell 乖乖留在自己電源域的 voltage area 內，不會被排到別的域裡去
- `-placeIOPins 0`：IO 已經在 floorplan 固定好時要關掉，避免 placement 又把它挪動

---

## Step 4：Placement — 要注意什麼

- Congestion 熱圖有沒有大面積熱點——壅塞會拖累後面 routing
- Pre-CTS setup WNS 要**轉正**，DRC 要乾淨，才能進 CTS
- 有 scan chain 的設計記得 `reorderScan`，避免掃描鏈繞線暴長

```tcl
specifyScanChain scan1 -start IOPADS_INST/scanin/C -stop IOPADS_INST/scanout/I
setPlaceMode -congEffort high -timingDriven 1 -reorderScan 1
```

> Placement 通常要**反覆跑好幾次**才會收斂，很少一次到位。

---

## Step 4：解決手法 — Congestion 太高怎麼辦

Congestion 熱圖出現大面積紅色熱點，代表這區要繞的線比可用資源多，繞不進去。

- 提高 congestion effort，重跑 placement，讓工具更積極分散 cell
- 調整巨集擺放位置／方向，留出繞線通道（避免巨集把走道全部擋死）
- 壅塞太嚴重時要**回頭調整 floorplan**（加大 core、降低 utilization），不要硬凹 placement

```tcl
setPlaceMode -congEffort high
place_opt_design -incremental
```

---

## Step 4：補充 — Horizontal vs Vertical Congestion

Congestion 不是單一數字，而是**分方向、分金屬層**算的：

- 每個 GRC（Global Routing Cell）有四個邊，量測這個邊「需要幾條走線（demand）」vs「實際能提供幾條（supply）」
- 每層金屬都有預設走線方向（如奇數層水平、偶數層垂直），壅塞天生分成兩個方向：
  - **Horizontal congestion**：左右方向的走線資源夠不夠
  - **Vertical congestion**：上下方向的走線資源夠不夠
- 若 H／V 溢出比例差很多，通常代表 floorplan 長寬比或巨集擺放方向有問題（例如晶片被拉得很長很扁，某一個方向的通道天生較窄）

```tcl
report_congestion -grc_based -by_layer
```

---

## Step 5：CTS — 要做什麼

指定 clock buffer／inverter cell → 建 clock tree spec → 實際蓋出時脈樹、平衡 skew。

```tcl
ccopt_design
```

---

## Step 5：ccopt_design 常用設定

```tcl
set_ccopt_property buffer_cells {CLKBUFX1 CLKBUFX2 CLKBUFX4 CLKBUFX8}
set_ccopt_property inverter_cells {CLKINVX1 CLKINVX2 CLKINVX4}
create_ccopt_clock_tree_spec
ccopt_design -cts
```

- `buffer_cells`／`inverter_cells`：限定蓋樹只能用哪些 cell，避免工具選到驅動力太強／太弱、不適合時脈樹的 cell
- `create_ccopt_clock_tree_spec`：先把「要蓋成什麼樣子」的規則存成 spec，之後可以重複套用或針對特定 clock domain 微調
- `ccopt_design -cts`：只跑 CTS 這個階段（`ccopt_design` 不加參數預設會連後面最佳化一起跑）

---

## Step 5：CTS — 要注意什麼

- **Hold time 要等 CTS 做完才能精確算**——CTS 前的 hold 檢查都不準
- 除了 skew，也要看 **DRV**（max transition／max capacitance／max fanout）有沒有超標
- 特定 clock domain 有問題時，可以只針對它做「by-item」局部重跑，不用整個 CTS 重來

```tcl
timeDesign -postCTS -drvReports -slackReports -outDir timingReports
timeDesign -postCTS -hold -slackReports -outDir timingReports
```

`-drvReports` 把 DRV 違規（max cap／tran／fanout）另外拉一份報告，跟時序分開看。

---

## Step 5：解決手法 — Skew 怎麼修

- `ccopt_design` 本質上就是在做「skew balancing」：自動調整每條分支的 buffer 尺寸與插入位置，讓每個暫存器收到時脈的時間盡量一致
- Skew 壓不下來時，可以直接設定明確的 **target skew** 目標，讓工具知道要修到多緊
- 關聯緊密的一群暫存器（例如同一條資料通路）可以獨立分成 **skew group**，要求彼此之間的 skew 比全域目標更緊
- 少數分支真的修不動時，可以用 **useful skew**（刻意讓 capture 端稍微晚到）換取更多可用時間，幫忙修 setup

```tcl
set_ccopt_property target_skew 0.1   ;# 設定明確的 skew 目標
ccopt_design -cts                     ;# 重跑，讓 buffer 尺寸/位置重新平衡
```

---

## Step 5：解決手法 — DRV 怎麼修

- `ccopt_design` 本身就會自動調整 buffer 尺寸、插入額外緩衝級來修 DRV，不用手動介入
- 時脈這種高扇出訊號，還可以套用 **NDR（非預設走線規則）**：加寬線寬降低電阻、加大線距減少串擾
- 修完記得重新跑一次 `timeDesign -drvReports` 確認真的清零

```tcl
set_clock_tree_options -routing_rule my_ndr   ;# 時脈套用加寬線寬/加大線距的規則
ccopt_design -cts                              ;# 重跑一次，順便修 DRV
```

---

## Step 6：Routing — 要做什麼

Global routing（分格子、抓大致路徑）→ Detail routing（畫出精確金屬線與 via）。

```tcl
routeDesign -globalDetail -viaOpt -wireOpt
```

---

## Step 6：Routing — 要注意什麼

- Routing 是「繞線 → 檢查 DRC／antenna → 再繞線」不斷疊代，不是跑一次就結束
- **Antenna 違規**（金屬走線過長累積電荷）要收斂到 0，通常靠跳層修復
- 大量疊代後若時序嚴重劣化，代表 placement／CTS 需要回頭調整，不要死磕 routing
- 少數違規路徑可以用**局部 ECO** 修正，不用整個階段重跑

```tcl
verifyProcessAntenna -report DESIGN.antenna.rpt -error 1000
ecoChangeCell -inst <hold_violating_reg> -downsize   ;# 局部修 hold，不動其他已收斂部分
```

> Routing 相關指令在一顆設計裡經常被呼叫上百次——這是常態，不是設計出了問題。

---

## Step 6：解決手法 — Antenna 違規怎麼修

金屬走線過長，蝕刻過程中會像天線一樣累積電荷，打穿閘極氧化層。兩種常見修法：

- **跳層（layer jumping）**：讓連到閘極的線提早跳到上層金屬，蝕刻低層時暴露面積變小——對既有繞線改動最小，router 預設就是用這招自動修
- **插二極體（diode）**：在受影響的網路上接一顆反向二極體到 GND，把多餘電荷導走，但需要額外空間放二極體 cell

```tcl
routeDesign -globalDetail -viaOpt   ;# 重跑一次，router 會自動嘗試跳層修 antenna
```

---

<!-- _class: lead -->

# Step 7–9：收尾、驗收、交付

---

## Step 7：DFM — 要做什麼

插入 filler cell 填滿 cell row 空隙，插 dummy metal fill 滿足金屬密度規則。

```tcl
addFiller -doDRC
addMetalFill
```

---

## Step 7：DFM — 要注意什麼

- Filler 要維持 N/P well 與電源軌連續，不能留空隙
- 量產設計還會插 **well tap**（防 latch-up）、**end cap**（保護 row 邊界）、**decap**（穩壓）
- 這些不是每個練習案例都會做，但正式產品線通常缺一不可

---

## Step 8：Verification／Sign-off — 要做什麼

三項驗證缺一不可：**DRC**（幾何規則）、**LVS 概念**（連接性）、**Antenna**（天線效應），加上 timing sign-off。

```tcl
verifyGeometry
verifyConnectivity -type all
```

---

## Step 8：Verification／Sign-off — 要注意什麼

- Sign-off 用的分析設定比平時疊代**更嚴謹**——要開 **OCV derate**、用真實寄生參數，不能只用平時疊代的寬鬆設定
- 嚴謹的產線流程還會**換一套獨立工具**重新檢查一次（DRC 用 Hercules／Calibre、timing 用 PrimeTime），不能只信任 P&R 工具自己的估算
- Hold 幾乎壓線時，下一步通常是針對那條路徑做局部 `optDesign -postRoute -hold`，不用整個階段重跑

```tcl
setAnalysisMode -analysisType onChipVariation   ;# 通常到 Route 階段才第一次開啟
optDesign -postRoute -hold                      ;# 精修壓線的 hold 路徑
```

---

## Step 8：解決手法 — Setup／Hold 怎麼修

- **Setup 違規**（訊號到得太慢）：`-upsize` 換驅動力更強的同功能 cell，或插 buffer 減少長線負載延遲
- **Hold 違規**（訊號到得太快）：`-downsize` 換驅動力較弱、延遲較大的 cell，或插 buffer 墊高延遲
- 兩者都是**局部 ECO**：只動違規那幾顆 cell，其餘已收斂的部分完全不受影響，不用整個階段重跑

```tcl
ecoChangeCell -inst <late_reg>  -upsize     ;# 修 setup
ecoChangeCell -inst <early_reg> -downsize   ;# 修 hold
```

---

## Step 8：完整 Sign-off 結果範例

| 檢查項 | 結果 |
|---|---|
| DRC | **0** |
| `verifyConnectivity` | **0 errors** |
| `verifyProcessAntenna` | **0 violations** |
| Setup WNS | **0.022 ns**（0 違規） |
| Hold WNS | **-0.000 ns**（1 條微幅違規） |

三項驗證 + timing sign-off **全部通過才算完成**；hold 幾乎壓線，代表這種結果背後往往需要一輪局部 ECO（downsize／upsize cell）才能收斂。

---

## Step 9：Data Export — 要做什麼

設定分析模式（如 `bcwc`），輸出 SPEF（真實寄生參數）、SDF（延遲）、post-APR 網表、GDSII 版圖、LEF abstract。

```tcl
streamOut ${TOP_DESIGN}.gds -mapFile ... -mode ALL
```

---

## Step 9：Data Export — 要注意什麼

- SPEF 是給獨立 sign-off STA 工具用的**真實**寄生資料，不是估算值
- GDSII 是實際送晶圓廠的檔案，輸出前務必確認前面所有 sign-off 都已通過
- 交付清單：GDS／SDF／SPEF／post-APR 網表／LEF abstract 缺一不可

```tcl
rcOut -spef_file ${TOP_DESIGN}.spef
write_sdf ${TOP_DESIGN}.sdf
```

---

<!-- _class: lead -->

# 總結

---

## 各階段檢查清單總表

| # | 階段 | 完成判斷 |
|---|---|---|
| 1 | Design Import | 無 load error，MMMC 涵蓋 setup/hold |
| 2 | Floorplan | 面積、utilization、aspect ratio 合理 |
| 3 | Power Planning | `verifyConnectivity` 0 error |
| 4 | Placement | Pre-CTS WNS 轉正、DRC 乾淨 |
| 5 | CTS | Post-CTS setup/hold 收斂、DRV 無超標 |
| 6 | Routing | DRC＝0、antenna＝0 |
| 7 | DFM | 密度規則通過 |
| 8 | Verification | DRC／LVS／Antenna／Timing 全過 |
| 9 | Data Export | 交付檔案齊全（GDS/SDF/SPEF/網表） |

---

## 貫穿全流程的一件事：疊代收斂循環

```
① STA 跑時序分析（找違規）
        ↓
② 局部 ECO 修正（downsize/upsize、插 buffer）
        ↓
③ APR 落實改動（只動受影響的局部）
        ↓
   再跑 STA 驗證 —— 還有違規？→ 回到 ①
        ↓
      收斂完成
```

---

## 為什麼這個循環重要

九個步驟裡每一次「反覆疊代」——Placement 跑好幾輪、Routing 呼叫上百次、CTS by-item 微調——本質上都是同一個循環：

**跑分析 → 找問題 → 局部修正 → 再驗證**，不斷重複直到全部違規清零。

> 真實 APR 流程不是九步驟走一次就結束，而是「做完 → 檢查 → 不合格就疊代重做」的持續循環。

---

<!-- _class: lead -->

# Thank You

Q&A
