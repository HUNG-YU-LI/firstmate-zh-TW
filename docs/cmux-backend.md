<a id="cmux-runtime-backend"></a>
# cmux 執行期後端

cmux 是實驗性的 macOS GUI 終端機後端。
它提供任務工作區與 surface，Git worktree 則仍由 Treehouse 提供。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 負責定義共用的選擇方式與中繼資料語意。

<a id="setup"></a>
## 設定

若你已將 cmux 應用程式當作終端機使用，並希望任務工作區出現在它的側邊欄，就可以選用 cmux。
cmux 僅支援 macOS，以 GUI 為主，不適合無介面或只能透過 SSH 的 Firstmate 工作階段。

先決條件：

- cmux 0.64 或更新版本，從 [cmux.com](https://cmux.com) 或透過 `brew install --cask cmux` 安裝。
- 用來處理 JSON 回應的 `jq`。
- 滿足 [`configuration.md`](configuration.md#toolchain) 的通用執行環境與工具鏈需求。

安裝應用程式時，不一定會將 CLI 安裝到 `PATH`。
轉接器優先使用 `command -v cmux`，否則使用 `/Applications/cmux.app/Contents/Resources/bin/cmux`。

<a id="required-socket-access"></a>
### 必要的 socket 存取

cmux 預設的控制模式會拒絕外部 shell，而 Firstmate 一律從外部程序控制它。
首次以 cmux 後端派生任務之前，請開啟 Settings > Automation，選擇可行的 Socket Control Mode。

| 設定 | 值 | Firstmate 支援 | 安全邊界 |
| --- | --- | --- | --- |
| 關閉（Off） | `off` | 否 | 停用 socket 接聽器。 |
| 僅 cmux 程序（cmux processes only） | `cmuxOnly` | 否 | 只有 cmux 應用程式的後代程序可連線。 |
| 自動化模式（Automation mode） | `automation` | 是，建議使用 | 僅擁有者可存取的 0600 socket，允許目前 macOS 使用者的程序連線。 |
| 密碼模式（Password mode） | `password` | 是 | 0600 socket 額外要求驗證交握。 |
| 完全開放（Full open access） | `allowAll` | 是，但不建議 | 0666 socket 允許每個本機使用者在未驗證下連線。 |

建議使用 Automation mode，維持同一使用者的存取邊界。
`allowAll` 可透過任何人都能寫入的控制 socket 執行指令，因此只能在明確接受安全取捨時選用。

使用 Password mode 時，將密碼存為本機、被 gitignore 排除的 `config/cmux-socket-password` 第一行，或在 Firstmate 環境中提供 `CMUX_SOCKET_PASSWORD`。
轉接器每次都會從有效設定目錄重新讀取檔案；檔案不存在時，不會覆寫環境中已有的密碼。
請透過 cmux UI 設定模式與密碼，不要編輯 `cmux.json`；應用程式不會保留手動加入的密碼鍵，而且以 socket 進行 reload，無法修復一個正在拒絕呼叫端的 socket。

可在本機 `config/backend` 寫入 `cmux`、為單次啟動設定 `FM_BACKEND=cmux`，或明確要求 Firstmate 使用 cmux。
Firstmate 本身在 cmux 內執行時，也可透過執行期自動偵測選用。
若應用程式、最低版本、`jq`、socket 存取或密碼不可用，spawn 會停止，並提供可採取行動的設定訊息。
只有 socket 尚未啟動時，轉接器才可能以 `open -a cmux` 啟動應用程式；存取遭拒或驗證錯誤時，不會重新啟動應用程式。

例行監督透過 `bin/fm-peek.sh <id>` 與 `FM_HOME=<home> bin/fm-send.sh <id> '<text>'` 進行，不會把 cmux 視窗帶到前景。
建立任務工作區與 surface 時，使用 `focus=false`。

派生一個小任務，並確認中繼資料包含 `backend=cmux`、`cmux_workspace_id=` 與 `cmux_surface_id=`，即可驗證設定。

<a id="runtime-detection"></a>
## 執行期偵測

`CMUX_WORKSPACE_ID` 是主要的 cmux 執行期標記。
`CMUX_SOCKET_PATH` 不足以判定，因為操作者可能在 cmux 外設定它。
標記優先順序由[後端選擇順序](configuration.md#backend-selection-order)定義，因此巢狀執行於 cmux 內的多工器仍會是作用中的後端。

cmux 內附的 Claude 包裝程式，在內部 socket 探測失敗時，可能移除所有 `CMUX_*` 變數，Password mode 也包含在內。
因此僅在 macOS 上，偵測會先備援檢查 `__CFBundleIdentifier=com.cmuxterm.app`，再檢查程序祖先鏈是否通往執行中的 cmux 應用程式。
只有沒有更高優先順序的執行期標記勝出時，才會採用這些備援。
若程序環境已被清除，或由 launchd 重新收養，且沒有可靠標記，就不會自動偵測。

自動偵測只選擇後端。
絕不變更 socket 存取或授予憑證。
Spawn 拒絕訊息會說明如何完成 cmux 設定，或改回 tmux。

<a id="task-shape-and-metadata"></a>
## 任務結構與中繼資料

每個任務各自擁有一個 cmux 工作區，其中有一個 surface。
呼叫端使用的標籤仍為 `fm-<id>`，可見工作區標題則為 `fm-<home-label>-<id>`。
主目錄標籤是 `firstmate` 或 `2ndmate-<id>`，再加上解析後 Firstmate 根目錄的短穩定雜湊。
cmux 不強制標題唯一，因此建立、復原、列出及清理路徑都會驗證這個具範圍標題。
搬移 Firstmate 安裝會改變雜湊，使舊標題無法再比對；這與記錄的 worktree 路徑也會過期的情況一致。

```text
backend=cmux
window=<workspace-uuid>:<surface-uuid>
cmux_workspace_id=<workspace-uuid>
cmux_surface_id=<surface-uuid>
```

這組 UUID 是單次應用程式執行期間的有效端點權威識別。
工作區 UUID 在應用程式重新啟動後不會保持穩定，因此復原會依具範圍標題搜尋，再解析目前的 surface id。

<a id="current-operation-and-safety"></a>
## 目前操作與安全性

全新的 surface 在尚未寫入任何內容之前，`read-screen` 會回傳內部錯誤。
因此目標就緒判定使用結構性的 `list-panes` 回應，而非讀取內容。
`read-screen` 可用後，擷取仍保持有界限，並在本機裁切。

`current_directory` 會跟隨最上層 shell 的 `cd`，但不會跟隨 `treehouse get` 開啟的前景子 shell。
派生時的 worktree 探索會在 `pwd` 前後送出開始與結束標記，擷取標記區塊，並接回因換行而分段的路徑。

一般透過中繼資料路由的 `fm-send.sh` 文字引導，會成為持久保存的引導收件匣紀錄；只有固定內容、盡力傳送的通知訊號會經過 cmux 的送出機制。
在鍵入操作層，字面文字傳送與 Enter 是兩次獨立呼叫。
支援 Enter、Escape 與 Ctrl-C。
輸入區驗證器是精簡轉接器：它擷取有限範圍的純文字尾端，連同 cmux 的能力事實交給 `bin/fm-composer-lib.sh` 中全船隊共用的分類器；所有結構都由分類器負責，包括 Claude 無邊框、以 U+00A0 分隔的 `❯` 列。
`read-screen` 是純文字，也沒有游標原語，因此共用分類器遇到符號列後帶有文字時，會降級為 `unknown`，不會將執行環境自己的閒置建議誤讀為未送出輸入。
無結構的單純提示列為 `unknown`，斜線指令彈出選單的占位文字則維持 `pending`，因此只重試 Enter，絕不重新鍵入文字。
cmux 不提供通用的 Agent 原生忙碌訊號，因此監督透過擷取／雜湊輪詢偵測畫面變化，工作 Agent 狀態則使用各執行環境轉接器的語意生命週期。
只有 Grok 保留其獨立的畫面尾端備援判讀。

任務工作區最後一個 surface 不能直接關閉。
清理程序負責整個工作區，使用 `close-workspace`。
cmux 也會拒絕移除 macOS 視窗中的唯一工作區，卻回傳容易誤導的成功回應。
任務是該視窗最後一個工作區時，Firstmate 會在同一視窗建立一個不取得焦點、未命名的同層工作區，關閉任務工作區，再留下 cmux 全新的預設工作區。
該同層工作區絕不使用 `fm-` 標題，復原時會忽略它。

操作前會重新讀取確切的視窗歸屬。
被選取的工作區若不是最後一個，就正常關閉；是否被選取本身不是觸發條件。
Firstmate 不會嘗試關閉 macOS 視窗，因為 cmux socket 無法關閉仍持有運作中終端機的視窗。

真實測試共用船長執行中的應用程式，不另建隔離 cmux 工作階段。
`tests/cmux-test-safety.sh` 只允許清理目前清單中、名稱精確符合的 `fm-test-` 工作區，絕不列舉並關閉無關工作區，也不重新啟動應用程式。

<a id="active-limits"></a>
## 目前限制

- cmux 是實驗性後端，僅支援 macOS、以 GUI 為主，且要求應用程式正在執行。
- Socket 存取需要手動修改一次 Settings。
- 在每主目錄的生命週期設計通過驗證之前，不支援派生二副。
- 沒有原生忙碌或推送事件訊號。
- 目標可能在結構就緒檢查後、操作前消失。
- 唯一工作區的清理路徑會留下全新的預設工作區，無法關閉視窗。
- 標籤查找與復原目前只限於當前 cmux 視窗，因此移到其他視窗的任務是已知的復原盲點。
- 工作區 id 不會跨應用程式重新啟動保留，也絕不作為復原的權威依據。

<a id="regression-entry-points"></a>
## 回歸測試入口

```sh
tests/fm-backend-cmux.test.sh
tests/fm-backend-cmux-smoke.test.sh
```

[`verification/runtime-backends.md`](verification/runtime-backends.md#cmux) 記錄目前有效的來源與真實證據，包含 socket 模式及視窗中最後工作區的清理。
