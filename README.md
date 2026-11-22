> **台灣繁體中文版**：此儲存庫是 [kunchenguid/firstmate](https://github.com/kunchenguid/firstmate) 的繁中 fork，文件與技能提示採台灣工程師慣用語。翻譯範圍、上游版本及保留原文的機器契約，請見 [翻譯說明](TRANSLATION.zh-TW.md)。
>
> 若要使用此繁中版本，請複製 `https://github.com/HUNG-YU-LI/firstmate-zh-TW.git`；原文上游的安裝命令仍保留供對照。

<h1 align="center">firstmate</h1>
<p align="center">
  <a
    href="https://img.shields.io/badge/platform-macOS%20%7C%20Linux-blue?style=flat-square"
    ><img
      alt="支援平台"
      src="https://img.shields.io/badge/platform-macOS%20%7C%20Linux-blue?style=flat-square"
  /></a>
  <a href="https://x.com/kunchenguid"
    ><img
      alt="X"
      src="https://img.shields.io/badge/X-@kunchenguid-black?style=flat-square"
  /></a>
  <a href="https://discord.gg/Wsy2NpnZDu"
    ><img
      alt="Discord"
      src="https://img.shields.io/discord/1439901831038763092?style=flat-square&label=discord"
  /></a>
</p>

<h3 align="center">只和一個 Agent 交談，由整支團隊完成交付。</h3>

<p align="center">
  <img alt="firstmate — 只和一個 Agent 交談，由整支團隊完成交付" src="assets/banner.png" width="100%" />
</p>

<a id="what-it-is"></a>
## 這是什麼

執行一個程式開發 Agent 很容易。
但只要你想同時完成三項專案工作，例如修正、調查、規劃或稽核，就會變成忙著切換分頁的人：盯著各個工作階段、在儲存庫之間複製貼上上下文，還會忘記哪個終端機裡有失敗的測試。

firstmate 翻轉了這種模式。
你只需和一個 Agent，也就是大副（first mate）交談，它就會替你帶領整支團隊：在可見的工作階段後端啟動自主 Agent，給每個 Agent 一份乾淨的 Git worktree，監督它們完成工作，再把完成的 PR、經核准的本機合併或獨立調查報告交給你。
船隊規模更大時，可以選擇啟用持續存在的二副（secondmate）：它們仍是一般的直接部屬，但各自在本機或可透過 SSH 連線的主機上，使用隔離的 firstmate home 執行。

firstmate 不是模型、執行環境、技能、MCP 伺服器，也不是 CLI。
firstmate 是一套用來帶領 Agent 團隊的 Agent 發行套件。
Agent 發行套件是一個可攜式目錄，包含指令、技能、工具、政策與狀態慣例，能把通用 Agent 轉成特定用途的 Agent。
不需要安裝應用程式：複製下來的儲存庫就是發行套件，內含 `AGENTS.md`、隨附的 firstmate 技能，以及任何終端機程式開發 Agent 都能遵循的輔助指令碼。
在這個目錄內啟動支援的執行環境作為主要工作階段，就建立了你的大副，而你就是船長。

<a id="features"></a>
## 功能

- **單一聯絡窗口** — 你只和大副交談；它負責派工、監督，只在真正需要決策時向上呈報，並以易懂的方式回報成果。
- **看得見的團隊** — 每位船員在自己的 tmux 視窗或 Herdr 分頁工作，也可以使用實驗性的 Zellij 分頁、cmux 工作區、Tern 區塊、Orca 終端機或 T3 Code 對話串；你能觀看或互動，大副會協調狀態。
- **用完可清除的 worktree** — 每項工作都在乾淨的 [treehouse](https://github.com/kunchenguid/treehouse) Git worktree 中執行；`backend=orca` 時則使用 Orca 管理的 worktree，因此同一儲存庫的並行工作不會互相衝突。
- **兩種工作類型** — ship 工作交付已授權的變更；當受理契約需要獨立研究時，scout 工作產出獨立調查報告。
- **明確的專案模式** — 各專案透過 `no-mistakes`、`direct-PR` 或 `local-only` 交付；可選擇加入 `+yolo` 合併自主權旗標、以 `branch=<prefix>` 覆寫預設的 `fm/` 交付分支前綴、使用 `forge=gerrit` 綁定讓工作 Agent 發布 Gerrit change 而非建立 pull request，或加入 `pr=manual`，由工作 Agent 撰寫 PR 標題與內容，而不是交給管線。
- **自身儲存庫邊界** — home 預設以 firstmate 使用者／操作者身分執行，因此 firstmate 的問題可以在本機診斷與修補，但不會自行向上游推送、建立 PR、開 Issue、留言或 fork；要發布到 firstmate 自己的儲存庫，必須在 `config/firstmate-role` 明確啟用維護者角色，或取得船長針對該工作的同意（[結構定義](docs/configuration.md#firstmate-role-configfirstmate-role)）。
- **可選的二副** — 選擇啟用持續存在的二副，各自在隔離的 firstmate home 中執行，具有自己的 `FM_HOME`、狀態、專案與工作階段鎖；可在本機執行，也可將整個 home 放在可透過 SSH 存取的主機。更新與復原都有防護，遠端路由不可用時絕不擅自改成本機替代。
- **事件驅動、零 Token 的監督** — Bash 監看程式平時等待船隊事件，只有需要你處理時才喚醒大副；經驗證的主要執行環境還有回合結束防護，在工作仍進行且監督未啟動時，阻擋或接續處理未確認狀態就停止的情況。
- **可選的 Relay** — 透過本機 `.env` 中的一個配對權杖啟用，讓 firstmate 回覆 X 與 Discord 上公開提及你的訊息，使用和對話要求相同的生命週期處理一般可逆要求，確認已啟動的工作，並在七天內針對實際里程碑與最終成果，最多發布三則適合公開的後續回覆，且不改變非 Relay 的行為；若承諾在對話串中提供最終回覆，該承諾會成為可從磁碟對帳的持久化狀態，因此重新啟動或壓縮對話都不會遺失；正式啟用前，dry-run 預覽會先在本機記錄預計發送的回覆與略過項目。
- **嚴格的專案邊界** — 除了[硬性規則 1](AGENTS.md#1-identity-and-prime-directives) 授權的少數受防護、經船長核准的操作，包括 fleet sync 受防護的安全分支清理，大副對你的專案一律唯讀；其他所有專案變更都由船員在設定好的合併授權之下執行。
- **不怕重新啟動** — 所有狀態都存於磁碟與目前使用的工作階段後端（參見[後端選擇](docs/configuration.md#backend-selection-order)）；重新啟動後，下個工作階段會完成狀態對帳，而一般監督流程也會主動復原已確認終止的二副 Agent，不必等待重新啟動。

各項功能的完整細節請見 [docs/architecture.md](docs/architecture.md)。

<a id="quick-start"></a>
## 快速開始

<a id="requirements"></a>
### 需求

- 經驗證的主要 Agent 執行環境：Claude Code、Grok、Pi、`pi-signed`、Oh My Pi（`omp`）、Codex、OpenCode、Cursor Agent CLI 或 Antigravity CLI（`agy`）。
- Git 與 GitHub CLI，並透過 `gh auth login` 完成驗證。
- 所選執行階段後端的 CLI 與相依套件；tmux 是預設參考後端。

大副會偵測支援範圍內缺少或過舊的工具，並在取得你的核准後提供安裝或升級。
各後端專屬設定請見[文件](#documentation)中的連結。

<a id="recommended-harnesses"></a>
### 建議的執行環境

執行 firstmate 主要工作階段時，**Claude Code、Grok 與 Pi 都是同等優先的建議選項**；`pi-signed` 也受支援，並以 Pi 獨立的簽章包裝器身分識別。
Claude Code 使用納入版本控制的 Stop 與 StopFailure hook，提供[不耗 Token 的監看程式重新啟用與喚醒](docs/watcher-continuity.md#claude-stop-hook)，也涵蓋 API 錯誤造成的回合結束；Grok 使用背景通知喚醒週期；Pi 使用納入版本控制的主要監看擴充功能。
依各自文件完成設定並啟動時，三者都有經過驗證的回合結束防護路徑。
選擇最符合你訂閱與工作流程的工具即可。

Oh My Pi（`omp`）是 Pi 的 fork，已驗證可作為主要執行環境；它採用與 Pi 相同、由擴充功能持有監看程式的模式，並提供更強的回合結束防護：阻擋式 `session_stop` hook 會強制繼續執行，而非只是提出請求。
Codex 與 OpenCode 也已驗證並支援作為主要執行環境；Codex 預設使用有界的前景檢查點，除非 home 選擇啟用不需模型的原生 Stop 停駐機制；OpenCode 則使用 TUI 外掛，因此相較於三個同等優先選項，兩者有更多執行環境專屬的監督取捨。
Cursor Agent CLI 也已驗證可作為主要執行環境，使用納入版本控制、以專案為範圍的 `.cursor/hooks.json`，其 `stop` hook 會在回合之間停駐於監看程式，形式最接近 Claude Code。
啟動時必須加上 `--trust`，否則專案 hook 都不會載入；無介面的 `cursor-agent -p` 也沒有回合結束 hook，因此主要工作階段請使用互動模式。
Antigravity CLI（`agy`）同樣已驗證可作為主要執行環境，使用納入版本控制、專案本機的 `.agents/hooks.json`，其 Stop hook 與 Cursor 一樣會在回合之間停駐於監看程式。

<a id="install-and-launch"></a>
### 安裝與啟動

```sh
gh auth login
git clone https://github.com/kunchenguid/firstmate
cd firstmate
```

接著啟動其中一個同等優先的執行環境；之後由 AGENTS.md 接手：

**Claude Code**

```sh
claude
```

**Grok**

```sh
grok --trust
```

**Pi**

```sh
pi
# 或者，已安裝簽章包裝器時
FM_PI_HARNESS=pi-signed pi-signed
```

**Oh My Pi**

```sh
omp
# 或者，從 Claude Code 窗格內啟動時
FM_OMP_HARNESS=omp omp
```

請以這份 checkout 作為工作目錄啟動 `omp`：它會自動探索納入版本控制的 `.omp/extensions/*.ts`，且不顯示信任對話框；若又透過 `-e` 指定，會讓每個擴充功能載入兩次。

Grok 每份 clone 都需要執行一次 `--trust`，專案 hook 與回合結束防護才會載入；也可在 Grok 內使用 `/hooks-trust`。
Pi 每份 clone 首次啟動時，需要核准一次專案信任提示，才會自動載入納入版本控制的 `.pi/extensions/*.ts`。
Pi 的 `/calm` 切換，以及 Claude Code 在預設關閉的搶先體驗 function-hooks 旗標下提供的同一功能，會隱藏支援的對話紀錄介面元素，包括依標準分類的 Firstmate 操作用戶訊息列；執行中會使用 Calm 專屬的動態工作船圖示，同時保留所有模型上下文與工作階段資料。
Calm 只改變呈現方式，不改變被隱藏操作輸入的使用者角色傳遞、順序、權限、持久化或匯出行為。
偏好會保留在實際生效的 Firstmate home 中，關閉後即可還原一般呈現。
[Calm 目前的行為與支援限制](docs/calm.md)，和[依版本區分的維護者驗證證據](docs/calm-mode-feasibility.md) 分別記錄。
Pi 的 `/supervision-model` 指令，可從 Pi 本身回報的合格模型與思考層級中，僅為監督分支固定較便宜的模型與較低的推理強度；未固定時，該分支通常沿用你目前對話的模型與強度。請見[設定結構定義](docs/configuration.md#pi-supervision-branch-model-and-effort-configsupervision-branch-model-configsupervision-branch-effort)。

<a id="talk-to-it"></a>
### 和它交談

```sh
> ahoy！看看我的 GitHub 專案 xyz，接著修好不穩定的登入測試，並加入深色模式

# firstmate 會檢查工具鏈（安裝或升級任何項目前都先取得你的同意），
# 將專案複製到 projects/，並在目前後端啟動兩個隔離的工作 Agent。
# 幾分鐘後：

  船長，PR 已可供審查：https://github.com/you/xyz/pull/42
  （修正不穩定的登入測試 — 風險：低 — CI 通過）

> 好，把它合併
```

<a id="more-backends"></a>
### 更多後端

tmux（預設）與其他所有支援後端（herdr、zellij、Orca、cmux、Paseo、Rex、T3 Code、tern、hc）的設定指南，都列在下方[文件](#documentation)中。

<a id="how-it-works"></a>
## 運作方式

```
            你（船長）
                  │  對話：要求、決策、「合併吧」
                  ▼
 ┌─────────────────────────────────────┐
 │ firstmate            （本儲存庫）   │
 │ 讀取 projects/ 與 firstmate 路由    │
 │ 受防護地寫入 backlog／briefs／state │
 └──┬──────────────┬───────────────┬───┘
    │ 後端傳送／狀態檔案           │
    ▼              ▼               ▼
 ┌────────┐   ┌────────┐      ┌────────┐
 │fm-task1│   │fm-task2│  ... │fm-taskN│   每項工作各有可見端點
 │ 船員   │   │ 船員   │      │ 船員   │   每個端點各有自主 Agent
 └───┬────┘   └───┬────┘      └───┬────┘
     ▼            ▼               ▼
  treehouse worktree、Orca worktree 或隔離的 secondmate home
     │
     ├─ ship：專案模式 ► PR／本機合併 ► 清理
     │
     └─ scout：data/<id>/report.md 報告 ► 決策清單 ► 轉達發現 ► 清理
```

你和大副交談。
它把每個要求交給具有獨立工作階段端點與 Git worktree 的船員，以零 Token、事件驅動的監看程式監督船隊，再把完成的 PR、經核准的本機合併或調查報告帶回給你。
可選的二副將這套模式延伸到持續存在的本機或整個 home 部署於遠端的二副；派工設定檔讓你決定哪種執行環境處理哪項工作；選擇啟用的 Relay 則讓同一支船隊能回覆公開提及。
`codex-app` 目前還不是執行階段後端；Codex App 的邊界以 [docs/codex-app-backend.md](docs/codex-app-backend.md) 為準。

完整架構，包括監督引擎、worktree 隔離、二副、派工設定檔、專案模式、可選的 Relay、船隊同步與自行更新，請見 [docs/architecture.md](docs/architecture.md)。

<a id="built-in-skills"></a>
## 內建技能

Firstmate 隨附以下可由使用者呼叫的內建技能。
Claude 與 grok 使用此處的斜線形式；codex 使用相同名稱搭配 `$`，例如 `$afk`。

| 技能 | 功能 |
| --- | --- |
| `/afk` | 進入離席監督模式：你離開時，由 Pi 程序內的分支、其他主要執行環境旁的[監督主機](docs/configuration.md#supervision-host-configsupervision-host)（Claude 預設開啟），或背景常駐程序處理喚醒；模式與返回契約請見[離席程序](.agents/skills/afk/SKILL.md) |
| `/quiet` | 你仍在場並繼續交談時，讓例行喚醒不出現在主對話；要求的操作立即執行，不等你返回。如果 Pi 分支或[有人在場的監督主機](docs/supervision-host.md#quiet-mode) 已提供此行為，就只回報這點；否則啟動 quiet 常駐程序，持續經過一般對話保持運作，直到 `/quiet off` |
| `/ahoy` | 回顧上一則真正船長訊息以來可見的工作階段事件，以及可見但尚未回答的船長決策，再依 Agent 判斷的影響順序，逐一引導船長處理未結決策；若這是工作階段中的第一則真正船長訊息，則改用 Bearings |
| `/bearings` | 根據有界的船隊狀態，在對話中產生簡潔的四節摘要，包含已登錄遠端 home 的帳本與所屬貢獻的實測追蹤；使用 `/bearings file` 可一併覆寫 `data/` 中今天日期的報告，加上 `include PRs` 則會取得即時 GitHub 補充資料 |
| `/lavish-url <url> [--mobile]` | 將不含 URL 憑證的 http 或 https 網頁，不論公開或本機，擷取為可加註的 Lavish 審查快照，提供 `[Fix Layout]`、`[Update Text]`、`[Mobile Bug]` 標籤與能識別專案的元素標示；加上 `--mobile` 可模擬行動裝置。回饋以通知返回，並歸入擁有該網站之已登錄專案的待辦清單；若無法確定網站屬於哪個專案，則提出決策 |
| `/updatefirstmate` | 在防護下更新目前執行的 firstmate 與二副：快轉，或處理 squash merge 後多餘的分歧；接著為所有成功停留在目標提交的運作中大副／二副保存狀態並重新啟動，包括本來就已是最新的 home。只有無法證明已重新啟動時，才如實要求重新讀取 |
| `/stow` | 掃描工作階段中尚未記錄的持久知識，保存此階段已知尚未歸檔或已失準的未結工作紀錄，透過衰減與冷封存整理分層啟動記憶，落實各 home 的預算或提出必要決策，串接處理已登錄二副，並回報哪些部分可以安全重設 |

Bearings 呼叫範例：

- `/bearings` 只在對話中回傳最新的四節摘要。
- 所屬貢獻的後續追蹤來自快取的涵蓋範圍投影；若要取得整個儲存庫的即時 PR 補充資料，仍須選擇加入 `include PRs`。
- `/bearings include PRs` 維持僅對話模式，並啟用即時 PR 補充資料。
- `/bearings file` 會從頭產生並覆寫今天的 `data/status-report-<YYYY-MM-DD>.md`，並在四節對話摘要中附上連結。
- `/bearings file include PRs` 結合附日期報告與即時 PR 補充資料。

僅供 Agent 使用的參考技能放在 `.agents/skills/`，由 firstmate 在 [`AGENTS.md`](AGENTS.md) 指定的觸發點載入。

<a id="two-tier-skill-layout"></a>
### 兩層技能配置

Firstmate 的技能分別放在兩個位置，服務不同對象：

- `.agents/skills/` — 由 Agent 載入的技能，包括本節表格中的技能，以及 firstmate 僅供 Agent 使用的參考技能。每個技能都假設有一個運作中的 firstmate home；安裝到其他位置時，可能毫無意義，甚至造成誤導，因此各技能的 frontmatter 都帶有 `metadata.internal: true`。這個旗標會讓它們不出現在安裝工具的探索結果中，例如 [skills.sh](https://skills.sh) 的 `npx skills add` 安裝程式，但不影響 firstmate 自己載入；frontmatter 中繼資料不會改變 Agent 自身技能載入器的行為。
- `skills/` — 面向安裝工具的公開技能，可獨立安裝到任何專案，不依賴 firstmate。
  每個技能都自給自足，不依賴 firstmate 的路徑、工具或專用詞彙。
  目前提供的是 `skills/stow`，一個通用的工作階段知識掃描技能：先依明確指示分流發現，再遵循既有本機慣例，最後才退回私人 `.stow-notes.md`；並透過衰減、本機封存及使用者核准的按需卸載提案，整理分層條目。
  它刻意不與同名的 firstmate 內部 `.agents/skills/stow` 共用程式碼，讓兩者可以獨立演進。

<a id="documentation"></a>
## 文件

- [docs/architecture.md](docs/architecture.md) — 提供維護者的團隊、監督、worktree、二副與專案模式架構。
- [docs/configuration.md](docs/configuration.md) — 環境變數、`FM_HOME`、執行階段後端選擇、可選的 Relay 與 X／Discord 設定步驟、受信任外部程序事件介接器設定、可設定的檔案，以及執行環境支援。
- [docs/extension-bindings.md](docs/extension-bindings.md) — 提供維護者的受限、受信任外部 `process-event-adapter/1` 套件、綁定、交握與證據邊界架構。
- [docs/remote-secondmates.md](docs/remote-secondmates.md) — 整個 home 部署於遠端之二副的目前設定、路由、傳輸、復原與安全行為。
- [docs/calm.md](docs/calm.md) — Pi 與 Claude Code 上目前的 `/calm` 行為及支援的呈現限制。
- [docs/voice-relay.md](docs/voice-relay.md) — 可選的語音介面：兩台機器的設定、實測往返成本、語音回答可讀取的範圍，以及此版本尚未提供的功能。
- [docs/fleet-ledger.md](docs/fleet-ledger.md) — 可選的活動帳本，供外部工具讀取並追蹤 home 的工作，以及其紀錄契約。
- [docs/wedge-alarm.md](docs/wedge-alarm.md) — 設定離席模式中向上呈報卡住時的主動警示。
- [docs/usage-warner.md](docs/usage-warner.md) — 設定可選的 Claude 用量區間本機門檻警示。
- [docs/tmux-backend.md](docs/tmux-backend.md) — tmux 參考後端的目前設定與限制。
- [docs/herdr-backend.md](docs/herdr-backend.md) — Herdr 後端的目前設定、CI 涵蓋範圍、安全邊界與限制。
- [docs/zellij-backend.md](docs/zellij-backend.md) — 實驗性 Zellij 後端的目前設定與限制。
- [docs/orca-backend.md](docs/orca-backend.md) — 實驗性 Orca 後端的目前設定與限制。
- [docs/cmux-backend.md](docs/cmux-backend.md) — 實驗性 cmux 後端的目前設定、socket 安全性與限制。
- [docs/paseo-backend.md](docs/paseo-backend.md) — 實驗性 Paseo 後端的目前設定、終端機名稱路由與限制。
- [docs/rex-backend.md](docs/rex-backend.md) — 實驗性 Rex 後端的目前設定、自動偵測、區塊 ID 路由、不存在證明與限制。
- [docs/t3code-backend.md](docs/t3code-backend.md) — 實驗性 T3 Code 後端的目前設定、登入與限制。
- [docs/tern-backend.md](docs/tern-backend.md) — 實驗性 tern 後端的目前設定、畫面擷取、原生狀態退化處理與限制。
- [docs/hc-backend.md](docs/hc-backend.md) — 實驗性 hc（herdr-core）後端的目前設定、工作標籤與限制。
- [docs/codex-app-backend.md](docs/codex-app-backend.md) — 目前受阻的 Codex App 後端邊界與推出契約。
- [docs/verification/runtime-backends.md](docs/verification/runtime-backends.md) — 執行階段後端保證的現行維護者驗證。
- [docs/gerrit-forge-integration.md](docs/gerrit-forge-integration.md) — 提供維護者的託管平台軸架構：為什麼以 change 為單位的審查不是單純的平台變體、mode／forge／shape 組合測試，以及平台機制的責任歸屬。
- [docs/external-review-forge.md](docs/external-review-forge.md) — 提供維護者的 `forge=external` 架構，由審查工具自行上傳快照：禁止推送的不變條件與三層防護、個人備份 remote，以及接受的殘餘風險。
- [docs/gitlab-merge-watch.md](docs/gitlab-merge-watch.md) — 在任意 GitLab 執行個體監看與合併 merge request 的維護者驗證。
- [docs/gerrit-change-watch.md](docs/gerrit-change-watch.md) — 唯讀監看 Gerrit change 的維護者驗證，以及合併路徑拒絕處理它的原因。
- [docs/forgejo-tea-integration.md](docs/forgejo-tea-integration.md) — 透過 `tea` 監看與合併 Forgejo／Gitea pull request 的維護者驗證。
- [docs/turnend-guard.md](docs/turnend-guard.md) — 主要工作階段目前「不盲目結束回合」的防護、範圍、迴圈安全性與相容性限制。
- [docs/verification/supervision.md](docs/verification/supervision.md) — 工作階段啟動、防護、連續性與卡住偵測整合的現行維護者驗證。
- [docs/supervision-protocols/](docs/supervision-protocols/) — 已展開的主要執行環境監看協定，涵蓋 Claude、Codex、OpenCode、Pi 與 `pi-signed`、omp、Grok、Cursor，以及未知執行環境的替代路徑。
- [docs/scripts.md](docs/scripts.md) — `bin/` 工具組參考。
- [docs/documentation-audiences.md](docs/documentation-audiences.md) — 文件讀者分類與由機器檢查的放置邊界。
- [`AGENTS.md`](AGENTS.md) — 監督者契約、角色邊界，以及依條件載入程序的路由索引。
- [CONTRIBUTING.md](CONTRIBUTING.md) — 貢獻方式，包含開發與測試指令。

<a id="contributing"></a>
## 參與貢獻

歡迎貢獻；工作流程、儲存庫慣例與測試執行方式請見 [CONTRIBUTING.md](CONTRIBUTING.md)。

<a id="license"></a>
## 授權

MIT，請見 [LICENSE](LICENSE)。

<a id="star-history"></a>
## Star 歷史

<a href="https://www.star-history.com/?repos=kunchenguid%2Ffirstmate&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=kunchenguid/firstmate&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=kunchenguid/firstmate&type=date&legend=top-left" />
   <img alt="Star 歷史圖表" src="https://api.star-history.com/chart?repos=kunchenguid/firstmate&type=date&legend=top-left" />
 </picture>
</a>
