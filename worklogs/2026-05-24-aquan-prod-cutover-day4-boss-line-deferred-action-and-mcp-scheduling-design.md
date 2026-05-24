# 阿全接 prod 衝刺 day 4 — 老闆 LINE 真實用 close_branch + 排程需求浮現 + MCP 版 todo_worker 架構設計

- 日期：2026-05-24
- 主機：公司主機
- 參與者：colombo0718 × Claude (claude-opus-4-7)
- 相關專案：cwsoft-aquan-manager、cwsoft-sqlserver-mcp、general-task-bot（todo_list.db / todo_worker.py）、cwsoft-project-tracker

> 接 [5/23 day 3 worklog](2026-05-23-aquan-prod-cutover-day3-branch-mcp-knowhow-sync-and-multicast.md)。day 3 上完 create_branch / close_branch、day 4 上午**老闆親自 LINE 用 close_branch 流程**——用完當下又冒出新需求（排程 / 月底執行）、開了排程功能設計討論、定案後寫進 TODO 明天動工。
>
> 主軸：(1) 老闆首次實際用 close_branch、(2) 排程需求浮現、(3) MCP 版 todo_worker 架構選擇 A1（純 Python import）、(4) MCP function 多 entrypoint 的設計含義。

---

## 1. 老闆 LINE 真實用 close_branch 流程（5/24 上午 11:01）

colombo 截圖 + chat log 對照、看到老闆（user_id 末 6 `ae0df1`）跟阿全的對話：

```
[11:00] 用戶：「我要在月底停用水源台哥大」

[11:01] 阿全（用 match_customer_name 校正完）：
       「如果你是指 鑫盛 的 水源-台哥大、我可以先幫你確認分店資訊。
        但要先說清楚：目前我只有「立即軟下架」的 tool、還沒有「排程到月底自動停用」的 tool。
        你是指 鑫盛 / 水源-台哥大 這間嗎？
        [QUICK_REPLY: 是，鑫盛這間|不是這間|取消]
        [WANT_NEW_TOOL] 分店軟下架的排程工具、可指定在月底自動執行 close_branch」

[11:01] 用戶：「是，鑫盛這間」

[11:01] 阿全（用 list_branches 確認）：
       「確認到了、是 鑫盛 的 水源-台哥大、分店編號 4、目前還是啟用中。
        但我現在只有「立即軟下架」的 tool、還沒有「排程到 2026/05/31 自動停用」的功能。
        你可以選：
        2026/05/31 當天再叫我停用、或
        如果你要、我現在就先幫你軟下架
        [QUICK_REPLY: 5/31再叫你|現在停用|取消]」

[11:01] 用戶：「5/31再叫你」

[11:01] 阿全：「好、2026/05/31 你再叫我、我再幫你把 鑫盛 / 水源-台哥大 做軟下架。」
```

### 阿全一段對話做對的 6 件事（端到端 demo）

1. ✅ **客戶名 match**——「水源台哥大」校正成「鑫盛 / 水源-台哥大」（`match_customer_name`）
2. ✅ **list_branches 確認**——回報分店編號 4、目前啟用中
3. ✅ **QuickReply 兩次**都用對情境（客戶名確認 + 排程 vs 立即）
4. ✅ **誠實「沒這個 tool」**——說「沒排程功能、5/31 當天再叫我」、不亂編
5. ✅ **`[WANT_NEW_TOOL]` 標記**——把「分店軟下架排程工具」記進 notebook
6. ✅ **寫入確認流程**——即使使用者明確要求、也問「現在 vs 5/31」、不擅自執行

是 prod 系統可用的硬證據。每條規則（match / list / 不懂就問 / QuickReply / 寫入確認）都被一個真實業務場景觸發 + 阿全自己 dispatch 對。

### 對照昨晚 (5/23 23:13-23:20) colombo 自己的 dev 測試

colombo 用趙士豪帳號 (`5c3751`) 在 `POSV3測試專用` DB 上測 create / close branch：

```
23:16 [ACTION] create_branch(name=POSV3測試專用, shop_name=大林店) 已新增成功
23:20 [ACTION] close_branch(name=POSV3測試專用, branch_code=20) 已軟下架成功
```

dev 端先測 → 隔天 prod 老闆真實用 → end-to-end 連起來。

---

## 2. 排程需求浮現 + commitment 持久化問題

阿全跟老闆約「5/31 你再叫我」——這個 commitment 目前**只在 codex session memory 裡**、Flask 重啟 / session 換 mint 就掉。**這不是真排程、只是社交契約**。

老闆要的是「**真的 5/31 23:59:59 自動跑、不用我再叫**」、不是「我那天記得叫」。

→ **需要真排程機制 persist 到 db**。

### 舊 GTB 早有排程機制

複習了 5/16 worklog + PROJECT.md：

- `todo_list.db`（SQLite）— 排程任務 storage
- `todo_worker.py` — 背景跑、polling、到 run_at 觸發
- `extract_run_at` — 舊 GTB extractor 用 LLM 抽自然語言時間 → ISO

但 codex 阿全 **完全沒接這層**、目前所有 `[ACTION]` 都是即時執行。

---

## 3. MCP 版 todo_worker 架構設計（A1 vs A2 vs B）

### 問題

舊 todo_worker 設計是「到時間 GET sqlgate URL」（基於 mission*.json 的 url_template）。但 close_branch 是**直接 SQL（cwsoft_sqlserver_mcp 的 server.py function）**、不走 HTTP。worker 沒「呼 MCP function」的能力。

### 3 條 sub-option

| 方案 | 機制 | 速度 | AI 中介 | 失誤機率 |
|---|---|---|---|---|
| **A1 純 Python import** | worker 直接 `import cwsoft_sqlserver_mcp.server` + `close_branch(name=..., branch_code=...)` | < 1s | 無 | 極低（已 verified function、零 AI 推理）|
| A2 ephemeral codex | worker spawn `codex exec`、給「請執行 close_branch(...)」訊息、codex 自己 dispatch | 10-30s | 有（codex 中介） | 偏低、但 AI 仍可能漏 |
| B 新建純 codex 版 scheduler | 結構化標記 + 自家 scheduler、不依賴 todo_list.db | 視實作 | 視實作 | 視實作 |

### 結論：A1

排程觸發**不需要 AI 推理**——spec 已 frozen（什麼 tool、什麼 args、什麼時間）、純執行。AI 中介只在「**使用者→排程指令**」那一步需要（codex 阿全在跟使用者對話、決定要排哪個 spec 進去）、一旦 spec 寫進 db、後續執行是純工程問題。

A2 養 codex 太重、token 成本浪費。
B 重造 storage layer 沒必要、todo_list.db 已穩定跑 N 個月。

### A1 含義：MCP function 可多種 entrypoint 呼叫

這次討論衍生出一個值得 internalize 的架構洞察：

```
@mcp.tool()
def close_branch(name: str, branch_code: int) -> dict:
    # 純 Python 邏輯
```

`@mcp.tool()` 只是「對 MCP 協定註冊」、function 本身**仍是普通 Python function、可從多種 entrypoint 呼叫**：

| 呼叫者 | 呼法 |
|---|---|
| codex agent（阿全跟使用者對話）| MCP 協定（JSON-RPC stdio） |
| 排程 worker（todo_worker、cron）| `import + call`（直接 Python） |
| 批次腳本（colombo ad-hoc）| 同上 |
| 單元測試 | 同上 |
| 未來客服 AI（cs_codex）| MCP 協定 |
| 內部 dashboard | 同上或 HTTP wrap |

**寫一次、N 種觸發方式都能用**——不必為了「給 worker 跑」開 API endpoint、不必為了「給測試跑」改 mock。MCP function = business logic 的 single source of truth。

對 cwsoft 之前「sqlgate HTTP 一層、todo_worker 直接 SQL 一層、UI 又一層」的分散實作是真正的改善——三層三處 bug 變一處。

---

## 4. TODO 寫定、明天動工

寫進 [`general-task-bot/TODO.md`](../../general-task-bot/TODO.md) [TODO 2026-05-24] [FEATURE] MCP 版 todo_worker、含完整 spec：

- 新 MCP tool `schedule_action(tool_module, tool_func, tool_args, run_at, note)`
- todo_list.db 加 column `tool_spec_json` + `executed_at` + `execution_result`（保留 url_template 向下相容舊 GTB 排程）
- todo_worker.py dispatcher 改造：若 row 有 tool_spec_json → import + call
- AGENTS.md 加「排程指令流程」段
- 邊角問題：poll 間隔、push 通知、取消修改、失敗重試
- 估工 3-4 小時

5/31 真實場景測法：**5/30 先約一個近期測試案例驗 dispatcher**、5/31 才真的當天觸發老闆要的那筆。

---

## 本輪新增 / 更新檔案

### general-task-bot
- `TODO.md`：加 [TODO 2026-05-24] [FEATURE] MCP 版 todo_worker 完整 spec

### cwsoft-aquan-manager
（無改動）

### cwsoft-project-tracker
- `worklogs/` 加本篇 day-4

### memory
（無新增）

---

## 待跟進（明天 5/25 動工）

- [ ] 動 MCP 版 todo_worker（schedule_action MCP tool + db schema + dispatcher）
- [ ] 寫 day 5 worklog
- [ ] 5/30 先排一個近期測試 schedule_action（驗 dispatcher）
- [ ] 5/31 監控真正觸發老闆要的 close_branch(鑫盛, 4)
- [ ] 5/26 開會跟老闆對「通知簡楓怡的時機」（5/23 colombo 提）

---

## 反思

- **「老闆親自實際用」是檢驗系統的最強指標**——比 unit test、smoke test、stress test 都重要。一段對話 6 個 do-right 行為被自然觸發 + dispatch 對、表示「不懂就問 / match / list / QuickReply / 寫入確認 / WANT_NEW_TOOL」這套規則是真在 prod work、不只是 sim 條件下 work
- **使用者的「commitment 表達」≠「真的會被執行」**——阿全跟老闆說「好、5/31 你再叫我」、session memory 有、但 Flask 重啟就掉。**這次 expose 出 codex session memory 跟業務 commitment 該分離 persist 的設計缺口**。排程機制就是補這個缺口
- **A1 vs A2 設計選擇的核心**：「**這一步需不需要 AI 推理?**」——排程觸發的「執行」步驟、spec 已 frozen、純工程、不需 AI。AI 中介只該插在「真的需要推理 / 對話 / 判斷」的環節
- **MCP function 多 entrypoint 性質我之前沒意識到、今天才 internalize**：之前的 broadcast_self_intro.py 我用 `from tools.cwsoft_ai_tools.server import generate_quote` 直接 call、那時就在用這個性質了、但沒抽象成設計原則。今天 colombo 問「MCP 不只可以被 agent 使用、也可以被一般的腳本呼叫 是嗎」、才意識到這是值得明確說的設計優勢。下次跟同事介紹架構時可以拿這個當賣點
- **跨日工作的時間錯位**：system context 寫的「今天」跟 wall-clock 已差兩天（5/22 vs 5/24）、寫 worklog 時要靠 chat log mtime / notebook 日期 / 對話內容推真實日期、不能 default 信 system date
