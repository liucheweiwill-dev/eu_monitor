# AGENT_BRIEF — 給後續開發者／AI Agent 的交接文件

**先讀完這份再動 `index.html`。** 這個專案的形狀是四條死路換來的，
文件的主要目的不是說明它做了什麼，而是**阻止你重走那四條路**。

最後更新：2026-09-22

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
是 JS／指紋挑戰。2026-09-22 複驗仍是 403，`<title>Browser check - Consilium</title>`。

**擋的是自動化指紋，不只是「不執行 JS」。** 會執行 JS 的自動化瀏覽器等 18 秒
仍停在 "Checking your browser before accessing a GSC Managed Website"（2026-09-22 實測）。

**任何「後端直接抓 consilium」的設計都不可行**，包含 Cloudflare Worker、
Node 腳本、serverless function，**以及 Playwright／Puppeteer 這類自動化瀏覽器**。
只有真人手動操作的瀏覽器過得去。

這也表示 §10 那套 curl 驗證法對這一站用不了——見 §10 結尾。
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
若要強制更新，得改 localStorage 的 key 名稱（目前是 `eu-nato-monitor-v2`，§10 那次改過一輪）。

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

---

## 10. EEAS 導覽列確實讓三個關鍵字失去篩選力（已修）

**狀態：成立，已於 2026-09-22 修掉。**
時序：2026-09-04 提出懷疑 → 09-10 **誤判為「不成立」** → 09-22 使用者拿反例推翻，確認成立。

**先讀下面那段「09-10 為什麼會判錯」**——那個錯誤的驗證方法看起來很有說服力，
不寫下來的話很容易有人再用一次。

### 機制：一個固定字串，不是零散污染

EEAS 的頂端 mega dropdown（Regional policies → See also）在**每一個 `/eeas/*` 頁面**
都放了這一行：

```html
<div class="col-md-6 paragraph paragraph--type--mega-dropdown-section ...">
  <a href="/eu-indo-pacific-strategy-topic_en">EU Indo-Pacific Strategy</a>
```

另有一個國家選單：

```html
<option value="/china_en">China</option>
<option value="/taiwan_en">Taiwan</option>
```

Google 確實據此配對。反例（使用者 2026-09-22 提供）：

<https://www.eeas.europa.eu/eeas/opening-remarks-un-side-event-humanitarian-diplomacy-safeguards-international-humanitarian-law_en>

這頁正文完全沒提印太，整份 HTML 裡 `Indo-Pacific` 只出現 2 次、都在上面那個選單連結。
它卻出現在 `site:eeas.europa.eu/eeas "Indo-Pacific"` 的結果裡，
而且 **Google 自己產生的摘要就是那段選單文字**：

```
... Indo-Pacific Strategy · Election ... Opening remarks for the UN Side Event on ...
```

「Indo-Pacific Strategy · Election」是下拉選單相鄰的兩個項目。鐵證。

### 09-10 為什麼會判錯：分母用錯了

當時比的是**全站**，看起來很乾淨：

| | 全站 `site:eeas.europa.eu` | 正確的 `site:eeas.europa.eu/eeas` |
|---|---|---|
| 基準頁數 | 7,210 | **843** |
| `Taiwan` | 703（10%）✅ 看起來正常 | 662（**78%**）❌ |
| `China` | 958（13%）✅ | 728（**86%**）❌ |
| `"Indo-Pacific"` | 730（10%）✅ | 728（**86%**）❌ |

**7,210 之中絕大多數是 `/delegations/*` 子站，那些頁用不同的模板、根本沒有這個 mega dropdown**
（實測 `delegations/jordan_en` 的 `indo-pacific` 出現次數為 0）。
真正在發新聞的 `/eeas/*` 只佔全站約 12%，於是 86% 的污染被稀釋成 10% 的假象。

**教訓：比例法的分母必須是「確實帶有那段 chrome 的路徑」，不是整個網域。**
而要知道哪些路徑帶有那段 chrome，你還是得去看 HTML——所以不如直接用下面的方法。

### 正確的驗證方法：curl 兩頁對照，不要用 Google 比例

```bash
# 抓兩個「正文與該關鍵字完全無關」的同站頁面
curl -sS -L -A "Mozilla/5.0 ... Chrome/131.0 ..." -o a.html "<該站任一篇無關文章>"
curl -sS -L -A "Mozilla/5.0 ... Chrome/131.0 ..." -o b.html "<該站的機構介紹頁>"

# 關鍵字若出現在這兩頁，它就在全站 chrome 裡
grep -o -i "<keyword>" a.html | wc -l
grep -o -i "<keyword>" b.html | wc -l
```

比 Google 比例法好在四點：離線、即時、**因果直接**（直接看字串在不在 chrome 裡，
而不是從排名回推）、而且**不會觸發 bot 偵測**。

實測結果（對照頁：那篇 UN 開場致詞 ＋ `about-european-external-action-service_en`）：

| 關鍵字 | 對照頁命中 | 判定 |
|---|---|---|
| `Taiwan` | 2 / 2 | ❌ 在 chrome 裡 |
| `China` | 4 / 4 | ❌ |
| `Indo-Pacific` | 2 / 3 | ❌ |
| `Chinese`／`PRC`／`NATO`／`drone`／`cables` | 0 / 0 | ✅ 乾淨 |
| `Taiwan Strait`／`cross-Strait`／`Beijing`／`EU-China` | 0 / 0 | ✅ |
| `the Indo-Pacific`／`Indo-Pacific region` | 0 / 0 | ✅ |

⚠️ **這個方法對 `consilium.europa.eu` 不能用**——它對所有不執行 JS 的客戶端回 403（見 §3）。
那一站只能退回比例法，而且要自己先找出正確的分母路徑。

### 兩個會咬人的細節

**1. 污染是固定字串，所以「加長片語」就能逃掉。**
`Indo-Pacific` 在無關頁面上只以 `EU Indo-Pacific Strategy` 出現，
所以任何**不是它子字串**的片語都是乾淨的。`Taiwan`／`China` 在選單裡是裸字，
所以任何兩字以上的片語都乾淨。

`"the Indo-Pacific"` 是這裡召回最高的選擇——導覽字串沒有 `the`，
而英文正文講這個區域幾乎都帶定冠詞（EEAS 印太主題頁：`Indo-Pacific` 共 72 次，
其中 `the Indo-Pacific` 20 次、導覽字串 11 次）。

**2. 但 Google 的詞形正規化會把片語打回原形。**
`China's` 在 HTML 裡是乾淨的，但 Google 會把所有格還原成 `China` → 又中污染。
**所以替代片語必須含第二個實詞**（`"Taiwan Strait"`、`"EU-China"`、`"Indo-Pacific region"`），
不能靠所有格或標點。

同理，`"the Indo-Pacific"` 依賴 Google 在引號內尊重 stop word `the`——
這一點**尚未實測**（Google 當時已對本機 IP 出 bot 驗證頁）。
因此 OR 群組裡同時放了 `"Indo-Pacific region"` 當保險。OR 群組多放一個片語成本為零。

### `intext:` 不能用（不要再試）

Google 沒有「只搜正文」的操作符。`intext:`／`allintext:` 已經失效：

| 查詢（皆 `qdr:y`） | 結果 |
|---|---|
| `site:eeas.europa.eu/eeas "Indo-Pacific"` | 728 |
| `site:eeas.europa.eu/eeas intext:"Indo-Pacific"` | **0** |
| `site:eeas.europa.eu/eeas intext:Indo-Pacific` | **1** |

0 筆不是過濾成功，是查詢被打壞——該站顯然有數百頁正文真的在談印太。

而且更根本：**對 Google 而言導覽列就是頁面文字**，它對人類訪客也確實顯示在畫面上。
「正文 vs. 選單」這個區分在查詢語言裡不存在。

### 實際改了什麼（2026-09-22）

EEAS 區塊的 `kw`：

```
舊： Taiwan;China;Chinese;PRC;NATO;drone;cables;Indo-Pacific
新： Taiwan Strait;cross-Strait;Chinese;PRC;Beijing;EU-China;the Indo-Pacific;Indo-Pacific region;NATO;drone;cables
```

同時把 localStorage key 從 `eu-nato-monitor` 改成 `eu-nato-monitor-v2`，
否則舊使用者瀏覽器裡存的舊關鍵字會蓋掉新預設值（見 §5），修了等於沒修。

**這是一次召回換精確度的交易，使用者知情後同意的。** 代價是：
正文只寫 `China`／`Taiwan` 而不寫 `Chinese`／`PRC`／`Beijing`／`EU-China`／`Taiwan Strait`
的文章會被漏掉。

之所以這個交易還算划算：舊的 `China` 命中 `/eeas/` 的 86%，
它並不是在提供召回，而是等同於沒有關鍵字——使用者本來就在看幾乎全部的頁面。

**但這條原則沒有變：一般情況下把關鍵字換窄是錯的方向**（見 §2）。
只有在實測證明某個字已經失去篩選力時，換掉它才不算損失。

### 另外兩站的結果（2026-09-22 同日驗完）

**`nato.int`：同一個病，已一併修掉。**

對照頁用 nato.int 的 **404 頁**（純 chrome、零正文，是最理想的對照組）
與「創始條約」頁。legacy 的 `/cps/en/natohq/topics_*.htm` 網址會 301 導到
新的 `/en/*` 結構並送出相同的 chrome，所以**整站都帶污染**。

| 關鍵字 | 404 頁 / 條約頁 | 判定 |
|---|---|---|
| `Taiwan` | 2 / 2 | ❌ `<option value="Taiwan">` |
| `China` | 6 / 6 | ❌ 同上，另有 Hong Kong／Macao SAR China |
| `Indo-Pacific` | 4 / 4 | ❌ 見下 |
| `Chinese`／`PRC`／`drone`／`cables` | 0 / 0 | ✅ |

⚠️ **關鍵教訓：安全片語是逐站不同的。**
NATO 的導覽列寫的是「Relations with partners **in the Indo-Pacific region**」，
所以 EEAS 那邊安全的 `"the Indo-Pacific"` 與 `"Indo-Pacific region"`
**在 NATO 站兩個都中招**。替某站挑片語時，必須拿**該站**的頁面重驗一次，
不能沿用另一站的結論。

nato.int 的 `kw` 改成：

```
舊： Taiwan;China;Chinese;PRC;drone;cables;Indo-Pacific
新： Taiwan Strait;cross-Strait;Chinese;PRC;Beijing;Indo-Pacific partners;Asia-Pacific;South China Sea;drone;cables
```

（仍然刻意不含 `NATO`，理由見 §10 開頭與 NewsSearch `findings.md` K.2。）

**這一站的印太覆蓋比 EEAS 弱，要知道。** 在 NATO 的印太夥伴關係頁上，
`the Indo-Pacific` 命中 38 次而 `"Indo-Pacific partners"` 只有 6 次——
但前者已被導覽列污染，不能用。目前靠 `"Indo-Pacific partners"` ＋ `"Asia-Pacific"`
＋ 中國／台海那組片語一起兜。若日後發現漏掉印太相關報導，這裡是第一個要查的地方。

**`consilium.europa.eu`：無法驗證，關鍵字維持原狀。**

curl 仍回 403（`<title>Browser check - Consilium</title>`）。
比 §3 記錄的更嚴格的是：**內建的自動化瀏覽器會執行 JS，等 18 秒仍停在
"Checking your browser before accessing a GSC Managed Website"。**
所以它擋的是自動化指紋，不只是「不執行 JS 的客戶端」。

這表示 §10 這套 curl 驗證法**對 consilium 完全用不了**，而比例法也不可靠
（要先知道哪條路徑帶 chrome，那又得讀 HTML）。

**目前唯一可行的驗法是使用者自己用真人瀏覽器打開該站任一篇無關文章，
檢視原始碼搜尋關鍵字。** 在那之前不要動 consilium 的 `kw`——
沒有證據就改，違反第 2 節的原則。

### 真正還沒驗的（只剩這三件）

1. **`"the Indo-Pacific"` 在 Google 是否尊重引號內的 stop word `the`。**
   已用 OR 群組對沖（同時放了 `"Indo-Pacific region"`），不影響現在能不能用，
   但確認之後可以把冗餘那個拿掉。
2. **替代片語在 Google 上的實際命中數。** curl 法證明了「乾淨」，
   沒證明「召回夠」。兩者是不同的問題。
3. **consilium 的污染狀況。** 只能由使用者用真人瀏覽器檢視原始碼。

第 1、2 項當天卡在 Google 的 bot 驗證頁（見下）。

### 附帶記錄：Google 的 bot 偵測門檻

2026-09-22 在幾分鐘內送出約 15 條 `site:` 查詢後，Google 即出示驗證頁。
**不得繞過**（見 §7）。這也是上面推薦 curl 法的理由之一，
以及為什麼 §7 那個「排程自動查詢」的方向風險比看起來高。
