<a id="secondmate-parent-channel-live-verification"></a>
# 二副父層通道：真實環境驗證

本維護者驗證紀錄，佐證 [`secondmate-parent-channel.md`](../secondmate-parent-channel.md) 的保證：二副（secondmate）主目錄內記錄、應呈報船長（captain）的結果，會到達父層通道，無須二副模型親自寫入。
變更 `bin/fm-parent-channel-lib.sh` 列出的任一發布者後，請重新執行下列測試夾具以更新紀錄。
這次執行早於送出時間戳記機制，因此下列每行已發布內容都是沒有時間戳記的 payload；如今重跑會寫出相同位元組，但在標頭末端加上 `[at=<epoch>]` 標籤，例如 `done [key=child-outcome-child-done-05b032a1] [at=<epoch>]: child ...`。
[`bin/fm-classify-lib.sh`](../../bin/fm-classify-lib.sh) 負責定義該標籤語法；本紀錄對投遞所證明的內容都不依賴它。

<a id="what-was-run"></a>
## 執行內容

日期：2026-09-03。
程式碼樹：此變更的分支，於 macOS 上使用 tmux 3.6a、作為 `/bin/bash` 的 bash 3.2，以及儲存庫自身腳本。
測試夾具：暫存目錄下的兩個隔離主目錄，`$P`（父層）與 `$M`（二副），兩者的 `config/backend` 均設為 `tmux`。
二副主目錄帶有 `.fm-secondmate-home`（`mate`），以及指向 `$P` 的本機 `.fm-secondmate-parent` 綁定；父層主目錄則有 `state/mate.meta`（`kind=secondmate`、`home=$M`）。
二副的子任務 `child` 是真實 tmux 窗格（`fmlive:fm-child`），記錄在 `$M/state/child.meta`，設定 `mode=no-mistakes` 與 `yolo=off`；二副本身則位於第二個真實窗格（`fmlive:fm-mate`）。
兩個主目錄都執行真正的 `bin/fm-watch.sh`（`FM_POLL=2`），並在每次喚醒後透過 `bin/fm-wake-drain.sh` 確認重新啟動，完全比照船隊監督在處理完回合後重新啟動監看器的方式。
測試夾具內完全沒有執行 Agent 執行環境或模型。

子任務唯一的動作，是透過 `tmux send-keys` 在自己的窗格輸入一般船員狀態附加指令。
二副唯一的動作，是執行 Firstmate 註冊 PR、將任務暫停等待船長決策，以及記錄答覆時使用的腳本。

<a id="transcript"></a>
## 逐字紀錄

```text
$ # Setup: mate home $M is a secondmate of parent home $P (local route); child task 'child' lives in a real tmux pane; both watchers are live

$ # Step 1: the child appends its terminal line from inside its own real tmux pane

$ tmux send-keys -t fmlive:fm-child "echo 'done: PR https://github.com/kunchenguid/firstmate/pull/9999 checks green' >> $M/state/child.status" Enter

$ cat $M/state/child.status
working: implementing the feature
done: PR https://github.com/kunchenguid/firstmate/pull/9999 checks green

$ cat $P/state/mate.status   (the parent channel)
working: delegated scope
done [key=child-outcome-child-done-05b032a1]: child child done: PR https://github.com/kunchenguid/firstmate/pull/9999 checks green pr=https://github.com/kunchenguid/firstmate/pull/9999 mode=no-mistakes yolo=off

$ mate watcher wakes so far
signal: $M/state/child.status

$ parent watcher wakes so far
signal: $P/state/mate.status

$ # Step 2: the mate registers the PR with fm-pr-check; the ready line with the canonical URL reaches the parent from the script itself

$ FM_HOME=$M FM_STATE_OVERRIDE=$M/state bin/fm-pr-check.sh child https://github.com/kunchenguid/firstmate/pull/9999
armed: state/child.check.sh

$ cat $P/state/mate.status
working: delegated scope
done [key=child-outcome-child-done-05b032a1]: child child done: PR https://github.com/kunchenguid/firstmate/pull/9999 checks green pr=https://github.com/kunchenguid/firstmate/pull/9999 mode=no-mistakes yolo=off
done [key=child-pr-child]: child child PR ready: https://github.com/kunchenguid/firstmate/pull/9999 mode=no-mistakes yolo=off

$ parent watcher wakes so far (2 signals)
signal: $P/state/mate.status
signal: $P/state/mate.status

$ # Step 3: the mate holds a task for the captain; the hold reaches the parent from fm-captain-hold itself

$ FM_HOME=$M bin/fm-captain-hold.sh hold child-call --title 'Pick the rollout window' --reason 'rollout window choice pending' --repo alpha
child-call

$ # Step 4: the captain's answer is recorded in the mate home; the close reaches the parent from fm-captain-hold itself

$ FM_HOME=$M bin/fm-captain-hold.sh answer child-call --decision-file decision.txt   (decision: roll out on Monday)
answered: child-call

$ cat $P/state/mate.status   (final parent channel)
working: delegated scope
done [key=child-outcome-child-done-05b032a1]: child child done: PR https://github.com/kunchenguid/firstmate/pull/9999 checks green pr=https://github.com/kunchenguid/firstmate/pull/9999 mode=no-mistakes yolo=off
done [key=child-pr-child]: child child PR ready: https://github.com/kunchenguid/firstmate/pull/9999 mode=no-mistakes yolo=off
needs-decision [key=captain-hold-child-call-1]: captain hold child-call: rollout window choice pending
resolved [key=captain-hold-child-call-1]: captain hold child-call: answered

$ parent watcher wakes, one per delivered line (4 signals)
signal: $P/state/mate.status
signal: $P/state/mate.status
signal: $P/state/mate.status
signal: $P/state/mate.status

$ mate watcher wakes
signal: $M/state/child.status
stale: fmlive:fm-child

$ # Every line above beginning 'done [key=child-' or carrying 'captain-hold-' was written by a script, never by a model; each one woke the parent watcher.
```

<a id="why-this-proves-the-hole-is-closed"></a>
## 為什麼這能證明缺口已補上

- 原本觀察到的缺陷是：二副模型處理完子任務結果後，直接在自己的聊天中向船長回報，而沒有附加到父層通道。
  本次執行完全沒有模型，但 `$P/state/mate.status` 仍出現四行應呈報船長的內容：子任務的最終 done 列，包含附註、標準 PR URL、模式及運作狀態；註冊時的 PR 就緒列；等待船長決策的 hold；以及該 hold 的答覆。
- 每一行都由記錄底層事實的腳本寫入：二副監看器輪詢時的 `bin/fm-inactive-reconcile.sh`、`bin/fm-pr-check.sh`，以及 `bin/fm-captain-hold.sh`。
- 每一行都在真正的父層監看器產生一次 `signal:` 喚醒；該事件會啟動父層 Firstmate 的回合，進而產生面向船長的回報。
- 二副監看器自己針對 `child.status` 產生的 `signal:`，顯示二副仍像原先一樣被喚醒；二副模型之後會在自己的聊天說什麼，都與投遞無關。
- 二副監看器日誌末端的 `stale:` 列，是子任務寫入最終列後，真實窗格進入閒置而觸發的一般存活性升級事件，與投遞無關。
