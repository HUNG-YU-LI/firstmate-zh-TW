# Cursor Agent

使用 Cursor Agent CLI 2026.08.11-e8db854，於 2026-08-11 在 tmux、2026-08-12 在 Herdr 驗證船員與偵察工作，並於 2026-08-13 驗證二副與主 Agent 工作。
跨執行環境的供應商與憑證身分由 `references/common/model-and-effort.md` 負責。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 執行檔 | `../../../bin/fm-cursor-lib.sh` 的 `fm_cursor_resolve_binary` 會解析穩定啟動器 `cursor-agent` 或舊版 `agent`，絕不使用 `cursor`；每個候選都必須通過該模組負責的限時啟動探測才會被接受（透過 `fm_run_timed`，所以不需要 coreutils `timeout`），讓偏好名稱故障時能繼續嘗試下一個。這兩個名稱通常都是連到 `~/.local/share/cursor-agent/versions/<version>/cursor-agent` 的符號連結，而自動更新會取代其目標。 |
| 啟動 | 清除其他主 Agent 標記後，以位置參數傳入指示，搭配 `--trust`、`--yolo`、選用的 `--model <model>` 與 `--workspace <absolute-task-worktree>`。 |
| 署名 | Cursor 可能在輸入的訊息之後附加 Co-authored-by 尾註。除非 home 設定 `config/keep-ai-trailers`（`../../../../../docs/configuration.md` 的「提交署名」），否則每次艦隊啟動都會安裝 `../../../bin/fm-git-strip-ai-trailers.sh` 中以窗格為範圍的 commit-msg 移除器，移除已知 AI 尾註，並保留人類共同作者與作者身分。 |
| 模型 | 使用目前帳號的 `cursor-agent --list-models` 或舊版 `agent --list-models`；持續變動的觀察清單中，Grok 只有 `cursor-grok-4.5-high` 與 `cursor-grok-4.5-high-fast`，另有數個 `xhigh` ID，因此應選擇實際回傳的推理 ID，絕不假設有 low 或 medium Grok。 |
| 忙碌狀態 | `../../../bin/fm-busy-lib.sh` 以 `cursor-transcript` 摺合各對話逐字紀錄：`role:user` 開啟忙碌，具型別的 `turn_ended` 在成功或中止時關閉，也涵蓋手動中斷；不會預先啟用或植入任何狀態，此來源與後端無關，在 tmux 與 Herdr 上相同。 |
| 結束命令 | `/exit`。 |
| 中斷 | 按一次 Escape 即返回預留文字，不需清除按鍵；控制層不保證取消已完成，因為部分測試中止後的逐字紀錄關閉事件會在數秒內出現，另一些則等了二十秒仍未出現。 |
| 技能呼叫 | `/<skill>`，例如 `/no-mistakes`；Cursor 會探索 Firstmate 的使用者技能。 |
| 恢復 | 沒有已驗證的原生窗格恢復方式；使用可確定重現的重新啟動。 |
| 自主性 | `--yolo`，文件記載它是 `--force` 的別名；頁尾為 `Run Everything`。 |
| 信任 | `--trust` 會抑制對話框；`--yolo` 不會，而且每項任務都有新的路徑。 |
| 識別標記 | Agent 與子程序上有 `CURSOR_INVOKED_AS=cursor-agent`，子程序或工具程序另有 `CURSOR_AGENT=1`；其他 `CURSOR_*` 變數不是身分標記。 |
| 推理強度 | 沒有已驗證旗標；不支援值的處理由 `references/common/model-and-effort.md` 負責。 |
| 輸入區 | 無邊框的單列，使用 `→`（U+2192）；淡化的預留文字在新工作階段為 `Plan, search, build anything`，之後為 `Add a follow-up`。 |

斜線彈出選單會吃掉第一次 Enter；該次 Enter 只會關閉選單，共用重試會以真正的第二次 Enter 送出。

<a id="detection"></a>
## 偵測

Cursor 不會清除繼承的 `CLAUDECODE`，因此 Claude 底下的 Cursor 工作 Agent 同時帶有兩種標記。
`../../../bin/fm-harness.sh` 會先測試 Cursor，啟動時也會清除其他執行環境的標記。
兩者都仍然必要：清理涵蓋 Firstmate 啟動，判定順序則涵蓋手動啟動的工作階段。
該順序只決定標記層；較近的 Claude 祖先程序仍優先於保留下來的 Cursor 標記。

Cursor 是打包後的 Node 指令碼，因此 tmux 可能只回報 `node`，但 `ps -o comm=` 會帶有安裝路徑。
單獨的 `node` 不符合任何身分；`../../../bin/fm-cursor-lib.sh` 會從路徑或 argv zero 中的 Cursor 名稱或安裝樹證明身分。
無關的 `node` 或 `agent` 保持為 `other`，摺合後為不明確，不會判為死亡。
自動更新改變的是目標，不是此規則。

<a id="composer-and-delivery"></a>
## 輸入區與送達

Cursor 會把終端機游標停在輸入區外：閒置與已輸入文字時，`#{cursor_y}` 都在頁尾下方，且 `#{cursor_flag}` 為零，因此依游標錨定的讀取一律未知。
`../../../bin/fm-tmux-lib.sh` 只有在結構上證明是 Cursor 後，才採用最下方的形狀。
合成判定接著可讀為空或待送出，已於 2026-08-13 驗證；其他執行環境仍維持嚴格的空白游標行為，已結束的 shell 也絕不會讀成空輸入區。
因此 `../../../bin/fm-supervise-daemon.sh` 可以在離開模式送達前要求確定為空，不需要 Cursor 專屬分支。

送出也會使用從閒置到忙碌的轉換。
比對穩定字串 `ctrl+c to stop`，不要比對旋轉指示器的動詞；它在不同回合間曾從 `Working` 改成 `Running`。

確認只在 tmux 與 Herdr 上驗證過。
Herdr 在所有狀態都將 Cursor 回報為 `blocked`，因此其原生閒置路徑無法到達；輸入區路徑則將 `ctrl+c to stop` 旁的回合中預留文字判為待送出。
`../../../bin/backends/herdr.sh` 會在 Enter 前建立基準，並確認頁尾轉換，因此原本就忙碌的窗格無法形成確認。

Zellij、cmux、Orca 與 tern 不會檢查此頁尾。
透過輸入平面的原生呼叫或明確後端傳送，文字會送達，但會回報未確認並以非零狀態結束；一般引導使用持久化收件匣，在入列時即以零結束。
應將此視為確認失敗，不是遺失，因為文字確實送達，忙碌狀態也來自逐字紀錄。
讓這些後端理解該頁尾屬於另一項跨執行環境工作，需要實際驗證。

反白預留文字殘留與 Herdr 半格邊緣由 `../../../bin/fm-composer-lib.sh` 負責；沒有這些邊緣時，無框線輸入區會吞入頁尾，讓閒置狀態讀成待送出。
擷取畫面由 `../../../docs/verification/runtime-backends.md` 負責。
使用 `FM_HARNESS_LIVENESS_DRIFT=1 ../../../bin/fm-test-run.sh ../../../tests/fm-harness-liveness-drift-live-e2e.test.sh` 重新驗證。

<a id="worktree-boundary"></a>
## Worktree 邊界

Firstmate 會進入已取得的 worktree，並透過 `--workspace` 傳入同一個絕對路徑。
絕不可傳入 Cursor 的 `-w` 或 `--worktree`；它會在 `~/.cursor/worktrees` 下再配置一份副本，破壞隔離。
CLI 支援可重複的 `--add-dir`，但配接器不會新增任何項目；位置參數指示不需要對其私有目錄授權。
範例：`../../../bin/fm-spawn.sh <task-id> <project> --scout --harness cursor --model cursor-grok-4.5-high`。

<a id="primary-integration"></a>
## 主 Agent 整合

主 Agent 監督使用 `../../../docs/supervision-protocols/cursor.md` 中的 stop-hook 停駐機制，透過受版本控制的 `.cursor/hooks.json` 實作；主 Agent 與二副啟動必須帶 `--trust`，否則 hooks 不會載入。
若 home 有 `config/supervision-host` 且沒有 `config/supervision-host-off`，停駐會執行監督主機，取代 `../../../bin/fm-watch-arm.sh`，並以 Claude 的列印模式作為無介面引擎；主機由 [`supervision-host.md`](../../../../../docs/supervision-host.md) 負責。
Cursor 提供 20 種專案事件，以及會載入 `.claude/settings.json` 的 Claude-Code 相容性對應。
受版本控制的 hooks 透過 `$CURSOR_PROJECT_DIR` 註冊 `stop`、`sessionStart` 與兩項 `preToolUse` 安全帶；依 `../../../docs/turnend-guard.md`，Claude 項目收到 Cursor 承載資料時會退讓。

`stop` 無法阻擋，因為結束碼 2 會被靜默忽略，因此 `../../../bin/fm-turnend-guard-cursor.sh` 會停駐監督，並回傳一則有界限的 `followup_message`。
它不會在無介面的 `cursor-agent -p` 中觸發。
`preCompact` 未註冊，因為它無法注入上下文，因此 Cursor 壓縮後重新送出摘要的工作仍待處理。
