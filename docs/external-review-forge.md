<a id="external-review-forge-forgeexternal"></a>
# 外部審查平台（forge=external）

有些專案透過專用工具審查程式碼；工具會自行上傳作者本機提交的快照，不進行 git push，也不建立 Pull Request。
在這類專案中，工作 Agent 完全不需要推送分支，也絕不可將任務分支推送到專案共用的 origin，避免同事看到 Firstmate 的交付分支。
本文件描述此流程的設計；`bin/fm-project-mode.sh` 的標頭負責登錄表綁定，`bin/fm-dod-lib.sh` 負責它對工作 Agent 帶來的行為變化，而 [configuration.md](configuration.md#external-review-tool-configexternal-reviewmd-configexternal-review-url-configexternal-review-backup) 負責主目錄設定結構。

<a id="why-it-is-a-forge-value"></a>
## 為什麼將它設為 forge 值

[gerrit-forge-integration.md](gerrit-forge-integration.md) 第 3 節將 forge 定義為發布成品的類型，進而決定由哪個工具建立它。
這裡的成品是外部工具根據本機提交快照建立的審查，因此此流程是既有封閉集合 `forge=` 綁定的第四個值，而非另一個獨立維度。
組合檢驗仍然成立：`local-only` 因不發布任何內容而被拒絕；`direct-PR` 先驗證，再用工具上傳；`no-mistakes` 目前被拒絕，因為管線的 push、pr 與 ci 步驟會推送到共用 origin。
此綁定永不自動偵測，因為沒有任何協定事實能識別「上傳快照的審查工具」；船長（captain）登錄表中的 `[direct-PR forge=external]` 項目就是完整綁定。
工具本身屬於主目錄的私人設定：`config/external-review.md` 保存工作 Agent 的驗證、上傳、讀取審查狀態及停止點指示；`bin/fm-dod-lib.sh` 會將它原樣呈現於完成定義（Definition of done）。

<a id="the-no-push-invariant"></a>
## 禁止推送的不變條件

這項不變條件由三層機制強制執行，依強度由強到弱排列：

1. 每個任務都有 `pre-push` 防護 `bin/fm-git-push-guard.sh`，從唯讀 hooks 目錄執行。該目錄由 `bin/fm-spawn.sh` 透過 `bin/fm-git-strip-ai-trailers.sh install --push-guard` 安裝，即使主目錄保留 AI trailers 也會安裝。
   它拒絕任何推送到 `origin` 或與 origin 任一 URL 相符的目的地、拒絕將任務分支或帶有任務前綴的 ref 推送到任意遠端，也拒絕在備份遠端上推送不屬於備份前綴的 ref，最後再串接儲存庫自己的 `pre-push` hook。
2. 在任何端點建立之前，`bin/fm-spawn.sh` 就會拒絕 `--yolo on`、缺少審查工具轉接器，或備份遠端無法使用的情況，並在任務中繼資料中記錄 `forge=external`、`backup_remote=` 與 `backup_prefix=`。
3. 產生的任務簡報第一條規則會禁止上述所有推送；完成定義則指定審查工具為唯一發布路徑。

就緒回報格式是 `done: review <url> uploaded for review`。
Firstmate 無法觀察、核准或合併該審查，因此 `bin/fm-review-check.sh` 只記錄 `review=` 與 `review_head=`，不會啟動合併監看；無論 yolo 如何設定，審查一律升級交由船長處理。
偵察員（scout）啟動時沒有防護機制與任務紀錄，因此 `bin/fm-promote.sh` 會拒絕將其提升為 `forge=external` 專案的交付工作；這類工作必須從一開始就以交付（ship）任務啟動。

<a id="the-personal-backup-remote"></a>
## 個人備份遠端

工作 Agent 副本的提交，只存在該副本與審查工具的快照中。
`config/external-review-backup` 可指定一個 Git 遠端，指向只有船長能控制的儲存位置，並在 `refs/heads/<backup-prefix><task-id>` 保存持久副本（預設前綴為 `backup/`）。
備份前綴不可為空，也不可等於任務分支前綴或以其開頭，確保持久副本不會被視為交付分支，且防護機制的任務前綴規則能維持無條件生效。
若專案不存在該備份遠端，或其 URL 與 origin 相同，spawn 會拒絕執行。
備份遠端指向何處由船長決定，Firstmate 無法代為管制；因此若專案程式碼不得離開某個網路，備份就必須位於該網路內。

設定備份後，只有 HEAD 已存在於備份遠端命名空間時，才會接受就緒回報；teardown 既有的落地工作檢查，也會將該副本視為可從遠端追蹤 ref 到達。
若未設定備份，只憑審查 URL 就會接受就緒回報，因為快照位於 Firstmate 無法讀取的工具中；`review_head=` 會記錄上傳的是哪個提交。
無論有無備份，teardown 的行為都不變；因此沒有備份時，在已審查內容進入 origin 預設分支之前，會拒絕清理，並保留工作 Agent 副本，直到人類合併審查。

<a id="accepted-residual"></a>
## 接受的殘餘限制

`git push --no-verify` 會略過所有 hooks；不經 Git 而直接寫入 refs 的工具不會觸發 hook；與 origin 使用不同寫法的別名 URL，例如相同伺服器的 ssh 與 https 形式，也不會被判定為相等。
這三種方式都能繞過防護，但任務簡報的第一條規則仍禁止它們。
此 forge 的任何關卡或清理步驟都不要求推送，因此消除了工作 Agent 為完成流程而嘗試推送的壓力。

<a id="deferred"></a>
## 延後處理

後續項目包括：`no-mistakes` 組合（略過管線中面向 forge 的步驟，並像 Gerrit 一樣恢復保管責任）、驗證上傳快照是否符合 HEAD 的轉接器 hook、每個專案各自的轉接器，以及將偵察員提升到此 forge 的支援。
