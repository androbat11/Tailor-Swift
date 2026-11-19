# Rust Ecosystem Survey: Building a Claude Code Terminal Wrapper

> **Purpose**: This document surveys all Rust crates relevant to building a terminal output wrapper/prettifier for Claude Code. Each crate is evaluated for fit in a tool that intercepts Claude's output stream, parses it, and re-renders it with better formatting.

---

## Table of Contents

1. [The PTY Problem](#the-pty-problem) ← **Read this first**
2. [Category 1: Terminal Color & Styling Crates](#category-1-terminal-color--styling-crates)
3. [Category 2: Raw ANSI Parsing Crates](#category-2-raw-ansi-parsing-crates)
4. [Category 3: TUI / Terminal UI Frameworks](#category-3-tui--terminal-ui-frameworks)
5. [Category 4: Process & PTY Wrapping](#category-4-process--pty-wrapping)
6. [Category 5: Streaming & Async](#category-5-streaming--async)
7. [Category 6: Config & CLI](#category-6-config--cli)
8. [Category 7: Markdown Rendering in Terminal](#category-7-markdown-rendering-in-terminal)
9. [Summary Table](#summary-table)
10. [Recommended Stack](#recommended-stack)

---

## The PTY Problem

> **This is the most important concept for this project. Everything else is secondary.**

### What Happens When You Pipe a Process

When you run a program normally, its stdout is connected to your terminal — a TTY (teletypewriter). But when you redirect its output to a pipe (`claude | my-tool`) or capture it in a subprocess, the program's stdout file descriptor points to a pipe, not a TTY.

Programs routinely check whether their output is a TTY before they start. The C standard library function `isatty(fd)` returns 1 if the file descriptor refers to a terminal, 0 if it's a pipe, file, or socket. This check changes behaviour dramatically:

```c
// What programs do internally (conceptually):
if (isatty(STDOUT_FILENO)) {
    enable_colors();
    enable_progress_bars();
    line_buffer_output();     // flush on each \n
} else {
    disable_colors();         // no ANSI codes — they'd look like garbage in a file
    disable_progress_bars();
    block_buffer_output();    // flush only when buffer is full (8KB+)
}
```

**Consequences for Claude Code:**
- Claude almost certainly detects non-TTY and **disables its own ANSI color output**
- Output may become **block-buffered**, causing chunks to arrive instead of a steady stream
- Interactive prompts, spinner animations, and progress indicators **disappear entirely**
- The UI experience is completely degraded — you'd be processing plain text instead of the rich ANSI-colored output the user actually sees

### What is a PTY (Pseudo-Terminal)?

A PTY (Pseudo-Terminal) is a kernel-provided pair of file descriptors that acts like a real terminal, without any physical hardware. It has two ends:

- **Master (PTM)**: The side your wrapper reads from and writes to — it sees everything the child process outputs, as if it were the terminal display.
- **Slave (PTS)**: The side the child process gets as its stdin/stdout/stderr — it looks exactly like a real `/dev/tty` to the child.

```
┌─────────────────────────────────────────────────────┐
│                 Kernel PTY Pair                     │
│                                                     │
│   wrapper process                 claude process    │
│   ┌──────────┐                   ┌──────────┐       │
│   │          │◄──── master ─────►│          │       │
│   │  our app │   (PTM fd)        │  claude  │       │
│   │          │◄──── slave  ─────►│          │       │
│   └──────────┘   (PTS fd)        └──────────┘       │
│                                                     │
│   isatty(slave_fd) == 1  ✓   Claude thinks it's     │
│                               talking to a real     │
│                               terminal!             │
└─────────────────────────────────────────────────────┘
```

Because `isatty()` on the slave end returns `true`, Claude behaves exactly as it would in a real terminal:
- Emits full ANSI color codes
- Uses line-buffered output (each line arrives immediately)
- Renders progress bars, spinners, the whole UX

Your wrapper sits on the master side, reads Claude's output in real-time, transforms it, and writes the result to the real user's terminal.

### PTY vs Pipe Comparison

| Feature | Pipe (`|`) | PTY |
|---------|-----------|-----|
| `isatty()` on child stdout | `false` | `true` |
| Child emits ANSI colors | ❌ Usually disabled | ✅ Full colors |
| Output buffering | Block-buffered (8KB chunks) | Line-buffered (immediate) |
| Interactive (stdin relay) | Partial | ✅ Full bidirectional |
| Cross-platform | ✅ Universal | ⚠️ Unix (Linux/macOS); Windows needs ConPTY |
| Implementation complexity | Low | Medium–High |

### Rust PTY Options

See [Category 4](#category-4-process--pty-wrapping) for detailed crate analysis. The short answer: **use `portable-pty`** from the WezTerm project — it's the most mature, cross-platform (including Windows ConPTY), and battle-tested PTY crate in the Rust ecosystem.

---

## Category 1: Terminal Color & Styling Crates

These crates handle **writing** styled/colored text to the terminal. For this project, they're used in the output stage — after parsing Claude's output, we re-render it with our own styling.

---

### `owo-colors`
- **crates.io**: https://crates.io/crates/owo-colors
- **Latest stable**: `4.4.0`
- **Description**: A zero-allocation terminal coloring library using Rust's extension trait pattern. Apply colors by calling methods directly on strings: `"hello".green()`.

**Key Features:**
- Zero heap allocation — colors via `Display` wrappers
- Compile-time color selection with trait bounds
- `OwoColorize` trait extends `&str`, `String`, and any `Display` type
- `should_colorize()` respects `NO_COLOR`, `TERM`, and `CLICOLOR` env vars
- Supports 3-bit, 8-bit (256-color), and 24-bit TrueColor
- Conditional coloring: `"text".if_supports_color(Stream::Stdout, |t| t.green())`

```rust
use owo_colors::OwoColorize;

println!("{}", "Error:".red().bold());
println!("{}", "→ file.rs".bright_cyan());
println!("{}", "✓ Done".on_green().black());
```

**Pros:**
- Extremely ergonomic API — reads like natural English
- No allocations; very fast
- Automatic NO_COLOR / CLICOLOR_FORCE support
- Actively maintained

**Cons:**
- Not a full terminal control library (no cursor movement, etc.)
- Requires knowing color at compile time for most patterns

**Fit for this project**: ⭐⭐⭐⭐⭐ **Excellent** — ideal for the output/rendering stage.

---

### `colored`
- **crates.io**: https://crates.io/crates/colored
- **Latest stable**: `3.1.1`
- **Description**: The classic Rust coloring library with a simple, chainable API.

**Key Features:**
- Chain methods: `"hello".red().bold().underline()`
- Runtime color enable/disable
- `ColoredString` type that wraps strings
- Supports standard 16 colors + some extended

```rust
use colored::Colorize;

println!("{}", "Warning".yellow().bold());
```

**Pros:**
- Very simple, widely known
- Good documentation and examples
- Large community

**Cons:**
- Allocates `ColoredString` for each styled item
- Fewer features than `owo-colors` for 256-color/truecolor
- Less active maintenance compared to `owo-colors`
- No built-in `NO_COLOR` support (manual check needed)

**Fit for this project**: ⭐⭐⭐ **Good** — works fine but `owo-colors` is strictly better.

---

### `termcolor`
- **crates.io**: https://crates.io/crates/termcolor
- **Latest stable**: `1.4.1`
- **Description**: A cross-platform terminal color library from the BurntSushi (ripgrep) ecosystem. Designed for correctness over ergonomics.

**Key Features:**
- Supports Windows Console (not just ANSI)
- `ColorSpec` structs for precise color specification
- `StandardStream` / `BufferedStandardStream` for thread-safe output
- Used by `grep`, `ripgrep`, and many CLI tools

```rust
use termcolor::{Color, ColorChoice, ColorSpec, StandardStream, WriteColor};

let mut stdout = StandardStream::stdout(ColorChoice::Always);
stdout.set_color(ColorSpec::new().set_fg(Some(Color::Red)))?;
writeln!(&mut stdout, "Error text")?;
stdout.reset()?;
```

**Pros:**
- Rock-solid, production-tested (used in ripgrep)
- Correct Windows support
- Thread-safe

**Cons:**
- Verbose API — not ergonomic for heavy use
- Low-level; lots of boilerplate
- Not designed for heavy inline coloring of strings

**Fit for this project**: ⭐⭐⭐ **Good for infrastructure** — use as the actual write-to-terminal layer under a friendlier API, or use `owo-colors` instead.

---

### `crossterm` (styling portion)
- **crates.io**: https://crates.io/crates/crossterm
- **Latest stable**: `0.29.0`
- **Description**: A full cross-platform terminal manipulation library. Covers colors, cursor movement, event handling, and more. See also [Category 3](#category-3-tui--terminal-ui-frameworks).

**Key Features:**
- `SetForegroundColor`, `SetBackgroundColor`, `SetAttribute` commands
- Works on Windows (native console API or ANSI mode), Linux, macOS
- Part of Ratatui's backend

```rust
use crossterm::{
    execute,
    style::{Color, Print, ResetColor, SetForegroundColor},
};

execute!(
    stdout(),
    SetForegroundColor(Color::Blue),
    Print("Blue text"),
    ResetColor
)?;
```

**Pros:**
- Best-in-class Windows support
- Comprehensive — one crate for all terminal needs
- Actively maintained; Ratatui ecosystem

**Cons:**
- Heavier dependency than a pure coloring crate
- More complex API for simple coloring tasks

**Fit for this project**: ⭐⭐⭐⭐ **Excellent** — especially if you also use Ratatui or want one library for all terminal I/O.

---

### `yansi`
- **crates.io**: https://crates.io/crates/yansi
- **Latest stable**: `1.0.1`
- **Description**: "Yet Another ANSI Styling" library. Focuses on correctness, global enable/disable, and clean API.

**Key Features:**
- Global paint disable: `yansi::disable()` — all styling becomes a no-op instantly
- `Paint` struct wraps values for styling
- Respects `NO_COLOR`
- Supports nested styles

```rust
use yansi::Paint;

println!("{}", "header".bold().underline().green());
```

**Pros:**
- Very clean global disable mechanism
- Zero-cost when disabled
- Good for conditional styling based on terminal capability

**Cons:**
- Less popular than `owo-colors` or `colored`
- Smaller community

**Fit for this project**: ⭐⭐⭐ **Decent** — `owo-colors` is preferred.

---

### `console`
- **crates.io**: https://crates.io/crates/console
- **Latest stable**: `0.16.6`
- **Description**: From the `indicatif` team (progress bars). A higher-level terminal utilities library including styled text, term width detection, and user prompts.

**Key Features:**
- `style()` function for ANSI styling
- Terminal width detection: `Term::stdout().size()`
- Key press reading
- Used by `indicatif` progress bars internally
- Strips ANSI for non-TTY automatically

```rust
use console::style;

println!("{} {}", style("Error:").red().bold(), style("file not found").dim());
```

**Pros:**
- Integrates with `indicatif` for progress bars (bonus feature)
- Higher-level utilities (term size, etc.)
- Good cross-platform support

**Cons:**
- Heavier than pure coloring crates
- API less ergonomic than `owo-colors` for heavy inline coloring

**Fit for this project**: ⭐⭐⭐⭐ **Good** — especially if you want progress indicators during processing.

---

### `nu-ansi-term`
- **crates.io**: https://crates.io/crates/nu-ansi-term
- **Latest stable**: `0.50.3`
- **Description**: Fork of `ansi_term` maintained by the Nushell project. Handles ANSI styles with `Style` and `ANSIString` types.

**Key Features:**
- `Style` struct composable at runtime
- Handles hyperlinks (OSC 8)
- Nushell uses it internally

```rust
use nu_ansi_term::Color;

println!("{}", Color::Blue.bold().paint("Blue bold text"));
```

**Pros:**
- Actively maintained by Nushell team
- Supports hyperlinks

**Cons:**
- Allocates `ANSIString` per styled item
- Less ergonomic than `owo-colors`
- Primarily maintained for Nushell's use case

**Fit for this project**: ⭐⭐ **Low** — use `owo-colors` instead unless you specifically need Nushell compatibility.

---

### `anstyle`
- **crates.io**: https://crates.io/crates/anstyle
- **Latest stable**: `1.0.14`
- **Description**: The low-level ANSI style representation used by the `rust-cli` working group. Intentionally a data structure, not a rendering engine.

**Key Features:**
- `Style`, `Color`, `Effects` types
- Used by `clap`, `env_logger`, and other CLI tools
- Minimal — just the data model

**Pros:**
- Standard type for interop between CLI crates
- Zero-dep
- Very small

**Cons:**
- Not a user-facing coloring library — you'd build on top of it
- Low-level

**Fit for this project**: ⭐⭐ **Infrastructure only** — useful if you're building your own styling engine or integrating with `clap`'s help coloring.

---

### Category 1 Recommendation

> **Use `owo-colors` as your primary coloring crate** for rendering styled output. It has the best ergonomics, zero allocation, and proper NO_COLOR support. If you also need cursor movement and cross-platform Windows support, add `crossterm` — the two work well together. Use `anstyle` only as an interop type.

---

## Category 2: Raw ANSI Parsing Crates

These crates handle **reading** and **parsing** ANSI escape sequences. Critical for intercepting Claude's output: before you can re-render it, you need to strip or understand Claude's own ANSI codes.

---

### `vte`
- **crates.io**: https://crates.io/crates/vte
- **Latest stable**: `0.15.0`
- **Description**: A parser for VT/ANSI terminal escape sequences, extracted from the Alacritty terminal emulator. The most complete and correct ANSI parser in the Rust ecosystem.

**Key Features:**
- State machine based on [Paul Williams' VT parser spec](https://vt100.net/emu/dec_ansi_parser)
- Handles all CSI, OSC, DCS, and private sequences correctly
- Zero-copy: works on byte slices
- Callback-based `Perform` trait: implement methods for each sequence type
- Used by Alacritty, WezTerm, Zellij, and others in production

```rust
use vte::{Parser, Perform};

struct MyHandler;

impl Perform for MyHandler {
    fn print(&mut self, c: char) {
        print!("{}", c);  // regular text character
    }

    fn csi_dispatch(&mut self, params: &vte::Params, _intermediates: &[u8], _ignore: bool, action: char) {
        // Handle CSI sequences (colors, cursor movement, etc.)
        // action == 'm' means SGR (color/style)
        println!("[CSI params={:?} action={}]", params, action);
    }

    fn osc_dispatch(&mut self, params: &[&[u8]], _bell_terminated: bool) {
        // Handle OSC sequences (hyperlinks, window title, etc.)
    }

    fn execute(&mut self, byte: u8) {
        // Handle C0/C1 control characters (\r, \n, \t, bell, etc.)
    }
}

let mut parser = Parser::new();
let mut handler = MyHandler;
parser.advance(&mut handler, b"\x1b[31mHello\x1b[0m");
```

**Pros:**
- Production-tested in major terminal emulators
- Most correct implementation of VT parsing
- Handles partial sequences (streaming-safe)
- Zero-copy, zero-allocation

**Cons:**
- Callback-based API requires implementing a trait (more boilerplate)
- Lower-level — you get bytes/params, not structured types like `Color::Red`
- Learning curve

**Fit for this project**: ⭐⭐⭐⭐⭐ **Excellent** — the right choice for correctly parsing whatever ANSI codes Claude emits. Alacritty and WezTerm trust it.

---

### `ansi-parser`
- **crates.io**: https://crates.io/crates/ansi-parser
- **Latest stable**: `0.9.0`
- **Description**: A simple crate that parses ANSI escape sequences and returns an iterator of `AnsiSequence` enum variants.

**Key Features:**
- Iterator-based API: yields `Output::TextBlock` or `Output::Escape(AnsiSequence)`
- Good for simple use cases
- Enum-based parsed output (more ergonomic than `vte`)

```rust
use ansi_parser::{AnsiParser, AnsiSequence, Output};

let parsed: Vec<Output> = "\x1b[31mHello\x1b[0m world".ansi_parse().collect();
for item in parsed {
    match item {
        Output::TextBlock(text) => print!("{}", text),
        Output::Escape(seq) => println!("[escape: {:?}]", seq),
    }
}
```

**Pros:**
- Very easy API — just `.ansi_parse()` on a string
- Returns structured enum types
- Good for simple parsing needs

**Cons:**
- Less complete than `vte` — may miss obscure sequences
- Not as battle-tested
- Doesn't handle streaming/partial sequences as robustly

**Fit for this project**: ⭐⭐⭐ **Good for simple cases** — if Claude's output is well-behaved and uses standard sequences, this works. For robustness, prefer `vte`.

---

### `strip-ansi-escapes`
- **crates.io**: https://crates.io/crates/strip-ansi-escapes
- **Latest stable**: `0.2.1`
- **Description**: A single-purpose crate: remove ANSI escape codes from byte strings. Built on top of `vte`.

**Key Features:**
- `strip(input: &[u8]) -> Vec<u8>` — strips all escapes
- `strip_str(input: &str) -> Cow<str>` — string version

```rust
let plain = strip_ansi_escapes::strip("\x1b[31mHello\x1b[0m world");
assert_eq!(plain, b"Hello world");
```

**Pros:**
- Dead simple, one function
- Correct (uses `vte` internally)
- Perfect for getting plain text from ANSI-colored output

**Cons:**
- One-way: strips, doesn't parse or transform
- Not useful if you need to understand the semantics

**Fit for this project**: ⭐⭐⭐⭐ **Very useful** — use to strip Claude's ANSI before passing to a markdown parser, then re-apply your own ANSI styling.

---

### `anstream`
- **crates.io**: https://crates.io/crates/anstream
- **Latest stable**: `1.0.0`
- **Description**: From the `rust-cli` working group. A smart wrapper around a `Write` that handles ANSI stripping for non-TTY targets and passthrough for TTY targets.

**Key Features:**
- Wraps any `Write` impl
- Automatically strips ANSI codes when writing to a non-TTY (pipe, file)
- Respects `NO_COLOR`, `TERM`, `CLICOLOR`, `CLICOLOR_FORCE`
- Used by `clap` for its help output

```rust
use anstream::println;

// This macro works like std::println! but strips ANSI when output is non-TTY
println!("{}", "\x1b[31mRed text\x1b[0m");
```

**Pros:**
- Automatic TTY detection and ANSI handling
- Drop-in replacement for `println!` / `eprintln!`
- Used in production by `clap`

**Cons:**
- Primarily a writer adapter, not a parser
- Less useful for intercepting/transforming ANSI input

**Fit for this project**: ⭐⭐⭐ **Useful for output** — good for ensuring our re-rendered output behaves correctly when our tool's own output is piped. Less relevant for parsing Claude's input.

---

### Category 2 Recommendation

> **Use `vte` as the core ANSI parser** for maximum correctness and streaming safety. Use `strip-ansi-escapes` as a convenience wrapper for the common operation of getting plain text. Use `anstream` for writing our tool's own output correctly.

```
Claude output stream
       │
       ▼
   vte parser  ──► classify sequences (color, cursor, text)
       │
       ▼
  strip ANSI  ──► plain text
       │
       ▼
  markdown parse + re-render
       │
       ▼
  owo-colors  ──► new styled output
       │
       ▼
  anstream  ──► write to user terminal
```

---

## Category 3: TUI / Terminal UI Frameworks

Full TUI frameworks take control of the entire terminal, enabling widgets, layouts, and event loops. For this project, they may be overkill — but are worth understanding.

---

### `ratatui`
- **crates.io**: https://crates.io/crates/ratatui
- **Latest stable**: `0.29.x` / `0.30.x`
- **Description**: The community successor to the abandoned `tui-rs`. The premier Rust TUI framework. Renders widgets (tables, lists, graphs, borders) into terminal "cells" using a retained/immediate mode hybrid.

**Key Features:**
- Widget system: `Paragraph`, `List`, `Table`, `Block`, `Gauge`, `Chart`, etc.
- Immediate-mode rendering with diffing (only redraws changed cells)
- Backend-agnostic (crossterm, termion, or termwiz)
- Layouts: `Layout::default().direction(Direction::Vertical).constraints(...)`
- Used by: lazygit, Zellij, many production tools

```rust
use ratatui::{
    backend::CrosstermBackend,
    widgets::{Block, Borders, Paragraph},
    Terminal,
};

let mut terminal = Terminal::new(CrosstermBackend::new(stdout()))?;
terminal.draw(|f| {
    let block = Block::default().title("Claude Output").borders(Borders::ALL);
    let text = Paragraph::new("Hello from Tailor Swift!").block(block);
    f.render_widget(text, f.size());
})?;
```

**Pros:**
- Extremely powerful for complex UIs
- Good ecosystem of widgets
- Active community (> 10k GitHub stars)
- Proper screen management (alternate screen, cleanup on exit)

**Cons:**
- **High complexity** for a simple pass-through formatter
- Takes full control of the terminal — not transparent
- Requires an event loop (may conflict with stdin relay)
- Output is not a "stream" — it's rendered frames
- Harder to integrate with PTY output relay

**When TUI is overkill vs useful:**
- **Overkill**: When the goal is transparent pass-through with better styling. If the user is typing to Claude, a full TUI complicates stdin relay.
- **Useful**: If you want a split-pane UI (input pane / output pane), a sidebar with metadata, or conversation history viewing.

**Fit for this project**: ⭐⭐⭐ **Situational** — if you want a full TUI experience with panes, use Ratatui. For a transparent stream formatter, it's overkill.

---

### `crossterm` (as a backend / terminal control)
- **crates.io**: https://crates.io/crates/crossterm
- **Latest stable**: `0.29.0`
- **Description**: Beyond just styling (covered in Category 1), crossterm is a full terminal control library: alternate screen, raw mode, cursor control, event reading.

**Key Features:**
- `terminal::enable_raw_mode()` / `disable_raw_mode()`
- Alternate screen: `EnterAlternateScreen` / `LeaveAlternateScreen`
- Cursor: `MoveTo`, `Hide`, `Show`
- Event system: keyboard, mouse, resize events (via `crossterm::event`)
- Ratatui's primary backend

**Pros:**
- Cross-platform (best Windows support of any Rust terminal crate)
- Comprehensive
- Active maintenance

**Cons:**
- Heavier than pure styling crates
- Some API verbosity

**Fit for this project**: ⭐⭐⭐⭐⭐ **Excellent** — use crossterm for raw mode (needed to relay keystrokes to Claude via PTY) and for direct ANSI writing when not using Ratatui.

---

### `termion`
- **crates.io**: https://crates.io/crates/termion
- **Latest stable**: `1.5.5` (note: unmaintained/minimal updates)
- **Description**: A pure Rust, bindless terminal manipulation library for Unix. Older alternative to crossterm.

**Key Features:**
- Raw mode, alternate screen, mouse, colors
- Unix-only (no Windows)
- Minimal dependencies

**Pros:**
- Simple, clean API
- Pure Rust, no C bindings

**Cons:**
- **Unix-only** — no Windows support
- Largely superseded by `crossterm`
- Minimal maintenance

**Fit for this project**: ⭐⭐ **Low** — use `crossterm` instead. Unless you're explicitly Linux/macOS-only and prefer a minimal dep tree.

---

### Category 3 Recommendation

> **Use `crossterm` as your terminal control library** for raw mode and event handling. Only add `ratatui` if you decide to build a full split-pane UI experience (which would be a bigger design choice). The initial version should use direct stream processing with crossterm, not a full TUI.

---

## Category 4: Process & PTY Wrapping

> **This is the most critical implementation category.**

---

### `portable-pty`
- **crates.io**: https://crates.io/crates/portable-pty
- **Latest stable**: `0.9.0`
- **Description**: From the WezTerm terminal emulator project. The most complete cross-platform PTY crate in the Rust ecosystem.

**Key Features:**
- Creates PTY pairs (master/slave)
- Spawns child processes attached to the PTY slave
- Reads from PTY master (child's output)
- Writes to PTY master (simulating keyboard input)
- **Cross-platform**: Linux, macOS, Windows (ConPTY), FreeBSD
- Sets terminal size (TIOCSWINSZ) — Claude can query terminal width!
- Async-friendly (the fd is pollable)

```rust
use portable_pty::{CommandBuilder, PtySize, native_pty_system};

let pty_system = native_pty_system();

// Allocate a PTY with the user's actual terminal size
let pair = pty_system.openpty(PtySize {
    rows: 24,
    cols: 80,
    pixel_width: 0,
    pixel_height: 0,
})?;

// Spawn Claude attached to the PTY slave
let mut cmd = CommandBuilder::new("claude");
cmd.args(&["--some-flag"]);
let mut child = pair.slave.spawn_command(cmd)?;

// Read Claude's output from the PTY master
let mut reader = pair.master.try_clone_reader()?;
let mut writer = pair.master.take_writer()?;

// Now read from `reader` — this is what Claude prints
// Write to `writer` — this is what Claude receives as keyboard input
```

**Pros:**
- Production-proven in WezTerm (a full terminal emulator)
- Best Windows support via ConPTY
- Sets terminal size (programs can query columns)
- Clean Rust API
- Actively maintained alongside WezTerm

**Cons:**
- Heavier dependency (pulls in WezTerm internals)
- Some complexity around async integration

**Fit for this project**: ⭐⭐⭐⭐⭐ **The recommended choice** for PTY wrapping.

---

### `pty-process` (crate: `pty-process`)
- **crates.io**: https://crates.io/crates/pty-process
- **Latest stable**: `0.5.4`
- **Description**: A lightweight PTY spawner that integrates with `std::process::Command`. Supports sync, Tokio, async-std, and smol runtimes.

**Key Features:**
- Extends `std::process::Command` with PTY support
- Async-first design
- Tokio feature flag for async reading

```rust
use pty_process::Command;
use std::process::Command as StdCommand;

let mut cmd = StdCommand::new("claude");
let mut child = cmd.spawn_pty(Some(&pty_process::Size::new(24, 80)))?;

// child.pty() gives you the master fd
let pty_fd = child.pty();
// read/write to pty_fd
```

**Pros:**
- Cleaner, more idiomatic API than `portable-pty`
- Async-native (Tokio integration)
- Lightweight

**Cons:**
- Unix-only (no Windows ConPTY)
- Less battle-tested than `portable-pty`
- Smaller community

**Fit for this project**: ⭐⭐⭐⭐ **Excellent for Linux/macOS-only** — if Windows support isn't needed, this is a cleaner API. For cross-platform, use `portable-pty`.

---

### `ptyprocess`
- **crates.io**: https://crates.io/crates/ptyprocess
- **Latest stable**: `0.5.0`
- **Description**: The backend for `expectrl` (a Rust `expect`-style library). Provides PTY process wrapping.

**Key Features:**
- Spawns processes in PTY
- Used by `expectrl` for automation

**Pros:**
- Works for basic PTY use
- Part of a larger automation ecosystem

**Cons:**
- Primarily designed for `expectrl`'s use case (scripted interaction)
- Less suitable for real-time streaming with async

**Fit for this project**: ⭐⭐ **Low** — `portable-pty` or `pty-process` are better fits.

---

### `nix` (raw PTY syscalls)
- **crates.io**: https://crates.io/crates/nix
- **Latest stable**: `0.29.x`
- **Description**: Safe Rust bindings to POSIX/Unix system calls, including `openpty`, `fork`, `execvp`, `waitpid`, `ioctl`, etc.

**Key Features:**
- `nix::pty::openpty()` — allocate a PTY pair
- `nix::unistd::fork()` / `execvp()` — spawn child in PTY
- `nix::ioctl::*` — set terminal size, etc.
- Full POSIX API surface

```rust
use nix::pty::{openpty, OpenptyResult};

let OpenptyResult { master, slave } = openpty(None, None)?;
// master fd: read Claude's output
// slave fd: give to child process as stdin/stdout/stderr
```

**Pros:**
- Maximum control — do exactly what you need
- Zero overhead over raw syscalls

**Cons:**
- Requires `fork()` and manual process management — complex and error-prone
- Unix-only
- Much more boilerplate than `portable-pty`

**Fit for this project**: ⭐⭐ **Low as primary** — use as a fallback or for custom PTY features. `portable-pty` abstracts this correctly.

---

### PTY Architecture Sketch

```rust
// Conceptual main loop using portable-pty + tokio

use portable_pty::{CommandBuilder, PtySize, native_pty_system};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. Query actual terminal size to pass to Claude
    let (rows, cols) = crossterm::terminal::size()?;

    // 2. Create PTY
    let pty = native_pty_system();
    let pair = pty.openpty(PtySize { rows, cols, pixel_width: 0, pixel_height: 0 })?;

    // 3. Spawn Claude in the PTY
    let cmd = CommandBuilder::new("claude");
    let _child = pair.slave.spawn_command(cmd)?;
    drop(pair.slave); // child owns it now

    // 4. Set up bidirectional relay
    let mut pty_reader = pair.master.try_clone_reader()?;
    let mut pty_writer = pair.master.take_writer()?;

    // 5. Put our terminal in raw mode so we can relay keystrokes
    crossterm::terminal::enable_raw_mode()?;

    let mut stdin = tokio::io::stdin();
    let mut stdout = tokio::io::stdout();

    // 6. Main relay loop
    tokio::select! {
        // PTY → our terminal (with transformation)
        _ = async {
            let mut buf = [0u8; 4096];
            loop {
                let n = pty_reader.read(&mut buf).await?;
                if n == 0 { break; }

                // HERE: parse buf with vte, transform, re-render
                let transformed = transform_output(&buf[..n]);
                stdout.write_all(&transformed).await?;
                stdout.flush().await?;
            }
            Ok::<_, anyhow::Error>(())
        } => {},

        // our terminal → PTY (pass through stdin unchanged)
        _ = async {
            let mut buf = [0u8; 1024];
            loop {
                let n = stdin.read(&mut buf).await?;
                if n == 0 { break; }
                pty_writer.write_all(&buf[..n]).await?;
            }
            Ok::<_, anyhow::Error>(())
        } => {},
    }

    crossterm::terminal::disable_raw_mode()?;
    Ok(())
}
```

### Category 4 Recommendation

> **Use `portable-pty` for maximum compatibility** (including Windows). If targeting Linux/macOS only and you prefer a cleaner async API, use `pty-process`. Avoid manual `nix` PTY management unless you need fine-grained control. Never use a plain pipe to spawn Claude — it will detect non-TTY and degrade.

---

## Category 5: Streaming & Async

---

### `tokio`
- **crates.io**: https://crates.io/crates/tokio
- **Latest stable**: `1.54.1`
- **Description**: The de facto async runtime for Rust. Provides async I/O, timers, channels, and task spawning.

**Key Features:**
- Async TCP/Unix sockets, file I/O, process spawning
- `tokio::sync::mpsc` / `broadcast` / `watch` channels
- `tokio::select!` for concurrent future racing
- `tokio::spawn` for concurrent tasks
- Used by virtually all serious async Rust projects

**Why it matters for this project:**
- Real-time output relay needs two concurrent loops (stdin→PTY and PTY→stdout)
- `tokio::select!` elegantly handles this
- Async channels let you pass output chunks from reader task to transform task to writer task
- If your transform involves any async work (e.g., syntax highlighting lookups), async is essential

```rust
// Channel-based pipeline for output transformation
let (tx, mut rx) = tokio::sync::mpsc::channel::<Vec<u8>>(32);

// Reader task: reads PTY output, sends chunks
tokio::spawn(async move {
    let mut buf = [0u8; 4096];
    loop {
        let n = pty_reader.read(&mut buf).await.unwrap();
        tx.send(buf[..n].to_vec()).await.unwrap();
    }
});

// Transformer/writer task: receives chunks, transforms, writes to terminal
while let Some(chunk) = rx.recv().await {
    let rendered = transform(&chunk);
    stdout.write_all(&rendered).await.unwrap();
}
```

**Pros:**
- Industry standard, massive ecosystem
- Excellent performance
- Well-documented

**Cons:**
- Compile times (large crate)
- Async Rust has a learning curve

**Fit for this project**: ⭐⭐⭐⭐⭐ **Strongly recommended** — the two-direction relay (stdin↔PTY↔stdout) is exactly what async was designed for.

---

### `async-std`
- **crates.io**: https://crates.io/crates/async-std
- **Latest stable**: `1.12.0`
- **Description**: Alternative async runtime mirroring the `std` API.

**Pros:**
- Familiar API (mirrors `std`)

**Cons:**
- Significantly less popular than Tokio
- Smaller ecosystem
- Less maintained in recent years
- Many PTY crates integrate with Tokio specifically

**Fit for this project**: ⭐⭐ **Low** — use Tokio. The ecosystem converges on it.

---

### Category 5 Recommendation

> **Use `tokio`**. It's the standard. The bidirectional PTY relay is a natural `tokio::select!` or task-spawn use case.

---

## Category 6: Config & CLI

---

### `clap`
- **crates.io**: https://crates.io/crates/clap
- **Latest stable**: `4.6.8`
- **Description**: The standard Rust CLI argument parser. Derive macro or builder API.

**Key Features:**
- `#[derive(Parser)]` macro for zero-boilerplate arg parsing
- Automatic `--help`, `--version`, completions
- Subcommands, flags, positional args, env var fallbacks

```rust
use clap::Parser;

#[derive(Parser)]
#[command(name = "tailor-swift", about = "Claude Code output prettifier")]
struct Cli {
    /// Path to custom theme config
    #[arg(short, long, default_value = "~/.config/tailor-swift/theme.toml")]
    config: String,

    /// Pass remaining args to claude
    #[arg(trailing_var_arg = true)]
    claude_args: Vec<String>,
}
```

**Pros:**
- Industry standard
- Excellent docs and community
- Derive API is very ergonomic

**Cons:**
- Compile time (significant, but unavoidable for a CLI)

**Fit for this project**: ⭐⭐⭐⭐⭐ **Essential.**

---

### `serde`
- **crates.io**: https://crates.io/crates/serde
- **Latest stable**: `1.0.229`
- **Description**: The Rust serialization/deserialization framework. Enables `#[derive(Deserialize)]` for config structs.

**Key Features:**
- `#[derive(Serialize, Deserialize)]`
- Works with all serialization formats (JSON, TOML, YAML, etc.)

**Fit for this project**: ⭐⭐⭐⭐⭐ **Essential** — needed alongside `toml` for config parsing.

---

### `toml`
- **crates.io**: https://crates.io/crates/toml
- **Latest stable**: `1.2.1` (TOML spec 1.1.0)
- **Description**: TOML parser/serializer for Rust. Works with `serde`.

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct Theme {
    heading_color: String,
    code_block_color: String,
    bullet_color: String,
}

// Load from ~/.config/tailor-swift/theme.toml
let content = std::fs::read_to_string(config_path)?;
let theme: Theme = toml::from_str(&content)?;
```

**Fit for this project**: ⭐⭐⭐⭐⭐ **Essential** — TOML is the natural config format for a Rust tool.

---

### Category 6 Recommendation

> **Use all three**: `clap` for CLI args, `serde` + `toml` for config files. This is the standard Rust CLI configuration stack.

---

## Category 7: Markdown Rendering in Terminal

Claude's output contains significant markdown: `**bold**`, `# headings`, `` `code` ``, code fences, bullet lists, numbered lists, and blockquotes. We need to render this beautifully in the terminal.

---

### `termimad`
- **crates.io**: https://crates.io/crates/termimad
- **Latest stable**: `0.35.5`
- **Description**: A terminal markdown renderer with customizable skins (themes). Used by `broot` and other tools. Renders markdown to styled terminal output with ANSI codes.

**Key Features:**
- Renders full CommonMark markdown to terminal ANSI output
- Customizable `MadSkin` (colors per element: heading, bold, code, etc.)
- Tables, code blocks, quotes, lists
- Respects terminal width (wraps correctly)
- Can render to a string (for capture) or directly to stdout

```rust
use termimad::MadSkin;

let mut skin = MadSkin::default();
skin.set_headers_fg(termimad::rgb(255, 180, 20));  // golden headers
skin.bold.set_fg(termimad::rgb(80, 220, 180));
skin.code_block.set_bg(termimad::rgb(30, 30, 50));

let markdown = "# Claude Says\n\n- Item one\n- Item two\n\n```rust\nfn main() {}\n```";
skin.print_text(markdown);
```

**Pros:**
- Purpose-built for this exact use case
- Rich customization via `MadSkin`
- Handles tables correctly (rare for terminal markdown renderers)
- Used in production by `broot`
- Returns `String` for further processing

**Cons:**
- Opinionated rendering (may not match your exact UX vision)
- Less control over individual elements than writing your own renderer
- May not handle Claude's specific output patterns perfectly

**Fit for this project**: ⭐⭐⭐⭐⭐ **Excellent starting point** — use `termimad` to do the heavy lifting. Customize via `MadSkin`. Write a custom renderer only if you hit limitations.

---

### `pulldown-cmark`
- **crates.io**: https://crates.io/crates/pulldown-cmark
- **Latest stable**: `0.13.6`
- **Description**: The reference CommonMark parser for Rust. Produces an event stream of parsed markdown elements.

**Key Features:**
- Streaming event-based parser (like SAX for XML)
- CommonMark-compliant
- Returns `Event` enum: `Start(Tag)`, `End(Tag)`, `Text`, `Code`, etc.
- Used by mdBook, rustdoc, and many others

```rust
use pulldown_cmark::{Event, Options, Parser, Tag};

let markdown = "# Hello\n\nThis is **bold** and `code`.";
let parser = Parser::new_ext(markdown, Options::all());

for event in parser {
    match event {
        Event::Start(Tag::Heading { level, .. }) => {
            print!("\x1b[1;33m");  // bold yellow
        }
        Event::Text(text) => print!("{}", text),
        Event::End(Tag::Heading { .. }) => {
            print!("\x1b[0m\n");  // reset
        }
        // ... handle all other cases
        _ => {}
    }
}
```

**Pros:**
- Maximum control over rendering
- CommonMark-compliant (correct parsing)
- Very fast (zero-copy design)
- Used by the entire Rust ecosystem for markdown

**Cons:**
- More work: you must implement the renderer yourself
- Lots of cases to handle (all markdown element types)

**Fit for this project**: ⭐⭐⭐⭐ **Excellent for custom rendering** — use if `termimad`'s output doesn't match your vision, or as the parsing backend for a custom ANSI renderer. The combination of `pulldown-cmark` + `owo-colors` for rendering is powerful.

---

### Strategy: Strip ANSI → Parse Markdown → Re-render

This is the core transformation pipeline:

```
Claude PTY output (with ANSI codes)
           │
           ▼
  strip-ansi-escapes        ← remove Claude's own ANSI
           │
           ▼
  detect: is this markdown? ← simple heuristic (starts with #, -, ```)
           │
           ├── yes → pulldown-cmark / termimad → re-render with our ANSI
           │
           └── no  → pass through (or apply subtle formatting)
           │
           ▼
  owo-colors / crossterm    ← emit to user's terminal
```

**Alternative**: Use `vte` to parse the raw stream, recognize text blocks, strip their ANSI, detect markdown structure, and re-render. This avoids a full strip+parse cycle and handles multi-line markdown that arrives in chunks.

---

### Category 7 Recommendation

> **Start with `termimad` for rapid prototyping** — it handles 90% of markdown rendering correctly with minimal code. Once you know what customizations you need, optionally replace or augment it with `pulldown-cmark` + a custom `owo-colors`-based renderer for full control.

---

## Summary Table

| Crate | Category | Version | Recommendation | Priority |
|-------|----------|---------|----------------|----------|
| `portable-pty` | PTY | 0.9.0 | ✅ **Essential** — spawn Claude in PTY | Critical |
| `tokio` | Async | 1.54.1 | ✅ **Essential** — async bidirectional relay | Critical |
| `vte` | ANSI Parse | 0.15.0 | ✅ **Recommended** — correct ANSI parsing | High |
| `crossterm` | Terminal Control | 0.29.0 | ✅ **Recommended** — raw mode + cross-platform | High |
| `owo-colors` | Styling | 4.4.0 | ✅ **Recommended** — ergonomic ANSI output | High |
| `termimad` | Markdown | 0.35.5 | ✅ **Recommended** — terminal markdown render | High |
| `clap` | CLI | 4.6.8 | ✅ **Essential** — argument parsing | High |
| `serde` | Config | 1.0.229 | ✅ **Essential** — config deserialization | High |
| `toml` | Config | 1.2.1 | ✅ **Essential** — TOML config format | High |
| `strip-ansi-escapes` | ANSI Parse | 0.2.1 | ✅ **Useful** — strip ANSI for markdown parse | Medium |
| `pulldown-cmark` | Markdown | 0.13.6 | ✅ **Optional** — custom markdown renderer | Medium |
| `anstream` | ANSI Output | 1.0.0 | ⚡ **Optional** — smart ANSI writer wrapper | Medium |
| `pty-process` | PTY | 0.5.4 | ⚡ **Alternative** to portable-pty (Unix only) | Medium |
| `colored` | Styling | 3.1.1 | ⬇ Superseded by `owo-colors` | Low |
| `termcolor` | Styling | 1.4.1 | ⬇ More verbose; use for infrastructure | Low |
| `yansi` | Styling | 1.0.1 | ⬇ Decent, but `owo-colors` preferred | Low |
| `console` | Styling | 0.16.6 | ⬇ Good if using `indicatif` too | Low |
| `nu-ansi-term` | Styling | 0.50.3 | ⬇ Nushell-specific | Low |
| `anstyle` | Styling | 1.0.14 | ⬇ Interop type only | Low |
| `ansi-parser` | ANSI Parse | 0.9.0 | ⬇ Less complete than `vte` | Low |
| `ratatui` | TUI | 0.29.x | ⚡ **Optional** — only if full TUI desired | Optional |
| `termion` | Terminal | 1.5.5 | ❌ Unix-only, superseded by crossterm | Skip |
| `nix` | PTY Raw | 0.29.x | ⬇ Too low-level; use portable-pty | Low |
| `async-std` | Async | 1.12.0 | ❌ Use tokio instead | Skip |
| `ptyprocess` | PTY | 0.5.0 | ⬇ Expectrl-specific | Low |

---

## Recommended Stack

```toml
[dependencies]
# PTY — spawn Claude thinking it's in a real terminal
portable-pty = "0.9"

# Async — bidirectional relay loop
tokio = { version = "1", features = ["full"] }

# ANSI parsing — understand Claude's output
vte = "0.15"
strip-ansi-escapes = "0.2"

# Terminal control — raw mode, cross-platform
crossterm = "0.29"

# Styling output — write beautiful ANSI
owo-colors = "4"

# Markdown rendering — Claude outputs lots of markdown
termimad = "0.35"
pulldown-cmark = "0.13"  # optional: for custom renderer

# CLI + Config
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
toml = "1"

# Error handling (not in brief but essential)
anyhow = "1"
```

**Minimum viable stack** (MVP):
```toml
portable-pty = "0.9"
tokio = { version = "1", features = ["full"] }
crossterm = "0.29"
owo-colors = "4"
termimad = "0.35"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
toml = "1"
anyhow = "1"
```

The core loop is: **PTY → read → vte/strip → markdown parse → owo-colors render → crossterm write**. Everything else is configuration and polish.
