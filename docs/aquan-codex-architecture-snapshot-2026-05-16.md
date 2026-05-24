# 阿全經理 × Codex 架構快照 — 2026-05-16

> 給 cwsoft 週會老闆 + 同事報告用。本篇是**現況快照**、不是長期規劃。
> 長期規劃見 [`cwsoft-ai-mcp-and-principal-architecture.md`](cwsoft-ai-mcp-and-principal-architecture.md)。
>
> 過去三場對話的 worklog（早 / 午 / 晚）：
> - [上午：LINE bot 上線、對話完整化、log 重組、MCP+Principal 長期架構規劃](../worklogs/2026-05-16-gtb-codex-line-bot-and-mcp-architecture.md)
> - [下午：MCP server MVP 上線 + capability 鎖死 + AGENTS.md instruction + GTB 角色重新定位](../worklogs/2026-05-16-mcp-server-mvp-and-gtb-vs-codex-reflection-2.md)
> - [傍晚：adjust_points 落地 + 寫入確認雙層 + tracker/cs_kb MCP 擴充 + session 翻車救回](../worklogs/2026-05-16-adjust-points-write-confirm-layers-and-tracker-cs-kb-mcp.md)

---

## 1. 一句話定位

**阿全經理（LINE OA `@526fdbzo`）= cwsoft 內部助手 AI。**
LINE webhook 收訊 → Flask 中繼 → codex CLI（gpt-5.4）推理 → 透過 **MCP 工具** 連接公司各資料源（檔案、文件、SQL Server）。對象：colombo + 同事；**不對客戶**。

跟舊版 GTB（基於 mission/prompts JSON cascade 那一支）的差別：
| 維度 | GTB（舊、prod 仍在跑）| 阿全 codex 版（新、測試中）|
|---|---|---|
| 推理層 | 多階段 LLM 抽取 + 規則路由 | 單一大模型 + tool use |
| 工具擴充 | 改 mission/prompts 設定檔 | 加 MCP tool 函式 |
| 記憶 | 無（每訊息獨立）| 有（單 SESSION_ID 跨 turn 記憶）|
| 自由度 | 嚴格、答錯難 | 高、答錯易（要靠 AGENTS.md 鎖）|

---

## 2. 架構圖

```
                  LINE OA「阿全經理(測式)」@526fdbzo
                            │
                            │ webhook POST /callback/@526fdbzo
                            ▼
              ┌──────────────────────────────────┐
              │  Flask: gtb_codex.py (port 6010) │
              │  cwd = cwsoft-aquan-manager      │
              │   - 讀 oa_registry.json 驗簽章   │
              │   - LINE 跳動點點 (loading)      │
              │   - 攔截 [TAG] → notebook        │
              │   - 寫 chat_log per session      │
              └──────────────┬───────────────────┘
                             │ codex exec resume <SID>
                             ▼
              ┌──────────────────────────────────┐
              │      codex CLI (gpt-5.4)         │
              │   -C cwsoft-aquan-manager        │
              │     → 自動載入 AGENTS.md         │
              │   --disable shell_tool           │
              │   --disable multi_agent          │
              │   --disable apps                 │
              │   --dangerously-bypass-sandbox   │
              │     (Windows job-object workaround) │
              └──────────────┬───────────────────┘
                             │ MCP stdio
              ┌──────────────┴──────────────────┐
              │                                 │
              ▼                                 ▼
   ┌─────────────────────────┐    ┌──────────────────────────┐
   │  cwsoft_ai_tools (MCP)  │    │  cwsoft_sqlserver (MCP)  │
   │  in general-task-bot/   │    │  in cwsoft-sqlserver-mcp/│
   │                         │    │                          │
   │ ・list_project_files    │    │ ・ping                   │
   │ ・read_doc              │    │ ・list_databases         │
   │ ・list_kb_docs          │    │ ・list_tables            │
   │ ・read_kb_doc           │    │ ・list_stored_procedures │
   │ ・list_tracker_docs ★   │    │ ・list_functions         │
   │ ・read_tracker_doc ★    │    │ ・get_table_schema       │
   │ ・list_cs_kb_docs ★     │    │ ・get_sp_definition      │
   │ ・read_cs_kb_doc ★      │    │ ・search_objects         │
   │ ・match_customer_name★★ │    │ ・readonly_query         │
   │                         │    │ ・query_points           │
   └────────┬────────────────┘    │ ・adjust_points (寫) ★   │
            │                     └─────────┬────────────────┘
            ▼                               ▼
   檔案系統（4 個資料源）：               SQL Server
   - aquan-manager/                     - POSConfig（設定中心）
   - cwsoft-ai-customer-service/        - POSV3shared（共享）
     kb-engineer/  ← 工程師視角         - 各資料庫設定（維運）
     kb-customer/  ← 客戶面向 SOP       - 167 客戶 DB
   - cwsoft-project-tracker/
     worklogs/ meetings/ minutes/
     projects/ business/

   ★ = 今日（5/16）新增   ★★ = 今晚新增
```

---

## 3. capability 鎖死策略（codex 看不到 / 不能用的東西）

| 風險面 | 處理方式 |
|---|---|
| 開 shell 跑任意命令 | `--disable shell_tool` |
| 自己再開 sub-agent 繞檢查 | `--disable multi_agent` |
| 用 GitHub plugin / mempalace 等其他 MCP | `--disable apps` + `~/.codex/config.toml` 全域關 |
| `web.run` 上網 | 沒 flag 能關、靠 AGENTS.md 硬規則 + audit |
| `apply_patch` 改任意檔 | 同上、靠 AGENTS.md + audit |
| OS 沙箱 | Windows job-object 跟 MCP server subprocess 衝突、改開 `--dangerously-bypass-approvals-and-sandbox`、由前述工具鎖代償 |

策略一句話：**鎖工具表面、不靠 OS 沙箱**。

---

## 4. 安全模型：寫入類 tool 的雙層保險

```
                      ┌──────────────────────┐
   使用者打 LINE → │  對話層保險            │ ← AGENTS.md prompt 規則
                      │  (codex 自我約束)      │   codex 必須先問「確認嗎?」
                      └────────┬─────────────┘   等下一 turn 明確同意
                               │
                               ▼
                      ┌──────────────────────┐
                      │  機制層保險            │ ← MCP tool SDK 強制
                      │  (codex 無法繞)        │   不可逆 → 兩段式 preview/commit
                      └────────┬─────────────┘   可逆 → 單 tool（信任對話層）
                               │
                               ▼
                           寫入 DB
```

依「**執行錯了能不能 5 分鐘內復原**」分兩種：

| 類型 | 機制層 | 對話層 | 例子 |
|---|---|---|---|
| **可逆** | 拔掉強制兩段、走單 tool | 必須問確認 | `adjust_points` |
| **不可逆 / 破壞性** | 強制 `xxx_preview` + `xxx_commit` 兩 tool 跨 turn | 同樣要問確認 | 未來的 `drop_customer`、`revoke_license`、`delete_database` |

**關鍵釐清**：「可逆 = 拔機制層強制」**不等於**「可逆 = 不用問確認」。對話層底線永遠在。

---

## 5. 客戶名解析流程（解決語音輸入錯字）

老闆多半用 LINE 語音輸入、客戶名常聽錯（**乙烯/乙希、林一/零壹、想想/相相**）。

直接拿原字串 JOIN 客戶 DB → 永遠 0 點 → 不是查無、是名字對不上 → 後續所有客戶相關操作全錯。

**新增 `match_customer_name(query)`** 解法：

1. 任何 customer-scoped tool（query_points / adjust_points / 未來查訂單 / 出帳單）**第一步先 match**
2. 拼音相似度（pypinyin + SequenceMatcher）+ 字面相似度、雙軌取最大、substring 命中高優先
3. 依 `top1.score`：
   - ≥ 0.95 → 直接用 `top1.name`（**不必再問**）
   - 0.7-0.95 → 跟使用者確認「你是說 X 嗎?」
   - < 0.7 → 請使用者重打

實測：「乙烯」→「乙希」score = 1.0（兩個都讀 `yi xi`）、「林一通訊」→「零壹通訊行」score = 0.828、「asdfqwer」→ top 0.4 拒絕。

---

## 6. self-report 標記系統

codex 在回覆**結尾**自加 4 種標記、Flask 攔截寫進 `notebooks/notebook_YYYYMMDD.md`、使用者**看不到**、給 colombo 後台 grep。

| 標記 | 觸發情境 | 用途 |
|---|---|---|
| `[KB_FIX]` | 對 cwsoft / POS 知識理解錯、被糾正 | 持續修正 codex 對公司領域的認知 |
| `[WANT_NEW_TOOL]` | 使用者要某功能、沒對應 MCP tool | **直接當下一輪開發 roadmap** |
| `[BUG_SIGHTED]` | 觀察到資料矛盾、tool 行為異常、系統怪事 | 拋給工程師排查 |
| `[ACTION]` | 真執行了寫入動作 | 留審計軌跡、之後出事可回溯 |

---

## 7. 今日 5/16 進度（給老闆看的「做了什麼」）

### 新增能力

- **第一個寫入類 MCP tool 落地：`adjust_points(name, delta)`**——仿 sqlgate `/adjust_points`、寫入確認雙層保險（單 tool + AGENTS.md 硬規則 + `[ACTION]` 標記）
- **MCP 工具集擴充 5 個**：
  - `list_tracker_docs` / `read_tracker_doc`（5 個 category：worklogs / meetings / minutes / projects / business）
  - `list_cs_kb_docs` / `read_cs_kb_doc`（客服面向客戶 SOP、阿全當 review 角度）
  - `match_customer_name`（拼音解析、解決語音錯字）

### 驗證指標

| 測項 | 結果 |
|---|---|
| 75 題老闆題本壓測（boss_exam_v1） | 0 error、avg 10.8s、44/45 寫入確認流程守住 |
| 對話層 smoke 6 題（新 MCP tool） | 6/6 PASS、avg 20.8s |
| 客戶名解析 unit test 8 case | 全 PASS（含「乙烯/林一通訊/想想/asdfqwer」邊界）|

### 開發過程發現的問題（誠實記）

- **法博誤寫 +6 事件**：壓測中 codex 把「**現在**法博加6點」的「現在」解讀為催促 = 隱含確認 → 直接寫入。已用 `adjust_points(法博, -6)` 回滾、DB 完全恢復。事故印證寫入類 prompt 規則在 corner case 會被 codex 自行擴展、可逆操作必須配回滾流程兜底
- **--fresh 翻車 + 救回**：為了讓 codex 忘記舊 tool 名稱擅自重啟、UUID 蓋掉舊 session、176KB 對話歷史險些丟失。從 `~/.codex/sessions/2026/05/16/` 撈舊 UUID 寫回 `.gtb_codex_session` 救回。教訓：codex 每 turn 重讀 MCP tool 清單 + AGENTS.md、`--fresh` 幾乎永遠不該是答案
- **webhook 404**：Flask 從錯誤的 cwd 啟動 → 讀錯 oa_registry.json → `@526fdbzo` 不在表內。修法：明確從 `cwsoft-aquan-manager/` 起 Flask、log 接到檔案

---

## 8. 老闆題本壓測得出的「下一輪 roadmap」（codex 自報）

22 個 unique `[WANT_NEW_TOOL]` payload + 多個 `[BUG_SIGHTED]`、節錄重點需求：

| 需求 | 出現次數 |
|---|---|
| 匯出客戶區間未稅帳單 / 請款單 | 5 |
| 依縣市列出客戶清單 | 2 |
| 客戶資料修改 | 1 |
| 客戶簡訊點數加值 | 1 |
| 新增客戶分店 | 1 |
| 產生請款單可加品項 | 1 |
| ... | |

外加多個客戶名找不到（雨新/林一通訊/想想/新園/吉園/吉元/全紅/華期）—— 推測題本來自舊 GTB 語音 ASR 錯字、real customers 拼錯。

---

## 9. 下一步（順序建議）

### Q2 內必做

1. **per-user session**（明天起動）：阿全要進公司群組 / 多人共用前**必須**完成。每個 line_user_id 一條獨立 codex SESSION_ID、機制層隔離跨人錯亂寫入
2. **principal / RBAC 接上**：依 `~/.claude/projects/.../memory/` 跟 [`cwsoft-ai-mcp-and-principal-architecture.md`](cwsoft-ai-mcp-and-principal-architecture.md) §7-8 規劃（per-user session 改路線後 setup_context 從 per-turn 降為 per-session-mint）
3. **群組 wakeword**：阿全進公司群組後、被 @tag / 喚醒詞才回、其他靜默（[TODO 已記](../../cwsoft-aquan-manager/TODO.md)）

### Q3 啟動

4. **客服 AI（cs_codex.py）**：fork 阿全架構、principals.json 加客戶 line_user_id mapping、tool policy 限縮成 `get_my_orders` / `faq_lookup` / `store_hours` 等
5. **第一批客服寫入 tool**：客戶改自己聯絡資訊 / 報修預約 等——**全走兩段式 preview/commit**（客戶端漏一次 = 商業事故、機制層保險不能省）

### Q4 目標：autonomous 10%

6. **confidence-gated handoff 機制**：codex 不確定就主動 say「我幫你轉真人」、不硬答。`[WANT_NEW_TOOL]` / `[BUG_SIGHTED]` / `[NEED_INFO]` 標記已是雛形信號、要接「→ 通知人類客服介入」protocol
7. **eval instrumentation**：上線抓「最終是否人介入」的 ground truth、跑兩週基線、再對照 codex 自報 tag 能不能預測 handoff 需求。**先 instrument、再優化**

### 預估時間軸

| 期間 | 目標 autonomous 處理率 |
|---|---|
| Q2 內 | 1-3%（純 FAQ + SOP 命中、查自己訂單）|
| Q3 | 5-7%（principal RBAC 上線後的個人化查詢）|
| **Q4 / 年底前** | **10%**（handoff protocol + 兩段式寫入機制都 tuned 過幾輪） |

---

## 10. 開放討論題（meeting agenda）

1. **per-user session 路線拍板** — 既有架構 doc §7-8 是「全域單 session + setup_context per turn 切 principal」、本週發現多人共用必須走「per-user session」、要更新 doc
2. **第二個寫入類 MCP tool 選哪個** — `adjust_points` 已上、下一個建議 `drop_customer` 樣板（不可逆兩段式 preview/commit 的教學範本）vs `modify_customer_basic`（可逆、複用 adjust_points 模式）
3. **法博 +6 事件的教訓要不要寫進 AGENTS.md 強化** — 列舉確認字眼白名單、明確排除「現在 / 快 / 立刻」等催促詞
4. **autonomous 10% 指標的 ground truth 怎麼採** — 純人工標、還是接 LINE OA 後端事件追蹤
5. **客服 AI 上線時程預期** — Q3 起跑、需要產品 / UX 配合決定 OA 帳號註冊流程
