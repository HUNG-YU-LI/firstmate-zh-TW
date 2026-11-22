<a id="trace-context-propagation-verification"></a>
# Trace context 傳遞驗證

預設關閉的原生 W3C trace context 能力之可重複驗證證據。
目前行為與設計理由由 [`../trace-context.md`](../trace-context.md) 負責，設定結構則由 [`../configuration.md`](../configuration.md) 的「Trace context propagation」章節負責；本頁只記錄證據。

日期：2026-08-03。
Shell：GNU bash 3.2.57（macOS）。
比較基準：`main` 的 `976d97f`。

同目錄的單元測試套件 `tests/fm-trace-context-lib.test.sh`（26 個斷言）涵蓋：驗證（接受有效值；拒絕格式錯誤、長度錯誤、大寫、全零、`ff` 版本及含 shell 特殊字元的值）；根追蹤產生（每次產生不同且已取樣的根追蹤，不接受沿用父追蹤的輸入）；復原重用路徑（已記錄的 carrier 優先於外部環境）；預設關閉時省略；`FM_TRACE_CONTEXT` 優先於 `config/trace-context` 的啟用順序，未設定或空值時改由檔案決定；正規化的主目錄工作階段狀態；以原子方式替換唯讀舊紀錄；發布失敗後拒絕過期工作階段；狀態遺失或無效時預設關閉；二副主目錄工作階段邊界（後續檔案狀態）與每任務追蹤邊界（在持續存在的同一個外部 `TRACEPARENT` 下解析兩次，會建立兩個不同根追蹤，兩者都不沿用外部值）；強制亂數熵失敗時安全省略；以及新產生根追蹤的固定格式檢查。

派生路徑整合套件 `tests/fm-trace-context-spawn.test.sh`（12 個斷言）會隔離外部 `FM_TRACE_CONTEXT`，以假的 tmux 窗格與真正隔離的 Git worktree，端到端驅動 `bin/fm-spawn.sh`：啟用時，只有在啟動指令字面值之前送出相同的 `TRACEPARENT` export 後，才把解析出的 carrier 以 `traceparent=` 記錄在中繼資料；停用時，兩者都不寫入也不送出（`GOTMPDIR` 仍會送出）；carrier 投遞失敗時，不會留下 `traceparent=` 宣告，但原任務仍會啟動；若投遞不安全且無法清除部分輸入，會在附加啟動指令前停止；中繼資料附加失敗時，會從已啟動任務移除 carrier，但不中止任務；重複的二副前置檢查不會改變繼承的追蹤設定；重新啟動會原樣重用已記錄的 carrier；派生會忽略之後的設定與環境編輯，改用已凍結的主目錄工作階段決策。
每任務邊界回歸測試精確模擬已審查的二副情境：同一主目錄先後派生兩個無關任務，而派生環境持續放著同一個固定的 `TRACEPARENT`（常駐二副啟動時的 carrier）；兩個任務會記錄並注入有效 carrier，其 trace id 彼此不同，也不同於外部 carrier；重新啟動第一個任務時，中繼資料紀錄與注入的 export 都會原樣重用原始 carrier。
另有兩個斷言驅動真正的兩層「主要 Agent → 二副 → 工作 Agent」鏈，以主要 Agent 注入二副的完全相同環境執行兩次 `bin/fm-spawn.sh`，證明主要 Agent 的有效覆寫在兩種方向都能控制巢狀工作 Agent：環境啟用、沒有設定檔時，巢狀工作 Agent 仍保持啟用，並建立與二副 carrier 不同的每任務根追蹤；環境停用、設定檔存在時，即使 `config/trace-context` 已複製到二副主目錄，巢狀工作 Agent 仍保持停用。
最後一個斷言走由檔案決定的路徑（未設定 `FM_TRACE_CONTEXT`），證明二副記錄／注入的 carrier 與送達的 `FM_TRACE_CONTEXT=on|off` 快照，永遠衍生自同一個凍結決策，因此不會把 carrier 與相反的啟用狀態配在一起。
此套件不接觸任何真正的執行環境或運作中的船隊。
`tests/fm-session-start.test.sh` 另外證明，只有擁有鎖的工作階段啟動才會寫入有效狀態；因鎖被拒絕、以唯讀方式啟動時，則維持原狀。

遠端路由套件 `tests/fm-remote-secondmate-trace-context.test.sh`（6 個斷言）涵蓋永遠不會走到本機 export 位置的二副路徑。它在可確定重現的 SSH 邊界上，搭配有狀態的假 Herdr CLI，驅動真實鏈路：父層的 `bin/fm-spawn.sh`、`bin/fm-on.sh`、真正的遠端入口、`bin/fm-remote-secondmate-control.sh`，以及遠端主機自己的 `bin/fm-spawn.sh`。測試採用預設遠端後端，因此遠端窗格收到的 carrier 會從該窗格自己的日誌讀回。停用時，父層不記錄 `traceparent=`，遠端窗格不收到 export，遠端主目錄不繼承啟用旗標，送達快照是 `FM_TRACE_CONTEXT=off`，但仍傳送 `GOTMPDIR`；啟用時，父層記錄的 carrier、遠端端點自己的紀錄及窗格 export 值，是完全相同的有效 carrier，於 `GOTMPDIR` 之後、啟動指令之前送出，並一併送達 `FM_TRACE_CONTEXT=on` 與繼承旗標；重新啟動時，父層紀錄與窗格 export 都會原樣保留該 carrier；從含有固定外部 `TRACEPARENT` 的環境解析第二條遠端路由，會建立一個與外部 carrier 及第一條路由都不同的根 trace id；遠端接收端將 `config/trace-context` 視為一般已宣告的繼承資料接受，但拒絕主要 Agent 刻意不傳遞的 `config/secondmate-harness`；將父層 carrier 傳到遠端主機的投遞引數，若用於 ship spawn、含 shell 特殊字元、trace id 全零或值為空，都會被拒絕，確保只有二副啟動時符合嚴格 W3C 格式的 carrier，能到達窗格 export。

```console
$ bash tests/fm-trace-context-lib.test.sh | tail -1
# fm-trace-context-lib.test.sh: all assertions passed
$ bash tests/fm-trace-context-spawn.test.sh | tail -1
# all fm-trace-context-spawn tests passed
$ bash tests/fm-remote-secondmate-trace-context.test.sh | tail -1
ALL TESTS PASSED
```

請從儲存庫根目錄執行全部三套 trace context 測試；每個斷言都會印出一行 `ok - ...`。
單次真實後端端到端檢查——真正派生任務，確認窗格在啟動列之前收到 `TRACEPARENT` export，並在 teardown 後不留下任何內容——是有界限的手動步驟。由於派生真實 Agent 會干擾運作中的船隊，此處延後執行。
