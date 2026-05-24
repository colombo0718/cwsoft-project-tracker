# 阿全接 prod 衝刺 day 1 — 簡訊點數 + 報價單 MCP × LINE QuickReply × AGENTS.md 對話風格鐵律 × 5/19 會議入帳

- 日期：2026-05-21
- 主機：公司主機
- 參與者：colombo0718 × Claude (claude-opus-4-7)
- 相關專案：general-task-bot、cwsoft-aquan-manager、cwsoft-sqlserver-mcp、autoQuotes、cwsoft-project-tracker

> 接續 5/16 三場 worklog（早 / 午 / 晚）後、再隔三天 5/21 重啟。原本是日常維護日，colombo 中段拋出「**明天就要把阿全測試版切上 prod 給老闆用**」的目標、整天節奏從 review 轉成衝刺。
>
> 主軸：補上正式版必備的兩個寫入類 tool（簡訊點數、報價單）、加 LINE QuickReply UX、把 colombo 的領域知識寫進 AGENTS.md 智慧預設、5/19 會議記錄入帳。

---

## 1. 5/19 會議記錄入帳（day 1 起手式）

### 1.1 逐字稿放錯位置

git pull 拉到 5/19 commit `ddfdec3 新增 2026-05-19 會議逐字稿`、但檔案被放在 repo **root**（`./逐字稿-2026-0519.txt`）、不是 `minutes/` 子資料夾。阿全的 `list_tracker_docs('minutes')` 因此讀不到。

`git mv` 進 minutes/、保留 history。

### 1.2 整理會議記錄

依 `會議記錄整理模板.md` + `語者、客戶、專案名稱校正.md`、寫 [`meetings/會議記錄_2026-05-19.md`](../meetings/會議記錄_2026-05-19.md)：

- **語者校正**：語者1=羿宏（POS 開發、列印商品明細）/ 語者2=彥偉（業務策略、客服 OA、AI 信心度）/ 語者3=士豪（codex/mcp/標記系統）
- **客戶名校正**：零一/靈異 → **零壹通訊行**（上下文混音、同一客戶被 ASR 拆成兩個詞）；瑀新/瑀忻 → 宇新【待確認、字首「瑀」非「宇」、可能新客戶】
- **8 個主題**：零壹 Channel API 危機、OK App 競爭策略、公司用客服 OA、阿全 codex 展示、AI 客服信心度堅守、零壹/宇新測試、綠界金流、商品明細列印
- **16 個待辦**含負責人 / 期限 / 交付物
- **5 個未定事項**

### 1.3 從 5/19 會議直接對阿全的兩個改動

跟我這邊 5/16 已寫的 TODO 對齊：
- **per-user session 拆分**——5/19 彥偉觀察到「多人同時對阿全講話會混茶」、跟我這邊 [TODO entry](../../general-task-bot/TODO.md) 一致、確認明天動工
- **點數類型區分**——5/19 彥偉指「有人加月租點數有人加簡訊點數、阿全要會分」、今天落地（見 § 3）

---

## 2. 開會用架構快照 doc

寫 [`docs/aquan-codex-architecture-snapshot-2026-05-16.md`](../docs/aquan-codex-architecture-snapshot-2026-05-16.md)、跟既有的 `cwsoft-ai-mcp-and-principal-architecture.md`（長期規劃）並列、本篇定位「**現況快照 + 5/16 進度報告**」。

10 章節 + 5 個開放討論題（直接當 meeting agenda）：

1. 一句話定位 + 跟舊 GTB 對比表
2. 架構圖 ASCII（LINE → Flask → codex → 2 個 MCP server → 4 個檔案資料源 / SQL Server）
3. capability 鎖死策略
4. 寫入類雙層保險圖（對話層 + 機制層、可逆 vs 不可逆）
5. 客戶名解析流程（語音輸入錯字）
6. self-report 標記系統
7. 今日 5/16 進度（**誠實列了 3 個翻車**：法博 +6 已回滾、--fresh 救回、404 cwd）
8. 老闆題本壓測 [WANT_NEW_TOOL] 統計
9. 下一步順序：Q2（per-user session）→ Q3（CS AI）→ Q4（autonomous 10%）
10. 開放討論題 5 條

---

## 3. 簡訊點數 MCP tool（5/19 會議落地）

### 3.1 領域 prior

找 sqlgate `app.py /query_sms_points` + `/adjust_sms_points`、確認跟一般點數的差別：

| 維度 | 一般月租點數 | 簡訊點數 |
|---|---|---|
| Table | `POSV3Shared.dbo.線上付款_已購買授權`（授權類別=1）| `POSV3Shared.dbo.SMS_客戶點數`|
| 欄位 | `授權數量` | `剩餘點數` |
| JOIN | `分店設定檔主索引 = 主索引` | `客戶分店編號主索引 = 主索引` |
| 參數 | `delta`（點）| **`amount`（金額元）**、內部 `ceiling(amount/1.2)` 換算成點 |

### 3.2 實作

[`cwsoft-sqlserver-mcp/src/cwsoft_sqlserver_mcp/server.py`](../../cwsoft-sqlserver-mcp/src/cwsoft_sqlserver_mcp/server.py) 加：
- `query_sms_points(name)`：唯讀
- `adjust_sms_points(name, amount)`：寫入、走 AGENTS.md 寫入確認流程

實測：乙希 月租 46 / 簡訊 658；法博 月租 30 / 簡訊 0；全葳 月租 5 / 簡訊 **52576**（歷史累積誇張多）。

---

## 4. 報價單 MCP tool

### 4.1 兩種報價單分清楚

colombo 強調：**月租報價單 vs 週邊產品報價單**——兩個獨立系統、不能混。

接 autoQuotes（**HTTP**、不是 SQL）：

| Tool | endpoint | 關鍵參數 |
|---|---|---|
| `generate_quote` | `https://cwsoft.leaflune.org/api/quote` | name + charge_months + due_month + include_tax + unit_price（後四個都可選、用客戶 JSON 預設）|
| `generate_perip_quote` | `https://cwsoft.leaflune.org/api/perip` | name + paper / carbon / machine 數量 + include_tax + month |

### 4.2 實作

[`general-task-bot/tools/cwsoft_ai_tools/server.py`](../../general-task-bot/tools/cwsoft_ai_tools/server.py) 加兩個 tool、用 `requests` 打 autoQuotes、自動處理 timeout / HTTPError / JSON parse error / API ok=false。

實測 4 case 全 PASS：既有 PDF 取出 / 重產 PDF / 週邊紙卷 PDF / 無數量擋下。

### 4.3 設計決策：報價單**不走寫入確認流程**

跟舊 GTB 一致——PDF 沒副作用、可重產、出錯重叫就好。但客戶名要先 `match_customer_name`（避免出錯客戶的 PDF）。

---

## 5. LINE QuickReply 結構化標記機制

### 5.1 觸發

5/16 notebook 5/21 11:49 阿全自報 `[WANT_NEW_TOOL] LINE 選擇按鈕`——對話中（客戶名候選 / 點數類型釐清 / 寫入確認）codex 純文字反問、體驗笨。colombo 觀察到應該用 LINE 原生 QuickReply。

### 5.2 設計：跟 notebook 標記同型機制

| 機制 | 結構標記 | Flask 攔截後做的事 |
|---|---|---|
| Notebook（既有）| `[KB_FIX]` `[WANT_NEW_TOOL]` etc. | 寫 notebook、從 reply 剝除 |
| **QuickReply（新）** | **`[QUICK_REPLY: 月租點數\|簡訊點數]`** | **build LINE QuickReply payload、剝除、訊息底部跳按鈕** |

**最漂亮的點**：LINE QuickReply 按下後送的是「按鈕文字當作 user 訊息」——就是 user 回了一個訊息、走正常 webhook→codex flow、codex 看到「月租點數」就知道是回答。**完全不用 state machine、不用 callback id**。

### 5.3 實作

[`general-task-bot/gtb_codex.py`](../../general-task-bot/gtb_codex.py) 加：
- `QUICKREPLY_PATTERN = re.compile(r"^\s*\[QUICK_REPLY:\s*(.+?)\s*\]\s*$")`
- `extract_quickreply_from_reply(reply)`——容錯 `|` / `,` 兩種分隔符、>13 截斷、>20 字截斷
- `build_text_message(text, options)`——若 options 不為空就附 LINE `QuickReply` + `QuickReplyItem` + `MessageAction`
- 在 `/callback` 跟 `/sim` handler 都加進去

AGENTS.md 加「結構化標記：LINE QuickReply 按鈕」段、列三個適用情境 + 規則 + 範例對話。

### 5.4 端到端驗證

`/sim` 打「全葳加 3 點」：

```
codex 原始輸出: 請問你要加的是月租點數還是簡訊點數？\n\n[QUICK_REPLY: 月租點數|簡訊點數|取消]
Flask 攔截結果: quick_reply_options=["月租點數","簡訊點數","取消"]
使用者 LINE 看到: 「請問你要加的是月租點數還是簡訊點數？」+ 底部 3 顆按鈕
```

**阿全自己加了「取消」第三選項**（AGENTS.md 範例寫的是兩個）——表示有抓到 QuickReply 設計精神、不是死板套模板。

---

## 6. AGENTS.md 大量擴增 + 阿全 4 次重讀

一天內 4 次叫阿全 `read_doc("AGENTS.md")` 重新內化、每次自報 `[KB_FIX]` 寫進 notebook：

| 時間 | 重讀觸發 |
|---|---|
| 5/20 23:37（前晚）| 客戶名解析流程（match_customer_name）|
| 早上 | 點數類型釐清 + 寫入確認流程強化（含「催促詞不算同意」修法博事件）|
| 中午 | QuickReply 結構化標記 |
| 下午 | 報價單類型釐清 + 不懂就問鐵律 + 智慧預設 |

### 6.1 新增「不懂就問、不要硬答」對話風格鐵律

colombo 接 prod 的核心要求——「**比 GTB 更人性化**」的關鍵差異化。AGENTS.md 加一段、放在工具規則之前。7 個情境列表 + 4 條禁止：

> 禁止：看到不確定就硬挑一個答 / 看到不會做就講「好的我幫你」結果什麼都沒做 / 看到 query 0 筆就假設「沒這客戶」（可能是名字錯、先 match_customer_name）/「應該是 X 吧?」這種帶推測的肯定語氣
>
> **反問的成本永遠低於 hallucination 的成本**——使用者多打一句話 vs 你寫錯客戶資料、選錯效率。

### 6.2 智慧預設（colombo 給的領域知識）

**點數類型**：
- 沒講類型 → 預設月租（業務常識）
- 個位 / 十位數字 → 高機率月租
- 千以上、尤其萬起跳 → 高機率簡訊（金額為單位）

**月租報價單參數**：
- charge_months 不傳 → 用客戶 JSON 預設
- due_month 不傳 → 預設今天 YYYY/MM（業務規則「沒指定起算月就用本月當到期月、下個月開始算 N 個月」）

**報價單類型**（後來加的）：
- 只講「報價單」→ 一律當月租
- 只在明確說「紙卷 / 碳帶 / 條碼機 / 週邊 / 商品」字眼才走週邊

---

## 7. 翻車事件：概念延伸失誤

我把 colombo「加點預設月租」的精神**自作主張延伸**到「報價單也預設月租」、AGENTS.md 只改了 charge_months / due_month 參數預設、沒改類型判讀規則。

colombo LINE 打「幫全葳出報價單」、阿全照舊規則反問月租/週邊、被罵「**怎麼還是這樣**」。

接著我用「跟加點預設月租同樣的智慧預設精神」傳給阿全、又被打斷：「**不是啦 加點是加點、報價單是報價單**」。

**教訓存進 memory**：[`feedback-domain-rules-standalone`](../../.claude/projects/c--Users-pos-Desktop-general-task-bot/memory/feedback_domain_rules_standalone.md)——colombo 給的業務規則彼此獨立、不要用共同抽象跨領域延伸。

修法：重寫 /sim 規則訊息、standalone 框架（不引用加點規則）；AGENTS.md 報價單類型判讀改為「**只講報價單預設月租**」明列。

---

## 8. 論文影片導讀（intermezzo）

colombo 丟一個 `AI智能體編排框架的降智真相_影片逐字稿.txt.md`、檔案副檔名 `.txt.md` 修成 `.md` 移到 [`general-task-bot/docs/`](../../general-task-bot/docs/)。

論文（Simon Dennis 教授團隊）核心結論：

| 模式 | 1200 場對照 |
|---|---|
| 外部編排（LangGraph / ADK / Semantic Kernel 等切節點、外部 if/else 路由） | **15 戰 0 勝**、全敗 |
| In-context（流程結構化文本一次塞給大模型、靠注意力機制自決）| **15 戰全勝**、所有差距統計顯著 |

對映到我們：

- **舊 GTB = 教科書級「外部編排」典型**（extract_view_todos → extract_run_at → identify_needs → gather_fields → build_command 八步 cascade、每步獨立 LLM call、嚴格 enum prompt）
- **gtb_codex.py = 「in-context」模式**（單 SESSION_ID、AGENTS.md 結構化規則一次塞、codex 自決工具用順序）

5/19 彥偉那句「Codex 比 GTP 贏一截」**不是錯覺、是必然**——他直覺察覺、論文 1200 場實驗驗證。

**反思**：AGENTS.md 規則正在膨脹（一天加了 3 段：QuickReply / 不懂就問 / 智慧預設），有 token 風險。論文精神是「信任模型整體判斷、規則越少越好」、跟「每出一個 KB_FIX 就補一條規則」的習慣有點衝突。短期可接受、中期值得 review 哪些可放鬆。

---

## 9. 接 prod 衝刺評估

colombo 拋出「**明天 5/22 切 prod 給老闆用**」的目標。我原本答「Q3 末才接」太保守、用「零容錯」標準算的、改成「測試版升 prod 留 rollback」中間值是合理。

### P0（明天上 prod 前必做）

- [x] **generate_quote + generate_perip_quote** ← 今天完成
- [ ] **per-user session** ← 明天動、prod 多人 / 群組必要
- [ ] **gtb_codex.py 進 git baseline commit** ← 501 行從沒 commit 過、prod 服務不該這狀態
- [ ] **切換 OA 設定** ← `@708juxdz` 全葳小助手 webhook 改指 6010

### P1（應做、能力允許）

- [ ] `query_credentials` MCP tool（查帳密、老闆會用）
- [ ] `company_name` MCP tool（查公司名 by 客戶）
- [ ] `extend_due_date` MCP tool

### P2（視時間可丟）

- [ ] `add_customer` / `add_device` / `refresh_companies` / `fallback_greeting`

### colombo 已明確不做（這輪）

- region_customers / create_branch / close_branch / restore_test_db / rebuild_inventory / detach_customer / Shadow mode / 客服訊息推送

---

## 10. notebook 累積分析（3 天 41 條）

| Tag | 數量 | 性質 |
|---|---|---|
| `[WANT_NEW_TOOL]` | 19 | 阿全想要的新工具（roadmap）|
| `[BUG_SIGHTED]` | 14 | 資料 / 系統異常 |
| `[KB_FIX]` | 7 | 阿全自學紀錄（今天 4 條 from re-read）|
| `[ACTION]` | 1 | 法博 +6（已回滾）|

阿全當「**副作用 audit 員**」抓到的：
- 客戶主檔沒縣市 / 行政區欄位（schema 缺欄、不是 tool 缺）
- 多個客戶帳單資料缺失 / 過時：35禾名（2025/12-2026/05）、49水里遠傳（最新到 2024/06）、5晨樂（無帳單）

這些原本可能沒人發現。**阿全副作用價值很高**。

---

## 11. shared state 概念釐清（meta）

colombo 問「shared state 是什麼」、我給的解釋：

> **「我下手後、你能不能在不知情的情況下繼續正常用?」**——能 = local、不能 = shared

具體區分（表略）：重啟 Flask / 清 session / 改 oa_registry 是 shared；寫 memory / 寫 worklog / 編輯檔案沒 reload 是 local。

這個概念之前只在我 memory 裡（`feedback-confirm-before-shared-state-changes`）、今天攤開來跟 colombo 對齊。

---

## 本輪新增 / 更新檔案

### cwsoft-sqlserver-mcp
- `src/cwsoft_sqlserver_mcp/server.py`：加 `query_sms_points` + `adjust_sms_points`、總計 13 個 tool

### general-task-bot
- `tools/cwsoft_ai_tools/server.py`：加 `generate_quote` + `generate_perip_quote`、總計 11 個 tool
- `gtb_codex.py`：加 LINE QuickReply 機制（`QUICKREPLY_PATTERN` + `extract_quickreply_from_reply` + `build_text_message`、import QuickReply / QuickReplyItem / MessageAction）、callback 跟 /sim 都接
- `docs/AI智能體編排框架的降智真相_影片逐字稿.md`（從 cwsoft-aquan-manager mv 過來、副檔名修正）

### cwsoft-aquan-manager
- `AGENTS.md`：大量擴增——「不懂就問」對話風格鐵律 + 點數類型釐清（智慧預設）+ 報價單類型釐清（智慧預設）+ 寫入確認流程強化（催促詞排除）+ 結構化標記 QuickReply
- `TODO.md`：加群組 wakeword TODO（5/16 寫的、5/19 會議再次背書）
- `logs/gtb_codex.log` + `.err.log`：archived `.bak.<時間戳>` 數版（Flask 重啟兩次）
- `notebooks/notebook_20260521.md`：累積 6 條 tag（含 4 條 [KB_FIX] from re-read）

### cwsoft-project-tracker
- `docs/aquan-codex-architecture-snapshot-2026-05-16.md`：架構快照、開會用、10 章節 + 5 個 agenda
- `meetings/會議記錄_2026-05-19.md`：5/19 三人例會整理、8 主題 16 待辦
- `minutes/逐字稿-2026-0519.txt`：從 repo root mv 進 minutes/（git rename）

### memory（local-only at `~/.claude/projects/.../memory/`）
- `feedback-domain-rules-standalone`（新）：colombo 給的業務規則彼此獨立、不要用共同抽象跨領域延伸（今天踩雷）

---

## 待跟進（明天的工作）

- [ ] **per-user session** 落實——前提是 prod 切換不能有跨人錯亂
- [ ] **gtb_codex.py 進 git** baseline commit
- [ ] **OA 切換**：`@708juxdz` 全葳小助手 webhook 從舊 GTB（port 6000）改指 codex 版（port 6010）；或者反過來把 codex 版搬到 6000、舊 GTB 退到備用 port
- [ ] **P1 三個 MCP tool**：query_credentials / company_name / extend_due_date
- [ ] 切換後第一週主動 monitor notebook、看 [ACTION] / [BUG_SIGHTED] 累積
- [ ] **「瑀新 / 瑀忻」**到底是不是宇新（5/19 會議「未定」項）
- [ ] AGENTS.md 規則總量 review、看哪些可以放鬆（論文影片的提醒）

---

## 反思

- **接 prod 評估的判斷我太保守**：早上答「Q3 末才接」是用「零容錯生產級」標準。colombo 用「測試版升 prod 留 rollback」中間值算合理、且符合 cwsoft 規模的風險偏好。下次評估「能不能上 prod」要多問一句「你能容忍的 rollback 成本是?」而不是預設 enterprise SLA
- **概念延伸的錯誤**：把「加點預設月租」的精神自動延伸到「報價單預設月租」、是典型的「過度抽象」失誤。colombo 的 mental model 是「每條業務規則 standalone、不要連結」、我接抽象連結太順手了。memory 抓住、明天起注意
- **規則堆疊 vs in-context 信任的張力**：今天每出一個 corner case 就加一條 AGENTS.md 規則（不懂就問 / 智慧預設 / 催促詞排除 / QuickReply 約定）、單條都對、但累加會稀釋 codex 的整體判斷力。論文影片是來得正好的提醒、明天 review 時要留意
- **阿全已經有「自報 KB_FIX」習慣**：今天 4 次重讀都自報、不是我提醒。這是真的有把 AGENTS.md 內化、不是表面套規則。對接 prod 是好兆頭
