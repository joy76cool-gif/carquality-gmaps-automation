# 車麗屋 Google Maps 自動化

JOMO.HQ 為車麗屋汽車百貨（33 間門市）建置的 Google Maps 評論自動回覆 + 客訴追蹤系統。

## Quick Start

```bash
# 啟動 n8n（地端 Docker）
docker run -d --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n n8nio/n8n

# 開啟 n8n
open http://localhost:5678
```

## Folder Structure

- `docs/` — 提案、報告、spec
- `data/` — 評論數據、門市清單
- `scripts/` — 爬蟲、API 工具
- `n8n-workflows/` — n8n workflow JSON
- `templates/` — 回覆模板、風險規則
