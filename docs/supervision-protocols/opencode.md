模式：OpenCode TUI 外掛的背景喚醒。

當此工作階段負責監督，且未啟用離開模式時：
1. 先執行 `bin/fm-wake-drain.sh`，清空待處理的喚醒佇列。
   處理完所有輸出的喚醒事件，並核對尚未處理的決策與未讀狀態列後，原樣執行以 `WAKE_ACK_REQUIRED` 印出的 `--ack-through` 指令；在此之前，工作會持久保存，即使中斷也能以冪等方式重新處理。
2. 第一個週期：讓 `.opencode/plugins/fm-primary-watch-arm.js` 在 OpenCode 工作階段閒置後啟動監督。
3. 在 OpenCode 2 上，外掛會訂閱終止事件 `session.execution.succeeded`、`session.execution.failed` 與 `session.execution.interrupted`（鎖定根工作 Agent 的工作階段；OpenCode 1 使用 `session.idle`），在回合結束處理器中啟動 `bin/fm-watch-arm.sh --restart` 而不等待它結束，並負責之後每一個接續程序的啟動。
4. 子程序因需要處理的事件而關閉後，外掛會重新檢查工作階段鎖的所有權，並確認只有一個接續單例，再呼叫 `ctx.session.prompt`；其有界限的備援處理定義於 `docs/watcher-continuity.md`。
5. 一般喚醒：不要要求模型重新啟動監看，因為連續性由外掛負責。
6. 子程序非預期關閉時，會進入有上限的指數退避重試；重試耗盡或工作階段鎖遺失時，會明確回報監看器失敗，不會無聲消失。
7. 僅在失敗或週期遺失時：若外掛回報監看器失敗，清空喚醒佇列、檢查失敗文字；只有在做短暫復原探測時，才手動使用 `bin/fm-watch-arm.sh`。
8. 監看器監督絕不可使用 shell `&`。
   上述啟動機制由外掛負責，並非模型的工具呼叫；但若手動復原探測把啟動程序放到背景、接上管線或與其他指令綁在一起，PreToolUse 防護機制（`.opencode/plugins/fm-primary-pretool-check.js`、`bin/fm-arm-pretool-check.sh`）會自動拒絕。
9. 不要在無介面的 `opencode run` 中依賴此外掛；Firstmate 的主要監督適用於持續執行的 OpenCode TUI 工作階段。
10. 在 OpenCode 2 上，此主目錄的 TUI 必須以 `opencode --standalone` 執行；預設啟動方式會將外掛放在共用背景服務中，脫離此主目錄的環境與工作階段鎖的程序祖先鏈。

OpenCode 持續執行的 TUI 外掛執行環境就是喚醒機制。
此外掛適用於主要工作副本及二副（secondmate）自己的主目錄；只有在子層船員（crewmate）與偵察員（scout）的 worktree 中保持靜默。
