<a id="devin-cli-worker-verification"></a>
# Devin CLI 工作 Agent 驗證

讀者：維護者驗證。

於 2026-09-21 驗證，並在 2026-09-22 重新驗證；環境為 macOS arm64，使用 `devin 3000.11.1 (cc4e349ca55e)`。
[轉接器參考文件](../../.agents/skills/harness-adapters/references/harness/devin.md) 負責操作事實；負責執行的程式則承載啟動與狀態機制。
本驗證涵蓋船員（crewmate）與偵察員（scout），以 tmux 作為實際測試的執行期後端，並使用 Herdr 0.9.0 實驗工作階段進行下列生命週期檢查。
主要 Agent、二副、ACP 及配額供應商整合，不在此保證範圍內。

<a id="refresh-commands"></a>
## 更新驗證的指令

```sh
devin --version
devin --help
devin auth status
devin models list
bin/fm-test-run.sh tests/fm-devin-harness.test.sh
FM_DEVIN_SIGNALS_LIVE=1 bin/fm-test-run.sh tests/fm-devin-signals-live-e2e.test.sh
FM_DEVIN_SIGNALS_LIVE=1 FM_DEVIN_MODEL=fusion-claude-fable-5-1-high-sidekick-swe-2-medium bin/fm-test-run.sh tests/fm-devin-signals-live-e2e.test.sh
```

使用憑證的驗證防護，會將檔案憑證複製到隔離主目錄，使用私人 tmux socket，並執行 `fm-spawn.sh` 實際產生的指令。
該主目錄帶有絕不可觸發的使用者 Claude Code hook；工作 Agent 自己建立的提交也不得帶有 Devin 署名。
worktree 配置與初始端點投遞使用可攜式測試夾具；後續引導與生命週期控制，則使用真正後端及供應商程序。
未登入或沒有可隔離的檔案憑證時會略過；工具不存在及選擇性啟用的行為，由共用真實測試關卡負責。
失敗訊息會指出已安裝的 Devin 版本。

<a id="live-guard-results"></a>
## 真實驗證防護結果

2026-09-21，兩次更新驗證呼叫都以 0 結束：SWE-2 Medium 耗時 99 秒，Fusion Fable High + SWE-2 Medium 耗時 194 秒。
2026-09-22，擴充後的驗證防護在 SWE-2 Medium 上執行 65 秒，以 0 結束：

```text
ok - devin 3000.11.1 (cc4e349ca55e): spawn brief, model, autonomy, trust, identity and native Stop
ok - devin 3000.11.1 (cc4e349ca55e): no Claude Code hook ran and the worker commit carries no attribution
ok - devin 3000.11.1 (cc4e349ca55e): real fm-send doorbell read and acknowledged
ok - devin 3000.11.1 (cc4e349ca55e): idle interrupt sends one press; an open revert picker blocks exit and is closed without reverting
ok - devin 3000.11.1 (cc4e349ca55e): double Escape cancels, preserves agent, and invalidates busy state
ok - devin 3000.11.1 (cc4e349ca55e): /quit and native -r session resume
```

<a id="observed-vendor-surfaces"></a>
## 觀察到的供應商介面

版本輸出：

```text
devin 3000.11.1 (cc4e349ca55e)
```

驗證狀態回報 `Logged in (via Devin).`，方案為 `Max`。
實際查詢帳號取得的模型清單列出：

```text
swe-2-medium  SWE-2 Medium  [262K context, Free]
fusion-claude-fable-5-1-high-sidekick-swe-2-medium  Fusion (Claude Fable 5.1 High + SWE-2 Medium)  [1M context, $10 / 1M Input · $0.25 / 1M Cached input · $50 / 1M Output · Sidekick: Free]
```

模型供應情況與價格是此帳號及此日期的觀察結果，不是由轉接器維護的目錄。
`--prompt-file <file>` 與放在 `--` 之後的提示詞，都能在無須額外輸入下啟動互動式回合。
Spawn 使用後者，搭配標準的操作輸入編碼器。
`--permission-mode dangerous --respect-workspace-trust false` 能在全新儲存庫處理初始提示詞並寫入檔案，不顯示權限或信任對話框。
根據 `--help`，Devin 的 `auto` 模式只自動核准唯讀工具；它不會由 Claude 定義不同的 auto 模式映射而來。

私人設定的原生 hook 日誌，為一次使用工具的回合記錄了以下順序：

```text
SessionStart source=startup
UserPromptSubmit
PreToolUse tool_name=exec
PostToolUse tool_name=exec
Stop stop_hook_active=false last_assistant_message=80235
```

產生的 hooks 建立一筆 `state=idle source=devin-hook event=stop` 紀錄，以及回合結束通知。
真實的 Fusion 恢復記錄了 `SessionStart source=resume`，從自己的 shell 工具執行 `bin/fm-harness.sh` 並輸出 `devin`，最後回傳 `17 × 29 = 493`。
頁尾識別為 `Fusion · Claude Fable 5.1 ◆ SWE-2 Medium`。

執行中的回合會同時顯示 `esc twice to interrupt` 與 `❭ Guide Devin while it works`。
對執行中回合按一次 Esc，會在旋轉指示器列顯示約三秒的 `(esc again to interrupt)`；再按一次 Esc，就會顯示 `Canceled. What should Devin do?`，保留程序，並恢復空白輸入區，不重新填入草稿（已用 0.6 秒間隔驗證）。
已完成過回合的閒置 Agent，若兩次 Esc 間隔 0.05 或 0.1 秒，會開啟 `/revert` 選擇器，標題為 `Revert to step:`，頁尾為 `type search · ↑↓ select · ↵ revert · esc cancel`，其中 Enter 會還原檔案變更；間隔 0.15 秒或以上則不會開啟；按一次 Esc 可關閉它。
該取消動作不會觸發 `Stop` hook，因此控制層會使忙碌狀態失效，改為 unknown。
取消後 `--export` 也會更新，但不採用它作為狀態來源：僅憑檔案變更時間，無法把完成狀態綁定到剛送出的新回合。

閒置輸入區為 `❭ Ask Devin to build features, fix bugs, or work on your code`。
占位文字使用 RGB `124;124;124`，一般鍵入文字使用 RGB `255;255;255`。
繼承的 `NO_COLOR=1` 會移除兩者差異，因此啟動時會清除該環境變數，讓共用的樣式輸入區防護能正常運作。
`/quit` 使用共用的斜線指令彈出選單穩定處理，返回 shell，印出 `devin -r <session-id>`，並發出 `SessionEnd reason=prompt_input_exit`。
原生 `-r <session-id>` 接受新的提示詞，並保留先前對話。

<a id="worker-config-imports-and-attribution"></a>
## 工作 Agent 設定匯入與署名

使用修正前的每任務設定時，專案 `.claude/settings.json` 記錄器會在 `SessionStart`、`UserPromptSubmit` 與 `Stop` 觸發，使用者自己的 Claude Code `SessionStart` hooks 也會執行。
設定 `read_config_from.claude` 為 false 後，同一記錄器在 print 模式及真實驗證防護中都完全沒有觸發。
`attribution` 為 false 是 Devin 文件定義、控制 `Co-Authored-By` trailer 的開關；工作 Agent 提交不帶該 trailer，也沒有 `Generated with Devin` 列。
在此版本中，SWE-2 Medium 或 Claude Sonnet 5 Low 自行撰寫提交訊息時，未重現預設開啟的 trailer，因此強制設定該值是依據文件行為，而非觀察到的修復效果。

<a id="herdr-lab-session"></a>
## Herdr 實驗工作階段

一次真正的 `fm-spawn.sh --backend herdr` 偵察任務，在具名 Herdr 0.9.0 實驗工作階段執行，且保留使用者真正的 `~/.claude/settings.json` hooks，產生：

```text
agent get: {"agent":"devin","agent_status":"idle",...}
agent explain: manifest remote:.../agent-detection/remote/devin.toml, rule welcome_prompt_footer
fm_backend_agent_state: alive
interrupt (idle):  interrupt-delivered ... backend=herdr verified=agent-alive cancel=not-running
interrupt (busy):  interrupt-delivered ... backend=herdr verified=agent-alive cancel=unconfirmed
raw fast Esc pair: Revert to step picker open; exit refused; interrupt closed it; worktree unchanged
```

Herdr 依自己的畫面偵測 manifest 命名窗格；即使保留 Claude hook 匯入，它仍回報 `devin`，因此原先擔心誤標為 Claude 的問題沒有重現。
透過 `fm-send` 鍵入 `/no-mistakes`，會以斜線指令送出並載入技能。
工作 Agent 執行 `sleep 40` 期間再次 `fm-send`，沒有顯示取消，而且執行中指示與佇列中指示都完成了。
Herdr 上會拒絕 Devin 工作 Agent 的 `exit`：不依賴游標的輸入區分類器找到 `❭` 列，卻將其下方普通水平線讀為未配對的 Pi 分隔線，因此回覆 `unknown`。

<a id="coverage-and-limits"></a>
## 覆蓋範圍與限制

可攜式回歸測試涵蓋程序祖先證據、拒絕無關程序名稱、保留草稿、分別檢查兩個投遞訊號、設定保留與世代拒絕，以及僅限工作 Agent 的啟動、模型及 effort 處理。
控制層回歸測試涵蓋已進入可中斷狀態的第二次按鍵與最小間隔、閒置 Agent 的單次按鍵、關閉還原選擇器與拒絕 exit，以及保守的狀態失效處理。
遭拒的過期世代事件不會發出回合結束通知。
真實驗證防護檢查主回合完成、Claude hook 隔離、提交署名、通知訊號確認、閒置及忙碌中斷、還原選擇器、程序存活性、結束及原生恢復。
共用程序分類器向 tmux 與 Herdr 提供相同原生身分；Herdr 中斷、引導及身分已在實驗工作階段實際測試，而 Herdr `exit` 會如上述拒絕。
Zellij、Orca 與 cmux 只透過既有的後端無關投遞及按鍵能力介面檢視，未在此進行真實測試。
Orca 既有的不支援 Escape 投遞限制，代表 Devin 中斷在該處會被拒絕。
繞過 `fm-control` 直接用鍵盤取消，可能讓 busy 紀錄持續保留，直到正常完成或工作階段結束；這些工作 Agent hooks 不隱含任何主要監督保證。
