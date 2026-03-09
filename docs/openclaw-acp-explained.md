# OpenClaw 中的 ACP (Agent Client Protocol) 完全解析

> **版本資訊**  
> - OpenClaw: 2026.3.8  
> - ACPX: 0.1.15  
> - 更新日期: 2026-03-09

## TL;DR

- **ACP** 是統一 IDE 與 AI 代理溝通的開放協議，OpenClaw 透過它支援 IDE 整合（如 Zed Editor）和聊天介面
- **ACPX** 是 OpenClaw 的 ACP 實作引擎，支援 Codex/Claude/Gemini/Pi 等 coding agents
- **三層架構**：IDE/聊天 → Gateway → ACPX Runtime → Coding Agents
- **主要用途**：IDE 整合、聊天介面、程式化呼叫

## 什麼是 ACP？

**你可以用 ACP 做什麼？**
- 在 Zed/VS Code 中直接呼叫 Codex/Claude
- 在 Discord/Telegram 中使用專業 coding agents
- 一次整合，支援多個 AI 工具

**ACP (Agent Client Protocol)** 是讓這些成為可能的開放協議，用來標準化「編輯器/IDE」與「AI 編碼代理」之間的溝通。

- 官方網站：https://agentclientprotocol.com/
- GitHub：https://github.com/agentclientprotocol/agent-client-protocol
- 類似概念：MCP (Model Context Protocol)，但 ACP 專注於編碼場景

## 為什麼需要 ACP？

### 問題：每個 AI 編碼工具都有自己的介面

```
Codex CLI     → 自己的命令格式
Claude Code   → 自己的命令格式  
Gemini CLI    → 自己的命令格式
Pi Agent      → 自己的命令格式
```

如果你想在工具中整合這些 AI，你需要：
- 為每個工具寫不同的整合代碼
- 處理不同的輸入/輸出格式
- 維護多套 PTY (終端) 抓取邏輯

### 解決方案：ACP 統一介面

```
你的工具 → ACP 協議 → 任何支援 ACP 的 AI 代理
```

只要實作一次 ACP，就能支援所有 ACP 相容的 AI 代理！

## OpenClaw 的 ACP 架構

### 三層架構

```
外部介面（IDE/聊天）
         ↓
OpenClaw Gateway（路由中心）
         ↓
ACPX Runtime（ACP 實作）
         ↓
Coding Agents（Codex/Claude/...）
```

**簡單說：**
- **Gateway**：統一的訊息路由中心
- **ACPX**：ACP 協議的實作引擎
- **Agents**：實際執行編碼任務的 AI 工具

## ACPX 是什麼？

**ACPX** = ACP eXecution/eXtension

- npm 套件：`acpx@0.1.15`
- 作用：ACP 協議的 CLI 客戶端實作
- 在 OpenClaw 中作為 plugin 存在 (`extensions/acpx/`)

### ACPX 支援的 Agents

| Agent 名稱 | 底層工具 | 指令 |
|-----------|---------|------|
| `pi` | Pi Coding Agent | `npx pi-acp` |
| `claude` | Claude Code (Anthropic) | `npx -y @zed-industries/claude-agent-acp` |
| `codex` | Codex CLI (OpenAI) | `npx @zed-industries/codex-acp` |
| `opencode` | OpenCode | `npx -y opencode-ai acp` |
| `gemini` | Gemini CLI (Google) | `gemini` |

## 使用場景

### 場景 1：聊天介面 (Discord/Telegram)

```bash
# 在 Discord 中使用
/acp spawn codex --mode persistent --thread auto

# 在 Telegram 中使用
/acp spawn claude --mode persistent
```

**流程：**
```
Discord/Telegram 訊息
  ↓
OpenClaw Gateway
  ↓
ACPX Runtime
  ↓
Codex CLI / Claude Code
```

### 場景 2：程式化呼叫

```typescript
// 透過 sessions_spawn tool
sessions_spawn({
  runtime: "acp",
  agentId: "codex",
  message: "Fix the bug in auth.ts"
})
```

## 實際配置範例

### 啟用 ACP Runtime

```json
{
  "acp": {
    "enabled": true,
    "backend": "acpx",
    "defaultAgent": "codex"
  }
}
```

### 配置 ACPX Plugin

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "config": {
          "permissionMode": "approve-reads",
          "timeoutSeconds": 300,
          "mcpServers": {
            "playwright": {
              "command": "npx",
              "args": ["-y", "@playwright/mcp"]
            }
          }
        }
      }
    }
  }
}
```

### 權限模式

- `approve-all`：自動批准所有操作
- `approve-reads`：自動批准讀取，詢問寫入（預設）
- `deny-all`：拒絕所有操作

## 常用命令

### 建立 ACP Session

```bash
# 建立持久化 session
/acp spawn codex --mode persistent --thread auto

# 一次性執行
/acp spawn claude --mode oneshot
```

### 管理 Session

```bash
# 查看狀態
/acp status

# 設定模型
/acp model anthropic/claude-3-5-sonnet-20241022

# 設定權限
/acp permissions approve-all

# 設定 timeout
/acp timeout 600

# 取消當前任務
/acp cancel

# 關閉 session
/acp close
```

### 引導 Session

```bash
# 在不重置 context 的情況下調整方向
/acp steer "focus on performance optimization"
```

## Thread Binding（執行緒綁定）

ACP sessions 可以綁定到聊天執行緒：

```bash
# 在 Discord thread 中
/acp spawn codex --mode persistent --thread auto
```

**效果：**
- 該 thread 的所有後續訊息都會路由到同一個 ACP session
- 輸出會回到同一個 thread
- 直到 session 關閉或 thread 過期

**支援的平台：**
- Discord threads/channels
- Telegram forum topics

## 安全注意事項

### 權限模式選擇

- **開發環境**：可使用 `approve-all` 提高效率
- **生產環境**：建議使用 `approve-reads` 或更嚴格的設定
- **多人共用**：務必設定 `deny-all` 並手動審核每個操作

### Sandbox 限制

⚠️ **重要**：ACP sessions 目前在 **host runtime** 運行，不在 OpenClaw sandbox 內

- 從 sandboxed session 無法 spawn ACP sessions
- ACP agents 可以存取 host 檔案系統
- 如需 sandbox 執行，請使用 `runtime="subagent"`

### Token 安全

```bash
# ❌ 不安全：token 會出現在 process list
openclaw acp --token <token>

# ✅ 安全：使用檔案
openclaw acp --token-file ~/.openclaw/gateway.token

# ✅ 安全：使用環境變數
export OPENCLAW_GATEWAY_TOKEN=<token>
openclaw acp
```

## Troubleshooting

### 症狀：`ACP runtime backend is not configured`
- **可能原因**：Backend plugin 未安裝或未啟用
- **處理方式**：
  ```bash
  openclaw plugins install acpx
  openclaw config set plugins.entries.acpx.enabled true
  /acp doctor
  ```

### 症狀：`ACP is disabled by policy (acp.enabled=false)`
- **可能原因**：ACP 功能被全域停用
- **處理方式**：
  ```bash
  openclaw config set acp.enabled true
  # 重啟 Gateway
  openclaw gateway restart
  ```

### 症狀：IDE 連接失敗或無回應
- **可能原因**：
  1. Gateway 未啟動
  2. URL/token 設定錯誤
  3. 網路連線問題
- **處理方式**：
  ```bash
  # 1. 檢查 Gateway 狀態
  openclaw gateway status
  
  # 2. 測試 ACP bridge
  openclaw acp client
  
  # 3. 檢查連線設定
  openclaw config get gateway.remote.url
  ```

### 症狀：`Sandboxed sessions cannot spawn ACP sessions`
- **可能原因**：嘗試從 sandboxed session 啟動 ACP
- **處理方式**：ACP 目前只能在 host runtime 運行，請使用 `runtime="subagent"` 或從非 sandbox session 啟動

### 症狀：`Permission prompt unavailable in non-interactive mode`
- **可能原因**：`permissionMode` 設定過於嚴格
- **處理方式**：
  ```bash
  # 允許所有操作（開發環境）
  openclaw config set plugins.entries.acpx.config.permissionMode approve-all
  
  # 或設定優雅降級
  openclaw config set plugins.entries.acpx.config.nonInteractivePermissions deny
  ```

## 總結

### ACP 的價值

1. **統一介面**：一套代碼支援多個 AI 代理
2. **標準協議**：不用自己處理 PTY 抓取
3. **靈活整合**：可以從 IDE、聊天、CLI 使用
4. **權限控制**：細緻的權限管理
5. **Session 管理**：持久化對話、平行工作流

### 關鍵概念

- **ACP** = 協議標準（像 HTTP）
- **ACPX** = OpenClaw 的 ACP 實作引擎
- **Agents** = 實際執行編碼任務的 AI 工具（Codex/Claude/Gemini/Pi）

## 參考資源

- ACP 官方網站：https://agentclientprotocol.com/
- ACPX npm：https://www.npmjs.com/package/acpx
- OpenClaw ACP 文檔：https://docs.openclaw.ai/tools/acp-agents
- OpenClaw 配置參考：https://docs.openclaw.ai/gateway/configuration-reference
