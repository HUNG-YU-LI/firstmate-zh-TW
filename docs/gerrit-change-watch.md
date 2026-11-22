<a id="gerrit-change-watch-verification"></a>
# Gerrit 變更監看驗證

這是 Gerrit 合併監看的實證紀錄，與既有 GitHub、GitLab 監看並列。
內容涵蓋監看讀取哪些資料、為何讀取該欄位而非相鄰欄位，以及為何合併路徑會拒絕執行。
以下所有輸出均逐字重現，只有伺服器主機名稱、專案名稱及變更編號，統一替換為測試夾具使用的占位值。

<a id="versions"></a>
## 版本

```
$ gerrit-axi --version
gerrit-axi 0.2.0

$ jq --version
jq-1.7

$ bash --version | head -1
GNU bash, version 5.2.21(1)-release (x86_64-pc-linux-gnu)
```

<a id="the-evidence-changes"></a>
## 作為證據的變更

此處真實證據讀取的是私人 Gerrit 伺服器上的兩筆變更，因此網路外的讀者無法對相同資料重跑這些指令。
以下將伺服器稱為 `review.internal`、專案稱為 `group/apps/console`，皆為夾具使用的占位值；其餘每個位元組都是工具本身的輸出。
這些逐字紀錄建立的是 Gerrit 自身紀錄結構的性質，而非某一台伺服器的特性；`tests/fm-pr-check-security.test.sh` 的隔離回歸測試，完全不需要伺服器就能固定驗證所有性質，因此可重現的檢查是該套件，而非這些逐字紀錄。
蒐集紀錄時，變更 4200 已合併，變更 4201 則仍開啟且因審查而受阻。

<a id="status-is-read-explicitly-because-submittability-is-a-different-question"></a>
## 明確讀取狀態，因為能否送出是另一個問題

這是整個轉接器依據的關鍵事實，蒐集於 2026-09-23。

```
$ gerrit-axi show 4200 --host review.internal --json
      "change": 4200,
      "status": "MERGED",
      "submit": "OK",
      "submittable": true,
      "blocked_on": "",

$ gerrit-axi show 4201 --host review.internal --json
      "change": 4201,
      "status": "NEW",
      "submit": "NOT_READY",
      "submittable": false,
      "blocked_on": "Code-Review",
```

已合併的變更仍回報 `submit: OK`、`submittable: true`，以及空白 `blocked_on`。
已收齊核准但尚未合併的開啟中變更，也會回報完全相同的三個欄位，因為那就是「可以送出」的意思。
因此 `submit`、`submittable` 與 `blocked_on` 回答的是「這筆變更能否送出」，只有 `status` 回答「是否已送出」。
若監看建立在前三個欄位任何一個之上，就會把已核准、但還沒有人送出的變更回報為已合併。

`blocked_on` 仍是判斷就緒狀態的正確欄位，投票值則不是：Gerrit 依自身送出要求決定阻礙項目，呼叫端無法藉由加總標籤值重建這項判斷。
此轉接器沒有讀取就緒狀態，但仍記錄這個區別，因為接下來基於此紀錄建立的功能會需要它。
新的 patch set 會移除兩個阻擋投票，而 rebase 也會產生新 patch set，因此就緒讀值只對讀取當時的 patch set 成立。

<a id="the-change-number-is-the-whole-match-and-the-servers-own-url-is-not"></a>
## 變更編號就是完整比對條件，伺服器自己的 URL 不是

`gerrit-axi` 直接從 `gerrit query` 取得變更的 `url`（`src/core/changes.js`，`url: row?.url ?? null`）；Gerrit 依 `gerrit.canonicalWebUrl` 組合該欄位，未設定時就省略。
因此，從未設定自己網站位址的伺服器，其欄位會是 null；有設定時，也會使用標準主機名稱，而非讀者貼上變更 URL 時可能使用的別名。
若拿它與已儲存 URL 比較，就會啟動永遠無法喚醒的監看：輪詢在每次失敗時都保持靜默，所以這類伺服器上的變更會被無限輪詢，合併卻永遠不回報，而且無法與「一直沒有人送出」區分。

Gerrit 的變更編號在整台伺服器全域唯一，而 `--host` 已固定伺服器，因此單靠編號就能指定變更。
監看只以編號比對，不讀取其他資料判定身分；記錄的專案路徑供人類定位變更，不參與讀取。

<a id="the-host-must-be-passed-explicitly"></a>
## 必須明確傳入主機

輪詢從 Firstmate 主目錄執行，不位於任何儲存庫內。
蒐集於 2026-09-23：

```
$ cd /tmp && gerrit-axi show 4200 --json
{
  "ok": false,
  "op": "show",
  "error": "cannot determine the Gerrit host",
  "code": "HOST_UNRESOLVED",
  "kind": "config",
```

`gerrit-axi` 會優先從目前目錄的 `origin` 遠端解析伺服器，因此離開儲存庫複本後就沒有可連線的目標。
輪詢在每次失敗時都保持靜默，因此若沒有 `--host`，監看就會永遠等待一筆從未真正讀過的變更。
因此，`bin/fm-pr-poll.sh` 會從已驗證紀錄傳入 `--host`；`bin/fm-crew-state.sh` 也透過同一個明確主機讀取開啟中變更的狀態。
`--host` 只固定伺服器；SSH 使用者與連接埠仍會先沿相同的目前目錄 `origin` 路徑解析，之後才退回本機登入名稱與 29418。因此監看變更時，必須在監看器環境或 `~/.config/gerrit-axi/config.json` 設定 `GERRIT_USER`；若伺服器不使用 29418，還須設定 `GERRIT_PORT`，因為輪詢無法回報自己其實從未通過驗證。

<a id="the-poll-against-the-real-server"></a>
## 對真正伺服器進行輪詢

從任何複本之外的 `/tmp` 執行已發布的輪詢程式，蒐集於 2026-09-23。

```
$ bash bin/fm-pr-poll.sh --validated gerrit https://review.internal/c/group/apps/console/+/4200 review.internal group/apps/console 4200
merged

$ bash bin/fm-pr-poll.sh --validated gerrit https://review.internal/c/group/apps/console/+/4201 review.internal group/apps/console 4201

$ bash bin/fm-pr-poll.sh --validated gerrit https://review.internal/c/group/apps/console/+/999999999 review.internal group/apps/console 999999999
```

已合併變更輸出一行 `merged`。
開啟中變更與不存在的變更，都不輸出任何內容。

<a id="the-merge-path-refuses"></a>
## 合併路徑拒絕執行

```
$ bin/fm-pr-merge.sh task-a https://gerrit.example/c/proj/+/1
error: firstmate does not submit a Gerrit change: submitting requires an attributed human approval it must not manufacture, so a human submits the change on the server
$ echo $?
2
```

拒絕發生在任何中繼資料讀取、forge 讀取或狀態記錄之前。
送出變更代表必須先記錄 Code-Review+2；這是具名歸屬的正面聲明，表示某位人類已核准，會被同事與儲存庫稽核讀取。
伺服器允許自行核准，正是這屬於政策邊界而非能力限制的原因，因此必須由程式碼強制執行，不能只是沒有實作供應商分支。

<a id="what-the-hermetic-regression-pins"></a>
## 隔離回歸測試固定驗證的內容

`tests/fm-pr-check-security.test.sh` 不需要伺服器，即涵蓋：

- 標準變更 URL 可解析成帶供應商標籤的身分，保留完整巢狀專案路徑；惡意 URL 測試矩陣則被拒絕。
- 只有完全符合 `MERGED` 的狀態才會喚醒監看；完全可送出的開啟中變更不會。
- 指向另一筆變更的紀錄不會喚醒監看，竄改的 sidecar 也不會。
- 已合併紀錄即使 `url` 為 null、缺少或使用別名主機，仍會喚醒監看，因為變更編號就是完整比對條件。
- 變更自由文字主旨中出現 merged 字樣，無法偽造狀態。
- 缺少 `gerrit-axi` 或 `jq` 不產生喚醒，啟動監看時則會回報缺少工具。
- 啟動監看不記錄 `pr_head`：Gerrit revision 指定的是某一個 patch set，而 `bin/fm-review-diff.sh` 沒有 Gerrit 路徑可解析目前 head；因此若記錄 revision，下一次 amend 後，它就會悄悄成為被視為已審查的內容。
- 合併路徑拒絕 Gerrit 變更。
- 只有真實讀取顯示變更目前的 patch set 帶有工作 Agent 的 HEAD tree 時，啟動監看才接受指向 Gerrit 變更的 done；即使 no-mistakes 關卡分支等遠端追蹤 ref 持有該 HEAD 也一樣。patch set 不相符、未知或無法讀取時，會在記錄任何內容之前拒絕。
- 一旦啟動監看已把變更記錄為 `pr=`，之後指向它的 done 就會依該紀錄接受，不再讀取 forge，因此伺服器端 rebase 或新 patch set 不會撤銷接受結果。
- no-mistakes 的 done 若指向 Gerrit 變更，也必須由副本持有通過管線的結果才會接受：執行結果缺少或不是 pass、執行回報 `recover_custody` 或 `continue_active_run`、HEAD tree 與管線 head 不同，或無法讀取執行時都拒絕；完成復原後，即使 Change-Id 註記重寫過分支提交訊息，仍會接受。
- `published for review` 的 done 若 URL 不是標準 Gerrit 變更 URL，就會被拒絕，即使遠端追蹤 ref 持有 HEAD 也一樣。

`tests/fm-crew-state.test.sh` 同樣不需要伺服器，即固定驗證船員狀態讀取：通過執行且變更仍開啟時回報 `PR open`，已放棄時回報 `PR closed`，已合併時回報 `PR merged`；無法讀取或指向另一筆變更的紀錄，則誠實回報未知，不會誤報合併。

重新執行上述套件即可更新此紀錄；升級 `gerrit-axi` 後，請重跑上方逐字紀錄中的指令。
