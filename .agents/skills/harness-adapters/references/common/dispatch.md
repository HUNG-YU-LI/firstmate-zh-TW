<a id="dispatch-and-start"></a>
# 派工與啟動

派工、啟動或介接器驗證時，搭配選定工具參考載入；涉及任一設定檔軸時，再加上 `references/common/model-and-effort.md`。

<a id="resolution"></a>
## 解析

靜態船員與二副執行環境解析，以及所有明確覆寫，都依路由器的偵測與安全區段。
`config/crew-dispatch.json` 可用具體的執行環境、模型與推理強度軸，為單一船員或 scout 覆寫靜態預設。
設定檔陣列須先在此確認執行環境與供應商事實，再載入 `quota-array-dispatch`。
選擇啟用 `bin/fm-dispatch-resolve.sh` 後，其 `clear` 回答已指定具體軸；該契約以 `docs/configuration.md` 的「Typed dispatch resolution」為準。

繼承的本機素材，以 `../secondmate-provisioning/SKILL.md` 為準。
對執行環境的影響是：二副工作 Agent 取得原樣的 `config/crew-harness` 與 `config/crew-dispatch.json`；僅主要 home 使用的 `config/secondmate-harness` 則絕不繼承，因為二副不啟動二副。
像 `codex` 這樣的具體 crew 值，會將該執行環境帶入二副 home。
未設定或 `default` 不攜帶具體值，因此二副的工作 Agent 使用該 home 自身或偵測到的執行環境，不是主要 home 實際生效的 crew 執行環境。
繼承的派工檔案，在該處套用相同的最符合設定檔。

<a id="owners"></a>
## 負責元件

啟動、自主權、具體旗標、工作類型相容性與工作 Agent 回合結束銜接，以 `../../../bin/fm-spawn.sh` 為準。
自然語言規則留在 firstmate，指令碼只接收具體軸。

語意忙碌狀態的信任，由 `../../../bin/fm-busy-lib.sh` 負責。
輸入區形式、符號、提示文字、彈出視窗、畫面傳遞訊號，以及 `empty`／`pending`／`pending-unproven`／`unknown` 判斷，只由 `../../../bin/fm-composer-lib.sh` 負責。
工具參考為這些可執行元件記錄實測知識。

<a id="adapter-verification"></a>
## 介接器驗證

經核准的新介接器檢查，只能針對簡單、受監督的工作，使用啟動元件的原始啟動例外入口。
驗證 `../../../bin/fm-harness.sh` 的偵測、`../../../bin/fm-spawn.sh` 的啟動、`../../../bin/fm-busy-lib.sh` 的忙碌狀態、`../../../bin/fm-composer-lib.sh` 的共用輸入區行為、`../../../bin/fm-control-lib.sh` 的生命週期；若支援二副使用，還要驗證 `../../../bin/backends/tmux.sh` 的 tmux 存活判定。
也要透過 `references/common/primary-hooks.md` 驗證主要執行環境整合，透過 `references/common/model-and-effort.md` 驗證模型探索，並建立一份工具紀錄。
某個值的可執行元件、可攜式回歸測試、適用的帶憑證真實防護測試與驗證紀錄，必須一起合併，才可啟用。
依賴執行環境的證明，以 `../firstmate-coding-guidelines/SKILL.md` 為準。
