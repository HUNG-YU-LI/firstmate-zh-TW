<a id="local-shellcheck-option-a-measurement"></a>
# 本機 ShellCheck 方案 A 量測

2026-09-05 的 lint 成本稽核，在提交 `f09de8a3d3a550b13b4d535346fbc7b9ac0d6c19` 上，量測了漏回覆事件涉及的七個分析入口：

```text
bin/fm-brief.sh
bin/fm-parent-channel-lib.sh
bin/fm-pending-reply-lib.sh
bin/fm-secondmate-report.sh
tests/fm-brief.test.sh
tests/fm-classify-corr-token.test.sh
tests/fm-pending-reply.test.sh
```

ShellCheck 使用儲存庫固定的 0.11.0 Darwin arm64 建置版本。
基準方案是一次呼叫，以可追蹤 source 相依檔案的方式納入全部七個入口。
方案 A 則每個入口各用一個程序，省略 `--external-sources`、保留延伸資料流程分析，並套用本機的跨檔案排除清單。
兩種方案都在相同的主機低負載時段量測：

| 方案 | 使用者 + 系統 CPU 時間 | 降幅 | 單一程序最高 RSS | 降幅 |
| --- | ---: | ---: | ---: | ---: |
| 追蹤 source 相依檔案的基準方案 | 140.1 s | 不適用 | 8.30 GB | 不適用 |
| 方案 A，每個入口各用一個程序 | 9.8 s | 93.0% | 0.56 GB | 93.3% |

<a id="reproduction"></a>
## 重現方式

簽出紀錄中的提交，使用 `bin/fm-install-shellcheck.sh` 安裝固定版本的執行檔，將它放到 `PATH` 的最前面，再於 macOS 執行下列指令。
指令未包含 `--extended-analysis=false`，因此資料流程分析仍保持開啟。
由於這次只量測程序成本，診斷輸出會被捨棄。

```bash
set -eu
[ "$(bin/fm-lint.sh --required-version)" = "$(shellcheck --version | awk '/^version:/ {print $2; exit}')" ]
roots=(
  bin/fm-brief.sh
  bin/fm-parent-channel-lib.sh
  bin/fm-pending-reply-lib.sh
  bin/fm-secondmate-report.sh
  tests/fm-brief.test.sh
  tests/fm-classify-corr-token.test.sh
  tests/fm-pending-reply.test.sh
)
rm -rf .lint-option-a-measurement
mkdir .lint-option-a-measurement
/usr/bin/time -lp -o .lint-option-a-measurement/baseline.time \
  shellcheck --norc --external-sources -- "${roots[@]}" >/dev/null || true
index=0
for root in "${roots[@]}"; do
  index=$((index + 1))
  /usr/bin/time -lp -o ".lint-option-a-measurement/option-a.$index.time" \
    shellcheck --norc --exclude=SC1091,SC2034,SC2153,SC2329 -- "$root" \
    >/dev/null || true
done
awk '
  /^user / {cpu += $2}
  /^sys / {cpu += $2}
  /maximum resident set size/ {if ($1 > rss) rss=$1}
  /bytes allocated/ {allocated += $1}
  END {printf "cpu_seconds=%.2f worst_rss_bytes=%.0f bytes_allocated=%.0f\n", cpu, rss, allocated}
' .lint-option-a-measurement/baseline.time
awk '
  /^user / {cpu += $2}
  /^sys / {cpu += $2}
  /maximum resident set size/ {if ($1 > rss) rss=$1}
  /bytes allocated/ {allocated += $1}
  END {printf "cpu_seconds=%.2f worst_rss_bytes=%.0f bytes_allocated=%.0f\n", cpu, rss, allocated}
' .lint-option-a-measurement/option-a.*.time
```

CPU 時間與 RSS 會隨主機負載而改變，因此宣稱百分比降幅時，必須比較同一個量測時段內的執行結果。
若必須跨時段比較結果，請使用回報的 `bytes_allocated` 總量作為穩定的工作量指標，不要引用 CPU 時間或 RSS 的比值。
