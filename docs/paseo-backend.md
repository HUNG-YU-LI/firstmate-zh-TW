<a id="paseo-runtime-backend"></a>
# Paseo 執行期後端

Paseo 是實驗性的 macOS GUI 終端機後端。
它提供任務工作區與終端機，Git worktree 則仍由 Treehouse 提供。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 負責定義共用的選擇方式與中繼資料語意。

<a id="setup"></a>
## 設定

若你已將 Paseo 應用程式當作終端機使用，並希望任務以分頁顯示在側邊欄，享有與 Herdr 相同的使用體驗，就可以選用 Paseo：Firstmate 本身在 Paseo 內執行時，任務放在 Firstmate 自己的工作區；否則每個專案使用一個 `firstmate` 工作區。
Paseo 僅支援 macOS，以 GUI 為主，不適合無介面或只能透過 SSH 的 Firstmate 工作階段。

先決條件：

- Paseo 0.8 或更新版本，從 [paseo.sh](https://paseo.sh) 安裝（`/Applications/Paseo.app`）。
- 用來處理 JSON 回應的 `jq`。
- 滿足 [`configuration.md`](configuration.md#toolchain) 的通用執行環境與工具鏈需求。

安裝應用程式時，不一定會將內附 CLI 安裝到 `PATH`。
轉接器優先使用 `command -v paseo`，否則使用 `/Applications/Paseo.app/Contents/Resources/bin/paseo`。

不需要設定 socket 存取。
常駐程序預設接聽 `127.0.0.1:6767`，`paseo status` 會回報可達性。
只有常駐程序單純尚未啟動時，轉接器才使用 `paseo start` 啟動它；若已啟動的常駐程序仍無法連線，就會快速失敗並指向本文件。

可在本機 `config/backend` 寫入 `paseo`、為單次啟動設定 `FM_BACKEND=paseo`，或明確要求 Firstmate 使用 Paseo。
它絕不會透過執行期自動偵測選用；見下方「明確選用」。
若 CLI、最低版本、`jq` 或常駐程序可達性不符合要求，spawn 會停止並提供可採取行動的設定訊息。

例行監督透過 `bin/fm-peek.sh <id>` 與 `FM_HOME=<home> bin/fm-send.sh <id> '<text>'` 進行，不會把 Paseo 視窗帶到前景。

派生一個小任務，並確認中繼資料包含 `backend=paseo`、`paseo_terminal_id=` 與 `paseo_workspace_id=`，即可驗證設定。

<a id="selection-is-explicit"></a>
## 明確選用

Paseo 與 zellij、orca 一樣，必須明確選用：只有 `config/backend`、`FM_BACKEND=paseo` 或 `--backend paseo` 會選擇它。
Paseo 將標記寫入每個後代程序：Paseo Agent 匯出 `PASEO_AGENT_ID`，終端機分頁匯出 `PASEO_TERMINAL_ID` 與 `PASEO_WORKSPACE_ID`，兩者都帶有 `__CFBundleIdentifier=sh.paseo.desktop`。
從 Paseo 分頁啟動的 tmux 伺服器會繼承全部標記，因此它們存在，只能證明某個祖先程序是 Paseo，不能證明船長選擇了它。
若將這些標記視為選擇依據，就會使 Paseo 成為環境隱含預設值，違反「權限必須明確，不得推論」；因此 `fm_backend_detect` 會忽略它們。

明確選用 Paseo 之後，這些標記仍然有用。
Firstmate 本身在 Paseo 內執行時，轉接器會用它們找到 Firstmate 自己的工作區，並將任務分頁放進去（見「任務結構與中繼資料」）。

<a id="task-shape-and-metadata"></a>
## 任務結構與中繼資料

Paseo 側邊欄的階層是專案 > 工作區 > 終端機分頁，每個任務對應一個終端機分頁。
Firstmate 本身在 Paseo 內執行時，不論任務屬於哪個專案，每個任務分頁都放進 Firstmate 自己的工作區，讓工作出現在船長工作階段旁邊，也不增加新的側邊欄項目。
Paseo 終端機分頁透過 `PASEO_WORKSPACE_ID` 指出所屬工作區；Paseo Agent 只匯出 `PASEO_AGENT_ID`，因此其工作區是唯一 cwd 為 Firstmate 主目錄的存活工作區；若有兩個以上，則視為未知。
其他情況下，轉接器每個專案使用一個共用工作區，與 Herdr 的容器結構相同：標題為 `firstmate`（二副主目錄則為 `2ndmate-<id>`），以 `workspace create --path <project> --isolation local --title <label>` 建立一次，後續 spawn 再依 `workspace ls` 中相同 cwd 與標題接管。
建立該工作區時，Paseo 會依路徑註冊或重用專案。
Paseo 允許重複的工作區與終端機名稱，因此若 spawn 無法讀取 `workspace ls` 或 `terminal ls` 清單，就會拒絕且不建立任何內容，不會把讀取失敗當作空清單。
任務分頁內的 Agent 可以自行開啟更多分頁或工作區；轉接器不依賴它們。

每個任務在該工作區擁有一個終端機分頁。
終端機的 NAME（`terminal create --name fm-<home-label>-<id>`）是 Firstmate 路由的權威依據，並以主目錄為範圍，讓共用同一常駐程序的兩個 Firstmate 主目錄，絕不會互相誤認對方的終端機。
工作區標題只用來識別要接管的共用工作區，絕不用於路由任務。

記錄的目標是 `<terminal_id>:<workspace_id>` 這組配對：

```text
backend=paseo
window=<terminal-uuid>:<workspace-uuid>
paseo_terminal_id=<terminal-uuid>
paseo_workspace_id=<workspace-uuid>
```

每次傳送前，都會將已記錄終端機 id 與即時 `terminal ls` 清單核對。
已記錄 id 消失時，復原會從同一清單依主目錄範圍的 NAME 重新解析終端機，絕不依標題。

<a id="current-operation-and-safety"></a>
## 目前操作與安全性

`terminal send-keys <id> -l -- <text>` 傳送字面文字，但不送出。
呼叫端另外傳送 Enter。
Paseo 0.8.0 的 send-keys 支援集合為 Enter、Tab、Escape、Space、BSpace、C-c、C-d、C-z、C-l、C-a 與 C-e。
其他按鍵名稱都會被當作字面文字鍵入，因此轉接器透過 `-l` 以原始 `0x15` 位元組送出 Ctrl-U，並拒絕其他所有未列出的按鍵。

`terminal capture <id> -S --json` 回傳已移除 ANSI 的純文字行。
轉接器會取得完整捲動歷史，再於本機裁切尾端；Paseo 將捲動歷史限制為 1024 行（已於 Paseo 0.9.2 驗證），因此無論任務執行多久，讀取都維持有界限。
不使用 `--start -<n>`，因為它會把提示列下方的空白畫面列也算進尾端。
由於擷取會移除樣式，能力描述宣告 `styled=0`；`bin/fm-composer-lib.sh` 的共用分類器遇到符號列後帶有文字時，會降級為 `unknown`，避免把閒置建議誤讀為未送出的輸入。

`terminal ls` 中終端機的 `cwd` 欄位固定在建立當下，絕不跟隨 `treehouse get` 開啟的前景子 shell。
因此派生時的 worktree 探索，會像 cmux 與 zellij 一樣，在 `pwd` 前後送出開始與結束標記，擷取標記區塊，並接回因換行而分段的路徑。

清理只透過 `terminal kill` 關閉任務自己的分頁，其他同層任務分頁保持存活。
它與每種後端的 kill 一樣採盡力而為，因此目標早已消失時會保持靜默。
若關閉的是每專案共用工作區中最後一個分頁，且專案是此主目錄 `projects/` 下自己的複本，清理會封存該工作區，但前提是它使用 Firstmate 自己的標題、不是 Firstmate 自己的工作區，且沒有 Paseo Agent 在其資料夾內工作；因為封存工作區也會封存它擁有的 Agent。
封存仍會將專案留在側邊欄，因此若沒有其他工作區使用該專案，清理接著會刪除該複本的專案；主目錄複本之外的工作區與專案絕不會被動到。
Spawn 與此清理共用一個主目錄範圍的鎖（`state/.paseo-workspace.lock`），因此新任務分頁絕不會在「確認工作區為空」與「封存」之間落入其中；spawn 無法取得鎖時會拒絕，清理無法取得鎖時則略過。
任何清單讀取失敗，也會略過此清理。
會變更狀態的 `--json` 呼叫，會將 stderr 排除於解析輸出之外，因為 Firstmate 本身在 Paseo Agent 內執行時，CLI 會在 stderr 印出 Electron 警告。
Paseo 不提供通用的 Agent 原生忙碌訊號，因此監督透過擷取／雜湊輪詢偵測畫面變化，工作 Agent 狀態則使用各執行環境轉接器的語意生命週期。

<a id="visible-tabs-versus-nested-agent-views"></a>
## 可見分頁與巢狀 Agent 檢視

Paseo 能在目前分頁內以巢狀方式顯示子 Agent，點擊後可開啟各自的檢視，而不必建立新的最上層分頁。
Firstmate 的設計仍以「每任務一個終端機」作為權威模型：每個任務在其工作區恰好取得一個終端機分頁，派生時開啟，清理時關閉。
因此巢狀子 Agent 檢視絕不會成為 Firstmate 路由或生命週期的權威依據；它只是船長可點入的額外檢視，任務的單一終端機仍是端點。

實際取捨是：任務終端機中的執行環境若自行派生子 Agent，它們會巢狀顯示在任務分頁內，而非新增最上層分頁，因此每個 Firstmate 任務恰好增加一個分頁。
這裡的取捨是 Agent 介面與終端機介面：巢狀 Agent 檢視可見且可點擊，但無法由 `fm-send`／`fm-peek` 單獨指定；這些指令永遠指向任務唯一記錄的終端機。

關閉的分頁不會保留可見的捲動歷史：`terminal kill` 回收端點後，不留下可供日後閱讀的對話紀錄。
因此，已完成任務的持久歷史位於狀態日誌、報告及 PR，絕不在分頁捲動歷史；這也正是 Firstmate 絕不把狀態列當作喚醒事件之外其他事物的原因。

<a id="active-limits"></a>
## 目前限制

- Paseo 是實驗性後端，僅支援 macOS、以 GUI 為主，且要求應用程式正在執行。
- 在每主目錄生命週期設計通過驗證之前，不支援派生二副；有此需求的主目錄請選用 `tmux`。
- 設定 `PASEO_AGENT_ID` 時，`paseo run --cwd` 會被忽略（已回報至上游 issue 討論串）；轉接器絕不使用 `paseo run`，只使用會遵守路徑的 `terminal create --cwd`。
- 沒有原生忙碌或推送事件訊號，`fm_backend_agent_state` 對 Paseo 回報 `unverified`。
- 目標可能在結構就緒檢查後、操作前消失。
- 不假設工作區與終端機 id 在常駐程序重啟後保持穩定；復原會依終端機 NAME 重新解析。

<a id="regression-entry-points"></a>
## 回歸測試入口

```sh
tests/fm-backend-paseo.test.sh
tests/fm-backend-paseo-smoke.test.sh
```

[`verification/runtime-backends.md`](verification/runtime-backends.md#paseo) 記錄目前有效的來源與真實證據。
