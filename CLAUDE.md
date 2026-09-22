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
Brave news 端點 ／ 深分頁加本地篩選 ／ Google Alerts ／ LLM Agent 自動摘要。

**特別注意**：`consilium.europa.eu` 擋的是**自動化指紋，不只是「不執行 JS」**。
curl 三種 UA 皆回 403；**會執行 JS 的自動化瀏覽器等 18 秒仍停在
"Checking your browser"**（2026-09-22 實測）。所以 Worker、Node 腳本、serverless
**以及 Playwright／Puppeteer** 都抓不到它——這擋掉的是一整類提案，不只是 RSS。

## 替任何站設關鍵字之前，先確認那個字還有篩選力

機構網站的全站導覽列與選單會把國名、主題名塞進**每一頁**的 HTML。
若 Google 據此配對，那個關鍵字對該站就等於不存在——而你不會發現，
因為查詢照樣回結果，只是回的是整個網域。

**驗證方法：curl 兩個「正文與該關鍵字無關」的同站頁面，grep 那個字。**
出現了就是在 chrome 裡。離線、即時、不會觸發 Google 的 bot 偵測。

```bash
curl -sS -L -A "Mozilla/5.0 ... Chrome/131.0 ..." -o a.html "<該站任一篇無關文章>"
grep -o -i "<keyword>" a.html | wc -l      # > 0 就是污染
```

**不要用「比 Google 結果數」的比例法。** 它在 2026-09-10 給過一次錯誤結論：
分母用了整個網域，而污染只存在於某條路徑上，86% 的污染被稀釋成 10% 的假象。
細節見 `docs/AGENT_BRIEF.md` §10。

已知狀況：

- **EEAS**：`Taiwan`／`China`／`Indo-Pacific` 已確認污染（佔 `/eeas/` 的 78–86%），
  2026-09-22 已換成片語。`Chinese`／`PRC`／`NATO`／`drone`／`cables` 乾淨。
- **nato.int**：同一個病。整站 `<option>` 選單含 `Taiwan`／`China`，導覽列含
  「Relations with partners in the Indo-Pacific region」，2026-09-22 一併換掉。
  `NATO` 本來就刻意不含（NewsSearch `findings.md` K.2）。
- **consilium.europa.eu**：**無法驗證**。curl 回 403，連會執行 JS 的自動化瀏覽器也過不去
  （18 秒仍停在 Checking your browser）。只能由使用者用真人瀏覽器看原始碼。
  **沒有證據之前不要動它的 `kw`。**

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

上面那個 EEAS 的例子是特例：`China` 命中該路徑 86%，它不是在提供召回，
而是等同於沒有關鍵字，所以換掉它不算損失。

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
