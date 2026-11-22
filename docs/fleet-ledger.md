<a id="fleet-activity-ledger"></a>
# 船隊活動紀錄簿

船隊活動紀錄簿是選擇性啟用、僅能附加的檔案，外部工具可透過它追蹤 Firstmate 主目錄正在做什麼：派送了哪些任務、工作 Agent 回報了什麼、PR 何時可供審查、工作何時合併，以及何時清理完成。
它是穩定且有文件定義的 Firstmate 狀態整合入口；本頁就是其契約。

<a id="turning-it-on-and-off"></a>
## 開啟與關閉

在 Firstmate 主目錄建立存在性旗標 `config/fleet-ledger` 即可啟用紀錄簿，刪除即可停用。
旗標是本機、被 gitignore 排除、以主目錄為單位的設定，不會繼承到二副主目錄；因此每個要發布紀錄簿的主目錄，都需要自己的旗標。
旗標不存在時，每個產生者只做一次檔案存在性檢查，除此之外什麼都不做：不啟動程序，也不寫入任何內容。

<a id="the-file"></a>
## 檔案

紀錄簿位於該主目錄的 `state/fleet-ledger.jsonl`，採 JSON Lines 格式：每行一個 JSON 物件，並以換行字元結尾。
紀錄只能依寫入順序附加。

每筆紀錄都包含以下成員：

| 成員 | 意義 |
| ------- | --------------------------------------------------------- |
| `v` | 紀錄格式版本，目前為 `1` |
| `ts` | 寫入紀錄時的 Unix 時間，單位為秒 |
| `event` | 下列五個事件名稱之一 |
| `task` | 此紀錄所對應的 Firstmate 任務 id |

讀取端必須忽略不認識的成員與事件，讓後續版本能新增內容，而不破壞既有讀取端。

<a id="events"></a>
## 事件

| 事件 | 額外成員 | 寫入時機 |
| ------------------ | ---------------------------------------------- | ------------ |
| `task.dispatched` | `kind`、`project`、`harness`、`model` | 啟動新的工作 Agent 或二副時。既有任務的重新啟動不會記錄。 |
| `task.status` | `state`、`key`、`text` | 擷取任務狀態日誌中完整且非空白的一行時。 |
| `task.pr_ready` | `pr` | Firstmate 將任務的 PR 記錄為可供審查時。 |
| `task.merged` | `via`（`"pr"` 或 `"local"`）；`via` 為 `"pr"` 時另含 `pr` | 記錄任務的 PR 合併，或其僅本機分支已落地時。 |
| `task.cleaned_up` | 無 | 任務的工作 Agent 與本機副本已移除時。 |

`task.dispatched` 的成員：`kind` 為 `ship`、`scout` 或 `secondmate`；`project` 是專案目錄名稱，遠端二副則為 `null`；`harness` 是 Agent 工具名稱；`model` 是要求的模型，使用工具預設值時為 `null`。

`task.pr_ready` 的成員：`pr` 是 PR 的完整 URL。
每當 Firstmate 為任務記錄 PR 時就會寫入；因此註冊替代 PR，或再次註冊同一個 PR，都會新增一筆紀錄；但若是在合併過程中再次記錄 PR，則不會新增。

`task.status` 的成員：`state` 是狀態列開頭的字詞，例如 `working`、`needs-decision`、`blocked`、`paused`、`done`、`failed` 或 `resolved`；若沒有則為 `null`。
`key` 是該行 `[key=...]` 的決策鍵，或 `null`。
`text` 是狀態列第一個冒號之後的原樣文字，上限 2000 個字元；若沒有冒號，則為整行。

範例：

```json
{"v":1,"ts":1790132857,"event":"task.dispatched","task":"fix-login","kind":"ship","project":"webapp","harness":"claude","model":null}
{"v":1,"ts":1790132870,"event":"task.status","task":"fix-login","state":"working","key":null,"text":" bug reproduced"}
{"v":1,"ts":1790133400,"event":"task.status","task":"fix-login","state":"done","key":null,"text":" PR https://github.com/acme/webapp/pull/7 checks green"}
{"v":1,"ts":1790133410,"event":"task.pr_ready","task":"fix-login","pr":"https://github.com/acme/webapp/pull/7"}
{"v":1,"ts":1790133900,"event":"task.merged","task":"fix-login","via":"pr","pr":"https://github.com/acme/webapp/pull/7"}
{"v":1,"ts":1790133960,"event":"task.cleaned_up","task":"fix-login"}
```

<a id="limits"></a>
## 限制

- 啟用紀錄簿時，使用現行指示中狀態指令的工作 Agent，會在附加狀態列後立即記錄該行。
  監督監視器的定期輪詢是最後防線：它會記錄即時寫入漏掉的行，也不會重複記錄已由即時寫入記錄過的行。
  下列內容仍可能比狀態日誌落後最多一個輪詢間隔；若沒有監視器執行，則要等到下次監視器執行：
  - Firstmate 自己寫入任務狀態日誌的行，例如已記錄的答覆、啟動失敗、轉送的待回覆訊息，或二副回報列；
  - 使用舊版指示的工作 Agent 所寫的行，或未執行指示完整指令、只自行附加的行；
  - 遠端二副回報、透過 Firstmate 轉送才到達此主目錄的行；
  - 即時記錄失敗期間寫入的行，例如紀錄簿檔案無法寫入時。
  記錄 `task.pr_ready`、`task.merged` 或 `task.cleaned_up` 之前，會先記錄該任務尚未擷取的狀態列。
- 除非寫入失敗，或當機遺失尚未刷新寫入的紀錄，否則擷取的狀態列至少會投遞一次：擷取中斷可能造成重複紀錄，因此不能重複計數的讀取端應能容忍重複。
- 若工作 Agent 恰好在啟動與寫入派送紀錄之間寫入狀態列，狀態紀錄可能出現在該任務 `task.dispatched` 之前。
- 主目錄啟用紀錄簿時，存活任務日誌中已存在的狀態列，會在該任務下一次擷取時記錄（可能由工作 Agent 狀態指令、PR 註冊、合併、清理或監視器輪詢觸發）；旗標不存在期間派送或清理的任務，不會補上那些事件紀錄。
- 沒有序號，也沒有缺漏偵測。
- 寫入採一般附加，不強制刷新到磁碟，因此機器當機可能遺失最新紀錄。
- 檔案不會輪替，會持續成長，直到被截斷。
  若要截斷，先停止讀取，再執行 `: > state/fleet-ledger.jsonl` 清空；後續紀錄會附加到空檔案。
- 紀錄簿會原樣複製主目錄 `state/` 內的狀態文字，不額外清除敏感內容，因此對讀取端的信任程度應與 `state/` 完全相同。

<a id="not-included"></a>
## 未包含的內容

以下是可能的後續項目，本版刻意不納入：

- 工作階段啟動、離開模式及安靜模式事件；
- 重新啟動事件；
- 工作 Agent 目前是忙碌或閒置，以及回合何時結束；原生活動觀察請參閱[執行期後端事件來源](architecture.md#runtime-session-backends)；
- 序號與缺漏偵測；
- 檔案輪替，以及跨輪替檔案的連續性；
- 啟用紀錄簿之前事件的回填或重播；
- 超出狀態列原有處理範圍的秘密資訊清除，以及比 `state/` 更強的隱私保證。

`bin/fm-fleet-ledger.sh` 的標頭負責定義寫入機制，並列出每個產生者。
