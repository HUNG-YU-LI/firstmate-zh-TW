<a id="tmux-runtime-backend"></a>
# tmux 執行期後端

tmux 是 Firstmate 經過驗證的參考執行期後端，也是完整支援二副主目錄的基準。
[`configuration.md`](configuration.md#runtime-backend-configbackend--fm_backend) 負責定義共用後端選擇方式與中繼資料語意。

<a id="setup"></a>
## 設定

使用 `brew install tmux` 或平台套件管理器安裝 tmux。
通用執行環境與工具鏈需求請參閱 [`configuration.md`](configuration.md#toolchain)。

若沒有明確設定或執行期自動偵測選擇其他後端，tmux 就是最終預設值。
可在本機 `config/backend` 寫入 `tmux`、為單次啟動設定 `FM_BACKEND=tmux`，或要求 Firstmate 使用 tmux，以明確選用它。
透過 `config/backend` 或 `--backend tmux` 明確選擇 tmux，會覆蓋執行期自動偵測。

第一個任務之前不需要任何佈建。

<a id="watching-the-crew"></a>
## 觀看船員工作

為取得最佳可見體驗，請在 tmux 工作階段內啟動主要執行環境：

```sh
tmux new -s firstmate
```

船員任務會成為該工作階段內的視窗，即使另一個用戶端透過群組工作階段（`tmux new-session -t <session>`）附加，也一樣。
`tmux display-message -p '#S'` 會印出其名稱，除非最近使用過另一個群組工作階段。
若主要執行環境在 tmux 外執行，Firstmate 會建立或重用一個名為 `firstmate` 的未附加工作階段：

```sh
tmux attach -t firstmate
```

每個任務視窗命名為 `fm-<id>`。

```sh
tmux list-windows -t <session-name>
tmux select-window -t <session-name>:fm-<id>
```

在已附加的任務視窗中鍵入內容，是具有權威性的直接介入。
例行監督不需要附加：`bin/fm-peek.sh <id>` 擷取有限範圍的尾端，`FM_HOME=<home> bin/fm-send.sh <id> '<text>'` 則引導已記錄端點。

派生一個小任務，並確認其 `fm-<id>` 視窗出現在選定工作階段，即可驗證設定。

<a id="current-behavior-and-safety"></a>
## 目前行為與安全性

<a id="agent-liveness-probe"></a>
### Agent 存活性探測

目標存在性檢查只能證明窗格存在。
更深入的 tmux Agent 存活性探測，會先驗證精確的視窗歸屬，再讀取程序名稱，以區分執行中的執行環境與單純閒置 shell。
它將可識別的 Claude、Codex、OpenCode、Pi、pi-signed、Grok、Kimi、Cursor、Muse、Rovo 與 AGY 程序身分分類為 `alive`，一般 shell 分類為 `dead`，可明確證實不存在的視窗分類為 `missing`，無法讀取的狀態分類為 `unreadable`，其他所有程序分類為 `ambiguous`。
這些判定背後的程序名稱詞彙由 `bin/fm-agent-process-lib.sh` 定義，並與 Herdr 轉接器共用；後者用相同名稱驗證已註冊 Agent（[herdr-backend.md](herdr-backend.md)「Restart and liveness behavior」）。
只有 `dead` 與 `missing` 允許復原，因為誤判死亡可能啟動重複 Agent。

為了正確確認身分，探測會結合兩個獨立名稱來源，不讓任一個成為唯一依據。
`#{pane_current_command}` 與窗格 tty 前景程序群組的核心 `comm` 值，呈現不同名稱欄位；哪一個保留執行檔身分，依平台而異。
前景探測也讀取 argv[0]，因此其他欄位顯示改寫後程序名稱時，精確的執行環境安裝路徑元件仍可支撐判定。
任一來源指出已驗證執行環境，就足以判為 `alive`，因為錯誤的 `dead` 是唯一會在存活 worktree 啟動重複 Agent 的判定；可讀的前景程序群組則用來確定否定判定。

刻意將第二個來源限制在前景程序群組，而非窗格所有後代程序：若整體閒置的窗格背景仍留著一個具有執行環境名稱的程序，不能因此讀成 Agent。
相同範圍界定也自然涵蓋多程序啟動器，無須特例；因此 Pi Launcher 路徑可透過 `pi-signed` 包裝器與 `pi` 引擎歸屬，即使其標題是精確的前景指令 `pi-launcher`。
直接執行檔身分 `pi`、`pi-signed` 與 `Pi` 仍只接受精確相符；相似或帶前綴的程序名稱，不會透過這些精確 Pi 家族項目被接受。
Muse 同樣固定比對精確 `muse` 啟動器身分，或已安裝的 `muse-bin-<version>` 前綴，因此 `musescore`、`amuse` 等無關名稱仍為有歧義。
omp 基於相同原因固定比對精確 `omp` 身分，因此 `ompd` 與 `comp` 仍為有歧義。
AGY、Devin 與 Command Code 同樣固定比對精確的 `agy`、`devin` 與 `command-code` 身分，因此含有這些片段的無關名稱仍為有歧義。
Polytoken 同樣固定比對精確 `polytoken` 身分，因此含有該片段的無關名稱仍為有歧義。
Cortex 也基於相同原因固定比對精確 `cortex` 身分。
Hermes 透過 `bin/fm-hermes-process-lib.sh` 使用錨定的原生 TUI 腳本或 Python 模組 argv；目前程序及輸入區證據由 [Hermes 驗證](verification/hermes.md) 定義。
若無關工具恰好提供一個精確名為 `cortex` 的執行檔，且它是窗格前景程序，仍會被分類為 Agent，因為精確名稱是 Cortex Code 唯一提供的身分。
Kiro 同樣固定比對精確的 `kiro-cli` 前景身分（其 toolbox／aim-sandbox 包裝器是前景程序，編譯後執行檔則是後代程序），因此含有該片段的無關名稱仍為有歧義。
Cursor 透過前景程序路徑或結構化 argv[0] 中的精確 `cursor-agent` 身分，或帶版本的安裝樹識別；單純 `node` 或無關的 `agent` 仍為有歧義。

CI 強制執行的可攜式回歸測試，以及選擇性啟用的真實執行環境漂移防護，依照 `.agents/skills/firstmate-coding-guidelines/SKILL.md` 定義的分工。
每次執行環境升級後，在信任更新證據之前，都要先執行真實執行環境防護。

<a id="composer-busy-state-and-delivery"></a>
### 輸入區、忙碌狀態與投遞

Agent 存活性與輸入區安全性是兩項獨立檢查。
tmux 讀取器是 `bin/fm-composer-lib.sh` 全船隊分類器的精簡轉接器：提供一次保留樣式的完整窗格擷取、`#{cursor_y}` 游標列，以及前景程序身分探測；通常由包含游標的結構決定判定結果——完整邊框（容許下框線帶標題；圓角且帶標題的上框線僅適用於前景程序精確為 `cortex` 的窗格）、單純 Agent 符號列及其換行輸入、OpenCode 左側欄，或有身分佐證的 Pi 分隔線配對。
從 OpenCode 1.18.0 起，其左側欄也會在 Build 頁尾上方各列內，靠右繪製工作目錄與分支；分類器會從結構識別這些 cwd:branch 介面元素，讓閒置窗格維持可證明的 `empty` 判定。
OpenCode 1.19.0 將模式／模型頁尾列改為弱化顯示，因此分類器會同時依據移除幽靈文字後及移除 ANSI 後的列來證明該頁尾；附日期擷取與純文字形式的安全性論證，見[執行期後端驗證](verification/runtime-backends.md#2026-10-14-opencode-1190-de-emphasised-modefooter-row-in-the-left-bar-composer)。
已識別結構中的真實文字為 pending；只有明確證明為空，才讀為 empty。
空白或其他無法識別的游標列為 `unknown`，所有使用端都延後操作；唯一例外是前景程序已證明為 Cursor 或 Cortex，因為兩者都將終端機游標停在頁尾下方，所以會改以不依賴游標的方式重讀。
這個受身分約束的例外，為其他所有窗格保留嚴格的容器證明規則，因此模態對話框、殘留框線之間的死亡 shell，或正在重繪的窗格，絕不會成為注入目標。
共用分類器只有在具邊框容器內，才接受 shell 符號作為空白 Agent 輸入區。
單純 shell 提示列為 `unknown`，因此離開模式的升級事件絕不會注入死亡 shell。

此後端不從畫面文字讀取忙碌狀態。
任務的 busy、idle、unknown 或 dead 判定，來自 `bin/fm-busy-lib.sh` 負責的語意忙碌狀態契約；其邊界由[架構文件](architecture.md#busy-state-is-semantic-per-adapter)定義。
仍保留的獨立畫面尾端忙碌備援，都限制於特定執行環境，因此某個轉接器的輸出絕不會用來分類另一個轉接器的任務。
下方送出確認及離開模式監督器窗格忙碌防護，仍會參考畫面輸出，但只用來判斷輸入能否送達，絕不決定已記錄任務狀態。
監督器防護只選擇已偵測主要執行環境的特徵，而非把所有供應商模式混成全域聯集。

`bin/fm-tmux-lib.sh` 負責精確的鍵入與送出機制。
它只鍵入訊息一次，之後只重試 Enter，直到輸入區清空。
只有已證明為空的輸入區，才是正面的投遞確認。
已建立結構中留下的文字維持 `pending`；歧義結構中的文字仍未獲證實；無法讀取或不安全的狀態仍為 unknown。
一般本機 `fm-send.sh` 文字引導，以及所有遠端文字引導，如今都不再走這個已驗證送出路徑：它們會成為持久保存的引導收件匣紀錄，再加上盡力傳送的固定通知訊號列（`bin/fm-task-inbox-lib.sh`）。
上述判定只有在本機鍵入操作層——執行環境原生呼叫及明確後端目標——才是投遞關鍵；在該路徑中，`fm-send.sh` 仍絕不重複鍵入，也不會對未確認判定假設送出已確認。其標頭定義了獨立的「已送出但未確認」結束狀態與操作者回應。

OpenCode 有一個忙碌佇列例外。
OpenCode 回合進行中時，Enter 會將訊息排入佇列，但文字會保持可見，直到回合完成。
正常重試預算耗盡後，只有在可證明忙碌的窗格中、結構上已證明 pending 的文字，才會被接受為已排入佇列；閒置窗格則仍維持 `pending`，視為 Enter 確實被吞掉。
有歧義的 pending 文字絕不套用忙碌佇列轉換。
第二種受基準狀態約束的轉換，涵蓋分類器無法識別回合中畫面的執行環境（Pi 工作時會取代其分隔式輸入區）：當且僅當鍵入文字之前窗格為閒置，且這次送出自己的 Enter 造成閒置轉忙碌，才確認投遞；這與 Herdr 原生讀取的回合開始訊號相同。
沒有該基準時，`unknown` 判定完全保留，因此看起來忙碌的窗格，絕不可能把未讀取成功的輸入區轉成確認。
`tests/fm-tmux-submit-busy.test.sh` 涵蓋忙碌及閒置窗格中，已證明、有歧義及已清空的輸入區。

<a id="limits-and-regression-entry-points"></a>
## 限制與回歸測試入口

- tmux 是參考路徑，並支援二副主目錄。

```sh
tests/fm-backend-tmux-smoke.test.sh
tests/fm-tmux-agent-liveness.test.sh
tests/fm-harness-liveness-drift-live-e2e.test.sh
tests/fm-composer-ghost.test.sh
tests/fm-kimi-harness.test.sh
tests/fm-kimi-trust-prompt-live-e2e.test.sh
tests/fm-cursor-harness.test.sh
tests/fm-muse-harness.test.sh
tests/fm-omp-harness.test.sh
tests/fm-tmux-submit-busy.test.sh
tests/fm-bootstrap.test.sh
```

[`verification/runtime-backends.md`](verification/runtime-backends.md#tmux) 記錄目前有效的前景程序與送出證據。
