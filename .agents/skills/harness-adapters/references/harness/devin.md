# Devin CLI

於 2026-09-21 與 2026-09-22，使用 Devin CLI 3000.11.1（cc4e349ca55e）驗證。
僅供船員／scout 使用的邊界由路由器負責；不支援主要工作階段與二副整合。
[驗證證據](../../../../../docs/verification/devin.md) 及其真實環境防護測試，負責更新下列供應商事實。

<a id="operating-facts"></a>
## 操作事實

| 項目 | 值 |
|---|---|
| 忙碌狀態 | 原生 `UserPromptSubmit` 開啟，`Stop` 關閉正常完成，`SessionEnd` 透過綁定世代的寫入器關閉結束狀態；信任由 `../../../bin/fm-busy-lib.sh` 負責。 |
| 結束指令 | `/quit`，按 Enter 前先走共用斜線指令穩定等待；印出 `devin -r <session-id>`。 |
| 中斷 | 先按一次 Esc；只有執行中回合顯示 `esc again to interrupt`，且至少經過 0.5 秒，才按第二次；不恢復草稿，也不需清除按鍵。閒置 Agent 只按一次，回報 `cancel=not-running`，因為閒置時快速連按兩次會開啟 `/revert` 選擇器，在其中按 Enter 會回復檔案變更。 |
| 技能呼叫 | `/<skill>`，例如 `/no-mistakes`；Devin 從 `~/.agents/skills` 探索 Firstmate 使用者技能，`fm-send` 透過彈出選單穩定等待路徑輸入斜線形式。 |
| 恢復 | `devin -r <session-id>`；`--model` 可切換恢復工作階段的模型。 |
| 模型旗標 | `--model <model-id>`，包括 `swe-2-medium` 及帳號清單中的 `fusion-<lead>-sidekick-swe-2-medium` ID。 |
| 推理強度旗標 | 無；強度編入模型 ID，Firstmate 記錄獨立軸但不傳入。 |
| 模型探索 | `devin models list`；驗證預檢為 `devin auth status`。 |
| 標記 | 無；錨定的原生 `devin` 祖先鏈識別介接器，優先於繼承的外來標記。 |
| 信任對話框 | 啟動時略過本次工作區信任；精確旗標由啟動負責元件攜帶。 |
| 匯入設定 | 工作 Agent 設定將 `read_config_from.claude` 設為 false，因此不匯入 Claude Code hook、`CLAUDE.md` 規則、`.claude/skills` 或 Claude MCP 項目；`AGENTS.md` 與 `.agents/skills` 仍會載入。 |
| 提交署名 | 除非 home 設定 `config/keep-ai-trailers`（`../../../../../docs/configuration.md` 的「Commit attribution」），否則工作 Agent 設定將 `attribution` 設為 false；這是 Devin 控制 `Co-Authored-By` trailer 與 `Generated with Devin` 行的開關；有旗標時，保留使用者設定（預設開啟）。 |

<a id="worker-lifecycle-limits"></a>
## 工作 Agent 生命週期限制

已進入可中斷狀態後連按兩次 Esc，會顯示 `Canceled. What should Devin do?` 並恢復空輸入區，但此版本不發出 `Stop` hook。
因此控制平面把被中斷的執行世代設為 `unknown`，回報 `cancel=unconfirmed`；絕不因按鍵送達就捏造語意閒置。
控制平面以外的手動鍵盤取消，可能讓最後一筆 busy 紀錄保留到下次正常完成或工作階段結束。
已開啟的 revert 選擇器以一次 Esc 關閉，絕不能按 Enter；控制平面在自身按鍵後執行此處理，並拒絕在選擇器內輸入結束指令。
工具回應不作為主回合完成訊號。
Herdr 透過自身畫面偵測 manifest 原生識別 Devin 窗格，中斷與指導在該處可用；但 Herdr 上拒絕 `exit`，因此也拒絕 `relaunch`，因為其不依游標的輸入區讀取，對 Devin 畫面回報 `unknown`。

自主權、信任、型別化簡報傳遞、色彩保留，以及省略 Claude 權限模式對應，由 `../../../../../bin/fm-spawn.sh` 負責。
私人使用者設定快照與追加生命週期 hook，由 `../../../../../bin/fm-devin-config.sh` 負責；使用者與專案設定仍由供應商管理。
設定快照可能含私人設定，檔案權限為 600。

<a id="composer-and-steering"></a>
## 輸入區與指導

已驗證的 `❭` 符號、暗色閒置提示文字、執行中回合輸入區與中斷提示，以 `../../../../../bin/fm-composer-lib.sh` 為準。
共用傳遞路徑必須保留 ANSI 樣式：帶樣式擷取後仍存在、看似提示文字的內容，仍是草稿，不得覆寫。
`../../../../../bin/fm-task-inbox-lib.sh` 的門鈴，已在 SWE-2 與 Fusion 上透過真實 `fm-send` 成功讀取並確認。
共用斜線指令穩定等待路徑，也處理 `/quit` 自動完成。

<a id="primary-integration"></a>
## 主要工作階段整合

Devin 尚未驗證主要 Stop 防護、監看協定、工具執行前保護或工作階段啟動契約。
不要用此介接器啟動主要工作階段或二副。
ACP、配額供應商整合與原生 Fusion 子 Agent 計帳，仍是獨立後續工作。
