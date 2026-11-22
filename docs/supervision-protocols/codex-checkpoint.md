模式：Codex 前景檢查點。

當此工作階段負責監督，且未啟用離開模式時：
1. 先執行 `bin/fm-wake-drain.sh`，清空待處理的喚醒佇列。
   處理完所有輸出的喚醒事件，並核對尚未處理的決策與未讀狀態列後，原樣執行以 `WAKE_ACK_REQUIRED` 印出的 `--ack-through` 指令；在此之前，工作會持久保存，即使中斷也能以冪等方式重新處理。
2. 啟用 Relay 時，先載入 `__FM_X_MODE_ENV__`。
3. 第一個週期：執行 `bin/fm-watch-checkpoint.sh --seconds "${FM_CODEX_WATCH_CHECKPOINT:-180}"`，進行一次前景監看器檢查點。
4. 一般喚醒：若指令印出 `signal:`、`stale:`、`check:` 或 `heartbeat`，先清空喚醒佇列、處理該喚醒，再啟動下一個檢查點。
5. 若指令印出 `checkpoint:`，或以 124 結束且沒有喚醒，仍須清空喚醒佇列、處理此時 Codex 可見的所有待處理使用者訊息，再啟動下一個檢查點。
6. Firstmate 監看器監督絕不可使用 shell `&` 或 Codex 背景工作。
7. 不要把 `bin/fm-watch-arm.sh` 當作 Codex 的正常監督指令。
   若仍透過 shell 執行，任何將它放到背景、接上管線或與其他指令綁在一起的反模式，都會由 `.codex/hooks.json` 註冊的 PreToolUse 防護機制（`bin/fm-arm-pretool-check.sh`）自動拒絕。
8. 僅在失敗或週期遺失時：清空喚醒佇列、檢查失敗原因，再啟動新的前景檢查點。

前景工具呼叫執行期間，Codex 無法進行推理。
有時間上限的檢查點會定期交還控制權，因此可以處理使用者訊息與佇列中的喚醒事件，無須依賴背景工作的喚醒語意。
