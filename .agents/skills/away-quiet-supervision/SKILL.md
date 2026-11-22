---
name: away-quiet-supervision
description: 每當呼叫 /afk 或 /quiet、存在離席或安靜模式紀錄，或收到已標記的 away-supervisor 訊息時載入。
user-invocable: false
metadata:
  internal: true
---

<a id="away-and-quiet-supervision-safety"></a>
# 離席與安靜模式的監督安全

`/afk` 與 `/quiet` 技能各自負責進入程序，並共用常駐程序機制；兩種模式對船長保留事項的重新檢查差異，以[架構](../../../docs/architecture.md) 為準。
以下安全規則同時適用兩者：

- 目前所有常駐程序注入，都使用 `bin/fm-operational-input.sh` 的 `away-supervisor` 類型，接在 `FM_OPERATIONAL_PREFIX`（U+2063 INVISIBLE SEPARATOR，後接 `FIRSTMATE_OP: `）之後；例外是 Claude Code 主要工作階段會移除 U+2063，因此改接收該負責元件具有紀錄依據的門鈴通知，只有 `bin/fm-operational-input.sh open <path>` 驗證紀錄後才算已標記；舊式裸標記的相容處理由 `/afk` 技能負責。
- `state/.afk-contract` 表示離席模式，必須在 `/afk` 同一回合、其他工作之前寫入，因為 `/afk` 本身就是開始指示：進入時不應先回讀形成關卡，也不等待再次開始；進入通知只說明哪些事項保留到返回時處理，離席工作階段再依自身判斷，透過受防護的指令碼、在持續有效的授權內執行，遇到疑慮則保留到返回時。
  若紀錄帶有 quiet 模式（`bin/fm-afk-contract.sh mode`），則屬於安靜模式：船長仍在場，不保留任何事項等待返回，要求的操作依一般有人在場的授權進行。
- `state/.afk` 存在時，監督由常駐程序負責；不要另行啟用監看程式。
  Pi 絕不啟動此常駐程序；它的一般監督工作階段會在紀錄之下繼續運作，主對話則停駐：分支盡可能處理所有安全且可執行的喚醒，只有被拒絕的喚醒（包括分支故障或掃描不安全），或監看程式失敗，才會喚醒主對話。
  執行監督主機的非 Pi home，在離席模式下亦同（Claude 預設如此；見 `docs/configuration.md` 的「Supervision host」），由監督主機扮演分支；交還的喚醒透過該執行環境自己的喚醒路徑抵達，絕不代表船長返回。
- 離席或安靜模式啟用時收到的已標記訊息，是內部向上呈報，不會退出該模式。
- 以 `/afk` 開頭的訊息重新更新離席模式；以 `/quiet` 開頭的訊息重新更新安靜模式。
- 其他未標記訊息，在離席模式中代表船長返回（載入 `/afk`、執行負責返回流程的元件；在持久化補齊進度關卡通過前，不要把該訊息當成一般工作處理）；在安靜模式中，則直接當成一般工作回答，保持旗標與常駐程序不變，直到明確收到 `/quiet off`。
- 離席與安靜模式都不會擴大合併、ask-user 發現、破壞性操作、不可逆操作或安全敏感選擇的核准權限。
- 輸入有歧義時，優先退出離席模式，因為在場的船長優先。
