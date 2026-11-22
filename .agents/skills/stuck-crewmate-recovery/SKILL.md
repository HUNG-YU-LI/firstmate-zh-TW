---
name: stuck-crewmate-recovery
description: >-
  當工作階段啟動摘要回報一般直屬工作 Agent 的端點死亡或中繼資料沒有 window，或遇到過期喚醒、窗格迴圈、反覆困惑、簡報已回答的問題、船員無回應、引導失敗，觀察到 Claude 或 Codex bypass 失效，以及存活船員回報 no-mistakes pipeline 死亡、無法存取或逾時時載入。
user-invocable: false
metadata:
  internal: true
---

# stuck-crewmate-recovery

當工作階段啟動摘要回報一般直屬工作 Agent 的端點死亡或中繼資料沒有 window，或直屬工作 Agent 過期、反覆迴圈、一再困惑、提出簡報已回答的問題、無回應，或引導未送達時，使用此處置流程。
若工作 Agent 的 pull request 已整合，過期或端點死亡回報不屬復原案例：工作已完成，應透過一般拆除關閉任務（firstmate 依 `AGENTS.md` 第 7 節，監督分支依 `bin/fm-branch-prompt.sh` 的已整合工作規則），不要走本流程，且絕不使用 `--force`。

因審查而停駐的工作線（`bin/fm-crew-state.sh` 讀為 `source: review-park`）是刻意停止 Agent，不屬復原案例；只能透過 `bin/fm-park.sh resume` 重新啟動。

復原託管看板的工作 Agent 時，遵循 [`docs/configuration.md`](../../../docs/configuration.md#crew-hosted-lavish-review-boards) 的船員託管 Lavish 看板契約。

透過 `bin/fm-control.sh <task-id> interrupt|exit|relaunch` 中斷、停止與重新啟動工作 Agent；此命令自行解析記錄的執行階段，驗證每個動作，且絕不拆除或丟棄任何內容（[`docs/agent-control.md`](../../../docs/agent-control.md)）。
此平面涵蓋在本 home 執行的工作 Agent；部署於遠端的二副會明確以名稱拒絕，改透過 `secondmate-provisioning` 對帳。
執行恢復命令、執行環境專屬技能呼叫，或配接器自身特性會影響處理時，先載入 `harness-adapters`。
目標視窗的執行環境記錄於 `state/<id>.meta` 的 `harness=`。

<a id="managed-approval-failover"></a>
## 受管理核准容錯切換

觀察到 Codex 在 bypass 下仍要求命令核准，或 Claude 政策拒絕 bypass 工作階段後，向船長要求明確核准，才能改變所屬 home 的工作 Agent 權限行為並重新啟動。
只有取得這句明確同意後，才載入 `harness-adapters`，使用 `FM_HOME=<owning-home> bin/fm-approval-failover.sh apply <id>`；持久設定與受防護復原由其檔頭負責，不做自動偵測。
回報已儲存、經模型審查的補救方式與實際復原結果，保留明確設定與工作，不改供應商政策，也不回答核准或 hook 信任提示。
`backend=t3code` 工作 Agent 已以 T3 的 `full-access` 執行階段模式運行，因此命令會在任何變更前拒絕；請在 T3 Code 本身核准待處理要求（[T3 待處理要求程序](../../../docs/t3code-backend.md#push-events-and-polling-fallback)），或將工作移到其他後端。

<a id="session-start-reconciliation-for-a-dead-ordinary-direct-report"></a>
## 一般直屬工作 Agent 死亡時的工作階段啟動對帳

此程序涵蓋一般 `kind=ship` 與 `kind=scout` 直屬工作 Agent。
`kind=secondmate` 復原改載入 `secondmate-provisioning`。

對**遠端**二副，`fm-crew-state` 與 `fm-peek` 透過 `fm-on.sh` 讀取實際遠端端點，`fm-send` 對已送達但待確認的引導會回報已送達（契約由各自檔頭負責）；`unknown-remote` 讀取或主機無法連線失敗，代表無法讀取遠端狀態，絕不代表二副死亡或傳送失敗。
真正卡住的遠端二副，只能透過 `bin/fm-spawn.sh <id> --secondmate` 復原，絕不直接操作 herdr 窗格 close/kill，否則會留下孤立端點綁定。

將摘要端點結果視為存在訊號，不是任務工作或驗證執行已消失的證明。
決定重新啟動前，先用 `bin/fm-crew-state.sh <id>` 讀取目標目前狀態。
與船員分支與目前程式碼相符的 no-mistakes 執行，在端點死亡時仍是權威來源：終止或停駐執行依一般生命週期處理，作用中執行則繼續監督，不建立重複工作 Agent。

沒有權威執行能說明任務時，只檢查其記錄後端與 worktree 清單。
以 treehouse 為基礎的 tmux、herdr、zellij、cmux、Paseo、Rex、t3code、tern 或 hc 任務使用 `treehouse status`；Orca 任務使用記錄的 `orca_worktree_id=` 與 `terminal=`。
不要掃描其他 home 的端點，也不要從相符視窗標籤推斷擁有權。

重新啟動前，證明沒有存活 Agent 仍擁有記錄任務，且既有 worktree 仍可用。
保留未提交變更與提交，維持同一任務身分，在該既有 worktree 中，用相同簡報加精簡進度註記，恢復或重新啟動記錄的執行環境。
HERDR 端點若不只是閒置，而是已被摧毀——Herdr 變動中移除了窗格或工作區——同一 relaunch 就能復原：在既有 worktree 建立一個新端點，並重新綁定任務紀錄；不需特殊處理，worktree 保持不動（[`docs/agent-control.md`](../../../docs/agent-control.md) 的「重新認領端點已消失的任務」）。
該 relaunch 在重新綁定前會證明端點已被摧毀，因此僅停止的 Herdr 伺服器會被重新接回，不會重複建立。
tmux 沒有重新認領：任務紀錄不帶端點 socket 身分，因此無法區分 `missing` 視窗與位於本席位無法存取的 tmux 伺服器上的視窗，`exit` 與 `relaunch` 都會拒絕。
T3 Code 沒有替換式 relaunch：執行緒保留對話，因此回合或供應商變更都是延續同一 Agent；保留任務紀錄與 worktree，遵循 [`docs/t3code-backend.md`](../../../docs/t3code-backend.md#restart-and-liveness-behavior)，不要嘗試重新 spawn。
不要以重新 spawn 繞過這些拒絕；保留記錄任務，遵循後端復原契約。
重新認領只由所屬 home 操作，二副是唯一例外：如上透過 `bin/fm-spawn.sh <id> --secondmate` 復原。
記錄 worktree 尚未釐清時，不要使用全新一般 spawn，因為配置另一個 worktree 可能把同一任務分散到兩份副本。
若無法安全對帳 worktree 或擁有權，保留全部狀態，並連同衝突證據回報任務失敗或受阻。

<a id="a-live-crewmate-claiming-the-pipeline-is-dead"></a>
## 存活船員宣稱 pipeline 已死亡

這與上述端點死亡案例相反：工作 Agent 還活著，而它宣稱死亡的 pipeline 通常也活著。
drive 呼叫會阻塞直到下一個關卡或結果，遠超過執行環境允許單一命令執行的時間；daemon 則立即接受回應，並在背景執行該輪。
因此船員的 drive 呼叫逾時、被終止或報錯，會讓它等待從未取得的讀取結果；據此推論「daemon 不見了」只是猜測，不是證據。

相信此說法前，親自讀取兩個權威來源：

1. `no-mistakes daemon status`，確認 socket。
2. `no-mistakes axi status --run <id>`，確認執行，或使用 `bin/fm-crew-state.sh <id>`；後者已摺合此矛盾，若執行仍 running 或 fixing 且有新活動，會將非 socket 的 daemon／逾時 `blocked:` 行回報為已被後續事實取代，因為執行仍存活。

`daemon status` 顯示連線遭拒或 socket 遺失，是 daemon 停止的明確證據，即使持久化執行紀錄仍寫 running 或 fixing，也必須升級處理；daemon 結束後該紀錄可能過期。
否則，若執行仍 running 或 fixing 且最近有活動，該說法就是錯誤：引導船員從自己的 worktree 用 `no-mistakes axi run` 重新附接，只要執行仍符合其 `HEAD`，此操作安全且具冪等性，並告訴它逾時不代表 daemon 死亡。
此情況無需通知船長。

絕不因船員的說法就重新啟動、停止或更新共用 daemon。
它是服務所有工作線與 home 的單一實例，重新啟動會終止其他工作線進行中的執行。
只有明確 socket 拒絕或不存在，才構成 daemon 停止的發現；將該發現，或明確指出 daemon 錯誤的失敗執行紀錄，升級給船長。

<a id="live-endpoint-escalation"></a>
## 存活端點逐級處理

依序升級：

1. 查看窗格，並檢查任務引導收件匣（`state/<id>.inbox/`）中未處理的 `*.msg` 紀錄；過期喚醒若指向未讀 firstmate 指示，代表工作 Agent 從未確認持久化引導，而紀錄本身精確顯示原意。
   工作 Agent 目前狀態若為已宣告的 `paused:` 等待，只代表它說明正在等什麼，因此應驗證它指出的條件，例如自己的背景工作的終止紀錄或程序；若條件已解除，應引導它讀取結果並恢復，不要把等待回報為健康。
2. 船員若在等待簡報已回答的問題，T3 執行階段要求依 [T3 待處理要求程序](../../../docs/t3code-backend.md#push-events-and-polling-fallback) 處理。
   窗格後端則從作用中的 firstmate 工作階段，透過 `FM_HOME=<this-firstmate-home> bin/fm-send.sh` 用一行回答，除非 `FM_HOME` 已設為目前作用中的 firstmate home。
3. 船員若困惑或反覆迴圈，先用 `FM_HOME=<this-firstmate-home> bin/fm-control.sh <task-id> interrupt` 中斷，再透過 `fm-send` 用一行修正指示重新導向。
4. 船員在重新導向後仍真正卡住，且後端支援替換時，使用 `FM_HOME=<this-firstmate-home> bin/fm-control.sh <task-id> relaunch --note '<progress so far>'` 重新啟動；命令會停止 Agent，將簡報與註記帶入同一本機副本的替換 Agent，若替換無法開始，還原先前紀錄。
   T3 Code 不要 relaunch；保留任務，連同目前伺服器狀態回報卡住執行緒，交由操作人員處理。
   若工作 Agent 應換成不同執行階段，在同一命令傳入 `--harness`、`--model` 或 `--effort`。
   真正卡住是反覆迴圈、無回應、重複同一障礙，或確實死亡。
   上下文讀值偏低不是卡住；現代執行環境會自動壓縮並繼續。
   worktree 與提交會保留，因此 relaunch 成本低。
5. 第二次 relaunch 也失敗時，在 backlog 寫入 `failed`，並依 `AGENTS.md` 第 9 節，以直白方式告訴船長失敗、保留工作與後果；除非路徑本身是採取行動所需，否則不要提中繼資料、執行環境、視窗或 worktree。
