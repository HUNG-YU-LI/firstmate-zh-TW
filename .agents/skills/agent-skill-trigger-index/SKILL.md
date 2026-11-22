---
name: agent-skill-trigger-index
description: 只有稽核或維護完整 Agent 專用技能觸發索引時才載入。
user-invocable: false
metadata:
  internal: true
---

<a id="agent-only-reference-skills"></a>
# 僅供 Agent 使用的參考技能

這些技能不能由船長呼叫；只在精確觸發條件成立時載入。

- `operational-home-layout` — 尋找、解讀或修改 Firstmate home、config、data、state、project 或產生的執行階段路徑時載入。
- `session-start-recovery` — 工作階段啟動摘要回報未完成檢查、需處理診斷、復原輸入，或需要解讀的輸出時載入。
- `bootstrap-diagnostics` — 工作階段啟動摘要的 bootstrap 或 network-checks 區段印出需處理的診斷行時載入，包括 `MISSING:`、`MISSING_MANUAL:`、`OUTDATED:`、`PRESENTATION_UNAVAILABLE:`、`BACKEND_INVALID:`、`NEEDS_GH_AUTH`、`TANGLE:`、`STARTUP_MEMORY_BUDGET:`、`CREW_DISPATCH: invalid`、`FLEET_SYNC:`、`NETWORK_CHECKS:`、`HOME_SUMMARY:`、`BACKLOG_RECONCILE:`、`SECONDMATE_SYNC:`、`SECONDMATE_LIVENESS:`、`SECONDMATE_HANDOFF:`、`NUDGE_SECONDMATES:` 或 `FMX:`；`BOOTSTRAP_INFO:` 若表示中斷的待辦清理可能遺留端點或本機副本，也需載入；沒有輸出或其他 `BOOTSTRAP_INFO:` 事實則不需載入。
- `diagnostic-reasoning` — 界定回報問題範圍前，以及依診斷報告行動前載入。
- `ask-user-authority` — 決定任何 ask-user 發現前載入。
- `validation-supervision` — Ship 開始或已有進行中的 no-mistakes 驗證，包括執行中需求變更或發現，以及決定或回答任何 ask-user 發現前載入。
- `ship-landing` — Ship 回報 PR 或就緒分支、決定或監看合併，以及清理工作前載入。
- `scout-completion` — Scout 回報完成、提出視覺產物供反覆修改，或考慮提升為實作時載入。
- `quota-array-dispatch` — 根據目前 quota-axi 預設 TOON，在符合的 crew-dispatch 設定檔陣列中選擇前載入。
- `harness-adapters` — 啟動或復原船員／二副、處理信任對話框、傳送執行環境專屬技能呼叫、中斷或結束 Agent、恢復已結束 Agent，或驗證新執行環境介接器前載入。
- `firstmate-orca` — 切換到 Orca、啟動或監督 Orca 後端工作、對 Orca 後端行為進行 smoke test、除錯 Orca 工作狀態，或對帳 Orca 工作中繼資料前載入。
- `project-management` — 新增、建立、移除或初始化專案前載入。
  複製或登錄專案屬新增受理，也使用相同觸發條件。
- `stuck-crewmate-recovery` — 工作階段啟動摘要回報一般直接部屬端點已終止或中繼資料沒有視窗，出現 stale 喚醒、窗格迴圈、反覆混亂、提出簡報已回答的問題、船員無回應或指導傳遞失敗後，以及仍存活的工作 Agent 回報其 no-mistakes 管線已終止、無法存取或逾時時載入。
- `secondmate-provisioning` — 建立、植入初始內容、驗證、啟動二副 home，向它交接待辦清單、復原它、推送繼承的本機素材、退役它，以及編輯 `data/secondmates.md` 前載入。
- `captain-hold-lifecycle` — 認定調查或視覺審查完成前、結束已揭露船長決策的視覺審查前、記錄或路由船長回答時，以及喚醒清空結果出現任何 `RECORD DIVERGENCE` 行時載入。
- `away-quiet-supervision` — 每當呼叫 /afk 或 /quiet、存在離席或安靜模式紀錄，或收到已標記 away-supervisor 訊息時載入。
- `process-event-sources` — 啟用長輪詢來源前、登錄具確定性的條件→動作監看（Y 成立就做 X）前、收到任何 `procevent <adapter> <source-id> <sequence>` 檢查喚醒，以及任何 `process-event source stranded` 或 `process-event source failed to start` 檢查喚醒時載入。
  絕不在對話回合中自行執行已登錄來源的阻擋式指令。
- `fmx-respond` — 收到 `x-mention <request_id>` `check:` 喚醒以處理提及、收到 `x-mode-error ...` `check:` 喚醒以回報 Relay 設定阻礙、收到 `public-followup ...` `check:` 喚醒或啟動時呈現的公開承諾，以及 Relay 關聯工作出現里程碑或終止喚醒、即將發布完成追蹤前載入；只有 Relay 開啟時相關。
- `firstmate-codexapp` — 協調可見 Codex Desktop 對話串、評估 Codex App 後端要求，或對帳 Firstmate 工作的 Codex Desktop 主機工具 smoke 證據前載入。
- `firstmate-coding-guidelines` — 修改第 1 節清單定義的 firstmate 共用版本控制素材前載入，不論是直接編輯，或為 firstmate 儲存庫工作撰寫船員簡報。
