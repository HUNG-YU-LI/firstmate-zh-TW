<a id="verification-the-agy-antigravity-cli-full-harness-adapter"></a>
# 驗證：agy（Antigravity CLI）完整執行環境配接器

firstmate 的 agy 配接器目前有效的實測事實。
以 [`.agents/skills/harness-adapters/SKILL.md`](../../.agents/skills/harness-adapters/SKILL.md) 為根的技能樹，透過 [`references/harness/agy.md`](../../.agents/skills/harness-adapters/references/harness/agy.md) 定義操作事實；本紀錄說明這些事實如何建立，以及哪些部分仍未證實。

<a id="subject"></a>
## 驗證對象

| 欄位 | 值 |
|---|---|
| 版本 | `agy 1.2.0`；下方的傳送確認時間於 `agy 1.2.1` 重新測量（2026-09-12） |
| 驗證日期 | 2026-09-10 |
| 二進位檔 | `/home/andpod/.local/bin/agy`，以 Go 編譯的 ELF 64 位元單一可執行檔 |
| 平台 | Linux x64（Arch，核心 7.2.3） |
| 後端 | Herdr，使用隔離且非 `default` 的實驗工作階段（透過 `bin/fm-herdr-lab.sh` 建立 `fm-lab-firstmate-agy-ad-*`）；整個過程中實際的 `default` 工作階段均未變更 |

以下每個指令都在可拋棄的 firstmate 任務工作樹，或具名的 Herdr 實驗工作階段中執行。
未觸碰船長的任何艦隊狀態。

<a id="detection-ancestry-only-no-marker"></a>
## 偵測：只依祖先程序，沒有標記

```
$ agy --version
1.2.0
```

執行中 TUI 的 `/proc/<pid>/environ` 不包含任何 `AGY_*` 或 `ANTIGRAVITY_*` 變數。
它確實帶有 `AGENT=1` 與 `CLAUDECODE=1`，但兩者都繼承自啟動環境，因此都不是 agy 身分，也不會被當成標記。
同一面板的 Herdr `pane process-info` 回報前景程序為 `name=agy`、`argv=["agy", ...]`，而 `ps -o comm=` 回報 `agy`。
因此 `bin/fm-harness.sh` 只比對完整定位的程序名稱 `agy`，spawn 則在啟動邊界清除 `CLAUDECODE`、`PI_CODING_AGENT`、`GROK_AGENT` 與 `FM_PI_HARNESS`。
`tests/fm-agy-harness.test.sh` 固定驗證完整名稱比對、拒絕只包含該片段的無關名稱，以及 spawn 清除繼承的 `CLAUDECODE` 後，它絕不會優先於真正的 `agy` 祖先程序。

<a id="launch-positional-prompt-interactive-with-auto-submit"></a>
## 啟動：位置式 prompt-interactive 並自動送出

```
$ agy --prompt-interactive "Reply with exactly AGY_LIVE_PROBE_OK and nothing else" --model gemini-3.8-flash-low --effort low --dangerously-skip-permissions
```

任務摘要不需額外 Enter 就會自行送出，回合開始執行，回覆顯示於面板。
第二次在相同目錄啟動時，也以相同方式回答新提示，因此此形式可重複，不是首次執行的偶然。
頁尾顯示 `Gemini 3.8 Flash · low`，證明兩個旗標能同時被接受。

<a id="trust-dialog-pre-registered-before-launch-gated-on-a-busy-turn-as-the-backstop"></a>
## 信任對話框：啟動前預先註冊，並以忙碌回合閘門作為後盾

在全新工作樹中第一次啟動會顯示以下對話框：

```
Accessing workspace:

/home/andpod/.treehouse/firstmate-7bab20/1/firstmate/agy-probe-tmp

Do you trust the contents of this project?

Antigravity CLI requires permission to read, edit, and execute files here.

> Yes, I trust this folder
  No, exit
```
`agy --help`（1.2.0）沒有列出信任旗標或預先註冊指令，但 agy 會遵從啟動前寫入 `~/.gemini/antigravity-cli/settings.json` 的 `trustedWorkspaces` 項目。
驗證時使用可拋棄的 `HOME`，其中放置 `~/.gemini` 的複本（從未寫入真實設定檔）：手動附加到該陣列的資料夾，執行 `--prompt-interactive` 後直接進入回合並顯示回覆，沒有對話框；未註冊的同層資料夾以相同方式啟動，則停在對話框。
agy 比較的是面板的邏輯工作目錄，而不是解析後路徑：使用符號連結的 cwd 即使註冊了真實路徑，仍會停在對話框，因此 `bin/fm-agy-trust.sh` 會在邏輯路徑與解析後形式不同時，記錄兩者。
`bin/fm-spawn.sh` 會在啟動前，於預先註冊 claude 信任的相同位置執行此輔助程式；它套用相同的結構範圍檢查（必須是啟動專案的連結工作樹，不能是主要 checkout、子目錄、一般目錄或主目錄），保留儲存區所有其他鍵，並以指紋檢查進行原子寫入。
註冊失敗會輸出 stderr 警告而非拒絕，因為 agy 對話框預先選取安全答案，且下述閘門能回答它。
兩次在 treehouse 工作樹中受監督的 Herdr 執行，在觀察時對話框仍未回答，卻已完成寫入檔案的回合（在任何 Enter 傳入該面板之前，已在磁碟確認工作 Agent 檔案與 `done:` 狀態行）。
在未信任的 `/tmp` 目錄進行隔離執行時，按 Enter 前從未進入工作區：回合改在 agy 自己的暫存目錄反覆探索工具呼叫，回答後才執行排隊中的提示。
其中一次放著數分鐘不回答的執行，最後回答後，將檔案寫到 agy 的暫存目錄而不是工作區。
尚未釐清造成差異的機制；各次執行調整了路徑、後端與延遲，但未隔離出單一原因。
因此 spawn 不依賴這個行為：預先註冊後，`bin/fm-spawn.sh` 會以 rovo／kimi 的先啟動再確認形式，執行啟動後就緒閘門（`agy_wait_for_working`）作為後盾。
它會輪詢面板擷取，第一次顯示 `Do you trust the contents of this project?` 時以單次 Enter 回答，且只有 `fm_busy_classify` 對面板回傳忙碌判定後才回報成功（Herdr 原生 `working` 狀態，或固定位置的 `esc to cancel` 狀態列）。
已知 Herdr 原生 `working` 判定可能與未回答對話框同時存在，因此閘門嚴格要求順序：只有工作樹在啟動前已預先註冊，或已看到並回答對話框後，忙碌判定才算就緒；未註冊路徑會繼續輪詢對話框，不接受提早出現的忙碌判定。
若無法在時間窗內確認摘要已執行（已回答對話框卻不轉忙碌、預先信任面板未轉忙碌，或未註冊面板從未顯示對話框），spawn 會失敗，在任務狀態記錄 `failed:`，並關閉端點，避免孤兒工作 Agent 在任務控制外繼續存活。
`tests/fm-agy-harness.test.sh` 以可拋棄儲存區涵蓋輔助程式的註冊與範圍拒絕，並驅動一個假面板，其對話框決策會讀取 spawn 剛寫入的儲存區：包含預先信任而無對話框的啟動、仍出現對話框時恰好回答一次、未註冊路徑提早出現忙碌判定時仍等待對話框，以及兩種失敗並關閉的路徑。

<a id="model-and-effort"></a>
## 模型與推理強度

```
$ agy models
Fetching available models...
gemini-3.8-flash-high	Gemini 3.8 Flash (High)
gemini-3.8-flash-medium	Gemini 3.8 Flash (Medium)
gemini-3.8-flash-low	Gemini 3.8 Flash (Low)
...
```

`agy --help` 將 `--effort` 說明為 `low|medium|high`，將 `--model` 說明為工作階段模型。
此主目錄先前設定中的裸 ID `gemini-3.8-flash` 不在清單中；只有帶 `-high`、`-medium` 與 `-low` 後綴的版本。
`bin/fm-spawn.sh` 的 `agy_model_validate`，會拒絕可連線的 `agy models` 清單未包含的指定 ID；清單無法連線時，則輸出 stderr 通知並以未驗證方式啟動。
清單來自遠端擷取（`Fetching available models...`），因此探測會中斷 stdin，並受 `bin/fm-timeout-lib.sh` 共用硬上限約束（預設 15 秒，`FM_AGY_MODELS_TIMEOUT`；非正數或非數字值會限制回預設值）；停滯擷取或登入提示會被中止，退回未驗證啟動，不會在面板尚未存在前卡住 spawn。
列印模式（`agy -p "Reply with exactly: AGY_PRINT_PROBE_OK" --model gemini-3.8-flash-low`）約 8 秒內回傳精確回覆並以 0 結束，證明不需面板也能走通憑證路徑。

<a id="busy-state-the-pinned-status-row-unknown-on-absence"></a>
## 忙碌狀態：固定狀態列，缺少時為 unknown

回合中，面板會同時顯示狀態列與轉圈提示行：

```
⣯  Generating...
└ Tip: When reviewing a file edit, press f to see the full diff.
...
esc to cancel                                                           Gemini 3.8 Flash · low
```

回合完成時顯示回覆，接著是閒置輸入區：

```
>
──────────────────────────────────────────────────────────────────────────────
? for shortcuts                                                         Gemini 3.8 Flash · low
```

`fm_busy_agy_tail_busy` 與 `bin/fm-composer-lib.sh` 的傳送防護，只比對 `esc to cancel` token：TUI 在整個回合都將此狀態列固定於面板底部，閒置時則替換為 `? for shortcuts`。
刻意不將 `Generating...` 轉圈提示字詞視為訊號：它是位置不固定的輸出行，否則一般工作 Agent 輸出如 `Generating report...`，就可能把閒置工作 Agent 誤判為忙碌，或確認其實未送達的送出。
真實觀察中，未出現缺少狀態列的忙碌階段；每張擷取的回合中畫面都帶有它。
token 缺少時，`fm_busy_classify` 回報 `unknown agy-regex`，因為長回合可能讓標記捲出擷取的尾端。
特徵寫死且沒有環境覆寫，因此無關變數絕不會改變工作 Agent 狀態分類。
Herdr 自己的登錄資訊全程一致：`agent get` 在回合中回報 `agent_status=working`，之後回報 `idle`，因此在 Herdr 上，原生判定不需新增程式碼就能提供忙碌狀態。

<a id="interrupt-and-exit"></a>
## 插斷與結束

回合中透過 `herdr pane send-keys` 傳送單次 `Escape`，會取消回合並印出以下行；輸入區回到閒置，沒有文字重新污染：

```
  ⎿  Interrupted · What should Antigravity CLI do instead?
```

傳送 `/quit` 加 Enter 會結束程序；採 `exec` 啟動時面板隨之關閉，Herdr 回報面板已不存在。
`bin/fm-control-lib.sh` 對 agy 記錄單次 `Escape`、沒有清除鍵、沒有確認來源，以及 `/quit`。

<a id="backend-liveness-herdr-recognizes-agy-tmux-names-it"></a>
## 後端存活：Herdr 原生辨識 agy，tmux 依名稱辨識

```
$ herdr agent get w2:p1 --session fm-lab-firstmate-agy-ad-1599574-8823
{"result":{"agent":{"agent":"agy","agent_status":"idle",...,"agent_session":{"agent":"agy","kind":"id","source":"herdr:antigravity_cli",...}}}}
```

Herdr 原生追蹤 agy（`antigravity-cli` 整合，偵測為 `agent=agy`），因此 `fm_backend_herdr_pane_agent_state` 對每個已登錄 agy 狀態回傳 `live`，不需要加強結束偵測。
tmux 配接器透過 `bin/fm-agent-process-lib.sh` 的共用名稱詞彙，將完整定位的程序名稱 `agy` 分類為 `agent`，沿用 muse／omp 對短裸名稱的先例。
agy 已加入 `bin/fm-session-lock-lib.sh` 的工作階段鎖名稱詞彙（`FM_HARNESS_RE` 與 `FM_HARNESS_NAMES`），將 agy 祖先程序辨識為已驗證的工作階段鎖持有者。

<a id="composer-unknown-by-design"></a>
## 輸入區：刻意判為 unknown

閒置面板的位元組層級擷取，顯示兩條全寬 `─` 分隔線之間有一個不帶樣式的裸 `>`，其下狀態列包含不帶樣式的 `? for shortcuts` 欄位，以及暗色（`SGR 2`）模型欄位。
共用分類器依失效 shell 規則，將該裸 `>` 讀為 `unknown`，絕不視為 `empty`。
引導仍可確認傳送：Herdr 送出核心優先使用原生 `idle` 到 `working` 轉換，agy 會進行該轉換；傳送頁尾 regex 則涵蓋 tmux 路徑。
對該確認迴圈而言，agy 的忙碌頁尾出現偏晚：在 `agy 1.2.1`（2026-09-12）真實測得，短引導約在 Enter 後 1.5 秒，較接近實際使用的長摘要則在 4–5 秒後才出現，相對於共用預算只有 3 × 0.4 秒；因此 `bin/fm-send.sh` 給 agy 鍵入目標較長的預設送出確認預算（20 次重試，預設節奏約 8 秒）。明確的 `FM_SEND_RETRIES` 仍優先，其他所有執行環境則維持共用的 3 次重試預設值。
`tests/fm-send-agy-confirm.test.sh` 固定驗證提高後的預設值，`tests/fm-agy-harness.test.sh` 則固定驗證 Herdr 轉換路徑。
這沿用 cursor 先例，不是需要在共用程式碼修補的缺口。

<a id="supervised-task-spawn-steer-relaunch-and-exit-through-the-new-path"></a>
## 受監督任務：透過新路徑啟動、引導、重新啟動與結束

一個簡單 scout 透過 `bin/fm-spawn.sh --harness agy` 在同一隔離實驗工作階段完成端到端執行：`spawned agy-e2e1 harness=agy kind=scout`，任務中繼資料記錄了由 treehouse 提供的工作樹、`--model gemini-3.8-flash-low` 與 `--effort low`。
工作 Agent 寫入工作樹檔案，並將 `done: agy e2e turn complete` 附加到工作樹外的狀態檔，證明提示處理、工具執行、工作區外檔案存取與新的完成事件皆有效。
持久引導有效：`bin/fm-send.sh` 訊息進入任務收件匣，工作 Agent 將引導要求的行附加到兩個檔案，收件匣紀錄也移入 `handled/`。
同一複本重新啟動有效：`bin/fm-control.sh relaunch --note` 在相同工作樹、模型與推理強度上原地替換工作 Agent；替代者確認先前兩行完整保留，並附加 `relaunched: done`。
結束有效：`bin/fm-control.sh exit` 停止工作 Agent，登錄回傳 `agent_not_found`，面板只留下位於工作樹的 shell，所有工作皆完整保留。
未測試或宣稱自動配額故障轉移；以上每次交接都是明確受監督的重新啟動。

<a id="primary-and-secondmate-integration"></a>
## 主要 Agent 與二副整合

主要 Agent 與持續存在的二副執行，透過原生 `.agents/hooks.json` 整合完成驗證。
工作階段開啟時的摘要注入，由 `bin/fm-sessionstart-agy.sh` 在 `SessionStart` 事件實作。
PreToolUse 安全帶透過 `bin/fm-arm-pretool-check.sh --agy`、`bin/fm-cd-pretool-check.sh --agy` 與 `bin/fm-subagent-pretool-check.sh --agy` 強制執行安全規則。
在主要 Agent 與二副主目錄，監看器指令由共用啟動監看指令政策分類，`cd` 指令會被拒絕並要求改用絕對路徑或 `cwd`，子 Agent 呼叫（`invoke_subagent`、`define_subagent`）則被拒絕，但檢查工具（`manage_subagents`、`manage_task`）仍允許使用。
監督連續性透過 `bin/fm-turnend-guard-agy.sh` 的同步 Stop hook 停駐維持：執行 `bin/fm-watch-arm.sh`（或監督主機），在 28800 秒 hook 逾時限制下等待傳入事件喚醒（只有設定選用的 `FM_AGY_PARK_TIMEOUT` 時，才有額外等待上限），並透過 `bin/fm-turnend-guard.sh --agy` 退回有界的修復提醒。
工作階段鎖驗證透過 `bin/fm-session-lock-lib.sh` 辨識 `agy` 程序祖先。

<a id="what-is-still-unproven"></a>
## 仍未證實的部分

尚未觀察到未驗證的失敗模式；此主機的 agy 已登入，因此任何驗證提示都視為需明確報錯的憑證阻礙，不是已處理的對話框。
未驗證斜線技能呼叫形式，因此技能仍以自然語言呼叫。
未測試 `--continue` 與 `--conversation` 接續；復原使用磁碟摘要進行確定性的重新啟動。

<a id="refreshing-this-record"></a>
## 更新這份紀錄

agy 每次升級後，都執行可攜式套件與真實環境防護測試，因為程序名稱、標記集合、信任對話框文字，以及顯示的忙碌／插斷文字，都是廠商控制且會被 spawn 閘門與忙碌備援逐字比對的介面：

```
bin/fm-test-run.sh tests/fm-agy-harness.test.sh
FM_AGY_SIGNALS_LIVE=1 bin/fm-test-run.sh tests/fm-agy-signals-live-e2e.test.sh
```
