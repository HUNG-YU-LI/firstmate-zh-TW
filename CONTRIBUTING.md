<a id="contributing"></a>
# 參與貢獻

感謝你願意參與貢獻。
先說明一條規則：

**由人類建立、目標為 `main` 的 pull request，必須透過 [`no-mistakes`](https://github.com/kunchenguid/no-mistakes) 提出。**
這項要求是為了減輕維護者審查與合併貢獻的負擔。

`no-mistakes` 會在真正的 remote 前面放置本機 Git 代理。
透過它推送時，會在隔離的 worktree 中執行 AI 驅動的審查／測試／lint 管線；只有所有檢查都通過，才會把推送轉送到上游，並自動建立整理好的 PR。

GitHub Actions 檢查（`Require no-mistakes`）會在目標為 `main` 的 PR 上執行，要求同時具備確定性簽章，以及 no-mistakes v1.46.0 以上版本產生、可解析的結構化驗證聲明。
聲明必須綁定目前的 PR head 提交，且將 review、test 與 document 步驟回報為已完成；過期聲明、缺少 `head_sha`，或略過必要步驟，都會造成失敗。
每次開啟 PR 或編輯內文都會獨立評估，head 同步或重新開啟後也會重跑，且後續編輯不能取代先前尚未完成的合規檢查。
GitHub Actions 與 Dependabot 可豁免，以維持自動化運作；其他不符合驗證聲明契約的貢獻 PR，不會進行審查或合併。

<a id="workflow"></a>
## 工作流程

1. Fork 儲存庫，再複製上游儲存庫，或把本機 `origin` 設回上游（`git@github.com:kunchenguid/firstmate.git`）。
2. 建立分支並進行變更。
3. 以你的 fork 作為推送目標初始化關卡：`no-mistakes init --fork-url git@github.com:<you>/firstmate.git`（貢獻 firstmate 需要 **no-mistakes v1.46.0+** 提供結構化驗證聲明；未使用 fork 時，具有推送權限的維護者仍可直接使用 `no-mistakes init`）。
4. 提交變更。
5. 透過關卡推送，不直接推送到 `origin`：

   ```sh
   git push no-mistakes
   ```

6. 執行 `no-mistakes` 連接管線、查看發現、授權自動修正，並視需要審查 ask-user 發現。
   關卡操作機制請遵循已安裝 no-mistakes 版本的 SKILL.md，以及即時 `axi` 說明。
7. 管線通過後，會把分支推送到你的 fork，並替你向上游儲存庫建立 PR。

<a id="the-full-ci-label"></a>
### `full-ci` 標籤

沒有寫入權限的作者提出 pull request 時，CI 最初只執行快速檢查：`Test coverage guard`、`Repo invariants` 與 no-mistakes 內文檢查。
Lint 與行為測試套件會等維護者或維護者的船隊接手審查，加入 `full-ci` 標籤後才執行。
在此之前，`Full suite` 檢查會失敗，訊息會指出所需標籤；這是預期行為，你不需要處理。
加入標籤後會執行完整套件，之後每次推送到該 pull request 都會再次執行。
這能讓公開儲存庫的 CI 維持在 GitHub 同時執行工作數量的限制內，同時 no-mistakes 已在本機審查、測試並 lint 你的變更。
來自本儲存庫分支，或作者具有寫入權限的 pull request，不需要標籤就會執行完整套件。

完整首次操作步驟請見 [no-mistakes 快速開始](https://kunchenguid.github.io/no-mistakes/start-here/quick-start/)。

<a id="agent-driven-contributions"></a>
## 由 Agent 進行貢獻

派發 Agent 貢獻工作前，請遵循 [Firstmate 角色與工作同意政策](docs/configuration.md#firstmate-role-configfirstmate-role)；僅僅複製並執行，並不會取得任何發布權限。

<a id="maintaining-required-checks"></a>
## 維護必要檢查

GitHub 的必要檢查設於儲存庫既有的 main 規則集，不會因為提交 workflow YAML 就自動啟用。
套用這套 CI 配置時，請保留既有的 pull request、合併方式、線性歷史、刪除、非快轉與管理員略過設定。
加入必要狀態檢查時，使用 `strict_required_status_checks_policy: false`；僅 main 更新，不應強迫分支更新並重新測試。
請將檢查綁定到已在產生這些結果的 GitHub Actions App，而不是接受任何整合來源提供的同名 context。
此設定不需要安裝新的 App 或手動設定 runner。

請要求實際的 job context：`Full suite`、`Lint 1`、`Lint 2`、`Test coverage guard`、`Repo invariants`、`Stock macOS Bash snapshot compatibility`、`Behavior portable parallel 1`、`Behavior portable parallel 2`、`Behavior portable serial 1` 到 `Behavior portable serial 10`、`Behavior tests (Herdr)`、`Behavior timing aggregate`，以及 `PR must be raised via no-mistakes`。
最後一項是合規工作的 context 名稱，不是 workflow 標題；既有自動化例外維持不變。
計時彙總不能取代各個獨立工作，因為它收集失敗執行的證據時仍可能成功。
外部 pull request 在加入 `full-ci` 標籤前，各執行通道會略過，而略過的工作仍算滿足必要檢查，因此也必須要求 `Full suite`：未加入標籤，或任一通道失敗／取消時，它都會失敗。

只有在對應 workflow 通過並合併後，才能套用已核准的規則變更；先從實際檢查確認精確名稱與 Actions 整合 ID。
先建立目前規則集的快照，再透過已驗證的 GitHub API 或設定介面修改同一條規則，並重新讀取規則集與分支實際生效的規則。
不建立測試合併，直接驗證缺少或失敗的檢查會阻擋一般合併；管理員覆寫功能刻意保留。
回復 workflow 時，需一併協調必要檢查名稱，避免已停用的檢查讓一般合併永遠等待。

<a id="repo-conventions"></a>
## 儲存庫慣例

- 本儲存庫是一個執行 firstmate 協調 Agent 的範本。
  [`AGENTS.md`](AGENTS.md) 是監督者契約、角色邊界與隨附 firstmate 技能觸發條件的權威來源；`CLAUDE.md` 是真正的 `@AGENTS.md` 匯入指標，`.claude/skills` 則是指向 `.agents/skills` 的符號連結。
- 只將共用素材納入版本控制：`AGENTS.md`、`README.md`、`CONTRIBUTING.md`、`.tasks.toml`、`.github/workflows/`、`bin/`、`.agents/skills/` 與 `skills/`。
  `.agents/skills/` 放置由 Agent 載入、假設有運作中 firstmate home 的技能，並帶有 `metadata.internal: true`，讓 [skills.sh](https://skills.sh) 等安裝工具不列入探索結果；`skills/` 則放置不依賴 firstmate、面向安裝工具的獨立公開技能（請見 README 的「兩層技能配置」）。
  `.claude/mods/` 放置 Claude Code mod，也就是將行為集中在單一 function-hooks 模組的外掛；每個 mod 都經由 `.agents/skills/<mod>` 符號連結存取，因為 Claude Code 只從 `.claude/skills` 採用專案外掛；它們不包含 `SKILL.md`，讓其他執行環境的技能載入器忽略該項目，且只匯入實際位於自身資料夾內的檔案，因為 Claude Code 會拒絕其他位置。
  模組可透過 `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS` 或 Claude Code 的 `tengu_plugin_hooks_modules` 推出旗標載入，但 Calm mod 只有在 `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS` 恰好為 `1` 時才啟用，否則完全不執行任何動作；Firstmate 絕不在任何設定檔中設定這個變數，契約以 [`docs/calm.md`](docs/calm.md) 為準。
  屬於某位船長船隊的所有個人內容（`.env`、`data/`、`state/`、`config/`、`projects/`、`.no-mistakes/`）都列入 gitignore，絕不可提交。
  根目錄 `.tasks.toml` 是納入版本控制、供 `data/backlog.md` 使用的 `tasks-axi` 設定；相容的 `tasks-axi` 是例行待辦清單修改的預設後端，相容性定義以 [`docs/configuration.md`](docs/configuration.md) 的「Backlog backend」為準。
  本機 `config/backlog-backend=manual` 可選擇停用，強制 firstmate 以手動編輯處理例行待辦清單更新，且該檔案維持 gitignore；經驗證的二副交接仍透過 `tasks-axi mv` 委派。
  本機 `config/backend` 檔案維持 gitignore；支援值、選擇順序與限制以[執行階段後端設定](docs/configuration.md#runtime-backend-configbackend--fm_backend) 為準。
  這不會讓 `data/` 納入版本控制。
- `bin/` 中的 Shell 輔助工具使用純 Bash。
  每個檔案開頭都有使用說明註解；改變行為時，請同步維持正確。
  `tests/` 中的測試指令碼與輔助工具也使用純 Bash。
  `bin/fm-lint.sh` 必須通過：它是 lint 定義的唯一來源，包含 shellcheck 檔案集合、設定、固定的 shellcheck 版本、固定的 actionlint workflow 檢查，以及禁止核心 `bin/` 指令碼直接呼叫 Beads CLI 的後端純度檢查。
  CI 使用完整的標準分區；no-mistakes 推送前關卡則使用依上下文選定的預設值。
  分區驗證與效能證據以 `docs/fm-test-portable-shards.md` 為準。
  精確的本機 lint 模式、檔案集合選擇與分析旗標，以檔案標頭及 `--help` 輸出為準。
  `.github/workflows/*.yml` 格式錯誤時，包括 `ci.yml` 自身損壞，都會在合併前由本機 lint 路徑檢出，因為損壞的 workflow 無法回報自身損壞。
  它固定一個精確的 shellcheck 版本與 actionlint 版本，其他版本一律拒絕執行。
  使用 `bin/fm-lint.sh --required-version` 顯示固定的 shellcheck 版本，使用 `bin/fm-lint-workflows.sh --required-version` 顯示固定的 actionlint 版本。
  使用 `bin/fm-install-shellcheck.sh` 與 `bin/fm-install-actionlint.sh` 在本機安裝這些精確版本；目的地用法與支援平台以各安裝程式的標頭為準。
- 執行環境介接器的責任涵蓋：`bin/fm-harness.sh` 的偵測、`bin/fm-spawn.sh` 的啟動與 hook 機制、`bin/fm-claude-trust.sh` 的啟動時 Claude 工作區信任及外部 CLAUDE.md 匯入預先核准、`bin/fm-busy-lib.sh` 的語意忙碌來源與信任關卡、`bin/fm-composer-lib.sh` 僅供傳遞使用的畫面防護、`bin/fm-teardown.sh` 的清理，以及根於 `.agents/skills/harness-adapters/SKILL.md` 技能樹的事實資料；依賴這些執行環境的檢查，其驗證政策以 `firstmate-coding-guidelines` 技能為準。
- 修改執行階段工作階段後端（`bin/fm-backend.sh`、`bin/backends/` 及透過它們派發的指令碼）時，將目前設定與限制放在對應後端指南，現行實測證據則放在 [`docs/verification/runtime-backends.md`](docs/verification/runtime-backends.md)。
- 說明文字的分類，以 [`docs/documentation-audiences.md`](docs/documentation-audiences.md) 及其供機器使用的清單為準；文件變更後執行 `bin/fm-doc-audience-check.sh`。
- Markdown 中，每個完整句子各占一行。
- `README.md` 維持簡潔概覽與連結，不放入大篇幅細節。
  請把細節放到最專門的 `docs/` 檔案，例如架構、設定或後端指南，再提供連結。

<a id="development"></a>
## 開發

對 firstmate 本身納入版本控制的內容進行變更，包括 `AGENTS.md`、`README.md`、`CONTRIBUTING.md`、`.tasks.toml`、`.github/workflows/`、`bin/`、`.agents/skills/` 與 `skills/`，都要在功能分支上透過 `no-mistakes` 管線交付，並取得明確合併核准。
進行任何這類變更前，請載入僅供 Agent 使用的 `firstmate-coding-guidelines` 技能（`.agents/skills/firstmate-coding-guidelines/SKILL.md`）。
其中包含知識放置規則，避免 `AGENTS.md` 每次精簡後又重新膨脹。
Firstmate 會手動將此技能的載入指示加入 firstmate 儲存庫的工作簡報；範本偵測的限制以[程式開發準則技能](.agents/skills/firstmate-coding-guidelines/SKILL.md#trigger-hygiene) 為準。
船員接手這類簡報時，即使簡報早於此指示建立，也應載入該技能。
監督運作中的船員時，將 firstmate 自身耗時的驗證或建置指令放在背景，讓監看程式的喚醒仍能得到處理。
船員驗證應遵循已安裝 no-mistakes 版本的 SKILL.md 與即時 `axi` 說明，不要在 firstmate 文件中重複記錄關卡機制。
Firstmate 的包裝層仍然重要：船員把每項 `ask-user` 發現送回 firstmate，由它套用 `ask-user-authority`；船員絕不傳入 `--yes` 或 `-y`，因為任一旗標都會略過該檢查及任何必要的船長決策呈報。
納入版本控制的 `.no-mistakes.yaml` 關卡預設值，以 [`docs/configuration.md`](docs/configuration.md#gate-defaults-no-mistakesyaml) 為準。
本機 no-mistakes Test 應針對意圖選測，而非設定 `commands.test`，這項規則以 `firstmate-coding-guidelines` 技能為準。
驗證方式應和關卡一致：針對關注主題使用 `bin/fm-test-run.sh`，不要串接 `bash tests/a.test.sh && bash tests/b.test.sh`，因為直接傳入多個指令碼路徑，也會取得和 `--changed` 相同的有界並行能力。
管線會自行發布證據，因此絕不可手動將 `.no-mistakes/` 路徑提交到功能分支；CI 會把它們視為納入版本控制的個人船隊路徑而拒絕。

推送前，檢查並測試工具組：

```sh
while IFS= read -r script; do /bin/bash -n "$script" || exit; done < <(bin/fm-lint.sh --list-files)   # 對 fm-lint.sh 涵蓋的 Shell 檔案檢查語法（本機為變更檔案，CI／main 為完整集合）
bin/fm-lint.sh   # 對該 Shell 範圍與 GitHub workflow 執行 lint，後者使用固定版本 actionlint；CI 與 no-mistakes 關卡都使用這個唯一入口
bin/fm-test-run.sh tests/<subject>.test.sh   # 單一指令碼（本機主要聚焦路徑，附計時）
bin/fm-test-run.sh tests/<a>.test.sh tests/<b>.test.sh   # 同時測試多個主題：自動有界並行
bin/fm-test-run.sh --family pure-contract-unit   # 一般依測試家族選擇的本機路徑（循序、附計時）
bin/fm-test-run.sh --changed   # 根據變更檔案選測的一般路徑，自動有界並行
bin/fm-test-run.sh --changed --jobs 1   # 明確覆寫為循序執行
bin/fm-test-run.sh --changed --max-wall-ms 300000   # 同一自動路徑，執行後檢查是否超過五分鐘
bin/fm-test-run.sh --proven-isolated --jobs 4   # 明確在本機並行執行逐一證明隔離的集合
bin/fm-test-run.sh --lane portable-serial   # 可攜式循序剩餘集合（watcher／AFK／tmux／有狀態測試）
bin/fm-test-run.sh --list-lanes   # 查詢精確通道名稱，包括目前 CI 循序分片
bin/fm-test-run.sh --check-coverage   # 證明可攜式分片＋循序＋循序分片＋Herdr 等於完整清單
bin/fm-test-run.sh --all   # 刻意執行完整回歸測試（本機可選的全測，不是 no-mistakes Test）
bin/fm-test-linux.sh   # 可選：在 Debian 容器與 GNU userland 中執行此分支變更的測試（需要 docker 或 podman）
bin/fm-test-isolation-proof.sh --list   # 已證明可攜且可並行的候選集合
bin/fm-test-isolation-proof.sh --jobs 4 --json /tmp/fm-isolation-proof.json   # 重跑可攜式候選集合的隔離證明
bin/fm-test-isolation-proof.sh --pool watcher-wake-lock --jobs 4   # 重跑已准入測試家族的證明
[ ! -L CLAUDE.md ] && cmp -s CLAUDE.md - <<'EOF'
<!-- Points Claude at AGENTS.md via import; edit AGENTS.md, not this file. -->
@AGENTS.md
EOF
[ "$(readlink .claude/skills)" = "../.agents/skills" ]
tmp=$(mktemp -d) && printf 'done: smoke\n' > "$tmp/smoke.status" && FM_STATE_OVERRIDE="$tmp" FM_SIGNAL_GRACE=1 FM_POLL=1 FM_HEARTBEAT=999999 bin/fm-watch-arm.sh  # 監看程式重新啟用的 smoke test（先輸出啟用狀態，再輸出需要處理的訊號）
```

`bin/fm-test-run.sh` 是行為套件選擇、可攜式 CI 通道組合、有界並行准入、逐指令碼計時標記、家族總計、涵蓋範圍防護與可選 JSON 計時產物的唯一權威來源。
旗標、家族標籤、通道與變更檔案對應表，以其標頭及 `--help` 為準；本節只記錄入口。
`bin/fm-test-isolation-proof.sh` 仍是可攜式候選證明與可重用家族證明執行環境的唯一來源；請見 `docs/fm-test-isolation-proof.md`。
可攜式分片平衡證據放在 `docs/fm-test-portable-shards.md`。
`bin/fm-test-linux.sh` 是可選工具，絕不屬於 CI；在 macOS 上，若分支測試依賴 `stat`、`date` 或程序行為，發布前請先執行，因為 BSD 與 GNU 工具有差異。
選測、執行環境偵測與結束碼，以其標頭為準。
依家族選測是一般本機路徑；`--all` 僅用於刻意執行完整回歸。
廣泛回歸由 CI 負責，涵蓋必要的可攜式並行分片、可攜式循序通道在不同 runner 上的分片、Herdr 通道、lint、不變條件、涵蓋範圍防護與原生 macOS Bash 相容性，定義於 [`.github/workflows/ci.yml`](.github/workflows/ci.yml)。
向 pull request 推送新 head 時，會取消該 pull request 尚在執行的 CI，確保只驗證目前 head；較新的 `main` 推送則取消較舊的 `main` 執行，確保只有最新合併的執行完成；這項契約、[`full-ci` 標籤](#the-full-ci-label) 中說明的關卡，以及兩者的理由，都以 workflow 為準。
Fork 可將儲存庫變數 `FM_CI_SKIP_PUSH` 設為 `1`，略過 `CI` 的 push 執行；也可將 `FM_CI_SKIP_MACOS` 設為 `1`，略過原生 macOS Bash 工作，而不必修改納入版本控制的檔案、造成與上游分歧。
每個略過都使用 job 層級條件，因此略過的工作會回報成功，不會讓 pull request 一直等待；兩個變數都未設定時，上游與所有 fork 的每個工作都照原本方式執行。
使用 `bin/fm-test-run.sh --list-lanes` 取得精確通道名稱；在本機重現通道時，使用 `--help` 查看 `--jobs` 規則及必要的關卡略過旗標。
不要修改測試套件有界條件等待中的 `sleep 0.1` 節奏。
這些 sleep 看起來像可省去的額外成本：watch-triage 套件（`fm-watch-triage.test.sh`、`fm-watch-triage-pause.test.sh` 與 `fm-watch-triage-wedge-defer.test.sh`）大約執行 1,900 次，在 macOS 上每次都有約 100ms 的固定排程喚醒成本；但這不是額外加到總時間上的成本，而是測試等待受測目標的方法，因為目標只會按 `fm-watch.sh` 自己一秒一次的 `FM_POLL` 節奏前進。
降低取樣頻率不會消除等待，只會延後偵測：把間隔提高到 0.5 秒，並依比例計入每次取樣，在 2026-09-03 接續量測未拆分套件時得到 435 秒與 440 秒，原樣指令碼則為 390 秒與 393 秒，因為約 40 次輪詢週期等待與約 73 次程序結束等待，每次都多付出最多半秒。
其中一些迴圈捕捉的是暫態，而非等待穩定條件，因此較粗的取樣可能直接跨過斷言需要的狀態。
列出 `tests/*.test.sh` 即可找到測試：每份都是名為 `<subject>.test.sh` 的獨立 Bash 指令碼，標頭註解說明涵蓋內容；把路徑傳給 `bin/fm-test-run.sh`，就能聚焦該主題並取得標準計時輸出。
共用測試輔助工具放在 `tests/lib.sh`（回報器、暫存根目錄、Git 測試資料）、`tests/fixtures.sh`（模擬工具鏈與啟動環境建構器）、`tests/wake-helpers.sh`、`tests/secondmate-helpers.sh`、`tests/supervision-host-helpers.sh`（拆分後監督主機套件的替身引擎測試資料）、`tests/watch-triage-helpers.sh`（拆分後 watch-triage 套件的共用測試資料），以及 `tests/git-config-helpers.sh`（將測試用 Git 與主機全域／系統設定及繼承的儲存庫位置，例如 `GIT_DIR`，隔離；`tests/lib.sh` 與 `tests/herdr-test-safety.sh` 已載入；兩者都未載入的套件，必須在第一次 Git 操作前自行載入，確保直接執行時仍維持隔離）。
請載入這些工具，不要把模擬工具鏈複製到新套件中。
測試資料會 exec 的每個主機工具，都應以 `command -v` 解析，不要寫死 `/bin` 路徑；將結果插入模擬工具 shim 本文（`tests/fm-backlog-atomicity.test.sh`），或透過環境變數傳入（`tests/fm-watcher-lock.test.sh`）：部分支援主機的 `/bin` 只有 `sh`，因此寫死 `/bin/ln`、`/bin/rm` 或 `/bin/ps` 的 shim 會指向不存在的路徑。
同樣的主機也會讓透過 `exec -a` 改名的 `sleep` 依 `argv[0]` 派發並立即結束，因此只需要可區分名稱的持有程序，應改用一般參數傳遞；只有測試主題本身就是 `argv[0]` 命名時，才保留 `exec -a`。
測試資料可以縮短正式程式的逾時時間，讓失敗路徑儘速返回，但不得短於高負載機器在該時間窗內完成真實工作的成本，例如 fork、exec、取得鎖、發布信標或第一次輪詢檢查。
案例斷言若不是針對逾時本身，該時間窗應高於實測高負載成本並留有餘裕；測試自身則用計次輪詢迴圈限制等待，因為它能隨負載延展，固定牆鐘時間預算則不能。
監督主機與 Stop hook 的交還等待是例外：請使用 `wait_until_deadline`（60 秒牆鐘預算，遠高於高負載下的交還成本），讓逾時時能輸出 `host.out`、`hook.out`／`hook.err`、監督主機日誌、健康檔案與程序樹。
需要真正可選後端或明確啟用的測試，例如真實 herdr／zellij／cmux／paseo／rex／tern／hc smoke test，以及真實 Pi 回歸測試，會自行略過，並印出啟用所需工具或環境條件，讓可攜式套件在沒有這些工具的機器上仍然安全。
完整的 [`tests/fm-watch-triage-pause.test.sh`](tests/fm-watch-triage-pause.test.sh) 套件，需要 `PATH` 上有相容的 `tasks-axi`，規格請見[待辦清單後端](docs/configuration.md#backlog-backend-taskstoml--configbacklog-backend)。
真實 Rex 套件的私人伺服器隔離與能力關卡，以 [`tests/rex-test-safety.sh`](tests/rex-test-safety.sh) 為準。
通道隔離邊界以 [Herdr 後端指南](docs/herdr-backend.md#destructive-lab-safety) 為準；現行實測證據則以[執行階段後端驗證](docs/verification/runtime-backends.md#herdr) 為準；真實執行環境的憑證測試仍須選擇啟用。

<a id="questions"></a>
## 問題

請開 Issue，或到 [Discord](https://discord.gg/Wsy2NpnZDu) 找我討論。
