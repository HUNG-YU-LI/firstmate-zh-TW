# Hermes Agent

Hermes 是只支援 tmux、供啟動船員／scout 使用的 TUI 介接器。
主要工作階段與二副監督、其他執行階段後端、桌面／網頁介面及訊息橋接，都不支援且會拒絕。
附日期的供應商證據與真實防護測試，以 [Hermes 驗證](../../../../../docs/verification/hermes.md) 為準。

<a id="operating-facts"></a>
## 操作事實

| 項目 | 值 |
|---|---|
| 啟動與設定檔 | 互動式 `hermes` CLI TUI，使用原生逐次啟動設定檔選擇；旗標、省略與重新啟動機制，以 [`fm-spawn.sh`](../../../../../bin/fm-spawn.sh) 標頭為準。 |
| 模型與推理強度 | 兩個軸都由選定設定檔負責。Firstmate 拒絕非預設模型覆寫，強度則記錄但省略。原生設定檔探索為 `hermes -p <profile> config get model --json` 與 `config get agent.reasoning_effort --json`；`/model` 是原生互動選擇介面。不從模型名稱推測供應商或目錄成員資格。 |
| 忙碌與回合結束 | 原生 `pre_llm_call` 與 `on_session_end`，透過 [`fm-hermes-hook.sh`](../../../../../bin/fm-hermes-hook.sh) 與綁定世代的忙碌寫入器處理；只接受原生 `tui` 平台，turn ID 讓關閉能跨越 session ID 輪替綁定。 |
| Hook 載體 | 由 [`fm-hermes-lib.sh`](../../../../../bin/fm-hermes-lib.sh) 暫存的私人原生 managed-scope 覆寫層；保留既有設定檔 hook，拒絕既有 managed policy，不編輯任何設定檔 SOUL 或設定。 |
| 自主權與 hook | 啟動授予原生工具自主權與原生 Shell hook 接受權，包括既有已設定 hook。啟動防護只確認完整可見、且已選 `No thanks` 的遙測詢問；原生 Hermes 記錄拒絕。其他首次設定仍需要預先準備的設定檔。 |
| 中斷 | 連按兩次 Escape，但只在已證明輸入區為空時使用，因為相同按鍵也會捨棄草稿；閒置時雙 Escape 保留工作 Agent。不用 Ctrl+C，因為它可能結束閒置 TUI。 |
| 結束與重新啟動 | `/quit` 返回窗格 Shell。重新啟動依儲存的工作簡報、已記錄設定檔與 home 開始，不連接 gateway，也不恢復不相關原生工作階段。 |
| 身分 | 啟動邊界的 `FM_HERMES_HARNESS=hermes` 加上結構性 Hermes 祖先鏈；錨定的 Node TUI／Python 模組程序身分，以 [`fm-hermes-process-lib.sh`](../../../../../bin/fm-hermes-process-lib.sh) 為準。 |
| 輸入區 | 狀態分隔線下方的原生 `❯`，帶設定檔前綴，或預設設定檔的無前綴提示；tmux 中顯示 `❯` 的窗格，會在通用游標讀取前先檢查 Hermes 身分，因此 Hermes 窗格只由 Hermes 分類器判定；合成忙碌提示游標與亮色草稿，由 [`fm-composer-lib.sh`](../../../../../bin/fm-composer-lib.sh) 區分。無法識別的佈景、換行、對話框或無法讀取的身分，會延後傳遞。 |
| 技能 | 使用一般文字指向已安裝技能的 `SKILL.md`；原生斜線技能呼叫尚未驗證。 |

<a id="profile-safety"></a>
## 設定檔安全

選定設定檔自身正在執行的 gateway，只有閒置時才接受：其原生 `gateway_state.json` 必須指出存活的 gateway 程序、回報 `gateway_state: running` 且未要求重新啟動、`active_agents: 0` 且沒有活躍工作，心跳也不能超過 Hermes 自己的 120 秒過期界線。
回合中、啟動中、排空中或過渡中的 gateway 都拒絕；沒有該閒置狀態卻存在活躍 gateway、持有的原生執行階段鎖或活躍 systemd 單元，也拒絕；任何無法讀取或格式錯誤的 gateway 狀態亦同。
活躍的 default-root gateway，或從另一個服務該設定檔的 home 啟動之多工器，也會拒絕，因為無法把其活動歸屬到此設定檔。
既有 managed scope 會在暫存 hook 前遭拒絕。
Firstmate 不會停止、重新啟動或傳訊給 gateway。
閒置共存證據與限制，以[驗證紀錄](../../../../../docs/verification/hermes.md) 為準。
這是啟動預檢，不是整個生命週期的預約：派工後 gateway 可能開始回合，而同一設定檔的重疊回合尚未驗證。

<a id="limits"></a>
## 限制

已驗證版本中，TUI 的 `--query-file` 路徑未送出初始提示，因此啟動改用原生互動 query 路徑。
早期供應商／建置錯誤若未執行原生結束 hook，可能讓啟動元件持有的 busy 紀錄保持開啟；請檢查可見窗格並走一般復原程序。
使用者設定檔設定、外掛、工具與推論憑證，仍由原生 Hermes 負責。
尚未驗證主要 Stop 防護、工具執行前保護、監看協定、二副 hook 或 Herdr 生命週期。
