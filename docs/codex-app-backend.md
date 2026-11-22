<a id="codex-app-backend-boundary"></a>
# Codex App 後端邊界

Codex App 不是可選用的 Firstmate 執行期後端。
Codex Desktop 的主機工具可以建立並監督可見的執行緒；提供已授權路徑時，這些執行緒也能寫入 Firstmate 狀態檔。但 Firstmate 目前沒有受支援、可由 shell 呼叫的橋接方式來存取這些主機工具。
手動維護的執行緒清冊不等於後端。

<a id="acceptance-contract"></a>
## 驗收契約

未來的 Codex App 後端必須滿足與終端機轉接器相同的生命週期契約：

1. 建立任務端點，並回傳可持久保存的執行緒 id。
2. 將初始指示及之後操作者的訊息傳送到該端點。
3. 讀取足夠的即時狀態或有範圍限制的對話紀錄，以監督任務。
4. 封存、終止，或用其他方式停止那個確切端點。
5. 讓執行緒將 Firstmate 正常的生命週期紀錄行附加至 `state/<id>.status`。

狀態回傳通道是必要條件。
可見的執行緒若無法回報到 Firstmate 正常的生命週期，就不算完整後端。

<a id="current-blocker"></a>
## 目前的阻礙

Firstmate 後端腳本使用[執行期後端參考文件](configuration.md#runtime-backend-configbackend--fm_backend)列出的 shell 可呼叫傳輸方式。
Codex Desktop 主機工具可供 Desktop 對話使用，但任意 Firstmate 子程序無法使用。
缺少的元件是 Codex Desktop 支援、可由 shell 呼叫的傳輸方式，而不是另一份本機清冊。

`codex app-server --stdio` 提供一些實用的 JSON-RPC 操作，例如啟動執行緒、啟動回合、讀取執行緒，以及封存執行緒。
單一程序的探測能建立並封存執行緒紀錄，但尚未找到受支援的橋接方式，讓 Firstmate 在完整生命週期中，建立、延續、讀取並封存同一個由 Desktop 擁有的可見端點。
直接代理 Desktop 控制 socket 並不是受支援的傳輸方式。
這些局部能力不足以支持將 `codex-app` 加入已知或可派生任務的後端登錄表。

<a id="required-bridge"></a>
## 必要的橋接介面

Codex Desktop 提供下列任一受支援介面後，才能開始實作：

- 封裝建立、傳送、讀取與封存等主機工具操作的 CLI；
- 有文件說明、訊框格式穩定的 JSON-RPC 或 MCP 傳輸方式；或
- 持續維護的輔助程式，可使用受支援的傳輸方式，並將純 JSON 回傳給 shell 轉接器。

橋接介面必須提供以下語意：

```text
create: task id, worktree request, initial instructions -> thread id, cwd, state
send: thread id, text -> accepted or rejected
read: thread id, bounded cursor -> transcript and live state
archive: thread id -> archived or stopped
return: thread appends state/<id>.status lifecycle lines
```

介面可用後，Firstmate 應新增真正的 `bin/backends/codex-app.sh`，持久保存 `backend=codex-app` 與 `codex_app_thread_id=`，並透過共用派送器路由 spawn、send、peek、watch 及 cleanup 操作。

<a id="rollout"></a>
## 推出順序

先支援交付（ship）與偵察（scout）任務。
在建立、傳送、讀取、狀態回傳及封存均透過正常後端派送器完成驗證之前，二副（secondmate）支援仍不在範圍內。
在那之前，Codex App 仍是受阻的後端邊界，具有已驗證的主機工具能力紀錄，但不是可選用後端。

[`verification/runtime-backends.md`](verification/runtime-backends.md#codex-app-host-tools) 負責維護目前有效的 Desktop 主機工具冒煙測試紀錄，且不公開特定任務的執行緒 id 或本機路徑。
