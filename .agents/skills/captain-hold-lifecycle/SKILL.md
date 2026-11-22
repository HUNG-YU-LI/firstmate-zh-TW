---
name: captain-hold-lifecycle
description: >-
  在將調查、偵察報告、結構化審查或 Lavish 審查視為完成前、結束曾提出船長決策的視覺審查前、記錄或路由船長回答時，以及喚醒 drain 印出任何 RECORD DIVERGENCE 行時載入。
user-invocable: false
metadata:
  internal: true
---

# 船長 hold 生命週期

決策不是另一種獨立事物：它只是等待船長的任務。
唯一原語是透過 `bin/fm-captain-hold.sh hold` 為船長保留的一般 backlog 任務；其身分就是 task id，該包裝器負責本政策仰賴的可確定重現機制。
Agent 負責語意盤點，因為指令碼不可從報告文字、視覺審查成品、終端機輸出或聊天推斷船長決策。

## 政策

在產出、閱讀、呈現或結束調查或視覺審查時，發現每個屬於船長、尚未解決的問題，都必須先由原始工作所屬 home 的權威 backlog 中，一個 captain-held 任務承接，才能將該工作或審查視為完成。
對以 Lavish 看板交接的情況，附加狀態前先將回覆傳入 `bin/fm-procevent-lavish.sh arm --agent-reply-file`；配接器負責依版本而異的接受順序。
優先 hold 該問題所阻擋的工作項目，不要另建新列；只有不存在可 hold 的工作項目時才建立新任務。
原始調查或審查本身絕不是自己的盤點項目，因此應為該決策 hold 另一項任務，並傳入 `--origin <origin-id>`，讓 `complete` 可檢查。
在 hold 原因中放入問題與選項，每個真正關卡保留一項任務：多問題審查是一個指向報告的 held 任務，不是每個問題一列。
用恰好一張看板卡片表示該任務，集中問題與選項；絕不將同一 task id 展開成多張相同 key 的重複卡片。
透過 `bin/fm-captain-hold.sh hold` 登錄或重新 hold，其對每個 task id 具冪等性。
盤點整份報告與審查介面後，firstmate 執行 `bin/fm-captain-hold.sh complete`，帶上每個 captain-held task id；若沒有任何事情等待船長，則用 `--none`。
偵察 Agent 只有透過產生簡報中完成定義的狹義命令例外，才能證明自己的空盤點。
盤點非空或命令被拒絕時，偵察 Agent 向 firstmate 回報，不寫中繼資料或 backlog 紀錄。
已完成調查與已結束視覺審查使用相同負責者與完成命令；包括 Lavish 在內的視覺工具，絕不另有平行完成政策。
命令在原始工作的權威 `FM_HOME` 執行；二副擁有的工作登錄於該二副 home 的 backlog，已在任何地方 hold 的問題絕不重複登錄為第二列。
絕不只因原始調查完成、報告封存、視覺審查結束或任務拆除，就關閉 captain-held 任務。
對問題所阻擋的工作項目採 hold，正因這個理由而安全：清理會保持該列開啟，記錄已完成工作的交付物，並送回佇列，使其仍顯示為船長自己的決策。
只有附船長原話的 `answer`、有證據支持的 `reconcile close`，或下述已宣告 PR 結清政策，才能解決。

啟用 [PR hold 結清](../../../docs/configuration.md#pr-hold-settlement-confighold-pr-settle)時，只有指定 forge 事件能回答整個 held 問題，才宣告 `pr-policy`。
只有純決策任務使用 `close`；任務仍含未完成工作時使用 `release`。
單有結構化 PR 連結絕不授權結清。
貢獻觀察器將事件記為證據，不作為船長回答，並可無須模型回合就結清相符 holds。
未符合政策、且未合併就關閉的 PR 仍是開放問題，會以貢獻通知送達。
呈現剩餘選擇前先讀取此事實，不要詢問是否合併已關閉 PR。
政策登錄、支援事件與重試機制由指令碼 help 負責。

記錄船長回答時，保留他實際說的話：`bin/fm-captain-hold.sh answer` 將原話寫入任務並關閉問題型決策，`--release` 則釋放受船長關卡阻擋的工作項目讓它繼續。
合併核准使用既有 release 路徑，因為核准是允許合併繼續；清理只有在整合完成並記錄出貨內容後才關閉工作。
若在核准合併時就關閉 held 列，便是在整合前記錄完成，使 backlog 宣稱完成早於工作實際交付。
回答改變任務必須建置的內容時，遵循 `AGENTS.md` 第 7 節任務中新增要求規則，將船長原話保存在簡報並引導工作 Agent。
船長說「以後再說」也是回答：用 `bin/fm-captain-hold.sh hold <id> --reason "<reason>" --until <date>` 重新 hold，讓項目離開當下 Captain's Call，並於指定日期再出現，不要留下看似仍需立即處理的卡片，也不要捏造關閉。
「帶 key 的回答解決對應 captain-held 任務」是同一項能力，只有一個負責者 `bin/fm-captain-hold.sh answers`；每個承載船長回答的通道都餵入相同 task id 與回答，通道本身絕不將 keys 對應到任務、記錄決策或自行解決任何事。
聊天已透過 `bin/fm-send.sh --resolve-key` 餵入；擷取回答來源在以 `bin/fm-captain-hold.sh bind <source-id>` 綁定後也會餵入；先 bind 再啟用來源，每個結構化問題都以 held 任務 ID 作 key。
未綁定來源，或 key 未對應任何 captain-held 任務，都只是不餵入任何內容：回答仍被擷取，firstmate 仍被喚醒，關閉改由上面的直接命令處理。
有一個保留回答值不會關閉任何項目：`reconcile` 代表「重新查驗現況」，絕不是「船長已回答」，因此共用接收端會拒絕每個通道傳來的該值，且不建立任何內容。
已綁定擷取來源使用另一介接點：配接器從帶 key 回答省略 reconcile，透過 `reconciles` 輸出選取 task id，一般執行器將其餵入 `reconcile-requests`，接收端驗證來源綁定與本機 captain-held 任務後，才建立持久化看板要求。
因此，遠端二副卡片若任務不在 main backlog，仍會被通知，但無法建立 main-home 要求；能識別擁有者、將要求與異動路由至權威二副 home 的工作另待處理。
看板建立的要求由你在接收回合處理：`bin/fm-captain-hold.sh reconcile close <id> --evidence-file <path>` 記錄**證據**並關閉已無須決策的項目；`reconcile note <id> --note-file <path>` 則註記真正仍有效的決策並維持 hold。
兩種結果都要求該任務仍有船長看板選取建立的待處理要求，否則拒絕，因此都不是可獨立異動船長決策的途徑。
一般船長回答也會退役待處理要求，因為決策已解決，包含 close、release 與冪等重播路徑。
退役失敗會讓命令失敗，但不回復已持久化的回答、關閉或註記，`reconcile list` 會持續顯示殘留要求供重試。
`reconcile list` 列出每個仍未完成要求。
絕不對只有證據顯示已無須決策的項目使用 `answer`：`answer` 記錄船長說的話，`reconcile close` 記錄已驗證證據。
繞過此負責者關閉 captain-held 任務，不會留下持久化回答，因此完成關卡持續失敗，直到 `answer` 記錄船長實際給予的決策。
已解決發現、不需船長選擇的建議，以及只是聽起來像決策的文字，都不建立 held 任務。
Bearings 讀取產生的結構化狀態，絕不可用擷取歷史報告、視覺審查成品、終端機輸出、聊天或其他文字來補償。

同一船長決策可能記兩次：一份是摺合器讀取的帶 key 狀態決策，一份是為船長 hold 的 backlog 任務，而兩份紀錄可能矛盾，兩個介面卻都沒說。
`bin/fm-captain-hold.sh diverged` 回報此矛盾，喚醒 drain 以 `RECORD DIVERGENCE` 印出；它不關閉任何項目，因為錯誤關閉的船長決策會完全離開審查，這比雜訊更糟。
這種行應讀為「兩份紀錄不一致」，絕不讀為「船長已裁示，只是有人忘記歸檔」：決策可能因前提錯誤而不再成立，也可能其實是事實問題，而非需要船長回答。
依實際發生情況對帳：有船長原話可記錄時用 `answer`；若解決並非來自船長原話，就用新的 `needs-decision` 行重新開啟狀態決策。
沒有路由工作項目不算分歧，防護也絕不要求必須有一個：決策本身**就是**交付物時，沒有工作可路由。

## 操作順序

1. 宣稱任一項完成前，先讀完整調查結果，並完成視覺審查。
2. 只盤點真正未解決、需要船長的選擇，找出每個選擇所阻擋的任務。
3. Firstmate hold 該任務，或為審查的開放問題建立一個 captain-held 任務，附上含問題與選項的精簡原因。
4. 依上述政策中的呼叫者權限，證明該次審查完整盤點。
5. 依 `AGENTS.md` 第 9 節，將選擇作為 Bearings 的 Captain's Call 區段決策轉達船長；與船長聊天不要使用 hold 一詞。
6. 透過 `answer`、看板要求的對帳，或上述已宣告 PR 結清政策解決每項決策；船長延後時使用 `--until`。
7. 確認 Bearings 反映結果：已回答或經對帳確定無須決策的項目離開 Captain's Call，釋放的工作恢復，仍有效的對帳項目維持 hold，延後決策則附日期留在 Charted Next。

命令語法、close 模式、舊版身分相容性、完成證明、重試行為與關閉順序由 `bin/fm-captain-hold.sh --help` 負責。
`docs/captain-hold-lifecycle.md` 記錄機制與回歸證據，不重述本政策。
