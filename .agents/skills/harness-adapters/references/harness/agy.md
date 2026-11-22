# Antigravity CLI

Antigravity 的 `agy` TUI，已在 Linux 透過 Herdr 後端，使用 agy 1.2.0+ 完成端到端驗證。
已驗證為完整 Firstmate 執行環境，涵蓋主要工作階段、二副與船員／scout。
下列每項事實如何建立，以及哪些仍未證明，以 `../../../../../docs/verification/agy.md` 為準。

<a id="operating-facts"></a>
## 操作事實

| 項目 | 值 |
|---|---|
| 執行檔 | 從 `PATH` 解析絕對 `agy` 路徑，不存在就拒絕；它是 Go 編譯的單一執行檔，因此活躍程序名稱精確為 `agy`，且 `argv[0]=agy`。 |
| 啟動 | 以解析後絕對執行檔執行 `agy --prompt-interactive "<brief>" --model <id> --effort <level> --dangerously-skip-permissions`；簡報自動送出，不需額外 Enter。啟動程序先將 worktree 預先登錄到 agy 信任儲存區，再等待 busy 回合（若仍顯示資料夾信任對話框，則先回答），之後才回報成功。 |
| 忙碌狀態 | 沒有 hook 或外掛寫入器，因此不啟用任何寫入器，也不植入紀錄；Herdr 上以原生 `working` 狀態分類 busy，其他環境則由 `../../../../../bin/fm-busy-lib.sh` 的 `agy-regex` 畫面尾端替代判定。 |
| 畫面尾端 | Busy 狀態列左側有 `esc to cancel`；idle 列則顯示 `? for shortcuts`。點字轉圈圖示旁的 `Generating...` 是游離輸出，不是訊號。 |
| 回合結束 | 沒有回合結束 hook 或通知 touch；完成透過工作 Agent 狀態協定，以及 Herdr 原生返回 `idle` 抵達。 |
| 結束 | `/quit`，一次 Enter；程序結束。 |
| 中斷 | 一次 `Escape`，印出 Interrupted 行並留下閒置輸入區，不會再次污染，因此不接清除按鍵。 |
| 技能 | 沒有已驗證斜線技能形式；使用自然語言。 |
| 自主權 | `--dangerously-skip-permissions` 對本次執行自動核准工具呼叫。 |
| 標記 | 無；活躍 TUI 不帶 `AGY_*` 或 `ANTIGRAVITY_*` 變數。 |
| 恢復 | 有 `--continue` 與 `--conversation`，但沒有已驗證窗格恢復契約；使用具確定性的重新啟動。 |
| 模型 | `--model <id>` 使用 `agy models` 的裸目錄 ID，例如 `gemini-3.8-flash-high`；可連線清單未列出要求 ID 時，`bin/fm-spawn.sh` 會拒絕。清單是遠端取得，因此探測在共用硬性時限下與 stdin 分離執行；清單無法連線或卡住時，會附通知並在未驗證模型下啟動。 |
| 推理強度 | `--effort low\|medium\|high`；`xhigh` 與 `max` 依記錄後省略契約保留在工作中繼資料。 |
| 輸入區 | 無邊框的單純 `>` 行；共用分類器依 dead-shell 規則讀為 `unknown`，絕非 `empty`；指導改依原生 Agent 狀態與傳遞頁尾確認，沿用 Cursor 先例。 |

<a id="trust-and-where-the-decision-persists"></a>
## 信任與決策保存位置

每個工作 worktree 都是 agy 未見過的路徑，因此未登錄啟動會停在 `Do you trust the contents of this project?`，預選安全選項 `Yes, I trust this folder`；未回答對話框，會把回合送進 agy 暫存目錄，而非 worktree。
沒有啟動旗標可抑制此對話框，但 agy 接受啟動前寫入船長自身 `~/.gemini/antigravity-cli/settings.json` 的 `trustedWorkspaces` 項目（已真實驗證），因此 `../../../../../bin/fm-spawn.sh` 在啟動前，透過 `../../../../../bin/fm-agy-trust.sh` 預先登錄 worktree，形式與 Claude 相同：輔助工具只接受啟動專案的連結 worktree，會記錄邏輯窗格路徑與解析後形式，因為 agy 比對邏輯 cwd，並保留儲存區所有其他鍵。
啟動後就緒關卡是安全網：若對話框仍出現，以一次 Enter 回答，再要求 busy 判定（Herdr 原生 `working` 狀態，或固定的 `esc to cancel` 行），才讓啟動回報成功；未預先登錄路徑時，在對話框回答前，busy 判定絕不算就緒，因為 Herdr 原生判定可能早於對話框出現。
無法確認簡報在 worktree 執行的窗格，會讓啟動失敗、在工作狀態記錄失敗，並關閉端點。
絕不向仍顯示對話框的窗格發送指導；回報成功的啟動，已完成該關卡。

<a id="credential-precondition"></a>
## 憑證前置條件

已驗證 agy 工作 Agent 在已登入 Google 帳號下執行，沒有匯出金鑰，也沒有對話框。
未觀察未驗證時的失敗模式，因此任何驗證提示或拒絕，都依 `../../../../../AGENTS.md` 第 9 節視為憑證阻礙；修正環境並停用端點，不要繼續往裡輸入。

<a id="detection"></a>
## 偵測

只由祖先鏈偵測：`../../../../../bin/fm-harness.sh` 比對錨定程序名稱 `agy`，絕不使用 `*agy*`。
不提升任何環境標記作為身分：活躍 TUI 上觀察到的 `AGENT=1` 是繼承啟動器值，不是 agy 身分；agy 也不清除繼承的 `CLAUDECODE`，但結構性 agy 祖先現在優先於該保留標記；這由 `../../../../../bin/fm-harness.sh` 判定，不依賴啟動程序自身在啟動邊界清除標記。
`../../../../../bin/fm-session-lock-lib.sh` 的工作階段鎖名稱詞彙已包含 agy，因此 agy 工作階段可持有 home 工作階段鎖。

<a id="worker-busy-state-and-turn-end"></a>
## 工作 Agent 忙碌狀態與回合結束

`../../../../../bin/fm-spawn.sh` 不替 agy 啟用 busy 世代，也不寫附屬檔案，正是因為沒有寫入器能清除預先植入的紀錄。
`fm_busy_agy_tail_busy` 只比對固定的 `esc to cancel` 狀態列，寫死且無環境覆寫；標記不存在時，`fm_busy_classify` 回報 `unknown agy-regex`，不是 idle，因為長回合可能讓標記捲出擷取尾端。
清理不移除任何 agy 專屬內容，因為啟動沒有留下。

<a id="primary-and-secondmate-integration"></a>
## 主要工作階段與二副整合

Antigravity CLI 從 `<workspace>/.agents/hooks.json` 載入 hook 生命週期事件，並從 `.agents/` 本身執行每條指令。
agy 1.4.9 把每個頂層鍵當成具名 hook 規格（firstmate 提供一個 `firstmate`），其中 `SessionStart` 與 `Stop` 是平面處理器清單，`PreToolUse` 則以 `matcher` 與 `hooks` 分組；任何其他頂層鍵，例如 `version` 欄位，都會默默停用整份檔案（已真實驗證）。
PreToolUse 處理器不輸出時允許呼叫；輸出物件但沒有 `decision` 時則拒絕，因此防護只在拒絕時輸出。
SessionStart 執行 `bin/fm-sessionstart-agy.sh`，將工作階段啟動摘要注入模型上下文。
PreToolUse 呼叫 `bin/fm-arm-pretool-check.sh --agy`、`bin/fm-cd-pretool-check.sh --agy` 與 `bin/fm-subagent-pretool-check.sh --agy`。
Stop 執行 `bin/fm-turnend-guard-agy.sh`，在登錄的 28800 秒 hook 逾時下，透過 `bin/fm-watch-arm.sh`（有監督主機的 home 則用 `bin/fm-supervision-host.sh park`）停駐於 home 範圍的監看週期，並傳遞需要處理的喚醒輸出；只有可選 `FM_AGY_PARK_TIMEOUT` 時限，才透過含 `[supervision-park]` 的 `{"decision":"continue","reason":"..."}` 重新啟用；`FM_AGY_TURNEND_LOOP_CEILING`（預設 180）限制 hook 驅動回合數。
透過 `bin/fm-turnend-guard.sh --agy` 強制執行最多三次阻擋的修復提醒，之後以 `{}` 失敗放行。
完整監督協定請見 `../../../../../docs/supervision-protocols/agy.md`。
