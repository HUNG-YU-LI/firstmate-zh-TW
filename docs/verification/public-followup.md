<a id="promised-public-reply-verification"></a>
# 已承諾公開回覆的驗證

適用對象：維護者驗證。

這份紀錄支持透過 myfirstmate relay 承諾公開回覆的七項目前保證：

1. 已承諾的最終回覆能跨越上下文壓縮與重新啟動，只靠磁碟即可核對復原，並恰好一次送達原始討論串。
2. 從未選擇啟用 relay 的主目錄，不會為這些功能付出任何額外成本。
3. 傳送最終回覆不會結束公開處理循環：註冊會以 `state=delivered` 保留，直到執行 `retire --reason`；工作階段啟動會顯示 `open-loop` 行，`rechain` 則能將後續工作綁定到同一討論串。
4. 尚未持有任何登錄鎖時，首次註冊在原生 macOS Bash 3.2 與 `set -u` 下可成功。
5. 若公開循環的工作位於遠端二副主目錄，當可讀取的遠端狀態證明連結不存在，或可讀寫的遠端狀態清除相符且已綁定的舊版 Relay 連結後，即可退役；狀態無法讀取、相符連結不可寫、身分不符、無法在上限內取得中繼資料鎖，或完成情況未確認時，會保留循環而不掛住，且 `--force` 仍只涵蓋未解決的承諾義務。
6. 綁定遠端二副主目錄的工作，可回報具型別的終止結果：指示會指定工作 Agent 自己機器上存在的路徑，擁有者主目錄透過該路由收集開啟註冊的結果；無法連線的路由會明確失敗，可連線但為空的路由是正常的無操作，非開啟狀態的註冊則不連線直接略過。
7. 即使承諾的最終回覆原先預期已合併的 pull request，最終失敗或停駐的工作仍可交付，因此原始討論串會恰好一次收到誠實的失敗結果，不會留下無法履行的承諾。

[`docs/configuration.md`](../configuration.md#promised-public-replies-statepublic-followup) 定義面向操作者的契約，[`docs/architecture.md`](../architecture.md#optional-relay) 定義機制邊界，`tasks-axi public-followup --help` 則定義具型別的承諾義務結構。
任務時序與傳送證據保留於本紀錄之外。

<a id="environment"></a>
## 環境

於 2026-09-01 在 Darwin 25.5.0（arm64）記錄，使用 GNU bash 5.3.9、tasks-axi 0.2.5、jq 1.8.1 與 ShellCheck 0.11.0（`bin/fm-lint.sh` 固定的版本）。
原生 macOS 相容性工作通道，還會使用 `/bin/bash` 3.2.57 與真正安裝的 `tasks-axi`，執行聚焦的首次註冊回歸測試。
每個案例的 relay 都是假二進位目錄中的 `curl`，因此絕不會公開發文；`tasks-axi` 與 `jq` 則是真正的工具，因為若將承諾義務狀態機也替換成 stub，就無法驗證任何事。
遠端路由案例只會在 `FM_SSH_BIN` 程序接點偽造 SSH 二進位檔，再對代替遠端 checkout 的本機 checkout 執行真正受追蹤的 `fm-remote-entrypoint.sh`，因此必須到達遠端主目錄的工作確實會在那裡執行；不涉及任何主機或網路。
失敗結果回歸測試另於 2026-09-22，在相同環境使用 tasks-axi 0.2.6 更新。

<a id="restart-end-to-end-and-regressions"></a>
## 重新啟動端到端與回歸測試

```sh
bash tests/fm-public-followup.test.sh
```

```
ok - outcome text is collapsed to one line, bounded by codepoint, and never corrupts characters
ok - restart end-to-end: typed result reconciles from disk and delivers one reply to the original thread
ok - duplicate terminal results, restart replay, and repeated delivery are all no-ops
ok - wrong source, wrong work id, stale generation, malformed, unsupported deliverable, and forged identity are all refused
ok - a relay transport failure is held as retryable with no false completion, and the retry posts once
ok - a dry-run records no public delivery and leaves the commitment retryable
ok - a late success receipt closes the exact attempt with no second post, and a mismatched attempt is refused
ok - typed terminal cleanup clears the legacy link without posting
ok - a delivery interrupted between post and receipt refuses to repost
ok - a child home reports typed results but can never become the outward-post owner
ok - typed delivery refuses to post when its cleanup registration is missing
ok - marked secondmate teardown resolves its parent and fails closed when unavailable
ok - local seeding publishes durable parent state before its identity marker
ok - a lost launch-time parent binding is recovered from the durable local record
ok - a durable local parent record does not bypass a genuinely missing parent-side registration
ok - unknown durable parent fields remain forward-compatible
ok - conflicting live and durable parent bindings fail closed
ok - unsafe durable parent records fail closed before cleanup
ok - a NUL-bearing durable parent record fails closed before cleanup
ok - relay-disabled unmarked teardown runs no public-followup work
ok - a marked child proceeds without tasks-axi when its parent relay is disabled
ok - secondmate parent resolution matches the durable registry id literally
ok - traversal-shaped registrations are rejected before path construction or posting
ok - pending keeps registrations when tasks-axi returns malformed JSON
ok - the retained private request context keeps the original thread deliverable after inbox cleanup
ok - cleanup refuses while a public reply is owed and proceeds once it has landed
ok - a relay-disabled home runs no tasks-axi call, prints nothing, and gains no artifact
ok - a relay-enabled home with no commitments makes no backlog call and stays silent
ok - a relay-exhausted follow-up binding is escalated rather than retried into the thread
ok - the relay poll stays inert without a token, silent with no commitments, and surfaces a new result once
ok - startup surfaces unresolved public commitments only in a relay home that owes one
ok - typed public-followup records carry only public-safe summaries and deliverables
ok - dropped-baton regression: delivery retains the loop and pending prints open-loop
ok - CONTROL: the identical teardown REFUSES the moment a commitment is registered
ok - rechain posts the shipped follow-on into the same thread
ok - rechain resumes the same obligation after an interrupted bind
ok - concurrent rechains cannot fork one delivered source
ok - failed rechain retirement keeps the source claimed by one resumable destination
ok - first register succeeds with an empty lock list under /bin/bash
ok - registration replay preserves delivered and retired loop states
ok - redelivery does not report a retired loop as open
ok - retire closes delivered loops after secondmate home removal
ok - retire fails closed for an unbound existing secondmate
ok - retire fails closed when a secondmate ID is reassigned
ok - rechain refuses an unrelated existing destination
ok - pending skips a registration retired during settlement
ok - retire --reason closes the loop and drops the open-loop line
ok - retention creates no false teardown refusal and pending no longer prunes
ok - expiry escalation is pinned by FMX_NOW_OVERRIDE
ok - brief fails explicitly when typed deliverable keys are unavailable
ok - pre-change registrations are open loops and un-rechainable, never a crash
ok - teardown reports an unreconciled legacy Relay link
ok - secondmate promotion matches teardown parent resolution
ok - a public loop bound to a remote secondmate home delivers and retires
ok - delivered remote registrations skip offline collection routes
ok - --force still covers only the unresolved obligation, not the link clear
ok - retire fails closed when a remote route is reassigned
ok - retire fails closed when remote state is unreadable
ok - retire fails closed when remote state is non-writable
ok - retire accepts link absence in non-writable remote state
ok - the guarded remote clear refuses a lock it cannot acquire instead of hanging
ok - an unconfirmed remote clear is unknown completion, never a silent close
ok - a typed terminal result emitted in a remote work home reaches the owning home
ok - an unreachable remote work home fails loudly instead of reporting an empty inbox
ok - an unreadable remote outbox fails collection without losing its result
ok - invalid registration fails collection without dropping the staged result
ok - unsafe registration entries fail collection without dropping staged results
ok - route loss fails brief and consume without dropping the staged result
ok - empty reachable remote collection remains a healthy no-op
ok - remote brief rejects traversal and empty route path components
ok - a local work home's emit path is unchanged
ok - a duplicate report from a remote work home stays a no-op
ok - staging requires the matching secondmate firstmate home
```

重新啟動案例是保證 1 的端到端證明。
它先重現卡住狀態（已綁定工作、沒有完成核對的終止結果、傳送因 `still waiting on its bound work` 遭拒且發文數為零），接著讓具有二副結構的子端回報具型別 `pr-merged` 結果，刪除已清空的收件匣酬載，從磁碟核對復原，並斷言恰好一次 `connector/followup` 呼叫帶有原始 `request_id`、已驗證的 `posted` 收據，以及 Done 的承諾義務。

聚焦的 tasks-axi 0.2.6 回歸測試，是保證 7 的證明：

```sh
FM_TEST_ONLY=test_failed_work_on_pr_merged_promise_delivers_honest_outcome bash tests/fm-public-followup.test.sh
```

```
ok - failed work on a pr-merged promise delivers its honest outcome exactly once
```

它將承諾的 `pr-merged` 最終回覆綁定到回報 `outcome=failed` 的工作，將這個已接受關係核對為 `ready`，把記錄的失敗文字送到原始請求一次，並驗證承諾義務關閉。
停駐工作使用相同的具型別失敗終止結果，因此遵循相同狀態機路徑。

接力棒遺落案例是保證 3 的端到端證明。
它傳送承諾的 `report-ready` 最終回覆，斷言註冊仍保留且 `pending` 印出 `open-loop`，接著證明未綁定的後續 ship 不會遭到拆除拒絕（只變更單一變數的對照案例，仍會在為該工作註冊承諾的瞬間拒絕）。
接著 `rechain` 將新的 `pr-merged` 承諾義務綁定到相同請求／討論串，第二次後續回覆帶上已交付文字。
`retire --reason` 在移除前記錄私人收據，是唯一的關閉方式；重播註冊無法重新開啟已退役循環。
並行與綁定中斷案例，驗證單一已傳送來源不能分叉，且重試會收斂到相同目的承諾義務。
變更前的磁碟紀錄（沒有 `state=`、沒有 `request_context_b64`）會被視為無法 rechain 的開啟循環，不會崩潰。
[`.github/workflows/ci.yml`](../../.github/workflows/ci.yml) 的原生 macOS Bash 工作通道設定 `FM_TEST_STOCK_BASH=1`，並在真正的 `/bin/bash` 3.2 下，透過 `bin/fm-test-tap-verify.sh` 執行 `tests/fm-public-followup.test.sh`，證明首次 `register` 在登錄鎖清單起始為空時仍安全。

八個遠端路由案例是保證 5 的證明。
遠端二副主目錄只存在於自己的機器，因此註冊不記錄本機路徑，而每個必須先清除已綁定舊版 Relay 連結的關閉操作，原本都沒有本機目標可處理。
第一個案例先固定檢查該空白紀錄路徑，避免測試失去實際涵蓋意義，再端到端驅動 `deliver` 與 `retire`，斷言遠端主目錄中的相符連結確實已不存在，且退役收據已寫入。
第二個案例證明 `--force` 仍只控制未解決承諾義務的拒絕：對未解決遠端循環執行一般 `retire` 仍遭拒，遠端連結不變；強制執行則會關閉並清除連結。
重新指派案例，將已傳送循環的路由替換為另一遠端主目錄，該主目錄重用的工作 ID 帶有其他 Relay 請求；測試斷言退役會保留註冊，不會觸碰替代連結。
不可讀狀態案例，讓遠端狀態目錄無法搜尋，但其中仍含相符連結，證明無法確認的路徑會採取拒絕策略且不修改內容。
兩個不可寫狀態案例證明：相符連結因無法修改而在取得鎖前拒絕；已確認不存在的連結則可成功，因為不需要修改。
無法取得鎖案例，證明受防護清除會拒絕而非卡死。
它保持遠端狀態目錄**可寫**，因此拒絕只能來自有界的鎖等待，不可能來自可寫性前置條件；並由真正存活的程序持有中繼資料鎖，讓該鎖絕不可能被視為過期而回收。
可寫性前置條件能縮小卡死時間窗，卻無法完全消除，因為父目錄可能在檢查與建立鎖之間變成不可寫，而取得鎖時無法將這與仍存活的鎖持有者區分；一般無界等待會永遠重試，因此在加入有界取得之前，此路徑會無聲掛住，而不回傳核對拒絕。
案例斷言拒絕、保留註冊、沒有收據、遠端連結未變，以及呼叫確實會回傳；最後一點就是它與永不結束等待的可觀察差異。
最後一個案例讓傳輸無法連線，斷言關閉遭拒、註冊保留、遠端連結未變，並明確指出完成狀況不明，而非回報為確定失敗。
執行較舊 Firstmate 複本的遠端主目錄，不認得受防護清除旗標，因此會透過相同的保留待核對訊息採取拒絕策略；操作者必須更新該主目錄後再重試，且刻意不提供未受防護的備援。

<a id="reporting-a-terminal-result-from-a-remote-work-home"></a>
## 從遠端工作主目錄回報終止結果

十二個收集與發出案例，是保證 6 的證明。
它們與上述退役案例共用相同的假傳輸測試資料，因此必要的收集確實發生，同時不需要實際主機與網路。

第一個發出案例，在任何其他斷言前先固定陷阱條件：綁定遠端的工作 Agent 所收到指示，不得指定只存在擁有者機器上的擁有者主目錄路徑。
接著它精確執行印出的指令，因此驗證的是工作 Agent 實際收到的指示，而非手寫近似版本；並斷言具型別結果到達擁有者主目錄收件匣、`consume` 回報循環已就緒，以及工作主目錄中的暫存複本之後被退役。
重複案例重播兩邊：工作 Agent 再次回報、擁有者主目錄再次收集；斷言沒有第二次就緒公告，承諾也沒有變更，因此退役失敗後保留的暫存複本，無法產生第二次公開回覆。
路由與紀錄拒絕案例斷言：傳輸不可達、寄件匣不可讀、註冊狀態無效或不安全，以及路由遺失，都會明確失敗，且不丟棄暫存結果。
空路由案例證明，可連線但沒有暫存內容的路由，是一般的靜默無操作；已傳送註冊案例則證明，已結束活動的循環絕不聯絡離線路由。
路由路徑與暫存目的地案例證明，印出的遠端指令不能指向不安全或不相符的主目錄。
本機案例以相同條件斷言路徑不變：綁定主要主目錄的工作，仍被要求使用此 checkout 的指令碼直接發出到此主目錄；該指令仍可依印出內容執行，並直接發布至 `events/`，不留下暫存。

遠端主目錄循環的對外傳送，涉及上述獨立驗證的舊版連結清除，不屬於此收集路徑。

<a id="relay-disabled-zero-overhead"></a>
## 停用 relay 時零額外負擔

`tests/fm-public-followup.test.sh` 中的停用 relay 案例，對沒有 `.env` 的主目錄呼叫每個 public-followup 入口，記錄每次 `tasks-axi` 呼叫，並比較前後狀態樹。
它證明此功能不會呼叫 `tasks-axi`、不印出內容，也不建立 `state/public-followup` 成品，且不將此保證綁定到工作階段啟動獨立管理的狀態檔。

該主目錄的全部新增成本只有啟用判斷；以下測量 1000 次程序內呼叫，包含迴圈負擔：

```sh
. bin/fm-public-followup-lib.sh
for i in $(seq 1 1000); do fm_pf_relay_active "$HOME_DIR" || true; done
```

```
total_ns=22305959 per_call_us=22
```

每次工作階段啟動約 0.02 毫秒，來自單一 `[ -f "$FM_HOME/.env" ]` 檢查；它在任何其他操作執行前就回傳 false。

<a id="compatibility-axes-reviewed"></a>
## 已檢視的相容性面向

主要執行環境（`claude`、`codex`、`opencode`、`pi`、`pi-signed`、`grok`、`kimi`）：檢視後判定不適用。
這裡沒有任何操作讀取或繪製執行環境專屬狀態。
唯一觸及的監督介面，是工作階段啟動摘要與既有 relay 輪詢產生的喚醒酬載；前者已由 `bin/fm-supervision-instructions.sh` 依執行環境繪製，不需知道本節存在，後者則由每個執行環境協定以相同方式使用。

執行階段後端（tmux、herdr、zellij、orca、cmux）：檢視後判定不適用。
此處沒有指令讀取 `state/<id>.meta` 的後端欄位、解析端點或擷取面板。
生命週期整合包括待辦交接警告、升任時的 rechain 提示，以及 `bin/fm-teardown.sh` 對欠回覆的拒絕，加上不阻擋操作的 open-loop 與舊版 `x_request=` 警告。
它們檢查的是主目錄、任務、父端綁定與註冊紀錄，而非後端欄位或端點，因此在每個後端都具有相同行為。
