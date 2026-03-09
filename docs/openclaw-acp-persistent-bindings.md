# OpenClaw 2026.3.7 重大更新：ACP 持久化綁定 (Persistent Bindings)

> **版本資訊**  
> - 適用版本: OpenClaw 2026.3.7+  
> - 當前版本: 2026.3.8  
> - 更新日期: 2026-03-09

## TL;DR

**問題：** 舊版 OpenClaw 的 ACP thread 綁定在 Gateway 重啟後會消失，需要重新執行 `/acp spawn`。

**解決：** 新版 (2026.3.7+) 支援持久化綁定，Gateway 重啟後自動恢復，對話無縫繼續。

**關鍵改進：**
- 綁定資訊持久化儲存（`bindings[]` 配置或資料庫）
- Gateway 重啟時自動恢復所有綁定
- 支援預先配置永久綁定
- 對話記憶完整保留並自動載入

## 什麼是 Thread Binding（執行緒綁定）？

Thread Binding 是將「聊天對話空間」綁定到「特定 AI 會話」的機制。

```
Discord Thread #123  ←綁定→  ACP Session (codex-abc123)
     (對話空間)                  (AI 代理會話)
```

**效果：**
- 在 Thread #123 發的所有訊息都會自動路由到 `codex-abc123` 這個 session
- Codex 的回覆也會回到 Thread #123
- 保持對話的連續性和上下文

## 解決的問題

### 舊版痛點（2026.3.7 之前）

**舊版流程：**

```
Day 1 - 10:00  /acp spawn codex --thread auto
               ✅ 綁定建立（只在記憶體）

Day 1 - 10:05  "fix bug in auth.ts"
               ✅ Codex 開始修

Day 1 - 12:00  Gateway 重啟
               ❌ 綁定消失（記憶還在磁碟）

Day 1 - 12:05  "修好了嗎？"
               ❌ 找不到綁定，路由到 main agent
               Main: "什麼修好了？"（沒上下文）

Day 1 - 12:10  😤 需要重新 /acp spawn
```

### 新版解決方案（2026.3.7+）

**核心改進：** 綁定持久化儲存，Gateway 重啟後自動恢復

**新版流程：**

```
Day 1 - 10:00  /acp spawn codex --thread auto
               ✅ 綁定建立並持久化

Day 1 - 10:05  "fix bug in auth.ts"
               ✅ Codex 開始修

Day 1 - 12:00  Gateway 重啟
               ✅ 自動恢復綁定

Day 1 - 12:05  "修好了嗎？"
               ✅ 找到綁定，路由到 codex
               Codex: "是的，已經修好了"（有上下文）

Day 1 - 12:10  😊 繼續工作，完全無感
```

**技術細節：**

```
持久化儲存：
{
  "bindings": [{
    "type": "acp",
    "match": { "channel": "discord", "peer": {"id": "123"} },
    "acp": { "agent": "codex", "mode": "persistent" }
  }]
}

Gateway 啟動時：
1. 讀取 bindings[] 配置
2. 恢復所有 ACP 綁定
3. 檢查 session 狀態
4. 自動重建 process 並載入歷史
```

## 新功能：預先配置綁定

除了動態建立，新版還支援在配置中預先定義綁定：

### 配置範例：

```json
{
  "bindings": [
    {
      "type": "acp",
      "match": {
        "channel": "discord",
        "peer": {
          "id": "1234567890"  // Discord channel/thread ID
        }
      },
      "acp": {
        "agent": "codex",
        "backend": "acpx",
        "mode": "persistent"
      }
    },
    {
      "type": "acp",
      "match": {
        "channel": "telegram",
        "peer": {
          "id": "chatId:topic:5678"  // Telegram topic ID
        }
      },
      "acp": {
        "agent": "claude",
        "backend": "acpx",
        "mode": "persistent"
      }
    }
  ]
}
```

### 效果：

```
Gateway 啟動時：
✅ 自動建立所有預先定義的綁定
✅ 不需要手動 /acp spawn
✅ 永久綁定，除非手動刪除

使用場景：
• 專案頻道：#backend-dev → codex
• 文件頻道：#docs → claude
• 測試頻道：#qa → gemini
```

## 支援的平台

### Discord
- ✅ Discord channels
- ✅ Discord threads
- 配置：`channels.discord.threadBindings.spawnAcpSessions=true`

### Telegram
- ✅ Telegram forum topics
- ✅ Telegram DM topics
- 配置：`channels.telegram.threadBindings.spawnAcpSessions=true`

## 記憶管理說明

### 重要：記憶一直都在！

很多人誤解舊版「記憶會消失」，其實不是：

```
┌─────────────────────────────────────────────────┐
│ 記憶層（一直都在磁碟上）                         │
│ ~/.acpx/sessions/codex-abc123/                  │
│ ├── history.jsonl    ← 對話歷史 ✅              │
│ ├── metadata.json    ← Session 資訊 ✅          │
│ └── state.json       ← 狀態 ✅                  │
└─────────────────────────────────────────────────┘
                    ↕
        舊版問題：找不到記憶（綁定消失）
        新版解決：能找到記憶（綁定持久化）
                    ↕
┌─────────────────────────────────────────────────┐
│ 綁定層                                           │
│ Thread #123 → codex-abc123                      │
│ 舊版：❌ 只在記憶體                              │
│ 新版：✅ 持久化儲存                              │
└─────────────────────────────────────────────────┘
```

### 完整的三層架構：

```
┌─────────────────────────────────────────────────┐
│ 1️⃣ OpenClaw Gateway 綁定層                      │
│ Thread #123 → codex-abc123                      │
│ 新版：持久化 ✅                                  │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│ 2️⃣ ACPX Session 記憶層                          │
│ ~/.acpx/sessions/codex-abc123/                  │
│ • history.jsonl (對話歷史)                      │
│ • metadata.json (session 資訊)                  │
│ 一直都持久化 ✅                                  │
└──────────────────┬──────────────────────────────┘
                   ↓
┌─────────────────────────────────────────────────┐
│ 3️⃣ Process 層（動態，可重建）                   │
│ ACPX Process (PID 12345)                        │
│   ↓                                             │
│ Codex CLI Process (PID 12346)                   │
│ 重啟時自動重建並載入記憶 ✅                      │
└─────────────────────────────────────────────────┘
```

## 使用指南

### 前置條件

**必要條件：**
- OpenClaw 2026.3.7+
- ACP backend 已啟用（`acp.enabled=true`）
- ACPX plugin 已安裝並啟用
- 在支援的平台（Discord/Telegram）

**檢查方式：**
```bash
# 檢查版本
openclaw --version

# 檢查 ACP 狀態
/acp doctor

# 檢查 plugin
openclaw plugins list
```

### 動態建立綁定（推薦新手）

```bash
# 最簡單
/acp spawn codex --thread auto

# 一次性（不保留 session）
/acp spawn codex --mode oneshot

# 命名 session（平行任務）
/acp spawn codex -s backend --thread auto
/acp spawn codex -s frontend --thread auto
```

### 預先配置綁定（推薦進階使用者）

**前置條件：**
- 已知 Discord channel ID 或 Telegram chat ID + topic ID
- 已配置對應的 agent runtime

在 `openclaw.json` 中：

```json
{
  "bindings": [
    {
      "type": "acp",
      "match": {
        "channel": "discord",
        "peer": {"id": "YOUR_CHANNEL_ID"}
      },
      "acp": {
        "agent": "codex",
        "backend": "acpx",
        "mode": "persistent"
      }
    }
  ]
}
```

### 管理綁定

**前置條件：**
- 已有執行中的 ACP session

```bash
# 查看狀態
/acp status

# 重置 session（清空記憶）
/acp reset

# 關閉 session（移除綁定）
/acp close

# 列出所有 sessions
/acp sessions
```

## 配置詳解

### 全域 ACP 配置

**前置條件：**
- 編輯 `~/.openclaw/openclaw.json` 或專案配置檔

```json
{
  "acp": {
    "enabled": true,
    "dispatch": {
      "enabled": true
    },
    "backend": "acpx",
    "defaultAgent": "codex",
    "allowedAgents": ["pi", "claude", "codex", "opencode", "gemini", "kimi"],
    "maxConcurrentSessions": 8,
    "stream": {
      "coalesceIdleMs": 300,
      "maxChunkChars": 1200,
      "repeatSuppression": true,
      "deliveryMode": "live",
      "maxOutputChars": 50000
    },
    "runtime": {
      "ttlMinutes": 120
    }
  }
}
```

**配置說明：**
- `enabled`: 全域 ACP 開關
- `dispatch.enabled`: 控制是否自動分派訊息到 ACP sessions
- `backend`: ACP 實作後端（目前為 `acpx`）
- `defaultAgent`: 預設使用的 agent（當未指定時）
- `allowedAgents`: 允許使用的 agent 清單
- `maxConcurrentSessions`: 最大並行 session 數量
- `stream.coalesceIdleMs`: 串流文字合併的閒置時間（毫秒）
- `stream.maxChunkChars`: 每個串流區塊的最大字元數
- `stream.repeatSuppression`: 是否抑制重複的狀態/工具投影行
- `stream.deliveryMode`: `live`（即時串流）或 `final_only`（僅最終結果）
- `stream.maxOutputChars`: 每個回合最大輸出字元數
- `runtime.ttlMinutes`: ACP session worker 的閒置 TTL（分鐘）

### Thread Binding 配置

**前置條件：**
- 使用支援 thread binding 的平台（Discord/Telegram）

```json
{
  "session": {
    "threadBindings": {
      "enabled": true,
      "idleHours": 24,
      "maxAgeHours": 0
    }
  },
  "channels": {
    "discord": {
      "threadBindings": {
        "enabled": true,
        "spawnAcpSessions": true,
        "idleHours": 24,
        "maxAgeHours": 0
      }
    },
    "telegram": {
      "threadBindings": {
        "enabled": true,
        "spawnAcpSessions": true
      }
    }
  }
}
```

**配置說明：**
- `session.threadBindings.*`: 全域預設值
- `channels.discord.threadBindings.*`: Discord 專屬覆寫
- `channels.telegram.threadBindings.*`: Telegram 專屬覆寫
- `enabled`: 啟用 thread binding 功能
- `spawnAcpSessions`: 允許為 ACP sessions 建立/綁定 threads
- `idleHours`: 閒置多久後自動解除綁定（0 = 不限制）
- `maxAgeHours`: 綁定最大存活時間（0 = 不限制）

### ACPX Plugin 配置

**前置條件：**
- ACPX plugin 已安裝（`openclaw plugins install acpx`）

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "command": "acpx",
          "expectedVersion": "0.1.15",
          "permissionMode": "approve-reads",
          "nonInteractivePermissions": "fail",
          "timeoutSeconds": 300,
          "queueOwnerTtlSeconds": 3600,
          "mcpServers": {
            "playwright": {
              "command": "npx",
              "args": ["-y", "@playwright/mcp"],
              "env": {
                "NODE_ENV": "production"
              }
            }
          }
        }
      }
    }
  }
}
```

**配置說明：**
- `command`: ACPX 執行檔路徑或指令名稱
- `expectedVersion`: 預期的 ACPX 版本（`"any"` = 不檢查）
- `permissionMode`: 權限模式
  - `approve-all`: 自動批准所有操作
  - `approve-reads`: 自動批准讀取，詢問寫入（預設）
  - `deny-all`: 拒絕所有操作
- `nonInteractivePermissions`: 非互動模式下的權限處理
  - `fail`: 中止 session（預設）
  - `deny`: 靜默拒絕並繼續
- `timeoutSeconds`: Runtime timeout（秒）
- `queueOwnerTtlSeconds`: Queue owner TTL（秒）
- `mcpServers`: MCP (Model Context Protocol) 伺服器配置

### 管理綁定

```bash
# 查看狀態
/acp status

# 重置 session（清空記憶）
/acp reset

# 關閉 session（移除綁定）
/acp close

# 列出所有 sessions
/acp sessions
```

## 對比總結

| 項目 | 舊版 (< 2026.3.7) | 新版 (>= 2026.3.7) |
|------|------------------|-------------------|
| **綁定儲存** | 記憶體 | 持久化 |
| **重啟後綁定** | ❌ 消失 | ✅ 自動恢復 |
| **對話記憶** | ✅ 保留 | ✅ 保留 |
| **重啟後能用記憶** | ❌ 找不到 | ✅ 自動載入 |
| **需要重新 spawn** | ✅ 是 | ❌ 否 |
| **預先配置綁定** | ❌ 不支援 | ✅ 支援 |
| **使用者體驗** | 中斷 | 無縫 |

## 升級建議

### 如果你是舊版使用者：

1. **升級到 2026.3.7+**
2. **現有的 `/acp spawn` 指令不需要改**，自動享受持久化
3. **考慮使用預先配置**，為常用頻道建立永久綁定
4. **不再需要擔心 Gateway 重啟**

### 如果你是新使用者：

1. **直接使用 `/acp spawn codex --thread auto`**
2. **享受無縫的對話體驗**
3. **進階使用者可以研究預先配置**

## 技術細節

### 綁定恢復流程：

```
Gateway 啟動
  ↓
讀取 bindings[] 配置
  ↓
對每個 ACP binding：
  1. 檢查 session 是否存在
  2. 如果存在，恢復綁定
  3. 如果 process 死了，標記為需要重建
  ↓
使用者發訊息時：
  1. 查詢綁定
  2. 找到 session
  3. 檢查 process 狀態
  4. 如果需要，重建 process 並載入歷史
  5. 路由訊息
```

### Session 重建機制：

```
檢測到 process 死亡
  ↓
啟動新 ACPX process
  ↓
發送 ACP session/load 請求
  ↓
載入 ~/.acpx/sessions/codex-abc123/
  ↓
恢復完整對話歷史
  ↓
繼續處理訊息
```

## 相關資源

- **OpenClaw ACP 文檔**: https://docs.openclaw.ai/tools/acp-agents
- **配置參考**: https://docs.openclaw.ai/gateway/configuration-reference
- **ACPX npm**: https://www.npmjs.com/package/acpx
- **ACP 協議**: https://agentclientprotocol.com/

## 結論

OpenClaw 2026.3.7 的 ACP 持久化綁定功能解決了舊版最大的痛點：**Gateway 重啟後對話中斷**。

**核心價值：**
- ✅ 綁定持久化，重啟後自動恢復
- ✅ 對話無縫繼續，不需要重新 spawn
- ✅ 支援預先配置，實現「永久工作區」
- ✅ 記憶管理完整，從綁定到對話歷史全部持久化

這個改進讓 ACP 在 OpenClaw 中的使用體驗提升到了新的層次，特別適合需要長期維護的專案和頻道。
