<a id="watcher-arm-pretooluse-seatbelt"></a>
# 監看器啟動 PreToolUse 安全帶

本文件是監看器啟動 PreToolUse 安全帶的權威人類可讀契約。
`bin/fm-arm-command-policy.mjs` 是唯一的語意負責者。
`bin/fm-arm-pretool-check.sh` 只負責穩定的執行環境傳輸與輸出呈現。
受版本控制的執行環境配接器只轉送指令文字，不自行分類。
`bin/fm-arm-command-policy.mjs` 也是 firstmate shell 分類的唯一負責者：它匯出詞彙分析器與指令位置分析，供同系列的 cd-guard 安全帶（`bin/fm-cd-pretool-check.sh`、`docs/cd-guard.md`）重用，不重複實作 shell 詞彙分析。

<a id="purpose-and-boundary"></a>
## 目的與邊界

firstmate 主要 Agent 必須透過可觀察的執行環境呼叫，啟動 `bin/fm-watch-arm.sh` 或執行 `bin/fm-watch-checkpoint.sh`。
shell 背景運算子、管線、重新導向、包裝程式或不相關指令串列，可能隱藏失敗，或讓監看器子程序隨工具呼叫一起死亡。
安全帶會在執行前拒絕這些指令形式。

此政策不是啟動後的存活保證。
在允許的呼叫之後，`bin/fm-guard.sh` 與 `bin/fm-turnend-guard.sh` 會分別對監看器鎖與信標套用各自的啟動後監督判斷。

分類器絕不執行、載入、評估或展開提交指令的任何部分。
它只對位元組進行詞彙分析，並分類詞彙層級的執行位置。

<a id="transport-and-fail-open-behavior"></a>
## 傳輸與失敗放行行為

`bin/fm-arm-pretool-check.sh` 支援以下入口形式：

- Claude 與 Codex 使用 stdin JSON 的 `.tool_input.command`。
- Grok 使用 stdin JSON 的 `.toolInput.command`。
- OpenCode、Pi、pi-signed 與 omp 使用 `--command <exact string>`。
- `--background` 是僅供相容的欄位，絕不改變決策。
- `--claude` 保留 Claude 只允許 stderr 拒絕輸出的要求。
- Antigravity CLI 使用 `--agy`，從 stdin JSON 的 `.toolCall.args.CommandLine`、`.toolCall.args.command`、`.toolInput.command` 或 `.tool_input.command` 讀取；監看器正常時拒絕多餘的手動啟動監看，並以 exit 0 向 stdout 輸出 `{"decision":"deny","reason":...}`。

包裝程式從自己的位置探索程式碼根目錄。
目前的 firstmate 主目錄是 `${FM_HOME:-<code-root>}`。
它將兩個根目錄與精確指令字串傳給 Node 政策負責者。

只有確定指令即使經過分類器解碼器，也不可能包含 `fm-watch` 位元組序列時，包裝程式才會不呼叫 Node 政策負責者而快速允許。
快速路徑只有在以下兩項同時成立時才能允許：

1. 模擬分類器成本最低的位元組正規化，移除續行與跳脫反斜線、引號與換行後，剝除後文字不含 `fm-watch` 監看器子字串。
2. 原始指令不含引號解碼標記：`$` 後緊接單引號（ANSI-C `$'...'`）或雙引號（bash locale `$"..."`）。

任何 `fm-watch` 比對或引號解碼標記，都會交給分類器。
先正規化可維持嚴格超集合：混淆為 `fm-watc\<newline>h-arm.sh` 或 `fm-"watch"-arm.sh` 的受保護監看器路徑，仍會交由分類器處理；只移除這些非英數位元組，也絕不會破壞既有的 `fm-watch` 連續序列。
引號解碼標記補上單純位元組剝除無法處理的案例：`bin/fm-$'\x77'atch-arm.sh` 與 `bin/fm-$"watch"-arm.sh`，都只有在分類器解碼編碼字元後，才會解析為 `bin/fm-watch-arm.sh`；否則低成本位元組剝除會看不到 `fm-watch` 而快速放行。
此標記集合與 `bin/fm-arm-command-policy.mjs` 的分類器解碼集合相互綁定：新增任何分類器可解碼的引號或展開形式，都必須在同一次變更擴充此標記集合，否則預篩選就不再是嚴格超集合。
預篩選沒有語意例外的決定權：它只能快速允許確定不是監看器指令的內容，因此絕不反轉分類，分類器仍是每個決策的唯一負責者。

安全帶的威脅模型是 Agent 失誤：沒有人會不小心寫出 ANSI-C 或 locale 混淆的監看器路徑，刻意混淆屬於啟動後存活防護的責任。
標記防護仍補上靜態缺口，因為成本低，且可依編碼類別證明。
警戒條件：若在此標記一般化後再發現第三個嚴格超集合缺口，就推翻「可依編碼類別證明」的主張，決策改為選項 B：移除預篩選，一律呼叫分類器。
超出相互綁定標記集合、需要更深層解碼的混淆，仍由分類器與啟動後存活防護負責。

stdin 格式錯誤或空白、JSON 無效、stdin 傳輸缺少 `jq`、缺少 Node、缺少分類器，或分類器回應無效時，都以 exit 0、無輸出方式失敗放行。
這種傳輸行為可避免損壞的 hook 拒絕所有 shell 工具呼叫。
含受保護指令的格式錯誤或不支援 shell 語法，則是語意分類結果，採取拒絕策略。

<a id="command-position-classification"></a>
## 指令位置分類

詞彙分析器可辨識帶引號來源資訊的解碼後字詞、註解、heredoc 本文、shell 串列運算子、管線、重新導向、指令與程序替換、括號子 shell、大括號群組，以及字面巢狀執行酬載。
引號文字、註解、heredoc 本文與後續引數字詞，除非被已辨識的執行接收端遞迴執行，否則都是資料位置。

執行位置中的指令字，若其正規化路徑後綴符合下列受保護監看器指令碼之一，就屬於受保護執行：

```text
bin/fm-watch-arm.sh          (arm; blessed entry point)
bin/fm-watch-checkpoint.sh   (checkpoint; blessed entry point)
bin/fm-watch.sh              (watch; protected but never blessed)
```

相對形式、以 `<code-root>` 定位的絕對形式，以及任何以 `/bin/<script>` 結尾的字詞，都解析為該身分。
後綴比對會靜態辨識展開路徑的前綴，因此 `$FM_HOME/bin/fm-watch-arm.sh`、`$HOME/firstmate/bin/fm-watch-arm.sh` 與 `~/firstmate/bin/fm-watch-arm.sh` 都屬於 arm 身分。
分類器絕不展開變數或波浪號，只比對字面位元組。
靜態引號形式會在後綴比對前解碼，因此被一般引號（`fm-"watch"-arm.sh`）、ANSI-C 引號（`fm-$'\x77'atch-arm.sh`）或 bash locale 字串（`fm-$"watch"-arm.sh`）切開的指令字，都解析為相同身分；這只是以 shell 解碼方式讀取固定字面位元組，絕不執行展開或指令。
涵蓋範圍是指令位置中靜態可見的字面字詞；`bash -lc "$WHOLE_COMMAND"` 這類不透明動態資料流仍不在範圍內。

`bin/fm-watch.sh` 受保護，但不是核准入口。
直接執行 `bin/fm-watch.sh`，不論相對、`<code-root>` 定位、`$VAR` 前綴或 `~` 前綴，都一律以 `watcher-direct` 拒絕，原因會引導呼叫端改用 `bin/fm-watch-arm.sh` 與 `bin/fm-watch-checkpoint.sh`。

相同位元組若出現在引數、註解、斷言、文件查詢、Python 字串、`printf` 或 `tmux send-keys` 酬載中，則是資料，不會讓外層指令變成相關操作。

字面的 `sh`、`bash` 或 `zsh` `-c` 酬載，以及字面 `eval` 酬載，會遞迴分類。
只執行帶資料指令的字面巢狀酬載會被允許。
執行受保護指令的字面巢狀酬載，會以 `watcher-nested` 拒絕，即使該內層受保護呼叫放在頂層本可允許也一樣。
餵給從 stdin 讀取程式之 shell 的 heredoc 或字面 here-string，也以相同方式分類。
使用 `-s` 時，後續運算元設定的是位置參數，不是指令碼名稱，因此 `bash -s sentinel <<< 'bin/fm-watch.sh'` 會遭拒。
`-s --` 後的運算元仍是位置參數，但第一個運算元位置中的受保護監看器路徑仍會遭拒。

`bash -lc "$WATCHER_COMMAND"` 之類動態酬載，無法靜態證明，仍由啟動後防護負責。
若提交指令先建構受保護的字面指定值，再將動態值餵給已辨識的 shell 或 `eval` 接收端，分類器會保守地以 `watcher-nested` 拒絕。

註解與 heredoc 本文不視為執行語法。
真正帶 heredoc 的受保護指令仍具有重新導向，因此會遭拒。

<a id="blessed-syntax-tree"></a>
## 核准語法樹

允許的監看器程式，是單一線性外層指令串列，包含零個或多個核准設定節點，最後恰好接一個直接受保護節點。
`bin/fm-watch-arm.sh` 與 `bin/fm-watch-checkpoint.sh` 是唯二核准的最終節點，包含其展開路徑形式；最終節點若為 `bin/fm-watch.sh` 絕不核准，會以 `watcher-direct` 拒絕。

核准的設定節點包括：

- `cd <one path word>`。
- `export NAME=<one shell word>`，不得含指令替換、程序替換或重新導向。
- `source <x-mode path>` 或 `. <x-mode path>`。
- `[ -f <x-mode path> ] && source <x-mode path>`，以及等效的點號形式。

允許的 x-mode 路徑是 `config/x-mode.env`、`./config/x-mode.env`，以及正規化後等於 `<active-firstmate-home>/config/x-mode.env` 的絕對路徑。
目前主目錄之外的絕對 x-mode 路徑，不屬於核准設定節點。

核准節點可由 `;`、真正的換行或 `&&` 分隔。
設定之後接受 `&&`，讓失敗的 `cd`、`export` 或 source 阻止受保護呼叫在錯誤設定下執行。

最終受保護節點可直接外包一層 `exec`。
其引數是一般 shell 字詞，可包含帶引號的分號或監看器名稱。
不核准其他包裝程式。

行內環境指定值、`env`、`sudo`、`nohup`、巢狀 shell、`eval`、子 shell 群組、替換、重新導向、管線、非同步串列、`disown`、不相關串列節點，以及不支援的複合語法，都不核准。

<a id="broad-watcher-kills"></a>
## 廣泛終止監看器

實際執行的 `pkill`，若解析後的模式引數指向 `fm-watch`，就會遭拒。
可辨識帶路徑的 `pkill`、`command pkill` 與 `sudo pkill`。

`kill "$(pgrep -f '/bin/fm-watch.sh')"` 也會遭拒，因為實際執行的 `kill` 使用了實際執行、涵蓋所有監看器的 `pgrep` 替換。
單獨的唯讀 `pgrep` 會被允許。
`echo 'pkill -f fm-watch'` 這類引號文字是資料，會被允許。

不支援的複合語法——迴圈、`case`、`if` 或分類器未建模的其他結構——對廣泛終止採取拒絕策略，與受保護執行相同。
當指令帶有這種語法，且原始位元組同時參照 `fm-watch` 目標與 `pkill` 或 `kill` 動詞時，分類器無法證明 kill 位於哪個指令位置，因此以 `broad-watcher-kill` 拒絕，不予允許。
這個後盾對照受保護執行的拒絕規則，涵蓋 `while true; do pkill -f fm-watch; done`、`for x in 1; do pkill -f fm-watch; done`、`case x in x) pkill -f fm-watch ;; esac` 與 `until false; do kill $(pgrep -f fm-watch); done` 等形式。
只有語法不受支援時才啟用此規則：對分類器已建模的語法，指令位置分析是權威來源，因此資料提及如 `echo 'pkill -f fm-watch'`，以及只提及監看器卻沒有 kill 動詞的迴圈如 `for f in 1; do echo fm-watch; done`，仍會被允許。

<a id="stable-reason-codes"></a>
## 穩定原因代碼

每個語意拒絕都會在文字原因之前，以方括號包含一個穩定代碼。

| 代碼 | 意義 |
| --- | --- |
| `watcher-background` | 受保護執行位於非同步串列中，或使用 `nohup` 或 `disown`。 |
| `watcher-pipeline` | 受保護執行參與任何管線。 |
| `watcher-redirection` | 受保護執行使用 shell 重新導向。 |
| `watcher-bundled` | 外層指令串列不是核准的「設定加最終節點」語法樹。 |
| `watcher-nested` | 包裝程式、群組、替換、巢狀 shell、`eval` 或建構的動態酬載執行受保護指令。 |
| `broad-watcher-kill` | 實際的廣泛程序終止指向監看器。 |
| `unclassifiable-protected-command` | 格式錯誤或不支援的語法含受保護指令，無法安全分類。 |
| `watcher-direct` | 直接執行 `bin/fm-watch.sh`；必須透過 `bin/fm-watch-arm.sh` 或 `bin/fm-watch-checkpoint.sh` 進入監看器。 |

原因代碼是測試與配接器的穩定契約。
文字說明可改善，不影響配接器行為。

<a id="output-contract"></a>
## 輸出契約

- 允許回傳 exit 0，兩個串流均為空。
- 拒絕回傳 exit 2，並向 stderr 寫入 `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny"},"systemMessage":"[code] reason"}`。
- 預設拒絕模式也會向 stdout 寫入 `{"decision":"deny","reason":"[code] reason"}`，供 Grok 使用。
- `--claude` 完全抑制 stdout，因為 stdout 非空時，Claude 會忽略 PreToolUse 拒絕。
- Codex 在 exit 2 時阻擋，並顯示 stderr。
- OpenCode 只有檢查器以 2 結束時才拋出錯誤。
- Pi、pi-signed 與 omp 只有檢查器以 2 結束時才回傳 `{block: true}`。

<a id="harness-wiring"></a>
## 執行環境接線

| 執行環境 | 精確指令欄位 | 檢查器 exit 2 時的配接器行為 |
| --- | --- | --- |
| Codex | `.tool_input.command` | `.codex/hooks.json` 指令轉送完整 stdin 酬載，Codex 在 exit 2 時阻擋。 |
| Claude | `.tool_input.command` | `.claude/settings.json` 以 `--claude` 轉送 stdin，保留 stdout 為空並回傳 stderr 拒絕物件。 |
| Grok | `.toolInput.command` | `.grok/hooks/fm-primary-pretool-check.json` 轉送 stdin，Grok 使用 stdout 的 `decision=deny` 物件。 |
| OpenCode | 1.x `bash` 工具為 `output.args.command`；2.x `shell` 工具的 `execute.before` 事件為 `event.input.command` | `.opencode/plugins/fm-primary-pretool-check.js` 傳入單一 `--command` 引數，只在 exit 2 時拋出錯誤。 |
| Pi / pi-signed | `event.input.command` | `.pi/extensions/fm-primary-turnend-guard.ts` 傳入單一 `--command` 引數，只在 exit 2 時回傳 `{block: true}`。 |
| omp | `event.input.command` | `.omp/extensions/fm-primary-turnend-guard.ts` 傳入單一 `--command` 引數，只在 exit 2 時回傳 `{block: true, reason}`；omp 將原因原樣呈現給模型（已驗證 18.1.2）。 |
| Cursor | `.tool_input.command` | `.cursor/hooks.json` 比對 `tool_name` `Shell`，以 `--cursor` 轉送 stdin。Cursor 讀取回傳物件而非結束狀態，因此 `--cursor` 向 stdout 印出 `{"permission":"deny","user_message":"[code] reason"}` 並以 0 結束；只有這種呈現方式已驗證能阻擋指令並顯示原因。 |

Cursor 也會載入 `<project>/.claude/settings.json`，因此受版本控制的 Claude 入口會收到相同事件。沒有 `--cursor` 時，Cursor 傳來的酬載屬於重複項，會直接允許、不重新分類；`bin/fm-hook-host-lib.sh` 依酬載自己的 `cursor_version` 決定；[`turnend-guard.md`](turnend-guard.md#harness-integrations) 說明為何此判斷讀取酬載而非環境。

Grok 專案 hook 需要資料夾信任。
Cursor 專案 hook 需要以 `--trust` 啟動工作區。
Grok hook 指令中的每個 shell 變數參照，都必須帶有 `${GROK_WORKSPACE_ROOT:-}` 這類行內預設值，因為 Grok 會在 `bash -lc` 執行前先展開原始 hook 指令。
因此受版本控制的 Grok 配接器直接參照 `${GROK_WORKSPACE_ROOT:-}`，不先指定再讀取 shell 區域 `$root` 變數。

<a id="live-validation-record-2026-07-09"></a>
## 真實驗證紀錄，2026-07-09

驗證在此任務工作樹下，已初始化 git 的暫存 firstmate 結構專案中進行。
暫存專案包含修改後檢查器與政策的複本、未變更的受版本控制配接器、假 checkpoint、假 arm 指令碼、無害的 `tmux` 引數擷取測試替身，以及私人哨兵路徑。
沒有將修改檔案安裝到主要 checkout 或真實執行環境設定。
未使用真實監看器、艦隊狀態或 herdr 生命週期指令。
OpenCode 互動檢查使用專用 tmux socket `fm-pretool-smoke`。

執行環境版本如下：

```text
Claude Code 2.1.206
codex-cli 0.144.0
grok 0.2.93 (f00f96316d4b)
OpenCode 1.17.15
Pi 0.80.5
```

每個執行環境都被要求以獨立工具呼叫，發出以下精確 shell 指令字串：

```sh
printf 'UNRELATED_EXECUTED\n'
pgrep -fl '/bin/fm-watch.sh' || true
source '<scratch-project>/config/x-mode.env'; bin/fm-watch-checkpoint.sh --seconds 180
tmux send-keys -t isolated-pi-lab "printf '%s\n' 'bin/fm-watch-arm.sh &'"; tmux send-keys -t isolated-pi-lab Enter
bin/fm-watch-arm.sh &
```

真正的執行環境啟動指令如下：

```sh
claude -p "$PROMPT" --dangerously-skip-permissions --output-format text
codex exec --dangerously-bypass-hook-trust --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$PROMPT"
GROK_HOME="$SCRATCH_GROK_HOME" RUST_LOG=xai_grok_hooks=debug GROK_LOG_FILE="$SCRATCH_LOG" grok --trust -p "$PROMPT" --permission-mode bypassPermissions --output-format plain
OPENCODE_CONFIG_CONTENT='{"permission":{"*":"allow"}}' opencode run --print-logs --log-level INFO "$PROMPT"
pi -p -e .pi/extensions/fm-primary-turnend-guard.ts --no-context-files --no-session "$PROMPT"
```

四個允許呼叫的觀察輸出分別是 `UNRELATED_EXECUTED`、成功的唯讀 `pgrep`、`CHECKPOINT_EXECUTED`，以及保留監看器文字為資料的兩行 `TMUX_ARGS:`。
每個執行環境都透過自己的原生配接器行為，將 exit 2 對應為阻擋最後一個指令。
穩定原因為 `[watcher-background] a protected watcher command cannot run in an asynchronous shell list or through nohup/disown`。
若遭拒指令真的執行，假的 arm 本文就會建立 `<harness>.sentinel`。
五個哨兵檔案全都維持不存在。

Codex 逐字紀錄對原先回報的三種誤判形式都顯示 `PreToolUse Completed`，只有背景 arm 顯示 `PreToolUse Blocked`。
Grok 除錯逐字紀錄顯示 `project/fm-primary-pretool-check` 四次 exit 0，接著背景 arm 為 exit 2，帶有 145 個 stdout 位元組、214 個 stderr 位元組與 `hook denied`。
OpenCode 顯示四個允許指令的輸出，接著是帶 stderr 拒絕物件的 `bin/fm-watch-arm.sh & failed`。
Claude 與 Pi 都回報前四次呼叫已執行，最後一次遭到阻擋。

原生監督路徑也在相同暫存專案驗證：

- Claude 以原生受版本控制的背景選項執行 `bin/fm-watch-arm.sh --restart`，產生 `watcher: started pid=<scratch> (scratch)`。
- Grok 使用 `background: true` 執行完全相同指令，hook 回傳 exit 0，假的 arm 產生相同 started 行。
- Codex 執行上述前景 checkpoint，產生 `CHECKPOINT_EXECUTED`。
- OpenCode 在 `tmux -L fm-pretool-smoke` 的互動式 TUI 執行，進入 `session.idle`，其未變更的 watch-arm 外掛建立暫存自動啟動監看標記。
- Pi 載入兩個主要擴充套件，呼叫 `fm_watch_arm_pi`，並建立暫存自動啟動監看標記。

所有原生路徑的自動標記都存在，所有拒絕哨兵都維持不存在。

<a id="automated-validation"></a>
## 自動化驗證

`tests/fm-arm-pretool-check.test.sh` 定義對抗式驗收矩陣。
每一列都透過 Codex 形式 stdin、Claude 形式 stdin、Grok 形式 stdin、OpenCode 形式 CLI 與 Pi 形式 CLI 入口執行。
套件也驗證真正換行位元組、直接分類器原因代碼、註解、heredoc 資料、格式錯誤與不支援的受保護語法、建構的動態酬載、格式錯誤傳輸的失敗放行、缺少執行階段的失敗放行、輸出結構，以及精確的配接器欄位轉送與 exit 2 對應。

執行：

```sh
bash -n bin/fm-arm-pretool-check.sh
shellcheck bin/fm-arm-pretool-check.sh tests/fm-arm-pretool-check.test.sh
node --check bin/fm-arm-command-policy.mjs
tests/fm-arm-pretool-check.test.sh
bin/fm-test-run.sh --all
```
