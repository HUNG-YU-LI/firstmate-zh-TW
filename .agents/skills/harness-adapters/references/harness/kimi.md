# Kimi Code

於 2026-09-17 使用 Kimi Code CLI 2.0.0 驗證。
資料夾提示於 2026-10-23 使用 Kimi Code CLI 2.3.0 與 2.3.1 再次記錄，這兩版改了用詞；`tests/fm-kimi-trust-prompt-live-e2e.test.sh` 會針對已安裝執行檔重新取得證據。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 執行檔 | 先解析 `PATH` 中的絕對執行檔路徑，再嘗試可執行的 `$HOME/.kimi-code/bin/kimi`；兩者皆不存在就拒絕建立工作 Agent。 |
| 啟動 | 以 `--auto` 啟動空白互動 TUI，之後通過就緒關卡才送達指標；位置參數提示會被拒絕。 |
| 模型 | 觀察到的預設值有 `kimi-code/kimi-for-coding`、`kimi-code/kimi-for-coding-highspeed`、`kimi-code/k3` 與 `kimi-code/k3-256k`；用 `kimi provider list --json` 查詢目前設定。 |
| 忙碌狀態 | 獨立 Kimi 在語意來源完成實際驗證前維持未知，優先考慮 Wire 的 `prompt` 生命週期，其次是包括 `Interrupt` 在內的文件化 hooks；Pi 後方的 Kimi 使用 Pi 生命週期，月相旋轉指示器絕不是狀態來源。 |
| 結束命令 | `/exit`。 |
| 中斷 | 按一次 Escape，會印出 `Interrupted by user`。 |
| 技能呼叫 | `/<skill>`，例如 `/no-mistakes`；會探索 Firstmate 技能。 |
| 自主性 | `--auto` 是 `Never Ask` 層級；`-y` 與 `--yolo` 現在會選擇不同且較弱的 `Ask When Needed` 層級，因此不使用。 |
| 信任對話框 | 新 worktree 會顯示 `Trust this folder?`，並預先選取肯定選項；spawn 會讀取可見窗格，辨識完整對話框（標題、兩個導覽提示字串 `↑↓ navigate` 與 `Enter select`——分開比對，讓窄窗格中換行的提示仍有效——以及一組已記錄選項：2.0.0 的 `❯ Trust this folder` 與 `Don't trust`，或 2.3.0 的 `❯ Trust and continue` 與 `Exit`），每次輪詢仍看到完整對話框時都送出 Enter，確認後續可見窗格擷取已不含該對話框，再繼續一般就緒關卡。若 2.3.0 提示宣告專案 MCP 伺服器（`Config: <file>` 上方的 `Start N MCP server(s) automatically`），絕不回答，因為 Enter 會在任何登入之前啟動伺服器，而 Kimi 的逐資料夾信任並不涵蓋對它們的同意；啟動會失敗並明確說明原因，由操作人員從 `Config:` 行指定的檔案移除伺服器。若提示將揭露內容分頁（`←→ page · N / M`），也不送任何按鍵，因為宣告可能位於未顯示的頁面；因終端機高度不足而收合的提示，也絕不視為完整對話框。判定條件由 `bin/fm-kimi-trust-lib.sh` 負責，記錄過的用詞由 `tests/captures/kimi-code-2.3.0/README.md` 負責。絕不在 `config.toml` 預先登錄信任；只在現場回答對話框。 |
| 斜線送出 | 一次 Enter 就送出，不會被彈出選單吃掉，也沒有穩定等待風險。 |
| 環境標記 | 無；身分來自祖先程序命令名稱 `kimi`，`../../../bin/fm-harness.sh` 會防止保留的其他標記覆蓋它。 |
| 輸入區 | 有邊框方框，只有 `>` 提示符號，未觀察到幽靈文字或預留文字。 |
| 推理強度 | `kimi provider list --json` 提供各模型的 `supportEfforts` 值 `low`、`high`、`max`，以及 `defaultEffort`；啟動旗標與對應仍未驗證，因此 spawn 依 `references/common/model-and-effort.md` 記錄但省略要求的推理強度。 |

<a id="readiness-gated-start"></a>
## 通過就緒關卡才開始

`../../../bin/fm-spawn.sh` 會先空白啟動 Kimi，若出現 `bin/fm-kimi-trust-lib.sh` 能辨識措辭的完整信任對話框，就處理它；接著等待輸入區方框或 `Welcome to Kimi Code!`，只送入 `Read the brief at <absolute-path> and follow it exactly.`，並要求輸入區已清空，且有回顯的 `✨` 送出訊號或非零上下文，才接受已送達。
Kimi 2.3.0 除了 `.kimi-code/mcp.json`，也會將 git worktree 根目錄的 `.mcp.json` 當作專案 MCP 宣告讀取，但其文件只提到前者；提示會在 `Config:` 行指出實際宣告伺服器的檔案，因此拒絕判定應讀取提示，而非任何檔案。
每個信任判定都讀取 `fm_backend_visible_capture`——不含回捲內容的可視區——絕不讀取送達關卡使用的 120 行歷史：對話框是 TUI 畫面，若擷取包含歷史，Kimi 重繪離開對話框後仍會持續回報它，造成對作用中輸入區連續灌入 Enter，最後讓已受信任的啟動失敗。
`bin/fm-backend.sh` 的 `FM_BACKEND_VISIBLE_CAPTURE` 是能力清單的權威來源；[`docs/rex-backend.md`](../../../../../docs/rex-backend.md#current-operation-and-safety) 說明 Rex 的推導可視區。
Orca 只能讀歷史；cmux 不帶 `--scrollback` 的 `read-screen` 看似只讀可視區，但尚未實際驗證。因此，在 cmux 完成驗證前，這兩個後端的 Kimi 啟動都會在前置檢查拒絕，此時 worktree 與窗格尚未建立，並指出後端與缺少的已驗證可視區能力。沒有回退到回捲讀取的途徑。可視區讀取若以非零狀態結束，就立即判定就緒失敗並指出後端，不會誤認為空白畫面。成功但空白的可視區讀取代表缺乏證據，不代表對話框已清除：該次輪詢不計入，重設下述兩次擷取就緒計數，並保留原有信任診斷。信任回答會持續重試，直到對話框清除——Kimi 在啟動期間可能吃掉按鍵，因此單次 Enter 可能遺失——而重送的前提是同一可見窗格仍有完整對話框，所以對話框清除後就不會再觸發。只有後續可見窗格擷取證明對話框已清除，才接受信任；對話框若卡住，失敗診斷會包含觀察到的對話框訊號與回答次數。
可見窗格只要出現任何一個對話框標記——`Trust this folder`、`Trust and continue` 或否定選項 `Don't trust`——就不會判定就緒，因為重繪中途或只畫出方框標題的擷取都可能缺少完整對話框，而上方橫幅原本會被讀成就緒。橫幅也會在對話框開始繪製前出現，任何單次擷取都無法區分這種情況與真正就緒的窗格，因此判定還要求連續兩次擷取都就緒且沒有對話框文字；任何未就緒或空白的擷取都會重設計數，避免橫幅出現前的開機畫面與重繪影格耗用這個計數。
這種先啟動再傳送的形式是必要的，因為 Kimi 會將位置參數指示當成未知命令拒絕。
路徑必須是絕對路徑，因為指示位於任務 worktree 之外，Kimi 不需 `--add-dir` 就能在該處讀取。

已重現就緒前傳送會靜默遺失的情況：結束碼為零、輸入區為空、`context: 0%`、沒有回顯使用者訊息，而且窗格看起來健康且閒置。
已確認原因是啟動期間的輸入就緒時間窗，而非橫幅。
太早按 Enter 可能讓輸入區擴展成多個內容列，使指標文字留在第一列，而游標位於後面的空白列。
因此，共用 tmux 讀取器會定位完整的有框線輸入區，並將任何內容列上的實際文字視為仍待送出的明確證據。
此時間窗內沒有任何繪製訊號能證明 Kimi 已可接受輸入，因此送達會透過共用送出核心重試 Enter，並保留後置條件驗證，而非放寬就緒判定。

觀察到的旋轉指示器擷取包含選用的前導空白、月相符號、`·` 周圍的空白，以及輪替提示文字，工具執行時也會出現。
僅用於送達的比對器要求已觀察到的空白，刻意排除未觀察到的零空白形式，且不要求尾端提示文字。
Kimi 閒置時的頁尾提示可能顯示 `ctrl+c: cancel`，閒置列也可能將小寫 `thinking` 作為推理強度標籤。
兩者都不是忙碌狀態來源。
僅用於送達的旋轉指示器比對涵蓋完整月相符號集，但仍對語系與 emoji 字型敏感，因為 Kimi 沒有穩定的 ASCII 忙碌字串。

<a id="crew-turn-end-hook-and-primary-limit"></a>
## 船員回合結束 hook 與主 Agent 限制

Kimi 不在主 Agent 回合結束防護範圍內。
它獨立的全域 hook 介面與經船長核准的船員喚醒整合，由 `../../../docs/turnend-guard.md` 負責。

`../../../bin/fm-spawn.sh` 會在 `$HOME/.kimi-code/config.toml` 安裝一段以標記分隔的 Firstmate 項目、一支靜默且一律回傳零的 hook 指令碼，以及 `$HOME/.kimi-code/fm-turn-end.d/` 下的一份私有權杖登錄表。
每個 Kimi 工作 Agent 的 worktree 都會收到被 gitignore 忽略的 `.fm-kimi-turnend` 指標檔。
只有 Stop 承載資料的 `cwd`、指標檔與登錄項目全部一致時，全域 hook 才會 touch `state/<id>.turn-ended`。
帶防護的靜默 hook 無法從「沒有作用」驗證，因此在斷定它未觸發前，先用沒有防護的探測證明有呼叫。
帶防護的回合結束訊號仍只是喚醒通知。
在來源完成實際驗證前，獨立 Kimi 沒有忙碌狀態來源。
