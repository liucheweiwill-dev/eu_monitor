# CLAUDE.md

本檔給 Claude Code 及**任何**在此 repo 工作的 AI Agent（Codex 等同樣適用）。

## 動 `index.html` 之前，先讀 `docs/AGENT_BRIEF.md`

這個專案只有一個 251 行的 `index.html`，按一個鈕開三個 Google 分頁。
它看起來像是「還沒做完的東西的暫時替代品」——**不是**。
它是四條實測死路之後唯一站得住的做法。

`docs/AGENT_BRIEF.md` 記錄了那四條死路與它們的證據。**不先讀就動手，
你會重走一遍已經走過的路。**

## 三條最容易被「優化」掉的約束

這三條看起來都像沒寫好，其實都是刻意的：

1. **`localStorage` 一律包 try/catch**——這個檔案也會被當本機 `file://` 開，
   Chrome 對 `file://` 的 localStorage 支援不穩，拿掉保護會讓整頁失效。
2. **關鍵字全空要退化成純 `site:` 查詢**——那是功能不是邊界情況，
   使用者要「這站這週所有新東西」時就靠它。
3. **footer 的「判讀提醒」要留著**——Google 不保證每頁都被收錄，
   所以結果為空只能代表「這次沒看到」。那段字是這個工具唯一誠實的地方。

完整清單（七條）在 `docs/AGENT_BRIEF.md` §6。

## 決定一切的一句話

**漏掉（recall）比抓錯（precision）嚴重得多。** 使用者用這個做輿情蒐集。

所以：**不要加自動摘要、自動評分、自動過濾。**
使用者無法分辨「今天真的只有三則」和「你只給我看了三則」，
而後者正是他最不能接受的失敗。同理，不要把判讀工作交給 LLM——
Agent 說「沒有新東西」不構成稽核證據。

## 不要重試的（證據見 `docs/AGENT_BRIEF.md` §3）

Brave `page_age` 排序 ／ Brave `freshness` 參數 ／ 日期字串注入 ／ RSS ingestion ／
Brave news 端點 ／ 深分頁加本地篩選 ／ Google Alerts ／ LLM Agent 自動摘要 ／
**三站合成一條查詢、只開一個分頁** ／ Google 的 `&num=100`。

最後兩條特別容易被當成「優化」：三個分頁合成一個看起來完全合理，
但實測巢狀的 `(site:A (…)) OR (site:B (…))` 會**整條回 0 筆、連警告都沒有**——
使用者會以為這週三站都沒新東西。`&num=100`（一頁顯示全部）Google 已經不接受，
照樣 10 筆一頁。兩者皆 2026-09-23 實測，細節見 `docs/AGENT_BRIEF.md` §3。

**特別注意**：`consilium.europa.eu` 擋的是**自動化指紋，不只是「不執行 JS」**。
curl 三種 UA 皆回 403；**會執行 JS 的自動化瀏覽器等 18 秒仍停在
"Checking your browser"**（2026-09-22 實測）。所以 Worker、Node 腳本、serverless
**以及 Playwright／Puppeteer** 都抓不到它——這擋掉的是一整類提案，不只是 RSS。

## 替任何站設關鍵字之前，先確認那個字還有篩選力

> **2026-09-24 起，三站的預設關鍵字刻意包含導覽列裡就有的字（Taiwan、China、Indo-Pacific），
> 由使用者指定。** 9/22 換成片語，是因為當時沒辦法分辨「正文命中」和「導覽列命中」，
> 只好在 Google 那層就把污染字拿掉——那是召回換精確度的交易。現在
> [eu_verify](https://liucheweiwill-dev.github.io/eu_verify/) 會拿每站的 404 頁當基準線，
> 逐頁分出導覽列命中，使用者選擇把召回換回來：Google 回傳幾乎所有頁面，篩選交給 eu_verify。
> **不要以「污染」為由把這些字換回片語。** 本節的驗證方法仍然成立，
> 但只在使用者要求 Google 那層也要過濾時才用得上。來龍去脈見 `docs/AGENT_BRIEF.md` §10 最後一節。

機構網站的全站導覽列與選單會把國名、主題名塞進**每一頁**的 HTML。
若 Google 據此配對，那個關鍵字對該站就等於不存在——而你不會發現，
因為查詢照樣回結果，只是回的是整個網域。

**驗證要兩步，缺一不可。**

**第一步（乾淨）：curl 兩個「正文與該關鍵字無關」的同站頁面，grep 那個字。**
出現了就是在 chrome 裡。離線、即時、不會觸發 Google 的 bot 偵測。

```bash
curl -sS -L -A "Mozilla/5.0 ... Chrome/131.0 ..." -o a.html "<該站任一篇無關文章>"
grep -o -i "<keyword>" a.html | wc -l      # > 0 就是污染
```

**第二步（有用）：在 Google 查一次，看頁首有沒有「找不到…的結果」那行。**
有的話代表這個片語零命中，Google 會**自動脫掉引號退回**成裸字——
而裸字正是污染源。一個「乾淨但零命中」的片語比原本的裸字更糟，
因為它看起來像修好了。（`"EU-China"` 在 EEAS、`"Taiwan Strait"` 在 nato.int
都是這樣中招的。）

⚠️ **`site:` 涵蓋子網域，而子網域有自己的 chrome。** `"Asia-Pacific"` 在
www.nato.int 乾淨，卻命中 `ndc.nato.int` 56% 的頁面（側欄分類）。子網域要另外看。

**不要用「比 Google 結果數」的比例法當主要判準。** 它在 2026-09-10 給過一次錯誤結論：
分母用了整個網域，而污染只存在於某條路徑上，86% 的污染被稀釋成 10% 的假象。
細節見 `docs/AGENT_BRIEF.md` §10。

已知狀況：

- **EEAS**：`Taiwan`／`China`／`Indo-Pacific` 已確認污染（佔 `/eeas/` 的 78–86%）。
  2026-09-22 換成片語，**2026-09-24 依使用者決定換回單字**
  （現行 `Taiwan;china;Chinese;PRC;Beijing;EU-China;Indo-Pacific;NATO;drone;cables`）。
  `Chinese`／`PRC`／`NATO`／`drone`／`cables` 乾淨。
- **nato.int**：同一個病，而且更嚴重。整站 `<option>` 選單含 `Taiwan`／`China`，導覽列含
  「Relations with partners in the Indo-Pacific region」。2026-09-22 換成片語，
  **2026-09-24 依使用者決定換回**
  （現行 `Taiwan;cross-Strait;China;Chinese;PRC;Beijing;Indo-Pacific;Asia-Pacific;South China Sea;drone;cables`）。
  `NATO` 仍刻意不含（NewsSearch `findings.md` K.2）。`Asia-Pacific` 會帶進 `ndc.nato.int` 側欄的雜訊；
  `cross-Strait` 在 NATO 是零命中片語，但跟裸字同組，就算退回也只是退成本來就在清單裡的字。
  詳見 `docs/AGENT_BRIEF.md` §10。
- **consilium.europa.eu**：**無法驗證**。curl 回 403，連會執行 JS 的自動化瀏覽器也過不去
  （18 秒仍停在 Checking your browser）。2026-09-24 依使用者指定改成跟 EEAS 同一組。
  它的文件 PDF 放在 `data.consilium.europa.eu`，那個子網域沒擋（curl 回 200），但 eu_verify 讀不了 PDF。

### 逃開污染的方法是加長片語，不是換窄的同義字

污染通常是**固定字串**（選單項、連結文字）。任何不是它子字串的片語都乾淨：
裸字 `Taiwan` 中招，`"Taiwan Strait"` 不會；`EU Indo-Pacific Strategy` 中招，
`"the Indo-Pacific"` 不會。

⚠️ **但安全片語逐站不同，不能跨站沿用。** NATO 的導覽列寫的是
「Relations with partners in the Indo-Pacific region」，所以在 EEAS 安全的
`"the Indo-Pacific"` 與 `"Indo-Pacific region"` 在 nato.int **兩個都中招**。
每一站都要拿該站的頁面重驗一次。

**但 Google 會把所有格與標點正規化掉**，所以片語必須含第二個實詞——
`China's` 會被還原成 `China` 又中招，`"EU-China"` 才安全。

`intext:`／`allintext:` 不能用來只搜正文，已實測失效（§10 有數據）。

### 一般情況下把關鍵字換窄仍然是錯的

上面那個 EEAS 的例子是特例：`China` 命中該路徑 86%，它在 Google 那層不提供篩選，
所以 9/22 判斷換掉它可以接受。但那筆交易有代價——正文只寫 China／Taiwan 的文章會被漏掉
（AGENT_BRIEF §10 有記）。有了 eu_verify 之後，使用者在 2026-09-24 把它換回來了（見本節開頭）。

**沒有實測證明失去篩選力之前，不要為了「減少雜訊」把關鍵字換窄。**
依「漏掉比抓錯嚴重」的原則，那是拿確定的召回損失換不確定的清爽。

## 技術約束

| 項目 | 規定 |
|---|---|
| 檔案 | 單一 `index.html`，**不要拆檔** |
| 建置 | 無，且不得引入 |
| 依賴 | 零，不裝 npm、不用框架、不用打包器 |
| 後端 | 無 |
| DOM 寫入 | 一律 `textContent`／`createElement`，**不用 `innerHTML`** |
| 部署 | GitHub Pages，`main` 分支根目錄，push 即上線 |

**未經使用者授權不得 push 或部署。**

## 相關 repo

證據與決策理由在 <https://github.com/liucheweiwill-dev/NewsSearch>
的 `findings.md` §F、§G 與 `progress.md`（2026-09-04 一節）。

**不要把兩個 repo 合併**——一個是 Cloudflare Worker + Brave API，
一個是靜態單頁 + Google UI，沒有共用的執行環境或部署路徑。
