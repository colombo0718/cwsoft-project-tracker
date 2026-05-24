# 阿全接 prod 衝刺 day 3 — 老闆親自試用 × 分店 3 MCP × 全分身 KNOWHOW 同步 × multicast 廣播 audience filter

- 日期：2026-05-23
- 主機：公司主機
- 參與者：colombo0718 × Claude (claude-opus-4-7)
- 相關專案：cwsoft-sqlserver-mcp、cwsoft-aquan-manager、cwsoft-project-tracker、general-task-bot、cwsoft-sqlgate（既有實作參照）

> 接 [5/22 day 2 worklog](2026-05-22-aquan-prod-cutover-day2-git-baseline-caddy-and-per-user-knowhow.md)。day 2 把架構切到 prod、day 3 開始有真實使用回饋：**老闆親自試 prod 阿全、給「幹的好」三字 + 主動要新功能**。基於這個信號補了分店 3 件 MCP tool、把同步機制 + 廣播 audience 控制做齊。
>
> 主軸：(1) 老闆首試正向回饋、(2) notebook [WANT_NEW_TOOL] 全掃排 roadmap、(3) 分店 3 MCP + 軟下架設計、(4) 全分身 KNOWHOW 同步機制、(5) multicast 廣播加 exclude 機制。

---

## 1. 5/22 會議記錄入帳（含 ownership 多輪修正、教訓進 memory）

### 流程

- `git mv` 不能用（檔未進 git）、改 plain `mv ./逐字稿-2026-0522.txt ./minutes/`（同 5/19 遇過的「逐字稿放錯 repo root」狀況）
- 依模板 + 語者校正表整理成 [`meetings/會議記錄_2026-05-22.md`](../meetings/會議記錄_2026-05-22.md)、8 主題 + 16 待辦
- 語者校正：「逸鴻 / 易宏 / 益弘 / 一紅」都是「羿宏」ASR 錯字

### Ownership 多輪修正（教訓）

我寫完待辦表後、colombo 連三次糾正 ownership 太鬆：

| 我寫的 | colombo 糾正 |
|---|---|
| 宇心 demo「士豪+羿宏 共同」 | 「**對外展示都是老闆**、demo 動作 = 彥偉、後端準備分頭做」 |
| 進階行銷會員系統重做「士豪」 | 「至些也不是我」（不是 colombo 這週的 deliverable） |
| 林嶽位置說明「士豪」 | 「也不是我」 |

最終 colombo 確認的本週 only deliverable = **「進階行銷文件 html 化」一件**。

教訓：會議記錄寫待辦時、**不要從 transcript 自動延伸 ownership**——transcript 講「我這裡剛剛又加了 1 個新功能」這種第一人稱、不代表「我」就是這項的 owner、可能只是分享而已。應該以**會議當下明確指派 + colombo 確認**為準。

### 上下文混淆教訓

中段我一直在分辨「colombo 跟士豪是不是同一人」、繞了好幾圈、最後 colombo 直接告知：「**士豪就是我、colombo 就是士豪**」。memory `reference_aquan_oa_friend_roster.md` 後來補上這個對照、未來不必再繞。

---

## 2. 老闆首試 prod 阿全：「幹的好」+ 主動要新功能（核心信號）

colombo 截圖給我看：老闆 LINE 上跟阿全 @708juxdz 互動、阿全回答業務問題時誠實標「我目前無法做完整精準統計、缺對應 MCP tool」、老闆回：

> **「阿全經理幹的好、等你上班的時候把相對應的權限再做個討論」**

三個信號中：

1. **老闆親自用 prod 阿全**——5/22 切過去後第一次直接用戶級回饋
2. **「不懂就問、誠實說沒這個 tool」鐵律 work**——阿全沒亂編、老闆不僅沒嫌、反而**主動要擴充權限**。要是阿全硬答唬掉、後果完全反過來
3. **[WANT_NEW_TOOL] 機制間接 work**——老闆透過對話自然看到阿全的需求、不必 grep notebook 也能形成 feedback loop

→ 之前 5/22 我們 defer 的 `create_branch` / `close_branch`（理由「等老闆確認」）—— 現在老闆主動要了、defer 解除。

---

## 3. notebook [WANT_NEW_TOOL] 全掃 + roadmap 重排

掃 4 天 notebook、共 **23 條 [WANT_NEW_TOOL] / 去重 18 個獨立需求**。狀態分類：

| 分類 | 數量 |
|---|---|
| ✅ 5/16-5/22 sprint 已完成 | 6 個（adjust_points、adjust_sms_points、SQL Server 整套、tracker docs、QuickReply）|
| 🟡 老闆 5/23 主動要、下週開會 | 2 個（**create_branch / close_branch**）|
| 🔴 強需求多次重複（仍未做）| 2 個（**匯出客戶區間帳單 6 次**、**依縣市列客戶 4 次**）|
| ⚪ 單次出現、可緩 | 2 個（修改客戶資料、產生請款單可加品項）|
| ❌ 明確拒絕 | 1 個（外部網站查詢、AGENTS.md 硬規則禁）|

**今天 sprint 撿了 🟡 那 2 條**（老闆當天剛要、最高 ROI、跟老闆對齊度最高）。

---

## 4. 分店 3 MCP tool 落地：list_branches / create_branch / close_branch

### 設計決策：直接 SQL、不走 sqlgate HTTP

跟 colombo 對齊「**adjust_points / adjust_sms_points 走直接 SQL 重做 sqlgate 邏輯、generate_quote 走 HTTP 中繼 autoQuotes**」的既定 pattern：

| MCP tool | 走 sqlgate? | 理由 |
|---|---|---|
| `adjust_points` / `adjust_sms_points` | ❌ 直接 SQL | SQL UPDATE 簡單、容易複製 |
| `generate_quote` / `generate_perip_quote` | ✅ HTTP 打 autoQuotes | PDF 生成邏輯複雜、無法只用 SQL 做 |
| **`create_branch` / `close_branch`**（新）| ❌ 直接 SQL | 純 `EXEC SP + SELECT verify`、SQL 容易、配對 sqlgate `_sp_connection`（其實就是 `autocommit=True`、我們 db.connect 早已是）|

### 「可逆」判準

colombo 的關鍵釐清：「**不可逆與否要看真實業務影響能否復原 + 客戶權益、不是 SQL 操作層的 INSERT/UPDATE/DELETE**」。

- `create_branch` 加錯 → `close_branch` 軟下架收回、**可逆**
- `close_branch` 是「**軟下架**」、set `是否已收店=1`、資料還在、要復原把 flag 改回 0、**可逆**

兩個都不用走機制層兩段式 preview/commit、單 tool + 對話確認流程即可（同 adjust_points 模式）。這是把 [`feedback_mcp_write_tool_design`](../../.claude/projects/c--Users-pos-Desktop-general-task-bot/memory/feedback_mcp_write_tool_design.md) 的判準從「機制層的可逆」具體應用到業務層的 instance。

### 實作

`cwsoft-sqlserver-mcp/src/cwsoft_sqlserver_mcp/server.py` 加 3 個 tool（591 → 724 行）：

- `list_branches(name)` — 唯讀、列某客戶旗下分店 + 是否已收店狀態。給 codex 在 create/close 前 reference
- `create_branch(name, shop_name)` — `EXEC [各資料庫設定].dbo.CreateBranch ?, ?` + 預檢客戶存在 + 重店名警告 + **獨立 connection verify**（抓 trigger silent rollback、5/11 sqlgate 加的保險、不能省）
- `close_branch(name, branch_code)` — `EXEC [各資料庫設定].dbo.CloseBranch ?, ?` + 預檢編號存在 + 不是已下架 + verify 確認 `是否已收店=1`

`cwsoft-aquan-manager/AGENTS.md` 新增「**分店操作流程**」段、明寫：

1. match_customer_name 校正
2. list_branches 預覽
3. 報「即將 ... 確認嗎?」+ `[QUICK_REPLY: 確認|取消]`
4. **跨 turn 等明確同意**
5. 執行 + `[ACTION]` 標記

### 驗證

- syntax check ✓
- `list_branches` 唯讀真實 DB 測 3 客戶（全葳 3 / 法博 4 / 乙希 7 家、含「總倉」+ 是否已收店 flag）全 PASS
- `create_branch` / `close_branch` 沒程式測（避免污染 DB）、留給 colombo LINE 真實情境測

colombo 後續 LINE 實測新增分店、回報「**驗證過他會新增分店了**」——E2E work。

---

## 5. 全分身 KNOWHOW 同步機制（PowerShell ping 4 分身重讀）

### 問題

我用 /sim 叫阿全重讀 AGENTS.md 後、colombo 問：「**剛剛讓他重看新規則 是所有分靈都重看了嗎?**」答案是 NO——只有 `Usim` 那條重讀。其他 session（colombo 自己 5/16 那條、Utest_alice、剛 lazy mint 的趙士豪）AGENTS.md 都是各自 mint 當下的舊版、不會自動 refresh。

### KNOWHOW 機制 vs 既有 session 的 gap

| 對象 | 新規則認知方式 |
|---|---|
| 未來新 user lazy mint | ✓ boot prompt 注入近 14 天 notebook 的 KNOWHOW + KB_FIX、自動繼承 |
| 既有 session | ✗ baked-in 是 mint 當下的 AGENTS.md、新規則不會自動進去 |

### 解法：iterate SESSIONS、逐條 /sim 通知重讀

寫 PowerShell：掃 `.gtb_codex_sessions/*.session` 全 user_id、對每條走 `/sim` 帶該 user_id、訊息「請重讀 AGENTS.md」、阿全會自報 KB_FIX / KNOWHOW 寫進共享 notebook。

實測 4 條 session（colombo / 趙士豪 / Usim / Utest_alice）全 ack、全自報 tags：

```
=== Pinging U34e144c9bf7d30bc07c543a4ebae0df1 ===
  elapsed: 53.1s, session: 019e2fe9...
  tags: KB_FIX

=== Pinging U38daae74d279bef697f99a22c65c3751 ===
  elapsed: 14.6s, session: 019e4e0c...
  tags: KB_FIX,KNOWHOW

=== Pinging Usim ===
  elapsed: 13.1s, session: 019e5562...
  tags: KB_FIX,KNOWHOW

=== Pinging Utest_alice ===
  elapsed: 13.0s, session: 019e4d78...
  tags: KB_FIX,KNOWHOW
```

注意 colombo 的 5/16 老 session 跑 53 秒（其他 13-15s）——context 累積最多、合理。

### 為什麼這機制重要

**AGENTS.md 大改後一律跑這 sync**——未來如果不做、既有 session 會「不知道新規則」（包括寫入 tool 流程、QuickReply 約定、智慧預設等）、行為就會跟新 mint 的不一致。

---

## 6. 第三 + 四發廣播 + multicast audience filter 機制

### 5/22 晚上的第三發：「會議記錄整理好了」廣播

colombo 要求阿全口吻、人擬稿。我寫、200 字、走 broadcast API 寄全 OA 好友。HTTP 200。

### 5/23 colombo 點出 audience 問題

colombo:「**簡楓怡是老闆娘兼會計、一般事情不要廣播喊到她那邊去**」+ 「**花銷流水 / 洪門神 都沒在用了**」。同時下次開會要跟老闆對「**什麼情況該通知簡楓怡**」。

→ 廣播 audience 不該是無差別「全好友」、要有 filter。

### LINE API 限制

`/v2/bot/followers/ids` 撈所有 follower 需要付費版、阿全 OA 帳號不支援、無法自動列出 user_id。

colombo 從**舊版 GTB 紀錄**（每次回覆會印「用戶身分：XXX<U...>」）整理出 3 個用 id：

- 趙士豪 `U38daae74d279bef697f99a22c65c3751`
- 小謝 `U34e144c9bf7d30bc07c543a4ebae0df1`
- 林羿宏 `Ua52624cb319bcccab4d12703ef28929c`

⚠️ 修正：之前我從 OA Manager 截圖瞎猜「U38daae74 = 花銷流水」、是錯的、其實是趙士豪。memory 已修正。

### 第四發：分店功能上線、multicast 模式

修改 `broadcast_self_intro.py` 加 `--to` 旗標：

- 不傳 `--to` → broadcast API（全好友）
- 傳 `--to <id1>,<id2>,...` → multicast API（只給指定 user_id）

實寄 multicast 給 3 個 user_id、HTTP 200。簡楓怡 + 花銷流水 + 洪門神都沒收到。

### 順帶 demo

`Ua52624...`（林羿宏）user_id **不在 SESSIONS dict** ——表示羿宏到此刻為止從沒跟 codex 阿全聊過。multicast 寄出後、他第一次回應就會 lazy mint 一條全新 session、boot prompt 自動繼承 14 天累積 KNOWHOW + KB_FIX（含「分店操作前先 list_branches」「軟下架的概念」「不懂就問」等）——**KNOWHOW 跨 session 累積機制的真實驗證場景**。

---

## 本輪新增 / 更新檔案

### cwsoft-sqlserver-mcp
- `src/cwsoft_sqlserver_mcp/server.py`（591 → 724 行）：加 `list_branches` / `create_branch` / `close_branch` 3 個 tool、學 sqlgate SP call + 獨立 connection verify 模式

### cwsoft-aquan-manager
- `AGENTS.md`：tool 清單加 3 條 + 新增「**分店操作流程**」段（軟下架概念、可逆理由、執行 5 步）
- `TODO.md`：加「**下次開會跟老闆討論：什麼情況需要通知老闆娘（簡楓怡）**」（含 broadcast script 改 multicast 的技術 side effect）
- `.gtb_codex_sessions/`：今日新增 `U38daae74d279bef697f99a22c65c3751.session`（趙士豪 lazy mint）、`Usim.session`、`Utest_alice.session`

### general-task-bot
- `scripts/broadcast_self_intro.py`：加 `--to` 旗標 + multicast API 分支、預設仍是 broadcast
- `scripts/intro_draft_20260522_meeting.txt`（新）：第三發稿件「會議記錄整理好了」
- `scripts/intro_draft_20260523_branch_features.txt`（新）：第四發稿件「分店功能上線」+ 開頭「各位同仁早安」

### cwsoft-project-tracker
- `meetings/會議記錄_2026-05-22.md`（新）：5/22 會議紀錄整理 + 多輪 ownership 修正
- `minutes/逐字稿-2026-0522.txt`：從 repo root mv 進 minutes/

### memory（local-only at `~/.claude/projects/.../memory/`）
- `reference-aquan-oa-friend-roster`（新）：阿全 OA 好友角色分類 + 廣播該不該寄、修正 5/23 user_id 對照錯誤
- `MEMORY.md`：加索引

---

## 待跟進

- [ ] **下次開會 5/26 跟老闆對「通知簡楓怡」規則**——什麼情況該寄、用什麼機制（multicast / push / 個別 LINE）
- [ ] **匯出客戶區間帳單 MCP tool**（notebook 出現 6 次的最強需求、roadmap top）——跟 generate_quote 套路一樣、應該不到 1 小時
- [ ] **modify_customer MCP tool**（單次需求、可順手做）
- [ ] **依縣市列客戶**——要先補資料 schema（客戶主檔沒縣市欄位）、跨工程議題
- [ ] **broadcast script 的 user_id roster 化**——目前 `--to` 還是要 colombo 記得 user_id 串、可以做個 `--audience colombo,boss,engineer` 別名機制（從 reference memory 讀 mapping）
- [ ] 監看林羿宏第一次跟阿全互動會發生什麼（驗 KNOWHOW 注入是不是真的 work for 全新 user）
- [ ] **下次 AGENTS.md 大改後一律跑「全分身 ping sync」**——這條應該變 SOP、寫進部署 checklist

---

## 反思

- **「老闆親自說幹的好」是切 prod 後最重要的 milestone signal**——比任何技術指標都重要。從 5/16 早上「Q3 末才接 prod」到 5/23 中午「老闆主動要更多權限」一週、節奏起伏跟最終結果都印證「colombo 評估 rollback 成本容忍度比我預設高」這條校正是對的
- **不可逆判準應用到業務 instance**：我 5/16 寫的 memory 規則太 abstract（「可逆 vs 不可逆」），colombo 5/23 把它具體 apply 到 close_branch：「**軟下架就是可逆、不必走兩段式機制**」——這是 abstract rule 走到 concrete decision 的成熟過程
- **per-user session 「分身同步」問題我事前沒想到**：完整設計時只規劃 mint 注入、沒想到既有 session 怎麼追新規則。今天臨時做 PowerShell ping 機制補上。下次設計類似機制要先想 「**老 instance 怎麼追新規則**」這層、不要只設計 happy path
- **OwnerShip 從 transcript 自動延伸是嚴重 anti-pattern**：5/22 會議記錄我寫了 5 條「士豪 owns」、colombo 3 條打回「不是我」。教訓：transcript 寫「我這裡剛剛加了 X」≠「X 的 owner 是我」。下次寫會議記錄時、ownership 欄保守標 「**【未明確指派】**」、等 colombo confirm 再 attribute
- **broadcast 對外訊息有 audience layer**：5/22 兩發都打到全好友（含老闆娘）、5/23 才知道老闆娘不該收。**對外 communication 應該預設「精準對象」、broadcast 是 fallback**、不是 default。broadcast_self_intro.py 改 multicast 為主後好點、但 audience identification 仍要人介入
- **「colombo 跟士豪是不是同一人」這種 identity confusion 我糾結太久**：應該更早直接問 colombo「你是誰」、不要靠 transcript 推測。下次遇到任何 actor identity 不明、直接問、不要自己猜
