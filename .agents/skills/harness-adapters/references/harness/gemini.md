# Gemini CLI

Google 的 `gemini` TUI，於 2026-09-04 在 Linux 上使用 gemini-cli 0.58.0 完成端到端驗證。
啟動形式：`GEMINI_CLI_TRUST_WORKSPACE=true gemini -y "$(cat <brief>)"`。
只驗證為 CREWMATE 與 SCOUT 配接器；`../../../../../bin/fm-spawn.sh` 拒絕使用它啟動二副，因為 `../../../../../docs/supervision-protocols/` 沒有 gemini 喚醒協定。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 忙碌狀態 | 語意來源 `gemini-hook`：`BeforeAgent` 開啟回合，`AfterAgent` 與 `SessionEnd` 關閉。手動中斷也會觸發 `AfterAgent`，因此取消的回合會自行關閉紀錄。 |
| 畫面尾端 | 不是狀態來源，但執行中回合的狀態列有唯一的 ASCII 忙碌字串：`(esc to cancel, <n>s)`，閒置時不存在。旁邊的階段文字由模型產生，每回合不同；旋轉指示器是點字，兩者都絕不作為訊號。 |
| 回合結束 | `AfterAgent` 每回合在最終回應後觸發一次，帶有 `cwd`、`session_id`、`prompt`、`prompt_response`、`stop_hook_active` 與 `transcript_path`。取消回合的 `prompt_response` 為 `[no response text]`。 |
| 結束 | `/quit`（別名 `/exit`），一次 Enter，結束狀態 0；印出 `To resume this session: gemini --resume <session-id>`。`Ctrl+C` 會取消，或在空白輸入時離開；`Ctrl+D` 在空緩衝區時結束。 |
| 中斷 | 按一次 `Escape`，印出 `ℹ Request cancelled.`，Agent 仍持續執行。輸入區不會再次污染，而是回到 `Type your message or @path/to/file` 預留文字。 |
| 技能 | `/<skill>`，例如 `/no-mistakes`；**一次** Enter 即送出，不會被彈出選單吃掉，回合會以 `Activate Skill` 工具呼叫開始。 |
| 自主性 | `-y` / `--yolo`，頁尾為 ` YOLO Ctrl+Y`，已透過實際檔案寫入驗證可無人值守且無核准關卡；`--approval-mode yolo` 是等效長形式。 |
| 識別標記 | 子程序與工具程序帶有 `GEMINI_CLI=1`。`AI_AGENT` **不是** Gemini 身分；見下方偵測。 |
| 恢復 | `gemini --resume <session-id>` 還原完整歷史；也接受 `--resume latest` 與索引，`--list-sessions` 可按專案列舉。 |
| 模型 | `-m` / `--model <model>`；透過互動式 `/model` 對話框探索。沒有 `gemini models` 子命令，工作階段結束時的用量表也會列出實際使用模型。 |
| 推理強度 | 無。0.58.0 的 `gemini --help` 沒有 effort、reasoning 或 thinking 旗標，因此適用 `references/common/model-and-effort.md` 的記錄並省略契約。`thinkingLevel` 與 `thinkingBudget` 只存在於 `settings.json` 的生成設定，**不是**已驗證的互動軸。 |

<a id="trust-and-why-the-two-documented-options-are-not-equivalent"></a>
## 信任，以及兩個文件化選項為何不等效

每個任務 worktree 都是 Gemini 從未見過的路徑，未處理的啟動會直接拒絕：
`Gemini CLI is not running in a trusted directory. To proceed, either use --skip-trust, set the GEMINI_CLI_TRUST_WORKSPACE=true environment variable, or trust this directory in interactive mode.`
無介面模式下，該拒絕以 55 結束。

CLI 將這兩個選項描述為等效，但實際不同。
在同一 worktree 的受控 A/B 比較中，設定 home、提示都相同，只改信任機制，結果 `--skip-trust` 會執行回合，卻不載入**專案**設定，因此專案自己的 hooks 不觸發，`.agents/skills` 也不被探索；`GEMINI_CLI_TRUST_WORKSPACE=true` 則兩者都載入。
firstmate 儲存庫任務正需要這些工作區技能，因此 spawn 使用環境變數，絕不可改用 `--skip-trust`。
Firstmate **自己的**忙碌 hooks 不依賴這一點，因為它們使用下述系統設定層。
信任工作區會載入專案的 `.gemini/settings.json`、hooks、MCP 伺服器與技能，這與其他配接器在任務 worktree 中既有的行為一致。

互動信任對話框為 `Do you trust the files in this folder?`，有三個選項。
與 Claude 不同，其預選是**安全**選項：`● 1. Trust folder (<name>)`，而 `2. Trust parent folder (<parent>)` 與 `3. Don't trust` 未選取。
接受會持久化至 `~/.gemini/trustedFolders.json`，因此偏好 spawn 的環境變數：它只適用單一工作階段，不會留下持續增長的可拋棄 worktree 路徑全域紀錄。

<a id="credential-precondition-and-the-wedge-it-causes"></a>
## 憑證前置條件與造成的卡住

Gemini 工作 Agent 需要無須對話框就能使用的憑證，firstmate 不負責管理。
在工作階段供應者 daemon 啟動**之前**，將 `GEMINI_API_KEY` 匯出到環境，或完成 `gemini` 自己的登入。
daemon 是關鍵：長時間執行的 tmux 或 Herdr 伺服器會將啟動時的環境傳給窗格，因此伺服器啟動後才匯出的金鑰，永遠不會傳到工作 Agent。
無法解析任何憑證時，無介面探測 `gemini --skip-trust -p '<prompt>'` 會以 41 結束，並顯示 `you must specify the GEMINI_API_KEY environment variable`，這是派工前成本最低的確認。
首次執行也會顯示驗證方式選擇器（`How would you like to authenticate for this project?`，預設 `● 2. Use Gemini API Key`）；回答一次會將 `security.auth.selectedType` 寫入使用者 `settings.json`，之後就不再出現。

沒有憑證時，窗格會卡在 `Enter Gemini API Key` 對話框，此對話框有兩種不同風險。
只要有值，它就會在窗格**以明文顯示金鑰**，任何擷取或偵錯日誌都可能保留；啟動簡報則在其後以 `API Error: Content generator not initialized` 失敗。
更糟的是，這個憑證欄位會接受接下來輸入的任何內容：對卡住的窗格送一般結束命令，會把 `/quit` **送進欄位**，並持久化為 `~/.gemini/gemini-credentials.json` 中的憑證。
這會污染該機器之後每次執行：原本沒有憑證時以 41 乾淨失敗，之後卻會真的呼叫 API，並在每個請求以 `API key not valid` 失敗；只能清除該儲存憑證來修復。
因此，gemini 窗格顯示此對話框時，絕不送入生命週期文字。
依 `../../../../../AGENTS.md` 第 9 節將其視為憑證阻擋，修正環境並退役端點，不要對提示輸入。

**不要**給工作 Agent 隔離的 `GEMINI_CLI_HOME`。
這會隱藏 `~/.agents/skills`，讓 `/no-mistakes` 與所有其他使用者技能從該工作 Agent 默默消失。

<a id="detection"></a>
## 偵測

`GEMINI_CLI=1` 是必要依據，不只是快速路徑，因此 `../../../../../bin/fm-harness.sh` 會在 `CLAUDECODE` **之前**檢查。
Gemini 不會清除繼承的 `CLAUDECODE`，所以 claude 主 Agent 下的 gemini 工作 Agent 同時帶有兩種標記，先測試者勝出；spawn 另外也會在啟動邊界清除其他標記。

祖先鏈無法彌補缺口。
發布的 CLI 是 node bundle（`~/.local/bin/gemini` -> `@google/gemini-cli/bundle/gemini.js`），而 Linux 的現代 Node 將 `comm` 回報為 `MainThread`，不是 `node`（於 Node v24.20.0 測量），因此命令名稱分支與直譯器分支都無法比對實際 gemini 程序。
不要以比對 `MainThread` 修補：這會讓每個 node 程序的引數都可被搜尋，讓無關命令也能宣稱身分。
`../../../../../tests/fm-gemini-harness.test.sh` 固定了標記優先序與此祖先鏈邊界。

絕不將 `AI_AGENT` 提升為標記。
同一已驗證工具程序帶的是 CLAUDE 主 Agent 的值（`claude-code_2-1-260_agent`），因此它識別啟動者，而非正在執行的執行環境。

窗格存活判定有同樣問題，且需自己的解法，因為程序掃描看不到環境標記。
實際 gemini 窗格的前景群組為 `comm=MainThread`、`argv0=<node path>`，所以 `bin/backends/tmux.sh` 既有兩種名稱來源都看不到它；修補前，`bin/fm-control.sh` 對每個生命週期動詞都以 `endpoint reads 'ambiguous'` 拒絕。
`../../../../../bin/fm-gemini-lib.sh` 負責修正用的狹義結構規則：身分來自 argv[1]，也就是指令碼引數，只有名稱為 `gemini` 或位於 `@google/gemini-cli/` 下才接受。
這是結構判定，不執行任何子程序，原因與 cursor 規則相同：要避免的風險就是在存活輪詢時探測陌生執行檔。
裸直譯器、無關 node 指令碼，以及只在命令列較後方出現 gemini 名稱，都會被拒絕，所以陌生 node 窗格絕不回報為存活 Agent。

<a id="worker-busy-state-and-turn-end"></a>
## 工作 Agent 忙碌狀態與回合結束

`../../../../../bin/fm-spawn.sh` 會在 `state/<id>.gemini-settings.json` 寫入 firstmate 擁有的個別任務設定檔，包含三個綁定新建忙碌世代的 hooks，啟動透過 `GEMINI_CLI_SYSTEM_SETTINGS_PATH` 存取。
此接線只屬於標準、精確的 `gemini` 配接器範本，它同時取得忙碌狀態接線、回合結束 hook 與可信忙碌狀態。
原始 Gemini 形式的啟動是未驗證的逃生口：不取得忙碌狀態接線或回合結束 hook，因此沒有可信忙碌狀態。
刻意**不是** worktree 的 `.gemini/settings.json`：與 Claude 的 `settings.local.json` 不同，該路徑是**專案自己已提交的設定檔**，寫入會覆蓋專案設定，退役則會刪除受追蹤檔案。
Gemini 各設定層的 hook 陣列會**合併**而非覆寫，因此專案自己的 hooks 仍與 firstmate 的一同執行；已觀察到單一回合兩者都觸發。
`../../../../../bin/fm-teardown.sh` 會移除此檔，因此不會殘留到回收池 worktree。
`BeforeAgent` 記錄忙碌，`AfterAgent` 記錄閒置，並保留 `state/<id>.turn-ended` touch 作為監看器**通知**，`SessionEnd` 則記錄閒置，避免異常結束留下忙碌紀錄。
每個 hook 命令都會印出 Gemini hook 契約要求的空 JSON 物件，並容忍事件被拒絕，讓過期世代寫入器絕不破壞 Gemini 自己的生命週期。

接線刻意處理兩個特殊行為。
已觀察到一次 `/quit` 會觸發**兩次** `SessionEnd`；重複閒置事件具有冪等性，不去重。
手動 Escape 中斷與正常完成都會觸發 `AfterAgent`，這比 Claude 好；Claude 中斷不發出 hook，通常會留下 `claude-hook` 忙碌狀態。

系統設定層也讓忙碌契約與信任決策無關：已驗證其 hooks 在不受信任資料夾搭配 `--skip-trust` 仍會觸發，不需要 Gemini 逐工作區 `~/.gemini/trusted_hooks.json` 的項目，該檔只記錄**專案** hooks。
因此工作區信任帶來的是技能，不是狀態。
也已證明 `~/.gemini/settings.json` 的帶防護使用者層級 hook 可運作，以 grok 形式的 worktree 指標與私有權杖登錄表設關卡；但此做法被否決，因為會修改船長自己的全域設定，影響機器上每個工作階段。

hook 執行中，狀態列會顯示 `Executing Hook: <name>`，而 `(esc to cancel,` 字串已消失，因此這個短暫時間窗會讀為閒置；此時回合確實已結束。

<a id="skills"></a>
## 技能

Gemini 從 `~/.gemini/skills/` 或 `~/.agents/skills/` 探索使用者技能，從 `.gemini/skills/` 或 `.agents/skills/` 探索工作區技能。
因此 `~/.agents/skills/no-mistakes` 被探索為使用者技能，即使在不受信任資料夾也能載入，讓 firstmate 送達路徑保持可用。
工作區技能需要啟動已授予的工作區信任，這讓 firstmate 儲存庫任務自己的 `.agents/skills` 可被存取。
Gemini **不讀取** `.claude/skills`。

<a id="primary-integration"></a>
## 主 Agent 整合

不支援，也未驗證。
`../../../../../docs/supervision-protocols/` 沒有 gemini 協定，沒有對應回合結束防護配接器；此配接器只驗證船員側啟動、忙碌狀態、中斷與結束。
適用 `references/common/primary-hooks.md` 的不支援邊界規則：絕不從相似 TUI 自行杜撰喚醒協定。
Gemini 的 `BeforeAgent`/`AfterAgent` 事件對與 `gemini hooks migrate` 命令，讓未來主 Agent 整合看似可行，但這仍是尚未建置的工作，不是可依賴的事實。
