# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

**一般用途的財務可行性試算工具**，不是為特定某家餐廳寫的企劃書——文案（H1／說明文字）刻意用「你的餐飲店」通用第二人稱框架，數字全部是可調整的預設值起點，不是描述某一家真實店家（2026-08-21 依使用者回饋修正過一次，原本文案讀起來像在講一家特定店，已改掉）。以台北／新北一線商圈中高固定成本結構為情境（55 坪／月租 38.5 萬是起始基準值，非強制），把「坪數配置 → 座位數 → 營收 → 成本 → 淨利」整條推算鏈拆成可調整的變數，即時重算損益並依官方建議健康帶判色。**單檔前端，無後端，無序號授權。**

2026-08-21 已從精簡版擴充到完整配置（比照 `行銷內容工具/coffee-ig-planner` 等成熟姊妹專案）：PDF 匯出（含浮水印）、頂部跑馬燈、`manual.html` 操作手冊、訪客計數器、PWA 加入主畫面，見下方各節。

**六種餐飲業態快速範例**（`BUSINESS_PRESETS`，變數面板上方的藥丸按鈕列）：精緻鍋物／涮涮鍋、燒肉、日式定食／輕食便當、主題餐酒館、早午餐／咖啡輕食、熱炒／合菜。每個 preset 覆寫空間配置、定價翻桌、成本占比**與人事編制**（`kitchenFT`/`kitchenPTHours`/`floorFT`/`floorPTHours` 等）——**人事編制一定要跟著業態一起換**，否則低客單價業態（早午餐、熱炒）套用高客單價業態的人力規模，人事占比會爆掉、淨利率會出現不合理的深度負值（曾實測到 -52%，已修正）。目前的預設值刻意讓「熱炒／合菜」在 55坪／38.5萬租金這個情境下維持小幅負利潤（約 -3.7%，租金占比近 30%）——這是刻意保留的洞察（翻桌慢、客單價不夠高的業態撐不住這個租金檔次的一線商圈），不是計算錯誤，不要為了讓所有 preset 都轉正而反過來調整假設。

## 架構

`index.html` 單一 IIFE `<script>`，無外部資料檔：

- `VAR_DEFS` — 所有變數的定義陣列（id/group/label/unit/min/max/step/def/desc，部分含 `band` 健康區間或 `subgroup` 子分類）。UI 由這份定義動態產生（`buildGroupUI`/`buildVarCard`），不是手刻 20+ 組幾乎重複的表單標記。
- `BUSINESS_PRESETS` — 6 組業態範例（見上）。`buildPresetUI()` 產生藥丸按鈕，`applyPreset(p)` 用 `Object.assign` 覆寫對應的 `state` 欄位並重算；使用者手動拖任何滑桿會清掉 `activePresetId`（表示目前是自訂組合，不再對應任何一個 preset），`syncPresetActiveUI()` 負責同步按鈕的 active 樣式。`activePresetId` 額外存一份到 `localStorage`（`restaurantCalcActivePreset`），reload 後還原選取狀態。
- `state` — 目前所有變數值，載入時從 `localStorage`（`restaurantCalcState`）還原，每次滑桿變動即時 persist。
- `calculate()` — 核心計算引擎：坪數配置 → 座位數 → 平/假日營收 → 各項成本金額與占比 → 淨利／淨利率／坪效。公式細節見計畫檔（`~/.claude/plans/api-38-5-synchronous-cerf.md`，若已被清除則以此函式本體為準）。
- `healthState(ratio, band)` — 依建議健康帶（如食材成本 28–34%）判定 `good`/`bad`，驅動票根與診斷的紅綠判色。店租占比與淨利率用單一門檻（15%）判定，其餘用區間。
- `renderFloorPlan` — 互動平面圖（signature element）：依廚房/動線/內用占比即時重排色塊，首次載入才做 stagger fade-in（`prefers-reduced-motion` 時跳過）。平面圖下方 `#fpFormula` 會即時印出完整算式（總坪數 × 內用占比 = 內用坪數 ÷ 每客席坪數 ≈ 座位數），讓「每客席坪數是套用在扣除廚房/動線後的內用坪數，不是總坪數」這件事在畫面上是透明可查的，不用只靠 `pingPerSeat` 的說明文字（2026-08-21 依使用者回饋補上，見 CLAUDE.md 記憶檔）。
- `renderTicket` — 損益試算票根（monospace 收據卡），比例跨過健康門檻時該行短暫 flash。
- `AI_PROVIDERS`／`callLLM`／BYOK 設定面板 — **逐字比照** `行銷內容工具/coffee-ig-planner/index.html` 已驗證的實作搬過來（Claude 需要 `anthropic-dangerous-direct-browser-access` header 才能瀏覽器直連；OpenAI/Gemini/OpenRouter 無此限制；429/500/503/529 自動重試 3 次）。修改任一邊的 AI 引擎邏輯時，考慮是否也要同步另外兩邊。
- `buildDiagnosisPrompt(res, extra)` — 把目前變數與試算結果（含哪些比例超出健康帶）整理成 prompt，請模型寫繁中診斷＋建議；`extra` 非空時以「特別要求：」附加在 prompt 最後一行。
- `AI_PROMPT_PRESETS` — 5 組診斷角度範例（整體健檢／坪效與選址適配度／成本優化排序／貸款募資簡報用語／悲觀情境壓力測試），`buildAiPromptUI()` 產生 `#aiPromptRow` 的按鈕，點擊即把對應文字填入 `#aiExtra`（使用者仍可自行編輯或清空）。**只影響 AI 診斷路徑**——`ruleBasedDiagnosis` 是純規則式 fallback，不讀取 `#aiExtra`，固定套用通用邏輯。`extra` 欄位一併存進 `restaurantCalcApiConfig`（見下）。
- `ruleBasedDiagnosis(res)` — 無金鑰或 AI 呼叫失敗時的規則式健檢 fallback，依超標項目組出對應建議句，確保沒有金鑰也能得到有意義的診斷。

### localStorage

- `restaurantCalcState` — 目前所有變數值（reload 後還原，不會跳回預設值）。
- `restaurantCalcApiConfig` — `{provider, model, apiKey, extra}`，金鑰只存本機瀏覽器，不經任何後端；`extra` 是診斷角度文字（見上）。
- `restaurantCalcActivePreset` — 目前選取中的業態 preset id（沒有選取或已手動調整過滑桿則不存在此 key）。

「重設為基準假設」按鈕清掉 `restaurantCalcState`／`restaurantCalcActivePreset` 並還原成 `VAR_DEFS` 裡的基準預設值（55坪／38.5萬租金／客單價700等，這組基準值不對應任何一個業態 preset，是中性起點）。

## PDF 匯出（`#pdfExportBtn`）與操作手冊連結

2026-08-21 依使用者回饋，「📄 匯出 PDF」與「📖 操作手冊」從頁尾移到 hero 區塊右上角的 `.hero-utility` 列（`#pdfExportBtn` id 不變，只是換了 DOM 位置，JS 綁定不受影響）；頁尾只保留「📲 加入主畫面」／「重設為基準假設」／訪客計數器。`.hero-utility` 在 `@media print` 一併隱藏（不是報表內容）。

不依賴任何 PDF 函式庫：按鈕呼叫 `window.print()`。

**2026-08-21 改版**：原本比照 `資料儀表板/Dashboard` 用 `@media print` 逐一切換數十個互動元件的 `display`（切分頁、隱藏滑桿、`.ai-card:has(...)` 等），使用者實測回報「匯出時候是沒有內容」——這條路線太脆弱，任何一條規則沒對到就整頁空白，且瀏覽器「列印背景圖形」選項預設關閉時 `background-image` 浮水印也不會印出來。**改成「獨立靜態報表」路線**，不再嘗試讓互動版 UI 本身可列印：

- `buildPrintReport()`：按下「匯出 PDF」時才呼叫，用目前的 `state`／`calculate()`／`GROUPS`／`VAR_DEFS` 組出一份純 HTML 報表字串（標題＋產生時間、空間配置算式、五組變數總覽表、損益票根表格、AI 診斷結果〔若曾產生過〕、免責聲明），寫入 `#printReportRoot.innerHTML`，才呼叫 `window.print()`。所有動態文字（尤其 AI 回傳的診斷內容）一律過 `escapeHtml()` 再拼字串，避免萬一 AI 回應內容含 HTML/script 被當成標記注入。
- `@media print` 規則簡化成一條：`body *{visibility:hidden} #printReportRoot,#pdfWatermark,及其子元素{visibility:visible}`，`#printReportRoot{position:absolute;top:0;left:0}` 拉到頁面最前面——不用再猜互動元件個別的 display 狀態，永遠只印這兩個容器的內容，其餘全部隱藏但不影響版面（`visibility:hidden` 保留 layout 但不繪製，`position:absolute` 讓報表脫離原本文件流從頁首開始）。已用 Node 建假 DOM 跑過 `buildPrintReport()`，確認輸出非空、標籤配對正確、AI 文字內的 `<script>` 有被正確跳脫。
- `#pdfWatermark`（浮水印）：**不用 `background-image`**（會被瀏覽器「列印背景圖形」選項擋掉），改成 HTML 直接寫死 21 個 `<span class="wm-text">僅供試算參考</span>`，`@media print` 用 CSS Grid（3欄×7列）排列、`rotate(-28deg)`、`color:rgba(27,42,68,.22)`——前景文字顏色不受「列印背景圖形」開關影響，永遠會印出來。跟 `IPA_Kano` 用真人照片圖片浮水印（課程授權情境）不同——這個工具沒有序號授權，浮水印純粹是免責聲明，用重複文字就夠。

## 頂部跑馬燈與 PWA 加入主畫面

逐字比照 `行銷內容工具/coffee-ig-planner/index.html` 已驗證的實作（各自獨立 IIFE `<script>`，跟主程式邏輯無關）：

- 跑馬燈：`MARQUEE_CHECK_URL` 是工作區多個工具共用的同一顆 Google Apps Script 端點，POST 空 `serial`，只取回傳的 `marquee` 陣列；localStorage key 為 `restaurantCalcMarquee`。本頁沒有 sticky topbar，`body.has-marquee{padding-top:30px}` 就夠。改跑馬燈內容直接編輯共用 Google Sheet，不需要重新部署 Apps Script。
- PWA：`manifest.json`＋`service-worker.js`（network-first＋同源快取備援，`fetch(request,{cache:'reload'})` 這個細節必須保留）＋`icons/`（PIL＋`C:\Windows\Fonts\msjhbd.ttc` 產生，深藍色底 `#1B2A44`＋米白色單字「坪」，192／512／maskable-512／apple-touch-icon 四種尺寸；產生腳本未進 repo，比照姊妹專案慣例）。安裝按鈕 `#installBtn`＋獨立 `#toast` 元素＋逐字沿用已修好 bug 的安裝腳本（見 [[pwa-install-rollout]] 記載的兩個踩坑：腳本執行時機須晚於按鈕元素解析、`notify()` 自帶避免跨作用域抓不到 `showToast`）。

## 操作手冊（manual.html）與訪客計數器

`manual.html` 自成一頁、內嵌 `<style>`（配色沿用本頁 token，非深色主題）。內容：操作步驟／業態範例說明／AI 診斷角度說明／PDF 匯出說明／PWA 安裝說明／隱私說明／使用警語（財務工具版本，強調「試算結果為估計值、不是正式財務報表、不能取代會計師」）／創作者資料／授權限制。**創作者資料區塊逐字比照** `行銷內容工具/coffee-ig-planner/manual.html` 等姊妹專案，更新其中一邊時同步其餘各邊。`index.html` footer 有連結到 `manual.html`。

訪客計數器：`visitor-badge.laobi.icu` 的 SVG badge，`page_id=m255525.restaurantfeasibilitycalculator`，免金鑰免後端，比照姊妹專案慣例，放在 footer。

## 部署

已推公開 GitHub repo：<https://github.com/M255525/restaurant-feasibility-calculator>，已用 `.github/workflows/deploy-pages.yml`（Actions 部署模式，比照 `workspace-git-repos` 記載的「不要用 legacy branch-source」慣例）啟用 GitHub Pages：<https://m255525.github.io/restaurant-feasibility-calculator/>（2026-08-21）。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8797`（`.claude/launch.json` 的 `restaurant-feasibility-calculator` 項目），用 Preview MCP 的 `preview_start` 啟動；若該 MCP 在當次工作階段不可用，退回 `python -m http.server 8797 --directory 資料儀表板/restaurant-feasibility-calculator` 暫起、測完關閉。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式，確認 `callLLM → showAiResult` 整條管線正確，測完記得還原 `window.fetch`。
