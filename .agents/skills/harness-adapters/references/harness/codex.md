# Codex

除個別事實註明較新版本外，於 2026-06-11 使用 codex-cli 0.139.0 驗證。

<a id="operating-facts"></a>
## 操作事實

以下命令列與終端機事實適用窗格式工作階段；T3 Code 請閱讀[原生生命週期](../../../../../docs/t3code-backend.md#current-lifecycle-and-safety) 與[供應商指令限制](../../../../../docs/t3code-backend.md#active-limits)。

| 項目 | 值 |
|---|---|
| 忙碌狀態 | 語意來源經真實驗證前皆為 unknown：窗格工作 Agent 無法存取 app-server 回合生命週期，而 Firstmate 啟動的工作 Agent 也未觸發專案生命週期 hook。 |
| 結束指令 | `/quit`；斜線彈出選單需要在輸入文字與 Enter 之間等待約一秒，由控制平面使用的共用提交路徑處理。 |
| 中斷 | 一次 Escape。 |
| 技能呼叫 | `$<skill>`，例如 `$no-mistakes`；`/<skill>` 是 Claude 形式，Codex 會以「Unrecognized command」拒絕。 |
| 恢復 | `codex resume <session-id>`，使用結束時印出的 ID。 |
| 模型旗標 | `--model <model>`。 |
| 推理強度旗標 | `-c 'model_reasoning_effort="<low\|medium\|high\|xhigh\|max>"'`；已於 codex-cli 0.142.1 驗證，已安裝結構含 `model_reasoning_effort`，有效設定也使用它；隨附目錄只列前四個值、未列 `max`。目前 codex-cli 0.153.4 位於 `${CODEX_HOME:-~/.codex}/models_cache.json` 的目錄資料，為 `gpt-5.6-luna` 宣告 `max`，Firstmate 會對該模型傳入。 |
| 模型探索 | 開啟目前互動工作階段的 `/model` 選擇器。 |
| 核准模式 | 請見 [Codex 自動核准審查](../../../../../docs/configuration.md#codex-automatic-approval-review) 與[持久化備援切換](../../../../../docs/configuration.md#persistent-managed-policy-failover)。 |
| 標記 | 無；身分來自祖先鏈，`../../../bin/fm-harness.sh` 會避免保留的外來 `CLAUDECODE` 將它誤識別。2026-09-01 使用 codex-cli 0.152.0 驗證：窗格程序是 `node` npm shim，原生 `codex` 執行檔作為其前景子程序，因此工具子程序可直接到達原生名稱，而 shim 本身則從指令碼路徑識別。 |

某儲存庫根目錄首次執行時，會出現目錄信任對話框：「Do you trust the contents of this directory?」。
按 Enter 接受，並確認指示開始處理。
決策按儲存庫保存，因此同專案後續 worktree 會略過。

<a id="hook-trust"></a>
## Hook 信任

只要機器的 `~/.codex/hooks.json` 或專案自己的 `.codex/hooks.json` 含有 Codex 尚未保存信任的 hook，就會出現第二個對話框：「Hooks need review - N hooks are new or changed」。
它不只是麻煩，而是目前無法回答：選擇停在「Review hooks」，既非信任也非拒絕；Firstmate 按鍵平面只提供 Enter、Escape 與 Ctrl-C，沒有方向鍵導覽。
預先寫入 Codex 自己的信任儲存區，相當於捏造操作者從未給予的同意。
因此窗格式船員與 scout 啟動會要求 Codex 關閉 hook 層（旗標以 `bin/fm-spawn.sh` 啟動範本為準）；這與 `--dangerously-bypass-hook-trust` 相反，後者會**執行**未信任 hook。
受管理政策可覆寫要求的停用，因此工作 Agent 仍可能遇到政策提供的 hook 或其審查；Firstmate 不修改操作者的 Codex 設定來繞過任何結果。
船員不會因此失去功能：回合結束訊號來自同次啟動的 `-c notify=` 程式；專案 `.codex/hooks.json` 中的 Firstmate hook 是主要工作階段基礎設施，在子 worktree 會停止介入。
二副在自己 home 中是主要工作階段，會保留 hook，因此仍可能出現無法回答的視窗，需由操作者自行完成 hook 審查。

<a id="skill-popup"></a>
## 技能彈出選單

`$<skill>` 呼叫會開啟 `$` 自動完成選單。
提交過快會讓選單吃掉 Enter，使呼叫未真正送出。
窗格工作階段中，只有精確工作中繼資料記錄 `harness=codex` 時，`../../../bin/fm-send.sh` 才會對前導 `$` 在第一次 Enter 前等待 1.2 秒，再以目標後端的提交重試作安全網。
這個範圍限制很重要，因為一般文字也常以 `$` 開頭，例如 `$5/month` 或 `$HOME`。
明確的 `session:window` 目標沒有中繼資料，因此執行環境未知，使用非 Codex 的快速路徑。
這就是 `$no-mistakes` 能送達 Codex 工作 Agent，而不被彈出選單吃掉的原因。

<a id="primary-integration"></a>
## 主要工作階段整合

主要 Stop 包裝器的範圍、回應串流與有界接續，以 [`turnend-guard.md`](../../../../../docs/turnend-guard.md#harness-integrations) 為準；目前解析器證據由[執行階段後端驗證](../../../../../docs/verification/runtime-backends.md#stop-response-parsing) 負責。

2026-07-08 使用 codex-cli 0.142.1 驗證：Codex 執行 Stop 指令時，程序 PWD 為載入 hook 的專案根目錄，但未設定 `CODEX_PROJECT_DIR`、`CODEX_WORKSPACE_ROOT` 或 `CODEX_CWD` 等根目錄變數。

Codex CLI 0.163.0 支援由 `bin/fm-codex-stop-park.sh` 登錄的原生非同步 Stop 停駐，home 需以 [`config/codex-stop-park`](../../../../../docs/configuration.md#codex-native-stop-park-configcodex-stop-park) 選擇啟用；此行為於 2026-10-19 在隔離的真實 CLI 實驗環境驗證。
Hook 在模型之外等待，並透過 `codex queue --thread` 傳遞需要處理的喚醒，也包含沒有共用 app-server 常駐程序的嵌入模式。
版本與原生工作階段選擇由指令碼負責；操作協定與失敗替代路徑，以 [`codex.md`](../../../../../docs/supervision-protocols/codex.md) 為準。
未啟用、版本未驗證，或沒有原生登錄的工作階段，維持 [`codex-checkpoint.md`](../../../../../docs/supervision-protocols/codex-checkpoint.md) 中不變的有界前景協定。
[監督主機](../../../../../docs/supervision-host.md) 在兩條路徑都保留既有模式決策。
Codex 的 PreToolUse 監看啟用防護，直接透過專案 hook 阻擋。
