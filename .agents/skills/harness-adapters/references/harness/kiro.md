<a id="kiro-cli-v2-engine"></a>
# Kiro CLI（V2 引擎）

Kiro 的 `kiro-cli` TUI，於 2026-09-13 在 Amazon Linux 2 上透過 tmux，以 kiro-cli 2.21.4 完成端到端驗證，僅限 V2 Agent 引擎。
於 2026-10-29 在 Linux 上透過 tmux，使用 kiro-cli 2.35.1 再次驗證，並加做 Herdr 輸入區探測，涵蓋 Auto 與 `glm-5`。
只驗證為 CREWMATE 與 SCOUT 配接器；`../../../../../bin/fm-spawn.sh` 會拒絕使用它啟動二副，因為 `../../../../../docs/supervision-protocols/` 沒有 kiro 喚醒協定。
下列每項事實的建立方式與尚未證明部分，由 `../../../../../docs/verification/kiro.md` 負責。

**僅限 V2。** 啟動固定 `--agent-engine v2`，這是 AL2 支援的引擎。v3/KAS 引擎不在範圍內，也未在此測試：這台 Amazon Linux 2 主機不支援它，其 hooks 也尚未對齊，因此不可將 v3 行為套用到此配接器。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 執行檔 | 從 `PATH` 解析的絕對 `kiro-cli`，缺少就拒絕；已安裝命令是 toolbox/aim-sandbox 包裝器，前景程序名稱恰為 `kiro-cli`。 |
| 啟動 | 使用解析後的絕對執行檔，執行 `kiro-cli chat --agent-engine v2 --agent firstmate --model <id> --effort <level> --trust-all-tools "<brief>"`，並將 `KIRO_HOME` 重新定位到個別任務 home；位置參數簡報會自動送出，不需額外 Enter。沒有先啟動再確認的關卡：與 claude 相同，窗格直接啟動。 |
| 忙碌狀態 | Claude 形式的 V2 Agent 設定 hooks 是**唯一**來源：`userPromptSubmit` 開啟回合，`stop` 關閉，寫為 `../../../../../bin/fm-busy-lib.sh` 的 `kiro-hook` 紀錄。沒有紀錄的 kiro 任務分類為 `unknown missing`；分類器絕不為 kiro 讀取窗格。 |
| 畫面尾端 | 忙碌輸入區頁尾為 `Kiro is working · <elapsed> · Type to steer · Ctrl+S to queue`（2.35.1 加入經過時間）；閒置時顯示 `Trust All Tools active ... /quit to exit` 或 `ask a question or describe a task` 預留文字。比對包含執行環境名稱的字面值 `Kiro is working`，絕不使用 kiro 與 agy 共用的單獨 `esc to cancel` 字串。它是 `../../../../../bin/fm-composer-lib.sh` 的**送達**防護，絕不是工作 Agent 狀態來源；目前唯一可到達的使用端，是 tmux 送出核心為確認送出而讀取的 `FM_DELIVERY_BUSY_REGEX_DEFAULT` 無執行環境聯集；`FM_DELIVERY_KIRO_BUSY_REGEX_DEFAULT` 依艦隊慣例登錄，但目前沒有呼叫端。 |
| 回合結束 | `stop` hook 保留 `state/<id>.turn-ended` 的通知 touch。與 claude 相同，手動 Escape 中斷**不會**觸發 `stop`；kiro V2 沒有已驗證的 StopFailure/SessionEnd 對等事件，因此異常回合結束會讓紀錄維持忙碌，直到下一次 `userPromptSubmit`，監督者也會將該紀錄讀為已證明仍在工作。 |
| 結束 | `/quit`，一次 Enter；程序結束並印出 `Session ended.`，接著 `Resume with: kiro-cli --resume-id <session-id>`。 |
| 中斷 | 按一次 `Escape`，會印出 `Cancelled ...` 列，留下閒置輸入區，不會再次污染，因此後面不接清除按鍵。 |
| 技能 | 沒有已驗證的斜線技能形式；使用自然語言。 |
| 自主性 | `--trust-all-tools` 會自動核准工具呼叫；原本會阻擋的確認模態視窗由預先植入的 `chat.disableTrustAllConfirmation` 設定抑制（見下方信任說明）。 |
| 識別標記 | 無；實際工具子程序不帶 `KIRO_*` 身分變數。`KIRO_HOME` 是 firstmate 設定的設定搬移路徑，不是身分。 |
| 恢復 | 互動結束行會印出 `--resume-id <id>`，工作階段存在個別任務 `KIRO_HOME` 下，因此 `--resume-id`（或使用 `--resume` 恢復 cwd 最近工作階段）會帶上下文恢復。Firstmate 不會自動化恢復：控制平面可確定重現的 `relaunch` 會重新讀取持久化簡報，與其他配接器一致。 |
| 模型 | `--model <id>`，使用 `kiro-cli chat --agent-engine v2 --list-models -f json` 目錄中的裸 `model_id`（例如 `claude-opus-5`、`auto`）；若可存取的清單未列出要求的 ID，`bin/fm-spawn.sh` 會拒絕。清單需要遠端擷取，因此探測會分離 stdin，並套用共用硬性時限；清單無法存取、卡住或不含 `model_id` 時，會提示後不驗證就啟動。 |
| 推理強度 | `--effort low\|medium\|high\|xhigh\|max`；完整共用詞彙直接傳入（與會省略 `xhigh`/`max` 的 agy 不同）。 |
| 輸入區 | 無框線的 `›`（U+203A）Agent 符號列，與 codex 使用相同符號。`ask a question or describe a task` 預留文字接近灰色，亮度 158；真彩色窗格使用 `38;2;158;158;158`，沒有 `COLORTERM` 的窗格則使用色盤索引 `38;5;247`，兩者是相同灰色。共用幽靈文字移除器使用接近無彩色的亮度上限，且只在呼叫端指定紀錄執行環境為 `kiro` 時套用（`FM_COMPOSER_HARNESS`，由 `../../../../../bin/fm-composer-lib.sh` 的 `fm_composer_kiro_scope` 負責），因此真正的閒置窗格在任一編碼都讀為 `empty`（已用 tmux 自己的描述元對實際擷取驗證）。在此上限加入前，預留文字未被移除，窗格讀為 `pending`，使所有引導都被抑制；此歷史由 `../../../../../docs/verification/kiro.md` 負責。kiro-cli 2.28.0 以預設前景色繪製同一列，任何移除規則都不會移除，因此 kiro 範圍的 `FM_COMPOSER_HINTED_IDLE_RE_DEFAULT` 會透過輸入文字絕不顯示的 `↵` 送出提示比對整列，並讀為 `empty`；不依賴游標的讀取也會在此停止，因此下方 `/copy to clipboard` 頁尾絕不會被當成換行輸入。kiro 沒有在艦隊全域閒置預留文字集合登錄項目，這不影響此處判定。 |

<a id="out-of-tree-agent-config-the-turn-end-wiring"></a>
## 樹外 Agent 設定（回合結束接線）

`--agent` 只接受名稱：設定路徑會被拒絕，探索位置是全域 `KIRO_HOME/agents/` 目錄，加上工作區 `<cwd>/.kiro/agents/` 目錄。
工作區目錄位於可拋棄 worktree 內，絕不可寫入，因此 `../../../../../bin/fm-spawn.sh` 會在 `state/<id>.kiro-home/agents/firstmate.json` 寫入 firstmate 擁有的個別任務 Agent 設定，並在啟動命令將 `KIRO_HOME` 重新定位到 `state/<id>.kiro-home` 以存取它，採 gemini 形式（專案外的專用個別任務設定，透過設定指標存取）。
同一個別任務 home 也包含 `settings/cli.json`，預植 `chat.disableTrustAllConfirmation`，抑制 `--trust-all-tools` 模態視窗（全新啟動唯一阻擋項）。
重新定位 `KIRO_HOME` 會搬移整個全域設定根目錄（agents、settings、sessions）；在 Amazon Linux 2 上，驗證不受影響，因為驗證資料位於 XDG 資料目錄（`~/.local/share/kiro-cli`），所以工作 Agent 使用操作人員的真實登入，同時完全不碰船長真正的 `~/.kiro`。
該路徑在 macOS 不存在，該平台的驗證資料位置也尚未確定，因此「搬移不影響驗證」的前提不能延伸至 macOS；見下方憑證前置條件。
由於這是整體搬移，而非 gemini 的新增單檔層，個別任務 home 不會繼承操作人員的全域 MCP 伺服器、技能或 Agent；個別任務 Agent 設定宣告 `tools: ["*"]`，船員依賴內建工具。
該設定的每個 hook `command` 都是單一 token 的絕對指令碼路徑，指令碼由 spawn 在旁邊的 `state/<id>.kiro-home/hooks/` 產生，因此不論 kiro 直接 exec 命令或交給 shell，hook 都以相同方式執行；忙碌事件、以 `|| true` 容忍過期世代被拒絕，以及回合結束 touch 都在指令碼內。
在這兩種解讀下，單一 token 無法安全帶有空白或 shell 特殊字元（引號、`$`、反引號、`;`、`&`、`|`、`<`、`>`、括號、`*`、`?`、方括號、大括號、`!`、反斜線），因此標準 kiro spawn 若發現解析後的 `state/<id>.kiro-home` 路徑含其中任一字元，就會在啟動前拒絕，指出路徑並要求使用不含這些字元的 `FM_HOME` 或 `FM_STATE_OVERRIDE`；沒有 hooks，kiro 就沒有其他狀態來源，所以絕不啟動永遠無法清除忙碌紀錄的窗格。
`../../../../../bin/fm-control-lib.sh` 將 Agent 設定檔與兩支 hook 指令碼列為退役路徑，讓重新啟動會退役該實例的 hooks；`../../../../../bin/fm-teardown.sh` 則移除整個 `state/<id>.kiro-home` 目錄。

<a id="credential-precondition"></a>
## 憑證前置條件

已驗證的 kiro 工作 Agent 在 Amazon Linux 2 上，以已登入帳號執行，沒有匯出任何金鑰。
macOS 尚未確立：Amazon Linux 2 的驗證路徑在該平台不存在，而此配接器無法判定 kiro 在 macOS 將驗證資料放在哪裡，因為判定就必須探查屬於操作人員的憑證儲存區。
因此 macOS 上建立的 kiro 工作 Agent 可能遇到互動驗證提示，使該執行環境無法在該平台無人值守；缺口與成本由 `../../../../../docs/verification/kiro.md` 負責。
任何驗證提示或拒絕都依 `../../../../../AGENTS.md` 第 9 節視為憑證阻擋，修正環境並退役端點，不要對提示輸入。

<a id="detection"></a>
## 偵測

只依祖先鏈偵測：`../../../../../bin/fm-harness.sh` 比對明確錨定的程序名稱 `kiro-cli`，絕不使用 `*kiro*`。
不提升任何環境標記，而 kiro 不會清除繼承的 `CLAUDECODE`，因此 spawn 在啟動邊界清除其他標記，再由祖先鏈分支決定。
kiro 刻意不列入 `../../../../../bin/fm-session-lock-lib.sh` 中可作主 Agent 的工作階段鎖定詞彙，其他僅限船員的配接器也一樣：僅限船員的配接器絕不可擁有 home 工作階段鎖定。

<a id="worker-busy-state-and-turn-end"></a>
## 工作 Agent 忙碌狀態與回合結束

`../../../../../bin/fm-spawn.sh` 會為 kiro 啟用忙碌世代，並嵌入 hook 命令，與 claude 及 gemini 完全相同。
`fm_busy_classify` 有有效紀錄時回傳 `kiro-hook` 紀錄，沒有則回傳 `unknown missing`；它沒有 kiro 窗格分支，因此畫面頁尾絕不會分類 kiro 工作 Agent。
因此異常回合結束會讓紀錄保持忙碌，直到下一次 `userPromptSubmit` 重新開啟，監看器也會將該紀錄讀為已證明仍在工作。
拆除會移除個別任務 home，並退役忙碌世代。

<a id="primary-integration"></a>
## 主 Agent 整合

不支援，也未驗證。
`../../../../../docs/supervision-protocols/` 沒有 kiro 協定，沒有對應的回合結束防護配接器；此配接器只在 V2 引擎驗證船員側啟動、忙碌狀態、中斷、結束與恢復。
適用 `references/common/primary-hooks.md` 的不支援邊界規則：絕不從相似 TUI 自行杜撰喚醒協定。
