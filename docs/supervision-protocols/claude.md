模式：由 Claude Stop hook 負責的監督。

當此工作階段負責監督，且未啟用離開模式時：
1. 先執行 `bin/fm-wake-drain.sh`，清空待處理的喚醒佇列。
   處理完所有輸出的喚醒事件，並核對尚未處理的決策與未讀狀態列後，原樣執行以 `WAKE_ACK_REQUIRED` 印出的 `--ack-through` 指令；在此之前，工作會持久保存，即使中斷也能以冪等方式重新處理。
2. 例行的監看器啟動與重新啟動，都由 Stop `asyncRewake` hook（`bin/fm-claude-stop-autoarm.sh`）負責，絕不由你負責。
   只要仍需監督，每次回合結束都會啟動或附加一個以主目錄為範圍的監看器週期，無須模型下指令，也不消耗模型 token。
   需要處理的關閉事件，會透過 hook 的 exit-2 重新喚醒機制，以 `Stop hook feedback` 訊息喚醒你。
3. 收到 `Stop hook feedback` 喚醒（`signal:`、`stale:`、`check:` 或 `heartbeat`）時，先執行 `bin/fm-wake-drain.sh`，再處理喚醒。
   一般喚醒之後不要執行 `bin/fm-watch-arm.sh`；只要仍需監督，下一次回合結束就會自動重新啟動。
   不要只看到附加狀態列就自行認定發生喚醒；請清空佇列，只依據真正的喚醒紀錄、清空操作產生的 `OPEN DECISIONS` 與 `UNREAD STATUS` 項目，或真正的監看器原因列採取行動。
4. 收到那一次 `Stop hook feedback` 自動機制失敗通知（`firstmate watcher auto-arm FAILED ...`）時，清空佇列並檢查自動機制的失敗原因；不要將通知轉變成反覆手動啟動的迴圈。
5. 若 Stop hook 未宣告負責此主目錄，或回報已耗盡重試，請在未確認狀態就結束之前，檢查其註冊與監看器啟動路徑。
   讓 Stop 負責的自動機制持續作為 Claude 唯一的監看啟動負責者。
6. 自動啟動輸出中的 `watcher: started ...` 與 `watcher: attached ...`，可視為存在一個運作中週期的證據。
   附加時，啟動程序會追隨身分識別相符且經過驗證的接續程序，而非在第一個週期結束時就退出。
7. 持久喚醒佇列會保存重新喚醒至下一次由 Stop 啟動監看之間需要處理的事件；有界限的回合結束防護，則會在復原未啟動時防止未確認狀態就 Stop。
   不會有 PreToolUse hook 根據監看器狀態拒絕船隊指令。
   [`watcher-continuity.md`](../watcher-continuity.md) 負責定義確切的工作階段鎖復原邊界。
8. 回合結束防護（`bin/fm-turnend-guard.sh --claude`）仍是最後防線。
   它要求在 Stop 邊界滿足嚴格 PID 比對的存活監看器及最新信標判定，唯一例外是 [`turnend-guard.md`](../turnend-guard.md#guard-predicates) 定義、Claude 專用的其他存活負責者安全結束情境；該文件也負責定義另一組可辨識模型狀態的回合中途拉取防護規則。
   其他情況下，監看器正常，或有尚未結束的自動啟動世代宣告負責復原時，會允許停止；新的失敗 epoch 則推進該文件描述、在有人值守下有界限且只執行一次的 fail-open 流程。
9. 等待由 hook 負責的週期時保持靜默：監看器停駐期間，不要傳送閒置進度訊息。

監看器本身仍是 `bin/fm-watch.sh`；在選擇停用[監督主機](../supervision-host.md)（`config/supervision-host-off`）的主目錄中，`bin/fm-watch-arm.sh` 仍是經過驗證、由 Stop hook 在前景執行的監看啟動包裝程式。
重新啟動時，若已有正常運作的週期，會附加至該週期並追隨經過驗證的接續鏈。
啟動層的接續程序、正常關閉失敗契約，以及 Claude 的權責模型，請參閱 [`watcher-continuity.md`](../watcher-continuity.md)。
