---
name: harness-adapters
description: >-
  啟動或復原船員／二副、處理信任對話框、傳送執行環境專屬技能呼叫、中斷或結束 Agent、恢復已結束 Agent，或驗證新執行環境介接器前載入。
user-invocable: false
metadata:
  internal: true
---

# harness-adapters

本技能是執行環境專屬 Firstmate 操作的唯一技能、觸發與路由負責者。
先載入本路由器，再精確載入下方選定的共用參考與一份執行環境參考。
動作跨越多列時，載入所需參考的聯集一次即可，不要載入所有參考。
`references/` 下的檔案是本技能資源，不是另行列入目錄的技能。

<a id="path-contract"></a>
## 路徑契約

技能目錄就是包含此 `SKILL.md` 的目錄。
按需參考連結，以及指向負責執行檔、文件或同層技能的相對連結，都以技能目錄解析，包括巢狀參考內指定的連結。
操作路徑維持負責元件指定的上下文：`config/` 與目前 home 設定屬於目前 Firstmate home，`state/` 屬該 home，`.claude/settings.json` 等專案設定則屬目標專案。

<a id="non-negotiable-safety"></a>
## 不可妥協的安全規則

絕不使用未驗證介接器派發船員或二副。
`config/crew-harness` 或 `config/secondmate-harness` 若指定未驗證介接器，依 `../../../AGENTS.md` 第 9 節告訴船長所要求的工作 Agent 執行環境尚未驗證，目前工作改用 firstmate 自身已驗證的執行環境，只詢問是否為未來工作驗證所要求的執行環境。
不要為了這項選擇暫停目前工作。

結果為 `unknown` 時，詢問船長，不要猜測。
當前船長覆寫優先於偵測；單次工作覆寫只約束該次派工。
復原與控制使用 `state/<id>.meta` 中精確的 `harness=`；絕不從模型或供應商推測。

生命週期動作只透過 `../../../bin/fm-control.sh <task-id> interrupt|exit|relaunch` 傳遞。
絕不透過 `fm-send` 輸入中斷按鍵或結束指令，因為帶路由標記的生命週期文字會變成對話。
只有檢查證明目標已開始處理指示，信任處理才算完成；傳遞成功本身不是證明。
Muse、Gemini、Devin、Command Code、Polytoken、Cortex、Kiro 與 Hermes 只驗證用於船員與 scout，不能作為二副或主要執行環境。

<a id="detection"></a>
## 偵測

`../../../bin/fm-harness.sh` 從已驗證環境標記與程序祖先鏈印出 firstmate 自身執行環境，並定義兩者如何組合。
標記指出其執行環境，但不同執行環境的結構性祖先優先，因為標記只是子程序或多工器可能保留的一般環境狀態，祖先鏈才能證明誰擁有程序樹。
只有啟動邊界的 `FM_PI_HARNESS=pi-signed` 搭配 `PI_CODING_AGENT=true`，才選擇 Pi-signed；共用且未標記的啟動器祖先鏈仍視為 Pi。
omp 不發布自己的標記；`FM_OMP_HARNESS=omp` 是 Firstmate 的啟動標記，`bin/fm-omp-process-lib.sh` 中錨定的身分（程序名稱 `omp`，或由 `bun` 執行 omp 啟動器）是其祖先鏈證據，記錄於 `references/harness/omp.md`。
工作 Agent 標記建立由 `../../../bin/fm-spawn.sh` 負責，簽章主要執行環境的邊界則以 README 啟動指令為準。
`../../../bin/fm-harness.sh crew` 解析 `config/crew-harness`；缺少或 `default` 代表 firstmate 自身執行環境。
二副固定選擇的解析，包括個別二副覆寫，以 [`docs/configuration.md`](../../../docs/configuration.md#choose-the-secondmate-harness) 為準；解析指令由 [`bin/fm-harness.sh`](../../../bin/fm-harness.sh) 定義。
`../../../bin/fm-spawn.sh` 每次啟動都重新解析，明確的單次啟動參數對該次啟動優先。
新介接器經驗證的標記與指令名稱，必須合併到 `../../../bin/fm-harness.sh`。

<a id="operation-to-reference-matrix"></a>
## 操作與參考對照矩陣

每份輸出計畫，都在具名共用參考後面追加選定或已記錄的執行環境參考。
`harness-adapter-routing-v1` 物件是機器可讀、人類可見的選擇契約：先選操作，再選其中情境，最後追加選定執行環境參考。
沒有更精確情境適用時，使用一般情境 `default`。
Kimi 在其執行環境參考中定義不支援主要執行環境的邊界；Muse 與 Gemini 遵循上方不可妥協安全規則。
新工具必須等 `verify` 計畫、執行環境項目、每個具名負責元件及真實環境檢查都合併後，才能派工。

```json harness-adapter-routing-v1
{
  "operations": {
    "start": {
      "default": ["references/common/dispatch.md", "references/common/model-and-effort.md"],
      "trust-dialog": ["references/common/control-and-recovery.md"]
    },
    "trust": {"default": ["references/common/control-and-recovery.md"]},
    "skill": {"default": ["references/common/control-and-recovery.md"]},
    "interrupt": {"default": ["references/common/control-and-recovery.md"]},
    "exit": {"default": ["references/common/control-and-recovery.md"]},
    "resume": {"default": ["references/common/control-and-recovery.md"]},
    "recovery": {
      "default": ["references/common/control-and-recovery.md"],
      "replacement-profile": ["references/common/control-and-recovery.md", "references/common/dispatch.md", "references/common/model-and-effort.md"],
      "secondmate": ["references/common/control-and-recovery.md", "references/common/primary-hooks.md"],
      "replacement-secondmate": ["references/common/control-and-recovery.md", "references/common/dispatch.md", "references/common/model-and-effort.md", "references/common/primary-hooks.md"]
    },
    "primary": {"default": ["references/common/primary-hooks.md"]},
    "model-effort": {
      "default": ["references/common/model-and-effort.md"],
      "configured-profile": ["references/common/model-and-effort.md", "references/common/dispatch.md"]
    },
    "verify": {"default": ["references/common/dispatch.md", "references/common/control-and-recovery.md", "references/common/primary-hooks.md", "references/common/model-and-effort.md"]}
  },
  "harnesses": {
    "claude": "references/harness/claude.md",
    "codex": "references/harness/codex.md",
    "opencode": "references/harness/opencode.md",
    "pi": "references/harness/pi.md",
    "pi-signed": "references/harness/pi.md",
    "grok": "references/harness/grok.md",
    "kimi": "references/harness/kimi.md",
    "cursor": "references/harness/cursor.md",
    "gemini": "references/harness/gemini.md",
    "muse": "references/harness/muse.md",
    "rovo": "references/harness/rovo.md",
    "omp": "references/harness/omp.md",
    "agy": "references/harness/agy.md",
    "devin": "references/harness/devin.md",
    "commandcode": "references/harness/commandcode.md",
    "polytoken": "references/harness/polytoken.md",
    "cortex": "references/harness/cortex.md",
    "kiro": "references/harness/kiro.md",
    "hermes": "references/harness/hermes.md"
  }
}
```
