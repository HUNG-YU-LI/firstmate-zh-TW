<a id="hc-runtime-backend"></a>
# hc 執行期後端

hc 是 herdr-core 的實驗性工作階段提供者後端；herdr-core 是 Herdr.app 底下的終端機核心。
它提供任務終端機，Git worktree 則仍由 Treehouse 提供，與 tmux 採用相同分工。
herdr-core 不是 Herdr 1：`herdr` 後端（[`herdr-backend.md`](herdr-backend.md)）與 `herdr` 多工器溝通，兩者不共用 socket、id 空間或 CLI。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 負責定義共用的選擇方式與中繼資料語意。
本頁負責操作者設定與支援限制；`bin/backends/hc.sh` 的轉接器標頭說明實作決策。
[`verification/runtime-backends.md`](verification/runtime-backends.md#hc) 記錄對真正核心實際測試過的內容。

<a id="setup"></a>
## 設定

先決條件：

- Herdr Core 0.1.14 或更新版本，自我識別為 `hc x.y.z`，且核心使用第 1 代協定。
  執行檔解析先使用可執行的 `FM_HC_BIN` 覆寫，再找 `PATH` 上的 `herdr-core`，最後找 `hc` 別名。
  `hc` 首次使用時會啟動自己的核心。
  [`bin/fm-install-hc.sh`](../bin/fm-install-hc.sh) 負責安裝用法及經 checksum 驗證的固定發行版本；更新仍由操作者執行。
- 用來處理 JSON 回應的 `jq`。
- 滿足 [`configuration.md`](configuration.md#toolchain) 的通用執行環境與工具鏈需求。

可在本機 `config/backend` 寫入 `hc`、為單次啟動設定 `FM_BACKEND=hc`，或明確要求 Firstmate 使用 hc。
明確設定永遠優先；hc 也會在何種情況下自動選用，請見下方「執行期自動偵測」。

任務會落在派生當下 `HC_SESSION` 指定的核心；未設定時則為 hc 的 `default` 工作階段。
Herdr 應用程式顯示的是啟動時綁定之工作階段的終端機；除非啟動應用程式時指定不同 `HC_SESSION`，否則為 `default`。

<a id="runtime-auto-detection"></a>
## 執行期自動偵測

hc 只依 Herdr 應用程式的終端機層級訊號自動偵測，絕不只靠 `HC_TERMINAL_*` 標記：hc 將它們寫入每個由 hc 終端機啟動的程序，而且應用程式建立的終端機環境，刻意與其他 hc 終端機完全相同；應用程式在 `terminal.run` 呼叫傳入 `TERM_PROGRAM=herdr`，但核心的 `child_env` 會對每個子程序都正規化為 `ghostty`，不論是否由應用程式建立。
真正標示終端機屬於應用程式的，是核心端紀錄：應用程式會為自己建立的每個終端機加上 `herdr.space`／`herdr.tab` 標籤，而非空 `herdr.space` 正是應用程式將終端機投影到視窗中的確切條件。
優先順序由[後端選擇順序](configuration.md#backend-selection-order)定義；本節則定義應用程式終端機訊號。

在其他所有介面標記都已完成判定後，只有一種訊號會選擇 hc：

- `app-terminal`：本程序自己的 `HC_TERMINAL_ID` 終端機，在其核心紀錄中帶有非空 `herdr.space` 標籤。
  檢查是唯讀且有前置關卡：必須先透過 Unix socket 連線證明 `HC_TERMINAL_SOCKET` 存活，因為 `hc` 接觸不存在 socket 時會啟動核心，直接執行 `hc get` 可能建立狀態；之後才透過解析出的執行檔做一次 `hc get` 讀取標籤。
  標記遺失、socket 已死、`hc`／`jq` 工具鏈缺少或無法使用、`get` 失敗，或標籤為空，都視為未命中，絕不算偵測成功。

刻意不將存活的 `herdr-app.sock` 或開啟的視窗當作訊號：主機上每個程序都看得到它，無法區分 Firstmate 真正在應用程式終端機內，還是只在普通 shell 執行、碰巧別處開著視窗。
對真正應用程式的實測確認了這項歧義：沒有標記的程序只因視窗存在，就選擇 hc。因此只有上述終端機層級宣告能自動選用 hc。
船長若希望應用程式外的 shell 派生 hc 工作 Agent，必須明確設定 `config/backend`、`FM_BACKEND` 或 `--backend`。

此訊號綁定的是 spawn 實際會到達的核心，也就是 `HC_SESSION`（否則為 `default`）工作階段，並要求核心已經執行中；因為轉接器的容器確保步驟會啟動不存在的核心，但偵測絕不可啟動核心。
`bin/fm-backend.sh` 不呼叫 `hc`，自行解析核心 socket：先使用 `FM_HC_SOCKET` 固定值；否則若設定 `XDG_RUNTIME_DIR`，使用 `$XDG_RUNTIME_DIR/herdr-core/<session>.sock`；再否則使用 `${XDG_STATE_HOME:-~/.local/state}/herdr-core/<session>/<session>.sock`，這是 hc 0.1.14 實測的配置。
`app-terminal` 額外要求 `HC_TERMINAL_SOCKET` 必須是同一個 socket，因此其他核心上帶標籤的終端機，絕不會影響 spawn 選擇。
應用程式顯示啟動時綁定的工作階段，因此若船長的應用程式執行非 `default` 工作階段，就要為 Firstmate 設定相同 `HC_SESSION`，或明確選用 hc。

明確的 `--backend`、`FM_BACKEND` 或 `config/backend` 永遠優先於偵測。
在傳統 Herdr TUI 內，`HERDR_ENV=1` 會在檢查應用程式訊號前勝出，因此 `herdr` 後端維持既有行為；在 tmux 內，`$TMUX` 同樣優先，因此這兩種介面上無標記或標記矛盾的程序，完全不會進入應用程式檢查。
矛盾的標記組合，例如另一種介面的標記與應用程式標記並存，會保留較早介面的判定，不切換；有歧義輸入代表不自動切換。
若存在存活 hc 終端機，但找不到可用 hc 工具鏈——可執行的 `FM_HC_BIN`，或 `PATH` 上回報 `hc x.y.z`、版本不低於轉接器已驗證最低版本的 `herdr-core`／`hc`，再加上 `jq`，也就是與 spawn 容器檢查相同的版本關卡——偵測不會切換，而是以通知指出缺少項目；單純終端機標記仍永不構成偵測成功。
自動偵測的 hc 會在 stderr 印出通知，指出勝出的訊號及停用方式；其任務紀錄由[任務結構與中繼資料](#task-shape-and-metadata)定義。

自動偵測 hc 所派生的任務，帶有與明確 hc spawn 相同的 `herdr.space`／`herdr.tab` 標籤，因此工作 Agent 會出現在船長視窗中，而 Firstmate 從未啟動應用程式。
應用程式只負責本機 GUI 呈現；偵測讀取終端機的核心端標籤，不要求應用程式視窗存活，因此 GUI 關閉後，仍存活的具標籤終端機依然可以選用 hc。
此處不改變遠端 `herdr` 路徑。

自動偵測特別有兩項限制：

- 自動偵測為 hc 時，`--secondmate` spawn 會依既有規則拒絕，不會悄悄選擇其他後端；需要派生二副的主目錄，請明確設定 `config/backend`。
- 離開模式監督器探索及 hc 拒絕規則，由[離開模式監督器後端](configuration.md#away-mode-supervisor-backend-fm_supervisor_backend--fm_supervisor_target)定義。

<a id="task-shape-and-metadata"></a>
## 任務結構與中繼資料

每個任務是一個 hc 終端機，在專案目錄執行使用者的登入 shell；以 `--no-parent --outlive-parent --no-notify` 啟動，使它獨立於 Firstmate 本身所在的任何 hc 終端機。
終端機帶有以下標籤：

| 標籤 | 值 | 用途 |
| --- | --- | --- |
| `firstmate.task` | `fm-<id>` | 輸入或關閉前核對的任務身分。 |
| `firstmate.home` | `bin/fm-backend-hometag-lib.sh` 提供的 `<hometag>` | 區分擁有相同任務 id 的兩個主目錄。 |
| `herdr.space` | `FM_HC_SPACE`，預設為 `fm-<hometag>` | 將同一主目錄的任務群組到一個 Herdr.app space。 |
| `herdr.tab` | `<hometag>-fm-<id>` | 每個任務一個 Herdr.app 分頁。 |

```text
backend=hc
backend_detect=app-terminal
window=<session>:<address>
hc_session=<session>
hc_terminal=<address>
```

只有該訊號自動選擇 hc 時，`backend_detect=` 才是 `app-terminal`；明確選用 hc 不會寫入 `backend_detect=` 列。

`<address>` 是 hc 的終端機位址，例如 `t7`。
每次轉接器呼叫都會精確設定已記錄的 `HC_SESSION`，並移除任何繼承的 `HC_SOCKET`，因為 hc 否則會優先採用後者；因此即使呼叫端環境指向另一個核心，任務仍會連到自己的核心。
`FM_HC_SOCKET` 是測試與實驗環境刻意使用的 socket 固定值，用來取代繼承的 `HC_SOCKET`。
帶有任務標籤的輸入、擷取或關閉操作，會拒絕 `firstmate.task` 或 `firstmate.home` 不相符的終端機；沒有任務標籤的關閉操作，也仍會拒絕缺少此主目錄標籤的終端機。

<a id="operations"></a>
## 操作

| 操作 | hc 指令 |
| --- | --- |
| 確保容器可用 | `hc server status`，加上透過 `hc connect` 進行的 `hello` 世代交握 |
| 建立任務 | `hc run --bg --no-parent --outlive-parent --no-notify --cwd <project> --label ...` |
| 擷取 | `hc read --lines <n>` |
| 擷取可見範圍 | `hc read --lines 0` |
| 傳送一行並送出 | `hc send` |
| 送出並確認 | `hc send --wait`；見限制中的送出安全性 |
| 具名按鍵 | `hc keys` |
| 目前路徑 | `hc get`，前景程序的 `cwd` |
| 忙碌狀態 | `hc get`，原生 `status.state`（`working` 代表忙碌） |
| Agent 狀態 | `hc get`，透過共用分類器判定前景程序及其程序群組 |
| 關閉 | `hc kill`，再 `hc rm`，最後 `hc get` 必須回覆找不到 |

<a id="limits"></a>
## 限制

- `hc send` 一律會按 Enter，因此 hc 無法只鍵入而不送出。
  `bin/fm-spawn.sh` 以 `hc send` 貼上並送出派生啟動列，並消費其後的 Enter。
- 傳送訊息使用一次有時間上限的 `hc send --wait`。
  回合已開始，或等待仍顯示工作中，都可由原生機制確認；否則轉接器備援檢查輸入區畫面，只有已證明 pending 的輸入區才重試 Enter。
  被阻擋的終端機（exit 9）會回傳 `unknown`，不鍵入、不重試，也絕不強制執行。
- hc 每個終端機只回報一個前景程序。
  若該程序是啟動器，轉接器也會從 `ps` 讀取其程序群組其他成員；這只適用於本機終端機。
- 終端機剛啟動時，`hc get` 可能尚未回報前景程序；shell 就緒之前，Agent 狀態讀為 `unreadable`。
- 拒絕 `--secondmate` spawn。
- 工作 Agent 會從終端機繼承 `HC_SESSION`、`HC_TERMINAL_ID`、`HC_TERMINAL_SOCKET` 與 `HC_SCOPE`，`config/launch-env-allowlist` 的基本保留集合會保留它們；真正執行環境在 Firstmate 啟動下是否會回報原生狀態，尚未驗證。
- 命名 Herdr.app space 需要應用程式視窗，因此沒有執行。
  上述 `herdr.space`／`herdr.tab` 標籤契約，是應用程式文件定義的橋接方式：視窗將具標籤終端機投影成 space／分頁，未標記者保持隱藏。已對真正視窗驗證；[`verification/runtime-backends.md`](verification/runtime-backends.md#hc) 記錄真實測試。
- 此轉接器沒有消費任何 hc 事件串流；監督像 tmux 一樣採輪詢。

<a id="tests"></a>
## 測試

`tests/fm-backend-hc.test.sh` 使用假的 `hc` 固定驗證轉接器。
`tests/fm-backend-hc-liveness-live-e2e.test.sh` 是預設開啟的真實驗證防護：在私人核心中直接啟動每個已安裝執行環境，並要求判為 `alive`。
`tests/fm-backend-hc-smoke.test.sh` 在隔離實驗環境驅動真正執行檔；[`bin/fm-hc-lab.sh`](../bin/fm-hc-lab.sh) 負責隔離與中繼資料防誤觸契約，[`tests/hc-test-safety.sh`](../tests/hc-test-safety.sh) 負責受防護的測試清理。
`tests/fm-backend-hc-appdetect-smoke.test.sh` 在相同類型實驗環境中，針對 app-terminal 與明確選用情境執行真正的 `bin/fm-spawn.sh`，並檢查：存活應用程式視窗絕不會替無標記程序選用 hc，沒有執行中核心的工作階段也不會啟動任何內容。
找不到已解析 Core 執行檔或 `jq` 時，冒煙測試會略過；佈建及清理失敗則使測試失敗。
CI 覆蓋範圍請見 [`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend)；工作流程要求 `jq`，並拒絕 `hc not found` 的略過結果。
[`bin/fm-hc-ci-cleanup.sh`](../bin/fm-hc-ci-cleanup.sh) 負責工作結束時的清理機制。
