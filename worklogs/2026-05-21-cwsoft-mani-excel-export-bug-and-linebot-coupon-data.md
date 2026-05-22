# 馬尼「庫存查詢表」匯 Excel 沒反應 — 追查到 LINE bot 優惠券資料塞長字串

## 客戶反映

馬尼通訊在「庫存查詢表」（路徑：主選單 → 結帳報表 → 庫存查詢表）篩選**馬尼東門店 / 手機配件 / 玻璃貼**後按「匯Excel」、再按「是」要含序號，**畫面完全沒反應**。

## Bug 定位

`統計\統計_庫存查詢表_2.vb:btn匯Excel_Click` 的 Catch 區塊只有 `Console.WriteLine(ex.Message)`，例外被吞掉，使用者看不到任何提示。

本機 VS 重現後改 `MsgBox(ex.Message & vbCrLf & ex.StackTrace)` 才抓到真正錯誤：

```
無法設定資料行 '商品資料'。該值違反了這個資料行的 MaxLength 限制。
   於 System.Data.DataColumn.CheckMaxLength(DataRow dr)
   於 System.Data.DataTable.Copy()
   於 統計_庫存查詢表_2.btn匯Excel_Click 行 405:
       Dim dt現有商品資料 As DataTable = Me.Ds庫存查詢表1.各商品現有序號.Copy
```

## Root cause

**XSD schema 寫死 maxLength=50，但實際資料有 87 字。**

| 來源 | 「商品資料」欄位長度 |
|------|------|
| `資料集\ds庫存查詢表.xsd` → `各商品現有序號.商品資料` | **maxLength=50** |
| 馬尼 DB `進貨商品資料.商品資料` 等表 | nvarchar(150) |
| 實際資料 | 87 字 |

抓進來時 `SDA.Fill` 不檢查長度，到 `.Copy()` 才驗證 → 超過 50 字就炸。

## 87 字長字串是什麼

掃了一下馬尼東門店所有超過 50 字的「商品資料」，**共 64 筆**（進貨 60、銷貨 4），全部都是同一個格式：

```
linebot/1657202632/users/U{32字 hex}/coupons/{20字英數}
```

拆解：
- `linebot/` → 來源是 LINE bot 系統
- `1657202632` → LINE bot 帳號 ID
- `users/U...` → LINE user ID（LINE 標準格式）
- `coupons/...` → 優惠券 ID

涉及的商品：

| 商品編號 | 商品類別 | 型號 |
|---|---|---|
| 60110400021 | 優惠券 | 玻璃貼兌換券 |
| 60110400097 | 優惠券 | 50元現金折抵券 |
| 60110400099 | 優惠券 | 會員生日300元現金券 |
| 60110400130 | 優惠券 | 200元現金折抵券 |
| 8020D0100287 | 手機配件 | iPhone 14 PM 玻璃貼 (滿版) |

看起來是 LINE OA 領券流程把領取記錄（誰、哪張券）寫到「商品資料」欄做 audit trail，**不是當序號用**。

**這部分是我看格式猜的，實際用途要請當初做 LINE OA 整合的同事確認。**

## 可能的處理路徑

### A. 內部 SQL 把超長記錄截短到 50 字
最快，但會破壞 LINE user ID（32 字 hex）/ coupon ID（20 字）的完整結構 → LINE bot 那邊的對帳/查詢功能可能壞掉。**不建議**。

### B. 改 XSD 放寬 maxLength
`資料集\ds庫存查詢表.xsd` 兩處 `商品資料` 的 `<xs:maxLength value="50" />` 改到 200（或拿掉約束）。順便把 btn匯Excel_Click 的 Catch 改成 MsgBox 顯示錯誤。
**治本**，但要走 SVN → MSBuild → WinRAR → FTP → 客戶端跑說明.exe 更新。

### C. 改 `dbo.現有商品資料` SP 過濾超長記錄
在 SP 各 UNION 分支加 `AND LEN(c.商品資料) <= 50`。
**對客戶端「庫存查詢表」零影響**：
- 4 個優惠券類別商品本來在「手機配件 / 玻璃貼」篩選下就看不到
- 8020D0100287 是玻璃貼，本來就不該有 IMEI 序號，linebot 塞進去的優惠券 ID 對客戶來說也是雜訊

但這 SP 在每個客戶 DB 各一份，要更新就得逐家跑。

## 需要同事確認的問題

LINE bot 系統把 `linebot/{bot}/users/{user}/coupons/{coupon}` 寫進「商品資料」欄是用來做什麼的：

1. 領券時記下「誰領的」這筆 audit trail？
2. 還是有別的 SP / 報表會 join 這個欄位、用完整字串做 key？

如果只是 audit trail，走 C（SP 過濾）對 LINE bot 那邊也沒影響、客戶端立即解。
如果有別處 join 完整字串，那走 B（改 XSD）比較穩，動到的範圍才不會誤傷 LINE bot 業務。

## 額外發現的程式問題（順便修）

`統計_庫存查詢表_2.vb` 的兩個小坑：

1. **`btn匯Excel_Click` Catch 區塊吞錯誤** — 改 `Console.WriteLine` 為 `MsgBox` 顯示給使用者
2. **`SDA取得現有商品資料.SelectCommand` 沒設 CommandTimeout** — 預設 30 秒，量大時容易誤判超時。建議設 500 秒對齊 `SqlSelectCommand1`

這兩個是順手能改的，跟主議題分開但值得一起處理。
