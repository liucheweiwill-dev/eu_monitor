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

**特別注意**：`consilium.europa.eu` 對所有不執行 JS 的客戶端回 403
（curl 三種 UA 皆同）。任何 Worker、Node 腳本或 serverless 後端都抓不到它——
這擋掉的是一整類提案，不只是 RSS。

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
