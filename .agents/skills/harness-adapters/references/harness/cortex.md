# Cortex Code

於 2026-10-12，在 Linux 上透過 tmux 與具名 Herdr 0.8.2 實驗工作階段，以 Cortex Code 1.1.104（`cortex`，建置版本 1.1.104+041358.66de1c44636f）驗證。
同日也在 1.2.14 通過即時工作 Agent 防護檢查；Herdr 實驗檢查只在 1.1.104 執行。
路由器負責限定僅供 crewmate/scout 使用；不支援主 Agent 與二副整合。
[驗證證據](../../../../../docs/verification/cortex.md)及其即時防護檢查會更新下列供應商事實。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 忙碌狀態 | 原生 `UserPromptSubmit` 開啟忙碌狀態，`Stop` 在正常完成與 Escape 中斷時都會關閉，`SessionEnd` 則在關機時關閉，全部透過綁定世代的寫入器處理；信任判定由 `../../../../../bin/fm-busy-lib.sh` 負責。 |
| Hook 通道 | 透過 `--plugin-dir` 傳入個別任務的外掛目錄；`--config <settings.json>` 會取代使用者設定檔，但絕不提供 hooks，也沒有旗標能停用使用者或專案的 hook 檔案。 |
| 結束命令 | `/quit`，按 Enter 前先套用共用的斜線命令穩定等待；會印出含有 `cortex --resume <session-id>` 的摘要。 |
| 中斷 | 一次 Esc 會取消執行中的回合，並印出 `Interrupted`；在閒置輸入區按一次或兩次 Esc 都不會有作用，重複按也不會結束程式。Ctrl+C 也會取消執行中的回合，但不使用它。 |
| 技能呼叫 | 在輸入區輸入 `/<skill>`；斜線彈出選單會將 `~/.claude/skills` 的使用者技能與 Cortex 本身的命令列在一起，但尚未測試透過 `fm-send` 端到端呼叫技能。 |
| 恢復 | `cortex --resume <session-id>` 會還原對話，並以 `source: resume` 觸發 `SessionStart`；`--continue` 與 `cortex resume --last` 有列在 `--help`，但尚未測試。互動工作階段會忽略 `--session-id`，因此 ID 必須從結束摘要或 hook 承載資料讀取，不可自行指定。 |
| 模型旗標 | 使用 `--model <id>`，ID 取自模型不可用錯誤列出的清單，例如 `claude-sonnet-5-5`、`claude-haiku-5-5`，或使用 `auto`。 |
| 推理強度旗標 | `--effort minimal\|low\|medium\|high\|max`；`xhigh` 沒有原生對應層級，因此省略。 |
| 模型探索 | 在輸入區使用 `/model`，或查看無效 `--model` 在第一個提示產生的錯誤；沒有列出模型的子命令。 |
| 識別標記 | 無；明確錨定的原生 `cortex` 祖先程序用來識別配接器，優先於繼承的其他執行環境標記。 |
| 信任對話框 | `--dangerously-allow-all-tool-calls`（別名 `--bypass`）也會略過 `Do you trust this project?` 對話框，且不會寫入 `~/.snowflake/cortex/cortex.json`；專案 hooks 與設定仍會載入。 |
| 匯入設定 | Cortex 除了自己的 `.cortex/` 檔案，也會載入相容 Claude 的 `~/.claude/settings.json`、專案 `.claude/settings.json` 與 `settings.local.json` hooks，因此在含 Claude hooks 的專案中執行工作 Agent 時，這些 hooks 也會執行；沒有停用匯入的開關。 |
| 提交署名 | `Generated with Snowflake CoCo` 行與 `Co-authored-by: Snowflake CoCo <noreply@snowflake.com>` 尾註是模型寫入的，不是用戶端設定；除非 home 設定了 `config/keep-ai-trailers`，否則 spawn 負責的 commit-msg 移除器（`../../../../../bin/fm-git-strip-ai-trailers.sh`）會移除兩者。 |

<a id="worker-lifecycle-limits"></a>
## 工作 Agent 生命週期限制

Claude 專案 hooks 也會在 Cortex 工作 Agent 中執行，且無法關閉。因此，若專案的 `.claude/` hooks 假設執行環境為 Claude Code，選擇此配接器前應先評估這一點。
外掛的 `SessionStart` hook 會在啟動時、第一個提示之前觸發，Firstmate 只用它向 Herdr 註冊窗格；忙碌狀態來自 `UserPromptSubmit`、`Stop` 與 `SessionEnd`。
如果回合因模型錯誤（例如模型不可用）結束，輸入區會進入閒置；尚未觀察到這種情況是否會觸發 `Stop`，因此若工作 Agent 顯示錯誤卻仍有忙碌紀錄，應依過期工作 Agent 處置流程處理，不應視為 hook 故障。
`--sql-read-only` 與模式列的 `guard:` 欄位是 SQL 寫入防護，不是工具權限設定；啟動不會傳入此旗標，因此 SQL 寫入政策仍沿用船長自己的 Cortex 設定，而啟動環境中的 `CORTEX_CLIENT_READ_ONLY=1` 會由原生機制套用。
Herdr 0.8.2 的偵測清單沒有 Cortex 項目（herdrdev/herdr discussion 783），因此沒有 hook 的 Cortex 窗格會回報 `agent_not_found`，存活探測會將其分類為 `dead`，也會讓引導門鈴略過該端點。
因此當 `HERDR_ENV=1` 時，個別任務外掛的 hooks 會向 Herdr 將窗格回報為來源 `firstmate:cortex` 的 `cortex` Agent。
由於 Cortex 會載入 `~/.claude/settings.json`，只要設定了 `CLAUDECODE=1`，Herdr 自己的 claude 整合 hook 就會將窗格認領為 `claude` 工作階段；該認領在 Cortex 結束後仍然存在，並默默丟棄該窗格之後的所有回報，因此啟動時一律清除 `CLAUDECODE`。
認領存在時，結束後 `agent get` 仍會回覆 `agent_not_found`，重新啟動也無法註冊，最後只剩語意忙碌紀錄能作為狀態真實來源。

`../../../../../bin/fm-spawn.sh` 負責自主性、信任、以輸入方式送達任務簡報、環境清理，以及省略 Claude 權限模式對應。
`../../../../../bin/fm-cortex-plugin.sh` 負責私有外掛目錄與生命週期 hooks；使用者和專案設定仍由供應商負責，保持不變。

<a id="composer-and-steering"></a>
## 輸入區與引導

`../../../../../bin/fm-composer-lib.sh` 負責已驗證的有框線 `›` 輸入區，其圓角頂框會顯示工作階段標題，閒置預留文字為 `Type your message...`（回合執行中為 `Type to queue message...`），畫面先繪製一格反白游標，再顯示淡色文字。分類器只有在呼叫端證明窗格前景程序是 Cortex 時，才接受這種帶標題的頂框（`cortex=1` 能力：tmux 的精確程序名稱、Herdr 的原生 Agent 探測）；其他窗格上的帶標題方框仍判為不明確。
輸入框下方的 `>> bypass safeguards  |  teams: off  |  plan: off ...` 模式列是固定介面元素，其文字隨終端機寬度調整。
Cortex 會把終端機游標停在頁尾下方空白列的第一欄，因此在 tmux 上，游標錨定讀取找不到輸入區；`../../../../../bin/fm-tmux-lib.sh` 對程序名稱恰為 `cortex` 的前景程序，會改為不依賴游標讀取窗格；Herdr 本來就不依賴游標讀取。
`../../../../../bin/fm-spawn.sh` 拒絕在 zellij、cmux、orca、paseo、rex 與 t3code 上啟動 cortex，因為這些後端無法提供 Cortex 窗格身分，無法證明帶標題的輸入區為空。
回合執行中輸入的文字會排入佇列，列在 `Queued (↑ to edit, Enter to send next):` 下方；以 Escape 中斷後，佇列訊息便會開始下一個回合。
回合結束後的淡色後續建議（`push it`）是幽靈文字，不是草稿。
`../../../../../bin/fm-task-inbox-lib.sh` 的門鈴已透過 tmux 上實際的 `fm-send` 完成讀取與確認；結束與重新啟動也已透過 Herdr 實驗窗格上實際的 `fm-control` 驗證。

<a id="primary-integration"></a>
## 主 Agent 整合

尚未驗證 Cortex 的主 Agent Stop 防護、監看器協定、工具執行前保護或工作階段啟動契約。
不要使用此配接器啟動主 Agent 或二副。
ACP、Cortex 自己的 worktree 與 teams 模式，以及配額供應商整合，仍為獨立的後續工作。
