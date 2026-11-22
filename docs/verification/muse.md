<a id="verification-the-muse-muse-code-crewmate-adapter"></a>
# 驗證：muse（Muse Code）船員配接器

firstmate 的 muse 配接器目前有效的實測證據。
以 [`.agents/skills/harness-adapters/SKILL.md`](../../.agents/skills/harness-adapters/SKILL.md) 為根的技能樹定義操作事實；本紀錄說明這些事實如何建立，以及哪些部分仍未證實。

<a id="subject"></a>
## 驗證對象

| 欄位 | 值 |
|---|---|
| 版本 | `Muse Code 0.1.0 (0.1.0-R708.1)`，建置 SHA `427a430436` |
| 驗證日期 | 2026-08-05，並於 2026-08-06 以需要憑證的多步驟冒煙測試擴充 |
| 成品 | `muse-aarch64-macos`，sha256 `4290bfafa5bbb81a6fd493aaea12f848c789b1d22edfa0c4b849151deba3e70c` |
| 平台 | macOS arm64（Darwin 25.5.0） |

二進位檔取自公開發布管道，並在執行前確認其校驗碼與公開資訊清單相符：

```
$ curl -sS 'https://api.meta.ai/muse-code/channels/muse-stable'
{"channel":"muse-stable","version":"0.1.0-R708.1",...,"state":"public","min_version":null}

$ shasum -a 256 muse-bin
4290bfafa5bbb81a6fd493aaea12f848c789b1d22edfa0c4b849151deba3e70c  muse-bin
```

以下每次執行都在暫存目錄使用隔離的 `XDG_CONFIG_HOME` 與 `XDG_DATA_HOME`，搭配可拋棄的 git 工作區，並以 firstmate 操作船員面板的方式透過 tmux 驅動。
刻意略過 `install.sh`，因此不會觸碰主機的 shell 設定檔或 `~/.local/bin` 項目。

<a id="what-the-model-provider-limits"></a>
## 模型提供者造成的限制

除了提供者驗證提示之外，以下真實 TUI 與工作階段行為都是針對內建的 `--provider echo` 啟動提供者觀察。
憑證路徑與未驗證等待，則另以預設的 `meta` 提供者探測。
回合邊界結構、信任對話框、插斷、結束、輸入區繪製、憑證行為與事件日誌結構，皆為真實且已驗證。
真正的多步驟、真實模型工具迴圈中的忙碌狀態行為，於 2026-08-06 另以預設 `meta` 提供者與實際模型驗證，記錄於[需要憑證的多步驟冒煙測試](#the-credentialed-multi-step-smoke-verified-2026-08-06)。

<a id="verified-facts"></a>
## 已驗證事實

<a id="process-identity"></a>
### 程序身分

公開啟動器會以 `exec` 執行帶版本後綴的二進位檔，因此每次自動更新都會改變執行中的程序名稱：

```
$ grep -nE 'muse-bin|exec ' launcher.sh
969:  candidate="$work/muse-bin"
977:  target="$dir/muse-bin-$version"
1035:  printf '%s/muse-bin-%s\n' "$dir" "$version"
1135:  exec "$binary" "$@"
```

`ps -o comm= -p <pid>` 回傳可執行檔完整路徑，其 basename 為 `muse-bin-<version>`。
因此，`bin/fm-harness.sh` 與 `bin/backends/tmux.sh` 都會比對固定在開頭的前綴 `muse-bin-*`，而非完整名稱；兩者也都無法依賴安裝路徑元件，因為 `~/.local/bin/muse-bin-<version>` 不含 `muse` 路徑元件。
Muse 啟動時，會在工作 Agent 開始前清除 `CLAUDECODE`、`PI_CODING_AGENT`、`GROK_AGENT`、`FM_PI_HARNESS`、`CURSOR_AGENT` 與 `CURSOR_INVOKED_AS`；這是已驗證的啟動行為，而非偵測所依賴的條件。
[執行環境偵測優先順序](runtime-backends.md#harness-detection-precedence) 說明為何殘留的其他環境標記無法覆蓋帶版本的祖先程序證據。

[`runtime-backends.md`](runtime-backends.md#agent-liveness-name-sources) 定義因此得到的 tmux 存活判定，以及它與可攜式誘餌回歸測試的關係。

<a id="turn-lifecycle"></a>
### 回合生命週期

兩回合工作階段恰好產生兩組 run 起訖，第二組由 Escape 插斷關閉：

```
9  {"kind":"run","run_id":"d352a097-...","event":{"kind":"started","prompt":"hello from firstmate"}}
45 {"kind":"run","run_id":"d352a097-...","event":{"kind":"terminal","terminal":"completed","turn_duration_ms":8152}}
49 {"kind":"run","run_id":"b50dac92-...","event":{"kind":"started","prompt":"second turn to interrupt"}}
78 {"kind":"run","run_id":"b50dac92-...","event":{"kind":"terminal","terminal":"cancelled","reason":"cancelled during model step"}}
```

日誌的第一筆紀錄帶有工作區綁定鍵：

```
"payload_type": "runtime.session.metadata",
"payload": {"kind":"metadata","record":{"workspace_root":".../muselab/ws1","provider_id":"echo",...}}
```

在持續 25 秒的執行中回合內取樣，狀態彙整會即時轉換：

```
T+ 5s fold=busy
T+10s fold=busy
T+15s fold=busy
T+20s fold=busy
T+25s fold=busy
T+30s fold=settled
```

真實日誌中觀察到兩種誘餌，由 `tests/fm-muse-harness.test.sh` 的回歸測試固定涵蓋：
一種是巢狀的 `"record":{"kind":"terminal"}` 清理效果酬載，並不是 run 終止事件；另一種是 `subagent/<child-session-id>/session.jsonl` 下獨立的子 Agent run 生命週期。
同一套回歸測試也驗證唯一解析結果會被快取、當日主要工作階段命名空間變更時會恢復為不明確的 unknown、替代 spawn 綁定會選取其新主要日誌、遺失快取日誌時採取拒絕策略，以及快取的子 Agent 路徑會被拒絕。

<a id="autonomy-trust-and-sandbox"></a>
### 自主性、信任與沙箱

新的未信任工作區會顯示信任對話框，預先選取選項 1：

```
Do you trust this workspace?
> 1  Trust and continue
  2  Quit
Use Up/Down or 1/2, then Enter. Esc quits.
```

`--yolo` 會完全略過它，狀態列則回報 `echo · <workspace> · YOLO`。
這點很重要，因為核准與沙箱預設都啟用，而 `--sandbox-network` 預設為 `proxy-only`；二進位檔表示這需要受管理的 shell 沙箱，但船員需要一般 git 與網路存取。

<a id="credentials"></a>
### 憑證

`muse auth set --provider` 只接受 `meta`。
未驗證的啟動不會結束，而是無限等待：

```
  Sign in at this page:
    https://auth.meta.com/oauth/device/?code=DGXZ-NRPR
  Waiting for approval…
  Esc cancel
```

因此，`bin/fm-spawn.sh` 會預檢工作 Agent 可取得的 `META_API_KEY` 或 `<config>/muse/auth.json`，並在建立端點前拒絕不符合條件的情況。
只有呼叫端擁有的 `META_API_KEY` 會被拒絕，因為長時間執行的後端常駐程序不會繼承它；非機密的 `XDG_CONFIG_HOME` 與 `XDG_DATA_HOME` 根目錄則會在預檢與轉送前解析成絕對路徑，讓已儲存憑證與工作階段日誌綁定到達同一個工作 Agent 環境。

<a id="foreign-personal-context"></a>
### 外部個人上下文

互動式 TUI 會拒絕僅供 `exec` 使用的旗標：

```
$ muse --no-foreign-personal-context --provider echo hi
invalid TUI options: error: unexpected argument '--no-foreign-personal-context' found
  tip: a similar argument exists: '--no-session-log'
```

`MUSE_EXPERIMENTAL_FOREIGN_PERSONAL_CONTEXT_KILL` 才是在 TUI 模式有效的控制項。
在操作者真實的 `~/.claude` 規則存在、且專案沒有 `AGENTS.md` 的情況下，比較其餘條件相同的執行所發出的 `context_block_diagnostic` 區塊 ID：

```
base     blocks=rules_file,workspace_identity,security_mode,skills_catalog,session_identity,subagent_delegation
killon   blocks=workspace_identity,security_mode,skills_catalog,session_identity,subagent_delegation
kill1    blocks=workspace_identity,security_mode,skills_catalog,session_identity,subagent_delegation
```

加入專案 `AGENTS.md` 後重複比較，確認關閉開關只會移除外部規則：

```
a4base   blocks=rules_file,workspace_identity,security_mode,skills_catalog,session_identity,subagent_delegation
a4kill   blocks=rules_file,workspace_identity,security_mode,skills_catalog,session_identity,subagent_delegation
```

`settings.json` 的 `tui.foreign_context_notice_shown` 旗標只會隱藏通知，不會停止載入，因此後續啟動沒有提示，不能當成上下文乾淨的證據。

<a id="composer-rendering"></a>
### 輸入區繪製

透過 `tmux capture-pane -p -e` 擷取：

```
^[[38;2;90;160;255m^[[48;2;38;56;84m⟩ ^[[38;2;204;211;219mhello from firstmate^[[39m
^[[0m^[[38;2;90;160;255m⟩ ^[[39m
```

提示字形 `⟩`（U+27E9）的亮度約為 149.9，預設幽靈文字門檻為 128；已輸入文字約為 209.8。
按一次 Escape 後，被插斷的提示會以相同約 209.8 的明亮文字還原到輸入區，`C-u` 可清除它。

<a id="the-credentialed-multi-step-smoke-verified-2026-08-06"></a>
## 需要憑證的多步驟冒煙測試（於 2026-08-06 驗證）

這是唯一延後到取得 `META_API_KEY` 才執行的項目，因為它決定已結束活動的日誌能否被分類為 `idle`。
未關閉的 run 一直都是回合執行中的正面證據，但已結束活動的日誌，只能證明當下沒有未關閉的 run；因此分類器原先將 idle 判定置於選擇啟用條件之後，以防真正的回合跨越多個 run。
以下冒煙測試回答了這個問題：一個 run 包住整個多步驟回合，Escape 插斷會以 `terminal=cancelled` 關閉該 run，不會讓回合在另一個 run 繼續。
這份需要憑證的結果，讓已結束活動的 Muse 日誌取得與 Claude、Pi 推送來源相同的閒置信任度，因此移除了選擇啟用條件，`bin/fm-busy-lib.sh` 直接將已結束活動的日誌分類為 `idle`。
Muse 會在艦隊執行期間自動更新廠商二進位檔；firstmate 會在忙碌分類前，將帶版本的程序身分正規化為 `muse` 執行環境，而工作階段日誌自己的中繼資料則帶有 semver `0.1.0` 與建置 SHA，兩者都無法與正規化後的身分比對。
針對這種粗略身分設立已驗證建置允許清單，會形成虛假的精確性，因為它無法區分正在執行的建置，也會讓維護工作疲於追逐自動更新的二進位檔。

以下兩次執行都使用預設 `meta` 提供者與模型 `muse-spark-1.2-contributor`，在真正由 firstmate 啟動的船員面板中進行，並透過 `muse auth set --provider meta --api-key-stdin` 寫入的 `~/.config/muse/auth.json` 驗證，讓金鑰完全不會進入 `argv`。

<a id="one-turn-stays-inside-one-run"></a>
### 單一回合維持在單一 run 內

單一 8 步驟工具迴圈（shell、讀取檔案、寫入檔案、shell 附加）以一次提交的回合執行，工作階段為 `629b3bc1-5dd7-4a0d-a901-69701850922c`，日誌為 `~/.local/share/muse/sessions/2026/08/06/629b3bc1-5dd7-4a0d-a901-69701850922c/session.jsonl`。
整個含 828 筆紀錄的回合，恰好由一對 run 起訖包住，期間包含 23 個工具批次：

```
$ grep -cE '"kind":"run","run_id":"[^"]*","event":\{"kind":"started"' session.jsonl
1
$ grep -cE '"kind":"run","run_id":"[^"]*","event":\{"kind":"terminal"' session.jsonl
1
$ grep -c '"payload_type":"tool_batch.effect.started"' session.jsonl
23

10  {"kind":"run","run_id":"db5869ed-...","event":{"kind":"started","prompt":"...launch-brief..."
827 {"kind":"run","run_id":"db5869ed-...","event":{"kind":"terminal","terminal":"completed",
      "reason":null,"turn_duration_ms":75243,"time_to_first_token_ms":69583,"eot_gate_ms":3907}
```

請如上將計數範圍限定為 `"kind":"run"`。
只用 `grep -c '"event":{"kind":"started"'`，同一份日誌會得到 56，因為每個工具批次效果都重用了這個內層事件結構。

<a id="busy-sampling-and-interrupt"></a>
### 忙碌取樣與插斷

工作階段 `e4e0b4f4-38d0-46dc-b669-dfb5de92e0e0` 在多步驟回合執行期間對彙整狀態取樣，接著在工具迴圈途中以 Escape 插斷。
對綁定日誌連續五次取樣 `fm_busy_muse_run_state` 都回傳 `busy`；相同取樣中的 `fm_busy_classify` 回傳 `busy muse-session-log`；插斷後彙整狀態立即結束活動。
其 run 以取消狀態關閉，沒有保持開啟：

```
10  {"kind":"run","run_id":"a098d532-...","event":{"kind":"started","prompt":"...launch-brief..."
103 {"kind":"run","run_id":"a098d532-...","event":{"kind":"terminal","terminal":"cancelled",
      "reason":"cancelled during model step","turn_duration_ms":7849}
```

這與 `echo` 提供者插斷產生的 terminal 結構相同，現在也在真實模型的工具迴圈途中獲得證實。

`tests/fm-muse-harness.test.sh` 固定驗證因此得到的分類器行為：以任一 terminal 結束活動的日誌讀為 `idle`、未關閉的 run 讀為 `busy`，只有解析失敗才讀為 `unknown`。

<a id="refreshing-this-record"></a>
## 更新這份紀錄

每次 muse 升級後都執行兩個真實環境防護測試，因為帶版本後綴的程序名稱、工作階段協定與帶樣式的輸入區，都是廠商控制的介面：

```
FM_HARNESS_LIVENESS_DRIFT=1 bin/fm-test-run.sh tests/fm-harness-liveness-drift-live-e2e.test.sh
FM_MUSE_SIGNALS_LIVE=1 bin/fm-test-run.sh tests/fm-muse-signals-live-e2e.test.sh
```

Muse 訊號防護測試需要真正的 `muse` 二進位檔與 tmux，但使用 `--provider echo`，因此不需要 `META_API_KEY`，也無法單獨重新檢查真實模型的回合對 run 關係。
防護測試會一路追蹤 SGR 狀態到最後一個提示字形，並在接受該字形的有效亮度之前，拒絕先亮後暗與格式錯誤 RGB 兩種負向對照。

muse 的啟動器可在艦隊執行期間替換執行中的二進位檔，因此改變工作階段協定的升級，也會讓上述需要憑證的證據失效。
影響協定的升級之後，請重做該冒煙測試：備妥憑證，執行一個真實的多步驟工具迴圈回合，確認限定於 run 範圍的 `started`／`terminal` 計數仍各為一，並確認 Escape 仍產生帶有 `cancelled` 的 `terminal`。
任何將單一回合拆成多個 run 的建置，都會讓已結束活動的日誌變成無法確定的狀態；這需要修改分類器，不能只在本檔案補註。

在一般 CI 中執行的對應可攜式測試為 `tests/fm-muse-harness.test.sh`、`tests/fm-tmux-agent-liveness.test.sh`、`tests/fm-composer-lib.test.sh` 與 `tests/fm-composer-ghost.test.sh`。
