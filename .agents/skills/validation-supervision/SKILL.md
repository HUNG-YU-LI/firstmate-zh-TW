---
name: validation-supervision
description: 當 ship 開始或已存在進行中的 no-mistakes 驗證，包括執行中需求變更或發現，以及決定或回答任何 ask-user 發現之前載入。
user-invocable: false
metadata:
  internal: true
---

<a id="validation-supervision"></a>
# 驗證監督

對 no-mistakes ship，在工作 Agent 提交實作後，使用 `harness-adapters` 定義的執行環境呼叫方式，於同一工作 Agent 啟動驗證。
啟動 no-mistakes 執行的工作 Agent 負責推進管線，並負責直到下一關卡或結果為止的每次 `no-mistakes axi run` 與 `no-mistakes axi respond` 呼叫。
Firstmate 絕不替船員持有的執行呼叫 `no-mistakes axi respond`。
工作 Agent 端的 `--intent` 契約以 `bin/fm-dod-lib.sh` 為準。
驗證開始後，除非新需求完全推翻正在驗證的工作，否則優先把新需求安排為後續工作，不擴大目前工作；不過，為維持已接受產品或工程行為正確、在存在可執行契約時加入行為測試，或保持文件正確所需的最小下游變更，即使涉及受理時未列出的檔案，仍屬於目前工作；為符合已接受意圖所需的修正，不算新需求。

只有船長當前的明確指示，且該指示完全推翻正在驗證的工作時，才保留同一工作 Agent，不改排後續工作或交給替代 Agent。
該工作 Agent 必須透過 no-mistakes axi 支援的中止指令取消進行中的執行，並用 axi status 確認已停止，才能修改任何程式碼。
接著遵循結構化 axi status 的 `branch_sync.next_action`：只有 code 為 `recover_custody` 時，才使用 axi sync 支援的受防護復原；其他情況，只有結構化狀態確認分支所有權已歸還、且不需復原後，才能繼續。
保管權復原解決的是分支所有權，不是內容：工作 Agent 必須從正確的失效前基準替換過時工作，不能繼續疊加在雖已復原但內容過時的 head 上，也不得把過時執行自身的管線修正提交帶入待驗證與交付的內容。
除了上述唯一支援的中止操作，在過時執行仍持有分支期間，不得手動編輯、提交、重新啟動，或開始第二次驗證。
所有權確定後，只對最終 head 驗證一次，確保過時或中間 head 絕不被視為權威結果。

ask-user 發現會以 `needs-decision` 返回；firstmate 載入 `ask-user-authority`，依該技能自行決策或向上呈報。
向同一工作 Agent 傳送一項精確決策，列出決策鍵、步驟、動作、受影響發現 ID、必要指示與精確回覆指令，並傳入 `--resolve-key`，讓工作 Agent 的未結決策紀錄在回答時關閉。
要求對應的 `resolved` 事件，禁止 `--yes`，並要求工作 Agent 處理每個同步返回結果，直到完成或出現真正新的呈報事項。
決策送達後，立即恢復船隊監督。

以 [`bin/fm-crew-state.sh`](../../../bin/fm-crew-state.sh) 解析後的狀態行判斷驗證；其標頭定義結果對應與 CI 監控／常駐程序例外，絕不以 Shell 是否存活、最後狀態事件或原始執行紀錄判斷。
停在 approval 或 fix-review 的工作 Agent，必須遵循目前關卡說明。
在進行中的驗證期間，若工作 Agent 手動編輯、提交、中止或重新啟動，且不屬於上述取代序列，就是在管線之外重複行使所有權；應引導它回到關卡回覆流程。
工作 Agent 應在 CI 首次通過時回報 PR，不必等合併監控結束。
