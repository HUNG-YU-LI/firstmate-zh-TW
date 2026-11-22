<a id="local-claude-usage-threshold-warner"></a>
# 本機 Claude 用量門檻警示器

`bin/fm-usage-warner.sh` 是選擇性啟用、僅支援 macOS 的 Claude 用量時間窗門檻警示器。
它負責警示，不是檢視器：不會呈現即時用量報告，只會在指定時間窗已消耗的比例達到設定百分比時，通知一次。

讀取功能本身並不是新的。
[`quota-axi`](configuration.md) 是這個船隊唯一依賴的 Claude 配額讀取器，已提供 `--json`、`--full`、`--tui` 與 `--no-credential-refresh`；本腳本使用完全相同的讀取機制（透過 `bin/fm-quota-axi-lib.sh` 自己的供應商資料列合併），絕不重新實作。
每次呼叫都傳入 `--no-credential-refresh`，因此讀取嚴格維持唯讀；腳本絕不傳入 `--allow-keychain-prompt` 或任何會更新憑證的旗標。

<a id="opt-in-is-load-bearing"></a>
## 明確啟用是核心條件

未設定的主目錄不會改變任何行為：`check` 完全靜默，`arm` 則直接拒絕，不會註冊一個永遠沒有警示目標的檢查。
這項功能不會讓其他功能也相依於 `quota-axi`。

<a id="configuring-thresholds-configusage-warner"></a>
## 設定門檻（config/usage-warner）

門檻放在本機且被 gitignore 排除的 `config/usage-warner`；其 `<window-id>:<percent>` 結構由 [`configuration.md`](configuration.md#local-claude-usage-warner-configusage-warner) 定義。
`<window-id>` 直接沿用 `quota-axi` 自己 `--json` 的 `id` 詞彙，例如 `five_hour`、`seven_day`、`model:<name>` 等，不另創一套。
若設定的時間窗 id 沒有由帳號回傳，就會靜默略過比較：這是本機便利功能，不是關鍵監控，因此過期或無法取得的 id 不會被當作設定錯誤提出警示。

帳號層級的兩個時間窗 `five_hour` 與 `seven_day` 是可靠目標：不論各帳號剛好回傳哪些個別模型時間窗，每個帳號都有這兩項。
可複製到本機 `config/usage-warner` 的起始設定，請參閱 [`docs/examples/usage-warner`](examples/usage-warner)。

<a id="behaviour-one-read-two-callers-edge-triggered"></a>
## 行為：一次讀取、兩個呼叫端、邊緣觸發

`check` 是同一套讀取機制，供兩個呼叫端使用：人或其他腳本直接執行 `fm-usage-warner.sh check`（隨需），以及啟用後由監看器依正常 `FM_CHECK_INTERVAL` 週期輪詢已註冊的包裝腳本（定期）。
兩者都呼叫完全相同的函式，因此是一個程式、兩個呼叫端，不是兩個程式；它沒有自己的常駐程序，也沒有自己的排程。
兩者重疊執行時，會透過 `state/.usage-warner.lock` 序列化；檢查若發現鎖已被持有，就靜默結束，確保一次越過門檻仍只警示一次。

時間窗越過設定門檻時，會通知一次；只要維持在門檻或以上，就保持靜默；下次讀到低於門檻的值時，立即重新待命（帳號層級時間窗自行重置時，就是這種情況）。
每項指示各自追蹤，因此同一時間窗設定分級門檻，例如 `five_hour:80` 與 `five_hour:95`，會各自在越過門檻時通知。
同一次讀取若有多個時間窗越過門檻，會合併成一則通知，而非每個時間窗各發一則。
持續性的讀取失敗（缺少 `quota-axi`、缺少 `jq`、回應格式錯誤，或未在時限內完成讀取），只會回報一次，直到狀況改變；這沿用 `bin/fm-mail-check.sh` 與 `bin/fm-tool-update-check.sh` 對持續性檢查的既有契約。
讀取上限為 10 秒；若 `FM_CHECK_TIMEOUT` 減 3 秒更短，則採較短值，確保監看器不會先終止檢查。若剩餘時間不足 1 秒，就略過讀取並回報未量測。

<a id="notification-path"></a>
## 通知路徑

警示透過通知中心（`osascript display notification`）送達 macOS，與 Firstmate 自己在此平台的離開模式卡死警報使用相同作業系統層路徑（[`bin/fm-supervise-daemon.sh`](../bin/fm-supervise-daemon.sh) 的 `wedge_alarm_via_osascript`），讓主目錄維持一套警示機制，而非兩套。
本腳本不會 source 或呼叫該常駐程序的內部功能：正式環境只以 exec 執行常駐程序；若改成 source，常駐程序自身的函式庫模式防護會將通知器替換點預設為「discard」，目的正是避免第二個載入者透過它發出真實通知。
離開模式的設定結構與最長延遲頻率限制，也專屬於緩衝的升級事件，不適用於本功能自己的邊緣觸發去重。
因此本腳本以自己的標題發出相同的作業系統呼叫；這就是「相同警示路徑」中可重用的部分，無須進入常駐程序內部、離開模式專屬的狀態。
通知中心只支援 macOS，因此在其他平台上 `arm` 會拒絕執行。

<a id="relationship-to-the-quota-process-event-source"></a>
## 與配額程序事件來源的關係

[`bin/fm-procevent-quota.sh`](../bin/fm-procevent-quota.sh) 讀取相同的 `quota-axi` 快照，因此有必要區分兩者。
該來源會啟動阻擋式輪詢，最多觸發一次，以持久保存的 `check:` 事件喚醒 Firstmate，比較受追蹤供應商的彙總 `effectivePercentRemaining`；若要繼續監控，必須手動重新啟動。
本警示器處理的是持續監控的另一半：逐一比較具名時間窗，且每當時間窗讀值回到門檻以下，就自行重新待命。
越過門檻會發出 macOS 通知；啟用監看後，每次越過也會以一次 `check:` 喚醒到達 Firstmate，因為監看器會把已啟用包裝腳本的回報列轉為該喚醒。
明確指出時間窗是重點，因為工作階段時間窗與每週時間窗會在非常不同的時點耗盡；彙總數值無法指出哪一個即將用完。
兩者不互相取代，同時啟用也合理：本功能在時間窗尚未重置期間，每次越過各自門檻都會警示；另一個功能則針對供應商整體彙總值觸發一次，直到手動重新啟動為止。

<a id="arming"></a>
## 啟用監看

每個主目錄執行一次 `bin/fm-usage-warner.sh arm` 啟用，並以 `bin/fm-usage-warner.sh disarm` 移除；兩者分別寫入或移除的內容，以及 `arm` 何時拒絕執行，由 [`configuration.md`](configuration.md#local-claude-usage-warner-configusage-warner) 定義。

<a id="what-this-never-does"></a>
## 絕不執行的事情

安裝、更新或寫入任何憑證；讀取或呈現即時用量報告；或讓這項選擇性功能之外的其他功能相依於 `quota-axi`。
帳號或席次切換明確不在範圍內：只會讀取預設帳號通道。

它也不會停止任何工作或調整執行節奏：越過門檻只會發出警告，因此派送、執行中的工作及合併行為都維持原樣。
覆蓋範圍恰好等於 `quota-axi` 在本機看得到的範圍。
因此，若 `quota-axi` 不回報某個方案或組織的支出上限，此處設定的任何門檻都無法涵蓋它，它只會以執行環境窗格中的錯誤浮現；警示器保持靜默，只代表沒有已設定的時間窗越過門檻，絕不代表沒有任何上限即將到達。
