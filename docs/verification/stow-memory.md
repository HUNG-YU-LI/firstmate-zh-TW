<a id="startup-memory-stow-verification"></a>
# 啟動記憶 `/stow` 驗證

讀者：維護者驗證。

這份紀錄佐證一項現行保證：Firstmate 能探索並即時載入（JIT-load）使用者擁有、但透過該複本的 `.git/info/exclude` 排除的本機技能。
內部的 [`stow` 技能](../../.agents/skills/stow/SKILL.md) 負責分層、整理、封存、卸載，以及完成回執的行為。
[`docs/configuration.md`](../configuration.md) 負責目前面向操作者的啟動記憶設定與估算說明。

<a id="git-excluded-local-skill-discovery-and-loading"></a>
## 探索與載入被 Git 排除的本機技能

內部技能的卸載目的地，仰賴執行環境能探索並即時載入技能目錄，即使該路徑已列入複本的本機 `.git/info/exclude`。
這項檢查於 2026-08-08 使用 Claude Code 2.1.226，在可拋棄的暫存儲存庫中執行。
獨有的哨兵值只出現在 frontmatter 下方的技能本文，因此新工作階段必須載入被排除的技能，才能回傳該值；只看到已索引的名稱或描述並不足夠。

從此儲存庫根目錄執行的確切指令如下：

```bash
set -eu
claude --version
PROBE_ROOT="$PWD/.stow-excluded-probe-tmp"
rm -rf "$PROBE_ROOT"
mkdir -p "$PROBE_ROOT"
cd "$PROBE_ROOT"
git init -q .
mkdir -p .claude/skills/excluded-probe
cat >.claude/skills/excluded-probe/SKILL.md <<'EOF'
---
name: excluded-probe
description: A neutral probe used when explicitly requested by name.
---

# Excluded probe

The sentinel token is STOW-EXCLUDE-LOAD-8F3K1.
EOF
printf '.claude/skills/excluded-probe/\n' >>.git/info/exclude
git check-ignore -v .claude/skills/excluded-probe/SKILL.md
claude --model haiku --allowedTools Skill -p "Use your Skill tool to load the skill named 'excluded-probe', then reply with exactly the sentinel token stated inside its body and nothing else."
cd ..
rm -rf "$PROBE_ROOT"
```

實際觀察到的確切輸出如下：

```text
2.1.226 (Claude Code)
.git/info/exclude:7:.claude/skills/excluded-probe/	.claude/skills/excluded-probe/SKILL.md
STOW-EXCLUDE-LOAD-8F3K1
```

`git check-ignore` 那一行證明，本機排除規則涵蓋了技能本文；精確符合的哨兵值回覆則證明，全新的 Claude Code 工作階段透過 Skill 工具載入了該本文。
同一天，在此儲存庫本身的 `.agents/skills/` 下，一個被 `.gitignore` 忽略的探測目錄，也透過 `.claude/skills` 符號連結，被全新工作階段與受版本控制的對照技能一同列出。
直接使用本機排除規則的探測建立了核心保證；儲存庫內的探測則獨立佐證：被忽略的狀態不會抑制檔案系統探索。
