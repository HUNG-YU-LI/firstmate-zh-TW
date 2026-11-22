<a id="pi-and-pi-signed"></a>
# Pi 與 Pi-signed

這份共用契約有實際依據：Pi 與簽章包裝器提供相同、已驗證的 CLI 與 TUI 行為。
除個別事實註明其他版本外，於 2026-07-27 使用 Pi 與 Pi-signed 0.82.0 驗證。

<a id="operating-facts"></a>
## 操作事實

| 項目 | 值 |
|---|---|
| 忙碌狀態 | Firstmate 管理的擴充功能以 `agent_start` 標記 busy，以 `ctx.isIdle()` 確認的 `agent_settled` 標記 idle；涵蓋重試、上下文壓縮、工具迴圈與排隊接續。 |
| 結束指令 | `/quit`。 |
| 恢復 | `--session <path-or-id>` 恢復精確指定的工作階段，檔案不存在時則在該路徑建立。`../../../bin/fm-spawn.sh` 重新啟動時傳入，讓 Herdr 窗格已綁定的狀態權威持續適用（`../../../bin/fm-control-lib.sh` 的 `fm_control_relaunch_resume_flag`；`../../../docs/herdr-backend.md` 的「Agent status authority and relaunch」）。仍沒有 `resume` 控制動詞。 |
| 中斷 | 一次 Escape。 |
| 技能呼叫 | 除一般指令行為外，沒有另行驗證的形式；精確指令不確定時，使用自然語言。 |
| 模型旗標 | `--model <model>`；home 固定工作 Agent 帳號時，模型必須是 `<provider>/<id>`，Firstmate 也會傳入 `--provider <provider>`（`../../../docs/configuration.md` 的「Worker account pin」）。 |
| 推理強度旗標 | `--thinking <low\|medium\|high\|xhigh\|max>`；兩種身分提供相同層級，並完成相同、以模型為條件的 max-thinking smoke 驗證。 |
| 模型探索 | 以 `<executable> --list-models [search]` 執行選定執行檔；內建、擴充功能登錄與自訂供應商／模型項目如何進入清單，以 Pi 已安裝的 `docs/models.md` 為準。 |

原生 Codex 工作階段可透過 `../../../bin/fm-spawn.sh` 說明的原生擴充旗標要求 `ultra`；它與 Pi 思考層級分開。
Pi 沒有權限系統，因此工作 Agent 一律自主執行。
全螢幕模式會重寫回捲紀錄，可能埋沒指導訊息，因此已安裝 CLI 支援覆寫時，Firstmate 會避免全螢幕。
執行檔固定與版本安全啟動機制，以 `../../../bin/fm-spawn.sh --help` 為準。

Pi-signed 是在 0.82.0 驗證的簽章包裝器身分。
Firstmate 原樣記錄 `pi-signed`，不正規化；包裝器不可用時拒絕，不退回 `pi`。
觀察到的簽章程序樹，父程序是精確 `pi-signed` 包裝器，Pi 應用程式是子程序；而 tmux 對兩種選定執行檔，都回報精確的前景指令名稱 `pi-launcher`。
已安裝的一般 `pi` 指令，也會 exec 該簽章啟動器。
啟動標記與祖先鏈如何選擇身分，以路由器的偵測區段為準。

指示保持為單一位置參數。
多個位置參數會變成分開排隊的訊息；啟動範本已保留單參數形式。

任何尚未信任、且含 `.pi/extensions/` 等需要信任資源的目錄，首次執行 Pi 都可能出現專案信任對話框，包括乾淨 worktree 與剛植入初始內容的二副 home。
按 Enter 接受，並確認指示開始處理。
決策按路徑保存於 `~/.pi/agent/trust.json`；固定工作 Agent 帳號時，則保存在固定根目錄的 `trust.json`，因此同一根目錄下相同池化槽位的後續啟動會略過。
無人值守、已植入初始內容的二副啟動，其依能力開啟的專案信任核准機制，以 `../../../bin/fm-spawn.sh --help` 為準；回歸證據以[執行階段驗證](../../../../../docs/verification/runtime-backends.md#pi-seeded-secondmate-project-trust) 為準。

<a id="worker-turn-end-extension"></a>
## 工作 Agent 回合結束擴充功能

`../../../bin/fm-spawn.sh` 將工作 Agent 回合結束擴充功能放在 worktree 外的 `state/`，因為專案本機擴充檔會加重信任關卡，並污染專案。
擴充功能監聽 Pi 的 `turn_end` 事件，不是 `agent_end`，因此每次完成回合後就通知監督，不必等整次執行退出。
原生執行環境進度使用由 `../../../bin/fm-busy-event.sh` 負責、另外綁定世代的標記；絕不捏造 Pi 回合完成。
可選的前景模型串流觀察，另有版本關卡；啟用方式以 `../../../docs/configuration.md`（Environment variables）為準，真實 Pi 證據以 `../../../docs/verification/runtime-backends.md`（Pi worker model-stream observation）為準。
Pi 會替子程序設定 `PI_CODING_AGENT=true`，作為執行環境偵測標記。

<a id="primary-integration"></a>
## 主要工作階段整合

主要回合結束行為，於 2026-07-09 使用 Pi 0.80.5 驗證。
`.pi/extensions/fm-primary-turnend-guard.ts` 監聽邏輯執行的 `agent_settled`，不是每次工具迴圈的 `turn_end`；`../../../bin/fm-turnend-guard.sh` 回傳 2 時，以 `pi.sendUserMessage(..., { deliverAs: "followUp" })` 強制加入一次受防護接續。
沒有 `deliverAs: "followUp"`，Pi 會在 Agent 仍處理時拒絕傳送。
原生 Windows 上，擴充功能透過 `bash` 執行工作階段啟動、兩個 PreToolUse、回合結束與操作輸入 Bash 輔助工具；macOS 與 Linux 則直接執行。

主要監看協定也需要 `.pi/extensions/fm-primary-pi-watch.ts`。
專案獲得信任後，Pi 引擎會自動探索這兩個納入版本控制的專案本機擴充功能。
模型透過 `fm_watch_arm_pi` 工具啟用監看，絕不以前景 Shell 啟用。
原生執行環境介接器可透過 `.pi/extensions/lib/fm-native-contract.ts` 的公開 Pi 事件匯流排契約，探索相同受防護 FirstMate 工具與操作訊息允許清單；沒有 Pi 內建工具跨越此契約。
工具結果與乾淨結束替代路徑，以 `../../../docs/supervision-protocols/pi.md` 為準。
活躍 Pi 家族工作階段未同時載入兩個擴充功能時，`../../../bin/fm-session-start.sh` 會回報，並指出完成專案信任後使用選定執行檔的修正方式，另提供免信任的 `-e` 替代路徑。

二副使用 Pi 或 Pi-signed 啟動時，`../../../bin/fm-spawn.sh --secondmate` 會以選定執行檔搭配 `-e .pi/extensions/fm-primary-turnend-guard.ts`、`-e .pi/extensions/fm-primary-pi-watch.ts`，以及父 home 的 `state/<id>.pi-ext.ts` busy 擴充功能之 `-e` 啟動。
兩個主要檔案已存在於二副 home 的 Git worktree。
Busy 擴充功能位於 home 之外，因此 Pi 專案信任關卡看不到它，其回合結束處理器也不碰父層 turn-ended 標記。
相當於 PreToolUse 的監看啟用防護，從 `tool_call` 事件回傳 `{block: true}`。
