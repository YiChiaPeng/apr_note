# Routing 筆記

> 資料來源：《數位積體電路後端設計》（田曉華主編，武漢理工大學出版社，2019）第 6 章「Routing」，頁 170–196，基於 Synopsys IC Compiler（積體電路編譯器，簡稱 ICC）。

> 本筆記下列專有名詞直接使用英文，不再翻譯成中文：**floorplan**、**script**、**place／placement**、**routing**。本章主題本身就是 routing，因此原書「布線」一詞全數譯為 **routing**。

---

## 縮寫對照表（全稱一覽）

| 縮寫 | 英文全稱 | 中文意義 |
|---|---|---|
| ICC | IC Compiler | Synopsys 的積體電路後端 placement／routing 工具 |
| DRC | Design Rule Check | 設計規則檢查 |
| LVS | Layout Versus Schematic | 版圖與電路圖比對驗證 |
| GR | Global Routing | 全域 routing（routing 三步驟之一） |
| TA | Track Assignment | Routing 軌道分配（routing 三步驟之一） |
| RVI | Redundant Via Insertion | 冗餘過孔插入 |
| CTS | Clock Tree Synthesis | 時脈樹合成 |
| CTO | Clock Tree Optimization | 時脈樹最佳化（routing 後對時脈偏移的修正） |
| NDR | Non-Default Routing（rule） | 非預設 routing 規則（如加寬、加間距） |
| SI | Signal Integrity | 訊號完整性 |
| ECO | Engineering Change Order | 工程修改（原指「工程修改單」，現泛指設計後期的局部修改） |
| STA | Static Timing Analysis | 靜態時序分析 |
| RTL | Register Transfer Level | 暫存器傳輸層級 |
| EM（本章） | Electromagnetic（field） | 電磁（場）——注意本章「EM 場」指電磁場，與別章「電遷移 Electromigration」的 EM 縮寫**意義不同** |
| DFM | Design For Manufacturing | 面向製造的設計（晶片收尾階段） |
| tapeout | （流片） | 芯片設計定案、交付晶圓廠加工的動作 |
| MOS | Metal-Oxide-Semiconductor | 金氧半導體（電晶體結構） |
| poly | Polysilicon | 多晶矽（閘極材料） |
| SiO₂ | Silicon Dioxide | 二氧化矽（閘極氧化層絕緣材料） |
| Gcell | Global routing cell | 全域 routing 單元格（與其他章節的 GRC 概念相同） |
| SBox | Search Box | 詳細 routing 處理用的小矩形區域 |
| tf | Technology File | 技術檔案（定義過孔、layer 等製程規則） |

---

## 章節地圖

| 小節 | 主題 |
|---|---|
| 6.1 | Routing 原理及流程 |
| 6.2 | ICC Routing 技術 |
| 6.3 | ECO 工程修改 |
| 6.4 | 串擾問題分析與解決 |
| 6.5 | 小結 |

隨著積體電路製程更新，90nm 節點及更先進節點的互連線延遲對 IC 晶片影響變得越來越大，routing 的重要性日益增加。Routing 階段將邏輯合成的邏輯連線替換為真實的物理電路連接，提取物理參數用於接近真實電路的功能與效能評價。Routing 及最佳化作為後端設計的重要環節，綜合評估各項設計邏輯約束和物理約束（DRC）條件，同步最佳化設計的速度、電路面積、功耗等指標。Routing 相對於其他設計步驟，占用更多計算資源、消耗設計周期較多時間，當設計收斂（達到指標要求）後，電路進入設計收尾階段，經過後續少量工序，晶片的設計結果將交付晶圓廠流片（tapeout）加工。

---

## 6.1 Routing 原理及流程

### 6.1.1 Routing 的目標

Routing 的目標是透過多層金屬導線和過孔實現元件接腳之間的電氣連接。Routing 要滿足 DRC 約束和電氣參數要求，包括時序的建立時間和保持時間、時脈參數、最大負載電容 `max cap`、最大訊號電平轉換時延 `max transition` 等。

Routing 階段除了實現所有訊號連通，還需考慮的因素包括：
- 根據設計規則控制 routing 的寬度和間距
- 根據線寬和阻抗選擇合適的過孔類型
- 根據 routing 方向控制過孔矩陣的旋轉或偏移
- 透過擴大線間距或增加屏蔽線減小串擾，提高訊號品質
- 透過屏蔽線和訊號交織的方式保護匯流排訊號
- 透過換層 routing 減小總走線占用的電路面積
- 對差分線 routing 進行特殊處理，平衡 routing 長度和電容

### 6.1.2 Routing 基本流程

Routing 前設計需要滿足的條件包括：
1. 電源網路的 routing 在設計規劃階段已經完成
2. 時脈樹的合成及最佳化完成
3. 壅塞估計的結果可以接受
4. 時序分析估計的時序裕度（slack）可以接受，裕度為 0ns 左右或更好結果
5. 估計的最大電容 `max cap` 和電平轉換時間 `max tran` 沒有違例

**Routing 流程圖（圖 6-1）**：

(a) 總體 routing 階段劃分：
```
Placement 與 CTS → 時脈訊號 routing → 訊號 routing → Routing 最佳化 → 晶片收尾（DFM）
```

(b) 具體 routing 步驟：
```
全域 routing GR → Routing 軌道分配 TA → 詳細 routing → 搜索問題並修復
```

Routing 階段的任務大致分為三部分：先完成高優先級的時脈訊號 routing，再進行訊號 routing，routing 完成後進行後續的 routing 效能最佳化。Routing 結束標誌流片前進行晶片收尾工作。需要注意，後端設計除了 routing 及最佳化階段的 routing 步驟外，在 placement 階段也可以開啟全域 routing 用於精確計算 routing 參數，評估 placement 方案代價（cost）。

時脈或普通訊號都要經過 3 個基本步驟：
1. 全域 routing（global routing, GR）
2. Routing 軌道分配（track assignment, TA）
3. 詳細 routing（routing 的最後階段，包括時序與 DRC 違例的解決）

3 個步驟的實現都是透過調用 routing 主命令 `route_opt` 完成。`route_opt` 執行之後，routing 完全連通到單元接腳。在 routing 過程中同時最佳化時序、面積、功耗等效能指標，且針對插入緩衝器 buffer 檢查 DRC 設計規則，同時檢查版圖的物理 DRC 規則。在 routing 完成全域 routing、routing 軌道分配、詳細 routing 三步操作後，如果還存在 DRC 違例，應在 routing 最後階段搜索並修復剩餘 DRC 問題。

### 6.1.3 Routing 技術演算法分類

按照整體功能類型和演算法特點對 router 進行分類（圖 6-2）：

```
Router
├─ 全域 Routing
│    ├─ 圖搜索
│    ├─ Steiner 斯坦納樹
│    └─ 迭代的
├─ 詳細 Routing
│    ├─ 受限的
│    │    ├─ River 河流
│    │    ├─ Switchbox 交換盒
│    │    └─ Channel 通道
│    └─ 通用目的的
│         ├─ Maze 迷宮
│         ├─ 線探頭 Line probe
│         └─ 線展開 expansion
│              └─ （層次化的／貪婪的／左邊沿）
└─ 特殊 Routing
     ├─ 電源與地網路
     └─ 時脈
```

Routing 技術的整體劃分與 routing 階段的劃分高度關聯。特殊的訊號 routing 需要先處理，包括在 floorplan 階段已完成的對電源和地訊號的電源網路合成 PNS 和在時脈樹合成 CTS 階段開始 routing 設計的時脈網路 routing。普通邏輯訊號的 routing 分為全域的和詳細的 router，其中全域的 router 與 routing 三步驟中的全域 routing GR 對應，而後兩個步驟（TA 和詳細 routing）的技術可以歸為詳細 router。

全域 router 實現訊號 routing 的初步劃分。可以結合前面章節內容的 Gcell 全域 routing 單元的配置理解 router 的訊號路徑的初步配置，實現的技術包括圖搜索、Steiner 斯坦納樹和疊代最佳化方式。全域 routing 的演算法涉及圖論等數學問題，例如斯坦納樹研究的是如何找到連通多點的最短通路，也就是具有多扇出網路的最短電路 routing 問題。

詳細 routing 技術包括條件受限的 routing，如河流、交換盒、通道 routing，以及通用目的的 routing，如迷宮、探頭、線展開形式。這裡列舉的大部分技術術語涉及底層的 EDA router 核心技術實現，而讀者在本章學習 ICC 的 routing 相關技術應用過程中，僅需了解命令相關的技術選項。

### 6.1.4 基於網格的 Routing 系統

**網格（grid）routing 系統的基本特徵（圖 6-3）**：示例顯示最下兩層金屬 M1 和 M2 的 routing 效果。圖中縱橫交錯走向的虛線被稱為 routing 軌道，而軌道的交匯點被稱為格點（grid point）。Routing 按預設規則需要以 routing 軌道為中心走線。兩條相鄰的 routing 軌道的間距稱為 routing 軌道間距 Pitch。該數值是在技術檔案的 DRC 約束中定義。由於定義了 routing 軌道和軌道間距，積體電路在每一層金屬的 routing 變得規則可控。這種基於網格的 routing 系統可以滿足積體電路製程對 routing 的基本要求，且便於 EDA 工具實現 routing 演算法。每層金屬都定義了各自的網格和預設的 routing 方向（例如 M1 為水平方向，M2 為垂直方向）。

**（1）基本單元（unit tile）**

基本單元是標準單元庫的一種特殊單元，包含了對每一層金屬預設 routing 方向和 routing 軌道的定義資訊。作為標準單元庫的一個單元，基本單元定義的資訊還包括：
1. 標準單元最小寬度和高度尺寸
2. 金屬層的預設方向 routing 的間距 pitch
3. 標準單元供電的 VDD 和 VSS 電源軌道（rail）位置（由標準單元高度決定）
4. 基本單元高度是 M1 金屬層線間距的倍數（M1 pitch 為水平 routing 軌道間距）
5. 基本單元寬度是 M2 金屬層線間距的倍數（M2 pitch 為垂直 routing 軌道間距）

**（2）Routing 軌道間距與單元尺寸關係**

結合標準單元物理庫的定義，可以發現 M1 和 M2 routing 軌道間距、基本單元的寬高、標準單元的寬高滿足整數倍關係。因此 routing 軌道間距確定了基本單元和標準單元的尺寸。Routing 軌道間距由半導體加工工藝的特徵尺寸決定。以 routing 軌道間距的整數倍定義基本單元和標準單元尺寸，便於晶圓加工（曝光機按照軌道間距移位，可以與標準單元精確對齊），也有利於 EDA 工具的 routing 路徑計算（routing 路徑計算複雜度降低）。

---

## 6.2 ICC Routing 技術

### 6.2.1 ICC Routing 基本步驟

**1. 全域 Routing（GR）**

全域 routing 的知識要點：
1. **全域 routing（GR）功能定義**：完成對訊號線走線層以及 Gcell（全域 routing 單元）的計算分配。該步驟的主要工作是完成對 routing 路徑的大致計算，選擇合適的 routing 通道。
2. **全域 routing 的輸入和輸出定義**：輸入條件包括 routing 完成的巨集與單元和 routing 通道的容量（routing 通道數）；routing 輸出結果為每條走線的拓撲（幾何路徑）。
3. **全域 routing 階段的設計目標**：包括最小化 routing 線長和 routing 面積、平衡設計的壅塞，時序驅動和雜訊驅動 routing，滿足匯流排 routing 緊湊排列。
4. **全域 router 框架類型**：主要包括斯坦納 Steiner 樹 routing、基於通道的 routing、迷宮 routing。
5. **全域 routing 的計算複雜度**：全域 routing 將設計區域劃分為較粗粒度的 Gcell，並限定每一個 Gcell 四邊邊界的最大 routing 軌道數。Routing 路徑規劃等價於尋找連通的 Gcell 單元序列。以 Gcell 為單元的計算複雜度相對於規劃具體 routing 路徑顯著降低。
6. **全域 routing 兩種實現方式**：
   - a. 順序實現：對執行的路徑計算的順序敏感，順序的排列通常決定於問題關鍵性和走線端點數量
   - b. 平行實現：計算量大，且計算難度大；常使用階層化的設計方式

由於每一個 Gcell 包含 routing 軌道總數和全域 routing 在 Gcell 中已分配 routing 資源，全域 routing 完成時，設計的壅塞熱圖已經建立。在本書的 floorplan、placement 相關章節的壅塞分析採用了全域 routing 結合 GRC（Gcell）的快速分析方式評估 floorplan 和 placement 的壅塞風險。

全域 routing 為訊號分配 Gcells 走線過程中自動計算 Gcell 壅塞程度，並採取繞線（detour）避免經過壅塞的 Gcell。如需要繞線，router 將盡可能減小繞線長度。Router 也會避免與電源網路包括電源環、條帶、軌道的走線衝突，並避開禁止 routing 線區。Router 通常根據路徑的時序分析決定是否繞線。

**全域 routing 以 Gcells 為單位 routing 示例（圖 6-4）**：路徑規劃只配置到 Gcell，而不指定 routing 軌道。圖中的虛擬路徑顯示的是 router 採用虛線 routing 規劃路徑。虛擬路線是一種相對於全域 routing 更早的技術。由於該技術不分析路徑的壅塞程度，虛擬路線 routing 可能會進入壅塞區域。

**全域 routing 階段鋪設的導線和前期 routing 完成的電源網路對比（圖 6-5）**：電源網路在 floorplan 階段已經鋪設了真實導線以及與電源網路連通的過孔，如圖中從電源條 strap 到電源軌道的過孔。而全域 routing 只指定虛擬導線，金屬導線並沒有真正布設。全域 routing 忽略後續 routing 細節，可以迅速評估 routing 壅塞，最佳化整體結果。

**2. Routing 軌道分配（TA）**

Routing 軌道分配（TA）對每一條訊號線指定 routing 軌道，從全域 routing 路徑上每一個 Gcell 的未使用 routing 軌道中指定一條，鋪設真正的金屬導線。TA 盡量採用 routing 層優先 routing 方向，並且倾向於採用直的長導線，減小過孔數量。在 TA 階段，router 不檢查 DRC 錯誤，可能造成 routing 間距或凹槽（notch）的違例。不同金屬層 routing 軌道最小間距的差異可透過各層導線（顏色區分）最小間距看出。

**3. 詳細 Routing**

詳細 routing 確保訊號線上所有接腳通過 routing 連通，並在設計區域內每一個訊號完成 routing 的前提下盡量減小 routing 面積。詳細 routing 可以採用多種演算法實現，較常用的 routing 方式包括迷宮 routing、線探索 routing 等。

**（1）迷宮 Routing**

迷宮 routing 的目標是計算一個平面圖上的起點到終點的最短路徑。以走迷宮的方式，尋找 A、B 兩點之間最小的連通路徑。按照 routing 的水平方向和垂直方向 routing 軌道間距將圖劃分為網格，並計算從起點網格到圖中各個網格點的路徑距離（即找到連通起點網格與目標網格的網格序列）。黑色格子為禁止 routing 區域，不參與路徑長度計算。A 到 B 的最短路徑長度為 14。由於多網格點同步計算，當路徑到達終點 B 後，其他與起點 A 路徑距離相同的網格同步確定，圖中其他還未計算的網格點不再需要計算路徑長度。迷宮 routing 的優勢是總能得出最短路徑，但缺點是由於有大量網格點需要順序計算，計算時延較大，而且需要占用較高的儲存資源。

> **思考題**：請根據上述迷宮 routing 演算法的介紹，舉例分析各點到起點 A 的路徑距離，並簡要設計迷宮 routing 的實現步驟。（提示：觀察圖中數字，聯想水面的波紋擴散現象）

**（2）線探索 Routing**

除了迷宮 routing 外，另一種速度更快的 routing 演算法被稱為線探索 routing（line-probe routing）。這種方式不需要使用離散的走線資源，計算量和資源占用小。路徑的起點和終點分別按合適的出線方向連接源探頭和目標探頭，而在 routing 平面的可用通道中規劃若干水平和垂直走向的逃脱線（escape line），透過逃脱線之間以及逃脱線與探頭之間的交匯點建立 routing 路徑。

**（3）多端點訊號線連接**

需要連接多端點的多扇出網路 routing 相對於上述兩種點對點 routing 演算法更複雜。常用的 routing 過程是：① 一次連一個端點；② 在樹的生長過程中將整個已連接的子樹作為源或目標繼續與其他端點連接；③ 採用斷開/重連（rip-up/reroute）的方式改進走線品質。

多端點連線總長度取決於連接方向的約束和對單個接腳的連線數量的約束。採用較多的多端點 routing 方式（圖 6-9）包括：routing 路徑較短的斯坦納樹、有 routing 主幹的斯坦納樹、串聯各端點的鏈條形式、最小展開範圍（span）樹、端點之間全連接的完整圖。各種連接方式名稱括號內的數值對應多點連接 routing 總長度，可看出斯坦納樹的 routing 長度相對較短：
- 斯坦納樹（14）
- 有主幹的斯坦納樹（15）
- 鏈條（17）
- 最小範圍樹（16）
- 完整圖（42）

**4. Routing 後的 DRC 修復**

在 routing 階段查看 DRC 錯誤的介面可顯示錯誤在版圖中的符號標記，錯誤瀏覽窗口可以根據錯誤的類型瀏覽各類錯誤，放大圖顯示了版圖的線間距的 DRC 錯誤標記。透過圖形化顯示定位，設計者更容易定位問題和修復問題。

**常見的 DRC 錯誤（圖 6-11）**：
- Routing 形成的凹槽（notch）間距過小，應調整過孔連接的走線走向，或者刪除不需要的過孔
- 對於同層間距小於最小間距限制的過孔，可以透過錯位方式調整過孔位置
- Routing 之間間距過小或過大則可以調整 routing 間距

檢查 DRC 違例錯誤可以採用以下的指令：
- Router 引擎的 `verify_route`
- 調用 Hercules 校驗工具 DRC 引擎的 `verify_drc`

修復 DRC 違例主要透過打開增量最佳化開關的 routing 最佳化指令 `route_opt -incremental` 實現，增量最佳化在修復 DRC 違例的同時最佳化時序。

除了上述列舉的 routing DRC 規則外，對於 90nm 及以下節點的設計，routing 的 DRC 規則還包括：線端（line end）、邊沿長度（edge length）、最小面積（min area）、過孔相關、非預設方向走線、限制 routing 規則等。

**SBox（Search Box）**：詳細 routing 階段 router 不是像 TA 階段針對區域內全部訊號處理 routing，而是在設定的小矩形區域（SBox）內局部 routing。SBox 的邊長通常為全域 routing 單元 Gcell 的整數倍，例如寬度為 5 個 Gcell 寬度，而高度為 7 個 Gcell 高度。Router 按照光柵掃描的方式逐個處理 SBox 內 routing。

SBox 邊界處更容易出現 DRC 違例。詳細 routing 階段清除 DRC 問題，也是先將設計平面劃分為固定大小的 SBox 矩形單元，再針對每一個 SBox 進行處理。ICC 工具採取**多重循環**，逐漸增大 SBox，修復 routing 後的 DRC 問題（圖 6-12：SBox 從小到大，避免了遺漏 SBox 邊界的 DRC 問題）。

小尺寸 SBox 對 routing 的改變較小，因而對時序的改變較小。隨著 SBox 增大，routing 區域增大，有更多資源可以供 DRC 修復，但是電路的調整增大會造成電路時序的更大變化。SBox 循環修復 DRC 過程通常控制在 50 次以內。如果循環修復次數達到上限仍沒能修復問題，需要再檢查設計的壅塞狀態，並分析 floorplan 是否合理。

Routing 階段的設計規則只是完整 DRC 規則的子集，且 ICC 在設計階段採用的是簡化的 FRAM 視圖，而不是詳細的電晶體級的 CEL 視圖。因此，routing 階段的 DRC 檢查依然有可能遺漏問題。即使 routing 階段修復了 DRC，簽核階段也需要單獨調用 DRC 檢查工具（如 Hercules）再次檢查，以確保設計沒有 DRC 問題。

**複習題（教材原文，未附解答，供自我檢測）**

1. 全域 routing 如何處理壅塞區域？
2. 指定 routing 的金屬層是在 routing 軌道分配（TA）階段完成的嗎？
3. ICC 是否會將一條金屬導線按照非預設方向放置？
4. 詳細 routing 是否會將不同金屬層的 DRC 違例問題合併？
5. ICC 在什麼階段使用 SBox？SBox 尺寸一直保持不變嗎？
6. ICC 在 routing 階段發現和解決所有 DRC 問題後，是否還有必要在簽核（sign-off）階段使用 Hercules 或其他 DRC 工具再次執行 DRC 修復？

### 6.2.2 ICC Routing 前設定與狀態查看

**1. Routing 前的設計狀態檢查**

Routing 前要完成的工作包括：placement 擺放和時脈樹合成 CTS 已經完成，電源網路完成 routing，壅塞估計程度可以接受，估計的時序結果合理（0ns 左右的裕度），以及最大電容和訊號轉換時延沒有 DRC 違例。

以下指令將檢查 routing 前設計約束狀態。根據虛擬 routing 或者無須保存的全域 routing 結果估計時序、壅塞、cap/transition 等指標：
```tcl
report_constraints -all
```

在 placement 和時脈樹合成完成後，也可以使用 `check_zrt_routability` 指令檢查設計的可 routing 通性，並輸出違例的問題列表。如下示例步驟使用 `check_physical_design` 指令檢查設計在 routing 前最佳化階段（`pre_route_opt`）的設計要求，並報告輸出違例資訊。`all_ideal_nets` 指令和 `all_high_fanout` 指令檢查設計中是否存在理想網路（如理想時脈訊號）和高扇出網路 HFN（如系統復位訊號）。在 placement 階段和時脈樹合成階段，這些訊號應該已合理添加緩衝樹，控制扇出，滿足設計約束和 DRC 要求。如果檢查發現存在理想網路和高扇出網路，則需要在網路中添加緩衝樹。高扇出網路的預設閾值是 1000，如果需要檢查扇出超過 500 的網路，則需要設定門限 `-threshold` 數值選項。此外，routing 前也可以報告金屬層預設 routing 方向，便於後續根據 routing 情況考慮 routing 方向局部調整。
```tcl
check_physical_design -stage pre_route_opt
all_ideal_nets
all_high_fanout -nets -threshold 501    # 預設閾值是 1000
report_preferred_routing_direction
```

**2. 多核心處理器多執行緒計算設定**

由於 routing 計算量相對其他設計階段更高，且可以並行計算實現，將計算任務分解到處理器的多核心實現，可以提高計算效率。ICC 的 Zroute router 是多執行緒 router。多執行緒方式與多台伺服器運行的分布式 router 不同，是將計算任務分配到同一處理器的多核心上運行。最新版本 ICC 一個 license 授權可支援 8 核心多執行緒運行，而早期版本一個 license 只支援 4 執行緒。設定多執行緒運行指令格式如下：
```tcl
set_host_options -max_cores 8
```

Routing 階段預設採用 Elmore 時延計算模型。將 routing 時延模型設定為 Arnoldi 模型，可以基於提取的 RC 參數計算更精確走線時延，提高 routing 效能。設定 Arnoldi 時延模型指令如下：
```tcl
set_delay_calculation -arnoldi
```

**3. Zroute Router 設定**

Zroute router 設定主要包括：通用設定、全域 routing 設定、routing 軌道分配設定、詳細 routing 設定。對應的指令是 `set_route_zrt_common_options`、`set_route_zrt_global_options`、`set_route_zrt_track_options`、`set_route_zrt_detail_options`。

**（1）設定通用 Routing 選項**

通用 routing 選項對全域 routing、routing 軌道分配以及詳細 routing 同時有效。通用 routing 選項包括 routing 運行設定、routing 的邊界和避止區設定、走線層控制、過孔選項、routing 規則等。通用選項可以設定在詳細 routing 後添加過孔的控制力度，在 routing 階段同步完成過孔添加：
```tcl
set_route_zrt_common_options -default true
set_route_zrt_common_options \
  -post_detail_route_redundant_via_insertion medium
```

**（2）設定全域 Routing 選項**

全域 routing 選項包括：routing 程序運行控制、巨集、時脈網路 routing 等。其中 routing 運行控制的選項包括時序驅動以及時序驅動的力度、串擾驅動、僅用於輸出壅塞熱圖（用於分析 routing 潛在的壅塞問題）、基於金屬層的壅塞熱圖及力度等。
```tcl
set_route_zrt_global_options -default true
```

**（3）設定 Routing 軌道分配 TA 選項**

Routing 軌道分配的設定選項相對簡單，包括時序驅動或者串擾驅動。
```tcl
set_route_zrt_track_options -default true
```

**（4）設定詳細 Routing 選項**

詳細 routing 控制選項主要控制天線效應（違例）修復、接腳和端口的連接、線寬和間距以及 routing 程序的運行控制。在運行控制中可以選擇時序驅動、最佳化現場和過孔的力度、生成偏離 routing 網格的 routing 軌道用於接腳連接等控制選項和設定。如果使用冗餘過孔插入，推薦使用走線和過孔最佳化選項。
```tcl
set_route_zrt_detail_options -default true
set_route_zrt_detail_options \
  -optimize_wire_via_effort effort_level medium
```

**4. Zroute Router 設定查看**

查看上述四類 router 的設定並輸出報告指令格式為：
```tcl
report_route_zrt_*_options
```
其中 `*` 代表 common、global、track、detail 之一，用以查看對應設定指令的選項。如果要得到設定的具體一項選項設定，可以採用如下指令獲取 routing 設定選項：
```tcl
get_route_zrt_*_options -name option_name
```
指令透過 `-name` 選項查看具體選項設定。

### 6.2.3 ICC Routing 控制流程

**1. 時脈樹 Routing**

ICC routing 流程在 routing 前完成 router 設定後，按照訊號的類別先後分別完成時脈訊號 routing 和其他訊號 routing，並進行 routing 後最佳化。時脈訊號相對於其他訊號，對時序電路具有更重要的意義，因此優先 routing。在 Zroute router 設定介面的訊號線組 routing（net group routing）設定介面勾選 `All clock nets`（所有時脈訊號 routing），並勾選 `Reuse existing global route` 重用現有的全域 routing 結果，利用前期時脈樹階段設計結果進行時脈樹訊號 routing。時脈樹 routing 的指令如下：
```tcl
route_zrt_group -all_clock_nets -reuse_existing_global_route true
```

**2. 訊號線 Routing 及最佳化**

訊號線 routing 及最佳化基本目標是：完成 routing，設計滿足時序、串擾、routing DRC 約束要求。在時脈訊號 routing 完成後，Zroute router 需要採用主指令 `route_opt` 完成初始 routing 和 routing 後最佳化工作。Routing 核心指令的 `route_opt` 格式如下，指令的多個選項支援訊號線的全域 routing、routing 軌道分配、詳細 routing、問題搜索與修復、結合 ECO routing 的邏輯與 placement 最佳化、routing 步驟控制。

以下列出的 `route_opt` 指令選項可以單個使用或多個聯合使用。Routing 時可以控制執行單個步驟或所有步驟按順序完成：
```tcl
route_opt
  -effort low | medium | high    # 控制 routing 和最佳化力度：低、中、高
  -stage global | track | detail # Routing 階段控制：GR、TA、詳細
  -power                         # Routing 最佳化功耗
  -xtalk_reduction                # 抑制串擾
  -initial_route_only             # 只進行初始 routing
  -skip_initial_route             # 跳過初始 routing，執行後續步驟
  -incremental                    # 增量最佳化
  -area_recovery                  # 電路面積恢復，減小電路面積佔用
  -num_cpus N                     # 配置 routing 並行使用的 CPU 數量，用於
                                   # 多 CPU 分布式計算，而 zroute 採用的是多核心 CPU，實現多執行緒計算
```

Routing 及 routing 後的最佳化過程主要包括：
1. **Routing 最佳化**：方法包括減少線長、過孔數量以及不必要的繞線（jog，同一金屬層較短距離改變 routing 方向）。對於不完全成熟的製程，減少過孔，增加直線 routing 可以提高設計良率，也可以改善路徑的時序。Routing 最佳化主要用於詳細 routing 後或者在 routing 修復階段的時序違例或用於不成熟的製程。
2. **Routing 後最佳化**：用於處理 routing 後的時序違例或者最大電容、電平轉換時間違例問題。Routing 後最佳化主要包括調整單元尺寸、修復保持時間以及基於拓撲結構最佳化。
3. **Routing 後時序最佳化**：用於解決 routing 前後時序結果差異。最佳化採用真實的時脈樹傳播延遲，並採用真實走線延遲計算時序。採用標準 Vt 單元修復剩餘的時序問題。
4. **Routing 後時脈樹最佳化（CTO）**：Routing 及最佳化過程有可能使之前最佳化的時脈偏移發生改變。因此 routing 後如果時脈偏移結果變差，可以進行 CTO 以及 ECO routing 改善時脈訊號。

在訊號線 routing 最佳化過程第一次調用 `route_opt` 時，最好採用 `-initial_route_only` 選項：
```tcl
route_opt -initial_route_only
```
該選項允許分析時序的建立和保持時間、DRC、時脈偏移等，根據結果確定後續步驟選項。這一步操作涉及全域 routing、TA routing 軌道分配以及詳細 routing，所有訊號線完全連接，但時序、最大的 cap/transition、DRC 有可能違例，需要後續最佳化解決。

初始 routing 除了連續完成三個階段 routing 外，也可以結合 `-stage` 選項控制 routing 完成的具體階段，例如使用 `-stage global` 指定 router 只完成全域 routing。

初始 routing 後進行 routing 最佳化指令，採取 `-effort` 設定最佳化力度等級為中等，採用 `-power` 最佳化功耗，指令需要設定跳過初始 routing：
```tcl
route_opt -skip_initial_route -effort medium -power
```

Routing 後最佳化邏輯 DRC 問題：可以採用 `set_app_var` 設定變數 `routeopt_drc_over_timing` 值為真，提升 DRC 相對於時序 timing 的最佳化級。採用的 DRC 修復指令使用增量最佳化，力度為高，且設定只修復 DRC 違例，指令如下：
```tcl
route_opt -effort high -incremental -only_design_rule
```

`route_opt` 指令的其他的最佳化選項還包括：只調整大小的 `-size_only`、只修復保持時間的 `-only_hold_time`、修復線寬的 `-wire_size`。

Routing 後在詳細 routing 階段添加冗餘過孔後，router 會輸出統計資料，包括最佳化的過孔轉化率以及三步 routing 的資料。輸出資料包括每一層過孔的轉化率，如雙過孔以及條狀（bar）過孔。冗餘過孔轉化效率一般晶圓廠會有明確要求，例如 85%。如下指令輸出 routing 資料報告，包括雙過孔（冗餘過孔）轉換率、全域 routing、routing 軌道分配、詳細 routing 的資料：
```tcl
report_design -physical
```

Routing 後的物理設計規則違例檢查可以採用 router 自帶的 DRC 工具，採用的指令為 `verify_zrt_route`。該指令主要檢查物理 DRC、開路、短路、天線效應等。

可以使用 `verify_lvs` 分析開路問題，指令如下：
```tcl
verify_lvs -ignore_short -ignore_min_area
```

修復 DRC 問題可以採用 `route_zrt_detail -inc` 進行增量最佳化，修復 DRC 違例。

**表 6-1　Routing 基本流程**

| Routing 指令 | 指令說明 |
|---|---|
| `source antenna_rules.tcl` | 導入天線效應規則，用於修復天線違例 |
| `set_route_zrt_common_options`／`set_route_zrt_global_options`／`set_route_zrt_track_options`／`set_route_zrt_detail_options`／`...` | Router 的通用設定和三個 routing 階段的設定控制 |
| `route_zrt_group -all_clock_nets` | 以 routing 組形式對時脈訊號 routing |
| `route_opt -initial_route_only` | 在初始 routing 階段，完成 routing，但不做任何最佳化 |
| `route_opt -skip_initial_route` | 初始 routing 後的 routing 最佳化階段 |
| `route_opt -incremental` | Routing 的增量最佳化，解決違例問題 |

最後兩個 routing 步驟 `-skip_initial_route` 與 `-incremental` 的相同點是都是在初始 routing 階段完成後進行，但區別在於前者是在初始 routing 後進行完整（全面）的 routing 最佳化，而後者只針對違例問題如時序違例、DRC 違例進行增量最佳化。

**3. 改變預設 Routing 方向**

Router 通常按照優先（preferred）routing 方向設定進行 routing，相鄰兩層的優先 routing 方向一般是垂直的。但在某些情況下，控制 router 利用非預設走線方向走線，可以顯著減小 routing 長度，改善設計時序。

**修改預設 routing 方向前後的跨巨集 routing 對比（圖 6-14）**：在巨集上方的大部分區域設置了從 M1 到 M6 的禁止 routing 區，而巨集中部有一個左右貫穿的狹窄通道允許在 M5 和 M6 層 routing。巨集兩側的兩個門的連通捷徑是在 M5 層從狹窄通道 routing，但預設設定 M5 的優先 routing 方向是垂直方向。Router 按照預設規則只能繞過巨集 routing。如果修改通道區域 M5 的優先 routing 方向，則邏輯門之間從通道的 M5 層 routing，減小 routing 長度。

以上的 routing 方向改變操作可以在圖形化介面中設置：在 `Create Object Tool` 建立物件工具設定中選擇 routing 指導 Route Guide，並設定物件名為 `route_guide0`，再勾選 `Switch Preferred direction` 更改預設 routing 方向，並在 Coordinates 設定框內輸入更改 routing 方向的矩形區域兩點座標值。新版本的 ICC 的介面設定直接選擇 `Create Route Guide` 建立 routing 指導，並支援在版圖中直接框選需要更改方向的區域。與介面操作對應的指令設定如下：
```tcl
create_route_guide -name route_guide0 \
  -coordinate {{270 340} {491 485}} \
  -switch_preferred_direction -no_snap
```

Routing 前設定禁止 routing 區，在指定層的設定區域內禁止放置金屬連線或過孔。例如上例中巨集 placement 區域設定的兩種禁止 routing 區。指定禁止 routing 層的名稱不能直接使用技術檔案的層名，而是採用專用於禁止 routing 的層名。

設定禁止 routing 區域有兩種方式，可以採用選項 `-bbox` 或者 `-boundry` 分別用於設定矩形區和直角多邊形區域。以下示例在設定的矩形區內禁止了金屬層 metal1 和過孔層 via1 的 routing。選項 `-layers` 設定層名稱列表中的 `metal1Blockage`、`via1Blockage` 是專用於禁止 routing 的層名：
```tcl
create_route_blockage -bbox {30 100 120 340} \
  -layers {metal1Blockage via1Blockage}
```

**4. Routing 冗餘過孔添加（RVI）**

冗餘過孔添加是指在 routing 過程中採用多過孔連接代替單過孔連接。冗餘過孔的主要作用是提高 routing 跨層連接的可靠性，提升晶片製造的良率。冗餘過孔添加方式可以採用基於軟規則（soft rule）的冗餘過孔同步插入（詳細 routing 階段）或 routing 後的冗餘過孔插入。請注意：在傳統的設計流程中，冗餘過孔的添加需要等到 routing 完成後執行，並在添加後檢查和修復 DRC 問題。新的 EDA 工具支援在初始詳細 routing 階段開始添加冗餘過孔，以減少 routing DRC 問題。

**冗餘過孔添加對電路特性的改變**在於增加過孔數量，提升電流導通截面積，降低過孔阻抗，從而減小 routing 的時延，但對於保持時間（hold time）約束造成負面影響。

**（1）ICC 的冗餘過孔檢查與設定**

Router（zrouter）routing 前從技術檔案（tf）讀入預設的過孔定義，並生成預設的過孔映射表。設計者需要 (a) 檢查預設過孔映射表的設定；(b) 如果需要，重新定義冗餘過孔規則，包括定義每層金屬的添加的冗餘過孔在 x 和 y 方向的過孔數量倍數。

查看冗餘過孔狀態採用 `insert_zrt_redundant_vias` 指令，如下所示：
```tcl
insert_zrt_redundant_vias -list_only    # 查看冗餘過孔設定，使用 -list_only 選項
define_zrt_redundant_vias \             # 設定新的冗餘過孔規則
  -from_via {VIA23 VIA34} -to_via {VIA23 VIA34} \
  -to_via_x_size {1 2} -to_via_y_size {2 1}    # x y 方向過孔冗餘倍數
```
查看設定採用 `-list_only` 選項，而重新定義冗餘過孔設定採用 `define_zrt_redundant_vias`，添加冗餘過孔需要指定添加過孔的起始和結束（`-from_via` 和 `-to_via` 選項）。下例中添加過孔的 `-to_via` 選項的過孔列表要配置 x 和 y 方向的過孔尺寸（size，即數量）列表對應。指令中連通 2 層和 3 層的過孔 VIA23 的過孔數量設定為 x 方向 1、y 方向 2，即垂直方向添加雙過孔；而 VIA34 的過孔配置為水平方向添加冗餘過孔，也是雙過孔。

**（2）執行冗餘過孔插入（RVI）操作**

冗餘過孔插入（RVI）操作包括檢查物理設計規則，並在所有的詳細 routing 步驟中執行過孔的添加操作，包括詳細 routing 階段的 routing 過孔最佳化力度設定、routing 選項的詳細 routing 後冗餘過孔插入力度選項、定義冗餘過孔設定規則。執行 routing 最佳化指令 `route_opt` 將按照配置自動執行 RVI 操作，包括了初始詳細 routing 步驟與後續 routing 最佳化步驟。Routing 階段 RVI 相關指令腳本示例如下：
```tcl
set_route_zrt_common_options \
  -post_detail_route_redundant_via_insertion medium
set_route_zrt_detail_route_options \
  -optimize_wire_via_effort_level medium
define_zrt_redundant_vias ...   # 定義冗餘過孔形式
route_opt -initial_route_only   # 執行 RVI 操作
...
route_opt -skip_initial_route   # 執行 RVI 操作
```

### 6.2.4 Routing 階段天線違例修復

**1. 天線效應和天線違例定義**

在積體電路製造過程中，MOS 電晶體的閘極（gate）可能承受不斷增加的電壓。在積體電路製造工藝中的蝕刻階段，帶電離子在強電磁（EM，Electromagnetic）場的作用下被激發並蝕刻保護層覆蓋範圍外的金屬層。在振盪的帶電離子作用下，蝕刻過程中金屬層與所連接的電晶體閘極（poly）積累了大量與蝕刻離子極性相反的電荷。累積的電荷在閘極形成持續上升的電壓。與電晶體的閘極連接的跨層金屬俯視圖中可見，閘極電壓受到所連接金屬導體的積累電荷的影響，隨著金屬走線面積加大而增大。當與閘極連接的金屬導線的總面積超過一定數值，增大的閘極電壓有可能造成電晶體閘極 SiO₂ 絕緣層被擊穿，電晶體永久損壞。這種積體電路製造過程中對電晶體閘極的破壞效應被稱為天線效應（antenna effect）。天線效應也被稱為離子蝕刻的積累電荷造成的閘極氧化層損壞。

晶圓廠為了避免天線效應而制定天線規則（antenna rule），定義了與閘極連接的金屬連線最大總面積。如果與閘極相連的 routing 面積超過該設定閾值，則觸發設計的天線違例（antenna violation）。天線違例如不解決，將對 MOS 積體電路可靠性造成影響。

**2. 天線違例修復原理**

Routing 階段的天線違例修復主要透過切換 routing 層（layer jumping）實現。調整原設計中與閘極連接的低層金屬連線的總面積，確保在頂層金屬通連前，連接總面積不超過天線規則限制，則可避免閘極氧化層被擊穿的風險。

**切換 routing 層修復天線違例的兩種方式（圖 6-18）**：
- **方式 1**：閘極首先透過過孔跨層連接到頂層（連接到頂層 M2），再 routing 連接驅動訊號。這種方式保證在加工頂層金屬前，與閘極連接的低層金屬面積最小，積累的電荷最少，因此風險較低。
- **方式 2**：保持了與原設計大部分相同的 routing，只是在閘極連接處將較短的一段 routing 切換到頂層 M2，該方式同樣減少了閘極在金屬蝕刻階段的電荷積累，而相對於方式 1，該方式對 routing 的修改較小，有利於 routing 後階段的違例修復。

**3. 天線違例的 ICC 同步修復**

ICC 支援在詳細 routing 階段同步修復天線違例。同步修復的好處是在 routing 階段解決天線違例，避免了在晶片收尾（chip finishing）階段額外的處理步驟。修復的方法主要是透過上述切換 routing 層等技術。首先加載晶圓廠提供的天線 DRC 規則，設定 zroute routing 選項，在詳細 routing 過程設定指令 `set_zrt_detail_route_options` 中打開天線修復選項 `-antenna true`。指令腳本如下：
```tcl
source antenna_rules.tcl
set_zrt_detail_route_options -antenna true
```
指令 `set_zrt_detail_route_options` 在 ICC 2010.3 版本後增加了一個新的指令選項開關 `-hop_layers_to_fix_antenna`，支援對切換 routing 層方式的單獨控制，預設值為 true，表示 zroute router 的預設天線違例修復工具是以上介紹的切換 routing 層方式。除切換 routing 層控制，指令支援另一個 routing 階段的天線違例修復選項，採用插入二極體修復方式，即 `-insert_diodes_during_routing`。該選項如果設定為 true，則 routing 階段即可在與閘極連接的金屬導線和地之間插入反向二極體，避免閘極電壓過高。

---

## 6.3 ECO 工程修改

**1. ECO 技術背景**

ECO（engineering change order）工程修改是一個歷史較久遠的術語。在早期的電路設計流程中，如果需要對電路已經定義的或規定的部分進行修改，需要先填寫「工程修改單」，簽字審批後才能修改。現在大部分公司不再使用工程修改單，而 ECO 現在主要是指在設計的後期或者接近結束階段進行的設計修改。

在晶片流片生產前或生產剛完成電晶體加工時，如果設計團隊發現晶片設計邏輯錯誤，將採取 ECO 工程修改技術，在盡可能保留後端 placement／routing 結果的前提下進行彌補性晶片修復。ECO 的主要意義在於保留前期設計成果，犧牲一部分產品效能，快捷修復錯誤，可以為設計項目節省時間開銷、人力成本以及晶片流片成本。

通常，前端設計流程首先確定大體架構和功能指標，然後前端工程師完成代碼並進行驗證仿真，在幾乎不能再發現新錯誤後進入 RTL freeze（代碼固化）階段，不能再修改 RTL 代碼。固化的 RTL 代碼將會進行邏輯合成和靜態時序分析 STA，再交付後端進行 floorplan、PNS、placement、CTS、routing 等步驟。後端流程相對於前端極其耗時，且通常操作不可逆，就是說同樣流程再做一遍，結果不能保證收斂。因此如果在後端流程完成或接近完成時，前端發現新的 RTL 代碼錯誤，重新合成和進行後端流程將耗費大量人力和時間。

ECO 的原理可以簡化理解為直接修改網表。修改網表過程跳過前端 RTL 代碼合成和後端從 floorplan 開始的多個步驟，利用現有結果，可降低各種成本。

**2. ECO 技術分類**

根據 ECO 工程修改時是否已經啟動流片的光罩加工流程，可以將 ECO 分為光罩前（pre mask）ECO 和光罩後（post mask）ECO。分類是根據晶片的電晶體底層的晶體管製造用光罩（mask）是否已經加工來區分。

**光罩前 ECO 與光罩後 ECO 流程對比（圖 6-19）**：兩種 ECO 技術的差異，主要差別在於單元 placement 是否可以改變。如果 placement 可以改變，則添加新功能單元實現修改後的 ECO 電路邏輯，在光罩加工前完成設計修改；如果 placement 不能改變，只能利用前期預留的空閒單元實現邏輯功能更改，則是在光罩加工並完成電晶體製造後進行。

```
ECO 修改後網表
   ↓
Placement 是否固定？
  ├─ Yes（光罩後 ECO）：不能移動、添加單元，使用空閒單元實現 ECO
  └─ No（光罩前 ECO）：ECO placement 工具計算並放置新添加單元，不需要空閒單元實現 ECO
   ↓
繼續完成 ECO Routing
```

**3. 光罩前 ECO**

光罩前 ECO 也被稱為矽片凍結前（non-freeze silicon）ECO 或流片前 ECO。因為流片生產之前依然允許在設計中添加電晶體、邏輯單元，調整 placement、routing。光罩前 ECO 流程主要包括三步：對設計進行 ECO 更新、ECO placement、ECO routing。

**（1）對設計進行 ECO 更新**

在當前已經完成 placement／routing 的設計上根據 ECO 修改後的網表進行更新，主要採用指令 `update_mw_design_eco`，對現有設計庫導入更新後的 eco 網表實現，或者採用 `read_mw_eco_list` 指令導入 eco 更改檔案（change file）實現。示例腳本如下：

**舊版本指令**：
```tcl
update_mw_design_eco -library orca_lib.mw \
  -change_verilog orca_eco.v -top_module ROUTED
# 或
read_mw_eco_list -library orca_lib.mw \
  -change_file orca_eco.changefile ROUTED
```

**新版本指令**：
```tcl
eco_netlist -by_verilog_file golden.v -write_changes filename.tcl
```
新版本指令採用 `eco_netlist` 實現網表檔案的導入和設計更改的 tcl 腳本導入。以上腳本的第二種方式導入設計更改檔案，資料量相對於完整的網表檔案更小，導入資料量小，更改設計流程被簡化。

**ECO 更改檔案語法（括號中為更新檔案的字段說明，符號「+」表示添加資訊，符號「-」表示刪除或斷開）**：

| 字段 | 說明 |
|---|---|
| `Date and Time (+D)` | 時間日期 |
| `Tool (+T)` | 工具 |
| `Author (+A)` | 作者 |
| `Comments (+C)` | 注釋行 |
| `Net Creation (+HN)` | 建立的網線 |
| `Net Deletion (-HN)` | 刪除的網線 |
| `Port Net Connection (+HC)` | 端口連接的網路 |
| `Port Net Disconnection (-HC)` | 端口斷開的網路 |
| `Instance Creation (+HI)` | 建立的新單元（空閒單元） |
| `Instance Deletion (-HI)` | 刪除的單元（問題單元） |

> **思考題**：請根據以上語法逐行解釋以下 ECO 更改檔案，並回答電路邏輯進行了哪些修改。
> ```
> +D Tue Jan 08 14:00:00 2013
> +T IC Compiler
> +C Invert DATA[3]
> +A ecoengineer
> +HN d3_spare
> -HC net_DATA[3] DATA_iopad_3/I
> +HI d3_spare_inv invbd7
> +HC net_DATA[3] d3_spare_inv/I
> +HC d3_spare d3_spare_inv/ZN
> +HC d3_spare DATA_iopad_3/I
> ```

除了透過 ECO 更改檔案導入 ECO 設計外，也可以透過 tcl 指令實現相同操作，例如建立新單元可以採用 `create_cell` 指令實現，單元的接腳連線可以透過 `connect_net` 指令實現，操作步驟的 tcl 指令可以保存到 tcl 檔案中，以便重複調用。

**（2）ECO Placement**

ECO placement 是基於更新的 eco 邏輯設計，對當前設計進行單元合規放置，即按照標準單元 placement 約束放置。指令如下：
```tcl
legalize_placement -eco -inc
```

**（3）ECO Routing**

ECO routing 是在盡可能保留原有 routing 結果前提下，對調整後的單元進行 routing 修改。調用指令如下。如果不使用 zroute router，則 routing 指令為 `route_eco`：
```tcl
route_zrt_eco
```

**4. 光罩後 ECO**

光罩後 ECO 也被稱作矽片凍結後（freeze silicon）ECO 或流片後 ECO，晶圓上的電晶體通過光罩完成加工，所有單元的 placement 位置不再改變。設計中如果預留了空閒邏輯單元（spare cells），則電路功能的修改可以透過更改上層金屬連線，連接空閒單元代替錯誤邏輯電路實現。金屬連線更改透過調整金屬層的 routing 實現。

**空閒單元**是在 placement 階段分散放置在功能邏輯單元 placement 空隙的簡單邏輯功能門單元，如與非門 NAND、異或門 NOR 等。相同功能的空閒單元在設計的晶片上有多處位置可以選擇，而光罩後 ECO 對挑選空閒單元的位置選擇是盡可能靠近需要替代的錯誤邏輯單元，以減小金屬連線修改難度。功能錯誤邏輯單元的信號連線則被斷開，變成實際上空閒單元。空閒單元替換錯誤邏輯單元示意圖：右下角替換錯誤邏輯單元，被替換單元的 3 個連接訊號需要與空閒單元 routing 連接。

**（1）光罩後 ECO 的空閒單元 Placement**

由於光罩後 ECO 需要使用空閒邏輯單元，而這些單元需要在 placement 或更早設計添加，添加流程：在 placement 階段，ICC 首先識別網表中是否已經配置了空閒單元。

如果有空閒單元，首先需要將空閒單元設定為空閒單元屬性，並在指定的 placement 區域採用 `spread_spare_cells` 指令分散擺放空閒單元，再用 placement 指令 `legalize_placement -eco -incr` 進行 ECO 模式增量最佳化，完成 placement。

如果當前設計中沒有空閒單元，則需要採用 `insert_spare_cells` 指令並以指定的單元名稱、單元數量以及命名規則，向設計中添加空閒單元。

添加的空閒單元列表是或非門和與非門，這兩個單元作為一組，placement 時將會被緊湊地靠近放置。空閒單元完成 placement 後需要設定 `dont_touch` 屬性和 `soft fixed` 屬性。`dont_touch` 屬性可以防止空閒單元因為沒有訊號輸出而被 ICC 刪除。`soft fixed` 屬性可以防止 placement 階段的增量初步（incremental coarse）placement 對空閒單元位置的較大移動，但允許 CTS 和 routing 階段最佳化工作對單元位置的微調。`-tie` 選項是為了讓右下角的示意圖高亮顯示空閒單元的 placement 效果，呈現規則的矩陣形式排列。Placement 完成後，按照正常流程進行 CTS 和 routing 設計。

**布局階段設置並放置空閒單元（圖 6-21）**：
```
原始網表
  → Placement
      → 網表中包含空閒單元嗎？
          ├─ Yes → 選擇一個區域內的空閒單元用於 place placement：
          │         set_attr [get_cells *spare*] is_spare_cell true
          │         spread_spare_cells \
          │           -bbox {{10 10}{80 50}} [get_cells *spare*]
          │         legalize_placement -eco -incr
          │
          └─ No → 添加空閒單元的種類和個數，緊湊 placement：
                   insert_spare_cells \
                     -lib_cell {NOR2 NAND2} -num_instances 20 \
                     -cell_name SPARE_PREFIX_NAME \
                     -tie -hier_cell ALU
  → 不能移動空閒單元，設定屬性 soft fixed：
      set_dont_touch [all_spare_cells] true
      set_attribute [all_spare_cells] is_soft_fixed true
  → CTS and Route（再執行 CTS 和 routing）
```

**（2）光罩後 ECO 的 Placement／Routing 流程**

光罩後 ECO 在 placement 前已經完成上述的空閒單元放置，而如何選擇合適的空閒單元並與當前電路進行修復性連接，需要按照以下 ECO placement／routing 流程進行（圖 6-22）：

```
從當前錯誤設計中導出網表單元（配合 Placement／Routing 網表）
  → 設定光罩後 ECO 模式：set_freeze_silicon_eco
  → 設計進行 ECO 更新：update_mw_design_eco / read_mw_eco_list（或新指令 eco_netlist）
      （修改 ECO 網表並導入 ECO 網表或更改文件）
  → ECO 階段 placement：place_freeze_silicon
  → ECO routing：route_zrt_eco
```

步驟說明：
1. 從錯誤設計中導出網表單元，並對網表進行 ECO 修改
2. 對設計設定為光罩後 ECO 模式，採用 `set_freeze_silicon_eco` 指令
3. 將修改後的 ECO 網表或者 ECO 更改檔案導入設計，採用 `update_mw_design_eco` 或者 `read_mw_eco_list` 指令，設計進行 ECO 更新；也可使用新版本指令 `eco_netlist` 導入 ECO 網表或設計的更改檔案
4. 基於更新的 ECO 設計進行增量化 placement，採用 `place_freeze_silicon` 指令
5. 進行 ECO routing，採用 `route_zrt_eco` 指令

`route_zrt_eco` 指令是 ECO 階段的 routing 工具指令。指令支援指定 routing 網路、使用懸空的連線以及設定 routing 計算循環次數等操作。基本的 routing 指令支援使用懸空的導線，使用全域 router 連接斷開的導線，分配大致的 routing 路徑，再為全域 routing 的走線分配 routing 軌道，使用詳細 router 修復 DRC 錯誤。可以在網表變化後或者手動調整後使用 ECO routing。

---

## 6.4 串擾問題分析與解決

### 6.4.1 串擾成因及現狀

Routing 階段的串擾（crosstalk）相關問題包括串擾的原因分析、物理設計的串擾避免措施、串擾修復以及串擾和壅塞等的關聯。串擾可以定義為訊號在一條電路傳輸時對另一條電路產生不利影響，可以將串擾描述為訊號線之間的能量耦合。串擾一般由耦合電容或耦合電感造成，容性耦合引發耦合電流干擾，而感性耦合引發耦合電壓干擾。

**表 6-2　訊號線的感性耦合與容性耦合對比**

| 項目 | 感性耦合 | 容性耦合 |
|---|---|---|
| 示意 | 入侵訊號 ─(Lm)─ 受害訊號 | 入侵訊號 ─(Cm)─ 受害訊號 |
| 公式 | V_noise,Lm = Lm · dI_driver/dt | I_noise,Cm = Cm · dV_driver/dt |
| 結果 | 耦合雜訊電壓訊號 | 耦合雜訊電流訊號 |

兩條平行 routing 的訊號線 net1 和 net2 透過耦合電容 Cc 形成的串擾影響：net1 的上升跳變訊號作為入侵訊號對 net2 造成影響。對靜止的 net2 訊號，net1 的跳變造成 net2 毛刺形式的靜態雜訊，可能造成邏輯電平的錯誤判斷；而 net2 的跳變訊號，則受到串擾影響形成跳變邊沿的擾動，改變了 net2 電平變化的時延（理想訊號變化與擾動後訊號對比），可能造成訊號的提前變化或推遲變化，從而影響訊號的時延和電路最高時脈頻率。

**減小串擾的 routing 技術包括**：
(a) 對於同層金屬的訊號耦合，要減小相鄰平行走線訊號的重疊長度
(b) 對於相鄰層金屬之間的訊號耦合，也要減小相鄰層訊號的平行重疊長度
(c) 增加地線作為屏蔽訊號線，減小訊號串擾，且屏蔽地線也可用於電源輸送
(d) 盡可能增加線間距，或調整訊號線拐角方向，或者透過跨層增大線間距（圖 6-24：調整 routing 拐角方向或透過跨層 routing 增加線間距）

集成電路隨著工藝更新，電晶體的特徵尺寸和金屬連線線間距不斷減小，單元密度增加，串擾雜訊在 90nm/65nm 以下節點的影響增大。串擾如上所述，增加訊號的延遲（delta delay, delay transition）以及訊號毛刺，影響電路效能。180nm 工藝可以不考慮串擾，130nm 工藝考慮可靠性，可以選做串擾流程，而對於 90nm 及以下的設計，串擾問題在簽核（sign off）階段必須修復。

**後端設計的串擾預防措施主要包括**：
1. Floorplan 階段增大出線多的模組與其他模組間距，避免狹長通道區域的過多 routing，控制單元 placement 的密度，減小壅塞
2. Placement 階段設定較小的訊號跳變時延（max transition），提高訊號的驅動能力和抗干擾能力
3. 時脈或者高翻轉率訊號要增大與其他訊號間距，增加屏蔽線，減小干擾
4. 數模混合設計的數位部分和類比部分訊號需要隔離，避免干擾，並可在數類之間放置去耦電容，過濾毛刺訊號
5. Routing 前設定時序和串擾驅動，並在 routing 後進行雜訊去除修復

### 6.4.2 Synopsys 的串擾控制機制

**1. 串擾控制設計**

ICC 在 placement、CTS、routing 階段的串擾防止功能透過下表的 tcl 指令模板實現，而 routing 後的串擾修復指令也在下表中列出。

**表 6-3　ICC 串擾控制指令**

| 設計階段 | 串擾控制指令 | 指令說明 |
|---|---|---|
| Placement | `set_max_transition`／`set_congestion_options ...`／`area_recovery_critical_range`／`power_recovery_critical_range` | 設定最大電平轉換時延約束；設定壅塞選項；設定面積恢復和功耗恢復的關鍵範圍值 |
| CTS 時脈樹合成 | `define_routing_rule -spacings {...} ...`／`set_clock_tree_options [-clock CLK] -routing_rule my_route_rule` | 採用前一章 CTS 介紹的 NDR 技術定義非預設 routing 規則，並應用 NDR 規則 |
| Routing 前設定 | `set_si_options -delta_delay true -route_xtalk_prevention true -static_noise true` | 布線設定訊號完整性 SI 選項設定，包括設定 routing 的預防串擾 `-route_xtalk_prevention`，設定 routing 訊號串擾的增量時延 `-delta_delay` 選項（XDD）和預防靜態雜訊的 `-static_noise` 選項 |
| Routing 後最佳化 | `route_opt -xtalk_reduction [-incremental]` | Routing 最佳化或增量最佳化階段開啟串擾減小最佳化 |

**（1）Placement 階段**

在 placement 階段除了採用 `max transition` 設定提高訊號驅動能力外，還可以設定壅塞選項避開過密的單元 placement。Placement 階段在非關鍵路徑減小電路面積和功耗的最佳化是透過放寬路徑時延實現。面積恢復和功耗恢復的關鍵範圍（critical range）通常推薦設定為時脈週期的 15%，作為最佳化過程的路徑裕量保護範圍，禁止保護範圍內對面積與功耗最佳化，避免過度最佳化增加路徑時延。

在 placement 階段開啟全域 routing，可以透過全域 routing 後的串擾雜訊熱圖（noise map）發現串擾問題。壅塞、耦合電容、雜訊的熱圖對比：三者有一定的關聯性，但耦合電容比壅塞現象更普遍，兩者分佈不完全匹配。耦合電容與雜訊熱圖差異較大，因為雜訊還受電氣特性影響。

**（2）CTS 階段**

CTS 階段設定並應用時脈等高翻轉率的訊號的 NDR 走線規則，減小訊號干擾。NDR 增加 routing 線寬，可以降低 routing 阻抗，最佳化 routing 時延；而增加線間距可減小耦合效應，減小串擾。

**（3）Routing 前階段**

在 routing 前階段單獨設定 SI 訊號完整性選項，可增強 routing 對串擾的一致控制，並減小串擾對路徑時序的影響。此外還需要設定 routing 各個階段的時序驅動選項，以及 GR、TA 階段的串擾驅動選項。Router 通常會分析串擾較大的訊號線，並採取加大干擾訊號之間距離、更換走線層等方式 routing。

**（4）Routing 後階段**

串擾修復包括基於單元和基於 routing 的最佳化，包括最佳化門的 placement、門的邏輯及 routing 調整，通常策略是增加線間距和提高受害訊號的驅動能力。少量不能自動修復的問題可以採取對受害訊號增加驅動單元，移動驅動單元位置，拉開單元和 routing 間距，以及換層 routing 的方式手動修復。修復的結果是 routing 後的串擾減小且時序結果得到改善。

**完整的布局 routing 階段串擾抑制腳本範例**：
```tcl
place_opt
set_clock_tree_options -max_transition 0.2 -max_fanout 32
set_clock_tree_references -reference MYMCLK_BUFFER_LIST
define_routing_rule MYRULE -spacings {M2 0.28 M3 0.28 ...}
set_clock_tree_options -routing_rule MYRULE
clock_opt
set_si_options -route_xtalk_prevention true \
  -route_xtalk_prevention_threshold 0.35   # 門限單位為 V，預設值 0.45V
  -delta_delay true \
  -static_noise true \
  -static_noise_threshold_above_low 0.3    # 數值為比值，即 30%，預設值 0.35
  -static_noise_threshold_below_high 0.3
route_opt -xtalk_reduction
  # -optimize_wire_via -wire_size   可選的 routing 最佳化選項
report_noise
report_timing -crosstalk_delta
```

> **思考題**：請結合本節內容完整描述上述腳本的串擾預防設定和串擾修復操作。

**2. Synopsys 後端設計平台的串擾控制流程**

在 Synopsys 後端的 Galaxy 設計平台提供較完整的串擾控制機制。串擾控制工具主要包括後端設計階段的 ICC 和簽核階段的 Star-RCXT 與 PrimeTime-SI。

ICC 在前期的 floorplan 階段和 placement 階段透過調整巨集 placement、階段性檢查壅塞和晶片使用率等來最小化壅塞風險，而後期 routing 壅塞是造成訊號串擾的重要因素。

**Galaxy 設計平台的串擾控制（圖 6-26）**：

```
串擾預防（IC Compiler）：
  ├─ 查看巨集 placement、壅塞、晶片使用率的潛在風險
  ├─ 最小化壅塞，設定 max_trans 約束
  ├─ 設定時脈訊號的 NDR 規則
  └─ 在 GR/TA 全域 routing／軌道分配兩個階段設定防止串擾選項

串擾修復：
  ├─ Routing 後串擾最佳化（IC Compiler）
  └─ 簽核階段的修復流程（Star-RCXT、PrimeTime-SI）
```

設計規則的最大電平轉換時延 `max_trans` 約束為較小數值（較嚴格的約束），有助於減小訊號串擾機率。在時脈樹合成階段，NDR routing 規則設定有利於減小時脈訊號與相鄰訊號的串擾。在 routing 階段的前兩個步驟全域 routing GR 和 routing 軌道分配 TA，設定防止串擾選項可以直接影響 router 的串擾分析和 routing 決策。

以上的 ICC 串擾控制方法都是針對串擾預防，而 routing 後針對嚴重串擾問題的修復則需要採用 routing 後串擾最佳化工具實現。ICC 支持最後手動添加單元或者更改 routing，以處理無法自動修復的問題。

簽核階段根據精確提取的 RC 參數和支援 SI 的 PrimeTime 工具確認串擾修復結果，保證抑制串擾後時序結果滿足簽核要求。

---

## 6.5 小結

經過本章學習，讀者能夠掌握以下後端設計 routing 階段的知識和技能：
- 了解 routing 的原理、關鍵技術和基本流程
- 掌握 ICC routing 的基本操作步驟
- 掌握 ICC routing 前的設定與狀態檢查
- 掌握 ICC routing 階段控制流程
- 了解天線效應原理，並掌握 routing 階段天線違例修復方法
- 了解 ECO 工程修改的原理、技術分類：
  - 光罩前 ECO 的 placement／routing 流程
  - 光罩後 ECO 的 placement／routing 流程
- 了解串擾問題的成因及處理現狀
- 掌握 ICC 從 placement、時脈樹合成到 routing 的串擾控制流程

---

## 附：全章 Tcl 指令速查表

| 分類 | 主要指令 |
|---|---|
| Routing 前設定與狀態檢查 | `report_constraints -all`、`check_zrt_routability`、`check_physical_design -stage pre_route_opt`、`all_ideal_nets`、`all_high_fanout`、`report_preferred_routing_direction` |
| 多執行緒／時延模型 | `set_host_options -max_cores`、`set_delay_calculation -arnoldi` |
| Zroute Router 設定 | `set_route_zrt_common_options`、`set_route_zrt_global_options`、`set_route_zrt_track_options`、`set_route_zrt_detail_options`、`report_route_zrt_*_options`、`get_route_zrt_*_options -name` |
| 時脈 Routing | `route_zrt_group -all_clock_nets -reuse_existing_global_route true` |
| Routing 核心指令 | `route_opt`（`-effort`／`-stage`／`-power`／`-xtalk_reduction`／`-initial_route_only`／`-skip_initial_route`／`-incremental`／`-area_recovery`／`-num_cpus`／`-only_design_rule`／`-size_only`／`-only_hold_time`／`-wire_size`） |
| 改變預設 Routing 方向／禁止區 | `create_route_guide -switch_preferred_direction`、`create_route_blockage -bbox -layers` |
| 冗餘過孔（RVI） | `insert_zrt_redundant_vias -list_only`、`define_zrt_redundant_vias` |
| Routing 後檢查與修復 | `verify_zrt_route`、`verify_lvs`、`route_zrt_detail -inc`、`report_design -physical` |
| 天線違例修復 | `source antenna_rules.tcl`、`set_zrt_detail_route_options -antenna true`、`-hop_layers_to_fix_antenna`、`-insert_diodes_during_routing` |
| ECO（光罩前） | `update_mw_design_eco`、`read_mw_eco_list`、`eco_netlist`、`legalize_placement -eco -inc`、`route_zrt_eco`／`route_eco` |
| ECO（光罩後） | `set_freeze_silicon_eco`、`spread_spare_cells`、`insert_spare_cells`、`set_dont_touch [all_spare_cells]`、`set_attribute [all_spare_cells] is_soft_fixed true`、`place_freeze_silicon`、`route_zrt_eco` |
| 串擾控制 | `set_max_transition`、`set_si_options`（`-delta_delay`／`-route_xtalk_prevention`／`-static_noise`）、`route_opt -xtalk_reduction`、`report_noise`、`report_timing -crosstalk_delta` |

---

## 附：實務案例對照（Cadence Innovus，gcd 設計，Step.6～9）

> 出處：`~/Downloads/2025_Fall_Training_Package/3.Post_layout_Simulation/APR/scripts/gcd_soce.tcl`（承接 `floorplan.md`、`placement.md` 附錄的 Step.1～5）

**Step.6　Routing**
```tcl
setNanoRouteMode -quiet -drouteStartIteration default
setNanoRouteMode -quiet -routeTopRoutingLayer default
setNanoRouteMode -quiet -routeBottomRoutingLayer default
setNanoRouteMode -quiet -drouteEndIteration default
setNanoRouteMode -quiet -routeWithTimingDriven false
setNanoRouteMode -quiet -routeWithSiDriven false
routeDesign -globalDetail
```
- **NanoRoute** 是 Innovus 內建的 router 引擎，對應 ICC 的 **Zroute**（見 **6.2.2 節**）。
- 前 4 行 `setNanoRouteMode`：詳細 routing 起始/結束疊代次數、可用走線層上下限，全部維持預設值（`default`）。
- `-routeWithTimingDriven false`：**關閉**時序驅動 routing；`-routeWithSiDriven false`：**關閉**訊號完整性（串擾）驅動 routing——這是教學範例為求簡化而關閉的最佳化選項，對應 **6.2.2 節** 的 routing 選項設定（`set_route_zrt_global_options` 等）與 **6.4.2 節** 的串擾防止選項（`set_si_options`）。實務設計通常會打開這兩項。
- `routeDesign -globalDetail`：一次執行全域 routing＋詳細 routing 兩階段，對應本章 `route_opt`（不含 `-initial_route_only`／`-skip_initial_route` 的細分步驟，**6.2.1／6.2.3 節**）。

**Step.7　Design For Manufacturing（DFM）**
```tcl
addFiller -cell FILL64 FILL32 FILL16 FILL8 FILL4 FILL2 FILL1 -prefix FILLER
addMetalFill -layer { M1 M2 M3 M4 M5 M6 M7 M8 M9 } -nets { VSS VDD }
```
> 教材第 7 章「芯片收尾階段 DFM 設計」尚未整理成筆記，這裡先記錄對應動作。

- `addFiller -cell FILL64 ... FILL1 -prefix FILLER`：在標準單元列的空隙插入填充單元，依寬度由大到小（FILL64→FILL1）優先填滿，維持 N/P 阱與電源軌道連續性——與 `floorplan.md` **3.2.4 節** 的 `insert_pad_filler` 概念相同，差異是這裡填的是核心區內標準單元列的空隙，而非管腳之間的間隙。
- `addMetalFill -layer {M1...M9} -nets {VSS VDD}`：在各金屬層插入虛擬金屬填充圖形（dummy metal fill），滿足晶圓廠對金屬密度（metal density）的製造要求，並將填充金屬接到 VSS/VDD 避免產生浮空金屬。

**Step.8　Verification**
```tcl
verifyGeometry
verifyConnectivity -type all -error 1000 -warning 50
verifyProcessAntenna -reportfile ${TOP_DESIGN}.antenna.rpt -error 1000
```
- `verifyGeometry`：實體設計規則檢查（DRC），檢查間距、寬度、重疊等幾何違例；對應本章 `verify_zrt_route` 或 Hercules DRC 工具（**6.2.1 節**）。
- `verifyConnectivity -type all -error 1000 -warning 50`：連接性檢查（LVS 概念：開路、短路等），最多回報 1000 個錯誤、50 個警告；對應本章的 `verify_lvs`。
- `verifyProcessAntenna -reportfile gcd.antenna.rpt -error 1000`：天線效應違例檢查，報告輸出到 `gcd.antenna.rpt`；對應 **6.2.4 節「Routing 階段天線違例修復」** 的天線規則檢查。

**Step.9　Data Exports**
```tcl
setAnalysisMode -analysisType bcwc
write_sdf -max_view func_mode_max -typ_view func_mode_max -min_view func_mode_min \
  -remashold -splitrecrem -recompute_delay_calc ${TOP_DESIGN}.sdf
saveNetlist ${TOP_DESIGN}_apr.v
streamOut ${TOP_DESIGN}.gds -mapFile .../streamOut.map \
  -libName DesignLib -structureName ${TOP_DESIGN} -units 2000 -mode ALL
write_lef_abstract ${TOP_DESIGN}.lef
saveDesign ${TOP_DESIGN}.enc
```
- `setAnalysisMode -analysisType bcwc`：設定時序分析模式為 **bcwc**（Best-Case Worst-Case，同時考慮最快與最慢角 corner），對應 `STA.md` 提到的 MCMM 多角分析概念。
- `write_sdf ... ${TOP_DESIGN}.sdf`：輸出 **SDF（Standard Delay Format）** 檔案，把 routing 後真實提取的延遲數值寫出，供後續 gate-level **post-layout simulation** 用 `$sdf_annotate` 反標注——這正是本次訓練包 `post_sim/testbench.v` 裡 `$sdf_annotate("../APR/run/gcd.sdf", u1)` 讀取的檔案，串接了「後端設計」與「post-layout 模擬」兩階段。詳見下方「與 `STA.md` 的關聯」。
- `saveNetlist ${TOP_DESIGN}_apr.v`：輸出 placement/routing 後的最終閘級網表——即 `post_sim/sim.sh` 裡拿去跑 VCS 模擬的 `gcd_apr.v`。
- `streamOut ${TOP_DESIGN}.gds -mapFile ...`：匯出 **GDSII** 版圖檔案（送晶圓廠 tapeout 的格式），`-mapFile` 指定圖層對應表（layer map）。
- `write_lef_abstract ${TOP_DESIGN}.lef`：輸出這個設計的抽象化 LEF 檔案（只含外框、pin、blockage，不含內部細節），供上層設計把這顆電路當作巨集（macro）使用。
- `saveDesign ${TOP_DESIGN}.enc`：儲存 Innovus 資料庫檔案（`.enc`），對應 ICC 的 `save_mw_cel`。

### 與 `STA.md` 的關聯：Post-layout Simulation

> 出處：`~/Downloads/2025_Fall_Training_Package/3.Post_layout_Simulation/post_sim/testbench.v`、`post_sim/sim.sh`

Step.9 的 `write_sdf` 輸出的 `gcd.sdf`，會被 post-layout 模擬的 testbench 讀入做時序反標注（back-annotation）：
```verilog
// testbench.v
$sdf_annotate("../APR/run/gcd.sdf", u1);
```
再用 VCS 編譯模擬（`sim.sh`）：
```bash
vcs testbench.v ../APR/run/gcd_apr.v -v /usr/cadtool/.../tsmc090.v \
  -full64 -R -debug_access+all +v2k +neg_tchk
```
- `testbench.v` 引入的 `gcd_apr.v` 就是 Step.9 `saveNetlist` 輸出的閘級網表；`-v tsmc090.v` 提供標準單元的行為模型（供模擬用，非時序用）；`$sdf_annotate` 把 SDF 中真實的 routing 後延遲數值套用到網表上的每個元件延遲弧（delay arc）。
- 這正是 `STA.md` 第 4 節「**Routing 後 STA**」所述「加入寄生電容和 RC 連線延遲……最接近實際情況」的實務體現：post-layout 模擬用的就是 routing 後、經 STA sign-off 確認過的真實延遲數值，而不是合成階段的統計估算值。
- `+neg_tchk` 讓模擬器檢查負的 timing check（例如 hold time 違例），`-debug_access+all` 開啟波形除錯存取。
