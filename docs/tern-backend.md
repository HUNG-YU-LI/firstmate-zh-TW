<a id="tern-runtime-backend"></a>
# tern 執行階段後端

tern 是實驗性的 Stencil Tern 工作階段提供者後端。
它提供任務工作階段、分頁與區塊，git 工作樹則繼續由 Treehouse 提供。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 定義共用的選擇與中繼資料語意。
`bin/backends/tern.sh` 的配接器標頭定義設計決策與實測 CLI 事實；本頁是面向操作者的對應說明，不得與它矛盾。

<a id="setup"></a>
## 設定

當 Firstmate 本身已在 Tern 區塊內執行（或你以其他方式操作執行中的 Tern 工作階段常駐程序），且接受下列[目前限制](#active-limits)時，可選擇 tern。

必要條件：

- `PATH` 上有 Tern 0.9.4 或更新版本（`tern --version`）；若缺少二進位檔，bootstrap 會提供 `https://stencil.so/tern` 的手動安裝指引，不會提供可直接執行的安裝指令。
- 用於 JSON 回應的 `jq`。
- [`configuration.md`](configuration.md#toolchain) 中通用的執行環境與工具鏈要求。
- 可連線的 Tern 工作階段常駐程序，以便透過指令碼執行 `tern ls`／`tern new`／`tern capture`。
- 只有在需要透過 `tern ctl` 讀取原生 Agent 狀態時，才需要 Tern 視窗控制端點；缺少它時原生狀態會降級，不會讓後端失敗。

可將本機 `config/backend` 設為 `tern`、以 `FM_BACKEND=tern` 指定單次啟動，或明確要求 Firstmate 使用 tern。
執行階段絕不會自動偵測並選用它；請參閱下方「選擇必須明確」。

若 `tern` 或 `jq` 不可用，spawn 會停止並顯示可據以處理的設定訊息；建立容器時會檢查版本至少為 0.9.4。
一般監督使用 `bin/fm-peek.sh <id>` 與 `FM_HOME=<home> bin/fm-send.sh <id> '<text>'`，不必將焦點切到任務區塊。

可啟動小型任務，並確認中繼資料包含 `backend=tern`、`tern_session_id=`、`tern_tab_id=`、`tern_block_id=` 與 `window=<session-id>:<block-id>`，以驗證設定。

<a id="operator-and-test-knobs"></a>
### 操作者與測試調整項目

| 變數 | 用途 |
| --- | --- |
| `FM_TERN_SESSION` | 選用的工作階段**名稱**覆寫值，僅供測試／冒煙測試；預設為 `bin/fm-backend-hometag-lib.sh` 產生的 `fm-<hometag>`。不可單憑此值進行破壞性清理。 |
| `FM_BACKEND_TERN_CTL_TIMEOUT` | 每次 `tern ctl` 呼叫的秒數上限（預設 `5`）。 |
| `FM_BACKEND_TERN_SUBMIT_POLLS` | 每次 Enter 確認預算中的原生狀態取樣次數（預設 `6`）。 |
| `FM_BACKEND_TERN_SUBMIT_MIN_SLEEP` | 在取得閒置基準後、輪詢原生狀態前，每次 Enter 的最短確認預算（預設 `0.6`）。 |

<a id="selection-is-explicit"></a>
## 選擇必須明確

tern 與 zellij、orca、Paseo 一樣，只能明確選用：只有 `config/backend`、`FM_BACKEND=tern` 或 `--backend tern` 能選擇它。
Tern 會在工作階段常駐程序啟動的每個區塊中，設定 `TERM_PROGRAM=tern` 與 `TERN_PANE=<block-id>`（以及 `TERN_PANE_SOCKET` 與 `TERN_WINDOW_KEY`）。
在區塊內啟動的 tmux 伺服器或 Herdr 用戶端會繼承全部標記，因此它們的存在只證明某個祖先是 Tern，絕不代表船長（captain）選擇了它。
若將它們解讀為選擇，就會把 tern 變成環境預設值，違反「權限必須明確，絕不推定」；因此 `fm_backend_detect` 會忽略這些標記。

明確選擇 tern 後，這些標記仍然有用。
清空環境的啟動會保留 `TERN_*` 身分，並與所有後端的基本啟動規則一致，保留面板原本的 `TERM_PROGRAM`；在 Tern 區塊內此值為 `tern`（請見 [`configuration.md`](configuration.md#variables-retained-and-where-values-come-from)）。`tests/tern-test-safety.sh` 也會拒絕關閉含有呼叫端自身 `$TERN_PANE` 區塊的工作階段。

<a id="task-shape-and-metadata"></a>
## 任務結構與中繼資料

每個主目錄擁有的容器，是一個名為 `fm-<hometag>` 的 Tern 工作階段（測試中則可為 `FM_TERN_SESSION`）。
`tern new session` 會為該工作階段建立一個初始的一般 shell 分頁；配接器不會動它，讓最後一個任務分頁關閉後，工作階段仍然存活。
每個任務是一個保持開啟的分頁，標題會重新命名為 `fm-<hometag>-<id>`（並讀回確認）；面向呼叫端的標籤仍為 `fm-<id>`。

ID 是 64 位元整數，在各 Tern 視窗中全域唯一。
firstmate 端點與紀錄的 `window=` 目標是 `<session-id>:<block-id>`（兩側都只能是數字）。
傳給 tern 的 CLI 選取器，一律使用單一工作階段格式 `<session-id>/<block-id>`，讓 tern 本身拒絕紀錄工作階段以外的區塊，也無法將單獨的數字 ID 視為重新命名工作階段。

```text
backend=tern
window=<session-id>:<block-id>
tern_session_id=<session-id>
tern_tab_id=<tab-id>
tern_block_id=<block-id>
endpoint_task_id=<id>
```

數字 ID 必須端到端保留完整 64 位元精度。
tern 的每份 JSON 文件在進入 jq 前，都會將帶有 ID 的鍵（`id`、`pane`、`session`、`tab`、`block`）之值加上字串引號，避免大於 2^53 的值被四捨五入。

每次目標操作都會透過最新的 `tern ls --json` 重新驗證：區塊必須存在於紀錄的工作階段，且提供標籤時，所屬分頁標題必須與限定範圍的標題相符。
若清單無法讀取、區塊移往其他工作階段或標題不符，都會拒絕操作，不會猜測。

<a id="current-operation-and-safety"></a>
## 目前操作與安全性

<a id="capture"></a>
### 擷取

- 歷史／`capture` 使用 `tern capture <session>/<block> --scrollback --surfaces`，再於本機裁切至要求的行數上限。
- 可視區域／`visible_capture` 與輸入區讀取使用 `tern capture <session>/<block> --surfaces`（不含 `--scrollback`），若清單提供區塊的 `rows`，再依該值裁切。
- Tern Surface Protocol（TSP）原生 Agent 畫面，透過一般／`--ansi` 擷取時回傳空值；必須使用 `--surfaces` 才能取得逐字紀錄文字。
- `--surfaces` 會印出 JSON 物件 `{"block","text"}`；兩條擷取路徑都會先讀取 `.text` 再裁切行數，非此類物件的輸出則原樣使用。
- 沒有原生身分或樣式分類器證據時，輸入區能力刻意採保守設定：`styled=0`、`cursor=0`、`identity=0`，`rows=` 則來自 `FM_COMPOSER_CAPTURE_LINES`。
  原生 `program` 名稱尚未證實可作為執行環境身分，而實測主機對原生繪製區塊使用 `--ansi` 會得到空值，因此不宣告樣式與身分能力，也不宣稱具有推測性的身分接線。
  缺少樣式或身分時，共用分類器會將字形後帶有文字的列降級為 `unknown`，不會將閒置建議當成已輸入內容。

<a id="input-and-submit"></a>
### 輸入與送出

輸入操作有明確區分：

- `tern send <session>/<block> text TEXT` 輸入字面文字，不按 Enter。
- `tern send <session>/<block> keys COMBO` 傳送組合鍵；配接器會將 `Enter`→`enter`、`Escape`／`Esc`→`escape`、`C-c`→`ctrl+c`、`C-u`→`ctrl+u` 正規化，其餘組合鍵則原樣傳遞。
- `tern run <session>/<block> TEXT` 一次完成輸入一行與 Enter（`send_text_line`）。

按鍵名稱、`send text` 處理看似旗標的文字，以及 `tern run` 語意，取自 Tern 說明與任務摘要，**尚未**在此主機進行真實驗證。

一般透過中繼資料路由的 `fm-send.sh` 文字引導，會成為持久保存的引導收件匣紀錄；只有盡力而為的固定門鈴訊息，會經過 tern 的送出機制。
在鍵入平面，文字只輸入一次；Enter 分開處理；重試只會按 Enter，絕不重新輸入文字。
可取得 `tern ctl` Agent 狀態時，送出確認會優先使用原生閒置基準，再等待 `working` 或 `waiting_input` 的回合已送達訊號；原生狀態為 `unknown` 時，與 cmux 一樣，送出會落入共用輸入區重試核心。
`waiting_input` 可作為送出確認的回合已送達訊號，但不是排隊 Enter 的忙碌訊號。

<a id="native-busy-and-agent-state"></a>
### 原生忙碌與 Agent 狀態

原生 Agent 狀態（僅限桌面視窗控制端點）透過有界的 `tern ctl` 呼叫讀取（`FM_BACKEND_TERN_CTL_TIMEOUT`，預設 5 秒）：

1. `tern ctl carly lua "$FM_BACKEND_TERN_AGENTS_LUA"`，儲存下列視窗 Agent 清單投影的回答：

   ```lua
   local out = {} for _, a in ipairs(cx.agents:list()) do table.insert(out, {pane = a.pane, state = a.state, alive = a.alive}) end return tern.json.encode(out)
   ```

2. `tern ctl state` → `.carly.last_tool.text`，移除開頭的 `→` 與任何全新 Lua 狀態提示橫幅。

Carly 的回答欄位放的是 Carly 顯示的程式結果文字，而不是由配接器控制的 JSON 文件。
顯示文字中，Agent 項目的自由文字（`last_message`、`title`、`running.line`、`last.line`）可能含有原始換行與其他控制字元，而 JSON 字串不允許這些字元，因此單一多行訊息就會讓整個陣列無法解析。
所以投影只要求配接器會讀取的欄位（`pane`、`state`、`alive`），絕不可擴充加入可能包含人類或模型所寫文字的欄位。
回答中的 ID 與其他 tern ID 一樣加上引號，且不會修補顯示的回答：無法解析時就讀為 `unknown`。

`native_state` 保留 Tern 自己的用語：`working` | `idle` | `waiting_input` | `unknown`。
每個 Agent 項目的 `pane` 等於區塊 ID。

`busy_state` 印出 `busy` | `idle` | `blocked` | `unknown`：

| 原生狀態 | `busy_state` |
| --- | --- |
| `working` | `busy` |
| `idle` | `idle` |
| `waiting_input` | `blocked`（Tern 表示等待人類的用語；對應 Herdr 的 blocked 狀態） |
| 缺少項目／`ctl` 失敗／無法讀取 | `unknown` |

Herdr 自己的 `busy_state` 會把 blocked 合併為 idle；tern 則保留額外的 `blocked` 區分。
派送器 `busy_state` 的共用消費端會檢查精確字詞 `busy`（`fm-busy-lib.sh` 原生分支、supervise-daemon 的 `pane_is_busy`、task-inbox 門鈴），因此 `blocked` 會與 herdr 的 idle 一樣被分類為非忙碌。
這個配接器不會修改這些消費端。
當 `tern process` 顯示前景為互動式 shell 時，`busy` 判定會降級為 `unknown`。

`agent_state` 使用清單加上 `tern process <session>/<block> --json`，並以共用執行環境程序分類器分類前景程序：

| 情況 | 判定 |
| --- | --- |
| 目標無效、`ls` 無法讀取或 `process` 失敗 | `unreadable` |
| `ls` 成功，但區塊不存在 | `missing` |
| 區塊存在於其他工作階段 | `ambiguous` |
| 區塊已結束（`live:false` 或非 null 的 `exited`） | `dead` |
| 前景被分類為 Agent | `alive` |
| 前景為互動式 shell，沒有指令／指令碼引數 | `dead` |
| shell 正執行指令（`sh -c ...`）或其他非 Agent 程序 | `ambiguous` |

配接器本身絕不印出 `unverified`。

`tern ctl` 失敗時（沒有視窗、無頭環境、遠端面板沒有本機視窗、逾時），原生路徑回傳 `unknown` 並退回畫面分類，不會讓後端報錯。
第一個 `ctl` 呼叫失敗，就會略過第二個。
此主機上的 `tern ctl state` 與 `tern ctl carly lua ...` 都立即失敗，回傳 `{"ok":false,"error":"no Tern window on this machine shows this pane"}`。
此主機無法成功透過 `ctl` 讀取 Agent 狀態。

<a id="path-kill-and-test-safety"></a>
### 路徑、終止與測試安全

`current_path` 採被動方式：讀取 `tern process` 的 `foreground.cwd`，失敗時回傳空值，不會用 shell 標記探測。

終止操作會以 `tern close <session>/<block>` 關閉任務區塊。
區塊已不存在時回傳 0；清單無法讀取、區塊移往其他工作階段、區塊仍在紀錄工作階段但其分頁不再帶有任務的限定範圍標題，或關閉失敗後區塊仍存在時，則回傳 1 並輸出 stderr（不關閉任何項目，因此 teardown 會保留任務紀錄）。

真實測試可能共用船長執行中的 Tern 常駐程序，而非自行建立隔離視窗。
`tests/tern-test-safety.sh` 只允許清理目前清單中，名稱帶有 `fm-test-` 前綴且 ID 是測試所建立的工作階段。
它會拒絕空白或非數字 ID、名稱不符、無法讀取的清單，以及任何含有呼叫端自身 `$TERN_PANE` 的工作階段。
唯一允許的拆除方式為 `tern_safe_kill_session <session_id> <want_name>`；測試絕不列舉並關閉無關工作階段或船長正在使用的遠端區塊。

<a id="active-limits"></a>
## 目前限制

- tern 屬於實驗性功能，沒有專用的真實後端 CI 工作通道。
- 不支援啟動二副（secondmate）（`fm-spawn.sh` 在 `backend=tern` 時拒絕 `--secondmate`）。
- 離席模式 `/afk` 監督程式後端不支援 tern；只接受 `tmux`、`herdr` 與 `rex` 監督面板。
- 原生 Agent 狀態需要 Tern 視窗控制端點；可執行 `ls`／`capture` 的遠端主機或無頭常駐程序存取，仍可能缺少 `ctl`。
- 多工器會吞掉 TSP：承載 tmux 或 Herdr 的 Tern 區塊，絕不顯示原生 Tern Agent 狀態。
- 沒有已驗證的原生推送事件等待（`tern events` 不視為 Agent 狀態轉換串流；`has_push` 仍僅限 herdr）。
- 在此觀察到的 Tern 0.9.4 遠端主機區塊中，對 omp TSP 畫面進行一般與 `--ansi` 擷取會得到空值，`--surfaces` 則回傳不含 omp 輸入區的逐字紀錄文字，因此只靠輸入區的送出證據對 omp 為 `unknown`；`ctl` 可用時，原生狀態才是送出路徑。
- 建立工作階段或分頁時不會嘗試還原焦點；此處尚未真實驗證焦點行為。
- 目標可能在結構就緒後、操作前消失。
- 大型 64 位元 ID 是權威值；強制轉換 JSON 數字的工具可能破壞端點。

<a id="regression-entry-points"></a>
## 回歸測試入口

```sh
tests/fm-backend-tern.test.sh
tests/fm-backend-tern-smoke.test.sh
```

[`verification/runtime-backends.md`](verification/runtime-backends.md#tern) 記錄目前有效的原始碼、假 CLI 與真實環境證據邊界，包含此主機能與不能證明的事項。
