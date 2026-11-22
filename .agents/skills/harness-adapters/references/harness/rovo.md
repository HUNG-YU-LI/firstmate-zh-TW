# Rovo CLI

於 2026-09-02 使用 Rovo CLI 202609.1.2 驗證，僅限 crewmate/scout 工作。
未驗證為二副或主 Agent，且現有機制也無法自然驗證：rovo 沒有回合結束 hook，也沒有主 Agent 監督協定，與 muse 因相同缺口而僅限 crewmate/scout 一致。

<a id="operating-facts"></a>
## 操作事實

| 事實 | 值 |
|---|---|
| 執行檔 | `../../../bin/fm-spawn.sh` 的 `resolve_rovo_binary` 先解析 `PATH`，再回退到 `$HOME/.local/bin/rovo`；兩者皆不可執行就拒絕 spawn。 |
| 啟動 | 空白 `rovo run --yolo`（不帶位置參數簡報），採 kimi 先啟動再傳送形式：以 `Welcome to Rovo!` 橫幅為就緒關卡，接著輸入絕對簡報指標，再經送達確認關卡。位置參數簡報一開始就無法運作（見下方「啟動與就緒」）。 |
| 模型 | `--model <model>`，從工作階段內 `/models` 命令或 ACP `session/new` 探索；實際觀察清單（GPT-5.6 Terra/Sol/Luna、GPT-5.5、GPT-5.4、數個 Claude Sonnet/Opus/Haiku ID、Gemini 3 ID）依帳號而異，絕不可寫死。 |
| 忙碌狀態 | 畫面尾端備援，像 Grok 一樣僅限 rovo：動畫 `Rovo is thinking...` 行，由 `../../../bin/fm-busy-lib.sh` 的 `fm_busy_rovo_tail_busy` 比對；因為 rovo 的 `eventHooks` 只在工具粒度觸發（`on_tool_start`/`on_tool_end`），絕不在回合結束觸發，所以沒有可啟用的語意寫入器。 |
| 結束命令 | `/exit`（也可 `/quit`，或閒置時一次 Ctrl-C）；印出 `Run rovo --restore <session-id> to resume your conversation`。 |
| 中斷 | 一次 Escape 是取消鍵，會印出 `Agent cancelled`；`../../../bin/fm-control-lib.sh` 將其確認來源記為 `none`（見下方「中斷：已在真正 tmux 確認」），與 claude/codex/grok/kimi/cursor 採相同保守選擇。 |
| 技能呼叫 | `/<skill>`，Claude/Grok 形式，但見下方「技能載入互通性缺口」：缺口解決前，rovo 工作 Agent 無法呼叫 firstmate 技能。 |
| 自主性 | `--disable-permission-checks`（別名 `--yolo`）會在不確認下執行每個檔案 CRUD 操作與 bash 命令，但其自身印出的注意事項仍保留存取 Atlassian 資料與使用者提供 MCP 伺服器工具的權限檢查；crew/scout 任務不碰這些工具。 |
| 檔案存取 | rovo 預設將所有檔案工具操作限制在啟動 worktree，因此標準指示／引導／狀態／報告循環的檔案位於 worktree 外的 firstmate home，未授權前會失敗。`../../../bin/fm-spawn.sh` 的 `rovo_config_override_flag` 在啟動時授予 `toolPermissions.allowedExternalPaths`，摺入單一 `--config-override`（見推理強度），範圍精確限定為此任務的簡報目錄、引導收件匣與狀態檔。授權只放寬檔案工具；rovo 的 bash 工具仍不論如何都受限於 worktree，因此船員狀態行的 `echo ... >> status` 之所以能落地，是工作 Agent 改用自己的檔案工具附加。見 `../../../../docs/verification/rovo.md`。 |
| 信任對話框 | 在新 worktree 乾淨啟動時未觀察到；`--yolo` 清除 crew/scout 的確認提示，但標準流程不只需要這項啟動授權，必要的 `allowedExternalPaths` 授權見檔案存取。 |
| 環境標記 | `ATLASSIAN_AGENT_TYPE=rovo`（最精確）與 `ROVODEV_CLI=1`，兩者與 `AGENT=rovodev_cli` 一起設在 rovo 工具子程序；rovo 不會清除繼承的 `CLAUDECODE`/`CURSOR_AGENT` 等，因此 `../../../bin/fm-harness.sh` 在 `CLAUDECODE` 行前測試 rovo 標記（cursor 已記錄相同順序風險，issue #3517），`../../../bin/fm-spawn.sh` 也會在啟動邊界清除其他標記。 |
| 程序名稱 | 工具子程序與 `rovo run` 程序本身都是 `comm=rovo`，因為已安裝包裝器會 exec 該世代的 `rovo` shim，即使磁碟執行檔為 `atlassian_cli_rovodev`，argv[0] 仍維持 `rovo`。 |
| 輸入區 | `../../../bin/fm-composer-lib.sh` 已可讀取的既有有框線 `box` 形狀家族（`╭─╮ │ │ ╰─╯`）；空輸入區顯示淡化建議標籤與 `? for shortcuts.` 提示，忙碌頁尾為 `Enter to queue, Ctrl+Enter to steer`。 |
| 推理強度 | `agent.efficiencyLevel`，接受 `low\|medium\|high\|max`（預設 `medium`，沒有 CLI `--effort` 旗標），透過 rovo 單一 `--config-override` 旗標即時設定；摺入與必要 `allowedExternalPaths` 授權**相同**的 JSON 物件，絕不輸出成獨立 override，因為 `--config-override` 只接受單值（見 `../../../../docs/verification/rovo.md`）。`xhigh` 要求會記錄於任務中繼資料，但依 `../../../references/common/model-and-effort.md` 的記錄並省略契約，從該物件省略，因為 rovo 沒有 `xhigh`。 |

<a id="detection"></a>
## 偵測

`../../../bin/fm-harness.sh` 在 `CLAUDECODE` 行前檢查 `ATLASSIAN_AGENT_TYPE=rovo` 與 `ROVODEV_CLI=1`，再回退到祖先鏈（`rovo)` case，位於 `kimi)` 旁）。
兩層都重要，原因與 cursor 相同：標記順序涵蓋人員在繼承其他標記的環境下手動啟動的 rovo 工作階段，而 `../../../bin/fm-spawn.sh` 啟動邊界的 `env -u` 清理，不受順序影響地涵蓋每個 firstmate 啟動的工作 Agent。

<a id="launch-and-readiness"></a>
## 啟動與就緒

啟動範本會行內清除 `CLAUDECODE`、`PI_CODING_AGENT`、`GROK_AGENT` 與 `FM_PI_HARNESS`（rovo 自己暴露的其他標記），共用外層包裝則像其他非 cursor 執行環境一樣，清除 `CURSOR_AGENT`/`CURSOR_INVOKED_AS`。
rovo **空白**啟動（`rovo run --yolo`，加上所需 `--model`/`--config-override` 旗標），只在 TUI 啟動後才接收簡報；與 kimi 相同的先啟動再傳送形式，透過同一組共用讀取器（`fm_backend_capture`、`fm_backend_composer_state`、`fm_backend_send_text_submit`）接線：

1. **就緒關卡**（`../../../bin/fm-spawn.sh` 的 `rovo_wait_for_ready`）：輪詢全新啟動的 `Welcome to Rovo!` ASCII 橫幅，再回退到輸入區為空。橫幅是主要訊號，因為輸入區為空的備援在 rovo 比 kimi 更弱：rovo 閒置輸入區繪製行內預留標籤，其亮度高於幽靈文字移除門檻（見下方「輸入區幽靈文字」），因此可能讀成非空。
2. **輸入指標**：`Read the brief at <absolute-path> and follow it exactly.`，透過 `fm_backend_send_text_submit` 送出（用詞與機制與 kimi 完全相同）。
3. **送達關卡**（`rovo_wait_for_delivery`）：輸入區為空，**且**回顯指標文字（`Read the brief at`）已捲入可視範圍，或 rovo 的 `Context:` 頁尾百分比已離開零。rovo 真正頁尾是 `Context: <bar> N.N% NN.NK/NNNK`（例如 `Context: ▎ 3.3% 30.1K/922K`）；送達正規表示式容忍條狀符號與任意空白，但錨定 `%` 前的數字，讓始終非零的分母（`.../922K`）絕不會假裝成使用量。

位置參數簡報一開始就無法運作：`rovo run --yolo "<brief>"` 會載入，但從未進入工作狀態，約 10–15 秒內就回到閒置 shell；已透過原始 PTY 獨立確認四次，並在真正 tmux 3.6a，以精確 `fm-spawn.sh` send-keys 形式確認一次。`--startup-receipt` 也救不了此形式：它要求「prompt-free interactive mode」（`Invalid value: --startup-receipt requires prompt-free interactive mode in a terminal`），因此無法為之後會輸入訊息的啟動設關卡。相對地，先啟動再傳送已完成實際端到端驗證（空白啟動 -> `Welcome to Rovo!` -> 輸入指標 -> 真正 bash 工具呼叫出現 `Rovo is thinking` -> 乾淨 `/exit`）；見 `../../../../docs/verification/rovo.md`。
rovo 完全不留下 worktree 內成品或 firstmate 擁有的 sidecar，也沒有可記錄的就緒回執或 session-id。

<a id="composer-ghost-text-a-known-unfixed-gap"></a>
## 輸入區幽靈文字：已知、尚未修復的缺口

rovo 空輸入區會直接在有框線內容列內繪製行內預留標籤（例如 `Summarize my open tasks`），不只是下方獨立建議清單。
實際測得該預留文字前景色為 `38;2;162;163;165`（亮度約 163），同一方框的真實輸入文字則為 `38;2;206;207;210`（亮度約 207）；差距確實存在，但兩者都高於 `../../../bin/fm-composer-lib.sh` 預設 `FM_COMPOSER_GHOST_LUMA_MAX` 128，因此 `fm_composer_strip_ghost` 不會移除它，全新 rovo 輸入區可能被誤判為 `pending` 而非 `empty`。
提高共用預設門檻來涵蓋它並不安全：muse 自己真實、不可移除的提示符號亮度約 149.9，低於 rovo 幽靈文字，因此沒有單一全域門檻能同時保留 muse 真實符號並移除 rovo 幽靈標籤。
此問題刻意暫不修復，以免透過改門檻冒險破壞 muse 已驗證行為；真正修正需要以執行環境為範圍的訊號，而共用輸入區分類器目前沒有。
實際影響限定於依賴輸入區為空的使用端：對閒置 rovo 窗格引導可能得到非空判定，因而循一般門鈴階梯重試，無法第一次就送達。
它不阻擋先啟動再傳送關卡：就緒優先使用 `Welcome to Rovo!` 橫幅（不是輸入區為空）；送達關卡雖確實要求輸入區為空作為其中一項合取條件，但它在 rovo 正在處理剛送達簡報時執行，而預留標籤只在閒置休息時繪製，回合中不會，所以送達時間窗內輸入區確實讀為空。

<a id="interrupt-confirmed-under-real-tmux"></a>
## 中斷：已在真正 tmux 確認

原始驗證偵察（`fm-rovo-smoke-s1`，PTY 冒煙測試）觀察到執行工具呼叫時，一次 Escape 會印出 `Agent cancelled`。
之後在真正 tmux 3.6a 的實際檢查，使用隔離的 `tmux -L <private-socket>` 工作階段／視窗，而非共用艦隊工作階段，重現偵察的精確發現：真正 bash 工具呼叫執行中送一次 Escape，擷取窗格會印出 `Agent cancelled`。
先啟動再傳送的即時防護檢查（`../../../../tests/fm-rovo-signals-live-e2e.test.sh`）現在也會在原始 PTY 重現：早期單一固定計時 Escape 不可靠（裸 PTY 的中斷時點對時序敏感），因此防護檢查會在實際工具呼叫時間窗持續送 Escape，直到顯示取消；這是可確定重現時序敏感中斷的方法，且已確認每次都印出 `Agent cancelled`。
Escape 是中斷鍵，也是 `fm_control_interrupt_key` 的回傳值。
`fm_control_interrupt_ack_source` 仍為 rovo 記錄 `none`，與 claude/codex/grok/kimi/cursor 已採相同保守選擇；這是控制平面事實，與畫面是否恰巧顯示無關，因此控制平面送鍵後，由自己的後置條件判定 Agent 是否確實停止，不靠解析字串。
中斷按鍵與畫面證據現在已完整互相印證，與程式碼沒有矛盾。

<a id="oauth-token-lifetime"></a>
## OAuth 權杖有效期

存取權杖約一小時有效，但 `rovo` 會用儲存的更新權杖（有效期約四週）靜默、非互動地更新，沒有瀏覽器提示，也沒有可見中斷；這是船長修正過的既定指引，不是本任務自行發現，但本任務的實際檢查也以實證印證：`rovo auth status` 顯示 `Access token expired ... but a refresh token is present`，接著一般 `rovo run` 成功完成，後續 `rovo auth status` 顯示剛更新的有效權杖，中間沒有任何互動步驟。
應將約 1 小時存取權杖有效期視為一般操作事實，不是不可妥協的安全阻擋；rovo 工作 Agent 不需要縮短任務範圍才能跨過它。
只有大約四週未使用，或更新權杖本身失效後，才需要 `rovo auth login`（互動瀏覽器 OAuth）。

<a id="skill-loading-interop-gap"></a>
## 技能載入互通性缺口

rovo 技能載入器拒絕每個 firstmate 技能：`Invalid skill definition in .../SKILL.md: 'metadata -> internal': Input should be a valid string`，因為 firstmate 的 `metadata.internal` 是布林值，rovo 結構描述則要求字串。
在 firstmate 的 `SKILL.md` frontmatter 改為相容 rovo 之前，這會阻擋 rovo 工作 Agent 內的 `/no-mistakes` 與其他所有 firstmate 技能呼叫；這是另一項延後處理的工作，依 `../../firstmate-coding-guidelines/SKILL.md`，會涉及每個技能檔與安裝器契約。
`no-mistakes` 模式的 rovo ship 船員會被此缺口阻擋；不呼叫技能的 rovo scout 不受影響。

<a id="acp-as-a-future-upgrade"></a>
## ACP 作為未來升級

`rovo acp`（Agent Client Protocol）與 `rovo serve --non-interactive` 提供完整結構化、機器可讀的回合生命週期：`session/prompt` 回傳真正的 `{"stopReason":"end_turn"}`，`session/cancel` 是協定原生中斷。
這比目前任何配接器都有更乾淨的完成訊號，但使用它代表 firstmate 必須執行 JSON-RPC 用戶端，並自行負責工作階段生命週期；這是新的後端形式介面，不是可直接替換的 TUI 配接器，因此不在此範圍。
它仍是未來將 rovo 作為結構化後端的刻意升級方向，不是近期路徑；不要在這個 TUI 路徑配接器中建置它。
