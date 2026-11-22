模式：Grok 背景通知監督。

當此工作階段負責監督，且未啟用離開模式時：
1. 先執行 `bin/fm-wake-drain.sh`，清空待處理的喚醒佇列。
   處理完所有輸出的喚醒事件，並核對尚未處理的決策與未讀狀態列後，原樣執行以 `WAKE_ACK_REQUIRED` 印出的 `--ack-through` 指令；在此之前，工作會持久保存，即使中斷也能以冪等方式重新處理。
2. 啟用 Relay 時，先載入 `__FM_X_MODE_ENV__`。
3. 第一個週期：使用 Grok 受追蹤的背景工具，以獨立呼叫啟動監看：

   透過 `run_terminal_command`，設定 `background: true`，執行：
   `[ -f __FM_X_MODE_ENV_SH__ ] && . __FM_X_MODE_ENV_SH__; exec __FM_GROK_ARM__`

4. 只信任啟動程序輸出的單行狀態。
5. `watcher: started ...` 或 `watcher: attached ...` 表示存在運作中的週期。
   附加時，背景工作會追隨身分識別相符且經過驗證的接續程序，而非在第一個週期結束時就退出。
6. 僅在失敗或週期遺失時：`watcher: FAILED ...` 表示監督已停止；請修復並重新啟動。
7. 收到成功啟動或附加狀態後，結束回合。
   背景啟動程序會持續作為有效的等待機制，直到回傳需要處理的喚醒或失敗。
8. 等待期間保持靜默。
9. Firstmate 監督絕不可使用 shell `&`。
10. 絕不可將啟動監看的指令與其他指令綁在一起。
    只要此專案的 Grok hooks 已受信任，shell `&`、會截斷輸出的管線，或指令綁定，都會由 PreToolUse 防護機制（`bin/fm-arm-pretool-check.sh`）自動拒絕。

背景啟動程序完成時，Grok 會注入帶有 `synthetic_reason: task_completed` 的合成使用者訊息。
當你看到該啟動程序的背景工作完成系統提醒時：
1. 先執行 `bin/fm-wake-drain.sh`。
2. 如有需要，可使用 `get_command_or_subagent_output(<task_id>)` 取得啟動程序輸出，查看原因列。
3. 依照 `AGENTS.md` 中與執行環境無關的契約，處理 `signal`、`stale`、`check` 或 `heartbeat`。
4. 一般喚醒：若依 `bin/fm-supervision-lib.sh` 的定義，此主目錄仍需監督，便以相同的背景 `__FM_GROK_ARM__` 呼叫重新啟動下一個週期。
5. 不要只看到附加狀態列，就自行認定發生喚醒。
   清空佇列，僅根據真正的喚醒紀錄、清空操作產生的 `OPEN DECISIONS` 與 `UNREAD STATUS` 項目，或真正的監看器原因列採取行動。
   重新啟動時，若已有正常運作的週期，會附加至該週期並追隨經過驗證的接續鏈。
   啟動層的接續程序與正常關閉失敗契約，請參閱 [`watcher-continuity.md`](../watcher-continuity.md)。

主要專案的 Stop hook 會執行 `bin/fm-turnend-guard-grok.sh` 作為最後防線，並非正常的喚醒路徑。
[`turnend-guard.md`](../turnend-guard.md) 負責定義它針對執行中 payload 的能力選擇：採用原生同程序阻擋，或在尚無原生支援時採用有界限的恢復備援。
任何強制延續之後，都要使用上述背景協定啟動監看器。

互動式 TUI 工作階段是受支援的 Grok 主要操作介面。
無介面的 `grok -p` 可能會等待背景程序結束，但無法可靠地呈現完整的自動喚醒模型輸出；不要把主要 Firstmate 當作一次性的無介面程序執行。
