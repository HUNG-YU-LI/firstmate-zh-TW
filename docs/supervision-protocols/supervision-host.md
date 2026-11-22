監督主機：此主目錄已啟用（`config/supervision-host-off` 可停用；設計由 [`supervision-host.md`](../supervision-host.md) 定義）。
{claude} Stop hook 會以監督主機取代一般監看啟動程序；上述規則仍然適用，並補充以下內容：
{cursor} `stop` hook 的停駐會以監督主機取代一般監看啟動程序；上述規則仍然適用，並補充以下內容：
{agy} Stop hook 的停駐會以監督主機取代一般監看啟動程序；上述規則仍然適用，並補充以下內容：
{opencode} OpenCode TUI 外掛會以監督主機取代一般監看啟動程序；上述規則仍然適用，並補充以下內容：
{omp} omp 監看擴充功能會以監督主機取代一般監看啟動程序；上述規則仍然適用，並補充以下內容：
{grok} 上述受追蹤的背景啟動程序，會執行監督主機（`bin/fm-supervision-host.sh park`）取代一般啟動程序；上述規則仍然適用，並補充以下內容：
{codex-park} 非同步 Stop 停駐會以監督主機取代監看器，並透過原生執行緒佇列投遞交還的喚醒。
{codex-checkpoint} 每個前景檢查點都會以監督主機取代監看器；上述規則仍然適用，並補充以下內容：
{claude,cursor} 1. 有人值守（沒有離開紀錄，包含只有安靜模式紀錄的情況；見[運作狀態](../supervision-host.md#postures)）：無介面的監督工作階段會接收監督分支可接手的喚醒，且絕不因例行結果喚醒你，因此到達你的喚醒較少；檢查喚醒、決策喚醒及它無法接手的其他內容，仍完全依上述方式到達你。
{opencode,omp,grok,codex,agy} 1. 有人值守（沒有離開紀錄；見[運作狀態](../supervision-host.md#postures)）：每次喚醒都完全依上述方式到達你，因為此執行環境尚無經驗證的對話鏡像可餵入監督工作階段。
{claude,cursor}    `supervision-host: branch-outcome: ...` 表示它已處理一次喚醒，並替你記錄了應呈報船長的結果：請執行 `bin/fm-wake-drain.sh`，以 Firstmate 身分，從任務目前狀態處理其 `BRANCH OUTCOMES` 區段的每個項目，因為每項都會註明多久以前記錄（告知船長、將已就緒的工作落地或合併、回答或升級決策，或處理阻礙；回覆只涵蓋仍未結案的項目，已解決的項目，例如後來已合併的 PR，視同從未列出），之後執行它印出的 `mark-processed` 確認指令；在確認前，每次清空佇列都會再次呈現它們。
{claude,cursor}    `supervision-host: the supervision session could not take this wake ...` 表示該喚醒由你負責：依上述方式處理。
{claude,cursor}    失敗回合可能附帶 `supervision-host:` 健康狀態註記，指出引擎重複錯誤：有需要時告知船長，並照常處理交還的喚醒；冷卻期間，之後有人值守狀態下的關閉事件會原樣到達你。
{claude,cursor}    例行結果絕不會喚醒你；下一次清空佇列，只會在 `BRANCH OUTCOMES, ROUTINE` 下列出可見的例行結果供你掌握，不需確認。靜默列不會出現在該處，但仍可透過 `bin/fm-branch-outcome.sh list` 取得。
2. 離開（存在離開紀錄，且沒有常駐程序執行）：主機將每次喚醒交給無介面的離開模式工作階段，由它依紀錄執行監督分支契約，而你維持停駐。
{claude}    只有主機交還的喚醒才會到達你，以 `Stop hook feedback` 附帶關閉事件及一行 `supervision-host: <why>` 的形式呈現。
{cursor,opencode,omp,agy}    只有主機交還的喚醒才會到達你，以 `watcher` 後續回合附帶關閉事件及一行 `supervision-host: <why>` 的形式呈現。
{grok}    只有主機交還的喚醒才會到達你，以啟動程序的背景工作完成通知呈現；輸出會附帶關閉事件及一行 `supervision-host: <why>`。
{codex-checkpoint}    只有主機交還的喚醒才會到達你，以檢查點輸出附帶關閉事件及一行 `supervision-host: <why>` 的形式呈現。
{codex-checkpoint}    只要離開紀錄存在，每個檢查點就會使用較長的離開模式上限（`FM_CODEX_WATCH_CHECKPOINT_AWAY`，預設 3600 秒，仍受主機停駐上限約束；見 [`supervision-host.md`](../supervision-host.md#the-park-boundary)），因此除非船長中斷檢查點，否則船長訊息會等到檢查點返回才處理。
   該喚醒屬於自動監督，不代表船長回來了：請在離開運作狀態下清空佇列並處理，絕不要因此執行返回流程。
   返回之後，若某個回合出現 `supervision-host:` 列，指出船長在該回合進行中返回，就代表該回合有返回簡報未涵蓋的可見結果，不論該喚醒已由主機處理或被交還：將後續每一行 `supervision-host: outcome ...` 轉告船長（這些列也會保留在 `bin/fm-branch-outcome.sh list` 中），再清空並處理任何待處理喚醒，最後才確認。
   每個這類可見結果也會成為佇列中的 `check: supervision-host outcome <n> ... was recorded after the captain returned` 喚醒，直到確認前都會由清空操作呈現：不論哪個先到，每項結果只轉告一次；若有對應的 `BRANCH OUTCOMES` 項目，也要一起確認。靜默結果會留在儲存區，但不產生交接列或 check 喚醒。
{claude,cursor,agy} 3. `supervision-host: cycle boundary ...` 表示主機已因到達上限而結束停駐：執行 `bin/fm-wake-drain.sh`、處理它呈現的所有內容、執行印出的確認指令（空佇列會印出 `--ack-through 0`），再結束回合；下一次停駐會在該回合結束時開始。
{opencode,omp} 3. `supervision-host: cycle boundary ...` 表示主機已因到達上限而結束停駐，且下一次停駐已開始：執行 `bin/fm-wake-drain.sh`、處理它呈現的所有內容，再執行印出的確認指令（空佇列會印出 `--ack-through 0`）。
{grok} 3. `supervision-host: cycle boundary ...` 表示主機已因到達上限而結束停駐：執行 `bin/fm-wake-drain.sh`、處理它呈現的所有內容、執行印出的確認指令（空佇列會印出 `--ack-through 0`），再以相同背景主機呼叫重新啟動。
{codex-park} 3. 安靜的主機停駐邊界會保留同一個監看器與復原世代，不呼叫模型。
{codex-checkpoint} 3. 主機的停駐邊界會以檢查點的一般 `checkpoint: no actionable wake within <n>s` 列返回；依上述步驟 5 處理。
4. 受防護指令若以 6 結束，並指出分支 actor 的租約，表示監督工作階段此刻正在處理該任務：不要動該租約，等它在回合結束時釋放後再重試。
5. 離開模式工作階段記錄、應呈報船長的結果，會留在結果儲存區，直到返回時清空佇列才呈現；返回簡報（`bin/fm-afk-return.sh`）會計算數量，並指引你到 `BRANCH OUTCOMES` 區段處理與確認（[船長結果](../supervision-host.md#captain-outcomes)）。
{claude,grok} 6. `/afk` 在此只寫入紀錄（`bin/fm-afk-launch.sh start-native` 會拒絕在此主目錄啟動離開模式常駐程序）；`/quiet` 則依照 [quiet 技能](../../.agents/skills/quiet/SKILL.md)。
{cursor,opencode,omp,codex,agy} 6. `/afk` 在此只寫入紀錄（`bin/fm-afk-launch.sh start` 會拒絕在此主目錄啟動離開模式常駐程序）；`/quiet` 則依照 [quiet 技能](../../.agents/skills/quiet/SKILL.md)。
{grok} 7. 工具執行前的防護機制不會分類主機指令，因此必須維持為上述單一背景呼叫：絕不可使用 shell `&`、管線，或與其他指令綁在一起。
