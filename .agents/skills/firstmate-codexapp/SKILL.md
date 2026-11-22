---
name: firstmate-codexapp
description: >-
  為 Firstmate 工作建立、閱讀、指導、封存、除錯或審查可見的 Codex App 對話串前，以及回應將 Codex App 原生整合到 Firstmate 的要求前載入。
user-invocable: false
metadata:
  internal: true
---

# firstmate-codexapp

<a id="overview"></a>
## 概覽

Firstmate 工作需要可見 Codex Desktop 對話串時，使用此操作手冊。
目前支援的是 Desktop 主機工具協調，加上明確的狀態檔回傳通道檢查，而不是在 `FM_BACKEND` 選擇 `codex-app`。

<a id="boundary"></a>
## 邊界

Codex Desktop 可見對話串，是輔助的主機工具工作流程，不是可選的 Firstmate 後端。
此 checkout 若有 `docs/codex-app-backend.md`，請閱讀；驗收契約、橋接需求、狀態回傳要求與分階段推出，都以該文件為準。

若有 Codex App 工作的本機輔助指令碼，只能使用操作者明確提供，或由 Firstmate 維護的工具。
`bin/` 以外的工具，執行 `--help` 前先檢查原始碼或標頭。

<a id="preflight"></a>
## 預檢

1. 確認此工作階段在 Codex Desktop 內執行，且已提供主機工具。
   需要時搜尋精確名稱：`create_thread`、`list_threads`、`read_thread`、`send_message_to_thread`、`archive` 與 `set_thread_archived`。
2. 確認目標儲存庫已儲存為 Codex Desktop 專案。
   目前沒有主機工具可替 Agent 建立 Codex App 專案，因此人類必須先在 Desktop 加入專案，建立的對話串才能可靠地落在該專案。
3. 不要為儲存庫工作建立沒有專案的對話串。
   專案不存在時，停止並請使用者加入，或改用一般 Firstmate 後端。
4. 判斷這是真正由 Firstmate 管理的工作，還是可見的輔助對話串。
   真正工作需要工作 ID、隔離 worktree 或由 Desktop 管理的 cwd、分支計畫，以及可寫入的 `state/<id>.status` 路徑。

<a id="create-and-send"></a>
## 建立與傳送

建立可見對話串時，使用 Desktop 主機工具，不以 Shell 模擬。
指定已儲存專案，並要求工作 Agent 先回報：

```text
pwd
git rev-parse --show-toplevel
git branch --show-current
git log --oneline --max-count=3
```

可寫入的儲存庫工作，應指示工作 Agent 使用 Codex 建立的目前目錄。
不要叫它 `cd` 到已儲存專案的 checkout 進行編輯、提交、no-mistakes、推送或 PR 工作。

傳送後續指示時，使用 `send_message_to_thread`。
使用者若直接在可見對話串輸入，視為有權威的指示，透過 `read_thread` 對帳，不要撤銷它。

<a id="status-return-channel"></a>
## 狀態回傳通道

只有提示提供絕對路徑，且 Desktop 權限上下文能寫入該 checkout 時，Desktop 管理的 Codex 對話串才能追加 Firstmate 狀態檔。
因此狀態寫入是必須驗證的回傳通道要求，不是可以假設的事實。

對 Firstmate 管理的工作，加入明確狀態指示：

```text
將監督者可見的狀態行追加到 <absolute-firstmate-home>/state/<task-id>.status。
狀態變更只使用以下前綴：working:、needs-decision:、blocked:、paused:、done:、failed:。
依工作簡報的狀態回報規則宣告與解除等待；該規則由 bin/fm-brief.sh 定義。
開始實質工作前，追加 "working: Codex Desktop thread started"。
```

認定對話串已受監督前，先驗證回傳通道：

- `read_thread` 顯示工作 Agent 嘗試寫入狀態。
- 本機 `state/<task-id>.status` 檔案包含預期行。
- 若可取得，對話紀錄中包含該狀態檔的檔案變更項目。

對話串若無法寫入狀態檔，就只將它保留為可見的輔助對話串。
不要宣稱它是完整 Firstmate 後端。

<a id="observe-and-reconcile"></a>
## 觀察與對帳

對話串事實以 `read_thread` 為準。
`list_threads` 只用於尋找或復原可見對話串 ID，不能取代閱讀對話紀錄。

Firstmate 對帳時，優先使用具體證據：

- 對話串 ID 與專案
- 目前由 Desktop 管理的 cwd
- 分支名稱
- 最後有意義的對話串狀態
- 最新狀態檔行
- 存在時的 PR URL

避免把長篇對話紀錄重複貼進 Firstmate 文件或 PR 內文。
只摘要主機工具呼叫、狀態檔結果與封存結果。
向船長回報 Desktop 對話串成果時，依 `AGENTS.md` 第 9 節，將狀態前綴與回傳通道證據轉成易懂說明。

<a id="archive"></a>
## 封存

透過 Desktop 主機工具封存：提供的基本操作若是 `archive` 就使用它，若工具名稱為 `set_thread_archived`，則使用 `set_thread_archived(threadId=<id>, archived=true)`。
封存可將對話串移出一般側邊欄／專案檢視，但不應刪除對話紀錄或已合併的工作。

輔助對話串可直接封存，並回報持久工作保存在哪裡。
如果已有真正 Firstmate 工作紀錄，清理決策應交回一般 Firstmate 工作流程，不由本技能決定。

<a id="failure-signals"></a>
## 失敗訊號

- 缺少 Desktop 專案：請人類在 Codex Desktop 加入目標專案，或使用一般後端。
- 缺少主機工具：不要用 Shell 檔案模擬，改用終端機後端。
- 狀態檔未更新：在回傳通道獲得證明前，將對話串視為未受監督。
- 工作 Agent 編輯已儲存專案 checkout，而非其 Desktop cwd：停止，先決定是否保留該分支，再繼續。
- 要求正式的 `codex-app` 後端：閱讀 `docs/codex-app-backend.md`，不要自行發明本機介接器。
