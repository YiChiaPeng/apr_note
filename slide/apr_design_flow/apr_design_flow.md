---
marp: true
theme: default
paginate: true
size: 16:9
footer: 'APR Design Flow — RTL to GDS'
style: |
  section {
    font-size: 26px;
  }
  section.lead h1 {
    font-size: 60px;
  }
  h1 { font-size: 36px; }
  h2 { font-size: 28px; color: #1a5276; }
  table { font-size: 20px; }
  code { font-size: 0.85em; }
  pre { font-size: 18px; }
---

<!-- _class: lead -->

# APR Design Flow
## RTL-to-GDS 後端設計流程

從 Floorplan 到 GDS：Cadence Innovus 實作步驟詳解

---

## Agenda

1. 什麼是 APR？在整個晶片設計流程中的位置＋名詞速查表
2. RTL-to-GDS 九大步驟總覽
3. 兩個對照案例：`gcd`（教學範例）vs `DTMF_CHIP`（180nm 完整實作）
4. 逐步驟詳解（含示意圖）：Design Import → Floorplan → Power Planning → Placement → CTS → Routing → DFM → Verification → Data Export
5. 動手練習：如何用 checkpoint 驗證自己真的理解每個階段
6. 各階段檢查清單總表
7. 進階補充：MMMC／OCV 設定（可跳過）

> 資料來源：`note/floorplan.md`、`note/placement.md`、`note/routing.md`、`note/STA.md`、`reference_design/gcd/`、`reference_design/180um/Version4`

---

## 什麼是 APR？

**APR（Automatic Place and Route）** = 後端物理設計，把合成後的**閘級網表（gate-level netlist）**轉換成可以送晶圓廠的**版圖（GDSII）**。

輸入：
- 合成後網表（netlist）＋ SDC 時序約束 ＋ LEF/Lib 製程檔

輸出：
- GDSII 版圖、SDF 延遲檔、post-APR 網表、LEF abstract

核心矛盾：**面積、時序、可繞線性（routability）、功耗**四者互相牽制，整個流程就是反覆疊代收斂這四個指標。

---

## APR 在整個晶片設計流程中的位置

```
[  RTL 設計  ]      [ 邏輯合成 Synthesis ]      [   APR（本簡報範圍）   ]      [ Sign-off ]      [ Tapeout ]
Verilog/VHDL   →   RTL 轉成 gate-level     →   Floorplan ~ Routing    →   最嚴謹的最終   →   送晶圓廠
描述電路行為        netlist（用邏輯閘拼         決定每顆 cell 的實際        時序/DRC 驗收      生產光罩、
                   出來的電路，還沒有           座標、每條線怎麼走           （下頁解釋）        製造晶片
                   座標、還沒繞線）
```

- 合成產出的 **netlist（網表）** 只描述「用哪些邏輯閘、怎麼連接」，還是抽象的邏輯關係
- **APR** 把網表變成實際的 **版圖（layout / GDSII）**：每顆標準單元放在哪裡、每一條金屬線怎麼繞
- APR 做完要通過 **sign-off**（最嚴謹的驗收）才能 **tapeout**（送晶圓廠生產）—— 這兩個詞下一頁就會解釋

---

## 名詞速查表（後面會一直用到）

| 名詞 | 白話解釋 |
|---|---|
| Netlist（網表） | 合成後「用哪些邏輯閘、怎麼連接」的電路描述，還沒有座標與繞線 |
| Corner（角落） | 製程／電壓／溫度的一種組合條件（如「慢＋低壓＋高溫」），設計要在多個 corner 下都過關 |
| OCV（On-Chip Variation） | 考慮同一顆晶片上不同位置也會有微小製程差異的分析方式，比單純 corner 更保守 |
| **Sign-off** | 用最嚴謹的分析標準做的正式驗收，通過才代表這批結果可信賴、能繼續往下走 |
| **ECO**（Engineering Change Order） | 只針對少數幾個 cell 做最小幅度修改解決特定問題，不必重跑整個階段 |
| WNS／TNS | 最壞路徑的違規時間／所有違規路徑加總，越正（或越接近 0）代表時序越健康 |
| Skew | 時脈訊號到達不同暫存器的時間差，CTS 的目標之一就是壓低它 |
| Congestion（壅塞） | 某區域要繞的線太多、資源不夠，容易繞不進去或繞很繞 |
| IR Drop／EM | 電源網路壓降過大／金屬線電流密度過高，兩者都是電源設計要顧慮的可靠度問題 |
| Tapeout | Sign-off 通過後，把最終版圖資料正式送交晶圓廠準備生產 |

> 這些詞後面章節會直接用到，忘記意思可以回來查這一頁。

---

## RTL-to-GDS 九大步驟總覽

| # | 階段 | 目的 |
|---|---|---|
| 1 | Design Import | 匯入網表、約束、製程檔 |
| 2 | Floorplan | 決定晶片/核心尺寸、IO 位置 |
| 3 | Power Planning | 建立電源 ring／stripe／rail |
| 4 | Placement | 擺放巨集與標準單元 |
| 5 | CTS | 合成時脈樹、平衡 skew |
| 6 | Routing | 訊號線 global + detail routing |
| 7 | DFM | Filler cell、dummy metal fill |
| 8 | Verification | DRC、LVS、Antenna 檢查 |
| 9 | Data Export | 輸出 GDS、SDF、網表給下游 |

> 每一步都伴隨著「做完 → 檢查時序/DRC/壅塞 → 不合格就疊代重做」的迴圈，不是單向線性流程。

---

<!-- _class: lead -->

# 兩個對照案例

---

## 案例一：`gcd`（教學範例）

- 路徑：`reference_design/gcd/scripts/gcd_soce.tcl`
- 設計：`gcd`（Greatest Common Divisor，最大公因數電路）— 小型教學電路
- 製程：90nm（`setDesignMode -process 90`）
- 工具：Cadence Innovus（前身 SoC Encounter，檔名 `soce` 即此縮寫）
- **特色**：九個步驟各只跑「一次」，關掉時序驅動/SI 驅動 routing，是最簡化的示範流程，方便一次看完全貌

## 案例二：`DTMF_CHIP`（180nm 完整實作）

- 路徑：`reference_design/180um/Version4/`（實際 checkpoint 資料庫）
- 設計：`DTMF_CHIP`，180nm 製程（`setDesignMode -process 180`）
- **特色**：真實課程作業，每個階段都反覆疊代很多輪才收斂 —— 是本簡報「每階段實際要做什麼」的第一手證據

---

<!-- _class: lead -->

# Step 1 — Design Import

---

## Step 1：Design Import（設計匯入）

**這階段要做什麼：**
- 匯入合成後的閘級網表（netlist）
- 讀入 SDC 時序約束
- 指定 LEF（製程幾何）與 Lib（時序）檔案
- 建立 MMMC（Multi-Mode Multi-Corner）多角多模分析設定
- 讀入 IO 接腳位置約束

> 對照 `note/floorplan.md` 附錄 Step.1；ICC 對應指令為 `create_mw_lib`、`import_designs`、`read_sdc`。

---

## Step 1：Design Import — `gcd` 範例

```tcl
set TOP_DESIGN "gcd"
loadConfig ../design_data/${TOP_DESIGN}.conf 1
```
`gcd.conf` 一次集中定義了 netlist、SDC、LEF、IO 檔、MMMC view 檔與 floorplan 預設參數；`gcd.view` 用 `create_library_set`／`create_analysis_view` 建立 slow.lib(setup 角)／fast.lib(hold 角) 的 MMMC 設定。

> DTMF_CHIP 實際用的完整 MMMC corner／OCV 設定較深入，整理在簡報最後「進階補充」章節。

---

<!-- _class: lead -->

# Step 2 — Floorplan

---

## Step 2：Floorplan 理論重點

**這階段要做什麼：**
- 決定晶片 **die / core** 尺寸（面積、長寬比）
- 決定 **cell utilization**（標準單元密度目標，通常 60–80%）
- 決定 core 到 IO 的邊界間距
- 評估巨集（macro）擺放位置（若有）
- 目標：面積最小、繞線總長最短、關鍵路徑延遲最小、routing 成功率最高

> 詳見 `note/floorplan.md` 3.1–3.2 節（ICC 用 `initialize_floorplan`）

---

## Step 2：示意圖 — Floorplan 長什麼樣子

```
┌───────────────────────────────────────┐  ← Die（晶片最外框）
│ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ │  ← IO Pad（晶片對外接腳）
│ □┌─────────────────────────────────┐□ │
│ □│                                 │□ │
│ □│           Core 區域              │□ │  ← 真正放 std cell/macro 的地方
│ □│   （長寬比、cell utilization      │□ │
│ □│    是本階段要決定的數字）          │□ │
│ □│                                 │□ │
│ □└─────────────────────────────────┘□ │
│ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ □ │
└───────────────────────────────────────┘
        ↕ core 到 IO 的間距（本例約 100μm）
```

- **Die**：晶片實際的矽晶粒範圍
- **Core**：die 扣掉 IO 之後，真正拿來放邏輯電路的區域
- **Aspect ratio**：core 的長寬比（1＝正方形）
- **Cell utilization**：core 面積裡預計被 standard cell 填滿的比例（通常 60–80%，留白給繞線與後續調整空間）

---

## Step 2：Floorplan 實作對照

**`gcd` 範例（90nm，簡化）：**
```tcl
floorPlan -r 1 0.7 5 5 5 5
```
高寬比 1（正方形）、cell utilization 0.7（70%）、core 到四邊各 5μm。

**`DTMF_CHIP` 實際案例（180nm）：**
```tcl
floorPlan -site tsm3site -r 0.75 0.702385 100.94 100.44 100.32 100.24
```
高寬比 0.75、utilization ≈70.2%、core 到 IO 四邊約 100μm —— 學生**先試算一次**（`-r 0.73 0.7 100 100 100 100`）**再定案**，可見 floorplan 尺寸不是一次到位，通常會先抓大概數字、看後面 placement/routing 結果再調整。

---

## Step 2 補充：什麼是 Halo？

巨集放好位置之後、加電源環之前，通常會先在巨集四周留一圈「保留區」——這就是 **halo**（也叫 keepout margin）。目的是避免 standard cell 貼著巨集邊界放，導致巨集接腳附近繞線塞爆，也預留空間給接下來要加的 power ring／stripe。

```
┌ halo（保留區，四邊可各自設定寬度）──────────────┐
│                                                │
│        ┌──────────────────┐                   │
│        │       Macro       │                  │
│        │   (RAM/ROM/PLL)   │                  │
│        └──────────────────┘                   │
│                                                │
└────────────────────────────────────────────────┘
▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤  ← standard cell 不會被放進 halo 裡
```

---

## Step 2 補充：Halo 的真實案例

**DTMF_CHIP 真實指令**（Version5，`Floorplan/02MacroHaloset` checkpoint）：
```tcl
addHaloToBlock {15 15 15 15} DTMF_INST/ARB_INST/ROM_512x16_0_INST
addHaloToBlock {25 35 15 25} DTMF_INST/PLLCLK_INST
```
`{左 下 右 上}` 四個數字是四邊的保留寬度（μm）——指令歷史裡這組數字被反覆調整了近 10 次，代表 halo 寬度跟 floorplan 尺寸一樣，是「先抓大概、看壅塞再調整」的疊代過程，不是一次定案。

> 對照 `note/floorplan.md` 3.2.12–3.2.15 節（硬性／軟性 blockage、`set_keepout_margin`、`create_route_guide`）

---

## Step 2 補充：Place Halo vs Routing Halo

Halo 概念上分兩種用途，只是不同工具的實作方式不太一樣：

| | Place Halo（置放保留區） | Routing Halo（繞線保留區） |
|---|---|---|
| 擋什麼 | 阻止 standard cell 被放進這個範圍 | 限制某些金屬層的訊號線不能繞過這個範圍 |
| 為什麼要擋 | 避免 cell 貼著巨集邊界，接腳附近繞線空間不夠 | 幫巨集自己的電源環/接腳留出走線空間，或避免數位訊號干擾類比巨集 |
| ICC 對應指令 | `set_keepout_margin -type hard/soft -outer {左 下 右 上}` | `create_route_guide -no_signal_layer {METAL5 METAL6} -coordinate {...}` |
| Innovus 對應指令 | `addHaloToBlock {左 下 右 上} inst` | 同一個指令一次設定，沒有再分開下第二道指令 |

**DTMF_CHIP 真實案例只用了 `addHaloToBlock` 一種指令**——代表 Innovus 把 place halo 跟 routing halo 合併成同一個保留區設定，不像 ICC 拆成 `set_keepout_margin`（置放）＋`create_route_guide`（繞線）兩道指令；概念上仍是同一件事：**在巨集周圍留一圈「別人不能進來」的緩衝區**，保留給接下來的電源網路與訊號出線空間。

---

<!-- _class: lead -->

# Step 3 — Power Planning

---

## Step 3：Power Planning 理論重點

**這階段要做什麼：**
1. 建立電源/地訊號的**邏輯連接**（把所有 instance 的 VDD/VSS pin 接上對應 net）
2. 在核心區周圍建立 **power ring**（VDD/VSS 電源環）
3. 建立 **power stripe**（電源網格，補足 ring 中間的供電密度）
4. 用 **sroute** 把電源網格一路連到標準單元的 **rail**（M1 電源軌）
5. 用 `verifyConnectivity` 確認電源網路無斷點

> 詳見 `note/floorplan.md` 3.5 節 PNS（Power Network Synthesis）；ICC 對應 `derive_pg_connection` + `create_pad_rings`

---

## Step 3：示意圖 — 電源網路長什麼樣子

```
┌═══════════ VDD/VSS Power Ring（繞 core 一圈，M5/M6）═══════════┐
│  ┃          ┃          ┃  ← Power Stripe（M5/M6 網格，補中間密度）
│  ┃          ┃          ┃
│  ▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭  ← Standard cell row
│  ══════════════════════════  ← M1 Power Rail（每一排 cell 都有）
│  ▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭▭
│  ══════════════════════════
│  ┃          ┃          ┃
└═════════════════════════════════════════════════════════════┘
```

- **Ring**：繞著 core（與巨集）外圍的電源環，像主幹道
- **Stripe**：一整條一整條的電源網格，補足 ring 中間的供電密度，像次幹道
- **Rail**：每一排 standard cell 底下的 M1 電源軌，像街道，直接接到每顆 cell 的電源腳
- `sroute` 做的事：把 ring → stripe → rail 一路接起來，中間不能有斷點（否則 `verifyConnectivity` 會報錯）

---

## Step 3 補充：金屬層 M0–M8 是什麼

上面的 ring／stripe／rail 其實都是**選在不同的金屬層**上做的。晶片內部是一層一層疊上去的金屬導線，由下往上大致長這樣（側視／剖面）：

```
M8  ██████████████████████████  ← 最上層：全晶片級電源網格 / 很長距離的訊號（線最粗、電阻最小）
M7  ██████████████████████████
M6  ▓▓▓▓   ▓▓▓▓   ▓▓▓▓   ▓▓▓▓   ← DTMF_CHIP：Power Ring/Stripe（垂直）
M5  ▓▓▓▓   ▓▓▓▓   ▓▓▓▓   ▓▓▓▓   ← DTMF_CHIP：Power Ring/Stripe（水平）
M4  ─ ─ ─  訊號 routing（中／長距離）─ ─ ─
M3  ─ ─ ─  訊號 routing  ─ ─ ─
M2  ─ ─ ─  訊號 routing（cell 之間短距離連線）─ ─ ─
M1  ══════ Standard cell 電源軌（rail）＋短距離訊號 ══════          （線最細、電阻最大）
M0  ▮▮▮ 顆粒最細的 local interconnect（只有先進製程才有）▮▮▮
     ▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬▬
              [ 電晶體／Standard cell 本體 ]
```

- **由下往上**：線越細、間距（pitch）越小 → 越適合精密的短距離連線，但電阻大；線越粗、間距越大 → 電阻小、能承受大電流，適合長距離走線與電源
- **M0**：只有較先進製程（通常 <45nm）才會有的「local interconnect」層，專門把電晶體接到 M1；**180nm 這種較舊製程通常沒有 M0**，電晶體直接由 M1 接出
- **M1**：幾乎所有製程共通的最下層通用走線層，standard cell 的電源 rail、同排 cell 間短距離訊號都在這層

---

## Step 3 補充：中高層金屬實際怎麼用

- **M2–M4（中層）**：cell 之間、跨 row 的訊號 routing 主力——DTMF_CHIP 最終總繞線長 320,640μm **全部落在 M1–M4**
- **M5–M6（次上層）**：線較粗、電阻較小，DTMF_CHIP 拿來做 Power Ring／Stripe；`gcd` 案例則是把 core ring 放在 M4/M5
- **M7–M8（最上層）**：層數更多的製程才會用到，通常給全晶片級電源網格或很長的 global 訊號

**金屬層數是製程決定的，不是每個設計都用到全部**：`gcd` 使用的 90nm 製程實際上支援到 **M9**（`addMetalFill -layer {M1...M9}`、via 可以跨到 M8），但因為電路太小，實際只用了 M1/M4/M5；DTMF_CHIP 的 180nm 製程整個流程最高只用到 **M6**——製程給的層數是上限，設計用多少層看複雜度與供電需求。

---

## Step 3：Power Planning — `gcd` 範例

```tcl
clearGlobalNets
globalNetConnect VDD -type pgpin -pin VDD -inst * -module {}
globalNetConnect VDD -type tiehi -pin VDD -inst * -module {}
globalNetConnect VSS -type pgpin -pin VSS -inst * -module {}
globalNetConnect VSS -type tielo -pin VSS -inst * -module {}

addRing -nets {VDD VSS} -type core_rings -follow core \
  -layer {bottom M5 top M5 right M4 left M4} -width 2 -spacing 0.5 -offset 0.1

sroute -connect { corePin } -layerChangeRange { M1 M5 } \
  -nets { VDD VSS } -corePinTarget { firstAfterRowEnd }
```
`globalNetConnect` 建立邏輯連接 → `addRing` 建 M4/M5 電源環 → `sroute` 把環一路接到標準單元列。

---

## Step 3：Power Planning — `DTMF_CHIP` 實際案例

```tcl
addRing -nets {VDD VSS} -type core_rings -follow core \
  -layer {top Metal5 bottom Metal5 left Metal6 right Metal6} \
  -width 7 -spacing 1 -offset 42.5
addRing -nets {VDD VSS} -type block_rings -around selected ...   ;# 巨集周圍再加一圈

addStripe -nets {VDD VSS} -layer Metal5 -direction horizontal \
  -width 7 -spacing 1 -set_to_set_distance 264 ...
addStripe -nets {VDD VSS} -layer Metal6 -direction vertical  ...  ;# 多條不同 offset

sroute -connect {blockPin padPin padRing corePin floatingStripe} \
  -layerChangeRange {Metal1 Metal6} -nets {VDD VSS}
verifyConnectivity -type all -error 1000 -warning 50   ;# 確認電源網路乾淨
```
真實案例除了 core ring，**巨集**也額外加了一圈 ring，並用多條 stripe 補密度，比 `gcd` 範例複雜得多。

---

## Step 3 補充：什麼樣的巨集需要 Block Power Ring

不是每個巨集都要另外加 ring，通常符合以下特徵才會加：

- **硬巨集（hard macro）**：像 RAM／ROM／PLL 這種內部電路固定的 IP，內部沒有 standard cell 那種 rail 結構，需要一圈環把周邊電源接腳整合起來，才能穩定跟外部電源網路對接
- **耗電量大、電流密度高**：容量大的記憶體耗電相對集中，只靠 core 的 stripe 供電容易在巨集周圍造成 IR drop 過大，加一圈 block ring 能就近補強
- **獨立／類比電源域**：像 PLL 這種類比電路通常吃自己單獨一組 VDD/VSS（避免被數位開關雜訊干擾），需要專屬的 ring 而不是直接共用數位 core ring
- **離 core ring 較遠、位於晶片內部**：巨集若不是貼著 die 邊緣擺放，離主要電源環較遠，加 block ring 再往外接 stripe，比單靠核心 stripe 硬牽線更穩定

**DTMF_CHIP 案例對照**：從 MMMC 的 library set 就能看到這顆晶片有 `pllclk`（PLL）、`ram_128x16A`、`ram_256x16A`、`rom_512x16A` 幾個硬巨集——這正是前一頁 `addRing -type block_rings -around selected` 特別再包一圈的對象。

---

## Step 3：Power Planning 檢查清單

- [ ] 所有 instance 的電源腳都已邏輯連接（`globalNetConnect` / `derive_pg_connection`）
- [ ] Core ring 已建立，巨集若有獨立供電需求也建立 block ring
- [ ] Stripe 密度是否足夠（依 IR drop／EM 需求決定 stripe 數量與寬度）
- [ ] `sroute` 是否把 ring/stripe 一路接到 M1 標準單元 rail、macro pin
- [ ] `verifyConnectivity` **必須 0 error** 才能進入下一階段（floating power net 會讓後面所有時序分析失真）

---

<!-- _class: lead -->

# Step 4 — Placement

---

## Step 4：Placement 理論重點

**這階段要做什麼：**
- 擺放巨集（若未在 floorplan 固定）與標準單元
- 時序驅動（timing-driven）＋壅塞驅動（congestion-driven）雙重最佳化
- 處理 DFT：scan chain 的 `reorderScan`（依實體位置重新排列掃描鏈，減少繞線）
- Placement 前設定：忽略層、clock gating cell、tie cell 插入
- Placement 後跑 **pre-CTS optDesign**：先修 max cap / max transition，確認時序在 CTS 前就先收斂

> 詳見 `note/placement.md` 4.1、4.3、4.4 節

---

## Step 4：示意圖 — Placement 長什麼樣子

```
Core 區域內部（俯視）：
┌─────────────────────────────────────────────┐
│ ▤▤▤▤▤▤▤▤▤▤▤   ┌──────────┐   ▤▤▤▤▤▤▤▤▤▤▤ │  ← row 1
│ ▤▤▤▤▤▤▤▤▤▤▤   │  Macro   │   ▤▤▤▤▤▤▤▤▤▤▤ │  ← row 2
│ ▤▤▤▤▤▤▤▤▤▤▤   │ (RAM/ROM)│   ▤▤▤▤▤▤▤▤▤▤▤ │  ← row 3
│ ▤▤▤▤▤▤▤▤▤▤▤   └──────────┘   ▤▤▤▤▤▤▤▤▤▤▤ │  ← row 4
│ ▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤▤ │  ← row 5
└─────────────────────────────────────────────┘
每個 ▤ 是一顆 standard cell（AND/OR/暫存器…），一排一排貼著放
```

- **Macro**：像 RAM/ROM/PLL 這種面積大、形狀固定的巨集，通常先放好、擺放時不太挪動
- **Standard cell**：面積小、高度統一的邏輯閘/暫存器，一排一排（row）自動塞進剩下空間
- Placement 的目標：時序驅動＋壅塞驅動雙重最佳化，讓需要互相連線的 cell 盡量靠近，減少後面繞線的難度

---

## Step 4：Placement — `gcd` 範例

```tcl
setPlaceMode -fp false
placeDesign -prePlaceOpt
addTieHiLo -cell {TIELO TIEHI} -prefix LTIE
setOptMode -fixCap true -fixTran true -fixFanoutLoad false
optDesign -preCTS
```
`-fp false`＝扁平化（非階層化）placement；`placeDesign -prePlaceOpt` 一次做完「擺放＋前置邏輯最佳化」；`addTieHiLo` 插入固定高/低電位的 tie cell；`optDesign -preCTS` 做 CTS 前時序最佳化（先不修 fanout load 違例）。

---

## Step 4：Placement — `DTMF_CHIP` 實際案例

```tcl
defIn scan_input_1.def
specifyScanChain scan1 -start IOPADS_INST/Pscanin1ip/C -stop IOPADS_INST/Pscanout1op/I
specifyScanChain scan2 -start IOPADS_INST/Pscanin2ip/C -stop IOPADS_INST/Pscanout2op/I

setPlaceMode -congEffort high -timingDriven 1 -clkGateAware 1 \
             -ignoreScan 1 -reorderScan 1 -placeIOPins 0
place_opt_design       ;# 反覆執行 5 輪才收斂
```

| 收斂結果（最後一輪 pre-CTS） | 數值 |
|---|---|
| Standard cell 數 | 5,561 顆／Net 5,914 條 |
| Cell utilization | ≈76.5–77.5% |
| Pre-CTS setup WNS | 全部為正值（0.04–0.15ns），0 違規路徑 |
| DRC（`verifyGeometry`） | 0 違規，乾淨 |

**真實案例比教學範例多做的事**：先匯入 scan chain DEF、指定兩條 scan chain 起訖點，並用 `place_opt_design` 反覆跑 **5 輪**才收斂 —— 教學範例只跑一次是因為電路太小、沒有真實收斂壓力。

---

## Step 4 補充：checkpoint 陷阱

> `01Placement.inn` 的存檔時機其實是**剛設完 `setDesignMode`、`place_opt_design` 都還沒下**的那一刻；真正的 5 輪 placement 是在同一個 Innovus session 裡繼續往下做、直到存下一階段的 `clk_tree.inn` 之前才發生。

**checkpoint 檔名不代表「做完該步驟後」的狀態，要配合指令歷史（`inn.cmd.gz`）才能還原真實時間點**——這個提醒之後在 Step 1 補充（附錄）比對 MMMC 設定時還會再用到同一招。

---

## Step 4：Placement 檢查清單

- [ ] 巨集位置是否合理（是否需要手動調整/固定）
- [ ] Scan chain 是否已 reorder（`reorderScan`），避免繞線暴長
- [ ] Congestion 是否可接受（壅塞熱圖無大面積熱點）
- [ ] Pre-CTS timing：setup WNS 是否為正（或至少可控）
- [ ] DRC（`verifyGeometry`）：cell / same-net / wiring 違規是否為 0
- [ ] 若時序/壅塞不理想 → 回頭調整 placement 策略或 floorplan，**不要硬撐進 CTS**

---

<!-- _class: lead -->

# Step 5 — Clock Tree Synthesis（CTS）

---

## Step 5：CTS 理論重點

**這階段要做什麼：**
1. 指定可用的 clock buffer／inverter cell（如 `CLKBUF*`、`CLKINV*`）
2. 產生並套用 clock tree spec（`create_ccopt_clock_tree_spec`）
3. 執行 `ccopt_design`：插入緩衝器、平衡 skew、實際長出時脈樹
4. **post-CTS optDesign（setup）**：CTS 後修時序
5. **post-CTS optDesign -hold**：時脈樹確定後，hold time 才能被精確計算與修復（呼應 `note/STA.md` 「CT 插入後 hold 才能精確計算」）

> 詳見 `note/placement.md` 附錄 Step.5（教材第 5 章尚未整理成獨立筆記）

---

## Step 5：示意圖 — Clock Tree 長什麼樣子

```
                      [ Clock Source ]
                             │
                      ┌──────┴──────┐
                   Buffer         Buffer
                ┌─────┴─────┐  ┌─────┴─────┐
             Buffer       Buffer Buffer   Buffer
                │            │      │        │
               REG          REG    REG      REG   ...（一路分到全晶片上千顆暫存器）
```

- CTS 要做的事：從時脈源開始插入一層層 buffer，把時脈訊號「均勻地」送到每一顆暫存器
- 目標是讓每個暫存器收到時脈訊號的時間盡量一致（**skew** 越小越好），而不是像訊號線一樣抄最短路徑就好
- 時脈樹確定之後，**hold time 才能被精確計算**——這也是為什麼 hold 檢查要等 CTS 做完才有意義

---

## Step 5：CTS — `gcd` 範例

```tcl
set_ccopt_property buffer_cells { CLKBUF* }
set_ccopt_property use_inverters true
create_ccopt_clock_tree_spec -file ccopt.spec
source ccopt.spec
ccopt_design -cts
setOptMode -fixCap true -fixTran true -fixFanoutLoad true
optDesign -postCTS
optDesign -postCTS -hold
```
六個指令一次做完：指定 buffer → 產生規格 → 合成 → setup 最佳化 → hold 最佳化。

---

## Step 5：CTS — `DTMF_CHIP` 實際案例

真實案例把 CTS 拆成好幾個 checkpoint，反覆調整：

```
clk_tree.inn   （placement 剛做完，CTS spec 還沒建）
   ↓
02CTS.inn      （建好 spec，尚未跑合成）
   set_ccopt_property buffer_cells {CLKBUFX1 CLKBUFX2 ... CLKBUFXL}
   set_ccopt_property inverter_cells {CLKINVX1 ... CLKINVXL}
   create_ccopt_clock_tree_spec
   ↓
03CCOPT.inn    （跑完合成：ccopt_design + 2 次 ccopt_design -cts）
   ↓
04ByitemCCOPT_1103.inn  （兩天後回頭「逐項」再微調一次 ccopt_design -cts，
                          大量用 timeDesign -preCTS/-postCTS 匯出報告比對前後差異）
```

**重點**：真實案例的 CTS **不是跑一次就結束**，而是先建 spec 存檔、跑完合成再存檔、事後還會回頭針對特定 clock domain 再跑一輪「by-item」微調 —— 這是 CTS 常見的疊代模式。

---

## Step 5：CTS 檢查清單

- [ ] Buffer/inverter cell list 是否符合製程建議（避免用到不該用的高驅動力 cell）
- [ ] Clock tree spec 是否需要手動編修（真實設計很少直接用預設值）
- [ ] `ccopt_design` 後檢查 **skew**、**latency**、**insertion delay** 是否在合理範圍
- [ ] Post-CTS setup timing：WNS 是否轉正
- [ ] Post-CTS **hold** timing：這是本階段最重要的新檢查項（CTS 前無法準確算 hold）
- [ ] 若特定 clock domain 仍有問題 → 針對該 domain 做「by-item」局部重跑，不用整個 CTS 重來

---

<!-- _class: lead -->

# Step 6 — Routing

---

## Step 6：Routing 理論重點

**這階段要做什麼：**
1. 設定 router 模式（走線層上下限、疊代次數、時序驅動/SI 驅動開關）
2. **Global routing**：規劃網路的大致走線路徑與資源分配
3. **Detail routing**：實際產生每條線的幾何繞線
4. 修 DRC 違例（間距、寬度、短路、開路）
5. 修 **antenna 違例**（金屬走線過長累積電荷，需插 diode 或跳層修復）
6. Routing 完成後重新做 timing/DRC 檢查，不合格就繼續疊代

> 詳見 `note/routing.md` 6.1–6.2 節；NanoRoute（Innovus）對應 ICC 的 Zroute

---

## Step 6：示意圖 — Global vs Detail Routing

```
Global Routing（先分格子，決定每條 net 大致要走哪個區域）
┌───┬───┬───┬───┬───┐
│ A │ A╲│   │ B │ B │
├───┼───┼───┼───┼───┤
│ A │   │ A╲│ B │   │
└───┴───┴───┴───┴───┘
        ↓
Detail Routing（在格子裡把每條線精確的金屬線路徑、via 都畫出來）
──┐         ┌────────
  └───┐  ┌──┘
      └──┘
```

- **Global routing**：把晶片切成很多小格子（GRC），估算每條 net 大致要經過哪些格子、會不會塞車（congestion）
- **Detail routing**：在格子裡把每條線實際的金屬層、幾何座標、via 都畫出來，必須符合製程 DRC
- Metal layer 由下往上：M1 留給 std cell 電源軌與短距離訊號，越上層（本例 M5/M6）留給電源網路（呼應 Step 3），中間層（M2–M4）主要跑訊號——DTMF_CHIP 最終總繞線長 320,640 μm 全部落在 M1–M4

---

## Step 6：Routing — `gcd` 範例

```tcl
setNanoRouteMode -quiet -routeWithTimingDriven false
setNanoRouteMode -quiet -routeWithSiDriven false
routeDesign -globalDetail
```
教學範例**關閉**了時序驅動與訊號完整性（SI）驅動 routing 以求簡化、一次執行完 global+detail —— **實務設計通常會打開這兩項**。

---

## Step 6：Routing — `DTMF_CHIP` 實際案例

```tcl
routeDesign -globalDetail                          ;# 01Route：初版繞線
routeDesign -globalDetail -viaOpt -wireOpt         ;# 反覆疊代 via/wire 最佳化
                                                    ;#（routeDesign 系列全流程共呼叫 103 次！）
                                                    ;# 03Byitemopt_routing：逐條 net 修違規
verifyConnectivity ...                             ;#（呼叫 74 次，穿插在每輪繞線之間檢查）
verifyProcessAntenna -report DTMF_CHIP.antenna.rpt -error 1000
                                                    ;#（共呼叫 27 次，違規數逐漸收斂到 0）
```
`routeDesign` 103 次＋`verifyConnectivity` 74 次＋`verifyProcessAntenna` 27 次，三個數字加總說明 Routing 階段的本質就是「繞線 → 檢查 → 再繞線」不斷疊代，而非一次執行到位。

**分支插曲**：11/2 第一次繞完線後手動修 DRC 到 `02Fix_done`，但學生後來回頭多做一輪 CTS 微調（Step 5 的 `04ByitemCCOPT_1103`），**重新繞線**、直接跳過 `02Fix_done`，走向更乾淨的 `03Byitemopt_routing → 04Antenna` —— 代表 `02Fix_done` 是被放棄的舊嘗試，**真實流程經常需要回頭重做前面階段**,不是嚴格單向。

---

## Step 6 補充：Routing 怎麼解決 Antenna 違規（問題根源）

**問題根源**：晶圓廠用電漿蝕刻（plasma etching）把多餘金屬蝕刻掉來刻出線路。蝕刻過程中，一段還沒接到上層金屬／擴散區的長金屬線，會像天線一樣收集帶電離子，在它連接的電晶體閘極（gate）上累積電壓——電壓太高會把閘極氧化層打穿，電晶體永久損壞。

```
蝕刻進行中，金屬只接到低層、還沒接上層 = 危險狀態：

   ══════════════════════  M1（暴露面積大，像一根天線收集電荷）
          │
        [ Gate ]  ← 電荷持續累積，電壓越來越高，可能打穿閘極氧化層
```

- **天線規則（antenna rule）**：晶圓廠規定「與某個 gate 相連的金屬總面積 ÷ 該 gate 面積」不能超過某個比值，超過就觸發 antenna violation
- Innovus 用 `verifyProcessAntenna` 檢查是否違反這個規則，DTMF_CHIP 案例被呼叫了 **27 次**，代表這是「繞線 → 檢查違規 → 再繞線」不斷疊代收斂的過程，不是一次到位

---

## Step 6 補充：Routing 怎麼修 Antenna（跳層／插二極體）

兩種常見修法：

```
方法一：跳層（layer jumping）             方法二：插二極體（diode）
   ═══  M1（先跳到上層，暴露面積變小）        ═══════════════ M1
    │                                          │           │
   ═══  M2                                  [ Gate ]    [ Diode ] → GND
        │                                                  ↑ 導走多餘電荷
      [ Gate ]  ← 累積電荷少很多，安全
```

- **跳層**：讓 gate 連接的線儘早跳到上層金屬，蝕刻低層金屬時暴露面積變小，累積電荷自然變少——這是對既有繞線改動最小、router 最常用的做法
- **插二極體**：在 gate 連接的網路上接一顆反向二極體到 GND，把多餘電荷導走，但需要額外空間放二極體 cell

**DTMF_CHIP 實際怎麼修的**：指令歷史裡**沒有**看到手動插二極體的指令，也沒有特別開類似 `-insert_diodes_during_routing` 的選項——代表是 **Innovus 的 NanoRoute 在 detail routing 時自動用跳層方式修**，搭配 `-routeTopRoutingLayer 4`（訊號最高只走到 M4，呼應 Step 3 補充的金屬層概念）反覆 `routeDesign` 疊代，讓 27 次檢查裡的違規數一路收斂到 0。

---

## Step 6：真實案例的 ECO 修正

postCTS 的 setup／hold 最佳化跑完後，指令歷史顯示學生發現 `SPI_INST` 底下好幾顆暫存器彼此距離太近、hold time 快不夠了，於是**手動下 ECO 指令**（而不是重跑整個 placement/CTS）：

```tcl
ecoChangeCell -inst DTMF_INST/SPI_INST/spi_sr_reg_1 -downsize
ecoChangeCell -inst DTMF_INST/SPI_INST/dout_reg_1  -downsize
... （同樣手法共套用在 16 顆 SPI_INST 暫存器上）
```

- `-downsize`＝換成驅動力較弱、但延遲較大的同功能 cell，藉此**墊高 hold margin**
- 只動這 16 顆 cell，其餘已經繞好線、收斂好的部分完全不受影響——這就是最典型的 **ECO（Engineering Change Order）**：局部手術式修正，不是重跑整個階段

> 對照 Step 8 sign-off 結果：hold WNS 最終只有 -0.000ns、228 條路徑僅 1 條微幅違規——這種「幾乎壓線但過關」的結果，背後往往就有像這樣的手動 ECO 在支撐。

---

## Step 6：Routing 檢查清單

- [ ] Global routing 壅塞是否可接受（溢出 GRC 比例、最大溢出量）
- [ ] Detail routing 完成後 DRC 違規數是否收斂到 0
- [ ] Antenna 違規是否逐輪收斂到 0（`verifyProcessAntenna`）
- [ ] 繞線後 timing 是否還在可接受範圍（若嚴重劣化，代表 placement/CTS 需要回頭調整）
- [ ] 若单一分支修不乾淨 → 考慮回到 CTS 或 placement 重跑，而不是無限做 routing ECO 硬撐（見上頁真實案例）

---

<!-- _class: lead -->

# Step 7 — DFM（Design For Manufacturing）

---

## Step 7：DFM 理論重點與實作對照

**這階段要做什麼：**
- **Filler cell**：填滿標準單元列間的空隙，維持 N/P 阱與電源軌連續
- **Metal fill（dummy metal）**：在各金屬層插入虛擬填充圖形，滿足晶圓廠金屬密度規則，並接到 VSS/VDD 避免浮空金屬

**`gcd` 範例：**
```tcl
addFiller -cell FILL64 FILL32 FILL16 FILL8 FILL4 FILL2 FILL1 -prefix FILLER
addMetalFill -layer { M1 M2 M3 M4 M5 M6 M7 M8 M9 } -nets { VSS VDD }
```

**`DTMF_CHIP` 實際案例（`05MetalFill` checkpoint，收尾階段）：**
```tcl
addFiller -prifix -doDRC     ;# 插 filler 同時做 DRC 檢查
addMetalFill
```

> 詳見 `note/routing.md` 附錄 Step.7；ICC 概念對應 `insert_pad_filler`（`floorplan.md` 3.2.4 節，填的是 pad 間隙而非 cell row 間隙）

---

<!-- _class: lead -->

# Step 8 — Verification

---

## Step 8：Verification 理論重點與實作對照

**這階段要做什麼（三項缺一不可）：**
- **DRC**（`verifyGeometry`）：幾何設計規則檢查（間距、寬度、重疊）
- **LVS 概念**（`verifyConnectivity`）：連接性檢查（開路、短路），對應 IC 業界的 LVS
- **Antenna**（`verifyProcessAntenna`）：天線效應違例檢查

**`gcd` 範例：**
```tcl
verifyGeometry
verifyConnectivity -type all -error 1000 -warning 50
verifyProcessAntenna -reportfile gcd.antenna.rpt -error 1000
```

---

## Step 8：DTMF_CHIP 最終 sign-off 結果

**這裡的「sign-off」是什麼意思？** 前面各階段（Placement 反覆跑 5 輪、CTS 反覆微調）用的都是比較寬鬆、求快的分析設定；`05MetalFill` 這一版是**用最嚴謹的 OCV 分析（見進階補充）重新驗收過**的最終結果——通過 sign-off，才代表這批數字真的可信賴、可以交付。

**`DTMF_CHIP` 最終（`05MetalFill`）sign-off 結果：**

| 檢查項 | 結果 |
|---|---|
| DRC（`routeDesign.DRC.total`） | **0** |
| `verifyConnectivity` | **0 errors** |
| `verifyProcessAntenna` | **0 violations** |
| Setup timing WNS(all) | **0.022 ns**（228 條路徑，0 違規） |
| Hold timing WNS(all) | **-0.000 ns**（228 條路徑中 1 條微幅違規） |
| 總繞線長 | 320,640 μm（M1 23,049／M2 97,520／M3 120,783／M4 79,288） |
| Via 總數 | 48,327（Via12 23,363／Via23 19,244／Via34 5,720） |
| Cell utilization（post-route） | 66.8% |
| Standard cell／Net 數 | ≈5,561–5,571 顆／≈5,914–5,923 條 |

**為什麼要看這些數字：**
- 總繞線長／Via 數：反映 routing 密度，間接對應後段良率與 EM（electromigration）風險
- Cell utilization（post-route）：驗證最終密度是否還落在 Floorplan 當初設定的目標範圍內（本例 Floorplan 設定 ≈70.2%，post-route 略降到 66.8%），形成頭尾呼應
- Standard cell／Net 數：確認 DFM（filler／metal fill）沒有改變邏輯規模，只補了物理填充

> 三項驗證 + timing sign-off **全部通過才算完成**；本案例 hold 幾乎壓線，代表這顆設計如果要再優化，下一步該做 `optDesign -postRoute -hold` 精修那條路徑。完整逐階段指令與數據見 `note/Version4_DTMF_CHIP_Innovus_flow.md`。

---

<!-- _class: lead -->

# Step 9 — Data Export

---

## Step 9：Data Export 理論重點與實作對照

**這階段要做什麼：**
- 設定分析模式（如 `bcwc`：Best-Case Worst-Case，同時看 setup 角與 hold 角）
- 輸出 **SDF**（Standard Delay Format）：routing 後真實延遲，供 post-layout gate-level 模擬反標注
- 輸出 post-APR **網表**、**GDSII** 版圖（送 tapeout）、**LEF abstract**（供上層當巨集用）、Innovus 資料庫存檔

```tcl
setAnalysisMode -analysisType bcwc
write_sdf ... ${TOP_DESIGN}.sdf
saveNetlist ${TOP_DESIGN}_apr.v
streamOut ${TOP_DESIGN}.gds -mapFile ... -mode ALL
write_lef_abstract ${TOP_DESIGN}.lef
saveDesign ${TOP_DESIGN}.enc
```

> **註**：`DTMF_CHIP` 180nm 練習案例的 checkpoint 紀錄**止於 `05MetalFill` 收尾與驗證**，並未包含 Step 9 匯出指令 —— 這是課程練習到 sign-off 為止，尚未做最終 tapeout 匯出，此處以 `gcd` 教學範例呈現完整 Step 9 該做的事。

**銜接下一階段**：`write_sdf` 輸出的 `.sdf` 會被 post-layout testbench 用 `$sdf_annotate` 反標注、跑 gate-level 模擬 —— 詳見 `note/STA.md` 附錄「SDF 反標注與 Post-layout Simulation」。

---

<!-- _class: lead -->

# 動手練習：驗證你真的懂了

---

## 動手練習：如何 source checkpoint 深化理解

每個 `.inn` 存檔都封裝了完整的 lib/lef/mmmc 設定，可直接 `source` 還原到該階段狀態。與其死記指令，不如照下面順序**自己重新做一次**，比對能不能得到跟 checkpoint 一致的結果 —— 做不到的地方，通常就是還沒真正理解該階段「為什麼」要這樣設定：

1. `source Floorplan/01Floorplan_set.inn` — 還原初始 floorplan，練習 `addRing`／`addStripe`／`sroute`
2. `source Placement/01Placement.inn` — 接著自己動手下 `place_opt_design`，比較是否能重現 Step 4 的收斂數據
3. `source CTS/02CTS.inn` — 練習下 `ccopt_design`，比對能否重現 `03CCOPT` 的 timing 數字
4. `source Route/01Route.inn` — 練習 `routeDesign -globalDetail -viaOpt -wireOpt` 疊代收斂，最後跑 `verifyProcessAntenna` 與 `addFiller`／`addMetalFill` 收尾，比對能否重現 `05MetalFill` 的最終 QoR

> 每一步都建議先自己跑一輪、再對照 checkpoint 裡的既有數據，這樣才看得出「哪一輪疊代做了什麼調整」，而不只是複製指令。

---

<!-- _class: lead -->

# 各階段檢查清單總表

---

## 九大步驟一覽（要做什麼＋完成判斷）

| # | 階段 | 要做什麼 | 完成判斷 |
|---|---|---|---|
| 1 | Design Import | 匯入 netlist/SDC/LEF/Lib/MMMC | 無 load error |
| 2 | Floorplan | 定 die/core 尺寸、utilization | 面積、aspect ratio 合理 |
| 3 | Power Planning | Ring/Stripe/Rail、`sroute` | `verifyConnectivity` 0 error |
| 4 | Placement | 擺放＋scan reorder＋pre-CTS opt | Pre-CTS WNS 轉正、DRC 乾淨 |
| 5 | CTS | 建 spec、`ccopt_design`、post-CTS opt | Post-CTS setup/hold 收斂 |
| 6 | Routing | Global+Detail routing、修 DRC/antenna | DRC＝0、antenna＝0 |
| 7 | DFM | Filler、metal fill | 密度規則通過 |
| 8 | Verification | DRC／LVS／Antenna 三項齊全 | 全部 0 違規 |
| 9 | Data Export | 輸出 SDF/網表/GDS/LEF | 交付檔案齊全 |

---

<!-- _class: lead -->

# 進階補充：MMMC／OCV 設定（可跳過）

---

## Step 1 補充：DTMF_CHIP 的真實 MMMC 設定

`gcd` 只用兩個 `.lib` 檔做示範；`reference_design/180um/` 的 checkpoint（`viewDefinition.tcl`）還原出一套完整的 corner-based MMMC：

```tcl
create_library_set -name dtmf_libs_min -timing {pllclk_fast.lib ram_128x16A_fast_syn.lib
    rom_512x16A_fast_syn.lib ram_256x16A_fast_syn.lib fast.lib tpz973gbc-lite_fast.lib} -si {fast.cdb}
create_library_set -name dtmf_libs_max -timing {pllclk_slow.lib ram_128x16A_slow_syn.lib
    ram_256x16A_slow_syn.lib rom_512x16A_slow_syn.lib slow.lib tpz973gwc-lite_slow.lib} -si {slow.cdb}

create_rc_corner -name dtmf_rc_corner -cap_table t018s6mlv.capTbl -qx_tech_file t018s6mm.tch \
    -preRoute_res 1 -postRoute_res 1 -preRoute_cap 1 -postRoute_cap 1 -postRoute_xcap 1

create_delay_corner -name dtmf_corner_min -library_set dtmf_libs_min -rc_corner dtmf_rc_corner
create_delay_corner -name dtmf_corner_max -library_set dtmf_libs_max -rc_corner dtmf_rc_corner

create_constraint_mode -name common -sdc_files {dtmf.sdc}
create_analysis_view -name dtmf_view_setup -constraint_mode common -delay_corner dtmf_corner_max
create_analysis_view -name dtmf_view_hold  -constraint_mode common -delay_corner dtmf_corner_min
set_analysis_view -setup {dtmf_view_setup} -hold {dtmf_view_hold}
```

---

## Step 1 補充：MMMC 結構拆解

2 組 library set（`_min`＝fast 製程角，含 PLL/RAM/ROM/std-cell 全部換成 fast 版；`_max`＝slow 版）交叉 1 個共用 `dtmf_rc_corner`（RC 抽取條件相同，只有 library 角落不同）→ 產生 2 個 delay corner → 搭配同一份 `dtmf.sdc` 的 1 個 constraint mode → 組成 2 個 analysis view：

**setup 用 slow 角（`corner_max`）、hold 用 fast 角（`corner_min`）**，符合 setup 抓最壞情況（慢）、hold 抓最壞情況（快）的物理直覺。

---

## Step 1 補充：MMMC 設定一次建立、全程沿用

比對 `Floorplan/01Floorplan_set` 與最終 `Route/05MetalFill` 兩個 checkpoint 的 `viewDefinition.tcl`：**完全相同**（僅多一行 GUI 用的 `set_interactive_constraint_mode`）。`Version5`（同設計的另一次練習）也是一字不差的同一套 MMMC。

**意義**：MMMC 的 corner／view 設定是跟著**製程與設計**綁定的，在 Design Import 階段建立一次之後，Floorplan → Placement → CTS → Route 全程都重複使用同一套視角，**不會**每個階段重建；不同 checkpoint 之間唯一會變的是「用哪個 view 做檢查」與「有沒有開額外的分析模式」（下一頁）。

---

## Step 1 補充：跑到哪個階段才切換分析模式？

從各階段 `inn.cmd.gz` 指令歷史比對 `setAnalysisMode` 的使用時機：

| 階段 | 指令 | 意義 |
|---|---|---|
| CTS（`03CCOPT` 起） | `setAnalysisMode -checkType setup` / `-checkType hold` | 交替切換 `timeDesign` 要看哪一種違規 |
| **Route（`01Route` 起，CTS/Placement 都沒有）** | `setAnalysisMode -analysisType onChipVariation` | **首次開啟 OCV（On-Chip Variation）derate 分析** |
| Route（`05MetalFill` 收尾） | `set locv_inter_clock_use_worst_derate false` | 微調 LOCV（Location-based OCV）跨時脈 derate 策略 |

**重點**：OCV 分析在 Placement／CTS 階段**沒有**打開，一路到 **Route 才第一次啟用**——實務上前段用單純 corner-based 分析先求快速收斂，等進入 routing、時序數字接近最終、才切換到較嚴格（也較耗時）的 OCV derate 分析做 sign-off 等級的檢查，呼應 Step 8 `05MetalFill` 最終驗證用的就是這套 OCV 設定下的時序結果。

---

<!-- _class: lead -->

# 參考資料

---

## 參考資料

**理論筆記（`note/`）：**
- `floorplan.md` — Floorplan 與 PNS 電源網路理論（含 gcd Step.1–3 對照）
- `placement.md` — Placement 與功耗控制理論（含 gcd Step.4–5 對照）
- `routing.md` — Routing 與 ECO 理論（含 gcd Step.6–9 對照）
- `STA.md` — 靜態時序分析理論（含 SDF 反標注案例）
- `Version4_DTMF_CHIP_Innovus_flow.md` — DTMF_CHIP 完整實作紀錄

**實作專案（`reference_design/`）：**
- `gcd/scripts/gcd_soce.tcl` — 90nm 教學範例，九步驟各跑一次
- `180um/Version4/` — 180nm 完整實作，checkpoint 資料庫（`.inn`/`.inn.dat`），含 MMMC／OCV 設定
- `180um/Version5/` — 同設計的另一次 Floorplan/Placement 練習，MMMC 設定與 Version4 完全一致，可互相驗證

<!-- _class: lead -->

# Thank You
