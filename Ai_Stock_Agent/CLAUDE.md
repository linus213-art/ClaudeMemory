# Ai_Stock_Agent — Claude Code 專案備忘

> 給未來 session 的 Claude：這份檔案是「每次都要重複問的東西」的常駐答案。
> 不要重複實作或重新查證已記錄的事實，除非懷疑它過期（對照當前檔案再決定）。
> 與此檔案搭配的還有 `README.md`（部署 SOP）、`ISOLATION_README.md`（與 EduRPG 隔離守則）、`FSD_AI_Stock_Agent.md`（功能規格）。

---

## 1. Work area / 路徑

- **Repo root**：`/Users/linus/Ai_Stock_Agent`
- **Worktrees dir**（dispatcher 用）：`/Users/linus/Ai_Stock_Agent.worktrees/`（與 repo root 平行，非 `.worktrees`）
- **Log dir**（launchd jobs 寫入）：`/Users/linus/Ai_Stock_Agent/logs/`
  - `api.{out,err}.log`、`daily.{out,err}.log`、`weekly.{out,err}.log`、`kpi.{out,err}.log`、`healthcheck.{log,err.log}`、`datasync_{price,price_tpex,fundamental}.{out,err}.log`、`leading_indicator_{us,jp}.{out,err}.log`、`glossary_unverified.{out,err}.log`、`data_retention.{out,err}.log`、`orphan_cleanup.{out,err}.log`、`indicators_pipeline.{out,err}.log`、`macro_indicators.{out,err}.log`、`market_breadth.{out,err}.log`、`market_breadth_margin.{out,err}.log`、`hedge_monitor_{preliminary,final}.{out,err}.log`、`nonfarm.{out,err}.log`、`capex.{out,err}.log`、`premarket_alert.{out,err}.log`、`dip_scan_intraday.{out,err}.log`、`data_gap_check.{out,err}.log`、`institutional_flow.{out,err}.log`、`position_overlay_eod.{out,err}.log`、`intraday_monitor.{out,err}.log`、`signal_collect.{out,err}.log`、`signal_universe.{out,err}.log`、`intraday_flow_eod.{out,err}.log`、`signal_scorecard_{monthly,weekly}.{out,err}.log`、`refresh_holdings_signals.{out,err}.log`
- **Dispatcher 工作 log**：`tasks/logs/TASK-NNN.log`（每個任務一份，retain on failure）
- **Backup dir**：`/Users/linus/Ai_Stock_Agent/backups/`（`AI_STOCK_AGENT_BACKUP_DIR` 可覆寫）
- **Main branch**：`main`
- **Python venv**：`/Users/linus/Ai_Stock_Agent/.venv`（launchd 與 dispatcher 都吃這個路徑；dispatcher 在每個 worktree 內各自再開一份 `.venv`）
- **套件管理**：`pip` + `venv`（**不是** `uv`，與 Cleo 不同）。requirements 從 `pyproject.toml` 的 `[project.dependencies]` / `[project.optional-dependencies].dev` 來，不要碰 `requirements.txt`（只是 bootstrap 後備）。
- **Python 版本**：3.11+（`pyproject.toml` 鎖 `requires-python = ">=3.11"`）
- **獨立性**：與 `/Users/linus/edurpg`、`/Users/linus/cleo` 物理隔離；env prefix 是 `AISTOCK_*`，**不可** 與 `EDURPG_*` / `CLEO_*` 在同一 shell 共存。

---

## 2. 服務啟動 / 停止順序

> 全部 launchd job 都是 **system 域 LaunchDaemons**（`/Library/LaunchDaemons/`），不是 user 域 LaunchAgents——headless Mac mini 無 GUI 登入也要跑。所有 `launchctl` 動作都需要 `sudo`（腳本會自動 prompt 一次）。

### 一鍵重啟（最常用）

```bash
cd /Users/linus/Ai_Stock_Agent
./scripts/restart_all.sh           # postgres → kickstart 常駐 job(api+healthcheck;**預設不含**報表 job)→ API health probe
./scripts/restart_all.sh --no-db   # 只 bounce API，不動 postgres
./scripts/restart_all.sh --no-api  # 只 bounce postgres，不動 API
./scripts/restart_all.sh --yes-reports  # 重啟時「也」kickstart daily/weekly/kpi → 立刻重發 Discord 報表。**預設不發**（restart 預設只 bounce api+healthcheck，不再 off-schedule 重發報表）；`--no-reports` 保留為 no-op
./scripts/restart_all.sh --health-only  # 只跑 `/healthz` probe，印 pid 表
```

底層用 `launchctl kickstart -k system/<label>` in-place 重啟，不做 bootout / bootstrap，避免 daemon 漂移。

### 啟動順序（restart_all.sh 內建）

> **重要：`restart_all.sh` 的 `JOBS` 陣列只 kickstart 下面 5 個「常駐 / 關鍵」label**，**不**碰排程型資料 job。
> 原因：`kickstart -k` 對排程 job 會立刻 off-schedule 跑一次；排程 job 由 launchctl_manage 安裝、各自按 calendar 觸發即可，不需要 bounce。
> 完整 job 目錄見下方「全部 launchd job 目錄」。

1. **PostgreSQL**（`system/homebrew.mxcl.postgresql@16`）—— 等 `pg_isready` 通過（最多 20 秒）
2. **restart_all 實際 kickstart 的 5 個 label**（`kickstart -k`，並行）：
   1. `com.linus.aistockagent.api`（FastAPI / uvicorn，常駐，`KeepAlive=true`）
   2. `com.linus.aistockagent.daily`（每日 14:00 跑 `ai_stock_agent.jobs.daily_report`）
   3. `com.linus.aistockagent.weekly`（每週六 09:00 跑 `weekly_report`）
   4. `com.linus.aistockagent.kpi`（每月 1 號 03:00 跑 `kpi_monthly`）
   5. `com.linus.aistockagent.healthcheck`（每 300 秒，呼 `/healthz`、失敗時 Discord 告警 + 嘗試自動 kickstart API）
3. **API health probe**（`curl $AI_STOCK_AGENT_HEALTHCHECK_URL`，30 秒內等 200）

#### 全部 launchd job 目錄（★ = restart_all 會 kickstart；§ = 已在 launchctl_manage JOBS）

> **權威完整清單(39 個 job,含排程/module/用途、分類)見 [`docs/Scheduled_Jobs.md`](docs/Scheduled_Jobs.md)**(由 plist 直接解析,最準)。下表為常用摘要;新增 job 請同步更新該 doc。

| Label | 排程 | 說明 |
|---|---|---|
| `api` ★§ | 常駐 | FastAPI / uvicorn，`KeepAlive=true` |
| `daily` ★§ | 每日 14:00 | `ai_stock_agent.jobs.daily_report` |
| `weekly` ★§ | 週六 09:00 | `weekly_report` |
| `kpi` ★§ | 每月 1 號 03:00 | `kpi_monthly` |
| `healthcheck` ★§ | 每 300 秒 | `/healthz` probe + 自動 recover |
| `data_retention` § | 每月 1 號 03:00 | 資料保留清理 |
| `orphan_cleanup` § | 每月 1 號 03:40 | 孤兒股清理(houseclean):`stock.source='onboarded'` 且不在任何 watchlist 且 `last_referenced_at` 超過 `ORPHAN_CLEANUP_DAYS`(預設 90)→ 刪該股 11 張表 + stock 列,發 admin。**seed universe 永不清**;既有股經 migration 0021 全標 seed(保守),只有此功能上線後新 onboard 的才可清。跑 `ai_stock_agent.jobs.orphan_cleanup`（`--days` 覆寫）|
| `datasync_price` § | 週一～五 14:30 | 日線增量同步(**上市 TWSE**;TWSEAdapter `STOCK_DAY`) |
| `datasync_price_tpex` § | 週一～五 14:40 | **上櫃 TPEX** 日線增量同步:FinMind `TaiwanStockPrice`(上市+上櫃通用)逐檔補 `price_daily`,因 TWSEAdapter 只含上市,上櫃股原本無價(overlay 損益/流動性護欄拿不到價)。跑 `ai_stock_agent.scripts.sync_price_tpex`(增量;首見 symbol 自動冷回補 420 天) |
| `datasync_fundamental` § | 週六 07:00 | 季財報同步 |
| `leading_indicator_us` § | 週日～五 06:00 | 美股領先指標 |
| `leading_indicator_jp` § | 週一～五 | 日股領先指標 |
| `glossary_unverified` § | 週一 08:00 | 名詞庫未驗證掃描 |
| `macro_indicators` § | 週一～五 06:30 | 全球風險溫度計：yfinance 抓 10 碼(`^VIX`/`^TNX`/`^IXIC`/`^SOX`/`^N225`/`^KS11`/`CL=F`/`GC=F`/`CNY=X`/`DX-Y.NYB`)寫 `macro_indicator_snapshot`。job 內含 staleness watchdog(>3 交易日逾期發 admin) |
| `indicators_pipeline` § | 週一～五 14:35 | M7：refresh `adjusted_ohlcv` → `daily_indicators` → `daily_signals`（詳見 `docs/Technical_Indicator_Module_README.md` §7）。增量模式，跑 `ai_stock_agent.scripts.run_indicators_pipeline`，斷跑後重跑一次即從水位回補 |
| `market_breadth` § | 週一～五 14:40 | HedgeMonitor P0：抓 FMTQIK 寫 `TWII`+`TW_TURNOVER` 進 `macro_indicator_snapshot`（詳見 `docs/Hedge_Monitor_Spec.md` §2.1；P1 後加 21:05 補 `TW_MARGIN_BALANCE`） |
| `market_breadth_margin` § | 週一～五 21:05 | HedgeMonitor P1：FinMind 全市場融資餘額 → `TW_MARGIN_BALANCE`（億元）。融資約 21:00 才更新，故獨立晚場一輪 |
| `hedge_monitor_preliminary` § | 週一～五 14:45 | HedgeMonitor P2：風險評分初版(融資 T-1)。寫 `market_risk_snapshot`、送 8 指標 report embed、排除 stale 融資仍紅燈才發初版 admin 預警 |
| `hedge_monitor_final` § | 週一～五 21:35 | HedgeMonitor P2：風險評分終版(融資 T)。跨級(比前一日終版)才發 admin。詳見 `docs/Hedge_Monitor_Spec.md` §4 |
| `nonfarm` § | 週日～五 06:35 | HedgeMonitor P1.5/II-4：FRED `PAYEMS` → `US_NONFARM`(就業水準/千人)+`US_NONFARM_CHG`(月變化)。需 `FRED_API_KEY`(免費)。idempotent，每日跑無妨 |
| `capex` § | 週六 07:30 | HedgeMonitor P1.5/I-4：FinMind 現金流 PP&E → `fundamental.capex_outflow`(去 YTD 累計成單季)。巨頭 2330/2454/2317 |
| `policy_calendar` § | 每日 23:00 | 把 `seeds/policy_calendar.yaml` 重新 upsert 進 `policy_event`。**非即時來源**——只在 seed YAML 被人手編輯後才有差異，所以「沒跑」不會漏即時資料，但 seed 改了要靠它同步。idempotent，跑 `ai_stock_agent.jobs.policy_calendar` |
| `premarket_alert` § | 週一～五 08:30 | HedgeMonitor P3c：盤前開盤衝擊預警。台指期夜盤(FinMind `TaiwanFuturesDaily` after_market)+ 美股隔夜估開盤衝擊,**只在 amber/red 才發**(red→admin、amber→report),避免每日噪音。跑 `ai_stock_agent.jobs.premarket_alert` |
| `dip_scan_intraday` § | 週一～五 11:00 | HedgeMonitor P3d/P2.5：盤中抄底掃描。跑 `ai_stock_agent.jobs.dip_scan --realtime`(TWSE MIS 近即時 + 即時 TAIEX 基準,MIS 抓不到退 EOD)。**每次都送 report**(含 0 檔)。交易時段內,抓盤中抗跌領先股 |
| `data_gap_check` § | 週一～五 22:00 | 資料缺口巡檢:跑 `scripts/check_data_gaps_alert.sh` → `check_data_gaps.sh`,**只在 DB 新鮮度發現缺口(exit≠0)才發 admin(含 mention)**。22:00 因所有當日 TW 資料 job(含 market_breadth_margin 21:05、hedge_monitor_final 21:35)都跑完。log-mtime 段為資訊性、不觸發告警。手動巡檢直接 `./scripts/check_data_gaps.sh`;測試告警 `./scripts/check_data_gaps_alert.sh test` |
| `institutional_flow` § | 週一～五 21:15 | 個股三大法人買賣超流向 job(TASK-164)。跑 `ai_stock_agent.jobs.institutional_flow`。21:15 因三大法人資料約 21:00 後才齊 |
| `position_overlay_eod` § | 週一～五 21:25 | 個人持股 overlay 收盤批次(TASK-178)。跑 `ai_stock_agent.jobs.position_overlay_eod`,對所有持股 user×symbol 算 overlay(P&L + 籌碼趨勢)upsert `user_position_overlay_snapshot`,供下次開頁秒載 / 即時 endpoint 報價 stale 時降級回讀。排在 21:15 法人報表後才拿得到當日籌碼趨勢 |
| `signal_collect` § | 週一～五 22:10 | **Signal Scorecard** 每日訊號採集(M11，TASK-183)。跑 `ai_stock_agent.jobs.signal_collect`:讀當日各已落地源(訊號等級/法人原始/`inst_report` 精選/抄底/避險/macro 18 碼)→ 映射成 signed `signal_observation`(curated 源可擴張焦點宇宙、baseline 全市場源只對 active∪持股∪當日 candidates 寫、market 軸對 TWII)→ upsert → 對**所有未結案列** sweep 前向報酬回填。排在 institutional 21:15 / hedge final 21:35 / data_gap_check 22:00 之後。詳見 `docs/Signal_Scorecard_Spec.md`。**焦點宇宙維護(TASK-184)未上線前** active 集合讀 `signal_universe`(初期空),屬正常 interim |
| `signal_scorecard_monthly` § | 每月 1 號 04:00 | **Signal Scorecard** 計分卡月報(M11，TASK-186)。跑 `ai_stock_agent.jobs.signal_scorecard_report --period-kind month`:對上月各 scope×source×horizon 算命中率/IC/累積報酬 → upsert `signal_scorecard` → 發 report webhook 文字排行榜。折線圖由 dashboard 讀 `/api/v1/scorecard` series 自畫(chart.js,不產 server PNG) |
| `signal_scorecard_weekly` § | 週六 09:30 | **Signal Scorecard** 計分卡週報(M11，TASK-186)。同上,`--period-kind week`,對上一 ISO 週計分 |
| `refresh_holdings_signals` § | 週一～五 22:50 | **Signal Scorecard** 持股資料自動回補(M11)。跑 `ai_stock_agent.jobs.refresh_holdings_signals`:解決「使用者持股(尤其法人同步 universe 未涵蓋的 off-universe 股如宏碁2353/正文4906)在計分卡缺資料」。① 對每檔持股回補 `stock_institutional_daily`(**新/稀疏持股(法人<60列)→深度420天**全歷史、已覆蓋→增量45天)② 重跑 signal observation backfill **限持股 symbol**,補上 inst_*/daily_signal 觀測。idempotent。排在 institutional 21:15 後。手動:`python -m ai_stock_agent.scripts.backfill_holdings_institutional` + `... backfill_signal_observations --symbols 2353,4906` |
| `intraday_flow_eod` § | 週一～五 13:35 | **Signal Scorecard** 盤中大單收盤落地(M11，TASK-185)。跑 `ai_stock_agent.jobs.intraday_flow_eod`:讀 `intraday_dashboard_state` singleton(收盤後仍存最後狀態)拆 per-symbol → upsert `intraday_flow_daily`,供 signal collector 採 `intraday_flow` 源。**不碰** intraday_monitor daemon;singleton 為當日(generated_at≠當日交易日)才落地,否則不寫(無 Shioaji feed 時正常空) |
| `signal_universe` § | 週一～五 22:30 | **Signal Scorecard** 焦點宇宙維護(M11，TASK-184)。跑 `ai_stock_agent.jobs.signal_universe`,排在 signal_collect 22:10 後:重算持股、累計 curated 點名熱度(EWMA)→ 套 TTL(10 交易日無 curated 點名)/ 容量(焦點 K=100,持股不計)下車 + re-activation,upsert `signal_universe`。**軟下車**(active=false)只擋 baseline collector 開新單,在途 observation 照回填。詳見 `docs/Signal_Scorecard_Spec.md` §6 |
| `intraday_monitor` § | 常駐(盤中 09:00–13:30) | **第二層**「盤中大單即時偵測」常駐 daemon(M9 Phase C，TASK-174)。跑 `ai_stock_agent.jobs.intraday_monitor`，訂閱 watchlist∪當日成交值前 N 大 → 聚合/偵測大單(代理主力)→ 維護 realtime Top10 dashboard state(`intraday_dashboard_state`)供 179 讀。**需 Shioaji 金鑰才實際連線**(未設則無 tick)。**用模擬模式即可**:`SHIOAJI_SIMULATION`(預設 `true`)→ 模擬模式仍給**真實盤中行情**(已驗證 snapshot/tick 為真),**不需正式/下單權限**,故只訂閱行情的本 daemon 用模擬即可、**免等永豐正式 API 審核**(正式只 gate `place_order`;正式開通後設 `false` 對行情無差異)。**每日 09:00 由 plist `StartCalendarInterval` 啟動**(`RunAtLoad` 只在載入跑一次,非交易時段 `exit 0` 後不被 KeepAlive 拉起,故需開盤觸發器)。連線/訂閱失敗 → clear log + 乾淨退出(started=False/exit 0),不 crash-loop。`KeepAlive={SuccessfulExit=false}`:收盤 13:30 正常 `exit 0` 不重啟、只有 crash 才拉起。健檢走獨立 `scripts/healthcheck_intraday_monitor.sh`(只在台北交易時段要求 pid)，**不**混進 API 的 `healthcheck.sh` |

> 註：原「既有漂移」三個 job（`macro_indicators`、`indicators_pipeline`、`policy_calendar`）已於 2026-06-09 全部納管進 JOBS 並安裝，漂移清零。
> 歷史教訓（2026-06-09）：`macro_indicators`、`indicators_pipeline`、`policy_calendar` 三個都曾被以為「有 plist 且實際在跑」，實際上**從未安裝**(無 `/Library/LaunchDaemons/` plist、無 log、不在 JOBS)。macro 與 indicators_pipeline 因此資料自 ~2026-05-27 靜默凍結兩週(macro 靠 job 內 staleness watchdog 在 6/9 發 admin 抓到、pipeline 靠 `scripts/check_data_gaps.sh` 掃出，皆已手動回補)；policy_calendar 因是 seed-based 非即時來源故無實際漏資料。診斷漂移類問題：**永遠用 log mtime + DB 新鮮度驗證(`./scripts/check_data_gaps.sh`)，不要信「應該在跑」**。

### 停止 / 反向順序

`restart_all.sh` 是 in-place restart，沒提供「乾淨關全部」的 flag。要完整下架時：

```bash
# 反向：先 jobs 後 postgres
./scripts/launchctl_manage.sh unload    # bootout 全部 aistockagent.* jobs(見 docs/Scheduled_Jobs.md)
sudo launchctl bootout system/homebrew.mxcl.postgresql@16   # 最後才停 postgres（一般情況不要停）
```

### 從零安裝 / 重新載入 plist

```bash
./scripts/launchctl_manage.sh load      # 把 launchctl_manage.sh JOBS 清單(全部,見 docs/Scheduled_Jobs.md)的 plist 安裝到 /Library/LaunchDaemons/ 並 bootstrap
./scripts/launchctl_manage.sh status    # 印每個 job 的 pid + last_exit_status
./scripts/launchctl_manage.sh restart   # bootout + bootstrap 每個 job（churn 比 kickstart 大）
```

`launchctl_manage.sh load` 會：
- 用 `perl -0pe` 把 plist 內的 `WorkingDirectory` / healthcheck 路徑改成目前 repo 位置（所以 repo 搬位置也不會壞）
- 跑 `plutil -lint` 才寫進 `/Library/LaunchDaemons/`
- 安裝為 `0644 root:wheel`，但 plist 都設 `UserName=linus` 讓實際程式跑在 user 身分

### 環境變數開關（restart / healthcheck / backup 都讀）

| 變數 | 預設 | 用途 |
|---|---|---|
| `AI_STOCK_AGENT_ENV_FILE` | `<repo>/.env` | 哪份 .env 餵進腳本 |
| `AI_STOCK_AGENT_BIND_HOST` | `127.0.0.1` | uvicorn 綁定 IP（正式 = `192.168.11.99`） |
| `AI_STOCK_AGENT_BIND_PORT` | `8080` | uvicorn 綁定 port |
| `AI_STOCK_AGENT_HEALTHCHECK_URL` | `http://<host>:<port>/healthz` | health probe URL |
| `AI_STOCK_AGENT_HEALTHCHECK_TIMEOUT` | `10` | curl 超時秒數 |
| `AI_STOCK_AGENT_RESTART_TIMEOUT` | `30` | restart_all 等 API 上線秒數 |
| `AI_STOCK_AGENT_RECOVERY_STATE_FILE` | `/tmp/aistockagent.recovery.lasttry` | healthcheck 自動 recover 去抖檔 |
| `AI_STOCK_AGENT_RECOVERY_THROTTLE_SECONDS` | `600` | 兩次自動 recover 最短間隔 |
| `AI_STOCK_AGENT_RECOVERY_RECHECK_DELAY` | `8` | recover 後驗證前等待秒數 |
| `AI_STOCK_AGENT_BACKUP_DIR` | `<repo>/backups` | pg_dump 輸出目錄 |
| `AI_STOCK_AGENT_BACKUP_RETENTION` | `7` | 保留幾份備份 |
| `AI_STOCK_AGENT_BACKUP_DATE` | `$(date '+%F')` | 強制覆寫備份檔日期（測試用） |
| `AI_STOCK_AGENT_PG_DUMP_BIN` | `pg_dump` | 自訂 pg_dump 路徑 |
| `AI_STOCK_AGENT_LOG_DIR` | `/usr/local/var/log/ai_stock_agent` | daily_status.sh 掃描的 log 目錄（**注意**：launchd plist 寫的是 `<repo>/logs/`，兩者不同——`daily_status.sh` 內讀的是這個 env，必要時要覆寫成 `<repo>/logs`） |
| `AI_STOCK_AGENT_DAILY_STATUS_DAYS` | `7` | daily status 回溯天數 |
| `BACKUP_PASSPHRASE` | （必填） | age 加密密碼，**必須**從 `.env` 注入 |

### 自動恢復（healthcheck）

`scripts/healthcheck.sh` 失敗時會：
1. 送 Discord `DISCORD_ADMIN_WEBHOOK`
2. 透過 NOPASSWD sudoers 嘗試 `sudo launchctl kickstart -k system/com.linus.aistockagent.api`（或 bootstrap 若未載入）
3. 等 `RECOVERY_RECHECK_DELAY` 秒重 probe，恢復後再送一封 `:white_check_mark:` 確認

要啟用自動 recover 必須裝 sudoers fragment：
```bash
sudo install -m 0440 -o root -g wheel \
  /Users/linus/Ai_Stock_Agent/scripts/sudoers.d/aistockagent-recover \
  /etc/sudoers.d/aistockagent-recover
sudo visudo -c -f /etc/sudoers.d/aistockagent-recover
```
這份 sudoers 只授權 4 條 launchctl 指令對單一 daemon label（api），不能被擴用。

#### 讓 Code / 自動化能非互動跑 restart_all.sh（第二份 sudoers）

`scripts/sudoers.d/aistockagent-restart` 是**擴充版** NOPASSWD：授權 `restart_all.sh` 實際會跑的
launchctl print/kickstart/bootstrap + postgres log tail，綁定 5 個 aistockagent.* job + postgres
label。裝了之後 `restart_all.sh` 會**完全非互動**執行（`require_sudo` 用 healthcheck print 探測到
NOPASSWD 就跳過互動 prime），所以 **Code 能自動重啟服務**（含改 dashboard 後 bounce api）。

```bash
sudo install -m 0440 -o root -g wheel \
  /Users/linus/Ai_Stock_Agent/scripts/sudoers.d/aistockagent-restart \
  /etc/sudoers.d/aistockagent-restart
sudo visudo -c -f /etc/sudoers.d/aistockagent-restart
```

未裝時 `restart_all.sh` 退回互動 sudo（prompt 一次），行為與舊版相同。另注:**只有 api label**
另有 `aistockagent-recover`（healthcheck 自動復原用），所以即使沒裝 restart fragment，Code 仍能
單獨 `sudo launchctl kickstart -k system/com.linus.aistockagent.api` 重啟 api（=重載 dashboard）。

### Postgres daemon 修補

`brew upgrade postgresql@16` 會把 plist 重寫掉（沒 UserName，導致以 root 跑起來失敗）。每次 upgrade 後要：
```bash
sudo bash /Users/linus/Ai_Stock_Agent/scripts/install_postgres_daemon.sh
```
**不要** 用 `sudo brew services start/restart postgresql@16` 對這個 label，會打掉修補。

---

## 3. Dispatcher（自動派工）

主腳本：`tools/agent_dispatch_aistock_mac.ps1`（PowerShell 7+ 必要，macOS）。Spec generator：`tools/generate_task_specs.py`。

### Pending tasks 目錄

```
tasks/pending/CODE/      # 給 Claude Code 的任務（L3 複雜邏輯、跨檔案重構、約 23 個原始 spec）
tasks/pending/Copilot/   # 給 GitHub Copilot 的任務（L1+L2 CRUD / scaffold / 單元測試，約 86 個原始 spec）
tasks/done/              # 完成後 dispatcher 自動搬過來，並寫 .result.md（commit hash / lint / tests / timestamp）
tasks/logs/              # 每次執行 log（`TASK-NNN.log`）
```

Agent 對應規則：dispatcher 用 spec 檔案的 **父資料夾名稱** 決定 agent。`-Agent` flag 可手動覆寫。

### Loop 模式（常用兩條）

```bash
pwsh ./tools/agent_dispatch_aistock_mac.ps1 -Loop -Agent CODE
pwsh ./tools/agent_dispatch_aistock_mac.ps1 -Loop -Agent Copilot
```

Loop 行為：每完成一個任務自動找下一個 dispatchable（depends_on 都在 `tasks/done/`），跑到沒 task 或失敗才停。

> 注意：此版本 dispatcher **沒有** Cleo 那種 `tasks/logs/.stop-requested` 軟停止旗標——要停只能 Ctrl+C 或讓它跑完當前任務。

### 常用 flags

| Flag | 用途 |
|---|---|
| `-TaskId TASK-NNN` | 指定單一任務（不指定 = 取最低 id 且依賴都已完成的） |
| `-DryRun` | 只印計畫，不開 worktree、不叫 agent |
| `-NoMerge` | 跑完 agent + lint + test，停在 merge 前（保留 worktree 供人工檢查） |
| `-SkipPush` | 跳過 `git push origin main`（離線、第一週用） |
| `-Loop` | 串連模式（見上） |
| `-Agent CODE\|Copilot` | 覆寫 spec 預期的 agent |

### Pipeline 順序（每個任務 8 步驟）

1. `git worktree add` off `main`（或重用已存在 worktree）
2. 呼叫 agent：CODE → `claude --dangerously-skip-permissions --print < spec`；Copilot → `copilot --allow-all-tools --add-dir /tmp --allow-url 127.0.0.1 --allow-url localhost --prompt <spec>`。spec 前面會自動 prepend `SYSTEM_PREFIX`（禁止 `.venv` / `__pycache__` 入 commit、禁止 `&&` 串接、強制單一 commit）
3. Sanity check：worktree 有新 commit、git 狀態正常
4. 在 worktree 內：`python3 -m venv --upgrade-deps .venv` → `pip install -e ".[dev]"`
5. **Lint gate**：`ruff check .`（阻擋）+ `mypy ai_stock_agent`（警告但不阻擋早期 M0/M1）
6. **Test gate**：`pytest -q`（阻擋；exit code 5 = "no tests collected" 視為 skipped）
7. Auto-merge：`git -C repo merge --no-ff <branch>` → `git worktree remove` → `git branch -d`
8. Promote：`git mv` spec 從 `pending/` 到 `done/` + 寫 `.result.md` → `git push origin main`（除非 `-SkipPush`）

任何 gate 失敗時 worktree 會被保留供人工檢查；agent 的 spec stdin 暫存檔（CODE 模式）會印出路徑。

### 跨專案 guard（必看）

Dispatcher 第 ~60 行的硬擋：
```powershell
if ($REPO_ROOT -match 'edurpg' -or $WORKTREES_DIR -match 'edurpg') {
    throw "Cross-project path detected. Aborting."
}
```
即：**只擋 `edurpg`**，沒擋 `cleo`。同一 shell **不可** 同時設 `EDURPG_*` 和 `AISTOCK_*` env vars。詳見 `ISOLATION_README.md` 的「七條保命守則」。

#### 例外：跨專案唯讀稽核（read-only audit）

當使用者明確要求跨專案盤點（API usage、技術債、成本分析、安全審計等），允許 Claude 讀取 `/Users/linus/edurpg` 與 `/Users/linus/cleo` 的檔案，**但必須同時滿足**：

1. **純唯讀**：只允許 `Read` / `grep` / `rg` / `fd` / `git log` / `git diff`。**禁止** `Write` / `Edit` / `git add` / `git commit` / `git push` / `launchctl` / `pip install` / 任何修改 worktree、env、daemon 的指令。
2. **不設跨專案 env**：不要在當前 shell 設 `EDURPG_*` 或 `CLEO_*` 變數,也不要 `source /Users/linus/edurpg/.env` 之類。
3. **不啟動 dispatcher**：絕對不執行 `agent_dispatch_edurpg*.ps1` 或 `agent_dispatch_cleo*.ps1`。
4. **輸出寫在三個 repo 之外**：稽核報告寫到 `~/`（如 `~/api_audit_report_YYYYMMDD.md`),不要寫進任何專案目錄。
5. **使用 Explore subagent 並行掃**：保護主對話 context;明確指示 subagent「read-only,不寫檔不 commit」。

若 harness 跳 security warning(因路徑含 `edurpg`),向使用者解釋是「read-only 稽核例外」並繼續。**首次此類例外**:2026-05-25 三專案外部 API 盤點(報告於 `~/api_audit_report_20260525.md`)。

### Dispatcher 環境變數

| Env | 預設 | 用途 |
|---|---|---|
| `AISTOCK_REPO_ROOT` | `/Users/linus/Ai_Stock_Agent` | repo 根目錄 |
| `AISTOCK_WORKTREES_DIR` | `/Users/linus/Ai_Stock_Agent.worktrees` | worktree 父目錄 |
| `AISTOCK_BASE_REF` | `main` | merge target |
| `AISTOCK_VENV` | `.venv` | venv 資料夾名（相對於 worktree） |

---

## 4. DB / 服務連線

連線字串都在 `/Users/linus/Ai_Stock_Agent/.env`（已 gitignored）。**不要** 把密碼寫進這份 CLAUDE.md 或 commit 任何 .env。範本見 `config/.env.example`。

| Service | 連線形式 | 備註 |
|---|---|---|
| Postgres | `postgresql://ai_stock_agent:<see .env>@127.0.0.1:5432/ai_stock_agent` | local Homebrew `postgresql@16`，daemon `system/homebrew.mxcl.postgresql@16`（修補版，跑在 `linus` 身分而非 root）。logs：`/opt/homebrew/var/log/postgresql@16.log`。 |
| FastAPI（正式） | `http://192.168.11.99:8080` | 由 launchd `com.linus.aistockagent.api` 起 uvicorn，常駐，`KeepAlive=true`、`ThrottleInterval=30`。健康檢查 `GET /healthz`。 |
| FastAPI（本機開發） | `http://127.0.0.1:8080` | `uvicorn ai_stock_agent.main:app --reload` |
| OpenClaw（唯一 client） | Windows 端 bot，over LAN | Bearer `AI_STOCK_AGENT_API_TOKEN`；admin / scheduled endpoints 用獨立 `AI_STOCK_AGENT_ADMIN_TOKEN` |
| Discord webhook（admin） | `DISCORD_ADMIN_WEBHOOK`（.env） | healthcheck DOWN / 失敗告警 |
| Discord webhook（report） | `DISCORD_REPORT_WEBHOOK`（.env） | daily / weekly / kpi 月報送這裡（KPI 月報的 routing 在 `400ab5b` 之後改成 report，不是 admin） |

### Postgres 連線必要 env（從 .env 讀）

```dotenv
AI_STOCK_AGENT_DB_HOST=127.0.0.1
AI_STOCK_AGENT_DB_PORT=5432
AI_STOCK_AGENT_DB_NAME=ai_stock_agent
AI_STOCK_AGENT_DB_USER=ai_stock_agent
AI_STOCK_AGENT_DB_PASSWORD=...        # 密碼留在 .env，不入 git
```

LLM keys 同樣放 .env：`ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GOOGLE_API_KEY`。

### Watchlist API（使用者追蹤清單，OpenClaw client 呼叫）

實作在 `ai_stock_agent/api/watchlist.py`，router prefix `/api/v1/watchlist`，model 在 `ai_stock_agent/models/watchlist.py`。**所有 endpoint 都要 `Authorization: Bearer <AI_STOCK_AGENT_API_TOKEN>`**（不是 admin token）。`{user_id}` 是 Discord user id，放在 path 上。

| 動作 | Method + Path | Body |
|---|---|---|
| **加入追蹤** | `POST /api/v1/watchlist/{user_id}/add` | `{"symbol":"2330","note":"選填≤200字"}`（`symbol` 必填 1–10 字；`note` 選填） |
| 看追蹤清單 | `GET /api/v1/watchlist/{user_id}` | 無 |
| 移除追蹤 | `DELETE /api/v1/watchlist/{user_id}/{symbol}` | 無（symbol 不存在回 404） |
| 批次分析整份清單 | `POST /api/v1/watchlist/{user_id}/scan` | 選填 `{"focus":[...],"language":"zh-TW"}` |
| **個人持股 overlay**（即時） | `GET /api/v1/watchlist/{user_id}/overlay` | 無（TASK-178）。取現價(`MarketHoursQuoteSource`:台北交易時段優先 TWSE MIS 公開即時(上市 tse_/上櫃 otc_,延遲數秒~分,best-effort),盤後/失敗退 `EodQuoteSource` 收盤價)→ 177 算稅後 P&L → 組 overlay(含主力進出/籌碼趨勢/alert_level/`day_change_pct` 供漲停跌停亮燈)。報價取不到 → 回 `user_position_overlay_snapshot` 快照 + 標 `stale=true` 不阻塞。共用 `analysis/position_overlay.py::build_overlay_item`(唯一 overlay service)；批次由 launchd `position_overlay_eod`(21:25)落地快照 |

加入追蹤範例：
```bash
curl -X POST "http://192.168.11.99:8080/api/v1/watchlist/$USER_ID/add" \
  -H "Authorization: Bearer $AI_STOCK_AGENT_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"symbol":"2330","note":"AI 供應鏈核心持股"}'
```

回應是 `WatchlistResponse`，`data.items[]` 回該使用者**整份**清單（每筆含 `symbol` / `note` / `added_at`）。

- 每位使用者上限 **30 檔**（`MAX_WATCHLIST_ITEMS`）。
- `symbol` 會被正規化（去空白、轉大寫）；每個操作都寫 `audit_log`。
- 對應 DB 表是 `user_watchlist`（見 §4 Schema）；批次掃描 per-symbol timeout = 60 秒（見 §5 慣例）。

### 個股法人資金流向報表 API（Latest Institutional Flow Report，OpenClaw / dashboard client 呼叫）

實作在 `ai_stock_agent/api/institutional_report.py`，router prefix `/api/v1/institutional`，**唯讀**。
資料由排程 job `institutional_flow`（週一~五 **21:15**，見 §2 job 表）每日算好 Top10 落地到
`institutional_flow_report_daily`（JSONB），本 API 只是查詢、**不觸發計算**。
**所有 endpoint 都要 `Authorization: Bearer <AI_STOCK_AGENT_API_TOKEN>`**（與 watchlist 同 token，**不是** admin token）。

| 動作 | Method + Path | 無資料行為 |
|---|---|---|
| 看**最新一日**報表 | `GET /api/v1/institutional/report/latest` | 完全無報表 → **200 + 空殼**（非 404） |
| 看**指定日**報表 | `GET /api/v1/institutional/report/{report_date}`（`YYYY-MM-DD`） | 該日不存在 → **404** |
| 列近 N 日有報表的日期 | `GET /api/v1/institutional/report/history?limit=N` | 無資料 → **200 + 空清單** |

```bash
curl "http://192.168.11.99:8080/api/v1/institutional/report/latest" \
  -H "Authorization: Bearer $AI_STOCK_AGENT_API_TOKEN"
```

回傳的是 job 落地的 structured payload（client 自行渲染表格/曲線圖），重點欄位：
- `report_date`、`total_scanned`、`delayed`（資料延遲旗標）
- `top_items[]`：每檔含 `symbol`/`name`、外資/投信/自營各自 `net` + `volume_ratio`/`percentile`/`streak_days`/`net_flow_3d/5d/20d`（**total 與 per-investor 都有**）、`direction`、`score`、`reason_codes`、近 20/60 日 net flow `series`
- `market_summary_text`：**客觀數據摘要**（規則引擎 template，可追溯，無 LLM）
- `ai_view_text`：**AI 研判**（明確標示為 AI 觀點，**由 DeepSeek 產生**；`DEEPSEEK_API_KEY` 未設則為 `null`）

注意:
- Top N = **10 檔**（每檔一列,`combined`=外資+投信同向只影響排序與 badge,不重複出現）。
- **coverage gate**:當日法人列數 / 有量股數 < `INSTFLOW_MIN_COVERAGE`(預設 0.7)時,job 判定資料不完整 → **不發 report、改發 admin 告警**,該日 `latest` 仍可能查到上一個完整日。
- 命名:對外顯示用「最新個股法人資金流向報表 / Latest Institutional Flow Report」;與第二層的「盤中大單」明確區隔。

### 訊號計分卡 API（Signal Scorecard，M11 / TASK-186，dashboard client 呼叫）

實作在 `ai_stock_agent/api/scorecard.py`,router prefix `/api/v1/scorecard`,**唯讀**。
資料由排程 job `signal_scorecard_{monthly,weekly}`(月初 04:00 / 週六 09:30)算好各 source 的
命中率/IC/累積報酬序列落地到 `signal_scorecard`,本 API **只查詢、不觸發計算**。
**所有 endpoint 都要 `Authorization: Bearer <AI_STOCK_AGENT_API_TOKEN>`**(與 watchlist 同 token,**不是** admin token)。
完整設計見 `docs/Signal_Scorecard_Spec.md`。

| 動作 | Method + Path | 無資料行為 |
|---|---|---|
| 看**最新一期**計分卡 | `GET /api/v1/scorecard/latest?period_kind=month\|week` | **200 + 空殼**(`period_label=null`, `rows=[]`) |
| 看**單一 source 跨期趨勢** | `GET /api/v1/scorecard/source/{source}?period_kind=month&horizon=20&limit=24` | **200 + 空清單** |

```bash
curl "http://192.168.11.99:8080/api/v1/scorecard/latest?period_kind=month" \
  -H "Authorization: Bearer $AI_STOCK_AGENT_API_TOKEN"
```

回傳 `data` 內含 `period_kind`/`period_label`/`rows[]`;每個 row:`scope`(stock/market)、`source`、
`horizon`(1/5/20)、`sample_size`、`directional_hit_rate`(命中率%)、`information_coef`(IC,score 與實際報酬
等級相關)、`avg_return`、`strategy_series`/`cum_return_series`(折線圖序列 `[[date, cum_return%], ...]`)。
- **折線圖 client 自畫**:本 API 回 series 陣列,dashboard 用 vendored chart.js 畫(不產 server PNG)。
- **Dashboard 第二頁**:`GET /dashboard/scorecard`(HTML shell,token 由前端輸入)= 個股/大盤兩軸的命中率+IC 排行表
  + 多源累積報酬折線圖。主 dashboard `/dashboard/intraday` 頂部有導覽連結。**改 HTML/route 後需 restart api 才生效**(烤在記憶體)。
- 「哪種分析最貼近實際漲跌」= 多種訊號源(封老師五維/法人原始+精選/訊號等級/盤中大單/美日先行/macro 18 碼/週報觀察/抄底/**阮慕驊避險模型:綜合 `hedge_risk` + 6 子項 `hedge_turnover/leverage/giant/ma/external/fundamental`(讀 `market_risk_snapshot` 子分,各對 TWII 計分)**/**全球風險溫度判讀 global_risk(macro 聚合)**/**盤前開盤衝擊 premarket_alert**/**財經節目分析師判讀:個股綜合 `analyst_view` + 各面向 `analyst_technical/chip/fundamental/theme/macro`(由 `analyst_method` 的 category 拆,逐股逐日對前向報酬算 IC →「哪一類分析師理由最準」)+ 大盤 `analyst_market`(對 TWII)**)
  統一進 `signal_observation` 台帳對前向報酬計分。個股軸對自身、大盤軸對 TWII;每日 `signal_collect`(22:10)採集、`signal_universe`(22:30)維護焦點宇宙。
  - **分析師訊號接法(2026-06-28)**:`analyst_method`(DeepSeek 已結構化抽 category+claim+direction)→ `map_analyst`/`map_analyst_market`(`analysis/signal_collectors.py`,curated)→ 進台帳。`sim_plan` 波段 `_swing_signals` **只吃 `analyst_view`**(一檔一個分析師共識訊號),per-category 5 源是「scorecard-only」(同批點名衍生,餵進去會在 equal-weight `signal_component` 重複計權),由 `_ANALYST_SCORECARD_ONLY` 在 SQL 排除。歷史回放:`backfill_signal_observations --sources analyst_view,...`。dashboard `/dashboard/analyst` 每檔分行顯示 `[category] 具體claim`(站上季線/營收創高/毛利率53%→66%…)+ 大盤 `market_methods`。

### 第二層：盤中大單即時偵測（Intraday Monitor，M9 / TASK-174 上線）

> **這是「第二層」**,與「第一層」（盤後個股法人資金流向報表,見上節）**正交**:
> - **偵測對象**:第二層偵測「盤中大單」當作**代理主力**——**不是**官方三大法人歸類,而是
>   盤中即時 tick 聚合出的大額成交（一個近即時、不等收盤的早期訊號）。第一層才是盤後權威法人數據。
> - **時間軸**:第二層盤中即時（09:00–13:30）、第一層盤後（約 21:15 後）。兩者互補不重疊。

**常駐服務（launchd `intraday_monitor`）**：
- Label `com.linus.aistockagent.intraday_monitor`,跑 `.venv/bin/python -m ai_stock_agent.jobs.intraday_monitor`,
  常駐 event loop:訂閱 watchlist∪當日成交值前 N 大 → 聚合/偵測大單 → 維護 realtime Top10 dashboard
  state(`intraday_dashboard_state`,供第三層 179 dashboard API 讀)。
- **`KeepAlive = { SuccessfulExit = false }`**:job 內 `is_trading_session` 守衛——非台北交易時段不啟動主迴圈、
  收盤 13:30 以 `exit 0` 正常結束。`SuccessfulExit=false` 表示**只有 crash(非 0 退出)才被重拉**,
  正常收盤退出不會被 launchd 立刻重啟(避免 `KeepAlive=true` 與「非交易時段不啟動」衝突造成 13:30 後 busy-loop)。
- **⚠️ 需 Shioaji 金鑰才實際啟用**:未設金鑰時 daemon 仍可載入但無真 tick(可注入
  `FakeStreamingQuoteSource` 做端到端測試,不引入 `shioaji`)。

**健康檢查（獨立 script,刻意不混進 API 的 healthcheck.sh）**：
- `scripts/healthcheck_intraday_monitor.sh`:用 `launchctl print system/com.linus.aistockagent.intraday_monitor`
  判斷 loaded/pid。
- **僅 Asia/Taipei 週一~五 09:00–13:30 交易時段才要求 pid 存在**;非交易時段印 `SKIPPED` 並 `exit 0`、
  不告警、**不** auto-recover(crash 重拉由 `KeepAlive=SuccessfulExit=false` 負責)。
- 交易時段內若未 loaded / 無 pid → 送 `DISCORD_ADMIN_WEBHOOK`(best-effort)並 `exit 1`。
- 手動巡檢:`./scripts/healthcheck_intraday_monitor.sh`。

### Schema 來源

**單一 SQL canonical**：`ai_stock_agent/db/schema.sql`。核心表：`stock`、`price_daily`、`fundamental`(P1.5 加 `capex_outflow`/`capex_raw` 欄)、`user_watchlist`、`analysis_result`、`audit_log`、`llm_usage`、`kpi_tracking`、`glossary`、`hidden_link`、`policy_event`、`macro_indicator_snapshot`；技術指標模組(M7)：`corporate_action`/`adjusted_ohlcv`/`daily_indicators`/`daily_signals`/`backtest_*`；**HedgeMonitor 加的**：`monthly_revenue`(P1c)、`market_risk_snapshot`+`market_risk_alert_log`+view `market_risk_latest`(P2)。

> **沒有 Alembic / migration 工具**——改 schema 直接編輯 `schema.sql` 並開新 SQL patch（如 commit `431ddfc` 加上 `llm_usage.success` 欄位的方式）。新環境用：
```bash
psql ai_stock_agent -f /Users/linus/Ai_Stock_Agent/ai_stock_agent/db/schema.sql
```
名詞庫 seed：`python -m ai_stock_agent.scripts.import_glossary seeds/glossary_seed.yaml`
個股 universe：`python -m ai_stock_agent.scripts.import_stock_universe`

### HedgeMonitor 避險監測模型（2026-06-09 完成 P0~P3，已上線）

> 阮慕驊「逃生/支撐/跨市場/黑馬」盤面邏輯系統化。**規格(權威來源、細節別在這重述)**：`docs/Hedge_Monitor_Spec.md`(經五輪 review)；來源邏輯 `docs/Stock_Monitor.md`。輸出時機/頻道見 §5「HedgeMonitor 輸出何時/哪裡看」。

**做什麼**：每日 14:45 初版 / 21:35 終版算 **0–100 風險分(6 子分)+ regime(性質軸)+ 8 指標白話文 Discord 報告**，跨級才 admin 告警；紅燈終版觸發抄底掃描；盤前 08:30 開盤衝擊預警；盤中 11:00 即時抄底。

**資料 code（都進 `macro_indicator_snapshot`，除非另註）**：
- P0：`TWII`、`TW_TURNOVER`(成交總值)、外圍 `^SOX`/`^IXIC`/`^N225`/`^KS11`(由 06:30 macro job 抓)。
- P1：`TW_MARGIN_BALANCE`(融資，**21:00 才更新**故 21:05 補)；`monthly_revenue` 表(月營收，running-max `is_ath`)。
- P1.5：`fundamental.capex_outflow`(I-4)、`US_NONFARM`/`US_NONFARM_CHG`(FRED，II-4)。
- P3：`TW_FOREIGN_NET`/`TW_TRUST_NET`/`TW_DEALER_NET`(三大法人)。

**純函式模組(無 I/O、好測)**：`analysis/hedge_monitor.py`(6 scorer+regime+combine)、`hedge_inputs.py`(序列數學)、`hedge_report.py`(8 指標 embed)、`dip_scan.py`(抄底三公式)、`premarket.py`(開盤衝擊)、`capex.py`(去 YTD 累計)；`data/quote_source.py`(EOD/MIS 即時，P3d)。jobs/repo:`jobs/hedge_monitor.py`、`market_breadth.py`、`premarket_alert.py`、`dip_scan.py`、`nonfarm.py`、`scripts/sync_capex.py`、`db/repositories/hedge_repo.py`。

**關鍵設計決定(別再重新推導)**：
- **兩個正交軸**：`risk_score`(過熱→燈號)與 `regime`(II-4 性質→建議動作)。低信心 regime 不下重判。
- **缺值優雅降級**：資料不足的子分從分母剔除(非補 0)、`coverage_ratio<0.7` 或核心缺失不發 admin。**新裝/資料未滿時 coverage<100%、低信心是正常**(均線要 240 天歷史)——跑 `backfill_market_breadth` 一次即補滿。
- **槓桿 z-score 防爆**：用 `融資5日% − β·指數5日%` 的 z-score(不除法，指數趨0/負不爆)。
- **Capex 是 YTD 累計**：FinMind 現金流 PP&E 要 `analysis/capex.py` 去累計成單季，QoQ 才有意義。
- **日期語意**：台股資料對齊 `target_trade_date`(最近交易日)不用 calendar today；回測 point-in-time 用 `available_at`/FRED vintage。
- **MIS 即時(P3d)**：`dip_scan --realtime` 用 TWSE MIS 近即時 + 即時 TAIEX 基準，抓不到退 EOD。未來可接富邦 fubon-neo / 永豐 Shioaji(實作 `QuoteSource` 即 drop-in；國泰零售無公開行情 API)。

### 常用查詢範例

```bash
# 連 DB
psql -h 127.0.0.1 -U ai_stock_agent -d ai_stock_agent

# 表清單
psql -d ai_stock_agent -c '\dt'

# 看最近 LLM usage
psql -d ai_stock_agent -c "SELECT model_used, COUNT(*), AVG(latency_ms) FROM analysis_result WHERE created_at > NOW() - INTERVAL '24 hours' GROUP BY model_used;"

# 健康檢查
curl http://192.168.11.99:8080/healthz
curl http://127.0.0.1:8080/healthz
```

---

## 5. 其他入口 + 過去 session 慣例

### 維運腳本

| 腳本 | 用途 |
|---|---|
| `scripts/restart_all.sh` | postgres + 全 jobs 一鍵重啟（見 §2） |
| `scripts/launchctl_manage.sh {load,unload,restart,status}` | 安裝 / 拔除 / 查詢全部 launchd jobs(見 docs/Scheduled_Jobs.md) |
| `scripts/healthcheck.sh` | 手動觸發 health probe（launchd 每 5 分鐘也跑一次） |
| `scripts/backup_helper.sh` | `pg_dump --format=custom` → gzip → age 加密 → 保留 7 份。**必須**設 `BACKUP_PASSPHRASE` 才會跑。 |
| `scripts/daily_status.sh` | 掃 healthcheck.log + `*.err.log`，組 JSON 餵 `ai_stock_agent.jobs.daily_status` 送 Discord report |
| `scripts/install_postgres_daemon.sh` | 修補 brew 寫壞的 postgres plist（讓它用 `UserName=linus` 而非 root） |

### Python module 入口（launchd 都呼這幾個）

| Module | 對應 launchd job |
|---|---|
| `ai_stock_agent.main:app` | `api`（uvicorn）|
| `ai_stock_agent.jobs.daily_report` | `daily`（14:00）|
| `ai_stock_agent.jobs.weekly_report` | `weekly`（週六 09:00）|
| `ai_stock_agent.jobs.kpi_monthly` | `kpi`（月初 03:00）|
| `ai_stock_agent.jobs.data_sync price` / `fundamental` | `datasync_price` / `datasync_fundamental` |
| `ai_stock_agent.jobs.data_retention` | `data_retention`（月初 03:00）|
| `ai_stock_agent.jobs.leading_indicator_{us,jp}` | `leading_indicator_*` |
| `ai_stock_agent.jobs.glossary_unverified` | `glossary_unverified`（週一 08:00）|

排程 endpoint `/api/v1/scheduled/kpi_calc` 目前回 `501`——月報用 launchd job 不走 HTTP（避免雙重觸發）。

### 一次性 / 補資料腳本（不在 launchd）

- `python -m ai_stock_agent.scripts.import_glossary <yaml>` — 灌名詞庫
- `python -m ai_stock_agent.scripts.import_stock_universe` — 灌個股清單
- `python -m ai_stock_agent.scripts.sync_price_daily` / `sync_fundamental` — 手動補日線 / 基本面
- `python -m ai_stock_agent.jobs.institutional_flow` — 個股三大法人買賣超流向 job（TASK-164）。手動觸發；也有 launchd `institutional_flow`（週一～五 21:15）自動跑
- `python -m ai_stock_agent.scripts.backfill_institutional_report [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD]` — Signal Scorecard:回填個股法人精選報表(institutional_flow_report_daily)歷史。偵測純函式 + point-in-time,重用 detect_institutional_flow + build_institutional_flow_report + _report_to_payload,讀已存的 stock_institutional_daily(282天)+ price_daily volume 逐日重放 Top10,**不打 FinMind/不呼 DeepSeek/不發 Discord**。回填後重跑 backfill_signal_observations 讓 inst_report 有數月歷史,可和原始 inst_foreign/trust/dealer 並排比 IC。idempotent
- `python -m ai_stock_agent.scripts.backfill_market_risk [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD]` — Signal Scorecard/HedgeMonitor:回填阮慕驊避險模型分數(market_risk_snapshot:綜合 risk_score + 6 子分)歷史。逐交易日重用 hedge_monitor 的 _assemble(point-in-time 收集 TWII/成交總值/融資/權值股/外圍指數/capex 等已回補的上游)+ 純函式評分 → upsert,**只算分不發 Discord/不觸發抄底**。早期資料未滿時 coverage 偏低屬正常。回填後重跑 backfill_signal_observations 讓阮慕驊 6 子分+綜合有數月歷史、20 日 horizon 可算 IC。idempotent
- `python -m ai_stock_agent.scripts.backfill_macro_history [--days 420] [--codes ^VIX,^SOX]` — Signal Scorecard:用 yfinance 回填 macro 歷史日線(VIX/SOX/DXY/黃金/WTI/US10Y/人民幣/Nasdaq/日經/KOSPI)進 `macro_indicator_snapshot`。macro_indicators job 約 2026-05 才開始寫,計分卡這些大盤訊號只 ~1 月算不出 IC;回填後重跑 `backfill_signal_observations` 即對 TWII(已有 2025-01 起歷史)有數月樣本+IC。idempotent
- `python -m ai_stock_agent.scripts.backfill_signal_observations [--start-date YYYY-MM-DD] [--end-date YYYY-MM-DD] [--sources daily_signal,macro_vix] [--symbols 2353,4906]` — Signal Scorecard P4（TASK-187）：把既有歷史（daily_signals/法人/抄底/避險/macro/inst_report/五維/週報觀察）重放成 `signal_observation`，**共用 live collector 的 mapper**（不另寫映射），baseline 源照 trailing-window candidate 分層（與 live 可比），跑完做全量前向報酬回填（無 as_of cap）。idempotent。**新裝後跑一次**讓計分卡上線即有數月樣本。`--symbols` 限定持股定向回補
- `python -m ai_stock_agent.scripts.backfill_monthly_revenue [--symbols ...] [--start-year 2010]` — HedgeMonitor P1c：FinMind 月營收全期間回補進 `monthly_revenue`（running-max `is_ath` + `available_at`）。首次種子用 `--start-year 2010`，之後每月再跑增量補新月並刷新近月 `is_ath`
- `python -m ai_stock_agent.scripts.sync_capex [--symbols 2330 2454 2317] [--start-year 2018]` — HedgeMonitor I-4：FinMind 現金流 PP&E → `fundamental.capex_outflow`。**FinMind 給的是 YTD 累計值**，腳本會去累計成單季（`analysis/capex.py`），QoQ/YoY 才有意義。也有 launchd `capex`（週六）自動跑
- `python -m ai_stock_agent.scripts.backfill_market_breadth [--start-date 2025-04-15]` — HedgeMonitor：回補 `TWII`/`TW_TURNOVER`（FMTQIK 月迴圈）+ `TW_MARGIN_BALANCE`（FinMind 範圍）歷史。**新裝後跑一次**讓 `leverage`/`ma` 子分立刻有料（均線要 240 天歷史），coverage 一次到 100%，不用等好幾週累積。預設回補 420 天
- `python -m ai_stock_agent.jobs.dip_scan [--as-of YYYY-MM-DD] [--realtime]` — HedgeMonitor P2.5 抄底/黑馬掃描（阮慕驊三公式：相對大盤強勢+收盤近高+開盤未跌停，月營收 ATH / EPS YoY / 轉型主題排序）。**非排程**：`hedge_monitor_final` 判紅燈時自動 in-process 觸發；此指令供手動掃描。送 report webhook。
  - `--realtime`（P3d）：用 **TWSE MIS 公開即時**（`mis.twse.com.tw`）報價 + 即時 TAIEX 當基準把公式 1 升級成近盤中，MIS 抓不到的退 EOD（`data/quote_source.py` `MisQuoteSource`/`EodQuoteSource`）。MIS 延遲數秒~分、ToS 灰色、best-effort。
  - 使用方式：
    ```bash
    python -m ai_stock_agent.jobs.dip_scan              # 盤後 EOD(預設)
    python -m ai_stock_agent.jobs.dip_scan --realtime   # 盤中近即時(MIS;要在交易時段 09:00-13:30 跑才有意義)
    ```
  - **未來考慮（券商 API）**：接券商即時行情 SDK 取代 MIS（更穩、真即時）—— **富邦 fubon-neo** 或 **永豐 Shioaji**（官方 Python、零售免費）。寫一個實作 `QuoteSource` 的 adapter 注入即 drop-in（`run_dip_scan_job(quote_source=...)`），dip_scan 零改動。**國泰證券零售只有 GUI 看盤、無公開個股行情 API**（2026-06 查證），不適用。
- `python -m ai_stock_agent.scripts.backtest_report` / `retest_golden_dataset` — 回測 / 黃金資料集驗證
- `python -m ai_stock_agent.scripts.bench` — 性能 benchmark

### HedgeMonitor 輸出何時 / 哪裡看（P2 + P3）

> 大盤避險監測的各項資訊出現時機與頻道。規格見 `docs/Hedge_Monitor_Spec.md`。

**排程自動（jobs 安裝後，`launchctl_manage.sh load`）：**

| 資訊 | 何時 | Discord 頻道 | 條件 |
|---|---|---|---|
| 8 指標風險報告（含 P3a 籌碼面：外資/投信買賣超） | **每天 21:35 終版**（14:45 初版） | report | 每天都有 |
| 跨級紅燈告警 | 21:35 跨級時 | admin + mention | 風險升級才發 |
| P3c 開盤衝擊預警（台指期夜盤+美股） | **每天 08:30** | red→admin / amber→report | **只在隔夜下跌(amber/red)才發**，平靜日不發 |
| P3b 抄底掃描（含融資清洗標記） | **紅燈日 21:35 後自動 in-process** | report | 只在大盤紅燈日 |

**手動觸發（會真的送 Discord）：**
```bash
python -m ai_stock_agent.jobs.hedge_monitor --run-type final              # 8 指標報告(含籌碼面)
python -m ai_stock_agent.jobs.market_breadth --codes TW_FOREIGN_NET TW_TRUST_NET TW_DEALER_NET  # 補當日三大法人
python -m ai_stock_agent.jobs.premarket_alert                            # P3c 開盤衝擊(只在 amber/red 才送)
python -m ai_stock_agent.jobs.dip_scan                                   # P3b 抄底(EOD)
python -m ai_stock_agent.jobs.dip_scan --realtime                        # P3d 盤中近即時(MIS;交易時段才有意義)
```
- 自然會看到：明晚起每天 21:35 報告多一行籌碼面（P3a）；下次隔夜大跌 → 隔天 08:30 開盤衝擊（P3c）；下次大盤紅燈日 → 當晚自動抄底掃描（P3b）。
- 不發 = 正常：P3c 平靜日不發、P3b 非紅燈日不跑（避免噪音），要看就用上面手動指令。

### 過去 session 慣例（從 git log / FSD / ISOLATION_README 留下的）

- **不要把 `.venv/` / `__pycache__/` / `.DS_Store` / `*.pyc` 加進 commit**——dispatcher 的 `SYSTEM_PREFIX` 已明文禁止，但人手改檔時要自律。
- **commit message 用 Conventional Commits**（`feat:` / `fix:` / `chore:` / `docs:` ...）——看 recent commits 一致用這風格。
- **healthcheck 失敗時 HTTP code 顯示 `000`** 是 curl 沒拿到任何回應的情況（commit `400ab5b` 修了重複的 `000`）。Discord 看到 `000` 等同 timeout，不是 0 byte response。
- **watchlist 批次掃描 per-symbol timeout = 60 秒**（commit `db1553c`，從 25s 改上來）。如果再撞超時，先想清楚是 LLM 慢還是 yfinance 慢，不要直接拉更高。
- **datasync_price 熔斷後會 cooldown resweep 殘餘 symbol**（`sync_price_daily.py`，commit `f29d364`）。`_sync_price_daily` 共用單一 `TWSEAdapter`（熔斷器全 symbol 共享）：TWSE 來源短暫逾時連敗 3 次就 `open`，60s recovery window 內後續 symbol 全被 fast-fail，整輪可能 `success=0 failed=78`（如 2026-05-29 早上）。現在改成**最多 `MAX_SYNC_PASSES`（=2）輪**：單輪結束後只對「被熔斷擋掉」（`DataAdapterCircuitOpenError` → `circuit_open=True`）的 symbol 睡 `recovery_timeout + 5s` buffer、`reset_circuit_breaker()` 再重掃；`no rows`（下市/冷門）與真錯誤不重試。
  - **判讀**：正常日 `elapsed_ms` 約 43–60s 且無 resweep log。若某次 `elapsed_ms` 暴增到 ~147s + err.log 出現 `circuit-open resweep: N symbol(s) blocked` → 熔斷觸發過但已自動補回，**不用人工介入**。只有看到 `circuit breaker still open after 2 passes` 才代表來源連 cooldown 後都沒恢復，需要隔天或手動補抓。
  - **告警**：resweep 仍救不回時（`circuit_exhausted > 0`），`run_price_sync_job` 會自動發一則 `:rotating_light:` 到 **`DISCORD_ADMIN_WEBHOOK`**（commit 見 `feat/circuit-exhausted-alert`）。Ai_Stock_Agent **沒有自己的 Discord bot**（不像 Cleo 走 Redis outbox 發真 DM），webhook 只能發頻道，所以靠 `<@DISCORD_ADMIN_USER_ID>` mention 來 ping 你（該 env 選填，見 `config/.env.example`；沒設就只發頻道不 tag）。告警是 best-effort：webhook 沒設或送失敗只 log、不會拖垮 sync job。
  - **手動補單檔缺口**：`python -m ai_stock_agent.scripts.sync_price_daily --symbols 1301 1503 ...`（incremental，從各 symbol 的 `MAX(trade_date)+1` 起抓；回 `no rows` 代表來源當天就是沒該股資料，例如停牌/無成交，非 bug）。查缺口 SQL：比對某交易日 vs 前一交易日的 symbol 差集。單股單日缺（如 1303 在 2026-05-29）通常是真的無成交，不影響彙總報告，不必追。
- **KPI 月報送 `DISCORD_REPORT_WEBHOOK`，不是 admin**（commit `19b35af` 修了 routing）。其他失敗 / DOWN 告警才送 admin。
- **EduRPG 是隔壁專案**（`/Users/linus/edurpg`，env prefix `EDURPG_*`，base branch `master`）。Dispatcher 內建 guard：路徑含 `edurpg` 字串就拒跑。**不要** 在同 shell 同時設 `EDURPG_*` 和 `AISTOCK_*`。完整守則見 `ISOLATION_README.md`。
- **Cleo 是另一個隔壁專案**（`/Users/linus/cleo`，用 `uv` / `pnpm`、daemon label 不同）。Dispatcher 沒擋 `cleo` 字串，但同 shell 也不要混 env。
- **沒有 Alembic / 沒有 migration tool**——schema 直接改 `ai_stock_agent/db/schema.sql`，新欄位另外開 patch SQL（如 `431ddfc` 加 `llm_usage.success` 的方式）。
- **相依用 `constraints.txt` 鎖定（repo root，TASK-181）**——repo 相依幾乎無版本上界，dispatcher 每個 worktree 都 fresh `pip install -e ".[dev]"`，沒有 lockfile 就會抓到比 production（`/Users/linus/Ai_Stock_Agent/.venv`，已驗證可跑的凍結組合）更新的數十個套件，造成**只在 fresh venv 出現**的全量 pytest 紅（asyncpg fixtures 在 starlette 1.3 / anyio 4.14 下「attached to a different loop」+ 422 常數 deprecation 行為變更）。`constraints.txt` 由 main 已驗證 `.venv` 的 `pip freeze` 產生（fully-pinned `==`，去掉 editable 自身行），dispatcher 的 `Invoke-PipInstall` 自動帶 `-c constraints.txt`。`fastapi<0.137` 上界（`pyproject.toml`）與 `pytest-asyncio>=0.23.8,<0.24` pin 是第二層保險。
  - **升級相依 SOP**：① 在 **main 的 `.venv`** 升級套件並跑 `ruff check . && pytest -q` 驗綠 → ② `pip freeze | grep -v -e '-e ' -e ' @ ' > constraints.txt`（去掉 editable 自身）→ ③ 用**全新 venv** 驗證 `python3 -m venv /tmp/v && /tmp/v/bin/pip install -e ".[dev]" -c constraints.txt && /tmp/v/bin/pytest -q` 全綠 → ④ 連同放寬的 `pyproject.toml` 上界一起 commit。**絕不**直接手編 `constraints.txt` 個別版本。

---

## 6. GitHub Actions / Self-hosted Runner

### 6.1 現況（2026-05-23）

- **Ai_Stock_Agent**：repo 沒有 `.github/workflows/`，**沒有任何 GitHub Actions CI**。`dispatcher` 本機跑 `ruff` + `pytest` 就是品質 gate。
- **Cleo**：私人 repo，原本三個 job（node / lint / test）跑在 GitHub-hosted Ubuntu，每 push 約 15 分鐘 × 月底用爆 2000 免費分鐘。**已遷移到 self-hosted runner** 跑在這台 Mac mini（label `mac-mini-cleo`，目錄 `~/actions-runner-cleo/`），workflow 內已加 `concurrency.cancel-in-progress` 與 `paths-ignore`。
- **edurpg**：只有 GitHub 自家的 `Copilot cloud agent` dynamic workflow，量極小（2 次/月），不需處理。
- **OpenClawBot**：0 workflows，不需處理。

### 6.2 Runner 物理位置與管理

| 項目 | 路徑 / 指令 |
|---|---|
| Runner 工作目錄 | `~/actions-runner-cleo/` |
| Runner 工作快取（job checkout、_tool） | `~/actions-runner-cleo/_work/` |
| LaunchAgent plist | `~/Library/LaunchAgents/actions.runner.linus213-art-Cleo.mac-mini-cleo.plist` |
| 即時日誌 | `tail -f ~/Library/Logs/actions.runner.linus213-art-Cleo.mac-mini-cleo/Runner_*.log` |
| 停 / 啟 / 狀態 | `cd ~/actions-runner-cleo && ./svc.sh {stop|start|status}` |
| 完全移除 runner | `./svc.sh uninstall && ./config.sh remove --token $(gh api -X POST /repos/linus213-art/Cleo/actions/runners/remove-token --jq .token)` |

**注意事項**：
- Runner 是 **user LaunchAgent**，重開機後要 `linus` 使用者**有 GUI session 或 auto-login** 才會自啟。Mac mini 目前**沒有開 auto-login**，所以重開後 runner 會 offline。**自動補救機制**見 §6.5（Cleo dispatcher 的 `Ensure-CleoRunnerOnline`）；想要無條件自啟就轉 LaunchDaemon（要 sudo + 改檔案權限）。
- plist 的 `EnvironmentVariables` 裡保留了 `AGENT_TOOLSDIRECTORY=/Users/linus/actions-runner-cleo/_work/_tool`。**目前未使用**（已改用 `setup-uv` 帶 Python 跳過 `setup-python` 的 toolcache 路徑衝突），但留著無副作用，未來若有 workflow 真的要用 `actions/setup-python@v5` 可能就會 work。
- Runner 跟 Ai_Stock_Agent 的 launchd jobs 共用 Mac mini CPU。Cleo CI 約 5 分鐘/次，平常 idle 不會干擾 14:00 daily_report / 06:30 macro_indicators。如果未來實測有衝突，plist 可加 `<key>Nice</key><integer>10</integer>` 降 CPU 優先級。

### 6.2.1 重開機後 runner 不會自啟，怎麼辦？

三條路按建議度排序（詳見 §6.5）：

1. **不做事，靠 Cleo dispatcher 自癒**（現行作法）—— 下次跑 `pwsh ./tools/agent_dispatch_cleo_mac.ps1 ...` 時會自動 `launchctl load`、等 runner 上線、再開始 dispatch。Cleo 是這個 runner **唯一觸發源**，所以「沒人跑 dispatcher 的時候 runner 沒上來」就跟「沒事情要跑」是同步的，剛好不用煩惱。
2. **SSH 進 Mac mini 手動啟動**——`launchctl load ~/Library/LaunchAgents/actions.runner.linus213-art-Cleo.mac-mini-cleo.plist` 或 `cd ~/actions-runner-cleo && ./svc.sh start`。
3. **啟用 macOS auto-login**——`System Settings → Users & Groups → Automatically log in as linus`，一勞永逸，但實體被偷風險升。

不採用 LaunchDaemon 轉換的理由：dispatcher 自癒已經完全覆蓋實務場景，加 sudo 結構不划算。

### 6.3 未來：要給 Ai_Stock_Agent 或 edurpg 加 CI

GitHub Runner 是**單 repo scope**。一個 runner 只能服務一個 repo（除非把 repo 移到 Organization 用 org-level runner）。所以再加一個 CI repo 時，要**裝第二個 runner instance**，步驟如下：

```bash
# 1. 拿一次性 registration token（1 小時內有效）
TOKEN=$(gh api -X POST /repos/linus213-art/<REPO_NAME>/actions/runners/registration-token --jq .token)

# 2. 開新目錄、下載 runner（最新版查 https://github.com/actions/runner/releases）
RUNNER_VER="2.334.0"   # 用最新版
RUNNER_DIR="$HOME/actions-runner-<repo-slug>"
mkdir -p "$RUNNER_DIR" && cd "$RUNNER_DIR"
curl -sSL -o runner.tar.gz "https://github.com/actions/runner/releases/download/v${RUNNER_VER}/actions-runner-osx-arm64-${RUNNER_VER}.tar.gz"
tar xzf runner.tar.gz && rm runner.tar.gz

# 3. 配置（換 --url / --name / --labels）
./config.sh --unattended \
  --url https://github.com/linus213-art/<REPO_NAME> \
  --token "$TOKEN" \
  --name "mac-mini-<repo-slug>" \
  --labels "self-hosted,macOS,ARM64,mac-mini-<repo-slug>" \
  --work "_work" --replace

# 4. 註冊 LaunchAgent + 啟動
./svc.sh install && ./svc.sh start
```

接著在那個 repo 寫 `.github/workflows/ci.yml`，**參考 Cleo 的範本**（commit `cdec728` 之後的版本）：

```yaml
on:
  push:
    branches: [main]
    paths-ignore: ['**.md', 'docs/**', '.gitignore', 'LICENSE']
  pull_request:
    branches: [main]
    paths-ignore: ['**.md', 'docs/**', '.gitignore', 'LICENSE']

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true     # ← dispatcher 連推時自動砍 stale run

jobs:
  test:
    runs-on: [self-hosted, macOS, ARM64]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3   # ← 不要用 actions/setup-python，會撞 /Users/runner 權限
        with:
          enable-cache: true
          cache-dependency-glob: "uv.lock"
          python-version: "3.11"   # Ai_Stock_Agent 用 3.11；Cleo 用 3.12
      - run: uv sync --all-packages   # 或 pip install -e ".[dev]"，依專案而定
      - run: uv run python -m ruff check .
      - run: uv run python -m pytest -q
```

**重要陷阱**（從 Cleo 遷移踩過）：
- ❌ **不要用 `actions/setup-python@v5`**：它的內部 install script 在 self-hosted Mac 上會試圖 mkdir `/Users/runner/hostedtoolcache`，失敗。即使設 `AGENT_TOOLSDIRECTORY` 也救不了——GHA worker 有環境隔離 bug。**改用 `astral-sh/setup-uv@v3` 帶 `python-version`**。
- ❌ **runs-on 不要寫 `[self-hosted]` 單一 label**：會抓到任何 self-hosted runner。要寫 `[self-hosted, macOS, ARM64]` 或更精確的 `[self-hosted, mac-mini-<repo-slug>]`。
- ⚠️ **私人 repo 才能用 self-hosted runner**：公開 repo 任何外部 PR 都能在你機器上跑 code，安全惡夢。Ai_Stock_Agent / Cleo / edurpg 目前都是私人，OK。
- ⚠️ **每個 repo 一個 runner instance**：不能共用。要清楚命名（`actions-runner-aistock`、`actions-runner-edurpg`），不要混。

### 6.4 故障排除

- **Runner 顯示 offline**：`./svc.sh start` 或檢查 `~/Library/Logs/actions.runner.*/Runner_*.log`。
- **Workflow stuck at "Waiting for a runner"**：labels 不符。`gh api /repos/linus213-art/<REPO>/actions/runners --jq '.runners[] | {name, status, labels: [.labels[].name]}'` 查 runner labels。
- **Runner 被搶完所有 CPU**：plist 加 `<key>Nice</key><integer>10</integer>`，stop+start 生效。
- **Job 跑到一半 host 重開機**：runner 自動斷線，GitHub 那邊 job 顯示 failed/cancelled。`launchctl load` 重新拉起來就好，下個 commit 觸發新 run。

### 6.5 Cleo dispatcher 的 runner 自癒機制

> 在 Cleo repo（`/Users/linus/cleo`），不在 Ai_Stock_Agent。這節是 cross-reference，方便未來 session 找到。

`/Users/linus/cleo/tools/agent_dispatch_cleo_mac.ps1` 在 pre-flight 階段呼叫 `Ensure-CleoRunnerOnline`（commit `b0fac8d1`，2026-05-23）：

| 行為 | 觸發條件 |
|---|---|
| 跳過（什麼都不做） | `-DryRun` 或 `-SkipRunnerCheck` |
| 印「already online」並繼續 | pgrep 找到 `actions-runner-cleo/bin/Runner.Listener` |
| `launchctl load` plist → 等 30 秒 polling → 印 "now online" | runner 不在 |
| 印 "did not come online within 30s" 並繼續 dispatch | launchctl load 後 30 秒內沒上來（不阻擋 push，CI 會 queue） |

**設計理由**：Cleo 是這個 runner 的**唯一觸發源**。沒人跑 dispatcher 就代表沒 CI 要跑，所以「dispatcher 啟動時自癒 runner」剛好覆蓋所有實務場景，不需要 LaunchDaemon 或 auto-login。

**手動覆蓋**：
- `-SkipRunnerCheck`：暫時禁用自癒（例如 runner 正在升級）
- `cd ~/actions-runner-cleo && ./svc.sh start`：純手動拉起來，跳過 dispatcher

如果未來 Ai_Stock_Agent 或 edurpg 加了 CI（見 §6.3），它們各自的 dispatcher 也應該各自加類似的 `Ensure-*RunnerOnline` 函式——但**各 dispatcher 不要互相啟動別人的 runner**，保持隔離。
