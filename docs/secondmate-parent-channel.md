<a id="secondmate-parent-channel"></a>
# 二副父層通道

本說明記錄：為何二副（secondmate）主目錄中應呈報船長（captain）的結果，由腳本而非二副模型投遞，以及每一種結果由哪個腳本投遞。
`bin/fm-parent-channel-lib.sh` 負責通道契約：通道位於何處、如何附加一行，以及所有發布者共用的回傳碼。
[`remote-secondmates.md`](remote-secondmates.md) 負責將遠端形式的通道傳回父層的傳輸機制。

<a id="the-problem"></a>
## 問題

二副是在自己主目錄運作的 Firstmate，沒有人會讀它的聊天；船長與主要 Firstmate 只能看見附加到父層通道的內容。
2026-09-02，兩個二副主目錄共有四個結果沒有到達船長。
每次監看器都在一分鐘內送達父層請求，二副也完成工作，卻在自己的聊天中向「captain」回話，而沒有附加到通道。
原因是結構性的，不是單次疏漏：二副可以在本機聊天滿足 [`AGENTS.md` 的稱呼規則](../AGENTS.md#firstmate)，卻忽略章程後段的回傳通道指示。
船長對需求的描述是：「根本問題並不只限於 PR，對吧？看起來二副的任何訊息或結果都有可能漏掉。我們必須確保修正是有原則、從根本解決問題，而不是只針對這次 PR 更新漏報的症狀做局部處理。」
PR 就緒回報是觀察到的症狀，但發現、決策、阻礙與失敗都會以相同方式漏掉，因為每一種都依賴二副模型記得寫入一行。

因此設計目標是：父層通道不得依賴模型記得寫入。

<a id="the-design"></a>
## 設計

投遞規則只有一句話：腳本回報事實，二副回報判斷。
凡是在二副主目錄留下持久證據、應呈報船長的結果，都由記錄該證據的腳本，在記錄當下或下一次監督輪詢時發布到通道；章程則將二副自己的附加內容保留給判斷。

| 結果 | 二副主目錄中的持久證據 | 發布者 |
|---|---|---|
| 交付子任務的 PR 就緒 | 子任務的 `done:` PR 就緒列，可接受的寫法由下方發布者定義；註冊後的子任務紀錄含 `pr=` | `bin/fm-inactive-reconcile.sh` 在下一次輪詢附上子任務列；`bin/fm-pr-check.sh` 在註冊時附上標準 URL，若設定 [`config/ready-after-green`](configuration.md#pr-ready-line-after-green-configready-after-green)，則等檢查全綠回報後發布 |
| 偵察子任務的發現 | 子任務的 `done:` 列與 `data/<child>/report.md` | `bin/fm-inactive-reconcile.sh` 在下一次輪詢發布，附報告指標 |
| 子任務失敗 | 子任務的 `failed:` 列 | `bin/fm-inactive-reconcile.sh` 在下一次輪詢發布 |
| 子任務決策升級交由船長處理 | 二副待辦中暫停等待船長的任務 | `bin/fm-captain-hold.sh hold`，答覆則由 `answer` 發布 |
| PR 已合併 | 合併輪詢，或二副自行合併 | `bin/fm-merge-outcome-lib.sh` |
| 子任務離開主目錄 | 其最後一筆紀錄簿列 | `bin/fm-teardown.sh`；該列尚未投遞時，拒絕移除子任務 |
| 子任務靜默結束 | 目前狀態已終止，但紀錄簿沒有結果 | `bin/fm-inactive-reconcile.sh` 既有的非活動結果掃描 |
| 對已標記請求的答覆 | 由待回覆紀錄防護的關聯列 | `bin/fm-secondmate-report.sh` 從二副主目錄解析父層通道；若一行卡在本機二副同檔名的狀態檔，待回覆防護會在復原或升級之前修復 |
| 只存在於二副推理中的結果 | 無 | 僅由章程與 `AGENTS.md` 的例外條款負責 |

紀錄簿投遞會讀取檔案；對尚無投遞紀錄的 ship `done:`，另做本機 Git 可達性檢查（`bin/fm-dod-lib.sh`）：它不呼叫執行環境、forge 或目前狀態讀取器，因此對每種執行環境與執行期後端都相同。
每次投遞以回執指紋的前八個十六進位字元作為鍵，並使用上述共用附加契約；紀錄簿路徑重用非活動掃描的每指紋回執，因此重播輪詢或重新啟動不會重複投遞事件，但真正新的終止事件仍會再次投遞。
重複一行無害，漏掉一行則不行，因此二副仍可附加自己對已投遞結果的判斷；父層把腳本列視為事實，把二副列視為評論。
對已標記回覆，回報輔助程式不接受呼叫端自選目的地，本機與遠端主目錄都使用通道解析器；其腳本標頭定義確切呼叫契約。
待回覆防護只可把本機二副 `state/<mate-id>.status` 中的關聯列重述到父層通道，修復常見的父層主目錄與二副主目錄混淆，但不把任意二副主目錄中的發現都當成確認。
其他帶關聯資訊的二副主目錄狀態列，仍是寫錯主目錄的證據；遠端主目錄路由後的 `state/parent-replies.status` 已是父層通道，不會被分類為錯誤主目錄。
二副主目錄自己的狀態掃描也以相同方式處理該遠端通道：`bin/fm-classify-lib.sh` 的 `status_scan_parent_channel_exclude` 透過同一個 `bin/fm-parent-channel-lib.sh` 綁定解析對外路徑；監看器的訊號掃描與 heartbeat 最後防線、離開模式常駐程序的全面掃描，以及全船隊彙整，都只略過那個解析出的確切路徑，絕不只看檔名。
遠端回覆轉接器已將每個通道列鏡像到父層主目錄，因此在此再次彙整通道，只會造成虛假喚醒與不存在的 `parent-replies` 任務；相對地，主要主目錄或本機二副中的 `parent-replies.status` 是一般任務日誌，仍會持續參與彙整與喚醒。
漏回覆升級訊息會以可讀、符合 shell 跳脫規則的形式，包含第一次發現的完整路徑與行號。

<a id="what-is-deliberately-not-built"></a>
## 刻意不實作的內容

- 不鏡像二副聊天：聊天可能混合結果與其他對話，要判斷哪一句是結果，本身就需要模型行為，而且每種執行環境呈現回合文字的方式不同。
- 不依門檻升級子任務未處理的決策或阻礙：二副升級的決策會成為 captain hold，並被發布；二副既不回答也不升級的決策，屬於監督品質問題，可與通道投遞分開處理。
- 不新增第二個監看器或獨立掃描器：輕量的紀錄簿掃描會在每次監看器輪詢中，於既有非活動結果指令內執行，重用其回執與向上附加機制。
- 不建立孤兒生命週期：teardown 會拒絕移除尚未投遞的結果，與遇到其他未落地條件時的拒絕方式相同。

<a id="regression-coverage"></a>
## 回歸測試覆蓋範圍

`tests/fm-inactive-reconcile.test.sh` 不使用執行環境，對真正紀錄簿測試投遞：立即投遞 done 與 failed，附帶註記、PR、模式、運作狀態與報告指標；跨輪詢只投遞一次；ship `done:` 指定的 head 只存在工作副本時暫不投遞；teardown 移除副本後，待投遞項目仍能送達；仍在附加中的行；之後的一般狀態文字不會產生新的父層事件，因為非活動回執身分只綁定結構化欄位；遠端路由；非活動路徑讓位給終止紀錄簿；以及真正監看器輪詢驅動整個流程。
`tests/fm-captain-hold-lifecycle.test.sh` 涵蓋二副主目錄發布 hold、答覆及再次 hold 時的不同發生次數，以及主要主目錄不發布任何內容。
`tests/fm-pr-merge.test.sh` 涵蓋註冊時的 PR 就緒列，以及合併結果向上回報。
`tests/fm-teardown.test.sh` 涵蓋 teardown 投遞子任務最後一行，以及通道無法寫入時拒絕執行。
它也涵蓋註冊時的就緒列，以及 `config/ready-after-green` 下在檢查全綠之前暫緩發布。
`tests/fm-brief.test.sh` 固定驗證章程的通道規則。
`tests/fm-pending-reply.test.sh` 涵蓋輔助程式選定的本機路由、遠端通道分類、在錯誤升級前重述同檔名紀錄、可讀的錯誤主目錄診斷，以及任意二副主目錄發現絕不構成回覆確認的規則。
`tests/fm-parent-channel-scan-exclusion.test.sh` 使用真正的遠端、主要主目錄及本機二副測試夾具，涵蓋能辨識主目錄結構的掃描排除：監看器訊號掃描、兩個 heartbeat 最後防線、全船隊彙整，以及真正 `fm-wake-drain.sh` 的端到端流程。

<a id="live-verification"></a>
## 真實環境驗證

[`verification/secondmate-parent-channel.md`](verification/secondmate-parent-channel.md) 記錄了附日期的真實執行：真正 tmux 窗格、兩個真正監看器在每次喚醒後重新啟動、完全沒有模型，並列出每一行已投遞的父層內容及其產生的父層喚醒。
