---
name: firstmate-orca
description: >-
  切換到 Orca、啟動或監督 Orca 後端工作、對 Orca 後端行為進行 smoke test、除錯 Orca 工作狀態，或對帳 Orca 工作中繼資料時載入。
user-invocable: false
metadata:
  internal: true
---

# firstmate-orca

將本技能用作 Firstmate 實驗性 Orca 執行階段後端的操作者檢查清單。
它不取代 `AGENTS.md`、`docs/orca-backend.md` 或 `harness-adapters`。

Orca 是執行階段後端，不是 Agent 執行環境。
執行階段後端負責工作端點；Orca 也負責工作 worktree。
Agent 執行環境則是端點內啟動的 Agent 程序，例如 `claude`、`codex`、`opencode`、`pi`、`pi-signed`、`grok` 或 `kimi`。
執行環境專屬的啟動、中斷、恢復、信任對話框與技能呼叫事實，請載入 `harness-adapters`。

實作細節、中繼資料欄位、清理保證與限制，記錄於 `docs/orca-backend.md`。
現行 smoke 證據以 `docs/verification/runtime-backends.md` 的「Orca」為準。
優先使用 `bin/fm-*` 輔助工具，不直接呼叫 `orca`。
只有輔助工具無法回答檢查問題時，才使用原始 `orca` 指令，且仍以已記錄的 firstmate 中繼資料作為工作身分。

<a id="preflight"></a>
## 預檢

從目前 firstmate home 或儲存庫根目錄操作。
若已設定 `FM_HOME`，請記住操作狀態放在 `$FM_HOME` 下，但輔助指令碼仍從本儲存庫的 `bin/` 執行。

切換或使用 Orca 啟動前：

- 確認 Orca 是透過 `--backend orca`、`FM_BACKEND=orca` 或本機 `config/backend` 刻意選定。
- 確認 Orca 應用程式正在執行，且後端就緒檢查通過，再預期啟動能成功。
- 改變後端選擇前，檢查活躍的 `state/*.meta` 紀錄。
- 後端切換只影響未來啟動；既有工作保留已記錄後端。
- 先對帳監看程式喚醒，再處理無關工作，尤其 Orca 工作已進行時。

<a id="spawn"></a>
## 啟動

使用 `bin/fm-spawn.sh`，讓 firstmate 一起建立簡報、worktree、終端機、中繼資料、狀態檔與監看介面。
單次 Orca 工作傳入 `--backend orca`；若 Orca 已是刻意選定的後端，也可沿用目前選擇。

啟動後，使用 firstmate 輔助工具檢查工作：

- `bin/fm-peek.sh fm-<id>`：查看啟動失敗、信任對話框或第一份輸出。
- `state/<id>.meta`：確認 `backend=orca`、`terminal=`、`orca_worktree_id=` 與 `worktree=`。
- 目前執行狀態重要時，使用 `bin/fm-crew-state.sh <id>`。
- 有工作進行中，且此工作階段負責監督時，使用 `bin/fm-watch.sh`。

一般 firstmate 工作，不要手動建立 Orca worktree 或終端機。
不要手動修補中繼資料，把外部建立的 Orca 終端機偽裝成 firstmate 工作。

<a id="supervision"></a>
## 監督

例行操作使用 `bin/fm-peek.sh`、`bin/fm-send.sh`、`bin/fm-crew-state.sh` 與 `bin/fm-teardown.sh`。
指導訊息使用 `bin/fm-send.sh <id> '...'`；穩定別名 `fm-<id>` 也可使用；一般本機文字指導可包含換行，因為它透過持久化收件匣傳送。
初始範圍保留在工作簡報；若指示含工作 Agent 應另行檢查的支援素材，暫存檔仍然有用。

監督時，以 `state/<id>.meta` 作為路由紀錄，Orca 自身 ID 則視為後端實作細節。
穩定的 firstmate 別名為 `fm-<id>`。
後端輔助工具內部使用已記錄的 `terminal=` 與 `orca_worktree_id=` 欄位。

一般指導訊息入列失敗，或型別化平面的 `fm-send` 提交失敗時，不要立即重送指示。
先閱讀失敗回報並查看端點，再判斷紀錄是否存在，或目標正在忙碌、等待提示、被彈出視窗阻擋，還是真的卡死。
執行環境專屬的中斷或結束，請載入 `harness-adapters`。

<a id="recovery"></a>
## 復原

Orca 後端工作狀態混亂時：

1. 先閱讀 `state/<id>.meta` 與相關狀態尾端。
2. 採用 Orca 專屬假設前，確認該工作確實使用 Orca 後端。
3. 以已記錄的 `terminal=`、`orca_worktree_id=` 與 `worktree=` 作為工作身分。
4. 查看、傳送、狀態與清理，優先使用 firstmate 輔助工具。
5. 避免直接刪除 Orca worktree 或手動清理分支。
6. 已記錄 worktree 路徑、Orca worktree ID 或專案 checkout 不再符合預期時，停止並檢查。

清理仍受一般 firstmate 合併規則約束。
Scout 工作須在報告存在，且 `captain-hold-lifecycle` 完成關卡通過後，才能清理。
Ship 工作只有依其專案模式完成合併後，才能清理。

<a id="smoke-test"></a>
## Smoke test

Orca smoke test 聚焦生命週期銜接：

1. 為可清除工作或 scout 刻意選擇 Orca。
2. 透過 `bin/fm-spawn.sh` 啟動。
3. 確認中繼資料記錄 Orca 後端、終端機、Orca worktree ID 與隔離 worktree 路徑。
4. 驗證 `bin/fm-peek.sh`、一則簡短 `bin/fm-send.sh` 指導、監看程式喚醒行為，以及 `bin/fm-crew-state.sh`。
5. 工作已可安全捨棄或已合併後，透過 `bin/fm-teardown.sh` 清理。
6. Orca 若只是為 smoke test 選定，還原先前後端選擇。

不要把後端 smoke test 與無關功能工作混在一起。
