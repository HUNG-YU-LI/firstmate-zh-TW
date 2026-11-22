---
name: quiet
description: >-
  船長呼叫 /quiet、要求安靜模式、在場時保持安靜，或仍留在工作階段但希望減少例行喚醒回合時，進入安靜監督模式。
  不同於 /afk，一般船長對話不會退出；只有明確的 `/quiet off` 才會退出。
user-invocable: true
metadata:
  internal: true
---

# quiet

安靜監督模式（kunchenguid/firstmate#2356）：採用與 `/afk` 相同、節省 Token 的常駐程序取捨，但明確服務仍在場、看著工作階段，且不希望只因開始交談就退出模式的船長。

有常駐程序執行的情況下，本技能只是薄包裝層。
注入、忙碌／輸入區防護與可靠性特性，由 `afk` 技能負責；安靜模式在船長仍在場時沿用這套機制。
Quiet 模式下保留給船長事項的重新檢查，請見[架構](../../../docs/architecture.md)。

<a id="what-it-does"></a>
## 功能

0. **先確認此處是否還需要任何動作才能保持安靜。**
   Pi 或 pi-signed 不必進入新模式：有人在場時的分支，已讓例行喚醒不出現在這段對話（`afk` 技能第 2 步）；直接告知船長。
   其他環境執行 `bin/fm-afk-launch.sh quiet-check`；各結果意義，以標頭的 QUIET MODE 為準。
   - 結束碼 0：不進入新模式，不寫紀錄、不設旗標、不啟動常駐程序，之後 `/quiet off` 也不需處理。
     依 `AGENTS.md` 第 9 節語言告訴船長，此處的監督本來就如此運作：例行船隊事件不進這段對話，但決策、失敗、憑證與可供審查工作仍會送達。
     輸出行若表示監督工作階段暫停，則改說明恢復前例行更新仍會送達，以及下次重試時間。
   - 結束碼 2：離席紀錄仍有效，因此代表船長已返回；執行 `afk` 技能的返回流程並通過補齊進度關卡，再執行 `quiet-check`，依新結果處理。
   - 結束碼 1：繼續第 1 步；若有印出說明行，先明白告訴船長，是什麼讓此處監督尚未保持安靜。

1. **和 `/afk` 完全一樣，透過 `bin/fm-afk-launch.sh` 進入生命週期，但先設定 `FM_AFK_MODE=quiet`。**
   遵循 `afk` 技能的紀錄進入、常駐程序啟動與通知步驟；例外是執行監督主機的 home，在 `quiet-check` 結束碼為 1 後，不適用 `/afk` 的不啟動常駐程序規則。
   絕不另行啟用 `fm-watch.sh`。
   在呼叫 `bin/fm-afk-launch.sh enter` 與 `start`（或 `start-native`）的 Shell 中匯出 `FM_AFK_MODE=quiet`，讓紀錄標示 quiet 模式，且 `state/.afk` 第一行為 `quiet`，不是 `away`。
   執行監督主機的 home，依此執行環境沒有主機時使用的路徑啟動常駐程序；`start` 與 `start-native` 從 `enter` 寫入的紀錄讀取 quiet 模式。
   更新 quiet 模式時，保持 `FM_AFK_MODE=quiet`：即使沒有新文字，`/afk` 進入操作仍會以離席紀錄取代 quiet 紀錄，並開始保留事項等待返回。

2. **確認已啟用**，使用 `AGENTS.md` 第 9 節語言：「船長，安靜模式已啟用；我會集中例行更新，只提出決策、失敗、憑證或可供審查的工作。一般交談不會退出此模式；想恢復每次喚醒的正常回覆時，請說 `/quiet off`。」

<a id="how-to-exit-quiet-mode"></a>
## 如何退出安靜模式

不同於 `/afk`，一般對話絕不是退出訊號，這正是此模式的目的（AGENTS.md 第 8 節離席模式入口的 quiet 分支）。

- 只有明確 `/quiet off`，或船長明白要求離開安靜模式／恢復一般監督，才會退出：原樣執行 `bin/fm-afk-return.sh`，精確遵循 `/afk`「How to exit afk」區段的返回程序，包括正確順序的常駐程序關閉、持久喚醒呈現與確認、呈報／卡住證據，以及返回補齊進度關卡。
  該指令碼不讀取、也不在意旗標模式，因此不需要 quiet 專屬版本。
- 已標記的常駐程序呈報，或 quiet 模式中以 `/quiet` 開頭的訊息（更新，不是退出），都留在 quiet 模式並處理；這與 `/afk` 為離席模式定義的兩個例外相同。
- Quiet 模式中的其他所有訊息，都直接當成一般工作回答，旗標與常駐程序保持不變。

<a id="orthogonal-to-approval-authority"></a>
## 與核准權限彼此獨立

安靜模式只改變 firstmate 提出事項的積極程度，不改變誰核准什麼。
可合併 PR 維持 `AGENTS.md` 第 7 節的合併授權，needs-decision 發現維持 `ask-user-authority` 政策。

船長仍在場，因此 quiet 模式不保留任何事項等待返回。
進入 quiet 所寫紀錄帶有 quiet 模式（`bin/fm-afk-contract.sh mode`），進入、回讀與工作階段啟動行也會表明。
船長要求或持續有效授權涵蓋的每項操作，包括合併 local-only 工作、合併或派工，都和沒有 quiet 模式時一樣立即進行；`afk` 技能的離席保留規則絕不適用 quiet 紀錄。

<a id="must-not-hide-a-decision-or-a-failure"></a>
## 不得隱藏決策或失敗

依該 Issue 作者自己的分類：安靜模式只改變呈現。
進度、重試與內部機制和離席模式一樣留在甲板下，但可供審查工作、發現、決策、失敗與憑證，每次都透過 `/afk` 負責的相同分類政策向上呈報。
安靜模式需選擇啟用，絕不在未同意下成為預設；只有明確 `/quiet` 呼叫才進入。
