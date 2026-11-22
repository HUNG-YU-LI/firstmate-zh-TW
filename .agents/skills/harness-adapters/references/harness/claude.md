# Claude

忙碌 hooks 於 2026-07-28 在 Claude Code 2.1.220 驗證。

<a id="operating-facts"></a>
## 操作事實

這些命令列與終端機事實適用於窗格式工作階段；T3 Code 請閱讀[原生生命週期](../../../../../docs/t3code-backend.md#current-lifecycle-and-safety)與[供應商命令限制](../../../../../docs/t3code-backend.md#active-limits)。

| 事實 | 值 |
|---|---|
| 忙碌 | 負責狀態的 hooks：`UserPromptSubmit` 開啟，`Stop`、`StopFailure` 與 `SessionEnd` 關閉；二副不取得 `Stop` 忙碌 hook，因為 home 受版本控制的 Stop 防護是唯一 Stop 寫入器，除非該防護早於 `.fm-busy-stop`，啟動才保留 `Stop` 閒置 hook（由 `../../../docs/turnend-guard.md` 負責）；手動中斷不發出 hook，因此控制層只回報按鍵已送達與端點存活，不發布閒置事件，也不宣稱已取消，通常會留下 `claude-hook` 忙碌狀態。 |
| 結束 | `/exit`。 |
| 中斷 | 按一次 Escape。 |
| 技能 | `/<skill>`，例如 `/no-mistakes`。 |
| 模型 | `--model <model>`；透過互動式 `/model` 選擇器探索，別名或完整名稱形式由 `claude --help` 說明。 |
| 推理強度 | `--effort <low\|medium\|high\|xhigh\|max>`，已於 2.1.196 驗證。 |
| 權限 | 見 [Claude 權限模式](../../../../../docs/configuration.md#claude-permission-mode-configclaude-permission-mode)與[持久化容錯切換](../../../../../docs/configuration.md#persistent-managed-policy-failover)；`auto` 啟動形式已於 2.1.269 驗證。 |

<a id="workspace-trust"></a>
## 工作區信任

Claude 對從未見過的資料夾設有互動式工作區信任對話框（標題為「Quick safety check: Is this a project you created or one you trust?」），因此每個全新任務 worktree 都會遇到它，每個操作人員尚未手動開啟過的二副 home 也是。
`--dangerously-skip-permissions` 不涵蓋此關卡：`claude --help` 記載，只有透過 `-p` 或非 TTY stdout 的非互動模式才會略過此對話框，而 Firstmate 窗格式與 T3 供應商工作階段都是互動式。
因此，每次 Claude spawn 都會在啟動前預先登錄啟動目錄，使對話框不會出現：ship 或 scout 使用任務 worktree，`--secondmate` spawn 則使用 home 本身，兩種預建形式（租用 worktree 或獨立 clone）皆同。

另一個獨立對話框「Allow external CLAUDE.md file imports?」，會在載入的 CLAUDE.md 鏈延伸到專案樹外時出現；每個船員都會透過船長自己的 `~/.claude/CLAUDE.md` 匯入 `~/.claude/RTK.md`，因此都符合。
只有 `../../../bin/fm-live-lab.sh` 會傳入的 `--setting-sources project,local`（`fm-spawn.sh` 工作 Agent 也會載入使用者設定）同樣不會抑制它；它與信任對話框一樣阻擋工作階段：游標停在「No, disable external imports」，而 firstmate 引導平面沒有移動選項的方式。

對 ship 或 scout spawn，`../../../bin/fm-claude-trust.sh` 會在 `${CLAUDE_CONFIG_DIR:-$HOME}/.claude.json`，同時為 worktree 與其主 checkout 記錄 `hasTrustDialogAccepted`；`CLAUDE_CONFIG_DIR` 由 home 的工作 Agent 帳號固定值，或 spawn 指定的已宣告帳號決定（`../../../docs/configuration.md` 的「工作 Agent 帳號固定」）。二副 spawn 只登錄自己的 home 項目，因為二副 home 沒有另一個主 checkout 項目可延續匯入同意。
對 ship 或 scout spawn，只有主 checkout 的專案項目已帶有先前互動工作階段明確設定的 `hasClaudeMdExternalIncludesApproved===true`，外部匯入旗標（`hasClaudeMdExternalIncludesApproved`、`hasClaudeMdExternalIncludesWarningShown`）才會與信任旗標一同延續；常見首次 spawn 情況是 claude 從未詢問過此專案，因此這兩個旗標不寫入，即使信任正常登錄，匯入對話框仍會出現。
若專案項目已明確拒絕（`hasClaudeMdExternalIncludesApproved===false` 且 `hasClaudeMdExternalIncludesWarningShown===true`），整個登錄會拒絕——包括信任旗標——以免製造人類從未給予的同意，因此該 spawn 會先卡在信任對話框，甚至還沒到匯入對話框。
兩個旗標皆為 `false` 是 Claude Code 對從未詢問過專案的預設項目，不代表拒絕，會視同旗標不存在：信任登錄，匯入對話框仍出現。
為何需要兩個項目的機制與同意關卡邏輯，位於指令碼自己的檔頭註解，該處是此契約的唯一負責來源；此處值得重述的是，若信任旗標未成功寫入，`../../../bin/fm-spawn.sh` 會拒絕 spawn，不會啟動必定卡在對話框的工作 Agent。

絕不嘗試用按鍵回答任一對話框。
Firstmate 的按鍵平面只支援 Enter、Escape 與 C-c，沒有方向鍵導覽，因此完全不能移動對話框選取；兩個對話框顯示時游標都停在拒絕選項，送 Enter 會結束工作階段，而不是接受。
看到信任對話框表示預先登錄未生效（或專案項目已有明確拒絕）；應檢查儲存區與 spawn 錯誤輸出，不要傳送按鍵。
若專案項目未記錄過明確核准，看到外部匯入對話框是預期行為，不是失敗訊號；這正是常見首次 spawn 情況。
`fm-control.sh <id> interrupt` 會送達 Escape，可安全清除卡住的工作區信任對話框以便檢查，而不作回答。
在外部匯入對話框按 Escape 則不同：它會記錄永久拒絕（`hasClaudeMdExternalIncludesApproved: false`、`hasClaudeMdExternalIncludesWarningShown: true`），之後該專案每次 spawn，`../../../bin/fm-claude-trust.sh` 都會正確拒絕覆蓋。
窗格若顯示外部匯入對話框，應保持不動，請人員互動回答，不要中斷。
若要從已記錄的拒絕復原，從 `~/.claude.json` 的專案項目移除兩個旗標，再手動核准一次匯入對話框。

每台機器一次的 bypass-permissions 確認是第三種獨立對話框，範圍是機器而非路徑，預先登錄不會處理它。
也絕不對它傳送 Enter：已觀察到它呈現與信任對話框相同形式，選取 `No, exit`，頁尾為 `Enter to confirm . Esc to cancel`，因此 Enter 會結束工作階段，不是接受。
Firstmate 只靠 Enter、Escape 與 C-c 無法移動選取，所以完全無法接受此對話框，必須由操作人員每台機器接受一次。
檢查工作階段辨識螢幕上是哪個對話框，並回報，不要回答。
在 `config/claude-permission-mode=auto` 下啟動不會遇到 bypass 確認，因為它不要求 bypass 模式：2.1.269 的 `claude --permission-mode auto` 直接到達輸入區，頁尾為 `⏵⏵ auto mode on (shift+tab to cycle)`，因此拒絕 bypass 對話框的船長可改選該處的 `auto`，不必接受。
工作區信任對話框不受權限模式影響，仍需上面的預先登錄。

<a id="composer-ghost"></a>
## 輸入區幽靈文字

回合完成後，空白輸入區可能顯示淡色預測文字，在純文字 `tmux capture-pane` 中無法區分。
Firstmate 產生的 Claude 命令列會將 `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false` 限定到每個工作 Agent 與二副，不更動全域設定。
T3 Code 負責自己的供應商命令，無法帶入此設定；該後端限制由 [`docs/t3code-backend.md`](../../../../../docs/t3code-backend.md#active-limits) 負責。
CLI `--prompt-suggestions` 只影響列印或 SDK 模式，在 v2.1.186 無法抑制互動式幽靈文字。

作為縱深防禦，`../../../bin/fm-composer-lib.sh` 的 `fm_composer_strip_ghost` 會在帶樣式的 tmux、Herdr 與 Zellij 讀取器進行待送出分類前，移除 SGR-2 片段。
深色 TRUECOLOR 的取捨由 `../../../docs/herdr-backend.md` 的「輸入區與注入安全」負責，擷取內容由 `../../../docs/verification/runtime-backends.md` 負責。
帶樣式擷取只留在布林偵測器內部；`fm-peek` 與面向模型的擷取維持不含跳脫碼的純文字。

<a id="feedback-drafts"></a>
## 意見回饋草稿

Firstmate 產生的 Claude 命令列會停用 Claude 的 `/bug` 與 `/feedback` 模型代擬意見回饋流程，避免工作 Agent 與二副代表船長排入或送出錯誤報告。
T3 Code 負責自己的供應商命令，無法帶入這些控制；該後端限制由 [`docs/t3code-backend.md`](../../../../../docs/t3code-backend.md#active-limits) 負責。
控制範圍僅限啟動的程序，絕不修改船長全域 Claude 設定；精確機制與縱深防禦理由由 `../../../../../bin/fm-spawn.sh` 的 `launch_template()` 負責。

<a id="task-control-channel"></a>
## 任務控制通道

Claude 任務工作 Agent 的啟動簡報與 Firstmate 引導收件匣訊息，以檔案形式內容送達，除此之外無法與間接提示注入區分。
`../../../../../bin/fm-spawn.sh` 的 `launch_template()` 透過 `--append-system-prompt`，將恰好這兩個 Firstmate 擁有的通道確立為第一方指示；專案檔案、擷取內容與其他外部資料仍適用模型正常的不信任規則，且不授予超出簡報範圍的合併、破壞性操作或安全敏感權限。
`--secondmate` 啟動省略此聲明，因為二副依自己的監督者契約運作，而非任務工作 Agent 契約。
t3code 後端請遵循[逐目錄工作 Agent 通道契約](../../../../../docs/t3code-backend.md#per-directory-harness-environment)。

<a id="skill-listing-budget"></a>
## 技能清單預算

Claude Code 會列出每個技能名稱，但技能說明只保留在模型上下文視窗 1% 的預算內；超出時，先丟棄最少被呼叫技能的說明（[技能](https://code.claude.com/docs/en/skills#skill-descriptions-are-cut-short)，於 2026-10-19 閱讀）。
Firstmate 僅供 Agent 的技能刻意很少呼叫，因此在小上下文模型，或與大量外掛技能並存時，其說明會最先消失，而這些說明正是 `AGENTS.md` 第 13 節仰賴的觸發索引。
執行 `/doctor` 查看清單估算成本與主要占用項，並用 `/skill-doctor`（Claude Code 2.1.252 或更新版本）查看每個技能的上下文成本與呼叫頻率（[找出未使用技能](https://code.claude.com/docs/en/skills#find-unused-skills)）。
兩個供應商設定可由各 home 自行選用，絕不是共用契約的一部分：

- `skillListingBudgetFraction` 可將預算從預設 `0.01` 提高，最高 `1`（[設定參考](https://code.claude.com/docs/en/settings-reference#skilllistingbudgetfraction)）。
- 將 `skillOverrides` 設為 `"name-only"`，會只列技能名稱而不列說明（[可見性覆寫](https://code.claude.com/docs/en/skills#override-skill-visibility-from-settings)）；只對此 home 絕不需要的技能使用，例如未使用 Relay 的 `fmx-respond`，或未使用 Orca 的 `firstmate-orca`，因為這會移除該技能的觸發條件。

兩者都應放在使用者設定或未追蹤的 `.claude/settings.local.json`，絕不放在受版本控制的專案 `.claude/settings.json`，否則會將某一 home 的選擇強加到每個 home 與工作 Agent 副本。

<a id="primary-integration"></a>
## 主 Agent 整合

[`../../../../../docs/verification/supervision.md`](../../../../../docs/verification/supervision.md#turn-end-guard) 記錄目前主 Agent 與 Stop 自動啟用的實際證據。
這與工作 Agent hook 不同，後者只透過 `.claude/settings.local.json` touch 任務標記。
Claude 二副的忙碌 hooks 不在該檔案：`bin/fm-spawn.sh` 透過 `--settings` 傳入父 home 的 `state/<id>.claude-settings.json`，不編輯二副 home 的 `settings.local.json`。

主 Agent 的 `.claude/settings.json` 在 `Stop` 下註冊 `../../../bin/fm-turnend-guard.sh --claude` 與 `../../../bin/fm-claude-stop-autoarm.sh`，並在 `StopFailure` 下再次註冊 `../../../bin/fm-claude-stop-autoarm.sh`；兩個自動啟用項目都帶有 `asyncRewake: true` 與 `timeout: 28800`。
防護結束碼 2 加 stderr 會強制接續。
任何 hook 驅動的接續（包括非同步重新喚醒）之後，Stop 承載資料都會帶 `stop_hook_active=true`，因此 Claude 模式忽略它，改用合作式認領、epoch 與有界限的再次阻擋；預設 Codex 模式則保留它作為一次阻擋的迴圈防護。

只有精確專案根目錄就是工作階段根目錄時，才會載入專案 `.claude/settings.json`；Claude 不搜尋父目錄，因此 Firstmate 從儲存庫根目錄啟動。
Hooks 仍透過對 cwd 敏感的 `/bin/sh` 執行，因此受版本控制的命令以 `"$CLAUDE_PROJECT_DIR"/bin/...` 錨定。
細節由 `../../../docs/turnend-guard.md` 負責。

Stop 負責的監看器 hook 在每個 Stop 與每個 StopFailure 執行，只有符合資格時才以前景執行 `../../../bin/fm-watch-arm.sh`，並使用結束碼 2 的非同步重新喚醒作為通知。
模型處理通知，但絕不負責例行重新啟用。
在 Claude Code 2.1.317，觸發的 `StopFailure` 帶有 `session_id`、`transcript_path`、`cwd`、`prompt_id`、`error`（例如 `authentication_failed`、`server_error`、`connection_failed`）與 `last_assistant_message`，並與 `Stop` 一樣遵守 `asyncRewake` 註冊及結束碼 2 的重新喚醒（2026-11-03 已在 Herdr 上的主 Agent 實際驗證；證據由 `../../../../../docs/verification/supervision.md` 負責）。
錯誤結束後不會執行 `Stop` 階段，因此防護的阻擋預算不會計入；改為每次 `StopFailure` 驅動、以結束碼 2 接續（重新喚醒、失敗通知、受抑制重試、警報或中斷復原）在確定執行時寫入 `state/.claude-autoarm-stopfailure-rewake`，下一次若位於 `FM_CLAUDE_STOPFAILURE_REWAKE_COOLDOWN`（預設 300 秒）內，就先等完剩餘時間再確定執行（期間若出現 AFK 或監督需求消失，就靜默退讓），讓 API 錯誤風暴以每個時間窗一次接續的節奏進行，同時絕不丟掉確保下一次 hook 觸發的接續。
除非 home 透過 `config/supervision-host-off` 退出，hook 會改以前景執行監督主機，該主機也以 Claude 列印模式作為無介面引擎；已驗證的引擎事實由 [`supervision-host.md`](../../../../../docs/supervision-host.md#engines) 負責。
Claude 的 PreToolUse 安全帶會直接阻擋，且只有 stdout 為空時拒絕才會被遵守；該契約由 `../../../docs/arm-pretool-check.md` 負責。

<a id="delegation-guard"></a>
### 委派防護

Claude 的委派、排程與 worktree 工具可以建立沒有 `state/<id>.meta` 的工作，使防護無法計數。
`../../../bin/fm-subagent-pretool-check.sh` 會拒絕具有委派形式的工具名稱。
主 Agent 也應在未追蹤、僅限該 home 的 `permissions.deny` 中列出已知委派工具，使它們從結構描述中消失。
絕不將其追蹤於專案 `.claude/settings.json`，因為它只適用 Claude，且會傳播到工作 Agent 副本，讓合理委派被停用。
契約、建議、`FM_ALLOW_SUBAGENT=1` 與適用性審查由 `../../../docs/subagent-guard.md` 負責。

在 Claude 2.1.217，工具顯示為 `Agent`；以無意義控制值作 A/B 比較時，`Agent` 與 `Task` 都可作為拒絕鍵。
`permissions.allow` 是預先核准，不控制可用性，因此不存在封閉式正向允許清單。
