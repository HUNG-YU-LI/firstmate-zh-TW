# Muse Code

於 2026-08-05 使用 Muse Code 0.1.0-R708.1、建置 SHA 427a430436 驗證。
Muse 的工作類型邊界由路由器負責。

<a id="operating-facts"></a>
## 操作事實

| 項目 | 值 |
|---|---|
| 執行檔 | 從 `PATH` 解析絕對 `muse` 路徑，不存在就拒絕；啟動器 `~/.local/bin/muse` 會 exec 具版本的 `muse-bin-<version>`，因此更新後活躍程序名稱會改變。 |
| 啟動 | 以位置參數傳入指示，和 Grok 或 Pi 相同。 |
| 模型 | `--model <model>`；只有供應商 `meta`。 |
| 忙碌 | 由 `../../../bin/fm-busy-lib.sh` 摺疊持久化工作階段事件日誌；沒有 hook 或外掛寫入器，不需啟用或植入 busy 紀錄。 |
| 結束 | `/exit`，一次 Enter；印出 `To continue this session, run muse resume <session-uuid>`。 |
| 中斷 | 一次 Escape 記錄 `terminal: cancelled`，並恢復亮色提示文字，因此控制隨後送出 `Ctrl+U`；舊型別化按鍵路徑也使用相同清除表。 |
| 技能 | `/<skill>`，與 Claude 或 Grok 相同形式。 |
| 恢復 | `muse resume --last` 或 `muse resume <session-uuid>`；單獨 `muse resume` 會開啟選擇器。 |
| 自主權 | `--yolo` 關閉核准與沙箱，並信任工作區。 |
| 信任 | 對話框 `Do you trust this workspace?`，預選 `1 Trust and continue`，按 Enter 即可；`--yolo` 會抑制它，全新工作路徑需要此旗標。 |
| 標記 | 無；身分來自錨定的 `muse-bin-*` 祖先鏈，`../../../bin/fm-harness.sh` 會防止保留的外來標記覆寫它；`MUSE_CURRENT_SESSION_LOG` 是路徑，不是身分，且是否匯出給工具尚未驗證。 |
| 輸入區 | 有邊框的 `⟩`，真彩色 `38;2;90;160;255`，亮度約 149.9，略高於 ghost 門檻 128；輸入文字為 `38;2;204;211;219`，約 209.8，未觀察到提示文字或 ghost。 |
| 推理強度 | `--reasoning-effort`，預設 `high`，接受 `none\|minimal\|low\|medium\|high\|xhigh\|ultra`；共用值提供 low 到 xhigh，船長明確指定 `max` 對應 `ultra`，`none` 與 `minimal` 維持不可選。 |

<a id="credential-preflight"></a>
## 憑證預檢

Muse 讀取優先生效的 `META_API_KEY`，或由 OIDC 裝置碼 `muse login`／`muse auth set --api-key-stdin` 寫入的 `${XDG_CONFIG_HOME:-$HOME/.config}/muse/auth.json`。
啟動只有在後端工作 Agent 已持有環境金鑰時才接受；僅呼叫端的變數不會跨越長期常駐程序，機密也絕不進入 argv。
儲存的憑證是受支援的船隊路徑。
預檢與轉送前，它會將非機密的 `XDG_CONFIG_HOME` 與 `XDG_DATA_HOME` 解析成絕對路徑，維持驗證與日誌一致。

工作 Agent 可存取的兩種憑證都沒有時，啟動拒絕。
否則未驗證 Muse 會永遠停在 `Sign in at this page: https://auth.meta.com/oauth/device/?code=XXXX-XXXX` 與 `Waiting for approval…`，看起來像卡死。
將拒絕呈報為憑證需求前，先檢查[工作 Agent 啟動環境契約](../../../../../docs/configuration.md#worker-launch-environment-configlaunch-env-allowlist)，確認是否有尚未放行的環境授權。

<a id="foreign-personal-context"></a>
## 外來個人上下文

Muse 每次執行都會把 `~/.claude` 的操作者規則送到 Meta 代管推論。
通知會提到 Claude 個人規則與 `/settings`，但透過 `tui.foreign_context_notice_shown` 只顯示一次，因此之後沒有通知不代表沒有載入；隔離 `XDG_CONFIG_HOME` 也無法阻止。

互動式 Muse 拒絕只適用 exec 的 `--no-foreign-personal-context`。
窗格模式的控制為 `MUSE_EXPERIMENTAL_FOREIGN_PERSONAL_CONTEXT_KILL=on`，每次啟動都設定，且已驗證可移除外來 `rules_file`，同時保留專案 `AGENTS.md`。

<a id="session-event-log"></a>
## 工作階段事件日誌

日誌位於 `${XDG_DATA_HOME:-$HOME/.local/share}/muse/sessions/YYYY/MM/DD/<session-uuid>/session.jsonl`。
啟動程序寫入 `state/<id>.muse-session`，包含根目錄、worktree、綁定執行世代與已存在且符合的主要日誌；接著以唯一解析固定 `state/<id>.muse-session-current`。
只要有界的當日主要命名空間不變，就摺疊該路徑；命名空間改變、路徑消失，或較新綁定優先時，再重新解析。

回合由 `{"payload":{"kind":"run","run_id":"<uuid>","event":{"kind":"started"` 與對應 `"event":{"kind":"terminal"` 夾住，已觀察的結束狀態為 `completed` 或 `cancelled`。
因此不同於 Claude Stop，中斷有真正的 terminal 事件。
絕不使用 `--no-session-log`，因為它會移除 Muse 唯一的忙碌來源。

摺疊必須拒絕巢狀 `"record":{"kind":"terminal"}` 清理副作用，並以深度邊界排除 `subagent/<child-session-id>/session.jsonl` 下的原生子 Agent 日誌。
已記錄、解析後的 `XDG_DATA_HOME` 也轉送給工作 Agent，維持與常駐程序一致。
開啟的 run 是可信 busy，已結束的日誌是可信 idle；缺少綁定或符合項目、日誌無法讀取，或沒有 run 的日誌，都屬 unknown。
帶憑證的 idle 證據與更新，以 `../../../docs/verification/muse.md` 為準。

<a id="native-sub-agents-and-worktrees"></a>
## 原生子 Agent 與 worktree

只有選擇啟用 `--subagent-worktree-isolation` 時，原生子 Agent 才使用各自 worktree；能力宣告說預設開啟，但省略旗標時仍共用，已驗證實驗也未產生巢狀副本。
`../../../bin/fm-teardown.sh` 不排除任何 Muse 路徑。
它排除 `.claude/settings.local.json`，因為該檔由 Firstmate 寫入；Muse 暫存內容則是工作 Agent 產出，未提交時必須拒絕清理。
遇到拒絕應檢查，絕不強制繞過。

<a id="maturity-and-primary-limit"></a>
## 成熟度與主要工作階段限制

Muse 0.1.0 是首日 beta；每小時的更新通道輪詢可能替換執行檔與程序名稱。
船長已接受此點，因此 Firstmate 不設定 `MUSE_NO_AUTO_UPDATE=1`；船隊可自行設定，不需修改介接器。
除非 `MUSE_EXPERIMENTAL_PLUGINS=on`，否則外掛回報不可用，因此忙碌狀態使用日誌。
相容性方言明確缺少 `asyncRewake` 與模型重新喚醒；由此產生的主要工作階段邊界由路由器負責。
