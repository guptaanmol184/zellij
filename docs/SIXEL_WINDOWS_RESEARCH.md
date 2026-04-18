# Sixel Rendering on Windows: Research & Root Cause Analysis

## Problem Statement

Sixel images do not render inside Zellij on Windows Terminal. Running:
```
chafa --format=sixel .\hello.jpg
```
inside a Zellij pane produces no visible output, despite Windows Terminal v1.22+ supporting sixel natively.

---

## Executive Summary

**Root Cause**: ConPTY (the Windows pseudo-terminal layer) **strips DCS (Device Control String) sequences by design**. Sixel images are transmitted via DCS sequences (`ESC P ... ESC \`). When chafa runs inside Zellij, the sixel data passes through Zellij's per-pane ConPTY, which discards the DCS content before it ever reaches Zellij's server process.

**Key Finding**: The `PSEUDOCONSOLE_PASSTHROUGH_MODE` flag that was suggested by multiple AI-powered web searches **does not exist** in the Windows API. This was confirmed by reading the actual Microsoft Terminal source code.

**Status**: This is a **fundamental platform limitation**, not a Zellij bug. No configuration change or flag can fix it with current Windows APIs.

---

## Investigation Timeline

### Phase 1: Pipeline Tracing

Traced the full sixel pipeline through Zellij's code:

```
chafa → ConPTY (per-pane) → named pipe → server async reader (terminal_bytes.rs)
  → VTE parser → grid.rs:hook() [DCS 'q']
  → GATEKEEPER: character_cell_size must be Some (line ~3348)
  → SixelGrid.start_image() → put() bytes → unhook() → create_sixel_image()
  → SixelImageStore → output/mod.rs serializes VTE → client stdout → Windows Terminal
```

### Phase 2: Diagnostic Instrumentation

Added 5 diagnostic checkpoints using `log::info!` with `[SIXEL]` prefix:

| CP | Location | File | What it logs |
|----|----------|------|-------------|
| CP1 | ConPTY creation | `os_input_output_windows.rs` | Passthrough mode success/fallback |
| CP2 | Byte stream reader | `terminal_bytes.rs` | DCS byte detection, hex preview |
| CP3 | VTE parser hook | `grid.rs` | Whether `hook()` fires for DCS 'q' |
| CP4 | Sixel image creation | `grid.rs` | Whether `create_sixel_image()` is reached |
| CP5 | Output serialization | `output/mod.rs` | Whether sixel VTE data reaches client |

**Result**: Only CP1 ever fired. CP2-CP5 never triggered, confirming DCS bytes never reach the server.

### Phase 3: Chafa Output Verification

Captured raw chafa output outside Zellij:
```powershell
chafa --format=sixel .\hello.jpg > sixel-raw.bin
```
- File size: 426,707 bytes
- Valid DCS header confirmed: `ESC P 0;1;0q` (bytes `1B 50 30 3B 31 3B 30 71`)
- **Conclusion**: chafa produces valid sixel; the data is lost in ConPTY.

### Phase 4: Passthrough Mode Attempt (Failed)

Attempted to enable ConPTY passthrough by adding flags to `CreatePseudoConsole`:

1. **Flag 0x8** — Ran, "succeeded" (S_OK returned), but no DCS passed through
2. **Flag 0x2** — Same result: S_OK returned, no DCS passed through

Both flags were silently accepted because `CreatePseudoConsole` ignores unknown bits.

### Phase 5: Source Code Verification (The Critical Discovery)

Fetched the actual Microsoft Terminal source code from `src/inc/conpty-static.h`:

```c
#define PSEUDOCONSOLE_INHERIT_CURSOR (0x1)

// Opt into the graphics widths mode for ConPTY
#define PSEUDOCONSOLE_GLYPH_WIDTH_GRAPHEMES (0x08)
#define PSEUDOCONSOLE_GLYPH_WIDTH_WCSWIDTH (0x10)
#define PSEUDOCONSOLE_GLYPH_WIDTH_CONSOLE (0x18)
#define PSEUDOCONSOLE_AMBIGUOUS_IS_WIDE (0x20)
```

**There is NO `PSEUDOCONSOLE_PASSTHROUGH_MODE` flag.** The entire concept was fabricated by AI web search results. The flags we tried mapped to:
- `0x8` = `PSEUDOCONSOLE_GLYPH_WIDTH_GRAPHEMES` (glyph width setting, unrelated)
- `0x2` = undefined bit (silently ignored)

---

## Root Cause: ConPTY DCS Stripping

### How ConPTY Handles Escape Sequences

ConPTY processes all VT sequences internally through its own state machine:

| Sequence Type | ConPTY Behavior | Example |
|--------------|-----------------|---------|
| **CSI** (Control Sequence Introducer) | Processed and re-emitted | Cursor movement, colors |
| **OSC** (Operating System Command) | **Passed through** even if unrecognized | Terminal title, clipboard |
| **DCS** (Device Control String) | **STRIPPED** unless specifically recognized | Sixel, DECRQSS, tmux passthrough |
| **APC** (Application Program Command) | **STRIPPED** | Kitty graphics protocol |

This is confirmed by multiple Microsoft Terminal issues:
- [#17313](https://github.com/microsoft/terminal/issues/17313) — DCS sequences not dispatched to hosting terminal
- [#8325](https://github.com/microsoft/terminal/issues/8325) — ConPTY tracking DCS passthrough
- [#12064](https://github.com/microsoft/terminal/issues/12064) — DCS filtering details
- [#15010](https://github.com/microsoft/terminal/issues/15010) — Recent discussions on DCS passthrough

### Why OSC Passes Through But DCS Doesn't

ConPTY has an internal mechanism (`_pfnFlushToTerminal`) that forwards unrecognized OSC sequences to the hosting terminal. This was added for compatibility with features like terminal title setting. However, **DCS sequences do not have this forwarding mechanism** — they are consumed and discarded by ConPTY's internal VT state machine.

### The Pipe Architecture

```
┌──────────┐    ┌────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  chafa   │ →  │  ConPTY    │ →  │  Named Pipe      │ →  │  Zellij Server  │
│ (child)  │    │ (per-pane) │    │  (async reader)  │    │  (VTE parser)   │
└──────────┘    └────────────┘    └──────────────────┘    └─────────────────┘
                      ↑
                 DCS stripped here
                 before reaching pipe
```

Pipe name format: `\\.\pipe\zellij-conpty-{terminal_id}-{seq}`

The pipe itself is fine — it's a raw byte stream. The problem is upstream: ConPTY never writes DCS bytes to the pipe.

---

## Terminal Image Protocols Comparison

### 1. Sixel (DCS-based) — BLOCKED by ConPTY

```
ESC P <params> q <pixel data> ESC \
```

- **Sequence type**: DCS (Device Control String)
- **ConPTY status**: ❌ STRIPPED
- **Windows Terminal support**: ✅ Native since v1.22 (Aug 2024) — but only for direct child processes, not through ConPTY
- **Tools**: chafa, libsixel, img2sixel

### 2. iTerm2 Inline Images (OSC-based) — POTENTIALLY viable

```
ESC ]1337;File=name=<base64name>;size=<bytes>;inline=1:<base64data> BEL
```

- **Sequence type**: OSC (Operating System Command)
- **ConPTY status**: ✅ PASSED THROUGH (unrecognized OSC sequences are forwarded)
- **Windows Terminal support**: ❌ Not implemented (as of early 2025)
- **Caveat**: Even if ConPTY passes it through, WT doesn't render it. Would need WT to add support.
- **Tools**: imgcat (iTerm2), some chafa versions

### 3. Kitty Graphics Protocol (APC-based) — BLOCKED by ConPTY

```
ESC _G <params>;<base64data> ESC \
```

- **Sequence type**: APC (Application Program Command) — `ESC _`
- **ConPTY status**: ❌ STRIPPED (APC sequences are also consumed by ConPTY)
- **Windows Terminal support**: ❌ Not implemented
- **Tools**: kitty's built-in, some chafa versions, timg

### 4. Summary Table

| Protocol | Escape Type | ConPTY Passes? | WT Renders? | Viable for Zellij? |
|----------|------------|----------------|-------------|---------------------|
| Sixel | DCS (`ESC P`) | ❌ No | ✅ Yes (v1.22+) | ❌ Blocked |
| iTerm2 | OSC (`ESC ]1337`) | ✅ Yes | ❌ No | ⚠️ Future possibility |
| Kitty | APC (`ESC _G`) | ❌ No | ❌ No | ❌ Blocked |

**Bottom line**: No existing terminal image protocol works end-to-end through ConPTY → Zellij → Windows Terminal.

---

## Alternative Approaches

### Tier 1: Workarounds (No Code Changes)

1. **Use WezTerm instead of Windows Terminal**
   - WezTerm renders sixel natively in its own GPU renderer
   - WezTerm doesn't use ConPTY for its own rendering output path
   - Sixel should work inside Zellij when hosted by WezTerm (needs verification)

2. **Run graphics commands outside Zellij**
   - `chafa --format=sixel .\hello.jpg` works directly in Windows Terminal
   - Use Zellij for text workflows, WT directly for image viewing

### Tier 2: Engineering Approaches (Significant Work)

3. **Wait for Microsoft ConPTY DCS passthrough**
   - Microsoft Terminal issues #8325, #12064, #17313 track this request
   - If implemented, sixel would work in Zellij automatically (our code already handles it)
   - Timeline: unknown, no committed plan from Microsoft

4. **OSC-based image side-channel**
   - Since ConPTY passes OSC sequences, could theoretically:
     - Detect sixel in child process output (before ConPTY)
     - Re-encode as OSC and forward alongside normal output
   - **Problem**: Would need Windows Terminal to implement OSC image rendering
   - **Problem**: Ordering of OSC sequences relative to text output is not guaranteed through ConPTY

5. **Bypass ConPTY for specific panes**
   - For panes that need graphics, use raw pipes instead of ConPTY
   - Implement own VT parsing for those panes (mini terminal emulator)
   - **Extremely complex** — would need to handle all of: cursor movement, colors, scrolling, etc.

6. **Dual-pipe architecture**
   - Keep ConPTY for normal terminal emulation
   - Add a second raw pipe/shared memory to capture unprocessed child output
   - Detect DCS sequences in the raw stream and forward them
   - **Complex** — synchronization between ConPTY-processed and raw streams is difficult

### Tier 3: Protocol Changes (Ecosystem-wide)

7. **Lobby for Windows Terminal iTerm2/OSC image support**
   - If WT supported OSC 1337 (iTerm2 protocol), and ConPTY already passes OSC through...
   - This would enable image display through the entire chain
   - Would need both WT rendering support AND Zellij OSC forwarding

8. **Contribute ConPTY DCS passthrough to microsoft/terminal**
   - Most impactful fix — would benefit ALL terminal multiplexers on Windows
   - Requires understanding ConPTY internals and getting PR accepted

---

## Files Modified During Investigation

### Changed (on branch `sixel-windows-fix`)
| File | Changes | Status |
|------|---------|--------|
| `zellij-server/src/os_input_output_windows.rs` | Added fake passthrough constant + fallback logic, CP1 logging | **Needs revert** |
| `zellij-server/src/terminal_bytes.rs` | DCS detection + large-chunk logging (CP2) | Diagnostic, can keep or remove |
| `zellij-server/src/panes/grid.rs` | CP3 (hook) + CP4 (create_sixel_image) logging | Diagnostic, can keep or remove |
| `zellij-server/src/output/mod.rs` | CP5 (sixel VTE emission) logging | Diagnostic, can keep or remove |

### Created (untracked)
| File | Purpose |
|------|---------|
| `hello.jpg` | Test image for sixel rendering |
| `sixel-raw.bin` | Raw chafa sixel output (426,707 bytes, valid DCS) |
| `sixel-debug.log` | Debug output from test sessions |

---

## Key Technical References

- **ConPTY source**: `microsoft/terminal` repo, `src/inc/conpty-static.h`
- **CreatePseudoConsole docs**: [Microsoft Learn](https://learn.microsoft.com/en-us/windows/console/createpseudoconsole)
- **ConPTY DCS issue**: [microsoft/terminal#17313](https://github.com/microsoft/terminal/issues/17313)
- **ConPTY DCS tracking**: [microsoft/terminal#8325](https://github.com/microsoft/terminal/issues/8325)
- **Zellij sixel issue**: [zellij-org/zellij#964](https://github.com/zellij-org/zellij/issues/964)
- **WT sixel support**: [Windows Terminal v1.22 release notes](https://devblogs.microsoft.com/commandline/)
- **Sixel spec**: DEC DRCS standard, DCS `q` introducer
- **iTerm2 image protocol**: [iterm2.com/documentation-images.html](https://iterm2.com/documentation-images.html)
- **Kitty graphics protocol**: [sw.kovidgoyal.net/kitty/graphics-protocol](https://sw.kovidgoyal.net/kitty/graphics-protocol/)

---

## Zellij Logging Reference

```powershell
# View sixel-related logs in real-time
Get-Content "$env:TEMP\zellij\zellij-log\zellij.log" -Wait | Select-String "SIXEL"

# Start test session
$env:TERM = "xterm-256color"
.\target\dev-opt\zellij --session sixel-test

# Inside Zellij pane:
chafa --format=sixel .\hello.jpg
```

- **Log framework**: log4rs (`zellij-utils/src/logging.rs`)
- **Default level**: `LevelFilter::Info` (line 92)
- **Log file**: `$TEMP\zellij\zellij-log\zellij.log`

---

## Conclusion

Sixel rendering inside terminal multiplexers on Windows is a **known, unresolved platform limitation** of ConPTY. The fix must come from Microsoft (ConPTY DCS passthrough support) or from the ecosystem adopting an OSC-based image protocol that ConPTY already forwards. No amount of Zellij-side code changes can work around ConPTY's DCS stripping behavior.

*Last updated: Investigation on branch `sixel-windows-fix`, April 2025*
