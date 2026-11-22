<a id="zellij-runtime-backend"></a>
# Zellij 執行期後端

Zellij 是實驗性、必須明確選用的工作階段後端。
它提供終端機工作階段，任務 worktree 則仍由 Treehouse 提供。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 負責定義共用的選擇方式與中繼資料語意。

<a id="setup"></a>
## 設定

若你已將 Zellij 當作終端機多工器使用，並接受其目前在焦點、存活性判定及輪詢上的限制，就可以選用它。

先決條件：

- Zellij 0.44 或更新版本。
- 用來處理 JSON 回應的 `jq`。
- 滿足 [`configuration.md`](configuration.md#toolchain) 的通用執行環境與工具鏈需求。

可在本機 `config/backend` 寫入 `zellij`、為單次啟動設定 `FM_BACKEND=zellij`，或明確要求 Firstmate 使用它。
它絕不會被自動偵測選用。
若缺少 Zellij 或 `jq`，或 Zellij 版本低於 0.44，spawn 會在建立工作階段或取得 worktree 之前停止。

Firstmate 預設使用一個名為 `firstmate` 的共用工作階段。
可透過 `FM_ZELLIJ_SESSION` 選擇其他名稱，進行隔離驗證。
附加方式：

```sh
zellij attach <session-name>
```

例行監督不需要附加到工作階段。
請對透過中繼資料路由的端點使用 `bin/fm-peek.sh <id>` 與 `FM_HOME=<home> bin/fm-send.sh <id> '<text>'`。

派生一個小任務，並確認中繼資料包含 `backend=zellij`、`zellij_session=`、`zellij_tab_id=` 與 `zellij_pane_id=`，即可驗證設定。

<a id="task-shape-and-home-isolation"></a>
## 任務結構與主目錄隔離

每個任務都會在共用 Zellij 工作階段中取得一個分頁。
呼叫端使用的標籤仍為 `fm-<id>`，可見標題則以主目錄為範圍，格式為 `fm-<home-label>-<id>`。
主目錄標籤是 `firstmate` 或 `2ndmate-<id>`，再加上解析後 Firstmate 根目錄的短穩定雜湊。
這能防止共用同一工作階段的主要 Agent、二副及不同 Firstmate 安裝之間發生任務 id 衝突。

Zellij 不強制分頁名稱唯一，因此轉接器會自行檢查具範圍標題是否重複。
建立、復原、列出與清理路徑，都使用 `bin/fm-backend-hometag-lib.sh` 中同一套範圍標題權責邏輯。
搬移 Firstmate 安裝會改變其路徑雜湊，使舊標題無法再比對；這與搬移後 worktree 路徑也會過期的行為一致。

在主目錄標籤機制導入之前建立的任務，只有在恰好一個存活分頁帶有舊式無範圍標題時，才仍可透過其已記錄的中繼資料存取。
若有多個相同標題的舊分頁，會拒絕操作，不會猜測。
批次復原絕不接管無範圍的舊分頁，因為無法安全識別它們所屬的主目錄。

```text
backend=zellij
window=<session>:<pane-id>
zellij_session=<session>
zellij_tab_id=<tab-id>
zellij_pane_id=<pane-id>
```

記錄的窗格 id 是數字；工作階段重建後，絕不只靠這個數字信任目標。
透過中繼資料路由的操作，也會驗證所屬分頁的預期範圍標題，或無歧義的舊式標題。
明確指定原始 `session:pane` 目標，仍是只檢查窗格存在與否、供操作者使用的直接操作出口。

<a id="current-operation-and-safety"></a>
## 目前操作與安全性

即使工作階段或窗格不存在，Zellij CLI 的 action 指令仍會以 0 結束。
因此轉接器會在操作前驗證工作階段、終端機窗格及預期標題，操作後則驗證 JSON 或整數回應結構。
驗證與操作之間，窗格仍可能消失；下游的送出、worktree 探索及過期偵測會回報這個狹窄競態，而不把 exit 0 當作成功。

每次窗格操作都明確傳入 `--pane-id`，因為新工作階段可能將焦點放在版本說明外掛窗格；該外掛數字 id 與終端機窗格 id 位於不同命名空間。

`pane_cwd` 會跟隨最上層 shell 的 `cd`，但不會跟隨 `treehouse get` 開啟的前景子 shell。
因此 worktree 探索會在 `pwd` 前後送出開始與結束標記，擷取標記區塊，並接回因換行而分段的路徑。
這個主動探測只限於派生時的 worktree 探索，不是一般的即時 cwd API。

`new-tab` 沒有避免取得焦點的旗標，會暫時將已附加用戶端的焦點移到新分頁。
轉接器會記錄先前的作用中分頁，並立即以 `go-to-tab-by-id` 恢復。
這兩次呼叫之間存在短暫可見的競態，目前沒有任何 Zellij 旗標能消除。

一般透過中繼資料路由的 `fm-send.sh` 文字引導，會成為持久保存的引導收件匣紀錄；只有固定內容、盡力傳送的通知訊號會經過 Zellij 的送出機制。
在鍵入操作層，字面文字傳送使用 bracketed paste，之後再獨立、明確地送出 Enter。
送出 Enter 前，轉接器必須證明選定輸入區正規化後的內容，恰好增加貼上的文字；輸入區無法讀取、貼到其他位置，或出現無關窗格輸出時，都會失敗且不送出。
轉接器透過共用按鍵詞彙支援 `Enter`、`Esc`，以及單一引數的按鍵表示式 `Ctrl c`。
Zellij 不提供游標列或 Agent 原生狀態訊號，但 `dump-screen --ansi`（已在 0.44.0 驗證）會保留樣式，因此輸入區採用與 tmux、Herdr 相同的全船隊分類器（`bin/fm-composer-lib.sh`）讀取，判定前會移除幽靈文字及占位文字。
送出確認要求明確分類為空白的輸入區。
已退役的內容差異確認法，只要窗格因任何原因改變，例如旋轉指示器、串流輸出或時鐘，就可能回報訊息已送達，進而無聲關閉船員其實沒收到的訊息所對應的決策紀錄；如今單純窗格變化不再能確認任何事情。
已停止的窗格仍會安全失敗：Zellij 無條件 exit 0 的 action 不會擷取到任何內容，而空擷取結果會被分類為 `unknown`，絕不視為確認。

視窗擷取沒有行數上限選項。
例行讀取使用 `dump-screen`，較大範圍的 peek 則使用 `dump-screen --full`，之後在本機裁切。
較短的可見視窗可能只能提供少於要求的行數。

關閉窗格會留下空分頁。
清理程序會解析並驗證所屬分頁，再使用 `close-tab-by-id`，讓任務窗格與分頁一起消失。
真實測試的清理只使用隔離、名稱不是 `firstmate` 的工作階段，以及 `tests/zellij-test-safety.sh` 的防護；絕不呼叫刪除全部工作階段的指令。

<a id="active-limits"></a>
## 目前限制

- Zellij 是實驗性後端，且必須明確選用。
- 所有主目錄共用同一個工作階段與分頁列；具範圍標題能避免跨主目錄身分衝突，但不會建立各主目錄專屬的視覺容器。
- 沒有原生忙碌或推送事件訊號，因此監督透過畫面擷取／雜湊輪詢偵測畫面變化，工作 Agent 狀態則使用各執行環境轉接器的語意生命週期。
  只有 Grok 保留其獨立的畫面尾端備援判讀。
- 沒有經驗證的 Agent 程序存活性訊號，因此死亡的 Zellij 二副會回報為無法確定，而非自動重新派生。
- 恢復新分頁焦點存在短暫可見的競態。
- CLI 結束狀態沒有判定意義；目標仍可能在結構就緒檢查後消失。
- worktree cwd 探索需要派生時的標記探測。
- 有歧義的舊式無範圍標題，需要手動清理並重新派生。

<a id="regression-entry-points"></a>
## 回歸測試入口

```sh
tests/fm-backend-zellij.test.sh
tests/fm-backend-zellij-smoke.test.sh
```

真實冒煙測試使用唯一工作階段與受防護的刪除操作。
[`verification/runtime-backends.md`](verification/runtime-backends.md#zellij) 記錄目前有效的 CLI 矩陣與生命週期證據。
