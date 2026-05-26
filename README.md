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
├── index.html          # 表單主體（前端全部邏輯在此）
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

## index.html 結構

### HTML 區塊
- `.header` — 商品名稱、badge
- `.info-row` — 規格／人數／售價三欄
- `.notice` — 當天食用提醒
- `#orderForm` — 表單本體（姓名、電話、取貨日期、數量、備註、總金額）
- `#successMsg` — 送出成功後顯示的確認畫面

### JavaScript 邏輯
- `SHEET_URL` — Apps Script endpoint，唯一需要改的設定值
- `changeQty(delta)` — 數量 +/- 並即時更新總金額
- `submitOrder()` — 驗證欄位 → fetch POST → 顯示成功畫面
- `showSuccess()` — 渲染訂單確認摘要
- 送出使用 `mode: 'no-cors'`，因為 Apps Script 不回傳 CORS header，無法讀取 response，一律視為成功

### CSS 設計
- 黑金色系（--gold: #C9A84C）
- 字型：Playfair Display（標題）+ Noto Serif TC（內文）
- 頂部金色漸層線（`.card::before`）
- 送出中：spinner 動畫 + button disabled 狀態

## Google Apps Script 程式碼

```javascript
const SHEET_ID = '1Hua56eTicWv5zkOaDGU57fbi3Jtq_30qDCYjUC1oAW8';

function doPost(e) {
  const sheet = SpreadsheetApp
    .openById(SHEET_ID)
    .getActiveSheet();

  if (sheet.getLastRow() === 0) {
    sheet.appendRow(['時間戳記', '姓名', '電話', '取貨日期', '數量(盒)', '總金額', '備註']);
    sheet.getRange(1, 1, 1, 7).setFontWeight('bold');
  }

  const d = JSON.parse(e.postData.contents);
  sheet.appendRow([
    new Date().toLocaleString('zh-TW', { timeZone: 'Asia/Taipei' }),
    d.name,
    d.phone,
    d.pickup_date,
    d.qty,
    d.total,
    d.note || ''
  ]);

  return ContentService
    .createTextOutput(JSON.stringify({ result: 'success' }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

## Sheet 欄位結構

| A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|
| 時間戳記 | 姓名 | 電話 | 取貨日期 | 數量(盒) | 總金額 | 備註 |

## 修改指引

### 改售價
`index.html` 第 349 行：
```js
const price = 580;
```

### 改商品資訊
`.info-row` 區塊內的文字直接改。

### 新增欄位
1. `index.html` 加 HTML input 欄位
2. `submitOrder()` 的 JSON body 加上新欄位
3. Apps Script `doPost` 的 `sheet.appendRow` 加上對應欄位
4. 重新部署 Apps Script（每次改 script 都要重新部署新版本）

### 重新部署 Apps Script
Apps Script → 部署 → 管理部署 → 編輯（鉛筆圖示）→ 版本選「新版本」→ 部署

## 注意事項

- Apps Script 每次修改後**必須重新部署新版本**，URL 不變但版本需更新
- `no-cors` 模式下前端無法判斷後端是否真的寫入成功，如需確認請直接查 Sheet
- GitHub Pages 更新後約需 1-2 分鐘生效
- Apps Script 免費版每日寫入上限 20,000 行，一般預購量不會觸及
