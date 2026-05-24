# 會員綁定系統正式上線前緊急搶修

- 日期：2026-05-19（整個下午，從零壹通訊行 LIFF 接通到端對端 happy path 走通）
- 主機：公司主機（pos@DESKTOP-P5EBFBE）
- 參與者：colombo0718 × Claude (claude-opus-4-7[1m]) + 同事（DB 設定、進階行銷系統端）
- 相關專案：cwsoft-liff-pages、cwsoft-sqlgate (bind_server.py)、cwsoft-super-manager (db_introspect)

> 承接 [2026-05-13-kbcs-fallback-chain-and-feature-catalog-drift.md](2026-05-13-kbcs-fallback-chain-and-feature-catalog-drift.md)。
> 零壹通訊行 LIFF 會員綁定要上線，從同事建好 LIFF channel 開始一路修到端對端 happy
> path 跑通。中間踩 6 個 bug 疊在一起，最久那個被前後兩個 bug 互相掩護、用了 5-6 輪
> 來回才挖到底。

---

## 一、起點：零壹通訊行 LIFF 接通要 demo

同事在 LINE Developers Console 為 `@snn0112i`（零壹通訊行的 LINE OA）建好新 LIFF
channel `2010132422-z4eAcFLJ`，endpoint 設成：

```
https://cwsoft-liff-pages.vercel.app/bind-membership.html?oaid=@snn0112i&liffId=2010132422-z4eAcFLJ
```

第一次 user 重打多次都卡在「**確認會員身分中…**」的 loading 圈圈、永遠不出綁定表單。

bind-server.py log 顯示 `/oa_lookup` + `/is_bind` 都 200 OK、`is_bound=false`，理論上 frontend 該 `hideGate()` 顯示表單，但**畫面就是不動**。

---

## 二、Bug #1：DB 漏填「進階行銷系統_網址」

bind_server.py `get_marketing_base_url` 從客戶 DB 的 `dbo.基本設定` 讀 `名稱='進階行銷系統_網址'`，零壹通訊行 DB 的設定值是**空字串**。

```sql
SELECT 設定值 FROM [零壹通訊行].dbo.基本設定 WHERE 名稱='進階行銷系統_網址';
-- → ''  (empty)
```

oa_lookup 回 `marketing_base_url: null` → frontend `buildMemberCenterUrl()` 回空 → `redirectToMemberCenter()` 跳過 → **但沒 `hideGate()`** → 永遠 loading。

同事 SSMS 補上：
```sql
UPDATE [零壹通訊行].dbo.基本設定
SET 設定值 = N'https://01phone.cwsoft.com.tw/'
WHERE 名稱 = N'進階行銷系統_網址';
-- _test DB 同步補
```

驗證：`oa_lookup` 回 `marketing_base_url: https://01phone.cwsoft.com.tw/` ✓。

---

## 三、Bug #2：frontend isBound + 空 URL 時沒 hideGate（自家洞）

[bind-membership.html:268-282](https://github.com/colombo0718/cwsoft-liff-pages/blob/master/bind-membership.html) 原邏輯：

```js
if (isBound) {
  setStatus("此帳號已綁定會員，表單已鎖定。", "success");
  lockAllUI();
  redirectToMemberCenter();  // marketing URL 空時 silently 跳過、不 hideGate
}
```

修補：commit **c0d0a12** — 加 fallback 訊息「會員中心連結尚未設定，請聯絡客服」+ 明確 hideGate。

但部署後 user 反映「還是卡轉圈圈」。

---

## 四、Bug #3（真兇）：CSS specificity 蓋過 `[hidden]`

對著 bind-server log 思考很久 — `is_bound=false` 該走 `else { hideGate(); }`、那條 path **跟我的 fix 無關、舊版本來也有**、為什麼也卡？

打開 CSS：

```css
#loading-gate {
  position: fixed;
  inset: 0;
  display: flex;     /* ← author stylesheet，specificity = #id */
  ...
}
```

HTML 規範 `<div hidden>` ≡ `display: none`（user-agent stylesheet），**但 author `display: flex` 永遠贏 user-agent**。所以 `loadingGate.hidden = true` 對這個 element 完全沒效果。

修補：commit **22eb940** — 加一條 CSS override：
```css
#loading-gate[hidden] {
  display: none;
}
```

部署後 user 終於看到綁定表單、能填資料 → 但**填完送出後卡在「綁定成功」訊息、不會跳轉到會員中心**。

---

## 五、Bug #4：submit 成功後沒 redirect（原作者 setTimeout 被注解掉）

[bind-membership.html:494-505](https://github.com/colombo0718/cwsoft-liff-pages/blob/master/bind-membership.html) 看到：

```js
setStatus("綁定成功，5 秒後將自動離開頁面。", "success");
lockAllUI();
// setTimeout(() => {            ← 整段被注解掉
//   liff.closeWindow();
//   ...
// }, 5000);
```

訊息**寫**「5 秒後自動離開」、但 setTimeout 整段被注解、永遠不會跳轉。

修補：commit **9b3bdb7** — 三件事一起做：
1. submit 成功後 redirect 到會員中心（跟 isBound=true path 一致）
2. isBound=true path 加 hideGate（防 loading 蓋住 redirect 中訊息）
3. **誤判**：buildMemberCenterUrl 拿掉 `/Products/MemberCenter` path（以為 404 是 path 錯）

第 (3) 點是**錯的**，下一個 bug 推翻它。

---

## 六、Bug #5：bind_server 把所有綁定都寫到測試 DB ⭐ 最大坑

部署 9b3bdb7 後 user 試「已綁定 path」— redirect 到 `https://01phone.cwsoft.com.tw/?lineUid=...` 但**畫面什麼都沒顯示**。手動 curl `https://01phone.cwsoft.com.tw/Products/MemberCenter?lineUid=test`：

```
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8

查無對應的 LINE 會員綁定資料或會員資料。
```

**404 不是 path 不存在**！是進階行銷系統自己回的「找不到該 LineUid 的綁定資料」。為什麼找不到？

查 DB：

```sql
SELECT * FROM [零壹通訊行].dbo.Line會員綁定 WHERE LineUid = N'U72ba...'  → 0 rows
SELECT * FROM [零壹通訊行].dbo.會員 WHERE LineUid = N'U72ba...'         → 0 rows
SELECT * FROM [零壹通訊行_test].dbo.會員 WHERE LineUid = N'U72ba...'      → 0 rows
```

零壹兩個 DB 都沒！但 bind-server log 看到 user 跑過 `/submit` 200 OK 兩次，**到底寫進哪？**

查 POSV3測試專用：
```sql
SELECT * FROM [POSV3測試專用].dbo.Line會員綁定 WHERE LineUid = N'U72ba...'
-- → 1 row：主索引 67、會員編號 2100008295、@snn0112i
```

找到了 — **全部綁定都寫進 POSV3測試專用**。

打開 bind_server.py [line 58, 254](https://github.com/colombo0718/cwsoft-sqlgate/blob/master/bind_server.py)：

```python
DB_NAME = "POSV3測試專用"   # line 58，hardcoded
...
@app.get("/submit")
def submit():
    ...
    with get_connection() as conn:
        cur = conn.cursor()
        cur.execute(f"USE [{DB_NAME}];")   # line 254，永遠 USE 測試 DB
```

**對照 `/is_bind` 已經是動態查 db_name（用 `oa_lookup_core`），但 `/submit` 沒做** — 兩個 endpoint 的 DB 路由邏輯不一致，導致：
- `/is_bind` 去客戶 DB 查 → 永遠 0 row（因為實際 INSERT 在測試 DB）→ `is_bound=false`
- `/submit` 寫進測試 DB → 該客戶 DB 永遠空
- 進階行銷系統去客戶 DB 查 → 找不到 → 404 + 「查無會員」

這條 bug **掩護了上一條對 path 的誤判** — 我以為 `/Products/MemberCenter` 是錯的、其實 path 對、是 server 回「查無會員」造成的 404。

修補：commit **a749eff**（cwsoft-sqlgate）

```python
# 改成跟 /is_bind 一致，動態查 db_name
lookup = oa_lookup_core(lineOAID)
if not lookup.get("ok") or not lookup.get("db_name"):
    return jsonify(ok=False, error="..."), 404
target_db = lookup["db_name"]

with get_connection() as conn:
    cur = conn.cursor()
    cur.execute(f"USE [{_safe_db_name(target_db)}];")
```

同時 commit **207c3fe**（cwsoft-liff-pages）— **revert** 9b3bdb7 拿掉 path 的部分、把 `/Products/MemberCenter` 加回去。

重啟 bind-server，重新 LIFF 綁定一次，DB 驗證：

```sql
SELECT * FROM [零壹通訊行].dbo.會員 WHERE LineUid = N'U72ba...'
-- → 1 row：會員編號 123175、姓名 趙士豪、入會時間 2026-05-19 15:52:00 ✓

SELECT * FROM [零壹通訊行].dbo.Line會員綁定 WHERE LineUid = N'U72ba...'
-- → 1 row：主索引 2710、會員編號 123175、@snn0112i ✓
```

prod 客戶 DB 終於有資料了。

---

## 七、Bug #6（UX）：已綁定 user redirect 等待時 form 露出來

修完 bug 5 後 user 點圖文選單（已綁定 path）：
- loading 關
- 1.2 秒 setTimeout 等待
- 期間 lockAllUI'd form 露出來給 user 看
- 1.2 秒後 location.replace 跳會員中心

「中間露出 form」UX 不對 — user 不需要看到他無法操作的表單。

修補：commit **221b67f**

```js
if (isBound) {
  ...
  if (memberCenterUrl) {
    // 不 hideGate，loading 蓋住 form 直到 redirect
    const gateText = document.getElementById("loading-gate-text");
    if (gateText) gateText.textContent = "正在前往會員中心…";
    redirectToMemberCenter();
  } else {
    hideGate();  // 沒 URL 才 hideGate 露出 fallback 訊息
  }
}
```

把 loading-gate 的 `<p>` 文字加 `id="loading-gate-text"`，已綁定要 redirect 時就改寫成「正在前往會員中心…」、loading 保持顯示直到 redirect 發生。

---

## 八、整條 commit 鏈時間線

| Commit | repo | 修什麼 |
|---|---|---|
| 同事 SSMS | DB | `零壹通訊行.基本設定` 補上 `進階行銷系統_網址=https://01phone.cwsoft.com.tw/` |
| `c0d0a12` | liff-pages | isBound + 空 URL 時 fallback 訊息 + hideGate |
| `22eb940` | liff-pages | CSS `#loading-gate[hidden] { display: none }`（**真兇**）|
| `9b3bdb7` | liff-pages | submit 成功後 redirect；含一個誤判（拿掉 `/Products/MemberCenter` path）|
| `a749eff` | sqlgate | **bind_server.py /submit 改動態查 db_name**（**最大 bug**）|
| `207c3fe` | liff-pages | revert 9b3bdb7 拿掉 path 的部分 |
| `221b67f` | liff-pages | 已綁定 redirect 等待期間不露 form（UX）|

---

## 九、本日成果

### cwsoft-liff-pages
- 5 個 commit（c0d0a12 / 22eb940 / 9b3bdb7 / 207c3fe / 221b67f）
- 主檔 `bind-membership.html` 端對端可用
- CSS / JS / UX 都修

### cwsoft-sqlgate
- 1 個 commit（a749eff）
- `bind_server.py /submit` 動態 DB 路由

### DB 設定
- `零壹通訊行.dbo.基本設定` + `零壹通訊行_test.dbo.基本設定` 補 `進階行銷系統_網址`

### 驗證
- 零壹通訊行 prod DB 已收到 2 筆真實綁定（趙士豪 + 林羿宏）
- 端對端 happy path 走通

---

## 十、待跟進（沒處理的）

### 立即（影響上線品質）
- [ ] **歷史資料 migration**：POSV3測試專用.dbo.Line會員綁定 有 10 筆**真實客戶綁定**誤寫的 row（`@613rdbxw` × 多、`@990imlnk` × 4、`@snn0112i` × 1）。要搬回各對應客戶 DB，不然這些既有用戶用 LIFF 進會員中心會踩同樣的 404
- [ ] **進階行銷系統的 DB 路由跟 bind_server 對齊** — 對方 ASP.NET 站可能還指 `_test`；要跟同事確認他們的 lookup 邏輯，必要時建一張共享 OAID→DBName 對照表

### 短期
- [ ] `各客戶分店` 表把 `_test` row 標停用 / 拿掉，避免 bind_server 排序 tie-breaker 翻轉導致誤寫
- [ ] bind_server.py oa_lookup_core SQL 加 secondary sort（`ORDER BY 分店代號 ASC, DBName ASC`）防止 prod/test 順序漂移
- [ ] cwsoft-liff-pages 待 commit/push 的兩個 untracked 檔（CLAUDE.md、PROJECT.md 等）

### 中期
- [ ] cwsoft-liff-pages PROJECT.md 加「綁定流程已知 quirk」段：CSS specificity for `[hidden]`、`/Products/MemberCenter` path 是進階行銷系統決定不能改、`/submit` 動態 DB 路由必須跟 `/is_bind` 一致
- [ ] 全 sqlgate 內部所有 endpoint audit「動態 DB 路由 vs hardcoded DB_NAME」一遍，避免類似 bug 殘留

---

## 附：當天的關鍵推導

### 「為什麼 bug 5 卡最久」

三個 bug 互相掩護：
- **Bug 4**（submit 沒 redirect）藏在 Bug 3（CSS hidden 失效）後面 — 修了 3 才看見 4
- **Bug 5**（DB 寫錯）藏在 Bug 4（修補方向誤判，以為 path 錯）後面 — revert path 才看清是 DB 寫錯
- **每修一層就跳出新症狀**，前面誤判 path 又繞遠路

教訓：bug 訊息（404 + 「查無會員」）一開始就在 server 回應裡寫得清楚，但因為 curl 沒下載 body 只看 status code、被「404 就是 path 不存在」直覺帶歪。**下次 404 要強制看 response body 才確認真兇**。

### 「為什麼 `/submit` 跟 `/is_bind` 路由邏輯不一致是個 trap」

`/is_bind` 是 5/12 才加（我之前 worklog 提的 governance cleanup 期間發現有人改了），那次改動是「動態查 db_name」；但 `/submit` 是更早寫的，**保留了原始 hardcoded 設計沒同步更新**。兩個 endpoint 在不同時間被改、改的人沒注意對齊。

教訓：**同類 endpoint 必須共用同一個 helper 拿 db_name**，不要各自寫各自的。修補後 `/submit` 直接 call `oa_lookup_core` — 跟 `/is_bind` 走同一函式。

### 「為什麼 CSS 的 [hidden] override bug 是 frontend 黑線」

`element.hidden = true` 看起來很 declarative — 但前端框架文化下，很多人會對 `#some-id` 設 `display: flex` 而沒意識到這會打架。debug 時 console 看 element 是有 hidden 屬性、邏輯也跑到 hideGate、就是不消失，**JS 沒 error、CSS 沒 warning、DOM 看起來對**，超難找。

教訓：**用 `display: none` (JS `style.display = 'none'`) 比 `hidden` 屬性可靠**，或者每個會 `[hidden]` 的 element 都該配套 CSS override 顯式聲明。

### 「進階行銷系統 vs bind_server 共用 DB 路由是個架構債」

兩套系統共用一個 DB，但**各自的程式碼決定碰哪個 DB**，這個架構天生有對齊風險。短期靠人工保持一致、長期該建一張 OAID→DBName 對照表（單一 source of truth），所有讀寫端都 reference 同一張。

這是 h2-direction.md 提的「Codex × GTP 並行」整合，類似精神 — 多個 agent 共用同一個 grounding source 才不會吵架。
