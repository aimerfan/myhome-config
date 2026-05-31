# Windows Terminal:Shift+Enter 換行

## 目標

在 Claude Code 的輸入框,讓 `Shift+Enter` 插入換行而非送出訊息(預設 `Enter` 直接送出)。

## 環境

- Windows 11 + Windows Terminal(`WT_SESSION` 有值)
- shell:Windows PowerShell 5.1

## 關鍵發現

- Claude Code 內建的 `chat:newline` 動作預設綁在 `Ctrl+J`;另外輸入 `\` 後按 `Enter` 也可換行。兩者都與終端機無關,一律可用(`Ctrl+J` 送的是 LF `0x0A`)。
- 直接在 Claude Code 的 `keybindings.json` 把 `shift+enter` 綁到 `chat:newline` 無效。原因:多數終端機(含此環境的 Windows Terminal)下,`Shift+Enter` 送進來的位元組與 `Enter` 相同(都是 CR `0x0D`),Claude Code 收不到獨立的 shift+enter 事件,綁定永遠不觸發。
- `/terminal-setup` 宣稱 Windows Terminal「原生支援 Shift+Enter、無需設定」,但本機實測失效(win32-input-mode 未協商成功)。不可依賴此宣稱。

## 解法

在 Windows Terminal 自身的 `settings.json` 加一條 `sendInput` 動作,讓 `Shift+Enter` 直接送出 LF(`\n`)。由於 `Ctrl+J`(同樣是 LF)已能觸發 `chat:newline`,此法等效且必定生效,完全繞過終端機按鍵序列協商問題。

設定檔位置:

```
C:\Users\<user>\AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\settings.json
```

在 `actions` 陣列加入動作定義:

```json
{
    "command": { "action": "sendInput", "input": "\n" },
    "id": "User.sendNewline"
}
```

在 `keybindings` 陣列加入按鍵對應:

```json
{
    "id": "User.sendNewline",
    "keys": "shift+enter"
}
```

(此為新版 Windows Terminal 的拆分格式:`actions` 放動作定義並帶 `id`,`keybindings` 把按鍵對應到 `id`。)

## 影響範圍

- 之後在此 Windows Terminal 的所有 profile 內按 `Shift+Enter` 都會送出 LF。對多數 CLI 而言 LF 等同 Enter,無副作用;對 Claude Code 則成為換行。
- Windows Terminal 偵測到 `settings.json` 變更會自動重載,通常不需重啟。
- `Enter` 維持送出;`Ctrl+J`、`\`+`Enter` 仍可用。

## 後續清理

由於改用 Windows Terminal 端的 `sendInput`,Claude Code 收到的是 LF(等同 `Ctrl+J`),不會再是 shift+enter 事件,因此 `~/.claude/keybindings.json` 內針對 `shift+enter` 的綁定無作用,已移除。
