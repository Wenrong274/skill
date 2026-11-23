# skill

自己寫、自己在用的 AI agent skill。每個 skill 是 `skills/` 底下的一個資料夾，裡面的 `SKILL.md` 是本體，可在 Claude Code、Codex、Antigravity（agy CLI）使用。

| Skill | 用途 |
|---|---|
| [brief-and-verify](skills/brief-and-verify/SKILL.md) | 把份量不小的任務交給 subagent 或另一家模型時，讓交辦更有野心，並由沒參與製作的一方重跑、重算來確認做對了。 |

## 安裝

先 clone：

```bash
git clone https://github.com/Wenrong274/skill.git
```

再把要用的 skill 資料夾（例如 `skills/brief-and-verify`）放到各工具讀取 skill 的位置：

| 工具 | 放哪裡 |
|---|---|
| Claude Code | `~/.claude/skills/brief-and-verify/` |
| Codex | `~/.agents/skills/brief-and-verify/` |
| Antigravity（agy） | 專案內的 `.agents/skills/brief-and-verify/`（agy 從工作目錄往上找到 repo 根目錄，載入每一層的 `.agents/skills/`） |

同時用 Claude Code 和 Codex 時，可以只放一份在 `~/.agents/skills/`，再讓 `~/.claude/skills/brief-and-verify` 指過去（Windows 用 Junction：`New-Item -ItemType Junction -Path "$HOME\.claude\skills\brief-and-verify" -Target "$HOME\.agents\skills\brief-and-verify"`）。

裝好後確認：在該工具裡問「你有沒有 brief-and-verify 這個 skill？原文貼出它 SKILL.md 的第一個標題」，貼得出來才算裝好。只列得出名稱、貼不出內文，見下方注意事項。

## 注意事項

以下都是在 Windows 11 上實測（codex-cli 0.171.0、Claude Code 2.1.330、agy 1.5.14），其他作業系統沒測過。

- **Codex 讀不到 skill 內文**：Windows 上 Codex 的 sandbox 以 `CodexSandboxUsers` 群組讀檔。`~/.agents/skills` 若關掉了權限繼承，Codex 列得出 skill 名稱，但打不開內容，而且不會主動說。用 `icacls "$HOME\.agents\skills"` 檢查有沒有 `CodexSandboxUsers` 的讀取權，沒有就補一條唯讀權限（只給讀取，不要打開繼承，以免連帶給到上層的寫入權限）：

  ```powershell
  icacls "$HOME\.agents\skills" /save "$HOME\acl-agents-skills.txt"   # 先備份，之後可用 /restore 還原
  icacls "$HOME\.agents\skills" /grant "CodexSandboxUsers:(OI)(CI)(RX)"
  ```

  `CodexSandboxUsers` 是本機群組，找不到時用 `icacls` 在 `~/.codex/skills` 上看到的完整名稱（例如 `<電腦名稱>\CodexSandboxUsers`）。
- **Antigravity 的全域安裝目前用不了**：在 `~/.gemini/config/skills.json` 宣告（`path` 要寫絕對路徑）可以讓 agy 載入名稱和描述，但內文要 agy 自己讀檔；skill 在工作目錄外時，`agy -p` 非互動模式的讀檔權限會被自動拒絕，整個沒有輸出。互動模式會不會詢問權限沒測過。所以目前建議放在專案內的 `.agents/skills/`。
- brief-and-verify 會呼叫其他家的 CLI 做交叉驗證，各 CLI 的呼叫方式和實測行為見 [skills/brief-and-verify/references/external-agents.md](skills/brief-and-verify/references/external-agents.md)。

## 反饋紀錄

brief-and-verify 在流程結束、你同意後，會把觀察到的問題追加到 `~/.agents/skill-feedback/brief-and-verify.md`。這個檔放在 skill 資料夾外，換版不會被蓋掉，也不會跟著 skill 分享出去。

## 授權

[MIT](LICENSE)
