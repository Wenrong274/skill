# skill

自己寫、自己在用的 AI agent skill。每個 skill 是 `skills/` 底下的一個資料夾，裡面的 `SKILL.md` 是本體，可在 Claude Code、Codex、Antigravity（agy CLI）使用。

| Skill | 用途 |
|---|---|
| [brief-and-verify](skills/brief-and-verify/SKILL.md) | 把份量不小的任務交給 subagent 或另一家模型時，讓交辦更有野心，並由沒參與製作的一方重跑、重算來確認做對了。 |

## 安裝

用 [skills](https://github.com/vercel-labs/skills) CLI（需要 Node.js）：

```bash
# 裝在目前的專案內：Claude Code、Codex、Antigravity 都讀得到
npx skills add Wenrong274/skill -s brief-and-verify -a claude-code codex antigravity-cli

# 裝成全域（所有專案都能用）：Claude Code、Codex
npx skills add Wenrong274/skill -g -s brief-and-verify -a claude-code codex

# 之後更新
npx skills update
```

專案內安裝會放在 `.agents/skills/brief-and-verify/`（Codex、Antigravity 共用），並從 `.claude/skills/` 連結過去。全域安裝沒有實測過；Antigravity 的全域安裝目前讀不到 skill 內文，見注意事項，所以 Antigravity 請用專案內安裝。

### 手動安裝

不想用 CLI 時，先 clone：

```bash
git clone https://github.com/Wenrong274/skill.git
```

再把要用的 skill 資料夾（例如 `skills/brief-and-verify`）複製到各工具讀取 skill 的位置，例如：

```powershell
New-Item -ItemType Directory -Force "$HOME\.agents\skills" | Out-Null
Copy-Item -Recurse skill\skills\brief-and-verify "$HOME\.agents\skills\"
```

各工具讀取的位置：

| 工具 | 放哪裡 |
|---|---|
| Claude Code | `~/.claude/skills/brief-and-verify/` |
| Codex | `~/.agents/skills/brief-and-verify/` |
| Antigravity（agy） | 專案內的 `.agents/skills/brief-and-verify/`（agy 從工作目錄往上找到 repo 根目錄，載入每一層的 `.agents/skills/`） |

同時用 Claude Code 和 Codex 時，可以只放一份在 `~/.agents/skills/`（先照上面複製好），再讓 `~/.claude/skills/brief-and-verify` 指過去（Windows 用 Junction：`New-Item -ItemType Junction -Path "$HOME\.claude\skills\brief-and-verify" -Target "$HOME\.agents\skills\brief-and-verify"`）。

裝好後確認：在該工具裡問「你有沒有 brief-and-verify 這個 skill？原文貼出它 SKILL.md 第一個以 # 開頭的標題」（應該是「Brief & Verify：交辦要有野心，完成要有證據」），貼得出來才算裝好。只列得出名稱、貼不出內文，見下方注意事項。

## 注意事項

以下都是在一台 Windows 11 上實測（codex-cli 0.171.0、Claude Code 2.1.330、agy 1.5.14），其他電腦、其他作業系統沒測過。

- **Codex 讀不到 skill 內文**：Windows 上 Codex 的 sandbox 以 `CodexSandboxUsers` 群組讀檔。實測那台的 `~/.agents/skills` 關掉了權限繼承（`icacls` 的輸出沒有 `(I)` 標記），Codex 列得出 skill 名稱，但打不開內容，而且不會主動說。你的電腦不一定這樣，先檢查：`icacls "$HOME\.agents\skills"` 的輸出裡有沒有 `CodexSandboxUsers`。沒有就補一條唯讀權限（只給讀取，不要打開繼承，以免連帶給到上層的寫入權限）：

  ```powershell
  icacls "$HOME\.agents\skills" /grant "CodexSandboxUsers:(OI)(CI)(RX)"
  # 要拿掉時（只移除這條自己加的，不動繼承來的）：
  # icacls "$HOME\.agents\skills" /remove:g "CodexSandboxUsers"
  ```

  實測一般使用者就能加、也能拿掉，不需要系統管理員權限（`icacls /restore` 則需要，一般使用者會失敗）。
- **Antigravity 的全域安裝目前用不了**：在 `~/.gemini/config/skills.json` 宣告（`path` 要寫絕對路徑）可以讓 agy 載入名稱和描述，但內文要 agy 自己讀檔；skill 在工作目錄外時，`agy -p` 非互動模式的讀檔權限會被自動拒絕，整個沒有輸出。互動模式會不會詢問權限沒測過。所以目前建議放在專案內的 `.agents/skills/`（上面的專案內安裝就是這樣放，實測 agy、Codex 都讀得到內文）。
- brief-and-verify 會呼叫其他家的 CLI 做交叉驗證，各 CLI 的呼叫方式和實測行為見 [skills/brief-and-verify/references/external-agents.md](skills/brief-and-verify/references/external-agents.md)。

## 反饋紀錄

brief-and-verify 在流程結束、你同意後，會把觀察到的問題追加到 `~/.agents/skill-feedback/brief-and-verify.md`。這個檔放在 skill 資料夾外，換版不會被蓋掉，也不會跟著 skill 分享出去。

## 授權

[MIT](LICENSE)
