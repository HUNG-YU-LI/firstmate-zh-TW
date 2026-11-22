<a id="documentation-audiences"></a>
# 文件讀者分類

[`documentation-audiences.json`](documentation-audiences.json) 是所有持續維護的文字文件之分類權威來源，供程式讀取。
`bin/fm-doc-audience-check.sh` 會驗證清單是否完整涵蓋所有文件、README 的設定導引、必要的權責文件連結，以及本機連結目標。
讀者中繼資料集中放在該檔案，而非複製到每一頁的 front matter。

每種讀者分類各有明確的內容歸屬用途：

- `public-product`：介紹產品，或提供可獨立閱讀的公開資料。
- `operator-current`：說明目前的行為、設定、支援限制、穩定不變條件、精簡設計理由，以及目前的驗證入口。
- `operator-example`：可直接複製使用的現行設定範例。
- `maintainer-architecture`：向貢獻者說明穩定的權責劃分、擴充點、機制邊界及安全設計理由。
- `maintainer-verification`：記錄現行保證可重複驗證的證據，可包含日期、版本、確切指令及確切輸出。
- `agent-runtime`：作為 Firstmate Agent 的操作契約載入或呈現，而非供人閱讀的產品文件。

知識內容的歸屬政策由 [`firstmate-coding-guidelines`](../.agents/skills/firstmate-coding-guidelines/SKILL.md) 負責定義。
特定任務的時間序列、交付逐字紀錄、暫存路徑、分支、被推翻的假設，以及單次使用的程序識別碼，預設保留在私人任務報告或 PR 證據中。
從受版本控制的頁面移除這些證據之前，請先將每一項獨有且仍然有效的事實整理到對應分類的權責文件，並保留聚焦的回歸測試連結。

直接執行以下指令進行結構檢查：

```sh
bin/fm-doc-audience-check.sh
```

這項檢查刻意不對日期、版本、指令、路徑、事故敘述或類似逐字紀錄的文字做 lint。
這些形式在維護者驗證文件中都是合理內容，需要語意審查，而非關鍵字啟發式判斷。
對每一份變更的文字文件，請審查其讀者、權威權責來源、目前相關性、證據的存放位置，以及獨有的安全事實；完成所有修正後，再針對整個分支的差異重做一次審查。
