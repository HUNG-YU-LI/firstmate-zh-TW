---
name: firstmate-self-repo
description: 派發、提升、重新啟動或交付目標為 firstmate 儲存庫本身的工作前，一般使用中出現 firstmate 問題或改善想法時，授予或撤銷上游發布權限前，以及判斷任何推送、pull request、Issue、留言或 fork 是否能送達 firstmate 自身上游前載入。
user-invocable: false
metadata:
  internal: true
---

# firstmate-self-repo

每個 firstmate home 都複製同一個公開儲存庫並從中執行，因此 firstmate 自身上游，是執行中的 home 一定能存取的儲存庫。
多數 home 屬於單純使用 firstmate 的人，而非維護者；執行中的 Agent 若把「我看得到儲存庫」當成「我可以發布到它」，就會讓使用者的 firstmate 意外向上游開 PR。
本技能負責區分這兩種關係的邊界。

<a id="the-default-useroperator"></a>
## 預設：使用者／操作者

沒有明確授權時，home 以 firstmate **使用者／操作者**身分執行。
它可以安裝、執行、更新、重新設定及在本機修復 firstmate，一般專案工作也照原本登錄的交付模式進行。
它絕不能自行向 firstmate 上游發布：不得推送分支、建立 pull request、開 Issue、留言、fork 或提交審查。

角色**絕不依推測**。
GitHub 身分、託管平台權限、`origin` 或任何 remote 的名稱與 URL、儲存庫所有權、驗證狀態、環境變數與 CI 上下文，都不能說明角色；唯一輸入是下列明確本機授權。

<a id="when-a-firstmate-bug-surfaces-in-ordinary-use"></a>
## 一般使用中發現 firstmate 問題

使用者／操作者 home 遇到 firstmate 問題，只做本機工作：

1. 私下診斷：在 home 與其工作紀錄內重現、隔離並找出根因。
2. 有替代方案時，在本機套用，例如修改設定、固定本機編輯，或使用不同呼叫方式。
3. 可選擇在工作資料目錄下準備去除敏感資訊的本機 Issue 草稿，或提出本機修補，供操作者審查並手動提交。
4. 向船長回報發生什麼，以及產出了什麼。

本機修補或替代方案可碰觸列入 gitignore 的操作內容與該工作自己的 worktree；未取得下述授權，任何內容都不得到達上游。
此模式對 firstmate 自我修復與派發工作一視同仁：邊界在於發布，不在於修補存放在哪個位置。

<a id="the-two-explicit-grants"></a>
## 兩種明確授權

<a id="maintainer-opt-in-standing"></a>
### 維護者啟用（持續有效）

只有操作者能將主要 home 設為啟用；遵循 [Firstmate 角色結構與解析政策](../../../docs/configuration.md#firstmate-role-configfirstmate-role)，不要代操作者建立授權。

<a id="one-task-contribution-consent-bounded"></a>
### 單次工作貢獻同意（有界）

船長明確核准一項特定上游貢獻時，只記錄在該工作：將 `--upstream-consent <host>/<owner>/<repo>` 傳入 `bin/fm-brief.sh`、`bin/fm-spawn.sh` 與 `bin/fm-promote.sh`。
同意會呈現在簡報 `Delivery contract:` 行的 `upstream-consent=` 代碼中，記錄於 `state/<id>.meta`；重新啟動沿用已記錄值。
它完全不改變 home 的持久角色；下一項工作重新套用關卡。
船長核准必須是當前、明確點名該貢獻的指示，絕不能因工作看起來像維護工作就推定。

<a id="where-the-boundary-is-enforced"></a>
## 邊界的執行位置

關卡由程式碼執行，不靠記憶：`bin/fm-role-lib.sh` 的 `fm_role_gate_self_publish`，對發布契約套用[儲存庫血緣與發布目標政策](../../../docs/configuration.md#firstmate-role-configfirstmate-role)。

- `bin/fm-brief.sh --project <dir>` 會拒絕產生 home 無權授權的發布型 ship 簡報。
- `bin/fm-spawn.sh` 在沒有維護者範圍或相符工作同意時，拒絕在此儲存庫啟動 ship；也會拒絕簡報 `upstream-consent=` 與旗標不一致的情況。
- `bin/fm-promote.sh` 對 scout 提升套用相同關卡，確保原位提升不比全新啟動多取得權限。
- `local-only` 不發布任何內容，因此不需要發布授權，也不接受貢獻同意。

已授權貢獻接著和其他專案一樣，走該工作的一般交付契約，通常是 no-mistakes；授權只回答「能否發布」，不回答「如何發布」。

關卡拒絕會指出精確補救：在 `config/firstmate-role` 寫入 `maintainer <scope>`，或取得船長明確核准後，只對該工作傳入 `--upstream-consent <scope>`。
絕不把工作改指向第二份 checkout、fork 或上游 clone 來繞過拒絕；`fm_role_self_repo` 將與 firstmate 同血緣的所有儲存庫都視為 firstmate 自身。

<a id="updating-is-not-contributing"></a>
## 更新不等於貢獻

`/updatefirstmate` 與每個受防護同步，都是**從**上游拉取，屬於使用而非發布，所有角色都仍可執行。
這道邊界只涵蓋寫入上游儲存庫的流量。

<a id="secondmates"></a>
## 二副

遵循[角色繼承與接收限制](../../../docs/configuration.md#firstmate-role-configfirstmate-role)；支援的傳遞方式，絕不授予子 home 高於主要 home 實際解析出的角色。
刻意偽造的本機紀錄，不在這道避免意外發布的防護範圍內；它不是來源真實性保證。
