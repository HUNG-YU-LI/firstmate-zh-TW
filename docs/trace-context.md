<a id="native-w3c-trace-context-propagation"></a>
# 原生 W3C 追蹤上下文傳播

Firstmate 可將 W3C [`traceparent`](https://www.w3.org/TR/trace-context/) 傳播給每個啟動的 Agent，讓外部觀察者將每個任務辨識為恰好一條 trace，並在同一身分下關聯該任務執行的所有工作。
追蹤邊界就是任務：持續存在的二副（Secondmate）是路由基礎設施，具有自己的 Agent 身分，絕不是經由它路由之無關任務的共用追蹤根。
此能力預設關閉、由原始碼定義、不依賴廠商，且範圍刻意維持狹窄。
本文件說明設計理由與目前行為；`docs/configuration.md` 定義設定結構，`bin/fm-trace-context-lib.sh` 的標頭定義精確機制，[`verification/trace-context.md`](verification/trace-context.md) 則記錄可重複的測試證據。

<a id="why-this-is-a-source-change-at-all"></a>
## 為何需要修改原始碼

Firstmate 持久保存的操作成品，已經讓下游觀察者能推導邏輯任務身分與生命週期。
觀察者無法在啟動後重建的原始碼能力，是在 Agent 啟動前透過環境傳入任務範圍的追蹤 ID，並在任務中繼資料中記錄相同身分。
此功能只新增這個載體接點。

<a id="what-it-does"></a>
## 功能行為

啟用後，Firstmate 會在每次 spawn 時，為任務解析一個 W3C `traceparent` 載體：任務第一次 spawn 時建立全新追蹤根，重新啟動時則原樣重用中繼資料中的值，並且：

- 使用 `00-<32 hex trace id>-<16 hex span id>-<2 hex flags>` 格式，新追蹤根採用隨機 ID；
- 在啟動前立即透過既有傳送 `GOTMPDIR` 的後端專屬環境通道，以 `TRACEPARENT` 環境變數傳入；以及
- 在 `state/<id>.meta` 中，以 `traceparent=` 記錄完全相同的值。

以環境變數 `TRACEPARENT` 攜帶 W3C 格式值，是 Firstmate 的慣例：W3C Trace Context 標準化的是 `traceparent` HTTP 標頭，不是環境變數；OpenTelemetry SDK 也不會自動從環境讀取它，因此下游觀察者必須明確讀取此環境值或 `traceparent=` 中繼資料欄位。
此功能本身不會替任何 SDK span 設定父項。

注入與記錄的載體是同一字串，因此讀取中繼資料的觀察者，能精確重建子程序收到的身分。
傳遞位於無條件執行的啟動前環境設定位置，因此涵蓋 `claude`、`codex`、`opencode`、`pi`、`pi-signed`、`grok`、`kimi`、`cursor`、`gemini`、`muse`、`rovo`、`agy`、`devin`、`commandcode`、`polytoken`、`cortex`、`kiro` 與 `hermes` 的 ship 與 scout 啟動，也涵蓋相同集合的二副啟動，但刻意僅支援船員的 `gemini`、`muse`、`rovo`、`devin`、`commandcode`、`polytoken`、`cortex`、`kiro` 與 `hermes` 配接器除外。
這與 `GOTMPDIR` 既有涵蓋範圍相同，不需要追蹤專用的 `launch_template()` 行為。
ship 與 scout 啟動在所有以面板鍵入的 spawn 後端（`tmux`、`herdr`、`zellij`、`orca`、`cmux`、`paseo`、`rex`）都會到達該位置；二副則會在所有接受二副啟動的面板後端（`tmux`、`herdr`、`zellij`、`rex`）到達，因為 `bin/fm-spawn.sh` 會在 `orca`、`cmux` 與 `paseo` 拒絕二副。
`t3code` 沒有面板，因此其 `claude` 與 `codex` 的 ship、scout 與二副啟動，會在相同條件下，透過後端的[各目錄執行環境設定](t3code-backend.md#per-directory-harness-environment) 收到相同的 `TRACEPARENT` 值。

<a id="remote-secondmate-routes"></a>
### 遠端二副路由

[遠端路由](remote-secondmates.md)上的二副，不會在父程序本身到達該 export 位置：父端將啟動交給設定的主機，由該主機執行自己的 `bin/fm-spawn.sh`。
身分仍屬於父端，因為父主目錄保存觀察者會讀取的任務中繼資料。
因此，父端會依自己的凍結決策，針對該任務自己的中繼資料解析載體：重新啟動時原樣重用，否則建立全新追蹤根，絕不採用父程序環境中現有的 `TRACEPARENT`。接著將載體傳給遠端主機，由其在相同的無條件啟動前位置 export，並回傳端點實際持有的載體。
父端會記錄回傳值，因此已存活且未重新啟動的遠端端點，會回報其 Agent 真正收到的身分，而不是父端原本打算傳入的身分。
遠端主機會在載體能到達任何面板之前，嚴格驗證它是合法 W3C 值；若父端停用功能，就不傳任何值，遠端啟動維持與未追蹤時相同。
若端點已存活，就不會重新啟動或注入；父端仍會記錄端點回報的任何載體，即使父端目前決策為 `off`，讓其中繼資料不會否認執行中 Agent 的實際身分。
啟用決策的傳遞與本機路徑完全相同：遠端主目錄將 `config/trace-context` 作為宣告繼承項目接收，而新的二副程序會收到父端凍結的 `FM_TRACE_CONTEXT=on|off` 快照。

<a id="root-and-recovery-semantics"></a>
## 追蹤根與復原語意

這些規則的目的是每個任務一條 trace：絕不合併無關任務，也絕不為相同任務建立第二個身分。

- **追蹤根**：若 spawn 的任務中繼資料沒有有效的已記錄載體，就建立全新的 trace ID、span ID 與取樣旗標（`01`）。
  這會開始新的 trace，每個任務一條。
  絕不採用啟動程序自己環境中的 `TRACEPARENT`：那個值是程序本身啟動時收到的 Agent 身分，持續存在的二副會在整個生命週期保留它，同時將無關請求路由出去。
  若採用它，就會將所有路由任務串成每個二副一條不斷成長的 trace；因此每個路由任務都建立自己的追蹤根。
- **復原**：已記錄在任務中繼資料中的有效 `traceparent=` 會原樣重用，讓重新啟動或復原的任務，在重啟之間保有穩定身分，而非開始第二條 trace。
  損毀的紀錄值會重新建立為全新追蹤根，不會繼續傳播。

因為絕不讀取環境中現有的 `TRACEPARENT`，監督程式恰好處於何種環境——二副啟動時的載體，或留有 `TRACEPARENT` 的操作者 shell——都不會滲入新任務身分。
停用傳播是刻意的追蹤邊界：停用的主目錄不會向新啟動或重新啟動的 Agent 注入任何載體，即使任務中繼資料已含有效的 `traceparent=`。
真正的停用狀態重新啟動，會重新產生不含 `traceparent=` 的任務中繼資料，因此稍後啟用狀態下再重新啟動時，會建立新追蹤根，不會延續邊界之前的身分；重用已存活的遠端端點並非重新啟動，會保留該 Agent 已持有的載體。

<a id="enablement-is-home-session-scoped"></a>
### 啟用狀態限定於主目錄工作階段

每次加鎖執行 `bin/fm-session-start.sh` 時，會且只會一次將該主目錄的 `config/trace-context` 加上 `FM_TRACE_CONTEXT`，解析為工作階段範圍的有效狀態。
決策透過同目錄暫存檔原子發布，並綁定目前工作階段鎖，因此發布失敗不會重新啟用先前工作階段過期的 `on` 紀錄。
該主目錄的每次 spawn 只讀取凍結的 `on` 或 `off` 決策。
後續設定或環境修改都會被忽略，直到該主目錄開始新的工作階段。
有效狀態缺少、過期、無法讀取、無效或發布失敗時，都安全地預設為 `off`。

主要 Agent 啟動本機或遠端二副時，會將 `config/trace-context` 傳到二副主目錄，並將主要工作階段的凍結決策，以非空的 `FM_TRACE_CONTEXT=on|off` 啟動覆寫值傳入。
二副會在自己的主目錄工作階段開始時，解析這個繼承的覆寫值。
該旗標是工作階段範圍的啟用狀態，而非持久設定，因此只在啟動收斂點傳送——凍結決策也在該處一起交接；對已執行主目錄進行即時收斂時不會觸碰它，本機與遠端路由皆然。
傳播的是啟用決策，絕不是追蹤身分：在啟用狀態下啟動的二副，會從主要 Agent 收到自己的任務載體，也就是二副 Agent 的身分，二副本身重新啟動時原樣重用；而它啟動的每個工作 Agent，都會建立自己各任務的追蹤根。
在停用狀態下啟動的二副，即使主目錄存在 `config/trace-context`，其工作 Agent 仍不會被追蹤。
啟用時，重新啟動會重用任務有效的已記錄載體；沒有載體的任務則建立全新追蹤根。
重複的二副啟動會在繼承追蹤上下文之前遭到拒絕，因此重複啟動預檢不會修改二副主目錄。

要變更整個艦隊的設定，必須手動完整重啟艦隊，讓每個主目錄開始新工作階段並凍結新決策。
Firstmate 不會監控設定漂移、偵測不一致、拒絕啟動，或自動停止／重新啟動任何主目錄。

<a id="sampling"></a>
## 取樣

新的追蹤根會將 W3C trace 旗標設為 `01`（已取樣）。
這是刻意且由原始碼定義的選擇：

- 此能力**需自行啟用**且預設關閉，因此主目錄啟用時，就是要求追蹤其啟動的工作；未取樣的（`00`）追蹤根會產生大多數下游依父項取樣器丟棄的 trace ID，讓選擇啟用的操作者得不到結果。
- **已記錄載體原樣保留旗標。**
  復原會逐位元組重用任務已記錄的載體，包含旗標，因此任務取樣決策在重新啟動之間維持穩定。
  Firstmate 只在建立*追蹤根*時選擇旗標，這也是建立新載體的唯一方式。
- **成本與隱私影響。**
  `01` 記錄取樣*決策*，符合規範的下游依父項取樣器會遵從它；但它本身不保證任何收集器會儲存 span，Firstmate 也不發出自己的 span，只會設定載體旗標。
  操作者若啟用此能力，並接上遵從取樣決策的監測工具，就應預期大約每個任務記錄一條 trace，其基數與保留時間由該監測工具的設定決定。
  想要未取樣追蹤根或頭部取樣的操作者，須在下游處理，或透過日後明確限定範圍的選項達成；Firstmate 不內嵌取樣器。

<a id="safety"></a>
## 安全性

- **預設關閉。**
  沒有 `config/trace-context` 與 `FM_TRACE_CONTEXT` 時，全新 spawn 或真正重新啟動都不會注入內容，也不會寫入 `traceparent=` 行，因此產生的中繼資料與啟動環境不變。
  重用已存活的遠端端點時，會記錄端點回報的任何載體，不注入新載體。
  加鎖的工作階段啟動會進行一次設定檔檢查，每次 spawn 也會多載入一個函式庫並讀取凍結的有效狀態檔，所以程序不是字面上的逐位元組完全相同，但 Agent、觀察者或任務中繼資料可見的內容沒有差異。
- **暴露與不暴露的資訊。**
  Firstmate *建立*的追蹤根使用隨機 ID，不讀取提示、路徑、任務文字、憑證或任意環境鍵，因此 Firstmate 絕不會在載體中*產生*敏感資料。
  Firstmate 注入的每個載體，不是這種新建立的值，就是同一任務先前記錄載體的原樣重用；絕不讀取環境現有的 `TRACEPARENT`，所以呼叫端控制的位元組不會進入新載體。
  暴露範圍限定於這個固定寬度載體：它無法攜帶 `tracestate`、`OTEL_*` 憑證變數或任何任意環境鍵，也沒有可設定或任意指令（只有用來取得熵的固定本機 `od`／`tr`）。
- **故障獨立。**
  建立載體是小型本機熵管線：透過固定本機 `od` 與 `tr`（從 PATH 解析），從 `/dev/urandom` 讀取少量位元組。
  沒有可設定的提供者指令、網路或 watchdog。
  一般成本很小，但 `od`／`tr` 是外部程序，因此沒有硬性的延遲保證，不能保證時間上限可忽略。
  任何會回傳的熵取得或自我驗證失敗，都只會省略該次 spawn 的載體，不中止原始工作；損毀的已記錄載體會重新建立為全新追蹤根，而不是傳播出去（這不是省略）。
  在面板後端，若啟動前載體 export 失敗，Firstmate 會省略 `traceparent=` 中繼資料宣告，仍然啟動任務。
  若面板後端回報無法清除失敗的追蹤輸入，Firstmate 會拒絕附加啟動指令，避免帶著不明的部分載體啟動。
  若面板 export 後記錄載體失敗，Firstmate 會在啟動指令中 unset `TRACEPARENT`，仍然啟動任務，讓子程序絕不收到中繼資料不存在的身分。
  T3 Code 會先記錄載體，再將其加入[各目錄執行環境設定](t3code-backend.md#per-directory-harness-environment)；記錄失敗會省略載體，環境安裝失敗則拒絕啟動。
- **僅中繼資料。**
  值存在於任務啟動環境與 `state/<id>.meta`。
  [T3 環境契約](t3code-backend.md#per-directory-harness-environment) 定義其覆蓋層清理；teardown 與以往一樣移除任務狀態，不需要結構遷移。

<a id="relationship-to-opentelemetry-and-later-increments"></a>
## 與 OpenTelemetry 及後續增量的關係

Firstmate 不需理解 OpenTelemetry、任何匯出器、收集器、儲存區或 UI。
它發出標準 W3C 載體並記錄相同身分；其餘都由下游觀察者負責，並從主目錄工作階段的凍結決策或 `traceparent=` 欄位，發現目前啟用的傳播。
原生生命週期事件發出、額外穩定 ID、接收中繼資料與任何內嵌 OTLP，都刻意延後，直到執行中的觀察者證明有具體的還原精度缺口，且無法由推導成品補足。

<a id="verification"></a>
## 驗證

可重複的測試證據——單元與 spawn 路徑測試套件，以及精確指令和輸出——位於 [`verification/trace-context.md`](verification/trace-context.md)。
