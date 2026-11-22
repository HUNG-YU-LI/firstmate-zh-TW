# Command Code

於 2026-10-05 在 Linux 使用 Command Code 1.74.1、模型 `deepseek/deepseek-v4.1-flash` 驗證。
介接器名稱是 Firstmate 解析的執行檔 `commandcode`；雖然也安裝 `cmd`，但名稱太通用，不適合當介接器名稱。
僅供船員／scout 使用的邊界由路由器負責；不支援主要工作階段與二副整合。
[驗證證據](../../../../../docs/verification/commandcode.md) 及其真實環境防護測試，負責更新下列供應商事實。

<a id="operating-facts"></a>
## 操作事實

| 項目 | 值 |
|---|---|
| 忙碌狀態 | 納入版本控制的 `../../../bin/fm-commandcode-mod.ts` 透過 `--mod` 按程序載入：`run_start` 開啟，`run_end` 關閉（包含 Escape 中斷，停止原因為 `interrupted`）；已完成回合還會讓寫入器在相同世代檢查下 touch `state/<id>.turn-ended`，工作階段結束也會關閉；信任由 `../../../bin/fm-busy-lib.sh` 負責。 |
| 結束指令 | `/quit`（`/exit` 是別名）；印出 `cmd --resume <session-id>`。 |
| 中斷 | 一次 Esc 取消進行中的回合，印出 `Interrupted · What should Command Code do instead?` 並留下空輸入區。閒置 Agent 約半秒內連按兩次 Esc，會開啟 Rewind 檢查點選擇器，在其中 Enter 會還原檢查點，因此控制平面只按一次；若選擇器已開啟，再按一次 Esc 關閉。 |
| 技能呼叫 | `/<skill>`，例如 `/no-mistakes`；Command Code 從 `~/.agents/skills` 與 `~/.commandcode/skills` 探索使用者技能，斜線選單以 `[skill]` 列出。 |
| 恢復 | `commandcode --resume <session-id>`（也可 `--session <id-prefix>`）重新開啟對話，但忽略同時傳入的訊息，因此下個提示需在輸入區讀為空後再輸入；`--model` 適用恢復的工作階段。 |
| 模型旗標 | `--model <model-id>`；權威清單為 `commandcode --list-models`，未列出的 ID 會讓啟動結束。 |
| 推理強度旗標 | `--effort <level>`，依模型而定；`deepseek/deepseek-v4.1-flash` 接受 `off`、`low`、`high` 與 `max`，不接受的值會讓啟動結束，因此 `../../../bin/fm-spawn.sh` 只傳入已驗證組合。 |
| 模型探索 | `commandcode --list-models`；驗證預檢為 `commandcode status`。 |
| 標記 | 無；程序標題是錨定名稱 `command-code`，用來識別介接器，優先於繼承的外來標記。 |
| 信任對話框 | `--trust` 略過全新 worktree 的資料夾信任提示，`--yolo` 略過權限提示。 |
| 提交署名 | Command Code 預設加入 `Co-authored-by: CommandCodeBot <noreply@commandcode.ai>`，且只有其設定檔能關閉；因此除非 home 設定 `config/keep-ai-trailers`（`../../../docs/configuration.md` 的「Commit attribution」），否則船隊 commit-msg 清除機制會移除它。 |

<a id="per-process-wiring-limits"></a>
## 逐程序銜接限制

`--config key=value` 不是逐程序設定：它會把設定持久寫入 `~/.commandcode/config.json`，因此 Firstmate 絕不傳入。
`settings.json` 的 hook 以檔案為範圍，且沒有 `UserPromptSubmit`，所以忙碌狀態由 mod 承載，不是 hook；使用者與專案 hook 仍在旁並行執行。
Taste learning 沒有逐程序開關，每次啟動都在專案植入 `.commandcode/taste/`，啟動程序會將其加入 worktree 的 Git exclude。

<a id="composer-and-steering"></a>
## 輸入區與指導

Command Code 在兩條分隔線之間繪製單純 `❯` 輸入區，把終端機游標停在頁尾下方，並繪製自己的反白游標格。
`Ask your question...` 提示文字的真彩色亮度足以超過船隊 ghost 上限，因此 `../../../bin/fm-composer-lib.sh` 對具有 Command Code 身分的畫面採用較高上限；輸入文字使用預設前景色，仍判為 pending。
啟動時清除 `NO_COLOR` 並固定 `COLORTERM=truecolor`，因為 256 色終端機會以未測試的調色盤顏色繪製提示文字。
`../../../bin/fm-tmux-lib.sh` 在 tmux 提供 `commandcode` 身分，Herdr 原生偵測則回報 `cmd`。
Zellij、cmux、orca、paseo 與 tern 不提供這類身分，因此提示文字在該處會被讀為 pending，`../../../bin/fm-spawn.sh` 會拒絕在這些後端啟動 Command Code。
傳遞忙碌訊號是 `esc to interrupt`，以及狀態列以圓點框出的經過時間格。

<a id="primary-integration"></a>
## 主要工作階段整合

Command Code 尚未驗證主要 Stop 防護、監看協定、工具執行前保護或工作階段啟動契約。
不要用此介接器啟動主要工作階段或二副。
配額供應商對應與型別化派工解析，仍是獨立後續工作。
