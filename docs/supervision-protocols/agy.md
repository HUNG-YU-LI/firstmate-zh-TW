<!-- fm:supervision-protocol:v3 harness=agy -->
模式：由 Antigravity Stop hook 負責的監督。

當此工作階段負責監督，且未啟用離開模式時：
1. 先執行 `bin/fm-wake-drain.sh`，清空待處理的喚醒佇列。
   處理完所有輸出的喚醒事件並核對尚未處理的決策後，原樣執行以 `WAKE_ACK_REQUIRED` 印出的 `--ack-through` 指令；在此之前，工作會持久保存，即使中斷也能以冪等方式重新處理。
2. 例行的監看器啟動與重新啟動，都由 Stop hook（`.agents/hooks.json` 執行 `bin/fm-turnend-guard-agy.sh`）負責，絕不由你負責。
   Antigravity 會同步執行並等待這個 hook，因此只要仍需監督，每次回合結束都會在一個以主目錄為範圍的監看器週期上，將回合邊界保持開啟並停駐；停駐期間無須模型下指令，也不消耗模型 token。
3. 當停駐程序輸出 `{"decision":"continue","reason":"..."}` 時，需要處理的關閉事件會以帶有 `watcher` 作業種類的後續回合喚醒你。
   收到該喚醒時，先執行 `bin/fm-wake-drain.sh`，再處理它。
   一般喚醒之後不要執行 `bin/fm-watch-arm.sh`；只要仍需監督，下一次回合結束就會自動再次停駐。
   不要只看到附加狀態列就自行認定發生喚醒；請清空佇列，只依據真正的喚醒紀錄、清空操作產生的 `OPEN DECISIONS` 項目，或真正的監看器原因列採取行動。
4. Stop hook 停駐程序每隔 `FM_AGY_PARK_POLL` 秒（預設 2）檢查受追蹤的子程序，並在 28800 秒的 hook 逾時限制下等待它，因此閒置週期不會消耗回合。
   只有在設定了選用的 `FM_AGY_PARK_TIMEOUT` 上限，且超過該上限仍無喚醒時，才會停止子程序並回傳帶有 `[supervision-park]` 的 `{"decision":"continue","reason":"..."}`；此時簡短結束回合，讓下一個回合邊界再次停駐。
   若沒有任何內容需要監督，或停駐程序關閉時旁邊已有經驗證且正常的監看器，hook 會回傳 `{}`，允許正常結束回合。
   若仍需監督，但 `.executionNum` 已達迴圈上限（`FM_AGY_TURNEND_LOOP_CEILING`，預設 180），hook 會回傳一次 `FIRSTMATE SUPERVISION FOLLOW-UP CEILING REACHED` 通知；請自行清空並處理喚醒佇列，超過上限後 hook 會回傳 `{}`。
   只有 `.terminationReason == "NO_TOOL_CALL"` 的正常回合結束才會觸發停駐；其他終止原因一律以 `{}` 放行。
5. hook 停駐期間，船長（captain）仍保有控制權。
   停駐期間送出的訊息，會透過 `state/.agy-park-owner` 取代較早的停駐程序，使其以 `{}` 乾淨退場，不送出重複喚醒。
6. 出現 `turn-end-guard` 後續回合，表示停駐程序無法建立運作中的週期。
   請檢查監看器啟動路徑，不要把通知變成反覆手動啟動的迴圈；`bin/fm-turnend-guard.sh --agy` 會先進行有界限、最多阻擋 3 次的修復提醒，之後以 `{}` fail-open 放行。
7. 停駐輸出中的 `watcher: started ...` 與 `watcher: attached ...`，可視為存在一個運作中週期的證據。
   附加時，啟動程序會追隨身分識別相符且經過驗證的接續程序，而非在第一個週期結束時就退出。
8. 持久喚醒佇列會保存後續回合至下一次停駐之間需要處理的事件。
   [`watcher-continuity.md`](../watcher-continuity.md) 負責定義確切的工作階段鎖復原邊界。
9. 等待由 hook 負責的停駐時保持靜默：監看器停駐期間，不要傳送閒置進度訊息。

監看器本身仍是 `bin/fm-watch.sh`；`bin/fm-watch-arm.sh` 仍是經過驗證、由 Stop hook 當作自己受追蹤子程序執行的監看啟動包裝程式。若主目錄執行監督主機，則會改在 `bin/fm-supervision-host.sh park` 停駐。
重新啟動時，若已有正常運作的週期，會附加至該週期並追隨經過驗證的接續鏈。
啟動層的接續程序與正常關閉失敗契約，請參閱 [`watcher-continuity.md`](../watcher-continuity.md)。
有界限的修復提醒與接替契約，請參閱 [`turnend-guard.md`](../turnend-guard.md)。
