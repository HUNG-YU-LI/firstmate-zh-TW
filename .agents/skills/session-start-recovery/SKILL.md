---
name: session-start-recovery
description: 當工作階段啟動摘要回報未完成檢查、可採取行動的診斷、復原輸入，或需要解讀的輸出時載入。
user-invocable: false
metadata:
  internal: true
---

<a id="session-start-recovery"></a>
# 工作階段啟動復原

摘要本身不呼叫外部網路，也絕不等待網路。
工作階段啟動必須進行的每項網路檢查——GitHub 驗證、二副復原、二副收斂、待送達交接，以及專案 clone 更新——都在摘要的阻塞路徑之外，由 `bin/fm-startup-network.sh` 負責的有界限工作程序執行，並回報於摘要自己的 `NETWORK CHECKS` 區段。
持有鎖定時的啟動非作用中結果掃描也加入該工作程序，避免緩慢的本機目前狀態讀取阻擋摘要；其發現使用一般持久化喚醒佇列。

1. **鎖定**——在任何共用狀態異動前，先取得個別 home 工作階段鎖定，再啟動上述延後啟動階段。
2. **Bootstrap**——只偵測的檢查（工具／版本問題、worktree 糾結檢查、執行環境覆寫、派工設定檔驗證、backlog 後端狀態）一律執行，但例行確認預設保持安靜。
   無法取得鎖定時，worktree 糾結檢查使用唯讀建議措辭，不提供 checkout 修復命令。
   home 本機過期 Herdr 投影清理，以及六項會**異動**的 bootstrap 掃描——同 home backlog 對帳、艦隊同步、二副收斂、二副存活、待處理遠端交接重試、Relay 成品寫入——只有本工作階段確實持有步驟 1 的鎖定才執行；其中四項網路操作在延後階段執行，不在本區段。
   二副存活掃描會涵蓋每個已登錄二副，並以 `SECONDMATE_LIVENESS:` 行回報略過或失敗的復原。
   掃描與監看器依節奏設關卡的 tick 共用[復原契約](../../../bin/fm-secondmate-liveness-lib.sh)；T3 執行緒請遵循[原生復原指南](../../../docs/t3code-backend.md#restart-and-liveness-behavior)。
3. **喚醒佇列**——持有鎖定時，排空並呈現持久化喚醒佇列，不在同一路徑執行非作用中結果掃描，並醒目印出原始紀錄，作為本回合第一個工作佇列；有效 `signal` 紀錄後可接明確標示的狀態事件註記，包含呈現游標處仍未讀取的每一行狀態，但絕不取代原始紀錄或目前狀態對帳；失效的監看器鏈也會透過相同防護警報在此浮現。
   呈現過的紀錄仍持久存在，直到處理回合執行 drain 印出的綁定世代確認命令。
   每次持鎖 drain，只要持久化決策紀錄仍開啟，就會印出有界限的艦隊全域 `OPEN DECISIONS` 區段，即使佇列本身為空也一樣；繼續前先對帳這些項目。
   若任務最新面向船長的狀態事件沒有相應監督分支結果涵蓋，main drain 也可能印出有界限、一次性的 `STATUS OUTCOME BACKSTOP`；即使佇列已無紀錄，也應視為恢復的喚醒處理。
   同一次 drain 會在無上限的 `UNREAD STATUS` 區段印出自上次呈現以來，仍未讀取的每一行 `note:` 與待回覆解決結果，避免被後續例行行埋住的答案遺失；呈現後不再重印那些行。
   它也印出有界限的 `RECORD DIVERGENCE` 區段，列出狀態日誌讀為已解決、但 backlog 任務仍保留 hold 的每個船長決策；不會代你關閉任何項目，對帳由 `captain-hold-lifecycle` 負責。
   無法取得並驗證鎖定時，不碰佇列，因為未授權任何工作階段異動；防護的糾結／監看器存活警報仍以唯讀建議模式印出，不提供 drain、監督修復或 checkout 修復命令。
4. **監督操作指示**——在喚醒佇列之後、兩份摘要之前，摘要針對偵測到的主 Agent 執行環境輸出恰好一個操作區塊，接著附上約束它們的一次閱讀契約。
   指令碼本身絕不啟動監督；輸出的執行環境協定負責精確等待或喚醒機制。
5. **艦隊狀態摘要**——在一次閱讀契約後、上下文摘要前，輸出 `bin/fm-session-start.sh` 負責的精簡 backlog 清單；每份 `state/<id>.meta`；每項任務 `state/<id>.status` 的有界限尾端（標為喚醒**事件**歷史，而非目前狀態，並印出完整日誌路徑供深入閱讀）；離開或安靜模式（`state/.afk-contract`，以及有 daemon 執行時的 `state/.afk` daemon 旗標）；還有對每項任務所記錄後端端點的一次低成本存活／死亡讀取。
   該存活行只是快速存在檢查，不是完整狀態讀取；需要船員實際目前狀態（執行步驟，不只是「窗格是否存在」）時，仍用 `bin/fm-crew-state.sh <id>` 讀取。摘要刻意不對每項任務做這種較深、較慢的讀取，以維持快速且有界限。
6. **網路檢查**——在艦隊狀態摘要後，呈現延後階段結果，或明確說明哪些尚未確認。
   唯讀工作階段完全不執行網路檢查，並會明說。
7. **上下文摘要與下一步**——大量內容區段中的最後一段，輸出 `data/projects.md`、`data/secondmates.md`、`data/captain.md`、`data/captain-shared.md` 與 `data/learnings.md` 的完整內容，每份都有明確分隔，最後附上結尾提醒。
   檔案不存在時會印出明確 `ABSENT` 標記，絕不與存在但為空的檔案混淆：不存在有其意義（沒有 `captain.md` 代表使用 firstmate 儲存庫內建預設值，沒有 `projects.md` 代表從 `projects/` 下的 clones 重建，等等）。
   結尾提醒會指回輸出的監督區塊，只保留 lock、afk、Relay 與一次閱讀提醒。

Bootstrap 先偵測、詢問同意，只有船長在目前工作階段核准後才安裝或升級。
必要啟動工具備齊、GitHub 驗證正常前，不可派工；呈現功能可用性依 `bootstrap-diagnostics`，不阻擋非視覺工作。
GitHub 使用 `gh-axi`，瀏覽器工作使用 `chrome-devtools-axi`，視覺決策或報告使用相容的 `lavish-axi`；查看目前 help，不要背旗標。
安靜的 bootstrap 區段不需任何動作；任何印出的可採取行動診斷行，都要載入 `bootstrap-diagnostics` 並遵循其負責程序。
`BOOTSTRAP_INFO:` 行是已完成、無需行動的事實，不必載入技能。
啟動時二副同步、存活與繼承本機資料的收斂，由 `secondmate-provisioning` 負責。
