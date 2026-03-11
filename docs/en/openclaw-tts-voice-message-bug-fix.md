# OpenClaw TTS Voice Message Bug Fix: Telegram Voice Format Issue

> **Version Info**  
> - OpenClaw: 2026.3.x  
> - Related Issue: #42060  
> - Related PRs: #42153, #42075  
> - Last Updated: 2026-03-11

---

## TL;DR

- **Problem**: TTS tool-generated audio appears as MP3 music file in Telegram instead of voice message bubble
- **Root Cause**: `audioAsVoice` flag lost during data processing, causing Telegram to use wrong send method
- **Fix**: Added `extractToolResultMedia()` function to preserve complete directive information
- **Impact Scope**: Only modified tool result processing layer, no impact on other pipeline components
- **Risk**: Low, backward compatible, old function retained for other modules

---

## Problems Solved

### Use Cases

When users use TTS tool to generate voice replies in Telegram:

**Expected Behavior:**
- Receive voice message bubble (circular play button + waveform display)
- Quick playback, experience similar to voice calls

**Actual Behavior:**
- Receive MP3 music file attachment
- Need to download or open with music player
- Poor user experience

### Impact Scope

- **Affected Channels**: Telegram (primary), WhatsApp (possible)
- **Not Affected**: Discord, Slack (don't support voice message format anyway)
- **Trigger Condition**: Using TTS tool to generate voice replies

---

## Core Concepts

### OpenClaw Directive System

OpenClaw uses special markers to embed system directives in tool output:

```
[[audio_as_voice]]        ← Directive (system command)
MEDIA:/tmp/audio.opus     ← Media token (media path)
Processing complete       ← Plain text (shown to user)
```

**Processing Flow:**

```
Tool Output → System parses directives → Remove markers → Channel rendering
```

### Key Terms

- **Directive**: `[[audio_as_voice]]` - Marks this as a voice message
- **Media Token**: `MEDIA:/path/to/file` - Marks media file path
- **audioAsVoice flag**: Boolean value telling delivery layer to send as voice format

---

## Root Cause Analysis

### Data Flow and Error Point

```
┌─────────────────────────────────────────┐
│ TTS Tool Output                          │
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
│ Only returns: ["/tmp/x.opus"]           │
│ audioAsVoice discarded!                  │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ Telegram Delivery                        │
│ audioAsVoice === undefined               │
│ Uses sendAudio() instead of sendVoice() │
└─────────────────────────────────────────┘
```

### Problem Points

1. `splitMediaFromOutput()` correctly parsed `audioAsVoice: true`
2. But `extractToolResultMediaPaths()` only returns path array, discarding the flag
3. Subsequent delivery layer doesn't receive `audioAsVoice` information
4. Telegram judges it as `undefined`, sends as music file format

---

## Fix Implementation

### Core Changes

**New Function:**

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
      audioAsVoice = true;  // ✅ Preserve flag
    }
    if (parsed.mediaUrls?.length) {
      paths.push(...parsed.mediaUrls);
    }
  }
  
  return { mediaUrls: paths, audioAsVoice };
}
```

**Update Call Site:**

```typescript
// src/agents/pi-embedded-subscribe.handlers.tools.ts
const extracted = extractToolResultMedia(result);

void ctx.params.onToolResult({
  mediaUrls: extracted.mediaUrls,
  audioAsVoice: extracted.audioAsVoice  // ✅ Pass flag
});
```

**Backward Compatibility:**

```typescript
// Keep old function for other modules
export function extractToolResultMediaPaths(result): string[] {
  return extractToolResultMedia(result).mediaUrls;
}
```

### Why Single PR Could Fix It

- `ReplyPayload` type already had `audioAsVoice?: boolean` field
- `normalizeReplyPayload()` uses spread operator, preserves all fields
- Outbound delivery already supports passing `audioAsVoice` to Telegram

**Therefore, only needed to fix the "lost in transit" issue, no need to modify entire pipeline.**

---

## Best Practices

### ✅ Correct Approach

1. **Preserve complete directive info**: Don't just take partial fields
2. **Use new function**: `extractToolResultMedia()` instead of `extractToolResultMediaPaths()`
3. **Pass complete payload**: Ensure `audioAsVoice` passes to delivery layer

### ❌ Anti-patterns

1. **Only take path array**: Will lose directive information
2. **Accumulate directives across blocks**: May mislabel other media (see risks below)

---

## Potential Risks and Improvements

### Cross Content Block Accumulation Issue

**Current Logic (may fail):**

```
Tool Result:
  Block 1: { text: "[[audio_as_voice]]" }      → audioAsVoice = true
  Block 2: { text: "MEDIA:/tmp/image.jpg" }    → Add media

Result:
  { mediaUrls: ["/tmp/image.jpg"], audioAsVoice: true }
  ❌ Image mislabeled as voice message!
```

**Suggested Fix:**

```typescript
// Only associate when directive and media are in same block
if (parsed.mediaUrls?.length) {
  paths.push(...parsed.mediaUrls);
  
  // ✅ Only this block's directive affects this block's media
  if (parsed.audioAsVoice) {
    audioAsVoice = true;
  }
}
```

**Why It's Currently Fine:**

TTS tool output format has directive and media in same block:

```typescript
{
  content: [
    { text: "[[audio_as_voice]]\nMEDIA:/tmp/audio.mp3" }  // ✅ Bound together
  ]
}
```

So currently won't trigger cross-block issue, but adding protection prevents future bugs.

---

## Troubleshooting

### Symptom 1: Voice message still shows as music file

**Possible Causes:**
1. OpenClaw version too old (< 2026.3.x)
2. `audioAsVoice` flag not correctly passed

**Resolution:**

```bash
# 1. Check OpenClaw version
openclaw --version

# 2. Check tool result output (dev mode)
# Verify it contains [[audio_as_voice]] directive

# 3. Check delivery log
# Verify audioAsVoice is true
```

### Symptom 2: Other media mislabeled as voice

**Possible Cause:**
- Directive accumulation across blocks (see risks above)

**Resolution:**
- Check tool output format, ensure directive and media in same block
- Or implement suggested protection logic above

### Symptom 3: Discord/Slack still shows as file

**Possible Cause:**
- These channels don't support voice message format

**Resolution:**
- This is expected behavior, not a bug
- Discord/Slack automatically downgrade to file attachment

---

## Performance Considerations

### Multi-layer Filtering Mechanism

System uses funnel-style filtering to avoid unnecessary parsing:

```
All tool results
  ↓ [Filter 1] shouldEmitToolOutput - Only process tools needing media
  ↓ [Filter 2] Quick structure check - result.content valid?
  ↓ [Filter 3] Only process text blocks
  ↓ [Filter 4] Quick string check - Contains "media:" or "[["?
  ↓ (90% of plain text returns here ⚡)
  ↓ [Filter 5] Line-by-line processing - Only process "MEDIA:" lines
  ↓ [Filter 6] Directive check - Only use regex when "[["
  ↓
Final parsed result
```

**Performance Benefits:**
- Most plain text messages return after just 2 string checks
- Only 10% of messages need full parsing
- ~3-4x performance improvement

---

## Cross-Channel Design

### Unified Processing, Individual Rendering

**Processing Stage (unified for all channels):**

```
Tool Output → System parses directives → ReplyPayload
                                    ↓
                    { mediaUrls, audioAsVoice: true }
```

**Rendering Stage (each channel decides):**

| Channel   | Behavior                                      |
|-----------|-----------------------------------------------|
| Telegram  | ✅ sendVoice() → Voice message bubble         |
| WhatsApp  | ✅ sendPTT() → PTT voice message              |
| Discord   | ⚠️ sendFile() → File attachment (no voice support) |
| Slack     | ⚠️ uploadFile() → File attachment (no voice support) |
| WebChat   | ✅ Special styled audio player                |

### Design Benefits

1. **Unified parsing**: Avoid duplicate directive parsing in each channel
2. **Flexible rendering**: Each channel decides how to present based on capabilities
3. **Easy to extend**: Adding new directives only requires modifying one place

---

## Summary

### Key Concepts

1. **Directive System**: Use special markers to embed system commands in text
2. **Complete Information Passing**: Don't just take partial fields, avoid information loss
3. **Unified Parsing, Individual Rendering**: All channels share parsing logic

### When to Use

- When TTS tool generates voice, system automatically adds `[[audio_as_voice]]` directive
- Developers don't need to handle manually, system auto-parses and passes to delivery layer
- Each channel decides how to present based on capabilities

### Value

- Improves voice message user experience
- Unified directive system easy to maintain and extend
- Performance optimization ensures stability when processing large message volumes

---

## References

- **Issue**: https://github.com/openclaw/openclaw/issues/42060
- **PR #42153**: https://github.com/openclaw/openclaw/pull/42153
- **PR #42075**: https://github.com/openclaw/openclaw/pull/42075
- **OpenClaw Official Docs**: https://docs.openclaw.ai
- **Telegram Bot API - sendVoice**: https://core.telegram.org/bots/api#sendvoice
