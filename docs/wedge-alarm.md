<a id="away-mode-injection-wedge-alarm"></a>
# 離開模式的注入卡死警報

離開模式的子監督器（`bin/fm-supervise-daemon.sh`）會緩衝需要升級處理的事件，再將它們注入 Firstmate 自己的監督器端點。
若注入超過 `FM_MAX_DEFER_SECS` 仍無法確認送出，`inject_wedge_alarm` 就會發出明顯且有頻率限制的警報，確保停滯不會一直隱而不見。
主動警示獨立於窗格，因為 tmux 狀態列閃爍在其他後端沒有對應機制，也無法可靠地通知不在電腦前的船長（captain）。
持久標記與 tmux 閃爍仍會保留，作為額外訊號。

<a id="channels"></a>
## 通道

`config/wedge-alarm` 是本機設定，已由 gitignore 排除。
每個非空白、非註解行列出一項通道指示；所有列出的非 `off` 通道都會盡力發送。
`FM_WEDGE_ALARM_CHANNEL` 可用單一指示覆寫檔案，方便針對性測試。

- `off`：停用所有主動警示，但保留持久標記與 tmux 閃爍。
- `auto` 或 `default`：在 macOS 解析為 `osascript`。
  其他平台沒有內建作業系統通道；若僅有持久標記仍不足，請設定 `command:`。
- `osascript`：在終端機窗格之外，透過 macOS 通知中心顯示橫幅。
- `herdr`：在受監督的窗格之外呼叫 `herdr notification show`。
- `rex`：在 Firstmate 自己的 Rex 窗格寫入 blocked 狀態的 `id=fm-wedge` 程式狀態紀錄，並發出桌面通知；下一份摘要成功送達後會移除該紀錄。這需要 Rex 監督器窗格，且 `auto` 絕不會選用它。
- `command:<cmd>`：透過 `sh -c` 執行 `<cmd>`，將警報摘要同時放在 `$1` 與標準輸入，便於傳送到手機或呼叫器服務。

若沒有 `config/wedge-alarm`，其行為等同 `auto`，也就是在 macOS 預設啟用。
這是刻意的設計：警報只會在真正超過最長延遲、發生卡死後觸發，且每個最長延遲時間窗最多發送一次。

每個通道都採盡力而為的方式。
若缺少執行檔或結束狀態不為零，會記錄警告並繼續下一個通道，不會讓常駐程序迴圈崩潰。
每次呼叫都受 `FM_WEDGE_ALARM_TIMEOUT_SECS` 的程序群組時間上限約束，預設為 10 秒；`command:`、`osascript`、`herdr` 與測試替換點均包含在內。
逾時或常駐程序關閉時，會終止通知器的程序群組，之後可執行下一個已設定通道。
AppleScript 透過 argv 項目接收摘要，而非將摘要插入原始碼，因此摘要文字無法改變腳本。
可直接複製使用的設定請參閱 [`examples/wedge-alarm`](examples/wedge-alarm)。

<a id="test-safety"></a>
## 測試安全

每個通知器都會經由 `wedge_alarm_emit` 中的 `FM_WEDGE_ALARM_EXEC` 轉送。
當常駐程序以函式庫形式載入時，該替換點預設為 `discard`，避免測試意外發出真正通知。
測試套件需要斷言通道選擇與摘要傳遞時，`tests/wake-helpers.sh` 會將它替換為記錄器。
正式環境不設定這個替換點，直接使用設定好的實際通道。

`tests/fm-daemon.test.sh` 涵蓋指示解析、頻率限制、逾時與程序群組清理、argv 安全派送、通道備援，以及 `command:` 的安全摘要傳送。
[`verification/supervision.md`](verification/supervision.md#wedge-alarm-channels) 記錄了 macOS 與 Herdr 通道的有界限手動驗證證據。
