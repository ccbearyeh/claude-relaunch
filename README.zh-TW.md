# claude-relaunch

**讓 Claude Code 換上最新版重開，並接回同一段對話 —— 同一個 pane、同樣的啟動參數、
同樣的環境變數。一個指令，大約兩秒。**

*[English README](README.md)*

```
/relaunch
```

---

## 為什麼需要它

Claude Code 會在背景自動更新，但正在跑的 session 會一直用舊版，直到你重開為止。
手動重開要 `/exit`、找出 session id、打 `claude --resume <id>`，還得記得當初帶了哪些參數
（`--model`、`--plugin-dir …`、`--dangerously-skip-permissions`、`--add-dir …`）。
如果終端機或 IDE 當初啟動 `claude` 時為了自己的整合功能注入了環境變數，手動重開也會一併遺失。

新版 Claude Code 內建了 `/restart`，但只要 Remote Control 連線中它就會拒絕執行 ——
而設定裡開了 `remoteControlAtStartup: true` 的話，每個 session 都是連線中。

`/relaunch` 把整個來回一次做完：

- **同一段對話** —— 用確切的 session id 接回，不是「最近一個」（好幾個 pane 共用同一個目錄時那只是用猜的）。
- **同樣的參數** —— 原封不動重放當初的命令列，只拿掉「選 session」和「帶 prompt」的部分。
- **同樣的環境** —— 原行程的完整環境變數，一個 byte 都不差，所以 pane／IDE 整合和 Remote Control 都會回來。
- **最新版** —— 原生安裝版透過 `~/.local/bin/claude` 重開，這個 symlink 永遠指向最新下載的版本。

## 運作方式

```
/relaunch  （或  ! claude-reload）
  │
  ├─ claude-reload            在 session 裡執行，立刻返回
  │    1. 從自己的 shell 往上找到 claude 行程
  │    2. 讀出 claude 確切的 argv + 環境變數（macOS KERN_PROCARGS2）和工作目錄
  │    3. 找出 session id（~/.claude/sessions/<pid>.json，否則 $CLAUDE_CODE_SESSION_ID）
  │    4. 全部存到 ~/.claude/reload/<session-id>.json   （權限 600）
  │    5. 分離出一個背景 helper
  │
  ├─ helper                   脫離 claude 獨立執行，claude 結束後仍在
  │    對 claude 送 SIGTERM（正常關閉）→ 等 pane 的 shell 提示字元回來
  │    → 在 pane 裡打入 `claude-relaunch <session-id>`（herdr 或 Orca）
  │
  └─ claude-relaunch
       讀取並刪除狀態檔，cd 回原工作目錄，用原本的環境變數 exec
       claude <原參數> --resume <session-id>
```

## 需求

- **macOS。** 用 `sysctl(KERN_PROCARGS2)` 讀行程參數與環境變數，其餘靠 `ps` 和 `lsof`。不支援 Linux。
- **PATH 上有 `python3`**（Homebrew、python.org 或 Xcode Command Line Tools 皆可）。只用標準函式庫，不必裝套件。
- **Claude Code 是從互動式 shell 啟動的**（zsh、bash、fish…）。如果 `claude` 本身就是 pane 的主行程，
  結束它會把 pane 一起關掉，所以工具會拒絕執行。
- **要全自動重開，得在 [herdr](https://herdr.dev) 或 [Orca](https://github.com/stablyai/orca) 的 pane 裡。**
  helper 用 `herdr pane run`／`orca terminal send` 把指令打進 pane。其他環境下 claude 一樣會正常結束，
  你自己打 `claude-relaunch` 即可 —— 30 分鐘內不用帶參數。

已在 macOS 27 的 Orca pane 中以 Claude Code 2.1.316 實測。herdr 路徑用相同方式實作，但還沒實測過。

## 安裝

```sh
git clone https://github.com/ccbearyeh/claude-relaunch.git
cd claude-relaunch

# 1. 兩支腳本放進 shell PATH 上的任一目錄
#    （~/.local/bin 就是原生 Claude Code 安裝程式放 `claude` 的地方）：
install -m 755 bin/claude-reload bin/claude-relaunch ~/.local/bin/

# 2. 斜線指令：
mkdir -p ~/.claude/commands
cp commands/relaunch.md ~/.claude/commands/
```

新開的 Claude Code session 就會有 `/relaunch`。想在不重開任何東西的情況下確認設定：

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

想讓 `/relaunch` 回覆別的文字（例如「重啟中…」），改 `~/.claude/commands/relaunch.md` 最後一行即可。

## 用法

| 指令 | 作用 |
|---|---|
| `/relaunch` | 斜線指令。會花一個極短的模型回合（回覆「Relaunching…」）。模型不能自行觸發（`disable-model-invocation: true`）。 |
| `! claude-reload` | 從 Claude Code 的 bash 模式做同一件事 —— 完全不經過模型。 |
| `claude-reload --dry-run` | 只顯示會怎麼重開，什麼都不動。 |
| `claude-relaunch [session-id]` | 後半段。pane 沒辦法被自動打字時自己輸入。不帶參數就取最近 30 分鐘內最新的一筆待重開紀錄。 |

## 會保留什麼

| 保留 | 拿掉 |
|---|---|
| 當初帶的每個參數 —— `--model`、`--plugin-dir`、`--add-dir`、`--dangerously-skip-permissions`… | 選擇或建立 session 的參數：`-c/--continue`、`-r/--resume`、`--session-id`、`--fork-session`、`-w/--worktree`、`--from-pr`、`--teleport`、`--cloud` —— 重開時會自己加 `--resume <id>` |
| 完整的原始環境變數 | 當初帶的位置參數 prompt（絕不重放） |
| 原本的工作目錄 | |

哪些參數要帶值，是從新版 binary 的 `claude --help` 解析出來的，所以 Claude Code 新增參數時
不必更新這個工具。解析不出來就拒絕執行，不用猜的。`--print`／`-p` 模式的 session 會被拒絕 —— 沒有東西可以接回。

## 安全性

- 啟動環境可能含有機密 —— 你 export 的 API key、IDE 注入的 hook token。狀態檔以 `600`
  權限寫在 `700` 目錄裡，內容絕不印出，`claude-relaunch` 在 exec claude 之前就會刪掉它。
- 如果重開沒有完成，狀態檔會留在 `~/.claude/reload/`。執行 `claude-relaunch` 會把它用掉；不然請自行刪除。
- 兩支腳本都以 `python3 -I` 執行（隔離模式：忽略 `PYTHON*` 環境變數、目前目錄和使用者 site-packages）。

## 疑難排解

helper 的每一步都記在 `~/.claude/reload/reload.log`。

| 症狀 | 處理 |
|---|---|
| claude 結束了但沒有重開 | 在 pane 裡打 `claude-relaunch`。看 log 了解為什麼沒辦法打字進 pane。 |
| log 寫 `claude still running after 20s` | 沒有重開任何東西。`/exit` 之後打 `claude-relaunch`。 |
| `claude was not started from a shell` | 這個 pane 直接執行 `claude`。用 `/exit`，再 `claude --resume <session-id>`。 |
| Orca 的 `orca terminal send` 記下 `provider: unsupported` 警告 | 無害 —— 文字一樣送得到。 |

## 檔案

```
bin/claude-reload      前半段：記下 session、結束 claude、安排重開
bin/claude-relaunch    後半段：用存下的參數 + 環境 + --resume 執行 claude
commands/relaunch.md   /relaunch 斜線指令
```

## 授權

[MIT](LICENSE)。與 Anthropic 無關。
