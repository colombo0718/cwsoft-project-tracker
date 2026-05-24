# 阿全接 prod 衝刺 day 2 — git baseline 進雲端 × caddy 切 prod × per-user session × KNOWHOW 跨人累積系統

- 日期：2026-05-22
- 主機：公司主機
- 參與者：colombo0718 × Claude (claude-opus-4-7)
- 相關專案：general-task-bot、cwsoft-aquan-manager、cwsoft-project-tracker、autoQuotes、Caddyfile（C:\Program Files\cloudflared\）

> 接 [5/21 day 1 worklog](2026-05-21-aquan-prod-cutover-day1-sms-quote-quickreply-defaults.md)、把 prod 切換需要的剩下三件 P0 都收掉、加碼一條「跨 session 經驗累積」機制。
>
> 主軸：(1) gtb_codex.py 第一次進 git baseline + 雲端、(2) caddy 加 @708juxdz 分流到 codex 版、(3) per-user session 拆 SID + KNOWHOW 標籤跨人累積。

---

## 1. git baseline 進雲端（gtb_codex.py 從 untracked → committed）

### 觸發

colombo 問「git commit baseline 是什麼意思」、解釋完發現另一個更急的問題——**`gtb_codex.py` 501 行從 5/16 寫到 5/22 從沒進過 git**、純活在 working tree + 跑中 process memory 裡。機器爆掉只剩 worklog 文字描述、要重寫。對 prod 服務不可接受。

### 順帶發現的安全狀況

查 `git remote -v`、repo 在 `https://github.com/colombo0718/general-task-bot`、`api.github.com` 查到 `private: False` ——**public**。已 push commit 到 5/12、`customerlist.txt`（142 個客戶名）+ `generate_customerlist_simple.py`（SQL schema）已外洩 9 天。

colombo 評估：「.env 有藏好就好、customerlist 還好」——不擔心、直接推進。OK 但留memory entry 給未來提醒。

### .gitignore 強化

從 3 行擴成 17 行、補了 `*.db` / `*.pyc` / `__pycache__/` / `.gtb_codex_session` / `.gtb_codex_sessions/` / `logs/` / `*.bak.*` / `.claude/`。`oa_registry.json` 跟 `.env` 早已擋住沒漏。

### Commit + push

```
commit dec4520 — feat: 阿全 codex × MCP 架構 baseline + 5/16-5/21 累積
42 files changed, 7022 insertions(+), 238 deletions(-)
```

涵蓋：gtb_codex.py / tools/ 整包 / scripts/ / docs/（12 篇）/ AGENTS.md / CHANGELOG.md / 既有 modified（PROJECT.md / TODO.md / gtb.py / etc.）

push 失敗——Bash 跟 PowerShell subprocess 都拿不到 Windows Credential Manager 認證、跳不出 GCM 視窗。**commit 安全在本機、push 交給 colombo 手動跑一次**（互動 terminal 有 cached creds）。

---

## 2. Prod OA 切換到 codex 版（caddy 加一條 rule）

### 發現現有分流邏輯

caddy admin API（`localhost:2019/config/`）live config 顯示：

```
/callback/@526fdbzo*  → 127.0.0.1:6010   ← 阿全測式 (codex)
/callback*            → 127.0.0.1:6000   ← 舊 GTB (gtb.py)
```

「match 越具體越先」、所以測試版 @526fdbzo 已分流到 codex、@708juxdz 走 fallback 進舊 GTB。

### 找 source of truth

`cwsoft-sqlgate/caddy.exe` 是舊版、實際運作的 caddy 在 `C:\Program Files\cloudflared\caddy.exe`、config 路徑 `C:\Program Files\cloudflared\Caddyfile`（cwsoft-super-manager `services.json` 指定）。

### Caddyfile 加新 rule

```caddy
handle /callback/@526fdbzo* {
    reverse_proxy 127.0.0.1:6010   # 阿全測式 (codex)
}
handle /callback/@708juxdz* {
    reverse_proxy 127.0.0.1:6010   # 2026-05-22 切到 codex、舊 GTB 留 rollback
}
handle /callback* {
    reverse_proxy 127.0.0.1:6000   # fallback / rollback path
}
```

順手修了 stale comment（原寫 `gtb_dev.py`、實際是 `gtb_codex.py`）。

### Reload + verify

`caddy reload --config Caddyfile` → admin API 確認 3 條 callback rule 都活。從外網 HTTPS POST 直接打 `https://cwsoft.leaflune.org/callback/@708juxdz` → **400 Bad Request（invalid signature）**——完整鏈路通：

```
LINE → Cloudflare Tunnel → Caddy → /callback/@708juxdz match → 127.0.0.1:6010 Flask → signature check
```

400 是 Flask 端的 `abort(400, "invalid signature")`、表示 webhook payload 到了正確 Flask、但 test POST 沒帶 X-Line-Signature header 所以擋下。**這個 400 是成功訊號**——分流沒過會是 404 或 502/504。

舊 GTB 6000 process 沒退役、`/callback*` fallback 仍指向它、保留 rollback 路徑。

---

## 3. per-user session 大重構

### 動機

5/19 會議定的 P0、prod 切換前必做。原因：codex 全域單一 session 跨人共用會混茶（5/16 worklog 已說明）、prod 給老闆 1-on-1 用沒事、但**未來 colombo 把同事拉進來或進群組就會踩雷**。

### 變更（gtb_codex.py 501 → 699 行）

**Module 狀態**：

```python
SESSION_ID = None                    # OLD: 全域單一
SESSION_FILE = PROJECT_DIR / ".gtb_codex_session"   # OLD

SESSIONS: dict[str, str] = {}        # NEW: user_id → session_id
SESSIONS_DIR = PROJECT_DIR / ".gtb_codex_sessions"  # NEW: 一檔一 user
LEGACY_SESSION_FILE = PROJECT_DIR / ".gtb_codex_session"
LEGACY_OWNER_USER_ID = "U34e144c9bf7d30bc07c543a4ebae0df1"   # colombo
SIM_USER_ID = "Usim"                 # /sim 後門 default user_id
```

**新增 helper**：
- `_session_file_for_user(user_id)`、`_load_persisted_sessions()`、`_save_session_for_user(user_id, sid)`
- `_migrate_legacy_session_file()` — 一次性 mv 舊單檔 → 該 owner 的 user-specific 檔
- `_get_or_mint_session(user_id)` — thread-safe lazy mint with double-check + `_SESSION_MINT_LOCK`

**改寫**：
- `_chat_log_path(user_id)` — log 檔名 `<uid_short>_<sid_short>_chat.log`、每 user 一檔
- `log_chat_turn(..., user_id="default")` — 多一參數透傳
- `_mint_new_session(user_id)` — 新增 user_id 參數 + KNOWHOW 注入（見 § 4）
- `boot_codex_sessions()` — replace `boot_codex_session()`、lazy mint 為主、boot 時只 load 既有 session、不 upfront mint
- `reset_codex_session_for_user(user_id)` — replace 全域版、支援 'all'

**ask_codex 加 user_id 參數**：

```python
def ask_codex(user_text: str, user_id: str = "default") -> tuple[str, dict]:
    sid = _get_or_mint_session(user_id)   # ← lazy mint per user
    ...
```

**Flask handler**：
- `/callback` 從 `event.source.user_id` 拿、傳給 ask_codex
- `/sim` 接受 payload.user_id（default `SIM_USER_ID`）
- `/reset` 改成 payload.user_id 必填、支援 `"all"`
- `/health` 顯示 `user_session_count` + 全 sessions 列表

### Legacy session 遷移

`boot_codex_sessions()` 啟動時自動 migrate：

```
[BOOT] migrated legacy session 019e2fe9... → U34e144c9bf7d30bc07c543a4ebae0df1.session
[BOOT] loaded 1 persisted user session(s):
[BOOT]   43a4ebae0df1 → 019e2fe9...
```

colombo 從 5/16 累積的 session `019e2fe9` 完整保留為他自己 user_id 的 session。

---

## 4. KNOWHOW 跨人累積機制

### colombo 點出的設計問題

「per-user session 隔離跨人錯亂寫入」+「每個人教阿全的事卻不共享」是衝突的——「分靈問題」。原本 5/16 TODO 寫的「共學靠 AGENTS.md 撐」過於樂觀。

### 兩種知識分流

| 層 | 性質 | 例子 | 累積方式 |
|---|---|---|---|
| **A. 系統事實**（`[KB_FIX]`）| POS / DB / cwsoft 業務的**事實理解** | 「POS 是單店」「客戶 DB 名 = 公司名」 | 人工 review → AGENTS.md baked-in |
| **B. 工作技巧**（`[KNOWHOW]`、新）| 阿全**該怎麼做事**比較聰明的 heuristic | 「問點數一次回月租+簡訊兩種」「客戶名先 match」 | **自動跨人共享、boot prompt 注入** |
| C. 個人偏好 | 該 user 個人習慣 | 「彥偉問報價單通常要 6 個月」 | per-user session 自然累積、不跨 |

### 實作

**新 tag** `KNOWHOW` 加入 `KNOWN_TAGS`、AGENTS.md 加觸發定義（區別 KB_FIX vs KNOWHOW 的 nuance 寫清楚）。

**注入機制**（`_build_knowhow_injection()`）：

```python
_KNOWHOW_TAGS_TO_INJECT = {"KNOWHOW", "KB_FIX"}
_KNOWHOW_INJECT_DAYS = 14

def _build_knowhow_injection() -> str:
    # 掃近 14 天 notebook_YYYYMMDD.md
    # 抽 [KNOWHOW] + [KB_FIX] 條目、dedup by payload 字串
    # 格式化成 boot prompt 一段
```

mint 時把這段 append 到 `BOOT_SYSTEM_PROMPT`、新 user 第一次來就繼承累積知識。

### 端到端驗證

`/sim` 送 user_id=`Utest_alice`：

```
[MINT] user=_alice model=gpt-5.4 sandbox=danger-full-access
       disabled=['shell_tool', 'multi_agent', 'apps'] knowhow_lines=9
[MINT] OK user=_alice sid=019e4d78... in 10.6s
```

**9 條 KNOWHOW + KB_FIX 自動注入**到 alice 的 boot prompt。alice 自我介紹直接反映繼承知識：

> 「**不確定就先問你、不會硬猜**」（← 對話風格鐵律 KB_FIX）
> 「**客戶名稱先校正、再查資料**」（← match_customer_name 規則）
> 「**點數、報價單這類容易搞混的、我會先幫你釐清**」（← 類型釐清規則）

不是空白 codex、是「繼承過去 colombo 累積經驗的 codex」。

---

## 5. @526fdbzo alias 前綴（測試版專屬）

### colombo 的需求

「我一個 LINE 帳號想模擬不同人測 KNOWHOW 跨 session 累積」——LINE 端 user_id 是綁帳號的、改不了。

### 解法：訊息開頭 alias 前綴

`/callback` handler 對 `@526fdbzo` 多一層處理：

```python
if oaid == "@526fdbzo":
    m_alias = re.match(r"^@(\w+)\s+(.+)$", user_text, re.DOTALL)
    if m_alias:
        alias = m_alias.group(1)
        user_id = f"Utest_{alias}"
        user_text = m_alias.group(2).strip()
```

範例：

- `@alice 哈囉` → user_id=`Utest_alice`、text=「哈囉」
- `@bob 你會什麼?` → user_id=`Utest_bob`
- 無前綴的訊息 → 真實 LINE user_id

**生產版 @708juxdz 不啟用 alias**——防止任何人用 alias hijack 寫錯人。

---

## 6. 共用 notebook 設計確認

colombo 問「三個分靈筆記寫同一個檔嗎?」——是。

```
測試版 @526fdbzo  →┐
                   ├─→ 同一個 Flask (port 6010)
正式版 @708juxdz  →┘     │
                          ├─ notebooks/notebook_YYYYMMDD.md  ← 共用
                          └─ .gtb_codex_sessions/<user_id>.session  ← 各 user 一條
```

兩個 OA + 各 alias session 全部共寫同一份每日 notebook。KNOWHOW + KB_FIX 自動跨人 + 跨 OA 累積、跨 user mint 時注入。

**含義**：測試版亂教阿全的東西會污染正式版的 mint base。測試 KNOWHOW 要謹慎、教錯了去 notebook 手動刪那行。

---

## 7. 群組部署討論（colombo 喊停、TODO 不動）

我提出「放群組必須先做 wakeword filter」——colombo 評：「群組部分太進階、老闆沒這個需求、不瞎操心」、TODO 維持原狀、不在這次切 prod 範圍。

---

## 本輪新增 / 更新檔案

### general-task-bot
- `gtb_codex.py`（501 → 699 行）：per-user SESSIONS dict、lazy mint、KNOWHOW 注入、@526fdbzo alias 前綴、/reset & /health 改 per-user
- `.gitignore`（3 → 17 行）：補 runtime artifacts
- **首次 git commit `dec4520`**（42 files / 7022 insertions / push 待 colombo 手動）

### cwsoft-aquan-manager
- `AGENTS.md`：加 `[KNOWHOW]` tag 定義 + 「KNOWHOW + KB_FIX 跨 session 累積機制」整段解說（強調 codex 標越精準阿全整體越聰明）
- `.gtb_codex_session` 已 migrate 走刪除
- `.gtb_codex_sessions/`（新）：含 `U34e144c9bf7d30bc07c543a4ebae0df1.session` 一條（colombo 的 019e2fe9）
- 重啟後新增 `Utest_alice.session`（驗證 mint）

### Caddyfile (C:\Program Files\cloudflared\)
- 加 `handle /callback/@708juxdz*` → 6010 規則
- 更新 stale comments（原 `gtb_dev.py` → 正確的 `gtb_codex.py`）
- reload 套用（live config 已生效）

---

## 待跟進

- [ ] **colombo 手動 git push origin master**（互動 terminal）— commit 已就位、就差最後一哩
- [ ] **老闆首次 LINE 試用 @708juxdz**——觀察前 1-2 小時的對話品質、grep notebook `[ACTION] [BUG_SIGHTED]`、發現問題立刻 caddy rollback 刪 `/callback/@708juxdz*` rule
- [ ] **P1 三個 MCP tool**（query_credentials / company_name / extend_due_date）——明天有空再做、老闆 first-day 體驗會差很多
- [ ] **群組 wakeword filter**（colombo 暫不需要、留 TODO）
- [ ] **AGENTS.md 規則 review**——5/16-5/22 累積 8 段、token 不算多但結構可以 distill、配合論文影片「規則越少 codex 整體判斷力越強」精神

---

## 反思

- **per-user session 實作比預估快**：colombo 早上說「這個其實很快」、我估 1-2 小時、實際結合 KNOWHOW 系統一起做大約 1.5 小時、加 syntax check 跟 restart 驗證 2 小時整體 wrap。lazy mint + thread-safe lock 的設計比上次估的單純（單 lock 全域就夠、不必 per-user lock）
- **KNOWHOW 跨 session 注入端到端首次成功**：alice 的自我介紹直接列舉繼承的知識點、不是泛泛而談——這是這次 sprint 最讓我覺得「對」的時刻、機制設計直接打中問題
- **gtb_codex.py 6 天未進 git 才被發現**：5/16 寫了就上線、整個 5/16-5/21 都活在 working tree + process memory。應該 5/16 worklog 寫完當下就立 baseline commit——把「進 git」當 deploy 流程的硬步驟、不留給人自然想起
- **跟 colombo 對 spec 的 friction 減少**：今天有幾個快速 Q＆A（KNOWHOW tag 命名、報價單預設、群組 wakeword 該不該管）他幾乎都是一句話拍板、我也學會「propose specific options + 等他 pick」比 open-ended 提問更省時。配合 5/21 memory entry `feedback_domain_rules_standalone`（不要自作主張延伸）、合作節奏漸入佳境
- **Caddyfile source of truth 一開始找錯**：先在 `cwsoft-sqlgate/` 找（因為有 caddy.exe）、結果是 stale 副本。實際運作的在 `C:\Program Files\cloudflared\`。super-manager 的 services.json 才是 cwsoft 服務啟動的 single source of truth、之後找 cwsoft 任何服務的「真正配置」都應該先看 services.json 而不是 grep 目錄
