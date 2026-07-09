# BNI 華都更分會 — 培訓進度回報工具

沿用 `C:\Users\Joan\bni-powerteam\` 同一套免費三件套架構：

```
夥伴手機/電腦
   │ 開網頁（GitHub Pages）
   ▼
index.html：下拉選自己名字 → 帶出目前 5 堂課勾選狀態 → 勾選/更正 → 送出
   │ GET  ?name=X    （讀目前狀態，seed 或最新一筆）
   │ POST no-cors + text/plain （寫入新一筆）
   ▼
Google Apps Script Web App（Code.gs：doGet / doPost）
   │ appendRow（保留完整歷史）
   ▼
Google Sheet：「填答紀錄」（每次送出一列） + 「即時統計」（自動重算）
```

## 跟 bni-powerteam 的主要差異

| | bni-powerteam（PowerTeam 接龍） | bni-training（培訓進度回報） |
|---|---|---|
| 填答對象 | 55～57 人「共用一份選項」（選群組） | **每人只看/只填自己那一份**（選名字才出現自己的課表） |
| 前端輸入 | 姓名自由輸入框 | **下拉選單**（57 人固定名單，避免手誤/避免看到別人資料） |
| 資料形狀 | 每人對 5 個群組打勾（是否想加入） | **每人對 5 堂課打勾**（是否已完成），且**帶入總部原始記錄當預設值** |
| 目前狀態讀取 | 前端不讀後端資料，直接顯示固定選項 | 前端 `doGet ?name=X` **主動去問後端**「這個人目前狀態」，得到「他自己最新一次送出」或「總部原始 seed」 |
| 重複送出 | 送出後清空欄位、重新填 | **保留選定的名字與已勾狀態**，方便同一人之後再回來更正/補登 |
| 統計重點 | 各群組想加入人數 + 名單 | 每堂課**已完成人數**（seed + 自行更正合計）+ **尚未回報名單**（催繳用，指從沒送出過的人） |
| 姓名比對 | `normalizeName` / `rosterFilled`（因自由輸入常有暱稱/英文別名變體） | 保留同一套 `normalizeName`，但因為改成**下拉選單鎖定選項**，實際發生變體的機會大幅降低，主要用途變成防呆（Joan 之後若手動編輯 Sheet 資料、或未來改回自由輸入時仍安全） |

## 前端在未部署狀態下怎麼運作（供本機預覽）

`index.html` 內嵌了一份 `ROSTER_SEED`（跟 `Code.gs` 的 `ROSTER` **內容逐筆核對過，57 人 + 5 堂課布林值完全一致**）：

1. 選名字的瞬間，**先用內嵌的 `ROSTER_SEED` 立刻渲染** 5 個 checkbox（不用等網路，未部署也能直接雙擊 `index.html` 在瀏覽器打開預覽）。
2. 接著才 `fetch(SCRIPT_URL + '?name=...')` 去問後端「目前狀態」：
   - `SCRIPT_URL` 還是預留字串（`__PASTE_YOUR_APPS_SCRIPT_URL_HERE__`）→ 直接跳過 fetch，維持步驟 1 的內嵌資料，畫面顯示「（預覽模式：尚未接上後端…）」。
   - 已填真網址但 fetch 失敗（未部署 / 網路問題）→ 同樣退回內嵌資料，畫面顯示「（連不到後端，暫時顯示內建的總部原始記錄…）」。
   - fetch 成功 → 用後端回傳的「目前狀態」**覆蓋**畫面（若這個人之前已經送出過修正，這裡才會是最新版，不是原始 seed）。

送出永遠走真的 `POST`（`no-cors` + `text/plain`），沒接後端會直接顯示「⚠️ 系統尚未設定完成」擋下，不會假裝送出成功。

## 自我測試（開發階段已做，未接真實 GAS）

- `node --check` 驗過 `Code.gs` 與 `index.html` 內嵌 `<script>` 兩者語法皆合法。
- 寫了一支 Node sandbox（mock `SpreadsheetApp` / `ContentService` / `HtmlService` / `LockService`），把 `Code.gs` 整份載入執行，跑了 24 個案例（seed 帶出、送出後覆蓋、二次更新取最新、歷史列保留、姓名正規化「035陳昭男」→「陳昭男」、統計人數/催繳名單正確性），**24/24 全過**。這只驗證「JS 邏輯本身正確」，**不等於「真實 Google Apps Script 環境跑起來沒問題」**（GAS 有自己的執行環境差異，例如 `Date` 序列化、`LockService` 真實併發），部署後仍需照《部署教學.md》第四段實機測試一次。

## 已知風險（本次未在真實環境驗證，建議部署後優先測）

- **`doGet` 的 CORS 可讀性**：前端用一般 `fetch(GET)`（非 `no-cors`）去讀 Apps Script 回傳的 JSON。這是業界常見寫法（GET 屬「simple request」，Google 對 `script.googleusercontent.com` 的回應通常帶允許跨網域的標頭），理論上可行，但這個專案先前的踩坑紀錄裡沒有「瀏覽器 fetch GET 讀 GAS JSON」的實測先例（`bni-powerteam` 唯讀 API 只被 curl/後台呼叫過，前端從沒真的 fetch 過）。**建議部署後第一件事就是照《部署教學.md》第四段測「選名字有沒有正確帶出勾選」**，如果讀不到，最保守的備援是把這支 GET 也包一層 `no-cors` 但那樣前端讀不到內容——屆時再回來調整（例如 fallback 到只用內嵌 seed、正確狀態由 Joan 事後看 Sheet）。

## 檔案清單

| 路徑 | 說明 |
|---|---|
| `C:\Users\Joan\bni-training\index.html` | 前端填答頁（含內嵌 57 人 seed，未部署也能預覽） |
| `C:\Users\Joan\bni-training\Code.gs` | 後端（ROSTER 57 人 + doGet/doPost/ensureStatsSheet） |
| `C:\Users\Joan\bni-training\部署教學.md` | 給 Joan 的白話部署步驟（含「絕不選新增部署作業」警告） |
| `C:\Users\Joan\bni-training\README.md` | 本檔 |
