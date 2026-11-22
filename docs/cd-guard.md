<a id="cd-guard-pretooluse-seatbelt"></a>
# cd-guard PreToolUse 安全帶

本文件是 cd-guard PreToolUse 安全帶的權威人類可讀契約。
`bin/fm-cd-command-policy.mjs` 是唯一的決策負責者。
`bin/fm-cd-pretool-check.sh` 負責穩定的執行環境傳輸、主要 checkout 範圍與輸出呈現。
受版本控制的執行環境配接器只轉送指令文字，不自行分類。

它是共用同一套跨執行環境 hook 機制之主要工作階段防護家族的第三個成員：
其餘兩者為監看器啟動 PreToolUse 安全帶（`bin/fm-arm-pretool-check.sh`、`docs/arm-pretool-check.md`），以及回合結束監督防護（`bin/fm-turnend-guard.sh`、`docs/turnend-guard.md`）。

<a id="purpose-and-boundary"></a>
## 目的與邊界

主要 firstmate shell 會在工具呼叫之間保留工作目錄。
因此，意外執行持續生效的頂層 `cd projects/<clone>`，會悄悄移動 shell，讓下一個 firstmate 自己的指令——待辦寫入或 `fm-*` 生命週期呼叫——在專案複本內執行，而不是主目錄。
這確實發生過：持續生效的頂層 `cd`，讓 firstmate 的待辦寫入在專案複本內執行，而不是主目錄。
安全帶會在執行前，精確拒絕這種指令形式，也就是會持續改變主要 shell cwd 的操作。

此防護不是通用沙箱。
它只分類 shell 指令位置，絕不評估、展開、載入或執行提交指令的任何位元組。
它的威脅模型與監看器啟動安全帶相同，針對 Agent 的失誤：意外的裸 `cd projects/foo`，而非刻意混淆的繞過方式。

<a id="scope-plain-firstmate-checkouts-only"></a>
## 範圍：僅一般 firstmate checkout

防護只在 git-dir 等於 git-common-dir 的一般 firstmate checkout 中生效。
其他位置皆為靜默無操作（exit 0、無輸出），因此絕不干擾合法在自己專案或 firstmate 任務工作樹中工作的船員或 scout。

checkout 偵測由 `bin/fm-cd-pretool-check.sh` 負責；回合結束防護會辨識標記的範圍，是另一份獨立契約（`docs/turnend-guard.md`）。
一般的非工作樹 checkout，其 `git rev-parse --git-dir` 與 `git rev-parse --git-common-dir` 相同。
船員或 scout 任務工作樹，也就是 `bin/fm-spawn.sh` 一律提供的形式，則是兩者不同的連結 git 工作樹，因此防護在該處不生效。
checkout 還必須含有 `AGENTS.md` 與 `bin/`；任何無法確認為主要 checkout 的情況，都視為不生效，絕不阻擋。

cd-guard 不檢查 `.fm-secondmate-home`。
因此，git-dir 等於 git-common-dir 的 git clone 二副主目錄會套用防護，但由 treehouse 租用、本身為連結工作樹的二副主目錄則不生效。
二副的子船員與 scout 工作樹，也會因連結工作樹檢查而不生效。

<a id="block-vs-allow"></a>
## 阻擋與允許

判別依據是是否持續改變父 shell 的 cwd，而非是否出現 `cd` token。

防護會**阻擋**在父 shell 中實際執行於頂層位置的 `cd`、`pushd` 或 `popd` 內建指令，因為這些指令會持續改變主要 shell 自己的工作目錄。
涵蓋裸 `cd projects/foo`、`cd ..`、`cd`、`cd -`、目標不是主目錄本身的絕對路徑 `cd /some/path`（仍會持續移動父 shell）、`pushd <dir>`、`popd`、前導指定值形式如 `X=1 cd foo`、經引號或跳脫處理後形成裸內建指令的指令字片段，以及任何在父 shell 執行該內建指令的串列形式（`cd x && cmd`、`cmd; cd x`、`cmd || cd x`、`command cd x`、`command -p cd x`、`command -- cd x`、`builtin cd x`、`command builtin cd x`、`cd x >/dev/null`，以及以換行分隔的串列）。

防護**允許**其他所有情況，包含以下絕不可阻擋的安全限定範圍形式：

- 不改變 shell 自己 cwd 就能存取目標的指令：`git -C <dir> ...`、`make -C <dir> ...`，或直接在指令本身使用絕對路徑。
- 不會持續影響父 shell 的目錄切換：子 shell `(cd x && ...)`、`bash -c 'cd ...'`／`sh -c`／`zsh -c` 酬載、`env -C <dir> ...`、`find ... -execdir` 執行器、管線階段（`cd x | cmd`），或背景執行的 `cd x &`。
- 位於會 fork 或 exec 的包裝程式（`env`、`sudo`、`nohup`、`timeout`、`gtimeout`、`exec`）之後的 `cd`；它在子程序執行，絕不持續影響父 shell（而且通常只會失敗，因為 `cd` 是內建指令，沒有外部程式）。
- 帶路徑限定且名為 `cd`、`command` 或 `builtin` 的外部指令，例如 `./cd`、`/usr/bin/cd`、`./command`、`/usr/bin/command` 或 `./builtin`，因為它們以子程序執行，無法改變父 shell 的 cwd。
- `command` 查詢，例如 `command -v cd`、`command -V cd` 或合併選項形式 `command -pv cd`，因為它只回報指令解析結果，不執行所指定的內建指令。
- 作為資料出現的 `cd` token：引號文字（`echo "cd projects/foo"`）、註解、其他字詞的子字串（`cdk`、`abcd`、`record`）、`printf` 酬載，或後續任何引數字詞。

其餘絕對路徑 `cd` 刻意遭到阻擋：絕對路徑的允許例外，是給以絕對路徑定址目標的指令，不是給 `cd`；不論引數為相對或絕對路徑，`cd` 都會移動 shell 本身。
阻擋頂層 `cd` 具有明確的安全性：防護的穩態是「永遠位於主目錄」，因此回到主目錄的 `cd` 是多餘而非必要，阻擋絕不會造成寫入錯誤目錄。

<a id="the-home-itself"></a>
### 主目錄本身

唯一允許的 `cd` 目標是主目錄：頂層 `cd` 的單一字面絕對路徑引數，必須在邏輯規則（以字面方式摺疊 `..`，Bash 預設）與實體規則（像 `set -P` 一樣逐元件追蹤符號連結）下，都精確解析到傳輸層限定的主要 checkout（`cd /path/to/home`、`cd -- /path/to/home`、`cd /link/to/home`、`cd /path/to/home/bin/..`）。
這種指令無法將 shell 移出主目錄，因此拒絕只會造成操作阻力。
傳輸層將 checkout 以 `--home` 傳給政策，政策在兩種規則下，以精確路徑比較解析後目標與解析後主目錄；任何一種規則下會離開主目錄的目標都遭拒，例如 `link` 指向主目錄外時的 `cd /path/to/home/link/..`。
其他情況仍全部拒絕：子目錄（`cd /path/to/home/bin`）、同層或父目錄、`cd .` 這類相對目標、帶有政策必須評估的展開／替換／波浪號／glob 之目標（`cd "$PWD"`、`cd ~`）、帶選項（`cd -P /path/to/home`）或額外引數的 `cd`、對任何目標使用 `pushd` 或 `popd`，以及同一指令串列中稍後的 `cd`（`cd /path/to/home && cd projects/foo`）。
沒有 `--home` 時，政策不豁免任何目標。

<a id="accepted-non-goals"></a>
### 接受的不處理範圍

配合 Agent 失誤的威脅模型，防護刻意不追逐每種混淆繞過方式：

- 透過指令替換重建的 `cd`（`$(echo c)d x`），或隱藏在大括號群組內的 `cd`（`{ cd x; }`），不會被阻擋。避免對大括號群組遞迴，是因為此分類器無法可靠區分 `{ cd; }` 群組與 `{cd,foo}` 大括號展開，而誤擋比漏掉罕見繞過更糟。
- 格式錯誤或無法詞彙化的語法，會採取失敗放行（允許）。監看器啟動安全帶對無法分類的受保護指令採拒絕策略，cd-guard 則優先確保零誤擋，而非攔截格式錯誤的繞過，因為阻擋待辦寫入會危及正確性，漏掉罕見 `cd` 則只是維持既有風險。

若發現真正模糊且有誤擋風險的指令形式，不會憑猜測擴充防護；會將歧義提報處理，維持防護精確，而非過度積極。

<a id="stable-reason-code"></a>
## 穩定原因代碼

每次拒絕都會在文字原因之前，以方括號帶上一個穩定代碼。

| 代碼 | 意義 |
| --- | --- |
| `persistent-cd` | 頂層 `cd`／`pushd`／`popd` 會持續改變主要 shell 自己的工作目錄。 |

原因會指引呼叫端使用 `git -C <dir>`、在預定指令本身放置絕對路徑，或將 `cd` 限定在子 shell，以不移動 shell 的方式存取目標。
它不允許 `cd /home/project`，因為對主目錄本身以外的任何位置執行絕對路徑 `cd`，仍是持續目錄變更，會遭拒。

<a id="transport-and-fail-open-behavior"></a>
## 傳輸與失敗放行行為

`bin/fm-cd-pretool-check.sh` 支援受版本控制配接器使用的每種執行環境引擎入口形式，pi-signed 與 Pi 共用形式：

- Claude 透過 stdin JSON 的 `.tool_input.command` 傳送，並加上 `--claude`，以保留 Claude 只允許 stderr 拒絕輸出的要求。
- Codex 透過 stdin JSON 的 `.tool_input.command` 傳送，不加 `--claude`。
- Grok 透過 stdin JSON 的 `.toolInput.command` 傳送。
- OpenCode 透過 `--command <exact string>` 傳送精確指令字串。
- Pi、pi-signed 與 omp 透過 `--command <exact string>` 傳送精確指令字串。
- Cursor 透過 stdin JSON 的 `.tool_input.command` 傳送，並加上 `--cursor`，將拒絕呈現為 Cursor 自己的回傳決策物件。

處理順序由成本低到高：先是嚴格超集合的預篩選，再檢查主要 checkout 範圍，最後交給 Node 政策負責者。
預篩選會移除一般單引號、雙引號、反斜線、CR 與換行，再快速允許任何既不包含 `cd`、`pushd` 或 `popd` 子字串，也不含引號解碼標記（`$'` ANSI-C 或 `$"` locale）的指令；因此帶引號或跳脫的指令字片段會交給政策，而大多數指令不需付出 git 範圍呼叫與 Node 程序成本。
引號解碼標記集合與 `bin/fm-arm-command-policy.mjs` 的分類器解碼集合相互綁定：新增任何分類器可解碼的引號或展開形式，都必須在同一次變更擴充預篩選標記集合，否則它就不再是嚴格超集合。

stdin 為空、JSON 無法解析、stdin 路徑缺少 `jq`、缺少 Node、缺少政策負責者，或政策回應無效時，都以 exit 0、無輸出方式失敗放行。
損壞的 hook 絕不可拒絕所有 shell 工具呼叫。

<a id="output-contract"></a>
## 輸出契約

形式與 `docs/arm-pretool-check.md` 相同：

- 允許（以及主要 checkout 外不生效）回傳 exit 0，兩個串流均為空。
- 拒絕回傳 exit 2，並向 stderr 寫入 `{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny"},"systemMessage":"[persistent-cd] reason"}`。
- 預設拒絕模式也會向 stdout 寫入 `{"decision":"deny","reason":"[persistent-cd] reason"}`，供 Grok 使用。
- `--claude` 完全抑制 stdout，因為 stdout 非空時，Claude 會忽略 PreToolUse 拒絕。
- Codex 在 exit 2 時阻擋，並顯示 stderr。
- OpenCode 只有檢查器以 2 結束時才拋出錯誤。
- Pi、pi-signed 與 omp 只有檢查器以 2 結束時才回傳 `{block: true}`。

<a id="shared-classifier-ownership"></a>
## 共用分類器的責任歸屬

`bin/fm-cd-command-policy.mjs` 從 `bin/fm-arm-command-policy.mjs` 匯入 shell 詞彙分析器與指令位置分析（`Lexer`、`splitProgram`、`commandPosition`）；後者是 firstmate shell 分類的唯一負責者。
`basename` 仍是共用 arm 分類器的私有輔助函式，因為 cd 政策以精確的解碼後字詞身分辨識 shell 內建指令。
cd-guard 絕不重複實作 shell 詞彙分析，只在共用分類器上新增 cd 專屬決策。
`bin/fm-arm-command-policy.mjs` 只有直接呼叫時才執行自己的 CLI 入口，匯入時絕不執行，因此兩種政策維持為共用單一解析器的獨立 CLI。

<a id="harness-wiring"></a>
## 執行環境接線

| 執行環境 | 入口 | 檢查器 exit 2 時的配接器行為 |
| --- | --- | --- |
| Claude | `.claude/settings.json` 的 PreToolUse Bash hook，以 `--claude` 轉送 stdin | 阻擋工具呼叫；stderr 為拒絕物件，stdout 為空。 |
| Codex | `.codex/hooks.json` 的 PreToolUse hook，從 `pwd -P` 定位、驗證載入 hook 的 firstmate 根目錄，並轉送酬載 | 在 exit 2 時阻擋並顯示 stderr。 |
| Grok | `.grok/hooks/fm-primary-cd-check.json` 的 PreToolUse hook，以 `${GROK_WORKSPACE_ROOT:-}` 定位 | 使用 stdout 的 `decision=deny` 物件。 |
| OpenCode | `.opencode/plugins/fm-primary-cd-check.js`，1.x 使用 `tool.execute.before`，2.x 使用 `shell` 工具的 `execute.before` | 拋出錯誤，呈現為失敗的工具結果。 |
| Pi | `.pi/extensions/fm-primary-turnend-guard.ts` 的 `tool_call` 處理器 | 回傳 `{block: true}`；沿用已載入的主要擴充套件，不需額外 `-e` 旗標。 |
| omp | `.omp/extensions/fm-primary-turnend-guard.ts` 的 `tool_call` 處理器 | 回傳 `{block: true, reason}`，omp 將原因呈現給模型；在同一個自動探索的擴充套件中，於監看器啟動安全帶之前執行，因此不需 `-e` 旗標。 |
| Cursor | `.cursor/hooks.json` 的 `preToolUse` hook，比對 `tool_name` `Shell`，以 `--cursor` 轉送 stdin | 向 stdout 印出 Cursor 自己的 `{"permission":"deny","user_message":...}` 物件，並以 0 結束，因為 Cursor 讀取回傳物件而非結束狀態。沒有 `--cursor` 時，Cursor 傳來的酬載是 Cursor 同時載入之 Claude 設定的重複項，會予以允許；該共用判斷由 `docs/arm-pretool-check.md` 定義。 |

每個執行環境都將 cd-guard 與監看器啟動安全帶並行使用；兩者是獨立檢查，任一拒絕都會阻擋指令。
Grok hook 指令中的每個 shell 變數參照都帶有行內預設值（`${GROK_WORKSPACE_ROOT:-}`），因為 Grok 會在 `bash -lc` 執行前先展開原始 hook 指令，這與 `docs/arm-pretool-check.md` 記錄的要求相同。

<a id="automated-validation"></a>
## 自動化驗證

`tests/fm-cd-pretool-check.test.sh` 定義驗收矩陣。
每個阻擋與允許案例，都會透過 Codex 形式 stdin、Claude 形式 stdin、Grok 形式 stdin、OpenCode 形式 CLI 與 Pi 形式 CLI 入口執行。
套件也證明主目錄豁免（以字面絕對路徑 `cd` 到主要 checkout，不論是否使用符號連結都允許；子目錄、同層、父目錄、`pushd`、帶選項、未展開或 glob 目標都拒絕）、端到端 cwd 外洩回歸（firstmate 的待辦寫入流入專案複本，再於精確指令處被拒絕）、checkout 範圍（git clone 二副測試環境會生效、船員／scout 連結工作樹不生效、firstmate checkout 外不生效、git 儲存庫外不生效）、失敗放行傳輸行為、預篩選快速路徑、政策 CLI 輸出契約，以及各執行環境接線。

執行：

```sh
bash -n bin/fm-cd-pretool-check.sh
shellcheck bin/fm-cd-pretool-check.sh tests/fm-cd-pretool-check.test.sh
node --check bin/fm-cd-command-policy.mjs
node --check bin/fm-arm-command-policy.mjs
tests/fm-cd-pretool-check.test.sh
tests/fm-arm-pretool-check.test.sh
```

<a id="live-validation-record-2026-07-11"></a>
## 真實驗證紀錄，2026-07-11

每個執行環境都針對暫存、具有主要 checkout 結構的 firstmate checkout 執行：一般 git 儲存庫，含 `AGENTS.md`、`bin/` 中真正的 `fm-cd-pretool-check.sh`、`fm-cd-command-policy.mjs` 與 `fm-arm-command-policy.mjs`，加上無操作的假 `fm-arm-pretool-check.sh`、代替專案複本的 `projects/foo/`，以及受版本控制的執行環境 hook 設定。
不涉及真實監看器、艦隊狀態或船長真正的主要 checkout。
每個執行環境都被要求以獨立工具呼叫執行頂層 `cd projects/foo && touch <abs>/BLOCKED`（必須拒絕），以及子 shell `(cd projects/foo && touch <abs>/ALLOWED)`（必須執行），並以哨兵檔案作為可觀察結果。

執行環境版本與結果：

- **Claude Code 2.1.207**：已阻擋。Claude 回報頂層指令「denied by the `PreToolUse` hook (`fm-cd-pretool-check.sh`)」，`BLOCKED` 哨兵不存在，子 shell 形式則獲准執行。先前的對照 `touch` 證明執行環境確實有執行指令。
- **codex-cli 0.144.0**：已阻擋。Codex 記錄 `error=Command blocked by PreToolUse hook: {"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny"},"systemMessage":"[persistent-cd] a persistent top-level directory change ..."}`，`BLOCKED` 哨兵不存在，子 shell 的 `ALLOWED` 哨兵已建立。每個指令都執行 arm 與 cd 兩個 PreToolUse hook（兩行 `hook: PreToolUse Completed`），確認 Codex 會將酬載重新提供給陣列中的每個 hook。
- **OpenCode 1.17.18**：已阻擋。`opencode run` 印出 `✗ cd projects/foo && touch ... failed`，附帶 `Error: {"hookSpecificOutput":...,"permissionDecision":"deny"},"systemMessage":"[persistent-cd] ..."}`，`BLOCKED` 哨兵不存在，子 shell 的 `ALLOWED` 哨兵已建立。
- **Pi 0.80.6**：已阻擋。`BLOCKED` 哨兵不存在，子 shell 的 `ALLOWED` 哨兵已建立；同一工作階段中，頂層遭拒、子 shell 可執行的差異，只可能來自防護。
- **grok 0.2.93**：真實執行無法下結論。Grok Build API 回傳 `402 Payment Required: Grok Build usage balance exhausted`，因此模型從未發出探測指令。grok cd hook（`.grok/hooks/fm-primary-cd-check.json`）在結構上與 2026-07-09 已真實驗證的 arm 安全帶 grok hook 相同（`docs/arm-pretool-check.md`）：使用相同 `${GROK_WORKSPACE_ROOT:-}` 定位與 PreToolUse 拒絕處理；grok 形式 stdin 路徑（輸入 `.toolInput.command`、輸出 `{"decision":"deny"}`）也由 `tests/fm-cd-pretool-check.test.sh` 涵蓋。Grok 餘額恢復後請重跑，以補上真實驗證缺口。

啟動指令對照 `docs/arm-pretool-check.md` 的驗證方式：

```sh
claude -p "$PROMPT" --dangerously-skip-permissions --output-format text
codex exec --dangerously-bypass-hook-trust --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$PROMPT"
OPENCODE_CONFIG_CONTENT='{"permission":{"*":"allow"}}' opencode run --print-logs --log-level INFO "$PROMPT"
pi -p -e .pi/extensions/fm-primary-turnend-guard.ts --no-context-files --no-session "$PROMPT"
grok --trust -p "$PROMPT" --permission-mode bypassPermissions --output-format plain
```
