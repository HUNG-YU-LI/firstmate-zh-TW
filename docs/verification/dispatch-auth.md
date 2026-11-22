<a id="dispatch-authentication-verification"></a>
# 派工驗證資訊的驗證

適用對象：維護者驗證。

這份紀錄支持 `.agents/skills/quota-array-dispatch/SKILL.md` 的派工判斷規則，以及 `bin/fm-vendor-auth-probe.sh` 的有界廠商探測。
只記錄產生端或廠商版本變更時，必須重新建立的事實。
任務時序、事件逐字紀錄與憑證中繼資料，保留於私人報告或 PR 證據中。

Firstmate 會閱讀下列證據並公開推理過程，判定候選項目的提供者系列、憑證介面與適用配額。
[工作 Agent 輔助程式](../../bin/fm-quota-choose.sh) 與[具型別的解析器](../configuration.md#typed-dispatch-resolution-env-fm_dispatch_provider-fm_dispatch_model) 說明其確定性對應的邊界；其餘模型目錄與憑證判斷由[資格判定程序](../../.agents/skills/quota-array-dispatch/SKILL.md#1-eligibility) 定義。
以下憑證路徑將主目錄替換為 `<home>`。

<a id="quota-granularity-the-judgment-depends-on"></a>
## 判斷所依賴的配額粒度

以下提供者與模型範圍的關係，於 2026-07-30 以 quota-axi 0.1.16 驗證。
該版本擷取的預設輸出包含 `quotaSemantics.description`；schema 5 的預設 TOON 與 JSON 備援欄位位置，則在下一節以 0.1.29 驗證。
目前派工會讀取 TOON 的範圍與 `limitedBy` 欄位；JSON 備援對應的 `scope` 與 `boundedBy` 欄位，保留相同的提供者／模型適用關係，不依賴僅限 `--full` 的說明欄位。

```json
{
  "provider": "codex",
  "state": { "status": "fresh", "stale": false },
  "quotaSemantics": {
    "status": "known",
    "description": "Codex base account windows bound every model. Named model windows add bounds for that model; code-review windows describe a separate workload and are not included in model availability.",
    "effectiveAvailability": [
      { "scope": "all_models", "status": "known", "effectivePercentRemaining": 64, "boundedBy": ["weekly"] },
      { "scope": "model:codex_bengalfox", "status": "known", "effectivePercentRemaining": 64, "boundedBy": ["weekly", "model:codex_bengalfox:7d"] }
    ]
  }
}
```

帳號與範圍適用關係由[資格判定程序](../../.agents/skills/quota-array-dispatch/SKILL.md#1-eligibility) 定義；這份擷取結果說明這些範圍限制：

- 擷取到的 Codex 帳號回報 `all_models` 的限制為 64%，即使模型沒有自己的視窗也適用。
- 範圍為 `model:` 的項目，是針對該單一模型的額外限制。`model:codex_bengalfox` 是 GPT-5.3-Codex-Spark 的視窗，不會限制其他模型。
- 具名模型視窗可能比帳號限制更嚴格，因此不可跨模型套用。同一份快照中，Claude 的 `all_models` 回報 `effectivePercentRemaining` 為 10，而 `model:fable` 回報 4，受到 `model:fable` 視窗本身限制。非 Fable 的 Claude 模型應讀取 10，而不是 4。

廠商未提供視窗的提供者，其 `quotaSemantics.status` 會是 `unknown`，完全沒有 `effectiveAvailability` 項目（已在 `cursor` 與 `copilot` 觀察到）。
`state.authStatus` 只在部分提供者出現（目前只在 `grok` 觀察到），因此缺少該欄位表示缺少證據，並非憑證故障。

<a id="completion-runway-and-selection-shape-the-judgment-depends-on"></a>
## 判斷所依賴的完成餘裕與選擇結構

於 2026-08-18 以 quota-axi 0.1.29 的 schema 5 驗證，擷取自隔離安裝的 `quota-axi@0.1.29`。
預設 TOON 顯示以下表格標頭，列數已正規化為 `N`：

```text
quota[N]{provider,scope,effectivePercentRemaining,spendPriority,runway,confidence,limitedBy,resetsAt}:
exhaustion[N]{provider,scope,usableRunwaySeconds,projectedExhaustedAt,limitingWindowId}:
attention[N]{provider,scope,kind,detail,remedy}:
```

`exhaustion[]` 與 `attention[]` 是稀疏表格，因此空表會顯示數量零，且不含列欄位。
下列指令會記錄 JSON 備援結構，不會持久保存特定帳號的配額值：

```sh
quota-axi --json | jq '{schemaVersion, effectiveAvailabilityFields: ([.providers[]?.quotaSemantics.effectiveAvailability[]? | keys] | unique), runwayFields: ([.providers[]?.quotaSemantics.effectiveAvailability[]?.runway? | select(type == "object") | keys] | unique), selectionFields: ([.providers[]?.quotaSemantics.effectiveAvailability[]?.selection? | select(type == "object") | keys] | unique), paceFields: ([.providers[]?.quotaSemantics.effectiveAvailability[]?.pace? | select(type == "object") | keys] | unique), windowPaceFields: ([.providers[]?.windows[]?.pace? | select(type == "object") | keys] | unique)}'
```

```json
{
  "schemaVersion": 5,
  "effectiveAvailabilityFields": [
    [
      "boundedBy",
      "effectivePercentRemaining",
      "limitingWindowIds",
      "pace",
      "runway",
      "scope",
      "selection",
      "status"
    ]
  ],
  "runwayFields": [
    [
      "projectionConfidence",
      "status"
    ]
  ],
  "selectionFields": [
    [
      "spendPriority",
      "status"
    ]
  ],
  "paceFields": [
    [
      "status",
      "worstReservePercentPoints",
      "worstReserveWindowId"
    ]
  ],
  "windowPaceFields": [
    [
      "burnMultiple",
      "reservePercentPoints",
      "status"
    ]
  ]
}
```

這份實際快照全部為 `through_reset`，因此省略有限餘裕時間欄位。
當 `runway.status` 為 `projected_exhaustion` 或 `exhausted_now` 時，預設 `--json` 仍包含 `usableRunwaySeconds`、`projectedExhaustedAt` 與 `limitingWindowId`。
同樣地，`selection.unmeasurableWindowIds`、範圍的 `aheadWindowIds`／`unknownWindowIds`，以及視窗的 `pace.reason`，在適用時也會保留於預設 `--json`。
`quotaSemantics.description`、`behindWindowIds`、`onPaceWindowIds` 與各視窗週期進度的內部欄位，僅限 `--full`。
沒有 `projectionBasis` 欄位；缺少此欄位代表 `cycle_average`。
`runway` 與 `selection` 巢狀存放於各有效可用範圍下，因此相同的提供者／模型適用規則，同時約束剩餘額度、餘裕時間與 `spendPriority`。
不是每個已知餘裕時間都包含預測信心值，因此選擇時必須將這種缺漏保留為不確定性，不可自行補造。
結構相容性與帳號比對契約由 [`quota-array-dispatch`](../../.agents/skills/quota-array-dispatch/SKILL.md#1-eligibility) 定義；這份 schema 5 證據不會重新詮釋缺少的 runway、pace 或 selection 欄位。

<a id="provider-family-counterfactual-that-this-producer-schema-supports"></a>
## 此產生端結構所支持的提供者系列反事實檢查

於 2026-07-30 以 Pi 0.82.0 與 quota-axi 0.1.16 驗證。

```sh
pi --list-models terra
```

```text
provider      model          context  max-out  thinking  images
openai-codex  gpt-5.6-terra  272K     128K     yes       yes
```

Pi 模型目錄是判定 Pi 模型支援狀態的權威來源，並在獨立欄位回報提供者系列。
這份擷取結果中，目錄列出 `openai-codex/gpt-5.6-terra`，上述 Codex 列則回報 `all_models` 剩餘 64%。
快照沒有 Terra 專屬視窗，且 `quota-axi auth --json` 沒有列出 `pi:openai-codex` 來源。
這兩個缺漏分別代表缺少模型層級與來源層級的細節，並非互相矛盾的證據，因此揭露模型層級不確定性後，仍可派工給此候選項目。

```sh
pi --list-models gpt-9.9-nonexistent
```

```text
No models matching "gpt-9.9-nonexistent"
```

能存取帳號但未回傳任何列的查詢結果，才是會阻擋候選項目的權威否定證據。

<a id="credential-sources-are-independent-per-provider"></a>
## 各提供者的憑證來源彼此獨立

於 2026-07-30 以 quota-axi 0.1.16 驗證。

`quota-axi auth --json` 會分別回報各提供者的憑證來源，讓候選項目可被限定到它實際用來驗證的單一介面：

```json
[
  { "provider": "claude", "sources": [
      { "source": "oauth-file", "path": "<home>/.claude/.credentials.json", "status": "missing" },
      { "source": "keychain", "status": "available" } ] },
  { "provider": "codex", "sources": [
      { "source": "auth-json", "path": "<home>/.codex/auth.json", "status": "available" },
      { "source": "cli-rpc", "path": "<path-to>/codex", "status": "available" } ] },
  { "provider": "grok", "sources": [
      { "source": "auth-json", "path": "<home>/.grok/auth.json", "status": "available" },
      { "source": "pi:xai", "status": "available" } ] },
  { "provider": "kimi", "sources": [
      { "source": "pi:kimi-coding", "status": "available" },
      { "source": "kimi-code-cli", "status": "expired", "error": "kimi_code_cli_credential_expired" } ] }
]
```

觀察到的來源狀態包括 `available`、`expired`（附帶 `error` 短代碼）與 `missing`。

- 同一提供者可能同時有正常來源，以及缺少或過期的來源，因此不可將提供者壓成單一狀態。Claude 的 `oauth-file` 缺少，但鑰匙圈來源可用；Kimi 的獨立 CLI 憑證已過期，但 Pi 來源可用。
- 在這份擷取的設定中，只有 `pi:xai` 與 `pi:kimi-coding` 具有 `pi:` 前綴來源。
  Pi 的 `openai-codex` 候選項目使用上述 Codex 儲存區；這個觀察結果無法證實其他帳號或設定使用的憑證來源。
  缺少驗證證據如何影響派工，由[資格判定程序](../../.agents/skills/quota-array-dispatch/SKILL.md#1-eligibility) 定義。

quota-axi 0.1.16 之前，既沒有這種依來源區分的結構，也沒有 `state.authStatus`。
`bin/fm-bootstrap.sh` 透過 `bin/fm-quota-axi-lib.sh` 強制執行目前的最低相容版本要求。

Grok 在正常帳號上也會同時回報 `credits.remaining: 0` 與 `percentRemaining: 41`。
該零值是預付餘額，不是訂閱視窗，任何時候都不可視為剩餘額度。

<a id="standalone-grok-discovery-probe"></a>
## 獨立 Grok 探索探測

於 2026-07-30 以 `grok 0.2.117 (f1c06093089f) [stable]` 驗證。

```sh
grok --version
grok models   # stdin closed, single attempt, hard-bounded
```

觀察結果：

- 已驗證的工作階段執行 `grok models` 時，會以 `0` 結束，stdout 第一行為 `You are logged in with grok.com.`。
- 若主目錄沒有 Grok 憑證，stdout 第一行為 `You are not authenticated.`，結束狀態同樣是 `0`。
- 兩種情況的狀態都是 `0`，因此結束狀態不能作為判定；只檢查 stdout 第一行的精確文字，第一行空白不代表已驗證。
- 在已驗證的執行前後，`<home>/.grok/auth.json` 的位元組完全相同（`mtime`、`size` 與 `0600` 模式均未變），因此這條路徑上的探測是讀取操作。

這些辨識字串是 Firstmate 無法控制的廠商介面文字。
`bin/fm-vendor-auth-probe.sh` 會固定已驗證版本；執行中的 CLI 版本不同時回報 `versionVerified=no`，並將任何無法辨識的第一行歸類為 `indeterminate`，而不是已驗證。
廠商 CLI 變更時，請重新執行上面兩個指令，並同步更新本節與固定版本。

<a id="regression-coverage"></a>
## 回歸測試涵蓋範圍

`tests/fm-vendor-auth-probe.test.sh` 會以假廠商 CLI 驅動真正的指令碼，記錄每次呼叫的 argv 與 stdin 上所有可讀取內容。
它會斷言指令碼不接受執行環境、模型或提供者輸入，絕不呼叫 `quota-axi`，因為不產生判定所以所有探測結果都以相同狀態結束，只會使用兩種固定且非破壞性的 argv 格式並關閉 stdin，即使設定的界限為零或格式錯誤也會維持真正的界限，而且絕不回顯原始廠商輸出。
`tests/fm-spawn-dispatch-profile.test.sh` 負責 spawn 的確定性設定檔與執行環境拒絕案例。
`tests/fm-bootstrap.test.sh` 負責 quota-axi 最低版本的診斷。
`tests/fm-quota-array-dispatch-live-e2e.test.sh` 會透過公開的 Pi 技能載入介面，針對每個案例提供一份假的 schema 5 快照，格式為 quota-axi 的預設 TOON。
它涵蓋通過資格、推理等級與餘裕可行性閘門之候選項目的 TOON 優先 `spendPriority` 排序、明確處理無法衡量的餘裕時間、最強推理限制，以及餘裕可行性下限優先於較高 `spendPriority` 的規則。
`tests/fm-dispatch-resolve.test.sh`、`tests/fm-quota-choose.test.sh` 與 `tests/fm-procevent-quota.test.sh`，透過公開指令碼介面涵蓋 schema 6 的帳號列綁定、帳號隔離與 schema 5 相容性。
技能的主要路徑就是這個預設 TOON；`--json` 是文件所述的防禦性備援，而本節記錄該備援所使用的產生端 `--json` 結構。
