# OpenClaw 多 Agent 路由架構：從 Telegram Bot 到 AI 人設

> 一篇搞懂 OpenClaw 怎麼把 Telegram Bot、Agent、記憶串在一起。

---

## 核心概念：三個角色

```
Telegram Bot          OpenClaw Agent          記憶（Session）
─────────────         ──────────────          ────────────────
你在 BotFather        你在 openclaw.json      AI 的對話歷史
建立的機器人          定義的 AI 人設           .jsonl 檔案

有自己的 token        有自己的 model、        每個 session key
能收發 Telegram       system prompt、         對應一份獨立的
訊息                  skills                  對話紀錄
```

---

## 場景一：最簡單 — 一個 Bot、一個 Agent

你只有一個 Telegram Bot，一個 AI 人設，自己一個人用。

```
                    openclaw.json
                    ┌──────────────────────────┐
                    │ agent:                   │
                    │   model: claude-sonnet   │
                    │                          │
                    │ channels:                │
                    │   telegram:              │
                    │     botToken: "111:AAA"  │
                    └──────────────────────────┘

你 (Telegram)
  │
  │  私訊 @MyBot
  ▼
┌─────────┐       ┌──────────────┐       ┌──────────────┐
│ @MyBot  │ ───→  │ Agent: main  │ ───→  │ 記憶 .jsonl  │
│ (TG Bot)│ ←───  │ (claude)     │       │ 你的對話歷史  │
└─────────┘       └──────────────┘       └──────────────┘

不需要 accounts、不需要 bindings、不需要 dmScope 設定。
一切都是預設值，直接能用。
```

---

## 場景二：多人使用 — 一個 Bot，多人聊

你把 Bot 開放給朋友用。需要 dmScope 隔離記憶。

```
                    openclaw.json
                    ┌──────────────────────────┐
                    │ session:                 │
                    │   dmScope: "per-peer"    │
                    │                          │
                    │ channels:                │
                    │   telegram:              │
                    │     botToken: "111:AAA"  │
                    │     allowFrom: ["*"]     │
                    └──────────────────────────┘

你 (peerId: 111)          小明 (peerId: 222)
  │                         │
  │  私訊 @MyBot            │  私訊 @MyBot
  ▼                         ▼
┌─────────┐              ┌─────────┐
│ @MyBot  │              │ @MyBot  │
└────┬────┘              └────┬────┘
     │                        │
     ▼                        ▼
┌──────────────┐       ┌──────────────┐
│ Agent: main  │       │ Agent: main  │    ← 同一個 Agent
└──────┬───────┘       └──────┬───────┘
       │                      │
       ▼                      ▼
┌──────────────┐       ┌──────────────┐
│ 記憶：你的    │       │ 記憶：小明的  │    ← 記憶分開！
│ direct:111   │       │ direct:222   │
└──────────────┘       └──────────────┘

dmScope: "per-peer" 讓每個人有自己的記憶房間。
你跟小明聊的內容互相看不到。
```

---

## 場景三：多 Bot、多 Agent — 不同機器人有不同靈魂

你想要一個寫程式的 AI 和一個做菜的 AI，各自有自己的 Telegram Bot。

```
                    openclaw.json
                    ┌──────────────────────────────────────────┐
                    │ agents:                                  │
                    │   list:                                  │
                    │     - id: "coder"                        │
                    │       model: "anthropic/claude-sonnet"   │
                    │                                          │
                    │     - id: "chef"                         │
                    │       model: "openai/gpt-4o"             │
                    │                                          │
                    │ channels:                                │
                    │   telegram:                              │
                    │     accounts:                            │
                    │       "code-bot":                        │
                    │         botToken: "111:AAA..."           │
                    │       "cook-bot":                        │
                    │         botToken: "222:BBB..."           │
                    │                                          │
                    │ bindings:                                │
                    │   - agentId: "coder"                     │
                    │     match: { accountID: "code-bot" }     │
                    │   - agentId: "chef"                      │
                    │     match: { accountID: "cook-bot" }     │
                    └──────────────────────────────────────────┘


你 (Telegram)
  │
  ├── 私訊 @CodeBot ──→ ┌──────────────┐    ┌──────────────┐
  │                      │ Agent: coder │───→│ coder 的記憶  │
  │                      │ (claude)     │    │ 程式對話歷史  │
  │                      └──────────────┘    └──────────────┘
  │
  └── 私訊 @CookBot ──→ ┌──────────────┐    ┌──────────────┐
                         │ Agent: chef  │───→│ chef 的記憶   │
                         │ (gpt-4o)     │    │ 食譜對話歷史  │
                         └──────────────┘    └──────────────┘

三個設定各司其職：
  accounts  → 定義有哪些 Telegram Bot（token）
  agents    → 定義有哪些 AI 人設（model、能力）
  bindings  → 把 Bot 跟 Agent 配對起來
```

---

## 場景四：跨平台 + 記憶合併

你同時用 Telegram 和 WhatsApp，希望 AI 記得你在兩邊聊過的內容。

```
                    openclaw.json
                    ┌──────────────────────────────────────────┐
                    │ session:                                 │
                    │   dmScope: "per-peer"                    │
                    │   identityLinks:                         │
                    │     "andy":                              │
                    │       - "telegram:1461779006"            │
                    │       - "whatsapp:886912345678"          │
                    │                                          │
                    │ channels:                                │
                    │   telegram:                              │
                    │     botToken: "111:AAA..."               │
                    │   whatsapp: {}                           │
                    └──────────────────────────────────────────┘


你 (Telegram)  ──→ ┐
                   ├──→  Agent: main  ──→  記憶：direct:andy
你 (WhatsApp)  ──→ ┘                       （合併！同一份）

沒有 identityLinks 的話：
  Telegram  ──→  記憶：direct:1461779006    ← 分開的
  WhatsApp  ──→  記憶：direct:886912345678  ← 分開的

identityLinks 告訴系統：
  「這兩個帳號都是 andy，請用同一個房間」
```

---

## 總結：什麼時候需要什麼設定

```
你的需求                              需要設定什麼
──────────────────────────────────    ──────────────────────────
一個人、一個平台、一個 AI              什麼都不用，預設就好
多人使用同一個 Bot                     dmScope: "per-peer"
多個 Bot 對應多個 AI 人設              accounts + agents + bindings
跨平台共用記憶                         identityLinks
```
