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

1. 什麼是 APR？九大步驟總覽
2. Step 1–3：Design Import → Floorplan → Power Planning
3. Step 4–6：Placement → CTS → Routing
4. Step 7–9：DFM → Verification／Sign-off → Data Export
5. 總結：檢查清單＋貫穿全流程的疊代收斂循環

---

## 什麼是 APR？

**APR（Automatic Place and Route）**：把合成後的**閘級網表**變成可以送晶圓廠的**版圖（GDSII）**。

```
RTL 設計 → 邏輯合成 → APR（本簡報範圍） → Sign-off → Tapeout
```

**核心矛盾**：面積、時序、可繞線性（routability）、功耗四者互相牽制——整個流程就是反覆疊代收斂這四個指標，不是單向線性流程。

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

## 資料來源

- `gcd`：90nm 教學範例，九步驟各跑一次，求簡化
- `DTMF_CHIP`：180nm 真實課程作業，每個階段都反覆疊代很多輪——本簡報的「真實案例教訓」都來自這裡

> 後面每個 Step 的指令與數字，都是從這兩個實際案例的 checkpoint／指令歷史直接還原，不是憑空舉例。

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

---

## Step 3：Power Planning — 要做什麼

建立電源 ring（環）→ stripe（網格）→ rail（標準單元電源軌），三層電源網路。

```tcl
addRing -nets {VDD VSS} -type core_rings -layer {top M5 bottom M5 left M6 right M6} -width 7
```

---

## Step 3：Power Planning — 要注意什麼

- `verifyConnectivity` **必須 0 error** 才能進入下一階段（浮接電源會讓後面所有時序分析失真）
- 巨集（尤其 RAM／ROM／PLL 這類硬巨集）通常要**額外加一圈 block ring**，不能只靠 core ring
- Stripe 密度要夠——依 IR drop／EM 需求決定數量與寬度

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

## Step 4：Placement — 要注意什麼

- Congestion 熱圖有沒有大面積熱點——壅塞會拖累後面 routing
- Pre-CTS setup WNS 要**轉正**，DRC 要乾淨，才能進 CTS
- 有 scan chain 的設計記得 `reorderScan`，避免掃描鏈繞線暴長

> **真實案例教訓**：DTMF_CHIP 這輪 placement 反覆跑了 **5 次** `place_opt_design` 才收斂——placement 幾乎不會一次到位。

---

## Step 5：CTS — 要做什麼

指定 clock buffer／inverter cell → 建 clock tree spec → 實際蓋出時脈樹、平衡 skew。

```tcl
ccopt_design
```

---

## Step 5：CTS — 要注意什麼

- **Hold time 要等 CTS 做完才能精確算**——CTS 前的 hold 檢查都不準
- 除了 skew，也要看 **DRV**（max transition／max capacitance／max fanout）有沒有超標
- 特定 clock domain 有問題時，可以只針對它做「by-item」局部重跑，不用整個 CTS 重來

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

> **真實案例教訓**：DTMF_CHIP 全流程 `routeDesign` 系列指令共呼叫 **103 次**、antenna 檢查呼叫 **27 次**——這是常態，不是設計出了問題。

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

**要注意**：Sign-off 用的分析設定比平時疊代**更嚴謹**（開 OCV derate、用真實寄生參數），嚴謹的產線流程還會換一套獨立工具（DRC 用 Hercules/Calibre、timing 用 PrimeTime）重新檢查一次，不能只信任 P&R 工具自己的估算。

---

## Step 8：DTMF_CHIP 最終 Sign-off 結果

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

九個步驟裡每一次「反覆疊代」——Placement 5 輪、Routing 103 次呼叫、CTS by-item 微調——本質上都是同一個循環：

**跑分析 → 找問題 → 局部修正 → 再驗證**，不斷重複直到全部違規清零。

> 真實 APR 流程不是九步驟走一次就結束，而是「做完 → 檢查 → 不合格就疊代重做」的持續循環。

---

<!-- _class: lead -->

# Thank You

Q&A
