模式：omp（Oh My Pi）擴充功能背景喚醒。

當此工作階段負責監督，且未啟用離開模式時：
1. 先執行 `bin/fm-wake-drain.sh`，清空待處理的喚醒佇列。
   處理完所有輸出的喚醒事件，並核對尚未處理的決策與未讀狀態列後，原樣執行以 `WAKE_ACK_REQUIRED` 印出的 `--ack-through` 指令；在此之前，工作會持久保存，即使中斷也能以冪等方式重新處理。
2. 確認 omp 主要工作階段已從 `.omp/extensions/` 自動載入兩個專案擴充功能；omp 沒有專案信任關卡，因此以此主目錄為工作目錄直接啟動 `omp`，就會在沒有對話框的情況下載入它們。
   若 `bin/fm-session-start.sh` 回報 omp 擴充功能未載入，__FM_OMP_RESTART_LONG__。
3. 僅限最初的程序週期：進行那一次必要的 `fm_watch_arm_omp` 呼叫；若啟動程序已擁有船隊鎖，這會是根據所有權判定、無須動作的操作。
   `/fm-watch-arm-omp` 僅可作為由人輸入的備援方式。
   絕不要透過 omp 的 bash 工具執行 `bin/fm-watch-arm.sh`，因為前景啟動監看可能卡住 Agent，並繞過擴充功能負責的清理。
4. 若擴充功能表示沒有存活工作階段持有鎖，請執行 `bin/fm-session-start.sh` 重新取得工作階段鎖，再呼叫 `fm_watch_arm_omp`。
5. 擴充功能會啟動 `bin/fm-watch-arm.sh --restart`，讓子程序持續附掛在存活的 omp 程序，並負責之後每一個接續程序的啟動。
6. 同一程序內的一般工作階段替換（`/new`、`/resume`、`/fork`），只會退役前一個世代；當替換後的工作階段擁有船隊鎖時，它的 `session_start` 會啟動新世代，無須模型回合，也無須再次呼叫 `fm_watch_arm_omp`。
   世代負責者契約與處理中、需要行動的關閉事件交接，定義於 `.omp/extensions/fm-primary-omp-watch.ts`；由於 omp 不回報關閉原因，每次關閉時若仍有待處理的行動型關閉事件，就會持久保存交接資料，並由任意程序中下一個取得所有權的 `session_start` 重播。
7. 子程序因需要處理的事件而關閉後，擴充功能會重新檢查工作階段鎖的所有權，並驗證一個接續程序，再送出後續喚醒；其有界限的備援處理定義於 `docs/watcher-continuity.md`。
8. 一般工作、回合完成，以及一般的 signal、stale、check、heartbeat 或其他喚醒處理：不要再次呼叫 `fm_watch_arm_omp`，因為連續性由擴充功能負責，而非仰賴模型記憶。
9. 子程序非預期關閉時，會進入有上限的指數退避重試；重試耗盡或工作階段鎖遺失時，會明確回報監看器失敗，不會無聲消失。
10. 僅限週期遺失、失敗或不健康時：若後續通知明確回報上述任一需要修復的情況，請清空喚醒佇列、檢查失敗文字、呼叫 `fm_watch_arm_omp`；若擴充功能未載入，則 __FM_OMP_RESTART_SHORT__。
   擴充功能已擁有監看啟動子程序或已排定重試時，重複呼叫會成為根據所有權判定的 `watcher: unchanged` 無動作操作，不是獨立的健康狀態宣告。
11. 監看器監督絕不可使用 shell `&`。
   上述啟動機制由擴充功能負責，並非模型的工具呼叫；但若手動復原探測把啟動程序放到背景、接上管線或與其他指令綁在一起，工具執行前的防護機制（`bin/fm-arm-pretool-check.sh`，接在 `__FM_OMP_TURNEND_EXT__` 的回合結束防護擴充功能中）會自動拒絕。

omp 的回合結束防護是結構性的強制機制，而非建議：`__FM_OMP_TURNEND_EXT__` 回應 omp 會阻擋的 `session_stop` hook；當 `bin/fm-turnend-guard.sh` 回傳 2 時，它會強制進行一次附帶防護文字的延續，並透過 omp 在該延續自身 stop 時設定的 `stop_hook_active` 旗標，限制每回合最多一次。
被中斷的回合不會觸發 `session_stop`，因此由監督器發起的中斷不受此防護約束；這項後置條件由 `bin/fm-control.sh` 負責。

Pi 監督分支（`docs/pi-supervision-branch.md`）是 Pi 程序內的對話，不會在 omp 上執行：若沒有監督主機，每次需要處理的喚醒都會送到此對話，租約、結果儲存區與 `fm_branch_processed` 契約不適用於此；若主目錄有 `config/supervision-host`，且沒有 `config/supervision-host-off`，則會執行主機的離開模式工作階段（[`supervision-host.md`](../supervision-host.md)）。

回合結束防護擴充功能位於 `__FM_OMP_TURNEND_EXT__`。
監看器擴充功能位於 `__FM_OMP_EXT__`。
兩者都是受版本控制、專案本機的 `.omp/extensions/*.ts` 檔案，omp 會從此主目錄自動探索它們，不顯示信任對話框；若執行中的 omp 工作階段尚未載入兩個必要擴充功能，`bin/fm-session-start.sh` 會回報。
