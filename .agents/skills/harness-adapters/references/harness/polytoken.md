# Polytoken CLI

Polytoken 是使用 TUI 窗格與逐工作階段 REST 控制的 crewmate/scout 配接器。
能力探索使用已安裝執行檔；派工不會依 Polytoken 發行版本號設關卡或警告。
路由器負責限定僅供 crewmate/scout 使用；不支援主 Agent 與二副整合。
[驗證證據](../../../../../docs/verification/polytoken.md)及其即時防護檢查會更新下列供應商事實。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 忙碌狀態 | `pre_user_prompt` 與 `pre_model_turn` 開啟，`stop` 關閉，全部透過綁定世代的寫入器處理；信任由 `../../../bin/fm-busy-lib.sh` 負責。REST 取消的回合不會發出 `stop`（已實際驗證），因此控制平面在型別化穩定確認後，會自行寫入 `idle fm-interrupt` 關閉；但若取消前快照之後寫入的 `pre_user_prompt` 顯示排隊提示重新開啟回合，或寫入前的 `GET /sync` 顯示回合再次執行，就不關閉（`fm_polytoken_interrupt_task`）。`post_model_turn` 每次模型回應都會觸發，發生在回合中、工具階段之間（已實際驗證），因此刻意絕不把它接成關閉事件。 |
| 結束命令 | 透過共用斜線命令選單穩定等待執行 `/quit`：Enter 接受預選列，TUI 結束，窗格回到 shell，daemon 刪除自己的憑證後不久便自行結束（全部已實際驗證）。已附接工作階段**不使用** `POST /terminate`：它會讓 TUI 因 SSE 重新連線錯誤卡住約 20 秒，並記錄當機日誌（已實際驗證）。 |
| 中斷 | 完全沒有 TUI 組合鍵（`print tui-command-actions` 在 Prompt 範圍沒有取消動作；單次 Escape 沒作用；Escape Escape 會開啟倒帶選擇器）。中斷會對工作階段 daemon 執行 `POST /turn/cancel`，得到 `{"status":"cancel_requested","generation":N}`，並以 `GET /sync` 的 `turn: null` 作為型別化穩定確認。Firstmate 絕不送中斷按鍵。 |
| 技能呼叫 | 在提示文字中使用 `@skill:<name>`，以及模型的 `skill` 工具；沒有斜線命令技能形式。 |
| 恢復 | `polytoken continue --no-attach <session-id>` 會在新連接埠，以輪替後的憑證從歷史重新啟動（偵察 Agent 已驗證）。Firstmate 仍以從任務簡報可確定重現的重新啟動作為復原途徑；控制平面會重新啟動全新工作階段。 |
| 模型旗標 | `--model <ref>`，必須是 `polytoken print models` 列出的可選參照，包含 `zai/glm-5.3-flash(low)` 這類推理強度變體。標準啟動要求明確的模型與推理強度軸；未列出的模型或無法存取的目錄會使 spawn 拒絕，而切換為 polytoken 的重新啟動會在停止現有工作 Agent 前，拒絕同樣無效的設定檔、下述含空白的處理器路徑，以及非 firstmate 產生的 hook 層。 |
| 推理強度旗標 | 沒有獨立旗標：`bin/fm-polytoken-lib.sh` 的 `fm_polytoken_model_ref` 負責將最接近層級對應至 `<model>(<effort>)`；任務中繼資料會保留要求的軸。 |
| 模型探索 | `polytoken print models`（每列一個可選參照）；完整的人類可讀報告使用 `polytoken models`。 |
| 識別標記 | 無；工具子程序祖先鏈中 daemon 明確錨定的 `polytoken` comm 用來識別配接器（工具子程序的父程序**就是** daemon，已實際驗證），結構上的 polytoken 祖先優先於繼承的其他標記。 |
| 信任對話框 | 啟動時沒有：權限行為來自產生的設定目錄中的 `default_permission_matcher: bypass`，首次啟動的授權條款對話框需要操作人員決定——絕不傳入 `--accept-license-terms`，若窗格一直沒有開始處理任務簡報，就緒關卡會明確失敗。 |
| 匯入設定 | worktree 的 `.polytoken/` 專案層（hooks 與 facets），加上產生的個別任務設定目錄，後者完全取代全域設定（已實際驗證：只有 `version: 4` 的專案 config.yaml 會因零個供應商而啟動失敗，且沒有 `--global-config-dir` 傳到 daemon）。 |
| 提交署名 | 共用 `state/<id>.git-hooks` 的 commit-msg 移除器，像涵蓋每個啟動的執行階段一樣涵蓋 polytoken。 |

<a id="worker-lifecycle-limits"></a>
## 工作 Agent 生命週期限制

Hook 處理器會從 stdin 收到事件 JSON 與 `POLYTOKEN_*` 變數，但**沒有** argv（已實際驗證：寫成 `<script> <arg>` 的處理器執行時 `$*` 為空），因此每個 firstmate hook 都是產生的自足指令碼，以絕對路徑參照；若狀態目錄中的忙碌指令碼路徑含空白，spawn 會拒絕 polytoken 接線，因為處理器的分割規則尚未驗證。
失敗的 hook 會中斷回合（已實際驗證：pre_model_turn hook 結束碼為 1 時，整個回合報錯），因此每個產生的 hook 都用 `|| true` 包住寫入器，並以 0 結束。
以名稱否定全域 hook 的專案 hook 會使 daemon 啟動失敗（偵察發現），因此個別任務 hooks 自足，且不否定任何項目。
回合執行中按 Enter 會將文字排到 Agent 下一次暫停——`Queued for the next agent pause`——與 OpenCode 完全相同（已實際驗證），排隊文字會顯示在輸入區分隔線上方。
沒有已驗證的輸入區清除按鍵：Ctrl+Shift+Y 會複製提示，卻把它留在輸入區（已實際驗證），因此只會對已證明為空的輸入區輸入文字，絕不在待送出文字上繼續輸入。
Ctrl+C 會結束工作階段；Escape Escape 會開啟倒帶選擇器；firstmate 絕不送出這兩者。
檔案工具受限於 worktree 加上工作階段授權路徑，且沒有啟動時授權（[證據](../../../../../docs/verification/polytoken.md#task-directory-file-access)）：在產生的 `bypass` 比對器下，直接 `file_read` 與 `file_write` 能存取 Firstmate 任務目錄，但 `tool_flow` 不能，因此工作 Agent facet 會指示模型對該目錄使用直接工具或 shell。
附接 TUI 的 daemon 在閒置時持續存活（已驗證數分鐘），工作階段結束後會自行結束，因此窗格設計不需要 daemon 生命週期監督。
daemon 不限制連續 `stop`-hook `continue` 結果的次數（已實際驗證：五次強制接續全部執行），因此未來若整合主 Agent 回合結束，必須自行加入有界限的鎖存。

`../../../../../bin/fm-spawn.sh` 負責自主性（產生的設定目錄）、worktree hook 與 facet 接線、將工作階段探索結果寫入任務中繼資料（搭配啟動的 `--config-dir` 與 `--sessions-dir` 執行 `polytoken sessions --format json`）、清除其他標記（固定 `env -u` 集合，包括一般性的 `AGENT`，加上 ORCA_*/FM_ 迴圈），以及就緒關卡。
全新 polytoken spawn 若失敗且未留下任務紀錄，也會關閉啟動前開啟的端點、移除接線、任務暫存根目錄與暫置啟動檔（只有重新取得個別 ID spawn 鎖定，且確認沒有任務紀錄後才執行；同 ID 的並行 spawn 若持有鎖定，就保留這些檔案），並且即使中繼資料已發布，也會釋放 slot 認領；其他執行環境維持共用中止行為。
`../../../../../bin/fm-polytoken-lib.sh` 負責每個產生的成品與 REST 控制核心。
`../../../../../bin/fm-control-lib.sh` 負責 REST 中斷傳輸（`fm_control_interrupt_transport`）、`/quit` 結束命令與接線表。

<a id="composer-and-steering"></a>
## 輸入區與引導

`../../../../../bin/fm-composer-lib.sh` 負責已驗證的分隔式輸入區形狀：一對全寬分隔線，下方是一列空白或待送出內容，再下面是頁尾。
共用分類器透過 Pi 形狀的身分合取條件到達它，polytoken 是此形狀第二個已驗證擁有者：閒置窗格證明輸入區為空，工作中的窗格則維持嚴格 `unknown`，讓忙碌佇列轉換保持如實。
忙碌頁尾字串是固定的 `Running for <elapsed>` 列，帶一個前導空白（已實際驗證）；`Completed in`、`Errored after` 與 `Canceled after` 是不同的回合結束列，絕不會符合它。
只有紀錄的執行環境是 polytoken，且輸入區身分也是 polytoken 時，才會讀取此字串；絕不透過沒有執行環境資訊的預設聯集，因此未記錄執行環境的窗格絕不會據此判為忙碌；送出核心只有在窗格輸入區身分（tmux）或原生身分（herdr）指明 polytoken 時，才把閒置到忙碌與忙碌佇列讀取的範圍限定為 polytoken。
送達防護絕不是忙碌狀態來源；記錄狀態來自 polytoken-hook 摺合。

<a id="primary-integration"></a>
## 主 Agent 整合

尚未驗證 Polytoken 的主 Agent Stop 防護、監看器協定、工具執行前保護或工作階段啟動契約。
不要使用此配接器啟動主 Agent 或二副。
工作 Agent facet 會從工具登錄表移除 `switch_facet`、`handoff_plan`、`subagent` 與 `message_subagent`，避免 facet 變更或委派將明確固定的工作 Agent 模型替換為路由模型。
