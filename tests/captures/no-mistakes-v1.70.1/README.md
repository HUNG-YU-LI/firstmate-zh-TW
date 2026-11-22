<a id="axi-run-state-input-captures"></a>
# AXI 執行狀態輸入擷取紀錄

這些檔案保存序列化 CLI 輸入與持久化的執行清單投影，供 `../../fm-crew-state.test.sh` 中的 `test_captured_*` 案例使用。
擷取時間為 2026-09-14 22:06 UTC，版本為 `no-mistakes version v1.70.1 (9c380d4) 2026-09-07T20:47:58Z`。
它們是重播輸入，不代表每個組合情境都曾在真實環境執行。

<a id="capture-provenance"></a>
## 擷取來源

每個狀態檔案都是在測試階段 worktree 中執行 `no-mistakes axi status --run <id>` 所得、未經修改的 stdout，未進入被記錄執行的 checkout。
五份特定執行的擷取，其指令結束碼都是零。
`uninitialized.toon` 是同一 worktree 中執行 `no-mistakes axi status` 得到、未經修改的 stdout，結束碼為一。
stderr 的更新通知不屬於擷取 stdout 的契約。

| 檔案 | 記錄的執行 ID | 觀察到的狀態 |
| --- | --- | --- |
| `replacement.toon` | `01M2GAWMSDQK4B5EA9GZW35RXE` | 重新執行後，CI 步驟仍在進行 |
| `superseded.toon` | `01M2FNFPK984YP0EHFTD1XEF8P` | 同分支的前次執行以 `superseded by new push` 取消 |
| `parked.toon` | `01M20NDQH0G96AQYH1EHWGKT5F` | 另一分支停在測試關卡，含一項發現 |
| `failed.toon` | `01M289YXN7V0513AKCF53MJ1BC` | 推送步驟失敗 |
| `completed.toon` | `01M2FG7SEEP1VBZ3B5SX35BJ6Q` | 驗證完成 |

`same-branch-inventory.json` 保留替代執行所屬分支的全部九筆資料，以唯讀交易從實際 `state.sqlite` 資料庫選出。
投影欄位為 `id, repo_id, branch, status, head_sha, created_at`，依 `created_at DESC, id DESC` 排序。
來源中，儲存庫 `acf4a767348a` 共有 78 次執行；其 `runs` 結構確認了測試資料中的文字身分／狀態／head 欄位，以及整數建立時間。
擷取當時，該儲存庫沒有任何分支同時記錄兩次進行中的執行。

`overview.toon` 是實際 `no-mistakes axi` 執行檔對這 78 筆執行紀錄的隔離資料庫副本輸出、未經修改的 `count` 與 `runs` 區段。
只有副本內儲存庫的 `working_path` 被移到允許的 worktree；沒有初始化或控制任何管線。
副本省略步驟資料，且沒有常駐程序，因此輸出中無關的進行中執行細節被刻意排除。
保留的區段呈現實際十筆上限、資料順序、引號與欄位配置。
被排除的標頭也包含總覽頂層的 `repo:` 行，也就是受上限限制的清單讀取器用來識別儲存庫的解析後 `working_path`；因此此區段缺少該行，不能推論真實輸出也沒有。
原始 stdout、來源投影與 SHA-256 摘要保留在測試階段證據目錄的 `real-anchors/` 下。

<a id="replay-transformations-and-limits"></a>
## 重播轉換與限制

`captured_axi_status` 只替換執行 ID、分支與 head 欄位，讓擷取資料綁定到可拋棄的 Git 儲存庫。
狀態、結果、步驟、發現與關卡的位元組保持不變。
清單重播會替換成可拋棄儲存庫的鍵與路徑，保留同分支所有擷取資料，並在讀取狀態前後對資料庫雜湊，以偵測寫入。
歧義案例明確把一筆隱藏的已取消資料改成執行中；這是反事實情境，不是擷取到的競爭中執行歷史。
原有的審查關卡、rebase 後 head、無關中繼資料與格式錯誤輸入斷言都保持不變。

| 所需情境 | 使用的真實依據 | 尚未在真實環境證明的部分 |
| --- | --- | --- |
| 被取代的取消狀態讓位給停駐中的替代執行 | 真實的取消／後繼歷史，加上另行擷取的停駐關卡輸出 | 擷取到的後繼執行在 CI，關卡則位於另一分支的 test 階段；未擷取同次重跑的替代執行，恰好停在尚未 fetch 的 rebase 後 head 上之 review 階段的情況 |
| 超出上限、且伴隨無關中繼資料的競爭中執行身分 | 真實的有上限總覽，以及完整九筆分支歷史 | 真實分支沒有兩筆執行中資料；把隱藏資料改成執行中，以及注入無關的異常中繼資料，都是受控測試資料 |
| 較新的失敗優先於較舊的執行中工作 | 真實失敗狀態與真實執行中狀態 | 同一分支上同時存在兩種狀態的相對順序是組合出來的，並非實際觀察 |
| 授權依據改變或無法驗證 | 真實的執行中與已取消狀態格式 | 兩次讀取間的轉換、格式錯誤紀錄、錯誤身分與無法讀取的清單都是注入的；未擷取真實競態或損毀的正式環境清單 |
| 未初始化的儲存庫仍保留工作 Agent 回報 | 實際未初始化 stdout；先前的真實生命週期事件／窗格證據 | 可攜式測試重播 stdout，並使用既有的窗格模擬 |
| 驗證完成後開發繼續進行 | 真實的已完成狀態 | 完成後推進 Git 與發送工作 Agent 事件，是在可拋棄儲存庫中的操作，不是實際觀察到並記錄的工作 Agent 序列 |
| 可選清單相依套件不存在 | 真實關卡輸出與有上限的清單 | 缺少 Python／SQLite，以及完整清單組合都屬模擬；擷取主機具備兩項相依套件 |

重播斷言通過，只能確立這些明確輸入下的行為，不能證明最後一欄尚缺的真實情境。
