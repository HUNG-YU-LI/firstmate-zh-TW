# OpenCode

於 2026-10-01 使用 OpenCode 2.0.21 驗證，先前已驗證 1.15.7 至 1.18.4；輸入區固定介面元素於 2026-09-18 使用 1.18.31 再次驗證，模式頁尾固定元素則於 2026-10-14 使用 1.19.0 再次驗證。
2026-10-15，具有雙重匯出的主 Agent 外掛在 1.15.7、1.18.20 與 1.18.31 都各載入一次，回合結束防護也只觸發一次；監看器連續性，以及 1.x 工作 Agent 啟動與結束，也在 1.18.20 與 1.18.31 通過，因此外掛不需要 1.18.29 的物件進入點（證據在 `docs/verification/runtime-backends.md`）。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 忙碌狀態 | `bin/fm-spawn.sh` 每次建立 opencode 工作 Agent 都會配置忙碌狀態外掛，但形式依偵測到的主版本決定。若偵測到主版本 >= 2，就寫入 OpenCode 2 外掛，將 `session.execution.started` 對應為忙碌，將 `session.execution.succeeded`/`interrupted`/`failed` 對應為閒置，並鎖定工作 Agent 自己的工作階段，同時保留 OpenCode 1 的 `session.status` busy/retry/idle 名稱作為舊版輸入。若為 1.x，或版本未偵測到／無法解析而回退至 1.x，就寫入移植前的外掛，對應 `session.status` busy/retry/idle 與 `session.idle`，讓 OpenCode 1 完整保留移植 OpenCode 2 前的忙碌狀態接線。 |
| 結束命令 | 依版本而異，以 `bin/fm-spawn.sh` 偵測並在任務中繼資料記為 `opencode_version` 的主版本為準。2.x 使用原始按鍵，由 `../../../bin/fm-control.sh` 透過按鍵平面送達，而非輸入送出路徑：先 Ctrl-U 清空輸入區，再 Ctrl-C，若 Agent 仍存活就再按一次 Ctrl-C。輸入區為空時，一次 Ctrl-C 就讓窗格回到 shell；若仍有輸入文字，Ctrl-C 只會清除它（已於 2.0.11、2.0.21、2.0.22 驗證；證據在 `docs/verification/runtime-backends.md`）。輸入 `/exit` 也會結束，但會開啟斜線命令完成彈出選單，因此按鍵平面的 Ctrl-C 可避免該選單干擾送出驗證。1.x（或未記錄版本／舊版任務）使用輸入式 `/exit` 命令，兩個版本線都能結束。已知限制：寬窗格且未連接供應商時，OpenCode 2.0.22 會在輸入區同一列繪製 `/connect` 側邊欄，因此 `fm-control.sh` 的結束操作會安全地以待送出文字為由拒絕；讓輸入區讀取忽略該側邊欄是後續工作。 |
| 啟動與任務簡報送達 | 依偵測到的主版本而異。2.x 的 `bin/fm-spawn.sh` 會啟動 `opencode --standalone --prompt "<brief>"`：2.x 預設啟動會附接到共用背景服務，該服務用自己的環境執行外掛，並忽略窗格的 `OPENCODE_CONFIG_CONTENT`。OpenCode 2.0.11 會讓 `--prompt` 簡報留在輸入區未送出，2.0.22 則會送出，因此 2.x 啟動後，spawn 會等待工作 Agent 忙碌狀態外掛回報回合開始，只有尚未開始且輸入區讀到待送出簡報時才送一次 Enter（若輸入區無法讀取就繼續等待，直到較晚的截止時間才送 Enter），若回合始終沒有開始就判定 spawn 失敗。1.x 或未偵測到版本時，啟動不帶 `--standalone`，也沒有送達關卡。 |
| 中斷 | 按兩次 Escape；已知長時間 shell 命令執行中可能不穩定，因此窗格卡住時使用 `../../../bin/fm-control.sh <task-id> relaunch`。 |
| 技能呼叫 | 除一般斜線命令行為外，沒有另外驗證過的形式；不確定精確命令時使用自然語言。 |
| 恢復 | 用 `--continue` 重新啟動以恢復目前目錄最近的工作階段，再等 TUI 就緒後傳入下一項指示，因為 `--prompt` 與 `--continue` 一起使用時不會自動送出。 |
| 模型旗標 | 依偵測到的主版本而異。OpenCode 2.0.21 沒有互動式頂層 `--model`（`opencode run --model` 只留在非互動路徑），因此 Firstmate 透過 `OPENCODE_CONFIG_CONTENT` 的全域 `model` 加 `agent.build.model` 固定互動模型。OpenCode 1.x 仍接受互動式 `--model` 旗標，因此 Firstmate 在該版傳入 `--model`，不寫入 2.x 全域/build 模型固定值。未偵測到或無法解析版本（無法解析 opencode，或 `--version` 沒有可解析的 x.y.z）時回退至 1.x 形式，所以只有解析後主版本 >= 2 才使用 2.x 固定方式。 |
| 推理強度旗標 | Firstmate 的互動式 `opencode --prompt` 啟動沒有此旗標；`opencode run` 有模型變體語法，但不是這條路徑。推理強度改由啟動的 `OPENCODE_CONFIG_CONTENT` JSON 傳遞，作為 `build` Agent 的 `variant`，依解析後模型決定。只有已知該模型供應商將此推理強度公開為變體時才輸出（`anthropic/*`：high、max；`openai/*`：low、medium、high、xhigh）；未解析模型、其他供應商，或推理強度不在該系列清單內時，省略 variant。在 2.x 即使省略 variant，全域/build 模型固定值仍保留；在模型透過 `--model` 傳入的 1.x，省略 variant 後只剩權限設定。 |
| 模型探索 | 執行 `opencode models` 列出可用供應商／模型識別字；OpenCode 2.0.21 已不接受供應商位置參數。 |
| 信任對話框 | 無。 |
| 識別標記 | 無；OpenCode 不發布身分標記，因此 `../../../bin/fm-harness.sh` 從程序祖先鏈識別。 |

OpenCode 可能在背景自動升級，執行中的 TUI 可能在任務中途結束。
從 1.15.7 升級至 1.17.3 時，已實際觀察到此行為。
若窗格顯示結束橫幅，使用上面已驗證的恢復途徑。

自 OpenCode 1.18.0 起（上游 anomalyco/opencode#36457，2026-07-11 合併），只要窗格有作用中工作階段且沒有更高優先序的頁尾提示，閒置輸入區就會在 Build 頁尾上方的輸入區列中，靠右繪製窗格工作目錄與 git 分支；內容最多會換行為三段欄軌寬度的片段，最後一段延伸到頁尾列右端。
這個 cwd:branch 指示是輸入區固定介面元素，絕不是輸入文字；`../../../../../bin/fm-composer-lib.sh` 的共用分類器（`_fm_composer_leftbar_cwd_start`）會依位置與形狀辨識它，支援 `$HOME` 下的 cwd（`~/...`）或外部的絕對 cwd，讓閒置與忙碌窗格都仍讀為已證明的 `empty`。
早期讀取將這些片段視為未送出的輸入，導致每個閒置 1.18.x tmux 工作 Agent 都無法接收引導（在 1.18.31 實際觀察到），直到手動輸入門鈴才恢復。
OpenCode 1.19.0 也會將模式／模型／分支頁尾列淡化繪製，其模式字詞與中點分隔符低於 `fm_composer_strip_ghost` 的亮度門檻，因此移除幽靈文字後的讀取看不到這些固定元素，分類器改從去除 ANSI 的列形式證明它；註明日期的擷取、安全使用純文字形式的理由，以及回歸測試指標都在 [runtime-backends.md](../../../../../docs/verification/runtime-backends.md)。
沒有游標錨點時，此版本的完整窗格擷取仍會保守延後判為 `unknown`，因為輸入區底列下方的狀態欄軌是連續的非空白內容；這種延後是嚴格策略，不是誤判。

<a id="busy-queued-enter"></a>
## 忙碌時排隊的 Enter

OpenCode 2.0.21 在回合進行中時，輸入區會把 Enter 當成「回合結束後送出」的按鍵，但直到回合完成才清除已輸入文字。
若沒有轉換規則，每次透過輸入平面傳送到忙碌 OpenCode 窗格都會誤報「Enter 被吃掉」，而主 Agent 回合中收到 daemon 升級訊息時也會看起來像卡住。

Tmux 與 Herdr 都將此例外交給 `../../../bin/fm-composer-lib.sh` 中唯一的 `fm_composer_queued_enter_verdict` 政策。
各後端專屬訊號記錄於 `../../../docs/tmux-backend.md` 與 `../../../docs/herdr-backend.md`。
回歸測試涵蓋 `../../../tests/fm-tmux-submit-busy.test.sh`、`../../../tests/fm-composer-lib.test.sh` 與 `../../../tests/fm-backend-herdr.test.sh`。
Herdr 即時防護檢查為 `FM_HERDR_SUBMIT_CONFIRM_LIVE=1 ../../../tests/fm-herdr-submit-confirm-live-e2e.test.sh`。

<a id="primary-integration"></a>
## 主 Agent 整合

主 Agent 整合於 2026-10-01 使用 OpenCode 2.0.21 驗證。
OpenCode 2 要求本機外掛匯出帶有 `id` 與 `setup` 函式的預設定義；Firstmate 的主 Agent 外掛從 `setup(ctx)` 安裝實際行為。
`.opencode/plugins/fm-primary-turnend-guard.js` 透過 `ctx.event.subscribe` 訂閱終止事件 `session.execution.succeeded`、`session.execution.failed` 與 `session.execution.interrupted`，當 `../../../bin/fm-turnend-guard.sh` 回傳 2 時，以 `ctx.session.prompt` 排入後續提示。
此移植也保留舊版 OpenCode 1 的 `server` 匯出，以支援較舊配接器。
每個主 Agent 外掛都同時帶有兩種匯出，而非在載入時設版本關卡，因此 OpenCode 1 執行 `server` 匯出，OpenCode 2 執行 `setup`。
啟動 OpenCode 2 主 Agent 或二副 home 時，使用 `opencode --standalone`，讓外掛在 home 自己的程序中，使用其 `FM_HOME` 與工作階段鎖定祖先鏈執行，而不是在共用背景服務；`bin/fm-spawn.sh` 對 `--secondmate` 啟動已這樣做。
若 home 有 `config/supervision-host` 且沒有 `config/supervision-host-off`，watch-arm 外掛會啟動監督主機，取代 `../../../bin/fm-watch-arm.sh`，並以 Claude 列印模式作為無介面引擎；主機由 [`supervision-host.md`](../../../../../docs/supervision-host.md) 負責。
`opencode run` 可能在顯示排隊後續提示前就結束，因此配接器在無介面模式會退讓。
原生 Windows 上，操作輸入配接器透過 `bash` 執行 Bash 輔助程式；macOS 與 Linux 則直接呼叫。

配套的 `.opencode/plugins/fm-primary-watch-arm.js` 負責一般 TUI 監看器監督，以 `ctx.session.prompt` 喚醒，並在無觀察回合後續提示之前與防護協調。
相當於 PreToolUse 的監看器啟用安全帶，透過從 `ctx.tool.hook("execute.before")` 擲出錯誤來阻擋。
Herdr 後端不需要 OpenCode 2 專用 Firstmate 外掛：`bin/backends/herdr.sh` 仍負責窗格生命週期與輸入區處理，而 Herdr 已安裝的 `herdr.opencode` 與 `herdr.opencode.session-selection` 外掛，在 2.0.21 與 Firstmate 外掛一同載入。
