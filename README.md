# claude-relaunch

**Restart Claude Code on its newest version and pick up the same conversation —
same pane, same flags, same environment. One command, about two seconds.**

*[中文說明 / Traditional Chinese README](README.zh-TW.md)*

```
/relaunch
```

---

## Why

Claude Code updates itself in the background, but a running session keeps the old
binary until you restart it. Doing that by hand means `/exit`, digging up the session
id, `claude --resume <id>`, and remembering every flag you started with
(`--model`, `--plugin-dir …`, `--dangerously-skip-permissions`, `--add-dir …`). If your
terminal or IDE injected environment variables for its own integrations when it first
launched `claude`, a hand-typed restart loses those too.

Recent Claude Code versions ship a built-in `/restart`, but it refuses to run while
Remote Control is connected — and with `remoteControlAtStartup: true` in your
settings, that is every session.

`/relaunch` does the whole round trip for you:

- **Same conversation** — resumes the exact session id, not "the most recent one"
  (which is a guess when several panes share a directory).
- **Same flags** — replays the original command line, minus anything that picks a
  session or carries a prompt.
- **Same environment** — the original process's full environment, byte for byte, so
  pane / IDE integrations and Remote Control come back.
- **Newest binary** — a native install is relaunched through `~/.local/bin/claude`,
  which always points at the latest downloaded version.

## How it works

```
/relaunch  (or  ! claude-reload)
  │
  ├─ claude-reload            runs inside the session, returns immediately
  │    1. walks up from its own shell to the claude process
  │    2. reads claude's exact argv + environment (macOS KERN_PROCARGS2) and its cwd
  │    3. finds the session id (~/.claude/sessions/<pid>.json, else $CLAUDE_CODE_SESSION_ID)
  │    4. saves all of it to ~/.claude/reload/<session-id>.json   (mode 600)
  │    5. detaches a helper
  │
  ├─ helper                   detached, survives claude's exit
  │    SIGTERM claude (graceful shutdown) → wait for the pane's shell prompt
  │    → type `claude-relaunch <session-id>` into the pane (herdr or Orca)
  │
  └─ claude-relaunch
       reads and deletes the state file, cd's to the original cwd, and execs
       claude <original flags> --resume <session-id>  with the original environment
```

## Requirements

- **macOS.** Process arguments and environment are read with `sysctl(KERN_PROCARGS2)`;
  `ps` and `lsof` do the rest. Linux is not supported.
- **`python3` on your PATH** (Homebrew, python.org, or the Xcode Command Line Tools).
  Standard library only — no packages.
- **Claude Code started from an interactive shell** (zsh, bash, fish, …). If `claude`
  is the pane's own process, exiting it would close the pane, so the tool refuses.
- **For the fully automatic restart, a [herdr](https://herdr.dev) or
  [Orca](https://github.com/stablyai/orca) terminal pane.** The helper types into the
  pane with `herdr pane run` / `orca terminal send`. Anywhere else, claude still exits
  cleanly and you type `claude-relaunch` yourself — no argument needed within 30 minutes.

Tested live with Claude Code 2.1.316 in an Orca pane on macOS 27. The herdr path is
implemented the same way but has not been tested live yet.

## Install

```sh
git clone https://github.com/ccbearyeh/claude-relaunch.git
cd claude-relaunch

# 1. The two scripts, into any directory on your shell's PATH
#    (~/.local/bin is where the native Claude Code installer puts `claude`):
install -m 755 bin/claude-reload bin/claude-relaunch ~/.local/bin/

# 2. The slash command:
mkdir -p ~/.claude/commands
cp commands/relaunch.md ~/.claude/commands/
```

`/relaunch` shows up in new Claude Code sessions. Check the setup without restarting
anything:

```sh
! claude-reload --dry-run
```

```
claude pid 97325 (shell zsh pid 97309), session 3f2a9c1e-7b4d-4e8a-9c2f-1d5e6a7b8c9d
cwd   /Users/you/project
run   /Users/you/.local/bin/claude --dangerously-skip-permissions --resume 3f2a9c1e-…
env   71 vars from the original launch (values not shown)
pane  orca
```

## Usage

| Command | What it does |
|---|---|
| `/relaunch` | Slash command. Costs one tiny model turn (the reply "Relaunching…"). The model cannot trigger it on its own (`disable-model-invocation: true`). |
| `! claude-reload` | Same thing from Claude Code's bash mode — no model turn at all. |
| `claude-reload --dry-run` | Show what would be relaunched; change nothing. |
| `claude-relaunch [session-id]` | The second half. Type it yourself if the pane could not be typed into. With no argument it takes the newest pending relaunch from the last 30 minutes. |

## What carries over

| Kept | Dropped |
|---|---|
| Every flag you started with — `--model`, `--plugin-dir`, `--add-dir`, `--dangerously-skip-permissions`, … | Flags that pick or create a session: `-c/--continue`, `-r/--resume`, `--session-id`, `--fork-session`, `-w/--worktree`, `--from-pr`, `--teleport`, `--cloud` — the relaunch adds its own `--resume <id>` |
| The full original environment | The positional prompt you started with (never replayed) |
| The original working directory | |

Which flags take a value is read from the new binary's `claude --help`, so new flags
are handled without updating this tool. If the help text cannot be parsed, it refuses
rather than guessing. `--print` / `-p` sessions are refused — there is nothing to resume.

## Security notes

- The launch environment can contain secrets — API keys you exported, hook tokens an
  IDE injected. The state file is written mode `600` inside a `700` directory, its
  values are never printed, and `claude-relaunch` deletes it before it execs claude.
- If a relaunch does not complete, the state file stays in `~/.claude/reload/`.
  `claude-relaunch` consumes it; otherwise delete it yourself.
- Both scripts run under `python3 -I` (isolated mode: ignores `PYTHON*` variables, the
  current directory and user site-packages).

## Troubleshooting

Everything the helper does is logged to `~/.claude/reload/reload.log`.

| Symptom | Fix |
|---|---|
| Claude exited but did not come back | Type `claude-relaunch` in the pane. Check the log for why the pane could not be typed into. |
| Log says `claude still running after 20s` | Nothing was relaunched. `/exit`, then `claude-relaunch`. |
| `claude was not started from a shell` | The pane runs `claude` directly. Use `/exit`, then `claude --resume <session-id>`. |
| Orca logs a `provider: unsupported` warning from `orca terminal send` | Harmless — the text is still delivered. |

## Files

```
bin/claude-reload      first half: capture the session, stop claude, schedule the relaunch
bin/claude-relaunch    second half: exec claude with the saved flags + env + --resume
commands/relaunch.md   the /relaunch slash command
```

## License

[MIT](LICENSE). Not affiliated with Anthropic.
