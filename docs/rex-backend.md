<a id="rex-runtime-backend"></a>
# Rex 執行階段後端

Rex 是 [Superlogical](https://www.superlogical.com/rex) 提供的實驗性終端機多工器後端。
它提供任務分頁與其中的終端機區塊，git 工作樹則繼續由 Treehouse 提供。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 定義共用的選擇與中繼資料語意。

<a id="setup"></a>
## 設定

若你已在 Rex 中執行 Firstmate，且希望每個任務在工作階段旁擁有自己的分頁，可選擇 Rex。
Rex 伺服器必須與 Firstmate 位於同一台機器，因為 spawn 與存活檢查會將 Rex 回報的程序與檔案系統資料，與本機狀態比較。
新 spawn 只接受 `rex whoami --json` 回報的 `unix://` 端點作為本機證據；其他傳輸方式，包含本機 TCP 端點，或無法讀取的端點，都會在建立分頁或租用工作樹之前遭到拒絕。
要在其他主機執行工作 Agent，請使用[遠端二副](remote-secondmates.md)；該頁定義遠端後端選擇與伺服器就緒要求。
伺服器停止行為請見[目前限制](#active-limits)。

必要條件：

- `rex` CLI 與執行中的伺服器，都必須是 API 修訂版 5 或更新版本。
- 用於 JSON 回應的 `jq`。
- [`configuration.md`](configuration.md#toolchain) 中通用的執行環境與工具鏈要求。

Rex 將 CLI 放在伺服器套件中，並向每個區塊 export `REX_BIN_DIR`，但 shell 啟動檔經常重設 `PATH`。
因此配接器優先使用 `$REX_BIN_DIR/rex`，接著找 `PATH` 上的 `rex`，最後使用 `~/Library/Application Support/com.superlogical.rex/` 下的伺服器套件路徑。

可直接在 Rex 區塊內執行 Firstmate 來選用 Rex（見「自動偵測」），也可將本機 `config/backend` 設為 `rex`、以 `FM_BACKEND=rex` 指定單次啟動，或明確要求 Firstmate 使用它。
若 CLI、`jq`、API 修訂版或伺服器不可用，spawn 會停止並顯示可據以處理的設定訊息。

一般監督使用 `bin/fm-peek.sh <id>` 與 `FM_HOME=<home> bin/fm-send.sh <id> '<text>'`，不必將任何分頁切到前景。

可啟動小型任務，並確認中繼資料包含 `backend=rex`、`rex_server=`、`rex_session_id=`、`rex_window_id=` 與 `rex_block_id=`，以驗證設定。

<a id="auto-detection"></a>
## 自動偵測

Rex 會向每個區塊 export `TERM_PROGRAM=rex`，以及 `REX_SESSION`、`REX_BLOCK`、`REX_SERVER` 與 `REX_BIN_DIR`。
`TERM_PROGRAM=rex` 加上 `REX_BLOCK` 會選用 Rex，檢查順序在 `$TMUX` 與 `HERDR_ENV=1` 之後、cmux 之前。
在 Rex 區塊內啟動的 tmux 或 Herdr 會改寫 `TERM_PROGRAM`，但繼承 `REX_*` 變數，因此巢狀多工器會被辨識為它自己，單靠 `REX_BLOCK` 絕不會選用 Rex。
自動偵測 Rex 時保持靜默，明確的 `--backend`、`FM_BACKEND` 或 `config/backend` 永遠優先。
啟動的工作 Agent 保留 `REX_*` 變數與 `TERM_PROGRAM`，因此從 Rex 主目錄啟動的二副也會持續使用 Rex。

<a id="task-shape-and-metadata"></a>
## 任務結構與中繼資料

Rex 的階層是伺服器 > 工作階段 > 視窗（分頁）> 區塊（面板），每個任務是一個含單一區塊的視窗。
當 Firstmate 本身直接在 Rex 區塊內執行時，伺服器確認 `REX_BLOCK` 指定區塊位於 `REX_SESSION` 指定工作階段後，每個任務分頁就會在 Firstmate 自己的工作階段開啟。
伺服器無法確認的宣告會遭拒，不會猜測放置位置。
其他情況，以及二副自己的主目錄，一律使用每個主目錄一個工作階段，標籤為主目錄標籤（`firstmate-<hash>` 或 `2ndmate-<id>-<hash>`）：恰好一個工作階段帶有該標籤時採用它、沒有時建立一次、有多個時拒絕。
可取得 `REX_SESSION` 時，新建的本機二副工作階段會緊接在啟動者工作階段之後；在已採用工作階段中重新 spawn 的二副分頁，會移到第一個位置、排在其船員之前。
兩個排序步驟都採盡力而為：失敗只警告，不讓啟動失敗。
二副工作階段標籤的雜湊以二副主目錄為基礎，不使用啟動者主目錄的程式碼根目錄。

既有工作階段中的分頁以 `window new --focus=false` 建立；必須建立各主目錄工作階段時，其初始視窗就成為第一個任務分頁。
尚未顯示的區塊起始大小為 80×24，因此配接器會嘗試將每個新任務區塊調為 200×50，失敗時警告；之後顯示它的用戶端會接管並套用自己的大小。
面向使用者的分頁標籤為 `fm-<id>`；區塊則帶有主目錄範圍的標籤 `fm-<home-tag>-<id>`，只用於拒絕重複與復原清單。

紀錄的目標是 `<session-uuid>:<block-uuid>`，因為 Rex ID 本身帶有冒號（`session:<uuid>`、`block:<uuid>`）：

```text
backend=rex
window=<session-uuid>:<block-uuid>
rex_server=<server endpoint>
rex_session_id=session:<uuid>
rex_window_id=window:<uuid>
rex_block_id=block:<uuid>
```

不可變的區塊 ID 是路由權威來源。
Rex 絕不重用 ID，因此紀錄的區塊在整個生命週期都精確指向同一區塊。
每次 CLI 呼叫都以 `-s` 傳入紀錄的工作階段，因為 CLI 否則會預設使用呼叫端自己的工作階段。

<a id="current-operation-and-safety"></a>
## 目前操作與安全性

`send -- <text>` 傳送字面、尚未送出的輸入，Enter 由呼叫端另外傳送。
按鍵對應到 Rex 名稱（`enter`、`escape`、`tab`、`ctrl+c`、`ctrl+u`）；其他按鍵會被拒絕，不會猜測。
已驗證的送出只輸入酬載一次，透過共用輸入區分類器只重試 Enter；只有共用排隊 Enter 政策接受可信的原生忙碌狀態或畫面忙碌頁尾時，才會將仍待送出的文字轉為成功。

`capture --trim` 以純文字回傳捲動歷史加畫面，配接器在本機裁切尾端。
Rex 沒有僅擷取可視區域的功能，因此可視區域由一次帶樣式的 `format` 區塊方法讀取推導而來（`vt` 搭配 `extra_cursor`）；它回傳捲動歷史加完整畫面網格，包含空白列，最後以 `ESC[<row>;<col>H` 附上游標位置。
vt 網格以 CRLF 串接列，因此底部空白列會讓內容以 CRLF 結尾，該列隱含存在；配接器會還原它，再取最後 `size.rows` 列作為可視區域。
全螢幕程式的網格只有畫面本身，因此推導在兩種畫面都成立；若大小讀取與網格讀取之間發生調整大小，結果可能依尺寸差產生偏移。
輸入區分類器讀取該帶樣式的可視區域，可取得樣式、列數、程序身分與游標列，與 tmux 提供的證據相同：游標所在形狀就是輸入區，因此執行環境在輸入區下方繪製的裝飾不能決定判定。
當游標定位讀取為 `unknown`，例如執行環境把游標停在輸入區外時，改由不使用游標、取最下方形狀的讀取決定。

`process` 區塊方法回報前景程序、其 cwd、祖先與上次結束狀態。
spawn 的工作樹探索讀取前景 cwd，會跟隨 `treehouse get` 開啟的巢狀子 shell。
Agent 狀態足以作為復原依據：前景程序、任一祖先，或 shell 前景的存活後代中存在已驗證執行環境時讀為 `alive`；shell 前景上下都沒有執行環境時讀為 `dead`；終端機子程序已結束讀為 `dead`；其他情況讀為 `ambiguous`；結構化的找不到結果讀為 `missing`；伺服器無法連線則讀為 `unreadable`。
檢查後代的原因是，登入 shell 若透過同一程序群組中的包裝程式啟動執行環境，例如 fish 函式或 shell shim，執行環境在下方執行時，回報的前景程序仍是登入 shell。
被動讀取絕不啟動伺服器。

Rex 依區塊記錄 Program Status Protocol（OSC 7501）回報；版本化的回報涵蓋範圍由[工作 Agent 工具驗證](verification/runtime-backends.md#worker-tools-on-rex) 定義。
只有回報程序是前景程序、其祖先之一，或 shell 前景下執行的執行環境時，紀錄才有效，因為沒有 shell 整合時，`working` 紀錄會比回報它的指令活得更久。
與 Herdr 完全相同，這種原生狀態只能作為正面的忙碌證據，因此忙碌權威仍是執行環境自己的生命週期。

<a id="push-events"></a>
## 推送事件

監看器在輪詢間等待時，Rex 主目錄會在承載任務分頁的工作階段執行一個有界 `rex do` 讀取器（[`bin/backends/rex-eventwait.lua`](../bin/backends/rex-eventwait.lua)），只監聽那些任務的區塊。
讀取器先訂閱，再核對各區塊目前程式狀態，因此兩次等待之間轉為 `blocked` 的區塊會回報一次，核對期間發生的狀態邊緣也不會遺失。
可信的 `blocked` 程式狀態會走立即升級處理路徑，不等待面板過期計時器；[推送驗證](verification/runtime-backends.md#push-events) 記錄實測延遲。
只有回報程序通過與原生忙碌狀態相同的前景程序鏈規則，該狀態才有效；喚醒原因包含 Rex 的 blocked 類型（`permission`、`question` 或 `auth`）與程式訊息，截斷於 160 個字元。
[`bin/fm-transition-lib.sh`](../bin/fm-transition-lib.sh) 定義共用政策與各區塊升級去重契約。
與 Herdr 推送路徑相同，監看器略過二副端點、宣告等待，以及由船長暫停的工作。

被監看的區塊程式結束或區塊關閉時，等待會立刻結束，讓監看器下一次輪詢立即核對該任務。
這本身絕不會喚醒監督者，因為一般清理也會關閉區塊。

推送路徑只縮短延遲，輪詢仍是備援。
當 CLI 或伺服器低於 API 修訂版 5 或無法連線（讀取器絕不啟動伺服器）、讀取器無法開始或在截止時間前結束，以及達到監看器連續讀取器失敗門檻後（預設三次），監督都會使用輪詢。
位於第二個 Rex 工作階段的任務分頁維持輪詢，因為單一讀取器只涵蓋一個工作階段。

<a id="cleanup-and-the-absence-proof"></a>
## 清理與不存在證明

清理只關閉任務區塊，連帶關閉其分頁，接著證明它已不存在：只有關閉後精確的找不到結果才算成功，因此關閉失敗或區塊仍存活時，都會保留任務紀錄。
原本就不存在的區塊視為已關閉。
若各主目錄工作階段已沒有分頁，接著會將其終止，但只限帶有此主目錄標籤的精確紀錄工作階段，絕不終止啟動者工作階段，也絕不在無法讀取其狀態時操作。
退役與採用時，省略 `windows` 欄位會被視為空工作階段，因此沒有船員的二副可重新啟動到已清空的工作階段。
主要端退役本機二副時，會以二副主目錄身分檢查並退役該二副工作階段；仍有分頁或標籤已改名的工作階段會保留。
啟動、回收與該退役操作共用主目錄範圍鎖（`state/.rex-session.lock`），因此新分頁絕不會在空值檢查與終止之間進入工作階段。

`bin/fm-control.sh` 的 `exit` 與 `relaunch` 可在 Rex 使用；[`agent-control.md`](agent-control.md#reclaiming-a-task-whose-endpoint-is-gone) 定義已記錄伺服器的不存在證明、停止伺服器復原、回收放置位置，以及只在回收時清理 shell 空殼的行為。
一般 spawn 會拒絕任何帶有任務限定範圍標籤的區塊。

<a id="attention-marks"></a>
## 注意事項標記

選擇啟用：將 `config/rex-status-marks` 設為 `on` 後，船長可直接從 Rex 看出哪些工作 Agent 需要處理。
`bin/fm-rex-marks.sh` 將每個 Rex 任務最新狀態轉成工作 Agent 面板上的一筆程式狀態紀錄（OSC 7501、`app=firstmate`、`id=fm`），讓 Rex 在其分頁、工作階段與 Activity 項目顯示標章，且不替換面板根狀態。

| 工作 Agent 狀態 | 紀錄 | 訊息 |
| --- | --- | --- |
| `needs-decision`，或船長暫停 | `blocked`，類型 `question` | `Needs your decision: <note>` |
| 提到憑證或登入的 `blocked` | `blocked`，類型 `auth` | `Blocked: <note>` |
| 其他 `blocked` | `blocked`，類型 `permission` | `Blocked: <note>` |
| `failed` | `error` | `Failed: <note>` |
| 帶 PR URL 的 `done` | `done` | `PR ready: <note>` |
| 其他 `done` | `done` | `Done: <note>` |
| 其他狀態 | 移除 | |

每筆紀錄的標題都是任務分頁標籤 `fm-<id>`，因此 Rex 使用 Firstmate 給分頁的名稱，而不是分支或 Agent 的終端機標題。
每筆需要船長處理的新紀錄，也附帶以分頁標籤為標題的 OSC 777 桌面通知，因為 Rex 音效與通知跟隨各面板的根程式狀態，而非子紀錄。
注意事項標記寫入器絕不寫入根紀錄：根紀錄由執行環境或[工作 Agent 端狀態橋接](configuration.md#osc-7501-worker-status-reporting) 負責，同一回合第二次根回報會讓 Rex 通知兩次。
Firstmate 自己的紀錄——注意事項標記與橋接的 `app=firstmate-<harness>` 根紀錄——在忙碌偵測與推送升級處理中，都絕不算工作 Agent 狀態。
通知每次變更只觸發一次，重複同步不會觸發。
Rex 沒有設定紀錄的 API，因此標記是對面板終端機裝置的一次小型寫入；Agent 的 shell 印出下一個提示或結束時，`working` 或 `blocked` 紀錄會被移除，下一次同步再重新寫入。
只有 `config/rex-status-marks` 或 `config/rex-tint` 存在，或仍有快取紀錄時，監看器才會在每次輪詢執行一次 `fm-rex-marks.sh sync`；清理會先清除快取標記，再關閉面板。
每次寫入都定址任務紀錄的區塊 ID，因此標記絕不會落到其他面板。

`config/rex-status-marks` 將標記設為 `on` 或 `off`（檔案不存在時的預設，因此從未建立它的主目錄不會看到任何變更）；關閉時會清除已顯示的所有標記與進度紀錄。
`state/.rex-mark-<id>` 快取上一個標記，讓同步只在變更時寫入。

<a id="validation-progress"></a>
### 驗證進度

標記啟用且 ship 正透過 no-mistakes 驗證、目前沒有標記時，其面板會帶有第二筆紀錄：`id=fm/validate`、狀態 `working`，進度為已完成 pipeline 步驟比例，訊息則列出目前步驟，例如 `Validating: review 3/9, auto-fix 1/3`。
Rex 會在分頁與面板的懸停卡片顯示它，讓船長不必開啟工作 Agent，就能看出每個驗證的進展。
同步透過一次有界的 `no-mistakes axi status`，讀取任務分支上的執行；每個任務最多每 30 秒一次（`FM_REX_PROGRESS_INTERVAL`）。執行結束或任務取得標記時移除該紀錄，清除標記也會移除其下方紀錄。
`state/.rex-progress-<id>` 快取上一份計畫。

<a id="state-tint"></a>
### 狀態背景色

`config/rex-tint` 可選擇為等待船長或協助的工作 Agent（`blocked` 標記），或失敗的工作 Agent（`error` 標記），套用淡背景色：`off`（檔案不存在時預設）、適用深色佈景的 `subtle`，或適用淺色佈景的 `subtle-light`。
執行中或已完成的工作 Agent 絕不套色；標記離開該狀態、清理或停用背景色時，都會重設。
Rex 透過 API 設定的佈景，只會套用到沒有用戶端擁有的面板，但應用程式擁有它顯示的每個面板，因此背景色改用寫入面板終端機的帶內 OSC 11，重設則用 OSC 111。
它只改變程式保留預設背景的儲存格；啟動時依背景選好顏色的執行環境會保留那些顏色，而監看器停止後留下的背景色，會持續到下一次同步。
Firstmate 絕不傳送 OSC 11 查詢，因為回應會作為鍵入文字進入工作 Agent。
`state/.rex-tint-<id>` 記錄寫入的背景色，因為背景色無法讀回。

<a id="away-mode"></a>
## 離席模式

Rex 中的 Firstmate 可執行離席與安靜監督：監督者目標從 `REX_SESSION` 與 `REX_BLOCK` 探索，離席常駐程序在自己的專用 Rex 工作階段執行，絕不成為你的分頁或分割面板；離開 `/afk` 時會以精確紀錄 ID 終止它。
在 `config/wedge-alarm` 指定 `rex`，會選擇為 Firstmate 自己的面板加入卡死警報：一筆 blocked 的 `id=fm-wedge` 紀錄，加上桌面通知，於下一次摘要成功傳送後移除；`auto` 絕不加入此警報。

<a id="active-limits"></a>
## 目前限制

- Rex 屬於實驗性且仍在 beta 階段；配接器拒絕低於 API 修訂版 5 的 CLI 或伺服器。
- Rex 伺服器停止會終止每個任務的 Agent 程序。
  重新啟動的伺服器會在相同端點與狀態目錄還原工作階段本身，每個區塊以保留工作目錄的全新登入 shell 返回。
  存活檢查會在還原的工作階段重新啟動已失效 Agent；工作階段確實未返回時，復原會證明任務已不存在，再將其回收到全新分頁。
- 可視區域依上述方式推導，並非直接讀取。
- 只有回報 OSC 7501 的執行環境，會提供原生忙碌證據；其他環境則依自己的生命週期與畫面頁尾分類。
- Command Code 在 Rex 完成真實驗證前會遭拒，Cursor Agent 也尚未在 Rex 真實驗證。
- 沒有 Rex CI 工作通道；真實 Rex 驗證請見下方「回歸測試入口」。

<a id="deferred-upstream-asks"></a>
## 延後提出的上游需求

以下對 Superlogical 的需求刻意尚未提報，等待後續決策；每項都列出若實現後可消除的 Firstmate 影響。

- **僅擷取可視區域**：可直接讀取可視區域，不必從完整擷取的最後 `size.rows` 列推導，消除調整大小造成的偏移。
- **以 JSON 欄位提供游標位置**：配接器不必再從 `format` 讀取末端的跳脫序列解析游標。
- **擁有者結束時移除 `working` 與 `blocked` 程式狀態紀錄**：不必每筆紀錄都與即時程序狀態交叉比對，即可信任原生忙碌狀態。
- **文件化的最低相容版本，以及含 API 修訂版的機器可讀版本輸出**：版本閘門不必再解析 `rex version` 文字。
- **可依工作階段或區塊停用 Agent hook 自動重新命名工作階段**：任務分頁中的執行環境就無法重新命名承載船長或其他任務分頁的工作階段。
- **可設定尚未顯示區塊的預設大小**：spawn 不必將每個新 80×24 任務區塊調為 200×50。
- **即使 shell 啟動檔重設 PATH，區塊內仍可從 `PATH` 找到 `rex`**：配接器可移除 `REX_BIN_DIR` 與伺服器套件備援。

<a id="regression-entry-points"></a>
## 回歸測試入口

```sh
tests/fm-backend-rex.test.sh
tests/fm-backend-rex-smoke.test.sh
tests/fm-backend-rex-eventwait-smoke.test.sh
FM_REX_COMPOSER_LIVE=1 tests/fm-backend-rex-composer-live-e2e.test.sh
```

[`tests/rex-test-safety.sh`](../tests/rex-test-safety.sh) 定義真實 Rex 套件的私有伺服器隔離、清理、能力略過與啟動失敗處理。
冒煙測試也會停止並重新啟動自己的私有伺服器，以驗證被動讀取與工作階段遺失後回收行為。
[`verification/runtime-backends.md`](verification/runtime-backends.md#rex) 記錄真實環境證據。
