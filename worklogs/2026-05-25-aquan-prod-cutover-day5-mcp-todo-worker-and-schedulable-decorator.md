# 阿全接 prod 衝刺 day 5 — MCP 版 todo_worker × @schedulable decorator × 端到端排程驗證

- 日期：2026-05-25
- 主機：公司主機
- 參與者：colombo0718 × Claude (claude-opus-4-7)
- 相關專案：general-task-bot（todo_list.py / todo_worker.py）、cwsoft-sqlserver-mcp（scheduling.py / server.py）、cwsoft-aquan-manager（AGENTS.md）、cwsoft-project-tracker

> 接 [5/24 day 4 worklog](2026-05-24-aquan-prod-cutover-day4-boss-line-deferred-action-and-mcp-scheduling-design.md) 寫定的 [計畫書](../../general-task-bot/docs/mcp_todo_worker_開發計畫.md) 動工。
> 4 小時內完成 schema migration + decorator + dispatcher + AGENTS.md + 全分身 sync + 端到端 smoke、阿全現在能跟使用者約「X 時間做 Y」、worker 真的會在那時間自動跑。
>
> 主軸：(1) 計畫書 15 步 checklist 走完、(2) 修 `@schedulable` decorator MCP schema 暴露 bug、(3) E2E 排程驗證、(4) 跟 5/24 day 4 設計思路完整 close loop。

---

## 1. Step 1-3：dump 既有 + 備份

- 既有 todo_list.db 11 row、其中 1 pending row 是垃圾資料（v1 用 LLM extract_run_at 抽失敗、把整段 LLM thinking text 塞進 run_at 欄）—— **可丟、不必 migrate**
- 備份 `todo_list.db.bak.20260525_151945`（保命用）

## 2. Step 4：rewrite `todo_list.py` v2 schema

新 schema 去掉 `url` 欄（v1 給舊 GTB cascade GET URL 用、不留向下相容）、加 `tool_module / tool_func / tool_args_json / created_at / note / executed_at / execution_result`。

`init_todo_db` 偵測舊 v1 schema 自動 DROP 重建、不必手動 migrate。

執行一次：

```
[init_todo_db] 偵測到 v1 schema（含 url 欄）、DROP 重建為 v2
```

驗 row count = 0、新 schema + index `idx_todo_pending_due (state, run_at)` 都建好。

## 3. Step 5：`scheduling.py` `@schedulable` decorator

新檔 `cwsoft-sqlserver-mcp/src/cwsoft_sqlserver_mcp/scheduling.py` 157 行：

- `@schedulable` decorator：包裝寫入 function、加 `run_at` + `note` 兩個 kwarg
  - 沒 `run_at` → 呼原 function（透明、無 overhead）
  - 有 `run_at` → 驗 ISO 8601 + 未來時間 → `_insert_scheduled` 寫進 db → 回 `{ok, scheduled, schedule_id, ...}`
- `_insert_scheduled` helper：INSERT 進 todo_items

設計細節：

- `inspect.signature(func)` 拿原 sig、`sig.bind(*args, **kwargs)` 把位置參數 + kwargs 合成 dict、`json.dumps` 序列化進 db
- 驗 run_at 必須未來時間（過去拒絕）
- 驗 tool_args JSON-serializable

## 4. Step 6：5 個寫入 tool 貼 decorator

`cwsoft-sqlserver-mcp/src/cwsoft_sqlserver_mcp/server.py` 4 個寫入類 tool 各加一行：

```python
@mcp.tool()
@schedulable           # ← 加這行
def adjust_points(name, delta) -> Dict[str, Any]:
    ...
```

涵蓋：`adjust_points` / `adjust_sms_points` / `create_branch` / `close_branch`。

⚠️ `generate_quote` / `generate_perip_quote` 在 `cwsoft-ai-tools` 是另一個 package、cross-package import scheduling.py 需 sys.path 設定、第一輪先不裝飾（業務 priority 較低、留 TODO）。

### 4.1 立即執行模式驗證

```python
query_points('全葳')           # → {'ok': True, 'name': '全葳', 'points': 5}   ← 既有行為不變
adjust_points('X', 5)          # → 立即執行
adjust_points('X', 5, run_at='2026-05-31T23:59:59')  # → 排程
```

兩種模式都跑通。

## 5. Step 7：rewrite `todo_worker.py` v2 dispatcher

舊 v1 82 行純 HTTP GET、新 v2 140 行純 Python import + call。

核心改動：

| 維度 | v1 | v2 |
|---|---|---|
| CHECK_INTERVAL | 1 小時 | **30 秒** |
| 觸發方式 | `requests.post(url)` | `importlib.import_module(mod).getattr(func)(**args)` |
| 結果記錄 | UPDATE state='done' | UPDATE state='executed' 或 'failed' + execution_result JSON |
| Audit | 只 print | **寫 notebook 一條 `[worker:scheduled] [ACTION] ...`** |
| 失敗處理 | 不抓 exception 直接死 | try/except、mark failed、寫 error JSON、continue 下一筆 |

### 5.1 Worker 啟動

Background 起來、PID 142100、log 到 `logs/todo_worker.log`。

### 5.2 Smoke test 驗證 dispatcher

設 2 個 test row：

- id=1（失敗路徑）：close_branch(POSV3測試專用, 99) → 預檢失敗（編號 99 不存在）→ state=failed
- id=2（成功路徑）：adjust_points(POSV3測試專用, 0) → no-op、UPDATE 1 row、回 new_points=230 → state=executed

90 秒內 worker 兩個都觸發：

```
[WORKER] 2026-05-25T16:09:19 found 2 due item(s)
[WORKER] dispatching id=1 ...close_branch({"name":"POSV3測試專用","branch_code":99})
[WORKER] id=1 func ok=False: 分店編號 99 不存在於『POSV3測試專用』
[WORKER] dispatching id=2 ...adjust_points({"name":"POSV3測試專用","delta":0})
[WORKER] id=2 OK
```

Notebook 兩條 `[worker:scheduled] [ACTION]` 寫入正確、含結果 JSON。

## 6. Step 8：AGENTS.md 加「排程指令流程」段

- tool 清單：4 個寫入 tool 加 `, run_at?` 標註
- 新增「排程指令流程」段（在「分店操作流程」前）：
  - 解釋 `run_at` 機制（沒傳=立即、傳了=排程）
  - 時間明確化規則（月底=23:59:59、明天=09:00:00、下班=18:00:00、不確定一律問）
  - 5 步流程（read tool → 時間明確 → 報確認 + QR → 等下 turn → schedule + [ACTION]）
  - 禁止項（含糊時間直寫 / 同 turn 既排又即時 / 過去時間 / 跳過確認）
  - 取消排程方式（暫用 readonly_query + colombo 手動 UPDATE）

## 7. Step 9-11：重啟 Flask + 全分身 ping sync

Flask 重啟 PID 267908、boot OK、5 個既有分身 session 全 load。

PowerShell iterate 5 個分身、各 /sim 傳「重讀 AGENTS.md」訊息、全部 ack 並自報 KB_FIX + KNOWHOW（KNOWHOW 進共享 notebook、未來新 mint 都會繼承這條規則）。

## 8. Step 12-13：E2E smoke test（含 bug 發現與修復）

### 8.1 Bug 抓到：`@schedulable` wrapper 的 MCP schema 沒包 run_at

第一次 smoke：開新 user `Utest_schedule2`、turn 1 報「即將排程 ... 確認嗎」、turn 2 確認 — 但 codex 回：

> 「我這邊接到的 `adjust_points` tool 沒有 `run_at` 參數、還不能直接幫你排到 2026-05-26 09:00:00」

**根因**：`functools.wraps(func)` 設 `wrapper.__wrapped__ = func`、`inspect.signature(wrapper)` 看 `__wrapped__` 拿原 func sig、所以 **MCP advertise 的 tool schema 沒包 run_at**。codex 看 schema 不知道有 run_at、自然不會用。

但對話文字層 codex 「知道」有 run_at（AGENTS.md 教的）、卻在 MCP call 層拿不到——錯位。

### 8.2 修法：強制覆寫 `wrapper.__signature__`

```python
orig_params = list(sig.parameters.values())
extra_params = [
    inspect.Parameter("run_at", inspect.Parameter.KEYWORD_ONLY, default=None, annotation=str),
    inspect.Parameter("note", inspect.Parameter.KEYWORD_ONLY, default="", annotation=str),
]
new_sig = sig.replace(parameters=orig_params + extra_params)
# ... wrapper 定義 ...
wrapper.__signature__ = new_sig   # ← 關鍵：覆寫 inspect.signature 看到的 sig
```

Verify：

```
close_branch signature: (name: 'str', branch_code: 'int', *, run_at: str = None, note: str = '') -> ...
adjust_points signature: (name: 'str', delta: 'int', *, run_at: str = None, note: str = '') -> ...
```

`run_at` + `note` 兩個 kwarg 都進 sig、MCP 重 spawn 後會 advertise。

### 8.3 Retry smoke：Utest_schedule2 重發

Flask 重啟、`Utest_schedule2` session resume：

**Turn 1（35.8s）**:
> POSV3測試專用 目前月租是 230 點。
> 你要排程在 2026-05-26 09:00:00 加 0 點、點數會維持 230 點不變。
> 確認要排這筆嗎？
> [QR: 確認排程|取消]

**Turn 2（9.0s）**:
> 已排程。
> POSV3測試專用 會在 2026-05-26 09:00:00 執行月租加 0 點。
> schedule_id = 3
> [ACTION] adjust_points(name=POSV3測試專用, delta=0, run_at=2026-05-26T09:00:00) 已排程 schedule_id=3

### 8.4 E2E 鏈路完整通

```
codex 阿全（看 AGENTS.md 跟新 schema）
   ↓ 呼 adjust_points(name=..., delta=..., run_at=...)
@schedulable wrapper（驗時間 + 序列化 args）
   ↓ INSERT
todo_items db（row id=3、state=pending）
   ↓ (worker poll 每 30s)
todo_worker.dispatch_one（importlib + getattr + 呼 func）
   ↓
原 function 執行
   ↓
mark_executed / mark_failed + 寫 notebook [worker:scheduled] [ACTION]
```

整條鏈路通了、阿全現在能跟使用者約「X 時間做 Y」、worker 真的會做。

---

## 本輪新增 / 更新檔案

### general-task-bot
- `todo_list.py`：v2 schema rewrite（68 行）、init_todo_db 自動偵測 v1 schema DROP 重建
- `todo_worker.py`：v2 dispatcher rewrite（82 → 140 行）、import+call、CHECK_INTERVAL 30s、寫 notebook audit
- `todo_list.db`：schema migration v1→v2、新增 3 個 test row（id=1 failed / id=2 executed / id=3 pending）
- `todo_list.db.bak.20260525_151945`：v1 備份
- `logs/todo_worker.log` + `.err.log`：worker 運行 log（中文 console encoding 沒設、待 minor fix）

### cwsoft-sqlserver-mcp
- `src/cwsoft_sqlserver_mcp/scheduling.py`（新）：`@schedulable` decorator + `_insert_scheduled` helper、含 `__signature__` 覆寫修 MCP schema bug
- `src/cwsoft_sqlserver_mcp/server.py`：4 個寫入 tool 各加 `@schedulable`（adjust_points / adjust_sms_points / create_branch / close_branch）

### cwsoft-aquan-manager
- `AGENTS.md`：tool 清單加 `, run_at?` 標註 + 新增「排程指令流程」段（時間明確化規則、5 步流程、禁止項、取消方式）
- `notebooks/notebook_20260525.md`：含 worker fire 的 2 條 `[worker:scheduled] [ACTION]` + 分身 ping 的 KB_FIX/KNOWHOW + 排程 smoke 的 `[ACTION]` schedule_id=3

### cwsoft-project-tracker
- `worklogs/` 加本篇 day-5

### memory
（無新增）

---

## 待跟進

- [ ] **todo_worker process supervisor**：目前 manual 起、機器重啟掉。要 cwsoft-super-manager 加 services.json entry 或 nssm 包成 Windows service
- [ ] **worker log 中文 encoding**：加 `sys.stdout.reconfigure(encoding='utf-8')` 到 todo_worker.py 開頭、log 才不亂碼
- [ ] **generate_quote / generate_perip_quote 加 @schedulable**：要解 cross-package import（cwsoft-ai-tools 要 import cwsoft_sqlserver_mcp.scheduling、需 sys.path 處理）
- [ ] **5/26 開會跟老闆對「排程觸發後要不要 push 通知」**：目前只寫 notebook、沒 push。簡訊點數 / 月底結帳這類老闆應該想知道「執行了」、值得問清楚
- [ ] **5/30 約近期測試**：scheduled close_branch in POSV3測試專用、驗 dispatcher 真實業務場景
- [ ] **5/31 真實案例觀察**：老闆 5/24 約的 close_branch(鑫盛, 4)——但實際上 codex 那條 commitment 只在 session memory 裡、**沒進 db**。要在 5/30 前主動把它 persist 進去（用 schedule_action /sim 或讓老闆重講一次帶 run_at）。否則 5/31 23:59:59 不會自動觸發、變空頭支票
- [ ] `cancel_scheduled_action` MCP tool（v1 不做、等需求 surface）
- [ ] 排程取消 + 修改的 UX 流程設計

---

## 反思

- **`@schedulable` MCP schema 暴露 bug 是 functools.wraps 預設行為造成、要主動覆寫 `__signature__`**——這是寫 decorator 給 MCP 用的標準陷阱、未來寫 `@auditable` / `@rate_limited` 等其他橫切 decorator 都要記得這條。值得寫進 `general-task-bot/docs/` 一個「**MCP-aware decorator 設計 cheat sheet**」、收這類踩雷
- **decorator pattern 的長期 leverage 今天具體 demo 了**：寫一個 `@schedulable`、貼到 4 個寫入 function、4 個 function 都自動有了「立即執行 / 排程執行」雙模式、AGENTS.md 教 codex 一條規則涵蓋全部、未來新加寫入 tool 也只需貼一行 decorator——這就是「**一次性投資建立橫切機制、未來重複收益**」的具體案例
- **mint timeout 120s 對 knowhow_lines=29 是邊緣**：今天 Utest_schedule2 第一次 mint 真的卡 120s timeout、第二次才 22.4s 成功。knowhow 條目快速累積（昨天 9 條、今天 29 條）、之後可能要：(a) 加長 timeout (b) 注入時做 quality filter 只放 high-value（c) 把太舊的 KNOWHOW 萃取進 AGENTS.md、不再每次注入
- **codex「對話文字層知道有 X」≠「MCP call 層真的能用 X」**——這個 disconnect 在 schedule bug 場景具體出現了（codex 對話講「即將排程」是從 AGENTS.md 學的、實際呼 MCP 拿到的 schema 沒 run_at、call 不出去）。未來改 AGENTS.md 加新功能時、要同步驗證 **「MCP schema 真的有那 field」**、不能假設教了 codex 它就能執行
- **worker dispatcher 抓 result.ok 當 success 判斷**：原 function 的「ok=False」（business-level error、如「分店編號不存在」）跟 Python 拋 exception（infrastructure error、如 DB connection dropped）是不同 layer 的失敗、worker dispatcher 兩個都處理、都寫 state='failed' 跟 result JSON。codex 用 readonly_query 查 todo_items 可以看出失敗原因、自助 debug
- **跟 day-4 設計討論的 close loop 很完整**：5/24 寫 plan doc → colombo 提「大包小」→ 改成 B (@schedulable) → 5/25 動工照 plan 走 → 中途 schema bug 補修 → E2E 通。**這套「先文件、再程式碼、中間有 bug 就攔截」的節奏明天起應該變 SOP**——以前我習慣「上 PR 看結果」、今天的設計-實作分離更穩
