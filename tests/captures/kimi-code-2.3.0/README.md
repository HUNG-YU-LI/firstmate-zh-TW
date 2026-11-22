<a id="kimi-code-cli-230-folder-prompt-captures"></a>
# Kimi Code CLI 2.3.0 資料夾提示擷取紀錄

這些檔案是實際 Kimi Code CLI 2.3.0 資料夾提示的可見窗格內容；`../../fm-kimi-harness.test.sh` 中的 `test_kimi_230_*` 案例會透過模擬窗格重播，每種措辭各一份。
它們於 2026-10-23 在 Linux、tmux 3.6 上記錄：以全新且隔離的 `HOME` 與 `KIMI_CODE_HOME`，直接啟動 `kimi --auto`，關閉更新器（`KIMI_CODE_NO_AUTO_UPDATE=1`），且未登入。
同一天發布的 Kimi Code CLI 2.3.1，除了目錄那一行以外，所有擷取內容都逐位元組相同。

<a id="capture-provenance"></a>
## 擷取來源

每份擷取都是提示繪製完成後，未經修改的 `tmux capture-pane -p` 輸出；只移除結尾空白列，並以 `{WORKDIR}` 取代啟動目錄。
重播時只把這個代碼換成模擬窗格的路徑。
這些目錄位於任何 Git checkout 以外的暫存根目錄下，因為啟動目錄若在 worktree 內，Kimi 會從 Git worktree 根目錄解析根層 `.mcp.json`。

| 檔案 | 窗格 | 目錄 | 提示顯示內容 |
| --- | --- | --- | --- |
| `plain.pane` | 120x40 | 空目錄 | `No project integrations or instructions to activate.` |
| `root-mcp-json.pane` | 120x40 | 根層 `.mcp.json` 中有一個 stdio 伺服器 | `Start 1 MCP server automatically`，下方為 `Config: .mcp.json` |
| `kimi-code-mcp-json.pane` | 120x40 | 同一伺服器位於 `.kimi-code/mcp.json` | `Start 1 MCP server automatically`，下方為 `Config: .kimi-code/mcp.json` |
| `small-terminal.pane` | 100x7 | 含根層 `.mcp.json` 的目錄 | 標題下方為 `Enlarge terminal to review sources. Esc exit.`，Kimi 忽略 Enter |
| `paged.pane` | 100x12 | 含根層 `.mcp.json` 的目錄 | 完整對話框，原公告位置改為 `←→ page · 1 / 5` |

宣告的伺服器指令會在標記日誌中追加一行。
同一實驗環境中，在 `root-mcp-json.pane` 提示按下 Enter，會在任何登入之前啟動該指令；模擬窗格在按鍵送達宣告或分頁提示時，重現的就是這個行為。

<a id="replay-limits"></a>
## 重播限制

這些重播證明 `bin/fm-spawn.sh` 如何回應這些完全相同的窗格。
已安裝的 Kimi 是否仍呈現相同內容，則由即時防護測試 `tests/fm-kimi-trust-prompt-live-e2e.test.sh` 驗證。
