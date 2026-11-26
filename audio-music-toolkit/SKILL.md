---
name: audio-music-toolkit
description: 音效與配樂工作流程：搜尋免費音效（Openverse）、用 AI 生成音效或配樂（MMAudio）、用 ffmpeg 剪輯／混音／調音量／淡入淡出，並把音訊配到影片上。只要使用者提到音效、配樂、背景音樂、BGM、SFX、foley、替影片加聲音、音訊剪輯、混音、降噪、音量平衡、淡入淡出、生成音效，或想讓影片「有聲音」，即使沒有明說「音效」也要使用這個技能。
---

# 音效與配樂工作流程 (Audio & Music Toolkit)

本技能負責影片之外的「聲音」部分：找音效、生成音效／配樂、剪輯與混音。
影片本身的剪輯、字幕、片頭請搭配 `video-edit-toolchain` 技能。

回覆語言：使用者以繁體中文提問時，一律用繁體中文回覆。

## 工作原則

1. **先問清楚需求**（若使用者沒說）：用途（短影音／Vlog／遊戲／簡報）、情緒風格、長度、是否商用。
2. **優先順序**：先找現成免費音效 → 找不到再用 AI 生成 → 最後用 ffmpeg 處理與混音。
3. **授權要確認**：商用專案只能用 CC0 / CC BY（需標註）的素材，並記錄來源與作者。
4. **不覆蓋原檔**：所有處理輸出到新檔案，檔名加上 `_edit`、`_mix` 等後綴。

## 步驟一：搜尋現成音效（Openverse）

Openverse（openverse.org）收錄大量可自由使用的 CC 授權音訊。

網頁搜尋：`https://openverse.org/search/audio?q=關鍵字`（關鍵字建議用英文，例如 `door slam`、`rain`、`whoosh`）。

API 搜尋範例：

```bash
curl "https://api.openverse.org/v1/audio/?q=whoosh&license_type=commercial&page_size=10"
```

- `license_type=commercial` 只回傳可商用的素材。
- 結果中記下 `title`、`creator`、`license`、`url`，作為來源紀錄。
- 下載後先試聽，確認長度與品質再使用。

來源紀錄格式（存成 `credits.md`）：

```
- 音效名稱 | 作者 | 授權 | 連結
```

## 步驟二：AI 生成音效或配樂（MMAudio）

MMAudio 可依「影片內容」或「文字描述」生成同步的音效。適合找不到合適素材、
或需要與畫面動作精準對齊的情況。

使用方式（本機或雲端需要 GPU 環境，先確認使用者有無環境）：

- **Text-to-Audio**：提供英文描述，例如 `footsteps on wooden floor, slow pace`。
- **Video-to-Audio**：提供影片片段加上簡短提示詞，生成與畫面同步的聲音。

撰寫提示詞的要點：

- 具體描述聲音來源、材質、環境、節奏（例如 `heavy rain on a tin roof, distant thunder`）。
- 一次只生成一種聲音，之後再混音，後製比較好控制。
- 生成長度以 8–10 秒為單位，較長的音訊用 ffmpeg 循環或拼接。

## 步驟三：Earshot

Earshot 為使用者指定要整合的音訊工具。使用前先確認其安裝方式與支援功能，
再把確認後的指令補在這一節。

> TODO（由使用者補充）：Earshot 的安裝指令、主要用途、範例指令。

## 步驟四：用 ffmpeg 剪輯與混音

常用指令速查（`in` 為輸入檔，請依實際檔名替換）：

**剪出片段**

```bash
ffmpeg -i in.mp3 -ss 00:00:05 -to 00:00:12 -c copy cut.mp3
```

**調整音量**（`0.5` 為一半音量，也可用 `-6dB`）

```bash
ffmpeg -i in.mp3 -filter:a "volume=0.5" quiet.mp3
```

**淡入淡出**（淡入 2 秒，從第 28 秒開始淡出 2 秒）

```bash
ffmpeg -i in.mp3 -af "afade=t=in:st=0:d=2,afade=t=out:st=28:d=2" fade.mp3
```

**音量標準化**（讓不同素材響度一致）

```bash
ffmpeg -i in.mp3 -af loudnorm=I=-16:TP=-1.5:LRA=11 norm.mp3
```

**循環背景音樂到指定長度**（例如 60 秒）

```bash
ffmpeg -stream_loop -1 -i bgm.mp3 -t 60 bgm_60s.mp3
```

**把多個音軌混在一起**（旁白 + 背景音樂，BGM 壓低）

```bash
ffmpeg -i voice.wav -i bgm.mp3 \
  -filter_complex "[1:a]volume=0.25[bgm];[0:a][bgm]amix=inputs=2:duration=first:dropout_transition=0" \
  mix.mp3
```

**在指定時間點加入音效**（第 3.5 秒加入音效）

```bash
ffmpeg -i video.mp4 -i sfx.wav \
  -filter_complex "[1:a]adelay=3500|3500[s];[0:a][s]amix=inputs=2:duration=first" \
  -c:v copy out.mp4
```

**把音訊配到影片上**（替換原聲音）

```bash
ffmpeg -i video.mp4 -i music.mp3 -map 0:v -map 1:a -c:v copy -shortest out.mp4
```

**簡易降噪**

```bash
ffmpeg -i in.wav -af "highpass=f=80,afftdn=nf=-25" clean.wav
```

## 混音建議值

| 元素 | 建議響度／音量 |
|------|------|
| 旁白／人聲 | 最大聲，約 -16 LUFS |
| 背景音樂 | 比人聲低 12–18 dB（約 volume 0.15–0.25） |
| 音效 | 比人聲略低或相當，依畫面重點調整 |
| 最終輸出 | 社群平台約 -14 ~ -16 LUFS，峰值不超過 -1 dBTP |

人聲出現時，可讓 BGM 自動壓低（ducking）：

```bash
ffmpeg -i bgm.mp3 -i voice.wav \
  -filter_complex "[0:a][1:a]sidechaincompress=threshold=0.05:ratio=8[duck];[duck][1:a]amix=inputs=2" \
  ducked.mp3
```

## 輸出與交付

1. 輸出格式：音訊用 `.wav`（編輯中）或 `.mp3`（成品）；影片保持原本編碼，只換音軌。
2. 交付時附上：成品檔案、使用的素材清單（`credits.md`）、使用過的 ffmpeg 指令。
3. 若素材需標註作者（CC BY），把標註文字整理好，方便貼在影片說明欄。

## 常見問題

- **聲音與畫面不同步**：檢查取樣率是否一致（建議統一 48000 Hz），或用 `adelay` 微調。
- **音量忽大忽小**：先對每個素材做 `loudnorm`，再混音。
- **爆音／削波**：降低各軌音量，最後輸出前確認峰值低於 -1 dBTP。
- **找不到合適音效**：換英文同義詞搜尋，或改用 MMAudio 生成。
