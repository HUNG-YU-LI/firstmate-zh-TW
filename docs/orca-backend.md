<a id="orca-runtime-backend"></a>
# Orca 執行期後端

Orca 是實驗性的 macOS 後端，由 Orca 應用程式同時管理任務的 worktree 與終端機端點。
船員（crewmate）的執行環境仍是該端點內啟動的 Agent 程序。
操作或復原此後端之前，Firstmate Agent 會載入 [`firstmate-orca`](../.agents/skills/firstmate-orca/SKILL.md)。

<a id="setup"></a>
## 設定

若你已使用 Orca macOS 應用程式，且希望由 Orca 管理 worktree 與終端機，取代 Treehouse 加上工作階段多工器，就可以選用 Orca。
Orca 僅限 macOS，必須明確選用，且不支援派生二副（secondmate）。

先決條件：

- 已安裝 `/Applications/Orca.app`，且應用程式正在執行並已就緒。
- 已透過 `brew install orca` 安裝 `orca` CLI。
- 滿足 [`configuration.md`](configuration.md#toolchain) 的通用執行環境與工具鏈需求。

可在本機 `config/backend` 寫入 `orca`、為單次啟動設定 `FM_BACKEND=orca`，或明確要求 Firstmate 使用 Orca。
它絕不會被自動偵測選用。

在任何 spawn 修改儲存庫狀態之前，Firstmate 都要求 `orca status --json` 回報 `reachable=true` 與 `state="ready"`。
專案的第一個任務會視需要使用 `orca repo add --path` 註冊該儲存庫。
不需要手動註冊儲存庫。

開啟 Orca 應用程式即可查看任務的終端機。
例行監督會透過 `bin/fm-peek.sh <id>` 與 `FM_HOME=<home> bin/fm-send.sh <id> '<text>'` 使用已記錄的端點。
支援 Enter 與 Ctrl-C，不支援 Escape。

僅靠 `config/launch-env-allowlist` 無法在 Orca 限定工作 Agent 憑證的範圍，因為允許清單中的值會從 Orca 終端機自身的環境解析。
若限定範圍的工作 Agent 憑證放在 tmux 工作階段中，請設定 [Orca 環境來源](configuration.md#orca-environment-source-configlaunch-env-tmux-session)。

<a id="task-shape-and-metadata"></a>
## 任務結構與中繼資料

每個任務各有一個 Orca 管理的 Git worktree 與一個 Orca 終端機。
`fm-spawn.sh` 不會為 Orca 任務呼叫 Treehouse。
正常的隔離規則與拒絕未落地工作的規則仍然適用。

```text
backend=orca
window=fm-<id>
terminal=<orca terminal handle>
orca_worktree_id=<orca repo id>::<absolute worktree path>
worktree=<absolute Orca worktree path>
```

`window=` 仍是呼叫端使用的 Firstmate 別名。
`terminal=` 與 `orca_worktree_id=` 則是操作與清理路徑使用的後端權威識別資訊。
Orca 回傳的 `orca_worktree_id=` 由 Orca repo id 與 worktree 路徑組合而成；清理驗證要求兩部分都正確，不會把這個值當成單純名稱。

<a id="current-lifecycle-and-safety"></a>
## 目前的生命週期與安全性

Spawn 會註冊儲存庫、建立獨立 worktree、只重用 Orca 回傳且已驗證的 `result.terminal.handle` 或明確建立終端機、安裝執行環境 hooks、記錄中繼資料，最後啟動選定的執行環境。
確切的指令旗標與回應解析，由 `bin/backends/orca.sh` 及腳本說明負責定義。

`fm-peek.sh` 使用 `orca terminal read` 讀取內容。
一般透過中繼資料路由的 `fm-send.sh` 文字引導，會成為持久保存的引導收件匣紀錄；只有固定內容、盡力傳送的通知訊號會經過 Orca 的送出機制。
在鍵入操作層，`fm-send.sh` 透過 `bin/fm-composer-lib.sh` 中全船隊共用的分類器確認輸入區已清空；若斜線指令彈出選單第一次先填入引數占位文字，便只重試 Enter，不重新鍵入。
讀取輸入區時，只會擷取即時終端機尾端的有限範圍，絕不向上翻查捲動歷史，因此過期的啟動橫幅不會干擾以底部為基準的輸入區判定。
單純 shell 提示列會被判為 `unknown`，而非空白 Agent 輸入區；純文字擷取若遇到符號列後帶有文字，也會降級為 `unknown`，避免誤判成 `pending`。
監看器沒有 Orca 原生忙碌訊號，因此工作 Agent 狀態由各執行環境轉接器的語意生命週期提供。
只有 Grok 保留其獨立的畫面尾端備援判讀。

清理會保留 Firstmate 所有共用安全檢查。
偵察員（scout）仍須具備報告與完整的決策清單。
交付（ship）任務仍會拒絕清理髒工作目錄或未落地的工作。
釋放之前，清理程序會解析已記錄的 Orca worktree id，並確認其路徑與記錄中的 worktree 路徑相符。
身分識別資訊遺失、無法讀取或不相符時，會保留中繼資料並停止，不會刪除任何內容。
通過這些檢查後，Firstmate 才會關閉那個確切終端機，並透過 Orca 的 worktree 指令釋放那個確切 worktree。
絕不直接刪除 Orca worktree 的檔案。
若因 `orca` 不在搜尋路徑中，CLI 根本沒有嘗試關閉端點，即使使用 `--force`，清理也會停止並保留中繼資料：若刪除這些紀錄，磁碟上就再也沒有資訊能指出那個可能仍在運作的終端機。
請重新安裝 CLI 後再執行；[`verification/runtime-backends.md`](verification/runtime-backends.md) 的「Endpoint close」章節負責說明，此操作分支對自身關閉結果能證明與不能證明的範圍。

<a id="active-limits"></a>
## 目前限制

- Orca 僅限 macOS，且必須明確選用。
- 應用程式必須正在執行並回報已就緒。
- 不支援派生二副。
- 不支援 Escape。
- Orca 不提供穩定的 CLI 版本或協定標記，因此相容性關卡依據就緒狀態，而非最低版本。
- 只接受已驗證的終端機 handle 與 worktree 結果欄位；推測性的回應結構會被拒絕。
- Orca 的 worktree 結構尚未通過派生時 `bin/fm-claude-trust.sh` 的 Claude 工作區信任檢查驗證。該檢查拒絕任何不是與專案共用 Git common dir 的連結式 Git worktree 路徑；因此若 Orca 採用複製而非連結，Orca 上的 claude spawn 會在該檢查明確失敗，不會繼續啟動。

<a id="regression-entry-points"></a>
## 回歸測試入口

```sh
tests/fm-backend-orca.test.sh
tests/fm-spawn-orca-worktree.test.sh
tests/fm-backend.test.sh
tests/fm-bootstrap.test.sh
tests/fm-teardown-endpoint-safety.test.sh
```

[`verification/runtime-backends.md`](verification/runtime-backends.md#orca) 記錄了真實就緒狀態與回應結構的冒煙測試。
