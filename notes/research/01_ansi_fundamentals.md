# ANSI Escape Codes & Terminal Output: A Comprehensive Reference

> **Context**: This document is written for someone building a **Claude Code terminal wrapper in Rust**. It covers ANSI/VT100 escape codes from first principles through practical implementation details.

---

## Table of Contents

1. [History & Why They Exist](#1-history--why-they-exist)
2. [How They Work: The Byte Sequences](#2-how-they-work-the-byte-sequences)
3. [Colors](#3-colors)
4. [Text Formatting](#4-text-formatting)
5. [Cursor Movement](#5-cursor-movement)
6. [Erase Commands](#6-erase-commands)
7. [Hyperlinks (OSC 8)](#7-hyperlinks-osc-8)
8. [How Terminal Emulators Parse ANSI (State Machine)](#8-how-terminal-emulators-parse-ansi-state-machine)
9. [Raw ANSI vs. Using a Library](#9-raw-ansi-vs-using-a-library)
10. [Streaming Output & The Interception Challenge](#10-streaming-output--the-interception-challenge)
11. [Practical Rust Relevance](#11-practical-rust-relevance)

---

## 1. History & Why They Exist

### The Problem: Dumb Terminals

In the early days of computing (1960s–70s), **terminals** were physical hardware devices — glass-screen boxes connected via serial cable to a mainframe or minicomputer. Each manufacturer made their own terminal with different, incompatible control sequences. Moving the cursor up on a DEC terminal required a different byte sequence than on an Lear Siegler ADM-3A, which required something different from an IBM 3270.

Writing software that worked across terminals was a nightmare. The classic workaround was **termcap** (terminal capabilities database, 1978), a file that mapped abstract capabilities like "move cursor up" to whatever specific byte sequences a given terminal model needed.

### VT100 and the ANSI Standard

In **1978**, DEC (Digital Equipment Corporation) released the **VT100** terminal. It was the first widely-adopted video terminal to implement the **ANSI X3.64** standard (later absorbed into **ECMA-48** and **ISO 6429**). The VT100 was enormously successful — it became the de facto standard that most terminal emulators still emulate today.

The key insight of the ANSI standard: define a **universal, in-band signaling protocol** using special byte sequences embedded directly in the character stream. No separate control channel needed. The terminal reads the byte stream, and when it sees an **ESC** character followed by specific bytes, it performs a control action instead of printing a character.

### The Legacy Lives On

When physical terminals became obsolete, **terminal emulators** (xterm, GNOME Terminal, iTerm2, Windows Terminal, etc.) took their place. They all emulate VT100/VT220 behavior because that's what software expects. Today, in 2024, every time you see colored output from `ls`, `git`, `cargo`, or Claude Code, it's using byte sequences invented for a physical terminal from 1978.

---

## 2. How They Work: The Byte Sequences

### The ESC Character

Everything starts with the **ESC** character:
- Decimal: **27**
- Hexadecimal: **0x1B**
- Octal: **033**
- Caret notation: **^[**
- Rust string literal: **`\x1b`** or **`\u{001B}`**

ESC is a non-printing ASCII control character. When a terminal sees it, it stops rendering text and starts interpreting a control sequence.

### The Basic Grammar

ANSI/VT sequences follow a consistent grammar:

```
ESC  [optional intermediate bytes]  final byte
```

There are several **sequence types**, distinguished by the byte(s) immediately after ESC:

| Prefix      | Bytes after ESC | Name                          | Purpose                              |
|-------------|-----------------|-------------------------------|--------------------------------------|
| `ESC [`     | `0x5B`          | **CSI** (Control Sequence Introducer) | Most common: colors, cursor, erase |
| `ESC ]`     | `0x5D`          | **OSC** (Operating System Command)   | Hyperlinks, window title, etc.     |
| `ESC (`     | `0x28`          | Designate character set       | Historical, rarely used today        |
| `ESC P`     | `0x50`          | **DCS** (Device Control String)      | Sixel graphics, etc.               |
| `ESC M`     | `0x4D`          | Reverse Index                 | Scroll up                            |
| `ESC 7`     | `0x37`          | Save cursor position          | DECSC                                |
| `ESC 8`     | `0x38`          | Restore cursor position       | DECRC                                |

### CSI Sequences (The Most Important)

CSI sequences have this structure:

```
ESC [ <parameter bytes> <intermediate bytes> <final byte>
```

- **ESC [** = `\x1b[` = the CSI introducer
- **Parameter bytes**: ASCII digits `0–9` and semicolons `;`, forming numeric parameters
- **Intermediate bytes**: `0x20–0x2F` (space through `/`), rare
- **Final byte**: `0x40–0x7E` (`@` through `~`), defines the command

**Example — Set foreground color to red:**
```
ESC [ 3 1 m
\x1b[31m
```
Breaking it down:
- `\x1b` = ESC
- `[` = CSI introducer
- `31` = parameter (31 = red foreground)
- `m` = final byte, meaning "Select Graphic Rendition" (SGR)

**Example — Move cursor up 5 lines:**
```
ESC [ 5 A
\x1b[5A
```
- `5` = parameter (how many lines)
- `A` = move cursor up

**Multiple parameters** are separated by `;`:
```
ESC [ 1 ; 31 m
\x1b[1;31m
```
Meaning: bold (1) AND red foreground (31).

---

## 3. Colors

### 3-bit / 4-bit Colors (Standard 8/16 Colors)

The original VT100 didn't support color; colors came with VT241 and ANSI terminals in the 1980s. The original standard defined **8 foreground colors** and **8 background colors**, selectable via SGR parameters:

#### SGR Color Parameters

| Effect              | Foreground Code | Background Code |
|---------------------|-----------------|-----------------|
| Black               | `30`            | `40`            |
| Red                 | `31`            | `41`            |
| Green               | `32`            | `42`            |
| Yellow              | `33`            | `43`            |
| Blue                | `34`            | `44`            |
| Magenta             | `35`            | `45`            |
| Cyan                | `36`            | `46`            |
| White               | `37`            | `47`            |
| Default (reset)     | `39`            | `49`            |

The "bright" (or "intense") variants — the 4-bit extension — use codes `90–97` (foreground) and `100–107` (background):

| Effect              | Bright Foreground | Bright Background |
|---------------------|-------------------|-------------------|
| Bright Black (gray) | `90`              | `100`             |
| Bright Red          | `91`              | `101`             |
| Bright Green        | `92`              | `102`             |
| Bright Yellow       | `93`              | `103`             |
| Bright Blue         | `94`              | `104`             |
| Bright Magenta      | `95`              | `105`             |
| Bright Cyan         | `96`              | `106`             |
| Bright White        | `97`              | `107`             |

**Examples:**
```
\x1b[32m    → green foreground
\x1b[41m    → red background
\x1b[1;33m  → bold + yellow foreground
\x1b[0m     → reset ALL formatting
```

> **Note on actual colors**: The exact RGB values are defined by the terminal emulator, not the escape code. "Red" on iTerm2 and "Red" on the Linux console may look different. This is why terminal themes exist.

### 8-bit Colors (256-color palette)

Introduced in xterm around 1999, the **256-color palette** extends the 16 standard colors to 256 entries using a different parameter structure:

```
ESC [ 38 ; 5 ; <n> m    → Set foreground to color #n
ESC [ 48 ; 5 ; <n> m    → Set background to color #n
```

Where `<n>` is `0–255`. The palette is structured as:

- **0–7**: Standard colors (same as 3-bit colors)
- **8–15**: High-intensity/bright colors (same as 4-bit bright)
- **16–231**: 6×6×6 color cube (216 colors)
  - `n = 16 + 36*r + 6*g + b` where r, g, b are each 0–5
- **232–255**: Grayscale ramp from dark to light (24 shades, not counting black/white)

**Examples:**
```
\x1b[38;5;196m   → bright red (color #196 in the cube)
\x1b[48;5;21m    → blue background (color #21)
\x1b[38;5;240m   → medium gray
\x1b[0m          → reset
```

**Color cube formula** — to find color code for RGB values (each 0–5):
```
n = 16 + (36 * r) + (6 * g) + b
```

### 24-bit TrueColor (16.7 million colors)

Modern terminals support full RGB via **TrueColor**, standardized around 2016. The format:

```
ESC [ 38 ; 2 ; <r> ; <g> ; <b> m    → RGB foreground
ESC [ 48 ; 2 ; <r> ; <g> ; <b> m    → RGB background
```

Where `<r>`, `<g>`, `<b>` are each 0–255.

**Examples:**
```
\x1b[38;2;255;100;0m    → orange foreground (R=255, G=100, B=0)
\x1b[48;2;30;30;46m     → dark blue-gray background (Catppuccin Mocha base)
\x1b[38;2;203;166;247m  → lilac text (Catppuccin Mocha mauve)
\x1b[0m                 → reset
```

> **Rust string example for TrueColor:**
> ```rust
> let orange = "\x1b[38;2;255;100;0m";
> let reset  = "\x1b[0m";
> println!("{orange}Hello, world!{reset}");
> ```

> **Terminal support check**: To verify TrueColor support, terminals often set the `COLORTERM` environment variable to `truecolor` or `24bit`. Check this in your Rust code:
> ```rust
> let supports_truecolor = std::env::var("COLORTERM")
>     .map(|v| v == "truecolor" || v == "24bit")
>     .unwrap_or(false);
> ```

---

## 4. Text Formatting

All text formatting is done through **SGR (Select Graphic Rendition)** codes — the same `m` final byte used for colors. Multiple attributes can be combined with semicolons.

### SGR Attribute Codes

| Code  | Effect                  | Reset Code |
|-------|-------------------------|------------|
| `0`   | Reset ALL attributes    | (itself)   |
| `1`   | **Bold** / bright       | `22`       |
| `2`   | Dim / faint             | `22`       |
| `3`   | *Italic*                | `23`       |
| `4`   | Underline               | `24`       |
| `5`   | Slow blink (< 150/min)  | `25`       |
| `6`   | Rapid blink (≥ 150/min) | `25`       |
| `7`   | Reverse video (swap fg/bg) | `27`    |
| `8`   | Concealed / hidden      | `28`       |
| `9`   | ~~Strikethrough~~       | `29`       |
| `21`  | Double underline        | `24`       |
| `53`  | Overline                | `55`       |

> **Note**: Not all terminals support all attributes. Italic, rapid blink, double underline, and overline have spotty support. Bold and standard underline are universally supported. Blinking is often ignored or disabled by user preference.

### Examples

```
\x1b[1m            → bold on
\x1b[22m           → bold off
\x1b[1;3m          → bold AND italic
\x1b[4m            → underline on
\x1b[1;31m         → bold red
\x1b[7m            → reverse video (invert fg/bg)
\x1b[9m            → strikethrough
\x1b[0m            → reset everything to defaults
```

**Combining formatting with color:**
```
\x1b[1;4;38;2;255;215;0m   → bold + underline + gold TrueColor
```

### The Reset Hierarchy

SGR code `0` (or just `\x1b[m` with no parameter, which defaults to 0) resets **everything** — all colors and all formatting attributes at once. Individual attributes can be reset with their specific reset codes (e.g., `22` for bold, `24` for underline).

**Best practice**: Always emit `\x1b[0m` at the end of colored/formatted output to prevent attributes from "bleeding" into subsequent terminal output.

---

## 5. Cursor Movement

Cursor movement uses CSI sequences with different final bytes.

### Basic Directional Movement

| Sequence         | Effect                              |
|------------------|-------------------------------------|
| `\x1b[<n>A`      | Move cursor **up** `n` rows         |
| `\x1b[<n>B`      | Move cursor **down** `n` rows       |
| `\x1b[<n>C`      | Move cursor **right** (forward) `n` columns |
| `\x1b[<n>D`      | Move cursor **left** (backward) `n` columns |
| `\x1b[<n>E`      | Move cursor to **beginning of line**, `n` lines down |
| `\x1b[<n>F`      | Move cursor to **beginning of line**, `n` lines up |
| `\x1b[<n>G`      | Move cursor to **column** `n` (1-based) |

> **Default value**: When `n` is omitted, it defaults to **1** for most commands.

**Examples:**
```
\x1b[A      → up 1 line
\x1b[3A     → up 3 lines
\x1b[10C    → right 10 columns
\x1b[1G     → move to column 1 (beginning of line, same as \r but without newline semantics)
```

### Absolute Positioning

```
\x1b[<row>;<col>H    → Move cursor to row, col (1-based; H = CUP, Cursor Position)
\x1b[<row>;<col>f    → Same as H (HVP, Horizontal Vertical Position — historical alias)
```

**Examples:**
```
\x1b[1;1H    → top-left corner of screen
\x1b[10;5H   → row 10, column 5
\x1b[H       → same as \x1b[1;1H (defaults to row 1, col 1)
```

### Save and Restore Cursor Position

There are two mechanisms, and they're not fully compatible:

**ANSI/SCO style (most compatible):**
```
\x1b[s    → Save cursor position
\x1b[u    → Restore cursor position
```

**DEC private (VT100 style):**
```
\x1b7     → Save cursor position + attributes (DECSC)
\x1b8     → Restore cursor position + attributes (DECRC)
```

> **Recommendation for Rust wrapper**: Use `\x1b[s` / `\x1b[u` for maximum compatibility.

### Request Cursor Position (CPR)

You can **ask the terminal** where the cursor is:
```
\x1b[6n    → Device Status Report: request cursor position
```
The terminal responds on stdin with:
```
\x1b[<row>;<col>R
```
This is a **bidirectional** operation — your process writes to stdout and reads the response from stdin. Very useful for figuring out the current cursor row to implement overlays.

### Scroll Operations

```
\x1b[<n>S    → Scroll up n lines (new lines appear at bottom)
\x1b[<n>T    → Scroll down n lines (new lines appear at top)
```

---

## 6. Erase Commands

### Erase in Line

```
\x1b[0K    → Erase from cursor to end of line (default when no parameter)
\x1b[1K    → Erase from beginning of line to cursor
\x1b[2K    → Erase entire line (cursor position unchanged)
```

> **Most commonly used**: `\x1b[2K` erases the whole current line, then `\r` (carriage return) moves back to column 1 — together forming the "overwrite current line" idiom used by progress bars:
> ```
> print!("\x1b[2K\r{}", new_content);
> ```

### Erase in Display (Screen)

```
\x1b[0J    → Erase from cursor to end of screen (default)
\x1b[1J    → Erase from beginning of screen to cursor
\x1b[2J    → Erase entire visible screen (cursor position unchanged)
\x1b[3J    → Erase entire screen including scrollback buffer
```

After `\x1b[2J`, you usually also move the cursor home:
```
\x1b[2J\x1b[H    → "clear screen" (what the `clear` command does)
```

### Erase Characters

```
\x1b[<n>X    → Erase n characters starting at cursor (replace with spaces, cursor doesn't move)
```

---

## 7. Hyperlinks (OSC 8)

**OSC (Operating System Command)** sequences use the prefix `ESC ]` and are terminated by either `BEL` (`\x07`) or `ST` (String Terminator = `ESC \`, i.e., `\x1b\x5c`).

### OSC 8 Hyperlink Format

```
ESC ] 8 ; <params> ; <uri> ST    → Start hyperlink
ESC ] 8 ; ; ST                   → End hyperlink (empty URI)
```

The full byte sequences in Rust:
```rust
// Start link: OSC 8 ; params ; url ST
let link_start = format!("\x1b]8;;{url}\x1b\\");

// Link text (displayed, clickable)
let text = "Click here";

// End link: OSC 8 ; ; ST
let link_end = "\x1b]8;;\x1b\\";

let hyperlink = format!("{link_start}{text}{link_end}");
```

**Concrete example:**
```
\x1b]8;;https://example.com\x1b\Hello, World!\x1b]8;;\x1b\
```

The `params` field (between the first and second semicolons) is optional and rarely used. Common params include `id=xyz` for multi-line links.

**BEL terminator variant** (less common, avoid for compatibility):
```
\x1b]8;;https://example.com\x07Hello\x1b]8;;\x07
```

> **Terminal support**: OSC 8 is supported in most modern terminals: iTerm2, GNOME Terminal (3.26+), Kitty, Alacritty, Windows Terminal, WezTerm, Foot. Notably **not** supported in older xterm or macOS's built-in Terminal.app (pre-Sonoma).

---

## 8. How Terminal Emulators Parse ANSI (State Machine)

Understanding the parser helps you understand what your Rust wrapper must handle when intercepting Claude Code's output.

### The Core State Machine

Paul Williams published the definitive VT500-series parser description, which most modern emulators implement. It has these main states:

```
         ┌──────────────────────────────────────────────────────────────────┐
         │                         GROUND (default)                         │
         │  - Printable bytes → dispatch as characters                      │
         │  - 0x1B (ESC) → transition to ESCAPE                             │
         │  - 0x9B (CSI in 8-bit mode) → transition to CSI_ENTRY           │
         └────────────────────────┬─────────────────────────────────────────┘
                                  │ ESC
                                  ▼
                           ┌────────────┐
                           │   ESCAPE   │
                           │  0x5B → [ │
                           └─────┬──────┘
                 ESC [           │       ESC ]
                 ┌───────────────┘        └──────────────────┐
                 ▼                                           ▼
          ┌─────────────┐                           ┌────────────────┐
          │  CSI_ENTRY  │                           │   OSC_STRING   │
          │ collect params│                         │ collect until  │
          │ collect intermediates│                  │ BEL or ST      │
          │ final byte → dispatch│                  └────────────────┘
          └─────────────┘
```

**Simplified states:**

1. **GROUND**: Default state. Bytes are either printed characters or control codes.
2. **ESCAPE**: Just saw `0x1B`. Waiting for the next byte to know which sequence type.
3. **CSI_ENTRY / CSI_PARAM / CSI_INTERMEDIATE**: Collecting a CSI sequence. Accumulate parameter digits and separators until a final byte (`@` through `~`) is seen.
4. **OSC_STRING**: Inside an OSC sequence. Collect bytes until `BEL` (0x07) or `ESC \` (String Terminator).
5. **DCS_STRING**: Inside a DCS sequence (e.g., Sixel graphics). Less common.

### Key Parsing Rules

- **Anywhere**: `0x1B` can interrupt any ongoing sequence and restart in ESCAPE state.
- **C0 controls** (0x00–0x1F, excluding ESC) are processed *immediately* even inside sequences.
- **Parameter accumulation**: Digits accumulate into the current parameter; `;` pushes the current parameter and starts a new one.
- **Missing parameters default to 0** (or 1 for movement commands — context-dependent).
- **Unknown final bytes**: Should be silently ignored (the terminal skips the sequence).

### Practical Implication for Your Wrapper

When you intercept Claude Code's byte stream, you will see **partial sequences** mid-chunk. For example, you might receive a chunk ending with `\x1b[3` and the next chunk starts with `1m`. You **must** implement a stateful parser that buffers incomplete sequences across chunk boundaries.

---

## 9. Raw ANSI vs. Using a Library

### Writing Raw ANSI

**Pros:**
- Zero dependencies
- Complete control — you know exactly what bytes are emitted
- Perfect for understanding and educational purposes
- Fast (no abstraction overhead)
- Required anyway if you're *intercepting/passthrough* rather than *generating*

**Cons:**
- Error-prone: easy to forget resets, miscount parameters
- No automatic terminal capability detection
- You must handle Windows Console API separately (pre-Windows 10 didn't support ANSI natively)
- Cross-platform pain: `\r\n` vs `\n`, TTY detection

**Example of raw ANSI in Rust:**
```rust
fn print_colored(text: &str, r: u8, g: u8, b: u8) {
    print!("\x1b[38;2;{r};{g};{b}m{text}\x1b[0m");
}
```

### Using a Library

**Popular Rust crates:**

| Crate        | Purpose                               | Notes                                    |
|--------------|---------------------------------------|------------------------------------------|
| `crossterm`  | Cross-platform terminal manipulation  | The go-to choice; handles Windows, TTY detection, raw mode, events |
| `termion`    | Unix-only terminal control            | Lightweight, type-safe, but no Windows   |
| `owo-colors` | Styling strings for output            | Ergonomic API, zero-cost formatting      |
| `colored`    | Simple string colorization            | Easy API, widely used                    |
| `termcolor`  | Color writing with capability detect  | Used by `env_logger`, conservative       |
| `ansi-parser`| Parse ANSI escape sequences           | Useful for processing existing ANSI output |
| `vte`        | Full VT/ANSI parser (state machine)   | Implements Paul Williams' parser; great for interceptors |
| `alacritty_terminal` | Full terminal emulator backend | Heavy, but complete                    |

**Pros of libraries:**
- Handle edge cases you haven't thought of
- Cross-platform (especially Windows)
- TTY detection (don't emit ANSI when output is redirected to a file)
- Proper raw mode handling, signal handling

**Cons:**
- Dependency weight
- May not expose the low-level control you need for a wrapper/interceptor
- Can abstract away things you *need* to see

### For a Terminal Wrapper (Your Use Case)

For intercepting Claude Code's output, you'll likely need **both**:
1. A **parser** (`vte` crate) to understand the incoming stream
2. Raw ANSI writes for your own additions/transformations
3. Possibly `crossterm` for the wrapper UI elements

---

## 10. Streaming Output & The Interception Challenge

This section is especially relevant to building a Claude Code wrapper.

### How Claude Code Produces Output

Claude Code (and similar tools) produces output as a stream of bytes through stdout. Because it uses ANSI codes for formatting, colors, cursor movement (for progress indicators, spinners, interactive UI), the byte stream is a mixture of:

- Printable UTF-8 text
- ANSI SGR codes (colors, formatting)
- Cursor movement codes (for inline progress, clearing lines)
- OSC sequences (potentially hyperlinks)
- Raw newlines and carriage returns

### The Buffering Problem

**Terminals and programs buffer output in different ways:**

1. **Line-buffered**: Output is flushed when a `\n` is written (default when stdout is a TTY)
2. **Block-buffered**: Output is flushed when an internal buffer fills up (default when stdout is a pipe)
3. **Unbuffered**: Every write is immediately flushed

**The critical issue**: When you create a pipe to intercept Claude Code's stdout, the program detects it's writing to a pipe (not a TTY) and switches to **block-buffered mode**. This means:
- Output arrives in large chunks, not line-by-line
- Interactive spinners/progress indicators may never flush until the program exits
- Claude Code might disable colors entirely if it detects non-TTY output via `isatty(1)`

### PTY (Pseudo-Terminal): The Solution

To intercept output while making the program *think* it's writing to a real terminal, you use a **PTY (pseudo-terminal)**:

```
Your Rust process creates a PTY pair:
  ┌─────────────┐    PTY master    ┌─────────────┐
  │  Your Rust  │◄─────────────────│ Claude Code │
  │   Wrapper   │──────────────────►│  (subprocess)│
  └─────────────┘                  └─────────────┘
                   PTY slave (looks like a real TTY to Claude Code)
```

With a PTY:
- Claude Code's `isatty()` returns true → it keeps colors enabled
- Output is line-buffered as if it's a TTY → more responsive streaming
- You receive the full ANSI-decorated output on the master end

**Rust crates for PTY:**
- `portable-pty` — Cross-platform PTY support (used by WezTerm)
- `pty-process` — Simpler async PTY
- `nix` — Low-level `openpty()` / `forkpty()` syscalls on Unix

### Partial Sequence Handling

When reading from a PTY in chunks, you **will** encounter partial ANSI sequences split across reads. Example:

```
Read 1: "Hello \x1b[3"
Read 2: "1m World\x1b[0m"
```

The sequence `\x1b[31m` (red foreground) is split across two reads.

**Strategy in Rust:**
```rust
struct AnsiParser {
    buffer: Vec<u8>,
    state: ParserState,
}

impl AnsiParser {
    fn feed(&mut self, chunk: &[u8]) -> Vec<AnsiEvent> {
        // Append to internal buffer
        // Run state machine, emit complete events
        // Leave incomplete sequences in buffer for next feed()
        todo!()
    }
}
```

The `vte` crate handles this for you — it's a streaming, stateful parser you feed bytes into one at a time or in chunks.

### The "Live Update" Problem

Programs like Claude Code use cursor movement to implement **live updates** — overwriting previous output in-place. For example, a spinner:

```
t=0: "Thinking... /"
t=1: "\r         \rThinking... -"   ← \r moves back, spaces clear, rewrite
t=2: "\r         \rThinking... \"
```

Or using cursor up/erase to update multiple lines:

```
t=0: "File 1: processing\nFile 2: processing\nFile 3: waiting"
t=1: "\x1b[2A"           ← move up 2 lines
     "\x1b[2KFile 2: done\n"  ← erase line, rewrite
     "\x1b[2KFile 3: processing\n"
```

If you naively log the byte stream to a file, you'd see the raw cursor codes — the "logical" display state isn't directly in the byte stream. To capture the **final rendered state**, you'd need a **full terminal emulator** (virtual screen buffer) in your wrapper.

**Practical approaches for your wrapper:**
1. **Passthrough + tee**: Let ANSI pass through to the user's terminal unchanged, while also saving the raw bytes for processing later
2. **Virtual terminal**: Use a crate like `vt100` to maintain a virtual screen and extract the final rendered text
3. **Strip ANSI**: For logging/search purposes, strip all escape sequences with a regex or parser
4. **Selective interception**: Only intercept specific patterns you care about (e.g., detect when Claude prints a file path)

**Useful regex for stripping ANSI (approximate):**
```rust
// Using the `regex` crate — covers most common sequences
let ansi_re = regex::Regex::new(
    r"\x1b(?:[@-Z\\-_]|\[[0-?]*[ -/]*[@-~]|\][^\x07\x1b]*(?:\x07|\x1b\\))"
).unwrap();
let clean = ansi_re.replace_all(&raw_text, "");
```

---

## 11. Practical Rust Relevance

### Embedding ANSI in Rust Strings

Rust string literals support several escape notations for the ESC byte:

```rust
// All of these produce the same byte (0x1B):
let esc1 = "\x1b";        // Hex escape — most common and readable
let esc2 = "\u{001B}";    // Unicode escape
let esc3 = "\x1B";        // Hex escape (uppercase — identical)
// Note: \033 (octal) is NOT supported in Rust string literals

// Complete sequences:
let red   = "\x1b[31m";
let reset = "\x1b[0m";
let bold  = "\x1b[1m";

// Rust raw strings can't contain these naturally — use regular strings
let colored_hello = format!("{red}Hello{reset}");
```

### Rust-Specific Patterns

**Constant sequences:**
```rust
const RED:    &str = "\x1b[31m";
const GREEN:  &str = "\x1b[32m";
const YELLOW: &str = "\x1b[33m";
const BOLD:   &str = "\x1b[1m";
const RESET:  &str = "\x1b[0m";
const CLEAR_LINE: &str = "\x1b[2K\r";
```

**Writing directly to stdout without buffering:**
```rust
use std::io::{self, Write};
let mut stdout = io::stdout();
write!(stdout, "\x1b[32mGreen text\x1b[0m\n").unwrap();
stdout.flush().unwrap();  // Important for non-newline-terminated writes
```

**Detecting TTY (don't emit ANSI to pipes/files):**
```rust
// Using the `is-terminal` crate (or `atty` crate)
use is_terminal::IsTerminal;

let use_color = std::io::stdout().is_terminal();
if use_color {
    print!("\x1b[31mred\x1b[0m");
} else {
    print!("red");
}
```

**Writing a byte stream with ANSI to a PTY master:**
```rust
use portable_pty::{CommandBuilder, PtySize, native_pty_system};

let pty_system = native_pty_system();
let pair = pty_system.openpty(PtySize {
    rows: 24,
    cols: 80,
    pixel_width: 0,
    pixel_height: 0,
})?;

let mut cmd = CommandBuilder::new("claude");
let child = pair.slave.spawn_command(cmd)?;
drop(pair.slave); // Close slave end in parent

// Read ANSI output from master:
let mut reader = pair.master.try_clone_reader()?;
let mut buffer = [0u8; 4096];
loop {
    let n = reader.read(&mut buffer)?;
    if n == 0 { break; }
    process_ansi_chunk(&buffer[..n]);
}
```

**Parsing ANSI with the `vte` crate:**
```rust
use vte::{Parser, Perform};

struct MyHandler;

impl Perform for MyHandler {
    fn print(&mut self, c: char) {
        // Regular printable character
        print!("{c}");
    }
    
    fn csi_dispatch(&mut self, params: &vte::Params, _intermediate: &[u8], _ignore: bool, action: char) {
        // CSI sequence: action is the final byte (e.g., 'm' for SGR, 'A' for cursor up)
        match action {
            'm' => { /* Handle SGR (colors, formatting) */ }
            'A' => { /* Handle cursor up */ }
            _ => {}
        }
    }
    
    fn osc_dispatch(&mut self, params: &[&[u8]], _bell_terminated: bool) {
        // OSC sequence: params[0] is the command number
        if params.get(0) == Some(&&b"8"[..]) {
            // OSC 8 hyperlink
        }
    }
}

let mut parser = Parser::new();
let mut handler = MyHandler;

for byte in input_bytes {
    parser.advance(&mut handler, byte);
}
```

### Quick Reference Card

```
# Colors (SGR)
\x1b[30m–\x1b[37m    standard foreground colors (black→white)
\x1b[40m–\x1b[47m    standard background colors
\x1b[90m–\x1b[97m    bright foreground colors
\x1b[100m–\x1b[107m  bright background colors
\x1b[38;5;<n>m        256-color foreground
\x1b[48;5;<n>m        256-color background
\x1b[38;2;<r>;<g>;<b>m  TrueColor foreground
\x1b[48;2;<r>;<g>;<b>m  TrueColor background
\x1b[0m               RESET everything

# Text attributes
\x1b[1m  bold          \x1b[22m  not bold
\x1b[2m  dim           \x1b[22m  not dim
\x1b[3m  italic        \x1b[23m  not italic
\x1b[4m  underline     \x1b[24m  not underline
\x1b[5m  blink         \x1b[25m  not blink
\x1b[7m  reverse       \x1b[27m  not reverse
\x1b[9m  strikethrough \x1b[29m  not strikethrough

# Cursor movement
\x1b[<n>A  up           \x1b[<n>B  down
\x1b[<n>C  right        \x1b[<n>D  left
\x1b[<r>;<c>H  go to row r, col c (1-based)
\x1b[s  save position   \x1b[u  restore position
\x1b[6n  request position → response: \x1b[<r>;<c>R

# Erase
\x1b[2K   erase whole line
\x1b[0K   erase to end of line
\x1b[2J   erase screen
\x1b[2J\x1b[H  clear screen and home cursor

# Hyperlinks (OSC 8)
\x1b]8;;<url>\x1b\<text>\x1b]8;;\x1b\
```

---

## Further Reading

- **[Paul Williams' VT500 Parser](https://vt100.net/emu/dec_ansi_parser)** — The definitive state machine diagram
- **[ANSI escape code (Wikipedia)](https://en.wikipedia.org/wiki/ANSI_escape_code)** — Comprehensive reference table
- **[XTerm Control Sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)** — The xterm manual, extremely detailed
- **[Terminal colors (Haoyi's blog)](https://www.lihaoyi.com/post/BuildyourownCommandLinewithANSIescapecodes.html)** — Practical tutorial
- **[VT100 User Guide (DEC)](https://vt100.net/docs/vt100-ug/)** — Original DEC documentation
- **[`vte` crate docs](https://docs.rs/vte)** — Rust VT/ANSI parser
- **[`portable-pty` crate docs](https://docs.rs/portable-pty)** — PTY support for Rust
- **[`crossterm` crate docs](https://docs.rs/crossterm)** — Cross-platform terminal control in Rust
