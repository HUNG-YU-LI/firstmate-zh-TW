<a id="primary-startup-and-hooks"></a>
# 主要工作階段啟動與 hook

修改工作階段啟動、回合結束處理、工具執行前保護、監看程式監督或二副整合前，搭配偵測到的主要執行環境工具參考載入。
工具參考會確立該身分的實測路徑，或不支援的邊界。

<a id="turn-end"></a>
## 回合結束

「不盲目結束回合」契約、hook 安裝、各介面的阻擋行為，以及 hook 無法阻擋時的取捨，以 `../../../docs/turnend-guard.md` 為準。
執行環境專屬喚醒協定，由 `../../../docs/supervision-protocols/` 與 `../../../bin/fm-supervision-instructions.sh` 負責。
絕不替換成其他執行環境的等待形式。
語意忙碌狀態仍由 `../../../bin/fm-busy-lib.sh` 負責；工具參考只指出來源與證據。

任何回合結束變更，都在暫存專案或可拋棄 home 中，使用真正執行環境驗證。
同步更新負責的執行檔或 hook、簡潔工具事實，以及 `../../../docs/verification/supervision.md` 的「Turn-end guard」。

<a id="pre-tool-protection"></a>
## 工具執行前保護

支援的主要執行環境會在執行前拒絕監看程式啟用的反模式，包括 Shell `&`、截斷管線、綁定多個指令，以及廣泛的 `pkill -f fm-watch`。
Hook 指令、輸出特性與證據，以 `../../../docs/arm-pretool-check.md` 為準。
工具參考指出整合形式。
信任變更前，先在暫存專案中使用真正執行環境驗證。

主要執行環境也必須處理內建委派，因為它可能建立 Firstmate 持久紀錄之外的工作。
Claude 已驗證的委派防護記錄於 `references/harness/claude.md`，Antigravity 則記錄於 `references/harness/agy.md`。
完整契約、本機強化、例外入口與逐執行環境適用性審查，以 `../../../docs/subagent-guard.md` 為準。
沒有真實證據，絕不泛化 Claude 工具名稱或權限。

<a id="session-start"></a>
## 工作階段開始

行為仍以 `../../../AGENTS.md` 第 3 節與 `session-start-recovery` 技能為準。
原生層級分配、傳輸、來源路由、執行時間界線與失敗時放行行為，以 `../../../docs/sessionstart-nudge.md` 為準。
修改工作階段開啟行為前，先閱讀該文件。
現行附日期證據，以 `../../../docs/verification/supervision.md` 的「Native session-start delivery」為準。

<a id="watcher-supervision"></a>
## 監看程式監督

`../../../bin/fm-session-start.sh` 只為偵測到的主要執行環境印出一個區塊。
只遵循該展開後協定。
修改監看介接器時，更新其 `../../../docs/supervision-protocols/` 下的檔案；共用閒置或回合結束行為改變時，也更新 `../../../docs/turnend-guard.md`，並刷新工具事實。
沒有專屬協定的身分，使用文件記錄的不支援或未知邊界；絕不從相似 TUI 自行發明協定。
