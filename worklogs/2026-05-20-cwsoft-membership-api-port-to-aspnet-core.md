# 會員綁定 + OTP 兩個 Flask server port 成 ASP.NET Core MVC .NET 8

- 日期：2026-05-20（接續 5/19 會員綁定緊急搶修後 1 天）
- 主機：公司主機（pos@DESKTOP-P5EBFBE）
- 參與者：colombo0718 × Claude (claude-opus-4-7[1m]) + 同事（IIS 部署接手、進階行銷系統架構諮詢）
- 相關專案：cwsoft-membership-api（新）、cwsoft-sqlgate（被取代的 Flask 版）、cwsoft-liff-pages（前端）

> 承接 [2026-05-19-cwsoft-membership-binding-emergency-fixes.md](2026-05-19-cwsoft-membership-binding-emergency-fixes.md)。
> 5/19 搶修完之後，趁正式上線前的測試緩衝期、把兩個 Flask server 移到公司正式 IIS stack
> （ASP.NET Core .NET 8）。Day-1 純 code translation + 文件，部署交給同事。

---

## 一、起點：對齊公司既有 stack

5/19 的會員綁定流程修補後，user 提出「趁測試緩衝期、把該移到公司正式環境的東西都移一移」。
首先就是 `cwsoft-sqlgate` 裡的兩個 Flask service：

- `bind_server.py`（port 4002）— LIFF 會員綁定後端
- `OTP_server.py`（port 4001）— SMS OTP 簡訊驗證

這兩個跑在公司主機這台 + super-manager 接管，是「個人探索 stack（leaflune tunnel + Caddy + Python）」的產物。
公司**正式 stack** 是同事管的 IIS server（.NET 8 ASP.NET Core MVC）— 跟既有「進階行銷系統」同一棧。

把這兩個 service 移到正式 stack 的好處：
1. 部署運維跟同事的「進階行銷系統」一致
2. 公司主機只剩 super-manager / aquan / cs-shadow / kbcs 這些「探索期還沒定型」的服務
3. DR / 多 instance / 環境分離都靠公司既有 IIS 機制
4. 同事熟悉的程式碼風格，後續 maintenance 不必每次找我

---

## 二、跟同事確認的架構規格（user 轉述）

問：「進階行銷系統」具體什麼樣？同事答：

| 項 | 答 |
|---|---|
| Framework | ASP.NET Core MVC **.NET 8**、SDK-style csproj |
| ASPNETCORE_ENVIRONMENT | Production（IIS web.config 沒設、預設值） |
| Connection string | **從站台根目錄 `.env` 來**（不是 appsettings.json）|
| Program.cs | 手動 `LoadEnv()`，跟 Python `python-dotenv` 同概念 |
| 沒 appsettings.Production.json | appsettings.json 的 connection string 是空、實際靠 .env |
| OAID → DB 路由 | **動態 lookup**（不是 hardcoded） |

**最關鍵這條**：對方也是 dynamic OAID lookup → 跟我們 5/19 修補後的 bind_server.py 邏輯**架構同源**。
跨系統一致性問題剩下「兩邊查哪張 table / 排序規則」。

→ port 計畫變 confident：可以**1:1 對齊 .env + dynamic lookup 模式**，直接 plug in 同事的 stack。

---

## 三、新建 `cwsoft-membership-api` repo

```
C:\Users\pos\cwsoft-membership-api\         ← 跟新世代 cwsoft-* 對齊位置
├─ MembershipApi.csproj                     ← net8.0 SDK-style
├─ Program.cs                               ← DotNetEnv 載 .env + CORS + controllers
├─ Controllers\
│   ├─ HealthController.cs                  ← /bind/health + /otp/health + /health
│   ├─ BindController.cs                    ← 4 endpoint（oa_lookup / is_bind / submit / branches）
│   └─ OtpController.cs                     ← 2 endpoint（send / check）
├─ Services\
│   ├─ DbHelper.cs                          ← SQL Server 連線、_safe_db_name 白名單
│   ├─ OaLookupService.cs                   ← OAID → DBName 反查 + marketing URL
│   └─ OtpStore.cs                          ← SQLite OTP 暫存（沿用 Flask 版同 schema）
├─ .env.example
├─ .gitignore                               ← 擋 .env / bin / obj / *.db
├─ README.md                                ← 技術細節
├─ HANDOVER.md                              ← 給同事的交付清單
└─ docs\
    └─ binding-flow.md                      ← 整條 chain 全景（前後端 + DB）
```

`dotnet new webapi --framework net8.0 --use-controllers` 起手，清掉預設 WeatherForecast 範本、加 4 個 NuGet：

| 套件 | 版本 | 用途 |
|---|---|---|
| DotNetEnv | 3.2.0 | .env 載入（對齊同事 stack） |
| Microsoft.Data.SqlClient | latest | SQL Server 純 TDS（**不需 ODBC**） |
| Dapper | 2.1.79 | query mapping，比手寫 reader 簡潔 |
| Microsoft.Data.Sqlite | latest | OTP 暫存（沿用 Flask 版相容） |

---

## 四、Port bind_server.py → BindController（4 endpoint）

對齊 Flask 版邏輯**1:1**，包括：
- SQL query 寫法（含 `ORDER BY 分店代號 ASC` + `db_names[0]` 取主 DB）
- `/submit` 用 **SERIALIZABLE transaction**、相同的 6 步流程（檢查已綁→用手機找→reuse 或新建→寫 LineUid→INSERT 綁定）
- `/oa_lookup` 順手讀 `[客戶DB].dbo.基本設定` 的 `進階行銷系統_網址` 給 LIFF 用
- CORS（cwsoft-liff-pages.vercel.app + develop.leaflune.org）

**含 5/19 a749eff 修補的 `/submit` 動態 db_name fix** — Python 版原本 hardcode `DB_NAME = "POSV3測試專用"`、5/19 才修；
port 過去直接走修補後版本、不會把這個 bug 帶到 .NET 版。

---

## 五、Port OTP_server.py → OtpController（2 endpoint）

幾個 1:1 對齊：
- `/send` 三件事連動的 transaction（查分店主索引 → 扣 SMS 點數 → USE 客戶 DB 寫簡訊記錄）+ commit 後 SQLite UPSERT
- `/check` SQLite 比對 + 5 分鐘有效 + check_token 重發保護（重按不重發、回同一 token）
- `secrets.token_urlsafe(24)` → 用 .NET `RandomNumberGenerator.Fill` + base64url 拼，相同 entropy

OTP 暫存 Day-1 **保留 SQLite**（沿用 Flask 版 `otp_store.db`），README 跟 HANDOVER 都標記「之後若多 IIS instance / DR 一致性考量、換 `OtpStore.cs` backend 即可」。

---

## 六、幾個關鍵設計決策

### (1) 一個 project 兩個 Controller、不是兩個獨立服務
Flask 是 2 個 process（4001 + 4002），.NET 我合併成一個 app。
理由：
- 共用同一個 SQL Server 連線設定（DotNetEnv 只載一次）
- 共用 `OaLookupService` / `DbHelper`
- IIS Site 一個比兩個好管
- URL prefix 靠 `[Route("bind")]` / `[Route("otp")]` 區分

代價：反向代理過來的請求**必須保留 `/bind/` `/otp/` prefix**，不能用「剝前綴」型路由（目前公司主機 Caddy 是剝前綴、但**這次部署不動公司主機 Caddy** — cutover 階段再決定）。

### (2) 不用 EF Core，選 Dapper + SqlConnection
EF Core 跟 dynamic `USE [...]` 不太合得來（DbContext 綁定特定 DB），會員綁定要 runtime 切 DB → 直接走 Dapper + SqlClient 更乾淨。
Dapper 只負責 query → DTO mapping，不引入 ORM 思維。

### (3) Static service vs DI
第一版 `OaLookupService` / `OtpStore` 寫成 static class、Controller 直接呼叫。理由：
- 沒有狀態（每次呼叫都新開連線、不維護 instance state）
- DI scope 對這個 case 沒意義
- 簡單 = 同事容易讀

之後若要加 `IDbContextFactory<>` / mocking for unit test 才會改 DI。

### (4) 保留 SQLite for OTP
原 Flask 版 OTP 存 `otp_store.db`（本機 SQLite）。.NET 版可以改 SQL Server table 或 in-memory cache，但 Day-1 選**保留 SQLite**：
- Schema 跟 Flask 版**完全相同**、可以 hot-swap 同一份 db file
- 沒 schema migration 風險
- 多 IIS instance 才需要分散式 store；Day-1 單 instance OK

### (5) `_safe_db_name` 略嚴於 Python 版
Python 只 `replace("]", "]]")` 防 `[name]` 注入。
.NET 版用白名單字元（letterOrDigit / `_` / 中文 > 127）。
理論上不會破壞既有客戶 DB 名（都符合白名單），更保守一些；HANDOVER 有寫明差異。

---

## 七、三層文件交付（給同事讀）

| 檔 | 給誰、什麼用 | 行數 |
|---|---|---|
| `HANDOVER.md` | 同事先讀；部署步驟 + 跟 Flask 差異 + 驗證 + cutover 策略 | 143 |
| `README.md` | 技術細節；NuGet / build / publish / 行為對照 | 108 |
| `docs/binding-flow.md` | 整條 chain 全景（前後端 + DB）— **給人看的流程文件，不是 API spec** | 279 |

`docs/binding-flow.md` 九段、含一張 ASCII 流程圖、含「**整條 chain 的 DB 觸碰一覽表**」（每個 step 動哪個中央 DB 或哪個客戶 DB），同事讀完這份就能理解 chain。

---

## 八、本日成果

### cwsoft-membership-api（新 repo）
- 完整 .NET 8 ASP.NET Core MVC 專案
- 6 個 endpoint 1:1 對齊 Flask 版（已含 5/19 的 bug fix）
- 3 份文件（HANDOVER / README / binding-flow）
- Build 0 errors 0 warnings
- **GitHub repo 尚未開**（等 user 決定 repo URL + push）

### cwsoft-sqlgate / cwsoft-liff-pages
- 沒動（Flask 版仍跑 prod、cutover 後再退役）

### 沒做的（明確排除）
- ❌ 我沒對真實 SQL Server 跑 integration test（code review level 對齊、不是 runtime test）
- ❌ 沒 publish release build（同事自己跑 `dotnet publish`）
- ❌ 沒動公司主機 Caddy / cloudflared / Flask service（不在本次 scope）
- ❌ 沒處理 POSV3測試專用 內 10 筆誤寫綁定 row 的 migration（5/19 待跟進、仍待跟進）

---

## 九、待跟進

### 立即（同事接手後）
- [ ] 同事 publish 出 release build、部署到公司 IIS server
- [ ] 同事在 publish 目錄根目錄放 `.env`（DB 密碼從 cwsoft-sqlgate/.env 抄）
- [ ] 確認 IIS 機器有 `.NET 8 Hosting Bundle`
- [ ] 端對端跑一輪 happy path（健康檢查 + oa_lookup + 跟 Flask 版 diff）

### 短期（部署後 1-2 天）
- [ ] 兩邊並存階段：對比 Flask（4002 / 4001）vs .NET 同樣 input、回應 byte-by-byte 一致確認
- [ ] LIFF 端切到新 IIS URL 做 dev 測試（不動公司 prod 流量）

### Cutover（1-2 週驗證後）
- [ ] LIFF 端 / 外層反向代理切流量到新 IIS
- [ ] super-manager `services.json` 拿掉 `bind-server (4002)` / `otp-server (4001)`
- [ ] 老 Flask 程式碼留 git history、不刪

### 跟 5/19 留下的尾巴
- [ ] POSV3測試專用 10 筆誤寫綁定 row migration 回各客戶 DB（@613rdbxw / @990imlnk / @snn0112i）
- [ ] 進階行銷系統 dynamic lookup 跟 bind_server 對齊（短期：兩邊都選 prod；長期：建一張 OAID→主 DB 對照 table）
- [ ] `各客戶分店` 拿掉 `_test` row（生產環境不該收 _test）

---

## 附：當天的關鍵推導

### 「為什麼 Day-1 不要做 integration test」

決策原因：
1. user 明確說「不用這麼複雜，純 code translation 就好、剩下同事負責部署」
2. 我這 session 沒 IIS 機器 access、test 也只是測 dev 本機
3. real test 是同事部署完跑 happy path 驗證 — 那個 test 比較有意義（驗證 IIS 部署 + .NET 8 Hosting Bundle + .env 配對等真實環境）

我這邊做的「test」level 是：build 過 + endpoint 對齊 Flask 版 SQL/邏輯。同事部署完跑端到端才是真 test。

教訓：**避免「在 dev 環境做 fake test 給自己安心」**。當生產環境完全不同棧時，本機 test 的 false confidence 反而是 risk。

### 「為什麼用 static class 而不是 DI」

ASP.NET Core 預設 DI everything 是潛規則，但這個 case：
- Service 本身**沒有狀態**（每次呼叫都新開連線）
- 沒 mocking 需求（沒寫 unit test）
- DI scope（Transient / Scoped / Singleton）對這個無狀態服務沒區別
- 同事熟悉「DI 重 service」、看到一堆 `[Inject]` 反而要 trace registration

第一版 simple wins。之後若要加 unit test mock、或要換 backend（OtpStore SQLite → SQL Server），那時才會引入 DI interface。

教訓：**不要為了未來不確定的擴展「先 DI 起來」**。YAGNI；該重構時再重構。

### 「為什麼 docs/binding-flow.md 是給人看而不是給 AI 看」

cwsoft 文件層次：
- `CLAUDE.md` / `AGENTS.md` — 給 AI 看的 agent 工作規範
- `PROJECT.md` — 給 AI + 人類技術人員看的專案概要
- `worklogs/` — 給未來 AI 回顧上下文用
- **`docs/binding-flow.md`** — 給**人類同事**讀的流程文件

差別：給人看的文件**著重「為什麼這樣 / 哪個 step 動哪個 DB / 出問題看哪」**；給 AI 看的文件著重「精確規格 + 可機器解析的決策樹」。
這次 user 明確要「給同事看、寫關鍵流程跟檢索哪個資料庫就好」— 是給人看的層次。寫法就用敘事 + 圖 + 表，不寫 endpoint param 細節（那在 README）。

教訓：**寫文件前先問「誰會讀」+「他讀完要做什麼決定」**，再選層次。同樣的事實，給不同 reader 的呈現方式天差地別。
