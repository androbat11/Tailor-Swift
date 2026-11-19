# Tailor-Swift: Research Summary & Decisions

> **Status**: Research complete — ready to build.  
> This document synthesises the three research docs into a single source-of-truth for decisions made before writing any code.

---

## Table of Contents

1. [What We're Building](#1-what-were-building)
2. [ANSI — The Short Version](#2-ansi--the-short-version)
3. [The Core Technical Challenge: The PTY Problem](#3-the-core-technical-challenge-the-pty-problem)
4. [Architecture Decision](#4-architecture-decision)
5. [Crate Stack Decision](#5-crate-stack-decision)
6. [Output Pipeline Decision](#6-output-pipeline-decision)
7. [Project Structure](#7-project-structure)
8. [What We're NOT Doing (and Why)](#8-what-were-not-doing-and-why)
9. [Next Steps](#9-next-steps)

---

## 1. What We're Building

**`tailor-swift`** — A Rust binary that wraps the `claude` CLI to give its terminal output a significantly better UX:

| Pain today | After tailor-swift |
|---|---|
| All output looks the same shade | Headers are highlighted in bold color |
| Markdown headings (`# Foo`) render as plain text | Rendered as big, bold, colored banners |
| Bullet points (`- item`) are hard to scan | Styled with Unicode symbols and indentation |
| Code blocks blend into prose | Distinct background color / border |
| Hard to tell where one answer ends and the next begins | Clear visual separators |

The user runs `tailor-swift [claude args...]` exactly as they'd run `claude [args...]`. Everything else is transparent.

---

## 2. ANSI — The Short Version

> See [`01_ansi_fundamentals.md`](./01_ansi_fundamentals.md) for the full deep-dive.

**ANSI escape codes** are special byte sequences that terminals understand as formatting instructions — not text to display. They start with `ESC` (byte `0x1B`, written `\x1b` in Rust strings) followed by `[`, then parameters and a command letter.

```
\x1b[1m        → bold ON
\x1b[0m        → reset ALL formatting
\x1b[32m       → foreground GREEN (3-bit)
\x1b[1;33m     → bold + foreground YELLOW
\x1b[38;2;255;100;0m  → TrueColor RGB orange (24-bit)
```

That's it at the basic level. When you write those bytes to a terminal, it acts on them silently — the user sees colored/bold text, not the escape sequence. This is how Claude's output has colors today, and it's exactly the mechanism we'll use to add *our* formatting layer.

**Key insight for this project**: Claude's output is a stream of raw bytes containing *both* regular text characters *and* embedded ANSI sequences. Our tool needs to:
1. Read that raw byte stream
2. Understand which parts are ANSI (and what they mean)
3. Strip or selectively keep Claude's own ANSI
4. Inject our own ANSI sequences for better formatting

---

## 3. The Core Technical Challenge: The PTY Problem

> See [`02_rust_ecosystem.md § The PTY Problem`](./02_rust_ecosystem.md) for the full explanation.

This is the **most important thing to understand** before building the wrapper.

### Why a simple pipe won't work

When a Unix process starts, it checks whether its stdin/stdout are connected to a **real terminal** or a **pipe/file**. This check is called `isatty()`. Claude Code (like most modern CLI tools) uses this to decide how to behave:

| Connected to… | Claude behaviour |
|---|---|
| A real TTY | Full colors, interactive mode, streaming output, nice formatting |
| A pipe (`|`) | May disable colors, buffer all output until done, change formatting |

If you do `claude | my-wrapper`, Claude sees a pipe and potentially degrades its output — exactly the opposite of what we want.

### The Solution: PTY (Pseudo-Terminal)

A **PTY (Pseudo-Terminal)** is a kernel-level mechanism that creates a fake terminal. It has two ends:
- **Master** — your program reads/writes here
- **Slave** — the child process (Claude) is started with this as its stdin/stdout/stderr

From Claude's perspective, it's talking to a real terminal. `isatty()` returns `true`. Full colors, full interactivity, real-time streaming. Meanwhile, **you** (tailor-swift) hold the master end and can read everything Claude writes before it reaches the screen.

```
User's real TTY
     │  ↑
     │  │  (tailor-swift renders here)
     ▼  │
 ┌──────────────────┐
 │   tailor-swift   │  ← holds PTY master
 └──────────────────┘
     │  ↑
     │  │  PTY (fake terminal — Claude thinks this is real)
     ▼  │
 ┌──────────────────┐
 │      claude      │  ← connected to PTY slave; isatty() = true ✓
 └──────────────────┘
```

This is the same technique used by `tmux`, `screen`, `ssh`, and terminal emulators themselves.

---

## 4. Architecture Decision

**Decision: Architecture 2 — PTY Wrapper** ✅

> See [`03_architecture.md`](./03_architecture.md) for all five architectures evaluated.

### Why PTY Wrapper wins

| Criterion | Pipe | **PTY** | Pipe Filter | Shell alias |
|---|---|---|---|---|
| Claude sees a TTY (isatty = true) | ❌ | ✅ | ❌ | ⚠️ |
| Fully interactive (user can type) | ⚠️ | ✅ | ❌ | ⚠️ |
| Real-time streaming output | ✅ | ✅ | ✅ | ✅ |
| Preserves Claude's own colors | ❌ | ✅ | ❌ | ⚠️ |
| Works transparently (same invocation) | ✅ | ✅ | ❌ | ⚠️ |
| Complexity | Low | Medium | Low | High |

**Fallback**: If PTY is unavailable (e.g., Windows without ConPTY), fall back to pipe mode automatically.

### How it works at runtime

```
main()
  │
  ├─ parse CLI flags
  ├─ load config (~/.config/tailor-swift/config.toml)
  │
  ├─ open PTY pair (master fd + slave fd)
  ├─ spawn `claude [args...]` with slave as its stdio
  │
  ├─ put user's terminal into raw mode
  │
  ├─ [Thread A] stdin relay:
  │     read from user's keyboard → write to PTY master
  │     (this sends keystrokes to Claude)
  │
  ├─ [Thread B] output pipeline:
  │     read raw bytes ← PTY master
  │       │
  │       ▼  vte (ANSI parser)
  │     AnsiEvent stream
  │       │
  │       ▼  line assembler
  │     complete lines (plain text, with semantic ANSI info)
  │       │
  │       ▼  state machine (markdown-aware)
  │     tokens: Heading | Bullet | Code | Prose | ...
  │       │
  │       ▼  renderer (applies our ANSI styling)
  │     formatted bytes → write to user's terminal
  │
  ├─ [Thread C] signal handler:
  │     SIGWINCH (terminal resize) → resize PTY to match
  │     SIGINT  → forward to Claude
  │
  └─ wait for Claude to exit → restore terminal → exit with same code
```

---

## 5. Crate Stack Decision

> See [`02_rust_ecosystem.md`](./02_rust_ecosystem.md) for full per-crate analysis.

### Core Stack (MVP)

```toml
[dependencies]
# PTY — spawn Claude in a fake terminal so isatty() returns true
portable-pty = "0.9"

# Async runtime — bidirectional relay (stdin→PTY, PTY→stdout)
tokio = { version = "1", features = ["full"] }

# ANSI parsing — understand Claude's existing escape sequences
vte = "0.15"
strip-ansi-escapes = "0.2"

# Terminal control — raw mode, cross-platform
crossterm = "0.29"

# Output styling — write our own beautiful ANSI
owo-colors = "4"

# Markdown rendering — Claude outputs lots of markdown
termimad = "0.35"

# CLI + Config
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
toml = "1"

# Error handling
anyhow = "1"
thiserror = "1"
```

### Crate Rationale (brief)

| Crate | Why this one |
|---|---|
| `portable-pty` | Used by WezTerm; cross-platform (Linux, macOS, Windows ConPTY); high-level API; avoids raw `nix` syscalls |
| `tokio` | Best async runtime; `select!` macro makes the bidirectional relay loop clean |
| `vte` | The ANSI parser used by Alacritty (the fastest terminal emulator); handles partial sequences from streaming correctly |
| `crossterm` | Cross-platform raw mode + cursor control; pairs with portable-pty cleanly |
| `owo-colors` | Zero-allocation, ergonomic; works at compile time for static strings |
| `termimad` | Renders markdown to terminal ANSI out-of-the-box; great for rapid prototyping of the renderer |
| `clap` | The standard; derive macros keep boilerplate minimal |

### Future / Optional

| Crate | When to add |
|---|---|
| `pulldown-cmark` | If you want a fully custom markdown renderer (more control than termimad) |
| `ratatui` | If you want a full TUI layout (split panes, sidebars) — probably overkill for v1 |
| `syntect` | Syntax highlighting inside code blocks |
| `signal-hook` | More robust Unix signal handling |

---

## 6. Output Pipeline Decision

**Decision: Hybrid State Machine** ✅

For real-time streaming of Claude's output, we use a three-stage pipeline:

```
raw bytes
    │
    ▼
[vte parser]            → handles ANSI sequences; extracts plain text chars
    │                      never confused by partial sequences mid-stream
    ▼
[line assembler]        → accumulates chars into complete lines
    │                      buffers until \n; passes through partial lines on timeout
    ▼
[state machine]         → tracks context: Prose / InCodeBlock / InList / InHeading
    │                      makes markdown-aware decisions per line
    ▼
[renderer]              → maps semantic tokens to styled ANSI output
    │                      uses owo-colors + raw sequences
    ▼
user's terminal
```

### Why not the simpler options?

- **Regex only**: Can't track multi-line context. A `# comment` inside a code block would be mis-styled as a heading. Quick for a proof-of-concept, wrong for production.
- **Full markdown parse + buffer**: Requires buffering the *entire* response before rendering. Defeats real-time streaming — the user would see nothing until Claude finishes. Not viable for interactive use.
- **State machine**: Right balance. Processes line-by-line in real time, but tracks context across lines to avoid mis-classification.

### What the state machine tracks

```
States:
  Prose         ← default
  InCodeBlock   ← after opening ``` fence (knows the language tag)
  InList        ← after first bullet; tracks indent level
  InHeading     ← single-line; immediately returns to Prose
  InBlockquote  ← after > prefix

Transitions:
  Prose + "^#{1,6} "   →  InHeading  →  Prose
  Prose + "^```"       →  InCodeBlock
  InCodeBlock + "^```" →  Prose
  Prose + "^[-*+] "    →  InList
  InList + blank line  →  Prose
  Prose + "^> "        →  InBlockquote
  InBlockquote + no >  →  Prose
```

---

## 7. Project Structure

```
tailor-swift/
├── Cargo.toml
├── Cargo.lock
├── README.md
│
├── src/
│   ├── main.rs              # Entry: CLI parsing, spawn strategy, wire-up
│   ├── cli.rs               # Clap CLI: --ts-* flags separated from claude passthrough args
│   │
│   ├── pty/
│   │   ├── mod.rs           # Public API: spawn_in_pty(), PtyHandle
│   │   ├── unix.rs          # Unix PTY via portable-pty
│   │   └── windows.rs       # ConPTY fallback
│   │
│   ├── pipe.rs              # Fallback pipe spawner (--no-pty flag or Windows w/o ConPTY)
│   │
│   ├── parser/
│   │   ├── mod.rs           # OutputParser, ParsedEvent enum
│   │   ├── ansi.rs          # Wraps vte; produces AnsiEvent stream
│   │   ├── state_machine.rs # Markdown-aware state machine; produces ParsedToken
│   │   └── line_buffer.rs   # Accumulates streaming bytes into lines
│   │
│   ├── renderer/
│   │   ├── mod.rs           # Renderer trait; dispatch to theme
│   │   ├── default.rs       # Default theme rendering logic
│   │   ├── themes.rs        # Theme data structures + loader
│   │   └── ansi_writer.rs   # Low-level helpers: bold(), fg_rgb(), reset()
│   │
│   ├── config.rs            # Load ~/.config/tailor-swift/config.toml
│   ├── signals.rs           # SIGWINCH → PTY resize; SIGINT → forward to child
│   └── error.rs             # TailorError with thiserror
│
├── themes/
│   ├── default.toml         # Built-in default theme
│   └── minimal.toml         # Minimal/plain theme
│
├── notes/
│   └── research/            # ← you are here
│
└── tests/
    ├── integration/
    │   ├── pipe_test.rs
    │   └── parser_test.rs
    └── fixtures/
        └── claude_output_*.txt   # Recorded outputs for regression tests
```

---

## 8. What We're NOT Doing (and Why)

| Option | Rejected because |
|---|---|
| Simple pipe (`claude \| tailor-swift`) | Claude detects non-TTY; degrades output; loses interactivity |
| Shell `script` command hack | Platform-specific; fragile; complex exit code handling |
| Named pipe / FIFO | No isatty() benefit; more complex than pipe; no real advantage |
| `ratatui` full TUI | Too heavy for v1; solves a problem we don't have yet |
| `termion` instead of `crossterm` | Unix-only; crossterm handles macOS + Windows too |
| `async-std` instead of `tokio` | Smaller ecosystem; `portable-pty` integrates better with tokio |
| Full markdown buffering | Breaks real-time streaming; unacceptable UX delay |

---

## 9. Next Steps

1. **`cargo new tailor-swift`** — initialise the Cargo project inside this workspace
2. **Scaffold modules** — create the directory structure above with empty `mod.rs` files
3. **PTY proof of concept** — `portable-pty` + spawn `echo "hello"` inside a PTY; confirm bytes flow through
4. **Parser proof of concept** — feed sample Claude output through `vte` + state machine; verify heading/bullet/code detection
5. **Renderer v1** — implement default theme: bold headers, indented bullets, code block borders
6. **Wire everything together** — connect PTY → parser → renderer → stdout
7. **Signal handling** — SIGWINCH resize + SIGINT forwarding
8. **Config file** — TOML theme loading
9. **Polish** — passthrough mode, error messages, `--help`

> **Recommended reading order before coding:**
> 1. [`01_ansi_fundamentals.md`](./01_ansi_fundamentals.md) — understand the byte layer
> 2. [`02_rust_ecosystem.md § The PTY Problem`](./02_rust_ecosystem.md) — understand why PTY is mandatory
> 3. [`03_architecture.md § Architecture 2`](./03_architecture.md) — understand the runtime design
> 4. [`03_architecture.md § Output Parsing Strategy B`](./03_architecture.md) — understand the state machine

---

*Research conducted: 2026-11-19 | Stack: Rust | Architecture: PTY Wrapper + Hybrid State Machine*
