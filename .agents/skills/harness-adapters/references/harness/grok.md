# Grok Build

xAI 的 `grok` TUI 與 Claude Code 相容。
最初於 2026-06-29 使用 0.2.73 驗證，2026-07-03 使用 0.2.82 驗證斜線命令送出，2026-07-13 使用 0.2.99 驗證推理強度，2026-07-19 使用 0.2.103 驗證結束，並於 2026-09-24 使用 1.0.41 驗證資料夾信任、訓練選擇加入，以及輸入區未送出的訊息送達。
啟動形式：`grok --always-approve "$(cat <brief>)"`。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 忙碌狀態 | 最後保留的畫面尾端備援，僅限 Grok，在有語意來源前暫用：回合中會出現 ASCII `Ctrl+c:cancel`，閒置列 `Shift+Tab:mode │ Ctrl+.:shortcuts` 則沒有；絕不使用容易受語系影響的點字旋轉指示器。 |
| 結束 | `/exit` 會印出 `Resume this session with: grok --resume <session-id>`；備援方式是在 1000ms 內按兩次 `Ctrl+Q`，VS Code 系列終端機用 `Ctrl+D` 離開，而 `Ctrl+C` 用來中斷。 |
| 中斷 | 按一次 `Ctrl+C`；Escape 只會將焦點移到回捲內容。 |
| 技能 | `/<skill>`，例如 `/no-mistakes`，已有使用者技能探索、呼叫，以及實際 `no-mistakes axi run` 的端到端證據；彈出選單可能吃掉 Enter 並填入引數預留文字，需要真正的第二次 Enter。 |
| 自主性 | `--always-approve`，頁尾為 `· always-approve`，已驗證可無人值守；`--permission-mode bypassPermissions` 是更強的等效選項。 |
| 識別標記 | 0.2.73 的子程序或工具程序帶有 `GROK_AGENT=1`，且沒有 `CLAUDECODE`；1.0.0 的 hook 則有 `GROK_HOOK_EVENT`、`GROK_HOOK_NAME`、`GROK_SESSION_ID` 與 `GROK_WORKSPACE_ROOT`，沒有 `GROK_AGENT`，因此由祖先程序確保身分。 |
| 恢復 | `grok --resume <session-id>`，或用 `grok -c` / `--continue` 恢復 cwd 最新工作階段；`--fork-session` 會建立新 ID。 |
| 模型 | `--model <model>`；用 `grok models` 探索目前帳號的模型。 |
| 推理強度 | `--reasoning-effort <low\|medium\|high>`，別名 `--effort`；0.2.99 會以 `use one of: high, medium, low` 拒絕 `xhigh` 與 `max`；備援與不支援值的處理由 `references/common/model-and-effort.md` 負責。 |

可靠的 Grok 規則必須同時考量 hook 標記與子程序快速路徑。
標記契約由 `../../../docs/turnend-guard.md` 的「執行環境整合」負責。

<a id="submission-and-startup"></a>
## 送出與啟動

斜線自動完成可能把第一次 Enter 轉成選取並顯示引數提示，而沒有送出，包括 `/no-mistakes` 的選用 task 引數或 `/compact compaction instructions`。
共用分類器會將該文字保留為待送出，重試則會在兩個已驗證後端送出第二次 Enter；Herdr 也可能透過原生狀態證明回合開始。

2026-07-03，兩個 Grok 0.2.82 Herdr 工作 Agent 在傳送回報成功後，仍讓 `/no-mistakes` 留在輸入區數分鐘。
舊版 Herdr 邏輯將任何窗格差異都視為送出，包括彈出選單關閉與預留文字填入。
Tmux 與 Herdr 現在都會將擷取內容送至 `../../../bin/fm-composer-lib.sh`，分類每一個已證明內容列上的實際文字。
邊界由 `../../../docs/herdr-backend.md` 負責，並由 `../../../tests/fm-backend-herdr.test.sh` 涵蓋。

2026-09-24，Grok 加入此艦隊後的第一批派工中，一則引導訊息進入 Grok 1.0.41 輸入區卻未送出。
窗格顯示 `Enter:send now`，文字持續待送出。
對此執行環境任何時間敏感的訊息，都應查看窗格確認送達，不要只信任傳送結果。
暫停指示悄悄未送達，是最不能遺失的訊息。

「Run Grok Build in a project directory?」選擇器只會在專案外出現，例如 home、Desktop、Downloads 或 `/tmp`。
spawn 會從隔離的 git 根目錄啟動，因此不會出現此選擇器，也不需要按鍵。
若無法避免在非專案位置啟動，可在 `~/.grok/config.toml` 設定 `[hints] project_picker_disabled = true` 抑制選擇器。
專案選擇器與資料夾信任關卡是不同的對話框。
2026-09-24，同一批初次派工中，Grok 1.0.41 在連結的 git worktree 顯示資料夾信任關卡。
對話框印出主 checkout 路徑，因為連結 worktree 的 git 根目錄會解析至主要副本；即使隔離完整，該文字看起來仍完全像是違反 worktree 隔離。
應使用 `/proc/<pid>/cwd` 檢查工作 Agent 的實際位置，絕不以對話框印出的路徑為準。
透過按鍵路徑的 Enter 回答關卡（`../../../bin/fm-send.sh <target> --key Enter`）。
`../../../bin/fm-send.sh` 只支援 Escape、Enter 與 C-c，沒有核准的途徑傳入字面 `y`。
2026-09-24 的 Grok 1.0.41 會將回答持久化到 `~/.grok/trusted_folders.toml`，以對話框顯示的路徑為鍵。
這正是 `../../../bin/fm-spawn.sh` 刻意不寫入，並稱為影響範圍很大的寫入的同一份儲存區。
因此在連結 worktree 中，信任會落在主 checkout，而非可拋棄副本，並持續套用到之後在該處的每次 Grok 執行。
這會默默啟用該 checkout 的 Grok 專案 hooks。
儘管如此，回答關卡仍是核准的途徑，因為 `../../../bin/fm-send.sh` 沒有其他方式清除它。
這是明知下採用、對避免寫入儲存區立場的例外，不是疏漏，因此預期該檔案會出現新項目。

<a id="training-opt-in"></a>
## 訓練選擇加入

2026-09-24，Grok 加入此艦隊後的第一批派工中，Grok 1.0.41 顯示「Help improve Grok」。
選擇加入會保留提示、追蹤紀錄與指標供訓練使用。
預設關閉，必須維持關閉。
此艦隊撰寫面向客戶的隱私聲明，表示客戶資料與音訊不會用於訓練；在發布這種聲明的同時，把我們自己的提示與追蹤紀錄送給供應商訓練，不能默默作出這種取捨。

<a id="composer"></a>
## 輸入區

新的預留文字 `Type a message...` 使用深色 24 位元 TRUECOLOR，而非 SGR-2。
`../../../bin/fm-composer-lib.sh` 的 `fm_composer_strip_ghost` 會移除 dim、faint，以及亮度低於 `FM_COMPOSER_GHOST_LUMA_MAX`（預設 128）的真彩色文字。
在 Grok 0.2.93，實際輸入 `38;2;224;222;244` 的亮度約 225，邊框與預留文字則從 `38;2;50;47;70` 到 `38;2;110;106;134`，亮度約 51–110，因此會被移除。
真彩色規則假設艦隊使用深色主題；SGR-2 則不受主題影響。
測試涵蓋於 `../../../tests/fm-composer-ghost.test.sh` 與 `../../../tests/fm-backend-herdr.test.sh`。

Tmux `#{cursor_y}` 可能指向全新輸入區的底框。
共用分類器會定位完整方框與所有內容列，因此邊框上的游標與多列輸入區都不需要配接器位移量。

<a id="worker-turn-end-hook"></a>
## 工作 Agent 回合結束 hook

Grok 每回合都會觸發 `Stop`。
專案 hooks 需要 `~/.grok/trusted_folders.toml` 中的資料夾信任；spawn 不會編輯該檔，但回答上面的資料夾信任關卡會寫入；全域 `~/.grok/hooks/` 則一律受信任。
spawn 會安裝帶防護的全域 `fm-turn-end.json` 與 `fm-turn-end.sh`。
只有工作區 `.fm-grok-turnend` 與 `~/.grok/hooks/fm-turn-end.d/` 下的登錄資料相符時，它們才會動作，接著透過始終設定、且等於 worktree 的 `GROK_WORKSPACE_ROOT`，touch 該任務的 `state/<id>.turn-ended`。
此機制位於 worktree 外，不需信任授權，且只寫入 Firstmate 檔案。
`../../../bin/fm-teardown.sh` 會在回收到池之前，移除被 gitignore 忽略的指標檔。
二副會略過此機制，因為閒置是健康狀態，一般過期窗格偵測不適用。

<a id="primary-integration"></a>
## 主 Agent 整合

於 2026-07-28 使用 0.2.112 與真正尚未支援原生功能的 0.2.73 驗證。
`.grok/hooks/fm-primary-turnend-guard.json` 會呼叫 `../../../bin/fm-turnend-guard-grok.sh`。
實際執行中的 Stop 承載資料，在 0.2.112 會選擇同程序接續；0.2.73 沒有該能力，需要一次帶防護的 `grok --resume`。
自適應與格式錯誤輸入的行為由 `../../../docs/turnend-guard.md` 負責。

Grok 也會載入 Claude 專案設定，因此在 `GROK_AGENT` 或 `GROK_HOOK_EVENT` 下，Claude 中對應 Grok 已涵蓋事件的項目會退讓；該負責文件記錄精確集合，以及排除 `GROK_SESSION_ID` 的原因。
專案本機 hooks 需要啟動時帶 `--trust`；沒有它時，防護會退讓，`../../../bin/fm-guard.sh` 會成為下一個命令的警報。
監看器監督仍採圍繞 `../../../bin/fm-watch-arm.sh` 的受追蹤背景通知，而非 Pi 式擴充擁有權。
若 home 有 `config/supervision-host` 且沒有 `config/supervision-host-off`，工作階段啟動區塊會將該背景呼叫呈現為 `../../../bin/fm-supervision-host.sh park`，並以 Claude 列印模式作為無介面引擎；主機由 [`supervision-host.md`](../../../../../docs/supervision-host.md) 負責。
PreToolUse 會直接阻擋，但 hook 命令中的每個 `$VAR` 都必須內嵌 `:-default`，否則 Grok 會拒絕該 hook。
