---
name: updatefirstmate
description: >-
  將執行中的 firstmate 與其二副自行更新至 origin 最新版本，或設定的自行更新來源。
  當船長呼叫 /updatefirstmate（例如「/updatefirstmate」、「update firstmate」、「pull the latest firstmate」）時使用。
user-invocable: true
metadata:
  internal: true
---

# updatefirstmate

就地自行更新 firstmate。
Firstmate 本身也是儲存庫，遵守與任何專案相同的 no-mistakes 關卡，因此新的受版本控制資料（`AGENTS.md`、`bin/`、`.agents/skills/` 與公開 `skills/`）進入 `main` 後，會留在那裡，直到每個執行中的 firstmate 拉取。
只有 `AGENTS.md`、`bin/` 與 `.agents/skills/` 是執行中 firstmate 的指示介面；公開 `skills/` 面向安裝器，不由 firstmate 載入。
此技能替執行中的主 firstmate 與每個二副完成拉取，不干擾任何進行中工作。
更新是取用，不是發布：它只從上游拉取，因此每個 home 不論角色都會執行，firstmate 儲存庫發布邊界（`firstmate-self-repo`）絕不適用。

拉取檔案只完成一半。
執行中的 Agent 持有啟動當下凍結的 `AGENTS.md` 與已載入技能，而沒有任何已驗證執行環境提供重新載入，因此磁碟上新位元組不會改變其行為，直到開始新對話。
重新閱讀無法取代：它只會附加第二份二副自己的職務說明，沒有明確優先序，也無法觸及已載入技能。
替換 Agent 也是唯一能重新解析啟動時接線的方式，包括回合結束 hooks、執行環境旗標、各執行環境功能開關；這些在二副啟動時就凍結，且磁碟上沒有任何檔案能描述它們。

因此，**每個存活二副在更新成功後都會重新啟動，包括原本已位於目標提交者。**
啟動時接線無法從檔案差異推導，所以受版本控制內容未變，不代表執行中的 Agent 已使用目前行為。
只有更新流程必須略過其 home，或執行階段無法證明可重新啟動的存活二副不會重啟；更新器如實保留這兩種情況，都不回報為已重新載入。

**一次性推出注意事項：**帶入此變更的更新仍由前一版執行，它只會重新啟動該次 `AGENTS.md` 或 `.agents/skills/` 有變動的二副。完成後，請用每個存活二副 ID 執行一次 `bin/fm-secondmate-restart.sh <fm-id>...`，不只傳入該版本列出的 ID；後續更新依下方正常流程。

來源預設為 `origin`，除非此 home 有 `config/self-update-source`，其中指定一個遠端分支供主 Agent 改為追蹤，本機二副則跟隨主 Agent 最終提交；此契約與遠端二副限制由 [`docs/configuration.md`](../../../docs/configuration.md#self-update-source-configself-update-source) 負責。
主 Agent 更新只允許 fast-forward；每個二副使用相同受防護收斂路徑，另加一個針對 squash 合併本機歷史的狹義復原方式。
遠端路由會從該主機設定的 Firstmate 程式碼根目錄自己的 origin 更新，再受防護地將持久 home fast-forward 至該程式碼根目錄提交。
絕不強制、絕不建立合併提交，也絕不 stash。
只有三方樹證明顯示完整本機結果已存在於目標時，乾淨的二副分歧才可用 `reset --keep` 前進；這能辨識已 squash 合併的貢獻，而不丟棄獨有內容。
其他所有 dirty、分歧、離線或錯誤分支目標都會略過並回報；真正分歧會留下持久化 `state/.secondmate-update-reconcile/<id>.pending` 紀錄，供未來 bootstrap 與更新流程持續呈現，直到收斂清除。
受追蹤檔案的 fast-forward 不碰 gitignore 忽略的操作目錄（data/、state/、config/、projects/、.no-mistakes/），因此絕不干擾二副進行中工作。
只會碰 firstmate 儲存庫與自己的 worktrees，絕不碰 `projects/` 內任何內容。

<a id="what-it-does"></a>
## 執行內容

1. **執行更新器：**
   ```sh
   bin/fm-update.sh
   ```
   它會從 origin 或設定的自行更新來源，將此 firstmate 儲存庫預設分支 fast-forward，再透過各部署位置專屬的受防護路徑，更新所有已登錄本機或遠端二副 home。
   有設定來源時，會先印出 `self-update source:` 行；設定檔不可用會在任何變動之前停止整個更新，應向船長回報該檔案，不要編輯或移除。
   每個目標會印出一行狀態（`updated <old>..<new>` / `reconciled redundant divergence <old>..<new>` / `already current` / `skipped: <reason>`），最後三行動作精確告訴你下一步：
   - `reread-firstmate: yes|no`
   - `restart-secondmates: fm-<id>...|none`
   - `nudge-secondmates: fm-<id>...|none`

   兩個二副集合互斥，分組由指令碼負責，不要自行重新推導。
   `restart-secondmates:` 包含該次流程結束後位於最新提交的每個存活二副，不論剛前進或原本就在那裡。
   二副只有在 home 被略過、此處沒有記錄存活端點，或端點已確定分類為死亡／遺失時，才不列入任何集合。
   被略過的真正分歧仍需透過持久化對帳紀錄處理；其他兩種情況不需要你執行更新動作。

2. **自己的指示有變更時，重新閱讀 AGENTS.md。**
   更新器印出 `reread-firstmate: yes` 時，受版本控制的指示介面（`AGENTS.md`、`bin/` 或 `.agents/skills/`）剛在你執行期間前進。
   **現在就閱讀 `AGENTS.md`**（CLAUDE.md 是真正的 `@AGENTS.md` 指標），先更新操作指示再做其他事，讓你依新指示行動，不沿用啟動時的過期版本。
   印出 `reread-firstmate: no` 時，你的內容未變，略過重新閱讀。

3. **重新啟動更新器指定的每個二副。**
   將完整 `restart-secondmates:` 清單傳給一個命令（若為 `none`，整個步驟略過）：
   ```sh
   FM_HOME=<this-firstmate-home> bin/fm-secondmate-restart.sh <fm-id>...
   ```
   除非 `FM_HOME` 已指向目前作用中的 firstmate home，否則必須包含 `FM_HOME=<this-firstmate-home>`。
   此步驟自動執行，不需要船長逐個二副確認。
   本機與遠端二副放同一份清單；命令負責傳輸、各替換 Agent 的設定檔，以及等待。

   它會先要求每個列出二副，寫下只存在於對話中的未完成工作，只有收到該二副自己回答後才重啟。
   二副若正在回合中，要求會排在該回合之後。
   這正是此步驟的目的，不要繞過：它可避免二副已形成但尚未登錄的船長決策，隨對話遺失。
   請求、時限與調整兩者的兩個參數，由其檔頭負責。

   以逐二副行與結尾 `summary:` 行判讀結果：
   - `restarted: <id>`——該二副現在確實執行目前指示與啟動時設定。
   - `nudged: <id>: <reason>`——無法安全重新啟動，因此改送舊版重新閱讀訊息；二副仍執行原對話與啟動時設定。
     絕不將此回報為乾淨重新載入。
   - `unreached: <id>: <reason>`——無法確認安全執行結果，包括不明確的重新啟動結果。

4. **對其餘二副傳送重新閱讀訊息。**
   對 `nudge-secondmates:` 行的每個目標（若為 `none` 則不做任何事），傳送單行重新閱讀引導：
   ```sh
   FM_HOME=<this-firstmate-home> bin/fm-send.sh <id> 'firstmate was updated to the latest - please re-read your AGENTS.md to pick up the new instructions.'
   ```
   這些二副磁碟內容已是最新，但無法證明可重新啟動，因此引導是此流程能如實做到的上限。
   這是溫和引導，不是中斷：二副已安全完成受追蹤檔案 fast-forward，引導絕不強制、拆除或丟棄其工作。
   絕不稱它們已重新載入；Agent 仍執行啟動時接線。

5. **用直白結果向船長回報，能一行說完就一行。**
   依 `AGENTS.md` 第 9 節摘要實際完成內容，不使用 firstmate 內部詞彙：艦隊哪些部分現在已是最新，哪些保持原樣，以及原因。
   例如：「船長，firstmate 與兩個二副現在都已更新到最新版。」
   若二副只收到訊息，沒有乾淨重新載入，明確說明情況與原因；絕不讓部分重新載入讀起來像完整完成。
   提出任何需要船長注意的略過目標，例如 home 有尚未整合的自身變更（分歧）或本機編輯（dirty），這些是刻意保留不動。

<a id="safety"></a>
## 安全

- **只允許受防護收斂。**
  dirty、離線、非預設分支，或有獨有分歧的目標都會略過並回報，絕不強制或 stash。
  只有完整本機結果已存在上游的乾淨二副分歧，才可不依祖先關係前進，而 `reset --keep` 仍拒絕衝突的工作樹變更。
  絕不丟棄任何尚未整合的工作；這是最高指令 #3。
- **只碰 firstmate 儲存庫與其 worktrees**，絕不碰 `projects/`。
  這與艦隊同步是同一種核准的自行寫入。
- **任何有工作的內容都不受干擾。**
  本機或遠端二副只有在自己的 checkout 能安全前進時，才進行受追蹤檔案 fast-forward；home 被略過的二副也不重啟。
  重新啟動是在未完成工作寫下後，於同一 home 與端點替換該二副 Agent，絕不是拆除，也絕不強制。
  其船員持續在各自端點執行，每份持久紀錄——backlog、保留中的船長決策、未讀狀態、未處理指示——都會在啟動時重新呈現給替換 Agent。
  重啟在嘗試前被拒絕時，二副維持重新閱讀路徑；一旦嘗試 relaunch，任何失敗或不明確結果都回報為未知，不歸因於任一實例。
