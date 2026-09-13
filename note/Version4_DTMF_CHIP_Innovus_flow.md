# Version4 實作紀錄：DTMF_CHIP Innovus 180nm APR 流程

> 資料來源：`~/Downloads/2025Innovus180製程實作casse/Version4/`（Cadence Innovus 存檔 checkpoint 資料夾）。本筆記透過解壓每個 `.inn.dat/inn.cmd.gz`（Innovus 指令歷史紀錄）與 `.metric.gz`（各階段 QoR 數據）還原出實際做了哪些操作、得到什麼結果，並非手動輸入的紀錄。
>
> 專有名詞維持英文：**floorplan**、**placement**、**routing**、**checkpoint**、**timing**、**slack**。與 [[floorplan]]、[[placement]]、[[routing]]、[[STA]] 筆記中的教科書理論可互相對照。

---

## 0. 專案基本資訊

| 項目 | 內容 |
|---|---|
| 設計名稱 | `DTMF_CHIP` |
| 使用工具 | Cadence Innovus 20.10-p004_1（Linux） |
| 製程 | 180nm（`setDesignMode -process 180`），standard cell site 名稱 `tsm3site` |
| 使用者 / 主機 | `kychen` @ `localhost.localdomain`，工作路徑 `~/cadence/cases/Practice_1/` |
| 流程效率設定 | `setDesignMode -flowEffort express`、`-earlyClockFlow true` |
| 時間軸 | 10/31 Floorplan → 11/1–11/2 Placement + CTS(初版) + Route(初版嘗試) → 11/4 CTS 補做 + Route 重跑 → 11/5 Metal Fill 收尾 |

資料夾（checkpoint）與實際流程順序對照：

```
Floorplan/01Floorplan_set → 02PowerRing → 03PowerStripe → 04PowerRail
        ↓
Placement/01Placement（存檔時機＝剛設完 designMode，place 指令尚未下）
        ↓
CTS/clk_tree → 02CTS（建立 clock tree spec）→ 03CCOPT（跑完 ccopt_design）
        ↓
   ┌───────────────┴───────────────┐
   │ (11/2 舊分支，已被取代)         │ (11/4 正式分支)
   │ Route/01Route → 02Fix_done     │ CTS/04ByitemCCOPT_1103（回頭補做CTS）
   │  （手動修 DRC，最後沒有續做）    │      ↓
   └───────────────                 │ Route/01Route（覆蓋重跑）
                                     │      ↓
                                     │ Route/03Byitemopt_routing
                                     │      ↓
                                     │ Route/04Antenna（修 antenna 違規）
                                     │      ↓
                                     │ Route/05MetalFill（最終交付）
                                     └───────────────
```

> `Route/02Fix_done.inn.dat` 的修改時間是 11/2，比 `CTS/04ByitemCCOPT_1103`（11/4）還早，而 11/4 那次是直接從 `03CCOPT` 分支出去，並沒有經過 `02Fix_done`。代表學生第一次繞完線後手動修 DRC 到 `02Fix_done`，後來覺得時序還能更好，回頭又補做了一輪 CTS 微調，再重新繞線一次，最後走到 `05MetalFill` 收尾。**`05MetalFill` 才是這個 Version 最終、有收斂的結果**；`02Fix_done` 是被放棄的舊嘗試,留著僅供對照。

---

## 1. Floorplan（`Floorplan/01~04`）

| 步驟 | 指令 | 說明 |
|---|---|---|
| 建立 core/die | `floorPlan -fplanOrigin center -site tsm3site -r 0.73 0.7 100 100 100 100`（試算）→ `floorPlan -site tsm3site -r 0.75 0.702385 100.94 100.44 100.32 100.24`（定案） | `-r` 格式為 `aspect_ratio cell_utilization core→IO(左/下/右/上)`。定案值：高寬比 0.75、cell utilization ≈70.2%、core 到 IO 四邊間距約 100μm |
| 加 power ring | `addRing -nets {VDD VSS} -type core_rings -layer {top/bottom Metal5, left/right Metal6} -width 7 -spacing 1 -offset 42.5`；再對巨集加 `-type block_rings`（width 7 / spacing 1 / offset 1） | Core 外圈與巨集周圍各加一圈 VDD/VSS ring，M5 走水平、M6 走垂直 |
| Pad 供電 | `setSrouteMode -viaConnectToShape {ring}` + `sroute -connect {padPin padRing} ...` | 把 IO pad 的電源腳接到 power ring |
| 加 power stripe | 多次 `addStripe -nets {VDD VSS} -layer Metal5/Metal6 -width 7 -spacing 1 -set_to_set_distance 264` | 水平（M5）＋垂直（M6，多條不同 offset）電源網格，補足 ring 中間的供電密度 |
| 標準單元 rail | `sroute -connect {blockPin padPin padRing corePin floatingStripe} -layerChangeRange {Metal1 Metal6}` | 把電源網格一路連到 M1 的 standard cell rail、macro 電源腳 |
| 驗證 | `verifyConnectivity -type all -error 1000 -warning 50` | 確認 P/G 網路無斷點 |

**這一階段做的事**：定義晶片尺寸與 cell 密度目標 → 建立 VDD/VSS 環與網格（M1–M6）→ 把所有巨集、標準單元的電源腳都接上電源網路，並用 `verifyConnectivity` 驗證電源網路完整。對照 [[floorplan]] 筆記 3.5 節的 PNS（Power Network Synthesis）觀念，這裡的 `addRing` + `addStripe` + `sroute` 就是 Innovus 版的 ring/strap/rail 三層電源網路建立。

---

## 2. Placement（`Placement/01Placement` → 實際 place 動作接續在 CTS 存檔的歷史裡）

`01Placement.inn` 存檔時機其實是**剛設完 `setDesignMode`、place 指令都還沒下**的那一刻；真正的 placement 動作是在同一個 Innovus session 繼續往下做、直到存 `clk_tree.inn` 之前才發生的。實際做了：

1. **匯入 scan chain 資訊**
   ```tcl
   defIn scan_input_1.def
   specifyScanChain scan1 -start IOPADS_INST/Pscanin1ip/C -stop IOPADS_INST/Pscanout1op/I
   specifyScanChain scan2 -start IOPADS_INST/Pscanin2ip/C -stop IOPADS_INST/Pscanout2op/I
   ```
   代表網表裡已經有 DFT scan chain，這裡用 DEF 檔指定兩條 scan chain 的起訖 pad，供後續 placement 時做 scan reorder。
2. **設定 placement 模式並執行**
   ```tcl
   setPlaceMode -congEffort high -timingDriven 1 -clkGateAware 1 -powerDriven 0 \
                -ignoreScan 1 -reorderScan 1 -placeIOPins 0 ...
   place_opt_design      # 共執行 5 次（含最後 place_design -noPrePlaceOpt -incremental 微調）
   ```
   `place_opt_design` 是「place + pre-CTS optDesign」合一指令，一輪包含 legalize + timing 最佳化，總共跑了 5 輪去收斂。
3. **收斂結果**（取最後一輪 pre-CTS 數據）：

   | 指標 | 數值 |
   |---|---|
   | Standard cell 數 | 5,561 顆 |
   | Net 數 | 5,914 |
   | 固定巨集 | 4 個、IO 71 個 |
   | Cell utilization | ≈76.5–77.5%（各輪之間微調） |
   | 總繞線長度估算 | ≈2.16×10⁵ ~ 2.39×10⁵ μm |
   | Pre-CTS setup WNS | 全部為正值（0.04–0.15ns 之間），0 條違規路徑（228 條路徑全部收斂） |
   | `verifyGeometry`（DRC） | cell / same-net / wiring 違規皆為 0，乾淨 |

**這一階段做的事**：把 scan chain 接上、用時序 + 壅塞雙重驅動的 placement 反覆跑了 5 輪，確認 pre-CTS 時序沒有 setup 違規、DRC 乾淨後才進入 CTS。對照 [[placement]] 筆記裡的 placement 收斂概念。

---

## 3. CTS（`CTS/clk_tree` → `02CTS` → `03CCOPT` → `04ByitemCCOPT_1103`）

| Checkpoint | 存檔時機 | 關鍵指令 |
|---|---|---|
| `clk_tree.inn` | Placement 剛做完，CTS spec 還沒建 | （承接 placement 結果的中繼存檔） |
| `02CTS.inn` | 建好 CTS spec，但**還沒跑合成** | `set_ccopt_property buffer_cells {CLKBUFX1 CLKBUFX2 ... CLKBUFXL}`<br>`set_ccopt_property inverter_cells {CLKINVX1 ... CLKINVXL}`<br>`create_ccopt_clock_tree_spec` |
| `03CCOPT.inn` | 跑完 clock tree 合成 | `ccopt_design`（完整跑一次）+ 兩次 `ccopt_design -cts`（局部重跑 CTS stage） |
| `04ByitemCCOPT_1103.inn` | 11/4 回頭補做的微調（比 `03CCOPT` 晚兩天） | 再加一次 `ccopt_design -cts`，並多次用 `timeDesign -postCTS ...` / `-preCTS ...` 匯出報告到 `timingReports/` 資料夾比對 |

**這一階段做的事**：
1. 先指定可以用來做 clock buffer/inverter 的 cell（`CLKBUFX*`、`CLKINVX*` 系列），建立 clock tree spec。
2. 用 `ccopt_design` 實際長出 clock tree（插入 buffer、平衡 skew）。
3. 之後又跑了一次額外的 `ccopt_design -cts`（就是資料夾名稱「ByitemCCOPT」＝逐項/局部再最佳化 clock tree 的意思），並大量使用 `timeDesign -postCTS/-preCTS -pathReports -drvReports -slackReports` 把 timing 報告寫檔比對前後差異。

對照 [[STA]] 筆記，這裡的 `ccopt_design`＝ concurrent clock and data optimization，是 Innovus 用來同時處理 clock skew 與 data path timing 的引擎；`timeDesign` 系列則是各階段的 timing 抽測（sign-off 用 `-postRoute`,平時檢查用 `-preCTS`/`-postCTS`）。

---

## 4. Route（`Route/01Route` → `03Byitemopt_routing` → `04Antenna` → `05MetalFill`）

> 如前所述，`02Fix_done` 是 11/2 的舊嘗試，未併入最終流程，故不列入下表主線。

| Checkpoint | 關鍵指令 | 這一步做了什麼 |
|---|---|---|
| `01Route.inn` | `routeDesign -globalDetail` | 一次做完 global + detail routing 的初版繞線 |
| （中間反覆優化） | `routeDesign -globalDetail -viaOpt -wireOpt`（共 10 餘次）、`globalDetailRoute` | 反覆做 via optimization + wire optimization，修 DRC／減少壅塞，整個 Route 資料夾裡 `routeDesign` 類指令總共呼叫了 **103 次**，`verifyConnectivity` 呼叫了 **74 次** — 顯示這是不斷「繞線 → 檢查 → 再繞線」的疊代收斂過程 |
| `03Byitemopt_routing.inn` | 承上 | 逐條網路（by-item）做繞線最佳化，處理個別違規 net |
| `04Antenna.inn` | `verifyProcessAntenna -report DTMF_CHIP.antenna.rpt -error 1000`（呼叫 **27 次**） | 反覆檢查/修 antenna 違規（金屬長天線效應），透過反覆 routeDesign 疊代讓 antenna 違規數從有到收斂為 0 |
| `05MetalFill.inn`（**最終版**） | `addFiller -prifix -doDRC`（filler cell，含 DRC 檢查）→ `addMetalFill`（dummy metal fill） | 補標準單元列間的 filler cell，並填 dummy metal 滿足金屬密度規則,完成收尾 |

**最終（`05MetalFill`）收斂結果**：

| 項目 | 數值 |
|---|---|
| Setup timing（`timeDesign -postRoute`） | WNS(all) = **0.022 ns**、TNS = 0.000 ns、228 條路徑、**0 條違規** |
| Hold timing（`timeDesign -postRoute -hold`） | WNS(all) = **-0.000 ns**（幾乎壓線）、228 條路徑中 **1 條**違規（僅 reg2reg 群組，數值極小接近 0） |
| `verifyConnectivity` | **0 errors**（電性/繞線連通性乾淨） |
| `verifyProcessAntenna` | **0 violations**（antenna 違規已全部修完） |
| DRC（`routeDesign.DRC.total`） | **0** |
| 總繞線長 | 320,640 μm（M1 23,049 / M2 97,520 / M3 120,783 / M4 79,288 μm，M5/M6 保留給電源網路未用於訊號） |
| Via 總數 | 48,327（Via12 23,363 / Via23 19,244 / Via34 5,720） |
| Cell utilization（post-route） | 66.8% |

**結論**：Setup 幾乎壓線通過（0.022ns），Hold 有一條路徑幾乎踩線（−0.000ns），實務上這樣的結果代表時序已經收斂但非常緊繃 — 如果要在 Innovus 裡繼續優化，可以從 `05MetalFill.inn` 這個 checkpoint 繼續下手，針對那條 hold 違規路徑跑 `optDesign -postRoute -hold` 做 hold fix。

---

## 5. 想在 Innovus 重現這些步驟時的操作順序

1. `source Floorplan/01Floorplan_set.inn` 還原初始 floorplan，練習 `addRing`/`addStripe`/`sroute`。
2. `source Placement/01Placement.inn`，接著自己動手下 `place_opt_design`，比較和上面第 2 節數據是否一致。
3. `source CTS/02CTS.inn`，練習下 `ccopt_design`，比對能否重現 `03CCOPT` 的 timing 數字。
4. `source Route/01Route.inn`，練習 `routeDesign -globalDetail -viaOpt -wireOpt` 疊代收斂，最後跑 `verifyProcessAntenna` 與 `addFiller`/`addMetalFill` 收尾，比對能否重現 `05MetalFill` 的最終 QoR。

（`.inn` 的 `source` 用法與各 checkpoint 內容已封裝完整 lib/mmmc，詳見前一則對話的說明。）
