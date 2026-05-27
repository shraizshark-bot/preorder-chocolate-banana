# preorder-chocolate-banana

黑金巧克力香蕉預購表單系統

## 架構總覽

```
使用者瀏覽器
    │
    │ 填表送出 (fetch POST, no-cors)
    ▼
GitHub Pages (靜態 HTML)
    │
    │ HTTP POST JSON
    ▼
Google Apps Script Web App (中介層)
    │
    │ appendRow()
    ▼
Google Sheets (訂單資料庫)
```

## 檔案結構

```
preorder-chocolate-banana/
├── index.html          # 預購表單（落地頁 + hero 圖 + 訂購表單）
├── report.html         # 匯款回報表（末5碼、金額、時間）
├── hero.jpg            # 商品 hero 圖
└── README.md           # 本說明文件
```

## 關鍵設定值

| 項目 | 值 |
|------|-----|
| 商品名稱 | 黑金巧克力香蕉 |
| 售價 | $580 / 盒 |
| 規格 | 一盒兩份，適合 4 人 |
| GitHub Pages URL | https://shraizshark-bot.github.io/preorder-chocolate-banana/ |
| Google Sheet ID | 1Hua56eTicWv5zkOaDGU57fbi3Jtq_30qDCYjUC1oAW8 |
| Apps Script URL | https://script.google.com/macros/s/AKfycbzpy2ykv16K06T7ZJx79rQaVAsHuTBC8tBC7bbHs8eChMVkzHTJMxfYJdBqi9-FoZ73Kw/exec |

## 流程設計（兩階段對帳）

```
[index.html 預購]                      [report.html 匯款回報]
   姓名 / 電話 / 數量                       電話 / 末5碼 / 金額 / 時間
   付款方式（現金 or 匯款）
        ↓                                       ↓
   POST { type: 'order', ... }            POST { type: 'report', ... }
        ↓                                       ↓
   Apps Script doPost ──分流──→ 「訂單」分頁         「匯款回報」分頁
        ↓
   成功畫面：
     現金 → 顯示「取貨時付現」
     匯款 → 顯示銀行帳號 placeholder + 「回報末5碼」按鈕（帶電話進 report.html）
```

對帳方式：開 Google Sheet，兩個分頁並排，用「電話」當 key 做 VLOOKUP 或人工核對。

## index.html 結構

### HTML 區塊
- `<img class="hero">` — 商品 hero 圖（`hero.jpg`）
- `.info-row` — 規格／人數／售價三欄
- `.notice` — 限量／不定期出貨提醒
- `#orderForm` — 表單本體（姓名、電話、數量、付款方式、備註、總金額）
- `.pay-options` + `.bank-info` — 付款方式 radio + 展開銀行資訊區塊
- `#successMsg` — 送出後動態渲染（依付款方式顯示不同內容）

### JavaScript 邏輯
- `SHEET_URL` — Apps Script endpoint，唯一需要改的設定值
- `changeQty(delta)` — 數量 +/- 並即時更新總金額
- `submitOrder()` — 驗證 → fetch POST `{ type: 'order', ... }` → 顯示成功畫面
- `showSuccess()` — 依 payment 分支：現金顯示提示／匯款顯示銀行資訊 + report.html 連結（帶 phone 參數）
- 付款方式 radio change listener — 表單內展開／收合銀行資訊
- 送出使用 `mode: 'no-cors'`，前端一律視為成功

## report.html 結構

- 從 URL `?phone=` 自動預填電話
- 電話自動格式化、末5碼僅允許數字
- POST `{ type: 'report', phone, last5, amount, paid_at }`

### CSS 設計
- 黑金色系（--gold: #C9A84C）
- 字型：Playfair Display（標題）+ Noto Serif TC（內文）
- 頂部金色漸層線（`.card::before`）
- 送出中：spinner 動畫 + button disabled 狀態

## Google Apps Script 程式碼（兩階段對帳版）

```javascript
const SHEET_ID = '1Hua56eTicWv5zkOaDGU57fbi3Jtq_30qDCYjUC1oAW8';
const ORDER_SHEET = '訂單';
const REPORT_SHEET = '匯款回報';

function doPost(e) {
  const d = JSON.parse(e.postData.contents);
  const ss = SpreadsheetApp.openById(SHEET_ID);
  const ts = new Date().toLocaleString('zh-TW', { timeZone: 'Asia/Taipei' });

  if (d.type === 'report') {
    const sheet = getOrCreateSheet(ss, REPORT_SHEET,
      ['時間戳記', '電話', '末5碼', '匯款金額', '匯款時間']);
    sheet.appendRow([ts, d.phone, d.last5, d.amount, d.paid_at || '']);
  } else {
    // 預設為訂單
    const sheet = getOrCreateSheet(ss, ORDER_SHEET,
      ['時間戳記', '姓名', '電話', '數量(盒)', '總金額', '付款方式', '備註']);
    sheet.appendRow([ts, d.name, d.phone, d.qty, d.total, d.payment || '', d.note || '']);
  }

  return ContentService
    .createTextOutput(JSON.stringify({ result: 'success' }))
    .setMimeType(ContentService.MimeType.JSON);
}

function getOrCreateSheet(ss, name, headers) {
  let sheet = ss.getSheetByName(name);
  if (!sheet) {
    sheet = ss.insertSheet(name);
    sheet.appendRow(headers);
    sheet.getRange(1, 1, 1, headers.length).setFontWeight('bold');
  } else if (sheet.getLastRow() === 0) {
    sheet.appendRow(headers);
    sheet.getRange(1, 1, 1, headers.length).setFontWeight('bold');
  }
  return sheet;
}
```

## Sheet 結構（兩個分頁）

**「訂單」分頁**

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| 時間戳記 | 姓名 | 電話 | 數量(盒) | 總金額 | 付款方式 | 備註 |

**「匯款回報」分頁**

| A | B | C | D | E |
|---|---|---|---|---|
| 時間戳記 | 電話 | 末5碼 | 匯款金額 | 匯款時間 |

### 對帳建議

在「訂單」分頁加一欄 H（末5碼），用 VLOOKUP 對：
```
=VLOOKUP(C2, 匯款回報!B:C, 2, FALSE)
```
有匹配的就會自動顯示末5碼，N/A 的就是還沒匯款的。

## 修改指引

### 改售價
`index.html` 內 `const price = 580;`

### 改商品資訊
`.info-row` 區塊內的文字直接改。

### 填入銀行帳號（取代 placeholder）
`index.html` 需要改 **兩處**：
1. `.bank-info` 區塊（表單內選「匯款」時展開的預覽）
2. `showSuccess()` 函式內的 `payBlock` HTML（送出後成功畫面）

`report.html` 不需要改（不顯示銀行資訊）。

### 換 hero 圖
取代 `hero.jpg`，或改 `<img src="hero.jpg">` 的路徑。建議圖檔 < 500KB、寬度 ≥ 800px。

### 新增訂單欄位
1. `index.html` 加 HTML input 欄位
2. `submitOrder()` 的 JSON body 加上新欄位
3. Apps Script `doPost` 內訂單 branch 的 `appendRow` 加上對應欄位 + 更新 headers
4. 重新部署 Apps Script

### 重新部署 Apps Script
Apps Script → 部署 → 管理部署 → 編輯（鉛筆圖示）→ 版本選「新版本」→ 部署

## 注意事項

- Apps Script 每次修改後**必須重新部署新版本**，URL 不變但版本需更新
- `no-cors` 模式下前端無法判斷後端是否真的寫入成功，如需確認請直接查 Sheet
- GitHub Pages 更新後約需 1-2 分鐘生效
- Apps Script 免費版每日寫入上限 20,000 行，一般預購量不會觸及
