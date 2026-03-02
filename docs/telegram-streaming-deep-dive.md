# Telegram Bot 如何實現「打字中...」的動態回覆？

> 深入解析 OpenClaw 的 Telegram 串流輸出機制

## 前言

你有沒有好奇過，為什麼有些 Telegram Bot 能像真人一樣，先顯示「正在輸入...」，然後訊息內容逐字逐句地出現？這不是魔法，而是精心設計的三層機制。

本文將帶你深入 OpenClaw 的原始碼，看看它如何用 **250 行程式碼** 實現媲美 ChatGPT 的串流體驗。

---

## 核心概念：編輯訊息 vs 發送新訊息

傳統 Bot 的做法：
```
使用者: 你好
Bot: [等待 5 秒...]
Bot: 你好！我是 AI 助手，很高興為你服務。
```

串流 Bot 的做法：
```
使用者: 你好
Bot: 💬 正在輸入...
Bot: 你好！
Bot: 你好！我是
Bot: 你好！我是 AI 助手
Bot: 你好！我是 AI 助手，很高興為你服務。
```

**關鍵差異**：不是發送多則訊息，而是**不斷編輯同一則訊息**！

---

## 架構總覽

```
┌─────────────────────────────────────────────────────────────┐
│                      Telegram Bot API                        │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        │                     │                     │
   ┌────▼────┐          ┌─────▼─────┐        ┌─────▼─────┐
   │ Typing  │          │   Draft   │        │   Lane    │
   │Indicator│          │  Stream   │        │ Delivery  │
   └─────────┘          └───────────┘        └───────────┘
   每 3 秒發送            編輯同一則訊息         多軌道並行
   "typing"              節流 1000ms           (答案+推理)
```

---

## 第一層：Typing Indicator（打字指示器）

### 原理

Telegram Bot API 提供 `sendChatAction` 方法，可以顯示各種狀態：

```typescript
await bot.api.sendChatAction(chatId, "typing");        // 正在輸入...
await bot.api.sendChatAction(chatId, "upload_photo");  // 正在上傳照片...
await bot.api.sendChatAction(chatId, "record_voice");  // 正在錄音...
```

**問題**：這個狀態只維持 **5 秒**，之後會自動消失。

### 解決方案：Keepalive Loop

```typescript
// src/channels/typing.ts

const keepaliveLoop = createTypingKeepaliveLoop({
  intervalMs: 3_000,  // 每 3 秒重發一次
  onTick: async () => {
    await bot.api.sendChatAction(chatId, "typing");
  }
});

// 開始回覆時啟動
keepaliveLoop.start();

// 回覆完成時停止
keepaliveLoop.stop();
```

### 生命週期圖

```
時間軸 ────────────────────────────────────────────────────▶

使用者發送訊息
    │
    ▼
┌───────┐  3s   ┌───────┐  3s   ┌───────┐  3s   ┌───────┐
│typing │ ────▶ │typing │ ────▶ │typing │ ────▶ │typing │
└───────┘       └───────┘       └───────┘       └───────┘
    │               │               │               │
    └───────────────┴───────────────┴───────────────┘
                    持續顯示「正在輸入...」
                                                    │
                                                    ▼
                                              回覆完成，停止
```

### 安全機制

1. **TTL（Time To Live）**：最多 60 秒自動停止
2. **401 錯誤退避**：Token 失效時避免無限重試

```typescript
// src/telegram/sendchataction-401-backoff.ts

let consecutive401Failures = 0;

try {
  await sendChatAction(chatId, "typing");
  consecutive401Failures = 0;  // 成功，重置計數器
} catch (error) {
  if (is401Error(error)) {
    consecutive401Failures++;
    
    if (consecutive401Failures >= 10) {
      // 暫停所有 typing 請求，避免被 Telegram 封禁
      suspended = true;
      console.error("CRITICAL: Bot token 可能失效！");
    } else {
      // 指數退避：1s → 2s → 4s → 8s → ...
      await sleep(Math.pow(2, consecutive401Failures) * 1000);
    }
  }
}
```

---

## 第二層：Draft Stream（草稿串流）

這是**核心機制**，負責動態更新訊息內容。

### 核心邏輯

```typescript
// src/telegram/draft-stream.ts

let streamMessageId: number | undefined;
let lastSentText = "";

const sendOrEditStreamMessage = async (text: string): Promise<boolean> => {
  // 1. 去除空白
  const trimmed = text.trimEnd();
  if (!trimmed) return false;
  
  // 2. 渲染 Markdown → HTML
  const rendered = renderTelegramHtmlText(trimmed);
  
  // 3. 檢查是否超過 Telegram 上限（4096 字）
  if (rendered.length > 4096) {
    console.warn("訊息過長，停止串流");
    return false;
  }
  
  // 4. 避免重複發送相同內容
  if (rendered === lastSentText) return true;
  
  lastSentText = rendered;
  
  // 5. 第一次發送 vs 後續編輯
  if (typeof streamMessageId === "number") {
    // 已有訊息 → 編輯它
    await api.editMessageText(chatId, streamMessageId, rendered, {
      parse_mode: "HTML"
    });
  } else {
    // 第一次 → 發送新訊息
    const sent = await api.sendMessage(chatId, rendered);
    streamMessageId = sent.message_id;
  }
  
  return true;
};
```

### 節流機制

為了避免 API Rate Limit，使用**節流控制**：

```typescript
const throttleMs = 1000;  // 最快每秒更新一次
let pendingText = "";
let throttleTimer: NodeJS.Timeout | undefined;

const update = (text: string) => {
  pendingText = text;
  
  if (throttleTimer) return;  // 已有排程，等待
  
  throttleTimer = setTimeout(async () => {
    await sendOrEditStreamMessage(pendingText);
    throttleTimer = undefined;
  }, throttleMs);
};
```

### 流程圖

```
AI 生成文字流
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  update("Hello")                                         │
│  update("Hello world")                                   │
│  update("Hello world! How")                              │
│  update("Hello world! How can I help?")                  │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              節流控制（Throttle）                         │
│  ┌──────┐  1s   ┌──────┐  1s   ┌──────┐  1s   ┌──────┐ │
│  │ 批次1 │ ────▶ │ 批次2 │ ────▶ │ 批次3 │ ────▶ │ 批次4 │ │
│  └──────┘       └──────┘       └──────┘       └──────┘ │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              Telegram API                                │
│  sendMessage(...)        → message_id = 12345            │
│  editMessageText(12345)  → "Hello world"                 │
│  editMessageText(12345)  → "Hello world! How"            │
│  editMessageText(12345)  → "Hello world! How can I help?"│
└─────────────────────────────────────────────────────────┘
```

### 最小字數門檻

為了改善推播通知品質，設定**最小 30 字**才發送第一條訊息：

```typescript
const minInitialChars = 30;

if (typeof streamMessageId !== "number") {
  // 第一次發送
  if (text.length < minInitialChars) {
    return false;  // 累積更多內容再發
  }
}
```

**為什麼？**
- 避免推播通知只顯示「你」、「你好」這種不完整的內容
- 使用者體驗更好

---

## 第三層：Lane Delivery（多軌道輸出）

OpenClaw 支援**同時串流兩個獨立的訊息**：

1. **Answer Lane**：主要回答內容
2. **Reasoning Lane**：AI 的推理過程（類似 ChatGPT 的 "Thinking..."）

### 架構圖

```
┌─────────────────────────────────────────────────────────┐
│                    AI 生成內容                           │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
    ┌───────────────┐       ┌───────────────┐
    │ Reasoning Lane│       │  Answer Lane  │
    │  (推理過程)    │       │   (最終答案)   │
    └───────────────┘       └───────────────┘
            │                       │
            ▼                       ▼
    ┌───────────────┐       ┌───────────────┐
    │  Message #1   │       │  Message #2   │
    │  (獨立訊息)    │       │  (獨立訊息)    │
    └───────────────┘       └───────────────┘
```

### 實作

```typescript
// src/telegram/lane-delivery.ts

const lanes = {
  answer: createDraftLane("answer", canStreamAnswerDraft),
  reasoning: createDraftLane("reasoning", canStreamReasoningDraft)
};

// AI 生成推理過程
lanes.reasoning.stream?.update("正在分析問題...");
lanes.reasoning.stream?.update("正在分析問題...找到關鍵字");

// AI 生成答案
lanes.answer.stream?.update("根據你的問題");
lanes.answer.stream?.update("根據你的問題，答案是...");
```

### 使用者看到的效果

```
┌─────────────────────────────────────┐
│ 🤔 推理過程                          │
│ 正在分析問題...找到關鍵字             │
│ 查詢資料庫...找到 3 筆相關資料        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ 💡 答案                              │
│ 根據你的問題，答案是...               │
└─────────────────────────────────────┘
```

---

## 完整流程圖

```
使用者發送訊息
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 1. 啟動 Typing Indicator                                 │
│    bot.api.sendChatAction(chatId, "typing")              │
│    每 3 秒重發一次                                        │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 2. AI 開始生成內容                                        │
│    stream.update("Hello")                                │
│    stream.update("Hello world")                          │
│    stream.update("Hello world! How can I help?")         │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Draft Stream 處理                                     │
│    ┌──────────────────────────────────────────────┐     │
│    │ 累積到 30 字 → sendMessage()                  │     │
│    │ 得到 message_id = 12345                       │     │
│    └──────────────────────────────────────────────┘     │
│    ┌──────────────────────────────────────────────┐     │
│    │ 每 1 秒 → editMessageText(12345, newText)     │     │
│    └──────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│ 4. 完成                                                  │
│    stream.flush()  → 最後一次編輯                        │
│    keepaliveLoop.stop()  → 停止 typing indicator         │
└─────────────────────────────────────────────────────────┘
```

---

## 關鍵技術細節

### 1. 錯誤處理

```typescript
try {
  await api.editMessageText(chatId, messageId, text);
} catch (err) {
  if (err.message.includes("message is not modified")) {
    // 內容相同，忽略錯誤
    return true;
  }
  if (err.message.includes("message to edit not found")) {
    // 訊息被刪除，停止串流
    streamState.stopped = true;
    return false;
  }
  throw err;
}
```

### 2. Markdown → HTML 轉換

Telegram 支援兩種格式：
- **Markdown**：簡單但功能有限
- **HTML**：功能完整，支援表格、程式碼區塊

OpenClaw 使用 HTML 模式：

```typescript
// src/telegram/format.ts

function renderTelegramHtmlText(markdown: string): string {
  return markdown
    .replace(/\*\*(.*?)\*\*/g, '<b>$1</b>')      // **粗體**
    .replace(/\*(.*?)\*/g, '<i>$1</i>')          // *斜體*
    .replace(/`(.*?)`/g, '<code>$1</code>')      // `程式碼`
    .replace(/```(.*?)```/gs, '<pre>$1</pre>');  // ```程式碼區塊```
}
```

### 3. 超過 4096 字的處理

Telegram 單則訊息上限 **4096 字元**，超過時：

```typescript
if (text.length > 4096) {
  // 停止串流，改用分段發送
  streamState.stopped = true;
  
  // 將內容切成多則訊息
  const chunks = splitIntoChunks(text, 4096);
  for (const chunk of chunks) {
    await api.sendMessage(chatId, chunk);
  }
}
```

---

## 效能優化

### 1. 節流（Throttle）

```
無節流：
AI 生成 → 立即發送 → API 請求 (100 次/秒) ❌ Rate Limit!

有節流：
AI 生成 → 累積 1 秒 → 批次發送 (1 次/秒) ✅
```

### 2. 去重（Deduplication）

```typescript
if (rendered === lastSentText) {
  return true;  // 內容相同，跳過
}
```

### 3. 最小字數門檻

```
無門檻：
推播通知：「你」 → 「你好」 → 「你好！」 ❌ 體驗差

有門檻（30 字）：
推播通知：「你好！我是 AI 助手，很高興為你服務。」 ✅
```

---

## 配置選項

在 `openclaw.json` 中可以自訂行為：

```json
{
  "channels": {
    "telegram": {
      "streaming": "block",
      "blockStreaming": false,
      "timeoutSeconds": 30,
      "throttleMs": 1000,
      "minInitialChars": 30
    }
  }
}
```

**streaming 模式：**
- `"off"`：完全關閉串流，等全部生成完才發送
- `"partial"`：部分串流（僅答案）
- `"block"`：完整串流（答案 + 推理）
- `"progress"`：顯示進度條

---

## 實戰案例

### 案例 1：長文章生成

```
使用者: 寫一篇關於 AI 的文章

Bot: 💬 正在輸入...

Bot: # AI 的未來
     人工智慧（AI）是...

Bot: # AI 的未來
     人工智慧（AI）是當今最熱門的技術之一。
     從自動駕駛到...

Bot: # AI 的未來
     人工智慧（AI）是當今最熱門的技術之一。
     從自動駕駛到醫療診斷，AI 正在改變我們的生活。
     
     ## 應用領域
     1. 自然語言處理
     2. 電腦視覺
     ...
```

### 案例 2：程式碼生成

```
使用者: 寫一個 Python 排序函式

Bot: 💬 正在輸入...

Bot: ```python
     def sort_list(arr):

Bot: ```python
     def sort_list(arr):
         return sorted(arr)

Bot: ```python
     def sort_list(arr):
         """
         排序列表
         """
         return sorted(arr)
     
     # 使用範例
     numbers = [3, 1, 4, 1, 5]
     print(sort_list(numbers))
     ```
```

---

## 常見問題

### Q1: 為什麼不用 WebSocket？

**A:** Telegram Bot API 是 HTTP-based，不支援 WebSocket。但透過 `editMessageText` 可以達到類似效果。

### Q2: 會不會被 Rate Limit？

**A:** 有三層保護：
1. 節流控制（1 秒最多 1 次）
2. 去重（相同內容不重複發送）
3. 指數退避（錯誤時自動減速）

### Q3: 如果使用者刪除訊息會怎樣？

**A:** 會捕捉 `message to edit not found` 錯誤，自動停止串流。

### Q4: 支援群組嗎？

**A:** 支援！包括：
- 普通群組
- 超級群組
- 論壇主題（Forum Topics）

---

## 總結

OpenClaw 的 Telegram 串流機制展示了如何用**簡單的技術**實現**複雜的體驗**：

1. **Typing Indicator**：讓使用者知道 Bot 正在工作
2. **Draft Stream**：透過編輯訊息實現動態更新
3. **Lane Delivery**：支援多軌道並行輸出

核心思想：
- ✅ 不是發送多則訊息，而是**編輯同一則**
- ✅ 不是即時發送，而是**節流批次處理**
- ✅ 不是無限重試，而是**智慧退避**

這套機制不僅適用於 Telegram，也可以應用到其他平台（Discord、Slack 等）。

---

## 參考資源

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Telegram Bot API 文件](https://core.telegram.org/bots/api)
- 原始碼位置：
  - `src/telegram/draft-stream.ts` - 核心串流邏輯
  - `src/channels/typing.ts` - 打字指示器
  - `src/telegram/sendchataction-401-backoff.ts` - 錯誤處理
  - `src/telegram/lane-delivery.ts` - 多軌道輸出

---

**作者註**：本文基於 OpenClaw 2026.2 版本原始碼分析，實際實作細節可能隨版本更新而變化。
