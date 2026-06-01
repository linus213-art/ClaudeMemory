# ClaudeMemory

Mac mini（`LinusdeMac-mini.local`）上三個專案的 `CLAUDE.md` 備份／同步。

每個資料夾對應一個 repo 根目錄的 `CLAUDE.md`（Claude Code 的常駐專案備忘）：

| 資料夾 | 來源 | base branch |
|---|---|---|
| `Ai_Stock_Agent/` | `/Users/linus/Ai_Stock_Agent/CLAUDE.md` | `main` |
| `edurpg/` | `/Users/linus/edurpg/CLAUDE.md` | `master` |
| `cleo/` | `/Users/linus/cleo/CLAUDE.md` | — |

> 這裡是**單向備份快照**,不是 source of truth——各專案 repo 內的 `CLAUDE.md` 才是。
> 更新方式:跑同步腳本(見下),或手動把對應的 `CLAUDE.md` 複製過來重新 commit。

## 自動同步

Mac mini 上有一個維運腳本 + launchd job 負責每天把三份 `CLAUDE.md` 推上來。

| 項目 | 路徑 / 值 |
|---|---|
| 同步腳本 | `/Users/linus/scripts/syncMemory.sh`(`--dry-run` 可預覽,不 commit/push） |
| plist 正本 | `/Users/linus/scripts/com.linus.claudememory.sync.plist` |
| launchd label | `com.linus.claudememory.sync`（**system LaunchDaemon**,跑在 `linus` 身分） |
| 已安裝位置 | `/Library/LaunchDaemons/com.linus.claudememory.sync.plist`（`0644 root:wheel`） |
| 排程 | 每天 **04:30**（`RunAtLoad=false`） |
| log | `/Users/linus/scripts/logs/syncMemory.{out,err}.log` |

行為:`git pull --ff-only` → 複製三份 `CLAUDE.md` → **只有真的有 diff 才** commit（訊息自帶時間戳）+ push;無變更靜默跳過。各專案 repo 全程**唯讀**,所有 git 寫入只對本 repo。

### 手動觸發 / 維運

```bash
/Users/linus/scripts/syncMemory.sh            # 立即同步
/Users/linus/scripts/syncMemory.sh --dry-run  # 只看會改什麼

# 重新安裝 / 載入 plist（改 plist 後,需 sudo;在互動終端機直接打,勿用 ! 前綴避免 zsh history expansion）
sudo install -m 0644 -o root -g wheel /Users/linus/scripts/com.linus.claudememory.sync.plist /Library/LaunchDaemons/
sudo launchctl bootout system /Library/LaunchDaemons/com.linus.claudememory.sync.plist   # 已載入才需先 bootout
sudo launchctl bootstrap system /Library/LaunchDaemons/com.linus.claudememory.sync.plist
sudo launchctl kickstart -k system/com.linus.claudememory.sync                           # 立刻試跑一次
```

> 用 **LaunchDaemon 而非 user LaunchAgent**:Mac mini 沒開 auto-login,LaunchAgent 重開機後不登入就不會跑,LaunchDaemon 則無條件照跑。
>
> **新增第四個專案**:在 `syncMemory.sh` 頂部的 `PROJECTS=(...)` 加一行 `"資料夾名:/絕對路徑/CLAUDE.md"`,並在這台 clone 建對應資料夾即可。
