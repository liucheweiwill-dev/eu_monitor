# AGENT_BRIEF — 給後續開發者／AI Agent 的交接文件

**先讀完這份再動 `index.html`。** 這個專案的形狀是四條死路換來的，
文件的主要目的不是說明它做了什麼，而是**阻止你重走那四條路**。

最後更新：2026-09-04

---

## 1. 這是什麼

一個 `index.html`，按一個鈕開三個 Google 分頁，用預先組好的
`site:` + 關鍵字 OR 組 + `tbs=qdr:` 時間篩選查詢，掃三個歐盟／北約機構網站。

- 線上版：<https://liucheweiwill-dev.github.io/eu_monitor/>（GitHub Pages，`main` 分支根目錄）
- 無建置步驟、無依賴、無後端、無測試框架。**單一檔案就是整個應用。**

## 2. 使用者是誰、他要什麼

使用者每天早上做輿情蒐集，固定掃 eeas.europa.eu／consilium.europa.eu／nato.int，
關鍵字大多不變（台海、中國、無人機、海底電纜、印太）。

**最重要的一句話：漏掉（recall）比抓錯（precision）嚴重得多。**

這條原則決定了幾乎所有設計取捨。任何「幫使用者過濾掉雜訊」「自動摘要重點」
的提議，在這個專案裡都是**負面**的，因為使用者無法分辨
「今天真的只有三則」和「你只給我看了三則」。

## 3. 前因：為什麼不是一個「真正的」應用

原本有一個正式專案在做同一件事：
<https://github.com/liucheweiwill-dev/NewsSearch>——Cloudflare Worker + Brave Search API，
已上線、87 個測試、走完六個開發階段。**它做不到時間排序的監看**，
原因記在那個 repo 的 `findings.md` §F 與 §G。摘要如下。

### 四條死路（全部有實測證據，不要重試）

| # | 嘗試過的方向 | 為什麼死 |
|---|---|---|
| 1 | Brave `page_age` 排序 | 機構官網大量頁面沒有這個欄位（R1） |
| 2 | Brave `freshness` 參數 | **它依 `page_age` 篩選**，所以對沒有日期的文章，這個參數在定義上就會排除它。實測目標文章在 `freshness=pw` 下完全缺席 |
| 3 | 把日期字串注入查詢（`"2 September 2026"`） | 兩條件 AND 後結果過少時 Brave 會降級回退，日期條件被實質忽略。要「過去 24 小時」卻回傳兩三個月前的內容 |
| 4 | RSS ingestion | **三站都沒有 feed。** nato.int／eeas.europa.eu 首頁無 feed link 且常見路徑全 404；consilium.europa.eu 對所有機器客戶端回 403 |

另外測掉的：

- **Brave news 端點**：日期品質很好（100% 有 `page_age`、`freshness` 真的生效），
  但**不索引這三站**——eeas 永遠 0 筆，www.nato.int 掛零，consilium 只回落地頁。
- **深分頁 + 本地日期篩選**：`site:nato.int` 翻 5 頁只有 42 筆相異結果、近 7 天 **0 筆**。
  Brave 對廣查詢回的是機構落地頁，分頁很快枯竭。**本地端做什麼都救不回來。**
- **Google Alerts**：文獻指出約 40% 重要新聞未被偵測、延遲 24–72 小時、
  且無執行紀錄——無法分辨「沒有新聞」與「Alert 沒跑」。違反第 2 節的原則。
- **LLM Agent 自動搜尋並摘要**（含 Antigravity／Gemini）：Agent 說「沒有新東西」
  不構成稽核證據。它可能少查一站、提前停止或漏掉連結，而使用者分不出來。

### 根因（2026-09-04 定位）

使用者提出反證：**Google 加「過去一周」找得到**那篇 Brave 找不到的文章。
三項實測把根因收斂到單一層：

1. Brave `site:nato.int Ministers Ireland` → 目標文章排**第 1 名**（索引沒問題）
2. 但該筆 `page_age` 與 `article.date` **皆為 null**
3. `freshness` 依 `page_age` 篩選 → 定義上排除它

**Brave 對機構網站的刊登日期推導遠弱於 Google。這是索引商能力差異，不是參數問題，
換參數、加程式碼都解決不了。**

所以這個專案不再重建搜尋層，直接用已被證實可行的東西：**Google 的介面 + 人的眼睛。**

### consilium.europa.eu 的 403（重要，會擋掉很多提案）

curl 預設 UA、Chrome UA、Googlebot UA **三種都被擋**，所以不是 UA 過濾，
是 JS／指紋挑戰。內建瀏覽器停在 "Checking your browser" 超過 15 秒未通過。

**任何「後端直接抓 consilium」的設計都不可行**，包含 Cloudflare Worker、
Node 腳本、serverless function——它們都不執行 JS。只有真人瀏覽器過得去。

---

## 4. 檔案結構

`index.html`（251 行）分三段，沒有其他原始檔：

| 行 | 內容 |
|---|---|
| 8–95 | `<style>`：CSS 變數 + `prefers-color-scheme` 深色，無框架 |
| 97–129 | HTML 骨架：`#seg`（時間切換）、`#go`（開三分頁）、`#warn`（彈窗被擋提示）、`#sites`（由 JS 產生） |
| 131–249 | `<script>`：純 ES5 語法，無模組、無 build |

### 關鍵函式

| 函式 | 行 | 契約 |
|---|---|---|
| `SITES` | 132 | 資料來源。`{name, domain, kw}`，`kw` 是分號分隔字串 |
| `load()` / `save()` | 143 / 155 | localStorage 讀寫，**必須包 try/catch**（見 §6） |
| `buildQuery(site)` | 163 | `site` → Google 查詢字串。含空白或連字號的關鍵字自動加雙引號 |
| `buildUrl(site)` | 174 | `buildQuery` 的結果 + `encodeURIComponent` + `&tbs=qdr:<qdr>` |
| `render()` | 179 | 產生三個區塊。**一律用 `textContent` / `createElement`，不要用 `innerHTML`** |
| `refreshLinks()` | 217 | 只更新 `href`，切換時間範圍時用，不重繪 DOM |

`buildQuery` 的行為，改動前先確認你沒有破壞：

```
kw = "Taiwan;China"          → site:x.com (Taiwan OR China)
kw = "Taiwan"                → site:x.com Taiwan          （單一關鍵字不加括號）
kw = " ; ; "                 → site:x.com                 （全空 → 純 site: 查詢）
kw = "Indo-Pacific;South China Sea"
                             → site:x.com ("Indo-Pacific" OR "South China Sea")
```

**全空退化成純 `site:` 查詢是刻意的功能**，不是邊界情況——
它配上時間篩選就是「這站這段期間所有新東西」，使用者要全掃時會用。

---

## 5. 常見修改怎麼做

### 加一個網站

改 `SITES`（行 132）加一筆 `{name, domain, kw}` 就好，其餘全部自動。
`render()` 依陣列產生區塊，`#go` 依陣列開分頁。

**但先確認那個站在 Google 上真的有內容**：手動搜一次
`site:<domain>` 加時間篩選，看得到東西再加。加了搜不到的站比不加更糟——
使用者會以為那站沒新聞。

### 改預設關鍵字

改 `SITES` 裡的 `kw`。注意**使用者瀏覽器裡的 localStorage 會覆蓋預設值**
（`load()` 行 143），所以改了預設之後，老使用者不會看到變化。
若要強制更新，得改 localStorage 的 key 名稱（目前是 `eu-nato-monitor`）。

### 加時間範圍選項

`#seg` 裡加一個 `<button data-qdr="X">`，`X` 是 Google 的 `qdr` 值
（`h` 小時、`d` 天、`w` 週、`m` 月、`y` 年，也支援 `d3`／`w2` 這種數字變體）。
事件處理（行 224）是通用的，不需要改。

### 不要改預設時間範圍成「過去 24 小時」

看起來合理，實際是錯的。**Google 收錄有延遲**，一天窗會漏掉昨天發布、
今天才被收錄的文章。一週窗會重複看到熟悉項目，但不會漏——見第 2 節的原則。

---

## 6. 不可破壞的約束

違反這些就會壞掉，或違背專案的目的：

1. **單一檔案、零建置、零依賴。** 不要引入框架、打包器、npm。
   使用者要能雙擊本機檔案就用，也要能放在任何靜態主機上。
2. **localStorage 一律包 try/catch。** 這個檔案也會被當本機 `file://` 開，
   Chrome 對 `file://` 的 localStorage 支援不穩，未保護會直接拋例外並讓整頁失效。
3. **不要用 `innerHTML` 寫入使用者輸入的關鍵字。** 目前全用 `textContent`／
   `createElement`。使用者輸入雖然只回到自己畫面，但沒有理由留這個洞。
4. **保留「判讀提醒」那段 footer 文字。** 它說明「結果為空不代表沒有新內容」。
   這不是免責聲明，是這個工具唯一誠實的地方——Google 不保證每頁都被收錄，
   任何方案都只能證明「這次查詢沒看到」。刪掉它等於讓使用者誤判。
5. **保留彈窗被擋的提示（`#warn`）與三個可獨立點擊的連結。**
   瀏覽器可能擋掉第 2、3 個分頁；沒有 fallback 使用者會以為工具壞了。
6. **不要加自動摘要、自動評分、自動過濾。** 見第 2 節。
7. **`<meta name="robots" content="noindex, nofollow">` 要留著。**
   頁面對拿到網址的人是公開的（GitHub Pages 一律公開，即使 repo 設 private，
   私有 Pages 需要 GitHub Enterprise Cloud），但沒有理由被搜尋引擎收錄。

---

## 7. 什麼情況才值得做大改

目前刻意停在「啟動器」這一層。**唯一已知值得升級的訊號**是使用者說：
「我不想重複看到已經看過的項目。」

那才需要做差異比對（記錄已見 URL、只顯示新增），而那是一個**新的、更難的東西**，
不是這個檔案加幾行。已知的阻擋（2026-09-04 由 Codex 覆核）：

- **Chrome 會鎖定 user-data-dir**，不能共用使用者正在使用的 profile。
  必須另建自動化 profile 並手動登入一次。
- **Windows 工作排程器的非互動工作階段無法可靠驅動 GUI Chrome。**
  「起床時 digest 已經在信箱」做不到；只能在使用者已登入、電腦醒著時跑。
- **Google 的 bot 偵測**：每天三次量很低，但不構成保證，且不得繞過挑戰。
- **ToS**：排程式的 Playwright／Puppeteer 屬自動查詢與擷取，
  與方案 1（人按鈕、人看結果）性質不同。這是風險判斷，要讓使用者知情後決定。
- **稽核性**：若要做，必須把「今天沒有新的」與「這次沒查成功」**分開顯示**。
  三站全部成功才能說前者；任一站遇到 CAPTCHA、同意頁或選擇器失效，
  一律顯示 `unknown`。沒有這個，整個工具在第 2 節的原則下就是不可信的。

**如果真的要做，開新的程式而不是把它塞進 `index.html`。**
這個啟動器的價值就在於它小到不會壞。

---

## 8. 會從外面壞掉的地方

| 風險 | 徵兆 | 處置 |
|---|---|---|
| Google 改 `tbs=qdr:` 語法 | 時間篩選沒生效，結果含舊文章 | 手動在 Google 調一次篩選，複製網址列的新參數 |
| 目標站改網域或路徑 | 某站長期 0 筆 | 手動確認該站是否還在發文，再調 `SITES` |
| 三站之一被 Google 降權 | 同上 | 先手動比對官網列表再下結論，不要直接改程式 |
| 使用者換裝置 | 關鍵字回到預設值 | 正常行為，localStorage 是 per-browser |

**任何一站長期空白時，先手動去該站官網的新聞列表對一次**，再決定是程式問題
還是真的沒新聞。這是第 4 條約束的實務版本。

---

## 9. 相關資料在哪

| 內容 | 位置 |
|---|---|
| 完整證據與決策理由 | <https://github.com/liucheweiwill-dev/NewsSearch> 的 `findings.md` §F、§G |
| 事件時序與判斷更正紀錄 | 同上 repo 的 `progress.md`（2026-09-04 一節） |
| 原本的 Brave 版應用 | 同上 repo，仍在線上，**刻意保留不退役**，它是可用的主題搜尋工具 |

**不要把這兩個 repo 合併。** 它們沒有共用的執行環境或部署路徑：
一個是 Cloudflare Worker + Brave API，一個是靜態單頁 + Google UI。
合併只會讓兩邊都變髒。
