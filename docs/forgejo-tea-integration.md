<a id="forgejo-pull-request-watch-and-merge-verification"></a>
# Forgejo Pull Request 監看與合併驗證

這是 Forgejo 合併監看與合併路徑的實證紀錄，與既有 GitHub、GitLab 機制並列（`docs/gitlab-merge-watch.md`）。
每一項發現都在真正、可拋棄的 Forgejo 執行個體上驗證，而非從 `tea --help` 推測，因為有些結果與單看說明文字時的理解相反。
以下所有證據均蒐集於 2026-09-13。
專案透過 `forge=forgejo` 登錄表綁定選用，由 `bin/fm-project-mode.sh` 負責；此處記錄的監看與合併路徑，依 Pull Request URL 本身選擇行為，綁定只決定產生的 ship 簡報指定 `tea` 而非 `gh-axi`。

<a id="versions"></a>
## 版本

```
$ tea --version
Version: development	golang: 1.26.4
```

這是本機建置的開發版執行檔；無法從執行檔確定精確原始碼 revision，因為沒有內嵌 VCS 註記。

```
$ jq --version
jq-1.7

$ bash --version | head -1
GNU bash, version 5.2.21(1)-release (x86_64-pc-linux-gnu)

$ docker --version
Docker version 29.8.0, build 88096ef
```

<a id="the-evidence-instance"></a>
## 證據用執行個體

與 GitLab 文件的公開測試夾具專案不同，自架 Forgejo 執行個體不是能直接提供網路位址給讀者的服務；這正是 Firstmate 需要支援它的原因。
此處證據來自專為本次驗證，在本機 Docker 執行、事後銷毀的 Forgejo 7 暫存執行個體（`codeberg.org/forgejo/forgejo:7`），內含一個儲存庫（`fixture/fixture-repo`），以及透過真實任務也會使用的同一個 `tea` CLI 建立的多個 Pull Request。
這些內容不依賴任何私人或正式環境執行個體；讀者若未自行建立等效的暫存執行個體，也無法直接重現。

<a id="why-tea-needed-real-verification-before-writing-any-code"></a>
## 為何寫程式碼前必須實測 tea

`tea --help` 在 `tea pulls` 上列出 `--fields`／`-f` 與 `--output`／`-o`，乍看像 `glab mr view <n> -R <url> -F json`。
實際並不是：

```
$ tea pulls 1 --login fm-fixture --repo fixture/fixture-repo -f state -o simple
  # #1 Add the merged example file (merged)
  @fixture created 2026-09-13 22:08	**main** <- **merged-branch**
  merges
  --------
  http://localhost:3980/fixture/fixture-repo/pulls/1
```

無論 `-f`／`-o` 為何，`tea pulls <index>` 永遠呈現相同的自由文字詳細檢視；說明文字「will show it in detail」就是全部行為，欄位與輸出選擇會被靜默忽略。
只有清單形式會遵守它們：

```
$ tea pulls list --login fm-fixture --repo fixture/fixture-repo --state all -f index,state -o csv
index,state
2,open
1,merged
```

清單讀取受 `--limit` 約束，因此被監看的 Pull Request 若比該時間範圍更舊，就永遠看不到它已合併。
因此合併監看（`bin/fm-pr-poll.sh`）、終端執行的 PR 詳細資訊（`bin/fm-crew-state.sh`），以及 teardown 的 PR 已合併證明（`bin/fm-teardown.sh`），都改以 `tea api ... /repos/{owner}/{repo}/pulls/{n}` 按編號讀取單一 Pull Request，也就是下方合併路徑讀取的同一份 REST 紀錄；並只接受 `merged` 欄位為布林值的紀錄（`bin/fm-pr-lib.sh` 中的 `fm_pr_forgejo_read_record`）。
**此處未驗證：**按編號讀取尚未在證據用執行個體重跑；它依據的是同一種 `tea api` Pull Request 紀錄，而其中的 `state`、`merged` 與 `head.sha` 欄位，已由下方合併路徑的讀回驗證過。

<a id="why-the-login-is-resolved-fresh-and-how"></a>
## 為何每次重新解析登入設定，以及解析方式

`tea pulls`／`tea api` 只能用 `owner/repo` slug 指定儲存庫。
主機來自具名 `tea login`（`tea login add --name <name> --url <host> --token <token>`），以 `--login <name>` 選擇；沒有像 `glab -R <url>` 那樣直接接受主機或完整 URL 的旗標：

```
$ tea pulls ls --repo https://example.com/owner/repo -o simple
Error: path segment [1] is empty
```

驗證機器註冊了三個登入設定：私人執行個體上的兩個真實設定，以及可拋棄測試夾具。前兩個位於私人網路，其識別資訊不列入本紀錄，與 `docs/gitlab-merge-watch.md` 不公開私人執行個體執行識別資訊的做法相同：

```
$ tea login list --output json
[
  {
    "name": "forgejo",
    "url": "<redacted, private instance>",
    "ssh_host": "<redacted, private instance>",
    "user": "",
    "default": "true"
  },
  {
    "name": "forgejo-admin",
    "url": "<redacted, private instance>",
    "ssh_host": "<redacted, private instance>",
    "user": "",
    "default": "false"
  },
  {
    "name": "fm-fixture",
    "url": "http://localhost:3980",
    "ssh_host": "localhost",
    "user": "",
    "default": "false"
  }
]
```

每次 Forgejo 讀取都會透過 `jq` 將此清單解析為 JSON，將已驗證 PR 主機與每個登入設定的單純 hostname 比對（移除 scheme、使用者資訊、連接埠與路徑）；若符合數為零或超過一個，就拒絕而不猜測。
絕不逐行切割輸出，因為若登入設定被印成一行精簡物件，就可能把 name 讀到應屬於 url 的位置。
`bin/fm-pr-lib.sh` 的 `fm_pr_forgejo_login` 為所有載入它的腳本負責此比對；`bin/fm-pr-poll.sh` 是靜態、獨立、沒有 source 相依性的監看器主體，因此保留自己的實作副本，並在每次輪詢重新推導比對結果，不信任任何持久保存的名稱。

<a id="url-shape"></a>
## URL 結構

Forgejo（及 Gitea）的 Pull Request 路徑是 `https://<host>/<owner>/<repo>/pulls/<number>`：使用複數「pulls」，沒有 `/-/` 路由分隔。測試夾具自己的 PR 建立輸出已確認：

```
$ tea pulls create --login fm-fixture --repo fixture/fixture-repo --head merged-branch --base main --title "..."
  ...
  http://localhost:3980/fixture/fixture-repo/pulls/1
```

這不會與 GitHub 的 `.../pull/<n>`（單數）、GitLab 的 `.../-/merge_requests/<n>`，或 Gerrit 的 `.../c/<project>/+/<n>` 衝突。

<a id="end-to-end-arming-and-polling-a-real-pull-request"></a>
## 端到端：啟動監看並輪詢真正的 Pull Request

```
$ fm-pr-check.sh task-f1 https://localhost/fixture/fixture-repo/pulls/4
armed: state/task-f1.check.sh

$ cat state/task-f1.pr-poll
forgejo
https://localhost/fixture/fixture-repo/pulls/4
localhost
fixture/fixture-repo
4

$ state/task-f1.check.sh
(nothing - PR 4 is still open)
```

接著透過下方描述、綁定 head 的 API 呼叫直接合併 PR 4，沒有其他變更，再次執行相同的已發布檢查：

```
$ state/task-f1.check.sh
merged
```

`localhost:3980` 登入設定的實際連接埠，從未出現在標準 PR URL 或輪詢主機比較中；登入設定比對一律只比對 hostname。因此，對於可透過標準連接埠存取、URL 不明列連接埠的執行個體，也能正確重現；這是實際自架環境常見的結構。

<a id="a-missing-tea-produces-no-wake-never-a-false-merge"></a>
## 缺少 tea 不產生喚醒，也不會誤報合併

```
$ PATH="$notea" bin/fm-pr-poll.sh --validated forgejo https://localhost/fixture/fixture-repo/pulls/1 localhost fixture/fixture-repo 1
(nothing)

$ PATH="$notea" bin/fm-pr-check.sh task-x https://localhost/fixture/fixture-repo/pulls/1
error: watching a Forgejo pull request requires tea on PATH
```

<a id="merging-teas-merge-subcommand-cannot-bind-a-head-but-the-api-can"></a>
## 合併：tea 的 merge 子指令無法綁定 head，但 API 可以

`tea pulls merge --help` 沒有列出 `--sha`／head commit 旗標，不像 `gh pr merge --match-head-commit` 與 `glab mr merge --sha`。
Forgejo／Gitea 底層 REST API 確實支援，以 tea 未記載的合併請求本文欄位 `head_commit_id` 提供（已對照執行個體自行提供的 Swagger 定義 `MergePullRequestOption` 確認），因此 `bin/fm-pr-merge.sh` 透過 `tea api` 直接呼叫該 API，而非 `tea pulls merge`：

```
$ tea api --login fm-fixture --repo fixture/fixture-repo -X POST '/repos/{owner}/{repo}/pulls/2/merge' \
    -f Do=merge -f head_commit_id=0000000000000000000000000000000000000000 -i
HTTP/1.1 409 Conflict
{"message":"head out of date", ...}

$ tea api --login fm-fixture --repo fixture/fixture-repo -X POST '/repos/{owner}/{repo}/pulls/2/merge' \
    -f Do=merge -f head_commit_id=<the real head sha> -i
HTTP/1.1 200 OK
```

過期或錯誤的 head 會以 `409 head out of date` 拒絕；精確符合目前 head 才成功並真正合併（讀回 `state` 為 `closed`、`merged: true`）。
這與 `--match-head-commit`／`--sha` 完全等效：若驗證讀取與合併呼叫之間有新 push 落地，就會拒絕，而不合併未經審查的內容。
`Do` 欄位是呼叫端指定的合併方式（`--merge`、`--rebase`、`--squash` 或 `--method <style>`）；未指定時，使用儲存庫自己的 `default_merge_style`，透過同一登入設定即時讀取 `/repos/{owner}/{repo}`；若無法讀取該值，就拒絕合併。

**`tea api` 回報 HTTP 層級失敗時，結束狀態仍為 0。**上方兩次呼叫都以 0 結束，只有 JSON 本文不同。
這是本次驗證最具影響的發現：若實作只依 `tea api` 結束狀態判定成功，就會將每次遭拒的合併都當作成功。
因此 `bin/fm-pr-merge.sh` 絕不信任該結束狀態；它一律在之後重新讀取 Pull Request（`forgejo_confirm_merged`），只將 `merged == true` 視為合併已落地的證據。

<a id="pre-merge-conditions"></a>
## 合併前條件

即時讀取 `tea api ... /repos/{owner}/{repo}/pulls/{n}`（`state`、`mergeable`、`draft`、`head.sha`）及 `tea api ... /repos/{owner}/{repo}/commits/{sha}/status?limit=50&page={p}`（合併提交狀態）。
合併狀態端點從每個 context 最新狀態的一頁資料計算 `state`，因此必須讀取每一頁直到空頁；只有每個非空頁都為 `success`，才認定 head 全綠。
明確的 `draft` 讀值會拒絕合併；若執行個體太舊而未回報此欄位，伺服器端仍會拒絕工作中 PR 的合併。
`tea api` 會原樣傳遞查詢字串，因為它只是將請求 URL 串接到登入設定的 `/api/v1` 基底。
已合併 Pull Request 的第二次合併嘗試，會被正確拒絕：

```
$ fm-pr-merge.sh task-f1 https://localhost/fixture/fixture-repo/pulls/4
error: refusing to merge https://localhost/fixture/fixture-repo/pulls/4
  - state is "closed", not open
  - the combined commit status is "none", not success
```

開啟中、可合併但未設定 CI 的 Pull Request，會回報空白合併狀態與 `total_count: 0`：

```
$ curl .../repos/fixture/fixture-repo/commits/<sha>/status
{"state":"","sha":"","total_count":0,"statuses":null,...}
```

這會被讀為 `none`，如同 `docs/gitlab-merge-watch.md` 將 GitLab 的 null `head_pipeline` 讀為 `none`，並拒絕合併，而非把缺少檢查視為無須驗證：

```
$ fm-pr-merge.sh task-f2 https://localhost/fixture/fixture-repo/pulls/<open-mergeable-pr>
error: refusing to merge ...
  - the combined commit status is "none", not success
```

**此處未驗證：**真正 Forgejo Actions 執行所產生、含資料的非空合併狀態回應結構、其分頁，以及 Pull Request 的 `draft` 欄位。
建立 Forgejo Actions runner 不在本次範圍內；合併關卡要求合併狀態的 `state` 精確為 `success`，與 GitHub、GitLab 自己的狀態列舉一致；其他任何值，包括無法識別的值，一律拒絕，與此處所有條件相同，採安全失敗即拒絕的原則。

<a id="registered-tea-logins-used"></a>
## 使用的已註冊 `tea login`

`fm-fixture`（`http://localhost:3980`）專為本次驗證建立，事後與測試夾具執行個體一併移除。
私人執行個體上原有的兩個真實登入設定（`forgejo`、`forgejo-admin`），只透過 `tea login list` 讀取，以真實資料證明主機比對邏輯；沒有讀取、合併或以其他方式觸碰其中任何 Pull Request。
