---
name: bootstrap-diagnostics
description: >-
  當工作階段啟動摘要的 bootstrap 或網路檢查區段，或獨立執行 bin/fm-bootstrap.sh、bin/fm-startup-network.sh，印出可採取行動的診斷行時載入：MISSING、MISSING_MANUAL、OUTDATED、PRESENTATION_UNAVAILABLE、BACKEND_INVALID、NEEDS_GH_AUTH、TANGLE、STARTUP_MEMORY_BUDGET、CREW_DISPATCH invalid、FLEET_SYNC、NETWORK_CHECKS、HOME_SUMMARY、BACKLOG_RECONCILE、SECONDMATE_SYNC、SECONDMATE_LIVENESS、SECONDMATE_HANDOFF、NUDGE_SECONDMATES、RECORD_HYGIENE 或 FMX；或回報中斷的 backlog 清理可能留下端點或本機副本時載入。
  安靜的 bootstrap 區段，或其他 BOOTSTRAP_INFO 事實，都不需載入技能。
user-invocable: false
metadata:
  internal: true
---

# bootstrap-diagnostics

在派出依賴該項目的工作前，依下列方式處理每個印出的行。
行格式本身由 `bin/fm-bootstrap.sh` 檔頭負責；本處置流程負責對可採取行動行的回應。
`session-start-recovery` 的工作階段啟動規則仍具約束力：先偵測、再同意、再安裝或升級；絕不更動船長未在本工作階段核准的工具，且必要工具備齊、GitHub 驗證正常前，不派工。
任何診斷需要船長注意時，依 `AGENTS.md` 第 9 節面向船長的轉述契約，以直白方式回報後果與要求動作；除非船長需要貼進命令或 issue，否則不提診斷標籤。

- `MISSING: <tool> (install: <command>)`——工具不存在；向船長列出缺少工具，每個附一行用途與印出的安裝命令，等待同意（一個核准可涵蓋整份清單），再執行 `bin/fm-bootstrap.sh install <approved tools...>`。
  已安裝但低於最低版本，或缺少必要能力的工具，改回報 `OUTDATED:`。
- `OUTDATED: <tool> (installed: <version|unparseable>; requires <floor|capability>; upgrade: <command>)`——工具存在，但未達此 home 所需，因此是升級要求，不是安裝。
  回報已安裝版本與印出的 `upgrade:` 命令，執行前依相同同意規則等待；若印出的命令是工具自己的安裝器，代表工具尚無自行更新器，因此升級沿用安裝命令。
  最低要求政策由 [`bin/fm-bootstrap.sh`](../../../bin/fm-bootstrap.sh) 負責，絕不把最低要求辯解下修成 home 剛好已安裝的版本。
  `no-mistakes` 最低版本為 1.46.0，因為本儲存庫 PR 關卡要求結構化 pipeline 證明，舊版不會寫入。
  `treehouse` 要求支援 `treehouse get --lease`。
  `tasks-axi` 要求也涵蓋獨立功能探測（定義由 `bin/fm-tasks-axi-lib.sh` 負責）；`config/backlog-backend=manual` 只抑制冗長的 `BOOTSTRAP_INFO: tasks-axi available` 事實，不抑制此回報。
  `quota-axi` 是 bootstrap 必要項，因為 firstmate 在解析每個 crew-dispatch 設定檔陣列前，會直接讀取其目前輸出；缺少時回報要求，不可繞過未檢查候選自行選擇。
- `PRESENTATION_UNAVAILABLE: lavish-axi ...`——說明視覺呈現不可用，並以純文字決策與報告繼續非視覺工作；不要為等待安裝同意而暫停無關派工。
  Lavish 達到 `bin/fm-bootstrap.sh` 規定最低要求前不可使用；視覺工作需要它時，請求核准印出的安裝或升級命令，再重跑 bootstrap 確認相容後才用。
  偵察簡報在建立骨架時也會檢查相同最低要求，未達時要求文字報告而非 Lavish 循環，因此只有重跑確認相容後才建立視覺偵察骨架。
- `MISSING_MANUAL: <tool> (instructions: <url>)`——告訴船長為何需要此工具，並提供印出的指示 URL，但不要將工具傳給 `bin/fm-bootstrap.sh install`；等船長完成手動安裝，再重跑工作階段啟動確認相依套件已存在。
- `BACKEND_INVALID: <name> (known: <names>)`——解析後的執行階段後端沒有已驗證相依套件或生命週期契約，因此在無效 `FM_BACKEND` 或 `config/backend` 修正為列出後端之一前，不可派工。
- `NEEDS_GH_AUTH`——請船長執行 `! gh auth login`（互動式，你不能代為執行）。
  此探測現在來自延後網路階段，因此網路無法存取也會以此顯示：`gh` 離線時無法驗證權杖，會回報相同失敗。要求船長重新驗證可能正常的憑證前，先確認連線可達。
- `NETWORK_CHECKS: <what did not complete>; rerun <command>`——延後網路階段本身未完成，因此列出的檢查只是未知，不是失敗。
  重跑印出的命令；它具有冪等性，會重新推導每項發現。
  `hit the ...s bound` 行代表其中一項檢查慢或無法連線，最常見是遠端二副主機，階段選擇停止而非卡住；`recorded fleet lock holder changed` 行表示鎖定第 1 行已不是請求掃描的 pid，因為另一工作階段取得鎖定，或此工作階段重新錨定鎖定，所以掃描被略過，改由目前持鎖工作階段負責。
- `TANGLE: <remediation>`——主 checkout 卡在功能分支，而非預設分支；`AGENTS.md` 第 8 節說明此防護存在原因與保護內容。
  工作安全地留在該分支 ref；用印出的 `git -C <root> checkout <default>` 將主 checkout 還原至預設分支，再於正確 worktree 重新驗證該分支。
  這是唯一核准由 firstmate 主動對主 checkout 執行的 git 寫入，是不會遺留任何工作的非破壞性分支切換。
- `STARTUP_MEMORY_BUDGET: invalid config/startup-memory-budget - <reason>`——可見啟動記憶預算不是安全的單行正十進位數檔案；不要推定預設值，也不要傳播。
  修正本機主 Agent 檔案，再重跑工作階段啟動，讓一般收斂路徑將已驗證值送到二副 home。
- `CREW_DISPATCH: invalid config/crew-dispatch.json - <reason>`——選用派工設定檔存在，但未通過低成本 bootstrap 驗證；停止以設定檔派工，回報可採取行動錯誤，要求修正格式錯誤結構描述、未驗證執行環境名稱，或無效執行環境／推理強度配對，不可繞過回退或選取壞設定檔。
- `FLEET_SYNC: <repo>: skipped: <reason>`——無害的一次性略過（離線、沒有 origin、僅本機）；bootstrap 已繼續，只有阻擋工作才調查。
  略過也可能回報有界限艦隊更新逾時（`FM_FLEET_SYNC_BOOTSTRAP_TIMEOUT`，或依艦隊規模調整且最低 20 秒的預設值）；逾時絕不阻擋啟動。
  `skipped: registry entry does not resolve to a delivery posture` 是唯一非一次性略過：在 `data/projects.md` 修正前，每次 bootstrap 都會保留 clone 不動，因此執行印出的 `bin/fm-project-mode.sh <repo>` 讀取拒絕並修正項目。
- `FLEET_SYNC: <repo>: recovered: <detail>`——clone 漂移到沒有獨有提交的乾淨 detached HEAD，同步已自行修復（重新附接預設分支並 fast-forward）；無需動作，回報只是讓自行修復可見。
- `FLEET_SYNC: <repo>: STUCK: on <state>, N commits behind <base> - needs attention`——clone 為 dirty、位於非預設分支、detached 且有獨有提交，或已分歧，因此同步保留不動（絕不強制或丟棄）；未處理前會持續落後。
  醒目的 STUCK，尤其多次 bootstrap 的 N 持續增加，代表該 clone 需要實際介入；派船員或先解決，避免工作被擱置。
- `HOME_SUMMARY: this home has never published state/home-summary.json` 或 `... has not been republished since <stamp>`——此 home 的結構化摘要發布已反覆失敗，行內帶有失敗次數與 `state/.home-summary-refresh.log` 最新記錄原因。
  發布刻意採盡力而為，不能改變其他工作階段啟動、spawn、teardown 或監看器輪詢結果；監看器將其分離執行，避免緩慢嘗試延遲存活信標。
  讀取指定紀錄的原因，再直接執行 `bin/fm-home-summary-refresh.sh` 重現（不帶 `--best-effort`，因為它會讓失敗保持安靜），讓更新錯誤傳到你這裡。
  記錄截止時間代表完整更新未在 `FM_HOME_SUMMARY_TIMEOUT` 內完成，因此先檢查鎖定取得與產生端完成，再看驗證或發布；修正受阻階段，不要提高這個關鍵時限。

- `BOOTSTRAP_INFO: closed the backlog item for <id> after interrupted cleanup; its endpoint or local copy may remain and should be reconciled`——重播已關閉項目，但持久轉換表示實體清理中斷。
  驗證程序回收、本機副本歸還與端點關閉，再對帳任何殘留資源。
- `BOOTSTRAP_INFO: kept the captain call for <id> open with its deliverable recorded after interrupted cleanup; its endpoint or local copy may remain and should be reconciled`——重播保留船長 hold 項目，但實體清理中斷。
  驗證程序回收、本機副本歸還與端點關閉，不關閉也不解除船長決策，再對帳任何殘留資源。
- `BACKLOG_RECONCILE: <id>: recorded backlog close could not be replayed: <reason>`——此次工作階段啟動找到帶有關閉或保留轉換的 pending-close 紀錄，但無法落實。
  有效 teardown 紀錄證明轉換已授權並記錄，但實體清理可能只完成一部分：假設資源已消失前，先驗證程序回收、本機副本歸還與端點關閉。
  驗證錯誤代表紀錄不可信，因此不可假設清理完成，也不可遵循其中儲存的任何路徑或引數。
  讀取指定原因；驗證失敗時，把標記視為不執行的資料檢查，修正紀錄或 backlog 檔案問題，再重跑工作階段啟動，讓有效記錄轉換重播。
  絕不手動刪除 `state/<id>.backlog-close`；這可能丟掉清理捕捉到的完成連結或船長決策保留資訊，而存留標記也會阻止紀錄掃描在此期間啟動該項目。
- `BACKLOG_RECONCILE: <id>: worker record exists but its backlog item could not be read: <reason>`——此 home 無法判定項目是否符合工作 Agent 紀錄。
  解決指定 backlog 讀取問題，再重跑工作階段啟動；絕不以啟動或關閉無法讀取項目的方式猜測。
- `BACKLOG_RECONCILE: <id>: worker record exists but its backlog item could not be moved to In flight: <reason>`——此 home 擁有的工作 Agent，其 backlog 項目仍排隊，而對帳無法修正。
  修正前，艦隊檢視會將該工作 Agent 視為沒有 backlog 項目擁有的工作；解決指定 backlog 問題，再重跑工作階段啟動。
- `BACKLOG_RECONCILE: code-root <file> is not this home's <file>; ...`——tasks-axi 寫入指向程式碼根目錄，而非此 home，因此佇列已分叉，任一副本都可能有另一份缺少的列；原因由 [`docs/configuration.md`](../../../docs/configuration.md) 的「Backlog 後端」負責。
  兩份都不能直接當作安全勝出版本：依 task id 聯集合併到此 home 檔案，對每個衝突 ID 找到最近的真實轉換，將缺少 Done 列視為遺失前先檢查此 home 封存，安裝前驗證合併 ID 集合等於兩份輸入的聯集。
  接著將程式碼根目錄檔案移到旁邊保留，不要刪除，告訴船長恢復了哪些列，之後每個 backlog 命令都透過 `bin/fm-tasks-axi.sh`；重新連結程式碼根目錄副本絕不是修正方式，因為下一次相對 cwd 的 tasks-axi 寫入又會取代連結。
- `RECORD_HYGIENE: <record inconsistency found in this home>`——向船長回報發現。
  孤立狀態日誌需說明，是由會異動的工作階段啟動封存到 `data/<id>/status.log`，還是只由偵測模式工作階段啟動或心跳掃描回報；`state/<id>.status` 仍存在就代表尚未封存。
  其他清理留待明確指示。
- `SECONDMATE_SYNC: secondmate <id>: skipped: <reason>`——二副收斂保留存活 home 現有 checkout，原因可能是 dirty、分歧、不安全、錯誤分支、缺少部署位置專屬目標提交、無法連線或其他無法 fast-forward 情況，也可能是繼承本機資料傳播失敗；bootstrap 已繼續，但應檢查原因，因為主 Agent 更新後，二副受追蹤指示、繼承設定或共用船長偏好可能已過期。
- `SECONDMATE_LIVENESS: secondmate <id>: skipped: <reason>|respawn failed after <cause>: <reason>`——工作階段啟動存活掃描無法保證已登錄二副正在執行真正 Agent 程序。
  應調查原因，因為無法保證該二副存活。
- `SECONDMATE_HANDOFF: secondmate <id>: pending delivery: <n> item(s)`——排隊工作已離開 main 可派工 backlog，因 backlog 收件或本機寄件匣清理尚未完成，仍安全留在指定遠端路由、採 backlog 格式的寄件匣；釋放契約由 [`bin/fm-backlog-handoff.sh`](../../../bin/fm-backlog-handoff.sh) 負責。
  保留該寄件匣，待路由、回執或清理問題解決後，重跑 `bin/fm-backlog-handoff.sh --resume-pending`；絕不重新加入 main backlog，也不從 main 再派這些項目。
  不安全寄件匣變體在任何重試前，都要求檢查路徑與檔案型別。
- `NUDGE_SECONDMATES: secondmate <id>: send failed: <reason>`——二副收斂改變了執行中 home 已載入指示或繼承設定，但可確定重現的 `fm-send.sh fm-<id>` 重新閱讀提醒失敗。
  檢查原因，完整保留 `state/.secondmate-nudge-pending/` 下的 pending 標記，端點或中繼資料問題修正後重跑工作階段啟動，讓 bootstrap 在同一本機或遠端路由重試完全相同、帶標記的傳送。
- `FMX: X mode on ...` / `FMX: X mode off ...`——bootstrap 已確認或移除本機 Relay 輪詢成品（`docs/configuration.md` 的「Relay (.env)」）；輸出行仍使用 Relay 舊名 `X mode`。
  只有執行中監看器需要立即套用節奏轉換時，才透過輸出的執行環境監督協定，重新啟動以 home 為範圍的監看器；bootstrap 刻意絕不自行重啟監看器。
