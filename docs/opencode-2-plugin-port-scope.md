<a id="opencode-2-primary-plugin-port---scoping-findings"></a>
# OpenCode 2 主要外掛移植：範圍評估結果

這是 2026-10-01 針對已安裝 OpenCode v2.0.21 記錄的當時範圍評估。
目的在估算 OpenCode 2 主要整合的移植工作量，讓團隊能依證據選擇立即移植，或讓 OpenCode 主要工作階段暫時維持 1.18.x。
這不是長期契約；移植落地後，轉接器參考文件 `.agents/skills/harness-adapters/references/harness/opencode.md` 與 `docs/turnend-guard.md` 仍是權責來源。

除非註明取自已安裝執行檔或 `@opencode-ai/plugin`／`sdk` 套件，以下事實都已在 v2.0.21 親自驗證（`opencode run --standalone --print-logs --auto`，模型 `openai/gpt-5.5-fast`）。

<a id="what-changed-in-the-plugin-model"></a>
## 外掛模型的變化

OpenCode 2 拒絕 v1 外掛模組，並要求 default export。
v1 結構的載入失敗訊息是 `PluginModule.LoadError: Plugin must export a default definition with an id and an effect or setup function.`。
以 `{ id, setup }` 作為 default export，可以成功載入。

v2 不採用 v1 的 `server` 入口及其回傳的 hooks。
一個匯出 `{ id, server, setup }` 的哨兵外掛顯示，只有 `setup` 會執行：`server` 函式從未被呼叫，雖然回合執行了真正的 bash 工具呼叫，它的 `event` hook 與 `tool.execute.before` hook 也都沒有觸發。
因此，目前 `{ id, server, setup(){} }` 的修改，只能讓全部五個主要外掛與 `fm-busy-state.js` 不報錯地載入，卻讓它們依賴的每個 hook 都失去作用。
受管理的 Herdr 外掛也使用相同 `{ id, server, setup(){} }` 結構，並記錄相同結論：其 `server` hooks 屬於舊版介面，實際行為透過另一個入口（`herdr-tui-session.js`，`tui`／`setup` 物件）執行。

<a id="the-v2-server-plugin-context"></a>
## v2 伺服器外掛上下文

伺服器外掛的 `setup(ctx)` 上下文提供以下領域（從運作中的哨兵外掛讀取）：
`app, location, options, agent, aisdk, command, event, experimental, generate, model, provider, integration, mcp, permission, plugin, reference, rpc, skill, storage, tool, vcs, websearch, worktree, session, shell`。

相關成員：

- `ctx.tool.hook("execute.before" | "execute.after", fn)`：註冊工具生命週期 hooks。
- `ctx.event` 為 `{ subscribe }`；`ctx.event.subscribe({ signal, onActivity })` 回傳可非同步迭代的 server-sent-event 串流，以 `for await` 消費；內建 effect 外掛也使用相同呼叫。
- `ctx.session` 為 `{ hook, create, get, switchAgent, switchModel, prompt, generate, command, synthetic, interrupt, update, move, wait, context }`。
- `ctx.session.hook(name, fn)` 只接受固定集合（`context`、`compaction`、`generate`、`retry`、`title`、`model.request`、`http.request`、`http.response`）；它會靜默接受其他名稱，但永不觸發，因此不能取代舊事件匯流排。

SDK 也提供相對應的伺服器操作 `EventSubscribe`、`SessionPrompt`、`SessionPromptAsync`，以及 `Tui*` 操作（`TuiShowToast`、`TuiSubmitPrompt`、`TuiAppendPrompt`、`TuiExecuteCommand`、`TuiPublish`）。

<a id="the-three-signals-firstmate-needs"></a>
## Firstmate 需要的三種訊號

三種訊號都可由 v2 伺服器外掛透過 `ctx.event.subscribe` 取得。
實際的 2.0.21 串流以 `session.execution.*` 表達回合與活動狀態，而非 v1 的 `session.idle`／`session.status`。
在一次使用工具的回合中，依序觀察到的事件類型為：
`session.inbox.enqueued`、`session.execution.started`、`session.instructions.updated`、`session.inbox.delivered`、`session.usage.updated`、`session.renamed`、`session.step.started`、`session.tool.input.started`、`session.tool.input.ended`、`session.tool.called`、`shell.created`、`session.tool.progress`、`session.step.streamed`、`shell.exited`、`session.tool.success`、`session.step.ended`、`session.text.started`、`session.text.delta`、`session.text.ended`、`session.execution.succeeded`、`location.shutdown`。

1. 回合結束（回合結束防護的後續回合）：`session.execution.succeeded`，另有 `session.execution.interrupted` 與 `session.execution.failed` 作為其他終止轉換。
   這取代 v1 的 `session.idle` 篩選。
2. 忙碌／閒置狀態（`fm-busy-state.js`）：`session.execution.started` 代表工作中；`session.execution.succeeded`／`interrupted`／`failed` 則使其進入穩定終止狀態。
   這取代 v1 的 `session.status` busy／retry／idle 篩選。
   必須根據事件 payload 的 session id 與 parent id，重新建立子工作階段與根工作階段的範圍界定；兩個 Herdr 外掛都採用這種方式。
3. 監看啟動觸發：沿用目前驅動它的回合終止訊號，但從 `session.idle` 對應到 `session.execution.succeeded`。

回合結束及工作階段啟動的後續回合注入（v1 的 `client.session.promptAsync`），對應到 `ctx.session.prompt` 與 `SessionPromptAsync` 操作。
工作階段啟動提醒的 `session.created` 篩選仍是一種事件類型，可透過相同訂閱取得。

<a id="seatbelts-cd-guard-and-pre-tool"></a>
## 防護機制（cd-guard 與 pre-tool）

`ctx.tool.hook("execute.before", fn)` 會在工具呼叫時觸發；在其中拋出例外會阻擋工具：模型回報 bash 指令遭拒，並顯示拋出的訊息，且沒有執行指令。
因此，將主體移到 `ctx.tool.hook("execute.before")` 後，`fm-primary-cd-check.js`、`fm-primary-pretool-check.js` 及監看啟動的工具前防護，都能維持「拋出即阻擋」的行為。

<a id="rough-effort-for-option-a-port-now"></a>
## 方案 A（立即移植）的粗略工作量

這是在 `setup(ctx)` 內進行伺服器外掛移植，不是本次實驗前曾列為最壞情況的完整 TUI 外掛重構。
防護腳本（`bin/fm-turnend-guard.sh`、`bin/fm-cd-pretool-check.sh`、`bin/fm-sessionstart-nudge.sh`、`bin/fm-busy-event.sh`）以及鎖存與協調邏輯都能重用；變更的是事件串接與事件名稱對應。

各外掛工作量：

- `fm-primary-cd-check.js`、`fm-primary-pretool-check.js`：小；將 `tool.execute.before` 主體移到 `ctx.tool.hook("execute.before")`。
- `fm-primary-sessionstart-nudge.js`：小至中；改由訂閱中的 `session.created` 驅動，透過 `ctx.session.prompt` 注入，並檢查 TUI 中外掛是否在工作階段建立前就已附掛。
- `fm-primary-turnend-guard.js`：中；由 `session.execution.succeeded` 驅動，保留防護腳本與監看啟動協調，將 `promptAsync` 換成 `ctx.session.prompt`。
- `fm-busy-state.js`（`bin/fm-spawn.sh` 中的 heredoc）：中；以 `ctx.event.subscribe` 迴圈取代 `event` hook，將 `session.status`／`session.idle` 重新對應到 `session.execution.*`，保留第一工作階段鎖存與回合結束 touch。
- `fm-primary-watch-arm.js`（約 560 行）：中至大；防護機制移到 `ctx.tool.hook("execute.before")`，閒置協調及未察覺狀態的回合喚醒則移到 `session.execution.*` 加上 `ctx.session.prompt`。

共用工作：每個外掛各自或共用的小型事件訂閱輔助程式、事件名稱重對應，以及從 default export 移除無效的 `server` 入口。
文件與測試：更新 `opencode.md`、`docs/turnend-guard.md`、`docs/arm-pretool-check.md` 與監督驗證紀錄；並依賴執行環境檢查的規則，為 busy、turn-end 與防護分類器新增可攜式回歸測試，以及使用已安裝執行環境的真實驗證防護。

粗略規模：約 1.5 至 3 個專注工作天，重點在 watch-arm、busy-state、兩個真實驗證防護及其測試，而非簡單的工具前防護。

<a id="risks-to-validate-during-implementation-not-resolved-here"></a>
## 實作時仍需驗證、此處尚未解決的風險

- `session.execution.*` payload 用於子工作階段範圍界定的確切結構（session id 與 parent id）。
- `session.idle`／`session.status` 在 2.0.21 是否還會觸發，或已完全被 `session.execution.*` 取代；真實執行只看見 `session.execution.*`，因此移植應依據這些事件，並把舊名稱視為舊版介面。
- `ctx.session.prompt` 的簽章，以及在 `session.execution.succeeded` 注入時，是否能乾淨啟動新回合而不與輸入區發生競態。
- 完整 TUI 工作階段及切換工作階段時，外掛與訂閱的生命週期；`run` 探測中的訂閱在 `location.shutdown` 結束。
- `onActivity` heartbeat 在長時間工作階段中，維持訂閱健康的語意。
