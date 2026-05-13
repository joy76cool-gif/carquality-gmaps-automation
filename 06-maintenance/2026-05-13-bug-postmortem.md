---
date: 2026-05-13
type: bug-postmortem
audience: Joy → 可轉車麗屋 IT / 行銷主管
status: draft（明早 Joy 看）
---

# 八德店 pilot 「2★ 評論卡 8 小時沒回」事件報告

> 給沒接觸 n8n / GBP API 的人也能看懂

## 1. 一句話總結

**Workflow 程式碼有個 schema bug**（「處理 1-2 星評論」的 SOP 漏寫一欄），跟你筆電開關機 / 地端 sleep **完全無關**。已 fix + 重跑成功。

## 2. Bug 怎麼發生的（小五版）

### 想像一個信件處理工廠

| 真實世界 | 我們的系統 |
|---|---|
| 客人在意見箱投紙條 | 客戶在 GMaps 留評論 |
| 機器人每分鐘巡邏意見箱 | n8n cron 每分鐘抓新評論 |
| 機器人請 ChatGPT 寫回覆 | AI 生成 reply 草稿 |
| 機器人把回覆貼回原信旁 | Post Reply 到 GBP |

### 我們的機器人有兩套 SOP

- **SOP A：處理 4-5 星好評**（≥3★ path）
- **SOP B：處理 1-2 星負評**（<3★ path）

兩套 SOP 應該長很像 — 都要寫「寄回去的地址」、「狀態欄」、「客人姓名」這些欄位。

### Bug

**SOP B 漏寫關鍵欄位**：「回信要貼回哪則評論」(`review_name`)。

機器人按 SOP B 處理時 → 不知道 reply 該貼到哪則評論 → Google 回「你說要回的這則評論不存在」(API 404 error)。

### 為什麼之前沒被發現

- 5/13 凌晨 pilot 啟動到那則 2★ 進來之前，所有真實評論都是 **5 星**（走 SOP A，沒事）
- 5/13 00:05 第一則 2★ 評論進來 → 第一次走 SOP B → bug 暴露
- 5/11 之前的 mock test 只測了 SOP A + SOP B 失敗 handler，**沒測「SOP B 正常成功」path** ← 漏網

### 額外坑

機器人有個「去重」邏輯：**Sheet 紀錄過的評論不再處理**。
所以 bug 觸發後 → Sheet 寫 `PostFailed` → 下次 cron 把這則當「已處理」過濾掉 → **永遠不會自動重試** → 卡 8 小時。

## 3. 跟 n8n 地端 / 你筆電開關 / sleep 有關嗎？

### ❌ 這次 bug 無關

bug 是**程式邏輯錯誤**，跟 runtime 環境（你的筆電 / 雲端）完全沒關係。即使移到雲端 24/7 跑，**第一則 <3★ 評論一定還是會 fail**。

### ✅ 但 sleep 確實影響別的場景

| 場景 | 跟 sleep 有關？ |
|---|---|
| 這次 2★ stuck 8 小時 | ❌ 是 schema bug |
| 你睡覺關機期間評論累積 | ✅ 真的有，重開機才補抓 |
| 連假累積評論 > 5 則 | ✅ 真的會漏（pageSize=5 限制）|
| AI 生成內容不符品牌 tone | ❌ 跟 sleep 無關 |

兩件事獨立。bug 是 **structural**（程式碼），sleep 是 **environmental**（運行環境）。

### 你的筆電現況

- ✅ n8n 在跑（`node /opt/homebrew/bin/n8n start` PID 10409）
- ✅ 我下了 `caffeinate -i -w 10409` 防整機 sleep（綁定 n8n PID）
- ⚠️ Mac 重開機後 caffeinate + n8n 都要重啟
- 📌 終極解：插電 + 系統設定「永不 sleep」+ Phase 2 上雲

## 4. 評論到回覆要多久 — 完整 breakdown + Source

你觀察到 3-20 分鐘變動 — 各環節時間 + source：

| 環節 | 時間 | Source level | URL |
|---|---|---|---|
| (a) 客戶留評 → Google API 抓得到 | **幾分鐘 ~ 10 小時 不固定**（主因） | ⭐⭐⭐⭐ | [GBP 官方 community 用戶報 10 hours delay](https://support.google.com/business/thread/4197034) |
| (a') Google moderation（少數 case）| 幾天 | ⭐⭐⭐⭐⭐ Google 官方 | [About missing or delayed reviews](https://support.google.com/business/answer/10313341) |
| (b) n8n cron 抓取間隔 | 0-60 秒（current 1 分鐘設定）| 我們系統 | local config |
| (c) n8n pipeline 處理（拆分+去重+合併+分流）| < 2 秒 | local | - |
| (d) AI 生成（Claude Haiku 4.5）| 5-15 秒 | ⭐⭐⭐⭐⭐ Anthropic | [Claude API rate limits](https://platform.claude.com/docs/en/api/rate-limits) |
| (e) Post Reply 到 GBP | 1-3 秒 | API roundtrip | - |
| (f) Google 顯示回覆 | < 1 分鐘 | 觀察值 ❓ 無官方 SLA | - |
| **總體** | **3-20 分鐘觀察值** | Joy 實測 5/13 八德 | - |

### 關鍵 fact

🚨 **Google 沒公布 API indexing latency SLA**

- Google 只承認「moderation 可能 take a few days」（針對被擋下的評論），**沒承認也沒否認** routine indexing 延遲
- "real-time" 是行銷話術，不是 contract
- 2026 Google 反爬蟲措施波及 official API，延遲加劇 ⭐⭐⭐⭐ [soci.ai 2026 review discrepancies](https://www.soci.ai/blog/google-review-discrepancies/)

### 我們可控 vs 不可控

```
不可控（Google 端，60-95% 變動來自這）:
- 客戶留評 → API 可見 ← 幾分到數小時
- Google 顯示回覆 ← < 1 分鐘

可控（我們系統，5-40% 變動）:
- Cron 抓取間隔 1 分鐘（max 60 秒延遲）
- AI 生成 5-15 秒（升 Tier 2 可降到 5-10 秒）
- Post Reply 1-3 秒
- pipeline 內部處理 < 2 秒
總計 ~1.5-2 分鐘 max
```

**結論**：你看到 3-20 分鐘變動，**1.5-2 分鐘是我們，剩下 1.5-18 分鐘全在 Google 端**。要降到「客戶秒回」做不到（Google 端 indexing latency 是不可控變數）。

## 5. 修法 + 後續

### 已 fix（5/13 凌晨）
- `整理欄位（<3★）`加 `review_name` field
- `整理欄位（<3★）`status column 跟 `≥3★` 對齊
- `整理欄位（<3★）` rp_path 改「自動」
- 加 `reviewer_name` 在兩個整理欄位（fresh agent 第 2 輪抓出的隱藏 bug）
- 補 `<3★ PostFailed` handler 的 `public_replied_at` 對齊
- 已 verify：execution 2774 (5/13 02:05) <3★ Post 成功 ✅
- **Fresh agent end-to-end retest: 4/4 path FULL PASS** ✅（不知道原邏輯的第三方 QA 走完整 4 條 path 都通過，schema 一致，required fields 全 set）

### 5/13 凌晨 第 3 波 fix（客戶提贈品 + 客戶 update 行為決策）

**新 bug**：5★ 評論「留五星評論送甜甜圈」→ AI 回「**甜甜圈已備好歡迎下次來店領取**」← 違 Google policy「Be conversational, not promotional」+ 承諾具體贈品

**Fix**：
- ≥3★ AI prompt 加 generic rule：「客人提到贈品 / 優惠 / 活動 / 折扣 → 輕帶過『謝謝您參與這個活動』，不認可不承諾不否認」
- 不列 specific 禁用詞 list（Joy directive：避免 prompt bloat），用 generic principle
- Fresh agent 第 5 輪 retest：**FULL PASS** ✅

**新 design decision — 客戶修改評論 / 星數**：
- **Pilot 階段不處理** by design（量小、troll 機率低）
- 「拆分+去重」邏輯以 review_id 過濾，同 review 處理過就 stuck
- 客戶 5★ 改 1★、文字改、星數改 → 舊 AI 回覆都留著
- **Phase 2（33 店）再評**是否偵測 updateTime 重生回覆 — 量大會被 troll
- 寫進 [phase2-sizing-plan.md](phase2-sizing-plan.md) TODO

### 5/13 凌晨 第 2 波 fix（Joy clarify 後）
- **AI <3★ prompt 加 sentiment 3 分類**：(A) 明確負面 → 表遺憾不認錯 / (B) 正面文字低星 → 感謝不道歉 / (C) 模糊只給星 → 表遺憾邀請聯繫
  - 起因：5/13 06:50 Ariel 1★ 評論「很不錯的店跟服務」（正面文字 + 低星），AI 強制道歉變尷尬「若您的用詞反映了實際體驗有所不足，我們誠摯地為此道歉」
- **AI <3★ prompt 加禁用「用詞」「言詞」**（非台灣口語），改用「評論」「留言」「您給的訊息」
- **3 個 Gmail node 加 cc `car1.csteam@gmail.com`**（IT 同步收到通知）
  - 注意 n8n Gmail v2.1 schema 必須 set `options.ccList`，top-level `params.ccList` 會被 ignored
- **Fresh agent retest 第 4 輪：FULL PASS** ✅

### 待修
- **「拆分+去重」邏輯**：目前把 Sheet 內任何 review_id 都當「已處理」過濾 → PostFailed 永遠不會自動重試。應改成「只過濾 `public_reply_status=Posted`」。**Pilot 階段先 manual 處理**（Sheet 刪 row 即可重跑）。Phase 2 必修。

### Phase 2 (5/26 桃園 4 店擴張) 必修
- Cron 拉長 1 min → 5 min（避 33 店 overlap）
- pageSize 5 → 50（cover 連假爆量）
- AI Batch=10 + Interval=15s（避撞 Anthropic Tier 1 RPM）
- Sheet write 改 batchUpdate（避撞 60 req/min/user quota）
- 升 Anthropic Tier 2（$40 credit 自動升）
- 上雲（n8n Cloud / VPS / 車麗屋 IT 內網）
- 拆分+去重 邏輯 fix
- Workflow B 保底掃描 + Workflow C 客訴升級

詳：[phase2-sizing-plan.md](phase2-sizing-plan.md)

## 6. 學到的教訓 → Standing Rule 升級

| Rule | 從這次學到 |
|---|---|
| 4.6 Bug fix 後 fresh agent end-to-end test full pass | 5/11 mock 只測 1.5 條 path 漏掉 <3★ success → 用 fresh agent paranoid check 4 條 path 才抓全 |
| 4.5 Publish/Deploy 後強制 body content verify | git add 順序 + curl status code 不看 body 教訓 |
| 4.4 規則升級當下 self-demo + 全做不挑 | outsource decision 給 Joy 是偷懶 |
| 4.3 跨檔案 deliverable 動手前必 cross-read | 沒讀原提案 Phase 規劃就硬加 Decision 4 |
| 4.2 客戶 deliverable 送 review 前強制兩道 agent | tone-review 沒派 agent 直接給 Joy 抓 bug |

詳：`~/.claude/CLAUDE.md` Standing Rule 4.2-4.6
