# 本機建置與設定（台灣華語 / Breeze-ASR-25）

完全本機的語音輸入：Whisper.cpp + [Breeze-ASR-25](https://huggingface.co/MediaTek-Research/Breeze-ASR-25)（台灣華語、中英夾雜）+ 內建 OpenCC `s2twp` 台灣用語轉換。不需要雲端、不需要 LM Studio。

實測環境：macOS 27、Xcode 27、Apple Silicon（24 GB RAM）。

## 1. 準備 Xcode

```bash
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -license accept
sudo xcodebuild -runFirstLaunch
brew install cmake
```

## 2. 建置 whisper.xcframework（只編 macOS）

`make setup` 會同時編 iOS / tvOS / visionOS；沒裝那些 SDK 會失敗（`iphonesimulator is not an iOS SDK`）。改用只編 macOS 的腳本：

```bash
mkdir -p ~/VoiceInk-Dependencies
git clone https://github.com/ggerganov/whisper.cpp.git ~/VoiceInk-Dependencies/whisper.cpp
cd ~/VoiceInk-Dependencies/whisper.cpp
bash /path/to/VoiceTwInk/scripts/build-macos-xcframework.sh
```

產出 `~/VoiceInk-Dependencies/whisper.cpp/build-apple/whisper.xcframework`，之後 `make` 會自動跳過這步。

## 3. 固定簽署（權限不會每次重編都要重給）

1. Xcode → Settings → Accounts → 用 Apple ID 登入（免費帳號即可）→ Personal Team → Manage Certificates → ＋ Apple Development。
2. 確認有效：

   ```bash
   security find-identity -v -p codesigning
   ```

   若顯示 `0 valid identities found`，但不加 `-v` 看得到憑證，代表缺 **WWDR G3 中繼憑證**（系統裡只有 2023 年過期的舊版）：

   ```bash
   curl -fsSLo /tmp/AppleWWDRCAG3.cer https://www.apple.com/certificateauthority/AppleWWDRCAG3.cer
   security add-certificates -k ~/Library/Keychains/login.keychain-db /tmp/AppleWWDRCAG3.cer
   ```

3. 把憑證名稱括號內的 ID 寫進 `.local-team`（已 gitignore），然後建置安裝：

   ```bash
   echo XXXXXXXXXX > .local-team
   make local      # 建置 + 簽署 + 安裝到 /Applications/VoiceInk.app
   make dev        # 之後改碼用：建置 + 重啟
   ```

## 4. 下載 Breeze-ASR-25 模型

```bash
mkdir -p models
curl -L -o models/ggml-breeze-asr-25-q5_k.bin \
  https://huggingface.co/alan314159/Breeze-ASR-25-whispercpp/resolve/main/ggml-model-q5_k.bin
```

約 1.1 GB（`models/` 已 gitignore）。這是社群轉檔；要自己轉可從官方權重用 whisper.cpp 的 `models/convert-h5-to-ggml.py`。

## 5. App 內設定

1. **Permissions**：麥克風、輔助使用都開。
2. **AI Models** → Import Local Model… → 選 `models/ggml-breeze-asr-25-q5_k.bin` → Set as Default；語言選 Chinese。
3. **Settings**：確認繁體中文轉換（OpenCC s2twp）開啟（預設開）。
4. **Enhancement**：保持關閉即完全本機。顯示的「Gemini」只是未設定時的預設供應商，沒有 API key 不會送出任何資料。
5. 預設錄音快捷鍵：按一下**右 ⌘** 開始／停止。

## 實測

| 原句 | Breeze-ASR-25 輸出 |
|---|---|
| 這個軟體的介面設計得不錯，但是影片載入速度有點慢。 | 這個軟體的介面設計的不錯但是影片載入速度有點慢 |
| 我們下週要跟客戶 demo 新的 feature，記得先 deploy 到 staging。 | 我們下週要跟客戶 demo 新的 feature記得先 deploy 到 staging |

台灣用語、中英夾雜正確；不含標點（需要的話再開本機 LLM 潤稿）。
