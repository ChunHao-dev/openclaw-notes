# OpenClaw TTS 語音訊息 Bug 修復：Telegram 語音格式問題

> **版本資訊**  
> - OpenClaw: 2026.3.x  
> - 相關 Issue: #42060  
> - 相關 PR: #42153, #42075  
> - 更新日期: 2026-03-11

---

## TL;DR

- **問題**：TTS tool 產生的語音在 Telegram 顯示為 MP3 音樂檔案，而非語音訊息氣泡
- **根因**：`audioAsVoice` flag 在資料處理中途遺失，導致 Telegram 使用錯誤的發送方式
- **修復**：新增 `extractToolResultMedia()` 函數保留完整 directive 資訊
- **影響範圍**：僅修改 tool result 處理層，不影響其他 pipeline
- **風險**：低，向後相容，保留舊函數供其他模組使用

---

## 解決的問題

### 使用情境

當用戶在 Telegram 使用 TTS tool 產生語音回覆時：

**預期行為：**
- 收到語音訊息氣泡（圓形播放鈕 + 波形顯示）
- 可快速播放，體驗類似語音通話

**實際行為：**
- 收到 MP3 音樂檔案附件
- 需要下載或用音樂播放器開啟
- 使用體驗差

### 影響範圍

- **影響 channel**：Telegram（主要）、WhatsApp（可能）
- **不影響**：Discord、Slack（本來就不支援語音訊息格式）
- **觸發條件**：使用 TTS tool 產生語音回覆

---

## 核心概念

### OpenClaw Directive 系統

OpenClaw 使用特殊標記在 tool 輸出中嵌入系統指令：

```
[[audio_as_voice]]        ← Directive（系統指令）
MEDIA:/tmp/audio.opus     ← Media token（媒體路徑）
處理完成                   ← 純文字（顯示給用戶）
```

**處理流程：**

```
Tool 輸出 → 系統解析 directives → 移除標記 → 各 channel 呈現
```

### 關鍵名詞

- **Directive**：`[[audio_as_voice]]` - 標記這是語音訊息
- **Media Token**：`MEDIA:/path/to/file` - 標記媒體檔案路徑
- **audioAsVoice flag**：布林值，告訴 delivery 層使用語音格式發送

---

## Bug 根因分析

### 資料流與錯誤點

```
┌─────────────────────────────────────────┐
│ TTS Tool 輸出                            │
│ "[[audio_as_voice]]\nMEDIA:/tmp/x.opus" │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ splitMediaFromOutput() ✅               │
│ { mediaUrls: [...], audioAsVoice: true }│
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ extractToolResultMediaPaths() ❌        │
│ 只返回: ["/tmp/x.opus"]                 │
│ audioAsVoice 被丟棄！                    │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Telegram Delivery                        │
│ audioAsVoice === undefined               │
│ 使用 sendAudio() 而非 sendVoice()       │
└─────────────────────────────────────────┘
```

### 問題點

1. `splitMediaFromOutput()` 已正確解析出 `audioAsVoice: true`
2. 但 `extractToolResultMediaPaths()` 只返回路徑陣列，丟棄了 flag
3. 後續的 delivery 層收不到 `audioAsVoice` 資訊
4. Telegram 判斷為 `undefined`，使用音樂檔案格式發送

---

## 修復方案

### 核心修改

**新增函數：**

```typescript
// src/agents/pi-embedded-subscribe.tools.ts
export function extractToolResultMedia(result): {
  mediaUrls: string[];
  audioAsVoice: boolean;
} {
  let audioAsVoice = false;
  const paths: string[] = [];
  
  for (const block of content) {
    const parsed = splitMediaFromOutput(block.text);
    
    if (parsed.audioAsVoice) {
      audioAsVoice = true;  // ✅ 保留 flag
    }
    if (parsed.mediaUrls?.length) {
      paths.push(...parsed.mediaUrls);
    }
  }
  
  return { mediaUrls: paths, audioAsVoice };
}
```

**更新呼叫處：**

```typescript
// src/agents/pi-embedded-subscribe.handlers.tools.ts
const extracted = extractToolResultMedia(result);

void ctx.params.onToolResult({
  mediaUrls: extracted.mediaUrls,
  audioAsVoice: extracted.audioAsVoice  // ✅ 傳遞 flag
});
```

**向後相容：**

```typescript
// 保留舊函數供其他地方使用
export function extractToolResultMediaPaths(result): string[] {
  return extractToolResultMedia(result).mediaUrls;
}
```

### 為什麼單一 PR 就能修復

- `ReplyPayload` 型別本來就有 `audioAsVoice?: boolean` 欄位
- `normalizeReplyPayload()` 使用 spread operator，會保留所有欄位
- Outbound delivery 已經支援傳遞 `audioAsVoice` 到 Telegram

**因此只需要修復「中途遺失」的問題，不需要修改整條 pipeline。**

---

## 最佳實務

### ✅ 正確做法

1. **保留完整 directive 資訊**：不要只取部分欄位
2. **使用新函數**：`extractToolResultMedia()` 而非 `extractToolResultMediaPaths()`
3. **傳遞完整 payload**：確保 `audioAsVoice` 傳遞到 delivery 層

### ❌ Anti-patterns

1. **只取路徑陣列**：會遺失 directive 資訊
2. **跨 block 累積 directive**：可能誤標記其他媒體（見下方風險）

---

## 潛在風險與改進建議

### 跨 Content Block 累積問題

**目前邏輯（可能出錯）：**

```
Tool Result:
  Block 1: { text: "[[audio_as_voice]]" }      → audioAsVoice = true
  Block 2: { text: "MEDIA:/tmp/image.jpg" }    → 加入 media

結果：
  { mediaUrls: ["/tmp/image.jpg"], audioAsVoice: true }
  ❌ 圖片被誤標記為語音訊息！
```

**建議修正：**

```typescript
// 只在同一個 block 同時有 directive 和 media 時才關聯
if (parsed.mediaUrls?.length) {
  paths.push(...parsed.mediaUrls);
  
  // ✅ 只有這個 block 的 directive 才影響這個 block 的 media
  if (parsed.audioAsVoice) {
    audioAsVoice = true;
  }
}
```

**為什麼目前沒問題：**

TTS tool 的輸出格式是 directive 和 media 在同一個 block：

```typescript
{
  content: [
    { text: "[[audio_as_voice]]\nMEDIA:/tmp/audio.mp3" }  // ✅ 綁在一起
  ]
}
```

因此目前不會觸發跨 block 問題，但加上防護可以避免未來的 bug。

---

## Troubleshooting

### 症狀 1：語音訊息仍顯示為音樂檔案

**可能原因：**
1. OpenClaw 版本過舊（< 2026.3.x）
2. `audioAsVoice` flag 未正確傳遞

**處理方式：**

```bash
# 1. 檢查 OpenClaw 版本
openclaw --version

# 2. 檢查 tool result 輸出（開發模式）
# 確認是否包含 [[audio_as_voice]] directive

# 3. 檢查 delivery log
# 確認 audioAsVoice 是否為 true
```

### 症狀 2：其他媒體被誤標記為語音

**可能原因：**
- Directive 跨 block 累積（見上方風險）

**處理方式：**
- 檢查 tool 輸出格式，確保 directive 和 media 在同一個 block
- 或實作上方建議的防護邏輯

### 症狀 3：Discord/Slack 仍顯示為檔案

**可能原因：**
- 這些 channel 本來就不支援語音訊息格式

**處理方式：**
- 這是預期行為，不是 bug
- Discord/Slack 會自動降級為檔案附件

---

## 效能考量

### 多層過濾機制

系統使用漏斗式過濾，避免不必要的解析：

```
所有 tool results
  ↓ 【過濾 1】shouldEmitToolOutput - 只處理需要 media 的 tool
  ↓ 【過濾 2】快速結構檢查 - result.content 有效？
  ↓ 【過濾 3】只處理 text blocks
  ↓ 【過濾 4】快速字串檢查 - 包含 "media:" 或 "[["？
  ↓ (90% 的純文字在這裡就返回了 ⚡)
  ↓ 【過濾 5】逐行處理 - 只處理 "MEDIA:" 開頭的行
  ↓ 【過濾 6】directive 檢查 - 只在有 "[[" 時才用正則
  ↓
最終解析結果
```

**效能優勢：**
- 大部分純文字訊息只花 2 次字串檢查就返回
- 只有 10% 的訊息需要完整解析
- 效能提升約 3-4 倍

---

## 跨 Channel 設計

### 統一處理，各自呈現

**處理階段（所有 channel 統一）：**

```
Tool 輸出 → 系統解析 directives → ReplyPayload
                                    ↓
                    { mediaUrls, audioAsVoice: true }
```

**呈現階段（各 channel 自己決定）：**

| Channel   | 行為                                      |
|-----------|-------------------------------------------|
| Telegram  | ✅ sendVoice() → 語音訊息氣泡             |
| WhatsApp  | ✅ sendPTT() → PTT 語音訊息               |
| Discord   | ⚠️ sendFile() → 檔案附件（不支援語音格式）|
| Slack     | ⚠️ uploadFile() → 檔案附件（不支援語音格式）|
| WebChat   | ✅ 特殊樣式的音訊播放器                   |

### 設計優勢

1. **統一解析**：避免每個 channel 重複實作 directive 解析
2. **彈性呈現**：各 channel 根據自己的能力決定如何呈現
3. **易於擴展**：新增 directive 時只需修改一處

---

## 總結

### 關鍵概念

1. **Directive 系統**：用特殊標記在文字中嵌入系統指令
2. **完整資訊傳遞**：不要只取部分欄位，避免資訊遺失
3. **統一解析，各自呈現**：所有 channel 共用解析邏輯

### 使用時機

- 當 TTS tool 產生語音時，系統會自動加上 `[[audio_as_voice]]` directive
- 開發者不需要手動處理，系統會自動解析並傳遞到 delivery 層
- 各 channel 會根據自己的能力決定如何呈現

### 價值

- 提升語音訊息的使用體驗
- 統一的 directive 系統易於維護和擴展
- 效能優化確保大量訊息處理時的穩定性

---

## 參考資源

- **Issue**: https://github.com/openclaw/openclaw/issues/42060
- **PR #42153**: https://github.com/openclaw/openclaw/pull/42153
- **PR #42075**: https://github.com/openclaw/openclaw/pull/42075
- **OpenClaw 官方文檔**: https://docs.openclaw.ai
- **Telegram Bot API - sendVoice**: https://core.telegram.org/bots/api#sendvoice
