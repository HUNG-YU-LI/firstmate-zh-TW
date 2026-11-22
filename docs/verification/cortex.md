<a id="cortex-code-worker-verification"></a>
# Cortex Code 工作 Agent 驗證

讀者：維護者驗證。

於 2026-10-12 在 Linux x86_64 使用 Cortex Code `1.1.104`（建置 `1.1.104+041358.66de1c44636f`）驗證，並在同一天以 `1.2.14` 重跑真實驗證防護。
[轉接器參考文件](../../.agents/skills/harness-adapters/references/harness/cortex.md) 負責操作事實；負責執行的程式則承載啟動與狀態機制。
本驗證涵蓋船員（crewmate）與偵察員（scout），以 tmux 作為實際測試的執行期後端，並使用具名 Herdr 0.8.2 實驗工作階段進行下列窗格身分及生命週期檢查。
主要 Agent、二副、ACP、Cortex 自己的 worktree 與 teams 模式，以及配額供應商整合，不在此保證範圍內。

<a id="refresh-commands"></a>
## 更新驗證的指令

```sh
cortex --version
cortex --help
bin/fm-test-run.sh tests/fm-cortex-harness.test.sh
FM_CORTEX_SIGNALS_LIVE=1 bin/fm-test-run.sh tests/fm-cortex-signals-live-e2e.test.sh
FM_CORTEX_SIGNALS_LIVE=1 FM_CORTEX_MODEL=claude-sonnet-5-5 bin/fm-test-run.sh tests/fm-cortex-signals-live-e2e.test.sh
```

使用憑證的驗證防護，會使用呼叫端自己的 Cortex 連線、私人 tmux socket，以及 `fm-spawn.sh` 實際產生的指令。
它會以少量簡短提示詞消耗呼叫端的 Snowflake 額度，不要求任何 SQL，也不指定資料庫物件。
worktree 配置與初始端點投遞使用可攜式測試夾具；後續引導及生命週期控制，則使用真正後端及供應商程序。
若啟動將專案新增到 Cortex 全域信任紀錄，驗證就會失敗。
未登入時會略過；工具不存在及選擇性啟用的行為，由共用真實測試關卡負責。
失敗訊息會指出已安裝的 Cortex 版本。

<a id="live-guard-results"></a>
## 真實驗證防護結果

2026-10-12，驗證防護在 `1.1.104`、`claude-haiku-5-5` 下以 0 結束：

```text
ok - cortex Cortex Code v1.1.104: spawn brief, model, autonomy, trust skip, identity and native Stop
ok - cortex Cortex Code v1.1.104: the worker commit carries no attribution through the spawn-owned strip
ok - cortex Cortex Code v1.1.104: real fm-send doorbell read and acknowledged
ok - cortex Cortex Code v1.1.104: one Escape cancels the turn, keeps the agent, and Stop settles the busy record
ok - cortex Cortex Code v1.1.104: /quit and native --resume session resume
```

已安裝的 Cortex 自行更新到 `1.2.14` 後，相同驗證防護再次執行 63 秒，以 0 結束；五行 `ok` 內容相同，但版本標示為 `Cortex Code v1.2.14`。
下方 Herdr 實驗檢查只在 `1.1.104` 執行，沒有於 `1.2.14` 重跑。

<a id="observed-vendor-surfaces"></a>
## 觀察到的供應商介面

版本輸出：

```text
Cortex Code v1.1.104
```

`--dangerously-allow-all-tool-calls`（別名 `--bypass`）能處理放在 `--` 之後的位置引數初始提示詞，並在沒有權限提示的情況下執行 shell 工具。
同一旗標會略過 `Do you trust this project?` 對話框，且不寫入 `~/.snowflake/cortex/cortex.json`。
未使用該旗標時，對話框提供 `1. Yes, proceed` 與 `2. No, exit`，回答 no 會結束程序。
輸入區下方頁尾隨後顯示 `>> bypass safeguards | teams: off | plan: off | guard: on`。
`guard` 是 SQL 唯讀設定（`--sql-read-only`、`CORTEX_CLIENT_READ_ONLY=1`），啟動時並未傳入它。

此建置讀取的 hook 來源包括：使用者 hook 檔案 `~/.snowflake/cortex/hooks.json`、`~/.claude/` 與 `~/.cortex/` 下的 `settings.json`、專案 `.claude/` 與 `.cortex/` 的 `settings.json` 及 `settings.local.json`，以及透過 `--plugin-dir` 傳入的外掛目錄。
外掛目錄必須在 `hooks/hooks.json` 旁具有 `.cortex-plugin/plugin.json` manifest，才會載入。
專案 hooks 不需授予信任就會載入。
`--config <file>` 會取代使用者 `settings.json`，其 `hooks` 隨之不再讀取；若缺少主題或連線，則會重新開啟初始設定對話框。
`--setting-sources` 不會限制 hooks。
`SNOWFLAKE_HOME` 會搬移 `~/.snowflake` 的位置，真實驗證防護不使用它。

外掛的原生 hook 日誌記錄以下 payload 欄位，每一筆都含 `session_id`、`transcript_path`、`cwd` 與 `hook_event_name`：

```text
SessionStart source=startup
UserPromptSubmit prompt=<text>
Stop stop_hook_active=false
SessionEnd reason=prompt_input_exit
```

正常完成與 Escape 中斷都會觸發 `Stop`。
每次 `Stop` 後，產生的 hooks 都會寫入 `idle cortex-hook` 語意狀態及回合結束通知。
互動式工作階段會忽略 `--session-id`，而每個 payload 的 session id 都等於結束摘要印出的 id。

執行中的回合會呈現旋轉指示器列，例如 `⠙ Snowdiving...  (esc to interrupt)`、輸入區占位文字 `Type to queue message...`，以及以 `view: compact • Type to queue next message • esc to i` 開頭的頁尾。
閒置輸入區是圓角框，上框線帶有工作階段標題，占位文字為 `Type your message...`，頁尾為 `view: compact • ? for help •`。
回合執行期間鍵入的文字會列在 `Queued (↑ to edit, Enter to send next):` 之下；Escape 中斷會讓佇列中的訊息接著成為下一回合。
框下暗色後續建議（`push it`）是幽靈文字。
Ctrl+U 可清空輸入區。
Cortex 把終端機游標停在頁尾下方空白列的第 0 欄，因此 tmux 依游標定位的讀取無法找到輸入區；對前景程序名稱精確為 `cortex` 的窗格，分類器會改用不依賴游標的讀取。
只有窗格前景程序已證明為 `cortex`（tmux 精確程序名稱、Herdr 原生 Agent 探測）時，帶標題的圓角上框線才被接受為輸入區；其他窗格的相同框體仍視為有歧義。
`NO_COLOR=1` 下仍能把空白輸入區分類為 empty。

對執行中回合按一次 Escape，會取消回合，印出指示 Cortex Code 接下來該做什麼的 `Interrupted` 列；`Stop` 會觸發，程序仍保留。
在閒置輸入區按一次或連按兩次 Escape，都沒有作用。
Ctrl+C 也會取消執行中的回合，閒置時沒有可見效果；轉接器不使用它。
`/quit` 後接 Enter，配合共用斜線指令彈出選單穩定處理，會返回 shell，並印出 `cortex --resume <session-id>` 與 `cortex --resume last`。
`cortex --resume <session-id>` 可恢復對話，並觸發 `SessionStart source=resume`。

`--model <id>` 接受此帳號在模型不可用錯誤中列出的 id（包括 `claude-sonnet-5-5`、`claude-haiku-5-5`、`claude-opus-4-6`、`auto`、`auto-fast` 等）；未知 id 不會在啟動時拒絕，而是在第一個提示詞後才報錯。
`--effort` 接受 `minimal`、`low`、`medium`、`high` 與 `max`。

<a id="commit-attribution"></a>
## 提交署名

模型會在自己撰寫的提交訊息中加入 `Generated with [Snowflake CoCo](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code)` 與 `Co-authored-by: Snowflake CoCo <noreply@snowflake.com>` trailer；內建 `/commit` 指令則寫入 `Generated with [Cortex Code](https://docs.snowflake.com/user-guide/snowflake-cortex/cortex-agents)`。
沒有任何用戶端設定能關閉兩者，因此由 spawn 管理的 commit-msg 清除機制是唯一控制點；真實驗證防護的工作 Agent 提交兩者皆無。
清除機制不會編輯模型放在 PR 本文的文字。

<a id="herdr-lab-session"></a>
## Herdr 實驗工作階段

在具名 Herdr 0.8.2 實驗工作階段中，保留船長真正的 `~/.claude/settings.json` hooks，以及船長自己的 Cortex Herdr hook，觀察到：

```text
unguarded launch (CLAUDECODE=1 inherited):
  agent_session {agent claude, source herdr:claude}   claim made by the Claude integration hook
  after /quit: agent_not_found; claim persists
  later report-agent --source firstmate:cortex: ignored, agent get still agent_not_found

guarded launch (env -u CLAUDECODE):
  no agent_session claim
  after /quit: agent_not_found
  later report-agent --source firstmate:cortex: accepted, agent get reports working
```

Herdr 0.8.2 的 Agent 偵測 manifest 沒有 Cortex 項目，因此沒有回報 hook 的 Cortex 窗格，對 `agent get` 回覆 `agent_not_found`；即使程序仍存活，`fm_backend_agent_state` 也會將它分類為 `dead`。
對 `dead` 端點，`fm_task_inbox_ring` 回傳 3，並略過通知訊號。
在同一實驗窗格加入每任務外掛 hooks 後，記錄的呼叫為：

```text
pane report-agent <pane> --source firstmate:cortex --agent cortex --state idle
pane report-agent <pane> --source firstmate:cortex --agent cortex --state working
pane report-agent <pane> --source firstmate:cortex --agent cortex --state idle
pane release-agent <pane> --source firstmate:cortex --agent cortex
```

此時 `fm_backend_agent_state herdr` 讀為 `alive`，`fm_backend_composer_state herdr` 讀為 `empty`，busy 紀錄讀為 `idle cortex-hook`。
透過實驗窗格上的真實控制層，閒置時 `fm-control interrupt` 回覆 `interrupt-delivered ... verified=agent-alive cancel=unconfirmed`；`exit` 停止工作 Agent，之後讀為 `dead`；`relaunch --note` 重新啟動它，新程序會讀取啟動簡報與引導收件匣。
與 Devin 不同，Herdr 上的 `exit` 不會拒絕，因為不依賴游標的輸入區分類器能證明 Cortex 輸入區。

<a id="coverage-and-limits"></a>
## 覆蓋範圍與限制

可攜式回歸測試涵蓋程序祖先證據及拒絕無關名稱、僅限工作 Agent 的解析、擷取的閒置／忙碌／草稿輸入區結構、獨立投遞訊號、外掛生命週期與過期世代拒絕、Herdr 內外的回報條件，以及偵察任務啟動的自主性、外掛、型別化簡報、模型與 effort 處理，並省略 `xhigh`。
真實驗證防護檢查啟動、簡報執行、模型旗標、略過信任、工具祖先鏈、原生 `Stop`、提交署名、通知訊號確認、閒置及忙碌中斷、程序存活性、結束與原生恢復。

下列項目尚未驗證，因此不支援：

- `--help` 中出現的 `cortex --continue` 與 `cortex resume --last`；只測過從結束摘要或 hook payload 讀取 session id 的方式。
- 透過 `fm-send` 端到端呼叫技能；斜線指令彈出選單會列出使用者技能，但未將任何技能當作指令送出。
- 回合因模型錯誤結束時，`Stop` 是否觸發；停在畫面錯誤上的這類工作 Agent 屬於過期工作 Agent 情境。
- 全新主目錄缺少主題或連線時開啟的 Cortex 初始設定對話框；啟動流程沒有針對它們的最後防線特徵判定。
- 較新 Herdr 版本是否原生識別 Cortex；屆時自行註冊 hooks 會從第二個來源回報。
- Zellij、Orca、cmux、Paseo、rex 與 T3 Code，均未實際測試；`fm-spawn.sh` 拒絕在其上啟動 cortex，因為只有 tmux 與 Herdr 提供 Cortex 窗格身分，足以證明帶標題的輸入區為空。
- 任何主要 Agent 或二副監督，因此轉接器拒絕兩者。

啟動不傳入 `--sql-read-only`，因此 SQL 寫入政策仍由操作者自己的 Cortex 設定決定，並依原設定使用帳號連線。
Cortex 會匯入 Claude 相容 hook 檔案，且沒有開關能停用匯入，因此含 Claude hooks 的專案，會在 Cortex 工作 Agent 中執行那些 hooks。
