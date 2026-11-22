# 台灣繁體中文翻譯說明

本儲存庫為 [kunchenguid/firstmate](https://github.com/kunchenguid/firstmate) 的繁體中文 fork，採台灣工程師常用語。

## 來源與範圍

- 上游基準提交：`427056c4181c41074ad24f051f79919330df0fa5`。
- 上游儲存庫與提交歷史完整保留。
- 翻譯範圍包括全部 Markdown 文件、Agent 技能與操作規範，以及設定範例中的自然語言。
- 程式碼、命令、CLI 旗標、設定鍵名、協定值、檔名與路徑維持可用；原英文章節錨點另行保留，讓既有連結繼續有效。

## 使用這份繁中副本

```bash
git clone https://github.com/HUNG-YU-LI/firstmate-zh-TW.git
cd firstmate-zh-TW
```

其餘環境需求與啟動方式請依 [README](README.md) 及相應執行環境文件。原有上游 URL 與命令範例保留，便於對照來源。

## 翻譯進度

目前為翻譯中的階段提交，完整文件仍在逐段處理與交叉檢查；本段會在全部完成後更新。

## 保留原文的機器契約

`CLAUDE.md` 是匯入 `AGENTS.md` 的機器入口，且上游會逐位元組比對它。原檔保持不變，完整繁中對照見 [CLAUDE.zh-TW.md](CLAUDE.zh-TW.md)。程式實際輸出、測試斷言、命令與設定識別字不改成中文，避免破壞原本行為。

原始 [LICENSE](LICENSE) 與著作權聲明完整保留。

## 用語

| 原文 | 本版用語 |
| --- | --- |
| repository | 儲存庫 |
| code / software | 程式碼／軟體 |
| file / directory | 檔案／目錄（資料夾） |
| configuration | 設定 |
| dependency | 相依套件或相依關係 |
| process / thread | 程序／執行緒 |
| commit / branch / merge | 提交／分支／合併 |
| context | 上下文 |
| agent / skill / harness | Agent（代理程式）／技能／執行環境 |
| captain / firstmate / secondmate | 船長／大副／二副 |

模型名、產品名、工具名與有機器語意的角色字串保留原文。
