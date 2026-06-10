# Cleo — Claude Code 專案備忘

> 給未來 session 的 Claude:這份檔案是「每次都要重複問的東西」的常駐答案。
> 不要重複實作或重新查證已記錄的事實,除非懷疑它過期(對照當前檔案再決定)。

---

## 1. Work area / 路徑

- **Repo root**:`/Users/linus/cleo`
- **Worktrees dir**(dispatcher 用):`/Users/linus/cleo/.worktrees`
- **PID / log dir**(start/stop 用):`/Users/linus/cleo/.cleo-run/{pids,logs}`
- **Main branch**:`main`
- **Monorepo 成員**:`apps/{api,bot,scheduler,worker,web,admin,mobile}`、`packages/{shared,providers}`
- **套件管理**:`uv` workspace(Python)+ `pnpm`(JS)。不要用 `pip` / `venv` / `npm`。

> **⚠️ 多 session 隔離(重要,踩過坑)**:主 checkout(`/Users/linus/cleo`)會被多個 session / dispatcher 同時用。**只要你要「工作或修改」(寫 code、改檔、跑 test、commit),先開自己的 worktree,不要直接在主 checkout 改、也不要在主 checkout 切分支** —— 否則會把別的 session 的檔案從腳下抽走(已發生:commit 落到別人分支、未追蹤 WIP 在 `git checkout` 後從工作目錄消失)。一鍵開隔離 worktree:
>
> ```bash
> ./scripts/new_session_worktree.sh feat/cle-NNN-slug          # 從 origin/main 開 .worktrees/<branch> 隔離分支
> ./scripts/new_session_worktree.sh feat/cle-NNN-slug main --sync  # 順便 uv sync(跑 pytest/mypy 前需要)
> ```
>
> 在它印出的路徑裡完成**所有**編輯 / commit / `git push -u`,完事再 `git worktree remove`。新 task 配號先 `./scripts/next_cle.sh reserve`(見 §3)。主 checkout 只留給「讀」或「快速、無風險的單檔 doc 改動」。

---

## 2. 服務啟動 / 停止順序

所有腳本在 `scripts/` 下,且設計成 idempotent(已啟動/已停止會跳過)。

### 一鍵重啟(最常用)

```bash
cd /Users/linus/cleo
./scripts/restartCleo.sh   # 內部依序呼叫 stop_cleo.sh → start_cleo.sh
```

### 單獨啟動 — `./scripts/start_cleo.sh`(順序固定,勿打亂)

1. **postgres**(`brew services start postgresql@XX`,自動偵測版本)
   - ⚠️ **PG16 是 system root LaunchDaemon**(`/Library/LaunchDaemons/homebrew.mxcl.postgresql@16.plist`),**且三專案共用**(cleo + edurpg + ai_stock_agent)。控制它要 `sudo`:`sudo brew services restart postgresql@16`;不加 sudo 只能看、不能管,還會誤報 `error 256` / `Running: false`。
   - `max_connections = 200`(2026-05-31 從 100 調高,寫進 `postgresql.auto.conf`)。cleo 用 **NullPool**(每查詢開關連線),重啟尖峰 + edurpg 用量曾衝破舊的 100 → `TooManyConnectionsError`。
   - **野生 postmaster 占 lock 的鬼故事**:若有人在 daemon 之外手動起 postmaster(`pg_ctl` / 非 sudo brew),它占住 data-dir lock → daemon 每 10s 啟動失敗噴 `lock file "postmaster.pid" already exists` + brew 顯示 error。解法:停掉野生 postmaster(`pg_ctl -D /opt/homebrew/var/postgresql@16 -m fast stop`)讓 daemon 起自己的乾淨 instance。詳見 memory `[[project_postgres_shared_root_launchdaemon]]`。
2. **redis**(`redis-server --daemonize yes`)
3. **alembic migrations**(`cd packages/shared && uv run alembic upgrade heads`)
   - 失敗會 **`exit 1`**(CLE-444),不會繼續啟動下游服務。stderr 會印 alembic tail + `alembic current` / `alembic heads` + 修法指令。
   - `CLEO_RUN_MIGRATIONS=0` 跳過 migration,但之後的 `verify_schema_current` 還是會檔 stale schema。
   - `CLEO_ALLOW_STALE_SCHEMA=1` 跳過 migration 失敗保護 + drift 檢查,允許 DB schema 落後 head 還是啟動 api/worker/bot。**只在你確定後續業務不會因 schema mismatch 噴 UndefinedColumnError 才用**(例如要 boot 起來看 log 排查 botched migration)。預設 `0`。
4. **cloudflared tunnel**(LaunchDaemon `system/com.cloudflare.cloudflared`)
5. **cleo-web**(pm2 → `ecosystem.config.js`,Next.js on `:3000`)
6. **api**(`scripts/run-api.sh` → `python -m cleo_api.main`)— 等 `/healthz` 健康後才繼續
7. **worker**(`scripts/run-worker.sh` → `python -m cleo_worker.main`)
8. **workflow fire-loop**(pm2 → `cleo-worker-fire-loop`,`scripts/run-worker-fire-loop.sh` → `python -m cleo_worker.main --mode workflow-fire-loop`)
   — 必須在 worker (RQ) 跟 scheduler 之間,因為 scheduler 會排 fire 進 Redis ZSET `workflow:scheduled`,fire-loop 要把它 pop 出來。pm2 管理 → 進程意外掛掉 5 秒內 auto-restart。
9. **scheduler**(`scripts/run-scheduler.sh` → `python -m cleo_scheduler.main`)
10. **bot**(`scripts/run-bot.sh` → `python -m cleo_bot.main`)
11. **public tunnel 健康檢查**(`CLEO_PUBLIC_URL/healthz`、`WEB_PUBLIC_URL/`)

可用 env 跳過個別步驟:`CLEO_START_WEB=0`、`CLEO_START_WORKER=0`、`CLEO_START_FIRE_LOOP=0`、`CLEO_START_SCHEDULER=0`、`CLEO_START_TUNNEL=0`。

### 單獨停止 — `./scripts/stop_cleo.sh`(反向順序)

1. bot → scheduler → fire-loop(pm2)→ worker → api(python 服務 SIGTERM,20 次 0.5s 內未停才 SIGKILL;fire-loop 走 `pm2 stop`)
2. cleo-web(`pm2 stop cleo-web`)
3. redis(`redis-cli shutdown nosave`)
4. cloudflared(`launchctl bootout`)
5. **postgres 預設不停**;要停得設 `CLEO_STOP_POSTGRES=1`

`CLEO_STOP_FIRE_LOOP=0` 可在 stop 階段保留 fire-loop(api/bot redeploy 不影響 fire-loop)。

### 個別重啟 cleo-web 的小提醒

Next.js 啟動時把 `apps/web/public/` cache 起來,新增靜態檔要 `pm2 restart cleo-web` 才看得到(`[[project_nextjs_public_dir_cache]]`)。

**新增 / 改動 `apps/web/app/` 下的 route 或 page**:pm2 跑的是 `next start`(production),只服務預先 build 好的 `.next/`。`pm2 restart cleo-web` 只是重啟 node process,**不會重編譯** — 新 route 會 404。pull / merge 後若 `apps/web/` 有改動,必須:

```bash
cd /Users/linus/cleo/apps/web && pnpm build && pm2 restart cleo-web
```

判斷是否需要 rebuild:比對 `apps/web/.next/BUILD_ID` 的 mtime 與 `git log -1 --format=%cd apps/web/` 的最新 commit 時間。

---

## 3. Dispatcher(自動派工)

主腳本:`tools/agent_dispatch_cleo_mac.ps1`(PowerShell 7+,macOS)

### Pending tasks 目錄

```
tasks/pending/CODE/      # 給 Claude Code 的任務(複雜邏輯、跨檔案重構)
tasks/pending/Copilot/   # 給 GitHub Copilot 的任務(CRUD、scaffold、單元測試)
tasks/pending/Codex/     # 給 codex CLI 的任務
tasks/pending/Human/     # 人類手動執行(不走 dispatcher)
tasks/done/              # 完成後 dispatcher 自動搬過來,並附 .result.md
tasks/logs/              # 每次執行 log;.stop-requested 為軟停止旗標
```

### 配號(避免多 session 撞號)

多個 session / dispatcher 共用這個工作目錄,各自「自己算 max CLE 號」會搶到同號(踩過 CLE-478 stock vs ecpay)。**開新 task 前一律用 allocator,不要手猜 max**:

```bash
./scripts/next_cle.sh reserve "stock watchlist"   # 原子配號 + 占號,印出 CLE-NNN
./scripts/next_cle.sh hold CLE-512 "label"         # 占用「指定號」(已被占就失敗)
./scripts/next_cle.sh peek                         # 只看下一個號,不占
./scripts/next_cle.sh check                        # 掃 tasks/ 有沒有 exact-id 撞號(CI/排查用)
```

占號標記 + 鎖放在**共用的 git common dir**(`<git-common>/cle-alloc/`),所以**跨所有 worktree 都看得到**(不是各 worktree 私有);spec 還沒寫就先卡號。用 `mkdir` 原子鎖序列化。掃號來源涵蓋 pending/done 檔名、共用占號標記、`git log --all` / branches(任何分支已 commit 的號都算)。`CLE-425` 與 `CLE-425a`(umbrella vs 子任務)視為不同 id;`CLE-NNN.result.md` 不算重複。

**gen-spec 自動配號**:`scripts/generate-spec.sh` 已接上 allocator —— outline 區段用 `## CLE-AUTO`(或 `CLE-NEW`)當佔位,gen-spec 會呼叫 `next_cle.sh reserve` 配真號再生成;明確指定的號則在寫檔前 `hold` 卡住(撞號直接 abort 並提示用 `peek`/`CLE-AUTO`)。所以批次 / 自動派工天生不撞號。

### Loop 模式(常用三條)

```bash
pwsh ./tools/agent_dispatch_cleo_mac.ps1 -Loop -Agent CODE
pwsh ./tools/agent_dispatch_cleo_mac.ps1 -Loop -Agent Copilot
pwsh ./tools/agent_dispatch_cleo_mac.ps1 -Loop -Agent Codex
```

Loop 行為:每個任務完成後檢查 `tasks/logs/.stop-requested`,有就乾淨退出。

### 軟停止

```bash
touch /Users/linus/cleo/tasks/logs/.stop-requested
```

### 常用 flags

- `-TaskId CLE-NNN`:指定單一任務
- `-DryRun`:只印計畫,不開 worktree、不叫 agent
- `-NoMerge`:跑完 agent + lint + test,停在 merge 前(保留 worktree)
- `-SkipPush`:跳過 `git push origin main`
- `-SkipCoverageGate`:跳過 pytest coverage gate(70% 門檻)
- `-SkipAutoRecovery`:關閉失敗時自動呼叫 fix agent
- `-FlushPendingPush`:不派工,只收尾(CLE-512)——掃 `<git-common>/pending-push/` marker → reconcile origin →(reconcile 有帶新 commit 才)跑完整 gate → push → 清 marker(bounded retry,永不 `--force`)。批次跑完呼叫一次。

### Pipeline 順序(每個任務)

`sync → uv sync → ruff(阻擋)→ mypy(阻擋)→ pytest(含 coverage gate)→ merge → promote(spec 從 pending 搬到 done + 附 .result.md)→ push`

### Pre-dispatch reconcile + pending-push marker / finalizer(CLE-512,確定性版)

多 session 並行時 dispatcher 從**本地 main** 開分支、push 也只推本地 main,origin 被別人推進 → 本地 main 落後 → `8b` push non-fast-forward 被拒、divergence 靜默累積。CLE-512 用純確定性 shell 收掉(方案 B + finalizer,設計見 `docs/dispatcher-merge-recovery-research.md` §3/§5.1/§6;方案 C / Q4 未做):

- **每棒開工前自動 pre-dispatch reconcile**:建 worktree 前、在 `MERGE.lock` 下 `git fetch origin`,若 `origin/main` ahead 就 `git merge --no-edit origin/main` 進本地 main(於是 `8b` push 是 fast-forward、順手把前一棒 pending 的 commit 帶上 origin)。**衝突 → `git merge --abort`(絕不留 `MERGE_HEAD`)+ 停 + 印訊息等人**,不建 worktree、不叫 agent。`-Loop` 每棒自然受惠;`-NoMerge` / `-DryRun` 跳過。
- **`8b` push 失敗寫 marker**:push 失敗不再只 `[warn]`,改在 **`<git-common>/pending-push/<TaskId>`**(= `.git/pending-push/`,跟 `.git/cle-alloc`、`MERGE.lock` 同套「跨 worktree、不進工作樹」慣例;**不可**放 `tasks/logs/.pending-push`——沒被 .gitignore 忽略、會自擋下一棒)寫一個 marker(含 timestamp + branch + HEAD sha + 說明)。**任何一次 push 成功會清掉所有 marker**(成功 push 會把先前 pending 的 commit 一起帶上 origin)。
- **沒有下一棒就用 `-FlushPendingPush` 收尾**:`pwsh ./tools/agent_dispatch_cleo_mac.ps1 -FlushPendingPush` → 有 marker 才動:reconcile origin →(reconcile **有帶進新 commit** 才跑 §5.1 預設完整 gate:`ruff` + `mypy` + `pytest`,coverage 關;reconcile no-op 就跳過——pending commit 早被各自 task gate 過)→ push → 清 marker(push race bounded retry ≤3,**永不 `--force`**)。無 marker = no-op、exit 0。
- ⚠️ 改的是 dispatcher 自身、動到共用 main 的 merge/push 路徑,Python gate 驗不到 `.ps1`。改 dispatcher 後**務必 `-NoMerge` 派工 + 人工跑 smoke** 再 merge;且改動只在「下一次 dispatcher 啟動」生效(正在跑的用舊碼)。

### 跨專案安全

Dispatcher 內建 guard:`REPO_ROOT` / `WORKTREES_DIR` 路徑含 `aistock` / `edurpg` 會拒跑;shell 裡有 `AISTOCK_*` / `EDURPG_*` 環境變數也會拒跑。**絕對不要**在同一個 shell 同時設多個專案的 env。

### Dispatcher 中斷 / 衝突的善後(CLE-487~496 批次跑出來的 SOP)

多 session 並行 + 串接一長串任務時,dispatcher 常見四種「沒跑完」狀況。**先判斷 work 在哪一步停的,再對症處理,不要重頭硬幹**:

1. **Dispatcher 半路死掉(踩過:CLE-487 卡在 pytest gate)** — 進程消失、`/tmp/CLE-NNN.log` 凍住、spec 還在 `pending/`。先進 worktree 確認 work 已 commit 且 `git status` 乾淨(`feat/cle-NNN-*` 上有 feature commit),**直接重跑 `pwsh ./tools/agent_dispatch_cleo_mac.ps1 -TaskId CLE-NNN`**:它會偵測既有 worktree(不重建)、重跑 agent(work 已在 → 近乎 no-op)、再走真正的 gate → merge → promote → push。不要手動 merge 繞過 gate。

2. **Push 被拒(non-fast-forward)** — dispatcher 只 `push` **本地** main、**不會主動 fetch/merge origin**;其他 session 一直推進 origin,所以本地 main 會落後 → push 失敗,task 會「merge 進本地 + promote,但沒上 origin」。善後:`git fetch origin && git merge --no-edit origin/main`(純本地操作,允許),把本地 main 拉成 fast-forward,**下一個任務的 dispatcher push 會把它一起帶上 origin**(`git rev-list --left-right --count origin/main...main` 看 `0 0` 才算同步)。
   - 串接多任務時,**在每個 task dispatch 前先 reconcile 一次** origin/main,push 幾乎不會再失敗。driver 範本見下。
   - ⚠️ **不能自己 `git push origin main`** — auto-mode classifier 會擋預設分支直推。但 dispatcher 內部那一步(整支 script 在一個 Bash call 裡跑)不受影響,**push 一律交給 dispatcher 代勞**。

3. **Merge 衝突(step 7b,踩過:CLE-494 撞 CLE-477)** — dispatcher 在 `REPO_ROOT` 做 merge,衝突會留下 `.git/MERGE_HEAD` + 保留 worktree + exit。`apps/bot/cleo_bot/main.py`、`packages/shared/cleo_shared/workflow/action_registry.py`、`workflow/recipes/__init__.py`、`actions/__init__.py` 這些**註冊樞紐是衝突熱點**(每個新 flow 都往同一處接線,見 `[[feedback_task_must_be_wired_in]]`)。善後:`git diff --diff-filter=U` 找衝突檔 → **雙方註冊/import 都保留**(這類衝突幾乎都是「各加各的」,不是二選一)→ ruff + `py_compile` + pre-commit hook 驗證 → `git add` + `git commit --no-edit` 補完 merge commit。

4. **補 promote + push** — 上面 1/3 手動補完 merge 後,branch tip 已在 main 裡。**再重跑一次 `-TaskId CLE-NNN`**:dispatcher 偵測「branch already merged into main」→ **idempotent 短路**(跳過 agent + 跳過 gate)→ 只做 promote(spec 搬 done + `.result.md`)+ push。這是把「手動 merge 完的 task」正式收尾的乾淨方式。

**串接多任務的 driver 範本**(放 `/tmp`,**不要放 repo 內** — dispatcher 對 dirty `REPO_ROOT` 會拒跑;我第一版把 driver 丟 `tasks/logs/` 直接害下一個 task 起不來):

```bash
# /tmp/cleo-driver/run_chain.sh — 串行、每任務前 reconcile origin、撞失敗就停
for n in 488 489 490 ...; do
  # 等 MERGE.lock 放掉再 reconcile,避免跟別的 dispatcher 撞 merge
  while [ -e .worktrees/MERGE.lock ]; do sleep 10; done
  git fetch origin --quiet
  git rev-list main..origin/main | grep -q . && git merge --no-edit origin/main
  [ -n "$(git status --porcelain)" ] && { echo "dirty root, abort"; exit 1; }
  pwsh ./tools/agent_dispatch_cleo_mac.ps1 -TaskId "CLE-$n" -SkipCoverageGate \
      > "/tmp/cleo-driver/CLE-$n.log" 2>&1
  ls tasks/done/ | grep -q "CLE-$n" || { echo "CLE-$n 沒 promote,停"; exit 1; }
done

# 收尾(CLE-512):dispatcher 每棒已自動 pre-reconcile,上面手動 fetch/merge 兩行其實可省;
# 但批次跑完要呼叫一次 finalizer,把最後一棒沒推上 origin 的 commit 帶上去:
pwsh ./tools/agent_dispatch_cleo_mac.ps1 -FlushPendingPush
```

有依賴鏈的任務(如 ecpay 488→496)**一定串行 + 撞失敗就停**(後面的 build 在前面之上);彼此獨立的才考慮並行。一次只跑「自己這串」一個 dispatcher,別跟別 session 的 task 搶同一份共用 toolchain 把 Mac mini 跑爆。**別人 session 正在跑的 task(看 `ps aux | grep agent_dispatch` 的 `-TaskId`)不要碰**。

---

## 4. DB / 服務連線

連線字串都在 `/Users/linus/cleo/.env`(已 gitignore),這裡只記結構:

| Service | URL / 形式 | 備註 |
|---|---|---|
| Postgres | `postgresql+asyncpg://cleo:<see .env>@localhost:5432/cleo` | local Homebrew,user `cleo`,db `cleo` |
| Redis | `redis://localhost:6379/0` | local Homebrew |
| Core API | `http://127.0.0.1:8000` | `cleo_api.main`,健康檢查 `/healthz` |
| Public tunnel (API) | `https://cleo.edurpg.org` | cloudflared LaunchDaemon |
| Web (Next.js) | `http://localhost:3000` | pm2 `cleo-web` |
| Public tunnel (Web) | `https://onboard.edurpg.org` | cloudflared → `:3000` |
| Schema 來源 | `cleo_schema_v0.3.sql`(canonical)+ `packages/shared` 內 Alembic | 改 schema 要建 migration,不要直接動 sql |

Alembic 操作:

```bash
cd /Users/linus/cleo/packages/shared
DATABASE_URL=postgresql+asyncpg://cleo:<pw>@localhost:5432/cleo uv run alembic upgrade heads
# 新 migration:uv run alembic revision --autogenerate -m "..."
```

`.env` 優先序:`.env.production` 覆蓋 `.env`(start_cleo.sh 跑 migration 時會自己 merge)。

---

## 5. 其他常用入口

- 部署 web:`tools/deploy-web.sh`
- 一次性查詢(如資料補洞):`scripts/backfill_*.py`、`scripts/migrate_*.py`
- Discord bot 不要 hot-reload,改完跑 `./scripts/restartCleo.sh` 比較穩
- 健康檢查:`curl http://127.0.0.1:8000/healthz`、`pm2 status`、`redis-cli ping`、`psql -d cleo -c '\dt' | head`

---

## 6. 慣例提醒(過去 session 留下的)

- **Cleo 是建議者,不是執行者** — 不要設計自動改寄信、自動改行事曆給其他與會者的功能。詳見 memory `[[feedback_cleo_advisor_not_executor]]`。
- **UX 優於 DB 約束** — 遇到使用者可恢復的衝突,先設計復原 UX,不要直接加 UNIQUE / CHECK。詳見 memory `[[feedback_ux_over_db_constraint]]`。
- **Microsoft `Mail.Read` 暫時關閉** — Publisher Verification 完成前不要 re-enable。詳見 memory `[[project_microsoft_publisher_verification]]`。
- **新 user-facing flow 一律走 WorkflowRecipe,不要再手刻 handler + cog** — Cleo 有兩套 workflow 子系統(WorkflowEngine 與 ConversationEngine + WorkflowRecipe),完整架構、決策樹、節點 DSL、可用 actions、自然語言 → workflow 轉化流程都在 **`docs/workflow-architecture.md`**。設計新 flow 之前先讀那份。
- **Task 開發完一定要「接上」使用者路徑才算完成** — 寫完 cog / recipe / handler / service 但忘了 wire-in(沒 register 進 bot loader、沒加進 `INTENT_TO_RECIPE`、scope_classifier prompt 沒新 intent、API route 沒掛到 router…)是 Cleo 反覆踩到的鬼故事:功能 code 寫好放著,真正用的時候才發現跑不到。**Task spec 一定要列「接線位置 checklist」、Acceptance 一定要含「使用者實際從 Discord/Web 觸發後,新邏輯有跑」的 smoke test**,不只是 unit test 綠燈就結案。詳見 memory `[[feedback_task_must_be_wired_in]]`。

---

## 7. 會議錄音 pipeline(從 Discord attachment → 會議紀錄)

短語音指令(<60s)經本地 WhisperX 轉逐字稿後走同一條 recipe;長語音仍走 meeting pipeline。

完整呼叫鏈:

1. **Bot 接收 attachment**:`apps/bot/cleo_bot/cogs/meeting_audio.py`(`MeetingAudioCog`)
   - 收 attachment → server-side 讀 payload → `self._audio_storage.upload_local()` 直傳 S3 → 建 `MeetingRecording` row(status=`awaiting_confirmation`)— line 174-268, 381-427
   - Step 2 modal 確認日期/名稱 → `repo.confirm_naming()` (line 562-583)
   - status 改 `queued` → enqueue `cleo_worker.tasks.process_meeting_audio.process_meeting_audio(meeting_recording_id)` (line 579-582)

2. **API upload endpoints**(給 web/外部 client 用,bot 沒走這條):`apps/api/cleo_api/routes/meeting_audio.py`
   - `POST /api/meetings/upload` (line 115) — 發 presigned S3 POST grant(`UploadPostGrant`,`method=POST`,`url`=presigned S3,`fields`=含 SSE headers),建 `MeetingRecording` row reserve
   - `POST /api/meetings/{id}/upload-complete` (line 186) — `storage_service.head()` 驗證 size + content-type + magic bytes → status=`pending`
   - `MAX_UPLOAD_SIZE_BYTES = 500MB`、grant TTL 15 分鐘、confirmation TTL 7 天
   - **注意**:目前只支援 session/cookie 認證(`CurrentUserDep`),沒有 token-based(若要從 Discord 一次性連結進來,需要新加 token 認證層)

3. **Storage**:`packages/shared/cleo_shared/services/audio_storage.py`
   - 只有 S3 / R2 兩種 S3-compatible backend(無 local fs)
   - `audio_blob_url` 格式 = `s3://bucket/key`(e.g. `s3://meeting-audio-bucket/raw/{user_id}/{recording_id}.mp3`)
   - `get_audio_storage()` 從 settings 讀 `cleo_object_store_backend`(預設 `s3`),env `CLEO_OBJECT_STORE_BACKEND` 切換
   - **檔案直傳 S3 → 不過 cloudflare tunnel → 沒有 100MB body 限制**

4. **Worker STT**:`apps/worker/cleo_worker/tasks/process_meeting_audio.py:129` (`process_meeting_audio`)
   - probe metadata → trim silence → STT(line 237-243,根據 `meeting_backend_used` 走 AssemblyAI 或 WhisperX+Ollama)→ status=`completed`
   - 最後 enqueue `notify_meeting_complete(recording_id)` (line 322-323)

5. **STT 後通知**:`apps/worker/cleo_worker/tasks/notify_meeting_complete.py:65`
   - POST 到內部 endpoint `/internal/send-meeting-summary-dm`(`apps/api/cleo_api/routes/internal/__init__.py:349`)
   - 摘要 + recording_id enqueue 到 Redis outbox → bot dispatcher 拾起 → `MeetingArtifactsCog.send_summary_dm()`(`apps/bot/cleo_bot/cogs/meeting_artifact_handlers.py:169`)

6. **會議紀錄(lazy / on-demand)**:user 在 DM 點「產生會議紀錄」按鈕 → `MeetingArtifactsCog.handle_button()` (line 138-148) → extraction service → 寫 `MeetingArtifact`(artifact_kind=`minutes`)

**LLM backend 分層(CLE-449 / CLE-450)**:會議產製是兩段式,各自綁不同 `AnthropicWrapper`:

- **Stage-1 transcript correction**(glossary 套用 + STT 修錯 + 講者標註)
  → **永遠走 Anthropic Sonnet**,不受 `CLEO_MEETING_LLM_BACKEND` / `CLEO_LLM_FORCE_BACKEND` 影響。
  原因:qwen3:8b 無法穩定吐 `TranscriptCorrection` 的 strict-JSON schema,
  一旦切到 Ollama 整條 stage-2 都會 fallback 到 raw transcript。
  Wrapper 在 `apps/bot/cleo_bot/main.py` 用 `force_backend="anthropic"` hard-lock。
- **Stage-2 artifact extraction**(`minutes` / `summary` / `follow_up` / `next_steps`)
  → 依 `CLEO_MEETING_LLM_BACKEND=ANTHROPIC_API`(預設)、`OLLAMA`,或 `DEEPSEEK_API`(CLE-450)切換。
  Boot 時固定,要改要 `restartCleo.sh`(不是 hot-reload)。

`CLEO_LLM_FORCE_BACKEND=ollama|deepseek_api` 是 CLE-430 / CLE-450 留下的全域逃生口,影響
非會議的 LLM 呼叫(chitchat、scope classifier 等)。Meeting 兩條 wrapper 因為都明確設了
`force_backend`,不會被它劫持。

CLE-464 把以下 12 個 shared service / route 從 raw `AsyncAnthropic` 遷移到
`AnthropicWrapper`(都不傳 `force_backend=`,直接繼承 `CLEO_LLM_FORCE_BACKEND`),
所以一鍵切 DeepSeek 也涵蓋它們:OAuth welcome(`api/routes/oauth`)、morning briefing
DI singleton(`worker/morning_briefing`)、weak-subject scan(`scheduler/weak_subject_scan`)、
worker LLM singleton(`worker/app`)、RestaurantService、TripDetector、TripPrepService、
WeeklyReviewGenerator、InboxSummarizer、ActionItemDetector(`shared/action_item_detector`)、
action item dispatch RQ task(`shared/tasks/action_item_detection`)、MarketplaceSafetyChecker
+ 其 `api/deps:get_safety_checker` 注入點。Email draft / Vision OCR / OpenAI 站點刻意
留在外,看 CLE-464 spec 的 out-of-scope 列表。

**DeepSeek backend(CLE-450)**:DeepSeek 在 `https://api.deepseek.com/anthropic` 提供
Anthropic-compatible endpoint,所以走同一個 `AsyncAnthropic` SDK,只是 `base_url` 換掉。
- 需要 `DEEPSEEK_API_KEY=sk-...` 在 `.env`。沒有 key 但又設了 `DEEPSEEK_API` 時,wrapper
  會印 `anthropic_wrapper.deepseek_force_without_api_key` error 並 fallback 回 Anthropic,
  讓 bot 還是能 boot 起來。
- 自動 model mapping:`claude-haiku-*` / `claude-sonnet-*` → `deepseek-v4-flash`,
  `claude-opus-*` → `deepseek-v4-pro`。所以 caller 不需要改 model 名稱。
- **`cache_control` 不支援**,wrapper 在送到 DeepSeek 前用 `_strip_cache_control()` 把
  欄位拔掉(原本傳進來的 messages list 不會被改動)。System prompt 也跳過
  `_build_system()` 的 ephemeral-cache wrapping。
- Thinking mode 預設 ON,wrapper 用 `extra_body={"thinking": {"type": "disabled"}}` 關掉。
- 成本暫時記成 `cost_usd=0.0`(`_PRICING` 表還沒有 deepseek-v4 entry);
  `external_api_calls` 還是有 token counts 可以後補。

**資料模型**:`packages/shared/cleo_shared/db/meeting_recordings.py`
- `MeetingRecording` (line 79-171):`audio_blob_url`、`transcript_blob_url`、`status`(`awaiting_confirmation` → `pending` → `queued` → `preprocessing` → `transcribing` → `completed` / `failed` / `cancelled`)、`backend_used`
- `MeetingArtifact` (line 173-230):`artifact_kind`(`minutes`/`summary`/`recap`/`follow_up`/`next_steps`)、`content_blob_url`(Word doc S3 URL)、`content_text_cipher`(加密文本)

> ⚠️ **新增 artifact kind 要同步改「四處」,漏一處就靜默炸(CLE-624 踩過)**:`recap`(會議總結)當初只加進
> `MeetingArtifactKind` Literal + DM 按鈕 + cog 顯示表 + DB CheckConstraint,**獨漏 runtime 白名單 tuple**
> `MEETING_ARTIFACT_KINDS`(同檔 `db/meeting_recordings.py`)。結果每次點「會議總結」,`repo.get_artifact()`
> 第一行 `_validate_artifact_kind()` 就丟 `ValueError: invalid meeting artifact kind` —— **連 LLM 都沒呼叫到**,
> 使用者只看到泛用「請稍後再試」DM(reason=None)。加新 kind 的 checklist:(1) `MeetingArtifactKind` Literal
> (2) `MEETING_ARTIFACT_KINDS` tuple (3) DB CheckConstraint (`MeetingArtifact.__table_args__`)
> (4) cog `_KIND_DISPLAY_ZH` / `_KIND_INTRO_EMOJI`。回歸測試
> `test_meeting_recordings_repo.py::test_migration_and_model_omit_raw_transcript_column` 現在用
> `get_args(MeetingArtifactKind)` 動態比對 tuple,tuple 與 Literal 一 drift 就紅燈 —— 別把它改回寫死的集合。
> 又一個 `[[feedback_task_must_be_wired_in]]` 變種:同一概念散在多個註冊點,unit test 綠燈不等於接好。

---

## 8. Magic-link 模式(Discord → 臨時 web 頁,寫新功能可抄這套)

既有實作 = location 確認頁(`/share-location` 指令):

- **入口**:`apps/bot/cleo_bot/cogs/location.py:31-47`(`/share-location`)→ core API `create_location_share_link()`
- **URL**:`{web_public_url}/location/confirm?token=<urlsafe_token>`,預設 `https://onboard.edurpg.org/...`(`packages/shared/cleo_shared/config/settings.py:259-260`)
- **走 cloudflare tunnel → cleo-web Next.js**(不是直連 API)
- **Token 設計**(`apps/api/cleo_api/services/location_sharing.py:115-158`、`apps/api/cleo_api/routes/location.py:113-206`):
  - `secrets.token_urlsafe(32)` 產生(256-bit entropy)
  - DB 存 SHA256 hash(`hash_token()`),不存明文 — table `user_location_share_tokens`
  - TTL by `location_share_token_ttl_seconds`,**一次性**(用後標 `confirmed`/`declined`/`used_at`)
  - Rate limit:每 user 10 分鐘最多 5 個 pending(`count_recent_pending_tokens()`)
- **前端**:`apps/web/app/location/confirm/page.tsx` + `apps/web/components/location/LocationConsentPage.tsx`,token 從 URL query 讀,**無 cookie / 無 OAuth**
- **回寫**:Next.js proxy POST → API `/location/confirm`,API 驗 token → 寫 `current_locations` → 標 token used → enqueue Redis `oauth_dm_queue` → bot 推 DM

**安全要點**:整條走 tunnel + HTTPS,API/web 本機不對外。攻擊面只剩 token 漏出去(任何 magic-link 共通天花板),已被短 TTL + 一次性 + rate limit 壓住。新功能(如檔案上傳臨時頁)直接抄這套即可,**不要做直連方案**。

---

## 9. 打招呼走 greeting recipe(CLE-466)

純打招呼(「Hi Dolly」「嗨」「早安」)**不打 API token**:scope_classifier 判成 `greeting`
→ `INTENT_TO_RECIPE["greeting"]` → `greeting` WorkflowRecipe(`packages/shared/cleo_shared/workflow/recipes/greeting.py`)
→ 單一 action `greeting.fetch_random` 從 `persona_greeting_pool` 表 `ORDER BY random()` 抽一句
→ 替換 `{nickname}` / `{persona_name}` → DM 回去。runtime **0 個 LLM call**。

招呼句子是**離線**用 local Ollama(qwen3:8b)按 persona variant 預生成的(每個 variant 10 句互異):
- 生成 task:`apps/worker/cleo_worker/tasks/regenerate_persona_greetings.py`
  (`AnthropicWrapper(force_backend="ollama")`,dedup ratio 0.8 + retry 2 次,失敗不刪舊池)。
- 手動觸發:`POST /admin/personas/variants/{variant_id}/regenerate-greetings`(admin token)。
- 自動補空池:scheduler job `refill_empty_greeting_pools`(每日 06:30 UTC)。
- Day-1 保底:migration `20260529010000_persona_greeting_pool` 對每個 variant seed 5 句
  (`generated_by='seed'`),所以即使 Ollama 沒跑、recipe 也抽得到。

**不**用 DeepSeek 生成(目的就是避免 API token);生成失敗時 wrapper 內建的 Anthropic
fallback 只是 JSON 安全網,不影響 runtime 抽句子那條路。

---

## 10. Stripe 訂閱 / plan_tier 降級(PR #17)

persona quota tier(`users.plan_tier`,A/B/C/D)的升降級**完全靠 Stripe webhook 事件驅動**:
本地**沒有**到期日欄位、scheduler 也**沒有**掃描訂閱到期的 job。webhook handler 在
`apps/api/cleo_api/routes/web.py` 的 `POST /web/stripe-webhook`,目前處理 4 種事件:

- `customer.subscription.created` → 從 metadata / price items 寫入 `plan_tier` + `stripe_subscription_id`
- `customer.subscription.updated` → `unpaid`/`canceled`/`incomplete_expired` 降回 `A`;
  `active`/`trialing` 依 live price items 同步 tier(支援 portal 換方案);
  `past_due`/`paused`/`incomplete` 屬寬限期,**不動 tier**
- `customer.subscription.deleted` → 降回 `A`、清空 `stripe_subscription_id`
- `invoice.payment_failed` → **不降級**(交給後續 unpaid/deleted),只 DM 提醒使用者更新卡片

⚠️ **必做的 dashboard 接線(code 之外)**:Stripe webhook endpoint 一定要**勾選訂閱**
`customer.subscription.updated` 與 `invoice.payment_failed`(**test + live 兩邊都要**)。
沒勾 → Stripe 不會把這兩個事件推過來 → 上面的 handler 形同不存在,「卡扣失敗白嫖高階 tier」
的缺口就還開著。重建 / 換 webhook endpoint 後務必重新確認這份事件清單。

### ⚠️ 綠界(ECPay)台灣金流改走「本地到期日 + scheduler 降級」(CLE-492)

上面那段「Stripe 推事件降級」**只適用 Stripe(國際,未啟用)**。台灣實際運作的綠界
**不推**「訂閱掉了」事件(只在每期扣款那刻通知),所以 ECPay 路徑改成**本地自己記到期日 +
scheduler 每日掃描降級**(spec `docs/ecpay-billing-spec.md` §1.5/§5):

- 到期日記在 `users.plan_paid_until`(+ `payment_orders.paid_until`),每期 `PeriodReturnURL`
  扣款成功就往後推一個週期(CLE-491)。
- `apps/scheduler/cleo_scheduler/jobs/subscription_lifecycle.py` 三個每日 job(錯開、避開
  06:30 greeting refill):
  - `expire_lapsed_subscriptions`(04:00 UTC)— `plan_paid_until` 過寬限期
    (`settings.ecpay_grace_period_days`,預設 3 天)且 tier≠A → 降回 `A`、清
    `ecpay_period_gwsr`、order 標 `cancelled`、DM 通知。
  - `process_period_end_changes`(04:30 UTC)— 到期的 `cancel_at_period_end` /
    `pending_change`:**先呼叫綠界停用 API 確認成功**才動本地(停用失敗 → 不降級、告警、隔天重試),
    換約則停舊 + 依 `pending_change` 建新定期定額。
  - `reconcile_subscriptions`(05:00 UTC)— 用綠界定期定額查詢 API 比對實際扣款次數,補漏收的
    period(負擔與扣款事件成正比,年繳幾乎不跑)。
- 綠界停用 / 查詢 / 建約 client:`apps/scheduler/cleo_scheduler/services/ecpay_periodic.py`
  (`EcpayPeriodicClient`,簽章重用 `cleo_api.services.ecpay`)。**鐵則**:取消 / 換約一定要真的
  呼叫綠界停用 API,光改本地旗標下期照扣 = 反向客訴(spec §3.6)。

---

## 11. 自然語言建立行程 / 會議 / 提醒(CLE-475)

「幫我新增今晚 18:30 的聚餐」「排禮拜三下午跟客戶開會」這類**用自然語言建立**的需求,走
`schedule_create` WorkflowRecipe(`packages/shared/cleo_shared/workflow/recipes/schedule_create.py`),
**不要**再手刻 handler。流程:

scope_classifier 判 `schedule_create`(與 `calendar_query` 的消歧靠
`prompts/scope_classification.py` 的 few-shot:時間+事件拿掉後剩「新增/排/安排」→ 建立,
剩「有什麼/查/看」→ 查詢)→ `recipe_launcher.pre_extract_slots` 用 Haiku 抽
`item_type/title/starts_at/location/people`(+ 衍生 `when_display`/`people_display`)→
recipe 用 `use_pre_extracted` + `source_slot` 直達 **`confirmation_card`**(CLE-473)→
確認後 `route_create` 依 `item_type` 分流:`event`/`meeting` 走 **`calendar.create_local_draft`**
action(寫 `user_draft_events` 本地草稿,CLE-474,**不 push 外部、不發邀請**),`reminder` 走
既有 `reminder.create`(one-off)。

要點:
- 確認卡「對象」欄位固定加「(僅記錄,不會代你通知)」顯性化 advisor 邊界
  ([[feedback_cleo_advisor_not_executor]])。
- 低信心防胡謅:confidence 落在 `[0.3, 0.6)` 且 intent=`schedule_create` →
  `recipe_launcher` 回**確定性 clarify 文案**(`launched=True`,無 LLM),不 fall through 到
  free-chat;`< 0.3` 才真的 fall through。
- persona 兩道禁則(`persona_engine.SERVICE_BOUNDARY_PROMPT` + `_AGE_FEEL_RULES_ZH`)擋掉自由
  回覆裡「我來幫你新增…/已經幫你排好」與任務回覆夾「真的耶」。
- `calendar.create_local_draft` **無 capability guard**,冪等鍵 = launcher 種進
  `trigger_context` 的 `workflow_run_id`(同一 session 連點 ✅ 只建一筆)。**不要**動既有
  `calendar.create_draft_event`(那條留給未來 push 外部)。

---

## 12. 查行程走 calendar_today_recap recipe(CLE-476,CLE-556 改日期感知)

「今天有什麼行程?」「明天有什麼?」「禮拜三有什麼?」仍由 scope_classifier 判
`calendar_query`,但 `INTENT_TO_RECIPE["calendar_query"] = "calendar_today_recap"` 後,
launcher 會先看 `CLEO_CALENDAR_QUERY_USE_RECIPE`。flag 預設 **off** 時直接 fall through 舊
`CalendarQueryHandler`;flag on 才啟動 `calendar_today_recap` recipe,方便 1 秒回退。

**日期感知(CLE-556)**:recipe 不再寫死今天。launcher 的 `pre_extract_slots` 對
`calendar_query` 重用既有 `CalendarIntentClassifier`(`apps/api/cleo_api/services/calendar_intent.py`)
抽目標日,塞 `recap_target_{kind,date,label}` 進 trigger_context 的 `pre_extracted_slots`
(`recipe_launcher._extract_calendar_query_slots`)。action `calendar.today_recap` 從 context
讀出 `CalendarRecapTarget`(kind = `today`/`tomorrow`/`on_date`),傳給
`ApiCalendarTodayReader.read_today(target=...)`,reader 再分派到既有
`calendar_today`(offset 0)/`calendar_tomorrow`(offset 1)/`calendar_on_date(date=...)`。
抽取失敗 / 多日 / 解析不出日期 → 退化成今天(CLE-476 行為),絕不答錯天。`today` 維持
「已過期過濾 + 進行中保留」;`tomorrow`/`on_date` 列整日(那兩個概念只對今天有意義)。

新 recipe 是**唯讀單發**:reader 輸出 `RenderDirective(kind="schedule_list")`,Discord 端用
`schedule_list` embed(title 來自 `metadata["intro"]`,已反映目標日,如「明天的行程:」),
無 View/按鈕;LINE 只吃 plain-text fallback。metadata 先保留 `items[].event_id`,但本版
不掛編輯/刪除。

結尾關心句由 `LLMPersonaCareWriter.write_closing(day_label=...)` 走 `LLMOrchestrator.call(...)`
一次,system prompt 來自 `PersonaEngine.build_system_prompt(task="query")`,user prompt 注入近
30 turns 的 `ConversationMemoryService.recent(limit=30)`、目標日與該日行程。prompt 禁止宣稱
已新增/修改/取消/通知任何人,禁「真的耶/對啊/哈哈」和罐頭祝福;LLM 或 memory/persona
依賴失敗時只退化成固定安全關心句(也吃 `day_label`),行程卡仍照常送出。Cleo 在這條路
只讀行程、只關心,**不代辦**。

---

## 13. 股票追蹤清單 watchlist(CLE-487)

使用者管理自己的 Ai_Stock_Agent 追蹤清單(≤30 檔)。**兩條入口共用同一套後端**
(gateway + enqueuer + quota gate),所以 NL 與 slash 行為一致:

- **NL(DM-only)**:scope_classifier 判 `watchlist_manage` → `INTENT_TO_RECIPE` →
  `watchlist_manage` WorkflowRecipe(`packages/shared/cleo_shared/workflow/recipes/watchlist_manage.py`)。
  launcher 用 Haiku 預抽 `{op, symbol, note}`(`recipe_launcher._extract_watchlist_slots`),
  recipe 用 `IntentRouterNode` 依 `op`(add/list/remove/scan)分流到對應 action。
- **Slash**:`/watchlist add|list|remove|scan`(`apps/bot/cleo_bot/cogs/watchlist.py`,`WatchlistCog`)。
  guild 內呼叫 → ephemeral 導去 DM。

四個 action 在 `packages/shared/cleo_shared/workflow/actions/watchlist_*.py`
(`watchlist.add` / `.remove` / `.list` / `.scan_enqueue`,都 low side-effect)。
add/remove/list 是**快速同步** provider 呼叫,經注入的 `WatchlistGateway` 打上游後直接回更新清單;
友善文案(滿 30、404 not-tracked、空清單)都當成 `ok` 的 `reply_text` 回,不吐 raw error。

**scan 很慢(整份 × 60s/檔),絕不 inline 跑**:`watchlist.scan_enqueue` 只做
「過 daily quota(`StockQuotaService`)+ enqueue worker」,即時回 ack。worker task
`apps/worker/cleo_worker/tasks/watchlist_scan.py:run_watchlist_scan` 呼上游 scan(長 timeout)
→ 每檔結果經既有 `/internal/send-stock-analysis-dm`(CLE-420)逐檔 DM,最後記一筆 billable
`stock_query` 計入 quota。

**接線位置**:provider client/models(`packages/providers/cleo_providers/stock_agent/`)、
`validator.py` action levels、`ActionServices`(gateway/enqueuer/quota Protocol)、`factory.py`
+ `main.py:try_workflow_router` 注入、`recipes/__init__` + scope_classifier/prompt、worker
`wiring/__init__`、bot adapter `apps/bot/cleo_bot/watchlist_adapter.py`、cog loader(`main.py`)。
上游契約見 `/Users/linus/Ai_Stock_Agent/CLAUDE.md` §Watchlist API。

---

## 14. 後端腳本發 Discord DM + 呼叫 Ollama(維運/排程腳本可抄)

任何後端(API / worker / scheduler / 一次性 cron 腳本)要**主動發一則 Discord DM 給使用者**,
**不要**自己連 discord.py;走 bot 既有的 Redis outbox:

- **最簡單**:`POST {CORE_API_URL}/internal/send-reminder-dm`
  - header:`X-Bot-Token: <CLEO_INTERNAL_TOKEN>`(驗證在 `apps/api/cleo_api/routes/internal/__init__.py:_verify_token`)
  - body:`{"discord_user_id": <int>, "content": "<文字>", "components": []}`
    (也可改用 `"user_id": "<UUID>"`,二選一;route model `SendReminderDMRequest`,~line 112)
  - 站長自己的 Discord id = `settings.admin_discord_id`(env `ADMIN_DISCORD_ID`,已在 `.env`)。
  - 機制:endpoint 把 `BotDmEnvelope(action="send_text")` LPUSH 進 Redis `oauth_dm_queue` →
    bot 用 `/internal/dm-queue/drain` 拾起 → `apps/bot/cleo_bot/dm_dispatch_handlers.py:handle_send_text`
    → `user.send(content)`。`quiet_hours_policy="respect"`(睡眠時段會延後;要即時設別的 policy 得走 redis 直送)。
  - **直接送 Redis**(省一層 HTTP):`cleo_shared.services.bot_dm_dispatch.enqueue_bot_dm(redis, envelope)`
    (`packages/shared/cleo_shared/services/bot_dm_dispatch.py`)。
  - 真實範例:`apps/worker/cleo_worker/tasks/traffic_alert.py`(`send_dm`)、`scripts/bot_exposure_report.py`。

呼叫**本地 Ollama**(離線、不花 API token):
- base url = `settings.ollama_base_url`(env `OLLAMA_BASE_URL`,預設 `http://127.0.0.1:11434`);
  model = `settings.ollama_model`(env `OLLAMA_MODEL`,預設 `qwen3:8b`)。
- 一次性生成最簡:`POST {base}/api/generate`,body `{"model","prompt","stream":false,"think":false}`,
  取 `.response`。**qwen3 預設會吐 `<think>…</think>`**:送 `"think": false` 關掉,並在 prompt
  開頭加 `/no_think`,保險再 `re.sub(r"<think>.*?</think>","",resp,flags=re.DOTALL)` 去殘留。
- 想要 metering / 重試 / 生命週期的正規包裝走 `cleo_shared.services.llm.ollama_backend.OllamaBackend`
  或 `AnthropicWrapper(force_backend="ollama")`;cron 腳本嫌重就直接 urllib 打 `/api/generate`(stdlib,免 venv)。

**維運哨兵範例**:`scripts/bot_exposure_report.py`(每日 cron 09:00)= 讀 api.log 流量快照 →
Ollama 寫成 zh-TW 自然語言 → DM 給站長。背景見 memory `[[project_calendar_webhook_tunnel_bound]]`
(2026-05-31 全域關 Bot Fight Mode 後,用它觀察對外曝險是否上升、評估要不要升 Cloudflare Pro)。

---

## 15. Outlook 行事曆同步(三方法 + 桌面 app 散布,CLE-546/547/548)

把 Outlook 行事曆同步進 Cleo 有**三條路**,公開教學頁 = `https://onboard.edurpg.org/outlook-sync`
(檔案 `apps/web/app/outlook-sync/page.tsx`,Next.js 靜態頁,用 `LegalShell` 元件)。背景見
memory `[[project_outlook_local_com_sync]]`、`[[project_microsoft_publisher_verification]]`。

1. **OAuth 直接連結**(個人帳號最適合)— Discord `/bind microsoft`(cog `apps/bot/cleo_bot/cogs/oauth.py`)
   → `/oauth/microsoft/start` → `Calendars.Read`(scope 已啟用;`Mail.Read` 仍關著等 publisher
   verification)。企業租戶常被 risk-based step-up consent 擋成「需要管理員核准」→ 要 IT admin consent。
2. **桌面同步程式**(公司帳號;Windows / macOS 皆可)— 本機匯出 .ics 自動上傳,**完全繞開 OAuth /
   publisher verification / 公司登記**。兩個平台**共用同一後端契約**(CLE-546 device token +
   ics-ingest),只差「怎麼從本機讀到行事曆」。
   - 後端 **CLE-546**:device token(`/connect-outlook` 發、`/devices` 撤;table `user_device_tokens`)
     + `POST /api/calendar/ics-ingest`(Bearer token、auto-commit、window reconcile soft-delete +
     bulk-delete 護欄)。route `apps/api/cleo_api/routes/calendar/ics_ingest.py`、cog
     `apps/bot/cleo_bot/cogs/connect_outlook.py`。
   - **Windows app CLE-547**:**原始碼留私有主 repo** `tools/cleo-outlook-sync/`(.NET 8 WinForms
     tray app,late-binding **COM** `GetCalendarExporter().SaveAsICal`,只支援 **classic Outlook**;
     build/簽章見該目錄 `BUILD.md`)。Outlook 原生匯出 → 週期性事件輸出 **RRULE**(後端展開)。
   - **macOS app CLE-548**:**原始碼留私有主 repo** `tools/cleo-outlook-sync-mac/`(Swift/AppKit
     `NSStatusItem` 選單列 app,`LSUIElement`)。**不靠 Outlook app、也不分新舊 Outlook**,改用
     **EventKit**(`EKEventStore`)讀系統層行事曆 → 前提是使用者把 Outlook/Exchange/M365 帳號加進
     **系統設定 → 網際網路帳號**(app 用 `hasExchangeAccount()` 偵測並引導)。token 存 Keychain、
     開機自啟用 `SMAppService`(預設關)、同步 `Timer`(預設 10 分)。⚠️ **EventKit 回傳的是已展開的
     occurrence、且同一週期序列共用一個 `calendarItemExternalIdentifier`**,後端按 UID 去重會把週會塌
     成一筆 → `CalendarReader.stableUID()` 對週期性事件併入 occurrence 時間給每次發生唯一穩定 UID
     (CLE-548 已修)。build/簽章/公證/發版見該目錄 `BUILD.md` + `RELEASE.md`。
3. **ICS 訂閱**(新 Outlook / 網頁版 / outlook.com)— Outlook 發佈行事曆取 ICS 連結 → Cleo 設定
   `/settings/connections` 貼上 → `POST /calendar/ics-subscribe` → worker `ics_calendar_sync` 定時抓,
   `IcsDiffMerger`(source_type=`ics`)會**硬刪** feed 裡消失的事件。device/手動上傳則是 source_type=`ics_upload`。

### ⚠️ 桌面 app 散布的鐵則(改版必看)

- **binaries 不放主 repo**(主 repo private → 它的 release **無法匿名下載**,實測 404;且 58MB 會撐爆 git)。
- 散布走**另一個公開 repo** `linus213-art/CleoOutlookSync`(只放 README + release binaries,**不放原始碼**)。
- ⚠️ **三平台 asset 全掛在同一個「latest」release 上**(目前 `outlook-sync-v0.1.0`)。網頁三個下載鈕
  都用 **`releases/latest/download/<固定檔名>`**,所以**發新版用 `gh release upload <現有tag> <新asset>`
  加到現有 latest,別 `gh release create` 另開**(另開會頂掉 latest → 另一平台連結 404,除非同時補齊全部
  平台 asset)。背景見 memory `[[project_cleo_outlook_sync_release_layout]]`。
- **固定 asset 檔名**(發新版**不可改名**,改名才要回頭改頁面):
  - Windows:`CleoOutlookSync-win-x64-self-contained.zip`(免 runtime)、
    `CleoOutlookSync-win-x64-framework-dependent.zip`(需 .NET 8)。
  - macOS:`CleoOutlookSync-macos-universal.dmg`(universal,**已簽章+公證**)。
- 發新版流程:
  - Windows:照 `tools/cleo-outlook-sync/BUILD.md` `dotnet publish` → 打包兩個同名 zip → `gh release upload`。
  - macOS:照 `tools/cleo-outlook-sync-mac/RELEASE.md` 步驟 3→4→5(`build-app.sh` 帶 `SIGN_ID` →
    `package-dmg.sh` 帶 `NOTARY_PROFILE=cleo-notary` → `gh release upload`)。憑證/notary profile 已一次性
    設好在 Mac mini keychain(Developer ID `ALGJXG7K45`)。
- .NET 8 runtime 連結用微軟官方 aka.ms 別名(永遠最新):
  `https://aka.ms/dotnet/8.0/windowsdesktop-runtime-win-x64.exe`。
- **簽章狀態不同**:Windows exe **未簽章**,首次執行跳 SmartScreen(頁面註明「其他資訊 → 仍要執行」);
  macOS .dmg **已簽章+公證**,別台 Mac 雙擊零 Gatekeeper 警告。

改 `apps/web/app/outlook-sync/` 後要 `cd apps/web && pnpm build && pm2 restart cleo-web` 才生效(見 §2)。
未來郵件功能範圍已定 = **只抓 metadata(寄件者/標題/時間)**,不抓內文、不自動回信。
