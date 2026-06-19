# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 語言規則（最高優先，務必遵守）

> **永遠用繁體中文回覆使用者。** 不論對話進行多久、不論工具輸出或程式碼是什麼語言，
> 對使用者的所有回覆（說明、摘要、提問、進度回報）一律用繁體中文。**不要中途自行切換成英文。**
> 程式碼、commit message、檔名、API 等技術名詞可保留原文（英文），但包覆它們的敘述文字要是中文。

## macOS 本機環境（最常被問,先看這段）

> 這個 repo 同時跑 Windows 工作站(`E:\AI_Workbench_Data\edurpg`)和 macOS(`/Users/linus/edurpg`)。
> 當前 session 在 macOS,以下是 Mac 上不會變的常駐答案,不必每次重問。

### 1. Work area / 路徑

- **Repo root + PROD 跑這**:`/Users/linus/edurpg`(branch `master`)。
  `restart_prod.sh` 第 38 行 `ROOT="$HOME/edurpg"`,backend 第 436 行 cwd 就是 `$ROOT/backend`。**Mac 跟 Windows 不同——Windows 是 `stable-3005` worktree 跑 prod、master 跑 dev;Mac 上 prod 直接從 master root 跑,沒有 dev/prod 樹的分裂**。所以 backend code / `.env` 改完不必先 promote,restart_prod.sh 跑下去就生效。
- **Worktrees dir**(**只給 dispatcher 用**,**不是 prod 部署目標**):`/Users/linus/edurpg.worktrees/`
  - `stable-3005/` ← 雖然存在,**Mac 上 prod 並不從這跑**。它是個 git-synced mirror,promote 流程裡照 push origin/stable-3005 是為了分支歷史一致(別人 / Windows 端可能還在用),但實際 process 都從 master root 起的
  - `task-NNN/` ← dispatcher 為每個任務開的暫時 worktree
- **Logs / PID**:`/Users/linus/edurpg/tools/logs/`(含 `.run-state.json`、`prod-backend.log` 等)
- **Env 檔**:`/Users/linus/edurpg/backend/.env`、`/Users/linus/edurpg/admin/.env`(gitignored)。**改完不必複製到 stable-3005**——prod 讀的就是 master root 那份

### 2. 服務啟動 / 停止(Mac)

主腳本是 bash,不是 PowerShell —— `scripts/restart_prod.sh`:

```bash
cd /Users/linus/edurpg
./scripts/restart_prod.sh                  # 啟動順序:postgres 檢查 → backend(3020)→ admin(3010)→ frontend(3005)→ cloudflared
./scripts/restart_prod.sh --stop-only      # 只停服務
./scripts/restart_prod.sh --build-all      # 強制重 build backend + frontend
./scripts/restart_prod.sh --no-tunnel      # 跳過 cloudflared
```

健康檢查 timeout 45s,任一失敗就終止並 dump 該服務最後 30 行 log。Postgres 由 Homebrew (`postgresql@16`) 起,腳本只檢查不重啟(共用 idempotent 服務)。

**重啟 PROD 前**仍要先確認沒人在線:
```bash
key=$(grep '^ADMIN_API_KEY=' /Users/linus/edurpg/backend/.env | cut -d= -f2)
curl -s -H "x-admin-key: $key" http://localhost:3020/api/admin/quiz/live-count
```
`count > 0` → 先問使用者再重啟。(讀 master root 的 `.env`,不是 stable-3005——理由在上一段)

### 3. Dispatcher — `tools/agent_dispatch_mac.ps1`

PowerShell 7+ (`pwsh`)寫的 mac 版派工器,跟 Windows 的 `agent_dispatch.ps1` 同 pipeline 但路徑改 mac。

**Pending tasks 目錄結構**:
```
tasks/pending/CODE/      ← 給 Claude Code(複雜邏輯、跨檔重構)
tasks/pending/Copilot/   ← 給 Copilot CLI(scaffold、單檔 CRUD)
tasks/pending/codex/     ← 給 Codex CLI
tasks/done/              ← 完成後自動搬過來(含 .result.md commit hash + diff stat)
tasks/logs/              ← 每次執行 log;`.stop-requested` 為軟停止旗標
```
Spec frontmatter 寫 Windows path(`E:\AI_Workbench_Data\edurpg.worktrees\task-NNN`),dispatcher 會抽 leaf 改成 `/Users/linus/edurpg.worktrees/task-NNN`,不必改 spec。

**常用呼叫**:
```bash
pwsh ./tools/agent_dispatch_mac.ps1                       # 自動撿下一個未 block 的任務
pwsh ./tools/agent_dispatch_mac.ps1 -TaskId TASK-005      # 指定任務
pwsh ./tools/agent_dispatch_mac.ps1 -Agent CODE -DryRun   # 只印計畫
pwsh ./tools/agent_dispatch_mac.ps1 -NoMerge              # 跑完不 merge,保留 worktree
pwsh ./tools/agent_dispatch_mac.ps1 -SkipPush             # 不 push 到 origin/master
```
Agent 名稱接受 `CODE` / `Claude` / `Copilot` / `Codex`(`codex` folder 也認)。

**軟停止**(等當前任務跑完再退):
```bash
touch /Users/linus/edurpg/tasks/logs/.stop-requested
```

**跨專案 guard**:`REPO_ROOT` / `WORKTREES_DIR` 路徑含 `cleo` / `aistock`,或 shell 有 `CLEO_*` / `AISTOCK_*` env,dispatcher 拒跑。同 shell 不要混設多專案 env。

**Pipeline 順序**:`worktree add → claude/codex/copilot run → sanity check(必須有新 commit)→ npm ci(root + backend)→ build → test → merge to master → 搬 spec 到 done/ + 寫 .result.md → push`。Usage limit 失敗自動 retry(最多 20 次,backoff 15min × n),偵測到 `CLAUDE_FALLBACK_API_KEY` 會切備用 key。

**Override env**:`EDURPG_REPO_ROOT`、`EDURPG_WORKTREES_DIR`、`EDURPG_BASE_REF`。

### 4. DB / 服務連線(local)

連線字串在各 worktree 的 `backend/.env`(gitignored),Mac 上的形式:

| 服務 | 連線 | 備註 |
|---|---|---|
| Postgres | `postgresql://edurpg:<pw>@localhost:5432/edurpg_db` | Homebrew `postgresql@16`,user `edurpg`,DB `edurpg_db` |
| Backend(PROD) | `http://127.0.0.1:3020` | `node dist/index.js`,健康檢查 `/api/health` |
| Admin(PROD) | `http://127.0.0.1:3010` | `node server.js`,bind loopback only |
| Frontend(PROD) | `http://127.0.0.1:3005` | `vite preview --strictPort` |
| Public tunnel | `https://edurpg.org` | cloudflared `vite-app` → `:3005` |
| Dev backend / frontend | 3120 / 3105 | 用 Windows 端的 `restart_dev_stack.ps1`;Mac 端目前只跑 PROD |

DB 同一台 Postgres 上多專案共用:`edurpg_db`(prod)、`edurpg_dev_local`(dev,Windows 用)、`cleo`、`ai_stock_agent` 各自獨立 schema。**Mac 上不要 `brew services stop postgresql@16`**,會打掛 cleo / aistock。

Prisma 操作(在 `backend/` 內):
```bash
cd /Users/linus/edurpg/backend
npx prisma generate
npx prisma migrate deploy          # PROD migration
npx prisma migrate dev --name xxx  # 開新 migration
```

---

## Repo layout — three packages, one repo

```
edurpg/
├── src/        Phaser 4 + React + TS frontend (game)        → port 3005 (prod) / 3105 (dev)
├── backend/    Express + Prisma + socket.io API + crons     → port 3020 (prod) / 3120 (dev)
└── admin/      Express static + bcrypt-auth admin console   → port 3010 (prod) / 3011 (dev)
```

Each has its own `package.json` / `node_modules`. Always `cd` into the right one before running scripts. The root `package.json` only builds the game frontend.

## Common commands

Frontend (`/`):
- `npm run dev` — Vite dev server on **3005** locally (config), but the dev workflow uses the wrapper scripts which start it on **3105**.
- `npm run build` — `tsc && vite build` → `dist/`.
- `npm run dev:all` → `tools/restart_dev_stack.ps1` (full dev+prod stack restart).
- `npm run dev:promote` → `tools/promote_to_prod.ps1`.

Backend (`backend/`):
- `npm run dev` — `ts-node-dev --respawn --transpile-only src/index.ts`.
- `npm run build` — `tsc` → `backend/dist/`.
- `npm start` — `node dist/index.js` (used in prod).
- `npm test` / `npm run test:watch` — vitest. Run a single test: `npx vitest run __tests__/autogenGenerators.test.ts -t "<name>"`.
- `npx prisma generate` / `npx prisma migrate deploy` / `npx prisma migrate dev`.
- One-shot diagnostic scripts live in `backend/scripts/*.cjs` — run with `node backend/scripts/<name>.cjs`.

Admin (`admin/`):
- `npm start` — `node server.js` (port 3010 prod / 3011 dev via `PORT` env).

## Versioning — 三平台共用版本號,別手動改

Web / Android / iOS 共用一個 `versionName`(對外版號)+ 一個 `versionCode`(整數 build counter)。完整規則看 `RELEASE.md`,這裡只列**版本號實際存在哪、由誰寫**:

| 欄位 | 檔案 / 位置 | 例值 | 誰會讀 |
|---|---|---|---|
| `versionName`(marketing) | `/VERSION` | `0.3.91` | Web(`vite.config.ts`)、`tools/build_ios.sh` 寫進 iOS `CFBundleShortVersionString`、`release_android.sh` 同步進 build.gradle |
| `versionCode`(build counter) | `/android/app/build.gradle` 第 20 行 | `versionCode 10` | Android 直接用、`build_ios.sh` 寫進 iOS `CFBundleVersion` |
| `versionName`(Android 鏡像) | `/android/app/build.gradle` 第 21 行 | `versionName "0.3.91"` | Android 直接用(由 `release_android.sh` 從 VERSION 同步,不要手動編) |

**不出現在版本號裡的地方**(別誤改):
- `package.json` / `backend/package.json` / `admin/package.json` 的 `"version"` — npm package metadata,跟對外版號無關,從來沒同步過
- `CHANGELOG.md` / `RELEASE.md` 內文提到的版本號 — 是 release notes,不是 source of truth

**標準 bump 方式**:跑 `tools/release_android.sh`(雖名為 android,實為三平台版本號 manager)。它會:
1. `versionCode +1`(Play Store 規則:嚴格遞增,跟 versionName 無關)
2. 視 flag bump `versionName`(`--version-name 1.0.1` 或 `--bump-patch`)
3. Android build + 上傳 Play Console internal track
4. (可選)commit + push + prepend CHANGELOG.md

**iOS-only 快速路徑**:如果這次只發 iOS、不想白等 5–10 min Android build,可以手動只改兩處(因為 `release_android.sh` 沒有 `--skip-build` flag):
1. `/VERSION` ← 寫新 versionName(例 `1.0.1`)
2. `/android/app/build.gradle` 第 20–21 行 ← `versionCode +1` 並把 `versionName` 對齊 VERSION

然後直接跑 `tools/build_ios.sh`(它會用 `agvtool` 把 VERSION + versionCode 寫進 Xcode project)→ `tools/upload_testflight.sh`。下次三平台一起發再用 `release_android.sh` 補 Android。

iOS 版本號是**衍生**的:`build_ios.sh` 跑時用 `agvtool` 從 VERSION + build.gradle 寫進 Xcode project,**不要自己 `agvtool` 去戳 iOS 那邊**,會被下次 build 覆寫。

## iOS code signing — 機器本地狀態(別人 checkout 不會有)

App Store / TestFlight build 需要的 cert + provisioning profile **不在 git**,只存在這台 Mac 的 keychain 跟 Xcode UserData。換機器或 keychain 被清就要全部重生一次。session 記錄如下,給未來 build 1.0.2 / 1.0.x 不必再 debug 一次:

| 物件 | 識別 | 哪裡 |
|---|---|---|
| Apple ID(Xcode + ASC) | `linus213@hotmail.com` | Xcode → Settings → Accounts |
| **Team ID** | **`ALGJXG7K45`**(Team name 是 "Chih-Wei Lee Developer Team") | 寫在 `ios/App/App.xcodeproj/project.pbxproj` 的 `DEVELOPMENT_TEAM` |
| Distribution cert | `Apple Distribution: Chih-Wei Lee (ALGJXG7K45)`,SHA-1 `CCB67887067CB9AEB87CE388C818AC136D2B8756` | login.keychain(`security find-identity -p codesigning -v`) |
| Distribution profile | name `eduRPG App Store`,UUID `03c652ec-bc46-4cc0-8747-75f534c3a377`,2027-05-28 過期 | `~/Library/Developer/Xcode/UserData/Provisioning Profiles/` |
| ASC API key(altool + xcodebuild 用) | KEY_ID 在 `APP_STORE_CONNECT_API_KEY_ID` env,.p8 在 `~/.appstoreconnect/private_keys/AuthKey_*.p8` | 這個是入 env,跨 session 都在 |

**`exportOptions.plist` 用 manual signing**(不是 automatic):因為現有 ASC API key 的 role 不夠 App Manager,跑 automatic signing 時 cloud-signed flow 會踩 `Cloud signing permission error`。manual 直接吃 keychain cert + 上面那個 profile name。**未來把 ASC key 升成 App Manager 後可以改回 automatic,profile 名 + cert 名都可以拿掉。**

### Cert subject 的坑(讀錯一次了)

`security find-identity` 顯示 `"Apple Development: Chih-Wei Lee (6M5T9ZY2V6)"`,但**括號裡的 `(6M5T9ZY2V6)` 不是 team ID,是個人 developer member ID**。真 team ID 在 cert subject 的 OU 欄位(`OU=ALGJXG7K45`)。設 `DEVELOPMENT_TEAM` 要用 OU 那個,不是 CN parens。確認指令:
```bash
security find-certificate -c "Apple Distribution" -p ~/Library/Keychains/login.keychain-db | openssl x509 -noout -subject
```

### 如果 cert / profile 全沒了(新 Mac、keychain 被清)

完整 recovery 路徑(這次 session 走過,踩過所有雷):
1. Xcode → Settings → Accounts → 登入 `linus213@hotmail.com`(team 自動出現)
2. 選 team → Manage Certificates → `+` → **Apple Distribution**(不是 Apple Development),自動進 login.keychain
3. https://developer.apple.com/account/resources/profiles/list → `+` → Distribution / App Store Connect → bundle id `org.edurpg.app` → 勾上面建的 Distribution cert → Generate → Download
4. 雙擊 `.mobileprovision`(或 cp 到 `~/Library/Developer/Xcode/UserData/Provisioning Profiles/`)
5. **profile name 跟 `tools/exportOptions.plist` 的 `provisioningProfiles` 對應值要一致**(目前是 `eduRPG App Store`),不一樣的話改 plist 或重命名 profile

跑 `tools/build_ios.sh && tools/upload_testflight.sh` 應該就過了。

## Restart / deploy — use the scheduled tasks, not raw scripts

There are TWO Windows Scheduled Tasks pre-registered (run with `Start-ScheduledTask <name>`, no UAC prompt):
- `EduRPGRestartProdOnly` — only restarts prod (3005/3010/3020 + Cloudflare tunnel). **This is the default for daily promote.**
- `EduRPGRestartStack` — restarts dev + prod (use only when both are broken).

Direct invocation of `restart_prod_only.ps1` / `restart_dev_stack.ps1` triggers UAC. Avoid.

**Before any prod restart**, check live multiplayer sessions — `SessionRunner` state is in-memory and is lost on restart:
```powershell
$key = (Select-String -Path E:\AI_Workbench_Data\edurpg.worktrees\stable-3005\backend\.env -Pattern '^ADMIN_API_KEY=').Line.Split('=')[1]
(Invoke-WebRequest http://localhost:3020/api/admin/quiz/live-count -Headers @{ 'x-admin-key' = $key } -UseBasicParsing).Content
```
`count > 0` → stop and ask the user before restarting.

Promote flow: commit on `master` → `git push origin master` → in `stable-3005` worktree `git fetch && git reset --hard origin/master` → `Start-ScheduledTask EduRPGRestartProdOnly`. **Also `git push origin stable-3005` afterwards** (master alone isn't enough — see `feedback_promote_push_stable3005.md`).

Logs: `tools/logs/prod-backend.log`, `tools/logs/dev-backend.log`, `.last-restart-transcript.log`, `.last-restart-marker`.

## Worktree topology — dev vs prod are separate file trees

```
E:\AI_Workbench_Data\edurpg\                              ← DEV (master branch)
E:\AI_Workbench_Data\edurpg.worktrees\stable-3005\        ← PROD (stable-3005 branch)
```

Both share PostgreSQL but have separate `node_modules` and own `backend/.env` / `admin/.env`. Never edit code in `stable-3005/` — it's overwritten by `git reset --hard` on every promote. Verify a listener is in the right tree by inspecting its cmdline (see `OPERATIONS.md` §1.3.1).

DB split (since 0.3.77): dev backend uses `edurpg_dev_local`, prod uses `edurpg_dev`. Migrations are applied to both via `prisma migrate deploy` during the respective restart.

## Frontend → backend networking — never hardcode localhost

All `fetch` calls **must** use a relative path so the Vite proxy can route them:
- prod: `https://edurpg.org/api/*` → Cloudflare tunnel → `:3005` Vite preview → proxy → `:3020` backend.
- dev:  `:3105` Vite → proxy → `:3120` backend.
- The Vite proxy also forwards `/socket.io` (with `ws:true`) and `/uploads`.

```ts
const API_BASE = (import.meta.env.VITE_API_BASE as string | undefined) ?? '';   // ← MUST be ''
fetch(`${API_BASE}/api/...`, { credentials: 'include' });
```

A hardcoded `'http://localhost:3020'` fallback works on the developer's desktop but silently breaks on phones / over the Cloudflare tunnel (DNS fails, code falls into a try/catch, user sees default/empty data without an error). Grep guard: `grep -rn "?? 'http://localhost:" src/` should always return zero hits.

## Admin auth — two-layer

1. Backend `/api/admin/*`: requires `x-admin-key: <ADMIN_API_KEY>` header. Vite proxy does **not** forward this header, so `https://edurpg.org/api/admin/*` always returns 403 — that's intentional. Only `admin/server.js` (loopback) and local scripts can call admin endpoints.
2. Admin UI on `:3010`: bcryptjs login + HMAC-signed cookie (12h, HttpOnly, SameSite=Strict). `admin/.env` must have `ADMIN_USERNAME` (≠ `'admin'`), `ADMIN_PASSWORD_HASH`, `ADMIN_SESSION_SECRET`. Cookie version is derived from sha256(password hash) so changing the password invalidates all old cookies (stateless revocation).

Admin server binds 127.0.0.1 only — never expose to LAN.

## 金流 / ECPay 綠界 — 狀態與待辦（2026-06-06）

兩處金流都已串好並實測入帳成功(`backend/src/routes/payment.ts` + `backend/src/lib/ecpay.ts`):

- **遊戲內**(edurpg.org,登入後) → `/api/payment/orders`(單包)、`/orders/multi`、`/orders/educational`,訂單綁登入帳號。
- **products.html**(網頁儲值,免登入) → 輸入玩家 8 碼**推薦碼** → `/api/payment/web-topup/resolve`(預覽收款人)→ 確認「儲值給 〇〇〇」→ `/api/payment/web-topup` 建單刷綠界 → notify 把鑽石撥入該推薦碼帳號。此路徑刻意免登入(只會加值、不能扣款)。
- 教育包 / 月卡在 products.html 都換算成鑽石(`EDU_RATE_PER_TWD = 3.2`💎/NT$),玩家再在遊戲內用鑽石開月卡(`subscription.ts` `payWith:'diamonds'`)。
- 後台稽核:admin `index.html` → 💎 金流頁,有 email 篩選 + 撥入前/後鑽石餘額(由 `currency_transactions` 累計推算)。

**正式商店代號 `3499022`(金流已申請通過),`.env`:`ECPAY_ENV=prod`。**

### ⚠️ 電子發票尚未開通 — 待辦進度

綠界**電子發票是跟金流分開的加值服務**,需要**公司營業登記** + 財政部字軌核配 + 後台「電子發票系統 → 資料管理與維護 → 字軌與配號設定」(B2C、字軌類別 **07 一般稅額**,對上 code 送的 `InvType='07'`)。**目前使用者尚無公司營業登記,無法申請。**

未開通前若送 `InvoiceMark='Y'`,綠界收銀台會以錯誤碼 **1200005「無商品數量」**擋下整筆交易。因此整合金流發票已用 env flag 關閉:

- `backend/.env` → `ECPAY_INVOICE_ENABLED=false`(預設關;`backend/src/lib/ecpay.ts` 的 `ecpayInvoiceEnabled()` 控制 `buildAioCheckoutForm` 是否帶發票欄位)。
- 涵蓋全部結帳路徑(orders / multi / educational / web-topup)。

**進度追蹤 — 之後要做的事(完成電子發票申請後):**
1. 公司營業登記辦好 → 向綠界申請電子發票加值服務(電話 02-2655-1775)→ 簽約 + 財政部授權。
2. 綠界後台設好 B2C 字軌與配號(07 一般稅額)。
3. 把 `backend/.env` 的 `ECPAY_INVOICE_ENABLED` 改成 `true` → 重 build backend + 重啟 prod。**code 不必再改。**
4. 過渡期(尚未開通)若需開發票,用綠界後台「B2C 電子發票 → 發票新增作業」手動補開。

## Google API / 模型(用到哪些、在哪、退役狀態）

所有 Gemini model 走 REST `generativelanguage.googleapis.com/v1beta/models/<model>:generateContent`(`backend/src/lib/geminiImageGen.ts`),model 名存在 DB provider row,admin 可用 `endpoint` 或 `notes` 的 `model=...` 逐列覆寫,下面是 fallback 預設值:

| 用途 | 檔案(預設) | 預設模型 | API |
|---|---|---|---|
| 出題配圖(quiz 圖片生成) | `lib/geminiImageGen.ts:58` | `gemini-2.5-flash-image` | GenAI generateContent(回 inline image) |
| 文字 LLM(三檔 tier) | `lib/llmText.ts:37-39` | `gemini-2.5-flash-lite` / `-flash` / `-pro` | GenAI generateContent |
| Vision(圖片判讀) | `lib/vision.ts:149,220` | `gemini-2.5-flash` | GenAI generateContent |
| 圖片審核(主)| `lib/imageModeration.ts:155` | `gemini-2.5-flash` | GenAI generateContent |
| 圖片審核(備援,NSFW)| `lib/imageModeration.ts:266` | — | **Google Vision API**(`GOOGLE_VISION_API_KEY`,fail-open) |

**Imagen 4 退役(2026-08-17)不影響本專案** — 全部走 `gemini-2.5-*`,無任何 `imagen-4.0-*` 端點。Google 通知點到 project `gen-lang-client-0836655205` 是以 GCP project 為單位偵測(同 key 下別的 app/手動測試曾打過 Imagen 4)。詳見跨專案盤點:cleo 的一次性頭像腳本已從 `imagen-4.0-fast` 遷到 `gemini-3.1-flash-image`;Ai_Stock_Agent 不受影響。

## High-level architecture

**Frontend (`src/`)** — Phaser 4 game with React overlays:
- `main.ts` registers all Phaser Scenes; `BootScene` → `TitleScene` is the entry flow.
- `scenes/` — game screens (Battle, Cute, Result, Multiplayer, etc).
- `systems/` — pure-logic singletons (VocabManager, BattleSystem, PlayerManager, ReviewSystem, AudioManager, SpellEffects, TutorialManager). Persisted state lives in localStorage; cloud sync via backend `/api/save`.
- `services/` — auth / cloudsave / presence / session / tts HTTP clients (use the relative-path convention above).
- `ui/` — React overlays mounted on top of the Phaser canvas (settings, store, leaderboard, etc).
- `i18n/` — 7 locales (zh, zh-cn, en, ja, ko, vi, ar). Always go through `t('key.path')`; missing keys fall back to English. `tools/sync_i18n.cjs` backfills missing keys from zh.

**Backend (`backend/src/`)**:
- `index.ts` — Express setup, CORS allowlist (private LAN only — public domains rejected), 1000/15min global rate limit (skips GET in prod, disabled in dev), `trust proxy 'loopback'` (only loopback proxies trusted). Wraps Express in `http.Server` so socket.io shares port 3020.
- `routes/*.ts` — REST endpoints, mounted under `/api/*`. Each file is one feature (auth, save, run, leaderboard, vocab, exam, assignment, teacherBanks, payment, shop, parent, family, presence, quiz, quizAutogen, …).
- `quiz/` — multiplayer quiz Phase 6: `quizWs.ts` (socket.io namespace), `sessionRegistry.ts` (lifecycle + crash recovery), `sessionRunner.ts` (in-memory live state), `recurringActivityCron.ts` (admin-managed daily activities), `autogenGenerators.ts` (AI-generated questions, three types).
- `lib/` — domain modules: prisma client, mailer + i18n templates, ECPay, Azure TTS / pronunciation, Gemini image gen, image moderation (Sightengine/Google Vision, fail-open), anti-cheat (`runAntiCheat`), stamina, mastery, subscription/expiry cron, progressEmail cron, classCode, referral, etc.
- `middleware/auth.ts` — single-device session policy: every login revokes all the user's other active sessions.
- `prisma/schema.prisma` — ~60 models. Migrations in `prisma/migrations/`.

**Admin (`admin/`)** — `server.js` is a thin Express app that proxies `/api/admin/*` to backend (auto-injecting `x-admin-key`) and serves static HTML pages from `admin/public/`. Pages are vanilla HTML + fetch.

## Encoding rules

Console default codepage on this box is 950 (Big5), but **PowerShell 7.6+ (`pwsh.exe`) is the canonical runner** for project scripts.

- **`.ps1` files: UTF-8 *no BOM***, executed via `pwsh.exe`. PS 7+ reads UTF-8 source by default so Traditional Chinese in comments / `Write-Host` works without BOM. Do NOT run them with `powershell.exe` (PS 5.1) — it parses .ps1 as the system ANSI codepage (cp950) and mojibakes the Chinese at parse time, before any `[Console]::OutputEncoding` line gets a chance to run. Write with `[System.IO.File]::WriteAllText($p, $c, (New-Object System.Text.UTF8Encoding $false))`. The two Scheduled Tasks (`EduRPGRestartStack`, `EduRPGRestartProdOnly`) are registered with `Execute = C:\Users\linus\AppData\Local\Microsoft\WindowsApps\pwsh.exe`; if you re-register, keep that path.
- **`.sql` migrations: UTF-8 *no BOM***. Postgres rejects BOM (`syntax error at or near "?"`) and `prisma migrate deploy` fails with P3018. Recovery requires `prisma migrate resolve --rolled-back <name>`. Write with `[System.IO.File]::WriteAllText($p, $c, (New-Object System.Text.UTF8Encoding $false))`.
- **`.ts` / `.tsx` / `.json` / `.md`: UTF-8 no BOM** (Vite/TS/Prisma defaults).

## Things that are easy to break

- **Don't `taskkill /F /IM node.exe`** — kills prod backend on 3020 too. Use `Stop-Process -Id <pid>` after locating via `Get-NetTCPConnection -LocalPort <port>`.
- **`ts-node-dev` and `npm` leave wrapper PIDs without listening ports.** They hold a lock on `backend/node_modules/.prisma/client/query_engine-windows.dll.node`. After `restart_dev_stack.ps1 -StopOnly`, always `Get-Process node` and kill stragglers before rebuilding (`prisma generate` will otherwise fail with EPERM).
- **PowerShell wrapper splat must use a hashtable, never `$args`** — `$args` is an automatic positional array and binds to `param()` order (e.g. `'-NoDb'` lands in `[int]$ProdFrontendPort`). Use `$h = @{ NoProd = $true; … }; & $inner @h`.
- **Azure provider `endpoint` field often stores a region code** (e.g. `'eastasia'`) not a URL. Adapters must build `https://${region}.{tts|stt}.speech.microsoft.com/...` themselves; don't pass the value into `fetch()` directly or it throws `TypeError: Invalid URL`.
- **Don't restart prod without confirmation** unless explicitly authorized for that turn. The standing-authorization phrases are listed in `Develop_MEMORY.md` §2.4. Frontend hot-reload / idempotent migrations are exempt.

## UI 慣例 — 按鈕

- **配色對比**：按鈕背景色與文字色必須有足夠對比，不可配色過於相近導致難以閱讀（例：淺灰底 + 白字、深藍底 + 黑字都不行）。
- **圓角外框**：所有按鈕一律做成圓角，不要使用直角外框。Phaser 用 `graphics.fillRoundedRect` / `strokeRoundedRect`，React/Tailwind 用 `rounded-*` class。
- 不要使用我Chrome裡面Oracle資料夾的bookmark裡面的URL。
- 使用chrome瀏覽器的時候, 請使用linus94213@gmail.com這個account。
## Reference docs in this repo

- `OPERATIONS.md` — full restart / deploy / recovery runbook.
- `Develop_MEMORY.md` — running log of decisions, bugs, env conventions.
- `BOT_HANDOFF.md` / `PROD_DEPLOY_QUEUE.md` — deploy-bot protocol.
- `PROJECT_STRUCTURE.md` — Phaser scene/system reference.
- `CHANGELOG.md` — version history (currently 0.3.83).
