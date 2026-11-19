# Architecture Research: Tailor-Swift Terminal Wrapper

> **Purpose**: This document surveys and evaluates possible architectures for `tailor-swift`, a Rust tool that wraps the `claude` CLI to intercept, parse, and re-render its terminal output with improved formatting, colors, and UX.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Architecture 1: Simple Pipe Wrapper](#architecture-1-simple-pipe-wrapper)
3. [Architecture 2: PTY Wrapper (Recommended)](#architecture-2-pty-wrapper-recommended)
4. [Architecture 3: Output Post-Processor / Filter](#architecture-3-output-post-processor--filter)
5. [Architecture 4: Shell Function / Alias Wrapper](#architecture-4-shell-function--alias-wrapper)
6. [Architecture 5: Named Pipe / FIFO Approach](#architecture-5-named-pipe--fifo-approach)
7. [Architecture Comparison Matrix](#architecture-comparison-matrix)
8. [Recommendation](#recommendation)
9. [Output Parsing Strategy](#output-parsing-strategy)
10. [Suggested Project Structure](#suggested-project-structure)

---

## Problem Statement

`claude` (Claude Code CLI) is an interactive terminal application. It:

- **Checks `isatty()`** on its file descriptors to determine whether it is talking to a real terminal or a pipe. If piped, it may change behaviour: buffer output differently, disable colors, alter formatting, or even refuse to run interactively.
- **Streams output in real time** — partial lines, ANSI escape sequences, and control codes arrive mid-stream.
- **Accepts interactive user input** — the user types prompts; Claude responds. A wrapper must relay stdin as well as capture stdout/stderr.
- **Outputs a mix of content types**: plain prose, markdown-flavoured text (headers, lists, code blocks), ANSI-colored diffs, progress indicators, and UI chrome.

A good wrapper must solve all three concerns simultaneously: fool `isatty()`, handle streaming, and pass through user input — while applying a richer rendering layer on top.

```
┌─────────────────────────────────────────────────────┐
│                   User's Terminal                    │
│                                                      │
│  ┌──────────────┐    stdin     ┌──────────────────┐  │
│  │              │ ──────────▶  │                  │  │
│  │  tailor-swift│              │  claude (child)  │  │
│  │  (wrapper)   │ ◀──────────  │                  │  │
│  │              │  stdout/err  │                  │  │
│  └──────────────┘              └──────────────────┘  │
│         │                                            │
│    re-render                                         │
│         ▼                                            │
│  ┌──────────────┐                                    │
│  │  Formatted   │                                    │
│  │  Output      │                                    │
│  └──────────────┘                                    │
└─────────────────────────────────────────────────────┘
```

---

## Architecture 1: Simple Pipe Wrapper

### How It Works

`tailor-swift` spawns `claude` as a child process with its `stdout` and `stderr` redirected to OS pipes. The wrapper reads from those pipes in a loop, processes each chunk or line, and writes re-formatted output to the wrapper's own stdout (which is the user's terminal).

Stdin from the user is either ignored (non-interactive mode) or forwarded to the child's stdin via another pipe.

```
User Terminal (TTY)
       │
       │  stdin (typed by user)
       ▼
┌─────────────┐    pipe    ┌──────────┐
│tailor-swift │──────────▶│  claude  │
│             │◀──────────│          │
└─────────────┘  pipe     └──────────┘
       │  stdout/stderr from claude (raw bytes)
       │
   [process & reformat]
       │
       ▼
  User's terminal (formatted output)
```

### Rust Implementation Sketch

```rust
use std::process::{Command, Stdio};
use std::io::{BufRead, BufReader, Write};

fn main() {
    let mut child = Command::new("claude")
        .args(std::env::args().skip(1))
        .stdin(Stdio::inherit())   // or Stdio::piped() for full relay
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .expect("failed to spawn claude");

    let stdout = child.stdout.take().unwrap();
    let reader = BufReader::new(stdout);

    for line in reader.lines() {
        let line = line.unwrap();
        let formatted = reformat_line(&line); // your transformation
        println!("{}", formatted);
    }

    child.wait().unwrap();
}
```

**Key crates:**
- `std::process` — built-in subprocess spawning with pipes
- `crossterm` / `owo-colors` — for re-rendering with ANSI
- `tokio` with `tokio::process` — if async processing is needed for concurrent stdin relay

**Main loop pattern (async, with stdin relay):**
```rust
use tokio::process::Command;
use tokio::io::{AsyncBufReadExt, BufReader};

#[tokio::main]
async fn main() {
    let mut child = Command::new("claude")
        .stdin(Stdio::inherit())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn().unwrap();

    let stdout = BufReader::new(child.stdout.take().unwrap());
    let mut lines = stdout.lines();

    while let Some(line) = lines.next_line().await.unwrap() {
        let formatted = reformat_line(&line);
        println!("{}", formatted);
    }
}
```

### Pros and Cons

| Aspect | Detail |
|--------|--------|
| ✅ Simplicity | Minimal code; uses only `std::process` |
| ✅ No OS-specific code | Works on Linux, macOS, Windows |
| ✅ Easy to reason about | Linear data flow |
| ❌ **isatty() problem** | Claude detects the pipe and may disable colors, change buffering, or behave differently |
| ❌ Stdin relay is complex | Concurrent stdin forwarding + stdout reading requires threading or async |
| ❌ Loss of interactivity signals | Control keys (Ctrl-C, Ctrl-Z) may not propagate correctly |
| ❌ Line-buffering vs block-buffering | Claude's stdout buffering changes in a pipe context, affecting real-time streaming |

### UX Implications

- If Claude detects non-TTY and disables its own colors, the wrapper receives plain text — which is actually easier to parse, but loses any semantic color hints Claude embedded.
- Claude may output full lines to stdout only after a newline, breaking the real-time streaming feel.
- Interactive prompts (where Claude waits for user input mid-session) may not work correctly without careful stdin threading.

### Cross-Platform Considerations

| Platform | Support | Notes |
|----------|---------|-------|
| Linux | ✅ Full | Pipes work natively |
| macOS | ✅ Full | Same pipe API |
| Windows | ✅ Mostly | `Stdio::piped()` works; ANSI support requires Windows 10+ or `crossterm` |

---

## Architecture 2: PTY Wrapper (Recommended)

### How It Works

A **PTY (Pseudo-Terminal)** is a kernel-provided mechanism that presents a fake terminal device to the child process. The child writes to what it believes is a real TTY; the parent reads from the "master" end, processes the data, and writes re-formatted output to the actual terminal.

```
                     PTY pair (kernel-managed)
                   ┌─────────────────────────┐
User's Real TTY    │  Master        Slave     │
       │           │  (read/write)  (TTY fd)  │
       │  stdin    │       │            │     │
       ▼           │       │            ▼     │
┌─────────────┐   │       │      ┌──────────┐│
│tailor-swift │──────────────▶   │  claude  ││
│  (Rust)     │   │       │      │          ││
│             │◀──────────────   │  (child) ││
└─────────────┘   │  raw output  └──────────┘│
       │           └─────────────────────────┘
  [strip ANSI]
  [re-render]
       │
       ▼
  User's terminal (better formatted output)
```

The key insight: `claude` calls `isatty(STDOUT_FILENO)` → it sees the PTY slave fd → `isatty()` returns `true` → Claude behaves exactly as if running in a real terminal.

### Rust Implementation Sketch

**Using `portable-pty` crate (highest-level, cross-platform):**

```rust
use portable_pty::{CommandBuilder, PtySize, native_pty_system};
use std::io::{Read, Write};

fn main() {
    let pty_system = native_pty_system();

    // Size of the PTY — should match the user's terminal
    let size = PtySize {
        rows: 24,
        cols: 80,
        pixel_width: 0,
        pixel_height: 0,
    };

    let pair = pty_system.openpty(size).expect("failed to open PTY");

    // Build the command
    let mut cmd = CommandBuilder::new("claude");
    cmd.args(std::env::args().skip(1));

    // Spawn claude inside the PTY
    let mut child = pair.slave.spawn_command(cmd).expect("failed to spawn");

    // Drop the slave after spawning (we only need the master end)
    drop(pair.slave);

    // Clone for stdin relay
    let mut master_writer = pair.master.try_clone_writer().unwrap();
    let mut master_reader = pair.master.try_clone_reader().unwrap();

    // Thread 1: relay user's stdin → PTY master (→ claude)
    std::thread::spawn(move || {
        let stdin = std::io::stdin();
        let mut stdin_lock = stdin.lock();
        let mut buf = [0u8; 1024];
        loop {
            let n = stdin_lock.read(&mut buf).unwrap_or(0);
            if n == 0 { break; }
            master_writer.write_all(&buf[..n]).ok();
        }
    });

    // Thread 2 (main): read from PTY master, process, write to stdout
    let stdout = std::io::stdout();
    let mut buf = [0u8; 4096];
    loop {
        match master_reader.read(&mut buf) {
            Ok(0) | Err(_) => break,
            Ok(n) => {
                let raw = &buf[..n];
                let formatted = process_output(raw); // parse & reformat
                stdout.lock().write_all(&formatted).ok();
            }
        }
    }

    child.wait().ok();
}
```

**Using `nix` for raw PTY on Unix (more control, Linux/macOS only):**

```rust
use nix::pty::{openpty, OpenptyResult};
use nix::unistd::{fork, ForkResult, setsid, dup2};
use std::os::unix::io::FromRawFd;

fn spawn_in_pty(args: &[&str]) -> (std::fs::File, nix::unistd::Pid) {
    let OpenptyResult { master, slave } = openpty(None, None).unwrap();

    match unsafe { fork() }.unwrap() {
        ForkResult::Child => {
            // Child: set up as a new session leader with slave as controlling TTY
            setsid().unwrap();
            // Set slave as stdin/stdout/stderr
            dup2(slave, 0).unwrap();
            dup2(slave, 1).unwrap();
            dup2(slave, 2).unwrap();
            // Close master in child
            nix::unistd::close(master).unwrap();
            // exec claude
            let err = nix::unistd::execvp(
                &std::ffi::CString::new(args[0]).unwrap(),
                &args.iter().map(|s| std::ffi::CString::new(*s).unwrap()).collect::<Vec<_>>()
            );
            eprintln!("exec failed: {:?}", err);
            std::process::exit(1);
        }
        ForkResult::Parent { child } => {
            nix::unistd::close(slave).unwrap();
            (unsafe { std::fs::File::from_raw_fd(master) }, child)
        }
    }
}
```

**Terminal size propagation (critical for correct wrapping):**

```rust
use nix::libc::{ioctl, winsize, TIOCGWINSZ, TIOCSWINSZ};

fn get_terminal_size() -> winsize {
    let mut size: winsize = unsafe { std::mem::zeroed() };
    unsafe { ioctl(1, TIOCGWINSZ, &mut size) };
    size
}

fn set_pty_size(master_fd: i32, size: &winsize) {
    unsafe { ioctl(master_fd, TIOCSWINSZ, size) };
}
```

**SIGWINCH handling (window resize):**

```rust
use signal_hook::iterator::Signals;
use signal_hook::consts::SIGWINCH;

// In a separate thread:
let mut signals = Signals::new(&[SIGWINCH]).unwrap();
for _signal in signals.forever() {
    let size = get_terminal_size();
    set_pty_size(master_fd, &size);
}
```

**Key crates:**
- `portable-pty` — Cross-platform PTY; used by WezTerm. Best for portability.
- `nix` — Low-level Unix PTY syscalls; Linux/macOS only.
- `signal-hook` — For SIGWINCH (terminal resize) handling.
- `crossterm` — For raw mode on the user's terminal side.
- `tokio` — For async I/O if preferred over threads.

### Pros and Cons

| Aspect | Detail |
|--------|--------|
| ✅ **isatty() transparent** | Claude sees a real TTY; behaves identically to direct use |
| ✅ Fully interactive | Stdin, special keys, Ctrl-C all work |
| ✅ Colors preserved | Claude's own ANSI codes arrive intact; we can keep or transform them |
| ✅ Real-time streaming | PTY flushes on each write, not line-buffered like pipes |
| ✅ Terminal resize support | SIGWINCH + TIOCSWINSZ propagates resize events |
| ❌ Complexity | More code, more edge cases (e.g., echo, raw mode) |
| ❌ Unix-centric | PTY is a POSIX concept; Windows support via ConPTY is recent and more complex |
| ❌ ANSI echo | PTY echoes input by default; must configure discipline carefully |
| ❌ Raw mode on host | The wrapper must put the user's terminal in raw mode to prevent double echo |

### UX Implications

- **Best possible UX**: Claude behaves normally; user experience is identical to running `claude` directly, plus the formatting layer.
- Must put the host terminal in **raw mode** (disable echo, disable canonical line mode) to avoid double-echoing user keystrokes.
- Must handle the PTY's own echo correctly — often the PTY line discipline echoes back what the user types; in raw passthrough mode, this may need to be disabled.
- Terminal resize (dragging terminal window) must trigger `SIGWINCH` → `TIOCSWINSZ` on the PTY.

### Cross-Platform Considerations

| Platform | Support | Notes |
|----------|---------|-------|
| Linux | ✅ Native | `openpty()`, `forkpty()` via `nix`; `portable-pty` works great |
| macOS | ✅ Native | Same POSIX PTY API as Linux |
| Windows | ⚠️ Partial | ConPTY (Windows 10 1809+) supported by `portable-pty` via WinAPI; more limited |

For Linux/macOS, PTY is the gold standard. `portable-pty` abstracts the differences.

---

## Architecture 3: Output Post-Processor / Filter

### How It Works

The simplest possible architecture: Claude runs normally, the user pipes output through `tailor-swift`:

```bash
claude | tailor-swift
# or
claude 2>&1 | tailor-swift
```

`tailor-swift` reads from its own stdin (Claude's output) and writes reformatted output to its own stdout (the terminal).

```
User runs: claude | tailor-swift

┌────────────┐  pipe  ┌──────────────────┐  stdout  ┌──────────┐
│   claude   │───────▶│  tailor-swift    │─────────▶│ Terminal │
│            │        │  (stdin → reformat → stdout) │          │
└────────────┘        └──────────────────┘           └──────────┘
```

### Rust Implementation Sketch

```rust
use std::io::{self, BufRead, Write};

fn main() {
    let stdin = io::stdin();
    let stdout = io::stdout();
    let mut out = stdout.lock();

    for line in stdin.lock().lines() {
        let line = line.expect("read error");
        let formatted = reformat_line(&line);
        writeln!(out, "{}", formatted).ok();
    }
}
```

**Key crates:**
- `std::io` — That's nearly it.
- `strip-ansi-escapes` — To strip any ANSI codes Claude emits (though in pipe mode Claude likely won't emit them).

### Pros and Cons

| Aspect | Detail |
|--------|--------|
| ✅ Trivially simple | A few dozen lines of Rust |
| ✅ Composable | Unix philosophy; works with any program |
| ✅ No PTY complexity | No process spawning at all |
| ❌ **isatty() breaks it** | Claude detects it's piped; disables colors, may change format entirely |
| ❌ No interactivity | Stdin goes to the user's shell, not Claude; Claude cannot prompt for input |
| ❌ Buffering | Pipe output is block-buffered; real-time streaming is lost |
| ❌ Limited use case | Only viable if Claude is used in a pure output mode (not interactive) |

### UX Implications

- Only usable for non-interactive Claude invocations (e.g., `claude -p "summarize this"` with a fixed prompt).
- Claude will likely output plain text without colors, reducing the richness of data available to the formatter.
- Output arrives in chunks, not in real time.

### Cross-Platform Considerations

| Platform | Support | Notes |
|----------|---------|-------|
| Linux | ✅ Full | Standard Unix pipes |
| macOS | ✅ Full | Standard Unix pipes |
| Windows | ✅ Full | PowerShell/cmd pipes work; ANSI requires Windows Terminal |

---

## Architecture 4: Shell Function / Alias Wrapper

### How It Works

A shell function (bash/zsh) uses the `script` command (or `faketty` / `unbuffer`) to run Claude inside a fake TTY, then pipes the output to the Rust formatter:

```bash
# ~/.bashrc or ~/.zshrc
claude() {
    script -q -c "claude $*" /dev/null | tailor-swift
}
```

Or more robustly using `unbuffer` from the `expect` package:

```bash
claude() {
    unbuffer claude "$@" | tailor-swift
}
```

The Rust `tailor-swift` binary is then a pure post-processor (like Architecture 3), but now Claude thinks it's in a TTY due to `script`/`unbuffer`.

```
User types: claude "hello"
      │
      ▼
Shell function intercepts
      │
      ▼
  script -q -c "claude hello" /dev/null
      │  (claude sees a fake TTY)
      │
      ▼  stdout (with colors, TTY behavior)
      │
     pipe
      │
      ▼
  tailor-swift (pure filter/reformatter)
      │
      ▼
  User's terminal (formatted)
```

### Rust Implementation Sketch

The Rust side is identical to Architecture 3 (a stdin filter). The shell handles the PTY faking:

```rust
// tailor-swift just reads stdin and reformats
fn main() {
    for line in std::io::stdin().lock().lines() {
        println!("{}", reformat_line(&line.unwrap()));
    }
}
```

**Key crates:**
- Same as Architecture 3; Rust is just a filter here.

### Pros and Cons

| Aspect | Detail |
|--------|--------|
| ✅ Rust side is simple | Pure filter, no process spawning |
| ✅ Claude sees a TTY | `script`/`unbuffer` fools `isatty()` |
| ❌ Requires shell integration | User must add a shell function; not a standalone tool |
| ❌ `script` varies by OS | macOS `script` has different flags than Linux `script` |
| ❌ `script` output includes timestamps/headers | The output format may need extra stripping |
| ❌ Interactivity limited | `script` replays stdin but interactive prompting is tricky |
| ❌ Not a real Rust solution | The complexity lives in shell, defeating the purpose |
| ❌ Portability | Requires bash/zsh; Windows incompatible |

### UX Implications

- Less seamless than Architecture 2; the shell function setup is a friction point for users.
- `script` on macOS adds a header line and footer line (`Script started on...`) that must be filtered.
- Difficult to handle interactive back-and-forth; `script` is more suited to recording sessions than live interaction.

### Cross-Platform Considerations

| Platform | Support | Notes |
|----------|---------|-------|
| Linux | ⚠️ Partial | `script -q -c "..."` works but flags differ |
| macOS | ⚠️ Partial | `script -q /dev/null claude` — different syntax |
| Windows | ❌ None | `script` doesn't exist; WSL required |

---

## Architecture 5: Named Pipe / FIFO Approach

### How It Works

A FIFO (named pipe) is a special filesystem node that provides a pipe-like channel between unrelated processes. In theory:

1. Create a named pipe: `mkfifo /tmp/claude-out.fifo`
2. Run Claude writing to the FIFO: `claude > /tmp/claude-out.fifo`
3. `tailor-swift` reads from the FIFO and reformats

Or more cleverly, `tailor-swift` creates the FIFO, sets up claude to write there, and reads from it:

```
┌──────────────┐  writes to   ┌──────────────┐  reads from  ┌──────────────┐
│    claude    │─────────────▶│  Named FIFO  │─────────────▶│ tailor-swift │
└──────────────┘              │ /tmp/ts.fifo │              └──────────────┘
                              └──────────────┘
```

### Rust Implementation Sketch

```rust
use std::fs::OpenOptions;
use std::io::{BufRead, BufReader};

fn main() {
    // Create FIFO (via nix or shell)
    nix::unistd::mkfifo("/tmp/claude-out.fifo",
        nix::sys::stat::Mode::S_IRWXU).ok();

    // Spawn claude writing to the FIFO
    std::process::Command::new("sh")
        .arg("-c")
        .arg("claude > /tmp/claude-out.fifo")
        .spawn().unwrap();

    // Read from FIFO
    let fifo = OpenOptions::new().read(true)
        .open("/tmp/claude-out.fifo").unwrap();
    for line in BufReader::new(fifo).lines() {
        println!("{}", reformat_line(&line.unwrap()));
    }
}
```

### Pros and Cons

| Aspect | Detail |
|--------|--------|
| ✅ Decoupled processes | Producer and consumer are independent |
| ❌ **isatty() problem** | Claude writing to a FIFO sees a non-TTY; same problem as Architecture 1 |
| ❌ Interactivity lost | FIFOs are unidirectional; stdin relay requires a second FIFO |
| ❌ Complexity without benefit | More complex than a plain pipe, without solving the core problem |
| ❌ Cleanup required | FIFO files must be cleaned up after use |
| ❌ Unix-only | Windows has named pipes but with a completely different API |

### Verdict

Named pipes solve IPC between unrelated processes, but that's not the problem here. For this use case, Architecture 5 offers no advantage over Architecture 1 and significantly more complexity. **Do not use this architecture.**

---

## Architecture Comparison Matrix

| Feature | Arch 1: Pipe | Arch 2: PTY | Arch 3: Filter | Arch 4: Shell | Arch 5: FIFO |
|---------|-------------|-------------|----------------|---------------|--------------|
| Fools `isatty()` | ❌ No | ✅ Yes | ❌ No | ⚠️ Via `script` | ❌ No |
| Interactive stdin | ⚠️ Partial | ✅ Full | ❌ No | ⚠️ Limited | ❌ No |
| Real-time streaming | ⚠️ Maybe | ✅ Yes | ⚠️ Maybe | ⚠️ Maybe | ⚠️ Maybe |
| Rust complexity | Low | High | Minimal | Minimal | Medium |
| Cross-platform | ✅ Good | ⚠️ Unix-primary | ✅ Full | ❌ Unix/bash | ❌ Unix-only |
| Standalone binary | ✅ | ✅ | ⚠️ Needs shell | ❌ Needs shell func | ⚠️ Partial |
| Claude colors available | ❌ Likely not | ✅ Yes | ❌ Likely not | ✅ Maybe | ❌ Likely not |
| Handle SIGWINCH | N/A | ✅ Supported | N/A | ⚠️ Limited | N/A |
| Production viability | ⚠️ Limited | ✅ High | ⚠️ Low | ❌ Low | ❌ Very Low |

---

## Recommendation

### **Use Architecture 2: PTY Wrapper**

The PTY wrapper is the only architecture that solves all three core requirements:

1. **`isatty()` transparency** — Claude sees a real TTY and behaves identically to running directly.
2. **Full interactivity** — The user can type prompts; Claude can prompt for input; Ctrl-C works.
3. **Real-time streaming** — PTY output is not block-buffered; each write from Claude arrives promptly.

```
Recommended primary architecture:
╔══════════════════════════════════════════════════════╗
║           tailor-swift (Rust binary)                 ║
║                                                      ║
║  ┌─────────────────────────────────────────────┐     ║
║  │  1. Put host terminal in raw mode           │     ║
║  │  2. Query terminal size (TIOCGWINSZ)        │     ║
║  │  3. Open PTY pair (master/slave)            │     ║
║  │  4. Set PTY size to match terminal          │     ║
║  │  5. Fork/spawn claude with slave as TTY     │     ║
║  │  6. Drop slave in parent                    │     ║
║  │                                             │     ║
║  │  Thread A: stdin → master (relay user input)│     ║
║  │  Thread B: master → parse → reformat → tty  │     ║
║  │  Thread C: SIGWINCH → resize PTY            │     ║
║  └─────────────────────────────────────────────┘     ║
╚══════════════════════════════════════════════════════╝
```

### Why Not the Others?

- **Architecture 1 (Pipe)**: Claude likely detects the pipe and changes behaviour. No true interactivity. Use only as a fallback for non-interactive invocations.
- **Architecture 3 (Filter)**: Same `isatty()` problem. Only for non-interactive post-processing pipelines.
- **Architecture 4 (Shell)**: Not a real Rust solution; portability nightmare; interactive support is weak.
- **Architecture 5 (FIFO)**: No advantages, more complexity.

### Recommended Rust Crate Stack

```toml
[dependencies]
# PTY management (cross-platform)
portable-pty = "0.8"

# Terminal raw mode and size detection  
crossterm = "0.27"

# ANSI sequence parsing
vte = "0.13"           # State machine ANSI parser (used by Alacritty)

# Terminal styling for re-rendered output
owo-colors = "4"       # Zero-dependency color output
# or
yansi = "1"            # Feature-rich styling

# Signal handling (for SIGWINCH)
signal-hook = "0.3"

# Async I/O (optional but recommended for the main loop)
tokio = { version = "1", features = ["full"] }

# CLI argument parsing (pass-through to claude)
clap = { version = "4", features = ["derive"] }

# Configuration file
serde = { version = "1", features = ["derive"] }
toml = "0.8"
```

### Fallback Strategy

Consider implementing **both** Architecture 1 and Architecture 2, with automatic fallback:

```rust
fn spawn_claude(args: &[String]) -> Result<ClaudeHandle> {
    // Try PTY first
    match spawn_in_pty(args) {
        Ok(handle) => Ok(handle),
        Err(e) => {
            eprintln!("[tailor-swift] PTY unavailable ({}), falling back to pipe mode", e);
            spawn_with_pipe(args)
        }
    }
}
```

This provides Windows support and graceful degradation.

---

## Output Parsing Strategy

Once we have Claude's raw byte stream from the PTY master, we need to parse and transform it. This section evaluates three strategies.

### Strategy A: Line-by-Line Regex/Pattern Matching

The simplest approach: split on newlines (or `\r\n`), apply regex patterns to identify content types.

```rust
fn reformat_line(line: &str) -> String {
    // Strip ANSI first
    let plain = strip_ansi(line);

    // Match patterns
    if let Some(caps) = HEADER_RE.captures(&plain) {
        let level = caps[1].len(); // number of #'s
        let text = &caps[2];
        return render_header(text, level);
    }
    if BULLET_RE.is_match(&plain) {
        return render_bullet(&plain);
    }
    if CODE_FENCE_RE.is_match(&plain) {
        return render_code_fence(&plain);
    }
    // Default: re-emit with standard prose formatting
    render_prose(&plain)
}

// Regex patterns
lazy_static::lazy_static! {
    static ref HEADER_RE: Regex = Regex::new(r"^(#{1,6})\s+(.+)$").unwrap();
    static ref BULLET_RE: Regex = Regex::new(r"^(\s*[-*+]|\s*\d+\.) ").unwrap();
    static ref CODE_FENCE_RE: Regex = Regex::new(r"^```").unwrap();
}
```

**Pros:**
- Very fast; zero-allocation for most lines
- Easy to extend with new patterns
- Works well for structured output (headers, bullets)

**Cons:**
- Stateless — cannot track multi-line constructs (code blocks, numbered lists that span lines)
- Regex can mis-fire (e.g., a code block containing `# comment` looking like a header)
- ANSI sequences embedded mid-line break regex matching

**Best for:** Quick wins; first implementation; prose and simple markdown.

---

### Strategy B: State Machine Parser (Recommended)

A state machine tracks the parser's context across lines/chunks. It knows whether we are inside a code block, a list, a paragraph, etc.

```
States:
  PROSE         ← default
  IN_CODE_BLOCK ← after ``` open fence
  IN_LIST       ← after first bullet/numbered item
  IN_HEADING    ← single-line state (transitions back to PROSE)
  IN_BLOCKQUOTE ← after > prefix

Transitions:
  PROSE + "```..."        → IN_CODE_BLOCK
  IN_CODE_BLOCK + "```"   → PROSE
  PROSE + "^#+ "          → IN_HEADING → PROSE
  PROSE + "^[-*] "        → IN_LIST
  IN_LIST + blank line    → PROSE
  PROSE + "^> "           → IN_BLOCKQUOTE
  IN_BLOCKQUOTE + no ">"  → PROSE
```

```rust
#[derive(Debug, Clone, PartialEq)]
enum ParseState {
    Prose,
    InCodeBlock { language: Option<String> },
    InList { indent: usize },
    InBlockquote,
}

struct OutputParser {
    state: ParseState,
    ansi_parser: AnsiParser, // wraps vte or similar
}

impl OutputParser {
    fn feed_chunk(&mut self, chunk: &[u8]) -> Vec<u8> {
        // Feed raw bytes through ANSI parser first
        // Collect text events (characters) and control events (ANSI sequences)
        // Apply state machine to text; pass through or discard control sequences
        // Emit reformatted output
        todo!()
    }

    fn process_line(&mut self, line: &str) -> String {
        match &self.state {
            ParseState::Prose => self.process_prose_line(line),
            ParseState::InCodeBlock { language } => self.process_code_line(line, language.clone()),
            ParseState::InList { indent } => self.process_list_line(line, *indent),
            ParseState::InBlockquote => self.process_blockquote_line(line),
        }
    }
}
```

**ANSI integration with `vte`:**

```rust
use vte::{Params, Parser, Perform};

struct VteHandler {
    text_buffer: String,
    ansi_actions: Vec<AnsiAction>,
}

impl Perform for VteHandler {
    fn print(&mut self, c: char) {
        self.text_buffer.push(c);
    }

    fn csi_dispatch(&mut self, params: &Params, _intermediates: &[u8], _ignore: bool, action: char) {
        // Record ANSI color/style actions
        self.ansi_actions.push(AnsiAction::Csi { params: params.clone(), action });
    }

    fn execute(&mut self, byte: u8) {
        // Newline, carriage return, etc.
        if byte == b'\n' {
            self.ansi_actions.push(AnsiAction::Newline);
        }
    }
}
```

**Pros:**
- Correctly handles multi-line constructs (code blocks, nested lists)
- Can preserve Claude's intent while re-styling
- Extensible: add new states for new content types
- Works at the byte/char level, not line level — handles partial lines from streaming

**Cons:**
- More code than regex approach
- Must handle edge cases (unterminated code blocks, etc.)
- State can get out of sync if Claude uses non-standard markdown

**Best for:** Production implementation; the right long-term choice.

---

### Strategy C: Full ANSI Strip + Markdown Parse + Re-render

Strip all ANSI from Claude's output, parse the resulting text as markdown, then render it with full control.

```rust
use pulldown_cmark::{Parser as MdParser, Event, Tag};

fn rerender_markdown(raw_output: &str) -> String {
    // Step 1: Strip ANSI
    let plain = strip_ansi_escapes::strip_str(raw_output);

    // Step 2: Parse as markdown
    let parser = MdParser::new(&plain);
    let mut output = String::new();

    for event in parser {
        match event {
            Event::Start(Tag::Heading(level, ..)) => {
                output.push_str(&format!("\x1b[1;33m")); // bold yellow
                output.push_str(&"=".repeat(level as usize * 2));
                output.push(' ');
            }
            Event::End(Tag::Heading(..)) => {
                output.push_str("\x1b[0m\n");
            }
            Event::Text(text) => {
                output.push_str(&text);
            }
            Event::Start(Tag::CodeBlock(kind)) => {
                output.push_str("\x1b[48;5;236m"); // dark background
            }
            Event::End(Tag::CodeBlock(_)) => {
                output.push_str("\x1b[0m");
            }
            // ... etc
            _ => {}
        }
    }
    output
}
```

**Pros:**
- Complete control over rendering
- Uses a battle-tested markdown parser (`pulldown-cmark`)
- Output is predictable and consistent

**Cons:**
- **Requires full document context** — doesn't work well on streaming chunks; must buffer until a logical block is complete
- **Loses real-time feel** — output is delayed until a section completes
- Claude's output is not always clean markdown — it may mix ANSI and markdown in ways `pulldown-cmark` doesn't understand
- Buffering may hold back partial responses for a long time
- Loses any semantic color Claude itself used (e.g., diff colors)

**Best for:** Post-processing complete outputs (e.g., saving to a file); not recommended for real-time interactive use.

---

### Recommended Strategy: Hybrid State Machine

For real-time interactive use, combine Strategy A and B:

```
                  raw bytes from PTY
                        │
                        ▼
              ┌─────────────────┐
              │  vte ANSI parser│  ← parse byte-level escape sequences
              └────────┬────────┘
                       │  events: text chars + ANSI codes
                       ▼
              ┌─────────────────┐
              │  Line assembler │  ← accumulate chars into lines
              └────────┬────────┘
                       │  complete lines (stripped of ANSI)
                       ▼
              ┌─────────────────┐
              │  State machine  │  ← track prose/code/list/heading state
              │  (markdown-aware)│
              └────────┬────────┘
                       │  semantic tokens
                       ▼
              ┌─────────────────┐
              │  Renderer       │  ← apply ANSI styling
              └────────┬────────┘
                       │  formatted bytes
                       ▼
                  user's terminal
```

This handles:
- Partial writes from the PTY (vte handles mid-sequence splits)
- Multi-line constructs (state machine)
- Real-time output (line-by-line, not document-level buffering)
- Both Claude's own ANSI (which we can selectively pass through or override) and semantic markdown

---

## Suggested Project Structure

```
tailor-swift/
├── Cargo.toml
├── Cargo.lock
├── README.md
│
├── src/
│   ├── main.rs              # Entry point; CLI arg parsing; spawn strategy selection
│   │
│   ├── cli.rs               # Clap-based CLI definition; argument pass-through to claude
│   │
│   ├── pty/                 # PTY wrapper module
│   │   ├── mod.rs           # Public API: spawn_in_pty(), PtyHandle
│   │   ├── unix.rs          # Unix PTY implementation (via portable-pty or nix)
│   │   └── windows.rs       # ConPTY implementation (Windows fallback)
│   │
│   ├── pipe.rs              # Fallback pipe-based spawner (Architecture 1)
│   │                        #   Used when PTY is unavailable or --no-pty flag set
│   │
│   ├── parser/              # Output parsing module
│   │   ├── mod.rs           # Public API: OutputParser, ParsedEvent
│   │   ├── ansi.rs          # ANSI sequence handling (wraps vte crate)
│   │   ├── state_machine.rs # Markdown-aware state machine
│   │   └── line_buffer.rs   # Assembles streaming bytes into lines
│   │
│   ├── renderer/            # Re-rendering / output formatting
│   │   ├── mod.rs           # Renderer trait; dispatch
│   │   ├── default.rs       # Default theme: headers, bullets, code blocks
│   │   ├── themes.rs        # Theme definitions (color schemes, symbols)
│   │   └── ansi_writer.rs   # Low-level ANSI emit helpers
│   │
│   ├── config.rs            # Config file loading (~/.config/tailor-swift/config.toml)
│   │                        #   Theme selection, feature flags, pass-through options
│   │
│   ├── signals.rs           # Unix signal handling: SIGWINCH, SIGTERM, SIGINT
│   │                        #   Propagates resize events to PTY
│   │
│   └── error.rs             # Custom error types (thiserror)
│
├── themes/                  # Built-in theme files (TOML)
│   ├── default.toml
│   └── minimal.toml
│
├── notes/                   # Developer research notes (this directory!)
│   └── research/
│       ├── 01_ansi_fundamentals.md
│       ├── 02_rust_ecosystem.md
│       └── 03_architecture.md   ← this file
│
└── tests/
    ├── integration/
    │   ├── pipe_test.rs     # Test pipe-mode output formatting
    │   └── parser_test.rs   # Test output parser with fixture inputs
    └── fixtures/
        ├── claude_output_1.txt   # Recorded Claude outputs for testing
        └── claude_output_2.txt
```

### Module Responsibilities

| Module | Responsibility |
|--------|---------------|
| `main.rs` | Parse `tailor-swift` flags (not claude's flags); select spawn strategy; wire up components |
| `cli.rs` | Define the `tailor-swift` CLI; separate `--ts-*` flags from pass-through claude args |
| `pty/mod.rs` | Unified PTY API: open PTY, spawn child, return reader/writer handles |
| `pty/unix.rs` | Unix-specific: `openpty`, raw mode, `TIOCSWINSZ` |
| `pty/windows.rs` | ConPTY via `portable-pty` WinAPI backend |
| `pipe.rs` | Alternative spawner without PTY; for fallback or non-interactive use |
| `parser/ansi.rs` | Wraps `vte::Parser`; produces `AnsiEvent` stream from raw bytes |
| `parser/state_machine.rs` | Markdown state machine; produces semantic `ParsedToken` stream |
| `parser/line_buffer.rs` | Accumulates byte chunks into complete lines for the state machine |
| `renderer/default.rs` | Maps `ParsedToken` to ANSI-formatted output bytes |
| `renderer/themes.rs` | Color/style theme data structures and loader |
| `renderer/ansi_writer.rs` | Helpers: `bold()`, `fg_rgb()`, `reset()`, etc. wrapping ANSI sequences |
| `config.rs` | Load `~/.config/tailor-swift/config.toml`; merge with CLI flags |
| `signals.rs` | Register signal handlers; relay `SIGWINCH` as PTY resize; propagate `SIGINT` |
| `error.rs` | `TailorError` enum with `thiserror`; clean error messages |

### Data Flow Diagram

```
main()
  │
  ├─ parse CLI flags (cli.rs)
  ├─ load config (config.rs)
  │
  ├─ spawn child (pty/unix.rs OR pipe.rs)
  │     │
  │     └─ returns (MasterReader, MasterWriter, ChildHandle)
  │
  ├─ spawn threads:
  │     ├─ [Thread A] stdin relay: read raw_stdin → write MasterWriter
  │     ├─ [Thread B] output pipeline:
  │     │     raw_bytes ← MasterReader
  │     │        │
  │     │        ▼
  │     │     AnsiParser (parser/ansi.rs)   [vte]
  │     │        │
  │     │        ▼
  │     │     LineBuffer (parser/line_buffer.rs)
  │     │        │
  │     │        ▼
  │     │     StateMachine (parser/state_machine.rs)
  │     │        │
  │     │        ▼
  │     │     Renderer (renderer/default.rs)
  │     │        │
  │     │        ▼
  │     │     write to stdout (user's terminal)
  │     │
  │     └─ [Thread C] signal handler (signals.rs)
  │           SIGWINCH → resize PTY
  │           SIGINT   → forward to child
  │
  └─ wait for child exit; restore terminal; exit with child's status
```

---

## Appendix: Key Design Decisions

### Decision 1: PTY Size Synchronization

The PTY must have the same dimensions as the user's terminal, or Claude will wrap/truncate text at wrong column widths:

```rust
// On startup
let size = crossterm::terminal::size()?; // (cols, rows)
pty.resize(PtySize { rows: size.1, cols: size.0, .. })?;

// On SIGWINCH
signals::on_resize(|new_size| {
    pty.resize(PtySize { rows: new_size.1, cols: new_size.0, .. }).ok();
});
```

### Decision 2: Terminal Raw Mode

The host terminal must be in **raw mode** while `tailor-swift` is running:

```rust
crossterm::terminal::enable_raw_mode()?;
// ... run wrapper ...
crossterm::terminal::disable_raw_mode()?; // always, even on panic
```

Without raw mode, the terminal's line discipline processes keystrokes before `tailor-swift` sees them, breaking special key handling.

### Decision 3: Exit Code Propagation

`tailor-swift` should exit with the same code as `claude`:

```rust
let status = child.wait()?;
std::process::exit(status.code().unwrap_or(1));
```

### Decision 4: Passthrough Mode

If `tailor-swift` is itself piped (e.g., `tailor-swift | grep`), it should detect this and act as a pure passthrough:

```rust
if !crossterm::tty::IsTty::is_tty(&std::io::stdout()) {
    // We're being piped; don't do any ANSI formatting
    // Just relay claude's output verbatim
}
```
