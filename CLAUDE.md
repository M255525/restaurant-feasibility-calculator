# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

初期經營餐飲店的財務可行性試算工具。以台北／新北一線商圈中高固定成本結構（月租 38.5 萬／55 坪）為原型模型，把「坪數配置 → 座位數 → 營收 → 成本 → 淨利」整條推算鏈拆成可調整的變數，即時重算損益並依官方建議健康帶判色。**單檔前端，無後端，無序號授權**（精簡版範圍，未來如需 manual.html／PWA／跑馬燈／訪客計數器可再擴充，比照 `行銷內容工具/coffee-ig-planner` 等成熟姊妹專案）。

## 架構

`index.html` 單一 IIFE `<script>`，無外部資料檔：

- `VAR_DEFS` — 所有變數的定義陣列（id/group/label/unit/min/max/step/def/desc，部分含 `band` 健康區間或 `subgroup` 子分類）。UI 由這份定義動態產生（`buildGroupUI`/`buildVarCard`），不是手刻 20+ 組幾乎重複的表單標記。
- `state` — 目前所有變數值，載入時從 `localStorage`（`restaurantCalcState`）還原，每次滑桿變動即時 persist。
- `calculate()` — 核心計算引擎：坪數配置 → 座位數 → 平/假日營收 → 各項成本金額與占比 → 淨利／淨利率／坪效。公式細節見計畫檔（`~/.claude/plans/api-38-5-synchronous-cerf.md`，若已被清除則以此函式本體為準）。
- `healthState(ratio, band)` — 依建議健康帶（如食材成本 28–34%）判定 `good`/`bad`，驅動票根與診斷的紅綠判色。店租占比與淨利率用單一門檻（15%）判定，其餘用區間。
- `renderFloorPlan` — 55 坪互動平面圖（signature element）：依廚房/動線/內用占比即時重排色塊，首次載入才做 stagger fade-in（`prefers-reduced-motion` 時跳過）。
- `renderTicket` — 損益試算票根（monospace 收據卡），比例跨過健康門檻時該行短暫 flash。
- `AI_PROVIDERS`／`callLLM`／BYOK 設定面板 — **逐字比照** `行銷內容工具/coffee-ig-planner/index.html` 已驗證的實作搬過來（Claude 需要 `anthropic-dangerous-direct-browser-access` header 才能瀏覽器直連；OpenAI/Gemini/OpenRouter 無此限制；429/500/503/529 自動重試 3 次）。修改任一邊的 AI 引擎邏輯時，考慮是否也要同步另外兩邊。
- `buildDiagnosisPrompt(res)` — 把目前變數與試算結果（含哪些比例超出健康帶）整理成 prompt，請模型寫繁中診斷＋建議。
- `ruleBasedDiagnosis(res)` — 無金鑰或 AI 呼叫失敗時的規則式健檢 fallback，依超標項目組出對應建議句，確保沒有金鑰也能得到有意義的診斷。

### localStorage

- `restaurantCalcState` — 目前所有變數值（reload 後還原，不會跳回預設值）。
- `restaurantCalcApiConfig` — `{provider, model, apiKey}`，金鑰只存本機瀏覽器，不經任何後端。

「重設為預設值」按鈕清掉 `restaurantCalcState` 並還原成 brief 原文的預設值（55坪／38.5萬租金／客單價700等）。

## 指令

無建置步驟。直接開啟 `index.html`（`file://`）或用伺服器託管即可。

預覽伺服器：port `8797`（`.claude/launch.json` 的 `restaurant-feasibility-calculator` 項目），用 Preview MCP 的 `preview_start` 啟動；若該 MCP 在當次工作階段不可用，退回 `python -m http.server 8797 --directory 資料儀表板/restaurant-feasibility-calculator` 暫起、測完關閉。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式，確認 `callLLM → showAiResult` 整條管線正確，測完記得還原 `window.fetch`。
