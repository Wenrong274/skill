# 外部模型 CLI：呼叫方式與實測行為

把工作交給另一家模型時查這份。內容是 2026-11-23 在一台 Windows 11 上實測的結果（codex-cli 0.171.0、Claude Code 2.1.330、agy 1.5.14；agy 在測試途中從 1.1.10 自動更新，`--print-timeout` 的預設值因此改變）。大部分行另由沒參與撰寫的一方重跑確認；標「僅製作方測過」的沒有。版本或作業系統不同時，以該 CLI 的 `--help` 為準並先實跑一次：在一個用完可丟的外層目錄裡建工作目錄，叫它在工作目錄寫一個檔、往上一層（仍在那個外層目錄內）寫一個檔、執行一個指令，看哪些被擋。

共通原則：

- 權限模式一律明寫，不靠預設值：預設值會被使用者自己的設定改掉，也會隨版本改變。逾時同理，由呼叫端加（例如 Bash 的 `timeout 1800`），agy 另有 `--print-timeout`。
- 唯讀模式只適合判斷型驗收（讀碼、核對 diff）；要它重跑、重算，用寫入範圍確實被限制的模式，並指定 scratch 路徑。目前實測只有 Codex `workspace-write` 擋得住寫到範圍外。
- 它要讀的檔案放在它的工作目錄內，或用旗標納入；放不進去就把內容貼進 prompt。
- 寫給它的 prompt 只用它執行指令那個 shell 的語法。
- 長的多行 prompt 盡量用 stdin 傳（`codex exec ... -`、`claude -p` 前面接管線），不當成指令引數：Windows PowerShell 5.1 傳引數給外部程式時可能吃掉內層雙引號（未實測；PowerShell 7 實測正常）。在 Windows PowerShell 5.1 用管線送非英文 prompt 前，先設 `$OutputEncoding = [System.Text.UTF8Encoding]::new($false)`，否則中文到對方那邊是亂碼（實測）。agy 不收純文字 stdin，見下方。
- 結束後確認回報檔不是空的。有的 CLI 中途被擋會結束代碼 0、卻沒有任何輸出。

## Codex（OpenAI）

```bash
# 判斷型（唯讀）
codex exec -s read-only -C <工作目錄> -o <scratch>/codex_report.md - < prompt.md
# 重算型（只能寫 -C 目錄）
codex exec -s workspace-write -C <scratch 工作目錄> -o <scratch>/codex_report.md - < prompt.md
```

- prompt 放在引數而不是 stdin 時，在 Bash 加 `< /dev/null`，免得它等 stdin；PowerShell 不支援這個語法，從非互動的 PowerShell 呼叫時直接省略（僅製作方測過，不會卡住）。
- 目錄不是 git repo 時加 `--skip-git-repo-check`（僅製作方測過）。
- 在 Windows 上，它執行指令用的是 Windows PowerShell 5.1（指令列寫 `pwsh.exe`，實測 `$PSVersionTable` 仍是 5.1）：給它的 prompt 不寫 heredoc、`$VAR` 形式的 Bash 變數。
- `read-only` 和 `workspace-write` 都擋得住寫到 `-C` 以外；`workspace-write` 可寫 `-C` 目錄內。
- sandbox 讀不讀得到 `-C` 以外的檔，取決於該目錄的權限。Windows 上 sandbox 以 `CodexSandboxUsers` 群組讀檔，目錄關掉權限繼承時就讀不到：用 `icacls <目錄>` 看有沒有這個群組的讀取權。實測有台機器的 `~/.agents/skills` 關掉了繼承，Codex 連已安裝的 skill 本身都讀不到；這種情況把要讀的檔（含 skill）複製到 `-C` 目錄內，或把 skill 裝在專案內的 `.agents/skills/`。
- sandbox 裡的 PATH 可能和你的不同（實測 `where.exe python` 找不到）。要它跑直譯器前，先叫它回報 `where.exe python` 或寫明完整路徑。

## Claude（Anthropic）

```bash
# 判斷型：讀不到工作目錄外、不能寫檔；echo、git --version 這類唯讀指令仍可執行
cd <工作目錄> && claude -p --permission-mode manual < prompt.md > <scratch>/claude_report.md
# 重算型：cwd 設成 scratch，只放行需要的指令（寫入範圍沒有真的被限制，見下）
cd <scratch> && claude -p --permission-mode acceptEdits --allowedTools "Bash(python:*)" "PowerShell(python:*)" < prompt.md > <scratch>/claude_report.md
```

- 一定要寫 `--permission-mode manual`（2.1.330 的 `--help` 列的名稱；舊名 `default` 實測仍可用）：使用者的 `~/.claude/settings.json` 若設了 `defaultMode: bypassPermissions`，不寫旗標時什麼都放行，實測讀 cwd 外和寫檔都成功。
- 重算型的寫入範圍沒有真的被限制：寫檔工具和 PowerShell 內建指令只能動 cwd，但被放行的 `python` 寫得到 cwd 以外（實測寫出 `../y.txt`）。需要確實隔離寫入時改用 Codex `workspace-write`。
- `--add-dir <目錄>` 可納入工作目錄外的檔，但在 `acceptEdits` 下該目錄也變成可寫（僅製作方測過）；重算型別用它納入原始資料，改成先複製進 scratch。
- 在 Windows 上它可能用 PowerShell 工具執行指令，放行規則要 `Bash(...)` 和 `PowerShell(...)` 都寫（僅製作方測過）。
- `--permission-mode plan` 不適合當唯讀隔離：它仍能執行唯讀指令、讀工作目錄以外的檔，只是自己不寫檔（僅製作方測過）。
- 它會載入使用者的全域設定（`~/.claude/`，含 settings.json、CLAUDE.md、hook），同一個指令在不同人的電腦上行為可能不同。

## Antigravity（Google Gemini，`agy` CLI）

```bash
# 判斷型：能讀檔、不改專案檔；非互動時執行指令會被擋
cd <工作目錄> && agy -p "<prompt>" --mode plan --print-timeout 15m > <scratch>/agy_report.md 2> <scratch>/agy_err.txt
```

- 沒有 `-o`，回報靠重導向存檔；被擋時的說明只出現在 stderr，所以 stderr 也要另存。
- `--print-timeout` 每次明寫：1.1.10 預設 5 分鐘，1.5.14 預設 0（等到結束）。
- 純文字 prompt 只能當引數，不收 stdin；`-p` 後面要緊接 prompt，其他旗標放前面或後面都行，但不能夾在 `-p` 和 prompt 中間。
- 預設模式在非互動下，只要它想執行任何指令，權限就被自動拒絕，而且整個任務 stdout 沒有輸出、結束代碼仍是 0。`--mode plan` 也遇過一次同樣情形（它改用指令讀檔），所以每次都檢查回報檔不是空的，空的就看 stderr、重跑或把要讀的內容直接貼進 prompt。
- plan 模式不改專案檔，但會在 `~/.gemini/antigravity-cli/brain/<id>/` 寫一份 `implementation_plan.md`。
- `--sandbox` 在 Windows 實測擋不住寫到工作目錄以外。重算型要搭 `--dangerously-skip-permissions` 才跑得動，等於沒有限制，不建議；需要重算時改派 Codex `workspace-write`。
- 它本身當主程式時，派 subagent 用 `invoke_subagent`（僅製作方查過它內建的說明文件）。
- 載入 skill 的位置：從工作目錄往上找到 repo 根目錄的每一層 `.agents/skills/`；要全域載入，在 `~/.gemini/config/skills.json` 宣告，`path` 要寫絕對路徑（實測 `~/` 開頭會被拒絕，日誌寫 `must be an absolute path`；用 Windows Junction 連到 `~/.gemini/config/skills/` 也沒被載入）。這樣載入的只有名稱和 description，內文要它自己讀檔；skill 在工作目錄外時，非互動模式的讀檔權限會被自動拒絕、整個沒輸出，所以 `agy -p` 用不了全域 skill（互動模式會詢問權限，未實測）。僅製作方測過：

  ```json
  { "entries": [ { "path": "C:/Users/<你>/.agents/skills", "include_only": ["brief-and-verify"] } ] }
  ```

## 其他工具（自行補上）

```
## <工具名稱>（<哪一家>）
判斷型指令：
重算型指令：
執行指令用的 shell：
實測：寫工作目錄／寫上一層／執行指令，各自是否被擋：
讀不到的位置：
注意事項：
```
