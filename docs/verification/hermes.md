<a id="hermes-tui-worker-verification"></a>
# Hermes TUI 工作 Agent 驗證

於 2026-11-02 在 Linux 上，以 `Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045` 與 tmux 3.6 驗證。
[配接器參考文件](../../.agents/skills/harness-adapters/references/harness/hermes.md) 定義操作限制；[spawn 標頭](../../bin/fm-spawn.sh) 定義具體的啟動與設定檔機制。
主要 Agent、二副（secondmate）、非 tmux 後端、桌面／網頁版、橋接與 Hermes 訊息功能均未測試，仍不受支援。

<a id="reusable-guards"></a>
## 可重複使用的防護測試

需要憑證的防護測試，請使用準備好的**可拋棄** Hermes 根目錄與設定檔。
它只會使用設定檔本身的推論模型與驗證資訊執行小型合成提示，不會傳送訊息給閘道。
防護測試的原生首次執行同意檢查，會使用另一份全新且不含憑證的設定檔。
兩個防護測試都會建立自己的私有 tmux 伺服器，不保留任何共用端點的擁有權。

```bash
FM_HERMES_SIGNALS_LIVE=1 \
FM_HERMES_LIVE_HOME="$throwaway_root" \
FM_HERMES_LIVE_PROFILE=worker \
bash bin/fm-test-run.sh tests/fm-hermes-signals-live-e2e.test.sh

FM_HERMES_GATEWAY_LIVE=1 \
FM_HERMES_LIVE_HOME="$throwaway_root" \
FM_HERMES_LIVE_PROFILE=worker \
bash bin/fm-test-run.sh tests/fm-hermes-gateway-coexistence-live-e2e.test.sh

FM_HERMES_PROFILE_SAFETY_LIVE=1 \
FM_HERMES_PYTHON="$installed_hermes_python" \
bash bin/fm-test-run.sh tests/fm-hermes-profile-safety-live-e2e.test.sh

bash bin/fm-test-run.sh tests/fm-hermes-harness.test.sh tests/fm-tmux-agent-liveness.test.sh
FM_TEST_STOCK_BASH=1 bash tests/fm-backend.test.sh
```

`FM_HERMES_PYTHON` 必須指定已安裝 Hermes 環境中的直譯器，讓原生設定檔安全防護測試匯入該版本自己的 API，而非以 Firstmate 的實作代替。
若單獨執行環境的直譯器無法匯入閘道模組，請將 `FM_HERMES_PYTHON` 指向一個小型包裝程式，透過 Hermes 自己的 `hermes_bootstrap` 執行它。
必要工具不可用時，共用的 `fm_live_gate` 會回報；明確啟用測試時，若缺少必要能力便會拒絕執行。

<a id="live-worker-observations"></a>
## 真實工作 Agent 的觀察結果

需要憑證的 tmux 防護測試，使用設定檔所設定的 Codex 模型得到以下結果：

```text
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: native first-run offer recognized, No thanks confirmed, TUI ready and decline persisted
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: interactive brief, native profile, tools, status, identity, busy and turn-end
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: real steering inbox read and acknowledgement
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: idle interrupt preserves process; double-Escape cancels a turn and native end settles
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: /quit and deterministic relaunch retain profile and task endpoint
```

實際的工作 Agent 算出 `80235`，偵測到自己的執行環境為 `hermes`，將任務狀態附加到工作樹外的檔案，並將真正持久保存的引導紀錄移入 `handled/`，以確認收到。
配置與初次啟動的準備階段使用隔離的測試資料；後續的 tmux 程序、忙碌事件、輸入區讀取、引導、插斷、結束與重新啟動都是真實操作。
spawn 的原生就緒閘門與重新啟動流程，都會採用配接器目前的 TUI／hook 實測資訊。

<a id="native-hooks"></a>
### 原生 hook

一個無害的 TUI 回合會發出 `pre_llm_call` 與 `on_session_end`，兩者都有 `extra.platform=tui`，且 `extra.turn_id` 相同。
正常完成與連按兩次 Escape 插斷，都會觸發原生結束事件，並關閉綁定世代的忙碌紀錄。
接收器會篩除其他平台，並比對原生回合 ID 而非工作階段 ID，因為壓縮可能會更換後者。
可攜式防護測試會驗證過期世代遭到拒絕，也驗證其他回合無法關閉目前的回合。
Hermes 會將 shell hook 指令解析為 argv，因此使用專用的可執行接收器指令，而非 shell 運算子。
私有的受管理覆蓋層會保留既有設定的 hook，並拒絕已存在的受管理範圍。
安裝 hook 時不會改寫使用者的 SOUL 或設定檔設定。

<a id="composer-and-lifecycle"></a>
### 輸入區與生命週期

實際閒置的設定檔輸入區，在原生狀態分隔線下方顯示 `worker ❯`。
忙碌時的提示文字為 `Ctrl+C to interrupt…`，擷取到的樣式如下：

```text
ESC[38;5;231m ESC[48;5;178m C ESC[38;5;178m ESC[49m trl+C to interrupt… ESC[39m
```

只會移除這個合成游標／提示序列；以一般明亮文字輸入的相同字詞仍視為草稿。
共用的畫面傳送防護也會辨識原生的每回合經過時間頁尾，不依賴插斷提示。
必須同時具有前景 Hermes 身分、原生狀態分隔線與正確提示位置，才能判定為設定檔輸入區。
預設設定檔不帶前綴的 `❯` 也符合通用的游標定位格式，因此 tmux 讀取會先檢查 Hermes 身分；可攜式 tmux 存活防護測試涵蓋該提示的閒置提示文字、原生忙碌提示、明亮草稿與缺少狀態分隔線的情況。
無法辨識的佈景主題或換行輸入會延後傳送，不會宣稱輸入區為空。
連按兩次 Escape 會取消執行中的回合，並保留閒置工作 Agent；它也會捨棄未送出的草稿，因此控制平面必須證實輸入區為空，才允許插斷。
`/quit` 會回到面板的 shell；確定性的重新啟動會保留設定檔、任務端點與本機工作樹。
未使用 Ctrl+C，因為原生閒置處理可能會結束 TUI。

<a id="foreground-process-liveness"></a>
### 前景程序存活狀態

可攜式 tmux 防護測試會執行真正的 `python3 -I -c` 啟動程序，並帶有同一程序群組中的 `node --expose-gc …/ui-tui/dist/entry.js` 子程序。
它會證實兩者都在面板的前景群組內，且兩者的程序名稱都無法辨識出 Hermes，接著要求後端探測回傳 `alive`。
不相關的 Node 指令碼仍屬於無法確定的情況；在對應路徑可用時，即使遺失核心 argv 或攤平引數其中一條取得路徑，肯定的判定仍會維持。
缺少／無法讀取輔助程式的防護測試，會在載入任何配接器前檢查全部五個相依後端，也包含原生 Bash 入口點。
2026-11-02，已安裝的 Hermes TUI 也通過不需 token 的原生設定檔安全防護測試，包含擴充的存活斷言：

```text
native TUI foreground liveness: alive
```

<a id="first-run-consent-and-query-path"></a>
### 首次執行同意與查詢路徑

原生 `hermes --help`、`hermes chat --help` 與已安裝的 `hermes_cli.observability.shared_metrics_consent`，定義了實際觀察到的啟動路徑。
全新設定檔的詢問畫面會顯示三個選項，且預先選取 `No thanks`；按 Enter 後記錄 `telemetry.shared_metrics.enabled=false`、`send=false` 與 `offer_version=2`，接著進入 TUI。
防護測試會透過 `hermes -p consent config get telemetry.shared_metrics --json` 讀回這些值。
標準工作 Agent 查詢通常會略過這個詢問；若它出現，啟動防護會處理完整的實測畫面。
此版本的 TUI 路徑不會送出 `--query-file`，而互動式 `-q` 會送出具型別的操作封套，並維持 TUI 可被操作。

<a id="native-profile-safety"></a>
## 原生設定檔安全性

原生設定檔安全防護測試產生以下結果：

```text
native gateway runtime lock: held
native TUI beside held gateway runtime lock: ready
native TUI foreground liveness: alive
two native session-store writers: distinct sessions retained
two native memory add writers: both entries retained
native gateway runtime lock: released
```

探測在同一份可拋棄設定檔中開啟 TUI 時，會持有 `gateway.status.acquire_gateway_runtime_lock()`。
接著由兩個獨立程序使用已安裝的 `SessionDB` 與 `MemoryStore.add` API，寫入不同工作階段 ID，並確認兩筆紀錄／項目都保留。
它只預先寫入可拋棄設定檔的拒絕遙測值；沒有訊息憑證，也不會啟動訊息閘道。
這證實了原生啟動可共存、不同工作階段可持久保存，以及已測試的新增記憶案例；尚未證實能安全地與真正處於回合中的閘道共用環境、寫入同一工作階段，或使用任意外部記憶提供者。

<a id="idle-gateway-coexistence"></a>
### 與閒置閘道共存

2026-11-02 的 05:40:45Z 到 05:43:51Z 之間，閘道共存防護測試為可拋棄設定檔啟動真正的 `hermes -p worker gateway run`，再於旁邊執行上述完整、需要憑證的 TUI 防護測試：

```text
idle gateway before: {"pid":208120,"gateway_state":"running","active_agents":0,"active_work":null,"updated_at":"2026-11-02T05:40:47.785255+00:00","session_store":{"status":"ok"}}
ok - real idle profile gateway accepted by the read-only preflight
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: native first-run offer recognized, No thanks confirmed, TUI ready and decline persisted
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: interactive brief, native profile, tools, status, identity, busy and turn-end
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: real steering inbox read and acknowledgement
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: idle interrupt preserves process; double-Escape cancels a turn and native end settles
ok - hermes Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: /quit and deterministic relaunch retain profile and task endpoint
idle gateway after: {"pid":208120,"gateway_state":"running","active_agents":0,"active_work":null,"updated_at":"2026-11-02T05:43:49.364081+00:00","session_store":{"status":"ok"}}
native gateway status: ✓ Gateway is running (PID: 208120)
session store: integrity ok, 7 prior sessions and their messages retained, 2 new worker sessions; memory unchanged
ok - Hermes Agent v0.21.7 (2026.9.24) · stable · local 71d34bad1045: TUI worker completed beside a real idle gateway; gateway, session store and memory intact
```

原生 Hermes 只有在設定 `gateway.standalone: true` 時，才會執行由設定檔擁有的閘道；防護測試會在可拋棄設定檔中設定此值，並在結束後還原原始設定。
閘道在已清空的環境中執行，使用私有的 `HOME`、`XDG_STATE_HOME` 與 `HERMES_GATEWAY_LOCK_DIR`，因此看不到繼承的平台憑證、操作者的 Hermes 根目錄或主機閘道會合點。
設定檔未設定任何訊息平台；若有設定，防護測試會拒絕執行。
spawn 與重新啟動的預檢都接受這個正在執行且閒置的閘道。
同一個閘道程序持續執行並維持閒置，心跳持續更新，原生 `gateway status` 仍回報它正在執行。
SQLite 儲存區通過 `integrity_check`，所有既有工作階段的訊息數均維持不變，工作 Agent 新增了自己的工作階段，且 `MEMORY.md` 的位元組完全相同。

因此，Firstmate 只有在設定檔所屬閘道的原生執行狀態顯示它正在執行、處於閒置，且心跳仍新鮮時，才會接受它。
這尚未驗證閘道回合與工作 Agent 回合重疊、寫入同一共用工作階段、平台流量或外部記憶提供者。
閘道可能在派工後開始一個回合，因此預檢並非整個生命週期的保留機制。
可攜式防護測試涵蓋閒置、回合中、排空中、過期與格式錯誤的狀態；systemd 活動與轉換；已持有與過期的原生鎖；預設根目錄、其他主目錄與自身主目錄的會合點；設定檔驗證；以及停止前拒絕重新啟動。

<a id="model-discovery-and-primary-boundary"></a>
## 模型探索與主要 Agent 邊界

已測試的原生介面包括 `hermes -p <profile> config get model.default --json`、`config get model.provider --json`、CLI 說明與實際的提供者回應。
Firstmate 將模型與推理選擇交由設定檔決定，不會將該帳號的模型目錄寫死。
主要 Agent hook、監看器協定、二副整合與 Herdr 生命週期皆未驗證；可執行的主要 Agent／二副／後端拒絕邏輯及其可攜式回歸測試，會落實此邊界。
