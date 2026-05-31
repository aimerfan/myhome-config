# Claude Code 自訂 statusline

## 目標

內建 `/statusline` 能顯示的項目有限,改用自訂腳本完全接管狀態列內容。

## 機制

`/statusline` 只是精靈;真正的機制是 `settings.json` 的 `statusLine` 設定:

- 設 `type: "command"`,`command` 指向任意可執行指令。
- 每次狀態列要更新時,Claude Code 執行該指令,並透過 stdin 餵一段 session JSON。
- 指令印到 stdout 的第一行即為狀態列內容(支援 ANSI 顏色)。

更新時機:收到新 assistant 訊息、`/compact` 完成、權限模式變更、vim 模式切換時觸發,並有 300ms debounce。可選 `refreshInterval`(秒,最小 1)額外加上固定計時刷新,適用於時間相關顯示。本實作不需要,維持事件驅動。

## stdin 可用欄位(摘要)

- 基本:`cwd`、`session_id`、`session_name`、`transcript_path`、`version`
- 模型/模式:`model.id`、`model.display_name`、`output_style.name`、`effort.level`、`thinking.enabled`、`vim.mode`
- 工作區/git:`workspace.current_dir`、`workspace.project_dir`、`workspace.git_worktree`、`workspace.repo.{host,owner,name}`、`worktree.*`
- 成本:`cost.total_cost_usd`、`cost.total_duration_ms`、`cost.total_api_duration_ms`、`cost.total_lines_added/removed`
- context:`context_window.{used_percentage, remaining_percentage, context_window_size, total_input_tokens, total_output_tokens, current_usage.*}`、`exceeds_200k_tokens`
- 額度:`rate_limits.{five_hour,seven_day}.{used_percentage, resets_at}`(僅 Pro/Max 且首次 API 回應後才有)
- 協作:`agent.name`、`pr.{number,url,review_state}`

部分欄位可能缺席或為 `null`(例如首次 API 呼叫前 `context_window.used_percentage` 為 null),腳本需防呆。

## 本實作的版面

```
資料夾(branch) | model.id[ctxsize](effort) | <bar> pct% | 5h:xx%(@reset) | 7d:xx%
```

1. 資料夾名(由 `cwd` 取 basename)+ git branch(以 `git -C cwd branch --show-current` 取,非 git repo 則省略)。
2. `model.id`(砍掉 `claude-` 前綴,再砍掉結尾的 `[...]` 變體後綴)+ context 視窗大小(`200k` / `1M`)+ effort 等級(缺席則省略)。`[1m]` 是 1M context 變體的 model id 後綴(`claude-opus-4-8[1m]`),語意與我們自附的 context size 重疊,故移除以由 size 段單獨呈現。
3. context 已用百分比 + 10 格 block bar;null 時顯示 `ctx --`。
4. 5 小時額度用量 + reset 時間(epoch 轉本地 HH:MM)。
5. 7 天額度用量。rate_limits 缺席時顯示 `--`。

顏色(256 色,門檻式):

- 資料夾 cyan 粗體、branch 暗黃。
- model 藍、context size 暗紫(`38;5;97`)、effort 依等級(high 洋紅 / medium 藍 / low 暗)。
- 百分比類(ctx / 5h / 7d):<50% 綠、50–80% 黃、>80% 紅。
- 分隔線與 reset 時間暗灰。

## 腳本語言

選用 Python 3.12(本機 `C:/Users/<user>/AppData/Local/Programs/Python/Python312/python.exe`)。JSON 解析內建、缺席欄位防呆乾淨。腳本檔:`~/.claude/statusline.py`。

## settings.json 設定

```json
"statusLine": {
  "type": "command",
  "command": "C:/Users/<user>/AppData/Local/Programs/Python/Python312/python.exe C:/Users/<user>/.claude/statusline.py"
}
```

## 關鍵陷阱(踩過)

1. **路徑必須用正斜線**。Claude Code 在 Windows 透過捆綁的 Git Bash / mingw `sh -c` 執行 statusLine 指令,不是 cmd.exe 或 PowerShell。bash 裡反斜線是 escape 字元,`C:\Users\...` 會被吃掉導致指令靜默失敗、狀態列空白。改用 `C:/Users/...`(Windows 原生 exe 接受正斜線)。
2. **stdout 編碼**。Windows 預設 stdout 為 locale codepage(此機為 cp950),無法輸出 block 字元(`█` `░`),會丟 `UnicodeEncodeError`。腳本開頭強制 `sys.stdout`/`sys.stdin` 用 UTF-8。
3. **設定載入時機**。`settings.json` 於啟動時讀取,新增 `statusLine` 後需重新觸發/重啟 Claude Code 才生效。
4. **workspace trust**。statusLine 執行 shell 指令,與 hooks 同樣需接受過當前目錄的信任對話框才會執行。

## 驗證方式

用 mock JSON 模擬 CC 的呼叫(注意要走 `sh -c` + 正斜線):

```bash
echo '{"cwd":"C:/Users/<user>/.claude","model":{"id":"claude-opus-4-8"},"context_window":{"context_window_size":200000,"used_percentage":63},"effort":{"level":"high"},"rate_limits":{"five_hour":{"used_percentage":23.5,"resets_at":1748600000},"seven_day":{"used_percentage":85.2}}}' \
  | sh -c "C:/Users/<user>/AppData/Local/Programs/Python/Python312/python.exe C:/Users/<user>/.claude/statusline.py"
```

也可測 null/缺席情境,確認降級為暗灰 `--` 而非報錯。
