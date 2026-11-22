<a id="omp-oh-my-pi"></a>
# omp（Oh My Pi）

於 2026-09-05 在 Herdr 上使用 omp 18.1.11 驗證船員、偵察、二副與主 Agent 工作，並以 2026-09-02 針對 18.1.2 的配接器調查為基礎。
omp 是 Pi 的 fork，因此 `references/harness/pi.md` 是最接近的參考；這裡列出與 Pi 的每一項差異。
跨執行環境的供應商與憑證身分由 `references/common/model-and-effort.md` 負責。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 執行檔 | `omp`，由 Bun 編譯成單一執行檔，`../../../bin/fm-spawn.sh` 從 `PATH` 解析；找不到執行檔就拒絕建立工作 Agent。 |
| 啟動 | 清除其他執行環境標記（`CLAUDECODE`、`PI_CODING_AGENT`、`GROK_AGENT`、`FM_PI_HARNESS`、`GEMINI_CLI`、Cursor 的標記），設定 `FM_OMP_HARNESS=omp OMP_SKIP_SETUP=1`，接著執行 `omp [--profile <name>] --config <.omp/fm-worker-overlay.yml> --auto-approve --cwd <worktree> [--model] [--thinking] -e state/<id>.omp-ext.ts <one positional brief>`；二副會傳入同一個 `-e` 以載入忙碌擴充，主 Agent 擴充仍依賴自動探索。只有依[個別二副 omp 設定檔](../../../../../docs/configuration.md#per-secondmate-omp-profile)固定設定的本機二副，才會出現 `--profile`。 |
| 忙碌狀態 | `../../../bin/fm-busy-lib.sh` 的來源 `omp-ext`：個別任務擴充在 `agent_start` 標記忙碌，並只在 `willContinue` 不是 true 時，於 `agent_end` 標記閒置；刻意不檢查 `ctx.isIdle()`，因為 TUI 自然結束的 `agent_end` 當下它讀為 false（穩定前會等待 `session_stop`）。 |
| 結束命令 | `/quit`（`/exit` 與 `/q` 為別名）。 |
| 中斷 | 按一次 Escape；輸入區保持空白，不需清除按鍵。 |
| 技能呼叫 | 除了一般命令行為外，沒有另外驗證過的形式；不確定精確命令時使用自然語言。 |
| 模型旗標 | `--model <provider>/<id>`（omp 接受模糊模式，但會略過 Firstmate 啟動前檢查）。 |
| 推理強度旗標 | `--thinking <off\|minimal\|low\|medium\|high\|xhigh\|max\|auto>`，是共用詞彙的超集合，因此包括 `max` 在內的每個層級都直接對應。 |
| 模型探索 | `omp models [--json]` 只列出內建與自動探索的供應商；由擴充註冊的供應商（例如 `claude-bridge`）永遠不會出現，因此 spawn 會直接傳入這些模型，不加驗證，並在 stderr 提示。`omp usage` 顯示供應商配額時段；使用 bridge 時，`quota-axi` 涵蓋 `claude` 供應商。 |
| 識別標記 | omp 本身沒有（已驗證：執行檔內沒有 `PI_CODING_AGENT`，預設設定檔也沒有 `PI_CODING_AGENT_DIR` 或 `OMP_PROFILE`）。`FM_OMP_HARNESS=omp` 是 Firstmate 的啟動標記；祖先比對使用精確程序名稱 `omp`，或由 bun 執行該套件的 `omp` 啟動器。 |
| 輸入區 | 覆寫層固定為 `composer.shape: borderless`，是共用分類器已能讀取的無框線 `❯`（U+276F）單列；忙碌文字是 `Working…`（U+2026），這是 omp 忙碌正規表示式唯一接受的拼法（無介面 `-p` 模式寫出的三個點版本不會進入受監督窗格），第二個訊號則是狀態列的點字旋轉指示器加上經過時間欄位。 |
| 自主性 | 核准由 `--auto-approve` 負責（omp 在此旗標下，會強制該工作階段使用 `tools.approvalMode: yolo`）；覆寫層固定 `plan.defaultOnStartup: false`、`prewalk.enabled: false`、`retry.usageReservePolicy: auto`。 |
| 信任 | 完全沒有專案信任關卡；全新的設定檔會顯示供應商登入精靈，以 `OMP_SKIP_SETUP=1` 抑制。 |
| 恢復 | 有 `-c/--continue` 與 `-r/--resume`，但沒有已驗證的窗格恢復契約；使用可確定重現的重新啟動。 |

指示必須維持為單一位置參數；第二個位置參數從未顯示為已送出的訊息。
openai-codex 模型透過 omp 的 `xd://` 虛擬檔案橋接存取擴充註冊的工具：模型會讀取 `xd://fm_watch_arm_omp` 取得說明，並寫入 `xd://fm_watch_arm_omp` 呼叫工具，因此逐字紀錄或 rpc 串流顯示的是對該路徑的 `write`，而不是直接呼叫 `fm_watch_arm_omp`；兩者是同一次呼叫（已於 18.1.11 驗證）。
omp 冷啟動到第一個 Agent 回合約需二十秒，每個工作 Agent 只付出一次成本。

<a id="detection"></a>
## 偵測

`../../../bin/fm-harness.sh` 與 Cursor 標記的處理相同，會先於 `CLAUDECODE` 測試 `FM_OMP_HARNESS=omp`；祖先鏈巡覽則在直譯器備援判定之前，使用 `../../../bin/fm-omp-process-lib.sh` 的明確錨定身分（精確程序名稱 `omp`，或 `bun` 的下一個引數為名稱恰為 `omp` 的指令碼）。
`../../../bin/fm-spawn.sh` 中的 omp 範本會在自己的啟動邊界清除全部其他標記，而 `FM_OMP_HARNESS=omp` 只有在真正的 `omp` 祖先程序之下才有效，因此被其他啟動繼承的標記沒有作用：omp 二副的工作 Agent 會保留自己的身分，繼承的 `CLAUDECODE` 也無法優先於 omp 啟動的工作 Agent。
`../../../bin/fm-session-lock-lib.sh` 對工作階段鎖定擁有權使用同一個 `fm_omp_process_matches` 身分；`../../../bin/backends/tmux.sh` 的存活判定則只將精確名稱 `omp` 分類為 `agent`。
選用的 claude-bridge 擴充會執行一個名稱就是 `claude` 的巢狀執行檔，它是工具執行的同層程序，絕不是祖先，因此 omp 自己的工具呼叫仍偵測為 omp；Firstmate 指令碼不會巡覽該子樹。

<a id="worker-posture-overlay"></a>
## 工作 Agent 行為覆寫層

絕不寫入船長自己的 `~/.omp/agent/config.yml`；受版本控制的 `.omp/fm-worker-overlay.yml` 會透過 `--config` 傳入單一工作階段，只固定那些若沿用船長層級值，就可能讓無人值守工作 Agent 停在提示、改變固定模型，或讓輸入區無法讀取的設定。
`../../../bin/fm-spawn.sh` 的檔頭負責精確清單及每項固定設定的原因。

<a id="extension-loading"></a>
## 擴充載入

omp 會自動探索 `<cwd>/.omp/extensions/*.ts`（只看頂層、只看 cwd、不向上巡覽祖先、不顯示信任對話框）與作用中設定檔的 `agent/extensions/`；`.pi/extensions/` 不是探索根目錄。
同時被自動探索並以 `-e` 指定的檔案會載入兩次，因此個別任務的忙碌擴充放在 `state/`，也是二副啟動唯一指定的 `-e`。
主 Agent 擴充維持自動探索，不重複指定。
沒有 `agent_settled` 事件；由 `agent_end` 加 `willContinue` 取代。

<a id="primary-integration"></a>
## 主 Agent 整合

omp 主 Agent 透過 `../../../docs/supervision-protocols/omp.md` 採用 Pi 的擴充負責監看器模型：`.omp/extensions/fm-primary-omp-watch.ts` 透過 `fm_watch_arm_omp` 工具啟用 `bin/fm-watch-arm.sh --restart`，並負責每個後繼監看器；`.omp/extensions/fm-primary-turnend-guard.ts` 則回應 omp 的阻擋式 `session_stop` hook，當 `../../../bin/fm-turnend-guard.sh` 回傳 2 時強制接續一次，且每回合受 omp 的 `stop_hook_active` 旗標限制。
同一檔案也移植 `tool_call` 安全帶，並在 Run 層級透過 `before_agent_start` 送達工作階段啟動摘要；omp 的 `session_start` 不帶原因，因此會推導來源（第一次啟動從啟動命令列判定為 `startup` 或 `resume`，之後程序內啟動為 `clear`，`session_compact` 則為 `compact`）。
omp 沒有非同步 Stop-hook 的對等機制，因此不適用 Claude 自動啟用模型；`fm_supervision_model` 將 omp 分類為 `extension`，而 `../../../bin/fm-wake-lib.sh` 的 `fm_omp_extension_owns_supervision` 是允許擴充自行交接監看器的擁有權證明。
Pi 的監督分支不會在 omp 上執行；沒有監督主機時，每個可採取行動的喚醒都送到 main；若 home 有 `config/supervision-host` 且沒有 `config/supervision-host-off`，監看擴充就會啟動主機取代 arm，並以 Claude 列印模式作為無介面引擎（[`supervision-host.md`](../../../../../docs/supervision-host.md)）。
在 home 內直接用 `omp` 啟動主 Agent（若從 Claude 窗格啟動則使用 `FM_OMP_HARNESS=omp omp`）；執行中的工作階段未載入兩個受版本控制的擴充時，`../../../bin/fm-session-start.sh` 會印出 `OMP_WATCH_EXTENSION: not loaded`。
`FM_OMP_LIVE_E2E=1 ../../../tests/fm-omp-primary-live-e2e.test.sh` 是需自行啟用的即時防護檢查；`../../../tests/fm-omp-harness.test.sh` 是可攜式回歸測試。
