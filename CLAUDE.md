# 車麗屋 Google Maps 自動化 — Project 指令

## Project 概要
為車麗屋汽車百貨（33 間門市）建置 Google Maps 評論自動回覆 + 客訴追蹤系統。
這是 JOMO.HQ 的第一個實戰案例 / portfolio。

## 客戶資訊
- 公司：車麗屋汽車百貨股份有限公司
- 門市數：33 間
- 決策者：CFO（需正式提案）
- GBP：一個帳號管全部 33 店
- 現況：有人手動回覆 Google Maps 評論，沒有統一系統

## 技術棧
- **自動化**：n8n（地端部署，Docker）
- **AI**：Claude API（回覆生成 + 語意分類）
- **資料庫**：Google Sheet（MVP）→ 之後可升級 DB
- **向量庫**：Qdrant（RAG 用，之後加）
- **通知**：Email（MVP）→ LINE Messaging API（第二階段）
- **提案**：single-file HTML（JOMO.HQ 品牌風格）

## Folder 結構
```
carplus-gmaps-automation/
├── CLAUDE.md           ← 你現在看的這個
├── docs/               ← 提案、報告、spec
├── data/               ← 評論數據、門市清單、分析結果
├── scripts/            ← 爬蟲、資料處理、API 工具
├── n8n-workflows/      ← n8n workflow JSON export
└── templates/          ← 回覆模板、風險規則表、報表模板
```

## 工作原則
- 提案用 single-file HTML，不依賴外部 CSS/JS
- JOMO.HQ 品牌色系：深墨綠 `#1B4332` + 橘 `#E76F51`
- 字體：DM Serif Display（標題）+ Noto Sans TC（內文）
- 不在提案寫技術細節，客戶只看結果
- 公開回覆不提補償/禮品細節（Google 禁止 incentivized reviews）
- 個資一律遮罩（電話、Email、姓名、車牌）

## 收費模式
- 建置費 = 年省金額 × 50%（一次性）
- 顧問費 + 提案費 = 免收（賣點）
- 知識移轉/教育訓練 = 另計
- 後續維護 = 按月另計

## 人力成本基準
- 行政人員月薪 $33,000 × 1.5（勞健保年終）= $49,500/月
- 時薪 $281/hr、每分鐘 $4.7

## Plan 位置
完整 plan 在 `~/.claude/plans/scalable-inventing-harbor.md`

## 相關 Skills
- `/生成提案` — 客戶提案 HTML 頁面
- `/carousel` — 案例包裝用 IG Carousel
