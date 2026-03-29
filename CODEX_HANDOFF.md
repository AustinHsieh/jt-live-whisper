# Codex Handoff

更新日期：2026-03-29

## 目的

這份文件是給下次接手這個 repo 的人或 Codex 用的，避免重新分析一次專案結構、音訊路徑和這次已確認過的 bug / 修正。

## 專案速覽

- 入口是 [start.sh](/Users/austin.hsieh/temp/jt-live-whisper/start.sh)，主要工作都在 [translate_meeting.py](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py)。
- 即時模式有三條主要路徑：
  - 本機 Whisper `run_stream()`：[translate_meeting.py:2696](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L2696)
  - 本機 Moonshine `run_stream_moonshine()`：[translate_meeting.py:3093](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L3093)
  - 遠端 Whisper 即時串流：從 [translate_meeting.py:3480](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L3480) 附近開始
- 離線轉錄 / 摘要 / 講者辨識也都集中在 [translate_meeting.py](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py)。
- 執行輸出預設放在：
  - `logs/`
  - `recordings/`
- 本機安裝產物：
  - `venv/`
  - `whisper.cpp/`

## 已確認的音訊行為

### 1. 即時字幕和錄音內容不一致，不是偶發 bug

原因是「即時 ASR 音源」和「錄音音源」原本就是分開的：

- 即時 ASR 預設優先抓 `BlackHole`，所以只看系統播放音訊
  - SDL2 / Whisper 路徑：[translate_meeting.py:763](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L763)
  - sounddevice / Moonshine / 遠端路徑：[translate_meeting.py:964](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L964)
  - CLI 自動選裝置：[translate_meeting.py:7095](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7095)
- 錄音可以另外選 `Aggregate Device`，因此能同時錄到播放音訊和麥克風
  - 互動式錄音選單：[translate_meeting.py:4577](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L4577)
  - 錄音裝置自動偵測：[translate_meeting.py:4078](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L4078)

結論：

- 即時字幕 / 即時 log：預設只含對方或系統播放聲音
- 錄音檔：可能含對方 + 自己麥克風
- 拿錄音檔做離線轉錄時，會看到兩者都被轉出來

### 2. 即時 log 不是錄音檔的事後回放

即時模式產出的 `logs/中文_逐字稿_*.txt` 是即時 ASR 當下直接寫出的內容，不是根據錄音檔重跑得到的結果。這也是使用者最初發現「文字檔和 MP3 不一致」的原因之一。

## 這次做的修改

### 1. 新增即時 ASR 音源選項

目的：讓即時字幕可選擇只看播放聲音，或同時看播放聲音 + 麥克風。

新增內容：

- 共用常數與顯示名稱：
  - `REALTIME_ASR_SOURCES` / `_asr_source_label()`：[translate_meeting.py:88](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L88)
- 裝置偵測與匹配：
  - `_detect_input_devices_sd()`：[translate_meeting.py:716](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L716)
  - `_match_sdl_device_name()`：[translate_meeting.py:745](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L745)
  - `list_audio_devices()`：[translate_meeting.py:763](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L763)
  - `list_audio_devices_sd()`：[translate_meeting.py:964](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L964)
  - `auto_select_device_sd()`：[translate_meeting.py:1022](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L1022)
  - `auto_select_device()`：[translate_meeting.py:7095](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7095)
- 互動式選單：
  - `_ask_realtime_asr_source()`：[translate_meeting.py:4343](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L4343)
- CLI 參數：
  - `--asr-source {playback,mixed}`：[translate_meeting.py:7050](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7050)
  - CLI 指令組裝 `_build_cli_command()`：[translate_meeting.py:7195](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7195)

目前行為：

- `playback`：優先抓 `BlackHole`
- `mixed`：優先抓 `Aggregate Device`

注意：

- `Whisper 本機即時 + mixed` 會先用 sounddevice 找到聚集裝置，再嘗試匹配 `whisper-stream` 的 SDL2 裝置名稱。
- 如果 sounddevice 找得到，但 SDL2 列表找不到同名裝置，程式會要求用 `--device` 手動指定 SDL2 裝置 ID。

### 2. 本機即時 Whisper 模型自動降級

背景：使用者執行 `./start.sh --mode zh --local-asr --asr-source mixed` 時，缺少 `ggml-large-v3.bin`，但機器上有 `ggml-large-v3-turbo.bin`。

修正：

- 新增 `_pick_local_realtime_whisper_model()`：[translate_meeting.py:7152](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7152)
- 中文即時模式優先 `large-v3`，缺檔時自動降級到 `large-v3-turbo`
- 英文即時模式則偏向 `large-v3-turbo` / `*.en` 模型
- 補上 `C_WARN = C_HIGHLIGHT`，避免 warning 訊息印出時 `NameError`

### 3. 修正靜音時重複輸出最後一段 chunk

使用者回報：沒人說話時，即時字幕會重複最後一段 chunk。

原因：

- 本機 Whisper 和 Moonshine 原本只做「和上一句完全相等」的去重
- 靜音時後端可能重送最後一句，或只改了空白 / 標點，會被當成新字幕

修正：

- 新增 `_normalize_live_text()`：[translate_meeting.py:137](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L137)
- 新增 `_RecentSubtitleDeduper`：[translate_meeting.py:144](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L144)
- 套用到三條即時路徑：
  - 本機 Whisper：[translate_meeting.py:2943](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L2943)
  - Moonshine：[translate_meeting.py:3197](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L3197)
  - 遠端 Whisper：[translate_meeting.py:3717](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L3717)

去重規則：

- 先正規化文字，忽略 ANSI、空白、一般標點差異
- 只在短時間窗內去重
- 長句才做包含關係比對，避免短詞誤傷

### 4. 新增 `.gitignore`

之前 repo 沒有 `.gitignore`。

已新增 [.gitignore](/Users/austin.hsieh/temp/jt-live-whisper/.gitignore)，忽略：

- `__pycache__/`
- `logs/`
- `recordings/`
- `venv/`
- `whisper.cpp/`
- `config.json`
- `.whisper_output.txt`
- `.DS_Store`

## 這次實際遇到的問題與結論

### 問題 1. 即時逐字稿與音檔不符

已確認原因不是轉錄模型壞掉，而是：

- 即時 log 走的是當下 ASR 音源
- 音檔可能是另一個錄音裝置錄下來的混合來源

### 問題 2. `--local-asr` 預設模型會因缺少 `large-v3` 而中止

已修正成自動降級，不需要每次手動補 `-m large-v3-turbo`。

### 問題 3. 靜音時重複最後一段字幕

已修正成「近期字幕去重」，使用者已回報這版測試 OK。

## 建議下次優先看的位置

如果下次要改即時模式，先看這幾段：

1. 裝置選擇
   - [translate_meeting.py:716](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L716)
   - [translate_meeting.py:763](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L763)
   - [translate_meeting.py:964](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L964)
   - [translate_meeting.py:7095](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7095)
2. 互動式選單
   - [translate_meeting.py:4343](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L4343)
   - [translate_meeting.py:4577](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L4577)
3. 即時辨識執行
   - [translate_meeting.py:2696](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L2696)
   - [translate_meeting.py:3093](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L3093)
   - [translate_meeting.py:3480](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L3480)
4. 去重邏輯
   - [translate_meeting.py:137](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L137)
   - [translate_meeting.py:144](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L144)
5. CLI 參數與預設
   - [translate_meeting.py:6995](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L6995)
   - [translate_meeting.py:7152](/Users/austin.hsieh/temp/jt-live-whisper/translate_meeting.py#L7152)

## 已驗證項目

- `./venv/bin/python -m py_compile translate_meeting.py`
- `./venv/bin/python translate_meeting.py --help`
- 用小測試確認 `_RecentSubtitleDeduper` 會把 `測試一下` 與 `測試 一下。` 視為同一句
- 使用者人工測試回報：重複 chunk 問題已解

## 尚未在這個 Codex 環境完成的事

這個 Codex sandbox 讀不到本機 macOS 音訊裝置，所以沒有在這裡完成「實機音源驗證」。先前檢查時：

- `sounddevice` 看不到任何 input device
- `system_profiler SPAudioDataType` 沒列出可用裝置
- `ffmpeg -f avfoundation -list_devices true -i ""` 也沒有列出音訊裝置

因此：

- 程式邏輯已接好
- 實際 `BlackHole` / `Aggregate Device` 是否抓對，仍需在使用者自己的終端機上驗證

## 常用指令

```bash
# 即時中文轉錄，只看播放聲音
./start.sh --mode zh --local-asr

# 即時中文轉錄，播放聲音 + 麥克風
./start.sh --mode zh --local-asr --asr-source mixed

# 即時 Moonshine，播放聲音 + 麥克風
./start.sh --mode zh --asr moonshine --asr-source mixed

# 離線轉錄錄音檔
./start.sh --input recordings/錄音_xxx.mp3 --mode zh --local-asr

# 語法檢查
./venv/bin/python -m py_compile translate_meeting.py
```

## 備註

- [install.sh](/Users/austin.hsieh/temp/jt-live-whisper/install.sh) 和 [start.sh](/Users/austin.hsieh/temp/jt-live-whisper/start.sh) 目前在 Git 狀態裡的差異是檔案權限，不是內容變更。
