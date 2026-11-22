<a id="polytoken-cli-worker-verification"></a>
# Polytoken CLI 工作 Agent 驗證

讀者：維護者驗證。

[轉接器參考文件](../../.agents/skills/harness-adapters/references/harness/polytoken.md) 負責操作事實；負責執行的程式則承載啟動與狀態機制。
轉接器會依已安裝執行檔選擇能力，沒有 Polytoken 版本關卡。
下列發行版本號用來識別證據，不代表需求。
本紀錄涵蓋 tmux TUI 窗格加 REST 控制路徑上的船員（crewmate）與偵察員（scout）。
主要 Agent、二副，以及原生 Herdr 窗格配置，不在此保證範圍內。

<a id="refresh-commands"></a>
## 更新驗證的指令

```sh
polytoken --version
polytoken new --help
polytoken print models
polytoken models --format json
polytoken print openapi
bash bin/fm-test-run.sh tests/fm-polytoken-harness.test.sh
FM_POLYTOKEN_SIGNALS_LIVE=1 bash bin/fm-test-run.sh tests/fm-polytoken-signals-live-e2e.test.sh
```

使用憑證的驗證防護，會使用有標記、可拋棄的 `FM_HOME`，以及 `bin/fm-lab-home.sh` 提供的私人 tmux socket 目錄。
它會準備一份不含操作者 MCP 服務的供應商／模型設定副本，並將操作者既有的授權接受紀錄複製到隔離的 HOME。
不會建立或延伸任何授權接受紀錄，也不會回應信任或授權對話框。
所有 Polytoken 工作階段、設定、快取及日誌寫入，都留在該實驗環境。
驗證防護使用暫存複本與 `treehouse` 替身，不接觸共用 worktree 資源池。
它涵蓋的真實 Firstmate 操作包括 spawn、持久收件匣投遞與確認、interrupt、relaunch、exit 及 teardown。
失敗訊息會指出已安裝的 Polytoken 版本。

<a id="session-model-proof"></a>
## 工作階段模型證據

2026-10-14，在 macOS arm64、`polytoken 0.9.0` 上，一次標準派生使用了 `--model zai/glm-5.3-flash --effort low`。
常駐程序的 `GET /state` 回傳以下欄位投影：

```json
{"active_model":"zai/glm-5.3-flash","active_reasoning_effort":"low","active_facet":"firstmate-worker"}
```

保留的工作階段日誌，在使用者提示詞之前記錄了一次模型選擇：

```json
{"index":1,"selection":{"from_model":"anthropic/claude-opus-5-5","to_model":"zai/glm-5.3-flash","reasoning_effort":"low","assignments":{},"effective_options":{},"nondefault_explicit":{},"quarantined":false,"created_seq":2}}
```

此執行檔的工作階段清單，以及 `session.json` 的初始模型，仍描述啟動預設值。
它們不能證明明確啟動覆寫之後實際使用的模型。
真實驗證防護檢查的是常駐程序狀態與保留的選擇紀錄，不只是 TUI 頁尾。
目錄為 `zai/glm-5.3-flash` 提供 `low`、`high` 與 `max`，為 `ogo/deepseek-v4.1-flash` 提供 `low`、`medium` 與 `high`。
可攜式回歸測試驗證最接近等級的對應，並確認目錄無法取得或基礎模型未列出時會拒絕。

<a id="lifecycle-evidence"></a>
## 生命週期證據

同一個真實任務透過 shell 計算寫入 `80235`，而執行 `bin/fm-harness.sh` 的真實工具子程序則寫入 `polytoken`。
語意紀錄變成 `idle polytoken-hook stop`，且存在回合結束標記。
實驗環境的監看器將任務狀態與回合結束標記都當作訊號輸出；`bin/fm-crew-state.sh` 根據任務完成狀態回報 `state: done`。
在中斷驗證之前，一則持久 `fm-send` 訊息已被讀取並移入 `handled/`。
`fm-control interrupt` 回報 `verified=agent-alive cancel=cancelled`，語意紀錄變成 `idle fm-interrupt interrupt`。
重新啟動綁定到不同工作階段，並完成要求的計算，得到 `493`。
Exit 回報 `stopped`；teardown 移除了任務中繼資料與產生的 Polytoken 成品。
這次執行沒有被授權或信任提示阻擋；操作者既有的接受紀錄已足夠。

<a id="earlier-surface-evidence"></a>
## 較早的介面證據

最初的 2026-10-07 驗證使用 Polytoken 0.8.19。
它確認 hook 處理器接收事件 JSON 與 `POLYTOKEN_*` 變數，但沒有處理器 argv；hook 失敗可能使回合失敗；`post_model_turn` 則會在工具階段之間觸發。
這些發現說明了為何忙碌狀態寫入器必須自給自足且容忍失敗，以及為何正常閒置結束採用 `stop`。
REST 取消不會發出 `stop`，因此中斷負責者會在鍵入操作穩定後關閉語意紀錄，同時防範回合重新開啟。
根層級 `--config-dir` 提供完整執行期設定；hook 與 facet 探索則使用 cwd 的 `.polytoken/` 層。
工作階段樹使用 `--sessions-dir` 旁帶版本號的 `<root>-v1` 同層目錄。
2026-10-14 的生命週期執行，在已安裝執行檔上實際測試了相同的產生設定、hook、facet、工作階段探索及中斷路徑。

<a id="task-directory-file-access"></a>
## 任務目錄的檔案存取

2026-10-15，在 macOS arm64、`polytoken 0.9.0` 上，Polytoken 沒有提供能讓檔案工具存取工作階段專案根目錄之外目錄的啟動時授權。
以下每次執行都使用 `polytoken --config-dir <cfg> --working-dir <project> exec --model zai/glm-5.3-flash`；其中 `<cfg>` 是操作者設定的鏡像，採用 `default_permission_matcher: bypass`，並讀取 `<project>` 之外同層目錄中的檔案。
沒有設定任何規則時，直接對外部路徑呼叫 `file_read` 與 `file_write` 都成功。
但透過 `tool_flow` 腳本呼叫 `file_read` 讀取相同路徑，會失敗並出現：

```text
[ToolCallError] path is outside this session's allowed paths: <outside>/in.txt. This session is anchored at <project>; file access is confined to the project root and paths the user has granted.
```

加入 `<cfg>/permissions.yaml`，針對 `<outside>{,/**}` 或 `<outside>/**` 設定 `filesystem.allow` 存取權限 `[read, write]`（另一次為 `[read, write, chdir]`）後，在 `bypass`、`bypass_plus` 與 `standard` 比對器下仍然被拒絕；即使先成功 `pushd` 進入目錄再 `popd`，結果也相同。
在該檔案刻意放入無效鍵名時，`polytoken doctor` 的權限檢查會失敗，證明每次執行都確實載入該檔；在 `standard` 下，規則也確實允許直接的 `file_read` 通過。
`tool_flow` 直接拒絕 `file_write`（`tool 'file_write' is not eligible for tool_flow`）。
`polytoken new --help`、`polytoken print schema app-config`、`polytoken print schema facet-frontmatter` 及 `polytoken print openapi`，都未提供其他工作階段路徑授權方式。
因此工作 Agent facet 會指示模型，針對 Firstmate 任務目錄使用直接檔案工具或 shell。
