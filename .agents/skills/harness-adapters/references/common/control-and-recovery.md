<a id="control-and-recovery"></a>
# 控制與復原

處理信任、技能呼叫、中斷、結束、恢復或復原時，搭配正在執行或已記錄工具的參考載入。

<a id="typed-data-and-lifecycle-control"></a>
## 型別化資料與生命週期控制

僅限生命週期的控制，以及已記錄執行環境的選擇，由路由器負責。
對話與執行環境原生技能呼叫，使用 `../../../bin/fm-send.sh`。
資料平面分工以 `../../../docs/agent-control.md` 為準，可執行能力則由 `../../../bin/fm-control-lib.sh` 定義。
工具參考中的結束與中斷值，是實測紀錄，不是可臨場猜測的按鍵；新介接器的這些值合併到負責元件前，仍不可控制。
讓控制平面驗證後置條件。

<a id="trust-and-skill-submission"></a>
## 信任與技能送出

啟動後，在工具就緒時間窗內檢查。
從目前 Firstmate home，只選擇該工具文件記錄的信任選項；除非已正確設定，否則綁定 `FM_HOME`，再依路由器定義的完成後置條件重新檢查。
未觀察到對話框，只能證明該次啟動沒有出現。

各支援執行環境處理資料夾信任關卡的方式不同，細節以各工具參考為準。
Claude 請載入 `references/harness/claude.md`；其工作區信任區段定義不能用按鍵回答的關卡，以及各種啟動類型在啟動時的預先登錄。
agy 也會對每個全新 worktree 設關卡；啟動程序以同樣方式預先登錄到 agy 自身儲存區，而啟動後的嚴格關卡會在回報成功前，回答仍然顯示的任何對話框。
Cursor 透過啟動時 `--trust` 抑制對話框，Muse 則使用 `--yolo`。
Kiro 透過啟動程序在每項工作的 `KIRO_HOME` 植入 `chat.disableTrustAllConfirmation` 設定，抑制 `--trust-all-tools` 確認視窗；流程與決策保存位置，以 `references/harness/kiro.md` 為準。
Grok 會在連結 worktree 中顯示資料夾信任關卡；如何驗證工作 Agent 真正位置、回答關卡，以及決策保存位置，以 `references/harness/grok.md` 為準；專案選擇器是另一個獨立對話框，啟動從 Git 根目錄開始時不會出現。
Pi 也會對全新 worktree 設關卡，但不同於 Claude，其對話框可用 Enter 回答；流程與決策保存位置，以 `references/harness/pi.md` 為準。
Codex 在某儲存庫根目錄首次執行時，會顯示目錄信任對話框。

使用工具精確的技能形式；只有未驗證獨立指令，或形式仍不確定時，才使用自然語言。
成功傳送或按鍵返回，不代表提交成功；必須符合工具專屬後置條件。
彈出視窗、排隊輸入與就緒處理，由 `../../../bin/fm-composer-lib.sh` 及選定後端負責。

<a id="interrupt-and-exit"></a>
## 中斷與結束

使用控制平面，先檢查能力。
Interrupt 保留 Agent 與工作；exit 只停止 Agent，保留端點、隔離副本與未提交變更。
清理與捨棄不是生命週期動詞。
工具參考記錄重複、確認與清除行為，可執行元件則傳送或拒絕該序列。

<a id="resume-and-recovery"></a>
## 恢復與復原

原生恢復的可用性與形式，只以選定工具參考為準。
只有該參考與復原程序都要求時，才使用原生恢復。
具確定性的重新啟動，則相信磁碟上的指示，而非私人工作階段，也不需要結束時印出的工作階段 ID。
重新啟動時唯一例外，是執行階段自身已記錄的工作階段身分；它只用來維持替代程序前後的狀態權威有效，規則以 `../../../docs/agent-control.md` 的「Transactional relaunch」為準。

工作 Agent 復原由 `../stuck-crewmate-recovery/SKILL.md` 負責，二副復原由 `../secondmate-provisioning/SKILL.md` 負責；兩者都保留已記錄工作。
路由器的復原情境，會為替代設定檔與二副選擇額外共用參考。
