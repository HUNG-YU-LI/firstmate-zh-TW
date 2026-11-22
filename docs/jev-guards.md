<a id="jev-guard-framework"></a>
# Jev 防護框架

Jev guard 是有執行界限、唯讀的主機診斷工具，可將某一類資源或狀態壓力轉為機器可讀的稽核紀錄，以及單行判定結果。
防護工具讓監督迴圈能區分真正卡死的工作 Agent，與只是看起來像卡死的主機狀況，同時不授予任何防護工具改變受測系統的權限。
本文件定義每一類防護工具都須遵循的框架契約；各類工具自身的腳本標頭，則負責定義其量測訊號與門檻。

<a id="shape"></a>
## 結構

每一類防護工具都以一對檔案及其測試交付。
`bin/fm-jev-<name>-guard.sh` 是精簡包裝腳本，會解析自己的目錄，並透過 `python3` 以 `exec` 執行該類工具的引擎。
`bin/fm-jev-<name>-guard.py` 是引擎，負責量測、分類及輸出。
`tests/fm-jev-<name>-guard.test.sh` 透過引擎公開的 CLI 驅動測試，斷言可觀察的輸出，絕不斷言引擎原始碼文字。

<a id="engine-contract"></a>
## 引擎契約

- 唯讀診斷：防護工具絕不寫入受測系統，也不修改 Agent、工作階段或儲存庫狀態。
- 失敗時放行（fail-open）：權限錯誤、缺少虛擬檔案及虛擬化環境的能力缺口，都應平順降級為附帶原因的 `UNKNOWN` 判定，絕不崩潰，也不誤報。
- 有界限：在正常主機上，單次執行應遠低於一秒；無法在時間預算內回答的防護工具，應回報 `UNKNOWN`，而非阻擋呼叫端。
- 結構化輸出：`--json` 輸出單一 JSON 物件，包含 `name`、`checked_at`、`status`、`recommendation` 與該類工具自己的量測欄位；人類可讀輸出則用簡短清單呈現相同事實。
- 確定性分類：`status` 只能是 `OK`、`WARNING`、`CRITICAL` 或 `UNKNOWN`；最後一種僅用於 fail-open 保留判定時。門檻定義於引擎，並在標頭具名列出，方便讀者稽核判定。

<a id="verdict-semantics"></a>
## 判定語意

- `OK`：量測條件正常，呼叫端應維持原樣繼續。
- `WARNING`：條件已退化，但原因明確；呼叫端記錄後繼續。
- `CRITICAL`：此條件足以解釋工作 Agent 為何沒有回應；在條件持續期間，呼叫端不應升級卡死事件，但仍須遵守呼叫端對延後處理設定的界限（見[呼叫端](#caller)）。
- 防護工具絕不建議破壞性操作；`recommendation` 是給操作者的診斷文字，不是指令。

<a id="caller"></a>
## 呼叫端

唯一的呼叫端是監看器的卡死計時器，而且只在主目錄的 `config/wedge-evidence-guards` 列有防護工具時使用。
在該情境下，運作中的窗格即將升級卡死事件前，會詢問每個列出的防護工具；收到 `CRITICAL` 時，會在有限個時間窗內延後升級，而非完全抑制。
[`configuration.md`](configuration.md#wedge-evidence-guards-configwedge-evidence-guards) 負責定義這項選擇性設定及其操作者範例類型；`bin/fm-watch.sh` 的 `wedge_guard_evidence` 則負責確切的呼叫方式與 fail-open 邊界。

<a id="adding-a-family"></a>
## 新增防護類型

複製最精簡的既有檔案組，讓包裝腳本保持在十行內，並將所有門檻留在引擎中，以註解標明它限制的是哪一項資源。
在旁邊加入該類工具的行為測試，並透過 `bin/fm-test-run.sh` 執行。
若某一類工具需要主機專屬來源，例如船隊登錄表、資源池管理器、配額服務或產品的 hook 儲存區，就應放在操作者自己的層次，而非本框架。
