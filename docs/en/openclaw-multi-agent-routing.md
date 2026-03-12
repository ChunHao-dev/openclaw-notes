# OpenClaw Multi-Agent Routing Architecture: From Telegram Bot to AI Personas

> A comprehensive guide to understanding how OpenClaw connects Telegram Bots, Agents, and Memory.

---

## Core Concepts: Three Key Components

```
Telegram Bot          OpenClaw Agent          Memory (Session)
─────────────         ──────────────          ────────────────
Bot created in        AI persona defined      AI conversation
BotFather             in openclaw.json        history

Has its own token     Has its own model,      Each session key
Can send/receive      system prompt,          corresponds to an
Telegram messages     skills                  independent .jsonl
                                              conversation log
```

---

## Scenario 1: The Simplest Setup — One Bot, One Agent

You have one Telegram Bot, one AI persona, used by yourself only.

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

You (Telegram)
  │
  │  DM @MyBot
  ▼
┌─────────┐       ┌──────────────┐       ┌──────────────┐
│ @MyBot  │ ───→  │ Agent: main  │ ───→  │ Memory .jsonl│
│ (TG Bot)│ ←───  │ (claude)     │       │ Your chat    │
└─────────┘       └──────────────┘       │ history      │
                                          └──────────────┘

No need for accounts, bindings, or dmScope configuration.
Everything works with default settings.
```

---

## Scenario 2: Multi-User — One Bot, Multiple Users

You open your Bot to friends. Need dmScope to isolate memory.

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

You (peerId: 111)          Alice (peerId: 222)
  │                         │
  │  DM @MyBot              │  DM @MyBot
  ▼                         ▼
┌─────────┐              ┌─────────┐
│ @MyBot  │              │ @MyBot  │
└────┬────┘              └────┬────┘
     │                        │
     ▼                        ▼
┌──────────────┐       ┌──────────────┐
│ Agent: main  │       │ Agent: main  │    ← Same Agent
└──────┬───────┘       └──────┬───────┘
       │                      │
       ▼                      ▼
┌──────────────┐       ┌──────────────┐
│ Memory: Your │       │ Memory:      │    ← Separate!
│ direct:111   │       │ Alice's      │
└──────────────┘       │ direct:222   │
                       └──────────────┘

dmScope: "per-peer" gives each user their own memory room.
Your conversations with Alice remain separate.
```

---

## Scenario 3: Multi-Bot, Multi-Agent — Different Bots with Different Souls

You want a coding AI and a cooking AI, each with its own Telegram Bot.

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


You (Telegram)
  │
  ├── DM @CodeBot ──→ ┌──────────────┐    ┌──────────────┐
  │                   │ Agent: coder │───→│ coder memory │
  │                   │ (claude)     │    │ coding chats │
  │                   └──────────────┘    └──────────────┘
  │
  └── DM @CookBot ──→ ┌──────────────┐    ┌──────────────┐
                      │ Agent: chef  │───→│ chef memory  │
                      │ (gpt-4o)     │    │ recipe chats │
                      └──────────────┘    └──────────────┘

Three configurations working together:
  accounts  → Define which Telegram Bots (tokens)
  agents    → Define which AI personas (model, capabilities)
  bindings  → Connect Bots to Agents
```

---

## Scenario 4: Cross-Platform + Memory Merging

You use both Telegram and WhatsApp, and want the AI to remember conversations from both platforms.

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


You (Telegram)  ──→ ┐
                    ├──→  Agent: main  ──→  Memory: direct:andy
You (WhatsApp)  ──→ ┘                       (Merged! Same one)

Without identityLinks:
  Telegram  ──→  Memory: direct:1461779006    ← Separate
  WhatsApp  ──→  Memory: direct:886912345678  ← Separate

identityLinks tells the system:
  "These two accounts are both andy, use the same room"
```

---

## Summary: When to Use Which Configuration

```
Your Need                             Required Configuration
──────────────────────────────────    ──────────────────────────
Single user, single platform, one AI  Nothing, defaults work
Multiple users with same Bot          dmScope: "per-peer"
Multiple Bots for multiple AI         accounts + agents + bindings
Cross-platform shared memory          identityLinks
```
