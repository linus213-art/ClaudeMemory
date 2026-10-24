# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> 這份只放「每次都要遵守」的規則 + 常用速查。主題細節已拆出去，**做到那個主題時再讀**：
> - `src/CLAUDE.md` — Phaser 場景縮放 / 邊界 / 踩坑清單 / 背包元件 / 靜態資源相對路徑（讀 `src/` 檔案時自動載入）
> - `backend/CLAUDE.md` — 新題庫授權（bank_accesses）/ ECPay / Google 模型 / TTS 補跑（讀 `backend/` 檔案時自動載入）
> - `docs/runbooks/` — Mac 環境完整版、dispatcher、發版 + iOS 簽章、FLUX 產圖流程、Windows 專用（索引見 `docs/runbooks/README.md`）

## 語言規則（最高優先，務必遵守）

> **永遠用繁體中文回覆使用者。** 不論對話進行多久、不論工具輸出或程式碼是什麼語言，
> 對使用者的所有回覆（說明、摘要、提問、進度回報）一律用繁體中文。**不要中途自行切換成英文。**
> 程式碼、commit message、檔名、API 等技術名詞可保留原文（英文），但包覆它們的敘述文字要是中文。

## 完工自我審查規則（每次任務做完務必執行）

> **每完成一項工作，交付前一定要用「資深工程師 code review」的角度，自己再審一遍有沒有問題或遺漏。**
> 不要做完就直接回報「完成」——先當自己的 reviewer，把以下都過一遍，發現缺口就補掉（或如實列出），再給使用者結論。

審查清單（依任務性質取用）：
- **需求對齊**：把使用者這次（含過程中追加 / 修正）的每一條要求列出來，逐條對照「是否真的做到、做的跟舉例一致」。
  特別注意自己**擅自簡化或偏離使用者明確舉例**的地方（例：使用者說 A，我做了相近的 B）。
- **完整性盤點**：同類的東西有沒有漏接？（例：要接語音的「所有」廣播句、要改的「所有」呼叫點、要改的「所有」locale）
  用 grep 把同類呼叫點全列出來，逐一確認每個都處理或刻意略過（並說明為何略過）。
- **正確性 / 邊界**：null / 空字串 / 找不到 / 跨裝置 / 多人 / iOS 等邊界；前後端共用邏輯有沒有保持同步。
- **驗證**：typecheck（前端 + backend 各自）綠；該 build 的有 build（改 `src/` → `--build-all`）；產出物實際可服（curl 200 / 看圖）。
- **副作用**：有沒有破壞既有行為、留下無用檔 / dead code、commit 了不該 commit 的東西。
- **誠實回報**：跑失敗就說失敗附輸出；刻意沒做的要明講原因；不要把「沒驗證」講成「已完成」。

做完 review 後，在給使用者的結論裡**主動說明**：完成了什麼、刻意保留沒做的（含原因）、還有什麼已知缺口或後續建議。
使用者若問「有漏掉沒做的嗎」，這份審查就是答案來源。

## macOS 本機環境（摘要；完整版 `docs/runbooks/mac-environment.md`）

- **Repo root + PROD 都在** `/Users/linus/edurpg`（branch `master`）。Mac 上 prod 直接從 master root 跑，**沒有 dev/prod 樹分裂**；`backend/.env`、`admin/.env` 改完不必複製到 `stable-3005`。
- `/Users/linus/edurpg.worktrees/` 只給 dispatcher 用（`task-NNN/`）；`stable-3005/` 只是 git mirror，**不是** prod 部署目標。
- Logs / PID：`tools/logs/`（`.run-state.json`、`prod-backend.log`…）。Windows 端操作看 `docs/runbooks/windows.md`。

### 服務啟動 / 停止

```bash
./scripts/restart_prod.sh                  # postgres 檢查 → backend(3020) → admin(3010) → frontend(3005) → cloudflared
./scripts/restart_prod.sh --stop-only
./scripts/restart_prod.sh --build-all      # 強制重 build backend + frontend
./scripts/restart_prod.sh --no-tunnel
```

> 🟥 **只要改了 `src/...` 或 `backend/src/...` 任何 .ts，restart_prod 一律帶 `--build-all`。**
> 預設只在 `dist/` 不存在時才 build，不帶的話 prod 會繼續服舊 bundle（重複踩雷過）。只改 `public/` 不必 build。

> 🟥 **重啟 PROD 前先確認沒人在線**（`SessionRunner` 狀態在記憶體，重啟會丟）；未經本回合授權不要重啟 prod：
> ```bash
> key=$(grep '^ADMIN_API_KEY=' /Users/linus/edurpg/backend/.env | cut -d= -f2 | tr -d '\r')   # .env 是 CRLF，不去 \r → 400
> curl -s -H "x-admin-key: $key" http://localhost:3020/api/admin/quiz/live-count
> ```
> `count > 0` → 先問使用者。常駐授權用語見 `Develop_MEMORY.md` §2.4。

### 服務 / DB

| 服務 | 連線 | 備註 |
|---|---|---|
| Postgres | `postgresql://edurpg:<pw>@localhost:5432/edurpg_db` | Homebrew `postgresql@16` |
| Backend(PROD) | `http://127.0.0.1:3020` | `node dist/index.js`，健康檢查 `/health` |
| Admin(PROD) | `http://127.0.0.1:3010` | bind loopback only |
| Frontend(PROD) | `http://127.0.0.1:3005` | `vite preview --strictPort` |
| Public | `https://edurpg.org` | cloudflared → `:3005` |

同一台 Postgres 多專案共用（edurpg / cleo / ai_stock_agent）。**🟥 不要 `brew services stop postgresql@16`**，會打掛別的專案。

### Dispatcher

`pwsh ./tools/agent_dispatch_mac.ps1`（`-TaskId TASK-005` / `-Agent CODE -DryRun` / `-NoMerge` / `-SkipPush`）。
軟停止：`touch tasks/logs/.stop-requested`。目錄結構、pipeline、跨專案 guard 見 `docs/runbooks/mac-environment.md`。

## Repo layout — three packages, one repo

```
edurpg/
├── src/        Phaser 4 + React + TS frontend (game)        → port 3005 (prod) / 3105 (dev)
├── backend/    Express + Prisma + socket.io API + crons     → port 3020 (prod) / 3120 (dev)
└── admin/      Express static + bcrypt-auth admin console   → port 3010 (prod) / 3011 (dev)
```

Each has its own `package.json` / `node_modules`. Always `cd` into the right one before running scripts. The root `package.json` only builds the game frontend.

## Common commands

- Frontend (`/`): `npm run build`（`tsc && vite build` → `dist/`）、`npm run dev`。
- Backend (`backend/`): `npm run build`（`tsc`）、`npm start`、`npm test`；單一測試 `npx vitest run __tests__/<file>.test.ts -t "<name>"`；一次性診斷腳本 `node backend/scripts/<name>.cjs`。
- Prisma（在 `backend/` 內）：`npx prisma generate` / `npx prisma migrate deploy`（PROD）/ `npx prisma migrate dev --name xxx`。
- Admin (`admin/`): `npm start`（`PORT` env 決定 3010 / 3011）。

## 版本號（細節 `docs/runbooks/release-and-ios-signing.md`）

Web / Android / iOS 共用 `/VERSION`（versionName）+ `android/app/build.gradle` 的 `versionCode`。**不要手動改**，用 `tools/release_android.sh`（三平台版本號 manager）；iOS 版號由 `tools/build_ios.sh` 衍生，別自己 `agvtool`。`package.json` 的 `"version"` 跟對外版號無關。

## 🟥 紅線（違反會在手機 / iOS / 公網上安靜壞掉）

- **Frontend fetch 一律相對路徑**：`const API_BASE = (import.meta.env.VITE_API_BASE as string | undefined) ?? '';`（必須是 `''`）。
  Hardcode `'http://localhost:3020'` 在桌機正常、手機 / tunnel 上靜默失敗。Guard：`grep -rn "?? 'http://localhost:" src/` 要 0。
- **Phaser 載 `public/assets/...` 一律相對路徑，不要 prefix `${API_BASE}`**（iOS WKWebView 的 spritesheet 會靜默失敗）。
  Guard：`grep -rE 'API_BASE.*assets/|VITE_API_BASE.*assets/' src/` 要 0。細節見 `src/CLAUDE.md`。
- **iOS 本地測一律走 `tools/build_ios.sh`**，不要直接 `npm run build`（漏 `VITE_API_BASE` → 登入炸 `"The string did not match the expected pattern"`）。
- **每個新 Phaser 場景 `create()` 都要 `setSceneBg(...)` + `setupResponsiveCamera()`**，否則手機邊條露出上個場景背景。完整解法見 `src/CLAUDE.md`。
- **背包 / 物品欄全遊戲只有一個共用元件** `src/ui/LexiconInventoryOverlay.tsx`，不要再做第二份。
- **`.sql` migration 一律 UTF-8 無 BOM**（BOM → P3018，要 `prisma migrate resolve --rolled-back` 才救得回）；`.ts/.tsx/.json/.md` 也無 BOM。
- **Admin auth**：backend `/api/admin/*` 要 `x-admin-key` header，Vite proxy 不轉發（公網 403 是故意的）；admin server 只 bind 127.0.0.1，不要暴露到 LAN。
- **新匯入的題庫在「選擇關卡」看不到** → 九成是 `bank_accesses` 沒授權，不是篩選器。先看 `backend/CLAUDE.md`，**絕不要跑 `migrate-existing-players-onboarding.ts`**（會清空全部人的授權）。

## High-level architecture

- **Frontend (`src/`)**：`main.ts` 註冊所有 Scene（`BootScene` → `TitleScene`）；`scenes/` 遊戲畫面；`systems/` 純邏輯 singleton（VocabManager、BattleSystem、PlayerManager…，localStorage + `/api/save` 雲端同步）；`services/` HTTP client；`ui/` React overlay；`i18n/` 7 語系，一律 `t('key.path')`，缺 key fallback 英文（**en 必須 100%**，否則 App Store 退件）。
- **Backend (`backend/src/`)**：`index.ts`（CORS 只允許私有 LAN、1000/15min rate limit、`trust proxy 'loopback'`、socket.io 共用 3020）；`routes/*.ts` 每檔一個功能掛 `/api/*`；`quiz/` 多人問答（`sessionRunner.ts` 是記憶體狀態）；`lib/` 領域模組；`middleware/auth.ts` 單裝置登入（登入即撤銷其他 session）；`prisma/schema.prisma` ~60 models。
- **Admin (`admin/`)**：`server.js` 代理 `/api/admin/*`（自動帶 `x-admin-key`）+ 服 `admin/public/` 靜態 HTML。

## UI 慣例 / 瀏覽器

- **按鈕配色對比**要夠（淺灰底白字、深藍底黑字都不行）；**所有按鈕圓角**（Phaser `fillRoundedRect` / Tailwind `rounded-*`）。
- 不要使用我Chrome裡面Oracle資料夾的bookmark裡面的URL。
- 使用chrome瀏覽器的時候, 請使用linus94213@gmail.com這個account。

## Reference docs（需要時才讀）

- `docs/ISSUE_HISTORY.md` — **CI / 測試 / dispatcher / 部署 / 環境卡關，先 `grep -i "關鍵字" docs/ISSUE_HISTORY.md`** 找前例；解掉新問題後追加一條。
- `docs/feedback/README.md` — Phaser / UI / 美術 bug pattern library，改 UI / 寫新場景前看索引。
- `docs/appstore/SUBMISSION_LOG.md` — **iOS 送審 / 被退件重送前逐條自查**，每次送審後追加更新。
- `docs/I18N_LOCALIZATION_RUNBOOK.md` — 多語系翻譯流程（UI 字串 + 題庫單字 + 離線 JSON）。
- `docs/ASSET_GEN.md` + `docs/runbooks/asset-gen-flux.md` — 產素材（AutoSprite / FLUX / Azure TTS）；產一批美術一律走「多變體 → checkbox 預覽 → 使用者勾 CSV」流程，不要自己選圖。
- `OPERATIONS.md` — restart / deploy / recovery runbook。`Develop_MEMORY.md` — 決策 / bug / 慣例流水帳。
- `PROJECT_STRUCTURE.md` — Phaser scene/system reference。`CHANGELOG.md` / `RELEASE.md` — 版本歷史與發版規則。
