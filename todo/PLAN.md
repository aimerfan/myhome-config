# myhome-config 重構計畫

## 目標

把這個 repo 從「扁平 dotfiles、直接 pull 進家目錄」改成「集中儲存、選擇性套用」的結構,讓:

- repo 維持單一完整來源(single source of truth),不打散成多個 gist。
- 每台機器只生效它需要的設定,選擇性由安裝階段控制,而非靠拆分儲存。
- 跨 Windows 與 Linux 通用。
- 把 user-level `CLAUDE.md` 納入管理,視為 home 層級的通用設定。

## 現狀與問題

目前 repo 的 working tree 直接疊在家目錄上,`git pull` 一次把所有檔案 materialize 到 `~`。限制:

- 沒有選擇性:repo 有什麼,該機器就被鋪上什麼。
- `.git` 與真實 home 檔案混雜。
- 無法表達「這台只要 gitconfig,不要 vimrc」。

## 設計決策

- **儲存位置與生效位置分離**:repo 放獨立目錄(如 `~/.dotfiles` 或 `D:\repo\myhome-config`),只當倉庫;透過連結讓設定在目標路徑生效。
- **連結方式**:symlink 為主(改 repo 即時生效、`git status` 直接可見)。建立失敗時(Windows 未開 Developer Mode 或非系統管理員)fallback 成 copy,並印出明確訊息,不靜默 fallback。
- **選擇性套用**:manifest(來源→目標對應表)+ profile(tag)。install script 依 profile 過濾,只處理被選中的條目。

## 目標結構

```
myhome-config/
  install.ps1        # Windows 安裝腳本
  install.sh         # Linux 安裝腳本
  manifest           # source | target | profiles 對應表
  home/              # 對到 ~ 的檔案
    .gitconfig
    .vimrc
    .gitignore_global
  claude/
    CLAUDE.md        # 對到 ~/.claude/CLAUDE.md
  PLAN.md
```

## manifest 格式

無依賴純文字,`|` 分隔,PowerShell 與 bash 皆可逐行 parse(不需 jq):

```
# source | target | profiles
home/.gitconfig         | ~/.gitconfig            | base
home/.vimrc             | ~/.vimrc                | base
home/.gitignore_global  | ~/.gitignore_global     | base
claude/CLAUDE.md        | ~/.claude/CLAUDE.md     | base,ai
```

- `~` 在各平台展開到對應 home(Windows `%USERPROFILE%` / Linux `$HOME`)。
- profiles 為 comma 分隔 tag。`install -Profile ai` 處理 tag 含 `ai` 或 `base` 的條目;`base` 視為一律套用。

## install 流程(兩支腳本同邏輯)

1. 讀 manifest,依傳入 profile 過濾。
2. 對每條目:展開 target 路徑 → 確保父目錄存在 → 若 target 已存在且非本 repo 連結,先備份為 `.bak`。
3. 嘗試建立 symlink;失敗則 copy,並 log `[fallback:copy] <target>`。
4. 結尾輸出每條結果:symlink / copy / skipped。

## 更新流程

- 舊流程(在家目錄 `git pull`)失效。
- 新流程:在 repo 目錄 `git pull`。
  - symlink 條目即時生效。
  - copy 條目需重跑 install。

## 待辦

- [ ] 確認 repo 設 public / private,並對 `CLAUDE.md` 做敏感資訊清理(含 email、個人工作流描述)。
- [ ] 建立 `home/` 並將現有 `.gitconfig`、`.vimrc` 移入(會改變現行用法)。
- [ ] 補上 `.gitignore_global`(若要納管)。
- [ ] 新增 `claude/CLAUDE.md`。
- [ ] 撰寫 `manifest`。
- [ ] 撰寫 `install.ps1` 與 `install.sh`,含 symlink → copy fallback 與備份邏輯。
- [ ] 在 Windows 與 Linux 各驗證一次安裝結果。

## 開放問題

- 機器群的 profile 切分(例:`base` / `work` / `ai`)實際要分幾類、各含哪些條目。
- 是否需要 uninstall / 還原備份的反向腳本。
