<a id="gitlab-merge-request-watch-and-merge-verification"></a>
# GitLab 合併請求監看與合併驗證

這份實測紀錄涵蓋 GitLab 的合併監看與合併路徑，與既有 GitHub 紀錄並列。
從啟動監看、輪詢、缺少 `glab`，到 GitHub 不受影響的案例，證據皆收集於 2026-07-21；「合併合併請求」則於 2026-08-22 執行。
所有輸出均精確重現。

<a id="versions"></a>
## 版本

```
$ glab --version
Current glab version: 1.53.0

$ bash --version | head -1
GNU bash, version 5.3.9(1)-release (x86_64-pc-linux-gnu)
```

日期為 2026-08-22 的合併證據，在另一台主機收集，版本如下：

```
$ glab --version
glab 1.82.0-<local build tag> (<local build commit>)

$ jq --version
jq-1.8.1

$ bash --version | head -1
GNU bash, version 5.2.15(1)-release (x86_64-amazon-linux-gnu)
```

該 `glab` 是本機建置的 1.82.0；只省略建置標籤與提交，因為它們指向私人建置，而非正式發布版本。

<a id="the-evidence-project"></a>
## 證據專案

此處所有真實證據都讀取 <https://gitlab.com/KarotKris/gitlab-merge-watch-fixture>，這是一個專門作為證據用途的公開專案。
它保有一個刻意合併的合併請求，以及一個刻意保持開啟的合併請求，讓兩種結果都能以真實資料展示。
對它執行的每個指令都只讀取公開合併請求，不需要憑證，因此讀者可重跑每個指令並看到相同輸出。
其 README 請求保持開啟的合併請求不要被合併或關閉。

下方非預設主機只以不會解析到任何位置的預留值 `gitlab.example` 出現。
這是刻意的：不依賴主機的特性，來自已儲存紀錄與輪詢的 URL 重建，因此透過檢查這兩者證明，而不連線到任何私人執行個體。

<a id="why-the-host-is-data-rather-than-a-constant"></a>
## 為何主機是資料而非常數

GitLab 多半執行於自行架設的執行個體，因此合併請求可能位於任何主機。
GitLab 專案也至少位於一個群組之下，深度不固定，無法像 GitHub 一樣只用擁有者與儲存庫配對定址。
因此，儲存紀錄攜帶 `provider`、`url`、`host`、`path` 與 `number`；每個消費端都會用這些部分重建 URL，若無法精確重建已儲存 URL，就拒絕該紀錄。
`tests/fm-pr-check-security.test.sh` 透過非預設主機的 sidecar 證明不依賴主機的路徑，並驗證 `glab` 收到重建的專案 URL。

<a id="how-plain-glab-is-invoked-and-why"></a>
## 一般 glab 的呼叫方式與原因

以下兩個一般 `glab` 的事實都透過實際執行確認，因為任意假設其中一項，都會悄悄失敗並永遠停在「尚未合併」。

首先，一般 `glab` 沒有欄位選取器。
`gh` 可用 `--json state -q .state` 讀取單一欄位；`glab mr view` 只提供 `-F, --output string  Format output as: text, json`。
因此合併檢查讀取 glab 本身的欄位輸出，不需要 JSON 處理器；輪詢的降級狀態讀取（head pipeline、merge train 車廂）只能使用 JSON，需要 `jq`；缺少 `jq` 時，輪詢會退回此欄位讀取，讓合併喚醒仍可運作。
只有精確的 `merged` 會喚醒 firstmate，因此輸出格式變更只會造成不喚醒，不會誤報合併。

其次，`glab` 無法像 `gh pr view` 一樣接受合併請求 URL。
該格式會呼叫 git 查詢目前儲存庫，但監看器並不在儲存庫中執行：

```
$ cd /tmp && glab mr view https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
fatal: not a git repository (or any parent up to mount point /)
Stopping at filesystem boundary (GIT_DISCOVERY_ACROSS_FILESYSTEM not set).
git: exit status 128
```

將專案 URL 傳給 `-R`，並提供合併請求編號，就能從任何位置運作，而且會從該 URL 解析執行個體，而不是使用 glab 設定的預設值：

```
$ cd /tmp && glab mr view 1 -R https://gitlab.com/KarotKris/gitlab-merge-watch-fixture
title:	Add the merged example file
state:	merged
author:	KarotKris
labels:	
assignees:	
reviewers:	
comments:	0
number:	1
url:	https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
--
This merge request is the merged half of the fixture. It is merged on purpose, so that reading its state returns merged.

$ cd /tmp && glab mr view 2 -R https://gitlab.com/KarotKris/gitlab-merge-watch-fixture | sed -n 's/^state:[[:space:]]*//p'
open
```

<a id="end-to-end-arming-and-polling-a-real-merge-request"></a>
## 端到端：啟動監看並輪詢真實合併請求

三個任務啟動監看，其中兩個指向測試專案，一個指向預留主機：

```
$ fm-pr-check.sh e1 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
armed: state/e1.check.sh
$ fm-pr-check.sh e2 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
armed: state/e2.check.sh
$ fm-pr-check.sh e3 https://gitlab.example/group/subgroup/project/-/merge_requests/7
armed: state/e3.check.sh
```

各自儲存的紀錄，顯示主機與完整專案命名空間皆作為資料保存：

```
$ cat state/e1.pr-poll
gitlab
https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
gitlab.com
KarotKris/gitlab-merge-watch-fixture
1

$ cat state/e3.pr-poll
gitlab
https://gitlab.example/group/subgroup/project/-/merge_requests/7
gitlab.example
group/subgroup/project
7
```

非預設主機的來源紀錄，顯示提升後的版本標籤：

```
$ cat state/e3.pr-poll-registration
fm-pr-poll-registration-v2
e3
gitlab
https://gitlab.example/group/subgroup/project/-/merge_requests/7
gitlab.example
group/subgroup/project
7
514b7e04f0cca3e2c913c9fd504c54dfe54c8a51a7f5ebc57279bbd4db5d4a60
1817b0f95db7148246434a4afa0b2c8e7b81fd8f74ef7d473bbd62023e47c439
70:957243
70:957244
```

以監看器相同的方式執行每個已發布輪詢；空結果代表輪詢保持靜默，沒有產生喚醒：

```
$ fm-pr-poll.sh --validated $(tr '\n' ' ' < state/e1.pr-poll)
merged
$ fm-pr-poll.sh --validated $(tr '\n' ' ' < state/e2.pr-poll)
$ fm-pr-poll.sh --validated $(tr '\n' ' ' < state/e3.pr-poll)
```

已合併的測試合併請求恰好產生一行 `merged`。
開啟中的合併請求不產生輸出，無法連線的預留主機也不產生輸出，不會誤報合併。

相同位元組也能在監看器由 sidecar 驅動的模式運作，此時已發布的檢查會定位自己的紀錄：

```
$ state/e1x.check.sh
merged
```

<a id="a-missing-cli-produces-no-wake-never-a-false-merge"></a>
## 缺少 CLI 不會喚醒，也絕不誤報合併

輪詢刻意在任何錯誤時保持靜默，因此若沒有其他處理，缺少 `glab` 就會與永不合併的合併請求無法區分。
將 `glab` 從 `PATH` 移除後，即使合併請求真的已合併，輪詢仍保持靜默：

```
$ PATH="$noglab" fm-pr-poll.sh --validated $(tr '\n' ' ' < state/e1.pr-poll)
$ PATH="$noglab" fm-pr-poll.sh --validated $(tr '\n' ' ' < state/e3.pr-poll)
```

啟動監看是唯一能回報此事的時機，因此會在此拒絕，不會啟動永遠無法觸發的監看：

```
$ PATH="$noglab" fm-pr-check.sh e5 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
error: watching a GitLab merge request requires glab on PATH
$ echo $?
1
```

GitHub 任務不受缺少 `glab` 影響：

```
$ PATH="$noglab" fm-pr-check.sh e6 https://github.com/kunchenguid/firstmate/pull/750
armed: state/e6.check.sh
```

<a id="registration-version"></a>
## 註冊版本

目前註冊標籤為 `fm-pr-poll-registration-v2`，包含提供者標籤。
`fm-pr-poll-registration-v1` 紀錄已無法解析。
請以 `bin/fm-pr-check.sh` 啟動目前版本的監看。

<a id="merging-a-merge-request"></a>
## 合併合併請求

`bin/fm-pr-merge.sh` 現在透過共用記錄輔助程式，以及 GitLab 本身的即時合併前防護，合併 GitLab 合併請求。
以下每次執行都使用可拋棄的 `FM_HOME`，因此不會觸碰任何實際任務紀錄；也使用完全拒絕任何 `merge` 子指令的 `glab` 包裝程式，因此即使檢查有誤，合併也無法送達程式碼託管平台。
正因為有該包裝程式，才能將保持開啟的測試合併請求用作證據：它是 `mergeable` 且討論皆已解決，所以只有 pipeline 條件阻擋真正合併。

合併需要 `glab` 讀取與 `jq` 解析；缺少任一工具，都會在記錄任何內容前拒絕：

```
$ PATH="$noglab" fm-pr-merge.sh e5 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
error: merging a GitLab merge request requires glab on PATH
$ echo $?
1
$ PATH="$nojq" fm-pr-merge.sh e6 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
error: merging a GitLab merge request requires jq on PATH
$ echo $?
1
```

兩種拒絕都未啟動輪詢，也未記錄 `pr=`，因此缺少工具不會留下準備一半的合併。

存在 `config/pr-degraded-watch` 時，`bin/fm-pr-check.sh` 會拒絕在缺少 `jq` 的情況下啟動 GitLab 監看，因為其降級狀態基準只能以 JSON 取得；沒有該旗標時，則可在缺少 `jq` 的情況下啟動既有的僅合併監看。
車廂端點回傳 404 時，會透過一次有界讀取專案的有效 merge train 清單確認；只有可讀取且未包含該合併請求的清單，才會觸發一次有界的 MR 狀態重讀，清單錯誤則保持靜默。
重讀得到 `opened` 或 `locked` 會回報退出佇列；`closed` 保持靜默，`merged` 回報合併，無法讀取的重讀則以 `mr_state=unconfirmed` 回報已確認的退出佇列。
監看器只有在曾記錄該合併請求進入 train 後，才會發出退出佇列事件。
合併路徑無法使用欄位輸出，因為 `detailed_merge_status`、`has_conflicts`、`blocking_discussions_resolved` 與 head pipeline 只出現在 glab 的 JSON 中。
輪詢在缺少工具時保持靜默是安全的，因為靜默代表「尚未合併」；但合併操作不能對此保持靜默，因此會回報需求，而不是自行假設。

測試專案中已合併的那一個會被拒絕，且會列出全部失敗條件，不只第一個：

```
$ fm-pr-merge.sh e1 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
armed: state/e1.check.sh
error: refusing to merge https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/1
  - state is "merged", not open
  - detailed_merge_status is "not_open", not mergeable
  - the head pipeline status is "none", not success
  - the head pipeline ran at "none", not at the current head 33762fcf6777c8d993220d25fb541e56c48081b9
$ echo $?
1
```

開啟中的那一個為 `mergeable`、無衝突，且討論都已解決，因此只有 pipeline 條件造成拒絕。
測試專案不執行 CI，所以 `head_pipeline` 是 `null`；這會回報為 `none`，不會當成沒有東西需要檢查：

```
$ fm-pr-merge.sh e2 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
armed: state/e2.check.sh
error: refusing to merge https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
  - the head pipeline status is "none", not success
  - the head pipeline ran at "none", not at the current head 66b8a6777bea5e291d7fa2fc20c42ad7686f6bc8
$ echo $?
1
```

因此，預設完全不執行 pipeline 的專案，無法透過此路徑合併。
這是需求的預期解讀，不是疏漏：head 上成功的 pipeline 是必要條件，而「沒有 pipeline」不符合條件。
唯一例外是在船長明確下令後，有人監督地使用 `--allow-no-pipeline`；它只豁免此處 null 的 `head_pipeline`，不豁免其他條件；其防護規則由 `bin/fm-pr-merge.sh` 標頭定義。

兩種拒絕都發生在已記錄 `pr=` 並啟動合併輪詢之後，與 GitHub 端即時驗證或 `gh pr merge` 失敗時相同，因此拒絕後仍保留稽核軌跡與監看。

若已記錄的 `pr_head=` 不再符合即時 head，會回報此事，並以即時 head 進行驗證。
下方過期值是手動寫入任務紀錄，因為 GitLab 任務本身絕不會自行記錄此值：

```
$ fm-pr-merge.sh e4 https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
armed: state/e4.check.sh
notice: recorded head 1111111111111111111111111111111111111111 disagrees with the live head 66b8a6777bea5e291d7fa2fc20c42ad7686f6bc8; verifying the live head
error: refusing to merge https://gitlab.com/KarotKris/gitlab-merge-watch-fixture/-/merge_requests/2
  - the head pipeline status is "none", not success
  - the head pipeline ran at "none", not at the current head 66b8a6777bea5e291d7fa2fc20c42ad7686f6bc8
```

其餘拒絕條件與合併本身，由 `tests/fm-pr-merge.test.sh` 使用測試資料涵蓋。
衝突、未解決討論與執行中 pipeline 條件，也另外在私人執行個體的真實合併請求上測試；那些執行無法在此重現，因此其識別資訊不納入本紀錄。
合併本身未對任何真實合併請求測試，任一方向皆然：`glab mr merge` 沒有 dry run，因此測試真實成功路徑，就等於為了產生證據而合併別人的工作。

<a id="why-the-head-is-read-live-and-bound-to-the-merge"></a>
## 為何即時讀取 head 並綁定合併

已驗證的 head 會傳給 `glab mr merge --sha`，因此若來源分支在讀取與合併之間移動，GitLab 會拒絕合併。
若沒有此條件，這個時間窗內到達的 push 就可能合併完全未驗證的提交。

傳入 `--yes` 的原因，與監看輪詢不需要終端機相同：無人值守的執行無法回答確認提示，而卡住的提示比拒絕更糟。
它只略過該提示；真正授權合併的是上述條件。

<a id="why-a-recorded-head-is-not-the-authority"></a>
## 為何已記錄 head 不是權威來源

`bin/fm-pr-check.sh` 只會為 GitHub 記錄 `pr_head=`，因為 `gh` 將 head 提交提供為可選取欄位。
它刻意設計為選用，其他消費端也已如此處理：`bin/fm-teardown.sh` 在拆除時從程式碼託管平台讀取 head，再退回不依賴提供者的內容檢查；沒有已記錄值時，`bin/fm-review-diff.sh` 會從遠端擷取 pull request head，但合併請求沒有對應 ref，因此 GitLab 任務會依該指令碼的警告，與本機分支比較 diff（此備援由 [architecture.md](architecture.md) 定義）。

合併路徑也不會記錄 head，且刻意不依賴已記錄值。
rebase 會移動 head，讓任何已記錄值過期，因此依中繼資料決定合併，可能驗證到已不存在的提交。
在合併時即時讀取 head、回報不一致的已記錄值，並將合併綁定實際驗證的內容，才能補上這個缺口。
