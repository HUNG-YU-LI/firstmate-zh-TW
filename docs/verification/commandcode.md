<a id="command-code-worker-verification"></a>
# Command Code 工作 Agent 驗證

讀者：維護者驗證。

於 2026-10-05 在 Linux x86_64（`Linux 7.2.5-3-omarchy`）驗證，使用 AUR `command-code` 套件的 Command Code `1.74.1`、Node `v26.8.1`、tmux `3.7c` 及 Herdr `0.9.3`。
[轉接器參考文件](../../.agents/skills/harness-adapters/references/harness/commandcode.md) 負責操作事實；負責執行的程式則承載啟動與狀態機制。
本驗證涵蓋使用模型 `deepseek/deepseek-v4.1-flash` 的船員（crewmate）與偵察員（scout），以 tmux 作為受防護的執行期後端，並使用具名 Herdr 實驗工作階段驗證 Herdr 上相同的生命週期。
主要 Agent、二副、配額供應商，以及型別化派送整合，不在此保證範圍內。

<a id="refresh-commands"></a>
## 更新驗證的指令

```sh
commandcode --version
commandcode --help
commandcode status
commandcode --list-models
bin/fm-test-run.sh tests/fm-commandcode-harness.test.sh tests/fm-tmux-agent-liveness.test.sh
FM_COMMANDCODE_SIGNALS_LIVE=1 bin/fm-test-run.sh tests/fm-commandcode-signals-live-e2e.test.sh
```

使用憑證的驗證防護，會將 `~/.commandcode/auth.json` 複製到隔離主目錄，使用私人 tmux socket，並執行 `fm-spawn.sh` 實際產生的指令。
該主目錄帶有使用者 `SessionStart` hook，它必須與每程序的 busy mod 一起持續運作；工作 Agent 自己建立的提交也不得帶有 Command Code 署名。
worktree 配置與初始端點投遞使用可攜式測試夾具；後續引導、中斷、結束及重新啟動，則使用真正後端、真正供應商程序與真正控制層。
未登入或沒有可隔離的檔案憑證時會略過；工具不存在及選擇性啟用的行為，由共用真實測試關卡負責。
失敗訊息會指出已安裝的 Command Code 版本。

<a id="live-guard-results"></a>
## 真實驗證防護結果

2026-10-05，驗證防護在 `deepseek/deepseek-v4.1-flash`、`--effort low` 下執行 208 秒，以 0 結束：

```text
ok - commandcode 1.74.1: spawn brief, model, effort, autonomy, trust, identity, and the busy mod
ok - commandcode 1.74.1: the worker commit carries no attribution
ok - commandcode 1.74.1: idle composer reads empty; real fm-send doorbell read and acknowledged
ok - commandcode 1.74.1: idle interrupt opens nothing; an open rewind picker blocks exit and is closed without restoring
ok - commandcode 1.74.1: one Escape cancels, preserves the agent, and the mod records idle
ok - commandcode 1.74.1: fm-control exit via /quit
ok - commandcode 1.74.1: native --resume <session-id>
ok - commandcode 1.74.1: fm-control relaunch re-arms the mod with a fresh generation
```

<a id="observed-vendor-surfaces"></a>
## 觀察到的供應商介面

`/usr/bin/commandcode`、`cmd`、`cmdc` 與 `command-code` 都是包裝腳本，會 exec `/usr/lib/node_modules/command-code/dist/index.mjs`；執行中的程序會設定自己的標題，因此 `ps -o comm=` 回報 `command-code`，改寫後的 argv 也是同一個字。
`commandcode status` 回報 `Authenticated as <account>`，供應商為 `Command Code`；`commandcode --list-models` 列出的 85 個模型中包含 `deepseek/deepseek-v4.1-flash` 與 `deepseek/deepseek-v4.1-flash-fast`。
兩個 DeepSeek V4.1 Flash 模型對 `--effort bogus` 都回覆 `Unknown effort "bogus". Supported: off, low, high, max.`，並以狀態 1 結束；`medium` 與 `xhigh` 也以相同方式遭拒，而 `low` 與 `max` 則印出 `Reasoning effort set to ... for DeepSeek V4.1 Flash.`。
未知模型會以 `Error: unknown model "deepseek/no-such-model".` 結束。
在全新儲存庫中，`--yolo --trust --skip-onboarding` 啟動時不顯示信任或權限提示，頁尾顯示 `» permission bypass on`。
放在 `--` 之後的訊息會啟動第一個回合；放在 `--resume <id>` 旁的訊息則被忽略，恢復的工作階段停在空白輸入區等待。

在暫存 `HOME` 下執行 `--config theme=light`，會將 `{"theme": "light"}` 寫入 `~/.commandcode/config.json`；`--config attribution.commit=` 則以 `Unknown config setting: attribution.commit` 結束，因此 `--config` 不是每程序限定的機制。
Hook 設定只提供 `PreToolUse`、`PostToolUse`、`Stop` 與 `SessionStart`。
透過 `--mod` 與 `--mod-option` 傳入的 mod，只會在該程序載入；單次回合依序記錄 `run_start`、`onRunEnd end_turn` 與 `run_end "end_turn"`；Escape 中斷記錄 `interrupted` 與 `run_end "interrupted"`；`/quit` 記錄 `onSessionEnd shutdown` 與 `session_shutdown`；`--resume` 則記錄 `onSessionStart resume`。
使用者的 `SessionStart` hook 與 mod 一同觸發，執行後使用者 `settings.json` 的位元組完全不變。
每次啟動時，CLI 都會在專案建立空白 `.commandcode/taste/taste.md` 用於偏好學習，相關開關只能透過設定檔控制。

閒置輸入區那一列，是預設前景色的 `❯`、反白字元格 `A`，接著是以 `38;2;138;148;168` 顯示的 `sk your question...`，上下夾著同色的兩條線；鍵入文字採預設前景色，後面接一個反白游標字元格。
未設定 `COLORTERM` 時，占位文字變成 `38;5;145`；設定 `NO_COLOR=1` 時，游標字元格不再反白，但真彩色仍保留。
終端機游標位於頁尾下方的空白列。
執行中的回合會呈現一列狀態，例如 ` ○ Contemplificating…  esc to interrupt • 2s • ↓ 41`，之後是 ` ⌘ Choreographing…  esc to interrupt • 1m 30s • ↓ 1.4k`；保留樣式的擷取結果中，`esc` 與 `to interrupt` 之間有跳脫序列。
對執行中的回合按一次 Escape，會印出 `❯ Interrupted · What should Command Code do instead?`，並留下空白輸入區。
Agent 閒置時，兩次 Escape 間隔 0.1 或 0.3 秒會開啟 `Rewind` 選擇器（`Select a checkpoint to restore your session`，頁尾 `Press Enter to select · Esc to cancel`）；間隔 0.6 秒或以上則不會；無間隔連續送出兩次會被解析為單一按鍵，不開啟任何內容；按一次 Escape 可關閉選擇器。
`/quit` 會返回 shell，並印出 `Resume this session with:`，後面接 `cmd --resume <session-id>`。
斜線指令彈出選單將 `~/.agents/skills` 的 `/no-mistakes` 列為 `[skill]`。
模型產生的工作 Agent 提交帶有 `Co-authored-by: CommandCodeBot <noreply@commandcode.ai>`，如今會由船隊的 commit-msg 清除機制移除。

<a id="herdr-lab-session"></a>
## Herdr 實驗工作階段

一次真正的 `fm-spawn.sh` Command Code 啟動，在透過 `bin/fm-herdr-lab.sh` 建立的具名 Herdr 0.9.3 實驗工作階段窗格中執行；任務紀錄指向該窗格，並將船長真正的使用者 `settings.json`（含 Herdr 自己的 Command Code `SessionStart` hook）複製到隔離主目錄，得到：

```text
agent get: {"agent":"cmd","status":"idle"}
fm_backend_agent_state: alive
fm_backend_composer_state: empty
fm-send steer: steer=1147, inbox acknowledged
interrupt (idle):  interrupt-delivered ... backend=herdr verified=agent-alive cancel=unconfirmed
raw Esc pair 0.2 s apart: Rewind picker open; exit refused; interrupt closed it
interrupt (busy):  interrupt-delivered ... backend=herdr verified=agent-alive cancel=unconfirmed; busy record idle commandcode-mod; Interrupted rendered
exit:      stopped ... backend=herdr; agent state dead
relaunch:  relaunched ... harness=commandcode backend=herdr; relaunched=91; busy record idle commandcode-mod on the new generation
```

Herdr 依自己的偵測將窗格命名為 `cmd`；共用輸入區分類器接受這個身分，也接受 tmux 探測的 `commandcode`。
實驗環境 teardown 的船隊狀態防誤觸檢查通過，`default` 工作階段維持不變。

<a id="coverage-and-limits"></a>
## 覆蓋範圍與限制

可攜式回歸測試涵蓋程序祖先證據、拒絕無關程序名稱、單次按鍵中斷與回溯選擇器判定表、在有及沒有 Command Code 身分時讀取擷取的輸入區畫面並斷言差異、讓鍵入文字維持 pending、分別檢查兩個投遞訊號、驗證 busy mod 生命週期並拒絕過期世代與未設定選項，以及驗證僅限工作 Agent 的啟動、每程序 mod 選項、只接受已驗證 effort 及排除 taste 狀態。
`tests/fm-tmux-agent-liveness.test.sh` 透過真正 tmux 程序證明，前景 `command-code` 會被判為 alive；即使無法取得游標列，仍能在該程序身分的約束下判為 `empty` 或 `pending`。
真實驗證防護檢查啟動簡報、模型、effort、身分、busy mod、使用者 hook 共存、設定不可變性、提交署名、通知訊號確認、閒置與忙碌中斷、回溯選擇器、結束、原生恢復及重新啟動。
Zellij、Orca 與 cmux 只透過既有的後端無關投遞及按鍵介面檢視，未在此進行真實測試；它們的輸入區讀取不帶 Command Code 身分，因此未證明引導訊息能送達。
Orca 既有的不支援 Escape 投遞限制，代表 Command Code 中斷在該處會被拒絕。
未觀察 macOS 的程序命名。
