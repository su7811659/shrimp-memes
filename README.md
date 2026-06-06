# shrimp-memes

蝦蝦蝦味仙的 meme image 倉庫。蝦在 LINE 群組偵測到行情情緒(漲噴 / 跌爛 / FOMO / 盤整)時會挑這裡的圖丟出來。

## 結構

```
shrimp-memes/
├── *.{jpg,png}        ← meme 圖
├── memes.json         ← canonical metadata(category / intensity / tags / OCR text)
└── README.md
```

## ID 命名規則

`<category>_<NN>.<ext>` — e.g. `down_01.jpg`、`fomo_05.png`、`up_02.png`

Category:
- **down** — 跌、套牢、絕望(蝦躺鍋、殼裂)
- **up** — 漲噴、爽、慶祝(蝦腳發燙、嗨爆)
- **fomo** — 沒上車、悔恨、追高(蝦跑追火車)
- **flat** — 盤整、無聊、攤平(蝦泡腳)

## memes.json schema

```json
{
  "id": "down_05",
  "category": "down",
  "url": "https://raw.githubusercontent.com/su7811659/shrimp-memes/main/down_05.jpg",
  "description": "黑龍蝦從慶祝突破到開高走低收最低的震驚",
  "intensity": "ouch",
  "tags": ["market_crash", "regret", "shrimp_themed"],
  "text_on_image": [
    "昨天突破盤整，今天果然繼續漲!!!",
    "(開高走低收最低)"
  ]
}
```

### 欄位用途

| 欄位 | 用途 |
|---|---|
| `id` | dedup + reference |
| `category` | meme 分桶 — pickMeme 第一層 filter |
| `url` | LINE imageMessage 來源(GitHub raw) |
| `description` | 給 audit / LLM 看的圖內容描述 |
| `intensity` | `mild` / `medium` / `ouch` — pickMeme 根據當下行情強度挑 |
| `tags` | 場景標籤 — pickMeme 算 overlap score |
| `text_on_image` | 蝦在圖內的台詞(OCR 抓出來) |

## 加新 meme 的工作流

蝦這邊已經寫了 helper script。流程:

### Step 1:把圖檔放到任何路徑

```bash
# 例如下載到 /root/Downloads/lobster_new.jpg
```

### Step 2:跑 add_meme.mjs

```bash
node ~/.openclaw/workspace/scripts/add_meme.mjs \
  --file /root/Downloads/lobster_new.jpg \
  --category down       # down / up / fomo / flat
```

腳本會:
- 自動 git pull 拉最新
- 算下一個 ID(e.g. `down_06`)
- 複製圖到 `/tmp/shrimp-memes/down_06.jpg`
- 加 stub 到 workspace + repo 的 `memes.json`
- 跑 vision LLM 自動寫 description / intensity / tags / text_on_image
- 印出 review summary

### Step 3:Review caption

如果 LLM 寫的 caption / intensity / tags 不對,直接編輯兩個地方(保持同步):
- `~/.openclaw/workspace/data/memes/memes.json`(workspace canonical,蝦讀這個)
- `/tmp/shrimp-memes/memes.json`(repo 副本)

### Step 4:Commit + push

```bash
cd /tmp/shrimp-memes
git add <new_file>.jpg memes.json
git commit -m "add down_06: 蝦看到 -10% 嚇到尿失禁"
git push
```

### Step 5:Done

蝦下次選 meme 時自動會挑到(`pickMeme` 每次都讀最新 `memes.json`,不用重啟 gateway)。

## 蝦怎麼挑這些 meme

`pickMeme(category, groupId, ctx)`:
- ctx.changePct → 推 intensity(\|chg\|≥5 ouch / ≥3 medium / ≥1 mild)
- exact intensity match: +30 分
- 相鄰 intensity: +15 分
- 每個 tag overlap: +10 分
- 隨機 tie-break:+0~5 分
- 14 天內 dedup(同檔不會連發)

所以同一張 meme 不會每天看到、輕跌不會丟「殼裂」級的 meme。

## 重 caption 已有的 meme

如果 LLM 描述要重做(或加新欄位):

```bash
# 全部重抓
node ~/.openclaw/workspace/scripts/caption_memes.mjs

# 只重抓某張
node ~/.openclaw/workspace/scripts/caption_memes.mjs --only down_03
```

輸出到 `memes-proposed.json`,review 後手動覆蓋 canonical。
