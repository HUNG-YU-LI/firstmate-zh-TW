---
name: scout-completion
description: 當 scout 回報完成、提出視覺產物供反覆修改，或考慮將它提升為實作工作時載入。
user-invocable: false
metadata:
  internal: true
---

<a id="scout-outcome-and-promotion"></a>
# Scout 成果與提升為實作

已完成的 scout 必須先留下可獨立閱讀的報告，才能捨棄暫存 worktree；請閱讀並轉達發現，將報告記為 Done 產物，並重新評估佇列。
報告可以建議實作，但不代表授權實作。
認定調查或任何視覺審查完成之前，載入 `captain-hold-lifecycle`；清理流程會強制執行這道共用完成關卡。
如果 scout 的交付成果是供船長反覆修改的視覺產物，請讓它持續可用，並遵循 `docs/configuration.md` 中由船員代管的 Lavish 看板契約，不要從 firstmate 啟用或輪詢看板。
實作另行獲得授權後，透過 `bin/fm-promote.sh` 提升現有 scout，不要建立重複工作。
提升後的工作 Agent 必須盤點暫存狀態，回到該工作基準的乾淨副本（已記錄的基底分支，否則使用預設分支），只帶入預期的修正變更，建立 ship 分支，遵循專案選定的交付路徑，留下暫存提交與除錯編輯，並將已重現的問題轉成回歸測試。
