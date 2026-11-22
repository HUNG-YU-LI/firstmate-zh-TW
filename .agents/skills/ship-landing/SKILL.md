---
name: ship-landing
description: Ship 回報 PR 或就緒分支、決定或監看合併，以及清理工作前載入。
user-invocable: false
metadata:
  internal: true
---

<a id="ship-landing"></a>
# Ship 合併交付

以 PR 交付的 ship 工作，就緒訊號依模式而定：`no-mistakes` 在 CI 通過後回報 `done [at=<epoch>]: PR <url> checks green`，`direct-PR` 則在開啟 PR 後回報 `done [at=<epoch>]: PR <url>`；兩者都只適用非草稿 PR；刻意保持草稿的通道應宣告等待，`bin/fm-pr-check.sh` 也會拒絕對草稿啟用合併監控。
執行 `bin/fm-pr-check.sh <id> <PR url>`，URL 必須複製自該就緒訊號，或 `fm-crew-state.sh` 已解析的 checks-green 行；它會在工作中繼資料記錄 `pr=`，並在可取得時記錄託管平台的 `pr_head=`，接著啟用監看程式的合併輪詢；若原本就緒的 PR／MR 後來退化，例如檢查失敗、head pipeline 失敗或退出 merge train，也會喚醒監督者，讓它指導工作 Agent 修正。
存在 `config/pr-register-scan` 時，監看程式可在檢查掃描中記錄就緒工作的 PR，包括替換具名 PR；完整規則請見 [`docs/configuration.md`](../../../docs/configuration.md#pr-registration-scan-configpr-register-scan)。
處理 `check: pr-register:` 通知時，視同手動執行得到相同結果。
該就緒訊號的具名 head 關卡，以 `bin/fm-dod-lib.sh` 為準：ship `done:` 指定的 head 若只存在於工作 Agent 可清除的副本中，就不算就緒（`bin/fm-crew-state.sh` 回報 blocked，`bin/fm-pr-check.sh` 拒絕登錄，二副也不向上游發布該 done）。
這些讀取器中，只有 `bin/fm-pr-check.sh` 會在決策前 fetch 副本的 remote；而 no-mistakes 管線從自己的 clone 推送，因此 no-mistakes 就緒訊號在登錄執行前可能顯示 blocked：先登錄，再發送指導。
拒絕訊息若表示無法更新 remote，代表它依最後已知 refs 決策；remote 恢復可連線後再重試。
其他 blocked 結果則代表關卡正常發揮作用，不是工作 Agent 卡住；應針對拒絕訊息點名的提交指導工作 Agent，不要空等。
`forge=external` 工作改回報 `done [at=<epoch>]: review <url> uploaded for review`：執行 `bin/fm-review-check.sh <id> <review url>`，不要執行 `bin/fm-pr-check.sh`，且不啟用合併監看；接著不論 yolo 設定為何，都必須把審查 URL 呈報船長，因為 firstmate 無法觀察或執行其核准或合併。
Direct-PR 工作 Agent 把該提交推送到 PR 分支，local-only 工作 Agent 則在 ship 分支提交。
No-mistakes 工作 Agent 透過 /no-mistakes 重新驗證，維持管線是唯一發布者；它絕不從自己的副本推送。
No-mistakes 模式中，較早的 `done [at=<epoch>]: {summary}` 是管線交接，不受此關卡限制。
向船長提供從工作 Agent 就緒行、已解析的 checks-green crew-state 行，或工作 `pr=` 中繼資料複製的完整 `https://...` PR URL、簡潔成果摘要，以及適用時的 no-mistakes 風險等級。
船長下令合併就是明確授權；`yolo` 是唯一持續有效的例行合併授權。
自行撰寫任何自訂 `state/<id>.check.sh` 時，保持為一般、單一硬連結、權限 `0700` 的檔案；只有 firstmate 應被喚醒時才印一行，否則不輸出，並在 `FM_CHECK_TIMEOUT` 前完成；監看程式可執行前，先用 `bin/fm-check-register.sh <id>` 綁定其目前位元組內容。
停用自訂檢查時，只能使用 `bin/fm-check-unregister.sh <id>`（已啟動工作則可用 `bin/fm-teardown.sh`）；絕不自行以 `$STATE`／`$ID` 組合 `rm`。

對已啟用合併輪詢、且已合併的通道，包括透過 `bin/fm-pr-merge.sh` 合併的 PR，若通道已停駐或 home 設有 `config/auto-land`，由監看程式負責清理（[`docs/configuration.md`](../../../docs/configuration.md#park-on-review)）。
否則，由 firstmate 在確認合併後清理。
停駐於審查的通道（`bin/fm-park.sh`）沒有 Agent，出現審查活動時會自行重新啟動；`bin/fm-park.sh park <id>` 可依要求停駐一個乾淨通道，`bin/fm-park.sh resume <id> --trigger <text>` 則可手動重新啟動一個。

其他 ship 工作，也只有確認合併後才能清理。
因未提交或尚未合併工作而拒絕清理，是要求停止並調查的結果，絕不是可以繞過的障礙。
未獲明確捨棄授權，絕不強制清理。
清理成功後，記錄完成，只保留設定數量的近期 Done 歷史，並重新評估阻礙與時間關卡已清除的排隊工作。

二副是持續存在的，空佇列是健康狀態。
只有船長或主 firstmate 明確決定後，才能在載入 `secondmate-provisioning` 後退役二副；其 home 不得有進行中工作，強制捨棄仍需要船長明確授權。
