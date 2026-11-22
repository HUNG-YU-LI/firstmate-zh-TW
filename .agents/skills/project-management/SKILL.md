---
name: project-management
description: >-
  新增、建立、移除或初始化專案前載入。
  Clone 或登錄專案也屬新增接案，使用相同觸發條件。
user-invocable: false
metadata:
  internal: true
---

# project-management

新增、建立、移除或初始化專案前，使用此程序。
Clone 或登錄專案也屬新增接案，使用相同觸發條件。
此技能是 Firstmate 專案管理程序的唯一負責來源。
它不取代 `secondmate-provisioning`，後者負責持久二副 home 內的專案 clones。

## 前置條件與登錄表

專案平放在 `projects/` 下，`data/projects.md` 是私有艦隊登錄表。
使用 `bin/fm-project-mode.sh` 檔頭負責的登錄格式與解析器契約。
每個登錄描述應有助辨識專案，但交付方式、船長私有狀態與詳細專案知識，要保留在既有指定位置。
不要把登錄表變成專案文件。

在 main home 新增、clone、建立或登錄任何專案前，檢查權威 `data/secondmates.md` 路由表，並逐一判斷每個既有自然語言 `scope:` 是否涵蓋擬議專案或領域。
套用 `AGENTS.md` 第 7 節權威二副路由規則；既有 scope 若擁有該領域，將新專案操作或工作路由過去，不要在 main home 建立或登錄重複 clone。
main 的 `data/projects.md` 登錄表沒有該專案，絕不代表沒有二副擁有該領域。
所屬二副若無法接受路由，回報具體阻擋，或取得船長明確重新指派，不要默默在 main home 複製專案。

變更本機或遠端狀態前，先確定專案名稱、目的地、交付方式與自主性設定。
新加入 clone 與登錄項目保持一致；後續初始化步驟失敗且可安全回復時，只回復該未完成操作建立的成品。
不要覆寫或改作他用既有路徑。

## 交付方式

登錄表記錄專案的長期交付方式與選用 ship 分支前綴，這是船長預設，不是任何單一任務的答案。
每項任務具體 mode、yolo 與分支前綴如何在接案時解析，並明確傳給簡報、spawn 與任何升級，由 `AGENTS.md` 第 7 節負責。
新增或建立專案時選定交付方式：

- `no-mistakes` 在 PR 前執行完整驗證 pipeline。
- `direct-PR` 直接 push 並開 PR，不執行 no-mistakes pipeline。
- `local-only` 不要求 remote 或 PR，只透過核准的本機 fast-forward 路徑整合。
- `no-mistakes-prod-only` 是條件政策，不是單一固定模式：真正僅限內部的工具、 automation、貢獻者或操作人員流程，以及發布或提交工作，使用 `direct-PR`；面向產品、混合與不確定工作使用 `no-mistakes`。

船長未指定時，新加入或建立、具有遠端的專案預設 `no-mistakes-prod-only`，無遠端專案預設 `local-only`。
確認來源、本機名稱與方式時，直接說明解析後預設，不要要求船長從零開始選；船長要求固定模式時，就記錄該模式。
既有登錄項目維持原有意義，絕不遷移或重新解讀，因此沒有方括號的舊項目仍是 `no-mistakes`。
登錄條件政策是一次性選擇，絕不需要分類任何變更；逐任務介面分類在每次接案進行，絕不從檔案位置或專案名稱推斷為僅限內部。

選用 `+yolo` 只改變合併權限，不改變交付模式。
每個專案、每種方式預設關閉，只有船長明確指示才啟用。
合併權限契約由 `AGENTS.md` 第 7 節負責。

選用 `forge=` token 記錄專案遠端實際是哪種 forge；值為 `forge=gerrit`、`forge=forgejo` 與 `forge=external`。
它與 mode 和 `+yolo` 互相獨立，因此絕不從兩者推導，也絕不在使用時從 remote 名稱、主機、連接埠或 push 目標推斷。
新增或建立接案時，clone 存在後執行一次 `bin/fm-forge-detect.sh projects/<name>`，將答案與交付方式一起提出；船長確認才建立綁定，登錄 token 是該確認的持久紀錄。
絕不只憑偵測就登錄綁定，之後也絕不從 clone 重新推導。
`bin/fm-forge-detect.sh` 絕不提出 `forge=forgejo`，所以遠端可能是 Forgejo 或 Gitea 時，要詢問船長，只有確認後才登錄 `forge=forgejo`。
forge 可搭配 `no-mistakes`、`direct-PR` 與 `no-mistakes-prod-only`（`forge=external` 只搭配 `direct-PR`）；登錄表拒絕在不發布任何內容的 `local-only` 使用 forge，保留本機的 Gerrit 託管專案應登錄 `local-only`，不帶 forge token。
`forge=gerrit` 專案的 `yolo` 不生效，因此絕不一起提出 `+yolo`。
`forge=external` 標示審查工具會自行上傳快照的專案，因此工作 Agent 絕不將任務分支 push 到共用 origin；它永遠不會被偵測出來，所以只有船長確認後，才以 `[direct-PR forge=external]`、不帶 `+yolo` 登錄，而且 home 必須已有 `config/external-review.md`（`docs/configuration.md` 的「外部審查工具」）。
綁定由 `bin/fm-project-mode.sh` 檔頭負責，對工作 Agent 造成的變更由 `bin/fm-dod-lib.sh` 負責。
選用 `pr=manual` token 標示 PR 標題與內文由人員撰寫的 `no-mistakes` 專案，因此 pipeline 的 `pr` 步驟不可改寫；只有船長確認才登錄，且不帶 `forge=` token（`docs/configuration.md` 的「PR 文字選擇」）。

## 新增或 clone 既有專案

確認來源 URL、本機專案名稱、交付方式與自主性設定，說明各項解析後預設，不要要求船長自行想一套。
確認目的地未使用後，才 clone 到 `projects/<name>` 並新增登錄項目。
`no-mistakes` 或 `no-mistakes-prod-only` 專案必須有 `origin` remote，且完成下述初始化程序，因為條件政策的產品工作會跑 pipeline，僅限內部工作仍走直接 PR。
`direct-PR` 專案需要 `origin` remote，但略過 no-mistakes 初始化。
`local-only` 專案可以沒有 remote，並略過 no-mistakes 初始化。
非 GitHub 主機需確認對應 CLI 已驗證：GitLab 專案使用 `glab auth login`，其合併監看與合併使用 glab；`forge=forgejo` 專案使用 `tea login add`，其合併監看、合併與產生的 ship 簡報使用 tea。

## 建立專案

建立 GitHub 儲存庫是對外操作。
進行該遠端變更前，先提出儲存庫名稱、擁有者或組織、可見性與交付方式，預設可見性為 private，交付方式為 `no-mistakes-prod-only`，再取得船長對這些精確值的明確同意；說明預設值絕不取代同意。
核准的 GitHub 操作使用 `gh-axi`，並查看目前 help，不依賴記憶中的旗標。
遠端建立成功後，clone 到本機、新增登錄項目，並依交付方式初始化。

純 `local-only` 專案在未使用的 `projects/<name>` 路徑下建立本機 Git 儲存庫、新增登錄項目，不呼叫 GitHub。
新儲存庫沒有提交，而 spawn 拒絕向尚未建立 `HEAD` 的專案派工。
告訴船長，專案需要第一個提交。
船長要求建立該本機專案，已授權此本機初始化，但不授權未提及的遠端儲存庫。

## 初始化

只對 `no-mistakes` 與 `no-mistakes-prod-only` 專案執行 no-mistakes 初始化：

```sh
(cd projects/<name> && no-mistakes init && no-mistakes doctor)
```

初始化設定本機關卡，不會將 no-mistakes 技能納入專案。
不要只因執行初始化就建立提交。
doctor 若回報環境、驗證或 daemon 問題，派工前先解決，絕不因專案操作重新啟動共用 daemon。

## 移除

專案移除是破壞性操作。
先取得船長明確移除決策，再檢查目前摘要與權威儲存庫，確認進行中或排隊工作、已登錄二副 clones、連結 worktrees、dirty 檔案、未 push 提交，以及其他任何未整合工作。
若有任何相依關係或未整合工作，在變更前停止並回報。
絕不由 Firstmate 直接發出原始移除命令。
前置檢查確認上述皆無，且船長核准已具體時，AGENTS.md 硬性規則 1 的船長核准專案操作例外，授權 firstmate 直接移除 clone 並同步更新登錄項目。
clone 已透過核准移除，或可證明沒有 clone、登錄已過期時，移除其登錄行，讓導覽符合現況。
